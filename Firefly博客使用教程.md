# Firefly 博客使用教程

> 这份教程的目标：**你照着做，就能独自完成博客日常维护**，不用再问我。
> 前提：你会在电脑上打开文本编辑器（记事本 / Notepad / VS Code 都行），会复制粘贴。

---

## 0. 先搞懂两件事（必读，省得后面懵）

### 0.1 你的博客在哪？
你的博客代码在两个地方：
- **`E:\Coding\Firefly\`** — 你平时改东西用这个（本地参考副本）
- **`E:\Coding\ghio_clone\`** — 这个才是线上真实版本（跟 GitHub 云端同步）

> **重要**：改完东西后，要把 `ghio_clone` 里的改动推送（push）到 GitHub，线上才会更新。`Firefly` 目录改了不会自动同步。

### 0.2 修改后怎么生效？
```
改文件 → git add → git commit → git push → GitHub Actions 自动部署 → 等 1-2 分钟 → 线上更新
```

如果你不想记命令，我帮你写成脚本了：双击 `E:\Coding\ghio_clone\部署.bat`（如果不存在就用下面的命令）。

或者你自己在终端（Git Bash）里依次执行：
```bash
cd E:/Coding/ghio_clone
git add -A
git commit -m "更新博客"
git push origin master
```

---

## 1. 改站点标题、副标题、描述

**文件**：`src/config/siteConfig.ts`

**改这几行**（搜索关键词就能找到）：
```ts
title: "箐のblog",                          // 站点标题（浏览器标签页标题）
subtitle: "君の指先を舞ってる電光は…",      // 副标题（首页大标题下面那行）
description: "君の指先を舞ってる電光は…",    // 站点描述（SEO 用，别人搜到你站时显示的摘要）
```

**注意**：`site_url` 也在同一个文件里，那是你的域名，**不要随便改**，改了 RSS/sitemap 全错。等你买好域名我再帮你换。

---

## 2. 改站长昵称、头像、社交链接

**文件**：`src/config/profileConfig.ts`

```ts
name: "莜之箐",                    // 昵称（文章作者名、页脚署名）
avatar: "assets/images/avatar.avif", // 头像路径（见第 6 节换头像）
```

**社交链接**（在同一个文件的 `links` 数组里）：
```ts
links: [
  { name: "GitHub", url: "https://github.com/youzhiqingw/Firefly", ... },
  { name: "Email",  url: "mailto:linuxai@qq.com", ... },
  // QQ 和 RSS 的 url 现在是空字符串 ""，要启用就填进去，不要就留着
],
```

**能改的**：`name`（显示名）、`url`（链接地址）
**不能改的**：`icon`（图标名，改了可能不显示）

---

## 3. 写新文章

**目录**：`src/content/posts/`

### 3.1 快速新建（推荐）
终端执行：
```bash
cd E:/Coding/ghio_clone
pnpm new-post
```
按提示填标题，它会自动生成文件并带好 frontmatter。

### 3.2 手动新建
在 `src/content/posts/` 下新建 `我的文章.md`，内容长这样：

```markdown
---
title: 文章标题
published: 2026-08-22          // 发布日期，格式 YYYY-MM-DD
description: 文章摘要，搜站时显示
tags: [标签1, 标签2]            // 自定义标签
category: 分类名                // 如：技术 / 随笔 / 读书
image: ""                       // 文章封面图，空则用默认
slug: my-post                   // URL 结尾，建议用英文小写+连字符
---

正文从这里开始，用 Markdown 写。
```

**关键坑（踩了会构建失败）**：
- 日期字段必须叫 **`published`**，**不是** `pubDate` / `date`。
- `---` 前后不要留空格，字段名拼写要对（`title` / `tags` / `category` / `image` / `slug`）。
- 不要新增字段名（如 `author`、`draft`），模板不认识会崩。
- 删除文章：直接删 `src/content/posts/` 下对应的 `.md` 文件。

---

## 4. 发动态（类似朋友圈）

**目录**：`src/content/dynamic/`

### 4.1 快速新建
```bash
cd E:/Coding/ghio_clone
pnpm new-dynamic
```

### 4.2 手动新建
在 `src/content/dynamic/` 下新建 `2026-08-22-093000.md`（文件名建议用 `日期-时分秒` 格式，防止重名）：

```markdown
---
published: 2026-08-22 09:30:00
location: "广西"          // 可选，显示定位
pinned: false             // 可选，true = 置顶
---

今天学了 Astro，把博客搭起来了。
```

**注意**：动态的 `published` 要带时间（`YYYY-MM-DD HH:MM:SS`），不像文章只要日期。

**配置总开关**：`src/config/dynamicConfig.ts`
- `showComment: true/false` — 是否允许评论
- `itemsPerPage: 20` — 每页显示几条
- `memos.enable: false` — 如果你想接 Memos 数据源，改成 true 并填 API 地址

---

## 5. 建相册

**两步走**：先在配置里加相册，再把图片放对文件夹。

### 5.1 配置相册
**文件**：`src/config/galleryConfig.ts`

在 `albums` 数组里加一项：
```ts
{
  id: "my-album",                    // 相册 ID，会出现在 URL 里
  name: "我的相册",                   // 相册名
  description: "相册描述",            // 简介
  location: "拍摄地点",              // 可选
  date: "2026-08-22",                // 排序用，格式 YYYY-MM-DD
  tags: ["旅行", "摄影"],            // 标签
  password: "",                      // 可选，留空 = 公开；填了 = 加密相册
  passwordHint: "",                  // 可选，密码提示
},
```

### 5.2 放图片
在 `public/gallery/my-album/` 下放你的图片（如 `1.jpg`、`2.jpg`、`cover.avif`）。

- 如果你不指定 `cover`，模板会自动把 `cover.*` 当封面，没有就用第一张图。
- 支持的格式：`jpg` / `png` / `webp` / `avif` / `gif`

**删除相册**：删掉 `galleryConfig.ts` 里对应的配置项，`public/gallery/<id>/` 里的图可以手动删掉。

---

## 6. 换图片（头像 / Banner / Favicon / 赞赏码）

Firefly 里放图片有两套规则，搞混了会 404：

### 6.1 `public/` 目录（推荐新手用）
- 路径：`public/assets/images/xxx.png`
- 访问路径：`/assets/images/xxx.png`（前面加 `/`）
- 优点：不会被构建优化，放进去就能访问，不出错
- 适合：赞赏码、相册图、广告图、特效素材

**当前实际图片位置一览**：

| 图片 | 本地路径 | 配置里怎么填 |
|---|---|---|
| 微信赞赏码 | `public/assets/images/sponsor/wechat.png` | `qrCode: "/assets/images/sponsor/wechat.png"` |
| 支付宝赞赏码 | `public/assets/images/sponsor/alipay.jpg` | `qrCode: "/assets/images/sponsor/alipay.jpg"` |
| 站长头像 | `src/assets/images/avatar.avif` | `avatar: "assets/images/avatar.avif"`（注意**没有**开头 `/`） |
| Banner | `src/assets/images/DesktopWallpaper/d1.avif` 等 | `backgroundWallpaper.ts` 里的 `desktop` / `mobile` 数组 |
| Favicon | `public/favicon/firefly-32.png` 等 | `siteConfig.ts` → `favicon[].src` |
| 樱花特效 | `public/assets/images/effects/sakura.png` | 模板内置，一般不动 |

### 6.2 `src/assets/` 目录（会被构建优化）
- 路径：`src/assets/images/avatar.avif`
- 配置里写：`assets/images/avatar.avif`（**没有**开头 `/`）
- 优点：构建时自动压缩优化，加载更快
- 适合：头像、Banner、OG 图

### 6.3 换头像
1. 把你的图复制到 `src/assets/images/avatar.avif`（覆盖）
2. 文件格式支持 `.avif` / `.webp` / `.png` / `.jpg`
3. 如果想换文件名（如 `my-face.webp`），同步改 `profileConfig.ts` 里的 `avatar` 字段

### 6.4 换 Banner / 背景壁纸
**文件**：`src/config/backgroundWallpaper.ts`

- 当前用的是 `src/assets/images/DesktopWallpaper/d1.avif` ~ `d6.avif` 随机
- 想换成自己的图：
  1. 把图放进 `src/assets/images/DesktopWallpaper/`（命名为 `my-bg.avif`，**不要**叫 `d1.avif`，防止模板更新覆盖）
  2. 在 `backgroundWallpaper.ts` 的 `desktop` 数组里加上 `"assets/images/DesktopWallpaper/my-bg.avif"`
  3. 移动端同理改 `mobile` 数组

**背景视频**（`playerUrl`）当前是作者的外链 `https://bed.twoleaf.cn/file/...`，如果你有本地视频，放进 `public/assets/videos/`，然后改成 `"/assets/videos/my-video.mp4"`。

**主页横幅文字**也在同一个文件里：
```ts
homeText: {
  title: "Lovely firefly!",          // 首页大标题
  subtitle: ["句子1", "句子2", ...],  // 轮播副标题
}
```

### 6.5 换 Favicon
直接替换 `public/favicon/` 下的 `firefly-*.png` 文件（保持文件名不变），或改 `siteConfig.ts` 里 `favicon[]` 数组的 `src` 路径。

---

## 7. 改关于页

**文件**：`src/content/spec/about.md`

直接编辑正文（Markdown 格式），原文件没有 frontmatter，你也不用加。

---

## 8. 改导航栏 / 链接菜单

**文件**：`src/config/navBarConfig.ts`

`links` 数组里的每一项就是一个导航菜单：
```ts
{ name: "链接", url: "/dynamic/", icon: "..." },          // 站内链接
{ name: "关于", url: "/about/", icon: "..." },
```

如果你看到 `external: true`，说明是外链（会在新标签页打开）。

"链接"下拉菜单在同一个文件的 `links.push({ name: "链接", children: [...] })` 里，现在只保留了一个 GitHub，想加别的就往 `children` 数组里加条目。

---

## 9. 打赏页配置

**文件**：`src/config/sponsorConfig.ts`

- `methods` 数组：每个对象就是一个打赏方式（支付宝 / 微信 / ko-fi 等）
- `qrCode`：收款码图片路径（`public/` 下的写 `/assets/images/...`）
- `link`：打赏者点击跳转的链接，空字符串 = 只显示二维码
- `enabled: false` = 隐藏这一项
- `sponsors: []`：打赏者列表，清空就是 []，想加就按同样格式填

**当前状态**：只保留了支付宝（`alipay.jpg`）和微信（`wechat.png`），ko-fi / 爱发电已删除。

---

## 10. 友情链接页（友链）

**文件**：`src/content/spec/friends.mdx`

这是友链页的内容，编辑方式和普通文章一样，frontmatter 在文件顶部。

---

## 11. 部署推送（三种方式，选一种）

### 方式 A：我帮你触发（找我）
你改完后告诉我"推送"，我用 `gh` CLI 帮你触发 GitHub Actions 工作流。

### 方式 B：双击脚本（如果有）
在 `ghio_clone` 目录下找 `部署.bat`，双击运行，等弹窗提示成功就行。

### 方式 C：自己跑命令
打开 **Git Bash**（或任意终端），依次执行：
```bash
cd E:/Coding/ghio_clone
git add -A
git commit -m "更新博客"
git push origin master
```
看到 `master -> master` 就是成功。等 1-2 分钟访问 `https://youzhiqingw.github.io` 看效果。

**推不上去怎么办？**
- 先跑 `git status` 看有没有冲突
- 如果是"拒绝非快进"，说明云端有人改过（一般不会），找我处理

---

## 12. 常见坑 & 避坑指南

| 坑 | 症状 | 解法 |
|---|---|---|
| frontmatter 字段名写错 | 构建报 `YAML Exception` / 文章不显示 | 日期只能用 `published`，不要用 `pubDate` / `date` |
| 图片路径没写 `/` | 图片裂图 / 404 | `public/` 下的路径前面必须加 `/`（如 `/assets/images/xxx.png`） |
| 图片路径多了 `/` | `src/assets/` 下的路径**不要**加 `/`（如 `assets/images/avatar.avif`） |
| 改了 Firefly 目录没推 | 线上还是旧的 | 记得在 `ghio_clone` 里改，或者把 Firefly 的改动复制过去 |
| Pages 显示 404 + Jekyll 报错 | 线上 404，日志里带 `jekyll` | 仓库 Settings → Pages → Source 改成 **GitHub Actions**（不是 Deploy from a branch） |
| 支付宝赞赏码不显示 | 还用的旧 `alipay.png` | 现在配置里是 `alipay.jpg`，别传成 `.png` |

---

## 13. 本地资源完整路径速查表

| 你要找的东西 | 本地路径 | 改配置文件 |
|---|---|---|
| 站点标题 / 副标题 / 描述 / 域名 | `src/config/siteConfig.ts` | 直接改值 |
| 站长昵称 / 头像 / 社交链接 | `src/config/profileConfig.ts` | 直接改值 |
| 赞赏码（微信 / 支付宝） | `public/assets/images/sponsor/` | `src/config/sponsorConfig.ts` |
| Banner / 背景壁纸 | `src/assets/images/DesktopWallpaper/` | `src/config/backgroundWallpaper.ts` |
| 背景视频 | `public/assets/videos/` | `backgroundWallpaper.ts` → `playerUrl` |
| Favicon | `public/favicon/` | `siteConfig.ts` → `favicon[]` |
| 相册配置 | `src/config/galleryConfig.ts` | 改 albums 数组 |
| 相册图片 | `public/gallery/<album-id>/` | 直接扔图进去 |
| 文章 | `src/content/posts/*.md` | 直接编辑 / `pnpm new-post` |
| 动态 | `src/content/dynamic/*.md` | 直接编辑 / `pnpm new-dynamic` |
| 关于页 | `src/content/spec/about.md` | 直接编辑正文 |
| 友链页 | `src/content/spec/friends.mdx` | 直接编辑 |
| 导航栏菜单 | `src/config/navBarConfig.ts` | 改 links 数组 |
| 评论系统 | `src/config/commentConfig.ts` | giscus 配置 |
| 友情链接配置 | `src/config/friendsConfig.ts` | 友链页的链接配置 |
| 书签导航 | `src/config/booknavConfig.ts` | 书签分类和链接 |

---

## 14. 本次已做的个性化变更（备案）

- 站点标题/副标题/描述 → 个人内容
- 昵称 → 莜之箐；GitHub / Email 更新；QQ / RSS 链接置空
- 导航栏"链接"菜单：只保留 GitHub，删除 Gitee / QQ群 / Firefly文档
- 打赏页：删除 ko-fi / 爱发电，清空示例打赏者；赞赏码换成你的新图
- 相册：删除模板示例相册（可爱流萤 / 加密示例）
- 动态：删除 4 个模板测试动态
- 文章：删除 11 个模板示例文章 + guide/images 子目录；新增 `hello-world.md`
- 关于页：改写为个人介绍
- 注释清理：移除作者注释里的 CuteLeaf 邮箱和示例域名
- `.gitignore`：追加 `GitHub_Pages构建源修复.md`（仅本地排错参考）

---

> 还有哪部分不会操作，直接把这个文档翻到对应章节照着做。做之前先备份（复制一份文件），出问题找我。
