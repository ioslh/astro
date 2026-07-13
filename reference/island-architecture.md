# Astro Island 架构调研

## 概览

Astro 的 island 架构可以拆成两条主线：

1. **客户端 islands（`client:*`）**：服务端先输出静态 HTML，只把需要交互的组件包装成 `<astro-island>`，再在浏览器中按需局部 hydrate。
2. **服务端 islands（`server:defer`）**：服务端先输出 fallback，占位内容随后通过单独的请求拉取真实 HTML，再替换回页面。

从实现上看，这套机制主要由以下阶段组成：

1. 编译阶段识别哪些组件需要变成 island
2. SSR 阶段提取 hydration 指令并输出 `<astro-island>`
3. 页面注入 island runtime 与各类 directive 脚本
4. 浏览器端按 `client:*` 策略加载组件并执行 hydrate
5. `server:defer` 通过专门的 endpoint 返回岛的 HTML

---

## 一、编译阶段：识别 island 组件

关键入口：

- `/home/runner/work/astro/astro/packages/astro/src/vite-plugin-astro/index.ts`
- `/home/runner/work/astro/astro/packages/astro/src/vite-plugin-astro/metadata.ts`

`vite-plugin-astro` 在调用编译器后，会把编译结果中的以下信息写入 `meta.astro`：

- `hydratedComponents`
- `clientOnlyComponents`
- `serverComponents`

这意味着 Astro 在编译阶段就已经区分出：

- 哪些组件需要客户端 hydrate
- 哪些组件是 `client:only`
- 哪些组件属于服务端 islands

这是 island 架构后续构建、SSR 和运行时行为的基础。

---

## 二、SSR 阶段：提取 `client:*` 指令

关键文件：

- `/home/runner/work/astro/astro/packages/astro/src/runtime/server/hydration.ts`

`extractDirectives()` 会从组件 props 中拆出 Astro 的特殊指令，包括：

- `client:load`
- `client:idle`
- `client:visible`
- `client:media`
- `client:only`
- `client:component-path`
- `client:component-export`

处理结果分成两部分：

1. **普通 props**：继续传给组件做 SSR
2. **hydration metadata**：记录 hydrate 指令、组件模块路径、导出名和指令参数

也就是说，Astro 会先把 island 所需的元信息从普通组件 props 里抽离出来，再交给渲染阶段使用。

---

## 三、SSR 阶段：把组件包装成 `<astro-island>`

关键文件：

- `/home/runner/work/astro/astro/packages/astro/src/runtime/server/render/component.ts`

这是客户端 island 最核心的服务端输出入口。

主要流程是：

1. 调用 renderer 执行组件 SSR，得到静态 HTML
2. 如果组件没有 `client:*` 指令，就直接输出 HTML
3. 如果组件带有 `client:*` 指令，就生成一个 island 描述对象
4. 最终把组件输出成 `<astro-island>` 自定义元素

几个关键点：

- `extractDirectives()` 的结果会写入组件 metadata
- `generateHydrateScript()` 会把 hydrate 所需信息序列化到 island 属性中
- SSR 输出的 HTML 会作为 `<astro-island>` 的 children 保留下来
- 嵌套 slot 会被包装和补齐，保证客户端 hydrate 时仍可恢复

Astro 的“部分水合”不是把整个页面交给前端框架，而是只把有交互需求的那部分组件包进 `<astro-island>`。

---

## 四、生成 hydration 元数据

关键文件：

- `/home/runner/work/astro/astro/packages/astro/src/runtime/server/hydration.ts`

`generateHydrateScript()` 会给 `<astro-island>` 挂上运行时需要的关键属性，典型包括：

- `component-url`
- `component-export`
- `renderer-url`
- `props`
- `client`
- `ssr`
- `opts`
- `before-hydration-url`

这些字段分别承载：

- 组件模块地址
- 组件导出名
- 前端框架 renderer 的 client entry
- 序列化后的 props
- hydrate 策略（如 `load`、`idle`、`visible`）
- 当前节点尚未 hydrate 的标记
- hydrate 参数
- hydrate 前需要预先执行的脚本

因此，浏览器端并不需要再额外查询 island 的定义；它只需要读取 `<astro-island>` 的属性，就能知道如何恢复组件。

---

## 五、页面注入 runtime 与 directive 脚本

关键文件：

- `/home/runner/work/astro/astro/packages/astro/src/runtime/server/scripts.ts`
- `/home/runner/work/astro/astro/packages/astro/src/core/client-directive/default.ts`

Astro 在 SSR 输出过程中，会按需注入两类脚本：

1. **directive 脚本**：定义 `client:load`、`client:idle`、`client:visible`、`client:media`、`client:only` 的触发逻辑
2. **island runtime 脚本**：定义 `<astro-island>` 这个自定义元素

默认指令的注册位置在：

- `/home/runner/work/astro/astro/packages/astro/src/core/client-directive/default.ts`

默认内置的 client directives 为：

- `idle`
- `load`
- `media`
- `only`
- `visible`

它们都以预构建脚本的形式注入，最终在浏览器里挂到全局的 `Astro[directive]` 上。

---

## 六、浏览器端 runtime：`astro-island` 自定义元素

关键文件：

- `/home/runner/work/astro/astro/packages/astro/src/runtime/server/astro-island.ts`

这是 Astro island 架构最关键的运行时代码。

它做的事情包括：

1. 定义自定义元素 `astro-island`
2. 在元素连接到 DOM 时启动 hydrate 流程
3. 根据属性读取组件 URL、renderer URL、props、slots 和 client directive
4. 动态 `import()` 组件模块和 renderer
5. 反序列化 props
6. 调用对应 renderer 的 hydrator 完成局部激活

这里有几个重要机制：

### 1. `connectedCallback()` 与 `await-children`

如果 SSR 输出还在流式进行，`astro-island` 可能先于子节点完成插入 DOM。

因此 runtime 会：

- 监听子节点变化
- 等待 `<!--astro:end-->` 标记出现
- 再继续后续 hydrate

这保证了流式渲染场景下，岛内 HTML 已完整可用后才开始激活。

### 2. `start()`

`start()` 会读取当前 island 的 `client` 属性，并调用对应的 directive：

- `load`
- `idle`
- `visible`
- `media`
- `only`

如果某个 directive 脚本尚未注册，runtime 会等待 `astro:${directive}` 事件后重试。

### 3. `hydrate()`

真正的 hydrate 发生在 `hydrate()` 中：

- 先恢复 slots
- 再反序列化 props
- 最后调用 renderer 的 hydrator 执行框架级别的激活

hydrate 完成后会：

- 移除 `ssr` 属性
- 派发 `astro:hydrate` 事件

### 4. 父子 island 的 hydrate 顺序

runtime 会检查：

- `this.parentElement?.closest('astro-island[ssr]')`

如果存在尚未完成 hydrate 的父 island，就等待父 island 先触发 `astro:hydrate`。

这意味着 Astro 明确保证：

- **父 island 先 hydrate**
- **子 island 后 hydrate**

从而避免父组件重新挂载时破坏子 island。

---

## 七、`client:*` 指令本身只是“hydrate 时机”

关键文件：

- `/home/runner/work/astro/astro/packages/astro/src/runtime/client/load.ts`
- `/home/runner/work/astro/astro/packages/astro/src/runtime/client/idle.ts`
- `/home/runner/work/astro/astro/packages/astro/src/runtime/client/visible.ts`
- `/home/runner/work/astro/astro/packages/astro/src/runtime/client/media.ts`
- `/home/runner/work/astro/astro/packages/astro/src/runtime/client/only.ts`

这些文件都非常薄，它们本质上只负责：

1. 等待一个时机
2. 调用 `load()` 加载 hydrator
3. 执行 hydrate

每个指令的语义如下：

- `client:load`：页面加载后立即 hydrate
- `client:idle`：浏览器空闲时 hydrate
- `client:visible`：元素可见时 hydrate
- `client:media`：匹配媒体查询时 hydrate
- `client:only`：跳过 SSR，仅在客户端渲染

因此，从架构角度看，Astro 的 client directives 不是另一套复杂框架，而只是控制 island **何时激活** 的触发器。

---

## 八、服务端 islands：`server:defer`

除了客户端 islands，Astro 还支持服务端 islands。

关键文件：

- `/home/runner/work/astro/astro/packages/astro/src/runtime/server/render/server-islands.ts`
- `/home/runner/work/astro/astro/packages/astro/src/core/server-islands/endpoint.ts`
- `/home/runner/work/astro/astro/packages/astro/src/core/server-islands/vite-plugin-server-islands.ts`

### 1. 识别服务端 island

在：

- `/home/runner/work/astro/astro/packages/astro/src/runtime/server/render/component.ts`

里，如果组件 props 中带有 `server:*` 指令，就会切换到 `ServerIslandComponent` 路径。

### 2. 先输出 fallback，再异步替换

`ServerIslandComponent` 在 SSR 输出时不会直接把完整内容塞进主页面，而是：

1. 输出 fallback
2. 插入一个运行时脚本
3. 请求 `/_server-islands/[name]`
4. 拿到真实 HTML 后替换占位内容

### 3. 请求载荷会被加密

在：

- `/home/runner/work/astro/astro/packages/astro/src/runtime/server/render/server-islands.ts`

中，组件导出名、props 和 slots 都会先加密，再通过 GET 或 POST 发给服务端 endpoint。

这样做的目的是：

- 避免明文暴露内部组件信息
- 限制被篡改的风险

### 4. endpoint 返回 HTML

在：

- `/home/runner/work/astro/astro/packages/astro/src/core/server-islands/endpoint.ts`

中，Astro 会：

1. 解析请求
2. 解密组件导出名、props 和 slots
3. 找到对应模块
4. 再次执行服务端渲染
5. 返回 HTML 片段

这和客户端 islands 的思路不同：客户端 islands 的目标是“局部 hydrate”；服务端 islands 的目标是“局部延迟服务端渲染并回填 HTML”。

---

## 九、构建期如何收集 server islands

关键文件：

- `/home/runner/work/astro/astro/packages/astro/src/core/server-islands/vite-plugin-server-islands.ts`

这个 Vite 插件负责：

- 从 `meta.astro.serverComponents` 中发现服务端 islands
- 为它们生成名字与模块路径映射
- 构建 `virtual:astro:server-island-manifest`

最终运行时可以通过这个 manifest：

- 从 island 名称找到实际组件模块
- 为 `/_server-islands/[name]` endpoint 提供可渲染的 import map

---

## 十、最关键的代码位置

如果只看最核心的实现，建议优先阅读以下文件。

### 客户端 islands

1. `/home/runner/work/astro/astro/packages/astro/src/vite-plugin-astro/index.ts`
   - 编译后产出 `hydratedComponents` / `clientOnlyComponents` / `serverComponents`

2. `/home/runner/work/astro/astro/packages/astro/src/runtime/server/hydration.ts`
   - 解析 `client:*` 指令
   - 生成 `<astro-island>` 的 hydration 元数据

3. `/home/runner/work/astro/astro/packages/astro/src/runtime/server/render/component.ts`
   - 决定组件是否要变成 island
   - 输出 `<astro-island>`

4. `/home/runner/work/astro/astro/packages/astro/src/runtime/server/scripts.ts`
   - 注入 directive 脚本和 island runtime

5. `/home/runner/work/astro/astro/packages/astro/src/runtime/server/astro-island.ts`
   - 浏览器端 island runtime 核心

6. `/home/runner/work/astro/astro/packages/astro/src/runtime/client/load.ts`
7. `/home/runner/work/astro/astro/packages/astro/src/runtime/client/idle.ts`
8. `/home/runner/work/astro/astro/packages/astro/src/runtime/client/visible.ts`
9. `/home/runner/work/astro/astro/packages/astro/src/runtime/client/media.ts`
10. `/home/runner/work/astro/astro/packages/astro/src/runtime/client/only.ts`
   - 定义不同 `client:*` 指令的触发时机

### 服务端 islands

11. `/home/runner/work/astro/astro/packages/astro/src/runtime/server/render/server-islands.ts`
    - 生成 `server:defer` 的占位和替换脚本

12. `/home/runner/work/astro/astro/packages/astro/src/core/server-islands/endpoint.ts`
    - 返回服务端 island 的 HTML

13. `/home/runner/work/astro/astro/packages/astro/src/core/server-islands/vite-plugin-server-islands.ts`
    - 构建期收集并生成 manifest

---

## 十一、总结

Astro 的 island 架构本质上是：

1. **编译期识别带 `client:*` / `server:*` 的组件**
2. **SSR 先输出静态 HTML**
3. **把需要交互的组件包装成 `<astro-island>`**
4. **把组件地址、renderer、props 和 hydrate 策略序列化进标签属性**
5. **浏览器端通过 `astro-island` 自定义元素按需局部 hydrate**
6. **`server:defer` 则通过独立 endpoint 延迟返回 HTML 并替换占位**

因此，Astro island 架构并不是一个单独的“渲染器”，而是一套横跨：

- 编译器元数据
- SSR 输出
- 页面脚本注入
- 浏览器 runtime
- 服务端延迟渲染

的完整机制。
