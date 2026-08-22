---
title: 'CC Switch 修复 Codex 桌面端：问题排查与解决手册'
published: 2026-08-22
description: 'CC Switch 修复 Codex 桌面端桌面端切换第三方/国产模型时的常见问题排查手册，涵盖路由断裂、显示异常、认证冲突等场景。'
tags: [CC Switch, Codex, 配置, 故障排查]
category: 技术笔记
slug: cc-switch-codex-troubleshooting
---

------

## 〇、先建立两个基本认知

### 0.1 Codex 的两个核心配置文件

| 文件          | 作用                                    | 典型位置                                          |
| ------------- | --------------------------------------- | ------------------------------------------------- |
| `config.toml` | 用什么模型、走哪个 provider、接口协议   | `C:\Users\<用户名>\.codex\config.toml`（Windows） |
| `auth.json`   | 认证方式（ChatGPT 官方登录 or API Key） | 同上目录                                          |

CC Switch 切换时**需要同时改这两个文件**，它有时只改了其中一个——这是至少一半"切换不生效"问题的根源。

如果你安装时改过 Codex 的数据目录（例如 `D:\codex-home`），电脑上可能存在两份 `config.toml`。CC Switch 可能改了 C 盘那份，而 Codex 实际读取的是 D 盘那份。排查时先确认 Codex 真正在读哪个文件。

### 0.2 路由 vs 显示：两类问题必须分开

- **路由断了**：请求根本送不出去（401/404/400/model-not-found）。这是配置或模型本身的问题。
- **路由通了但桌面版不显示**：CLI 里 `/model` 能看到模型、请求也正常，只是桌面版选择器看不见。这是显示问题（客户端过滤缺陷），有专门的绕行方案。

**一条命令区分两者**：在同一个文件夹打开终端运行 `codex`（CLI），输入 `/model`。

- CLI 也失败 → 路由问题，看第三章。
- CLI 成功、只有桌面版看不见 → 显示问题，看第二章。

------

## 一、30 秒快速诊断表

| 你看到的现象                         | 实际的问题                                                  | 跳转    |
| ------------------------------------ | ----------------------------------------------------------- | ------- |
| 切完模型重启 Codex，还是显示 GPT-5.5 | `config.toml` 与 `auth.json` 没同步，或改错了路径的那份     | 问题 1  |
| 选择器只显示 "Custom"，没有模型名    | 模型内联设置、缺少目录元数据（正常现象，能用）              | 问题 5  |
| 模型名称位置显示空白                 | `[model_providers.xxx]` 下缺 `name` 字段                    | 问题 6  |
| CLI 的 `/model` 能列出，桌面版没有   | 桌面版客户端过滤缺陷（issue #19694）                        | 问题 7  |
| 选择器下拉列表整个是空的             | 目录缺失或格式错误（旧版 CC Switch）                        | 问题 8  |
| 发图片直接报错，之后纯文字也报错     | DeepSeek API 不支持图片输入，污染了会话历史                 | 问题 2  |
| Key 明明正确却报 401                 | Key 夹带空格/不可见字符/乱码                                | 问题 3  |
| 每个请求都 404                       | `wire_api = "chat"` 已废弃，或网关无 `/responses` 端点      | 问题 9  |
| 切回 GPT-5.5 后依然显示国产模型      | `auth.json` 残留 api-key 模式 / `config.toml` 残留 provider | 问题 4  |
| 切换后会话记录消失、插件变灰         | 认证体系切换导致（非数据丢失，新版已改善）                  | 问题 10 |
| 启动时打印 provider 被忽略的警告     | provider 写在了项目级 `.codex/config.toml`                  | 问题 11 |
| 配置全对但列表还是旧的/空的          | `models_cache.json` 缓存过期                                | 问题 12 |

------

## 二、切换类问题（改了不生效 / 改不回来）

### 问题 1：切完模型重启 Codex，还是显示 GPT-5.5

**典型报错**（CLI 中）：

```
{"type":"error","status":400,"error":{"type":"invalid_request_error",
"message":"The 'deepseek-v4-flash' model is not supported when using Codex with a ChatGPT account."}}
```

**根因**：CC Switch 只改了 `config.toml` 和 `auth.json` 中的一个；或者改的是另一路径下的副本。于是 Codex 检测到"ChatGPT 账号登录态 + 第三方模型名"对不上，直接拒绝。

**排查步骤**：

1. 确认 Codex 实际读取的配置目录（默认 `~/.codex/`，改过安装路径的去找对应目录）。
2. 打开该目录的 `config.toml`，检查是否包含 `model = "deepseek-v4-flash"` 之类的模型行和 `[model_providers.custom]` 配置块。
3. 打开同目录的 `auth.json`，检查 `auth_mode` 是否已从 `chatgpt` 变为 api-key 模式。
4. 两处只改了一处 → 把没改的那份补齐，或从备份恢复后重新操作。

**修复（推荐用备份恢复）**：

- CC Switch 每次改配置前会自动备份，文件名带时间戳，如 `config.toml.bak.20260602121223`。
- 找到切换前的正确备份，**两个文件一起**覆盖回去，再在 CC Switch 里重新操作一遍。
- 有 Claude Code / opencode 等本地 agent 的话，直接把两个文件路径丢给它，让它对比差异并修复，效率远高于手动翻。

### 问题 4：切回 GPT-5.5 后依然显示国产模型

**根因**：与问题 1 同源。禁用了 DeepSeek，但 `auth.json` 仍停留在 api-key 模式，或 `config.toml` 里残留 `[model_providers.custom]` 配置。

**修复**：

1. 用 CC Switch 的备份（或你自己手动备份的）把 `config.toml` 和 `auth.json` **同时**恢复到切换之前的状态。
2. 重启 Codex，重新登录 ChatGPT 账号。
3. 恢复后插件、历史会话会一起回来。

### 问题 10：切换模型后会话记录全没了、插件变灰

**这不是 bug，也不是数据丢失。** 切换模型等于换了一套认证体系，Codex 把你当成了新用户，所以旧会话和插件暂时不可见。

- CC Switch 升级到 **v3.16.1+** 后，切回原模型的流程已修复：切回 GPT-5.5 且配置恢复正确，会话记录、插件全部原样回来。
- **实操建议**：切到国产模型前，重要会话内容先复制出来存一份；用完切回即可，不用慌。

------

## 三、请求类问题（路由断了）

### 问题 2：能用了，但一发图片就报错

**典型报错**：

```
CC Switch local proxy failed while handling Codex endpoint /responses.
Provider: DeepSeek; model: deepseek-v4-flash; upstream_status: HTTP 400;
cause: Failed to deserialize the JSON body into the target type:
messages[6]: unknown variant `image_url`, expected `text`
```

**根因**：**部分模型不支持图片输入**，与 CC Switch 和你的配置都无关。

**最坑的一点**：一旦某会话里发过图片报错，该会话后续**纯文字也会持续报错**——因为 Codex 每次都会把含图片的历史消息一起发给 API。

### 问题 3：API Key 明明正确，却报 401

**典型报错**：`unexpected status 401 Unauthorized: Incorrect API key provided: sk-bb527*****0175`

**根因**：CC Switch 输入框的玄学问题——复制粘贴进去的 Key 可能夹带首尾空格、不可见字符，甚至保存时出现乱码。肉眼看不出来。

**修复**（注意顺序）：

1. **先关掉 Codex**。
2. 在 CC Switch 里删除该供应商的 API Key。
3. 重新去服务商后台**重新复制一次** Key，粘贴保存。
4. 再打开 Codex。
5. 还不行就**手动输入** Key（反人类但能彻底排除剪贴板问题）。

### 问题 9：启动报错，或每个请求都 404

**根因**：`wire_api = "chat"`。Codex 在 2026 年 2 月移除了旧的 chat/completions 路径，`responses` 现在是唯一合法取值（也是默认值）。或者你的网关根本没有 `/responses` 端点。

**修复**：

```
[model_providers.custom]
wire_api = "responses"
base_url = "https://你的网关地址/v1"   # 以 /v1 结尾，末尾不带斜杠
```

- 网关必须在 `{base_url}/responses` 暴露 OpenAI 兼容的 Responses 端点。只支持 `/chat/completions` 的网关会让每个请求 404——看起来像模型问题，实际是协议不匹配。
- 顺带检查：`base_url` 末尾多一个斜杠或路径写错，会导致连接时不时被重置。

### 附：其他伪装成显示问题的路由错误

| 症状                          | 真正原因                                       | 修复                                                |
| ----------------------------- | ---------------------------------------------- | --------------------------------------------------- |
| 每个请求 401（手写配置场景）  | 配了 `env_key` 但环境变量从未导出              | 在 shell 配置里 `export XXX_API_KEY=sk-...`         |
| 模型能跑但返回错误的输出      | 目录里的 `slug` 与 provider 实际模型 ID 不一致 | `slug` 改成与发送给 provider 的字符串**逐字符一致** |
| curl 网关返回 model-not-found | 模型字符串写错                                 | 核对服务商后台的模型 ID，别猜                       |

------

## 四、显示类问题（路由通了，桌面版看不见）

### 问题 5：选择器只显示 "Custom"，没有模型名

**根因**：你只在 `config.toml` 里内联写了 `model = "xxx"`，没有模型目录，选择器没有可展示的元数据。

**结论：这是正常现象，不用修。** 请求会正常发往你的模型。如果不在乎下拉列表好不好看，到这里就可以停了。

### 问题 6：模型名称位置显示空白

**根因**：`[model_providers.custom]` 配置块下缺少 `name` 字段（Windows 上 CC Switch 偶尔写入不完整）。

**修复**：手动补上，重启 Codex：

```
[model_providers.custom]
name = "deepseek"        # ← 就是这个字段控制界面显示名称
base_url = "https://api.deepseek.com"
wire_api = "responses"
```

`name` 的值随意，写你能认出来的名字即可（"deepseek"、"qwen" 都行）。

### 问题 7：CLI 的 /model 能列出模型，桌面版选择器没有

**根因**：Codex 桌面版的**客户端过滤缺陷**（openai/codex issue #19694，2026-04-26 提交，截至目前未关闭）。app-server 的 `model/list` 端点正常返回了你的模型，但桌面版渲染层在到达下拉列表前把本地配置的条目丢掉了。**后端知道你的模型，前端拒绝显示。改配置修不好。**

**修复（按顺序）**：

**修复 A——内联模型绕行（永远管用，最先做）**：

```
# ~/.codex/config.toml（用户级，不是项目文件夹里）
model = "moonshotai/kimi-k2.7-code"
model_provider = "custom"


[model_providers.custom]
name = "myprovider"
base_url = "https://你的网关/v1"
env_key = "MY_API_KEY"
wire_api = "responses"
```

彻底退出 Codex 桌面版再重开。选择器显示 "Custom" 无妨，请求都会发往正确模型。换模型只需改 `model` 那一行字符串。

**修复 B——加模型目录，让选择器显示真名**：在 `config.toml` 顶部加 `model_catalog_json` 指向一个 JSON 文件：

```
model_catalog_json = "C:/Users/你/.codex/my-models.json"
```

最小目录条目：

```
{
  "models": [
    {
      "slug": "moonshotai/kimi-k2.7-code",
      "display_name": "Kimi K2.7 Code",
      "description": "Coding model via gateway",
      "context_window": 262000,
      "max_context_window": 262000,
      "supported_in_api": true,
      "visibility": "list",
      "priority": 1
    }
  ]
}
```

`slug` 必须等于发给 provider 的模型字符串。目录只在**启动时**读取一次，改动后必须重启桌面版。

### 问题 8：选择器下拉列表整个是空的

**根因**：早于 **v3.16.5** 的 CC Switch 生成的目录格式与 Codex 选择器期望的对不上（cc-switch issue #3668），路由正常但 `/model` 返回空。

**修复**：

1. **升级 CC Switch 到 v3.16.5 或更高版本**——该版本会为使用原生 Responses 端点（`apiFormat: "openai_responses"`）的供应商生成 `~/.codex/cc-switch-model-catalog.json`。
2. **升级后必须把每个原生 provider 重新保存一次**——目录只在保存时重新生成，旧 provider 不会自动迁移。
3. 验证目录已生成：

```
# Windows PowerShell 里用 findstr 代替 grep
cat ~/.codex/cc-switch-model-catalog.json | grep -o '"slug":[^,]*'
# 如果为空，回 CC Switch 重新保存该 provider 后再查
```

**v3.16.5 的两个注意点**：

- 目录生成与"本地路由"开关已解耦：无论本地路由是否开启，原生 Responses 供应商都会生成目录；Chat 格式供应商仍走代理转换。
- 少数国产模型（MiMo、LongCat、MiniMax、Qwen3-Coder）的网关不支持 OpenAI 内置 `web_search`，v3.16.5 默认对它们关闭该工具以避免 400 报错——预期这些模型的网络搜索不可用，不是故障。

### 问题 11：启动时打印 provider 被忽略的警告

**根因**：`model_provider` / `model_providers` 写在了项目级 `.codex/config.toml`（某个仓库目录里）。provider 定义**只在用户级 `~/.codex/config.toml` 生效**，项目级的会被忽略并打印警告。

**修复**：

```
# 检查 provider 到底写在哪个文件
grep -rn "model_providers" ~/.codex/config.toml ./.codex/config.toml 2>/dev/null
```

把 `[model_providers.*]` 配置块和 `model_provider = "..."` 挪到 `~/.codex/config.toml`，项目级配置只留仓库专属内容（如指令文件）。

### 问题 12：配置全对，但列表还是旧的/空的

**根因**：`~/.codex/models_cache.json` 缓存过期。切换 provider 或编辑目录后它不总会重新同步。

**修复**：**删掉 `models_cache.json`**，下次启动会强制重建。在断定目录本身出错之前，先试这一招。

------

## 五、通用排查流程（一张图记住）

```
桌面版看不到模型 / 切换不生效
        │
        ▼
终端运行 codex（CLI），输入 /model
        │
   ┌────┴────┐
 CLI 也失败   CLI 正常、只有桌面版不行
   │              │
   ▼              ▼
 路由问题        显示问题（#19694 过滤）
 检查：          处理：
 · config/auth   · 修复 A：内联 model 绕行
   两文件是否     · 修复 B：补 model_catalog_json
   同步且路径对   · CC Switch 升 v3.16.5+
 · wire_api=     并逐个重新保存 provider
   responses
 · Key 无空格    仍不行：
   乱码          · 删 models_cache.json
 · base_url      · 检查 name 字段
   以 /v1 结尾
```

**验证模型真正加载的可靠方法**：不要相信选择器（坏的就是它），在更下层验证——

1. CLI 里 `/model` 能列出 → 目录和 provider 都正确；
2. 用密钥对网关直接 `curl {base_url}/responses` → 确认路由本身能解析；
3. 用网关的话，在网关的请求日志面板确认模型路由命中了预期后端。

------

## 六、操作规范与预防（比修复更重要）

1. **任何切换前先手动备份** `config.toml` 和 `auth.json`。CC Switch 虽会自动备份（`.bak.时间戳`），但自己再存一份双保险。
2. **操作顺序铁律：先关 Codex → 改配置 → 再开 Codex**。顺序反了，很多"不生效"就是这么来的。
3. **CC Switch 保持最新版**。v3.16.1 修复了切换后会话丢失，v3.16.5 修复了目录格式。很多坑在新版本已经不存在。
4. **切换三种场景速查**：
   - 官方 → 国产：关 Codex → CC Switch 加供应商填 Key → 设置里开路由总开关+Codex 路由 → 启用供应商 → 开 Codex。
   - 国产 → 官方：关 Codex → 禁用供应商 → 关路由 → 开 Codex 重新登录（配置乱了就用备份恢复）。
   - 国产 A → 国产 B：关 Codex → 直接启用新供应商（旧的自动禁用）→ 开 Codex，路由不用动。
5. **善用本地 agent 排查**：把 `config.toml`、`auth.json` 和备份文件丢给 Claude Code / opencode，让它对比差异并修复，比手动翻文件高效得多。
