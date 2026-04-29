# 第二夜 · Token 路由的迷宫

> 这章要解决什么问题？
> `runtime.py` 做三件事——token 路由、会话引导、多轮循环。三件事连成一个心脏。

---

## 一

第二天晚上他翻开了 `runtime.py`。

这个文件只有 200 行，但翻译者用了三个小时才看完第一遍。

第一件事**让他愣住**的是：runtime 里有一段**打分匹配**。

```python
def token_routed_query(prompt: str, candidates: list[Command]) -> Command | None:
    # 把 prompt 分词，逐个 token 对 candidate.trigger_tokens 打分
    ...
```

"为什么要打分？"他自言自语。"LLM 的 tool_use 决策不是由 LLM 自己做的吗？"

他读下去才明白——这不是"让 LLM 选 tool"，而是**在 LLM 启动前**，根据用户 prompt 的**字面 token**，预筛选出"可能有用的 tool"。

比如用户说 "edit my config file"，"edit" 和 "file" 两个 token 命中 `edit_file` 工具的触发词；"config" 命中 `config_read` 工具。其他 200 个工具的分数都是 0。

于是 LLM 的 system prompt 里只注入最高分的 N 个工具。——**Token routing 是一个 prompt-size 优化**。它让 Claude 每次只需要看 5 个工具的描述，而不是 200 个。

翻译者在旁边写了一行注释：

```python
# 这不是 tool dispatch, 是 tool pre-filtering.
# 目的: 把 2000 个字的 tool 描述压缩到 200 字.
# 代价: 如果打分错, 该用的 tool 看不见 -> 需要 fallback 机制.
```

---

## 二

第二件让他愣住的事：**会话引导**。

原作里有个函数叫 `bootstrap_session`，它做的事看着繁琐——读 CLAUDE.md、扫描 cwd、塞进一段"你现在在什么目录"的 context、拼接 system prompt。

翻译者本来想把这一坨塞进一个大函数里就完事了。但他注意到原作用了 7 个小函数：

```python
resolve_workspace_dir()
load_claude_md()
build_cwd_context()
resolve_permissions()
build_skills_snapshot()
build_agent_metadata()
assemble_system_prompt()
```

一开始他嫌碎。后来他明白了——**每一个小函数都是一个可插入的 hook**。如果有一天你想换掉 "CLAUDE.md 的读取规则"，你只改 `load_claude_md()`，其他 6 个函数不动。

> 作者在每一个看起来多余的分解里，都在预留"以后要改的地方"。

翻译者在日记里补了一句："朴素不等于粗糙。朴素是**明天我还要改**的承诺。"

---

## 三

第三件让他愣住的事：**多轮循环的优雅**。

Claude 的 tool-use loop 本质上是：

```
while stop_reason != "end_turn":
    response = claude.generate(messages, tools)
    append(response)
    if response.stop_reason == "tool_use":
        for call in response.tool_calls:
            result = dispatch(call)
            append(tool_result(result))
```

但原作用了一个闭包+递归的形态让这段代码**只出现一次**——无论是 fallback 重试还是正常推进，都走同一份循环体。翻译者抄下来时忍不住感慨：

"一个玩具版的 agent loop 真的只要这么多行。"

他想起他之前读过一篇博客，说"写一个 agent 要 5000 行"。那是错的——**写一个 agent 的产品化外围**要 5000 行。Loop 本身可以 20 行写完。

他把这句话钉在了书桌的软木板上。

---

## 关键概念

- **Token routing**：根据 prompt 的字面词汇打分，选出最可能用到的 tool。是 **prompt-size 优化**，不是 tool dispatch。
- **Bootstrap session**：把工作目录、CLAUDE.md、权限、skills、元数据组装成 system prompt 的流程。多个小函数串起来，每个都是未来可插的 hook。
- **Agentic loop**：`stop_reason` 状态机驱动。核心循环 20 行就能写完；剩下的 4980 行都是"生产化"。

## 我的理解

runtime.py 告诉我三件事：
1. **预筛选 > 让 LLM 全看**（token routing 是 attention saving）
2. **小函数不是拆分主义，是未来修改性**（7 个 bootstrap 小函数都是 hook point）
3. **Loop 本体便宜，外围昂贵**（记住这个 80/20 反转）

## 类比

runtime.py 像一个老中医出诊前的仪式——
- 先"望"（token routing：看看这个人可能什么病）
- 再"闻问切"（bootstrap：采集场地、病史、允许的疗法）
- 再开药（agentic loop：每开一味药看反应再改下一味）

他做的 90% 是"出诊前"和"观察之间"，真正开处方的动作反而是最短的那一瞬。

## 遗留问题

- Token routing 的打分公式原作里有没有？我没看到——是启发式还是学习出来的？
- 闭包+递归 vs 显式 while——在 Python 里哪种更合适？我翻译时用了 while，但心里不确定。

## 与其他章节的连接

- ← [[ch01-entry|第一夜]]：大门让他进来
- → [[ch03-tools|第三夜]]：runtime 用到的 tools 是怎么来的？下一夜讲。
