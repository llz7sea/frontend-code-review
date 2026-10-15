# 前端代码审查评分

这是一个面向前端及跨端 UI 仓库的代码审查 Skill。它根据代码与 Git 证据进行审查和 100 分制评分，并使用仓库内置的 HTML 模板生成完整审查报告。

## 使用方式

在支持该 Skill 的 Copilot 环境中，明确调用：

```text
$frontend-code-review-score
```

也可以说明审查范围，例如：

```text
$frontend-code-review-score 审查当前修改，包含暂存和未暂存内容
```

Skill 默认只读审查代码。只有同时明确要求修复时，才会在审查后修改代码并重新验证。报告生成成功后，回复只包含生成状态和报告的绝对路径。

## 支持范围

- 浏览器 Web、SPA、SSR/SSG 和微前端
- 微信、支付宝等小程序，以及 uni-app、Taro 等编译型跨端项目
- Hybrid/WebView、Electron、Tauri 等桌面端和跨端 UI
- Vue、React、Angular、Svelte、Solid、原生 JavaScript/TypeScript 等技术栈

项目形态根据依赖、目录、配置和入口等实际证据识别；未使用的框架规则不会作为缺陷扣分。原生 iOS/Android 平台工程不属于本 Skill 的主要审查范围。

## 仓库内容

| 路径 | 用途 |
| --- | --- |
| `SKILL.md` | Skill 触发条件、审查流程、评分和输出要求 |
| `templates/CODE_REVIEW_REPORT_TEMPLATE.html` | 唯一允许使用的完整 HTML 报告模板 |
| `references/project-modes.md` | 项目形态识别与平台专项检查 |
| `references/issue-catalog.md` | 前端问题分类码表与根因归并规则 |
| `references/scoring-model.md` | 评分维度、扣分、风险封顶和团队评分 |
| `references/ai-coding-rate.md` | 基于代码特征估算 AI 编码率的方法 |
| `references/report-generation.md` | 模板变量、六章结构及隐藏 JSON 严格合同 |

## 报告生成物

默认将报告保存至被审查项目根目录下：

```text
review-output/frontend-code-review-report-YYYYMMDD-HHmmss.html
```

用户指定输出位置时优先使用指定位置；除非用户明确授权，不覆盖已有报告。报告是独立 HTML 文件，包含完整样式、审查数据及可交互的问题展示。顶层章节固定为以下六章：

1. 审查概览
2. 提交日志
3. 问题清单
4. 问题详情与脱敏代码片段
5. 问题跟踪
6. 结论与后续动作

### 模板生成物示例（结构节选）

下面是示意性片段，用来展示生成报告中的内容形态；示例数据为虚构数据，并非一次真实审查的结论。实际报告使用模板中的完整布局、样式、统计表和符合严格合同的 `reviewDataJson`，而不是此节选。

```html
<div class="wrap">
  <header class="title score-good">
    <h1>前端代码审查报告 - 订单页修改｜需修改</h1>
    <div class="sub">
      审查范围：订单列表与详情页的当前修改<br>
      结论：发现示例问题，建议修复后复审
    </div>
  </header>

  <section class="card">
    <h2>一、审查概览</h2>
    <p>得分：86 ｜ 问题数量：1 ｜ 必须修复：0</p>
    <p>分支、审查依据、目录树及 AI 编码率说明</p>
  </section>

  <section class="card">
    <h2>二、提交日志</h2>
    <table>
      <thead><tr><th>人员</th><th>提交数</th><th>代码变更统计</th><th>代码总量占比</th><th>AI 编码率</th></tr></thead>
      <tbody><tr><td>示例人员</td><td>1</td><td>新增 / 删除行数与有效文件数</td><td>示例比例</td><td>示例比例与置信度</td></tr></tbody>
    </table>
  </section>

  <section class="card">
    <h2>三、问题清单</h2>
    <table>
      <thead><tr><th>编号</th><th>级别</th><th>扣分</th><th>人员</th><th>提交</th><th>代码位置</th><th>是否必须修复</th><th>问题摘要</th><th>操作</th></tr></thead>
      <tbody>
        <tr>
          <td>示例编号</td><td>次要</td><td>示例扣分</td><td>示例人员</td>
          <td>工作区修改</td><td>src/pages/orders.tsx:42</td><td>否</td><td>示例问题摘要</td><td>编辑 / 删除</td>
        </tr>
      </tbody>
    </table>
  </section>

  <section class="card"><h2>四、问题详情与脱敏代码片段</h2>…</section>
  <section class="card"><h2>五、问题跟踪</h2>…</section>
  <section class="card"><h2>六、结论与后续动作</h2>…</section>
</div>
```

打开生成的 `.html` 文件即可查看报告。完整报告还会在模板末尾嵌入与可见内容一致、满足严格字段合同的 `reviewDataJson`。章节、模板变量、问题代码证据、动态内容转义和隐藏 JSON 的完整要求见 [`references/report-generation.md`](references/report-generation.md)。
