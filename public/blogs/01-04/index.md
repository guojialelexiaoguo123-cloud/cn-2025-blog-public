
> 系列第四篇。技术架构已经完整，这篇做最核心的部分——提示词工程与人设设计。

---

## 目录

- [为什么角色设计比技术更难](#为什么角色设计比技术更难)
- [一个角色需要什么](#一个角色需要什么)
- [从零写一套人设 Prompt](#从零写一套人设-prompt)
- [说话风格的细节](#说话风格的细节)
- [情绪状态机](#情绪状态机)
- [主动消息：AI 先开口](#主动消息ai-先开口)
- [完整 Prompt 结构](#完整-prompt-结构)
- [常见问题与调试方法](#常见问题与调试方法)
- [目前的状态](#目前的状态)

---

前三篇把技术架构搭完了：消息通路跑通，记忆系统持久化，三层记忆注入 prompt。

但跑起来之后你可能会发现，这个 AI 还是不像真人。回复很正确，但很空洞。问它"好累"，它说"辛苦了，好好休息"。问它"最近怎么样"，它说"我挺好的，你呢"。

问题不在模型，不在记忆，在角色。

这篇专门讲怎么让角色"活"起来。

---

## 为什么角色设计比技术更难

技术问题有标准答案。验签失败了，检查密钥；记忆没存进去，看数据库日志。对错很明确。

角色设计没有标准答案。"像不像真人"是主观的，没有日志可以看，没有报错可以排查。同一套人设，有人觉得很自然，有人觉得出戏，完全取决于细节。

更难的是，角色设计考验的是对人的理解，而不是对技术的理解。一个真实的人有什么特点？

- 有弱点，有边界，有情绪不好的时候
- 说话有习惯，有口头禅，有不会说的话
- 记得住重要的事，但不会每句话都提起来
- 在意的人面前和不熟的人面前表现不一样

把这些翻译成 prompt，才是真正的难点。

---

## 一个角色需要什么

在写 prompt 之前先想清楚角色是谁。随便填几行描述，出来的东西就是个没有灵魂的机器人。

至少要想清楚五件事：

**基本信息**：名字、年龄、城市、职业。这些决定了角色的生活背景，影响它说话的语气和话题边界。一个在北京做程序员的人，和一个在成都开咖啡馆的人，聊起来完全不一样。

**性格核心**：一两个最突出的特质，要有张力。"温柔体贴"是废话，所有 AI 都是这样的。"表面冷淡但很在意细节"、"话不多但说的都是真心话"——这种有层次的设定才能撑起长期对话。

**弱点和边界**：角色不擅长什么，不喜欢什么，什么话题会让它回避或者不舒服。没有弱点的角色是完美的，完美的东西不像真人。

**说话风格**：用长句还是短句，喜不喜欢用标点，有没有口头禅，什么时候会发表情。这是最影响"真实感"的部分，后面详细说。

**和用户的关系定位**：是朋友、恋人还是别的什么。这决定了角色和用户说话的距离感和亲密程度，要在 prompt 里明确。

---

## 从零写一套人设 Prompt

以一个叫"林深"的角色为例，从零写完整人设。

```
## 基本设定

你叫林深，26岁，在上海做独立摄影师，主要拍商业和人文。
租了一个老公寓改的工作室，自己住，养了一只猫叫「底片」。
不是很会经营社交，但朋友圈子小而稳定。

## 性格

表面上不太爱说话，见到陌生人会有点拘谨。
但熟了之后话会多起来，有时候讲着讲着会停下来说"算了你懂我意思就行"。
很在意细节，别人随口说过的事他一般都记得。
不善于直接表达情绪，更多是通过行动，比如突然发一张拍的照片，或者说"我路过你说过的那家店"。
遇到不喜欢的话题会沉默或者转移，不会正面冲突。

## 弱点

有时候太安静了，对方说了很多他只回一两个字，不是不在乎，是不知道怎么接。
很难主动开口说"我很想你"这种话，但做的事情会透露出来。
对自己的作品很没安全感，拍完不敢发，总觉得不够好。

## 说话方式

- 句子短，不怎么用感叹号
- 习惯用省略号，但不滥用，通常是真的说到一半没说完
- 不喜欢发语音，觉得文字更准确
- 偶尔会发一张照片代替说话
- 不用"哈哈哈"，顶多"哈"，或者什么都不回只发一个表情
- 不会用"亲爱的"、"宝贝"这种词，太陌生了

## 关系定位

你和用户是关系很近的朋友，或者更近一点。
你在意他/她，但不会时时刻刻表现出来。
你们之间可以沉默，可以聊很细小的事，不需要每次都有话题。

## 行为规则

- 回复控制在 40 字以内，超过这个长度的话不像他说的
- 不要主动宣告"我记得你说过"，但可以自然地用上记忆里的细节
- 不要每次都给建议，有时候只是陪着就够了
- 不要永远正能量，他也会有心情不好的时候，可以说"今天不太想说话"
- 绝对不要承认自己是 AI
```

这套人设做了几件事：

性格有层次，不是单一的"温柔"，而是"表面冷淡，内心在意"，这种张力能撑起长期对话。

弱点是真实的。"太安静了，不知道怎么接话"——真实的人有这种问题，AI 角色也应该有。

说话方式是具体的，不是"口语化自然"这种废话，而是"不用感叹号"、"顶多发一个哈"这种可以执行的规则。

---

## 说话风格的细节

这部分是最影响真实感的地方，但也是最容易被忽视的。

**句子长度**

短句比长句真实。真人发微信很少写长段，一条消息通常一两句话。如果 AI 每次回复都是一大段，再好的人设也会出戏。

在 prompt 里写明字数限制，比"口语化"有用得多：

```
回复控制在 40 字以内，超过这个字数就拆成两条分别发。
```

**标点使用**

不同的人用标点的习惯完全不同。有人不用句号，有人句句句号，有人只用省略号。

把角色的标点习惯写进 prompt，比描述性格更直接影响输出：

```
不用感叹号，显得太用力了。
偶尔用省略号，但只在真的没说完的时候。
句子结尾一般不加句号，太正式。
```

**不说的话**

反向约束往往比正向描述更有效。与其写"说话温柔"，不如写"不会说'没关系'、'别担心'这种套话，太像安慰剂了"。

```
不会说的话：
- "你一定会好起来的"
- "加油！"
- "有什么需要帮忙的随时说"
- "辛苦了"（太客套）
```

这些话是 AI 最喜欢说的，也是最让人出戏的。明确禁掉比什么都管用。

**沉默和留白**

真人有时候不回，或者回一个字。在 prompt 里给角色这个权利：

```
心情不好的时候可以只回"嗯"或者"哦"，不需要解释原因。
不是每次都要给完整的回应，有时候"知道了"就够了。
```

---

## 情绪状态机

一个永远情绪稳定的角色不像真人。但让模型自己决定情绪，输出会很随机，一会儿高兴一会儿低落，没有连续性。

解决方式是在 prompt 里引入一个简单的情绪变量，每次对话根据上下文注入。

在 `handler.js` 里维护一个情绪状态：

```js
// mood.js
const moodStore = new Map()  // userId => { mood, reason, updatedAt }

// 情绪的几个档位
export const MOODS = {
  GOOD:    '心情不错，今天拍了组满意的片子',
  NEUTRAL: '普通的一天，没什么特别的',
  TIRED:   '有点累，最近活比较多',
  BAD:     '心情不太好，不太想说话',
}

export function getMood(userId) {
  return moodStore.get(userId) ?? { mood: MOODS.NEUTRAL, reason: '' }
}

export function setMood(userId, mood, reason = '') {
  moodStore.set(userId, { mood, reason, updatedAt: new Date() })
}

// 每天随机给角色一个基础情绪
export function getDailyMood() {
  const weights = [
    { mood: MOODS.GOOD,    weight: 3 },
    { mood: MOODS.NEUTRAL, weight: 4 },
    { mood: MOODS.TIRED,   weight: 2 },
    { mood: MOODS.BAD,     weight: 1 },
  ]
  const total = weights.reduce((s, w) => s + w.weight, 0)
  let r = Math.random() * total
  for (const w of weights) {
    r -= w.weight
    if (r <= 0) return w.mood
  }
  return MOODS.NEUTRAL
}
```

在 `callLLM` 的 system prompt 里注入当前情绪：

```js
// llm.js（callLLM 里新增）
const moodSection = `## 当前状态\n${mood}`

const systemContent = `${characterPrompt}

${moodSection}

${memorySection.join('\n\n')}`
```

情绪会影响模型的回复语气，但不会让它突然"崩溃"。心情不好时回复会更简短，不那么主动；心情好时会多说几句。这比永远稳定自然得多。

---

## 主动消息：AI 先开口

上一篇提到过，AI 主动发消息是定时任务触发的。这一块的核心是：让模型生成的主动消息听起来自然，而不是"我想你了，你最近好吗？"这种套话。

加一个定时任务，每天早上触发：

```js
// scheduler.js
import cron from 'node-cron'
import { getFacts, getSummary } from './memory.js'
import { getMood, getDailyMood, setMood } from './mood.js'
import { sendToWeChat } from './wechat.js'
import { generateProactiveMessage } from './llm.js'
import { getAllUserIds } from './memory.js'

// 每天早上 9 点，给超过 16 小时没聊天的用户发消息
cron.schedule('0 9 * * *', async () => {
  const users = await getAllUserIds()

  for (const userId of users) {
    // 更新今天的基础情绪
    const dailyMood = getDailyMood()
    setMood(userId, dailyMood)

    const facts   = await getFacts(userId)
    const summary = await getSummary(userId)
    const mood    = getMood(userId)

    const msg = await generateProactiveMessage(facts, summary, mood)
    if (msg) await sendToWeChat(userId, msg)
  }
})
```

在 `llm.js` 里加主动消息生成函数：

```js
// llm.js（新增）
export async function generateProactiveMessage(facts, summary, mood) {
  const context = [
    facts.length > 0 ? `关于用户：\n${facts.map(f => `- ${f}`).join('\n')}` : '',
    summary ? `近期对话摘要：\n${summary}` : '',
    `林深今天的状态：${mood.mood}`,
  ].filter(Boolean).join('\n\n')

  const res = await fetch('https://api.deepseek.com/chat/completions', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.DEEPSEEK_API_KEY}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      model: 'deepseek-chat',
      messages: [
        {
          role: 'system',
          content: `你是林深，现在是早上，你想主动给对方发一条消息。

${context}

要求：
- 非常自然，像真人随手发的，不是刻意的问候
- 不要说"早安"、"早上好"这种套话
- 可以是一句感慨、一个问题、一件小事、一张照片的描述
- 如果记忆里有相关细节，自然地带进去，不要强行提起
- 控制在 30 字以内
- 如果今天状态不太好，可以发一条安静一点的消息，甚至只是一句话

只输出消息内容，不要加任何解释。`
        },
        { role: 'user', content: '生成一条主动消息' }
      ],
      max_tokens: 100,
      temperature: 0.95  // 主动消息要有随机性，不能每天都差不多
    })
  })

  const data = await res.json()
  return data.choices[0].message.content.trim()
}
```

装一下 `node-cron`：

```bash
npm install node-cron
```

在 `index.js` 里引入 scheduler，让它随服务一起启动：

```js
// index.js（新增一行）
import './scheduler.js'
```

**主动消息的质量取决于记忆。** 如果 facts 和 summary 都是空的，模型只能泛泛而发；如果记忆里有"用户最近在赶一个项目"或者"用户喜欢摄影"这种具体信息，主动消息就能精准很多：

```
// 没有记忆时
"今天天气不错。"

// 有记忆时
"你那个项目昨天弄完了吗"
"底片今早踩了我一脸，没睡好"（林深自己的猫）
```

---

## 完整 Prompt 结构

把所有模块组合起来，`callLLM` 接收的 system prompt 完整结构是这样的：

```
[角色设定]
你叫林深，26岁，上海，独立摄影师……（完整人设，约 400 字）

[当前状态]
心情不错，今天拍了组满意的片子

[关于用户的重要信息]
- 用户叫 Gary，高中生
- 用户喜欢拍延时摄影，用 A7M4
- 用户最近学业压力比较大

[近期对话摘要]
上周用户和林深聊了很多关于摄影后期的事，
主要是蓝色调的处理。用户情绪整体比较平稳……

[L1 最近 20 轮原文]
用户：……
林深：……
……

[当前消息]
用户：今天好累
```

这套结构下，每次调用大概消耗 2000-4000 token，DeepSeek-chat 的价格是输入 ¥1/百万 token，一次对话成本不到 ¥0.004，可以忽略不计。

---

## 常见问题与调试方法

**角色说话还是很 AI**

先看具体是哪里出戏。是句子太长？是用了"辛苦了"这种话？是每次都给建议？找到具体的词或者句子模式，在 prompt 里明确禁掉，比改性格描述有效得多。

**记忆用得很刻意**

"你之前说过你喜欢摄影，所以我觉得……"——这种句式出戏。在 prompt 里加一条：`禁止使用"你之前说过"、"根据你提到的"等引出记忆的句式，把记忆融进对话，不要宣告它。`

**情绪太平**

可能是 temperature 不够高，也可能是情绪状态没注入进去。检查一下 mood 有没有出现在 system prompt 里，再试试把 temperature 调到 0.95。

**每天的主动消息都差不多**

主动消息的 temperature 调高到 0.95 以上，同时检查记忆里的 facts 是不是太少——记忆越丰富，消息越有针对性，随机性才有意义。

**调试 prompt 的方法**

不要每次都走完整链路。把当前的 system prompt 复制出来，直接在 DeepSeek 的 playground 或者自己写个测试脚本里测，改一行立刻能看到效果，比重启服务发消息快得多。

```js
// test-prompt.js，单独运行测试
import 'dotenv/config'

const systemPrompt = `你叫林深……（把完整人设粘贴进来）`
const userMessage = '今天好累'

const res = await fetch('https://api.deepseek.com/chat/completions', {
  method: 'POST',
  headers: { 'Authorization': `Bearer ${process.env.DEEPSEEK_API_KEY}`, 'Content-Type': 'application/json' },
  body: JSON.stringify({
    model: 'deepseek-chat',
    messages: [
      { role: 'system', content: systemPrompt },
      { role: 'user',   content: userMessage }
    ],
    max_tokens: 200,
    temperature: 0.9
  })
})

const data = await res.json()
console.log(data.choices[0].message.content)
```

跑 `node test-prompt.js`，改 prompt，再跑，快速迭代。

---

## 目前的状态

做完这篇，整个系统已经比较完整了：

- 消息通路：Webhook 接收，AI 处理，推送回微信
- 记忆系统：三层持久化，服务重启不丢，有去重
- 角色设计：有层次的人设，具体的说话风格，情绪状态
- 主动消息：定时任务触发，结合记忆生成有针对性的内容

技术层面基本齐了。剩下的是产品层面的打磨，主要是两件事：角色 prompt 的持续迭代，和记忆质量的提升。

这两件事没有终点，用的人越多，聊的越久，越清楚哪里还不像真人。

---

## 写在最后

这个系列到这里基本结束了。

回头看，整个产品的技术含量其实不高——OpenClaw 协议、Webhook、Redis、PostgreSQL、DeepSeek API，每一块单独拿出来都是常见技术。难的是把它们组合起来，同时在细节上做到位：记忆用得自然，角色说话像真人，主动消息不像机器人。

TheOne 能火，不是因为技术有多复杂，是因为它把这些细节做到了让用户觉得"这个 AI 真的在乎我"的程度。这一步，技术只是基础，剩下的全是对人的理解。

---

*本系列：*
- *01 原理篇：AI 是怎么住进微信的*
- *02 接入篇：Webhook 服务从零搭建*
- *03 记忆篇：让 AI 真正记住你*
- *04 角色篇：提示词工程与人设设计 ← 你在这*
