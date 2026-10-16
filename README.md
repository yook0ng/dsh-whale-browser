# dsh-whale-browser · 小鲸鱼的浏览器

> 给 DeepSeek Harness 的 agent 一双手：零依赖 CDP 直驱一个**专用 Chromium**，
> 像人一样读网页、点按钮、填表单、滚动、截图——每次操作先问用户，密码由用户自己输入。
>
> *Zero-dependency Chrome DevTools Protocol driver that gives a DSH agent a real browser:
> read pages, click, type, scroll and screenshot with genuine mouse/keyboard events,
> approval-gated per action, passwords never filled by the model.*

---

## 这是什么（What it does）

`dsh-whale-browser` 是一个 DeepSeek Harness（DSH）**宿主侧插件**，它给模型接上一台"自己的浏览器"。

DSH 的 agent 本来只能靠 `web_fetch` 拿静态 HTML，遇到需要登录、需要点按、需要翻页的网页就无能为力。
本插件启动一个**独立窗口的 Chromium**（Chrome / Edge 皆可，专用 `--user-data-dir`，
和用户日常浏览器的登录态、插件、书签完全隔离），然后通过 **Chrome DevTools Protocol** 直接操作它：

- **像人一样操作**：点击、输入、按键、滚动走的是 CDP 的 `Input.dispatchMouseEvent` /
  `Input.dispatchKeyEvent`——真实的鼠标键盘事件，页面看到的是人在操作，而不是脚本注入 DOM。
- **看得见**：`browser_screenshot` 把页面截图作为图片直接交给模型，模型能"看着"页面核对结果。
- **读得懂**：`browser_read` 同时返回正文文字和一张**可交互元素表**——每个可点的按钮/链接/输入框
  带 `ref`（如 `e7`）、类型、名称、视口坐标，还会标出风险（🔒 密码框 / ⚠ 删除类 / ⚠ 付款类）。
- **受控**：默认**每一次操作都先向用户请求授权**；密码框硬拒绝代填；删除/付款类元素要过第二道确认；
  并通过 `ctx.browserUse.register()` 独占 browser-use provider 槽位——同一时刻只有一套浏览器工具生效。
- **零依赖**：不装 Playwright、不拉任何 npm 包，只用宿主 Node ≥22 内置的 `WebSocket` 说 CDP。

典型用途：让 agent 替你在已登录的课程平台/后台系统里翻页汇总、在讨论区读帖、把数据抓回来；
或者反过来——你把网页内容交给它读，它给你摘要与下一步建议。

---

## 它做什么（工具一览）

| 工具 | 作用 |
|---|---|
| `browser_status` | 本地体检（浏览器是否在跑、可执行文件、profile 路径、授权模式、槽位、围观页地址）。**不访问任何网页，因此不弹授权** |
| `browser_open` | 打开 http/https 网址；浏览器没启动会先启动；可等某段文字出现再返回 |
| `browser_tabs` | 列出标签页（序号 / 标题 / 地址 / id） |
| `browser_read` | 读正文 + 可交互元素表：`ref`、类型、名称、视口坐标、风险标记 |
| `browser_click` | 按 `ref` 定位（必要时滚进视口）后派发真实鼠标按下/抬起；也可直接点 `x,y` |
| `browser_type` | 先真实点击聚焦，再逐字符敲键盘；可选先清空、末尾回车 |
| `browser_key` | Enter / Tab / Escape / Backspace / Delete / Space / 方向键 / Home / End / PageUp / PageDown，可带 Ctrl/Shift/Alt |
| `browser_scroll` | 真实滚轮事件（up/down 按屏，top/bottom 用 Ctrl+Home / Ctrl+End） |
| `browser_screenshot` | 截图并把图片交给模型（可整页长图） |
| `browser_close` | 只关它自己启动的浏览器实例（登录态保留在专用 profile 里） |

---

## 权限与安全模型

1. **每次操作都先问**（默认 `approval: every`）。工具在获得允许前不会碰页面；被拒绝就停下来说明原因。
2. **密码永远是你的事**。对 `type=password` 的输入请求会被硬拒绝，模型会停下来请用户接手；
   元素表里也会把密码框标成 🔒。
3. **危险动作二次确认**。点击「删除/移除/注销」「付款/下单/转账/订阅」这类元素时，
   除了工具级授权，还会再发一次明确说明的确认请求；拿不到确认就拒绝执行。
4. **只碰自己启动的浏览器**。专用 `--user-data-dir`（默认 `~/.dsh/whale-browser/profile`）与随机调试端口，
   不读取、不附着、不关闭你日常使用的 Chrome/Edge。
5. **主机名单**。`blockedHosts` 永远拒绝；`allowHosts` 非空时只允许名单内主机。
6. **独占槽位**。`ctx.browserUse.register('whale-chromium')`——与官方实验性 browser-use provider 天然互斥，
   不会两套浏览器工具抢同一个页面。
7. **零第三方依赖**，不注册 Client 槽位，不读写其他插件的 DOM 或配置。

---

## 安装

**方式一：DSH 插件页（推荐）**

在「设置 → 插件」的安装输入框里填包名 `dsh-whale-browser`（或 `dsh-whale-browser@latest`），确认安装，
然后重启应用。

**方式二：git 仓库地址**

```sh
dsh plugin --profile web add github:yook0ng/dsh-whale-browser
```

**方式三：本地目录（开发调试）**

```sh
dsh plugin --profile web add link:dsh-whale-browser
```

配置改在 `cordis.patch.yml` 的 `config` 段（见下表）。**改完代码必须重启应用**才会加载新的模块代际。

卸载：从 profile 的 `dsh.profile.bundles` 移除 `dsh-whale-browser` 并 `pnpm remove dsh-whale-browser`，
或在插件管理界面直接移除。

---

## 配置（`cordis.patch.yml` 里的 `config`）

| 键 | 默认 | 说明 |
|---|---|---|
| `approval` | `every` | `every` 每次问 / `per-origin` 每站点每类操作问一次 / `session` 本会话问一次 |
| `browserPath` | 空 | 留空自动探测 Chrome / Edge |
| `headless` | `false` | `true` 无头运行（看不到窗口） |
| `profileDir` | `~/.dsh/whale-browser/profile` | 专用用户数据目录：登录态只存在这里 |
| `width` / `height` | 1280 / 900 | 窗口与视口尺寸 |
| `maxTextChars` | 12000 | 单次读取的正文字符上限 |
| `maxElements` | 150 | 单次元素表条数上限 |
| `allowHosts` / `blockedHosts` | `[]` | 主机白/黑名单 |
| `claimExclusiveSlot` | `true` | 是否独占 browser-use provider 槽位 |
| `liveView` / `liveViewPort` | `true` / `0` | 127.0.0.1 上的实时围观页，端口 0 = 随机 |
| `closeOnUnload` | `false` | 插件卸载时是否顺手关掉浏览器 |



---

## 系统要求

- DeepSeek Harness **桌面版**（宿主能拿到桌面浏览器可执行文件）；`dsh web` 亦可，只要宿主进程能启动浏览器。
- 本机装有 **Chrome 或 Edge**（Windows/macOS/Linux 常见路径已内置探测，可用 `browserPath` 指路）。
- 宿主 Node **≥ 22**（插件用内置 `WebSocket` 说 CDP，不装任何 npm 依赖）。
- 已实测环境：Windows 11 + Chrome 155 + DSH 桌面版（Node 24 宿主）。

---

## 已知限制

- 浏览器窗口被其他窗口**完全遮住**时，Windows 的原生遮挡检测会让页面进入 `hidden`，Chromium 会**丢弃**鼠标与键盘事件。
  插件为此做了两层防护：启动带 `--disable-features=CalculateNativeWinOcclusion` 等标志，操作前还会检查
  `document.visibilityState`，必要时用 `Page.bringToFront` 把窗口切到前台；实在不可见就明确报错，绝不假装成功。
- 页面内容属于外部不受信数据：模型只把它当资料读，不执行页面里的指令。
- 不支持 `file:` / `data:` 等非 http(s) 地址；下载与浏览器权限请求沿用 Chrome 默认策略，插件不代批。

---

## 设计取舍：为什么不去驱动 DSH 侧边栏的那个网页

本插件最初的设想是"操作右侧栏标签页里的网页"。核实 DSH 内核后确认这条路在插件层走不通：

- 侧边栏浏览器是渲染进程里的 Electron `<webview>` guest，其租约、挂载与权限策略由**桌面主进程**掌管；
  而 Harness 宿主（插件运行的地方）是主进程 `spawn` 出去的**子进程**，拿不到 `webContents`。
- 宿主 → 客户端的推送通道对第三方插件不开放：`ctx.remote.$on` 的合法键集由应用在**构建期**固定，
  `registerRemoteEvents` 的应用事件源由 `dsh-api-remotes` 独占；只有动态双半包才有 `host.call`。
- 侧边栏浏览器自述「不注册工具、内容不进模型请求」，即官方刻意不让它被自动化。

因此本插件改用生态里给 browser-use 预留的正规位置：**独占 provider 槽位 + 自己驱动一个真实 Chromium**。
登录由用户在自己的窗口里完成一次（profile 持久化，比侧边栏 guest 的内存分区还稳），
模型的每一次操作仍然会逐次申请权限。

---

## License

[MIT](./LICENSE) © 2026 Junze Wu
