# Race Condition (TOCTOU) — Set-UID Root Exploitation & Defense (SEED Lab)

An end-to-end study of a **Time-of-Check to Time-of-Use (TOCTOU)** race condition
in a Set-UID root program: first exploiting it to gain root, then applying and
evaluating two independent defenses. The project covers both the attack and the
mitigations, with an explanation of *why* each defense works.

> **Author:** Rayan Kurdi — BSc Cybersecurity, University of Jeddah
> **Lab:** SEED Race Condition Vulnerability Lab (Software Security)
> **Skills:** TOCTOU / race conditions · Set-UID & Linux privilege model · symlink attacks · least privilege · C · shell scripting

---

## The vulnerability

The target `vulp.c` is a root-owned Set-UID program that appends user input to
`/tmp/XYZ`. It guards the write with an `access()` check, then opens the file
with `fopen()`:

```c
if (!access(fn, W_OK)) {     // CHECK  (uses real UID)
    fp = fopen(fn, "a+");    // USE    (runs with root's effective UID)
    ...
}
```

Between the **check** and the **use** there is a time window. If an attacker can
swap `/tmp/XYZ` for a symlink to a protected file (e.g. `/etc/passwd`) inside
that window, the root-privileged write lands on the protected file — a classic
TOCTOU race.

## Task 1 — Exploit (attack succeeds)

1. Disabled Ubuntu's protections (`fs.protected_symlinks=0`, `fs.protected_regular=0`).
2. Compiled `vulp.c` as a root-owned Set-UID binary (`chmod 4755`).
3. Ran two scripts in parallel:
   - `loop.sh` — repeatedly runs the vulnerable program.
   - `attack.sh` — repeatedly relinks `/tmp/XYZ` → `/etc/passwd`, watching the
     file's timestamp to detect a win.
4. The input was a crafted `/etc/passwd` entry with **UID 0** (a root-equivalent
   account).
5. On a winning race, the entry was appended to `/etc/passwd`, and switching to
   the new account gave a **root shell**.

**Why it worked:** the program holds root's effective UID for its whole lifetime,
so winning the race redirects a root write to `/etc/passwd`.

## Task 2 — Defense A: Principle of Least Privilege (attack fails)

`vulp.c` was modified to drop effective privilege before the check/use:

```c
uid_t realUID = getuid();
seteuid(realUID);            // drop root before access()/fopen()
```

Re-running the same attack produced only `NO permission`. **Why:** even when the
attacker wins the symlink race, the program no longer has root's effective
privilege at write time, so the write is denied. The TOCTOU bug still exists, but
the exploitation window is closed by minimizing privilege.

## Task 3 — Defense B: OS-level sticky-symlink protection (attack fails)

Reverted to the original code and re-enabled Ubuntu's protection
(`fs.protected_symlinks=1`). The attack failed again, for a different reason:

- `NO permission` — the kernel refuses to follow a symlink in a world-writable
  sticky directory (`/tmp`) when the link owner differs from the directory owner,
  so `access()` fails at the OS level.
- Intermittent `Segmentation fault` — when `attack.sh` deletes `/tmp/XYZ` just as
  `vulp` opens it, `fopen()` returns `NULL` and the program uses it without
  checking — a **secondary bug** (missing return-value validation) distinct from
  the race itself.

## Results summary

| Scenario | Result | Reason |
|----------|--------|--------|
| Task 1 — original code, protections off | **Attack succeeds** | Root privilege held across the whole TOCTOU window |
| Task 2 — `seteuid` least-privilege fix | Attack fails | Effective privilege dropped before the write |
| Task 3 — original code, OS protection on | Attack fails | Kernel blocks following the symlink in `/tmp` |

## Key takeaways

- Race conditions come from the **non-atomicity** of check-and-use (TOCTOU).
- Defense is strongest in layers: fix the program (**least privilege**) **and**
  enable **OS-level protections**.
- A missing `fopen()` return-value check was a separate latent bug surfaced by
  the testing — input/return validation matters independently of the race.

## Full report

Step-by-step commands, scripts, and screenshots are in
[`docs/race-condition-lab-report.pdf`](docs/race-condition-lab-report.pdf).

---

*Performed in an isolated SEED VM against a deliberately vulnerable program, with
OS protections toggled intentionally, for learning purposes only.*
