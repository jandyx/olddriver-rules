# 订阅聚合系统

一个代理订阅聚合 + 转换 + 分发系统，将多家机场订阅链接整合为一条统一链接，支持自动识别客户端（Surge/Clash/Quantumult X/Loon/Shadowrocket/SingBox 等），配合分流规则输出完整配置。

## 目录

- [项目概述](#项目概述)
- [方案选型](#方案选型)
- [快速开始](#快速开始)
- [部署步骤](#部署步骤)
- [当前部署信息](#当前部署信息)
- [订阅源状态](#订阅源状态)
- [遇到的问题](#遇到的问题及解决方案)
- [规则仓库](#规则仓库)
- [Surge Panel](#surge-panel)
- [待办事项](#待办事项)
- [常见问题](#常见问题)

## 项目概述

该系统通过多个开源组件协作，实现从多个机场的订阅源聚合，到客户端特定格式转换，再到最终配置分发的完整流程。

### 核心功能

- **订阅聚合**：从多个机场源拉取订阅数据，合并为统一链接
- **格式转换**：根据客户端类型自动转换为相应格式（Surge/Clash/Quantumult X/Loon/SingBox 等）
- **UA 识别**：根据请求头自动识别客户端，返回对应格式配置
- **分流规则**：集成国内外分流规则，支持远程引用和本地注入
- **KV 存储**：利用 Cloudflare KV 存储订阅数据，支持动态更新

## 方案选型

### 整体架构

```
┌─────────────────────────────────────────────────────────┐
│  多家机场订阅源                                         │
│  - sntp.me (Trojan)                                      │
│  - 47.102 机场 (SS)                                      │
│  - Flower_SS (SS + obfs)                                 │
│  - oics.net/dler.io (SS 2022)                            │
└─────────────────────────┬───────────────────────────────┘
                          │
                          ↓
            ┌─────────────────────────┐
            │  CF Worker              │
            │  UA 识别 + 聚合 + KV     │
            │ (CF-Workers-SUB)        │
            └──────────────┬──────────┘
                           │
                           ↓
        ┌──────────────────────────────────┐
        │  subconverter 转换引擎            │
        │  (Render Docker)                 │
        │  - 全格式转换                    │
        │  - 规则注入                      │
        │  - 策略组配置                    │
        └──────────────┬───────────────────┘
                       │
                       ↓
        ┌──────────────────────────────────┐
        │  最终配置                        │
        │  - Surge                         │
        │  - Clash                         │
        │  - Quantumult X                  │
        │  - Loon / Shadowrocket / SingBox │
        └──────────────────────────────────┘
```

### 核心组件

| 项目 | 功能 | 部署位置 | 说明 |
|------|------|---------|------|
| **cmliu/CF-Workers-SUB** | 前端入口、聚合、UA 识别、KV 管理 | Cloudflare Worker | JS 编写，处理请求路由和参数解析 |
| **tindy2013/subconverter** | 格式转换引擎、规则注入、策略组 | Render (Docker) | C++ 编写，处理复杂的转换逻辑 |
| **dler-io/Rules 或 ACL4SSR** | 分流规则集 | 远程 URL 引用 | 支持动态引用，无需自建 |

### 为什么选这套方案

| 优点 | 说明 |
|------|------|
| **完全免费** | CF Worker (10万次/天)、Render (免费容器)、GitHub (免费 CDN)，零成本运行 |
| **成熟生态** | subconverter 是全球最成熟的代理转换后端，格式覆盖最全 |
| **技术解耦** | 前端路由与后端转换分离，便于独立维护和扩展 |
| **自动识别** | Worker 通过 UA 头自动识别客户端，用户体验无感 |
| **云原生** | 无需自建服务器，全部云托管，高可用无运维 |

### 为什么不选其他方案

| 方案 | 为什么不选 |
|------|----------|
| Render 直接部署 subconverter | 无 UA 识别能力，无法聚合多源 |
| CF Worker 直接运行转换逻辑 | Worker 只支持 JS，无法运行 C++ 程序 |
| 自建 VPS 部署 | 成本高（$5+/月），维护负担重 |
| Vercel + Function | 仍需后端运行 subconverter，解决不了问题 |

## 快速开始

### 前置要求

- Cloudflare 账号（免费）
- Render 账号（免费，需信用卡验证）
- GitHub 账号（用于 Fork 规则仓库、存储脚本）
- 命令行工具：Git、Node.js 12+、npm/yarn

### 最小化部署流程

1. **Render 部署 subconverter**
   ```bash
   # 访问 https://render.com，创建 Web Service
   # 镜像：docker.io/tindy2013/subconverter:latest
   # 得到后端 URL（如 https://subconverter-xxx.onrender.com）
   ```

2. **Cloudflare Worker 部署前端**
   ```bash
   git clone https://github.com/cmliu/CF-Workers-SUB.git
   cd CF-Workers-SUB
   npm install -g wrangler
   wrangler login
   wrangler deploy
   ```

3. **配置 KV 和环境变量**
   ```bash
   # 在 Cloudflare Dashboard 创建 KV namespace
   # 将订阅链接写入 KV 的 LINK.txt 键
   # 在 Worker 设置中配置环境变量：SUBAPI、SUBCONFIG、SUBNAME
   ```

4. **测试订阅链接**
   ```
   https://your-worker.workers.dev/auto
   ```

详细步骤见 [部署步骤](#部署步骤)。

## 部署步骤

### 第一步：Render 部署 subconverter

**目的**：部署开源转换引擎，CF Worker 将通过调用其 API 进行格式转换。

**步骤**：

1. 访问 [render.com](https://render.com)，注册账号
   - 需绑定信用卡进行身份验证（不会扣费）
   - 免费账户提供免费实例资源

2. 点击 "New" → "Web Service"

3. 选择 "Existing Image" 选项

4. 填写镜像 URL：
   ```
   docker.io/tindy2013/subconverter:latest
   ```

5. 配置部署参数：
   - **Name**：`subconverter-latest`（或自定义）
   - **Region**：Singapore（离中国最近，延迟最低）
   - **Instance Type**：Free（免费型）
   - **其他选项**：全部保持默认

6. 点击 "Deploy Web Service" 开始部署

7. 等待部署完成，获得 URL：
   ```
   https://subconverter-latest-xxxx.onrender.com
   ```

**重要注意事项**：

- **冷启动时间**：首次请求可能需要 30-50 秒，因为免费实例闲置 15 分钟后会进入休眠状态
- **保活方案**：
  - 方案 A（不推荐）：用 CF Cron Trigger 每 14 分钟 ping 一次后端，但可能被视为滥用免费额度
  - 方案 B（推荐）：接受偶尔的冷启动延迟，用户订阅更新通常是后台行为，不频繁
- **容器更新**：Render 会自动更新镜像，无需手动干预

### 第二步：Cloudflare Worker 部署前端

**目的**：部署前端入口，实现 UA 识别、订阅聚合、参数处理和 KV 管理。

**步骤**：

1. 注册 Cloudflare 账号
   - 访问 [dash.cloudflare.com](https://dash.cloudflare.com)
   - 如已有域名在 CF 托管，可直接使用；否则注册 workers.dev 子域名

2. 创建 API Token
   - 用户头像 → "API Tokens"
   - 点击 "Create Token"
   - 选择权限：
     - `Account.Workers Scripts - Edit`
     - `Account.Workers KV Storage - Edit`（可选，如用 CLI 创建 KV）
   - 复制 Token 并保存

3. 克隆 CF-Workers-SUB 仓库
   ```bash
   git clone https://github.com/cmliu/CF-Workers-SUB.git
   cd CF-Workers-SUB
   npm install
   ```

4. 清空默认订阅数据
   - 打开 `src/_worker.js` 或 `_worker.js`
   - 找到第 15-17 行的 `MainData` 或默认订阅配置
   - 删除所有默认链接，保留空数组或空字符串
   - 这是必须的，否则会混入无效节点

5. 登录 Cloudflare 并部署
   ```bash
   export CLOUDFLARE_API_TOKEN="your-token-here"
   export CLOUDFLARE_ACCOUNT_ID="your-account-id"
   npm install -g wrangler
   wrangler login
   wrangler deploy
   ```

6. 创建 KV 命名空间
   ```bash
   wrangler kv namespace create "sub-worker-kv"
   # 输出示例：
   # ┌─────────────────────────────────┐
   # │ ✓ Created namespace             │
   # │                                 │
   # │ [[kv_namespaces]]               │
   # │ binding = "sub-worker-kv"       │
   # │ id = "5082e93d2ad54aa2a72ce..." │
   # └─────────────────────────────────┘
   ```

7. 更新 `wrangler.toml`
   - 将返回的 KV namespace ID 和 binding name 填入：
   ```toml
   [[kv_namespaces]]
   binding = "sub-worker-kv"
   id = "YOUR_KV_NAMESPACE_ID"
   preview_id = "YOUR_KV_NAMESPACE_ID"
   ```

8. 配置环境变量（Secrets）
   ```bash
   wrangler secret put SUBAPI
   # 输入：subconverter-latest-xxxx.onrender.com

   wrangler secret put SUBCONFIG
   # 输入：https://raw.githubusercontent.com/cmliu/ACL4SSR/main/Clash/config/ACL4SSR_Online_MultiCountry.ini

   wrangler secret put SUBNAME
   # 输入：MyProxy（或自定义项目名称）
   ```

9. 重新部署以应用配置
   ```bash
   wrangler deploy
   ```

10. 验证部署
    - 访问 `https://your-worker.workers.dev/auto` 应该返回配置文件

### 第三步：写入订阅数据到 KV

**目的**：存储聚合后的订阅链接，Worker 从 KV 读取并处理。

**方案 A：通过 wrangler CLI 写入**（推荐用于初始化）

```bash
# 准备订阅数据，可以是：
# 1. 直接的 base64 订阅
# 2. 多个订阅链接用 | 分隔
# 3. JSON 格式的节点列表

# 写入 LINK.txt
wrangler kv key put --namespace-id=YOUR_KV_NAMESPACE_ID \
  "LINK.txt" "订阅数据内容"

# 验证写入
wrangler kv key list --namespace-id=YOUR_KV_NAMESPACE_ID
```

**方案 B：通过 CF Dashboard 写入**

1. 登录 Cloudflare Dashboard
2. Workers → KV → 找到你的 namespace
3. 点击 "Create Key"
4. 名称：`LINK.txt`
5. 值：粘贴订阅数据或订阅链接
6. 点击 "Save"

**方案 C：通过 Worker 前端页面写入**（如 CF-Workers-SUB 支持）

- 访问 `https://your-worker.workers.dev/admin`
- 在管理页面输入订阅链接
- 点击保存

### 第四步：验证部署

**测试不同客户端的订阅链接**：

```bash
# 测试自适应（自动识别）
curl -H "User-Agent: Surge" \
  "https://your-worker.workers.dev/auto"

# 测试 Clash
curl -H "User-Agent: Clash" \
  "https://your-worker.workers.dev/auto"

# 测试 Quantumult X
curl -H "User-Agent: Quantumult" \
  "https://your-worker.workers.dev/auto"
```

**预期结果**：返回对应格式的配置文件。

## 当前部署信息

### 部署地址

| 组件 | 地址 | 说明 |
|------|------|------|
| **CF Worker 入口** | `https://sub-worker.YOUR_CF_SUBDOMAIN.workers.dev` | 前端聚合和路由 |
| **subconverter 后端** | `https://subconverter-latest-ovre.onrender.com` | 转换引擎 |
| **KV Namespace ID** | `YOUR_KV_NAMESPACE_ID` | 数据存储 |
| **CF Account ID** | `YOUR_CF_ACCOUNT_ID` | 账号标识 |

### 客户端订阅地址

| 客户端 | 订阅链接 | 说明 |
|--------|---------|------|
| **自适应（推荐）** | `https://sub-worker.YOUR_CF_SUBDOMAIN.workers.dev/auto` | 根据 UA 自动识别格式 |
| **Surge** | `https://sub-worker.YOUR_CF_SUBDOMAIN.workers.dev/auto?surge` | 强制 Surge 格式 |
| **Clash** | `https://sub-worker.YOUR_CF_SUBDOMAIN.workers.dev/auto?clash` | 强制 Clash YAML 格式 |
| **Clash Meta** | `https://sub-worker.YOUR_CF_SUBDOMAIN.workers.dev/auto?clashmeta` | Clash Meta 专用格式 |
| **Quantumult X** | `https://sub-worker.YOUR_CF_SUBDOMAIN.workers.dev/auto?quanx` | Quantumult X 专用格式 |
| **Loon** | `https://sub-worker.YOUR_CF_SUBDOMAIN.workers.dev/auto?loon` | Loon 专用格式 |
| **Shadowrocket** | `https://sub-worker.YOUR_CF_SUBDOMAIN.workers.dev/auto?sr` | Shadowrocket 专用格式 |
| **SingBox** | `https://sub-worker.YOUR_CF_SUBDOMAIN.workers.dev/auto?singbox` | SingBox 专用格式 |
| **Base64** | `https://sub-worker.YOUR_CF_SUBDOMAIN.workers.dev/auto?b64` | 原始 base64 编码 |

### 使用方法

**在客户端中导入订阅**：

1. 打开客户端（如 Surge、Clash 等）
2. 找到 "订阅管理" 或 "导入配置" 选项
3. 复制对应的订阅链接（如自适应链接 `/auto`）
4. 粘贴到客户端
5. 客户端自动识别格式并导入

**定期更新**：

- 大多数客户端支持定时自动更新订阅
- 通常设置为每 6-24 小时更新一次
- 也可手动点击"更新订阅"立即获取最新节点

## 订阅源状态

### 订阅 1：sntp.me（Trojan）

| 属性 | 值 |
|------|-----|
| **链接** | `https://sub1.sntp.me/baseStr?token=YOUR_SNTP_TOKEN&sign=YOUR_SNTP_SIGN&ts=TIMESTAMP&nonce=YOUR_SNTP_NONCE&v=v2&timestamp=152005` |
| **协议** | Trojan |
| **节点数** | 20 个 |
| **节点区域** | HK01-04, TW01-02, SG01-04, JP01-04, US01-04, KR, DE01 |
| **Worker 自动拉取** | ✅ 正常 |
| **subconverter 转换** | ✅ 正常 |
| **状态** | 正常运行 |

**说明**：该订阅源稳定可靠，支持自动拉取和格式转换，无已知问题。

---

### 订阅 2：47.102 机场（SS）

| 属性 | 值 |
|------|-----|
| **链接** | `https://YOUR_WCLOUD_IP:9090/api/sub?token=YOUR_WCLOUD_TOKEN` |
| **协议** | Shadowsocks (aes-128-gcm) |
| **节点数** | 47 个 |
| **节点区域** | 香港 01-04, 台湾 01-02, 新加坡 01-04, 日本 01-04, 美国 01-04, 韩国, 德国, 英国等全球节点 |
| **Worker 自动拉取** | ❌ 失败 |
| **subconverter 直接拉取** | ✅ 可以 |
| **当前方案** | 手动解码 base64 后写入 KV |
| **问题根因** | HTTPS 自签证书，CF Worker 不信任 |

**问题说明**：
- CF Worker 发出的 HTTPS 请求无法跳过证书验证
- subconverter 可以用 `scv=true` 参数跳过验证，但 Worker 不支持
- **临时解决**：在本地或信任的服务器手动拉取，解码后写入 KV
- **长期解决**：等待 CF 支持证书验证跳过，或机场升级正式证书

---

### 订阅 3：Flower_SS（SS + obfs 插件）

| 属性 | 值 |
|------|-----|
| **链接** | `https://api-huacloud.dev/sub?target=clash&insert=true&emoji=true&udp=true&clash.doh=true&new_name=true&filename=Flower_SS.yaml&url=https%3A%2F%2Fapi.xmancdn.com%2Fosubscribe.php%3Ftoken2%3DYOUR_FLOWER_TOKEN%26sip002%3D1` |
| **协议** | Shadowsocks (aes-128-gcm) + obfs 插件 |
| **节点数** | 88 个 |
| **节点区域** | 香港/日本/新加坡/美国/韩国/台湾 IEPL 专线 |
| **Worker 自动拉取** | ❌ 失败 |
| **当前方案** | 本地 YAML 文件转换为 ss:// URI 后写入 KV |
| **问题根因** | 机场检测到服务器/CDN IP，返回 403 Forbidden |

**问题说明**：
- 机场部署了反爬虫机制，检测到请求来自 Cloudflare/CDN IP 即拒绝
- 原始订阅链接 `api.xmancdn.com` 同样被 403
- **临时解决**：从本地网络拉取 YAML，手动转换后写入 KV
- **转换方法**：
  ```python
  # 解析 YAML，提取节点信息
  # 按 Shadowsocks URI scheme 格式转换：ss://method:password@host:port#name
  # 多个节点用换行符分隔
  ```
- **长期解决**：寻找支持代理或反检测的转换服务；或与机场联系获取服务器直连链接

---

### 订阅 4：oics.net / dler.io（SS 2022）

| 属性 | 值 |
|------|-----|
| **链接** | `https://oics.net/api/v3/download.getFile/YOUR_OICS_FILE_ID?mu=smart&lv=2%7C3%7C5%7C6.ppt` |
| **协议** | Shadowsocks 2022 (blake3-aes-256-gcm) |
| **节点数** | 62 个 |
| **节点区域** | 香港/台湾/新加坡/日本/美国/韩国 IEPL 专线，含普通和 AC 双线路 |
| **Worker 自动拉取** | ✅ 能拉到 |
| **subconverter 转换** | ❌ 不支持 2022-blake3 加密 |
| **当前方案** | 手动解码后写入 KV；或利用 dler.io API 直接转换 |
| **问题根因** | subconverter v0.9.0 不支持 SS 2022-blake3-aes-256-gcm 加密方式 |

**重要发现**：
- `oics.net` 是 `dler.io` 的新域名，两者完全相同
- dler.io 官方支持多种格式 API，可直接转换而无需 subconverter
- **dler.io 格式 API**：
  ```
  ?surge=smart     → Surge 格式
  ?clash=smart     → Clash 格式
  ?mu=ss           → ss:// base64
  ?mu=ssr          → ssr:// base64
  ?mu=v2ray        → v2ray://
  ```
- **使用方法**：
  ```
  https://oics.net/api/v3/download.getFile/YOUR_OICS_FILE_ID...?surge=smart
  https://oics.net/api/v3/download.getFile/YOUR_OICS_FILE_ID...?clash=smart
  ```
- **文档**：https://docs.dler.io/

**解决方案**：
- 改用 dler.io 的原生转换 API，完全绕过 subconverter
- 在 CF Worker 中直接调用 oics.net/dler.io 的 API，追加对应的格式参数
- 不需要部署 subconverter，无需处理 2022-blake3 兼容性问题

---

## 遇到的问题及解决方案

### 问题 1：CF Worker 不能直接运行 subconverter

**现象**：部署 subconverter 到 CF Worker 时报错：不支持 C++ 程序。

**原因**：
- CF Worker 是 JavaScript runtime，只支持 JS 和 WebAssembly
- subconverter 用 C++ 编写，需要完整的 C++ 运行环境
- 即使编译为 WASM，复杂度和大小也超过 Worker 限制

**解决方案**：
- 使用 Render 免费 Docker 容器托管 subconverter
- CF Worker 作为前端，通过 HTTP 调用 Render 后端的 API
- 架构分离，各司其职

**参考命令**：
```bash
# Render 后端 API 调用示例
curl "https://subconverter-latest-xxxx.onrender.com/sub?target=clash&url=...&config=..."
```

---

### 问题 2：CF Worker 环境变量配置错误

**现象**：Worker 无法拉取订阅，日志显示 `LINKSUB` 为空。

**根因**：
- CF-Workers-SUB 源码中有多个变量名：`SUB`、`LINK`、`LINKSUB`
- 初始配置错误地使用了 `SUB` 变量，但代码实际读取的是 `LINKSUB`
- 绑定 KV 后，代码优先读取 KV 的 `LINK.txt` 键，`LINKSUB` 环境变量被忽略
- 混淆了"自建节点"（`LINK`）和"订阅链接"（`LINKSUB`）的作用

**解决方案**：
- 理解变量作用：
  - `LINK`：自建的 ss:// 等 URI 格式节点，直接写入 KV 的 `LINK.txt`
  - `LINKSUB`：远程订阅链接（base64 或其他格式的源地址），通过 Worker 拉取后与 LINK 聚合
  - `KV` 优先级最高，绑定后代码走 KV 分支
- 正确流程：
  ```
  多个订阅链接（或聚合后的节点）
          ↓
  写入 KV 的 LINK.txt
          ↓
  Worker 读取 KV
          ↓
  发送给 subconverter
  ```

**验证方法**：
```bash
# 查看 KV 内容
wrangler kv key get --namespace-id=xxx "LINK.txt"

# 如果为空或不存在，写入测试数据
wrangler kv key put --namespace-id=xxx "LINK.txt" "test-data"
```

---

### 问题 3：默认 MainData 包含无效节点

**现象**：聚合后的配置包含大量无效、已过期或非预期的节点。

**根因**：
- `_worker.js` 第 15-17 行包含硬编码的默认订阅数据
- 其中包括 `https://cfxr.eu.org/getSub` 等来源不明、易过期的订阅
- 如不清空，这些默认节点会被混入聚合配置

**解决方案**：
- 部署前必须清空 `MainData` 默认值
  ```javascript
  // 修改前
  const MainData = "https://cfxr.eu.org/getSub|https://...";

  // 修改后
  const MainData = "";  // 或 null，根据代码结构
  ```
- 所有节点数据都从 KV 的 `LINK.txt` 读取，避免硬编码混入

**验证**：
```bash
# 部署前检查源码
grep -n "MainData" _worker.js | head -5

# 清空后确认
sed -i '' 's/const MainData = ".*"/const MainData = ""/' _worker.js
```

---

### 问题 4：机场 403 封锁（Flower_SS）

**现象**：
- Worker 或 subconverter 调用 `api-huacloud.dev` 返回 403 Forbidden
- 原始链接 `api.xmancdn.com` 同样 403

**原因**：
- 机场部署了 IP 黑名单，检测到服务器/CDN IP（包括 Cloudflare、Render 等）即拒绝
- 可能原因：防止滥用、防止被破解、防止被转卖等

**解决方案**：

**方案 A：从本地网络拉取（推荐）**
- 在家里或有代理的网络环境下拉取订阅
- 解码 base64，转换为 ss:// URI 格式
- 写入 KV
```bash
# 示例：从本地拉取并转换
curl "https://api-huacloud.dev/..." | base64 -d > flower_ss.txt
# 手动解析或用脚本转换格式
# 写入 KV
wrangler kv key put --namespace-id=xxx "LINK.txt" "$(cat flower_ss.txt)"
```

**方案 B：使用机场的其他转换链接**
- 某些机场提供多个镜像地址或转换 API
- 尝试 `https://api-huacloud.dev` 的其他路径或参数

**方案 C：请求机场支持**
- 联系机场客服，请求提供服务器直连的订阅链接或 API
- 部分机场会为大用户单独提供

**长期**：转向提供反爬虫友好订阅的机场，或监控订阅可用性，失败时自动降级。

---

### 问题 5：HTTPS 自签证书（47.102 机场）

**现象**：
- Worker 访问 `https://YOUR_WCLOUD_IP:9090/api/sub?...` 失败
- 错误：`SSL certificate verification failed` 或 `CERTIFICATE_VERIFY_FAILED`

**原因**：
- 47.102 机场使用自签名 SSL 证书，不受信任的 CA 签发
- CF Worker 的 HTTPS 客户端默认验证证书合法性
- subconverter 可以用 `scv=true` 参数跳过验证，但 Worker 无法透传此能力

**解决方案**：

**方案 A：subconverter 直接拉取（推荐）**
- 让 subconverter 而非 Worker 拉取该订阅
- 在转换 URL 中包含原始订阅：
  ```
  https://subconverter-latest-xxxx.onrender.com/sub?
    target=clash&
    url=https://YOUR_WCLOUD_IP:9090/api/sub?token=xxx&
    scv=true  # 跳过证书验证
  ```
- subconverter 会自动处理自签证书

**方案 B：从本地网络拉取（同问题 4）**
- 在可信网络环境拉取，转换后写入 KV

**方案 C：等待机场升级**
- 机场改用合法 CA 签发的证书
- 或提供 HTTP 镜像地址

**验证 subconverter 是否支持 scv 参数**：
```bash
# 调用 subconverter，尝试 scv=true
curl "https://subconverter-latest-xxxx.onrender.com/sub?target=clash&url=https://47.102...&scv=true"

# 如果成功返回 Clash 格式，说明该订阅可用
```

---

### 问题 6：Shadowsocks 2022-blake3 加密不兼容

**现象**：
- oics.net 的 62 个节点无法通过 subconverter 转换
- 错误：`Unsupported encrypt method` 或配置为空

**原因**：
- oics.net 使用 Shadowsocks 2022-blake3-aes-256-gcm，这是最新加密标准
- subconverter v0.9.0 未更新到最新 Shadowsocks 2022 标准
- 转换引擎无法识别 `2022-blake3-aes-256-gcm` 参数

**解决方案**：

**方案 A：升级 subconverter（推荐）**
- 检查 subconverter 最新版本，是否已支持 2022-blake3
- 更新 Render 部署的镜像：
  ```bash
  # Render 会自动从 docker.io 拉取最新镜像
  # 如已拉取旧镜像，清除缓存并重新部署
  ```
- 验证新版本是否支持：
  ```bash
  curl "https://subconverter-latest-xxxx.onrender.com/version"
  ```

**方案 B：利用 oics.net/dler.io 原生格式转换（推荐）**
- **关键发现**：oics.net 本身就是 dler.io，支持多种格式 API
- 不需要 subconverter 参与转换
- 直接调用 dler.io 的格式 API：
  ```
  https://oics.net/api/v3/download.getFile/{id}?surge=smart
  https://oics.net/api/v3/download.getFile/{id}?clash=smart
  ```
- CF Worker 根据 UA 选择格式参数，直接请求 dler.io，返回客户端
- **改进方案**：
  ```javascript
  // CF Worker 代码示例
  async function handleRequest(request) {
    const ua = request.headers.get('user-agent') || '';
    let format = 'clash'; // 默认
    if (ua.includes('Surge')) format = 'surge';
    if (ua.includes('Quantumult')) format = 'quanx'; // 如果 dler.io 支持

    const url = `https://oics.net/api/v3/download.getFile/YOUR_OICS_FILE_ID?${format}=smart`;
    return fetch(url);
  }
  ```

**方案 C：手动导入并转换**
- 本地拉取订阅，手动提取节点信息
- 写入 KV

**优先级**：
1. 方案 B（dler.io 原生 API）—— 完全规避兼容性，最稳定
2. 方案 A（升级 subconverter）—— 一劳永逸，但依赖上游更新
3. 方案 C（手动导入）—— 保险但不可扩展

---

### 问题 7：Render 免费实例休眠导致转换失败

**现象**：
- 长时间不使用 subconverter 后，首次请求返回超时或 502
- Surge/Clash 等客户端收到原始 base64 而非转换后的格式
- 等待 30-50 秒后重试成功

**原因**：
- Render 免费实例闲置 15 分钟会进入休眠（sleep）状态
- 首次请求需要唤醒容器，冷启动需要 30-50 秒
- CF Worker 默认超时 30 秒，可能在容器启动前超时
- Worker 设置了降级逻辑：转换超时后回退到 base64

**解决方案**：

**方案 A：接受冷启动延迟（推荐）**
- 用户订阅更新通常是后台行为，不需要立即响应
- 第二次或后续请求会很快（容器已启动）
- 告知用户首次更新可能稍慢
- **优点**：零成本，无运维负担
- **缺点**：偶尔体验不佳

**方案 B：定时 Cron 保活（有风险）**
- 使用 CF Cron Trigger，每 14 分钟 ping 一次 subconverter
- 保持容器始终运行
- **配置示例**（`wrangler.toml`）：
  ```toml
  [[triggers.crons]]
  crons = ["*/14 * * * *"]  # 每 14 分钟
  ```
- Worker 代码：
  ```javascript
  export default {
    async scheduled(event, env, ctx) {
      await fetch('https://subconverter-latest-xxxx.onrender.com/version');
    }
  };
  ```
- **风险**：可能被视为滥用免费额度，账户被限制或禁用
- **费用影响**：Cron 触发也计入 CF Worker 10 万次/天的配额

**方案 C：付费 Render 实例**
- 升级到付费实例（从 $5+/月 起）
- 容器始终运行，无冷启动延迟
- 不符合本项目"完全免费"的初衷

**方案 D：迁移到其他后端**
- Heroku（已停止免费方案，不可行）
- Railway.app（$5/月起，或免费但需绑卡）
- 自建 VPS（$5+/月）

**当前推荐**：
- 短期：方案 A，接受冷启动
- 如要求响应速度，方案 B 但谨慎使用
- 长期：寻找其他免费 Docker 托管方案，或升级到付费

---

## 订阅自动更新方案

Worker 无法直接拉取部分订阅源（自签证书、403 封锁、加密方式不兼容），需要借助外部脚本定时拉取并写入 KV。

### 方案对比

| | 方案 A：大陆机器做中继 API | 方案 B：大陆机器推送 KV（推荐） | 方案 C：本地 Mac 脚本 |
|---|---|---|---|
| **原理** | 大陆机器跑 HTTP 服务，Worker 从中继拉取 | 大陆机器定时拉取订阅，通过 CF API 写入 KV | Mac 本地 cron 定时拉取并写入 KV |
| **Worker 改动** | 不用改 | 不用改 | 不用改 |
| **实时性** | 实时（每次请求都拉） | 定时（每 N 小时） | 定时（每 N 小时） |
| **要求** | 需开放 HTTP 端口，长期运行 | 只需 cron 定时任务，无需开端口 | Mac 需常开 |
| **安全性** | 中继 API 暴露在公网 | 无需暴露端口 | 最安全 |
| **可靠性** | 高（服务器 7x24） | 高（服务器 7x24） | 低（Mac 关机就停） |
| **适合场景** | 需要实时性 | 最推荐，简单可靠 | 没有服务器时的备选 |

### 推荐方案：方案 B — 大陆机器定时推送 KV

```
大陆机器（阿里云 47.102 等）
    │
    │ crontab 每 6 小时执行
    │
    ├─ curl -sk 拉取订阅 2（47.102，跳过自签证书）
    ├─ curl -s  拉取订阅 3（Flower_SS，大陆直连不会 403）
    ├─ curl -s  拉取订阅 4（oics.net，解码 base64）
    │
    ├─ 解码 + 格式转换（YAML → ss:// URI）
    ├─ 合并所有节点
    │
    └─ CF API PUT → KV LINK.txt
         │
         └─ Worker 下次请求时直接读取更新后的 KV ✅
```

### 脚本示例（方案 B）

```bash
#!/bin/bash
# sync-subscriptions.sh
# 部署在大陆机器上，crontab: 0 */6 * * * /path/to/sync-subscriptions.sh

CF_ACCOUNT_ID="YOUR_CF_ACCOUNT_ID"
CF_NAMESPACE_ID="YOUR_KV_NAMESPACE_ID"
CF_API_TOKEN="your-cf-token"  # 需要 Workers KV Storage Edit 权限

# --- 订阅 1：sntp.me（Worker 能自动拉取，但也可统一管理）---
SUB1="https://sub1.sntp.me/baseStr?token=xxx"

# --- 订阅 2：47.102 机场（-k 跳过自签证书）---
SUB2_RAW=$(curl -sk "https://YOUR_WCLOUD_IP:9090/api/sub?token=xxx")
SUB2_NODES=$(echo "$SUB2_RAW" | base64 -d | sed 's/\r//g' | grep "^ss://")

# --- 订阅 3：Flower_SS（大陆直连，不会 403）---
# 需要 Clash YAML → ss:// URI 转换，用 Python 脚本处理
SUB3_NODES=$(python3 /path/to/convert_clash_yaml.py "$FLOWER_SS_URL")

# --- 订阅 4：oics.net（base64 解码，过滤非空行）---
SUB4_RAW=$(curl -s "https://oics.net/api/v3/download.getFile/xxx?mu=smart&lv=2%7C3%7C5%7C6.ppt")
SUB4_NODES=$(echo "$SUB4_RAW" | base64 -d | sed 's/\r//g' | grep "^ss://")

# --- 合并所有节点 + 订阅 1 的 URL ---
KV_CONTENT="${SUB1}
${SUB2_NODES}
${SUB3_NODES}
${SUB4_NODES}"

# --- 写入 CF KV ---
curl -s -X PUT \
  "https://api.cloudflare.com/client/v4/accounts/${CF_ACCOUNT_ID}/storage/kv/namespaces/${CF_NAMESPACE_ID}/values/LINK.txt" \
  -H "Authorization: Bearer ${CF_API_TOKEN}" \
  -H "Content-Type: text/plain" \
  --data "$KV_CONTENT"

echo "[$(date)] Synced: $(echo "$KV_CONTENT" | grep -c "://") nodes"
```

### 部署定时任务

```bash
# 在大陆机器上
chmod +x /path/to/sync-subscriptions.sh
crontab -e
# 添加：
# 0 */6 * * * /path/to/sync-subscriptions.sh >> /var/log/sub-sync.log 2>&1
```

### 注意事项

- CF API Token 只需 `Workers KV Storage: Edit` 权限，最小化授权
- 脚本中的订阅 token 是敏感信息，注意文件权限 `chmod 600`
- 建议脚本执行后检查节点数量，异常时发告警（如节点数突然为 0）
- 订阅 1（sntp.me）既可以放在 KV 里让脚本管理，也可以保持 URL 让 Worker 自动拉取

---

## 规则仓库

所有分流规则统一维护在 [jandyx/Rules](https://github.com/jandyx/Rules)（fork 自 dler-io/Rules）。

### 仓库结构

```
jandyx/Rules
├── config.ini                     ← subconverter 外部配置
├── Clash/Provider/                ← Clash 格式规则（YAML）
│   ├── AI Suite.yaml              ← dler 原有 + SNTP 的 Claude/Cursor/Copilot/Gemini/Perplexity
│   ├── Social.yaml                ← 合并：Telegram + Discord + Twitter
│   ├── Gaming.yaml                ← 合并：Steam + miHoYo + Xbox
│   ├── Emby.yaml                  ← SNTP 独有
│   ├── Xbox.yaml                  ← SNTP 独有
│   ├── Media/
│   │   ├── CN Mainland TV.yaml    ← 合并：Bilibili/优酷/爱奇艺/腾讯视频
│   │   ├── Asian TV.yaml          ← 合并：巴哈姆特/ViuTV/KKTV 等
│   │   ├── Global TV.yaml         ← 合并：Max/Hulu/Twitch/Amazon/BBC 等
│   │   ├── Netflix.yaml           ← 独立保留
│   │   ├── YouTube.yaml           ← 独立保留
│   │   ├── Disney Plus.yaml       ← 独立保留
│   │   └── Spotify.yaml           ← 独立保留
│   └── ...                        ← 其他 dler 原有规则
├── Surge/Provider/                ← Surge 格式规则（.list，每行一条）
│   ├── AI Suite.list
│   ├── Social.list
│   └── ...                        ← 与 Clash/Provider 一一对应
├── Surge/Module/
│   └── Panel.sgmodule             ← 解锁检测 + IP 风险 Panel
└── Surge/Script/
    ├── claude_detect.js           ← Claude 可用性检测
    ├── cursor_check.js            ← Cursor 可用性检测
    └── ip_risk_check.js           ← IP 风险检测（ipapi.is）
```

### 规则格式

规则同时提供两种格式：

| 格式 | 目录 | 用途 |
|------|------|------|
| Clash Provider YAML | `Clash/Provider/` | Clash/Stash 客户端直接引用 |
| Surge .list | `Surge/Provider/` | Surge RULE-SET 引用（subconverter config.ini 也用这个） |

注意：subconverter 的 config.ini 里 ruleset 指向的是 Surge `.list` 格式，因为 Surge 的 RULE-SET 不能直接读取 Clash YAML 格式。其他客户端（Clash/QX/Loon）由 subconverter 自动转换，不受影响。

### 策略组布局

每个策略组内节点平铺显示，顶部有 Proxy/Direct 快捷选项。

```
🚀 Proxy          → Auto-Smart / Auto-UrlTest / Direct + 全部节点
🏠 Domestic        → Direct
🛑 AdBlock         → Reject
🚫 HTTPDNS         → Direct
🤖 AI Suite        → ChatGPT/Claude/Cursor/Copilot/Gemini/Perplexity
📺 SNTP Emby       → Emby 影院
📺 Netflix         → Netflix
📺 Disney Plus     → Disney+
📺 YouTube         → YouTube
📺 CN Mainland TV  → Bilibili/优酷/爱奇艺（默认 Direct）
📺 Asian TV        → 巴哈姆特/ViuTV 等
📺 Global TV       → Max/Hulu/Twitch/BBC 等
🎵 Spotify         → Spotify
🍎 Apple           → iCloud/App Store（默认 Direct）
🍎 Apple TV        → Apple TV+ 流媒体
📲 Telegram        → Telegram
📢 Google FCM      → Google 推送
💬 Social          → Twitter/Discord
📱 TikTok          → TikTok
💰 Crypto          → 加密货币
💳 PayPal          → PayPal（默认 Direct）
Ⓜ️ Microsoft       → 微软服务
📖 Scholar         → 学术（默认 Direct）
🎮 Steam           → Steam
🎮 Xbox            → Xbox（默认 Direct）
🎮 miHoYo          → 米哈游（默认 Direct）
🚄 Speedtest       → 测速
⚡ Auto - UrlTest   → 全节点自动测速
🔄 Auto - Smart    → 全节点智能回退
🐟 Final           → 兜底
```

### 节点重命名

通过 subconverter rename 简化 Flower 节点名称：
- `实验性/标准/高级 IEPL 专线` → `IEPL [Flower]`
- SNTP 节点保持英文名（HK01/SG01 等）
- WCloud 节点保持中文名（香港01/日本01 等）

### CDN 加速

所有规则文件通过 jsdelivr CDN 引用，解决国内无法访问 raw.githubusercontent.com 的问题：
```
https://cdn.jsdelivr.net/gh/jandyx/Rules@main/Surge/Provider/AI%20Suite.list
```

缓存清除命令：
```
https://purge.jsdelivr.net/gh/jandyx/Rules@main/path/to/file
```

---

## Surge Panel

通过 Surge Module 提供流媒体解锁检测、AI 可用性检测和 IP 风险检测面板。

### 安装

Surge → Modules → Install from URL：
```
https://cdn.jsdelivr.net/gh/jandyx/Rules@main/Surge/Module/Panel.sgmodule
```

### 检测项

| Panel | 检测方式 | 图标 |
|-------|---------|------|
| Netflix | 请求 Netflix API，检测解锁状态和地区 | ✅ 绿色 / ❌ 红色 |
| Disney Plus | 请求 Disney API，检测解锁状态和地区 | ✅ 绿色 / ❌ 红色 |
| YouTube Premium | 请求 YouTube API，检测支持状态和地区 | ✅ 绿色 / ❌ 红色 |
| ChatGPT | 通过 `chat.openai.com/cdn-cgi/trace` 获取地区，对比支持列表 | 蓝色 |
| Claude | 通过 `claude.ai/cdn-cgi/trace` 获取地区，对比支持列表 | 橙色（brain.head.profile） |
| Cursor | 通过 `cloudflare.com/cdn-cgi/trace` 获取地区，对比封锁列表 | 蓝色（cursorarrow.click.2） |
| IP Risk | 通过 `api.ipapi.is` 获取 IP 风险信息（机房/代理/住宅/滥用评分） | 绿🛡️/橙🛡️/红🛡️ |

### 检测原理

每个 Panel 脚本的请求会经过 Surge 的分流规则，走对应策略组选中的节点。例如：
- Claude Panel 请求 `claude.ai` → 匹配 AI Suite 规则 → 走 AI Suite 策略组的节点
- IP Risk 请求 `api.ipapi.is` → 匹配 Final 规则 → 走 Proxy 节点

### 脚本来源

| 脚本 | 来源 |
|------|------|
| Netflix/Disney+/YouTube/ChatGPT | [dler-io/Rules](https://github.com/dler-io/Rules) 社区脚本 |
| Claude/Cursor/IP Risk | 自写，维护在 `jandyx/Rules/Surge/Script/` |

---

## 待办事项

### 已完成 ✅

- [x] Render 部署 subconverter 后端
- [x] CF Worker 部署 CF-Workers-SUB 前端
- [x] 3 条订阅源聚合（SNTP 自动拉取 + WCloud 手动 + Flower 手动）
- [x] 全客户端支持（Surge/Clash/QX/Loon/Shadowrocket/SingBox）
- [x] 自定义分流规则（dler + SNTP 合并，jandyx/Rules 仓库）
- [x] 策略组设计（AI/流媒体/社交/游戏等 27 个组）
- [x] 节点平铺展示（每个策略组内所有节点可选）
- [x] Surge Panel 检测（Netflix/Disney+/YouTube/ChatGPT/Claude/Cursor/IP Risk）
- [x] jsdelivr CDN 加速（解决国内下载问题）
- [x] Surge .list 格式规则（解决 RULE-SET 兼容性问题）
- [x] 节点重命名（Flower IEPL 简化）
- [x] 完整备忘文档

### TODO

- [ ] **大陆机器同步脚本**
  - 定时拉取 WCloud/Flower/oics.net 节点并写入 KV
  - 在脚本中给节点名加机场标签（[SNTP] [Flower] [WCloud]）
  - 处理 Flower 403 问题（通过代理拉取）
  - 部署位置：大陆机器（方案 B：推送 KV）
- [ ] **TG Bot token 分发管理**
  - 一人一 token，群成员校验
  - 管理员审批/吊销
  - 退群自动吊销
- [ ] **域名绑定**
  - 获取免费域名绑定到 Worker
  - 隐藏 workers.dev 地址
- [ ] **Render 保活评估**
  - 评估 CF Cron Trigger 保活是否有滥用风险
- [ ] **dler-io 规则同步**
  - 定期从上游 dler-io/Rules 同步更新到 jandyx/Rules
  - 保留自定义合并的规则不被覆盖
- [ ] **CF API Token 安全**
  - 部署完成后吊销当前 Token
  - 需要时创建最小权限 Token

## 常见问题

### Q: 为什么我的订阅导入后显示 0 个节点？

**A: 可能原因**：

1. **KV 数据为空**
   ```bash
   # 检查 KV 内容
   wrangler kv key get --namespace-id=xxx "LINK.txt"
   ```
   - 如果输出为空，说明还未写入数据
   - 解决：按 [第三步](#第三步写入订阅数据到-kv) 写入订阅数据

2. **subconverter 后端异常**
   - 访问 `https://subconverter-latest-xxxx.onrender.com` 检查是否在线
   - 如果返回 502 或 503，说明 Render 实例崩溃或休眠
   - 解决：等待几分钟让实例启动；或检查 Render 日志

3. **订阅格式不兼容**
   - 确保 KV 中的数据是有效的 ss://、ssr://、trojan:// 等格式
   - 检查是否被错误地 URL encode 或损坏

4. **Worker 代码有 bug**
   - 访问 `https://your-worker.workers.dev/auto?debug=true`（如支持）
   - 查看 CF Worker 的实时日志：
     ```bash
     wrangler tail
     ```

### Q: 为什么某些客户端收到的是 base64 而非格式化配置？

**A: 可能原因**：

1. **subconverter 转换超时**
   - Render 实例冷启动，响应时间过长
   - Worker 在 30 秒内未收到响应，降级为 base64
   - 解决：重新更新订阅，或等待 Render 实例启动

2. **subconverter 不支持该加密方式**
   - 例如 SS 2022-blake3（见 [问题 6](#问题-6shadowsocks-2022blake3-加密不兼容)）
   - 解决：使用 dler.io 原生格式 API，或升级 subconverter

3. **UA 识别失败**
   - 客户端 User-Agent 过于奇特，Worker 无法识别
   - 解决：明确指定格式参数，如 `/?clash` 或 `/?surge`

### Q: 如何手动更新某个订阅源而不更新其他源？

**A: 方法**：

```bash
# 1. 拉取单个订阅源
curl -k "https://YOUR_WCLOUD_IP:9090/api/sub?token=xxx" > sub2.txt

# 2. 解码并转换为 ss:// 格式（如需要）
cat sub2.txt | base64 -d > sub2_decoded.txt
# 手动或脚本转换...

# 3. 合并所有源（sntp.me + 47.102 + Flower_SS + oics.net）
cat sntp.txt sub2.txt flower.txt oics.txt > combined.txt

# 4. 写入 KV
wrangler kv key put --namespace-id=xxx "LINK.txt" "$(cat combined.txt)"

# 5. 验证
wrangler kv key get --namespace-id=xxx "LINK.txt" | head -20
```

### Q: 如何在多个客户端使用同一订阅链接？

**A: 使用自适应链接**：

```
https://your-worker.workers.dev/auto
```

- Worker 通过 User-Agent 自动识别客户端类型
- Surge 访问时返回 Surge 格式
- Clash 访问时返回 Clash 格式
- 依此类推

**如果自动识别不生效，显式指定**：

```
https://your-worker.workers.dev/auto?surge
https://your-worker.workers.dev/auto?clash
https://your-worker.workers.dev/auto?quanx
```

### Q: 如何排查 Worker 故障？

**A: 调试步骤**：

1. **查看实时日志**
   ```bash
   wrangler tail --format pretty
   ```

2. **测试 Worker 响应**
   ```bash
   curl -v "https://your-worker.workers.dev/auto" \
     -H "User-Agent: Surge"
   ```

3. **检查环境变量和 KV**
   ```bash
   # 查看已配置的 secrets（不显示值，仅确认存在）
   wrangler secret list

   # 检查 KV 内容
   wrangler kv key list --namespace-id=xxx
   wrangler kv key get --namespace-id=xxx "LINK.txt"
   ```

4. **检查 subconverter 后端**
   ```bash
   curl "https://subconverter-latest-xxxx.onrender.com/version"
   curl "https://subconverter-latest-xxxx.onrender.com/sub?target=clash&url=..."
   ```

5. **重新部署**
   ```bash
   wrangler deploy --env production
   ```

### Q: 订阅链接可以分享给别人吗？

**A: 可以，但有注意事项**：

1. **安全性**
   - 分享前确认 TOKEN 是否敏感（某些 Worker 实现可能包含令牌）
   - 如果 TOKEN 泄露，机场可追踪到你的身份
   - 建议为每个使用者创建不同的 TOKEN，便于管理和撤销

2. **流量和限制**
   - Cloudflare Worker 免费额度为 10 万次请求/天
   - 如果分享人数过多，可能触及配额限制
   - 建议监控使用情况，超限后升级到付费方案

3. **使用链接**
   ```
   # 分享自适应链接
   https://your-worker.workers.dev/auto

   # 或明确指定格式
   https://your-worker.workers.dev/auto?surge
   ```

4. **收回权限**
   - 如要禁用某个使用者，修改 Worker 代码添加黑名单逻辑
   - 或删除 KV 中的订阅数据，所有订阅都会失效

### Q: 如何自定义分流规则（例如让某些域名走代理，某些不走）？

**A: 使用 config.ini 配置**：

1. **Fork ACL4SSR 仓库**
   ```bash
   git clone https://github.com/ACL4SSR/ACL4SSR.git
   cd ACL4SSR
   # 编辑 Clash/config/ACL4SSR_Online_CustomizedChina.ini
   ```

2. **自定义规则**
   - 编辑 `[custom]` 等section 添加自定义规则
   - 例如：
     ```ini
     [custom]
     ;自定义规则
     DOMAIN-SUFFIX,example.com,PROXY  # example.com 走代理
     DOMAIN-SUFFIX,github.com,DIRECT  # github.com 不走代理
     ```

3. **上传到 GitHub**
   ```bash
   git add Clash/config/ACL4SSR_Online_CustomizedChina.ini
   git commit -m "Custom rules"
   git push origin main
   ```

4. **更新 Worker 的 SUBCONFIG**
   ```bash
   wrangler secret put SUBCONFIG
   # 输入：https://raw.githubusercontent.com/your-username/ACL4SSR/main/Clash/config/ACL4SSR_Online_CustomizedChina.ini
   ```

5. **重新部署**
   ```bash
   wrangler deploy
   ```

### Q: 我可以在多个地方部署相同的 Worker 吗？

**A: 可以，方法**：

1. **相同的 Cloudflare 账号下**
   - 创建多个 Worker 脚本
   - 复用相同代码，不同环境变量
   - 好处：方便管理，环境隔离

2. **不同 Cloudflare 账号**
   - 各自独立部署
   - 不共享 KV 数据
   - 需分别配置环境变量

3. **建议**
   - 一个主 Worker + 多个备用 Worker
   - 主 Worker 提供完整功能
   - 备用 Worker 作为故障转移（failover）
   - 在客户端配置多条订阅链接

---

## 参考资源

### 官方文档

- [Cloudflare Workers 文档](https://developers.cloudflare.com/workers/)
- [subconverter GitHub](https://github.com/tindy2013/subconverter)
- [CF-Workers-SUB GitHub](https://github.com/cmliu/CF-Workers-SUB)
- [dler.io 文档](https://docs.dler.io/)

### 相关项目

- [ACL4SSR](https://github.com/ACL4SSR/ACL4SSR) - 分流规则
- [Rules](https://github.com/dler-io/Rules) - dler.io 规则
- [Render 文档](https://render.com/docs)

### 客户端官方链接

- [Surge](https://nssurge.com/)
- [Clash for Windows](https://github.com/Fndroid/clash_for_windows_pkg)
- [Quantumult X](https://quantumultx.com/)
- [Loon](https://apps.apple.com/us/app/loon/id1373567447)
- [Shadowrocket](https://apps.apple.com/us/app/shadowrocket/id932747118)
- [SingBox](https://github.com/SagerNet/sing-box)

---

## 许可证

本项目为个人使用文档。

## 更新日志

### v1.0.0 (2026-03-15)

- 初始化项目文档
- 完整记录架构设计、部署步骤、已知问题及解决方案
- 列出待办事项优先级

---

**最后更新**：2026-03-15
**维护者**：jandyx
