# 🛡️ Privilege Escalation Mastery Notes

> **A structured, exam-and-lab-ready knowledge base on Windows & Linux Privilege Escalation**, built from real hands-on lab sessions (unquoted service path exploitation, Linux PrivEsc room walkthroughs) and expanded with expert-level theory, defensive guidance, and cheat sheets.

This repo is written the way a mentor would explain it to a junior penetration tester or OSCP/CEH student — **not just "run this command"**, but *why* it works, *what's happening under the hood*, and *how a blue teamer would catch/stop you*.

---

## 🎯 Who This Is For

- Students preparing for **OSCP, eJPT, CEH, PNPT, THM/HTB paths**
- Anyone who just finished a hands-on lab and wants **retention-focused notes**
- SOC analysts/blue teamers who want to understand attacker tradecraft to build better detections

## ⚖️ Ethics & Legal Notice

Everything in this repo is for **authorized, legal use only** — your own lab VMs, CTF platforms (TryHackMe, HackTheBox), or environments you have **explicit written permission** to test. Unauthorized access to computer systems is illegal in virtually every jurisdiction (e.g., Computer Fraud and Abuse Act in the US, Computer Misuse Act in the UK). Treat every technique here as a **skill for defense-through-offense understanding**, not a tool for harm.

---

## 📚 Repo Structure

```
privesc-mastery-notes/
├── README.md                              ← you are here
├── 01-windows-privesc/
│   └── 01-unquoted-service-path.md        ← full walkthrough + theory
├── 02-linux-privesc/
│   ├── 01-lab-setup-and-enumeration.md
│   ├── 02-etc-shadow-and-passwd-abuse.md
│   ├── 03-sudo-misconfigurations.md
│   └── 04-cron-job-privesc.md
├── 03-tools-and-cheatsheets/
│   ├── 01-master-cheatsheet.md
│   └── 02-enumeration-scripts.md
└── 04-lab-writeups/
    └── 01-session-log-summary.md
```

## 🧭 Suggested Learning Path

1. Start with **`03-tools-and-cheatsheets/01-master-cheatsheet.md`** to get the big-picture mental model of "what privilege escalation actually is."
2. Work through **`01-windows-privesc/`** — a full, step-by-step unquoted service path exploitation chain (msfvenom → netcat → service abuse → SYSTEM shell).
3. Move to **`02-linux-privesc/`** — covers weak file permissions on `/etc/shadow` & `/etc/passwd`, sudo misconfigurations, and cron job abuse, using a TryHackMe-style Debian VM.
4. Use **`04-lab-writeups/`** as a template for documenting your *own* labs — professional documentation is a core pentesting skill, not an afterthought.

## 🔑 Core Concepts You'll Master

| Concept | Where |
|---|---|
| Windows service permission abuse (unquoted paths) | `01-windows-privesc/01-unquoted-service-path.md` |
| Payload generation & delivery (`msfvenom`, HTTP server, netcat) | `01-windows-privesc/01-unquoted-service-path.md` |
| Windows privilege/token enumeration (`whoami /priv`) | `01-windows-privesc/01-unquoted-service-path.md` |
| ACL analysis with `accesschk` | `01-windows-privesc/01-unquoted-service-path.md` |
| Linux world-readable `/etc/shadow` abuse + hash cracking (`john`) | `02-linux-privesc/02-etc-shadow-and-passwd-abuse.md` |
| Writable `/etc/passwd` → instant root | `02-linux-privesc/02-etc-shadow-and-passwd-abuse.md` |
| Sudo shell escapes & GTFOBins mentality | `02-linux-privesc/03-sudo-misconfigurations.md` |
| Cron job & PATH hijacking | `02-linux-privesc/04-cron-job-privesc.md` |

---

## 🧠 How to Use This Repo for Real Retention

- **Don't just read — re-type every command** in your own lab. Muscle memory beats reading.
- After each `.md` file, close it and try to **write the attack chain from memory** on paper. That's the real test of understanding.
- Every file ends with a **"Blue Team / Defensive Notes"** section — read it even if you only care about offense. Understanding detection makes you a sharper attacker.

---

*Compiled from hands-on lab sessions covering Windows Service PrivEsc and Linux PrivEsc (TryHackMe-style Debian VM), enriched with structured theory for durable learning.*
