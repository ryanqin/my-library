# 第五夜 · 楼梯井里的暗语

> 这章要讲什么?
> 这家店的楼梯井不只是楼梯, 它还是一根贯穿各层的"传话筒". 楼上楼下用同一种暗语沟通—— Gemma 4 的 function-calling 格式. 阿衍今晚要看的是这门暗语的真实样子.

---

## 一

第五天晚上.沈姨先没让阿衍上楼,而是带他到楼梯口——一道宽两步、高五层的木楼梯,从地下室一直伸到屋顶.

"看这个井."她指着楼梯中间贯穿的那段空芯,"我每天上下来回好几趟,有一天我意识到——这不光是楼梯.这是一根**传话筒**."

她拿出一支铜哨,对着楼梯井下方轻轻一吹.

"叮——"

阿衍听见从楼下传来同样一声"叮".然后从楼上.再然后从地下室.

"楼上楼下用同一种声音沟通.每一层的人都听得懂.如果哪一层的人**听不懂这种声音**,那这一层就和别人断联了."

阿衍点头.他知道她要说的是什么.

> **function-calling 格式 = 整栋楼的暗语. 每一层都得听得懂.**

---

## 二

沈姨拿出一张草纸,放在台阶上.

"客人说一句话,经过主管,主管把这话送给灶火.灶火回出来的话——**长这样**:"

她在草纸上写:

```
<start_of_turn>user
请把这段日文翻成中文: メニュー
<end_of_turn>
<start_of_turn>model
我看到这是日文菜单.
[tool_call]
{"name": "translate", "args": {"text": "メニュー", "target_lang": "zh"}}
[/tool_call]
<end_of_turn>
```

"——这就是暗语.两端是楼梯井顶上的木牌(`<start_of_turn>` / `<end_of_turn>`),中间是模型自己写的话.话里**夹了一个 tool_call 的 JSON**——这是模型在告诉主管:'你帮我去工具墙调 translate 这个工具,参数是这些.'"

"主管怎么把这个 tool_call 取出来?"

"自己写一个 parser.**这件事不能依赖任何库**.因为这套暗语是 Gemma 4 自己的方言——不是 OpenAI 标准的 message.tool_calls 字段."

阿衍在笔记本上记:

> ⚠ **tool_call 藏在 content 里. 自己写 parser. ~80 行.**

---

## 三

"陷阱."沈姨说,"你写这个 parser 会踩四个雷."

她伸出手.

"**第一**, 不能凭印象写两端的木牌长什么样.必须**从官方 model card 抄回来**.你 Day 1 第一件事是跑一次 hello-world function-call, 把模型真实输出**逐字保存**到文件里——这个文件以后是你 parser 的回归测试 fixture."

"**第二**, 一次回话里可能有**多个 tool_call**.比如:

```
[tool_call]{"name":"detect_image_content","args":{"img":"..."}}[/tool_call]
[tool_call]{"name":"extract_text_regions","args":{"img":"..."}}[/tool_call]
```

主管要**按顺序串行执行**,每一个的结果塞回去再喂给模型."

"**第三**, 模型在写完 tool_call 之后,**经常还会继续说话**——再吐几行自然语言.如果不打断它,token 就会一路烧到 max_tokens.主管要**边收边解析**,一旦看见完整的 tool_call 落地,就**主动 break**."

"**第四**, 偶尔模型会写半截 JSON——少个引号、少个右括号.parser 要 try/except,丢弃这条但写到 events 流里——**留一条线索**, 不要静默失败."

阿衍记下:

> **parser 四雷: 字面要从官方抄 / 多 tool_call 串行 / 早停 / 半截 JSON 不静默丢.**

---

## 四

"那我们告诉模型工具的'说明书',要怎么写?"

沈姨把草纸翻过来,在背面写:

```python
class TranslateArgs(BaseModel):
    text: str = Field(description="原文")
    target_lang: Literal["en","zh","ja"]
    formality: Literal["formal","neutral","casual"] = "neutral"
```

"在我们这边——用 Pydantic 写一个 BaseModel.每个字段的 description 是给模型看的.Pydantic 自动生成 JSON Schema,**这一份 schema 走三个地方**:"

她在草纸上画了一个三叉:

```
TranslateArgs (Pydantic)
   ├─→ 工具墙的 registry      (L2)
   ├─→ Gemma 的 tool 说明书   (送给模型看的)
   └─→ HTTP API 的请求体校验  (3F 那座桥)
```

"**一份类型走三处**.我们不重复定义.重复定义就一定会漂移."

阿衍记下:

> **Pydantic schema 复用: 工具 / 模型说明书 / HTTP 校验 三处共用.**

---

## 五

"还有一件事我今晚必须告诉你."沈姨说,"——**不要夸大模型 tool_call 的能力**."

"什么意思?"

"模型偶尔会调一个**不存在的工具名**——比如 'tranlsate' 拼错.或者会传一个**不符合 schema 的参数**——比如 target_lang 写成 'CHinese' 而不是 'zh'.这两件事都会发生."

"怎么处理?"

"在工具墙那一侧,**收到 tool_call 时再校验一次**:

```
1. 名字不在 registry → 写 events 流: "unknown tool: tranlsate"
                       塞回模型: "tool not found, available: [...]"
                       让它自己改

2. 参数不过 Pydantic → 写 events: "schema validation failed: ..."
                       塞回模型: "args invalid: ..."
                       让它自己重试
```

不要让 agent 因为'工具拼错一个字母'就当机.要让它**有自纠的反馈回路**."

阿衍记下:

> **tool_call 容错: 名字校验 + schema 校验 + 失败信息塞回让模型自纠.**

---

## 六

"沈姨,我有一个直觉问题."

"问."

"如果模型每次调工具的写法都不太一样——比如它今天用 [tool_call], 明天用 <function_call>——我的 parser 是不是要不停改?"

"会改.所以你的 parser 要**写在最薄的一层** (L1 runtime 里),**和上面所有逻辑解耦**.将来 Gemma 5、Gemma 6 出新格式,**只改这八十行,楼上一行都不用动**.这是为什么 L1 必须是接口."

阿衍懂了.整栋楼为什么要分六层——**因为格式会变,接口不变**.

> **格式会变, 接口不变. 这就是分层的全部意义.**

---

## 七

外面下起小雨.灯笼在雨里晃.

沈姨把草纸折好,塞进笔记本封皮里.

"明晚我们看三楼的桥, 顺便看四楼的彩排厅."

"彩排?"

"对.我有一个**没有真模型**的剧场——只有几把椅子和一本剧本.你会喜欢的."

她抬头看了一下楼梯井顶上.

"——你今晚先把这套暗语在心里默一遍.记住一句话:

**'格式抄回来的, 不是写出来的.'**"

阿衍点点头.他在笔记本封面上,用很小的字写下:

> **Day 1 第一件事 → 跑 hello-world function-call → 存样例.**
