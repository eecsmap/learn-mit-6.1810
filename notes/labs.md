---
layout: default
title: Labs
nav_order: 4
has_children: true
permalink: /labs/
---

# Labs

xv6 的实验按 git 分支组织,每个 lab 一个分支:

```
util → syscall → pgtbl → traps → cow → net → lock → fs → mmap
```

开始一个 lab 就 `git checkout <lab>`,你的提交叠在那个分支上。
`git diff origin/<lab>` 可以看出起始代码和你改了什么。

跑测试:

```bash
make grade                      # 当前分支全部测试
make GRADEFLAGS=sleep grade     # 只跑名字里带 sleep 的测试
```

{: .warning }
> MIT 要求不要公开发布 lab 解答。以下页面里**完整代码都放在折叠区**,
> 展开前建议先自己写一遍 —— 折叠区是用来对照的,不是用来抄的。

## 进度

| Lab | 状态 |
|-----|------|
| [util]({{ site.baseurl }}/labs/util/) | 进行中 — Ex1 `sleep` ✅ · Ex2 `sixfive` ✅ |
| syscall | 未开始 |
| pgtbl | 未开始 |
| traps | 未开始 |
| cow | 未开始 |
| net | 未开始 |
| lock | 未开始 |
| fs | 未开始 |
| mmap | 未开始 |
