# Cloudflare 域名绑定 - 方案 A：CDN 加速 GitHub Pages

> **方案 A**：只把 Cloudflare 当 DNS / CDN，源站仍是 GitHub Pages。
> 域名通过 Cloudflare 解析到 GitHub Pages 的服务器 IP，Cloudflare 提供免费 CDN 加速。
> **不改一行代码、不影响现有部署流程**，是最稳妥的起步方案。

---

## 1. 方案对比（为什么选 A）

| 方案 | 含义 | 适合谁 |
|---|---|---|
| **A. Cloudflare 当 CDN，源站仍是 GitHub Pages** | 域名指向 GitHub Pages 的 IP；Cloudflare 只做缓存和加速 | 不想动博客代码；想用 Cloudflare 的免费 CDN 加速国内访问 |
| B. 迁移到 Cloudflare Pages，绑定新域名 | 把博客重新部署到 Cloudflare，GitHub Pages 仅作代码托管 | 想关掉 GitHub Pages；用 CF Pages 全球加速；以后加 Workers / Functions |

**方案 A 的优势**：
- 零代码改动，`base: "/"` 配置不用动
- GitHub Pages 部署流程（`git push` → Actions 自动部署）完全不变
- 老地址 `youzhiqingw.github.io` 和新域名同时可用
- 绑定过程渐进切换，旧地址始终可访问

---

## 2. 前置条件

- 域名已在 Cloudflare 控制台添加（"Add a site"）
- Cloudflare 分配的 Nameserver 已在域名注册商处修改
- DNS 状态显示 "Active"（不是 Pending）

---

## 3. 操作步骤

### 3.1 删掉 Cloudflare 默认的占位记录

进入 Cloudflare Dashboard → 选择你的域名 → **DNS → Records**
删除所有 Cloudflare 自动生成的占位记录（一般是 A 记录 `xxx.cloudflare.com` 之类的）。

### 3.2 添加 GitHub Pages 解析记录

在同一个 DNS 页面，添加 4 条 A 记录（主机名都填 `@`，即根域名）：

```
类型  主机名  内容                  代理状态
A     @      185.199.108.153        DNS only（灰色云）→ 暂时先关掉代理
A     @      185.199.109.153        DNS only
A     @      185.199.110.153        DNS only
A     @      185.199.111.153        DNS only
CNAME www    youzhiqingw.github.io  DNS only
```

> ⚠️ **为什么要 DNS only（灰云）？**
> 一开始不要开代理（橙色云），否则 GitHub Pages 的 SSL 验证会卡住。等 3.4 步完成、证书签发后再开代理。

如果使用子域名（如 `blog.yourdomain.com`），则只需要一条 CNAME 记录：

```
类型   主机名  内容                    代理状态
CNAME  blog   youzhiqingw.github.io   DNS only（灰色云）
```

### 3.3 仓库加 CNAME 文件

在部署仓库的 `public/CNAME`（无扩展名）里写一行：

```
blog.yourdomain.com
```

把 `blog.yourdomain.com` 换成你的实际域名。想用子域名前缀 `blog` 就填 `blog.yourdomain.com`；想用根域名就填 `yourdomain.com`。

> 你的本地路径可能是 `E:/Coding/Firefly/public/`（教程版）或 `E:/Coding/ghio_clone/public/`（部署版），改的是 ghio_clone 那个。

### 3.4 仓库 Settings 加 Custom Domain

打开 `https://github.com/youzhiqingw/youzhiqingw.github.io/settings/pages`
在 **Custom domain** 输入 `blog.yourdomain.com`，点 **Save**。
等 1-5 分钟，GitHub 会自动通过 Let's Encrypt 签 SSL 证书。

### 3.5 勾上 Enforce HTTPS

签发成功后（页面会显示绿色 "Your site is live at..."），勾上 **Enforce HTTPS**。

### 3.6（可选）回头开 Cloudflare 代理

回到 Cloudflare DNS，把那 4 条 A 记录 + 1 条 CNAME 记录的**代理状态**改成 **Proxied（橙色云）**。
这会启用 Cloudflare CDN 加速、HTTPS 强制、缓存等。

---

## 4. 验证

```bash
nslookup blog.yourdomain.com
# 预期：返回 GitHub Pages 的 IP（方案 A）或 Cloudflare IP（开了代理后）

curl -I https://blog.yourdomain.com
# 预期：HTTP/2 200
```

绑定成功后，把 `siteConfig.ts` 的 `site_url` 改成 `https://blog.yourdomain.com`，这样 RSS / sitemap / canonical 全部跟着走。

---

## 5. 常见问题排查

### 5.1 "Custom domain is already taken"

**原因**：GitHub Pages 验证 CNAME 时，发现该域名已被其他 GitHub 仓库用过。
**解决**：把原来绑过的那个仓库的 CNAME / Settings 里的 Custom domain 删掉。

### 5.2 "Improperly configured"

**原因**：Cloudflare 端 DNS 还没生效，或 DNS 记录配错。
**解决**：等 10 分钟到 24 小时；用 `dig blog.yourdomain.com` 验证。

### 5.3 证书签发失败

**原因**：Cloudflare 代理开着（橙色云），GitHub 验证不到你的 IP。
**解决**：临时关代理，等证书签发成功再开回来。

### 5.4 双平台冲突（同时部署到 GitHub Pages + Cloudflare Pages）

**原因**：Astro 站点的 `base: "/"` 配置在两个平台下表现不一样。
**症状**：在一个平台能访问、另一个平台资源 404。
**解决**：要么只用一个平台，要么做 `assetPath()` 动态路径适配（要改源码）。方案 A 不涉及此问题。

---

## 6. 对现有博客的影响

### 6.1 不会出问题的情况

- 只是在 Cloudflare 加 DNS 记录（步骤 3.1-3.2）
- 仓库加 CNAME 文件（步骤 3.3）
- 这些操作 GitHub Pages 自动接管，访问 `youzhiqingw.github.io` **不变**

### 6.2 可能有 5-30 分钟中断的情况

- 域名解析切换期间：DNS 缓存未刷新，部分用户可能 522
- 解决：方案 A 是渐进切换，旧地址 `youzhiqingw.github.io` 一直可访问

### 6.3 应该避免的操作

- ❌ 不要在 Cloudflare DNS 里把 `youzhiqingw.github.io` 也加一条 CNAME 解析走（会和新域名打架）
- ❌ 不要在 Cloudflare 直接关 GitHub Pages（除非你已经迁移到 Pages）
- ❌ 不要在绑定生效前去删 `youzhiqingw.github.io` 的 `CNAME` 文件（会触发 GitHub Pages 重签证书）

---

## 7. 常见 Q&A

**Q：必须用 Cloudflare 吗？**
A：不一定。也可以用阿里云 / 腾讯云 / Porkbun 等其他 DNS 提供商。但已经托管到 CF 了，省钱省事。

**Q：绑定后能不能同时访问 `youzhiqingw.github.io` 和新域名？**
A：可以。两者会指向同一份内容。SEO 上建议在 `siteConfig.ts` 选一个做主站（canonical），避免搜索引擎重复抓取。

**Q：CF 代理（橙色云）会影响 GitHub Pages 部署吗？**
A：会稍微延迟一点（CF 缓存），但部署本身在 GitHub 完成，CF 只是中间缓存层。

**Q：能不能绑定多个子域名（blog、www、cdn...）？**
A：可以。每个子域名在 GitHub Settings → Pages → Custom domain 里加一条。

**Q：绑定后我能随时撤回吗？**
A：能。在 Cloudflare DNS 删除记录 + GitHub Settings → Pages 删除 Custom domain + 删仓库的 CNAME 文件。三步即可还原。

---

## 8. 实操建议（按时间顺序）

**现在先不做**（如果还在本地大改）：
- 在 Cloudflare 控制台提前把域名加好（如果还没加）
- 记下 Cloudflare 分配的 Nameserver，去域名注册商改 DNS 服务器（这一步**现在就可以做**，不影响博客）
- 等 Cloudflare 显示 DNS "Active"

**本地大改完之后**（推送前）：
- 给我域名（如 `example.com`）
- 告诉我用什么子域名前缀（如 `blog`，那就是 `blog.example.com`）
- 执行步骤 3.3-3.5（加 CNAME、加 DNS 记录、设 Custom domain）
- 等绑定成功后，改 `siteConfig.ts` 里的 `site_url`

**可选项**（后期考虑）：
- 等博客稳定后再决定要不要切到 Cloudflare Pages（方案 B）

---

**参考来源**：
- Cloudflare Pages 官方迁移文档：https://developers.cloudflare.com/pages/migrations/migrating-jekyll-from-github-pages/
- Cloudflare Pages 部署与自定义域名配置指南：https://www.gydblog.com/sop/cloudflare-pages-deployment.html
