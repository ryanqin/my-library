# 第五夜 · 对等审计——未完成的地图

> 这章要解决什么问题？
> `parity_audit.py` 是一个**反讽的文件**——它的存在意味着"翻译没翻完"。但这个文件教会了翻译者最后一课。

---

## 一

翻译到第五夜，他以为已经结束了。

直到他在原作仓库里看到了一个他没见过的文件——`parity_audit.py`。

打开，里面是一个表：

```python
# Module: runtime.py               | Status: COMPLETE
# Module: tools.py                 | Status: COMPLETE
# Module: permissions.py           | Status: COMPLETE
# Module: session_compaction.py    | Status: TODO (compaction not yet ported)
# Module: mcp_integration.py       | Status: TODO (MCP client pending)
# Module: canvas_host.py           | Status: SKIPPED (out of scope)
# Module: voice_forwarder.py       | Status: SKIPPED (macOS-only)
```

这是**一份翻译者写给自己的债务清单**。

他呆坐了一会儿。他以为作者是完美的移植者，实际上——**作者知道自己没翻完**。而且他知道得非常清楚，精确到每一个模块。

---

## 二

他想起自己在公司里见过的"技术债文档"。那些文档通常写在 Confluence，6 个月没人更新，内容模糊："该模块有历史问题，建议重构"。

`parity_audit.py` 不一样。它是**可执行的文档**：

```python
def audit() -> ParityReport:
    """遍历原版 TS 模块，对比 Python 版的覆盖情况，产出报告。"""
    ts_modules = scan_ts_source(TS_ROOT)
    py_modules = scan_py_source(PY_ROOT)
    return ParityReport(
        complete=[...],
        todo=[...],
        skipped=[...],
    )
```

跑一下 `python -m parity_audit` 就能看到当前的翻译完成度。

翻译者第一次感觉到 "**诚实的工程**" 是什么样子——

- 不是"所有事都做完了"
- 不是"所有未做的事都藏起来"
- 是 "未做的事明确列出来，可检查、可追踪、可接力"

---

## 三

他在第五夜的最后一页写了一段很长的话：

> 我以为翻译的终点是"翻完所有模块"。
>
> 但原作者教我了一件更重要的事——**承认没翻完**。
>
> `parity_audit.py` 把 "TODO" 这个词从松散的注释提升成了**可执行的合约**。
> 它不再是"我打算以后做"，而是"这些是我欠的债，数量和位置精确如此"。
>
> 这不是 laziness，这是 **calibrated honesty**——
> "我知道我做了什么，我知道我没做什么，我知道两者的分界线在哪里。"
>
> 很多人写软件时喜欢装"我全都懂"。parity_audit.py 是一份**谦卑的盾牌**——"我不全懂，但我的不懂是清单上的不懂，不是模糊的不懂"。

---

## 四

翻译的工作到此结束。

他看着自己右侧屏幕上的 Python 代码——`runtime.py`、`tools.py`、`permissions.py`、`main.py`、`commands.py`、`execution_registry.py`、`models.py`、`context.py`、`setup.py`、`bootstrap_graph.py`——十个核心模块，一千多行代码。

他创建了自己的 `parity_audit.py`，抄下原作的格式，标注：

```python
# == 翻译者自己的债务清单 ==
# Module: compaction.py         | Status: TODO (需要先理解 token budget 管理)
# Module: voice.py              | Status: SKIPPED (不懂 macOS 不冒险)
# Module: mcp_client.py         | Status: NEXT (下一个六个月打算做)
```

保存。关掉编辑器。

他站起来，走到阳台。外面已经是清晨五点，天空微白。

他想起五个夜晚自己学到的东西——分派器的朴素，runtime 的 80/20 反转，tools 的跨语言合约，permissions 的沉默，parity 的诚实。

这些不是技术。这些是**对软件的一种态度**：

> **"朴素地做事，诚实地记录，悄悄地保护，小心地放行。"**

他关上电脑，第一次觉得自己有资格去写自己的 agent harness 了——不是因为他会写代码，是因为他**懂了原作者在想什么**。

---

## 五 · 翻译完, 然后呢

第二天醒来, 八点。他做的第一件事是泡茶, 第二件事是新建一个 git repo:

```bash
$ mkdir my-harness && cd my-harness
$ git init
$ touch runtime.py tools.py permissions.py main.py
$ touch parity_audit.py
```

四个文件 + 一份债务清单。这是他给自己定下的"最小骨架"——和原作一样, 朴素到不能再朴素。

他打开 `main.py`, 准备照搬 elif 链——

——**手停在键盘上**。

他想抄, 但他抄不动。

因为他这个 harness **没有 20 个子命令**。他只是想做一个最简单的"运行一次 agent" 的 CLI——他可能只有 `agent` 一个子命令, 加上 `--version` 和 `--help`。**为这一个命令写 elif, 没意义**。

那他应该用 dict？还是干脆不要 dispatcher, 直接 `if args.command == "agent"`?

——**第一夜的所有教训, 在这里**遇到了第一次**用不上**的情况。

他坐在那儿看了五分钟空 buffer。

他突然意识到——**翻译者的工作和创造者的工作不一样**。

翻译者要忠于原作的**意图**, 哪怕变形实现。
创造者要从零问:**我的场景, 原作的意图适用吗?**

——elif 链的意图是"高频被调用 + 自文档"。我的 harness **不一定高频**, 我可能一天才跑两次。那 elif 链的意图就**不适用**——dict, if-else, 甚至完全不要 dispatcher 都行。

他把 `main.py` 删掉, 重新建了一个:

```python
# main.py — my-harness, 一个 agent 子命令而已
import argparse, sys
from runtime import run_agent

def main():
    p = argparse.ArgumentParser()
    p.add_argument("--prompt", required=True)
    args = p.parse_args()
    run_agent(args.prompt)

if __name__ == "__main__":
    main()
```

十行。比原作的 elif 链短得多, 但**符合他自己场景的 empathy**。

他在 `parity_audit.py` 里给自己加了一条:

```
# == 我和原作的"刻意分歧" ==
# Module: main.py
#   原作: 20 个子命令的 elif 分派
#   我的: 单子命令, 直接 ArgumentParser
#   分歧理由: 我的场景不需要分派. elif 的意图(自文档+零成本)
#             在单命令场景退化为"过度设计"。
#   何时该改回 elif: 如果未来子命令 ≥ 5
```

他靠回椅背. 第三天早上的太阳已经斜进窗户.

——**翻译完, 不是终点**. 翻译完, 是你开始能**有理由地不抄**的那个起点。

那一刻他才真的觉得自己**毕业了**。不是因为他懂原作者, 是因为他**第一次知道自己跟原作者哪里不一样, 而且能说出为什么**。

> **"忠诚不是抄。忠诚是你能说出哪里不抄, 以及为什么。"**

---

## 关键概念

- **Parity audit**：把"未完成"做成可执行文档，让翻译/移植的进度可检查、可追踪。
- **Calibrated honesty**：明确知道自己做了什么、没做什么，以及两者的边界。工程美德中最难的一种。
- **谦卑的盾牌**：不是承认无能，是**精确描述无能的范围**。

## 我的理解

- **翻译的终点**不是"全部翻完"，是"清楚知道什么没翻完"。
- **TODO 注释**是松散承诺；**parity audit 脚本**是硬合约。两者的区别是**能不能自动 check**。
- 写一个 harness 的最高境界不是"代码多聪明"，是"我能精确说出自己的边界在哪"。

## 类比

像一个地图制作者。一张好的地图不是"所有地方都画上"——那是神话。一张好的地图是：
- 画清楚**已勘察的区域**
- 用阴影标出**未勘察的区域**
- 在边角写明"以下地带为猜测，不保证准确"

`parity_audit.py` 就是这张地图边角的那行小字。

## 遗留问题

- 我自己的债务清单要怎么执行？我不像原作有完整的对等源码可以 diff——我对比的是什么？
- 6 个月以后我会回来看这个清单吗？还是它会变成另一份被遗忘的 Confluence？

## 与其他章节的连接

- ← [[ch04-permissions|第四夜]]
- ↻ [[ch00-preface|回到序章]]：翻译者的三条义务——保留结构、保留边界、不改行为——回头看，他做到了吗？

---

## 后记

五夜笔记写完。翻译者关上了这本书。

但他知道，**真正的书从明天开始**——他要写**自己的** agent harness 了。

这就是下一本书的序章了。
