---
layout: default
title: 首页
nav_order: 1
permalink: /
---

# MIT 6.1810 学习笔记
{: .fs-9 }

自学 **Operating System Engineering (Fall 2026)** —— xv6-riscv、QEMU、GDB,
从环境搭建到每个 lab 的原理拆解。
{: .fs-6 .fw-300 }

[环境搭建]({{ site.baseurl }}/setup/){: .btn .btn-primary .mr-2 }
[速查表]({{ site.baseurl }}/cheatsheet/){: .btn }

---

## 这是什么

一个人跟着 MIT 6.1810 公开课程材料学操作系统的过程记录。不是官方内容,
也不是标准答案 —— 是**踩过的坑 + 原理拆解**。

重点记录三类东西:

1. **环境里真实出错的地方** —— 课程 tools 页面在 Ubuntu 26.04 上已经不准了,
   照着抄会得到一个「QEMU 启动但完全没有输出」的黑屏。
2. **原理链路** —— 一个 `pause(10)` 从用户态到内核再回来,中间经过哪些文件、
   哪些寄存器、哪个中断。
3. **可复用的速查表** —— QEMU / GDB / git-labs 工作流的命令,不用每次翻文档。

## 目录

| 页面 | 内容 |
|------|------|
| [环境搭建]({{ site.baseurl }}/setup/) | 工具链安装、仓库 clone、`make qemu` / `make qemu-gdb` 验证,以及 Ubuntu 26.04 上必须打的补丁 |
| [速查表]({{ site.baseurl }}/cheatsheet/) | QEMU 快捷键、GDB 命令、lab 分支工作流、xv6 shell、构建内部机制、故障排查表 |
| [Lab 1: util]({{ site.baseurl }}/labs/util/) | sleep / sixfive / memdump / find / find -exec |

## 当前进度

- [x] 环境搭建并验证(QEMU 10.2.1 · gcc-riscv64 15.2 · gdb-multiarch 17.1)
- [x] Lab util — Exercise 1: `sleep`
- [x] Lab util — Exercise 2: `sixfive`
- [ ] Lab util — Exercise 3: `memdump`
- [ ] Lab util — Exercise 4~5: `find` / `find -exec`

## 关于 lab 解答

{: .warning }
> MIT 要求不要公开发布 xv6 lab 的解答代码。
> 本站保留**全部原理讲解**,但完整代码放在默认折叠的区块里。
> 如果你也在学这门课,强烈建议先自己写 —— 折叠区是给写完之后对照用的。

## 环境信息

本站所有命令和输出都在这台机器上真实跑过:

| | |
|---|---|
| 系统 | Ubuntu 26.04 LTS (x86_64) |
| QEMU | 10.2.1 |
| 交叉编译器 | `riscv64-linux-gnu-gcc` 15.2.0 |
| 调试器 | `gdb-multiarch` 17.1 |
| 课程仓库 | `git://g.csail.mit.edu/xv6-labs-2025`(Fall 2026 复用 2025 仓库) |
