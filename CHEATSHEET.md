---
layout: default
title: 速查表
nav_order: 2
permalink: /cheatsheet/
---

# 6.1810 / xv6 速查表
{: .no_toc }

写代码时开着这一页。所有命令都在 Ubuntu 26.04 + QEMU 10.2 上真实跑过。

<details open markdown="block">
  <summary>目录</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## Environment (one-time)

```bash
sudo apt-get update && sudo apt-get install -y \
    git build-essential gdb-multiarch bc \
    qemu-system-riscv qemu-system-misc \
    gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu

qemu-system-riscv64 --version     # need >= 7.2.0   (Ubuntu 26.04: 10.2)
riscv64-linux-gnu-gcc --version
gdb-multiarch --version           # need >= 8.3
```

{: .gotcha }
> **Ubuntu 26.04:** the RISC-V QEMU binary is in `qemu-system-riscv`, NOT
> `qemu-system-misc` (the course page is outdated). Install both.

Repo (Fall 2026 → 2025 repo):
```bash
git clone git://g.csail.mit.edu/xv6-labs-2025 && cd xv6-labs-2025
```

### ⚠️ Ubuntu 24.04+/26.04 + QEMU 10 — patch before first boot

`make qemu` gives **no console output at all**? gcc-15 emits `c.mul` (Zcb) in
`entry.S`; default QEMU `virt` CPU rejects it → silent reset loop. Fix in `Makefile`:

```make
CFLAGS += -march=rv64gc -mabi=lp64d          # add near -mcmodel=medany

$K/%.o: $K/%.S
	$(CC) -march=rv64gc -mabi=lp64d -g -c -o $@ $<   # add flags to the .S rule
```
`make clean && make qemu`. Check: `riscv64-linux-gnu-objdump -d kernel/entry.o | grep mul`
→ must be `mul`, not `c.mul`. (Already committed on this checkout.)

## QEMU

| Action | Command |
|--------|---------|
| Build + boot xv6 | `make qemu` |
| Boot, wait for GDB | `make qemu-gdb` |
| **Quit QEMU** | `Ctrl-a` `x` |
| QEMU monitor toggle | `Ctrl-a` `c` |
| QEMU escape help | `Ctrl-a` `h` |
| Fewer CPUs | `make qemu CPUS=1` |
| Rebuild clean | `make clean && make qemu` |

`Ctrl-a` means: press Ctrl+A together, release, then the next key.

## GDB (two terminals)

```
term1$ make qemu-gdb
term2$ gdb-multiarch          # reads repo .gdbinit, auto-connects
```

One-time trust of the repo's .gdbinit:
```bash
echo "add-auto-load-safe-path $(pwd)/.gdbinit" >> ~/.gdbinit
```

| Purpose | Command |
|---------|---------|
| Breakpoint | `b main` / `b syscall` / `b sys_exec` |
| Break at file:line | `b kernel/proc.c:120` |
| Continue / next / step / finish | `c` / `n` / `s` / `fin` |
| Step one instruction | `si` |
| Registers | `info reg` / `p $pc` / `p/x $sp` |
| Backtrace | `bt` |
| Examine memory | `x/16xw addr` , `x/s addr` |
| Print expr / struct | `p myvar` , `p *p` , `p p->name` |
| Source view TUI | `layout src` (`Ctrl-x a` to toggle) |
| Switch hart/CPU | `info threads` , `thread 2` |
| Watch a variable | `watch panicked` |
| Load kernel symbols (if needed) | `file kernel/kernel` |
| Quit | `q` |

Port used = `26000 + (uid % 100)` (see `.gdbinit`); irrelevant if you use the
provided `.gdbinit`.

## Git / labs workflow

| Action | Command |
|--------|---------|
| List lab branches | `git branch -a` |
| Start a lab | `git checkout util` |
| See lab spec diff | `git diff origin/util` |
| Save work | `git commit -am "message"` |
| What changed | `git status` , `git diff` |
| Grade current lab | `make grade` |
| Grade one test | `make GRADEFLAGS=sleep grade` |
| Time spent (required) | edit `time.txt` |
| Submit | `make handin` (or course submission site) |

Lab order: `util → syscall → pgtbl → traps → cow → net → lock → fs → mmap`.

## xv6 shell (inside QEMU)

```
ls              cat FILE          echo hello        grep pattern FILE
mkdir d         rm FILE          wc FILE           kill PID
sleep N         find . name      usertests         forktest
```
- No `cd` as external binary — `cd` is a shell builtin.
- Pipes and redirection work: `ls | wc -l`, `echo hi > f`.
- `usertests` = kernel stress/regression suite.
- `Ctrl-p` in xv6 dumps process list (procdump), if enabled.

## Build internals (good to know)

| File | Role |
|------|------|
| `Makefile` | Top-level build; `TOOLPREFIX` autodetect, QEMU opts, `qemu`/`qemu-gdb`/`grade` targets |
| `kernel/` | Kernel source; entry `kernel/entry.S` → `start()` → `main()` |
| `user/` | User programs + `user/user.h`, `user/usys.pl` (syscall stubs) |
| `kernel/syscall.c` | Syscall dispatch table `syscalls[]` |
| `kernel/syscall.h` | `SYS_*` numbers |
| `mkfs/mkfs.c` | Builds `fs.img` root filesystem image |
| `UPROGS` in Makefile | List of user programs baked into the image |

Adding a syscall touches: `user/user.h`, `user/usys.pl`, `kernel/syscall.h`,
`kernel/syscall.c`, `kernel/sysproc.c` (or new file), Makefile `UPROGS` if a
new user program.

QEMU flags xv6 uses (from Makefile): `-machine virt -bios none -kernel
kernel/kernel -m 128M -smp 3 -nographic -global virtio-mmio.force-legacy=false
-drive file=fs.img,... -device virtio-blk-device,...`

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| **`make qemu`: no output at all, no `$`** | gcc-15 `c.mul`/Zcb vs QEMU `virt`. Pin `-march=rv64gc -mabi=lp64d` (see Environment section). |
| `make qemu` hangs, no prompt | Wrong/old QEMU; check `>= 7.2`. Try `make clean`. |
| `riscv64-linux-gnu-gcc: not found` | Install `gcc-riscv64-linux-gnu binutils-riscv64-linux-gnu`. |
| `qemu-system-riscv64: not found` | Install **`qemu-system-riscv`** (not just `qemu-system-misc`). |
| `bc: not found` / `newfs.img Error 1 (ignored)` | `sudo apt-get install bc`. Harmless otherwise. |
| `mv: cannot stat 'fs.img'` on first build | Harmless race in `newfs.img` target; `fs.img` builds anyway. Delete stray `fs.img.bk`. |
| gdb: `.gdbinit auto-loading declined` | `echo "add-auto-load-safe-path $(pwd)/.gdbinit" >> ~/.gdbinit` |
| gdb: `Remote connection closed` | QEMU side not running / already exited; restart `make qemu-gdb` first. |
| `git clone` hangs | `git://` port 9418 blocked by network/firewall. |
| Stuck in QEMU, terminal frozen | `Ctrl-a` then `x`. If truly stuck: kill from another shell `pkill qemu-system-ri`. |
| Port already in use on `qemu-gdb` | Old QEMU alive: `pkill qemu-system-riscv64`. |
| `make grade` fails to run | Needs `python3` (present on Ubuntu). |

---
_Last updated: 2026-09-04_
