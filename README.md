[项目说明文档.md](https://github.com/user-attachments/files/33934788/default.md)
# CodeDetective · AI 代码验真器

> 用**元宝搜索 API** 构建的代码幻觉检测工具。把 AI 生成的代码逐条比对官方文档，把幻觉 API、废弃写法、编造的 npm 包名揪出来。

- **线上地址**：https://xiunao.github.io/codedetective/
- **中转服务**：`https://<your-worker>.<your-subdomain>.workers.dev/`（Cloudflare Worker，部署者自填）
- **上游 API**：腾讯云元宝搜索 SearchPro
- **项目背景**：WaytoAGI 1024 超能 Coding 周末 · 侦探赛道

---

## 一、项目简介

### 1.1 为什么做这个

AI 写代码越来越强，但**幻觉**问题一直没解决——尤其是"听起来合理、实际不存在或已废弃"的 API 和包名。

这类错误不会立刻报错，它会安静地躺在你的代码里，直到某天线上崩掉，或者更糟：你执行了一条 `npm install`，把别人抢注的同名恶意包拉进了构建流程。

CodeDetective 的定位很简单：**AI 给你的代码，先查再用。**

### 1.2 与侦探赛道主题的关系

本次 1024 切磋大会的侦探赛道，主题是**"有据可查"**——用检索去核实一个说法的真伪。

CodeDetective 是这个主题在**编程场景**下的一个具体落点：

| 侦探赛道的通用命题 | 在 CodeDetective 中的对应 |
|-------------------|--------------------------|
| 一个说法是真的吗？ | 这行 API 调用是真的吗？ |
| 证据来自哪里？ | 官方文档、升级指南、npm registry |
| 能不能复核？ | 每张验真卡都附可点开的来源链接 |

作品的全部价值建立在**检索**之上。把元宝搜索 API 抽掉，它就退化成一个普通的代码片段展示页——没有任何查证能力。反过来说，**搜索是它的唯一核心**。

### 1.3 核心流程

```
用户粘贴 AI 生成的代码
        ↓
前端（GitHub Pages）发到中转服务
        ↓
Cloudflare Worker 带上 Key 请求元宝 SearchPro
        ↓
返回带官方来源的实时检索结果
        ↓
前端渲染成"验真卡"
```

即使不填 Key，页面也内置了 3 张经过人工查证的验真卡（React 19 / Tailwind v4 / npm 幻觉包），保证任何情况下都能完整演示。

### 1.4 内置的三张演示卡

| 卡片 | 判定 | 问题 |
|------|------|------|
| React 19 移除了 `ReactDOM.render` | 🔴 已确认废弃 | v18 标记废弃，v19 正式移除，需迁移到 `createRoot` |
| Tailwind v4 已删除 `tailwind.config.js` | 🟡 版本陷阱 | v4 改用 CSS-first 的 `@theme`，大量 AI 仍输出 v3 写法 |
| AI 常编造 npm 包名 `unused-imports` | 🔴 高危供应链风险 | 模型编造不存在的包名，攻击者抢注植入恶意代码（slopsquatting） |

---

## 二、技术架构

```
┌─────────────────┐
│  GitHub Pages   │  纯静态 HTML/CSS/JS，零依赖
│  前端页面        │
└────────┬────────┘
         │ POST { code, apiKey }
         │ （跨域，走 CORS）
         ▼
┌─────────────────┐
│ Cloudflare      │  ① 处理 OPTIONS 预检
│ Worker 中转     │  ② 校验参数、截断长代码
│                 │  ③ 带 Key 转发上游
└────────┬────────┘
         │ POST + Authorization: Bearer <API_KEY>
         ▼
┌─────────────────┐
│ 元宝搜索         │  api.wsa.cloud.tencent.com/SearchPro
│ SearchPro API   │  返回 Response.Pages（JSON 字符串数组）
└─────────────────┘
```

### 为什么需要中转层？

1. **跨域**：浏览器无法直接请求 `api.wsa.cloud.tencent.com`（无 CORS 头）
2. **中转可控**：可以在中转层做参数校验、长代码截断、错误兜底
3. **解耦**：前端只需改一个 `DEFAULT_RELAY` 常量，随时可换中转实现

---

## 三、完整部署步骤

> 全程零费用：GitHub Pages 免费 + Cloudflare Worker 免费额度 + 元宝搜索免费额度。

### 第 1 步：获取元宝搜索 API Key

1. 打开 https://cloud.tencent.com ，扫码登录
2. 完成实名认证（免费，必须）
3. 控制台顶部搜索框输入「**元宝搜索**」，进入产品页
4. 点「立即开通」
5. 左侧菜单 →「API Key / 密钥管理」→「新建密钥」
6. 名称随意（如 `codedetective`），点创建
7. ⚠️ **Key 只在创建时完整显示一次**，请立即复制保存

> Key 形如 `sk-xxxxxxxx`，是调用凭据，**不要**写入代码或提交到任何公开仓库。

### 第 2 步：创建 Cloudflare Worker

1. 打开 https://dash.cloudflare.com ，注册/登录
2. 左侧菜单 → **Workers & Pages**
3. 点 **Create** → **Create Worker**
4. 起个名字，例如 `cd-relay`（名字会出现在最终地址里）
5. 点 **Deploy** 完成初始创建

创建后你会拿到一个地址，形如：

```
https://<your-worker>.<your-subdomain>.workers.dev
```

记下来，后面要填进前端。

### 第 3 步：写入 Worker 代码

进入 Worker → **Edit Code**，把默认代码全选删掉，粘贴文末「附录 A：Worker 完整代码」，点 **Deploy**。

> 关于「使用者自己输入 Key」的场景：Key 由前端传入、Worker 转发，因此**不需要**在 Cloudflare 配置任何密钥环境变量。

### 第 4 步：部署前端页面

把文末「附录 B：前端核心代码」整合进 `index.html`，推到你的 GitHub 仓库并开启 Pages。

**必须修改的一行**：

```js
const DEFAULT_RELAY = "https://<your-worker>.<your-subdomain>.workers.dev/";
```

换成你自己的 Worker 地址。

### 第 5 步：验证

1. 等 GitHub Pages 构建完成（约 1 分钟）
2. 打开页面，**Ctrl + Shift + R** 强制刷新
3. 在 Key 输入框填入你自己的 Key
4. 点「开始验真」
5. 出现绿色「🔎 实时检索结果」卡片 → 成功 ✅

---

## 四、核心代码说明

### 4.1 Worker 关键逻辑

| 环节 | 处理 |
|------|------|
| OPTIONS 预检 | 返回 `204` + 完整 CORS 头，否则浏览器会拦截后续请求 |
| 方法校验 | 非 POST 返回 405 |
| 参数校验 | 缺 `code` 或 `apiKey` 返回 400 |
| 长代码截断 | 超过 500 字符截取前 500 + `...(代码过长已截断)` |
| 转发上游 | 带 `Authorization: Bearer <API_KEY>` 请求元宝 SearchPro |
| 501 兜底 | 上游 501 时返回可读的错误提示 |

### 4.2 前端关键逻辑

| 环节 | 处理 |
|------|------|
| 防重复点击 | 请求期间按钮 disabled 并显示「验真中…」 |
| 超时控制 | `AbortController`，15 秒自动中断 |
| 长代码兜底 | 前端也做一次 500 字符截断，双保险 |
| Pages 解析 | 上游返回的是 **JSON 字符串数组**，需逐条 `JSON.parse` |
| 错误分级 | 401/403→Key 问题，501→Worker 未部署，405→Worker 未更新，超时→网络 |
| 失败回落 | 失败时展示红色提示卡 + 排查清单，内置验真卡仍可正常演示 |

### 4.3 关于 `Response.Pages` 的解析（重要）

元宝 SearchPro 返回的 `Pages` 是**字符串数组**，每一项都是需要二次解析的 JSON 字符串：

```js
// 错误写法 —— 字符串没有 .site / .url / .passage
pages.map(p => p.site)   // → undefined → "未知站点"、链接点不了、内容为空

// 正确写法
let pages = j?.Response?.Pages || [];
pages = pages.map(p => {
  if (typeof p === 'string') {
    try { return JSON.parse(p); } catch { return null; }
  }
  return p;
}).filter(Boolean);
```

---

## 五、踩坑与排查记录

这一节记录了从零到跑通过程中实际遇到的问题，按现象 → 原因 → 解法排列。

### 坑 1：`资源解析服务请求失败` / URL 上带着 `?u=`

- **现象**：请求地址后面跟了一长串 `?u=https%3A%2F%2Fapi.wsa...`
- **原因**：仍在用最初版本的旧逻辑，前端把目标地址当参数传给 Worker 做通用转发
- **解法**：前端改为直接 POST 到 Worker，body 里只传 `{ code, apiKey }`，Worker 端目标地址写死

### 坑 2：HTTP 405 Method Not Allowed

- **现象**：网络面板出现 `preflight` + `405`
- **原因**：前端发带 `Content-Type: application/json` 的 POST 时，浏览器会**先自动发一个 OPTIONS 预检**；Worker 只处理了 POST，把 OPTIONS 拒了，浏览器随即拦截真正的请求
- **解法**：Worker 里显式处理 OPTIONS，返回 `204` + CORS 头：

```js
if (request.method === "OPTIONS") {
  return new Response(null, { status: 204, headers: corsHeaders });
}
```

### 坑 3：HTTP 501 Not Implemented

- **现象**：预检已通过（204），但实际 POST 返回 501
- **原因**：上游对请求体参数有要求，多余参数会触发 501
- **解法**：精简请求体；同时在 Worker 里对 501 做兜底，返回可读的中文提示
- **排查清单**：① Key 是否为元宝搜索创建的（非腾讯云 SecretKey）② 服务是否已开通 ③ 账号是否欠费

### 坑 4：结果全是「未知站点」、链接点不了、内容为空

- **现象**：绿卡显示「真实调用成功」，但来源全是「未知站点」，点不动，摘要空白
- **原因**：`Response.Pages` 是字符串数组，前端直接当对象读属性
- **解法**：加二次 `JSON.parse`（见 4.3）

### 坑 5：只能搜几个字符，长代码搜不了

- **现象**：短代码正常，粘贴几十行代码就返回空或失败
- **原因**：元宝 SearchPro 的 `Query` 有长度限制（约 500 字），超长文本会导致调用失败或空结果
- **解法**：前端 + Worker 双重截断，超过 500 字符保留前 500 并追加 `...(已截断)`

### 坑 6：改完代码还是报旧错

- **现象**：明明更新了代码，依然报 405/501
- **原因**：① Worker 改完没点 **Deploy**；② 浏览器缓存了旧结果
- **解法**：Worker 每次改完**必须点 Deploy**；前端强制刷新；DevTools → Network → 勾选 **Disable cache**

### 坑 7：中转服务在国内网络下超时

- **现象**：请求发到 `*.workers.dev` 后长时间无响应，最终 `ERR_CONNECTION_TIMED_OUT`
- **原因**：Cloudflare 免费节点的二级域名在国内网络环境下可达性不稳定
- **解法**：可改用国内云厂商的函数计算/边缘函数作为中转，或在演示环境下由使用者自行配置可达的中转地址

---

## 六、安全说明

| 风险 | 处理方式 |
|------|----------|
| API Key 泄露 | Key 由使用者本人输入，仅用于当次请求；不写入代码、不提交仓库 |
| 中转被刷量 | 可在 Worker 层增加令牌校验或接入限流规则 |
| 任意转发（SSRF） | Worker 内目标地址**写死**为元宝端点，不接受外部传入的 URL |
| 跨域滥用 | CORS 头在生产环境建议收紧为指定域名，而非 `*` |

> **重要**：本项目没有把任何 Key 硬编码进前端或仓库。GitHub 仓库中绝不能出现 `sk-` 开头的字符串。中转地址在本文档中一律以占位符表示，实际部署时替换为部署者自己的地址。

---

## 七、使用说明

### 不填 Key（默认演示）

直接滚动查看「内置验真卡」，三张卡片均经过人工查证，附官方文档链接可点开核对。可切换不同卡片，也可一键复制整张卡。

### 填 Key（真实检索）

1. 前往腾讯云控制台 → 元宝搜索 → 创建 API Key
2. 粘贴到页面输入框
3. 在代码框粘贴任意 AI 生成的代码（长代码会自动截取）
4. 点「开始验真」
5. 页面在内置卡上方生成绿色实时检索卡，列出检索来源与摘要

### 失败时怎么办

失败会显示红色卡片并给出具体原因，常见项：

- **Key 无效或已过期** → 重新去腾讯云控制台创建
- **Worker 未更新** → 确认 Cloudflare 上已点 Deploy
- **请求超时（15 秒）** → 检查网络是否能访问中转服务
- **元宝返回为空** → Key 权限 / 代码太短 / 服务未开通

---

## 八、关于元宝搜索 API

### 8.1 它是这个作品的基座

CodeDetective 的思路可以概括为：**把"AI 的说法"和"公开可查的事实"做一次对齐**。

在这个对齐过程中，元宝搜索 API 承担的是唯一的取证环节——所有判定、所有来源链接、所有置信度的依据，都来自它返回的结果。没有它，作品就只是一个静态页面；有了它，作品才具备"侦探"的意义。

这也是它在本项目的立足点：**搜索不是它的一个功能，而是它的全部。**

### 8.2 它好用在哪

实际接入后的几个直观感受：

- **响应结构清晰**：返回的 `Pages` 里每条都带标题、链接、摘要、站点、时间，几乎不需要额外清洗就能直接渲染
- **检索质量稳定**：对技术类问题，能比较稳定地命中官方文档与升级指南，而不是泛泛的博客
- **接入成本低**：一个 POST 请求、一个 Bearer 鉴权即可，参数精简，调试路径短
- **参数可控**：支持时效、站点限定等维度，便于把检索约束在某个官方文档站或某个时间范围内

### 8.3 同一个能力，能长出的其他作品

代码验真只是元宝搜索 API 在"查证"这个方向上的一个切面。同一套检索能力，换一个提问对象，就能变成完全不同的产品：

| 场景 | 它解决的问题 | 怎么用检索 |
|------|-------------|-----------|
| **查家族群传言的真假** | "吃这个致癌""某地要封城"——长辈群里最常见的转发 | 把传言原文丢进去检索权威来源与辟谣信息，输出结论 + 来源，让家人不用再靠直觉判断 |
| **查演唱会还有没有票** | 票务信息变动快，二手平台和黄牛消息混杂 | 检索官方票务平台、主办方公告的最新状态，给出可直接跳转的购票入口 |
| **找一部只记得剧情的剧** | 只记得"主角失忆了""结局在雪地里"，记不得名字 | 用剧情片段做语义检索，从讨论帖、剧情简介里把片名捞出来 |
| **查证 AI 生成的代码**（本项目） | AI 写的 API / 包名可能不存在或已废弃 | 检索官方文档与 registry，输出验真卡 |

这些场景看起来毫不相关，但底层是同一件事：**当你手上有了一个说法，而不是一个确定的答案时，用检索把它对应到公开可查的证据上。**

---

## 九、局限与说明

需要坦白的是，这个作品**并不完整**。

它目前只做到了"验证代码真实性"这一件事，而且只做到了最基础的形态：

- 只覆盖了几类典型的代码幻觉（废弃 API、版本陷阱、编造包名），远未穷尽
- 判定逻辑依赖检索结果的文本比对，没有做结构化的规则引擎或语义判断
- 交互与界面较为简单，没有做批量校验、历史记录、结果导出
- 中转链路依赖第三方免费服务，在部分网络环境下可达性不稳定
- 内置的 3 张验真卡为人工查证后预置，仅用于保证任何环境下都能完整演示

这些都受限于本次开发的时间与个人能力。在 1024 切磋大会两天的时间内，能把它从想法推进到"可打开、可演示、可真实验证"的状态，已经是目前能交付的最好结果。

如果要往下走，比较自然的几个方向是：接一个真正的后端来替代中转层、把判定规则做成可配置的规则集、扩展到更多语言与框架、把验真能力做成编辑器插件或 CI 环节。

---

## 附录 A：Worker 完整代码

部署位置：Cloudflare → Workers & Pages → `cd-relay` → Edit Code

```js
const corsHeaders = {
  "Access-Control-Allow-Origin": "*",
  "Access-Control-Allow-Methods": "POST, OPTIONS",
  "Access-Control-Allow-Headers": "Content-Type, Authorization",
  "Access-Control-Max-Age": "86400"
};

export default {
  async fetch(request, env, ctx) {
    // 1. 处理浏览器 OPTIONS 预检请求（解决 405 问题）
    if (request.method === "OPTIONS") {
      return new Response(null, { status: 204, headers: corsHeaders });
    }

    // 2. 仅允许 POST
    if (request.method !== "POST") {
      return new Response(JSON.stringify({ error: "Method not allowed. Use POST." }), {
        status: 405,
        headers: { ...corsHeaders, "Content-Type": "application/json" }
      });
    }

    let body;
    try { body = await request.json(); } catch {
      return new Response(JSON.stringify({ error: 'Invalid JSON' }), { status: 400, headers: corsHeaders });
    }

    let code = body.code || '';
    const apiKey = body.apiKey || '';

    if (!code || !apiKey) {
      return new Response(JSON.stringify({ error: 'missing code or apiKey' }), { status: 400, headers: corsHeaders });
    }

    // 3. 长代码智能截断（超过 500 字符易导致上游失败/空结果）
    let queryStr = code;
    if (code.length > 500) {
      queryStr = code.slice(0, 500) + "...(代码过长已截断)";
    }

    // 4. 调用腾讯云元宝 SearchPro
    const r = await fetch('https://api.wsa.cloud.tencent.com/SearchPro', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': 'Bearer ' + apiKey
      },
      body: JSON.stringify({
        Query: queryStr + ' 官方文档 API 是否存在 是否已弃用',
        Cnt: 10
      })
    });

    const text = await r.text();

    // 5. 501 兜底提示
    if (r.status === 501) {
      return new Response(JSON.stringify({
        error: "腾讯云 SearchPro 返回 501。请检查：1. Key 是否为腾讯云元宝搜索创建的 API Key；2. 服务是否已开通；3. 账号是否欠费。"
      }), {
        status: 501,
        headers: { ...corsHeaders, "Content-Type": "application/json" }
      });
    }

    // 6. 原样透传上游响应
    return new Response(text, {
      status: r.status,
      headers: { "Content-Type": "application/json", ...corsHeaders }
    });
  }
};
```

---


## 附录 B：曾评估但未采用的方案

| 方案 | 未采用原因 |
|------|-----------|
| 腾讯云云开发 CloudBase（Web 函数） | 免费体验版每个账号仅 1 次机会，希望保留额度 |
| 腾讯云 SCF 事件函数 + API 网关 | 同样需要云开发环境，且配置链路更长 |
| 前端直连元宝 API | 上游无 CORS 头，浏览器必然拦截 |

最终选择 **Cloudflare Worker**：免费额度充足、部署即改即生效、与 GitHub Pages 天然配套。

> 补充说明：该方案的短板是 `*.workers.dev` 二级域名在国内部分网络环境下可达性不稳定（见坑 7）。若部署环境对国内可达性要求较高，可替换为国内云厂商的函数计算服务，前端仅需修改 `DEFAULT_RELAY` 一个常量。

---

*CodeDetective · 用元宝搜索 API 构建 · WaytoAGI 1024 超能 Coding 周末 侦探赛道*
