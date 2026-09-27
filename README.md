# scratch-deepseek-web-panel

把 **chat.deepseek.com 的网页版模型** 接进 Scratch 扩展编辑器的插件：用浏览器里的登录态直接调用网页模型，不需要 API Key、不消耗 API 额度。

这是一个**外置插件**，通过编辑器的「设置 → 插件市场」一键安装，不需要手动放文件。

## 安装

编辑器 → 设置 → 插件市场 → 找到「DeepSeek 网页版」→ 安装。

安装时编辑器会：

1. 把本仓库的文件拉到 `%APPDATA%\scratch-extension-editor\plugins\deepseek-web-panel\`；
2. 把 `server.mjs` fork 成独立子进程；
3. 把子进程回报的路由前缀 `/deepseek-web-vision/api` 记进路由表并反向代理。

装完在「设置 → DeepSeek 网页版」里就能看到界面。卸载走「设置 → 插件管理 → 删除」。

也可以手动安装：把整个目录放到上述路径即可。

## 它做什么

能力 | 说明
---|---
浏览器登录态捕获 | 复用你在浏览器里已登录的 DeepSeek 账号，不需要填 API Key
账号库 | 可存多个账号，随时切换 / 重登 / 备注
PoW 求解 | 处理网页端的 proof-of-work 挑战，纯本地计算
SSE 流式 | 流式接收回复，编辑器里能边生成边看
图片理解 | 支持把图片一起投喂给模型
台账 | 记录调用次数、成功率、请求间隔分位数，便于观察限流

## 目录结构

```
deepseek-web-panel/
├── plugin.json                  # 插件清单（id / 名称 / 版本 / 路由前缀）
├── index.js                     # 浏览器侧界面：设置面板左侧入口 + 完整管理界面
├── server.mjs                   # Node 侧入口：由编辑器 fork 成独立子进程
└── vendor/
    └── deepseek-web-host.mjs    # 编译后的宿主产物（含全部业务逻辑）
```

`server.mjs` 只做三件事：

1. 喂一个最小 `ctx` shim，把 `vendor/deepseek-web-host.mjs` 跑起来；
2. 起一个**只监听 127.0.0.1** 的 HTTP server，挂上 host bundle 注册的 prefix handler；
3. 通过 fork 的 IPC 把实际端口回报给父进程（编辑器），由编辑器把 `/deepseek-web-vision/api/*` 反向代理过来。

端口用 `0`（系统分配）而不是固定值：编辑器可能同时开着多个版本，固定端口会互相抢占。

`index.js` 是浏览器侧插件，走编辑器的 ExtAddons 机制：导出一个带 `setup` 的对象，`setup` 里挂一个 MutationObserver —— 设置面板一出现就把入口按钮补进左侧导航，关掉就清理。它不依赖宿主何时调用，也不会在 React 重渲染时掉出来。

## 通信协议

父子进程之间只有两条 IPC 消息：

```js
// 子进程 → 父进程：已就绪
{ type: 'listening', port: <number>, routes: ['/deepseek-web-vision/api'] }

// 子进程 → 父进程：启动失败
{ type: 'error', error: '<message>' }
```

父进程据此建一张路由表，把匹配前缀的请求反代到 `127.0.0.1:<port>`。父进程退出时子进程跟随退出（`process.on('disconnect')` / `SIGTERM`），不留孤儿进程。

## 数据放在哪

所有本地状态都在 `${DSH_HOME || ~/.dsh}/deepseek-web-vision/` 下，**不写进** 编辑器的 settings / credentials：

* 账号库与登录态
* 调用台账

这样插件可以独立升级、独立卸载，不会污染主程序的配置面。

## 安全说明

* 服务只监听 `127.0.0.1`，不对外暴露；
* 登录凭据只存在本机上述目录，不随插件文件分发；
* 仓库里**不含** 任何 API Key、Token 或账号信息；
* 调用走你本人的网页端登录态，请遵守 DeepSeek 的服务条款，不要用于高频刷量。

## 许可

GPL-3.0
