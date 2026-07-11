---
title: Hexo 博客重建总结
date: 2026-07-11
abbrlink: 2026071101
tags: [Hexo, GitHub Actions, 博客, Fluid]
---

# Hexo 博客重建总结

## 背景

### 旧博客现状

- **仓库**：`yq3/yq3.github.io`，通过 GitHub Pages 托管于 `https://yq3.github.io`
- **状态**：仓库中仅有 Hexo 编译后的静态 HTML/CSS/JS 产物，Hexo 源文件（Markdown 文章、`_config.yml`、主题等）因老电脑丢失而遗失
- **规模**：16 篇博客文章（2019 年 7 月至 2020 年 2 月），涵盖 Python/数据分析、Java/Spring、机器学习、MySQL、建站等笔记
- **主题**：Landscape（Hexo 默认主题，未安装第三方主题）
- **分类**：2 个（技术 15 篇、闲聊 1 篇）
- **标签**：14 个（Deep Learning、Hexo、JDBC、Java、Java Swing、Jupyter、Kaggle、Machine Learning、Maven、MySQL、Pandas、Python、Spring、数学建模）
- **旧博客决定**：完全封存不动，不进行任何修改

### 核心问题

旧工作流中 Hexo 源文件仅在本地存储，未纳入版本控制。老电脑丢失后源文件不可恢复，仅剩编译产物。

## 新博客方案

### 架构设计

**仓库**：`yq3/codelife`  
**访问地址**：`https://yq3.github.io/codelife/`  
**工作流**：单仓库双分支，源文件和编译产物均在版本控制中，永不丢失

| 分支 | 内容 | 作用 |
|------|------|------|
| `main` | Markdown 文章、`_config.yml`、主题等 Hexo 源文件 | 日常 `git push` 到这个分支 |
| `gh-pages` | 编译后的静态 HTML/CSS/JS | GitHub Actions 自动部署产物，浏览器实际访问的内容 |

**流程**：本地写 Markdown → `git push main` → GitHub Actions 触发 `hexo generate` → 输出推送到 `gh-pages` → 网站自动更新

### 方案选择过程

| 方案 | 说明 | 决定 |
|------|------|------|
| 同仓库共存 | 在旧仓库中用 Hexo 编译产物与旧 HTML 共存 | ❌ 存在 CSS 冲突、导航/归档不显示旧文章等兼容问题 |
| 新建项目站点 | 独立新仓库，URL 为 `yq3.github.io/xxx/` | ✅ 零兼容问题，新旧完全隔离 |

> GitHub Pages 限制：一个账号只能有一个 `<username>.github.io` 用户主站点，但可创建无限个项目站点（URL 为 `yq3.github.io/<仓库名>/`）。项目站点 URL 与仓库名一一对应，不可更改。

### 技术选型

| 项目 | 选择 | 说明 |
|------|------|------|
| 静态站点生成器 | Hexo | 与旧博客一致，上手快，插件丰富 |
| 主题 | Fluid | Material Design 风格，支持暗色模式、代码高亮、标签云，中英文友好，适合技术+生活混合内容 |
| 部署方式 | GitHub Actions | push 即部署，无需本地执行 `hexo deploy` |

## 详细操作步骤

### 一、创建 GitHub 仓库

1. 访问 https://github.com/new
2. Repository name：`codelife`
3. 选择 Public（免费账号 Pages 需公开）
4. **不要**勾选 Initialize 选项（README、.gitignore、License）
5. 记录远程地址：`git@github.com:yq3/codelife.git`

### 二、本地初始化 Hexo 项目

```bash
npm install -g hexo-cli        # 安装 Hexo CLI（如已安装可跳过）
hexo init codelife             # 创建项目
cd codelife
npm install                     # 安装依赖
```

目录结构：

```
codelife/
├── _config.yml          # 站点配置文件
├── package.json
├── scaffolds/           # 文章模板
├── source/              # Markdown 文章、页面
│   └── _posts/          # 文章存放目录
├── themes/              # 主题目录
└── node_modules/
```

### 三、配置 `_config.yml`

关键配置（**`root` 必须正确填写**）：

```yaml
# Site
title: YQ's Code Life
subtitle: ''
language: zh-CN
timezone: 'Asia/Shanghai'

# URL（项目站点关键配置）
url: https://yq3.github.io
root: /codelife/            # 项目站点必须设置，否则所有 CSS/JS 路径 404
permalink: :year/:month/:day/:title/

# Extensions
theme: fluid
```

### 四、安装 Fluid 主题

```bash
npm install --save hexo-theme-fluid
```

在项目根目录创建 `_config.fluid.yml`（独立配置文件，方便主题升级时保留自定义配置）：

```yaml
navbar:
  blog_title: "YQ's Code Life"

index:
  banner_img: /img/banner.jpg

post:
  banner_img: /img/banner.jpg
  comments:
    enable: false

dark_mode:
  enable: true
  default: auto

code:
  enable_highlight: true
```

### 五、配置 Git 和 GitHub Actions

**5.1 初始化 Git**

```bash
cd codelife
git init
git remote add origin git@github.com:yq3/codelife.git
```

**5.2 创建 `.gitignore`**

```gitignore
node_modules/
public/
.deploy_git/
db.json
```

**5.3 创建 `.github/workflows/deploy.yml`**

```yaml
name: Deploy Hexo Blog

on:
  push:
    branches: [main]

permissions:
  contents: write        # 必须，允许 Push 到 gh-pages 分支

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Generate static files
        run: npx hexo generate

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./public
          publish_branch: gh-pages
```

### 六、启用 GitHub Pages

1. 首次 push 后，访问 `https://github.com/yq3/codelife/settings/pages`
2. Source 选择 **"Deploy from a branch"**
3. Branch 选择 **`gh-pages`**，目录选 **`/ (root)`**
4. 保存后等待 1-2 分钟，页面显示部署地址

> **注意**：必须在第一次 push 后 Actions 创建了 `gh-pages` 分支，才能在 Pages 设置中选择它。

### 七、首次提交

```bash
git add -A
git commit -m "init: hexo blog with fluid theme"
git branch -M main
git push -u origin main
```

推送后访问 `https://github.com/yq3/codelife/actions` 查看构建进度。

### 八、日常写作流程

```bash
# 1. 创建新文章
npx hexo new "文章标题"

# 2. 编辑 source/_posts/文章标题.md

# 3. 本地预览（可选）
npx hexo server

# 4. 提交并推送
git add -A
git commit -m "post: 文章标题"
git push
```

推送后约 1 分钟自动部署上线。

## 遇到的问题和解决

### 问题：GitHub Actions 推送权限被拒

**错误信息**：

```
remote: Permission to yq3/codelife.git denied to github-actions[bot].
fatal: unable to access 'https://github.com/yq3/codelife.git/': The requested URL returned error: 403
Error: Action failed with "The process '/usr/bin/git' failed with exit code 128"
```

**原因**：`GITHUB_TOKEN` 默认只有只读权限，无法推送代码。

**解决方案（两个都做）**：

1. **Workflow 文件添加权限声明**（在 `deploy.yml` 最外层）：

   ```yaml
   permissions:
     contents: write
   ```

2. **仓库设置中开启写权限**：
   - 访问 `https://github.com/yq3/codelife/settings/actions`
   - Workflow permissions 选择 **"Read and write permissions"**
   - 保存

## 注意事项

1. **`root: /codelife/`** 是项目站点关键配置，忘记设置会导致所有 CSS/JS/图片 404
2. **Markdown 中引用图片**：图片放在 `source/img/` 目录，文章中用 `![](/codelife/img/xxx.png)` 引用
3. **`.gitignore` 排除了 `public/`**：编译产物仅在 CI 中生成，不提交到 `main` 分支
4. **主题配置**：使用 `_config.fluid.yml` 而非 `themes/fluid/_config.yml`，便于升级主题时保留自定义配置
5. **项目站点 URL 不可更改**：`yq3.github.io/<仓库名>/` 路径与仓库名固定绑定
