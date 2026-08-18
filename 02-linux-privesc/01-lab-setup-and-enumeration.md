# Linux Privilege Escalation — Lab Setup & Enumeration Mindset

> **Goal:** Go from a low-privileged SSH shell (`user`) to `root` on a deliberately misconfigured Debian VM, learning the *systematic enumeration mindset* that applies to every real-world Linux box you'll ever touch.

---

## 1. Lab Environment

This lab series is built around **TryHackMe's "Linux PrivEsc"** room — an intentionally vulnerable Debian VM created by **Tib3rius**, based on **Sagi Shahar's** original Linux Privilege Escalation workshop. It's one of the most widely used training boxes for this exact topic because it packages **9+ distinct misconfiguration classes** into one machine.

| Detail | Value |
|---|---|
| Target OS | Debian (intentionally misconfigured) |
| Difficulty | Medium |
| Est. time | ~75 minutes |
| Initial access | SSH (already provided — this room focuses purely on *post-exploitation* escalation, not initial access) |
| Credentials | `user` : `password321` |

### Connecting

```bash
ssh user@<MACHINE_IP>
```

If you hit a key-exchange algorithm error (very common with modern Kali connecting to an older/legacy SSH config — Debian boxes built for older labs often only offer `ssh-rsa`, which newer OpenSSH clients disable by default for security):

```bash
ssh -oHostKeyAlgorithms=+ssh-rsa user@<MACHINE_IP>
```

> 🧠 **Why does this error happen?** OpenSSH deprecated `ssh-rsa` as a *host key* algorithm (not to be confused with RSA user auth keys) in recent versions due to weaknesses in SHA-1. Older or purpose-built vulnerable VMs often haven't been updated to support newer algorithms like `rsa-sha2-256`. The flag above tells your *client* to still accept it for this one connection — never do this outside of a lab/trusted context.

### Room Task Map (what this vulnerable VM teaches)

| Task | Topic |
|---|---|
| 1 | Deploy the vulnerable Debian VM |
| 2 | Service exploits (version-specific/CVE-driven escalation) |
| 3 | Weak file permissions — readable `/etc/shadow` |
| 4 | Weak file permissions — writable `/etc/shadow` |
| 5 | Weak file permissions — writable `/etc/passwd` |
| 6 | Sudo — shell escape sequences |
| 7 | Sudo — environment variables |
| 8 | Cron jobs — file permissions |
| 9 | Cron jobs — PATH environment variable |

Each of these is covered in its own note file in this repo (see `02-`, `03-`, `04-` in this directory).

---

## 2. The Enumeration Mindset

Before touching any specific technique, internalize this: **privilege escalation is 90% enumeration, 10% exploitation.** The actual "hack" step (cracking a hash, running an exploit, abusing a cron job) is usually trivial once you've *found* the misconfiguration. Finding it is the real skill.

### The Systematic Checklist (memorize this order)

1. **Who am I? What do I have?**
   ```bash
   id
   whoami
   groups
   sudo -l          # what can I run as another user without a password (or with one I know)?
   ```

2. **System & kernel info** (for known CVEs / service exploits — Task 2's category):
   ```bash
   uname -a
   cat /etc/os-release
   ```

3. **File permission sweeps** (Tasks 3–5's category):
   ```bash
   ls -la /etc/shadow /etc/passwd
   find / -writable -type d 2>/dev/null
   find / -perm -u=s -type f 2>/dev/null   # SUID binaries
   ```

4. **Scheduled tasks** (Tasks 8–9's category):
   ```bash
   cat /etc/crontab
   ls -la /etc/cron.d/
   crontab -l
   ```

5. **Sudo configuration** (Tasks 6–7's category):
   ```bash
   sudo -l
   ```

> 💡 **Pro tip:** In real engagements, don't do all of this by hand every time — use automated enumeration scripts (**LinPEAS**, **linux-smart-enumeration (LSE)**, **linuxprivchecker.py**) to save time, but **always understand what they're checking and why**, so you're not helplessly dependent on the tool when it's flagged by AV/EDR or simply missing.

---

## 3. Quick Enumeration One-Liners Used/Referenced in This Lab

```bash
# Check shadow file permissions
ls -l /etc/shadow

# View shadow file contents (only works if world-readable — Task 3/4)
cat /etc/shadow

# Check passwd file permissions (Task 5)
ls -l /etc/passwd

# Check what you can run with sudo
sudo -l

# Look at cron jobs
cat /etc/crontab
ls -la /etc/cron.d/ /etc/cron.hourly/ /etc/cron.daily/
```

Continue to the next files in this folder for the full exploitation walkthrough of each finding.
