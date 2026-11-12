# Stack Buffer Overflow — Set-UID Root Exploitation (SEED Lab)

An end-to-end exploitation of a classic **stack-based buffer overflow** in a
Set-UID root program, following the SEED Buffer-Overflow Vulnerability Lab. The
write-up walks from confirming the vulnerability, through locating the exact
stack layout in GDB, to crafting a NOP-sled + shellcode payload that hijacks
control flow and yields a **root shell** — then verifies the system returns to a
normal state.

> **Author:** Rayan Kurdi — BSc Cybersecurity, University of Jeddah
> **Lab:** SEED Buffer-Overflow Vulnerability Lab
> **Skills:** binary exploitation · GDB · x86 assembly / shellcode · memory safety · privilege escalation

---

## Environment

| Item | Value |
|------|-------|
| OS | SEED Ubuntu 16.04 (32-bit), kernel 4.8.0-36 |
| Target | `stack.c` — vulnerable via an unchecked `strcpy()` into a 100-byte buffer |
| Privilege | Set-UID root (`rwsr-xr-x`, owner root) |
| Countermeasures disabled | ASLR, StackGuard (stack canary), non-executable stack |
| Goal | Exploit the overflow to obtain a root shell |

> All countermeasures were deliberately disabled and a deliberately vulnerable
> program was used, inside an isolated SEED VM, for educational purposes.

## Methodology

**1. Confirm the vulnerability.**
Disabled ASLR (`sysctl -w kernel.randomize_va_space=0`), compiled with stack
protector and NX off (`gcc -z execstack -fno-stack-protector`), and made the
binary Set-UID root (`chmod 4755`). 100 bytes returned properly; 160 bytes
caused a segmentation fault — confirming the 100-byte buffer overflows.

**2. Map the stack in GDB.**
With a debug build, set a breakpoint at the vulnerable function and read:
- `$ebp` → `0xbffff178`
- `&buffer` → `0xbffff10c`

The ~108-byte distance gives the offset to the saved return address (offset + 4).

**3. Build the payload.**
A Python script constructs a 300-byte `badfile`: a NOP sled (`0x90`), x86
shellcode that spawns `/bin/sh` at the end, and the saved return address
(offset 112–116) overwritten with an address landing inside the NOP sled, so the
function "returns" into the injected shellcode.

**4. Exploit & escalate.**
Running the Set-UID binary against the payload redirected execution into the
shellcode and spawned a shell. `id` confirmed `euid=0(root)` — full privilege
escalation from a normal user.

**5. Cleanup.**
A fresh session confirmed the system returned to an unprivileged state
(`uid=1000(seed)`, no `euid=0`).

## Key takeaways

- Disabling ASLR made stack addresses deterministic and the exploit reproducible.
- GDB precisely located the frame pointer and buffer, from which the return-address
  offset was derived.
- A NOP sled widened the margin for small addressing errors in the jump target.
- A single unchecked `strcpy()` in a Set-UID root binary is enough for complete
  local privilege escalation.

## Defensive lessons (why modern systems resist this)

- **ASLR** randomizes stack/library addresses, breaking hardcoded return addresses.
- **Stack canaries** detect overwrites of the saved return address before `ret`.
- **NX / DEP** marks the stack non-executable, stopping injected shellcode from running.
- **Safe APIs** — `strncpy`/bounds checks instead of `strcpy` — remove the root cause.

## Full report

Step-by-step commands, GDB output, and screenshots are in
[`docs/buffer-overflow-lab-report.pdf`](docs/buffer-overflow-lab-report.pdf).

---

*Performed in an isolated SEED VM against a deliberately vulnerable, Set-UID
program for learning purposes only.*
