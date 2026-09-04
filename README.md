# learn-mit-6.1810

MIT **6.1810 Operating System Engineering (Fall 2026)** 自学笔记。
以 GitHub Pages 站点形式发布(Jekyll + Just the Docs)。

- 课程主页 <https://pdos.csail.mit.edu/6.1810/2026/>
- xv6 book <https://pdos.csail.mit.edu/6.1810/2026/xv6/book-riscv-rev4.pdf>
- Lab 仓库 `git://g.csail.mit.edu/xv6-labs-2025`(Fall 2026 复用 2025 仓库)

## 内容

| 文件 | 站点路径 | 内容 |
|------|----------|------|
| `index.md` | `/` | 首页、进度总览 |
| `CHEATSHEET.md` | `/cheatsheet/` | QEMU / GDB / git-labs / xv6 shell / 构建机制 / 故障排查 |
| `notes/00-environment-setup.md` | `/setup/` | 环境搭建全过程 + Ubuntu 26.04 的坑 |
| `notes/labs.md` | `/labs/` | Lab 索引与进度 |
| `notes/01-lab-util.md` | `/labs/util/` | Lab util:sleep / sixfive / memdump / find / find -exec |

`xv6-labs-2025/` 是 clone 下来的课程代码(独立 git repo),已在 `.gitignore` 中排除,
不会随本仓库发布。

## 关于 lab 解答

MIT 要求不要公开发布 xv6 lab 的解答代码。本站保留全部原理讲解,
**完整代码放在默认折叠的 `<details>` 区块里**,并在页面顶部有声明。

## 本地预览

```bash
bundle install
bundle exec jekyll serve --baseurl ""
# http://127.0.0.1:4000
```

## 发布到 GitHub Pages

仓库 Settings → Pages → Source 选 **Deploy from a branch**,
branch `main`,folder `/ (root)`。

> **私有仓库注意:** GitHub Pages 对私有仓库需要 GitHub Pro / Team / Enterprise。
> 免费账号下私有仓库无法发布 Pages —— 要么把仓库改成 public,要么升级套餐。
