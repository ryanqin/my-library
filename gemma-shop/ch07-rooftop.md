# 第七夜 · 屋顶上接的三根天线

> 这章要讲什么?
> 屋顶有三根天线伸向不同的方向. 沈姨告诉阿衍, 每一根都通向手机这层世界的不同入口—— ML Kit GenAI / MediaPipe / LiteRT-LM. 选对那一根, 才能把整栋楼装进客人的口袋.

---

## 一

第七天晚上.沈姨从二楼又上了一层.

"还有屋顶?"

"有.屋顶不在四楼之上,在四楼之外."

她推开一扇侧门,后面是一条铁皮楼梯,直通屋顶.阿衍跟上去,推开顶门——

四月的横滨夜风扑面.屋顶不大,水泥地,围着一圈半人高的女儿墙.墙边竖着**三根天线**,每一根都朝不同的方向倾斜——

```
天线 A: 朝东南     ← ML Kit GenAI Prompt API
天线 B: 朝东北     ← MediaPipe LLM Inference SDK
天线 C: 朝西       ← LiteRT-LM (raw, JNI)
```

远处隔着一段距离,还有**第四根细的**,几乎贴着屋顶平面.阿衍指了指它.

"那根?"

"AICore. Google 在 Pixel 上偷偷布了一根——但只 Pixel 能用,且还在 Developer Preview. 我们这次不爬这根杆."

阿衍懂了:**屋顶 = 手机端 runtime**.要选一根天线,把整栋楼装进客人的口袋.

---

## 二

"为什么有三根?为什么不只用一根?"

"因为一根不够用.三根各有各的强项."

她带阿衍走到第一根天线下面.

"**天线 A: ML Kit GenAI Prompt API**.最高层的接口,Google Play 服务里集成的——也就是说,**模型不用我们 ship**,Google 后台分发,版本更新 Google 帮你管."

"客人不用下载?"

"不用.他装我们的 APK,我们的 APK 调一个 Android 系统服务,系统服务本地跑 Gemma.一切都在他手机上发生,**离线、隐私、零下载**——我们要的三个属性,这一根天线全占了."

"还有一个甜点:Google 官方文档里就有一份**Function Calling Guide**,我们的场景几乎是它的范本."

阿衍记下:

> **MVP 走 ML Kit GenAI Prompt API. 零模型分发 + 官方 function-calling 范本 + Kotlin DSL 友好.**

---

## 三

"短板呢?"

"两个."沈姨说,"**一**, 模型版本是 Google 决定的,我们锁不住.评审窗口短,差异微乎其微,但这件事**写进 README** 让评审知道."

"**二**, function-calling 的 schema 在 Android 这一边是 **Kotlin Schema DSL**,不是 JSON.我们要写一层小的转换器,把 Pydantic 出来的 JSON Schema 翻成 Kotlin 的 Schema.Builder."

阿衍记下:

> **ML Kit 限制: 模型版本不可锁 + schema 是 Kotlin DSL 不是 JSON.**

---

## 四

她带阿衍走到第二根.

"**天线 B: MediaPipe LLM Inference SDK**.中间层.要我们自己 ship 一个 .litertlm 模型文件 (~2GB),但**控制粒度更细**——可以选量化档、可以选 chat template、跨平台 Android/iOS 都行."

"为什么不用它?"

"因为我们 ship 一个 2GB 的模型文件,APK 立马变成 2GB.客人下载体验不好.而且 iOS 我们这次根本不做."

"备选方案?"

"对.是备选.如果某天 ML Kit 那一根天线坏了——比如评审手机型号刚好不支持 ML Kit GenAI——我们临时切到 MediaPipe.接口在 L1 已经抽好了,**切只是改一个 runtime 注册项**."

阿衍记下:

> **MediaPipe 是 plan B. L1 抽象让切换零业务代码改动.**

---

## 五

她带阿衍走到第三根.

"**天线 C: LiteRT-LM**.底层.直接 JNI 调.控制力**满分**,代码量也**满分**——一个 Kotlin 工程师上手要花一周.我们**绝对不爬**."

"为什么提这一根?"

"因为 ML Kit 和 MediaPipe **底下都是它**.了解它的存在,你才知道'我们走 ML Kit 走的是它的高层封装,出问题时也能往下查一两层'.**不爬 ≠ 不知道**."

阿衍记下:

> **LiteRT-LM = 底层根. 高层封装出问题时, 知道往下两层查.**

---

## 六

"那 Android 这一侧的代码怎么起?"

"fork **Google AI Edge Gallery**——Google 官方的 Kotlin 开源 Android 应用,本身就集成了 ML Kit GenAI、模型生命周期管理、对话 UI、相机/麦克风权限.我们 fork,保留它的'本地推理'能力,**自己加一层 HTTP 客户端**——指向我们的 Core API."

"双 Backend?"

"对——这是关键设计."

她在屋顶水泥地上,用粉笔画:

```
┌──── Android Shell ────┐
│                        │
│   UI / Camera / Audio  │
│        ↓               │
│   AgentBackend (接口)   │
│   ┌────┴────┐          │
│ Local       Remote     │
│ ML Kit      HTTP       │
│ GenAI       → Core API │
└────────────────────────┘
```

"评审网络不行 → 切 Local.评审本地资源不够 → 切 Remote 跑笔记本.两边都不行 → 还有 CLI 兜底.**三层防线**."

阿衍记下:

> **双 Backend (Local ML Kit / Remote HTTP) + CLI 兜底 = 三层 reproducibility 防线.**

---

## 七

"风险是什么?"

"**Day 21 还没接通就回 CLI**."

阿衍抬头.

"什么意思?"

"32 天的预算里,Week 3 给 Android.如果到 Day 21,模型加载、function-call、相机捕获——任何一环还在调,**就立刻砍 Android,只交 CLI + Core**.评审 reproducibility 不会受影响——因为评审的眼睛只到一楼前台,Android 是甜点不是主菜."

阿衍把这一条**用红笔**写到笔记本最末:

> ⚠ **Day 21 检查点: Android 不通就砍, 不留恋. 主菜是 Core.**

---

## 八

风更大了.阿衍看着远处横滨港的灯,几艘船在海上慢慢移动.沈姨站在屋顶边缘,背对他.

"沈姨,我有一个问题."

"嗯."

"——你为什么要造这栋楼?"

她沉默了一下.

"因为我五十岁那年,觉得自己再也没办法记住每一个客人.以前我能.三十年下来,每一个常客的语病、每一个学日文的小孩在哪个语法上卡过——我都记得.可我五十岁之后,记不住了.我就想,**有没有一种东西,能替我记**."

"agent."

"对.它没情绪,它不会累,它**记得比我清楚,但又不替我说话**.它就是替我守着这个店——让我可以上楼,做我自己想做的文学翻译."

她转过身.

"——明晚你会见到她.守夜人.她就是这种'记得但不说话'的化身."

阿衍点点头.他想到——agent 不是替代沈姨的,**是替沈姨守夜的**.

> **agent ≠ 替你做事的奴仆. agent = 替你记着的伙伴.**
