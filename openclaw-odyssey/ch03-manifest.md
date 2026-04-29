# 第三幕 · Manifest 的沉默

> 这章要解决什么问题？
> Tool 和 provider 到底怎么进入 agent 的视野? 不是代码硬编码——是**一份 manifest 在启动前就决定了一切**.

---

## 一

凌晨五点一刻. 他继续追 75% 失败的原因.

他看到 `runEmbeddedPiAgent` 里有一行:

```ts
ensureRuntimePluginsLoaded({
  config: params.config,
  workspaceDir: resolvedWorkspace,
  allowGatewaySubagentBinding: params.allowGatewaySubagentBinding,
});
```

一个叫 `ensureRuntimePluginsLoaded` 的函数. 它在 agent 跑之前, **加载所有 plugin**.

他跳到 `src/plugins/contracts/registry.ts`——719 行.

看到一堆 **`createLazyArrayView`** 的导出:

```ts
export const providerContractRegistry = createLazyArrayView(...)
export const webSearchProviderContractRegistry = createLazyArrayView(...)
export const speechProviderContractRegistry = ...
// ... 十几个类似
```

"懒加载的数组视图." 他念道.

然后他读到 `src/plugins/CLAUDE.md` 里的一句话, 他停住了:

> **"Preserve manifest-first behavior: discovery, config validation, and setup should work from metadata before plugin runtime executes."**

---

## 二

**Manifest-first**.

他瞬间懂了——

- OpenClaw 启动的时候, **不加载任何 plugin 的代码**
- 它只读每个 plugin 的 `openclaw.plugin.json` — 那份 20 行的 manifest
- 从 manifest 知道:这个 plugin 是什么 id, 声明了什么能力, 要什么环境变量, 有什么 config schema
- **只在 agent 真的要用这个 plugin 时**, 才动态 import 它的 runtime

他想起自己做过的一个老系统——启动时 `import` 所有模块, 初始化所有 provider. 冷启动 8 秒. 大部分 provider 从来不被调用.

**OpenClaw 把这 8 秒压到了 200ms**. 因为 manifest 能回答"有哪些 plugin", runtime 的加载推迟到**第一次真需要**.

---

## 三

他翻开笔记本, 画了一张图——

```
     ┌─────────────────────────────────────────┐
     │              启动时刻                    │
     │                                         │
     │    ┌─────────────┐    ┌─────────────┐  │
     │    │ manifest.json│ →  │ 编目表      │  │
     │    │ (20 行)      │    │ (内存)       │  │
     │    └─────────────┘    └─────────────┘  │
     │     ✓ 读取快          ✗ plugin 代码未加载│
     └─────────────────────────────────────────┘

                       ↓ agent 请求用到 plugin 时

     ┌─────────────────────────────────────────┐
     │        执行时刻 (lazy)                   │
     │                                         │
     │    ┌─────────────────────┐              │
     │    │ await import(...)   │ → plugin 真  │
     │    │  runtime            │   代码加载   │
     │    └─────────────────────┘              │
     └─────────────────────────────────────────┘
```

**控制面(启动时)读 manifest, 执行面(运行时)懒加载 runtime**. 这就是**控制面/执行面分离**.

---

## 四

他突然理解为什么 OpenClaw 里那么多 `*.runtime.ts` 的 2 行壳文件.

```ts
// delivery.runtime.ts
export { deliverAgentCommandResult } from "./delivery.js";
```

以前他以为这是"神秘的文件命名习惯". 现在懂了——

- 静态 import 走 `delivery.js` (一般的代码路径)
- 动态 import 走 `delivery.runtime.js` (lazy 路径)
- 两条路**永远不走同一个 module ID**——不会互相污染 bundler 的 tree-shaking

"这是一个**二元门**." 他自言自语. "**要 lazy 就去 runtime, 要 eager 就走直接 import. 不允许混用**."

他想起 `src/plugin-sdk/CLAUDE.md` 里那条严厉的规则:

> **Do not mix static and dynamic imports for the same runtime surface.**

"现在我完全懂这条规则的动机了——**一旦混用, prompt-cache 会漂移, tree-shaking 会失败**."

他 在 笔记 旁 边 又 多 写 了 一 行——这 一 行 是 给 今晚 自己 的:

> ⚠ **如果 那 一刻 我 看到 的 是 mixed import, 我 至少 还要 多 花 一小时**——
>     花 在 chase tree-shaking artifacts: 为什么 这个 module 启动 时 没 加载, 但 有 一些 它 的 副作用 又 跑了? 是不是 webpack 配置 错了? 是不是 esbuild 哪里 漏了?
>
> 但 二元门 的 严格 规则, **直接 把 这 一类 bug 排除 在 外**.
> 我 今晚 节省 的 那一 小时, 就是 这条 规则 替 我 守住 的.

——他 第一次 觉得 一条 看起来 形式主义 的 规则, 在 凌晨 五点 的 incident 里, **比 任何 design doc 都 实在**. 设计 上 的 严苛, 是 debug 时的 福利.

---

## 五

他在笔记本上写下这一章的结论——

> **Plugin SDK 是 3 条契约**:
>
> 1. `definePluginEntry(...)` — 通用 plugin(tool/command/service/memory)
> 2. `defineSingleProviderPlugin(...)` — Provider(anthropic/openai/...)
> 3. `defineBundledChannelEntry(...)` — Channel(telegram/discord/...)
>
> 每个契约都薄薄几百行, 但**它们划出了 openclaw 和 extension 之间的边界**.
>
> Extension 写代码只能看见 `openclaw/plugin-sdk/*`. 看不见 `src/**`.
>
> **Plugin 不 load host internals. Host loads plugins**.
>
> 这是典型的 **反转的依赖**.

他继续追 bug. 他现在知道——agent 看见的 tool 列表, 是**从 registry 查出来的**, 而 registry 是从 manifest 建的. 这意味着他今晚的 75% 失败里, 有一类 bug 是**某个 plugin 的 manifest 或 runtime 出了问题, 但 manifest 没反映出来**.

这是今晚他追的第三层可能性.

---

## 关键概念

- **Manifest-first control plane**: 启动只读 manifest 元数据, runtime 懒加载.
- **控制面 / 执行面分离**: 配置/发现/validation 走控制面, 执行走 runtime.
- **`.runtime.ts` 二元门**: 每个懒加载表面都有 eager/lazy 两个 module ID.
- **Plugin SDK 三契约**: plugin-entry / provider-entry / channel-entry.
- **反转依赖**: Host loads plugins; plugins never load host.

## 我的理解

- **Manifest** 不是配置文件, 是**控制面的 API**. 它回答"系统里有什么"而不执行它们.
- **懒加载**不是性能优化, 是**架构边界**. 它让控制面和执行面的代码路径不会互相污染.
- **Plugin SDK** 的价值不在 "能插 plugin", 在于 "**plugin 不能接触 host 内部**"——这是可维护性的根基.

## 类比 · 空间感

像一个**图书馆的目录卡 + 书架**:

```
         ┌────── 目录卡抽屉 ──────┐
         │  书名:猎人笔记         │  ← 控制面
         │  作者:屠格涅夫         │     (manifest 层)
         │  位置:B-3-12          │
         └───────────┬────────────┘
                     │
                     │ 找书的那一刻
                     ↓
         ┌────── 书架 B-3-12 ──────┐
         │  《猎人笔记》(实体书)    │  ← 执行面
         │  需要时才去取            │     (runtime 层)
         └──────────────────────────┘
```

图书馆在开门时只维护目录卡. 具体的书只在有人借时才从书架拿. 这就是 manifest-first.

## 遗留问题

- `ensureRuntimePluginsLoaded` 如果某个 plugin 的 runtime import 失败, 怎么处理? 是全局炸还是 graceful?
- 75% 的失败里, 是不是有一部分是 plugin runtime 首次加载就抛错, 但控制面认为"plugin 可用"?

## 与其他章节的连接

- ← [[ch02-loop-not-here|第二幕]]
- → [[ch04-two-faces|第四幕]]: agent 跑完了, reply 怎么回到用户? **两张脸**: 可靠的交付 vs best-effort 的可观测性.
