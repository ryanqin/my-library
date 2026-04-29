# 序章 · 凌晨三点的告警

> 这章要解决什么问题？
> 从一个真实的 incident 切入——为什么 production-grade agent harness 的复杂度**远超**一个玩具 loop？

---

## 一

凌晨三点。他的手机震动了两下。

屏幕上是 PagerDuty 的红色通知——

> **P1 · openclaw-gateway · High error rate on /agent/ingress**
> 过去 5 分钟, 32 个 Telegram 入站消息, 只有 8 个 agent 回复成功.

他坐起来。窗外是黑的。妻子翻了个身嘟囔了一句什么, 又睡着了.

他走到书房, 打开笔记本. 屏幕亮起, 映出他的脸——**一个三十多岁, 见过太多凌晨三点的 oncall 工程师**.

---

## 二

他打开 OpenClaw 的监控面板.

- Inbound rate: **正常**
- Gateway latency: **正常**
- Agent lifecycle.end 事件: **只有 25%**

有 75% 的 agent run **没有正常结束**. 它们在某个地方**卡住了**——既没报错, 也没完成, 像一个被吞掉声音的话筒.

他心里一凉. 这种 bug 是最难的——不是"A 坏了", 是"A 没坏但 B 也没开始".

他打开 VS Code, 开始看代码. 凌晨三点十五分.

---

## 三

他在笔记本边上写下今晚要追的问题:

> **从 "一条 Telegram 消息进来" 到 "用户收到 agent 回复", 中间到底发生什么?**
>
> 如果这条链断了, 断点可能在哪?

他想了想, 又补了一行:

> 我以前觉得这事简单. **今晚开始, 我要重新评估.**

---

## 四

他的上一家公司做 chatbot. 那个 chatbot 很简单——用户说话, 发给 OpenAI, OpenAI 回话, 发回用户. 他能画出来:

```
Telegram → webhook → OpenAI API → Telegram 回复
```

四层. 他以为 agent harness 就是**在中间加一个 loop**.

"OpenAI 改成 Claude, 加个 tool-use loop, **不就 agent 了吗?**"

他 一边 想, 一边 在 终端 里 敲:

```bash
$ cd openclaw-source
$ ls src/
```

按下 回车.

——屏幕 一下子 满了.

他 数 不清 列出来 多少个 子目录. 拉到 终端 的 顶端, 滚轮 往上 推, 推, 推, 还是 没看 到 头. 他 切到 大写 字母, 让 ls 排序——

```
src/
├── agents/
├── auth/
├── channels/
├── compaction/
├── delivery/
├── events/
├── gateway/
├── ingress/
├── lifecycle/
├── manifest/
├── memory/
├── models/
├── plugins/
├── prompts/
├── retry/
├── routing/
├── runtime/
├── sdk/
├── sessions/
├── skills/
├── ...
```

他 数 到 第 60 个 子目录, 停 下来.

——**60 个子目录**.

"……搞 什 么 啊."他 听 见 自己 嘴 里 漏 出 一句.

他 又 切 到 `extensions/`. 23 个 channel + 20 多 个 provider. 再 grep 一下 `*.runtime.ts`——47 个 懒加载 壳. 再 grep 一下 retry——5 种 策略.

他 靠 回 椅 背, 在 心里 把 自己 那 张 老 PPT 上 "Telegram → webhook → OpenAI → Telegram" 的 四 层 图 撕 掉.

"加一个 loop" 和这个**差了一万倍的复杂度**.

他深呼吸. 敲下第一个搜索命令:

```bash
grep -r "Telegram" src/agents/command/
```

他要追的是一条路径——**从 Telegram 的 inbound webhook 到 agent loop 到 delivery 到 Telegram 的 outbound**. 中间每一个节点, 他都要看清.

这一夜, 他的笔记本分成两列——

**左列是正在跑的 Grafana 仪表盘**.
**右列是源码, 一层一层往里读.**

今晚不是 debug. 今晚是**他第一次真正理解 production agent harness 的一夜**.

---

## 关键概念

- **Production agent harness**：不是"玩具 + 一个 channel"。真实系统会有多 channel 分发、多 provider 容灾、manifest-first 控制面、双输出通道、两条入站路径、五种 retry 策略。
- **Lifecycle.end 覆盖率**: 一个 agent run 是否"正常结束"的工程指标. 不是 error rate, 是 **completion rate**.
- **懒加载 runtime**: `.runtime.ts` 壳——不 import 就不 load. 启动成本优化.

## 我的理解

- "玩具 agent" 和 "生产 agent" 差的**不是 loop**, 是**loop 外围的所有东西**.
- 最恶心的 bug 不是 error, 是**没有结束也没有报错的状态**.
- Debug 一个陌生系统的第一步是**画出数据流**, 不是猜因果.

## 类比

像一个小镇医生搬到大城市急诊室的第一个夜班——

- 小镇: 一个病人来, 自己看完送走.
- 急诊: 分诊台 → 内科/外科分流 → 多科会诊 → 影像 → 化验 → 决策 → 住院登记 → 病房转运 → 医嘱开立 → 护理执行.

"10 个病人同时进来, 70% 的病人在某一步停住了, **不是所有卡点都在同一个环节**"——这是他今夜面对的 bug.

## 遗留问题

- Lifecycle.end 为什么没触发? 要一直追到 agent loop 内部.
- 75% 全部失败? 还是 75% 在不同位置失败? 这两个的解法完全不同.

## 与其他章节的连接

- → [[ch01-brain|第一幕]]: 从入口开始追. 第一个要看的是 **`agent-command.ts` 那个 1111 行的大脑**.
