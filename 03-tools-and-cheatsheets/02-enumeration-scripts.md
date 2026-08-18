# Enumeration Tools Reference

> The tools that turn hours of manual checking into minutes. Know what each one does, where to get it, and — critically — **what it might trigger on a monitored/EDR-protected system**, since "just run winPEAS" isn't always the right call on a real engagement.

---

## Windows

| Tool | Purpose | Notes |
|---|---|---|
| **winPEAS** (`winPEAS.exe` / `.bat`) | All-in-one automated enumeration: services, registry, scheduled tasks, tokens, creds, unquoted paths, AlwaysInstallElevated, etc. | Very noisy — heavily signatured by most AV/EDR. Great for CTFs, risky for stealthy real engagements. |
| **PowerUp.ps1** | PowerShell module, `Invoke-AllChecks` runs the full suite of common Windows privesc checks | Also well-known to AV; consider AMSI bypass awareness if needed on real engagements (only with authorization). |
| **Seatbelt** | .NET-based situational awareness / enumeration tool (part of the GhostPack suite) | More modular than winPEAS — can run only specific checks to reduce noise. |
| **accesschk64.exe / accesschk.exe** (Sysinternals) | Query effective ACLs/permissions on files, folders, services, registry keys | **Signed by Microsoft** — rarely flagged, which is exactly why it's used constantly in real engagements, including this lab. |
| **SharpUp** | C# port of key PowerUp checks, for environments where PowerShell is restricted/logged | Good AMSI/logging evasion alternative. |
| **PrintSpoofer / RoguePotato / GodPotato** | Exploit `SeImpersonatePrivilege` to get SYSTEM almost instantly | Only useful if `whoami /priv` shows the relevant privilege enabled. |

---

## Linux

| Tool | Purpose | Notes |
|---|---|---|
| **LinPEAS** (`linpeas.sh`) | The Linux equivalent of winPEAS — checks SUID, sudo, cron, capabilities, kernel version, writable files, and more, with color-coded severity | Very thorough, but large and can be slow on constrained systems. |
| **linux-smart-enumeration (LSE)** | Similar to LinPEAS but with adjustable verbosity levels (`-l0/1/2`) | Good when you want a quicker, less overwhelming first pass. |
| **linuxprivchecker.py** | Older, Python-based, lighter-weight alternative | Useful when you can't get a compiled/bash tool to run. |
| **pspy** (`pspy32` / `pspy64`) | Watches `/proc` for new process creation **without needing root** | THE tool for catching cron jobs and scheduled tasks in the act, even when you can't read the crontab files directly. |
| **GTFOBins** (web resource, not a tool per se) | Lookup table for SUID/sudo/capabilities abuse of common Unix binaries | https://gtfobins.github.io/ — always cross-reference any interesting binary you find here. |
| **LOLBAS** (Windows equivalent of GTFOBins) | "Living Off The Land Binaries and Scripts" for Windows | https://lolbas-project.github.io/ |

---

## General Workflow With These Tools

1. **Get an initial low-priv shell** (out of scope for this repo — covered elsewhere, e.g. web app exploitation, phishing, etc.)
2. **Stabilize the shell** (especially on Linux — see the cheatsheet's shell stabilization section).
3. **Transfer and run an automated enumeration tool** — but always **read its output carefully rather than skimming**; these tools produce a LOT of text, and the real finding is often buried.
4. **Cross-reference flagged items** against the manual technique notes in this repo (or GTFOBins/LOLBAS) to understand *why* something was flagged and *how* to actually exploit it.
5. **Verify manually** before committing to an exploitation path — automated tools sometimes produce false positives (e.g., flagging a "writable" directory that's actually owned by a group you're not in).

---

## A Note on Tool Detection

On a real, monitored engagement (not a CTF), running a giant known-signature script like LinPEAS or winPEAS unmodified is likely to trigger EDR alerts. In those scenarios:
- Use **manual, native OS commands** (as demonstrated throughout this repo's technique notes) — they blend in far better with normal admin activity.
- If you must use a known tool, consider **obfuscation, AMSI bypass techniques (Windows)**, or hosting a modified/renamed copy — **only within the scope of an authorized, signed engagement.**
- Always prioritize **understanding the underlying technique** over tool dependency — this is the difference between a script-kiddie and a professional penetration tester.
