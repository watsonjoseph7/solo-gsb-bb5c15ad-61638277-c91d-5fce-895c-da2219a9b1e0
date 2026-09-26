# 🍪 Cookie 罐演算 Web 工具 · Cookie Jar Lab

一个用于**演算 HTTP Cookie 写入与发送规则**的教学/调试 Web 工具：你按顺序提交
「响应 URL + 单条 Set-Cookie」，服务端按 [RFC 6265](https://www.rfc-editor.org/rfc/rfc6265)
更新内存 Cookie 罐；再输入请求 URL（以及顶层页面 URL、方法、是否顶层导航），即可看到生成的
`Cookie` 请求头，以及每一项**被发送或被排除的具体原因**（含 SameSite 站点演算）。

此外支持 **Cookie Priority 与容量压力模拟**：为每个可注册域与全局设置条目数上限，
观察入罐时「先域级、后全局」的淘汰过程——先清过期项，再按
**优先级低 → 高、最近访问早 → 晚、创建序号小 → 大**逐条淘汰，并给出完整淘汰清单与排序依据。

还支持 **Partitioned（CHIPS 独立分区 cookie）**：带 `Partitioned` 的 Set-Cookie 仅在同时带
`Secure`、且写入时提供有效 http(s) 顶层页面 URL 时入罐，**分区键 = 该 URL 的 scheme +
可注册域**（IP、localhost 用完整主机）；普通项与不同分区的同名同域同路径项并存，
发送时只随分区键相同的顶层页面发送，其余分区项在排除明细中说明原因。

技术栈：**TypeScript + Express + 原生 HTML/CSS/JS**（前端无框架、无构建），数据仅存内存；
same-site 判定使用 [tldts](https://github.com/remusao/tldts)（内置公共后缀列表 PSL，含私有域段）。

---

## 🚀 单条启动命令（Docker Compose）

前置条件：安装 **Docker**（含 Compose 插件，Docker Desktop 或 Docker Engine +
`docker-compose-plugin` 均可）。无需本机安装 Node.js。

在**项目根目录**执行这一条命令即可完成构建并在后台启动 Web/API：

```bash
docker compose up --build -d
```

启动后浏览器访问：

> **http://localhost:3000**

- Web 界面与全部 API（`/api/store`、`/api/send` 等）都在 **同一端口 localhost:3000** 提供；
- 容器内服务端口：**3000**；
- 宿主机默认映射端口：**3000**（想换端口改 `docker-compose.yml` 左侧，如 `"3001:3000"`）。

常用命令：

```bash
docker compose logs -f          # 跟踪服务日志（看到「🍪 Cookie Jar Lab 已启动」即就绪）
docker compose ps               # 查看运行状态
docker compose down             # 停止并删除容器（镜像保留）
docker compose down --rmi local # 同时删除本工具镜像
docker compose up --build -d    # 改完代码后重新构建并启动
```

> 不想用 Docker？见文末[不用 Docker 的本地运行](#不用-docker-的本地运行)。

---

## 🧭 使用流程

界面分为三步：

1. **写入 Set-Cookie**（可重复执行，模拟多个响应、按顺序入罐）
   - 填「响应 URL」（cookie 来源页面），如 `https://shop.example.com/admin/dashboard?tab=1`
   - 填「单条 Set-Cookie」（**不要**带 `Set-Cookie:` 前缀），如：
     `session=abc123; Domain=example.com; Path=/; Max-Age=3600; Secure`
   - 当 Set-Cookie 带 **`Partitioned`**（CHIPS）时，还需填「顶层页面 URL」
     （如嵌入方页面 `https://embed.example.net/`）：分区键取它的 scheme + 可注册域；
     普通 cookie 该栏留空即可
   - 点击 **入罐**，查看入罐 / 更新 / 删除 / 拒绝结果与原因
2. **发起请求**：填请求 URL（如 `https://api.shop.example.com/orders/123/detail`），
   并可指定 SameSite / 分区判定上下文——**顶层页面 URL**（留空 = 与请求 URL 同站/同分区）、
   **请求方法**、**是否顶层导航**；点击 **生成 Cookie 头**，查看：
   - 最终 `Cookie:` 请求头（可一键复制）与本请求的**分区键**
   - 请求站 vs 顶层站（scheme + 可注册域）的同站/跨站结论
   - 📤 **发送项**：命中的主机 / Path / Secure / SameSite / 分区键匹配点
   - 🚫 **排除项**：过期、Domain 不匹配、Path 不匹配、Secure 受限、SameSite 拦截、
     **分区键不符（Partitioned）**等逐项原因
3. **🫙 当前 Cookie 罐**：右侧表格实时展示全部条目（域、作用域、Path、**分区键**、Secure、
   Priority、有效期剩余、创建序号、**最近访问时间**）；点击 **重置** 清空内存罐。
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
| 13 | `https://widget.vendor.com/frame`（顶层页 `https://site-a.example/`） | `ch=1; Path=/; Secure; SameSite=None; Partitioned` | 入罐（**分区键 `https://site-a.example`**） |
| 14 | 同上（顶层页 `https://site-b.example/`） | `ch=2; Path=/; Secure; SameSite=None; Partitioned` | 入罐（**另一分区，与第 13 条同名同域同路径并存**） |
| 15 | `https://widget.vendor.com/frame`（顶层页留空） | `ch=3; Path=/; Secure; Partitioned` | **拒绝**：Partitioned 缺少有效顶层页面 URL |
| 16 | `https://widget.vendor.com/frame`（顶层页 `https://site-a.example/`） | `ch=4; Path=/; Partitioned` | **拒绝**：Partitioned 未带 Secure |
| 17 | `https://widget.vendor.com/frame`（顶层页 `https://site-a.example/`） | `ch=; Path=/; Secure; Partitioned; Max-Age=0` | **删除** site-a 分区的 ch；site-b 分区与普通项保留 |

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

Partitioned（CHIPS）演算示例（写入时填「顶层页面 URL」，发送时由顶层页面 URL 计算请求分区键）：

- 依次写入上表第 13、14 条（同名为 `ch`、同域同路径，分属 `site-a` / `site-b` 两个分区），
  再写一条不带 `Partitioned` 的 `ch=u` —— 罐中共存 **3 条**；
- 请求 `https://widget.vendor.com/`，顶层页 `https://site-b.example/` ——
  **仅 site-b 分区项与普通项发送**，site-a 分区项在排除明细中给出
  「条目分区键 … ≠ 请求分区键 …」；
- 顶层页换成 `https://site-c.example/` —— 两个分区项都排除，**只剩普通项 `ch=u`**；
- 顶层页留空 —— 请求分区键回退为请求站（`https://vendor.com`），分区项全部不匹配；
- IP 顶层页（如 `http://192.168.1.6/`）分区键用完整主机且 scheme 参与：
  换端口仍同分区，换 IP 或换 scheme 即跨分区。

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
- **Partitioned**（CHIPS，RFC 6265bis 独立分区 cookie；无值标志属性）
  - **入罐三个条件缺一不可**：声明了 `Partitioned`、**同时声明 `Secure`**、写入时提供的
    **顶层页面 URL 是有效的 http(s) 地址**；任一不满足都**拒绝入罐**（普通 cookie 不受影响，
    其顶层 URL 字段被忽略）；
  - **分区键 = 顶层页面 URL 的 scheme + 可注册域（eTLD+1）**；IP、localhost 等无可注册域的
    主机用**完整主机**（路径 / 端口 / 查询串不参与，scheme 参与，如
    `https://www.embed.example.net:8443/app` → `https://embed.example.net`）；
  - 分区键是**存储身份**的一部分：`(name, domain, path, partitionKey)` 才唯一确定一项，
    因此**普通项与不同分区的同名同域同路径项并存**；覆盖更新与 `Max-Age <= 0` 删除都
    **只影响对应分区**，不触及其他分区或普通项（删除不存在时明确说明「不影响其他分区」）；
  - 分区项与普通项一样计入**域级与全局容量**，参与同一条淘汰排序。
- **同名同域同路径（分区项还需同分区键）**再次写入 → 覆盖更新，**沿用首次创建序号**
  （保证排序稳定），并刷新该条目的最近访问时间；普通项与各分区项互不覆盖。
- **容量压力淘汰**（入罐通过以上全部校验后执行；未设置任何上限时完全不干预）
  - 可分别为每个**可注册域**（eTLD+1，tldts/PSL 判定；IP、localhost 按完整主机分桶）
    与**全局**设置条目数上限；
  - 入罐后**先域级（仅新条目所属可注册域）后全局**；修改容量设置则对全部已配上限立即执行；
  - 范围内条数超限时：**先清理过期项**，再按
    **Priority 低 → 高、最近访问早 → 晚、创建序号小 → 大**淘汰至回到上限；
  - **发送成功**的条目刷新最近访问时间，作为同级 Priority 的淘汰依据。

### 发送（处理请求 URL + SameSite / 分区上下文）

除请求 URL 外，演算还需要三个上下文参数（界面第 2 步）：

- **顶层页面 URL**：同时决定 same-site 基准站与 **Partitioned 请求分区键**；
  留空时视为与请求 URL 相同（同站、同分区）；
- **请求方法**：GET / POST / …（大小写不敏感）；
- **是否顶层导航**：地址栏/链接跳转属于顶层导航；子资源、iframe、fetch 不属于。

**站点（site）= scheme + 可注册域（eTLD+1）**，可注册域由公共后缀列表
（tldts 内置 PSL，含私有域段，如 `github.io`）判定；**IP 地址与 localhost
等无可注册域的主机按完整主机比较**（`localhost` ≠ `127.0.0.1`，端口不参与）。
请求站与顶层站一致即「同站」，否则「跨站」（`http://example.com` 与
`https://example.com` 因 scheme 不同也算跨站）。

逐条检查（顺序与 RFC 6265 一致）：

1. **过期检查**：已过期不发送（罐中保留并标灰显示）；
2. **分区检查（Partitioned / CHIPS）**：请求分区键由顶层页面 URL（缺省取请求 URL）按
   scheme + 可注册域计算；**分区项只在自身分区键与请求分区键完全相同时继续后续检查**，
   键不同直接排除并在明细给出两个键；普通（未分区）项不参与分区隔离，照常检查；
3. **Domain 匹配**：host-only 仅精确匹配原主机；Domain cookie 匹配该域及其子域；
4. **Path 匹配**：请求路径与 cookie-path 相等，或以其开头且边界字符是 `/`；
5. **Secure 检查**：Secure cookie 只在 `https:` 请求中发送（分区项同样受此约束）；
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
| GET | `/api/jar` | — | 当前罐快照（`cookies` + `capacity`：容量设置与各可注册域用量） |
| POST | `/api/store` | `{ "url": "...", "setCookie": "...", "topUrl"?: "..." }` | 顺序写入一条 Set-Cookie；`topUrl` 为顶层页面 URL，仅 `Partitioned` cookie 必需（计算分区键），普通 cookie 可省略。响应含 `eviction`（本次触发的域级/全局淘汰清单与排序依据，无限额时为 `null`） |
| POST | `/api/send` | `{ "url": "...", "topUrl"?: "...", "method"?: "GET", "topLevelNav"?: true }` | 演算 Cookie 头与逐项原因；请求分区键由顶层页面 URL（缺省取请求 URL）计算并以 `partitionKey` 返回，仅键相同的 `Partitioned` 项发送；发送成功的条目刷新最近访问时间，响应附最新 `cookies` 与 `capacity` |
| POST | `/api/capacity` | `{ "globalLimit"?: number \| null, "domainLimits"?: { "example.com": 3 } }` | 替换式设置全局 / 每可注册域容量（正整数；`null`/缺省 = 不限），立即执行一轮淘汰并返回报告 |
| POST | `/api/reset` | — | 清空内存罐（**容量设置保留**） |

curl 示例：

```bash
# 写入（SameSite 大小写不敏感；SameSite=None 必须带 Secure）
curl -s -X POST http://localhost:3000/api/store \
  -H 'Content-Type: application/json' \
  -d '{"url":"https://shop.example.com/admin/dashboard",
       "setCookie":"sid=abc; Domain=example.com; Path=/; Max-Age=3600; Secure; SameSite=Lax"}'

# 写入 Partitioned（CHIPS）：必须带 Secure，并提供顶层页面 URL 计算分区键
curl -s -X POST http://localhost:3000/api/store \
  -H 'Content-Type: application/json' \
  -d '{"url":"https://widget.vendor.com/frame",
       "setCookie":"ch=1; Path=/; Secure; SameSite=None; Partitioned",
       "topUrl":"https://embed.example.net/"}'

# 演算请求（partitionKey 回显请求分区键；仅分区键相同的 Partitioned 项发送）
curl -s -X POST http://localhost:3000/api/send \
  -H 'Content-Type: application/json' \
  -d '{"url":"https://widget.vendor.com/frame",
       "topUrl":"https://embed.example.net/page", "method":"GET", "topLevelNav":true}'

# 演算请求（跨站顶层导航 GET：Strict 被拦截，Lax/None 放行）
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
│   └── cookieJar.ts        # 全部 Cookie 演算规则（解析/入罐/匹配/排序/SameSite/Priority/Partitioned/容量淘汰）
└── public
    ├── index.html          # 原生前端页面（含顶层 URL 输入与容量压力配置面板）
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
> Domain / Path / Max-Age / Secure / SameSite / Priority / **Partitioned（CHIPS 分区）**
> 与容量压力淘汰等规则；
> 不处理 HttpOnly 发送限制（仅展示标签）、Expires、单条 cookie 字节大小上限等。
> 分区键与站点一致，取**顶层 URL 的 scheme + 可注册域**（IP、localhost 用完整主机），
> 不实现浏览器额外的第三方 cookie 策略开关。
> 容量按**条目条数**模拟（RFC 6265bis 的每可注册域 180 条、全局 3000 条同型机制），
> 时间精度取毫秒级，同一毫秒内的次序由创建序号兜底。
