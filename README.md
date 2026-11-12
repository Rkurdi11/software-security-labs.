# Software Security Labs

Hands-on low-level exploitation labs from my Software Security coursework
(SEED Labs). Each lab exploits a real memory/privilege vulnerability end-to-end
in an isolated VM, then explains the defenses that stop it.

> **Author:** Rayan Kurdi — BSc Cybersecurity, University of Jeddah
> **Skills:** binary exploitation · GDB · x86 shellcode · TOCTOU race conditions · Linux Set-UID & privilege model · least privilege

## Labs

| Lab | Vulnerability | Outcome |
|-----|---------------|---------|
| [Buffer Overflow](buffer-overflow/) | Stack-based buffer overflow in a Set-UID root program | Root shell via NOP sled + shellcode |
| [Race Condition](race-condition/) | TOCTOU race in a Set-UID root program | Root via symlink race, then two defenses evaluated |

Each folder has its own README and the full lab report with commands and
screenshots.

---

*All work was performed in isolated SEED VMs against deliberately vulnerable
programs, with OS countermeasures toggled intentionally, for learning purposes.*
