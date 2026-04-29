# 第六夜 · 三楼的桥与四楼的彩排厅

> 这章要讲什么?
> 三楼有一道玻璃走廊, 横跨马路, 通向对面的一栋. 四楼是一间没有观众的剧场, 几把椅子, 一本剧本, 但灯光全开. 这两个空间一起, 是这栋小楼"通向外界"和"自我证明"的双重出口.

---

## 一

第六天.沈姨在三楼楼梯口等他.

阿衍上去.三楼一进门,他就愣住——这一层的格局完全不同.东侧是一面巨大的窗,窗外能看到对面的店面和马路.正中,从这一侧的墙伸出去**一根长长的玻璃走廊**,横跨整条街,通向对面的一栋小楼.

"这条桥——"

"——是这家店和外界的唯一连接."沈姨说,"客人来不一定是从一楼那扇玻璃门进来.他可能在马路对面的电脑上、可能在手机上、可能在另一个城市——都可以**从这条桥**走过来."

阿衍看见桥上有两条互相分开的小道:

```
     ┌─────────────── 桥 ────────────────┐
     │                                     │
     │  delivery 道  ←──  客人从这条道走   │
     │  events 道    ←──  检查员从这条道走 │
     │                                     │
     └─────────────────────────────────────┘
```

"两条道?"

"对.同一座桥,但**两条道分开走**.客人那边只看 delivery——最终翻译.检查员那边看 events——agent 内部活动.两条道**绝不交叉**."

> **HTTP 桥 = Core ↔ Clients 唯一通信. 双 SSE 流: delivery / events.**

---

## 二

"具体长什么样?"

沈姨在窗户的玻璃上,用一支白笔写:

```
POST /session                ← 建一次会话
POST /session/{sid}/turn     ← 一轮 (text / image / audio)
GET  /session/{sid}/delivery ← SSE 流 (用户可见)
GET  /session/{sid}/events   ← SSE 流 (调试可见)
```

"四个端点.不多.再加就过早."

"为什么用 SSE 不用 WebSocket?"

"WebSocket 要 reconnect、要心跳、要 ping/pong——一个 32 天的项目里,我们没时间养这些.SSE 是单向流,从服务器到客户端,够用,且**和 HTTP/1.1 同栈**——评审打开 curl 都能看见流."

阿衍记下:

> **SSE 优先 WebSocket. 单向够用 + 调试友好.**

---

## 三

"陷阱."沈姨说,"我踩过三个."

"**第一**, SSE 的 `yield` 后**必须 `await asyncio.sleep(0)`**.不然 uvicorn 会缓冲一段, 表现像'卡住几秒一爆发'.前端用户体验完全不对."

"**第二**, CORS.开发期 Android 模拟器、Web playground 都跨域,统统拒.开发期一律 `allow_origins=['*']`.生产再收紧."

"**第三**, 大图片 upload.FastAPI 默认 multipart 1MB 上限,客人拍的菜单一张 3MB,直接报 413.这件事**不是'图片太大',是'我们没配上限'**——明确把上限调到 5MB."

阿衍把这三条贴在他笔记本最显眼那一页.

---

## 四

"那同一份 Pydantic 类型——"

"——走三处.我昨晚说过."沈姨打断他,"工具墙那侧、模型说明书那侧、HTTP 这侧.HTTP 这边只是它的第三个用途,代码上**不写新的 schema**——直接用 TranslateArgs."

阿衍点头.他在心里把这条线和昨晚的暗语连起来——一份类型,从工具到模型再到 HTTP,**贯穿整栋楼**.

---

## 五

"上四楼."

阿衍跟上.三楼到四楼的楼梯比下面几段都要陡一点.推开门,他看见——

四楼空荡荡的.中间一块木地板,几把木椅子排成两排.地板正前方是一块小黑板,**没有人**.

"这是?"

"我的彩排厅."沈姨说.

阿衍走到中间.地板上贴着几条胶带,胶带上写着名字:

```
[椅子 1]  MockRuntime
[椅子 2]  MockTool::translate
[椅子 3]  MockTool::add_drawer
[椅子 4]  Test fixtures
```

"这里没有真模型?"

"没有.这里**没有任何'真'的东西**.椅子上坐的全是替身.我让替身按剧本演一遍,如果演得对——**真模型来了也会演得对**."

> **MockRuntime = 替身. 让 agent 框架在没有真模型时, 先把整出戏排好.**

---

## 六

"剧本长什么样?"

她拿出一张纸:

```python
def test_image_translation_writes_drawer(runtime, agent, db):
    runtime.queue_response(tool_calls=[
        ("extract_text_regions", {"image_id": "img_001"})
    ])
    runtime.queue_response(tool_calls=[
        ("translate", {"text": "メニュー", "target_lang": "zh"})
    ])
    runtime.queue_response(tool_calls=[
        ("add_drawer", {"input": "メニュー", "output": "菜单"})
    ])
    runtime.queue_response(text="原文意思是: 菜单.")

    result = agent.run_turn(InputItem(modality="image", data=b"..."))

    tool_names = [e.tool_name for e in agent.events.snapshot()
                              if e.type == "tool_call"]
    assert tool_names == ["extract_text_regions", "translate", "add_drawer"]

    rows = db.execute("SELECT * FROM drawers").fetchall()
    assert len(rows) == 1
    assert "菜单" in result.delivery
```

"看见没.我**事先把模型该说的每一句话排好**——`queue_response` 一句一句写——然后让 agent 去跑.最后我**断言**:它有没有按顺序调对工具?数据库里的抽屉对不对?最终输出有没有那个词?"

"——这就是 demo 录像的剧本啊."

"对.故事 A、故事 B、故事 C、故事 D——四个故事每一个都有这样一段彩排.评审看的 demo 视频,**就是从这间彩排厅出来的**.因为这里的戏排好了,真演的时候才不慌."

> **Agent Behavior 测试 = demo 剧本. 演员是 Mock, 但戏是真的.**

---

## 七

"但有时候要测真模型."

"对.我有第三层戏台子,在另一个房间——叫 e2e.每天只演一遍,慢得很,断言也宽:'有这个词出现就行',不要求逐字一致.因为真模型有随机性,**断言要宽**."

她在黑板上画了三层:

```
Tier 1  Unit         纯函数 / parser / schema     50-100 case  最快
Tier 2  Agent Behav  MockRuntime + queue          30-50  case  核心
Tier 3  E2E          真 GGUF + 真 SQLite          ~5     case  慢, nightly
```

"评审 reproducibility 加分点,就在 Tier 1 + Tier 2.**`pytest -q` 一行,60 秒内绿**——这是评审看到你这个项目'像不像样'的最直观一击."

阿衍记下:

> **三层测试. Tier 2 (MockRuntime) 是 demo 信心来源.**

---

## 八

外面已经深夜.

"沈姨,这层楼为什么这么空?"

"因为这一层的'空',是整栋楼最重要的属性."

她指着地板上贴着的胶带.

"——'空'就是'没有真模型也跑得通'.评审拿到我的 repo,**装个 pip,什么都不下载,什么都不接电源**,就能跑 100 个测试用例,看到 100 个绿点.他对这栋楼的信任,**从那 100 个绿点开始**."

阿衍站在彩排厅中央,环顾四周,忽然觉得这间空的房子比之前任何一层都更踏实.

"——明晚我们上屋顶."沈姨说,"你该见见我们这栋楼接的那三根天线了."
