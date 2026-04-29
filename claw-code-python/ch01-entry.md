# 第一夜 · 大门与分派器

> 这章要解决什么问题？
> `main.py` 里一个 argparse，20 多个子命令。这是"分派器"最朴素的样子。我们能从它身上读出什么？

---

## 一

他打开了 `main.py`。

第一行是 shebang，第二行是 `import argparse`。没有 Click，没有 Typer——**argparse**。在 2026 年还用 argparse 的人，要么是被命运驱使，要么是对可预期的依赖关系有执念。他猜是后者。

然后是 20 多个 `subparsers.add_parser(...)`。

```python
subparsers.add_parser("agent", help="Run an agent")
subparsers.add_parser("login", help="...")
subparsers.add_parser("config", help="...")
subparsers.add_parser("doctor", help="...")
subparsers.add_parser("channels", help="...")
# ... 又 16 行
```

他想起自己第一次做 CLI 时的冲动——"这么多子命令，必须抽象成 plugin system！每个子命令一个文件，动态发现！"

作者没有。作者把每个子命令平铺在同一个文件里，最后用一个 `if args.command == "agent": do_agent(args)` 的长 elif 链分派。

翻译者的手指停在键盘上。他本能地想 refactor，但他想起三条义务里的第二条——**保留边界**。他叹了口气，把 elif 照抄下去。

```python
if args.command == "agent":
    run_agent(args)
elif args.command == "login":
    run_login(args)
# ... 20 个 elif
else:
    parser.print_help()
    sys.exit(2)
```

---

## 二

他抄完 elif 链已经凌晨一点半。屏幕上那二十个 elif 一行一行排得齐齐——他盯着看，越看越觉得**这写得真丑**。

他打开一个新 buffer，开始按本能 refactor：

```python
# DISPATCH = {
#     "agent": run_agent,
#     "login": run_login,
#     ...
# }
# DISPATCH[args.command](args)
```

写到一半他停下了。

倒不是因为三条义务——是因为他突然想起一件事。**作者是从 TS 翻过来的**。TS 那边写 `if/elif` 比写 dict 麻烦得多——TS 里 dict-based dispatch 反而是惯用法。**作者明明可以那样写，他偏偏选了 elif**。

——这不是懒，是**故意**。

他把 refactor buffer 关掉，把光标挪回原版 elif 链，多看了两分钟。

然后他做了一件事——打开终端，给两份分别 benchmark 一下启动时间：

```
$ time claw-code --version
real    0m0.041s

$ time some-plugin-cli --version
real    0m0.310s
```

41ms vs 310ms。

他靠回椅背。**七倍**。

他想起 `claw-code` 这个工具被设计来跑在哪里——shell 脚本里、git hook 里、CI pipeline 里、IDE 自动补全的回调里。**每秒可能被调用几十次**。每次多 270ms 意味着——

```
shell 脚本调用 1000 次     → 多浪费 270 秒 = 4.5 分钟
CI pipeline 跑十次          → 多浪费 2.7 秒
IDE autocomplete 一天      → 用户感知到的"卡"
```

而且他注意到一件事——**elif 链是自文档的**。读 `main.py` 你一眼看到这个工具有什么命令。plugin 系统会把你送到 20 个不同的文件去，每次跳来跳去。

朴素不是笨。朴素是**对使用者的 empathy**——既包括跑这个 CLI 的脚本作者，也包括读这份代码的下一个翻译者。

他把光标移到 elif 链最上面，加了一行注释:

```python
# 不是没想过 dispatch dict。dict 表面上更"干净"，
# 但 elif 链 (1) 启动成本为零, (2) 一眼能读出有哪些命令。
# 给高频被调用的 CLI 用 elif，是 empathy，不是懒。
```

——他觉得自己第一次**真的**理解了这个文件。

---

## 三

那天晚上他还翻了一段小插曲——`--version` 的快速路径。

```ts
// main.ts
if (isRootVersionInvocation(argv)) {
  console.log(`claw-code ${VERSION}`);
  process.exit(0);
}
```

这行代码跑在所有解析逻辑**之前**。作者不让你为了看一个版本号就把整个 CLI 初始化一遍。

他翻译成 Python 时发现 argparse 默认行为是"先 parse 所有 subparser 再处理 `--version`"，就是说**它会付出完整的启动成本**。他在这里破例了——偷偷把 version fast-path 提前。

```python
if "--version" in sys.argv or "-V" in sys.argv:
    print(f"claw-code {VERSION}")
    sys.exit(0)
```

他在边上加了一行注释：

```python
# 原作的 fast-path。翻译本来要保留 argparse 的默认行为，
# 但这里的设计意图比具体实现更重要——让一条 --version 便宜得不能再便宜。
```

翻译者第一次为了"忠于原作的**意图**"而违背了"忠于原作的**实现**"。他觉得自己毕业了一小步。

---

## 四 · 关于破规矩

但他立刻意识到自己刚刚做了一件**违反义务第二条**的事——保留边界。

他怔了一下。

序章的三条义务是不是说着玩的？还是说，**有的边界可以破, 有的不可以破**？

他在笔记本里给自己写了三条**"什么时候可以违背义务"**:

```
1. 当原作的"实现"是被宿主语言绑住的, 而不是作者主动选的
   (TS 里 argparse 没有, version fast-path 必须手写)

2. 当违背"实现"才能保住"意图"
   (作者要 --version 便宜, Python argparse 的默认行为反而违背了这个意图)

3. 当违背的代价 < 保留的代价
   (这里多写三行 Python, 换来 270ms 的启动加速)
```

第三条最关键——**所有破规矩都要付得起代价**。如果哪天他想 refactor 那个 elif 链, 跑 benchmark 发现性能差一千倍, 他自己都会觉得不值得.

义务不是教条, 是**默认值**。默认是保留实现, 例外要有理由。

他把这三条钉在 `ch01-entry.md` 笔记的末尾, 给自己以后破规矩时当**许可证**——也当**约束**。

---

## 关键概念

- **CLI 分派器 (dispatcher)**：把"用户敲什么命令"映射到"跑哪段代码"。最朴素的形态就是 `if/elif/else`。
- **启动成本 (startup cost)**：一个 CLI 从被 shell 调用到输出第一个字符之间花的时间。当你的工具被自动化管道高频调用时，它是第一类性能指标。
- **fast-path**：在主流程之前处理"便宜的特殊情况"——`--version`、`--help`、缓存命中。

## 我的理解

分派器的哲学：
1. **启动便宜**比**代码漂亮**重要
2. **自文档**（一眼看到有哪些命令）比**抽象好**（所有命令同一个 shape）重要
3. **fast-path** 不是性能 hack，它是体现 empathy 的具体形式

## 类比

像地铁站的大门——不是每个乘客都需要你检票、打印票根、广告一张大大的欢迎图。你只要让他能以最短的路径刷卡进站。所有的优化，先从"让 90% 的人快"开始。

## 遗留问题

- 如果子命令数量变成 200 呢？elif 还扛得住吗？那时候是该拆 plugin 了——但门槛在哪？

## 与其他章节的连接

- ← [[ch00-preface|序章]]
- → [[ch02-runtime|第二夜]]：从入口进来，第一个要抵达的是 runtime——心脏的位置。
