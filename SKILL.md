---
name: voc-exercise
description: "词汇选择题与配对训练器。从一个合并词表中按序抽 10 词，生成选择题（Type 1）和句子配对题（Type 2），自动维护错词库与已掌握库。"
trigger: voc
keywords:
  - voc
  - 词汇
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

1. 解析子命令（无子命令即开始训练 / status / reset）
2. 读取词表文件 `vocabulary/all_words.md`
3. 读取/初始化状态文件 `~/.openclaw/state/voc-exercise/state.json`
4. 按规则抽词、生成两种题型
5. 收集用户答案后，更新状态文件
6. **禁止**跳过状态读写——所有进度必须持久化
7. **禁止**在出题阶段显示任何中文释义——中文意思**只能**在回执（给出答案）时出现，出题时显示释义视为泄题

---

## 子命令

### `voc`
开始训练。直接输入 `voc` 即可，从合并词表中按序抽词。

### `voc status`
查看整体进度（已掌握数 / 错词库数 / 剩余新词数 / 总词数）。

### `voc reset`
清空全部状态（需用户二次确认）。

### `voc answer <答案序列>`
提交本轮答案。格式见下方答题说明。

---

## 词表数据源

**路径**：`vocabulary/all_words.md`（单一合并文件，共 241 词）

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
  "version": 2,
  "cursor": 0,
  "mastered": ["mystery", "speed"],
  "wrong_bank": {
    "alien": { "consecutive_correct": 0, "last_attempt": "2026-06-06T10:00:00+08:00" }
  },
  "pending": {
    "words": ["explore", "speed", "pass", "alien", "increase", "object", "star", "direction", "technology", "astronomer"],
    "questions": {  },
    "issued_at": "2026-06-06T10:00:00+08:00"
  }
}
```

**读写**：使用内置 Read / Write / Edit 工具。文件不存在时初始化为 `{"version":2,"cursor":0,"mastered":[],"wrong_bank":{}}`。

**迁移说明**：v1 → v2 将 `units` 字典展平为顶层字段，所有 unit 合并到一个词池。若检测到 v1 格式，提示用户运行 `voc reset`。

---

## 抽词算法（每轮 = 10 词）

### 首次训练（cursor = 0 且 wrong_bank 为空）：
1. 读取状态。
2. 从词表开头取 10 个新词，跳过已在 `mastered` 中的词。
3. 将本轮 10 个词及生成的题目写入 `pending`。

### 后续训练（wrong_bank 非空 或 cursor > 0）：
1. 读取状态。
2. **优先**从 `wrong_bank` 中按加入时间最早顺序（`last_attempt` 升序）取词，最多 **3 个**。
3. 从 `cursor` 开始向后取新词，跳过已在 `mastered` / `wrong_bank` 的词，补足至 **10 个**（即 3 复习 + 7 新词）。
4. 若 `wrong_bank` 不足 3 个：全部取出，剩余用新词补足。
5. 若词表已耗尽且 `wrong_bank` 为空：返回「所有词汇已全部掌握」。

---

## 题型一：选择题（Type 1 — Multiple Choice）

**格式**：每道题给一个英文句子（含空格），下方列出 **4 个选项**（A/B/C/D），用户选择正确答案。

**规则**：
- 每个目标词生成 **1 道选择题**
- 每题 **4 个选项**：1 个正确答案 + 3 个干扰项
- 干扰项优先从**当前轮次的其他目标词**中选取；若不足 3 个，从词表中其他词补充
- 句子应尽量基于词表中的例句改编
- 选项顺序随机排列
- 正确答案为该题对应的目标词

**输出格式**：
```
题型一：选择题
DIRECTIONS: Choose the correct word to complete each sentence.

1. He dreamt of being an astronaut, so he could ________ space.
   A. explore
   B. increase
   C. pass
   D. compete

2. In 2134, Halley's Comet will ________ very close to Earth.
   A. sink
   B. pass
   C. strike
   D. grow
...
```

---

## 题型二：句子配对题（Type 2 — Match Sentence Starters with Endings）

**格式**：每道题是**两句配对**。**第一句**（编号项）用英文**描述或解释**某个目标词的含义，但**句中不出现该词**；**第二句**（字母选项）是一个**含有该目标词的短句**，目标词**斜体**标注。用户把每个第一句连到正确的第二句。

**规则**：
- 每个目标词生成 **1 组配对（两句）**：
  - **第二句（字母选项 / ending）**：含该目标词的自然短句，目标词用斜体 `*word*` 标注 —— **这是被考点**
  - **第一句（编号项 / starter）**：用英文描述 / 解释该词的含义或使用情境，**不得出现该目标词本身**
- 被考目标词**永远在第二句**，绝不出现在第一句
- 第一句尽量基于词表的释义 / 例句改编，使描述准确指向**唯一**的目标词
- 正确答案必须唯一确定（每个第一句只对应一个第二句）
- 干扰项 = 本轮**其余目标词的第二句**；若本轮目标词不足，从全词表补充
- 把所有第二句（正确项 + 干扰项）打乱后统一列为 a / b / c… 选项

**输出格式**：
```
题型二：句子配对题
DIRECTIONS: Match each sentence starter with the correct ending.
（第一句描述词义；第二句含被考词，斜体）

选项（endings — 每项含一个目标词）：
a. they *explore* space.
b. it is an *asteroid*.
c. it is very *mysterious*.
d. he is an *astronomer*.
...

1. Astronauts travel to new places to learn about them; ____
2. A small rocky object is orbiting the Sun; ____
3. Nobody can explain where the object came from; ____
4. He studies stars and planets for a living; ____
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
向用户报告：
1. **逐词结果**：每个目标词一行，包含 —— 单词 + **中文意思**（取自词表 `Meaning` 列）+ Type 1 正误 + Type 2 正误 + 正确答案（**答错时必须给出**）+ 去向（已掌握 / 错词库）
2. 本轮新增 / 移出错词库的情况
3. 当前进度统计（已掌握 / 错词库 / 剩余新词 / 总词数）

> **必须**为每个词附上中文意思，方便用户对照记忆。

**逐词结果格式示例**：
```
本轮结果：
✅ explore（探索）        Type 1 ✓ | Type 2 ✓ → 已掌握
❌ asteroid（小行星）     Type 1 ✓ | Type 2 ✗（正确答案：b）→ 进入错词库
✅ mysterious（神秘的）   Type 1 ✓ | Type 2 ✓ → 已掌握
❌ astronomer（天文学家） Type 1 ✗（正确答案：C）| Type 2 ✓ → 进入错词库
...

进度：已掌握 12 / 错词库 3 / 剩余新词 226 / 总计 241
```

---

## 完整流程示例：`voc`

1. Read `~/.openclaw/state/voc-exercise/state.json`（不存在则初始化）
2. Read `vocabulary/all_words.md` → 解析词表
3. 抽词：cursor=0, wrong_bank 空 → 取前 10 词 `[mystery, mysterious, astronomer, asteroid, interstellar, speed, direction, increase, decrease, alien]`
4. 生成 **Type 1**：10 道选择题（4 选 1）
5. 生成 **Type 2**：10 组句子配对（第一句释义 + 第二句含被考目标词，斜体），10 个选项（含干扰）
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

- **state.json 损坏**：提示用户备份后运行 `voc reset` 或手动修复
- **vocabulary 文件缺失**：提示检查仓库路径
- **无 pending 时收到 answer**：提示先运行 `voc`
- **v1 状态格式**：提示版本已升级，运行 `voc reset` 清空旧数据
