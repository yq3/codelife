# AGENTS.md

## 项目

Hexo 7.3.0 静态博客（"CodeLife"），使用 Fluid 主题 v1.9.9。源语言：zh-CN。

## 常用命令

```
npm run build    # hexo generate -> 生成到 public/
npm run clean    # hexo clean（清除 public/ 和 db.json）
npm run server   # hexo server（本地开发，http://localhost:4000/codelife/）
```

**没有 test 或 lint 命令**。验证方式 = `npm run build` 退出码为 0。

干净的完整构建（最接近"测试"的操作）：
```
npx hexo clean && npx hexo generate --debug
```

## 架构与注意事项

- **根路径为 `/codelife/`**：所有 URL 都有此前缀。本地开发访问 `http://localhost:4000/codelife/`，而不是 `/`。
- **主题通过 npm 安装**：当前主题（`hexo-theme-fluid`）位于 `node_modules/`，不在 `themes/` 目录下。`themes/` 目录仅有一个 `.gitkeep` 占位文件。主题配置覆写在 `_config.fluid.yml` 中进行，而非 `_config.yml`。
- **`public/` 已加入 .gitignore**：该目录在 CI 上生成并推送到 `gh-pages` 分支，绝不提交到 `main` 分支。
- **部署通过 GitHub Actions 完成**（`.github/workflows/deploy.yml`），而非 `hexo deploy`。`_config.yml` 中的 `deploy` 字段为空。

## 文章发布流程

1. 确认当天下一可用 ID：`ls source/_posts/YYYYMMDD* | sort | tail -1`（只需读文件名，不消耗 token 读取文件内容）
2. 在 `source/_posts/` 下创建 `YYYYMMDDNN_标题.md`（文件名为 `年月日+序号+下划线+标题`，如 `2026071101_hello-world.md`）
3. 文件内 front matter 填写 `abbrlink: YYYYMMDDNN`（与文件名前缀一致），插件根据此值生成 URL
4. `npm run server` → 本地预览 `http://localhost:4000/codelife/`
5. `git add . && git commit -m "..." && git push origin main` → CI 自动构建并发布到 `https://yq3.github.io/codelife/`

**注意**：不使用 `hexo new`，直接手动创建文件以节省 token。文件名不影响网页展示，URL 由 front matter 中的 `abbrlink` 决定。

## 文章 URL（hexo-abbrlink）

- 已安装 `hexo-abbrlink` 插件，`permalink` 配置为 `:abbrlink/`。
- **`abbrlink` 命名规则**：`年月日 + 两位数序号`，如 `2026071101` 表示 2026 年 7 月 11 日的第 1 篇博客，当天第 2 篇为 `2026071102`。
- **文件命名规则**：文件名前缀与 `abbrlink` 保持一致（`YYYYMMDDNN_标题.md`），以便通过 `ls source/_posts/YYYYMMDD*` 快速查找当天已有序号，无需读取文件内容。
- **插件行为**：如果文章 front matter 中已有 `abbrlink` 值，插件不会覆盖；如果留空，插件自动生成 CRC32 数字并**写回 MD 文件**，此时需手动改为符合命名规则的值。
- 最终 URL：`/codelife/{abbrlink值}/`（如 `/codelife/2026071101/`）。
- 插件配置位于 `_config.yml:18`，`force: false`（默认，不覆盖已有值）。

## 主题（Fluid）

- `_config.fluid.yml`（约 1093 行）覆盖主题默认配置，只包含自定义过的设置项。
- 暗色模式：已启用，`default: auto`（跟随浏览器偏好 + 本地时间 18:00-06:00）。
- 搜索配置引用了 `/local-search.xml`，但所需的 `hexo-generator-search` 插件 **不在** `package.json` 中——搜索功能可能无法正常工作。
- 评论功能默认关闭。

### 已知问题修复：文章目录（TOC）只显示 H1

**问题**（2026-08-19 发现）：H1 标题较长的文章（如《基于OpenCode的多Agent代码开发工作流》），Chrome 下侧边栏目录只显示 H1，h2/h3 子条目不可见；缩放到 50% 或触发 resize/refresh 后恢复。DOM 完整（条目数正确、无折叠类、每项 rect height > 0），属绘制层问题，无头 Chrome（软件渲染）无法复现。

**根因**：Fluid v1.9.9 的 `source/css/_pages/_base/_widget/toc.styl:77-83` 给每个 TOC 条目 `.toc-list-item` 设置：

```styl
display -webkit-box
-webkit-box-orient vertical
-webkit-line-clamp 2
overflow hidden
```

本意是长标题最多显示 2 行省略号，但 H1 条目的 `<li>` 内嵌套整个 h2/h3 子树的 `<ol>`——`line-clamp: 2` 钳制的是整个盒子内的行盒数。H1 长标题换行占满 2 行配额后，子树所有 h2/h3 的文字被钳掉不绘制（布局高度保留）。旧文 H1 短（1 行）故不触发。

**修复**：通过 `custom_css` 注入覆写（不修改 `node_modules`）：

- 文件：`source/css/toc-fix.css`
- 引用：`_config.fluid.yml` 中 `custom_css: /css/toc-fix.css`（模板 `css()` helper 自动加 `/codelife/` 前缀）

内容：把 `.toc-list-item` 恢复为 `display: block`，将"两行省略"移到其直接子级 `a.tocbot-link` 上——保留主题原意的长标题截断，子树不再被钳制。正常浏览器下视觉零变化，纯防御性修正。

**诊断要点**（如再遇类似"DOM 在但不显示"问题）：用 F12 Console 检查 `document.querySelectorAll('#toc-body li').length`（DOM 完整性）与 `getBoundingClientRect().height`（布局层）——两者都正常而视觉缺失，查 `line-clamp` / `overflow: hidden` / 旧 flexbox（`display: -webkit-box`）。

## Git 与发布

- **Git 推送必须用 SSH**：本机 keychain 无 GitHub HTTPS 凭据，HTTPS push 会报认证错误。remote 已配置为 `git@github.com:yq3/codelife.git`，请勿改回 HTTPS。推送到 GitHub 前先确认 `ssh -T git@github.com` 可用（首次可能需 `ssh-add ~/.ssh/id_ed25519`）。

## GitHub CLI（gh）

- 已安装（v2.97.0，2026-07-31），但**不在默认 PATH 里**：二进制在 `$HOME/install/gh_2.97.0_macOS_arm64/bin/gh`，PATH 是通过 `~/.zshrc` 加的，非交互式 shell（如 opencode 的 bash 工具）不会加载 `.zshrc`。
  - 使用前先 `export PATH="$HOME/install/gh_2.97.0_macOS_arm64/bin:$PATH"`，或直接调用完整路径。
- 已登录 GitHub 账号 `yq3`（凭据存系统 keychain，git 操作用 SSH）。
