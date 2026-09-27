# scratch-deepseek-web-panel

把 [chat.deepseek.com](https://chat.deepseek.com) 的网页模型接进 **Scratch 扩展编辑器**的插件。

**这是插件，不是独立程序。** 它依赖编辑器提供的 Node 侧插件运行时（fork 子进程 + 路由反代 + IPC 握手），单独跑没有任何界面。

> A plugin for the Scratch Extension Editor that brings chat.deepseek.com into the editor
> — browser login capture, account vault, PoW solving, SSE streaming and image understanding.
> It is **not** standalone: it requires the host runtime described below.

---

## 它做什么

编辑器本身是个纯前端页面，但这个插件需要真正的 Node 能力：

- **浏览器登录态捕获** —— 复用你在 Edge / Chrome 里已登录的 chat.deepseek.com，不碰 API Key
- **账号库** —— 多账号保存、切换、定期只读校验登录态
- **PoW 求解** —— DeepSeek 的 proof-of-work 反爬，在本地算
- **SSE 流式** —— 把网页版的分块响应转成标准流式输出
- **图片理解** —— 把图片随对话投喂给网页模型
- **用量台账** —— 调用次数、成功率、限流分布、请求间隔分位数

数据落在 `${DSH_HOME || ~/.dsh}/deepseek-web-vision/` 下（账号库、配置、台账），不进 `settings` / `credentials` 缝隙。

---

## 目录结构

```
plugin.json                    插件清单（id / name / version / routes）
server.mjs                     Node 侧入口：ctx shim + http server + IPC 握手
vendor/deepseek-web-host.mjs   上游 host bundle（418 KB，见下方「关于 vendor」）
```

只有三个文件。**没有构建步骤**，克隆下来就能用。

---

## 怎么装

### 方式一：随编辑器内置（推荐）

编辑器启动时会把它自己的 `plugins-src/` 铺到用户插件目录。本仓库的内容就是那个 `plugins-src/deepseek-web-panel/`。

### 方式二：手工放进插件目录

把整个目录拷到编辑器的插件目录：

| 平台 | 路径 |
|---|---|
| Windows | `%APPDATA%\scratch-extension-editor\plugins\deepseek-web-panel\` |
| macOS | `~/Library/Application Support/scratch-extension-editor/plugins/deepseek-web-panel/` |
| Linux | `~/.config/scratch-extension-editor/plugins/deepseek-web-panel/` |

拷贝后**重启编辑器**（插件在启动时 fork，热改不生效）。

---

## 宿主运行时接口

本插件不是被动加载的模块，它要求宿主满足以下约定。换个宿主就得照着实现。

### 1. 启动方式

宿主在插件目录里按 `server.mjs` → `server.js` 的顺序找入口，`fork` 成**独立子进程**：

```js
fork(entry, [], {
  cwd: dataDir,              // 注意：不要用插件目录本身，见下方「为什么 cwd 不是插件目录」
  stdio: ['ignore', 'pipe', 'pipe', 'ipc'],
  env: {
    ...process.env,
    SCRATCH_EDITOR_PLUGIN_DIR: dir,   // 插件自己的目录
    SCRATCH_EDITOR_DATA_DIR: dataDir, // 编辑器数据目录
    DSH_HOME: process.env.DSH_HOME || dataDir
  }
});
```

### 2. IPC 握手

子进程监听成功后，必须通过 `process.send` 回报端口和它要占用的路由前缀：

```js
process.send({ type: 'listening', port, routes: ['/deepseek-web-vision/api'] });
```

宿主收到后建路由表，把该前缀下的 HTTP 请求反代到这个端口。

**端口用 `0`（系统分配）而不是固定值。** 用户可能同时开着多个编辑器实例，固定端口会互相抢占，而「端口被占用」这种错误对用户毫无意义。

### 3. 生命周期

- 宿主 `disconnect` 时子进程自行退出，避免留孤儿进程
- `SIGTERM` 同样退出

### 4. ctx shim

`vendor/deepseek-web-host.mjs` 原本是 DSH 的 cordis 插件，它只需要宿主 `ctx` 上很少几个面。`server.mjs` 里喂了一个最小实现：

| 成员 | 说明 |
|---|---|
| `ctx.logger.{info,warn,error,debug}` | 日志 |
| `ctx.effect(fn)` | 跑副作用函数并返回 disposer |
| `ctx.get(name)` | 一律返回 `undefined`（见下） |
| `ctx.llm.registerAdapter(route)` | 记录 provider id |
| `ctx.llm.listProviders()` | 返回已注册 provider |
| `ctx.webServer.register({kind, path, handler})` | **关键**：插件借它挂路由 |

`registerAdapter` 和 `listProviders` 必须**成对**提供。管理界面用 `listProviders()` 渲染「Provider 已注册」那一栏；只给 `registerAdapter` 的话该调用会抛错，被上游 `catch {}` 吞掉后面板长期显示「否」——看起来像插件没装配成功，实际只是缺个查询方法。

`ctx.get('attachments')` 返回 `undefined` 是**预期行为**：host bundle 会把图片输入降级为文本，并在界面上如实说明，而不是抛异常让整个插件挂掉。

---

## 关于 vendor

`vendor/deepseek-web-host.mjs`（418 KB）**不是本仓库的源码**，它是从 DSH 的 DeepSeek 网页版插件的构建产物复制过来的单文件 bundle。之所以整个塞进来而不是走依赖，是因为它本来就不是为公开发布准备的包，没有可安装的 npm 入口。

**该文件的权利归其上游作者，本仓库的许可证不覆盖它。** 如果你的使用场景对这一点敏感，请自行向 DSH 确认授权。

本仓库自己写的部分只有 `server.mjs` 和 `plugin.json`，这两个文件在 MIT 之下。

---

## 已知限制

- **不支持 `chromium` 传输层。** 面板上「期望 chromium / 实际生效 node」是正常的：`electron.net.fetch` 只有 Electron 主进程里有，而插件跑在纯 Node 子进程里，探测失败后自动降级为 Node 传输层。功能不受影响。
- **图片理解是降级的。** 见上方 `ctx.get('attachments')`。
- **`DSH_HOME` 不会被强行改写。** 从 DSH 会话里起的编辑器会继承用户真实的 `~/.dsh`，那里已有的账号库和台账照常使用；强行改指会让面板显示「未登录」、台账清零。

---

## 为什么 cwd 不是插件目录

Windows 下「某目录是某个活进程的当前目录」会锁住该目录，导致升级 / 卸载 / 删除插件时报 `EPERM`。所以宿主 fork 时 `cwd` 用 `dataDir`，插件要定位自己的文件走 `SCRATCH_EDITOR_PLUGIN_DIR` 或 `import.meta.url` —— 两者都不依赖 cwd。

---

## 许可证

`server.mjs`、`plugin.json` 及本说明文档：MIT，见 `LICENSE`。
`vendor/` 目录：权利归上游，不适用 MIT，见上方「关于 vendor」。
