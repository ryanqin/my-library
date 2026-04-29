# 第四夜 · 守门人的黑名单

> 这章要解决什么问题？
> `permissions.py` 是一份沉默的黑名单。为什么不是"白名单"？黑名单的哲学是什么？

---

## 一

第四夜他打开 `permissions.py`，文件开头就是一行注释：

```python
# Denylist, not allowlist.
# If a tool is not explicitly denied, it is allowed.
```

翻译者起初不理解。安全不是应该用白名单吗？"只有明确允许的事情才能做"——这是教科书的标准答案。

读了几页他才意识到——**这不是一个 security sandbox。这是一个 agent orchestration**。

"Agent 要的是能做事的自由。" 他这么记笔记。"**安全放行，不是为了防黑客，是为了不让 agent 被废掉**。"

黑名单和白名单的区别在这里不是安全强度，而是**默认态的哲学**：

- 白名单的默认态是 "不能做任何事" → agent 废。
- 黑名单的默认态是 "什么都能做" → agent 能跑，但要把"确定不能做的"圈出来。

Claude Code 不是在宿主机上保护你不被 agent 入侵（那是容器化的事），它是在**让 agent 有能力**的同时，**把几种具体的灾难型工具圈出来**（比如在只读模式下禁用 `rm`）。

---

## 二

翻译者在 permissions.py 里看到一个数据结构叫 `DenyRule`：

```python
@dataclass(frozen=True)
class DenyRule:
    tool_name: str
    reason: str
    mode: Literal["always", "readonly", "group"]
```

三个模式，三种典型场景：
- `always`：无论什么情况都禁（比如 `format_disk`）
- `readonly`：只读模式下禁（比如 `edit_file`）
- `group`：群聊里禁（比如 `exec_shell`）

三个模式之外，没了。**没有细粒度 RBAC，没有策略引擎，没有 OPA**。

他想起自己去年在上一家公司维护的那套权限系统——RBAC + ABAC + 策略 DSL，运行在一个独立的 policy 服务里。要 200ms 的 RPC，3 个团队维护。

对比起来 permissions.py 就是 50 行的一个文件，一个 dict lookup。

"但它能满足 90% 的 agent 需求。" 翻译者写道。"剩下 10%——比如企业 compliance、audit trail——**不是 harness 该管的事**。那是 production platform 的事。"

这是他翻译到现在第一次觉得：**原作者知道自己在做什么，也知道自己不做什么**。

---

## 三

守门人的黑名单是沉默的——用户看不见它。

permissions.py 在 tool-filtering 的第一层被调用，把几个 tool 从 list 里 pop 掉，没有日志、没有提示。LLM 根本不知道自己"本来有" `exec_shell` 这个工具。

翻译者第一次翻译这段时忍不住想加一段 `log.debug(f"tool {name} denied: {reason}")`。

他停住了手。

"如果加日志，每一次过滤都会产出噪音。而且 agent loop 一旦看到日志，可能会**把 denial 当成一个信号重新 reason**。" 他心里推演，"沉默才是正确的——**让 agent 连'这个 tool 被我禁了'都不知道**。"

沉默是一种权力形态。**你不告诉 agent 有什么没有，它就不会去要**。

他在笔记最后写道：

> 黑名单 + 沉默 = 默认能做事 + 禁事不露痕迹。这是 Claude Code 对 agent 态度的缩影——**信任它，同时悄悄护着它**。

---

## 四 · 黑名单的代价

但翻译到 `permissions.py` 第 47 行时，他撞上了一个**矛盾**。

```python
# Note: tools registered AFTER first_check() are not denied
# even if they match a deny rule.
```

——什么意思？

他往下读 `_apply_denies()` 的实现，看明白了。**denylist 只在 agent loop 启动那一刻校验一次**。校验后注册的新 tool（比如通过 MCP plugin 在中途加进来的），**不会经过黑名单**。

那如果 agent 跑到一半，某个 MCP server 上线了一个新的 `exec_shell_v2` 工具——它**不在原始 denylist 里**——agent 就**真的能用它了**。

翻译者愣住。

这不是 bug。这是**黑名单哲学的代价**。

白名单的代价是 agent 被废。黑名单的代价是——**未知的工具默认被允许**。前者是"我什么都不能做", 后者是"我能做的事永远比你想的多一点"。

他翻到 `commits` 里看作者怎么处理这个——**作者没处理**。注释里就一行字:

```
// We accept this risk. The harness is not a sandbox.
```

——**我们接受这个风险。这个 harness 不是沙盒**。

翻译者读到这一行的时候, 第一反应是想加一段防御代码——比如让所有新注册的 tool 都重新过一遍 denylist。第二反应是停手。

如果他加了，这就**不再是 Claude Code 的 harness 了**——这是他自己的 harness。是他自己对"安全"的定义，不是原作者的。

他在笔记里加了一段，第一次用红笔:

> ⚠ **黑名单 ≠ 安全**。它是"agent 默认能干事 + 几样灾难型工具圈出来"。
> ——MCP 中途加 tool 这种事, 黑名单管不到。
> ——这件事**不应该靠 harness 解**。这件事应该靠 **deployment 层**——容器隔离、网络出口控制、文件系统 readonly mount。
>
> 黑名单不是错的, 它只是**只能护住一段**。要护住全段, 你需要**纵深防御**——但纵深的事不是 harness 的事。

他把这段贴在 permissions.py 文件顶上做注释——给六个月后的自己看。也给将来 fork 这份代码的人看。

---

## 关键概念

- **Denylist**：默认允许，显式禁止某些。适用于"agent 要能跑"而不是"绝对不能越界"的场景。
- **三种 deny mode**：`always` / `readonly` / `group` — 覆盖绝大多数 harness 场景。
- **沉默过滤**：被禁的工具对 agent 不可见，不暴露 denial 信号。

## 我的理解

- **Agent 的 permissions ≠ Kubernetes 的 RBAC**。前者是"让它能干活"，后者是"防它越权"。目标不同，形态自然不同。
- **复杂策略引擎**不属于 harness——它属于部署它的那一层。
- **沉默**是一个微妙的设计选择：不是"更省"，是"不干扰 reasoning"。

## 类比

像一个父亲陪孩子第一次学做菜。厨房门上没挂"不许碰刀"的牌子——那样孩子会好奇去摸。父亲只是**悄悄把刀收进抽屉**，孩子看不见刀的存在，自然不会要。

## 遗留问题

- 如果要加 audit（审计哪些 tool 被禁了），怎么加而不破坏"沉默"？可能要一个**旁路日志通道**，不经过 agent 的 observation。
- `group` mode 说明 permissions 知道 channel——耦合了？好像是无法避免的耦合。

## 与其他章节的连接

- ← [[ch03-tools|第三夜]]：permissions 是多层过滤的第一层
- → [[ch05-parity|第五夜]]：最后一夜——翻译完成以后，我看到的**未完成的地图**。
