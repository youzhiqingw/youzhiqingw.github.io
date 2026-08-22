# Cloudflare 域名绑定 - 方案 B：迁移至 Cloudflare Pages

> **方案 B**：把博客从 GitHub Pages 迁移到 Cloudflare Pages 部署，绑定自定义域名。
> GitHub 仓库仅作代码托管，实际构建和部署由 Cloudflare Pages 完成。
> 适合想完全摆脱 GitHub Pages、使用 CF Pages 全球加速、以后加 Workers / Functions 的场景。

---

## 1. 方案对比（为什么选 B）

| 方案 | 含义 | 适合谁 |
|---|---|---|
| A. Cloudflare 当 CDN，源站仍是 GitHub Pages | 域名指向 GitHub Pages 的 IP；Cloudflare 只做缓存和加速 | 不想动博客代码；想用 CF 的免费 CDN 加速 |
| **B. 迁移到 Cloudflare Pages，绑定新域名** | 把博客重新部署到 Cloudflare，GitHub Pages 仅作代码托管 | 想关掉 GitHub Pages；用 CF Pages 全球加速；以后加 Workers / Functions |

**方案 B 的优势**：
- Cloudflare Pages 全球 CDN 加速，国内访问优于 GitHub Pages
- 支持 Cloudflare Workers / Functions 扩展
- 自动构建 + 预览部署（每个 PR 生成预览链接）
- SSL 证书自动签发，无需手动配置

**方案 B 的注意事项**：
- 如果之前同时部署到 GitHub Pages + Cloudflare Pages，Astro 的 `base: "/"` 配置在两个平台下会冲突，需要二选一或做路径适配
- 迁移后 GitHub Pages 的部署可以关闭，仓库仅保留代码

---

## 2. 前置条件

- 域名已在 Cloudflare 控制台添加（"Add a site"）
- Cloudflare 分配的 Nameserver 已在域名注册商处修改
- DNS 状态显示 "Active"（不是 Pending）
- GitHub 仓库已推送博客代码

---

## 3. 操作步骤

### 3.1 在 Cloudflare 创建 Pages 项目

1. 登录 Cloudflare Dashboard → **Workers & Pages** → **Create application** → **Pages** → **Connect to Git**
2. 授权 Cloudflare 访问你的 GitHub 账号
3. 选择博客仓库 `youzhiqingw/youzhiqingw.github.io`

### 3.2 配置构建设置

| 配置项 | 值 |
|---|---|
| Framework preset | **Astro** |
| Build command | `pnpm build` |
| Build output directory | `dist` |
| 环境变量 | `NODE_VERSION=22`、`PNPM_VERSION=11` |

> 如果项目有 `package.json` 里定义了 `pnpm` 版本约束，确保环境变量与之匹配。Node.js 版本需 >= 22.23.0。

### 3.3 触发首次构建

点击 **Save and Deploy**，Cloudflare 会拉取仓库代码并执行构建。
构建成功后，Cloudflare 自动分配一个 `xxx.pages.dev` 临时域名。

### 3.4 绑定自定义域名

1. 在 Pages 项目页面 → **Custom domains** → **Set up a custom domain**
2. 输入你的域名（如 `blog.yourdomain.com`）
3. Cloudflare 自动添加所需的 DNS 记录（CNAME 指向 `xxx.pages.dev`）
4. SSL 证书自动签发，无需手动操作

### 3.5 关闭 GitHub Pages 部署（可选）

迁移验证通过后：
1. 打开 `https://github.com/youzhiqingw/youzhiqingw.github.io/settings/pages`
2. 在 **Source** 下拉框选择 **None**，保存
3. 仓库代码保留不动，CF Pages 会继续自动拉取并部署

> 如果哪天不想用 CF Pages 了，重新在 GitHub Settings → Pages 里开启 Source 即可恢复 GitHub Pages 部署。

---

## 4. 关键配置改动

### 4.1 修改 siteConfig

`siteConfig.ts` 的 `site_url` 改为新域名：

```typescript
site_url: "https://blog.yourdomain.com"
```

这样 RSS / sitemap / canonical 全部指向新域名。

### 4.2 base 路径

Astro 站点的 `base: "/"` 在 Cloudflare Pages 下正常工作，无需修改。
但如果同时保留 GitHub Pages 部署，会出现资源 404 冲突——需要二选一或做 `assetPath()` 动态路径适配。

### 4.3 GitHub 仓库

仓库可以保留不动，CF Pages 会自动监听 `git push` 并触发构建。GitHub Actions（如果之前用于部署到 GitHub Pages）可以保留也可以删除。

---

## 5. 常见问题排查

### 5.1 构建失败

**原因**：Node.js / pnpm 版本不匹配，或依赖安装超时。
**解决**：
- 确认环境变量 `NODE_VERSION=22`、`PNPM_VERSION=11`
- 检查 `package.json` 的 `engines` 字段是否约束了版本
- 如果依赖安装超时，尝试在 `pnpm` 命令后加 `--no-frozen-lockfile`（不推荐长期使用）

### 5.2 双平台冲突

**原因**：Astro 站点的 `base: "/"` 配置在 GitHub Pages 和 Cloudflare Pages 两个平台下表现不一样。
**症状**：在一个平台能访问、另一个平台资源 404。
**解决**：要么只用一个平台（推荐迁移完后关掉 GitHub Pages），要么做 `assetPath()` 动态路径适配（要改源码）。

### 5.3 自定义域名不生效

**原因**：DNS 记录未正确添加，或 Cloudflare 代理状态不对。
**解决**：
- 在 Cloudflare DNS 页面检查是否有 CNAME 记录指向 `xxx.pages.dev`
- 确保代理状态为 **Proxied（橙色云）**
- 等 10 分钟到 24 小时让 DNS 生效

### 5.4 旧域名 301 重定向

如果之前用 `youzhiqingw.github.io` 域名被搜索引擎收录了，可以在 Cloudflare Pages 设置重定向规则：
1. **Workers & Pages** → 你的 Pages 项目 → **Settings** → **Redirects**
2. 添加规则：`youzhiqingw.github.io/*` → `https://blog.yourdomain.com/:splat`（301 永久重定向）

---

## 6. 常见 Q&A

**Q：迁移后 GitHub 仓库还需要吗？**
A：需要。CF Pages 从仓库拉取代码构建，仓库是源码的唯一来源。只是 GitHub Pages 的部署功能可以关掉。

**Q：能不能同时保留 GitHub Pages 和 CF Pages？**
A：技术上可以，但 `base: "/"` 配置会冲突。建议迁移完后关掉 GitHub Pages。

**Q：CF Pages 构建次数有限制吗？**
A：免费版每月 500 次构建。个人博客每次 push 触发一次构建，通常足够。

**Q：迁移后老链接会失效吗？**
A：`youzhiqingw.github.io` 的链接仍可访问（只要 GitHub Pages 不关），但建议设置 301 重定向到新域名。

**Q：能不能绑定多个子域名？**
A：可以。在 Pages 项目的 Custom domains 里添加多个域名，每个都会自动签发 SSL。

---

## 7. 实操建议（按时间顺序）

**准备阶段**：
- 在 Cloudflare 控制台把域名加好
- 记下 Cloudflare 分配的 Nameserver，去域名注册商改 DNS 服务器
- 等 Cloudflare 显示 DNS "Active"

**迁移阶段**：
- 在 Cloudflare Pages 连接 GitHub 仓库
- 配置构建设置（Astro + pnpm）
- 触发首次构建，验证 `xxx.pages.dev` 能正常访问
- 绑定自定义域名，等 SSL 签发
- 改 `siteConfig.ts` 的 `site_url`
- 验证通过后关掉 GitHub Pages 部署

**验证清单**：
- [ ] `xxx.pages.dev` 能正常访问
- [ ] 自定义域名 HTTPS 正常
- [ ] RSS / sitemap 指向新域名
- [ ] 文章页面路由正常
- [ ] 静态资源（图片/CSS/JS）加载正常
- [ ] 旧域名设置 301 重定向（可选）

---

## 8. 方案 A vs 方案 B 选择建议

| 维度 | 方案 A（CDN 加速） | 方案 B（迁移 Pages） |
|---|---|---|
| 代码改动 | 零 | 改 `site_url` |
| 部署平台 | GitHub Pages | Cloudflare Pages |
| 构建方式 | GitHub Actions | Cloudflare 自动构建 |
| 国内速度 | CF CDN 加速（中等） | CF Pages 全球加速（较好） |
| 可扩展性 | 无 Workers | 支持 Workers / Functions |
| 风险 | 极低 | 低（迁移过程可回退） |
| 推荐时机 | 刚上线、还在迭代 | 博客稳定后、想升级 |

**建议**：先用方案 A 不动代码就绑上域名，等博客稳定后再考虑是否迁移到方案 B。

---

**参考来源**：
- Cloudflare Pages 官方迁移文档：https://developers.cloudflare.com/pages/migrations/migrating-jekyll-from-github-pages/
- Cloudflare Pages 部署与自定义域名配置指南：https://www.gydblog.com/sop/cloudflare-pages-deployment.html
