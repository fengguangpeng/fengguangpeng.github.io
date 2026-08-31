---
title: "Hugo + GitHub 博客部署和使用"
subtitle_zh: "从零搭建并持续发布你的免费博客"
date: 2026-08-31T22:15:00+08:00
draft: false
tags: ["Hugo", "GitHub Pages", "GitHub Actions", "Blog"]
---

本文记录本博客（Hugo + PaperMod + GitHub Pages）的完整部署与日常使用流程，方便以后换机器或重建时参考。
由AI生成总结，已测试

> 文中所有 `<占位符>` 请替换为你自己的信息，例如 `<YOUR_USERNAME>` 替换为你的 GitHub 用户名。

## 1. 整体架构

- **Hugo**：静态站点生成器（需安装 Extended 版）
- **PaperMod**：Hugo 主题（以 git 子模块引入）
- **GitHub Pages**：免费托管站点
- **GitHub Actions**：推送代码后自动构建并部署

发布流程：本地写文章 → `git push` → Actions 自动构建 → 线上更新。

## 2. 安装 Hugo

Windows 推荐用 winget 安装：

```powershell
winget install Hugo.Hugo.Extended
```

验证是否安装成功：

```powershell
hugo version
```

注意两点：

- 必须装 **Extended（扩展）版**，处理 SCSS 等资源时需要
- 主题可能要求较高的 Hugo 版本。例如 PaperMod 要求 **Hugo ≥ 0.146.0**，版本太低会直接构建失败

## 3. 克隆仓库

```powershell
git clone --recurse-submodules https://github.com/<YOUR_USERNAME>/<YOUR_USERNAME>.github.io.git
cd <YOUR_USERNAME>.github.io
```

如果克隆时没有带上子模块，手动拉取主题：

```powershell
git submodule update --init --recursive
```

主题位于 `themes/PaperMod` 目录。

## 4. 配置 GitHub Pages 与 Actions

### 4.1 开启 Pages

进入仓库 **Settings → Pages**，在 "Build and deployment" 的 **Source** 下拉框选择 **GitHub Actions**。

### 4.2 添加构建工作流

在仓库中创建 `.github/workflows/hugo.yaml`，内容如下（`HUGO_VERSION` 要与主题匹配，PaperMod 需要 ≥ 0.146.0）：

```yaml
name: Deploy Hugo site to Pages

on:
  push:
    branches: ["main"]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      HUGO_VERSION: 0.146.0
    steps:
      - name: Install Hugo CLI
        run: |
          wget -O ${{ runner.temp }}/hugo.deb https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.deb
          sudo dpkg -i ${{ runner.temp }}/hugo.deb
      - name: Checkout
        uses: actions/checkout@v4
        with:
          submodules: recursive
          fetch-depth: 0
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5
      - name: Build with Hugo
        env:
          HUGO_ENVIRONMENT: production
        run: |
          hugo --gc --minify --baseURL "${{ steps.pages.outputs.base_url }}/"
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public

  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

### 4.3 检查站点配置

`hugo.toml` 中的 `baseURL` 要改成自己的地址：

```toml
baseURL = "https://<YOUR_USERNAME>.github.io/"
```

## 5. 本地预览

```powershell
hugo server
# 浏览器打开 http://localhost:1313
```

加 `-D` 参数可以预览尚未发布的草稿（`draft: true` 的文章）：

```powershell
hugo server -D
```

## 6. 写新文章并上传

### 6.1 新建文章

在 `content/posts/` 下新建 `.md` 文件，例如 `my-post.md`：

```markdown
---
title: "我的新文章"
date: 2026-08-31T22:30:00+08:00
draft: true
tags: ["Hugo"]
---

这里是正文，支持 Markdown 语法。
```

- `date` 填当前时间
- `draft: true` 表示草稿（不会发布），确认无误后改为 `false`

### 6.2 本地确认效果

```powershell
hugo server -D
```

### 6.3 提交并推送

```powershell
git add .
git commit -m "Add post: my-post"
git push origin main
```

如果本机 git 全局账号不是这个仓库的账号，可以用仓库级身份提交（不会改动全局配置）：

```powershell
git -c user.name="<YOUR_USERNAME>" -c user.email="<YOUR_EMAIL>" commit -m "Add post: my-post"
git push origin main
```

推送成功后，GitHub Actions 会自动构建部署，约 1-2 分钟后访问 `https://<YOUR_USERNAME>.github.io/` 即可看到更新。

## 7. 常见问题

| 现象 | 原因与解决 |
|------|-----------|
| 访问显示 404 | Pages 未启用或 Source 未选 GitHub Actions，检查 **Settings → Pages** |
| 构建失败，提示 "Hugo vX or greater is required" | 工作流中 `HUGO_VERSION` 太低，升级到主题要求的最低版本（PaperMod 需 ≥ 0.146.0） |
| 页面样式错乱、资源 404 | `hugo.toml` 的 `baseURL` 与访问域名不一致 |
| 主题目录为空 | 子模块未拉取，执行 `git submodule update --init --recursive` |

---

以上即本博客的完整部署与使用指南，希望对你有帮助。
