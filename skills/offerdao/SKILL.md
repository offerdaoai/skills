---
name: offerdao
description: 通过 Offer岛 API 搜索和发布 AI 行业招聘信息。可以按公司 / 岗位 / 地点 / 工作类型 / 时间范围筛选已审核的岗位，也可以把一份 JD / 招聘原文，或一个 JD 链接 / 帖子 URL（如小红书、公众号）交给 agent 解析并发布——agent 发布的岗位需经审核通过后才会公开。当用户说"找岗位 / 找工作 / 查招聘 / 搜一下有什么职位"，或"发布岗位 / 帮我把这份 JD 发到 Offer 岛 / 把这个招聘发出去 / 把这个链接发出去"，或想按公司 / 岗位 / 地点 / 工作类型 / 时间筛选岗位时使用。默认连接 https://offerdao.ai。
---

# Offer岛 API 使用指南

Offer岛是一个聚合 AI 行业招聘信息的平台。这个 skill 教你通过它的开放 API 做两件事：

1. **搜索岗位**（`GET /api/postings/search`）——只读，在"已审核通过"的岗位库里按条件筛选。
2. **发布岗位**（`POST /api/postings/agent`）——把一份 JD 或招聘原文解析成结构化字段并提交。agent 发布的岗位**总是**进入审核流程，需管理员审核通过后才会对外可见。

> 搜索库里只有审核通过的岗位。未通过审核的岗位在搜索里看不到——所以你刚通过 agent 发布的岗位，在审核通过之前是搜不到的。

## Base URL（不要写死）

API 的域名**不要硬编码**。先解析一次，后面所有请求复用它：

```bash
OFFERDAO_BASE="${OFFERDAO_BASE:-https://offerdao.ai}"
```

下面所有示例都假设 `$OFFERDAO_BASE` 已经这样设置好了。

> 主域名目前是 `https://offerdao.ai`，后续可能更换。正因为可能换域名，**务必通过 `OFFERDAO_BASE` 解析，不要在代码里写死任何域名**；将来换了新域名，只需把 `OFFERDAO_BASE` 设成新地址，其余完全不变。

快速连通性探测（需带 key，见下文「鉴权」）：

```bash
curl -s "$OFFERDAO_BASE/api/postings/search?limit=1" \
  -H "Authorization: Bearer $OFFERDAO_API_KEY" | head -c 200
```

如果报连接错误，可能是域名已更换——请向用户确认正确的地址，不要瞎猜。

## 鉴权：API key（必读）

**搜索和发布两个接口都必须带 API key**，否则返回 `401`。key 形如 `sk-xxxxxxxxxxxxxxxxx`，**每个用户自己**在网站上创建管理。

### 如何获取 API key（一步步）

key 必须登录后在网站上自助创建——**没有 key 之前，本 skill 的搜索和发布都用不了**。引导用户按下面的步骤拿到 key：

1. **打开网站并登录**：浏览器访问 `$OFFERDAO_BASE`（默认 `https://offerdao.ai`），点右上角**登录 / 注册**，用邮箱或 Google 登录。
2. **进入设置**：登录后点右上角**头像**，在菜单里选**我的设置**。
3. **切到 API keys**：在设置弹窗左侧栏点 **API keys** 标签。
4. **新建 key**：点**新建 key**，系统会生成一个 `sk-` 开头的字符串。**新建后请立刻复制保存**——它与你的账号绑定（也可随时回到该页「查看 / 复制」已有的 key）。
5. **填进环境变量**：把复制到的 key 交给 agent，存进 `OFFERDAO_API_KEY`（见下方），后续所有请求复用。

> 整条路径：登录 `$OFFERDAO_BASE` → 右上角**头像 → 我的设置 → API keys → 新建 key** → 复制保存。

把 key 放进环境变量，后续所有请求复用（**不要把 key 写死在代码或粘进聊天里**）：

```bash
OFFERDAO_API_KEY="${OFFERDAO_API_KEY:-sk-请替换成你的key}"
```

每个请求带上鉴权头，两种写法二选一：

```bash
-H "Authorization: Bearer $OFFERDAO_API_KEY"
# 或
-H "X-API-Key: $OFFERDAO_API_KEY"
```

> **key 与用户绑定。** 用某个 key 发布的岗位会归属到该 key 的拥有者——发布后岗位会出现在 TA 登录态的「我的发布」里（待审核状态）。所以发布前要确认你用的是**用户本人**的 key。
>
> 没有 key / key 无效时：搜索与发布都会返回 `401`（`{ "error": "missing_api_key" }` 或 `{ "error": "invalid_api_key" }`）。此时请引导用户去「我的设置 → API keys」创建 / 确认 key，**不要绕过鉴权，也不要伪造 key**。

## 接口一：搜索岗位 `GET /api/postings/search`

服务端会帮你做条件筛选（多条件 AND 组合）和分页。

所有参数都是可选的；不带参数默认返回最近发布的 30 条已审核岗位。

| 参数 | 类型 | 说明 |
|---|---|---|
| `q` | string | 多词 AND 搜索。按空格切分，每个词都必须出现在 公司 / 团队 / 岗位 / 地点 / 简介 / 描述 / 要求 / 联系方式 / 标签 / 工作类型 / 多岗位 的某个字段里。中文要 URL 编码。 |
| `organization` | string | 对公司字段做子串匹配（"字节"能匹配到"字节跳动"）。 |
| `role` | string | 对岗位字段做子串匹配。 |
| `location` | string | 对地点字段做子串匹配（"北京"能匹配到"北京 / 上海"）。 |
| `company_tag` | string | 对单个 `company_tag` 字段做**精确匹配**（非子串）。合法值（枚举，由管理员维护）：`国内大厂` / `知名外企` / `独角兽` / `知名初创` / `明星团队`。传枚举外的值会一条都搜不到。 |
| `employment_type` | string 或 CSV | 每个值必须等于该岗位 `employment_type` 数组里的某一项。可重复传参或逗号分隔。常见值：`正式`、`实习`、`访问`、`研究助理`、`PhD`、`Postdoc`。 |
| `role_tag` | string 或 CSV | 语义同上，作用于 `role_tags` 数组。 |
| `published_within_days` | number | 只保留 `published_ts` 在最近 N 天内的岗位。没有可解析发布时间的岗位会被这个过滤器剔除。 |
| `has_contact` | `1` / `true` | 只保留至少有一个联系方式的岗位（`contact_email`、`contact_wechat` 或 `contact_link` 任一）。 |
| `limit` | int 1–30 | 默认 30。 |
| `offset` | int ≥ 0 | 分页偏移，默认 0。过大时返回 `400 { error: "offset_too_large", max_offset }`（响应里会带上当前上限）。 |
| `sort_by` | 枚举 | `published`（默认，按 `published_ts`，回退 `fetched_at`）/ `fetched` / `organization` / `role`。 |
| `order` | `asc` / `desc` | 默认 `desc`。 |

返回：

```json
{
  "items": [ { /* 岗位对象 */ }, ... ],
  "total": 1234,
  "limit": 30,
  "offset": 0
}
```

`total` 是匹配该筛选条件的总条数（不受 limit/offset 影响），方便分页。

### 岗位对象结构

`items` 里每一条：

```jsonc
{
  "posting_id": "manual_xxxxxxxx" | "agent_xxxxxxxx" | "<系统生成 id>",
  "source_url": "https://example.com/jobs/..." | "<主页>",
  "organization": "字节跳动",
  "team": "Seed",                  // 可选
  "role": "多模态算法工程师",
  "employment_type": ["正式"],     // 字符串数组
  "role_tags": ["CV", "Infra"],    // 字符串数组，≤ 5
  "company_tag": "国内大厂" | null,   // 管理员维护，搜索可按它精确过滤
  "location": "北京 / 上海",
  "intro": "...",                  // 简短简介
  "description": "...",            // JD 正文
  "requirements": "...",           // 招聘要求
  "contact": "邮箱: a@b.com",      // 自由文本联系方式，可能为空
  "contact_email":  "a@b.com" | null,
  "contact_wechat": "wx_id" | null,
  "contact_link":   "https://..." | null,
  "referral_code": "ABC123" | null,
  "logo_url": "data:image/jpeg;base64,..." | null,   // ⚠️ 可能很大；输出 offerdao-cards 时一律置 null，勿照搬
  "positions": "[ {role, employment_type[], description, requirements}, ... ]" | null,
  "published_at": "2026-05-12",   // 可解析时为 YYYY-MM-DD
  "published_ts": 1747008000,     // unix 秒，无法解析时为 null
  "fetched_at":   1747010000      // unix 秒
}
```

多岗位的招聘，`positions` 是一个 JSON 字符串——需要 `JSON.parse` 解析。单岗位的招聘 `positions` 为 `null`，直接用顶层的 `role` / `description` / `requirements`。

### 搜索示例

下面的例子都用 `curl --get --data-urlencode`，这样中文和空格的 URL 编码不会出问题。**每个请求都带上鉴权头**（`-H "Authorization: Bearer $OFFERDAO_API_KEY"`）。

```bash
# 1)「找北京的 AI infra 岗位，只要正式，最近 30 天」
curl -sG "$OFFERDAO_BASE/api/postings/search" \
  -H "Authorization: Bearer $OFFERDAO_API_KEY" \
  --data-urlencode "q=infra" \
  --data-urlencode "location=北京" \
  --data-urlencode "employment_type=正式" \
  --data-urlencode "published_within_days=30" \
  --data-urlencode "limit=20"

# 2)「看看字节有什么岗位」
curl -sG "$OFFERDAO_BASE/api/postings/search" \
  -H "Authorization: Bearer $OFFERDAO_API_KEY" \
  --data-urlencode "organization=字节"

# 3)「PhD / Postdoc / 实习，按公司名排序」
curl -sG "$OFFERDAO_BASE/api/postings/search" \
  -H "Authorization: Bearer $OFFERDAO_API_KEY" \
  --data-urlencode "employment_type=PhD,Postdoc,实习" \
  --data-urlencode "sort_by=organization" --data-urlencode "order=asc"

# 4)「只看能联系上的（有联系方式的）多模态岗位」
curl -sG "$OFFERDAO_BASE/api/postings/search" \
  -H "Authorization: Bearer $OFFERDAO_API_KEY" \
  --data-urlencode "q=多模态" --data-urlencode "has_contact=1"
```

返回结果可以用 `jq` 提炼，便于阅读：

```bash
curl -sG "$OFFERDAO_BASE/api/postings/search" \
  -H "Authorization: Bearer $OFFERDAO_API_KEY" \
  --data-urlencode "q=多模态" --data-urlencode "limit=10" \
  | jq '.items[] | {posting_id, organization, role, location, contact_email, contact_link, source_url}'
```

把岗位推荐给用户时，可以用 `posting_id` 拼出一个可点开的详情页短链：`$OFFERDAO_BASE/j/<posting_id>`（例如 `https://offerdao.ai/j/manual_ab12cd34`）。这个链接会打开该岗位的详情页，分享到微信 / Slack 等还会渲染岗位卡片，比直接甩 `source_url` 更友好。

## 接口二：发布岗位 `POST /api/postings/agent`

当用户把一份招聘内容交给你、让你发布时（可能是粘贴的文本，也可能只是一个链接），你先把它解析成下面的结构化字段，再以 JSON 形式 `POST`（`Content-Type: application/json`）。**这个接口必须带 API key**（见上文「鉴权」），匿名发布已不再支持。

agent 发布的岗位**总是**进入审核流程，`posting_id` 以 `agent_` 开头——**审核通过前不对外可见**。岗位会**归属到你所用 API key 的拥有者**，因此发布后会出现在 TA 登录态的「我的发布」里（待审核状态，可在那里查看 / 编辑 / 撤下）。

### 输入来源与解析

用户提供的原始内容通常是三种形态，先把它**归一化成结构化字段**，再走下面的「字段」与「发布流程」：

1. **结构化信息** —— 用户已把公司、岗位、联系方式等列清楚，直接映射到字段即可。
2. **非结构化文本** —— 用户粘贴整段 JD、招聘原文、群消息等。你通读后抽取字段，能填尽填；**必填项缺失就停下来请用户补齐，绝不编造**（公司名、联系方式尤其不能臆造）。
3. **一个链接（URL）** —— 用户给的是 JD 原帖、公司招聘页、小红书 / 公众号帖子等。**不要直接拿 URL 发布**，先抓取页面渲染后的正文，再按第 2 种方式解析：
   - 用 **browser-act** skill 打开并提取页面正文——小红书、需要登录或有反爬的页面必须用真实浏览器渲染，普通 `curl` 抓不到内容。
   - 抓取参考：`browser-act stealth-extract <URL>`（受保护 / 反爬页面），或开会话逐步操作 `browser-act --session <name> browser open <id> <URL>`。
   - **若环境未安装 browser-act**，先帮用户安装：让 agent 执行「Install browser-act，源地址 `https://github.com/browser-act/skills/tree/main/browser-act`」，安装后验证可用，再继续抓取。
   - 拿到正文后同样抽取字段，**必填项缺失仍要请用户补齐，不要从页面瞎猜**。

> 这一步只负责拿到并解析内容；最终发布前仍要走「发布流程」里的确认与多岗位合并判断。

### 字段

四组**必填**——任何一组缺失，先问用户补齐再发布。**绝不要编造**任何值（联系方式尤其不能凭空捏造）：

| 字段 | 类型 | 用途 / 约束 |
|---|---|---|
| `organization` | string | **必填。** 公司 / 学校 / 机构名称。 |
| `role` | string | **必填（单岗位）。** 岗位名称。多岗位时改用 `positions`，并省略顶层 `role`。 |
| `employment_type` | string[] | **必填。** 工作类型，至少一个。常见值：`正式` / `实习` / `访问` / `研究助理` / `PhD` / `Postdoc`。每项 ≤16 字符。 |
| 联系方式（三选一） | — | **至少填一个**：`contact_email` / `contact_wechat` / `contact_link`。 |

**可选**字段——JD 里有就填，没有就留空，不要猜：

| 字段 | 类型 | 用途 / 约束 |
|---|---|---|
| `contact_email` | string | 招聘邮箱，必须是合法邮箱格式。 |
| `contact_wechat` | string | 招聘微信号，≥2 字符。 |
| `contact_link` | string | 投递 / 岗位链接，必须以 `http(s)://` 开头；服务端会做**可达性探测**，错误或失效的链接会被拒。 |
| `team` | string | 团队 / 部门（如 `Seed`）。 |
| `intro` | string | 列表卡片上展示的一句话简介。 |
| `description` | string | JD 正文。 |
| `requirements` | string | 招聘要求。 |
| `location` | string | 工作地点，多个地点用 `/` 分隔（如 `北京 / 上海`）。 |
| `source_url` | string | 原帖 / 主页链接，必须以 `http(s)://` 开头（不做可达性探测）。建议填，方便审核人核对来源。 |
| `role_tags` | string[] | 岗位标签，≤5 个，每个 ≤7 字符（如 `["CV", "多模态"]`）。被 `role_tag` 搜索过滤器使用。 |
| `referral_code` | string | 内推码，≤64 字符。 |
| `poster_type` | string | 发布者身份标签（如 `HR` / `内推`），≤24 字符。agent 代发通常无需填写。 |
| `positions` | object[] | **多岗位**：每项 `{ role*, employment_type*[], description, requirements }`。存在时，顶层 `role` / `employment_type` 取自 `positions[0]`；每个岗位都需要至少一个 `employment_type`。 |
| `logo_url` | string | 品牌 logo 的 `data:image/...` URL，≤256KB。通常省略。 |

> **`company_tag`（公司类型）不能由 agent / 发布接口设置**——它是管理员审核时在后台设定的编辑字段，发布时即使带上也会被服务端**静默忽略**。它只用于上面「搜索接口」的精确过滤（合法值见搜索参数表）。

### 发布流程（agent 应遵循的步骤）

0. **确认有 key**：发布必须带 API key（见「鉴权」）。没有就先引导用户去「我的设置 → API keys」创建，并确认用的是**用户本人**的 key（岗位会归属到 key 拥有者）。
1. **拿到原始内容**：按上面「输入来源与解析」处理——文本直接解析；URL 先用 browser-act 抓取正文再解析（未装则先帮用户装）。
2. **解析并填字段**：通读内容，尽可能多地填好下面的字段，能填尽填。
3. **判断单岗位 / 多岗位**：如果同一家公司（同一个帖子或链接）里有多个岗位，**合并成一张卡片**——把每个岗位放进 `positions` 数组，顶层 `role` / `employment_type` 省略（自动取自 `positions[0]`）。**绝不要把同一家公司的多个岗位拆成多张卡片分别发布**；只有不同公司才分多张卡片。
4. **检查四组必填项**：任何一组缺失，就明确告诉用户缺什么并请其补齐，**绝不编造**。
5. **回显结构化结果给用户确认**（至少包含 公司、各岗位、工作类型、联系方式、标签）。
6. 用户确认后，再带上 key `POST $OFFERDAO_BASE/api/postings/agent`。
7. 告诉用户岗位已进入审核，需管理员审核通过后才会公开；返回的 `posting_id` 以 `agent_` 开头；同时可在「我的发布」里看到这条待审核岗位。

### 返回

```json
{ "ok": true, "posting_id": "agent_ab12cd34..." }
```

- 相同的 公司 + 岗位 + 联系方式 在短时间内重复提交：`{ "ok": true, "posting_id": "...", "duplicate": true }`（不会重复入库）。
- 校验失败：返回 `4xx` + `{ "error": "..." }`（如缺必填字段、邮箱格式错误、链接不可达）。

### 发布示例

```bash
# 单岗位
curl -s "$OFFERDAO_BASE/api/postings/agent" \
  -H "Authorization: Bearer $OFFERDAO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "organization": "字节跳动",
    "team": "Seed",
    "role": "多模态算法工程师",
    "employment_type": ["正式"],
    "location": "北京 / 上海",
    "intro": "Seed 多模态团队招算法",
    "description": "负责多模态大模型预训练与对齐……",
    "requirements": "熟悉 PyTorch，有顶会论文优先……",
    "role_tags": ["CV", "多模态"],
    "contact_email": "hr@bytedance.com",
    "source_url": "https://example.com/jobs/xxxx",
    "referral_code": "ABC123"
  }'

# 多岗位：顶层 role / employment_type 省略，由 positions 提供
curl -s "$OFFERDAO_BASE/api/postings/agent" \
  -H "Authorization: Bearer $OFFERDAO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "organization": "某 AI 初创",
    "contact_wechat": "ai_hr_2026",
    "positions": [
      {"role": "LLM 训练工程师", "employment_type": ["正式"], "description": "...", "requirements": "..."},
      {"role": "推理优化实习生", "employment_type": ["实习"], "description": "...", "requirements": "..."}
    ]
  }'
```

## 限流与错误处理

两个端点都有按 IP 的频率限制（搜索较宽松，发布较严格）。每个响应都带 `X-RateLimit-Remaining` 头，告诉你当前剩余配额。超限时返回：

```
HTTP 429
Retry-After: <秒>
{ "error": "rate_limit_exceeded", "retry_after": <秒> }
```

撞到 429 时，**按 `retry_after` 秒退避后再重试，不要立即重发**。批量搜索 / 发布时主动放慢节奏。

**鉴权错误（401）：**

```
HTTP 401
{ "error": "missing_api_key" }   // 完全没带 key
{ "error": "invalid_api_key" }   // key 无效 / 已被删除
```

撞到 401 时**不要重试**，先去「我的设置 → API keys」确认 / 重建 key 再来。

其他错误（缺必填、邮箱格式、链接不可达等）返回 `4xx` + `{ "error": "..." }`，其中 `error` 已是面向用户的中文文案（如「招聘链接无法访问，请确认是否拼写错误或目标网站已下线」），可直接转述给用户。

## 注意事项 / 易踩的坑

- **搜索和发布都必须带 API key。** key 形如 `sk-...`，在「我的设置 → API keys」自助创建；放进 `OFFERDAO_API_KEY` 复用，**别写死、别外泄、别伪造**。401 时去那里确认 / 重建。
- **发布岗位归属到 key 拥有者。** 用谁的 key 发，岗位就进谁的「我的发布」（待审核）。务必用**用户本人**的 key。
- **域名可配置，绝不写死。** 默认是 `https://offerdao.ai`（后续可能更换）；将来换新域名时，通过 `OFFERDAO_BASE` 切换即可。
- **只能搜到已审核的。** 搜索永远看不到未通过审核的岗位。通过 agent 端点刚发布的岗位，在管理员审核通过前搜不到。
- **子串匹配 vs 标签精确匹配。** `organization` / `role` / `location` 是子串包含匹配；`employment_type` / `role_tag` 是数组，每个值必须等于数组里的某一项——拼错不会模糊匹配。
- **参数之间是 AND，`q` 内部也是 AND。** `q=北京 多模态&location=上海` 会返回空集，因为岗位既要同时匹配 `北京` 和 `多模态`，又要 `location` 含 `上海`。搜不到时去掉一个条件再试。
- **URL 里的中文。** 一定要 URL 编码（`curl --data-urlencode`，或 JS 里 `encodeURIComponent`）。URL 里直接放中文在有些客户端能用、有些会出错。
- **搜索只读；发布受审核限制。** agent 发布端点（`/api/postings/agent`）提交的岗位永远需要审核。审核、拒绝、编辑岗位都是管理员专属，不在本 skill 范围内。
- **URL 不能直接发布。** 用户给链接（JD 原帖、小红书 / 公众号帖子等）时，先用 browser-act skill 抓取渲染后的正文再解析；环境没装 browser-act 就先帮用户安装（源地址见「输入来源与解析」）。
- **同公司多岗位 = 一张卡片。** 一个帖子 / 链接里同一家公司有多个岗位时，用 `positions` 内联成一张卡片发布，不要拆成多张单独卡片；不同公司才分卡片。
- **必填项缺失就问，不臆造。** 不论输入是文本还是 URL，公司 / 岗位 / 工作类型 / 联系方式这四组必填项缺失时，一律停下来请用户补齐，绝不凭空编造（联系方式尤其）。
- **limit 上限为 30。** 要拉更多请用 `offset` 分页。

## 什么时候不要用这个 skill

- 用户想**自己在网站上填表发布** → 引导他去站点首页（`$OFFERDAO_BASE/`）。只有当用户把 JD / 招聘原文交给*你*、让你代为提交时，才用上面的 agent 发布端点。
- 用户想**审核**（通过 / 拒绝岗位）→ 这是管理员专属，不通过本 skill。
- 用户**还没有 API key**（或不愿意创建）→ 本 skill 的搜索 / 发布都用不了，先引导去「我的设置 → API keys」创建，别试图绕过鉴权。
