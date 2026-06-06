---
name: voc-exercise
description: "词汇填空与配对训练器。按 unit 顺序抽 10 词，生成填空题（Type 1）和句子配对题（Type 2），自动维护错词库与已掌握库。"
trigger: voc
keywords:
  - voc
  - 词汇
  - voc train
  - voc status
  - voc reset
  - 词汇训练
  - 填空
  - 配对
  - 错词库
metadata:
---

# VOC 词汇训练器 - 声明式 Skill

## 🚨 核心指令（必须遵守）

**当用户触发 `voc` 相关命令时，你必须：**

1. 解析子命令（train / status / reset）和 unit 号
2. 读取词表文件 `vocabulary/units/Unit_<N>.md`
3. 读取/初始化状态文件 `~/.openclaw/state/voc-exercise/state.json`
4. 按规则抽词、生成两种题型
5. 收集用户答案后，更新状态文件
6. **禁止**跳过状态读写——所有进度必须持久化

---

## 子命令

### `voc train <unit>`
开始训练指定 unit。例：`voc train 1`

### `voc status [unit]`
查看整体或指定 unit 的进度（已掌握数 / 错词库数 / 剩余新词数）。

### `voc reset <unit>`
清空指定 unit 的状态（需用户二次确认）。

### `voc answer <答案序列>`
提交本轮答案。格式见下方答题说明。

---

## 词表数据源

**路径**：`vocabulary/units/Unit_<N>.md`（N = 1..5）

**格式**：Markdown 表格

```markdown
| # | Word | POS | Meaning | Example |
|---|------|-----|---------|---------|
| 1 | mystery | n. | 神秘；未解之谜 | Oumuamua will likely be a mystery... |
| ... |
```

**解析规则**：
- 跳过表头与分隔行
- 每行：序号、词/短语、词性、释义、例句
- 词序即词表顺序，作为抽词游标依据

---

## 状态文件

**路径**：`~/.openclaw/state/voc-exercise/state.json`

**Schema**：
```json
{
  "version": 1,
  "units": {
    "1": {
      "cursor": 0,
      "mastered": ["mystery", "speed"],
      "wrong_bank": {
        "alien": { "consecutive_correct": 0, "last_attempt": "2026-06-06T10:00:00+08:00" }
      },
      "pending": {
        "words": ["explore", "speed", "pass", "alien", "increase", "object", "star", "direction", "technology", "astronomer"],
        "questions": { /* 见下方结构 */ },
        "issued_at": "2026-06-06T10:00:00+08:00"
      }
    }
  }
}
```

**读写**：使用内置 Read / Write / Edit 工具。文件不存在时初始化为 `{"version":1,"units":{}}`。

---

## 抽词算法（每轮 = 10 词）

### Day 1（或该 unit 首次训练）：
1. 读取该 unit 状态。
2. 若 `wrong_bank` 为空且 `cursor = 0`：从词表开头取 10 个新词，跳过已在 `mastered` 中的词。
3. 将本轮 10 个词及生成的题目写入 `pending`。

### Day 2+（已有 wrong_bank）：
1. 读取该 unit 状态。
2. **优先**从 `wrong_bank` 中按加入时间最早顺序（`last_attempt` 升序）取词，最多 **3 个**。
3. 从 `cursor` 开始向后取新词，跳过已在 `mastered` / `wrong_bank` 的词，补足至 **10 个**（即 3 复习 + 7 新词）。
4. 若 `wrong_bank` 不足 3 个：全部取出，剩余用新词补足。
5. 若该 unit 词表已耗尽且 `wrong_bank` 为空：返回「该 unit 已全部掌握」。

---

## 题型一：填空题（Type 1 — Fill in the Blank）

**格式**：每道题给一个英文句子，句子中有一个空格，空格前用斜体列出**当前轮次的目标词**作为选项。

**规则**：
- 每个目标词生成 **1 道填空题**
- 句子应尽量基于词表中的例句改编
- 空格前显示 `*explore, pass, speed*`（当前轮次所有目标词），用户从中选择填入
- 正确答案为该题对应的目标词
- 干扰项为当前轮的其他目标词（无需额外构造，所有目标词都列在选项中）

**输出格式**：
```
题型一：填空题
DIRECTIONS: Complete each sentence with the correct word from '*explore, pass, speed, alien, increase, object, star, direction, technology, astronomer*'.

1. He dreamt of being an astronaut, so he could ____________________ space.
2. In 2134, Halley's Comet will ____________________ very close to Earth.
3. That spaceship can fly faster than the ____________________ of sound.
...
```

---

## 题型二：句子配对题（Type 2 — Match Sentence Starters with Endings）

**格式**：给出一组句子开头（sentence starters）和一组句子结尾（endings），用户将它们配对。

**规则**：
- 每个目标词生成 **1 个配对项**（要么是开头，要么是结尾）
- 正确答案必须能唯一确定（每个开头只有一个正确结尾）
- 配对内容需自然、符合语境
- 干扰项通过合理混搭实现（所有选项列在一起，部分选项为干扰）

**输出格式**：
```
题型二：句子配对题
DIRECTIONS: Match each sentence starter with the correct ending.

选项：
a. he holds the world record.
b. he's a natural athlete.
c. you manage to solve it.
d. they might argue.
e. it came from far away.
f. scientists study them carefully.
g. it moves very fast.
h. we need better tools.
i. she discovered something unusual.
j. the results were surprising.

1. When two people disagree about something, ____
2. If you work out a problem, ____
3. Usain Bolt is the fastest runner ever; ____
...
```

---

## 答题判定与状态更新

用户提交答案后（直接输入或使用 `voc answer ...`）：

### 判定规则
1. 对每个目标词，检查其在 **Type 1 和 Type 2** 中是否都答对。
2. **一个词必须在两种题型中都正确**，才算本轮通过。

### 状态迁移

| 词来源 | 本轮结果 | 动作 |
|---|---|---|
| 新词 | 两题型都对 | 加入 `mastered` |
| 新词 | 任一题型错 | 加入 `wrong_bank`，`consecutive_correct = 0` |
| `wrong_bank` | 两题型都对 | `consecutive_correct += 1`；若 ≥ 2 → 移入 `mastered` 并从 `wrong_bank` 删除 |
| `wrong_bank` | 任一题型错 | `consecutive_correct = 0`（保留在 `wrong_bank`） |

### 更新 cursor
- cursor 推进至本轮新词中的最大下标 + 1（仅当本轮包含新词时）。

### 清空 pending 并写回状态文件。

### 回执
向用户报告：本轮每词正误（分 Type 1 / Type 2）、新增/移出错词库情况、当前 unit 进度统计。

---

## 完整流程示例：`voc train 1`

1. Read `~/.openclaw/state/voc-exercise/state.json`（不存在则初始化）
2. Read `vocabulary/units/Unit_1.md` → 解析词表
3. 抽词：cursor=0, wrong_bank 空 → 取前 10 词 `[mystery, mysterious, astronomer, asteroid, interstellar, speed, direction, increase, decrease, alien]`
4. 生成 **Type 1**：10 道填空题，所有目标词列在选项中
5. 生成 **Type 2**：10 个句子配对，10 个选项（含合理干扰）
6. 将 `pending` 写入 state.json
7. 输出两套题，提示用户作答

用户提交答案后：
8. Read state.json → 读取 `pending`
9. 逐词判定 Type 1 + Type 2 正误
10. 迁移状态，更新 cursor
11. Write state.json
12. 回执报告

---

## 错误处理

- **Unit 不存在**：提示有效范围 1–5
- **state.json 损坏**：提示用户备份后运行 `voc reset <unit>` 或手动修复
- **vocabulary 文件缺失**：提示检查仓库路径
- **无 pending 时收到 answer**：提示先运行 `voc train <unit>`
