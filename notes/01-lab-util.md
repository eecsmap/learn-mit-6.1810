---
layout: default
title: "Lab 1: util"
parent: Labs
nav_order: 1
permalink: /labs/util/
---

# Lab: util
{: .no_toc }

分支 `util` · [官方 spec](https://pdos.csail.mit.edu/6.1810/2026/labs/util.html) · xv6 book ch.1

<details open markdown="block">
  <summary>目录</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

{: .warning }
> 下面每个练习的**完整代码都在折叠区里**。原理讲解是公开的,代码不是。
> 如果你也在学这门课 —— 先自己写,写不出来再看思路,最后才展开代码对照。

2026 版的 util lab 是 **5 个练习**(和老版本不一样了,没有 pingpong/primes/xargs):

| # | 练习 | 文件 | 分值 |
|---|------|------|------|
| 1 | sleep | `user/sleep.c`(新建) | 5 + 5 + 10 |
| 2 | sixfive | `user/sixfive.c`(新建) | 10 + 10 + 10 |
| 3 | memdump | `user/memdump.c`(给了骨架) | 10 + 10 |
| 4 | find | `user/find.c`(新建) | 10 + 10 + 10 |
| 5 | find -exec | 扩展 `user/find.c` | 10 + 10 + 10 |
| — | time.txt | `time.txt` | 1 |

---

## 背景:xv6 里的「用户程序」到底是什么

xv6 = 一个很小的内核 + 十几个用户程序,程序住在一个小文件系统镜像 `fs.img` 里。
所谓「程序」就是一个 ELF 二进制,shell `fork` + `exec` 它。

### 一个用户程序的生命周期

```
user/sleep.c ──gcc──▶ user/sleep.o ──ld──▶ user/_sleep   (ELF, 入口 _main)
                                               │
                       mkfs 把它烤进 ────── fs.img ─────▶ 在 xv6 里叫文件 "sleep"
                                               │
      sh:  fork();  exec("sleep", ["sleep","10"])  ──▶  你的 main(argc=2, argv)
```

让它出现需要两样东西:

- **`user/sleep.c`** —— 源码
- **`Makefile` 里 `UPROGS` 加一行** —— 这是「要编译并放进镜像的程序清单」

{: .gotcha }
> 忘了加 `UPROGS` 那一行,**不会有任何编译错误**,只是在 xv6 shell 里
> 敲命令时得到 `exec sleep failed`。这是新手最常见的第一个坑。

### 没有 libc

用户程序能用的只有:

| 来源 | 提供什么 |
|------|---------|
| `kernel/types.h` | `uint` / `uchar` / `uint64` —— xv6 版的 `<stdint.h>` |
| `user/user.h` | 所有系统调用原型 + 下面这些库函数的原型 |
| `user/ulib.c` | `strcpy` `strcmp` `strlen` `strchr` `memset` `memmove` `atoi` `gets` `stat` |
| `user/printf.c` | `printf` / `fprintf`(建在 `write` 系统调用上) |
| `user/umalloc.c` | `malloc` / `free`(建在 `sbrk` 系统调用上) |
| `user/usys.S` | 由 `user/usys.pl` **生成**的系统调用汇编桩 |

每个程序链接 `ULIB = ulib.o usys.o printf.o umalloc.o`。
`main` **必须**以 `exit(n)` 结束 —— 没有 C 运行时会接住 `main` 的返回。

### 系统调用是怎么走的

以 `pause(10)` 为例,完整链路:

```
user/sleep.c        pause(10)
user/usys.S (生成)   pause:  li a7, 13 (SYS_pause)
                            ecall                    ← 陷入内核
 ───────────────────────────────────────────────────────────
kernel/trampoline.S  保存寄存器,切换到内核页表
kernel/trap.c        usertrap() 判断 scause = "U-mode 的 ecall" → syscall()
kernel/syscall.c     num = p->trapframe->a7 = 13
                     syscalls[13]() → sys_pause()
kernel/sysproc.c     sys_pause():
                       argint(0, &n);              // 从保存的 a0 取参数
                       acquire(&tickslock);
                       while(ticks - ticks0 < n)
                         sleep(&ticks, &tickslock); // 让出 CPU,不空转
kernel/trap.c        时钟中断 clockintr(): ticks++; wakeup(&ticks);
 ───────────────────────────────────────────────────────────
                     sret → 返回值放 a0 → 用户态 pause() 返回
```

这条链路里值得记住的三件事:

1. **调用约定**:系统调用号放 `a7`,执行 `ecall`,返回值在 `a0`。
   `argint(0, &n)` 就是去 trapframe 里把用户当时的 `a0` 取出来。
2. **`ticks`** 是一个全局计数器,由时钟中断处理函数递增,QEMU 里约 100ms 一个 tick。
3. **阻塞不等于空转**。`sleep()` / `wakeup()` 是 xv6 的条件变量原语(book ch.7),
   `sys_pause` 睡在 `&ticks` 这个地址上,时钟中断 `wakeup(&ticks)` 把它叫醒。

{: .gotcha }
> 经典 xv6 里这个系统调用就叫 `sleep`。**2026 的代码把它改名成 `pause` 了**,
> 这样用户程序才能叫 `sleep` 而不冲突。看 `user/user.h` 第 25 行:`int pause(int);`
> 在用户程序里写 `sleep(10)` 是编译不过的。

---

## Exercise 1 — sleep
{: .d-inline-block }

已完成 3/3
{: .label .label-green }

**任务:** 写 `user/sleep.c`,一个用户态的 `sleep`,暂停用户指定的 tick 数;
参数缺失时报错。

### 需要理解的概念

这题本身两行代码就能写完,MIT 把它放第一个是为了让你走通**整条工具链**:

- 新建源文件 → 改 `UPROGS` → `make qemu` → 在 xv6 shell 里能跑
- `argc` / `argv` 的 UNIX 约定(`argv[0]` 是程序名)
- fd 0/1/2 = stdin/stdout/stderr,诊断信息要写 fd 2
- `exit(0)` 成功 / `exit(非0)` 失败
- 触发一次真实的系统调用,理解上面那条链路

### 思路

1. 检查 `argc` —— 恰好要一个参数,所以 `argc` 应该等于 2
2. 参数不对:用 `fprintf(2, ...)` 打 usage 到 stderr,然后 `exit(1)`
3. 用 `atoi()`(在 `user/ulib.c`)把 `argv[1]` 转成整数
4. 调用 `pause()`(**不是** `sleep()`)
5. `exit(0)`

<details markdown="block">
<summary><strong>▶ 展开:完整代码 + 逐行讲解</strong></summary>

```c
// user/sleep.c
#include "kernel/types.h"
#include "user/user.h"

int
main(int argc, char *argv[])
{
  if(argc != 2){
    fprintf(2, "usage: sleep <ticks>\n");
    exit(1);
  }

  pause(atoi(argv[1]));
  exit(0);
}
```

逐行:

| 代码 | 为什么 |
|------|--------|
| `#include "kernel/types.h"` | 给出 `uint` `uchar` `uint64` 等类型。几乎每个 xv6 程序都以它开头。 |
| `#include "user/user.h"` | 系统调用桩(`pause` `exit` `write`…)和 ulib/printf(`atoi` `fprintf`…)的原型。这是 xv6 的「libc 头文件」。 |
| `main(int argc, char *argv[])` | UNIX 约定。`argv[0]` 是程序名,真正的参数从 `argv[1]` 开始,`argc` 把程序名也算进去。 |
| `if(argc != 2)` | 要求恰好一个参数。`sleep` → `argc==1`;`sleep 10` → `argc==2`。第一个 grader 测试就是不带参数跑,期待有非空的错误输出。 |
| `fprintf(2, ...)` | 写到 **fd 2 = stderr**,UNIX 里诊断信息的去处。xv6 的 `printf(x)` 等价于 `fprintf(1, x)`。 |
| `exit(1)` | 非 0 = 失败。`exit` 是系统调用,标了 `__attribute__((noreturn))`,后面的代码不会执行。 |
| `atoi(argv[1])` | `user/ulib.c` 里手写的:逐个吃数字字符,遇到非数字就停。所以 `atoi("xyz")` 返回 `0`。 |
| `pause(n)` | 正主。阻塞当前进程 `n` 个 tick 后返回。 |
| `exit(0)` | 成功。**必须写** —— 没有 C 运行时接住 `main` 的返回,不写就跑飞了。 |

`Makefile` 的改动(保持大致字母序):

```diff
 	$U/_sh\
+	$U/_sleep\
 	$U/_stressfs\
```

</details>

### 验证

在 xv6 里:

```
$ sleep
usage: sleep <ticks>
$ echo A; sleep 30; echo B      ← 中间能看到约 3 秒的停顿
A
B
```

grader:

```console
$ make GRADEFLAGS=sleep grade
== Test sleep, no arguments == sleep, no arguments: OK (3.3s)
== Test sleep, returns == sleep, returns: OK (0.4s)
== Test sleep, makes syscall == sleep, makes syscall: OK (0.7s)
```

### 踩的坑 / 值得记的点

- `sleep xyz` → `atoi` 返回 0 → `pause(0)` 立刻返回。grader 不测非法数字输入,
  所以够用;真实的 `sleep` 应该校验。
- 在用户程序里写 `sleep(10)` 编译不过 —— 系统调用叫 `pause`。
- 忘记加 `UPROGS` 那行:没有编译错误,只在 shell 里 `exec sleep failed`。
- 忘记 `exit()`:`main` 返回后行为未定义。

---

## Exercise 2 — sixfive

**任务:** 读入文件,把里面所有能被 5 或 6 整除的十进制数打印出来。
数字之间的分隔符是:空格、`-`、`\`、回车、制表符、换行、`.`、`/`、`,`。
文件的开头和结尾也算隐式分隔符。

提示里点名了 `strchr()` 和「一次处理一个字符」,建议的系统调用是 `open` / `read`。

**状态:** 未开始

---

## Exercise 3 — memdump

**任务:** 实现 `memdump(char *fmt, char *data)`,按格式串打印内存内容。
仓库里已经给了 `user/memdump.c` 的 `main`(包含 5 个示例和从 stdin 读 512 字节的分支),
只需要填 `memdump()` 函数体。

格式字符:

| 字符 | 含义 |
|------|------|
| `i` | 接下来 4 字节,按 32 位十进制整数打印 |
| `p` | 接下来 8 字节,按 64 位十六进制打印 |
| `h` | 接下来 2 字节,按 16 位十进制打印 |
| `c` | 接下来 1 字节,按 ASCII 字符打印 |
| `s` | 接下来 8 字节是一个指向 C 字符串的指针,打印它指向的字符串 |
| `S` | 剩下的数据是一个以 `\0` 结尾的 C 字符串,直接打印 |

这题实际在练的是 **C 的指针 / 强制类型转换 / 内存对齐布局** —— 后面做页表
(pgtbl)和文件系统时全靠这个直觉。

**状态:** 未开始

---

## Exercise 4 — find

**任务:** 实现简化版 UNIX `find`:在目录树里递归查找指定名字的文件。
递归时要跳过 `.` 和 `..`(否则无限递归)。

提示:照着 `user/ls.c` 学怎么读目录;用 `strcmp` 而不是 `==` 比较字符串;
建议的系统调用 `open` / `read` / `fstat`。

这题的核心是理解 **xv6 的目录就是一个普通文件**,内容是一串
`struct dirent { ushort inum; char name[DIRSIZ]; }`,你 `read()` 它就能拿到目录项。

**状态:** 未开始

---

## Exercise 5 — find -exec

**任务:** 给 `find` 加 `-exec cmd` 选项 —— 对每个匹配到的文件执行 `cmd file`,
而不是打印文件名。

提示:`fork` + `exec`,父进程 `wait`;`kernel/param.h` 里的 `MAXARG` 是 argv 数组上限。
用 `user/findtest.sh` 测。

这题练的是 **UNIX 进程模型的核心**:`fork` 复制进程,`exec` 替换镜像,`wait` 收尸。
book ch.1 讲的就是这个。

**状态:** 未开始

---

## Run log

- 2026-09-04 — Ex1 `sleep` 完成,3/3 grader 测试通过。
