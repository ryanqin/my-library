# 第五幕 · 两扇门

> 这章要解决什么问题？
> Telegram 消息从**哪条路径**到达 agent? 原来**不是一条, 是两条**. 两条路径在哪里汇合, 为什么要分岔?

---

## 一

凌晨六点十五分. 他追进了入站路径.

一开始他以为很简单——`agentCommandFromIngress` 这个入口. 他 grep 了一下——

```bash
grep -r agentCommandFromIngress src/
```

11 个文件, 都在 `src/gateway/` 下.

"所以 gateway 是入站主入口."他想.

然后他打开 Telegram 插件看看——在 `extensions/telegram/src/` 的 `bot-message-dispatch.ts`——

```bash
grep -r agentCommand extensions/telegram/src/
```

**没有结果**.

"Telegram 不走 `agentCommand`?"

他愣了三秒, 开始想办法查 Telegram 到底怎么调 agent 的.

---

## 二

他顺着 Telegram 的代码往下追.

```
Telegram grammy bot.on("message", ...)
  ↓
handleInboundMessageLike (dedupe/allowlist)
  ↓
processInboundMessage (text fragment buffer / media group buffer)
  ↓
inboundDebouncer.enqueue → flush
  ↓
createTelegramMessageProcessor
  ↓
dispatchTelegramMessage
  ↓
dispatchReplyWithBufferedBlockDispatcher (openclaw/plugin-sdk/reply-dispatch-runtime)
  ↓
dispatchInboundMessage (auto-reply/dispatch.ts)
  ↓
dispatchReplyFromConfig (auto-reply/reply/dispatch-from-config.ts)
  ↓
runReplyAgent (agent-runner.runtime.ts)
  ↓
loadPiEmbeddedRuntime() → runEmbeddedPiAgent
```

**最后一行**让他瞪大了眼睛——

"**直接跳到 runEmbeddedPiAgent**?"

"绕开了 agent-command.ts?"

他再看另一条路径(Gateway node events):

```
server-node-events.ts
  ↓
agentCommandFromIngress
  ↓
agentCommandInternal (agent-command.ts:424)
  ↓
runAgentAttempt
  ↓
runEmbeddedPiAgent
```

**两条路径, 殊途同归**——都最终进了 `runEmbeddedPiAgent`. 但前面的"谁说话, 说什么, 回谁"**完全不同的路径**.

——他 又 多 看 了 一 步: **它们 在 哪 里 真的 汇合?**

他 跳 进 `runEmbeddedPiAgent` 的 函数 签名:

```ts
export async function runEmbeddedPiAgent(
  params: EmbeddedAgentRunParams
): Promise<EmbeddedAgentRunResult>
```

——`EmbeddedAgentRunParams`. **就 是 这 个 类型**.

```ts
export interface EmbeddedAgentRunParams {
  agentId: string;
  prompt: PromptInput;
  session: SessionContext;
  runtime: RuntimeEnv;
  delivery: DeliveryPlan;
  config: ResolvedAgentConfig;
  // ...
}
```

两 条 路径 在 进 这个 函数 之前, **必须 各自 把 自己 的 输入 normalize 成 这 个 接口**.

```
路径 A (Telegram):
  MsgContext  →  buffer  →  resolve agent from config
              →  build EmbeddedAgentRunParams  ←──┐
                                                  │
路径 B (Gateway):                                 │  汇合点
  IngressOpts  →  validate  →  agent from opts    │
              →  build EmbeddedAgentRunParams  ←──┘
                                                  ↓
                                       runEmbeddedPiAgent
```

——**汇合点 是 一个 typed contract, 不是 一个 函数**. 两 条 路径 都 必须 把 自己 的 输入 翻译 成 `EmbeddedAgentRunParams`. 翻译 完 之后, 它们 就 是 等价 的.

> **接口 是 真正 的 汇合 点. 函数 是 接口 的 第一个 消费者.**

这 也 解释 了 一件 事——**测试 怎么 写**? 直接 mock 一份 `EmbeddedAgentRunParams`, 不用 模拟 真 Telegram 也 不用 启 gateway. 接口 就 是 测试 边界.

---

## 三

他把两条路径画成一张图:

```
        ┌────────── 入站源头 ──────────┐
        │                                │
  Telegram/Discord                CLI / Gateway / iOS Share
  WhatsApp/Slack...               Voice event
        │                                │
        ▼                                ▼
  ╔═══════════════╗              ╔═══════════════╗
  ║  路径 A        ║              ║  路径 B        ║
  ║  (channel)    ║              ║  (ingress)    ║
  ╠═══════════════╣              ╠═══════════════╣
  ║ MsgContext    ║              ║ Opts object   ║
  ║ 驱动          ║              ║ 驱动          ║
  ║ agent 从      ║              ║ agent 从      ║
  ║ channel config║              ║ opts 显式指定  ║
  ║ 选出          ║              ║              ║
  ║ 多层 buffer   ║              ║ 不 buffer     ║
  ╚═══════╤═══════╝              ╚═══════╤═══════╝
          │                              │
          └──────────────┬───────────────┘
                         ▼
            ╔══════════════════════╗
            ║  runEmbeddedPiAgent  ║
            ║  (两条路径汇合)        ║
            ╚══════════════════════╝
```

---

## 四

他理解**为什么要两条路径**.

"如果只保留路径 B——所有东西都走 `agentCommandFromIngress`——那 Telegram 的 channel-specific 逻辑(text fragment buffer/debounce/group allow)**要塞进 opts**. 那个 opts 会变成 monster."

"如果只保留路径 A——所有东西都走 `dispatch-from-config`——那 CLI 调用(`openclaw agent --message`) 会被迫先变成一个假的 channel event, 再绕回来. 徒劳."

**分两条路径的本质**:
- 路径 A 解决"channel 原生事件直接驱动 agent". 富上下文, 多 buffer, config-selects-agent.
- 路径 B 解决"已经归一化成 opts 的事件驱动 agent". 扁平输入, 直接选 agent.

**两种 "绑定" 问题, 两种解法. 最后汇合执行.**

他在笔记里写:

> **入站不是出站的反向**.
>
> - 出站是执行(prompt → loop → reply)
> - 入站是绑定(谁 → 哪个 agent → 怎么回)
>
> 绑定问题有两种独立来源(channel 原生事件 vs 归一化 ingress), 所以需要两条独立路径.

---

## 五

他又追了一层——在 `dispatch-from-config.ts` 里有一段:

```ts
if (text && !isCommandLike) {
  if (text.length >= TELEGRAM_TEXT_FRAGMENT_START_THRESHOLD_CHARS) {
    // buffer 起来等下一段
    startFragmentBuffer(...)
    return;  // 先不跑 agent, 等合并
  }
}
```

**这是 Telegram 的 4096 字符限制的处理**——用户粘贴一段 8000 字符的长文本, Telegram 会拆成 2 条发过来. 如果 agent 收到第一条就跑, 它答得很懵; 所以先 buffer, 等第二条来, 合并完再跑.

"这**不是 agent 的问题**, 是 Telegram 平台的 quirk." 他说. "**放在 channel 插件里解决, 不污染 agent 核心**."

这和 Claude Architect 笔记里的 **Hooks vs Prompt** 同构——**平台行为用代码处理, 不让 agent 去 reasoning 解**.

---

## 六

凌晨六点四十. 天开始亮了.

他回到 Grafana, 现在终于知道**要看哪些指标**.

- 路径 A 的失败率 (按 channel 分组)
- 路径 B 的失败率
- 两者汇合后 runEmbeddedPiAgent 的失败率

图表一拉——

**路径 A 的 Telegram 分组**在过去 30 分钟里掉了 82%. 其他 channel 正常.

**Telegram 独有的问题**. 他几乎可以肯定——结合第三幕 manifest bug 的发现——是 **刚 deploy 的 Telegram plugin 新 manifest 有问题**.

他打开最近的 PR 列表——

**两小时前, 有人合并了一个 Telegram plugin 的 config schema 重构**. 改动里**修改了 manifest**.

他叹了一口气. **找到了**.

---

## 关键概念

- **两条入站路径**: A (channel-native, MsgContext 驱动) vs B (ingress, opts 驱动).
- **汇合点**: `runEmbeddedPiAgent`.
- **入站 ≠ 出站反向**: 绑定问题 vs 执行问题.
- **平台 quirk 归 channel 处理**: Telegram text fragment buffer 不污染 agent core.

## 我的理解

- **两条路径不是冗余**, 是**两种 input contract 的分离**.
- Debug production agent **必须先知道用户是从哪条路径进来的**——否则 error 定位会偏.
- **Channel plugin 的 80% 代码**处理平台自身复杂度, **10% 填 agent 合约**.

## 类比 · 机场的两扇门

```
        ┌──── 机场航站楼 ────┐
        │                     │
   国内到达门              国际到达门
   (路径 A · 本地)          (路径 B · 经标准化)
        │                     │
  值机/拖行李/           海关/入境盖章/
  TSA 检查               手续归一化
        │                     │
        └──────────┬──────────┘
                   ▼
             登机口 (runEmbeddedPiAgent)
             两扇门进来的人, 到这里都是"乘客"
```

两扇门之所以分开, 是因为**入场的处理流程不同**. 一旦到登机口, 就都是"待上飞机的乘客", 执行路径相同.

## 遗留问题

- 两条路径能不能**合并成一条抽象**? 理论上可以——把 MsgContext 归一成 opts. 但会引入一个"假 channel layer". 不优雅.
- 路径 A 是否应该也支持 `opts.modelOverride`? 目前 config 决定 agent, 用户没法临时切模型.

## 与其他章节的连接

- ← [[ch04-two-faces|第四幕]]
- → [[ch06-dawn|尾声]]: 天亮了. Bug 定位了. 我回头看整晚追的东西, 想写一篇给自己的反思.
