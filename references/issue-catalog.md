# 前端问题分类码表

## 目录

1. 文档定位
2. 编号方案
3. 条目字段结构
4. 去重与合并规则
5. 码表条目
6. AI 使用流程
7. 扩展与维护

## 一、文档定位

本文是前端代码审查的**问题分类字典（码表）**。AI 在触发代码审查时，对照本表识别问题、分配唯一编号、映射级别、执行去重与合并。

- 评分规则、级别定义参见 [scoring-model.md](scoring-model.md)，报告输出参见 [report-generation.md](report-generation.md)，三文档配合使用。
- 每条问题拥有稳定 `id`（如 `SEC-001`），AI 命中后必须输出该 `id`，作为跨报告的稳定引用。
- 本表覆盖浏览器 Web、SSR/SSG、微前端、微信/支付宝等小程序、Hybrid/WebView、uni-app/Taro、Electron/Tauri/NW.js 桌面端，以及 Vue、React、Angular、Svelte、Solid、原生 JavaScript/TypeScript 和 React Native/Flutter 的前端 UI 层。BladeX/Saber、Element Plus/Element UI、Avue 仅在项目实际使用时启用对应规则。原生 iOS/Android 平台工程由移动端码表负责；后端专属问题（SQL 注入、事务、线程池、Redis 缓存等）不在本表范围。
- 本表与后端码表使用相同的条目字段结构、去重规则和 AI 输出格式；前缀相同的分类语义对齐，但条目在前后端各自独立编号，不跨仓库引用。

## 二、编号方案

| 前缀 | 分类 | 说明 |
| --- | --- | --- |
| SEC | 安全与漏洞 | XSS、凭证泄露、开放重定向、动态执行、鉴权绕过、上传下载 |
| PRV | 隐私与合规 | 脱敏、埋点上报、内部信息暴露 |
| BIZ | 业务逻辑正确性 | 条件、计算、金额精度、时间、边界、状态、字段映射 |
| ROB | 健壮性 | 空值、异常兜底、状态复位、超时取消、重试边界 |
| CON | 异步与竞态 | 请求竞态、重复提交、防抖节流、并行异步 |
| CMP | 组件与状态 | Props、key、响应式、副作用清理、Store、Avue 回调 |
| UI | 界面与可访问性 | 样式污染、页面状态、布局溢出、可访问性 |
| PERF | 性能 | 循环请求、重复请求、大列表、包体、资源泄漏 |
| LOG | 日志与调试 | 敏感输出、吞异常、调试残留 |
| CFG | 配置 | 硬编码、环境混用、开关残留、公共构建配置 |
| DEP | 依赖 | 已知 CVE、版本漂移、冗余依赖 |
| API | 接口调用 | 请求层复用、契约一致性、响应处理、分页 |
| QUAL | 代码质量 | 复制粘贴、死代码、复杂度、命名、注释 |
| TEST | 测试与验证 | 核心变更验证、纯逻辑测试覆盖 |

- 前端无线程级并发，`CON` 在本表定义为异步与竞态问题（请求时序、重复触发）。
- 编号一旦分配不再复用；新增条目末尾追加，废弃条目标记 `deprecated: true` 但保留编号。

平台专项条目沿用上述分类前缀，只有仓库实际命中对应平台边界时才启用。

## 三、条目字段结构

每条问题以列表项组织，字段固定如下（AI 输出时必须按同结构给出命中信息）：

```yaml
- id: SEC-002                                  # 唯一编号，稳定不变
  title: XSS / 危险 HTML 未净化                 # 中文标题
  level: 阻断                                  # 建议级别：阻断/严重/主要/次要/提示；AI 可依上下文在 ±1 级内调整
  rootCauseKey: SEC-002:xss-unsanitized-html   # 去重合并键，AI 必须输出
  signals:                                     # 识别线索，仅供参考，AI 可依上下文自行补充
    keywords: [ "v-html", "innerHTML", "dangerouslySetInnerHTML" ]
    whitelist: [ "DOMPurify.sanitize", "escapeHtml" ]
  description: 用户可控内容未净化直接渲染为 HTML，可执行任意脚本。
  fixHint: 默认插值渲染；富文本必须走白名单净化。
```

- **id**：`前缀-三位数字`，AI 输出的每个问题必须携带。
- **title**：中文，简短一句。
- **level**：建议级别，五档中文：阻断/严重/主要/次要/提示；与评分模型英文枚举一一对应（阻断=Blocker、严重=Critical、主要=Major、次要=Minor、提示=Info）。
- **rootCauseKey**：去重合并主键，格式 `<id>:<归一化根因特征>`；AI 必须输出。
- **signals**：识别线索，`regex` / `keywords` / `whitelist` 三选；**仅供参考**，AI 应结合语义判断，不得只做机械匹配。
- **description**：一句话说明 WHY。
- **fixHint**：一句修复方向（≤ 30 字）。
- **downgradeToInfo**（可选）：列出必须降级为“提示”且不扣分的证据不足场景；命中即按提示处理。
- **excludeFromReview**（可选）：列出默认不进入问题清单的业务边界场景；命中即不输出、不扣分。

## 四、去重与合并规则

AI 每次扫描按下列规则计算与合并，保证多次扫描同一份代码结果稳定。

### 4.1 三层键定义

```
rootCauseKey        = "<id>:<归一化根因特征>"   例：SEC-002:xss-unsanitized-html
locationFingerprint = <文件路径> + "#" + <组件名/函数名/代码块>   （不含行号）
descriptionNorm     = 去除变量名、行号、字面量后取前 8 位哈希
```

### 4.2 合并与扣分规则

| 编号 | 规则 | 说明 |
| --- | --- | --- |
| R1 | 根因合并 | 同一 `rootCauseKey` 只算 1 条问题；`locations[]` 记录所有 `locationFingerprint`，扣分不因数量线性叠加 |
| R2 | 影响面上浮 | 同一 `rootCauseKey` 命中位置 ≥ 3 处，可在该 id 建议级别范围内上浮 1 档（如 主要 → 严重），不得超过 id 允许上限 |
| R3 | 描述归一 | `rootCauseKey` 相同、`descriptionNorm` 不同的多次记录，视为同一问题，不重复扣分 |
| R4 | 多类别命中 | 同一位置同时命中多个 id，按最高级别 id 扣分，其他 id 仅列为「关联问题」，不再重复扣 |
| R5 | 扫描幂等 | 同一份代码重复扫描，`rootCauseKey` 集合应完全一致；不一致视为扫描不稳定，需回归 |
| R6 | 证据不足 | 无法明确判定的疑似项，标记为「待确认」，不进入扣分池 |
| R7 | 白名单豁免 | 命中条目 `signals.whitelist` 时不入问题池 |
| R8 | 级别冲突 | 同 id 在不同证据下级别不一致时，取该次扫描中的最高级别 |

### 4.3 AI 输出结构约定

AI 在报告中每个问题至少输出：

```yaml
id: CMP-004
rootCauseKey: CMP-004:side-effect-not-cleaned
level: 主要
mustFix: true
locations:
  - src/views/desk/notice.vue#onMounted
  - src/components/custom-chart/index.vue#initChart
occurrenceCount: 2
mergedFrom: 5           # 归一前的原始命中次数
evidence: <脱敏问题代码片段；mustFix=true 时必填>
fixHint: 卸载钩子成对清理；封装自动清理的组合式函数
```

- **mustFix**：是否必须修复；阻断、严重问题固定为 `true`，主要问题按实际门禁判断，次要和提示通常为 `false`。
- **evidence**：已确认问题的脱敏问题代码；`mustFix=true` 时必须包含精确位置对应的非空代码片段。

### 4.4 误报排除、降级与证据边界

- 命中安全类关键词但缺少上下文证据时，不得直接按阻断、严重或主要扣分；应先降级为“提示”或标记为“待确认”。
- 命中条目的 `downgradeToInfo` 时只按提示处理且不扣分；命中 `excludeFromReview` 时默认不进入问题清单，也不得写入问题摘要、阻断原因、必须修复项或复审要求。
- 只有能证明外部输入可达该代码路径，并可实际造成脚本执行、凭证泄露、越权访问或数据破坏时，才可按对应安全问题的建议级别扣分。
- 后端职责范围内的问题不在本表输出：接口是否具备服务端鉴权、SQL 与事务实现、服务端限流与缓存策略等，只有在前端实现本身形成可利用路径（如仅靠隐藏按钮当作鉴权）时才作为前端问题。
- 仅从命名、注释、目录结构或提交文案推断的“疑似”问题不进入扣分池；租户、组织、人员等业务边界字段的传递按业务规则处理，除用户明确要求业务建议外不作为安全问题输出。
- 重复扫描时，被降级或排除的判定必须保持一致；`rootCauseKey` 集合变化时须在报告中说明是新代码还是误报修正。

---

## 五、码表条目

### 5.1 SEC · 安全与漏洞

```yaml
- id: SEC-001
  title: 明文密钥/Token/密码硬编码进前端代码
  level: 阻断
  rootCauseKey: SEC-001:credential-hardcoded
  signals:
    regex:
      - (?i)(password|passwd|token|secret|access[_-]?key|api[_-]?key|client[_-]?secret)\s*[:=]\s*["'][^"'${}]{4,}["']
    keywords: [ "Authorization: Bearer ", "privateKey" ]
    whitelist: [ "import.meta.env", "process.env", "VITE_" ]
  description: 有明确证据证明生产真实凭证、高权限凭证或可直接调用付费/管理接口的密钥被硬编码，泄露即事故。
  fixHint: 凭证由后端下发或走环境变量；仓库不得保留明文。
  downgradeToInfo:
    - "设计上即公开的第三方前端 Key（地图 JS API、统计 SDK、客服组件等），且无证据表明可越权调用或产生资损。"
    - "mock 数据、示例配置、开发代理配置、本地调试账号，且未证明会进入生产构建产物。"
    - "仅凭字段名 token、secret、password 命中，但值为占位符、空串、常量枚举或校验提示文案。"

- id: SEC-002
  title: XSS / 危险 HTML 未净化
  level: 阻断
  rootCauseKey: SEC-002:xss-unsanitized-html
  signals:
    keywords: [ "v-html", "innerHTML", "insertAdjacentHTML", "document.write(", "dangerouslySetInnerHTML" ]
    whitelist: [ "DOMPurify.sanitize", "xss(", "escapeHtml" ]
  description: 用户可控内容未净化直接渲染为 HTML，可执行任意脚本。
  fixHint: 默认插值渲染；富文本必须 DOMPurify 白名单净化。

- id: SEC-003
  title: 动态代码执行（eval/new Function/字符串定时器）
  level: 严重
  rootCauseKey: SEC-003:dynamic-code-execution
  signals:
    keywords: [ "eval(", "new Function(", "setTimeout(\"", "setInterval(\"" ]
  description: 字符串进入 eval/new Function 等执行环境，可被注入任意代码。
  fixHint: 禁止 eval；配置驱动逻辑改用白名单映射表。

- id: SEC-004
  title: 开放重定向 / 不可信跳转
  level: 严重
  rootCauseKey: SEC-004:open-redirect
  signals:
    regex:
      - (location\.href|location\.replace|window\.open)\s*\(?[^;\n]*(query|params|redirect|returnUrl)
    keywords: [ "redirect", "returnUrl" ]
    whitelist: [ "域名白名单校验", "router.push({ name:" ]
  description: 跳转目标来自 URL 参数且未校验，可被钓鱼利用。
  fixHint: 跳转目标做域名白名单；站内跳转用命名路由。

- id: SEC-005
  title: postMessage 通信未校验来源
  level: 严重
  rootCauseKey: SEC-005:postmessage-origin-unchecked
  signals:
    regex:
      - postMessage\s*\([^)]*,\s*['"]\*['"]
    keywords: [ "addEventListener('message'", "window.postMessage" ]
    whitelist: [ "event.origin ===" ]
  description: 消息监听未校验 event.origin 或发送使用 *，可被任意页面伪造或窃取。
  fixHint: 收发都固定目标 origin；监听内校验 origin 白名单。

- id: SEC-006
  title: 仅前端隐藏当作鉴权 / 路由权限缺失
  level: 阻断
  rootCauseKey: SEC-006:frontend-only-authz
  signals:
    keywords: [ "v-if=\"permission", "v-show", "meta: { auth: false }" ]
  description: 按钮或路由只在前端隐藏、后端未确认鉴权，或路由绕过权限守卫。
  fixHint: 前端隐藏仅是体验层；敏感操作必须确认后端鉴权与权限编码。

- id: SEC-007
  title: 敏感数据明文落地 Web 存储
  level: 严重
  rootCauseKey: SEC-007:sensitive-in-web-storage
  signals:
    keywords: [ "localStorage.setItem", "sessionStorage.setItem", "document.cookie" ]
    whitelist: [ "项目既有 Token 存储封装", "加密存储工具" ]
  description: 密码、身份证、完整个人信息等明文写入本地存储，可被 XSS 窃取。
  fixHint: 敏感数据不落地；确需缓存走统一加密封装并设过期。

- id: SEC-008
  title: target="_blank" 缺少 noopener
  level: 次要
  rootCauseKey: SEC-008:target-blank-noopener
  signals:
    regex:
      - target="_blank"(?![^>]*rel=)
  description: 新窗口页面可通过 window.opener 反向控制来源页。
  fixHint: 外链统一加 rel="noopener noreferrer"。

- id: SEC-009
  title: 上传/下载绕过统一鉴权或缺少限制
  level: 严重
  rootCauseKey: SEC-009:upload-download-unrestricted
  signals:
    keywords: [ "el-upload", ":action=", "createObjectURL(" ]
    whitelist: [ "统一上传封装", "headers 携带 Token", "accept 与大小校验" ]
  description: 上传直连地址绕过拦截器，或未限制类型/大小；下载未校验归属。
  fixHint: 上传下载走统一封装带 Token；前端校验类型大小，后端二次校验。

- id: SEC-010
  title: URL 参数拼接未编码
  level: 主要
  rootCauseKey: SEC-010:url-param-unencoded
  signals:
    regex:
      - "[?&][A-Za-z0-9_]+=\\s*['\"]?\\s*\\+"
    keywords: [ "'?id=' +", "`?name=${" ]
    whitelist: [ "encodeURIComponent", "params:" ]
  description: 参数直接拼进 URL，特殊字符破坏请求语义或造成注入。
  fixHint: 用 params 传参；手工拼接必须 encodeURIComponent。
```

### 5.2 PRV · 隐私与合规

```yaml
- id: PRV-001
  title: 个人敏感信息未脱敏展示或导出
  level: 严重
  rootCauseKey: PRV-001:pii-not-masked
  signals:
    keywords: [ "phone", "idCard", "bankCard", "身份证", "手机号" ]
    whitelist: [ "maskPhone", "desensitize", "脱敏工具" ]
  description: 手机号、身份证、银行卡等在页面、表格、导出文件中未脱敏。
  fixHint: 展示统一脱敏工具；完整值仅在授权场景按需请求。

- id: PRV-002
  title: 埋点 / 监控 SDK 上报敏感数据
  level: 严重
  rootCauseKey: PRV-002:tracking-sensitive-data
  signals:
    keywords: [ "track(", "gtag(", "sensors.", "Sentry.setUser", "captureException" ]
  description: 埋点或异常监控把 Token、身份证、手机号等上报到第三方。
  fixHint: 上报前统一过滤敏感字段；监控 SDK 配置脱敏钩子。

- id: PRV-003
  title: 错误提示暴露内部信息
  level: 主要
  rootCauseKey: PRV-003:internal-info-exposed
  signals:
    keywords: [ "message.error(error", "$message.error(err", "error.stack" ]
    whitelist: [ "统一错误文案", "错误码映射" ]
  description: 接口地址、堆栈、内部结构直接透出到用户可见的错误提示。
  fixHint: 用户侧展示业务文案；技术细节仅进监控。
```

### 5.3 BIZ · 业务逻辑正确性

```yaml
- id: BIZ-001
  title: 条件分支错误
  level: 主要
  rootCauseKey: BIZ-001:branch-logic-error
  signals:
    regex:
      - if\s*\([^)]*==\s*null[^)]*&&
    keywords: [ "TODO", "FIXME" ]
  description: 逻辑运算符误用、空值判断顺序颠倒、分支缺失。
  fixHint: 明确前置条件；补齐 else 分支；用例覆盖每条分支。

- id: BIZ-002
  title: 金额 / 浮点精度错误
  level: 严重
  rootCauseKey: BIZ-002:money-precision
  signals:
    keywords: [ "* 100", "/ 100", "toFixed(", "parseFloat(" ]
    whitelist: [ "Big.js", "decimal.js", "currency.js", "分为单位整数运算" ]
  description: 金额用浮点直接运算或依赖 toFixed 舍入，出现 0.1+0.2 类误差。
  fixHint: 金额按分为单位整数运算或用 decimal 库；展示层再格式化。

- id: BIZ-003
  title: 时间 / 时区 / 日期边界错误
  level: 主要
  rootCauseKey: BIZ-003:time-timezone-boundary
  signals:
    keywords: [ "new Date('", "getTime()", "toISOString(" ]
    whitelist: [ "dayjs(", "统一日期工具" ]
  description: 字符串直接 new Date 存在浏览器差异；时区混用；区间闭开错误。
  fixHint: 统一 Dayjs 显式格式解析；跨时区以后端为准；区间左闭右开。

- id: BIZ-004
  title: 提交字段映射 / 取值错误
  level: 严重
  rootCauseKey: BIZ-004:payload-field-mapping
  signals:
    keywords: [ "params", "data", "form" ]
  description: 表单字段与接口契约不一致、单位或枚举取值错误，写入错误数据。
  fixHint: 对照接口文档逐字段核对名称、类型、单位与枚举。

- id: BIZ-005
  title: 边界 / 空数据处理错误
  level: 主要
  rootCauseKey: BIZ-005:boundary-error
  signals:
    regex:
      - \.length\s*-\s*1
    keywords: [ "[0]", "split(" ]
  description: off-by-one、空数组取下标、分页边界、除零未防护。
  fixHint: 空集合走占位分支；下标访问前判长度；边界值单独用例。

- id: BIZ-006
  title: 状态流转 / 操作可用性错误
  level: 主要
  rootCauseKey: BIZ-006:state-transition
  signals:
    keywords: [ "status", "state", "disabled" ]
  description: 未按前置状态控制操作可用性，允许非法流转或重复流转。
  fixHint: 状态与可用操作集中映射管理；操作前校验当前状态。

- id: BIZ-007
  title: 参数组合不合法未校验
  level: 主要
  rootCauseKey: BIZ-007:param-combination-invalid
  signals:
    keywords: [ "rules", "validator" ]
  description: 单字段合法但组合非法（如开始时间晚于结束时间）未做跨字段校验。
  fixHint: 表单增加跨字段 validator；提交前统一组合校验。

- id: BIZ-008
  title: 危险操作缺少二次确认
  level: 主要
  rootCauseKey: BIZ-008:dangerous-action-no-confirm
  signals:
    keywords: [ "handleDelete", "batchDelete", "remove(" ]
    whitelist: [ "$confirm", "ElMessageBox.confirm" ]
  description: 删除、批量变更等不可逆操作无确认，或确认取消后仍执行。
  fixHint: 危险操作统一确认弹窗；确认结果必须阻断后续调用。
```

### 5.4 ROB · 健壮性

```yaml
- id: ROB-001
  title: 空值 / undefined 访问
  level: 主要
  rootCauseKey: ROB-001:null-undefined-access
  signals:
    regex:
      - \.data\.\w+\.\w+
    keywords: [ "res.data.records", ".map(", ".filter(" ]
    whitelist: [ "?.", "??", "|| []" ]
  description: 深层取值和集合方法未防空，接口异常时页面白屏报错。
  fixHint: 深层取值用可选链加默认值；集合兜底空数组。

- id: ROB-002
  title: 异步异常无兜底
  level: 主要
  rootCauseKey: ROB-002:async-error-unhandled
  signals:
    keywords: [ "await ", ".then(" ]
    whitelist: [ "try {", ".catch(", "统一拦截器兜底" ]
  description: await 无 try/catch、Promise 链无 catch，失败后无提示且状态悬挂。
  fixHint: 异步链路必须有 catch 或统一拦截器兜底；失败给出可感知反馈。

- id: ROB-003
  title: 失败分支状态未复位
  level: 主要
  rootCauseKey: ROB-003:state-not-reset-on-failure
  signals:
    keywords: [ "loading = true", "done()", "this.loading" ]
    whitelist: [ "finally" ]
  description: loading、disabled、组件回调或异步状态在失败分支未复位，页面卡死。
  fixHint: 复位逻辑放 finally；组件回调成功失败都要收口状态。

- id: ROB-004
  title: 类型转换 / 隐式比较错误
  level: 主要
  rootCauseKey: ROB-004:type-coercion-error
  signals:
    keywords: [ "parseInt(", "== " ]
    whitelist: [ "===", "Number(", "parseInt(x, 10)" ]
  description: parseInt 缺基数、== 隐式转换、字符串数字直接比较大小。
  fixHint: 一律 ===；parseInt 显式基数；比较前显式转型。

- id: ROB-005
  title: 请求超时 / 取消缺失
  level: 主要
  rootCauseKey: ROB-005:timeout-cancel-missing
  signals:
    keywords: [ "axios", "fetch(" ]
    whitelist: [ "timeout:", "AbortController", "CancelToken" ]
  description: 长请求无超时，路由切换或组件卸载后未取消请求。
  fixHint: 统一请求层设默认超时；页面级请求随卸载取消。

- id: ROB-006
  title: 失败自动重试 / 轮询无边界
  level: 严重
  rootCauseKey: ROB-006:retry-unbounded
  signals:
    keywords: [ "retry", "setInterval(", "轮询" ]
    whitelist: [ "maxRetries", "指数退避", "clearInterval" ]
  description: 失败重试或轮询无最大次数与退避，形成请求风暴。
  fixHint: 重试设上限加退避；轮询失败即停并可手动恢复。

- id: ROB-007
  title: JSON / 存储解析未防错
  level: 次要
  rootCauseKey: ROB-007:unsafe-parse
  signals:
    keywords: [ "JSON.parse(" ]
    whitelist: [ "try {", "安全解析工具" ]
  description: JSON.parse 直接解析接口或本地存储内容，脏数据即抛错。
  fixHint: 解析包 try/catch 并给默认值；封装安全解析工具。
```

### 5.5 CON · 异步与竞态

```yaml
- id: CON-001
  title: 请求竞态覆盖（过期响应回写）
  level: 严重
  rootCauseKey: CON-001:stale-response-overwrite
  signals:
    keywords: [ "watch", "@change", "activeName" ]
    whitelist: [ "AbortController", "请求序号比对", "最新请求标记" ]
  description: 快速切换筛选或 Tab 时旧响应后到，覆盖新数据造成显示错乱。
  fixHint: 发新请求前取消旧请求，或以请求序号丢弃过期响应。

- id: CON-002
  title: 重复提交未防护
  level: 严重
  rootCauseKey: CON-002:duplicate-submit
  signals:
    keywords: [ "@click=\"submit", "handleSubmit" ]
    whitelist: [ "loading 锁", "disabled", "防抖" ]
  description: 提交按钮可连点，产生重复请求与重复数据。
  fixHint: 提交期间按钮 loading 禁用；必要时配合后端幂等键。

- id: CON-003
  title: 高频事件未防抖 / 节流
  level: 主要
  rootCauseKey: CON-003:debounce-throttle-missing
  signals:
    keywords: [ "addEventListener('scroll'", "@input=", "resize" ]
    whitelist: [ "debounce", "throttle" ]
  description: scroll/resize/input 直接触发请求或重计算，高频抖动。
  fixHint: 输入查询防抖、滚动节流；卸载时移除包装函数。

- id: CON-004
  title: 并行异步结果处理错误
  level: 主要
  rootCauseKey: CON-004:parallel-async-mishandled
  signals:
    keywords: [ "Promise.all(", "forEach(async" ]
    whitelist: [ "Promise.allSettled", "for...of + await" ]
  description: Promise.all 部分失败即整体中断未处理；forEach 内 async 不等待。
  fixHint: 可部分失败场景用 allSettled；顺序依赖用 for...of。
```

### 5.6 CMP · 组件与状态

```yaml
- id: CMP-001
  title: 直接修改 Props
  level: 主要
  rootCauseKey: CMP-001:props-mutation
  signals:
    regex:
      - props\.\w+\s*=[^=]
    whitelist: [ "emit('update:", "$emit('update:", "computed 代理" ]
  description: 子组件直接改 props，破坏单向数据流并触发告警。
  fixHint: 用 update: 事件让父组件改；本地副本用 data/computed。

- id: CMP-002
  title: v-for key 缺失或不稳定
  level: 主要
  rootCauseKey: CMP-002:vfor-unstable-key
  signals:
    regex:
      - v-for(?![^>]*:key)
    keywords: [ ":key=\"index\"" ]
  description: 无 key 或用下标作 key，增删排序后行状态与表单数据错位。
  fixHint: 用稳定业务主键作 key；无主键时生成稳定 uid。

- id: CMP-003
  title: watch/computed 循环更新或副作用
  level: 严重
  rootCauseKey: CMP-003:reactive-loop
  signals:
    keywords: [ "watch", "deep: true", "immediate: true" ]
  description: watch 内修改被监听源形成循环；computed 内发请求或改状态。
  fixHint: computed 保持纯函数；watch 改源必须有收敛条件。

- id: CMP-004
  title: 副作用未清理（监听 / 定时器 / Observer）
  level: 主要
  rootCauseKey: CMP-004:side-effect-not-cleaned
  signals:
    keywords: [ "addEventListener(", "setInterval(", "new ResizeObserver", "$on(" ]
    whitelist: [ "removeEventListener", "clearInterval", "onBeforeUnmount", "beforeDestroy", "disconnect()" ]
  description: 监听、定时器、Observer、事件总线订阅未随组件卸载清理，泄漏或跨页串扰。
  fixHint: 卸载钩子成对清理；封装自动清理的组合式函数。

- id: CMP-005
  title: 全局状态越界 / 重复维护
  level: 主要
  rootCauseKey: CMP-005:global-state-misuse
  signals:
    keywords: [ "$store.state", "useStore(" ]
    whitelist: [ "commit(", "dispatch(", "action 内修改" ]
  description: 组件直接改 store state，或同一状态多处重复维护（如登录态双份）。
  fixHint: 状态变更走 mutation/action；同一事实只留一个权威来源。

- id: CMP-006
  title: 组件/CRUD 回调未收口
  level: 主要
  rootCauseKey: CMP-006:avue-callback-not-closed
  signals:
    keywords: [ "rowSave", "rowUpdate", "rowDel", "done", "loading" ]
  description: 组件或 CRUD 回调部分分支未结束 loading/done，或配置与视图契约不符。
  fixHint: 成功失败分支都收口状态；配置对照项目契约。

- id: CMP-007
  title: 响应式失效写法
  level: 主要
  rootCauseKey: CMP-007:reactivity-broken
  signals:
    regex:
      - const\s*\{[^}]+\}\s*=\s*(reactive|props)\b
    keywords: [ "Object.assign(this." ]
    whitelist: [ "$set", "toRefs", "computed(" ]
  description: Vue2 对象新增属性/数组下标赋值不触发更新；解构 reactive/props 丢失响应式。
  fixHint: Vue2 用 $set 或整体替换；Vue3 解构用 toRefs/computed。

- id: CMP-008
  title: 生命周期误用
  level: 次要
  rootCauseKey: CMP-008:lifecycle-misuse
  signals:
    keywords: [ "created", "mounted", "activated", "document." ]
  description: created 内操作 DOM、重复初始化、keep-alive 页面未用 activated 刷新。
  fixHint: DOM 操作放 mounted 之后；keep-alive 数据刷新走 activated。
```

### 5.7 UI · 界面与可访问性

```yaml
- id: UI-001
  title: 全局样式污染
  level: 主要
  rootCauseKey: UI-001:global-style-pollution
  signals:
    regex:
      - ^\s*\.el-
      - ^\s*\.avue-
    keywords: [ "!important" ]
    whitelist: [ "scoped", ":deep(", "页面级前缀类" ]
  description: 无隔离或无前缀直接覆盖组件库/宿主页面样式，其他页面或微前端应用回归。
  fixHint: 使用 scoped、CSS Modules 或业务前缀；避免全局覆盖。

- id: UI-002
  title: 页面状态缺失（loading/empty/error/无权限）
  level: 主要
  rootCauseKey: UI-002:page-state-missing
  signals:
    keywords: [ "v-loading", "el-empty" ]
  description: 缺 loading、empty、error、无权限等必要状态，异常时页面空白。
  fixHint: 列表与详情补齐四态；异常给出重试入口。

- id: UI-003
  title: 布局溢出 / 遮挡
  level: 主要
  rootCauseKey: UI-003:layout-overflow
  signals:
    keywords: [ "position: fixed", "overflow: hidden", "width: 19" ]
  description: 目标视口出现遮挡、裁剪、表格撑破、关键操作不可见。
  fixHint: 目标视口实测；表格容器限宽滚动；长文本换行省略。

- id: UI-004
  title: 可访问性缺失
  level: 次要
  rootCauseKey: UI-004:accessibility-missing
  signals:
    regex:
      - <img(?![^>]*alt=)
    keywords: [ "placeholder 当 label" ]
  description: 表单缺可识别标签、图片无 alt、纯键盘无法完成主要操作。
  fixHint: 表单项配 label；图片补 alt；关键操作可聚焦可回车。

- id: UI-005
  title: 平台样式规则误用
  level: 次要
  rootCauseKey: UI-005:mobile-rules-in-desktop
  signals:
    keywords: [ "env(safe-area", "rem", "vw", "rpx", "wxss", "-webkit-app-region" ]
  description: 某一运行形态无必要引入另一端专属单位、窗口或安全区方案，布局异常。
  fixHint: 按目标端选择单位和适配策略；不要跨端机械复制样式。
```

### 5.8 PERF · 性能

```yaml
- id: PERF-001
  title: 循环内逐条请求
  level: 主要
  rootCauseKey: PERF-001:request-in-loop
  signals:
    keywords: [ "forEach(async", "map(async", "for" ]
  description: 循环内逐条调用接口，延迟叠加且压垮后端。
  fixHint: 改批量接口；确需并行用受控并发。

- id: PERF-002
  title: 重复请求未合并 / 未缓存
  level: 主要
  rootCauseKey: PERF-002:duplicate-request
  signals:
    keywords: [ "getDict", "字典", "onMounted" ]
  description: 同一数据多组件重复拉取，字典、枚举等公共数据无缓存。
  fixHint: 公共数据入 store 或带缓存的组合式函数；同参请求合并。

- id: PERF-003
  title: 大列表无分页 / 虚拟化
  level: 主要
  rootCauseKey: PERF-003:large-list-unbounded
  signals:
    keywords: [ "size: 9999", "pageSize: 10000", "v-for" ]
  description: 一次拉取渲染全量数据，滚动卡顿甚至崩溃。
  fixHint: 接口分页；确需全量用虚拟滚动。

- id: PERF-004
  title: 大依赖整包引入 / 未懒加载
  level: 主要
  rootCauseKey: PERF-004:heavy-import
  signals:
    keywords: [ "import echarts from", "import _ from 'lodash'", "import * as" ]
    whitelist: [ "() => import(", "按需引入", "lodash-es" ]
  description: 整包引入重依赖、路由未懒加载，首屏体积显著膨胀。
  fixHint: 依赖按需引入；路由组件懒加载；构建产物分析确认。

- id: PERF-005
  title: 内存 / 资源泄漏
  level: 主要
  rootCauseKey: PERF-005:resource-leak
  signals:
    keywords: [ "URL.createObjectURL(", "new Blob(", "keep-alive" ]
    whitelist: [ "revokeObjectURL" ]
  description: Object URL 未释放、大对象常驻、keep-alive 滥用致内存持续增长。
  fixHint: createObjectURL 用后 revoke；keep-alive 白名单控制。

- id: PERF-006
  title: 渲染抖动 / 不必要重渲染
  level: 次要
  rootCauseKey: PERF-006:render-thrash
  signals:
    keywords: [ "deep: true", "JSON.parse(JSON.stringify(" ]
  description: 深度监听大对象、频繁整体替换大数据结构，触发大范围重渲染。
  fixHint: 监听精确路径；大结构增量更新。

- id: PERF-007
  title: 静态资源未优化
  level: 次要
  rootCauseKey: PERF-007:asset-unoptimized
  signals:
    keywords: [ ".png", ".jpg", "base64" ]
    whitelist: [ "loading=\"lazy\"", "压缩产物" ]
  description: 大图未压缩、未懒加载，或大资源内联 base64。
  fixHint: 图片压缩与懒加载；大资源走 CDN 或静态目录。
```

### 5.9 LOG · 日志与调试

```yaml
- id: LOG-001
  title: 敏感信息输出到控制台 / 日志
  level: 严重
  rootCauseKey: LOG-001:sensitive-in-console
  signals:
    regex:
      - console\.(log|info|warn|error)\s*\([^)]*(token|password|idCard|phone|secret)
    keywords: [ "console.log(res" ]
  description: 控制台打印 Token、密码、身份证、完整响应等敏感内容。
  fixHint: 删除敏感打印；必要日志走统一封装并脱敏。

- id: LOG-002
  title: 异常被吞
  level: 主要
  rootCauseKey: LOG-002:swallow-exception
  signals:
    regex:
      - catch\s*(\([^)]*\))?\s*\{\s*\}
    keywords: [ "catch (e) {}", "catch {}" ]
  description: 空 catch 或 catch 后不提示不上报，失败被静默吞掉。
  fixHint: catch 内至少提示用户或上报监控；不可恢复的继续抛出。

- id: LOG-003
  title: 调试代码残留
  level: 次要
  rootCauseKey: LOG-003:debug-code-left
  signals:
    keywords: [ "console.log(", "debugger", "alert(" ]
    whitelist: [ "构建剔除配置", "统一日志封装" ]
  description: console.log、debugger、alert 残留到提交代码。
  fixHint: 提交前清理；构建配置生产环境剔除 console。
```

### 5.10 CFG · 配置

```yaml
- id: CFG-001
  title: 接口地址 / 环境配置硬编码
  level: 主要
  rootCauseKey: CFG-001:config-hardcoded
  signals:
    regex:
      - "http[s]?://[a-zA-Z0-9.-]+(:\\d+)?"
    keywords: [ "localhost:", "127.0.0.1", "192.168." ]
    whitelist: [ "import.meta.env", "process.env", "VITE_" ]
  description: 接口地址、端口、第三方地址硬编码进源码。
  fixHint: 收敛到 .env 与环境变量；按环境隔离。

- id: CFG-002
  title: 环境判断散落业务代码
  level: 严重
  rootCauseKey: CFG-002:env-config-mixed
  signals:
    keywords: [ "NODE_ENV === ", "import.meta.env.PROD", "if (env" ]
    whitelist: [ "构建配置内环境分支", "统一环境工具" ]
  description: 业务代码按环境走不同逻辑，或测试地址混入生产配置。
  fixHint: 环境差异收敛到配置层；业务代码不感知环境。

- id: CFG-003
  title: Mock / 调试开关残留
  level: 主要
  rootCauseKey: CFG-003:debug-switch-left
  signals:
    keywords: [ "mock: true", "if (true)", "// debug", "VITE_MOCK" ]
  description: mock 开关、临时强制分支、调试代理指向残留到提交。
  fixHint: 上线前关闭 mock；开关走环境变量并默认关闭。

- id: CFG-004
  title: 公共构建 / 工程配置破坏性修改
  level: 主要
  rootCauseKey: CFG-004:build-config-broken
  signals:
    keywords: [ "vite.config", "vue.config", "lintOnSave: false", "eslint-disable" ]
  description: 为个别页面关闭 lint、改公共构建配置或代理，影响全工程。
  fixHint: 公共配置改动需说明影响面；局部问题局部解决。
```

### 5.11 DEP · 依赖

```yaml
- id: DEP-001
  title: 已知漏洞版本依赖
  level: 严重
  rootCauseKey: DEP-001:vulnerable-dependency
  signals:
    keywords: [ "npm audit", "已知 CVE 版本" ]
  description: 依赖版本存在公开 CVE 且社区已发布修复版。
  fixHint: 升级到修复版；不可升级时评估替代或缓解方案。

- id: DEP-002
  title: 版本漂移 / 锁文件缺失
  level: 主要
  rootCauseKey: DEP-002:version-drift
  signals:
    keywords: [ "\"latest\"", "\"*\"" ]
    whitelist: [ "锁文件已提交" ]
  description: 依赖用 latest/*，或锁文件未提交、与 package.json 不一致，构建不可复现。
  fixHint: 固定或收敛版本范围；锁文件必须提交并保持一致。

- id: DEP-003
  title: 冗余 / 重复依赖
  level: 次要
  rootCauseKey: DEP-003:redundant-dependency
  signals:
    keywords: [ "moment", "dayjs", "lodash", "lodash-es" ]
  description: 同类库并存（如 moment+dayjs）、引入未使用依赖。
  fixHint: 同类库收敛为项目既定一个；定期清理未用依赖。
```

### 5.12 API · 接口调用

```yaml
- id: API-001
  title: 绕过统一请求层
  level: 主要
  rootCauseKey: API-001:bypass-request-layer
  signals:
    keywords: [ "axios.create(", "new XMLHttpRequest", "fetch(" ]
    whitelist: [ "@/axios", "项目统一请求封装" ]
  description: 自建 axios 实例或 fetch 直连，绕过 Token、租户、错误处理拦截器。
  fixHint: 一律走项目统一请求封装；特殊需求在封装层扩展。

- id: API-002
  title: URL/method/params 与契约不一致
  level: 严重
  rootCauseKey: API-002:contract-mismatch
  signals:
    keywords: [ "method:", "params:", "data:" ]
  description: 请求方式、参数位置（params/data）、字段与接口文档不一致。
  fixHint: 对照接口文档逐项核对；GET 用 params、体参用 data。

- id: API-003
  title: 响应层级 / 错误码处理错误
  level: 主要
  rootCauseKey: API-003:response-handling-error
  signals:
    keywords: [ "res.data.data", "code === 200", "success" ]
  description: 响应取值层级错误、未按业务 code 判定成功、把 HTTP 200 当业务成功。
  fixHint: 按项目响应结构统一取值；成功判定沿用既有约定。

- id: API-004
  title: 列表接口分页参数缺失
  level: 主要
  rootCauseKey: API-004:pagination-missing
  signals:
    keywords: [ "current", "size", "page" ]
  description: 列表查询不传分页参数，或把 size 设为极大值变相全量。
  fixHint: 列表一律分页；全量需求走导出或专用接口。

- id: API-005
  title: 接口地址 / 权限编码靠猜测
  level: 主要
  rootCauseKey: API-005:api-guessed
  signals:
    keywords: [ "/api/", "permission" ]
  description: 未核对接口文档或菜单配置，凭经验编造 URL、参数或按钮权限编码。
  fixHint: 以接口文档与菜单配置为准；无文档先向后端确认。
```

### 5.13 QUAL · 代码质量

```yaml
- id: QUAL-001
  title: 大段复制粘贴
  level: 主要
  rootCauseKey: QUAL-001:copy-paste-code
  signals:
    keywords: [ "// copy", "同上" ]
  description: 多处代码高度相似，修改遗漏导致行为分叉。
  fixHint: 抽公共函数、组合式函数或公共组件。

- id: QUAL-002
  title: 死代码 / 注释代码块
  level: 次要
  rootCauseKey: QUAL-002:dead-code
  signals:
    keywords: [ "// eslint-disable", "unused" ]
  description: 未使用的组件、函数、变量、成块注释代码长期留存。
  fixHint: 直接删除，历史靠 Git 追溯。

- id: QUAL-003
  title: 上帝组件 / 超长函数
  level: 主要
  rootCauseKey: QUAL-003:god-component
  signals:
    keywords: [ "export default", "methods" ]
  description: 单文件组件职责混杂、体量过大（> 800 行）或单函数过长（> 80 行）。
  fixHint: 按职责拆子组件或组合式函数；模板逻辑下沉 computed。

- id: QUAL-004
  title: 圈复杂度过高
  level: 主要
  rootCauseKey: QUAL-004:high-complexity
  signals:
    keywords: [ "if", "else if", "switch" ]
  description: 深层嵌套 if、三元链，单函数圈复杂度过高难测试。
  fixHint: 卫语句提前返回；分支表或策略映射替代深嵌套。

- id: QUAL-005
  title: 魔法值 / 硬编码字面量
  level: 次要
  rootCauseKey: QUAL-005:magic-number
  signals:
    regex:
      - ===\s*\d{2,}
    keywords: [ "status === 1" ]
  description: 状态码、类型值等业务字面量散落各处，含义不清。
  fixHint: 提炼常量或枚举并集中管理。

- id: QUAL-006
  title: 命名 / 语义模糊
  level: 次要
  rootCauseKey: QUAL-006:naming-ambiguous
  signals:
    regex:
      - \b(temp|tmp|obj|info|flag|foo|bar|test)\d?\b
  description: 变量、函数、组件使用无语义命名。
  fixHint: 命名体现业务含义；遵循项目命名规范。

- id: QUAL-007
  title: 复杂逻辑缺少必要注释
  level: 次要
  rootCauseKey: QUAL-007:comment-missing-stale
  signals:
    keywords: [ "reduce(", "正则", "位运算" ]
  description: 复杂业务、兼容分支、框架桥接缺少说明，或注释与实现不符。
  fixHint: 补充业务含义与关键分支注释；失效注释同步删除。
```

### 5.14 TEST · 测试与验证

```yaml
- id: TEST-001
  title: 核心变更缺少验证
  level: 主要
  rootCauseKey: TEST-001:core-change-unverified
  signals:
    keywords: [ "router", "store", "axios", "permission" ]
  description: 入口、请求层、路由权限、全局状态等核心设施改动未运行项目已有校验或定向回归。
  fixHint: 按项目脚本执行 lint/typecheck/test/build 与定向回归。

- id: TEST-002
  title: 复杂纯逻辑无测试覆盖
  level: 次要
  rootCauseKey: TEST-002:pure-logic-untested
  signals:
    keywords: [ "utils", "工具函数" ]
  description: 新增可单测的纯逻辑（计算、转换、校验）无任何测试。
  fixHint: 纯函数补单测覆盖主路径与边界。
```

### 5.15 平台与运行形态专项

```yaml
- id: SEC-011
  title: 桌面端 IPC / Bridge 信任边界缺失
  level: 严重
  rootCauseKey: SEC-011:desktop-bridge-trust-boundary
  signals:
    keywords: [ "ipcMain.handle", "ipcRenderer", "postMessage", "WebViewJavascriptBridge" ]
    whitelist: [ "通道白名单", "来源校验", "参数 schema 校验" ]
  description: 渲染层可直接调用任意主进程能力，或 Bridge 消息未校验来源、通道和参数。
  fixHint: 通过 preload 暴露最小白名单 API，并校验来源、参数和返回错误。

- id: SEC-012
  title: 桌面协议 / 外部导航可控
  level: 严重
  rootCauseKey: SEC-012:desktop-external-navigation
  signals:
    keywords: [ "shell.openExternal", "window.open", "setWindowOpenHandler", "file://" ]
    whitelist: [ "固定协议白名单", "受控外部链接" ]
  description: 用户输入可驱动外部 URL、协议、文件路径或新窗口，形成钓鱼、命令或本地文件风险。
  fixHint: 仅允许固定协议和来源；对路径、窗口和外部打开做显式确认。

- id: CFG-005
  title: 桌面壳安全配置削弱
  level: 主要
  rootCauseKey: CFG-005:desktop-web-preferences-weakened
  signals:
    keywords: [ "nodeIntegration: true", "contextIsolation: false", "webSecurity: false", "enableRemoteModule" ]
    whitelist: [ "隔离的受控本地页面", "安全评审记录" ]
  description: Electron/Tauri 等桌面壳关闭进程隔离、启用不必要 Node 能力或放宽 Web 安全策略。
  fixHint: 默认开启隔离与安全策略；能力通过最小 preload/命令白名单暴露。

- id: API-006
  title: 平台 API 权限 / 域名约束缺失
  level: 主要
  rootCauseKey: API-006:platform-permission-boundary
  signals:
    keywords: [ "wx.request", "uni.request", "授权", "request合法域名", "permissions" ]
    whitelist: [ "项目权限清单", "运行时拒绝处理", "域名白名单" ]
  description: 小程序、Hybrid 或桌面能力调用未声明、未校验或未处理用户拒绝、域名和权限边界。
  fixHint: 对照平台配置与权限清单；拒绝、过期和非白名单目标都要可恢复。

- id: CMP-009
  title: 平台生命周期 / 状态恢复错误
  level: 主要
  rootCauseKey: CMP-009:platform-lifecycle-mismatch
  signals:
    keywords: [ "onLaunch", "onShow", "onHide", "onUnload", "visibilitychange", "app.on('activate'" ]
    whitelist: [ "生命周期设计说明", "取消或恢复策略" ]
  description: 页面、窗口或小程序生命周期切换时重复初始化、状态丢失、回调落到已销毁实例或资源未释放。
  fixHint: 按目标端生命周期建模；初始化幂等并在隐藏/卸载时取消和清理。

- id: ROB-008
  title: 跨端能力无降级 / 弱网恢复
  level: 主要
  rootCauseKey: ROB-008:cross-platform-fallback-missing
  signals:
    keywords: [ "#ifdef", "process.platform", "Platform.OS", "navigator.onLine", "bridge unavailable" ]
    whitelist: [ "明确不支持矩阵", "功能降级提示" ]
  description: 平台能力、网络或原生桥不可用时直接失败，缺少探测、降级、重试、取消或用户可感知反馈。
  fixHint: 建立能力矩阵；失败时提供降级路径、重试和可解释提示。

- id: PERF-008
  title: 小程序包体 / 桌面资源无界增长
  level: 主要
  rootCauseKey: PERF-008:platform-bundle-unbounded
  signals:
    keywords: [ "subPackages", "分包", "extraResources", "asarUnpack", "bundleAnalyzer" ]
    whitelist: [ "包体预算", "按需分包", "资源清单" ]
  description: 分包、预加载、原生资源或桌面打包内容无预算约束，导致安装包、首屏或平台限制超标。
  fixHint: 建立目标端包体预算；按路由和能力拆分并持续检查产物。

- id: UI-006
  title: 端特有输入与辅助功能缺失
  level: 次要
  rootCauseKey: UI-006:platform-input-accessibility
  signals:
    keywords: [ "键盘快捷键", "safe-area-inset", "软键盘", "触控", "Dynamic Type", "字体缩放" ]
    whitelist: [ "已定义设备/视口矩阵", "等价替代操作" ]
  description: 桌面键盘、小程序触控/软键盘、WebView 安全区或跨端字体缩放导致关键操作不可达。
  fixHint: 按目标端输入和视口矩阵验证焦点、触控、安全区和缩放。
```

---

## 六、AI 使用流程

AI 触发代码审查时，按以下步骤对照本表：

1. **对照识别**：扫描代码文件，逐段与本表 signals 对照，命中即候选。
2. **归一根因**：为每个候选计算 `rootCauseKey`，同键合并。
3. **计算位置指纹**：为每个 `rootCauseKey` 汇总 `locations[]`。
4. **描述归一去重**：对同键多描述做 `descriptionNorm` 归一，取最详尽一条作为主描述。
5. **级别裁定**：按本表 `level` 建议，可依上下文 ±1 级调整，多类别命中同位置按最高级别扣分（R4）。
6. **影响面上浮**：`locations` ≥ 3 处时按 R2 上浮一档。
7. **输出**：按 [scoring-model.md](scoring-model.md) 计分并按 [report-generation.md](report-generation.md) 结构输出，每条问题携带 `id / rootCauseKey / level / mustFix / locations / mergedFrom / evidence / fixHint`；所有已确认问题展示脱敏问题代码，`mustFix=true` 时不得省略。
8. **幂等校验**：重复扫描时对比上次 `rootCauseKey` 集合，不一致必须给出差异说明（新代码 / 误报修正）。

## 七、扩展与维护

- **新增条目**：末尾追加，编号连续，不复用已废弃编号；本表未收录的问题按临近分类临时定级，并在报告中说明属于码表外新增。
- **废弃条目**：保留编号并标 `deprecated: true`，说明原因与替代 id。
- **级别调整**：以本表 `level` 为建议基线；某项目/团队约定不同的，在项目根 `AGENTS.md` 或 `.codex/rules` 里做覆盖说明，不直接改本文件。
- **命中率反馈**：AI 生成报告时应记录本次命中的 id 集合，用于后续调整 signals 与级别。
