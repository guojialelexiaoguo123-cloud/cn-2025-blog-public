
> 系列第三篇。上篇把链路跑通了，但记忆存在内存里，服务一重启全没。这篇把记忆系统做扎实。

---

## 目录

- [为什么内存不够用](#为什么内存不够用)
- [记忆的三层结构](#记忆的三层结构)
- [准备存储环境](#准备存储环境)
- [第一层：持久化对话历史](#第一层持久化对话历史)
- [第二层：滚动摘要](#第二层滚动摘要)
- [第三层：关键信息提取与去重](#第三层关键信息提取与去重)
- [把三层记忆注入 Prompt](#把三层记忆注入-prompt)
- [更新 handler.js](#更新-handlerjs)
- [验证效果](#验证效果)
- [目前的状态](#目前的状态)

---

上篇跑通了基本链路，但有个很明显的缺陷：记忆存在 `Map` 里，服务一重启全没了。

更大的问题是，就算不重启，聊久了之后历史对话越来越长，每次都把全部历史塞进 prompt，token 数会爆，成本也跟着上去，到最后模型会直接报错说 context 超限。

这篇把记忆系统重新做一遍，用三层结构解决这个问题。

---

## 为什么内存不够用

先把问题说清楚。

大模型的 context window 是有上限的，DeepSeek-chat 是 64k token，听起来很多，但实际一轮对话的 prompt 里包含：角色设定 + 历史对话 + 当前消息，聊个几十轮之后历史就占满了。

超出上限有两种结果，一是 API 直接报错，二是模型默默把最早的内容截掉，相当于它"忘了"很久之前的事。

解决思路是分层存储，不同时效的记忆放不同的地方：

- 最近几轮：直接放 prompt，模型能直接看到
- 近期发生的事：压缩成摘要，每次注入
- 用户说过的重要信息：单独提取存起来，用到时再拿

---

## 记忆的三层结构

```
L1 短期记忆   最近 N 轮对话原文，直接放 prompt
L2 摘要记忆   每 N 轮触发一次，LLM 压缩成几百字，注入 system prompt
L3 关键记忆   实时提取用户说的重要信息，以结构化条目存入数据库
```

三层配合之后，prompt 里的结构变成这样：

```
[角色设定]
[L3 关键记忆条目]
[L2 近期摘要]
[L1 最近 N 轮原文]
[当前消息]
```

token 数是可控的，记忆是持久的，重要的事不会忘。

---

## 准备存储环境

这篇用两个存储：

- **Redis**：存最近的对话历史（L1），读写快，适合高频操作
- **PostgreSQL**：存摘要（L2）和关键记忆条目（L3），适合持久化结构化数据

可以这样理解两者的分工：Redis 像人的瞬时反应，读写极快，但断电就没；PostgreSQL 像大脑的海马体，负责把重要的事固化成长期记忆，重启也在。两个配合，才有"快"又"稳"。

本地开发最简单的方式是用 Docker：

```bash
# 启动 Redis
docker run -d --name redis -p 6379:6379 redis

# 启动 PostgreSQL
docker run -d --name postgres \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=yourpassword \
  -e POSTGRES_DB=chatbot \
  -p 5432:5432 postgres
```

`POSTGRES_USER` 虽然默认就是 `postgres`，但显式写出来，后面连接时不容易搞混。

没装 Docker 的话，Redis 和 PostgreSQL 都有 Windows/Mac 的安装包，官网下就行。

装 Node.js 依赖：

```bash
npm install ioredis pg
```

在 `.env` 里加上数据库连接信息：

```
REDIS_URL=redis://localhost:6379
PG_HOST=localhost
PG_PORT=5432
PG_USER=postgres
PG_PASSWORD=yourpassword
PG_DATABASE=chatbot
```

然后建数据库表。新建一个 `init.sql`：

> **Windows 用户注意：** 如果 `init.sql` 里有中文注释（比如 `-- 近期摘要表`），务必把文件保存为 **UTF-8 无 BOM** 格式。Windows 记事本默认保存的 UTF-8 带 BOM，`psql` 执行时会把 BOM 字节当成 SQL 语法解析，直接报语法错误。用 VS Code 保存时在右下角点编码，选"UTF-8"而不是"UTF-8 with BOM"。

```sql
-- 近期摘要表（L2）
CREATE TABLE IF NOT EXISTS memory_summary (
  user_id     TEXT PRIMARY KEY,
  summary     TEXT NOT NULL,
  updated_at  TIMESTAMPTZ DEFAULT NOW()
);

-- 关键记忆条目表（L3）
CREATE TABLE IF NOT EXISTS memory_facts (
  id          SERIAL PRIMARY KEY,
  user_id     TEXT NOT NULL,
  fact        TEXT NOT NULL,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX IF NOT EXISTS idx_facts_user_id ON memory_facts(user_id);
```

在 PostgreSQL 里执行一下：

```bash
psql -h localhost -U postgres -d chatbot -f init.sql
```

---

## 第一层：持久化对话历史

把上篇的 `memory.js` 改成用 Redis 存。Redis 的 List 结构很适合存对话历史，从左边插入，只保留最近 N 条。

```js
// memory.js
import 'dotenv/config'
import Redis from 'ioredis'
import pg from 'pg'

const redis = new Redis(process.env.REDIS_URL)

const pool = new pg.Pool({
  host:     process.env.PG_HOST,
  port:     Number(process.env.PG_PORT),  // .env 读出来全是字符串，pg 库的 port 要求数字，不转换偶尔会报连接错误
  user:     process.env.PG_USER,
  password: process.env.PG_PASSWORD,
  database: process.env.PG_DATABASE,
})

const HISTORY_KEY   = (userId) => `history:${userId}`
const MAX_HISTORY   = 40   // 最多保留 40 条（20 轮对话）
const SUMMARY_EVERY = 40   // 每满 40 条触发一次摘要压缩

// ─── L1：对话历史 ─────────────────────────────────────────

export async function getHistory(userId) {
  const raw = await redis.lrange(HISTORY_KEY(userId), 0, -1)
  // lrange 返回从新到旧，reverse 变成从旧到新，符合模型期望的时间顺序
  return raw.map(r => JSON.parse(r)).reverse()
}

export async function saveHistory(userId, userMessage, aiReply) {
  const key = HISTORY_KEY(userId)

  // 分两次 lpush，顺序更清晰
  await redis.lpush(key, JSON.stringify({ role: 'assistant', content: aiReply }))
  await redis.lpush(key, JSON.stringify({ role: 'user',      content: userMessage }))

  // 只保留最近 MAX_HISTORY 条
  await redis.ltrim(key, 0, MAX_HISTORY - 1)

  // 检查是否需要触发摘要
  const len = await redis.llen(key)
  if (len >= MAX_HISTORY) {
    triggerSummary(userId).catch(err => {
      console.error(`[${userId}] 摘要生成失败:`, err.message)
    })
  }
}
```

**关于 `MAX_HISTORY` 该设多少：**

上面设的 40 条（20 轮）是个比较舒适的值。DeepSeek-chat 的 context 上限是 64k token，一轮普通对话大概 100-200 token，40 条历史也就 4000-8000 token，加上角色设定和记忆注入，总量控制在 15000 token 以内完全没问题，成本极低。

上篇用的 20 条是过于保守的起点。实际上可以调到 40-60 条，对话连贯性会明显更好，用户不会频繁感觉"它忘了刚才说的事"。粗略原则：角色设定 prompt 越长就把 `MAX_HISTORY` 调小，角色设定简短就大胆往上调。

---

## 第二层：滚动摘要

对话历史攒到上限之后，用 LLM 把它压缩成一段摘要，然后清空历史，重新开始积累。摘要会注入到每次对话的 system prompt 里，让模型知道"之前大概发生过什么"。

```js
// memory.js（续）
import { summarizeHistory } from './llm.js'

async function triggerSummary(userId) {
  const history = await getHistory(userId)
  if (history.length === 0) return

  // 把旧摘要也传进去，新摘要会在旧摘要基础上累积，保持连续性
  const oldSummary = await getSummary(userId)
  const newSummary = await summarizeHistory(history, oldSummary)

  // upsert：有就更新，没有就插入
  await pool.query(
    `INSERT INTO memory_summary (user_id, summary, updated_at)
     VALUES ($1, $2, NOW())
     ON CONFLICT (user_id) DO UPDATE
     SET summary = $2, updated_at = NOW()`,
    [userId, newSummary]
  )

  // 清空 Redis 历史，重新开始积累
  await redis.del(HISTORY_KEY(userId))
  console.log(`[${userId}] 摘要已更新`)
}

export async function getSummary(userId) {
  const res = await pool.query(
    'SELECT summary FROM memory_summary WHERE user_id = $1',
    [userId]
  )
  return res.rows[0]?.summary ?? ''
}
```

在 `llm.js` 里加上摘要生成函数：

```js
// llm.js（新增）
export async function summarizeHistory(history, oldSummary) {
  const historyText = history
    .map(m => `${m.role === 'user' ? '用户' : 'AI'}：${m.content}`)
    .join('\n')

  const prompt = oldSummary
    ? `以下是之前的摘要：\n${oldSummary}\n\n以下是新的对话记录：\n${historyText}`
    : `以下是对话记录：\n${historyText}`

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
          content: `你是一个对话摘要助手。
请从对话记录中提炼关键信息，输出一段简洁的摘要，控制在200字以内。
重点保留：用户的情绪状态、提到的重要事件、个人信息、偏好、以及值得记住的细节。
不需要总结AI说了什么，只关注用户相关的信息。`
        },
        { role: 'user', content: prompt }
      ],
      max_tokens: 400,
      temperature: 0.3
    })
  })

  const data = await res.json()
  return data.choices[0].message.content
}
```

摘要用 `temperature: 0.3`，比对话生成的 0.9 低很多。

**为什么低温度更稳定？** 直觉上可以这么理解：temperature 控制的是模型在每个 token 上"要不要选次优答案"的概率。temperature → 0 时，模型每步都选概率最高的 token，输出几乎是确定的；温度高了，模型开始随机采样，有时候会选到意想不到的词，产生创意，也可能产生幻觉——把用户没说的事"总结"进摘要里。所以：对话生成要创意用 0.9，摘要/结构化提取要准确用 0.1-0.3。

---

## 第三层：关键信息提取与去重

摘要是批量处理的，每隔一段时间才触发一次。但有些信息需要即时捕捉——用户说"我今天失恋了"或者"我叫 Gary"，应该马上存下来，不管什么时候都能用到。

这层一个常见问题是**重复提取**：用户多次提到同一件事，同一条 fact 会被反复存入数据库。`getFacts` 拉出来之后 prompt 里出现一堆重复条目，既浪费 token，也让模型困惑。

解决方式是在提取时把已有的 facts 也传给模型，让它自己判断是否重复：

```js
// memory.js（续）
import { extractFacts } from './llm.js'

export async function saveFactsAsync(userId, userMessage) {
  // 先拿到已有的 facts，传给提取函数做去重判断
  const existingFacts = await getFacts(userId)

  extractFacts(userMessage, existingFacts).then(async newFacts => {
    if (newFacts.length === 0) return

    for (const fact of newFacts) {
      await pool.query(
        'INSERT INTO memory_facts (user_id, fact) VALUES ($1, $2)',
        [userId, fact]
      )
    }
    console.log(`[${userId}] 新增 ${newFacts.length} 条关键信息`)
  }).catch(err => {
    console.error(`[${userId}] 关键信息提取失败:`, err.message)
  })
}

export async function getFacts(userId) {
  const res = await pool.query(
    `SELECT fact FROM memory_facts
     WHERE user_id = $1
     ORDER BY created_at DESC
     LIMIT 30`,
    [userId]
  )
  return res.rows.map(r => r.fact)
}
```

在 `llm.js` 里加上提取函数，去重逻辑直接写进 prompt：

```js
// llm.js（新增）
export async function extractFacts(userMessage, existingFacts = []) {
  const existingSection = existingFacts.length > 0
    ? `\n\n已知事实（若提取内容与此重复则忽略）：\n${existingFacts.map(f => `- ${f}`).join('\n')}`
    : ''

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
          content: `你是一个信息提取助手。
从用户的消息中提取值得长期记忆的关键事实，以 JSON 数组格式输出。
只提取明确的个人信息、重要事件、情绪状态、偏好。
如果没有值得提取的信息，或内容与已知事实重复，返回空数组 []。
只输出 JSON，不要有其他文字。${existingSection}

例子：
输入："我今天和男朋友分手了，心情很差"
输出：["用户今天分手了，情绪低落"]

输入："天气真好"
输出：[]`
        },
        { role: 'user', content: userMessage }
      ],
      max_tokens: 200,
      temperature: 0.1
    })
  })

  const data = await res.json()
  const text = data.choices[0].message.content.trim()

  try {
    const facts = JSON.parse(text)
    return Array.isArray(facts) ? facts : []
  } catch {
    // 模型偶尔不按格式输出，解析失败返回空数组，不影响主流程
    return []
  }
}
```

提取任务用 `temperature: 0.1`，是所有任务里最低的——需要严格输出 JSON 格式，不需要任何随机性，越接近 0 越稳定。

---

## 把三层记忆注入 Prompt

```js
// llm.js（更新 callLLM）
export async function callLLM(history, userMessage, summary, facts) {
  const memorySection = []

  if (facts.length > 0) {
    memorySection.push(`【关于用户的重要信息】\n${facts.map(f => `- ${f}`).join('\n')}`)
  }

  if (summary) {
    memorySection.push(`【近期对话摘要】\n${summary}`)
  }

  const systemContent = `你叫林深，25岁，上海，做平面设计。
性格有点内敛，但在意的人面前话会多一点。
说话习惯用短句，偶尔发省略号。
回复控制在50字以内，口语化，不要说废话。
如果记忆里有相关信息，自然地用上，不要刻意提起"我记得你说过"。

${memorySection.join('\n\n')}`

  const messages = [
    { role: 'system', content: systemContent },
    ...history,
    { role: 'user',   content: userMessage }
  ]

  const res = await fetch('https://api.deepseek.com/chat/completions', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.DEEPSEEK_API_KEY}`,
      'Content-Type': 'application/json'
    },
    body: JSON.stringify({
      model: 'deepseek-chat',
      messages,
      max_tokens: 200,
      temperature: 0.9
    })
  })

  if (!res.ok) {
    const err = await res.text()
    throw new Error(`DeepSeek API 错误: ${res.status} ${err}`)
  }

  const data = await res.json()
  return data.choices[0].message.content
}
```

"不要刻意提起'我记得你说过'"这条规则很重要。如果让 AI 每次都说"根据你之前提到的..."，用户会立刻出戏，感觉在跟一个在背台词的机器人聊天。记忆应该像人一样自然地融入对话。

---

## 更新 handler.js

```js
// handler.js（更新）
import { callLLM } from './llm.js'
import { sendToWeChat } from './wechat.js'
import {
  getHistory,
  saveHistory,
  getSummary,
  getFacts,
  saveFactsAsync
} from './memory.js'

export async function handleMessage(userId, userMessage) {
  // 并行拉取三层记忆，总耗时取决于最慢的那个，而不是三个加起来
  const [history, summary, facts] = await Promise.all([
    getHistory(userId),
    getSummary(userId),
    getFacts(userId)
  ])

  const reply = await callLLM(history, userMessage, summary, facts)

  // 存历史（内部会判断是否触发摘要压缩）
  await saveHistory(userId, userMessage, reply)

  // 异步提取关键信息，不阻塞回复
  saveFactsAsync(userId, userMessage)

  await sendToWeChat(userId, reply)
}
```

---

## 验证效果

用一个具体的案例来测，比抽象的"发几条消息"更直观。

先发几条带有具体信息的消息：

```
你：林深，我刚用 A7M4 拍了一组延时摄影，蓝色的色调怎么都处理不好
AI：延时的话蓝调确实挑后期……你用什么软件剪的？
你：Lightroom，主要是暗部的蓝色偏灰，不够纯
AI：可以试试在 HSL 里单独拉蓝色的饱和度，再压一点高光……
```

然后**重启服务**（`Ctrl+C` 再 `node index.js`），问一个跨会话的问题：

```
你：你觉得我刚才拍的那组照片，用什么滤镜风格比较好？
```

如果记忆系统工作正常，AI 应该能联系到上次提到的 A7M4 和蓝色调，给出有针对性的建议，而不是问"你在拍什么"。比如：

```
AI：你那组蓝调延时……要是想再冷一点，电影感的话可以试试青橙 LUT，
    把橙色推暖、蓝色往青偏……A7M4 的暗部宽容度够，这样拉不会糊。
```

这就是记忆系统真正工作的感觉——不只是"记住了名字"，而是记住了上下文，能基于过去的对话做出有深度的回应。

**用数据库直接验证：**

```bash
psql -h localhost -U postgres -d chatbot

-- 看关键信息有没有存进去
SELECT * FROM memory_facts ORDER BY created_at DESC LIMIT 10;

-- 看摘要
SELECT user_id, LEFT(summary, 100) FROM memory_summary;
```

**测去重效果：** 连续发两条类似的消息：

```
你：我喜欢拍延时摄影
你：对了我特别喜欢摄影，尤其是延时
```

查 `memory_facts`，应该只有一条关于摄影偏好的记录，而不是两条。

---

## 目前的状态

做完这篇之后，记忆系统已经相对完整：

- 服务重启记忆不丢失（Redis + PostgreSQL 持久化）
- 对话历史有弹性上限，不会撑爆 context
- 重要信息实时提取，有去重，长期保留
- 三层记忆注入 prompt，模型能"想起"过去的事
- 摘要在旧摘要基础上累积，记忆有连续性

还没有的：

- 角色人设还很粗糙，只有几行描述
- 没有主动发消息的能力（定时任务）

---

## 下一篇

下篇做角色设计。

提示词工程是这类产品最核心的部分，没有之一。同样的技术架构，提示词写得好不好，直接决定用户有没有"这个角色是真实的人"的感觉。

会从零写一套完整的人设 prompt，包括角色背景、说话风格、情绪状态机、以及一些让角色真正"活"起来的细节。

---

*本系列：*
- *01 原理篇：AI 是怎么住进微信的*
- *02 接入篇：Webhook 服务从零搭建*
- *03 记忆篇：让 AI 真正记住你 ← 你在这*
- *04 角色篇：提示词工程与人设设计（即将发布）*