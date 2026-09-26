# 🍪 Cookie 罐演算 Web 工具 · Cookie Jar Lab

一个用于**演算 HTTP Cookie 写入与发送规则**的教学/调试 Web 工具：你按顺序提交
「响应 URL + 单条 Set-Cookie」（带 `Partitioned` 时再附顶层页面 URL），服务端按
[RFC 6265](https://www.rfc-editor.org/rfc/rfc6265) 更新内存 Cookie 罐；再输入请求 URL
（以及顶层页面 URL、方法、是否顶层导航），即可看到生成的 `Cookie` 请求头，以及每一项
**被发送或被排除的具体原因**（含 SameSite 站点演算与 **Partitioned（CHIPS）分区演算**）。

此外支持 **Cookie Priority 与容量压力模拟**：为每个可注册域与全局设置条目数上限，
观察入罐时「先域级、后全局」的淘汰过程——先清过期项，再按
**优先级低 → 高、最近访问早 → 晚、创建序号小 → 大**逐条淘汰，并给出完整淘汰清单与排序依据
（普通项与各分区项都计入域级 / 全局容量）。

技术栈：**TypeScript + Express + 原生 HTML/CSS/JS**（前端无框架、无构建），数据仅存内存；
same-site / 分区键判定使用 [tldts](https://github.com/remusao/tldts)（内置公共后缀列表 PSL，含私有域段）。

---

## 🚀 单条启动命令

需要安装 Docker（含 Compose 插件）。在项目根目录执行：

```bash
docker compose up --build -d
```

启动后浏览器访问（**Web 页面与全部 `/api/*` 接口同在该地址 / 端口**）：

> **http://localhost:3000**

- Web 界面：`http://localhost:3000/`
- API 根：例如 `http://localhost:3000/api/jar`（见下方 [HTTP API](#-http-api)）
- 容器内服务端口：**3000**
- 宿主机默认映射端口：**3000**（想换端口改 `docker-compose.yml` 左侧，如 `"3001:3000"`）

停止与清理：

```bash
docker compose down          # 停止并删除容器（镜像保留）
docker compose down --rmi local   # 同时删除本工具镜像
```

> 不想用 Docker？见文末[不用 Docker 的本地运行](#不用-docker-的本地运行)。

---

## 🧭 使用流程

界面分为三步：

1. **写入 Set-Cookie**（可重复执行，模拟多个响应、按顺序入罐）
   - 填「响应 URL」（cookie 来源页面），如 `https://shop.example.com/admin/dashboard?tab=1`
   - 仅当 Set-Cookie 带 `Partitioned` 时，必须再填「**顶层页面 URL**」（嵌入该第三方内容的
     顶层站点，如 `https://portal.example/`），用于计算分区键；普通 cookie 留空即可
   - 填「单条 Set-Cookie」（**不要**带 `Set-Cookie:` 前缀），如：
     `session=abc123; Domain=example.com; Path=/; Max-Age=3600; Secure`
   - 点击 **入罐**，查看入罐 / 更新 / 删除 / 拒绝结果与原因
2. **发起请求**：填请求 URL（如 `https://api.shop.example.com/orders/123/detail`），
   并可指定 SameSite / Partitioned 判定上下文——**顶层页面 URL**（留空 = 与请求 URL
   同站、同分区）、**请求方法**、**是否顶层导航**；点击 **生成 Cookie 头**，查看：
   - 最终 `Cookie:` 请求头（可一键复制）
   - 请求站 vs 顶层站（scheme + 可注册域）的同站/跨站结论，以及**当前分区键**
   - 📤 **发送项**：命中的主机 / Path / Secure / SameSite / 分区键匹配点
   - 🚫 **排除项**：过期、Domain 不匹配、Path 不匹配、Secure 受限、**分区键不一致**、
     SameSite 拦截等逐项原因
3. **🫙 当前 Cookie 罐**：右侧表格实时展示全部条目（域、作用域、Path、Secure、
   Priority、**分区键（Partitioned）**、有效期剩余、创建序号、**最近访问时间**）；
   普通项与各分区项即使同名同域同路径也会**并存为独立行**；点击 **重置** 清空内存罐。
4. **🎚️ 容量压力模拟**（右栏顶部）：设置**全局容量**与**每个可注册域的容量**（按条目条数，
   留空 = 不限），点击 **应用容量设置** 立即按新上限执行一次淘汰；之后每次入罐通过校验都会
   自动依次执行域级、全局淘汰。面板中实时显示全局与各域用量（`当前条数/上限`），
   入罐结果区会展示**本次触发的淘汰清单**（过期清理 / 容量淘汰、逐条排序依据）。

### 容量淘汰的顺序

每次入罐（含覆盖更新）通过既有全部校验后，按以下顺序处理压力：

1. **域级淘汰**：仅针对新条目所属的**可注册域**（eTLD+1；IP / localhost 按完整主机），
   且仅当该域设置了容量上限；
2. **全局淘汰**：随后针对全罐，且仅当设置了全局限额。

每一级超限时（范围内条数 > 上限）：

1. **先物理清理范围内已过期的条目**；
2. 若仍超限，把存活条目依次按
   **Priority 从低到高（Low → Medium → High）→ 最近访问时间从早到晚 → 创建序号从小到大**
   排序，从最前开始逐条淘汰，直到条数回到上限；
3. 新入罐的条目自身也可能立刻被淘汰（结果中会明确提示）。

**最近访问时间（LRA）**在创建 / 覆盖更新时写入；条目在请求演算中**发送成功**即刷新。
修改容量设置时，会对所有已配置的上限（域级逐个、随后全局）立即执行一轮上述流程。

### 推荐演算示例（按顺序）

| # | 响应 URL | Set-Cookie | 预期 |
|---|----------|-----------|------|
| 1 | `https://shop.example.com/admin/dashboard` | `sid=host123; Path=/admin; Max-Age=600` | 入罐（host-only，默认 Path 这里被属性指定为 `/admin`） |
| 2 | `https://shop.example.com/admin/dashboard` | `tracker=xyz; Max-Age=3600` | 入罐（host-only，**默认 Path=/admin**，SameSite 记为默认 Lax） |
| 3 | `https://example.com/` | `prefs=a=1; Domain=example.com; Path=/` | 入罐（Domain cookie，覆盖 `example.com` 及所有子域） |
| 4 | `http://evil.example.org/` | `bad=nope; Domain=example.com` | **拒绝**：Domain 不匹配响应主机 |
| 5 | `https://shop.example.com/admin/dashboard` | `tracker=; Max-Age=0` | **删除**同名同域同路径的 tracker |
| 6 | `https://example.com/` | `secure=1; Path=/; Secure` | 入罐（仅 https 发送） |
| 7 | `https://example.com/` | `strict=1; Path=/; SameSite=STRICT` | 入罐（SameSite 大小写不敏感 → Strict） |
| 8 | `https://example.com/` | `xs=1; Path=/; SameSite=None` | **拒绝**：SameSite=None 未带 Secure |
| 9 | `https://example.com/` | `xs=1; Path=/; Secure; SameSite=None` | 入罐（跨站也发送，但仅 https） |
| 10 | `https://example.com/` | `weird=1; Path=/; SameSite=Banana` | 入罐（非法值 → **默认 Lax** 并提示） |
| 11 | `https://example.com/` | `cart=1; Path=/; Priority=Low` | 入罐（Priority=Low；容量超限时最先被淘汰） |
| 12 | `https://example.com/` | `vip=1; Path=/; Priority=Nope` | 入罐（Priority 非法 → **Medium** 并提示） |

容量压力演算（右栏 **🎚️ 容量压力模拟**，或 `POST /api/capacity`）：

- 先设置 `example.com` 域容量为 **2**，再连续入罐 3 条同 Path 的 cookie：
  第 3 条入罐后触发域级淘汰，**Low 先于 Medium 先于 High** 出局；
- 把其中一条的 `Max-Age=1` 设为已过期：下次超限时**先清过期项**，清完不超限就不再淘汰；
- 连续请求某条 cookie（同站 GET 命中它）刷新其最近访问时间，再制造超限：
  同优先级下**最久未访问**的先出局；
- 再设一个较小的**全局容量**（如 3），观察跨域入罐时「域级 → 全局」两级报告依次出现。

然后分别用以下请求 URL 观察差异：

- `https://api.shop.example.com/admin/orders/9` —— 域 + Path 匹配
- `http://shop.example.com/admin/dashboard` —— 明文 http，Secure 项被排除
- `https://shop.example.com/home` —— Path 不匹配的项被排除并给出原因

SameSite 演算示例（第 2 步的「顶层页面 URL / 方法 / 顶层导航」）：

- 请求 `https://example.com/`，顶层页留空 —— 同站，`strict`/`xs` 都发送
- 请求 `https://example.com/`，顶层页 `https://other.org/`，方法 GET，勾选顶层导航
  —— 跨站：`strict` 被排除；显式/默认 Lax 与 `xs`（None）放行
- 同上但方法 POST —— 显式 Lax 被排除；**默认 Lax**（如第 10 条的 `weird`）
  在创建 120 秒内仍放行（Lax+POST 宽限）
- 同上但取消勾选顶层导航 —— 模拟子资源请求，跨站时只剩 `xs`（None）发送

Partitioned（CHIPS）分区演算示例（第 1 步需填「顶层页面 URL」，或 `POST /api/store` 带 `topUrl`）：

- 响应 `https://ads.example.com/widget`，`setCookie="p=1; Path=/; Secure; SameSite=None; Partitioned"`，
  **顶层页 `https://portal.example/`** —— 入罐，分区键 `https://portal.example`
- 同一条 cookie 再用 **顶层页 `https://shop.example/`** 写一次 —— **另存为独立项**
  （同名同域同路径、分区键不同，罐中两行并存）
- 同上但**不带 `Secure`** —— **拒绝**：Partitioned 必须同时声明 Secure
- 带 Partitioned 但**不填顶层页**（或填 `not-a-url`）—— **拒绝**：无法确定分区键
- 请求 `https://ads.example.com/widget`，顶层页 `https://portal.example/` —— 命中
  `https://portal.example` 分区，该分区项发送（`shop.example` 分区项进入排除明细并说明键不一致）
- 同请求但顶层页换成 `https://other.org/` —— 两个分区项都因**分区键不一致**被排除
- 对 `https://portal.example` 分区的该项再发一次 `p=...; Partitioned; Secure; Max-Age=0`
  （顶层页仍为 portal）—— **只删除该分区项**，`shop.example` 分区项与普通同名项保留

---

## 📐 实现的演算规则

### 写入（处理 Set-Cookie）

- **Domain**
  - 不带 `Domain` 属性 → **host-only cookie**，只匹配原主机；
  - 带 `Domain` → 规范化（小写、去前导点），且必须是响应主机本身或其上级域，
    否则**拒绝入罐**；
  - 主机是 IP 地址时不做域/子域匹配（Domain 不一致即拒绝）。
- **Path**
  - 合法属性值必须以 `/` 开头，否则忽略属性；
  - **缺省（或非法）时取响应 URL 路径最后一个斜杠之前的目录**
   （`/admin/dashboard` → `/admin`；根路径 → `/`）。
- **Max-Age**
  - 正整数：相对有效期（秒），换算成绝对过期时间；
  - **`<= 0`（含 `0` 与负数）：删除同名 `(name, domain, path)` 项**，不存在则说明无可删除；
  - 非法值（非整数）：忽略并提示，按会话 cookie 处理。`Expires` 属性不参与本演算。
- **Secure**：记录该属性；在明文 `http` 响应中设置会给出提示，且之后绝不会经 http 发送。
- **SameSite**
  - 解析 `Strict` / `Lax` / `None`（**大小写不敏感**）；
  - **未声明或值非法 → 记为默认 Lax**（非法值会给出提示）；
  - **`SameSite=None` 未同时声明 `Secure` → 拒绝入罐**（现代浏览器的强制要求）。
- **Priority**（RFC 6265bis，仅参与容量淘汰，不影响发送）
  - 解析 `Low` / `Medium` / `High`（**大小写不敏感**）；
  - **缺省或非法值 → 按 Medium**（都会给出提示；缺省提示「按默认 Medium 参与容量淘汰」）。
- **Partitioned**（[CHIPS](https://developers.google.com/privacy-sandbook/3pcd/chips) 独立分区 cookie）
  - 带 `Partitioned` 的 Set-Cookie **必须同时带 `Secure`**，否则**拒绝入罐**；
  - 写入时**必须提供有效 http(s) 的顶层页面 URL**（`POST /api/store` 的 `topUrl` 字段、
    界面第 1 步的「顶层页面 URL」），缺失或非法同样**拒绝入罐**；
  - **分区键 = 顶层 URL 的 scheme + 可注册域（eTLD+1）**；IP、localhost 等无可注册域的
    主机用**完整主机**（与 site 完全同型，如 `https://portal.example`、`http://127.0.0.1`）；
  - 存储身份为 `(name, domain, path, 分区键)`：**普通项（未分区）与不同分区的同名同域同路径
    项并存**；覆盖更新与 `Max-Age<=0` 删除都**只作用于对应分区**，不影响其他分区或普通项。
- **同名同域同路径（且同分区）**再次写入 → 覆盖更新，**沿用首次创建序号**（保证排序稳定），
  并刷新该条目的最近访问时间。
- **容量压力淘汰**（入罐通过以上全部校验后执行；未设置任何上限时完全不干预）
  - 可分别为每个**可注册域**（eTLD+1，tldts/PSL 判定；IP、localhost 按完整主机分桶）
    与**全局**设置条目数上限；**普通项与所有分区项（Partitioned）都按条目计入域级 / 全局用量**；
  - 入罐后**先域级（仅新条目所属可注册域）后全局**；修改容量设置则对全部已配上限立即执行；
  - 范围内条数超限时：**先清理过期项**，再按
    **Priority 低 → 高、最近访问早 → 晚、创建序号小 → 大**淘汰至回到上限；
  - **发送成功**的条目刷新最近访问时间，作为同级 Priority 的淘汰依据。

### 发送（处理请求 URL + SameSite 上下文）

除请求 URL 外，演算还需要三个上下文参数（界面第 2 步）：

- **顶层页面 URL**：决定 same-site 的基准站，同时是 **Partitioned 分区 cookie 的当前分区键**
  来源；留空时视为与请求 URL 相同（同站、第一方分区）；
- **请求方法**：GET / POST / …（大小写不敏感）；
- **是否顶层导航**：地址栏/链接跳转属于顶层导航；子资源、iframe、fetch 不属于。

**站点（site）= scheme + 可注册域（eTLD+1）**，可注册域由公共后缀列表
（tldts 内置 PSL，含私有域段，如 `github.io`）判定；**IP 地址与 localhost
等无可注册域的主机按完整主机比较**（`localhost` ≠ `127.0.0.1`，端口不参与）。
请求站与顶层站一致即「同站」，否则「跨站」（`http://example.com` 与
`https://example.com` 因 scheme 不同也算跨站）。**分区 cookie 的当前分区键 = 顶层站**。

逐条检查（顺序与 RFC 6265 / CHIPS 一致）：

1. **过期检查**：已过期不发送（罐中保留并标灰显示）；
2. **Domain 匹配**：host-only 仅精确匹配原主机；Domain cookie 匹配该域及其子域；
3. **Path 匹配**：请求路径与 cookie-path 相等，或以其开头且边界字符是 `/`；
4. **Secure 检查**：Secure cookie 只在 `https:` 请求中发送（`Partitioned` 入罐时已强制 Secure）；
5. **Partitioned 分区检查**：分区 cookie 仅当**存储分区键 = 当前分区键（顶层站）**时发送，
   否则列入排除明细并给出两个分区键；普通（未分区）项不受此限制；
6. **SameSite 检查**（仅跨站时拦截）：
   - `Strict`：**仅同站发送**；
   - 显式 `Lax`：同站，或**跨站顶层导航的安全方法**（GET/HEAD/OPTIONS/TRACE）；
   - **默认 Lax**（未声明/非法值）：同上，另允许**创建不超过 120 秒的跨站顶层 POST**
     （对齐 Chrome 的 Lax+POST 缓解，按首次创建时间计算）；
   - `None`：跨站照常发送（入罐时已强制 Secure）。

通过检查的条目按：

> **Path 长度降序 → 首次创建顺序（#序号）升序**

排序后以 `; ` 拼接成 `Cookie` 请求头（RFC 6265 的 cookie 排序要求）。

---

## 🔌 HTTP API

| 方法 | 路径 | Body（JSON） | 说明 |
|------|------|--------------|------|
| GET | `/api/jar` | — | 当前罐快照（`cookies` + `capacity`：容量设置与各可注册域用量；每条含 `partitioned`/`partitionKey`） |
| POST | `/api/store` | `{ "url": "...", "setCookie": "...", "topUrl"?: "..." }` | 顺序写入一条 Set-Cookie；`topUrl` 为顶层页面 URL，带 `Partitioned` 时**必需**（据此算分区键，缺失/非法会拒绝）。响应含 `eviction`（本次触发的域级/全局淘汰清单与排序依据，无限额时为 `null`） |
| POST | `/api/send` | `{ "url": "...", "topUrl"?: "...", "method"?: "GET", "topLevelNav"?: true }` | 演算 Cookie 头与逐项原因；`topUrl` 同时决定 SameSite 上下文与分区 cookie 的当前分区键（响应字段 `partitionKey`）；发送成功的条目刷新最近访问时间，响应附最新 `cookies` 与 `capacity` |
| POST | `/api/capacity` | `{ "globalLimit"?: number \| null, "domainLimits"?: { "example.com": 3 } }` | 替换式设置全局 / 每可注册域容量（正整数；`null`/缺省 = 不限；普通项与分区项一并计数），立即执行一轮淘汰并返回报告 |
| POST | `/api/reset` | — | 清空内存罐（**容量设置保留**） |

curl 示例：

```bash
# 写入（SameSite 大小写不敏感；SameSite=None 必须带 Secure）
curl -s -X POST http://localhost:3000/api/store \
  -H 'Content-Type: application/json' \
  -d '{"url":"https://shop.example.com/admin/dashboard",
       "setCookie":"sid=abc; Domain=example.com; Path=/; Max-Age=3600; Secure; SameSite=Lax"}'

# 写入 Partitioned（CHIPS）分区 cookie：必须带 Secure，并提供顶层页面 URL 作为分区键来源
curl -s -X POST http://localhost:3000/api/store \
  -H 'Content-Type: application/json' \
  -d '{"url":"https://ads.example.com/widget",
       "topUrl":"https://portal.example/",
       "setCookie":"sid=chips; Path=/; Max-Age=3600; Secure; SameSite=None; Partitioned"}'
# -> partitionKey = "https://portal.example"；同一 cookie 换一个 topUrl 会在另一分区独立存放

# 演算请求（顶层站为 portal.example：命中该分区；换成 other.org 则分区项进入排除明细）
curl -s -X POST http://localhost:3000/api/send \
  -H 'Content-Type: application/json' \
  -d '{"url":"https://ads.example.com/widget",
       "topUrl":"https://portal.example/", "method":"GET", "topLevelNav":false}'

# 跨站顶层导航 GET：Strict 被拦截，Lax/None 放行（分区项还需分区键一致）
curl -s -X POST http://localhost:3000/api/send \
  -H 'Content-Type: application/json' \
  -d '{"url":"https://api.shop.example.com/orders/123",
       "topUrl":"https://other.org/", "method":"GET", "topLevelNav":true}'

# 设置容量：example.com 域 3 条、全局 10 条（立即按新上限执行一次淘汰）
curl -s -X POST http://localhost:3000/api/capacity \
  -H 'Content-Type: application/json' \
  -d '{"globalLimit":10,"domainLimits":{"example.com":3}}'

# 取消全部限制
curl -s -X POST http://localhost:3000/api/capacity \
  -H 'Content-Type: application/json' -d '{"globalLimit":null,"domainLimits":{}}'

# 重置（罐清空，容量设置保留）
curl -s -X POST http://localhost:3000/api/reset
```

---

## 📁 项目结构

```
.
├── Dockerfile              # 多阶段构建：tsc 编译 → 精简运行镜像
├── docker-compose.yml      # 单条命令启动（3000:3000）
├── package.json
├── tsconfig.json
├── src
│   ├── server.ts           # Express 入口与 API（含容量设置接口）
│   └── cookieJar.ts        # 全部 Cookie 演算规则（解析/入罐/匹配/排序/SameSite/Priority/Partitioned 分区/容量淘汰）
└── public
    ├── index.html          # 原生前端页面（含容量压力配置面板）
    ├── styles.css
    └── app.js
```

## 不用 Docker 的本地运行

需要 Node.js 18+（推荐 20）：

```bash
npm install
npm run build      # tsc 编译到 dist/
npm start          # 打开 http://localhost:3000
```

开发模式（免编译，自动 ts-node 直跑）：

```bash
npm run dev
```

> 说明：本工具是**演算/教学工具**，Cookie 仅存于服务端进程内存，刻意聚焦
> Domain / Path / Max-Age / Secure / SameSite / Priority / **Partitioned（CHIPS）分区**
> 与容量压力淘汰等规则；
> 不处理 HttpOnly 发送限制（仅展示标签）、Expires、单条 cookie 字节大小上限等。
> 容量按**条目条数**模拟（RFC 6265bis 的每可注册域 180 条、全局 3000 条同型机制），
> 时间精度取毫秒级，同一毫秒内的次序由创建序号兜底。
