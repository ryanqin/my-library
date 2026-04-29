# 工具国 · 描述的魔法

> 这章要解决什么问题？
> LLM 凭什么选 tool？答案只有一个字——**描述**。`tool_choice` 又是什么？MCP 在哪个位置？

---

## 一

旅行者从编排国走出来，跨过一条木桥，就到了工具国的首都——MCP 镇。

镇口有一块木牌：

> **所有的工具, 都活在它们的描述里.**

旅行者第一次看见这句话时不理解。读了几天他才懂——

LLM 看不见你的代码。LLM 看不见你的函数名。LLM 看不见你的类型系统。
LLM **只看见你写的那段 description**。

所以——

- 如果你的工具叫 `search_files`，description 写"搜索文件"，LLM 有时用有时不用
- 如果你把它描述成"当用户提到查找、定位、找某个文件或目录的内容时使用"，LLM 99% 会选对

**同一个工具，描述换一下，选中率差 30%+**。

旅行者把这一条抄进笔记：

> **工具的能力在代码里，工具的存在感在描述里。**

---

## 二

MCP 镇的市集上，一个摊主在卖三种"选择方式"。

- 牌子 A:"`tool_choice: auto`" — 让 LLM 自己决定要不要用 tool
- 牌子 B:"`tool_choice: any`" — 强迫 LLM 至少用一个 tool（但选哪个它自己定）
- 牌子 C:"`tool_choice: {name: 'xxx'}`" — 强迫用这个特定 tool

旅行者指着 B 和 C 问:"这俩什么时候用?"

摊主笑了:"B 是'我给你一堆 tool 但你总得挑一个干', 比如结构化输出——**LLM 用不上 tool 就不结构化了**, 你强制它挑一个. C 是'这一步必须做这个', 比如合规检查——**不容选择**."

A 是日常. B 是候补强制. C 是 pinpoint 强制. 三档从松到严.

---

## 三

镇最深处有一座图书馆。门口写着——**MCP (Model Context Protocol)**.

旅行者推门进去. 里面是**一排排外部服务的目录卡**:

- "GitHub 的 issue 操作, 在这"
- "Linear 的项目查询, 在这"
- "内部数据库的只读访问, 在这"

"这不就是 tool 吗?" 他问图书管理员.

"是 tool," 管理员说, "**但是跨 agent 共享的 tool**."

"你可以自己给你的 agent 写一个 search_github tool——但你写出来的只有你自己用. 如果你把它发布成一个 MCP server, **别人的 agent 也能直接接入**."

"MCP 不是新 tool," 管理员重复一遍, "**是 tool 的跨边界合约**."

旅行者在笔记上写:

> Tool 本地写 → MCP 跨边界共享 → 描述决定 LLM 选不选

三个层次, 同一件事的规模放大.

---

## 四

最后一块学习内容是**结构化错误响应**.

工具国的反模式墙上贴着:

> ❌ tool 报错返回 "Something went wrong"
>
> ✅ 返回 `{ isError: true, errorCategory: "network_timeout", isRetryable: true, message: "..." }`

第一种写法 agent 看了一脸懵, 会把它当成 assistant 文本一起读. 第二种写法 agent 立刻知道:**这是个 tool 级错误, 分类是网络超时, 可以 retry**.

> **错误要结构化. 自由文本报错 = 把判断责任推给 LLM.**

---

## 关键概念

- **Tool 描述决定选择**：LLM 选 tool 的唯一机制是 description。代码、名字、schema 都比不过 description 的细腻。
- **`tool_choice` 三档**：`auto`（日常）/ `any`（候补强制）/ `{name: "x"}`（定点强制）。
- **MCP = Tool 的跨 agent 合约**：不是新能力，是**共享层**。
- **结构化错误响应**：`isError` + `errorCategory` + `isRetryable` — 让 agent 能做确定性决策。

## 我的理解

- Tool 设计的 80% 工作是在**写 description**, 不是在写代码
- `tool_choice` 是给你**从概率选择 → 确定性选择**的渐进开关
- MCP 让你**不再是自己用自己的 tool**, 是**整个 agent 生态用彼此的 tool**

## 类比

Tool 像**餐厅菜单**:
- 代码是后厨
- description 是菜单描述——"温润的汤,适合饥饿的夜"
- LLM 是点餐的客人, 它只看菜单

`tool_choice: auto` = "客人自己点"
`tool_choice: any` = "你必须点一道"
`tool_choice: {name: x}` = "今天只有这道菜"

MCP 是**全城餐厅共享的菜单库**——不只我家的菜,所有城里的馆子的菜都列进来了,任何一个顾客(agent)都能点.

## 遗留问题

- 一个 tool 的 description 要多长才够? 我猜"1-3 句",但场景题里可能会考 edge case.
- MCP server 之间怎么避免 description 冲突? 两个 server 都声明了一个 search——LLM 怎么分?

## 与其他章节的连接

- ← [[ch01-orchestration|编排国]]
- → [[ch03-cave|洞窟国]]：tool 搞清了, 但 agent 在什么项目、什么目录、什么角色下跑? 那是 CLAUDE.md 的事.
