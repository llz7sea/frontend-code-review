# 前端项目形态识别与专项检查

本文件只负责识别项目运行形态和选择检查范围。最终问题仍必须使用 `issue-catalog.md` 的编号、`scoring-model.md` 的级别和唯一 HTML 模板输出。

## 识别原则

1. 先读取根目录及目标目录的 `AGENTS.md`、`package.json`、锁文件、构建配置、入口文件和测试脚本。
2. 以多个相互印证的证据确认形态：依赖、目录、脚本、配置和入口至少命中两类；单个包名只能作为候选信号。
3. 一个仓库可以同时命中多个形态。按代码边界分别检查，并在 `reviewScope`、`reviewTarget` 和 `reviewBasis` 中记录实际纳入的层。
4. 未命中的框架规则不产生扣分项；无法确认的平台行为标记为“待确认”，不凭经验判定缺陷。

## 形态信号与检查入口

| 形态 | 典型证据（示例，不是固定白名单） | 启用的专项检查 |
| --- | --- | --- |
| 浏览器 Web / SPA | `index.html`、`src/main.*`、Vite/Webpack/Rspack、React/Vue/Angular/Svelte/Solid 依赖 | 路由、渲染状态、浏览器兼容、键盘/焦点、资源与包体 |
| SSR / SSG / 微前端 | Next/Nuxt/Remix/Astro/SvelteKit、服务端入口、Module Federation、`server/` | hydration、服务端数据注入、跨应用依赖/样式隔离、边界错误页 |
| 小程序 | 微信开发者工具配置、`app.json`、`project.config.*`、`pages/`、`miniprogram`、`uni-app`/`Taro`脚本 | 生命周期、`setData`/编译边界、分包包体、域名白名单、平台权限、分享/订阅消息、真机差异 |
| Hybrid / WebView | WebView 容器、JSBridge、Cordova/Capacitor、`android/`/`ios/` 壳与 Web 目录并存 | Bridge 能力探测、消息来源校验、超时/取消、原生降级、深链、离线与平台分支 |
| 桌面端 | Electron/Tauri/NW.js 依赖、`main`/`preload`、`electron-builder`、Rust 壳、窗口入口 | 主/渲染进程边界、IPC 白名单、`contextIsolation`、`nodeIntegration`、协议/文件路径、更新/签名、窗口与崩溃恢复 |
| 编译型跨端 UI | uni-app、Taro、React Native、Flutter、跨端路由/平台目录与多目标构建脚本 | 平台分支、能力矩阵、原生模块边界、降级策略、构建目标和端间状态一致性 |
| 原生移动端混合仓库 | Swift/Objective-C/Kotlin/Java、Xcode/Gradle 工程与前端目录并存 | 前端层按本 Skill；原生层转交 `mobile-code-review-score` 或拆分范围审查 |

## 形态无关的证据要求

- 检查真实入口到页面、窗口或小程序页面的调用链，包含 loading、empty、error、无权限、取消、重复触发和恢复路径。
- 检查统一请求层、认证凭据、存储、路由/导航、全局状态、日志和错误上报；Electron 额外检查 IPC 通道和文件系统能力，小程序额外检查平台 API 授权与请求域名。
- 构建通过只证明对应构建目标可编译，不等价于浏览器、开发者工具、真机、桌面打包产物或原生桥接运行通过；报告必须区分静态、构建、自动化和运行态证据。
- 对 SSR、微前端、桌面壳和跨端编译产生的生成文件，按实际业务逻辑与生成器边界排除，不能整仓库忽略。

