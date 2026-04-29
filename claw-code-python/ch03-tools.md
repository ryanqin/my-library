# 第三夜 · 工具册与多层过滤网

> 这章要解决什么问题？
> `tools.py` 维持一份**工具名册**。注册工具、筛选工具、让正确的工具在正确的时刻进入 LLM 视野——怎么做到？

---

## 一

翻译者翻到 `tools.py` 时发现了一个让他意外的设计——**工具不是在代码里注册的，是在 JSON 里注册的**。

```python
def load_tools_snapshot() -> list[ToolSpec]:
    with open(BUNDLED_TOOLS_JSON) as f:
        return [ToolSpec(**entry) for entry in json.load(f)]
```

他愣了一下。"为什么不写个 `@tool` 装饰器？Python 就应该用装饰器啊。"

读了几页，他明白了——**JSON 快照是跨语言的真相**。

原作的 Claude Code 是 TypeScript 写的。如果 tools 用装饰器注册，Python 的翻译版就必须重新写一遍装饰器、再重新注册一次。两边会漂移——TS 新增一个工具，Python 版得跟着改。

用 JSON 快照就不一样。JSON 是合约，TS 和 Python 都读同一份文件，只要 schema 不变，**工具定义同源**。翻译版永远不会落后。

他在边上记了一句：

```python
# JSON over decorator: 合约跨语言，装饰器不跨语言。
```

---

## 二

真正让他停下来的是**多层过滤**。

load 出来的 tools 有 200 个左右。但最终进入 LLM prompt 的只有 5 个。中间发生了什么？

他数了数，**有四层过滤网**：

1. **Permissions 过滤**：黑名单里的工具先剔除（例如 `rm` 在只读模式下被禁）
2. **Channel 过滤**：group channel 里不允许危险工具（`exec` 在 Telegram 群聊里被剔除）
3. **Skill 过滤**：只保留当前 agent/session 绑定的 skill 里声明的工具
4. **Token-routing 过滤**（上一章讲过）：按 prompt 打分取 top-N

"四层过滤网，每一层都是一个**不同维度的约束**。"他这么写。"**安全 × 社交 × 角色 × 相关性**。"

四层的顺序很讲究——先剔除"绝对不能用的"（permissions），再剔除"这里不合适的"（channel），再剔除"你现在不该用的"（skills），最后从剩下的里挑"最可能有用的"（token routing）。

从硬约束到软约束，从绝对到相对。**过滤网的次序反映了约束的强度**。

---

## 三

翻译者还注意到一个小细节——**tools 和 commands 是对称注册的**。

`commands.py` 里有几乎一模一样的结构：

```python
def load_commands_snapshot() -> list[CommandSpec]:
    with open(BUNDLED_COMMANDS_JSON) as f:
        return [CommandSpec(**entry) for entry in json.load(f)]
```

他刚开始以为是"自然巧合"，后来想明白——工具和命令本来就**是同一件事的两种调用方式**。

- LLM 调用的叫 tool
- 用户在 CLI 里敲的叫 command

同一个底层操作（比如"读文件"），可以暴露为 tool（让 Claude 读）或 command（让用户敲 `claw-code read-file xxx`）。它们的 schema、权限、元数据完全同构。

一份 JSON 可以同时生成两种入口——这就是**对称注册**的优雅。

他在笔记最后写下一句自己喜欢的话：

> **"同一件事的两面，就用同一张纸写。"**

---

## 关键概念

- **JSON 快照注册**：工具定义存 JSON 而不是代码里。跨语言合约 > 语言原生 ergonomics。
- **多层过滤网**：permissions → channel → skills → token routing。**从绝对约束到相对相关性**。
- **对称注册**：tools 和 commands 用一份 schema，只是暴露方式不同。

## 我的理解

- **装饰器注册**好看但**让合约跨不出语言**
- **过滤网次序**反映了**约束的优先级**（安全 > 社交 > 角色 > 相关性）
- **对称**不是重复——是在提醒你"这两件事本质一样"

## 类比

像一个图书馆的借阅系统：
- 书目（JSON 快照）是所有书的清单
- 第一层过滤：这本书有没有被销毁（permissions）
- 第二层：这本书适不适合借阅给未成年人（channel）
- 第三层：你的借阅卡允许不允许借这个类别（skills）
- 第四层：按你的检索词，这本最相关（token routing）

而书也可以不被借——直接在阅览室内读（commands）。**同一本书的两种用法，同一个 catalog 记录**。

## 遗留问题

- JSON 快照什么时候重新生成？手工？自动？原作没说清。
- 多层过滤之间的顺序能不能调换？我想不出反例，但直觉告诉我顺序**是故意的**。

## 与其他章节的连接

- ← [[ch02-runtime|第二夜]]：runtime 用 tools，但不管怎么注册 tools
- → [[ch04-permissions|第四夜]]：第一层过滤网——permissions——到底长什么样？
