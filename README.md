## WebUI + AutoX v6 Demo

面向 LLM 的项目说明和开发约定。

本仓库是一个将 **Vite + Vue 3 + TypeScript Web UI** 嵌入到 **AutoX v6 脚本应用** 中的示例工程，通过 WebView + JS Bridge 与 AutoX 交互，并提供了一系列脚本用于打包、推送到设备和保存工程。

---

## 1. 目录结构概览

顶层主要目录和文件：

- `src/`：Web 前端源码（Vue 3 + Framework7）
  - `App.vue`：应用根组件，挂载 Framework7 布局，包含退出按钮等示例逻辑。
  - `components/HomePage .vue`：示例列表页面，点击列表项会通过 JS Bridge 通知 AutoX。
  - `js_bridge.ts`：Web 与 AutoX 的 JS Bridge 包装层，封装 `$autox` 的调用。
  - `main.ts`：Web 端入口，创建 Vue 应用并挂载到 `#app`。
- `src-autox/`：AutoX v6 脚本工程源码
  - `main.tsx`：AutoX 入口脚本，创建 UI 布局并嵌入 WebView，注册 JS Bridge 处理函数。
  - `start-ui.js`：AutoX 启动文件，加载打包后的 `main.js`。
  - `project.json`：AutoX 工程配置（名称、包名、入口脚本等）。
- `scrpits/`：Node 工具脚本（使用 ESM）
  - `tasks.mjs`：公共构建工具，封装 Rollup 打包、拷贝资源、获取设备 IP 等。
  - `dev-autox.mjs`：开发时将 AutoX 项目打包并推送到设备运行。
  - `build-autox.mjs`：构建生产用 AutoX 工程（包含本地 Web 资源）。
  - `save-autox.mjs`：构建并打包后上传到设备并保存为 AutoX 工程。
- `dist/`：Web 构建输出（由 Vite 生成，**不要手动修改**）。
- `dist-autox/`：AutoX 工程构建输出（由脚本生成，**不要手动修改**）。
- `index.html`：Vite 入口 HTML，包含 `autox://sdk.v1.js`，在 AutoX 内提供 `$autox` 等对象。
- `tsconfig*.json`：TypeScript 配置（分别针对 Web、Node 脚本和 AutoX 源码）。
- `.deviceIpAddress`：缓存最近使用的设备 IP（由脚本自动读写）。

LLM 在修改代码时，应优先在 `src/`、`src-autox/`、`scrpits/` 下工作，避免直接编辑 `dist/`、`dist-autox/`。

---

## 2. 运行与开发流程

### 2.1 环境要求

- Node.js（建议 ≥ 18）
- npm
- 一台安装了 **AutoX v6**（或兼容版本）的 Android 设备
  - 设备需开启 AutoX 提供的 HTTP 接口（脚本默认访问 `http://<ip>:9317/api/v1`）
  - 设备与开发机必须在同一局域网

### 2.2 安装依赖

```bash
npm install
```

### 2.3 仅 Web UI 本地开发grwgrw

当只需要开发和预览 Web 页面（不连手机）时：

```bash
npm run dev
```

- 默认在 `http://localhost:5173` 提供服务。
- `index.html` 中引入的 `autox://sdk.v1.js` 在普通浏览器中不会真正加载，但不会影响页面主体渲染。
- `js_bridge.ts` 内部会检查 `$autox` 是否存在；在普通浏览器中不会抛错，只会打印警告日志。

### 2.4 与 AutoX 联调开发（推荐）

当需要在手机 AutoX 中真实运行并联调 Web UI 与脚本交互时：

1. 确保手机和开发机在同一网络，并开启 AutoX 的 HTTP 接口（端口 `9317`）。
2. 在开发机上启动 Web 开发服务器：
   ```bash
   npm run dev
   ```
3. 在另一个终端运行：
   ```bash
   npm run dev-autox
   ```
4. 首次运行会提示输入设备 IP：
   - 输入后脚本会写入 `.deviceIpAddress`，下次会以此为默认值。
   - 脚本会访问 `http://<ip>:9317/api/v1?type=getip` 获取 `remoteHost`。
5. `dev-autox.mjs` 会调用 `rollupBuild(remoteHost)`：
   - 使用 `src-autox/main.tsx` 为入口，打包到 `dist-autox/main.js`。
   - 在打包过程中会用实际的 `remoteHost` 替换代码中的 `__DevIp__`。
6. 打包完成后会打成 zip 并通过 `axios.post(uri, zip, { params: { type: 'runProject', dirName: outDir } })` 推送到设备并运行。

此模式下，AutoX 里的 WebView 会加载 `http://<remoteHost>:5173/` 上的 Web UI，前端改动可立即生效。

### 2.5 构建 AutoX 工程（本地）

构建完整的 AutoX 工程输出（但不推送到设备）：

```bash
npm run build-autox
```

流程概览：

1. 执行 `npm run build`：Vite 构建 Web 应用到 `dist/`。
2. 调用 `rollupBuild()`：打包 AutoX 入口脚本到 `dist-autox/main.js`。
3. 调用 `copyWebsite()`：将 `dist/` 拷贝到 `dist-autox/website/`。

构建后的 AutoX 工程位于 `dist-autox/`。

### 2.6 构建并保存到设备为 AutoX 工程

如需构建并保存为 AutoX 本地工程（而不仅是运行一次），使用：

```bash
npm run save-autox
```

流程：

1. 执行 `npm run build`（构建 Web）。
2. `rollupBuild()` 打包 AutoX 入口。
3. `copyWebsite()` 拷贝 Web 资源。
4. 自动读取 `dist-autox/project.json` 中的 `name` 作为默认保存名（若存在）。
5. 通过命令行交互让你输入 `saveName`。
6. 将 `dist-autox` 打包成 zip，并通过 AutoX 接口 `type=saveProject` 上传并保存。

---

## 3. AutoX 与 Web UI 的通信机制

### 3.1 JS Bridge 封装（Web 侧）

文件：`src/js_bridge.ts`

- 全局声明（由 `autox://sdk.v1.js` 注入）：
  ```ts
  declare const $autox: {
    registerHandler(name: string, handler: (data: string, callback?: (data: string) => void) => void): void
    callHandler(name: string, data?: string, callback?: (data: string) => void): void
  }
  ```
- 导出两个包装函数：
  - `registerHandler(name, handler)`：在 AutoX 环境下实际调用 `$autox.registerHandler`，否则仅输出警告。
  - `callHandler(name, data?, callback?)`：在 AutoX 环境下调用 `$autox.callHandler`，否则输出警告。

**LLM 开发约定：**

- 在 Web 端代码（Vue 组件等）中 **不要直接访问 `$autox`**，统一通过 `js_bridge.ts` 中的 `callHandler` 及 `registerHandler`。
- 在非 AutoX 环境（浏览器）中，Bridge 会自动降级为仅输出警告，方便本地调试。

### 3.2 AutoX 入口脚本与 WebView

文件：`src-autox/main.tsx`

主要逻辑：

- 使用 `ui.layout(...)` 定义界面布局：
  ```tsx
  ui.layout(
    <frame>
      <webview id="webview" />
    </frame>
  );
  ```
- 获取 WebView 和 JS Bridge：
  ```ts
  const webView = ui.webview;
  const jsBridge = webView.jsBridge;
  ```
- 根据是否定义了 `__DevIp__` 加载页面：
  - 当打包时传入了 `remoteHost`（如联调模式）：加载 `http://__DevIp__:5173/`。
  - 否则：加载本地静态资源 `./website`。
- 注册示例处理函数：
  - `test`：在 Web 端调用 `callHandler('test', 'xxx')` 时，AutoX 会弹出 toast。
  - `exit`：在 Web 端调用 `callHandler('exit')` 时，AutoX 会退出当前脚本。

### 3.3 Web 端示例调用

- 文件：`src/components/HomePage .vue`
  - 在列表项点击时调用：
    ```ts
    function onClick(c?: string) {
      callHandler('test', '' + c)
    }
    ```
- 文件：`src/App.vue`
  - “退出 APP” 按钮点击时调用：
    ```ts
    function exit() {
      callHandler('exit')
    }
    ```

**LLM 扩展建议：**如需新增更多交互，建议：

- 在 AutoX 侧（`src-autox/main.tsx` 或拆分出的模块）中 `jsBridge.registerHandler('yourEvent', handler)`。
- 在 Web 侧通过 `callHandler('yourEvent', payload)` 调用。
- 事件名保持一致，并尽量使用常量或统一管理，避免魔法字符串散落。

---

## 4. 对 LLM 的开发约定与建议

本节是专门给 LLM 的操作指南，用于约束修改范围和方式，减少对构建流程的破坏风险。

### 4.1 一般性约定

- **不要手动修改构建产物：**
  - 不修改 `dist/`、`dist-autox/` 下的任何文件。
  - 如需变更行为，修改对应的源码文件并通过脚本重新构建。
- **保持 TypeScript 严格配置：**
  - 遵守 `tsconfig.app.json` / `tsconfig.autox.json` 的约束。
  - 尽量避免使用 `any` 和未类型标注的函数参数。
- **保持 Node 脚本为 ESM：**
  - `scrpits/*.mjs` 使用 ESM 语法（`import` / `export`），不要改回 CommonJS。
  - 如需新增脚本，请参考 `dev-autox.mjs` 和 `save-autox.mjs` 的风格。

### 4.2 Web UI（`src/`）修改建议

针对 Web 界面逻辑：

- 小改动（文案、样式）：
  - 修改 Vue SFC (`*.vue`) 模板或样式块即可。
  - 遵循现有的 Framework7 使用方式（如 `f7App`, `f7View`, `f7Page` 等）。
- 新增功能/按钮：
  - 在现有组件中增加新的按钮或列表项。
  - 如需和 AutoX 交互，使用 `callHandler('eventName', data)`。
  - 如仅在浏览器中使用，可不依赖 AutoX。
- 组件组织：
  - 新组件请放到 `src/components/` 下，并通过 `App.vue` 或其他组件引用。

### 4.3 AutoX 脚本（`src-autox/`）修改建议

- 布局调整：
  - 仍然使用 JSX 写法和 `ui.layout`，保持风格一致。
  - 避免过度复杂的 UI；此项目的核心是 WebView + JS Bridge。
- JS Bridge 扩展：
  - 新增 `jsBridge.registerHandler` 时，确保和 Web 端使用的事件名一致。
  - 处理函数要考虑可能来自 Web 的无效数据（做基本校验）。
- 工程配置：
  - 如需修改工程名称、包名等，优先修改 `src-autox/project.json`，再通过脚本构建生成 `dist-autox/project.json`。

### 4.4 构建脚本（`scrpits/`）修改建议

如非必要，不建议 LLM 大幅改动 `tasks.mjs`，因为它牵涉：

- Rollup 配置（入口为 `dist-autox/main.js`，插件若干，替换 `__DevModel__` 和 `__DevIp__`）。
- TypeScript 编译（内部调用 `vue-tsc -b`）。
- 与 AutoX 设备的 HTTP 通信逻辑。

如确需修改，请遵守：

- 保持 `rollupBuild(remoteHost?)` 的签名和返回值（返回输出目录路径）。
- 保持 `copyWebsite()` 的语义为 “将 `dist` 拷贝到 `dist-autox/website`”。
- 不要硬编码与当前业务无关的 IP / URL；尽量通过参数或配置传入。

---

## 5. 常见任务示例（给 LLM）

以下是一些典型任务及推荐操作路径：

1. **修改首页列表项文案：**
   - 文件：`src/components/HomePage .vue`
   - 修改 `<f7-list-item>` 的文本内容即可。
2. **新增一个调用 AutoX 的按钮：**
   - 在 `src/App.vue` 中新增一个 `<f7-button>`。
   - 在 `<script setup>` 中新增处理函数：
     ```ts
     function doSomething() {
       callHandler('yourEvent', 'payload')
     }
     ```
   - 在 AutoX 侧（`src-autox/main.tsx`）新增对应 `jsBridge.registerHandler('yourEvent', handler)`。
3. **调整 WebView 加载 URL 逻辑：**
   - 文件：`src-autox/main.tsx`
   - 仅在保持 `__DevIp__` 替换机制的前提下进行修改。
   - 不要删除本地 `webView.loadLocalFile(files.path("./website"));` 分支，否则生产构建会失去本地加载能力。

---

## 6. 总结

- 本项目由三部分组成：**Web 前端（Vite + Vue）**、**AutoX 脚本入口** 和 **Node 构建/部署脚本**。
- LLM 在修改时应主要聚焦在 `src/` 和 `src-autox/` 的业务逻辑，以及必要时的 `scrpits/` 增强。
- 避免直接编辑构建产物，保持 JS Bridge 统一入口 (`js_bridge.ts`)，并遵守现有 TypeScript 和目录结构约定。

> **重要（给大模型）**：在修改任何 AutoX 相关代码（如 `src-autox/*`、`scrpits/*` 等）之前，请务必先使用Context7检索https://context7.com/websites/autoxjs_dayudada，其中包含 AutoX 专用的开发规范和注意事项。
