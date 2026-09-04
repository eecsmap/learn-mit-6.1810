---
layout: default
title: 环境搭建
nav_order: 3
permalink: /setup/
---

# 环境搭建
{: .no_toc }

Machine: **Ubuntu 26.04 (x86_64)**. Goal: build + boot xv6-riscv under QEMU,
and attach GDB. Status: **done and verified 2026-09-04** (see Run log).

<details open markdown="block">
  <summary>目录</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## 1. What the course requires

From https://pdos.csail.mit.edu/6.1810/2026/tools.html :

- QEMU 7.2 or newer (RISC-V system emulation)
- RISC-V GCC + binutils cross toolchain (`riscv64-linux-gnu-*`)
- GDB 8.3+ with RISC-V support (`gdb-multiarch`)
- `git`, `build-essential` (make, gcc, etc.)

## 2. Install the toolchain (Ubuntu/Debian)

```bash
sudo apt-get update
sudo apt-get install -y \
    git build-essential gdb-multiarch bc \
    qemu-system-riscv qemu-system-misc \
    gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu
```

Package notes:
- **`qemu-system-riscv`** provides `qemu-system-riscv64`. On Ubuntu 26.04 the
  course's `qemu-system-misc` package NO LONGER contains the RISC-V binary —
  RISC-V was split into its own package. If you only install `qemu-system-misc`
  you get `qemu-system-riscv64: command not found`.
- `gcc-riscv64-linux-gnu` + `binutils-riscv64-linux-gnu` provide the
  `riscv64-linux-gnu-{gcc,ld,objdump,objcopy,...}` cross tools xv6's Makefile
  looks for (it also accepts `riscv64-unknown-elf-*`).
- `gdb-multiarch` is the debugger; xv6's `make qemu-gdb` uses `gdb-multiarch`
  when a plain `riscv64-*-gdb` is absent.
- `bc` is used by a Makefile arithmetic check for the `newfs.img` target.
  Without it you get a harmless `bc: not found` + `Error 1 (ignored)` line.

### Versions actually installed here (2026-09-04)

| Tool | Version |
|------|---------|
| `qemu-system-riscv64` | 10.2.1 (Debian 1:10.2.1+ds-1ubuntu3.2) |
| `riscv64-linux-gnu-gcc` | 15.2.0 |
| `gdb-multiarch` | 17.1 |
| `git` | 2.53.0 |
| `make` | 4.4.1 |
| `python3` | 3.14.4 |

Verify:
```bash
qemu-system-riscv64 --version      # >= 7.2.0
riscv64-linux-gnu-gcc --version
gdb-multiarch --version            # >= 8.3
```

## 3. Get the lab code

Fall 2026 **reuses the 2025 lab repo** (`xv6-labs-2025`). Confirmed on the
2026 lab page.

```bash
cd ~/projects/main/learn_mit_6.1810
git clone git://g.csail.mit.edu/xv6-labs-2025
cd xv6-labs-2025
```

- The `git://` (TCP 9418) clone worked from this network. If it hangs for you,
  9418 is firewalled — no official HTTPS mirror is documented; use another
  network or a tunnel.
- The default branch **is already `util`** (the first lab). `git branch -a`
  shows the rest: `syscall pgtbl traps cow net lock fs mmap` (+ `riscv` base).
- Branch-per-lab: `git checkout <lab>` to start each one; your commits ride on
  top of that branch. See lab spec / starter diff with `git diff origin/<lab>`.

## 4. ⚠️ Required fix for Ubuntu 24.04+/26.04 + QEMU 10  (THE gotcha)

**Symptom:** `make qemu` builds fine, QEMU launches, but there is **zero
console output** — no "xv6 kernel is booting" — and `Ctrl-a x` is the only way
out.

**Cause:** `riscv64-linux-gnu-gcc` 15 on Ubuntu 26.04 defaults to a rich
`-march` (RVA23-ish: `Zba/Zbb/Zbs/Zcb/V/...`). xv6's Makefile does **not** pin
`-march`, so the assembler encodes `mul a0,a0,a1` in `kernel/entry.S` as the
compressed **`c.mul`** (Zcb extension). The default QEMU `virt` CPU does not
implement Zcb → illegal instruction *before* the trap vector is installed →
the hart resets to `0x1000` in a loop. Diagnosed with:
`qemu-system-riscv64 ... -d in_asm` (shows execution looping entry → reset).

**Fix (committed here as the first commit on `util`):** pin the classic xv6
target. In `Makefile`:

```make
CFLAGS += -mcmodel=medany
CFLAGS += -march=rv64gc -mabi=lp64d      # <-- added

# and the assembly rule:
$K/%.o: $K/%.S
	$(CC) -march=rv64gc -mabi=lp64d -g -c -o $@ $<    # <-- was: $(CC) -g -c ...
```

Then `make clean && make qemu`. Verify the fix landed:
```bash
riscv64-linux-gnu-objdump -d kernel/entry.o | grep mul
#   ... 02b50533   mul  a0,a0,a1     <-- plain 'mul', NOT 'c.mul'
```

(Alternative without editing the Makefile: run QEMU with an extended CPU,
`make qemu QEMU='qemu-system-riscv64 -cpu rv64,zba=true,zbb=true,zbs=true,zcb=true'`
— fragile, the Makefile edit is cleaner and `make grade` uses it too.)

## 5. Boot xv6 —  `make qemu`   ✅ verified

```bash
make qemu
```

Verified output:
```
xv6 kernel is booting

hart 1 starting
hart 2 starting
init: starting sh
$ ls
.              1 1 1024
..             1 1 1024
README         2 2 2425
...
$ echo hello-from-xv6
hello-from-xv6
```

At `$` xv6 is live. Try `ls`, `cat README`, `echo hi`, `usertests` (long).

**Quit QEMU:** `Ctrl-a` then `x`.
`Ctrl-a c` = QEMU monitor toggle · `Ctrl-a h` = escape help.

Notes:
- First build after `make clean` prints
  `mv: cannot stat 'fs.img' ... newfs.img Error 1 (ignored)` — harmless race in
  the `newfs.img` target; `fs.img` is then built normally. It also drops a
  stray `fs.img.bk` you can delete.
- The linker warns `kernel/kernel has a LOAD segment with RWX permissions` —
  expected for xv6, ignore.
- Fewer cores: `make qemu CPUS=1`.

## 6. Debug with GDB —  `make qemu-gdb`   ✅ verified

GDB port here = `expr $(id -u) % 5000 + 25000` = **26000** (uid 1000).
`make print-gdbport` prints it.

One-time: trust the repo's generated `.gdbinit`:
```bash
cd ~/projects/main/learn_mit_6.1810/xv6-labs-2025
echo "add-auto-load-safe-path $(pwd)/.gdbinit" >> ~/.gdbinit
```

Terminal 1:
```bash
make qemu-gdb        # prints "*** Now run 'gdb' in another window." then freezes (-S)
```

Terminal 2, same dir:
```bash
gdb-multiarch        # reads ./.gdbinit: sets riscv:rv64, symbol-file kernel/kernel,
                     # target remote 127.0.0.1:26000
```

Verified session:
```
(gdb) break main
Breakpoint 1 at 0x8000031c: file kernel/main.c, line 13.
(gdb) continue
Thread 2 hit Breakpoint 1, main () at kernel/main.c:13
13        if(cpuid() == 0){
(gdb) bt
#0  main () at kernel/main.c:13
(gdb) info registers pc sp
pc  0x8000031c  <main+8>
sp  0x8001d430  <stack0+8160>
```

The `warning: No executable has been specified ...` line before `.gdbinit`
runs `symbol-file` is benign.

Quit gdb: `quit`. Stop QEMU: `Ctrl-a x` in terminal 1.

## 7. Run the grader

```bash
make grade                      # all tests for the current branch
make GRADEFLAGS=sleep grade     # only tests matching "sleep"
./grade-lab-util sleep          # direct
```

Needs `python3` (present: 3.14.4). NOTE: `make grade` compiles the test user
programs, so it only runs once you've at least created the files the lab asks
for (e.g. `user/sleep.c` and its `UPROGS` entry) — on a pristine branch it
errors at the build step. Not part of env verification.

---

## Run log

- 2026-09-03: Documented setup. Toolchain not yet installed (`apt` needed
  interactive sudo). Directory + docs scaffolded.
- 2026-09-04: Passwordless sudo enabled by user. Installed toolchain. Had to
  add **`qemu-system-riscv`** (Ubuntu 26.04 split it out of `qemu-system-misc`)
  and `bc`. Cloned `xv6-labs-2025` via `git://` (worked). Hit the **no-console-
  output** boot bug → root-caused to gcc-15 emitting `c.mul` (Zcb) that the
  default QEMU `virt` CPU rejects → fixed by pinning `-march=rv64gc -mabi=lp64d`
  in the Makefile (committed as `b41c002` on `util`).
  - `make qemu` ✅ boots to shell; `ls`, `echo` work; `Ctrl-a x` quits.
  - `make qemu-gdb` + `gdb-multiarch` ✅ connects on :26000, `break main` hits.
  - Environment fully verified.
