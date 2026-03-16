# 分流规则设计方案

## 规则来源

基于两套规则合并：
1. **dler-io/Rules** — 基础规则（按服务拆分，远程引用）
2. **SNTP 自定义配置** — 补充 dler-io 没有的服务规则

## 节点排序

按机场顺序：**SNTP → Flower → WCloud**

### 节点重命名规则

| 机场 | 原始节点名 | 重命名后 |
|------|-----------|---------|
| SNTP | HK01, HK02... | HK01 [SNTP], HK02 [SNTP]... |
| SNTP | TW01, SG01, JP01, US01, KR, DE01 | 同上加 [SNTP] |
| Flower | 香港实验性 IEPL 专线 1 | 香港IEPL 01 [Flower] |
| Flower | 日本标准 IEPL 专线 1 | 日本IEPL 01 [Flower] |
| WCloud | 香港01, 香港02... | 香港01 [WCloud] |
| WCloud | 香港家宽HKT | 香港家宽HKT [WCloud] |

## 策略组设计

### 主选择组
```
🚀 节点选择 (select)
├── ♻️ 自动选优 (url-test, 全部节点)
├── 🇭🇰 香港 (url-test, 匹配港/HK/香港/Hong)
├── 🇯🇵 日本 (url-test, 匹配日/JP/日本/Japan)
├── 🇸🇬 新加坡 (url-test, 匹配新/SG/新加坡/Singapore)
├── 🇺🇸 美国 (url-test, 匹配美/US/美国/United)
├── 🇨🇳 台湾 (url-test, 匹配台/TW/台湾/Taiwan)
├── 🇰🇷 韩国 (url-test, 匹配韩/KR/韩国/Korea)
├── 🇬🇧 欧洲 (url-test, 匹配英/德/俄/DE/GB/UK)
├── ☑️ 手动切换 (select, 全部节点)
└── DIRECT
```

### 用途组

| 策略组 | 默认走向 | 规则来源 |
|--------|---------|---------|
| 📺 Emby影院 | 🚀 节点选择 | **自建** Emby.list（SNTP 配置提取） |
| 📺 YouTube | 🚀 节点选择 | dler-io Media/YouTube.yaml |
| 📺 Netflix | 🚀 节点选择 | dler-io Media/Netflix.yaml |
| 📺 Disney | 🚀 节点选择 | dler-io Media/Disney Plus.yaml |
| 📺 HBOMAX | 🚀 节点选择 | dler-io Media/Max.yaml |
| 🐭 Twitch | 🚀 节点选择 | **自建** Twitch.list（SNTP 配置提取） |
| 🎵 Spotify | 🚀 节点选择 | dler-io Media/Spotify.yaml |
| 📺 Bilibili | DIRECT | dler-io Media/Bilibili.yaml |
| 🌏 巴哈姆特 | 🚀 节点选择 | dler-io Media/Bahamut.yaml |
| 💬 AI Suite | 🚀 节点选择 | dler-io AI Suite.yaml |
| 💬 Cursor | 🚀 节点选择 | **自建** Cursor.list |
| 💬 Copilot | 🚀 节点选择 | **自建** Copilot.list（含 OpenAI 详细规则） |
| 💬 Claude | 🚀 节点选择 | **自建** Claude.list |
| 💬 Gemini | 🚀 节点选择 | **自建** Gemini.list |
| 💬 Perplexity | 🚀 节点选择 | **自建** Perplexity.list |
| ✈️ Bing | 🚀 节点选择 | **自建** Bing.list |
| 📲 Telegram | 🚀 节点选择 | dler-io Telegram.yaml |
| 🌏 Discord | 🚀 节点选择 | dler-io Discord.yaml |
| 🌏 Twitter | 🚀 节点选择 | **自建** Twitter.list（SNTP 更详细） |
| 🌏 Steam | DIRECT | dler-io Steam.yaml |
| 🌏 TikTok | 🚀 节点选择 | dler-io TikTok.yaml |
| 🌏 Dropbox | 🚀 节点选择 | **自建** Dropbox.list |
| 💰 Crypto | 🚀 节点选择 | dler-io Crypto.yaml |
| 🍎 Apple TV | 🚀 节点选择 | dler-io Media/Apple TV.yaml |
| 🍎 Apple | DIRECT | dler-io Apple.yaml |
| 🌏 Xbox | DIRECT | **自建** Xbox.list |
| Ⓜ️ 微软 | DIRECT | dler-io Microsoft.yaml |
| 📢 Google | 🚀 节点选择 | dler-io Google FCM.yaml |
| 🌏 GFWlist | 🚀 节点选择 | SNTP 配置内嵌的 GFW 列表 |
| 🌏 PrivateTracker | DIRECT | **自建** PrivateTracker.list |
| 🛑 广告拦截 | REJECT | dler-io AdBlock.yaml |
| 🎯 国内直连 | DIRECT | dler-io Domestic.yaml + GEOIP,CN |
| 🐟 漏网之鱼 | 🚀 节点选择 | FINAL |

## 需要自建的规则文件

从 SNTP 配置中提取，放到 GitHub 仓库 `rules/` 目录：

1. `Emby.list` — Emby 影院域名和 IP
2. `Cursor.list` — Cursor IDE
3. `Copilot.list` — GitHub Copilot + OpenAI 详细规则
4. `Claude.list` — Anthropic Claude
5. `Gemini.list` — Google Gemini
6. `Perplexity.list` — Perplexity AI
7. `Bing.list` — Bing 搜索
8. `Twitch.list` — Twitch 直播
9. `Twitter.list` — Twitter/X
10. `Dropbox.list` — Dropbox
11. `Xbox.list` — Xbox 游戏
12. `HBOMAX.list` — HBO Max（dler-io 有 Max.yaml 但可能不够全）
13. `PrivateTracker.list` — PT 站
14. `GFWlist.list` — GFW 列表（可选，用现有 GFW 列表替代）

## 实施步骤

1. 创建 GitHub 仓库（如 github.com/jandyx/proxy-rules）
2. 从 SNTP 配置提取自建规则文件
3. 编写 subconverter config.ini
4. 上传到 GitHub
5. 更新 CF Worker 的 SUBCONFIG 环境变量指向新 config.ini
6. 测试各客户端
