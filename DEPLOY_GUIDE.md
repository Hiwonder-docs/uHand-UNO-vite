# VitePress 文档部署指南

> 适用于多个项目的 VitePress 文档部署，每个项目独立仓库，通过宝塔 Nginx 反向代理统一对外提供服务。

---

## 架构说明

```
用户访问 wiki-test.hiwonder.com/projects/项目名/en/latest/docs/xxx.html
         │
         ▼
    宝塔 Nginx（反向代理）
         │
         ▼
    GitHub Pages（每个项目独立仓库）
    hiwonder-docs.github.io/仓库名/projects/项目名/en/latest/...
```

### 访问地址

| 方式 | 地址格式 |
|------|---------|
| GitHub Pages 直连 | `https://hiwonder-docs.github.io/仓库名/projects/项目名/en/latest/docs/xxx.html` |
| 宝塔反代（对用户） | `https://wiki-test.hiwonder.com/projects/项目名/en/latest/docs/xxx.html` |

---

## 一、本地项目结构

每个项目的 VitePress 项目结构如下：

```
项目名/
├── docs/
│   ├── .vitepress/
│   │   └── config.mts          ← VitePress 配置
│   ├── 1_xxx.md                ← Markdown 文档
│   ├── 2_xxx.md
│   └── public/
│       └── favicon.ico
├── scripts/
│   └── stage_main_site.mjs     ← 构建产物整理脚本
├── package.json
└── .gitignore
```

---

## 二、配置文件

### 1. VitePress 配置 `docs/.vitepress/config.mts`

```typescript
import { defineConfig } from 'vitepress'

export default defineConfig({
  // ⚠️ 关键：base 路径格式为 /projects/项目名/en/latest/
  base: process.env.DOCS_BASE || '/projects/项目名/en/latest/',
  lang: 'en-US',
  title: '项目名 Documentation',
  description: '项目名 robot documentation',
  // ... 其他配置
})
```

### 2. 构建脚本 `scripts/stage_main_site.mjs`

```javascript
import { mkdir, rm, cp } from 'fs/promises'
import { fileURLToPath } from 'url'
import { dirname, join } from 'path'

const __dirname = dirname(fileURLToPath(import.meta.url))
const repositoryRoot = join(__dirname, '..')

await rm(join(repositoryRoot, 'projects'), { recursive: true, force: true })

// ⚠️ 把"项目名"改成你的项目名
const targetDir = join(repositoryRoot, 'projects/项目名/en/latest')
await mkdir(targetDir, { recursive: true })

await cp(
  join(repositoryRoot, 'docs/.vitepress/dist'),
  targetDir,
  { recursive: true }
)

console.log('Staged files to:', targetDir)
```

### 3. `package.json` 的 scripts

```json
{
  "scripts": {
    "docs:dev": "vitepress dev docs",
    "docs:build": "vitepress build docs",
    "docs:stage-main": "node scripts/stage_main_site.mjs"
  }
}
```

### 4. 首页重定向 `index.html`（仓库根目录）

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <meta http-equiv="refresh" content="0;url=/projects/项目名/en/latest/docs/1_xxx.html">
  <title>项目名 Documentation</title>
</head>
<body>
  <a href="/projects/项目名/en/latest/docs/1_xxx.html">Click here if you are not redirected automatically.</a>
</body>
</html>
```

### 5. `.nojekyll`（仓库根目录）

创建一个空的 `.nojekyll` 文件，防止 GitHub Pages 忽略 `_` 开头的文件。

---

## 三、构建流程

每次更新文档后，执行以下步骤：

```bash
# 1. 本地修改 Markdown 文档
# 2. 构建文档
npm run docs:build

# 3. 整理构建产物到 projects/项目名/en/latest/
npm run docs:stage-main

# 4. 提交到 GitHub
git add projects/ index.html .nojekyll .gitignore
git commit -m "更新文档"
git push origin main
```

> **注意**：只提交 `projects/`、`index.html`、`.nojekyll`、`.gitignore`，不提交 `docs/`、`scripts/`、`node_modules/` 等。

---

## 四、GitHub Pages 配置

1. 打开 GitHub 仓库 → **Settings** → **Pages**
2. **Source**: Deploy from a branch
3. **Branch**: `main` / `root`
4. **不要绑定自定义域名**（Custom domain 留空）

---

## 五、宝塔 Nginx 反向代理配置

### 1. 添加站点

宝塔面板 → 网站 → 添加站点：
- 域名：`wiki-test.hiwonder.com`
- 不需要创建数据库

### 2. 配置 Nginx

站点设置 → **配置文件**，在 `server {}` 块内添加每个项目的反向代理规则：

```nginx
# ============ 文档反代规则 ============

# 项目1：LanderPi
location ^~ /projects/LanderPi/ {
    proxy_pass https://hiwonder-docs.github.io/LanderPi-vite/projects/LanderPi/;
    proxy_set_header Host hiwonder-docs.github.io;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_ssl_server_name on;
}

# 项目2：SOARM101
location ^~ /projects/SOARM101/ {
    proxy_pass https://hiwonder-docs.github.io/SOARM101/projects/SOARM101/;
    proxy_set_header Host hiwonder-docs.github.io;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_ssl_server_name on;
}

# 项目3：XXX（按此格式添加更多项目）
# location ^~ /projects/XXX/ {
#     proxy_pass https://hiwonder-docs.github.io/XXX仓库名/projects/XXX/;
#     proxy_set_header Host hiwonder-docs.github.io;
#     proxy_set_header X-Real-IP $remote_addr;
#     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
#     proxy_set_header X-Forwarded-Proto $scheme;
#     proxy_ssl_server_name on;
# }
```

### 关键说明

| 配置项 | 说明 |
|--------|------|
| `^~` | 优先匹配，防止被 PHP 和正则规则拦截 |
| `proxy_pass` 带 URI | 保留完整路径，不会截断前缀 |
| `proxy_set_header Host` | 让 GitHub 识别请求的域名 |
| `proxy_ssl_server_name on` | 支持 SNI，否则 HTTPS 反代会失败 |

### 3. 每加一个新项目只需

1. 在 GitHub 创建新仓库，按本文档配置好项目
2. 在宝塔 Nginx 配置中加一条 `location ^~ /projects/项目名/` 规则
3. 重载 Nginx

---

## 六、DNS 配置

在阿里云 DNS 控制台：

| 主机记录 | 记录类型 | 记录值 |
|---------|---------|--------|
| `wiki-test` | A | 宝塔服务器的公网 IP |

> ⚠️ 不要用 CNAME 指向 `hiwonder-docs.github.io`，必须指向宝塔服务器 IP。

---

## 七、HTTPS 配置（可选）

1. 宝塔面板 → 站点 → SSL → **Let's Encrypt** → 申请证书
2. 勾选 **强制 HTTPS**
3. 反向代理配置无需修改，自动支持

---

## 八、.gitignore 参考

```
node_modules/
docs/
scripts/
.github/
*.log
.DS_Store
Thumbs.db
```

> 只提交 `projects/`、`index.html`、`.nojekyll`、`.gitignore`、`README.md`

---

## 九、快速检查清单

搭建新项目时，按此清单逐项确认：

- [ ] `config.mts` 的 `base` 设置为 `/projects/项目名/en/latest/`
- [ ] `stage_main_site.mjs` 中的项目名已替换
- [ ] `index.html` 的重定向路径已替换
- [ ] `.nojekyll` 文件存在
- [ ] 本地执行 `npm run docs:build && npm run docs:stage-main` 成功
- [ ] `projects/项目名/en/latest/` 目录下有 `assets/` 和 `docs/`
- [ ] HTML 中 CSS 路径为 `/projects/项目名/en/latest/assets/xxx.css`
- [ ] GitHub Pages Source 设为 `main` / `root`
- [ ] GitHub Pages 未绑定自定义域名
- [ ] 宝塔 Nginx 已添加对应的 `location ^~` 规则
- [ ] Nginx 已重载
- [ ] DNS 指向宝塔服务器 IP

---

## 十、常见问题

### Q1: CSS 样式丢失

**原因**：CSS 路径不对，或文件未推送到 GitHub。

**检查**：
1. 浏览器 F12 → Network → 看 CSS 请求返回什么状态
2. 直接访问 `https://hiwonder-docs.github.io/仓库名/projects/项目名/en/latest/assets/xxx.css` 是否能打开
3. 确认 `base` 配置正确

### Q2: 502 Bad Gateway

**原因**：Nginx 无法连接到 GitHub。

**检查**：
1. `proxy_pass` 地址是否正确
2. `proxy_ssl_server_name on` 是否添加
3. 服务器能否访问 GitHub：`curl -v https://hiwonder-docs.github.io`

### Q3: 500 服务器异常

**原因**：请求被 PHP 拦截。

**解决**：location 前缀加 `^~`，如 `location ^~ /projects/项目名/`

### Q4: 访问 `wiki-test.hiwonder.com` 报域名已占用

**原因**：另一个 GitHub 仓库还绑着这个域名。

**解决**：在该仓库 Settings → Pages → Custom domain → Remove
