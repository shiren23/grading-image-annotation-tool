# `questions.txt` — 文本驱动题号识别接口

当 VLM 未配置（API key 留空）时，标注工具不调用 VLM、不显示"未配置 VLM"错误，转而读取与学生文件夹同级的 `questions.txt`，按文件内容填充该卷的题号列表与正误初值。这是一份**接口约定**，便于人工或前序工具产出可被标注工具直接消费的题号数据。

## 1. 文件位置与命名

```
root/
├── 张三/
│   ├── metadata.yaml         # 现有约定，定义 task_id
│   ├── questions.txt         # 新增：本题号文件（可选）
│   ├── 1.jpg
│   └── 2.jpg
├── 李四/
│   ├── metadata.yaml
│   ├── questions.txt
│   └── 1.jpg
```

- 文件名固定：`questions.txt`（区分大小写）。
- 位置：与学生文件夹下的 `metadata.yaml` 同级。
- 可选：缺失时该卷不启用文本路径，行为退化为原来的"VLM 未配置则隐藏题号栏"。

## 2. 行格式

```
题型 | 题号 | 小题号 | 正误
```

- 列分隔符：半角竖线 `|`，**按 `|` split 后逐列 trim，仅取前 4 列**（多余的列直接丢弃，便于在题号/题型里夹带备注）。
- 列数：2 ~ 4 列皆可（详见下表）。
- 列首尾的空白会被 trim。
- 空行：忽略。
- 注释行：首个非空字符为 `#` 时整行忽略。

### 列定义

| 列序 | 字段 | 必填 | 说明 |
|---|---|---|---|
| 1 | 题型 | ✓ | 自由文本。例：`选择题` / `填空题` / `解答题` / `判断题`。仅用于 hover 提示与数据保留，不影响渲染。 |
| 2 | 题号 | ✓ | 大题号。例：`1` / `2` / `10`。允许字母（如 `1A`）。 |
| 3 | 小题号 | ✗ | 形如 `(1)` / `（2）` / `(3)`。**直接与题号字符串拼接**得到 `question_no`，不强制括号格式。留空表示该题无小题。 |
| 4 | 正误 | ✗ | 取值集合见下表。留空视为默认（`correct`，与新 UI 行为一致）。 |

### `question_no` 拼接规则

| 题号 | 小题号 | 拼接结果 |
|---|---|---|
| `1` | _(空)_ | `1` |
| `2` | `(1)` | `2(1)` |
| `3` | `（2）` | `3（2）` |
| `10` | `(3)` | `10(3)` |

> 注意：拼接是**纯字符串相连**，不做归一化。`(1)` 与 `（1）` 视为不同题号。建议在同一份文件内保持风格统一。

### 正误取值

| 写法 | 解析为 |
|---|---|
| `对` / `正确` / `✓` / `correct` | `correct` |
| `错` / `错误` / `✗` / `wrong` | `wrong` |
| `未判` / `未判断` / `-` / `unmarked` | _(不写入 judgments，按默认 `correct` 显示)_ |
| _(空)_ | 同上 |
| 其他 | 解析为未识别，记一条 error，**但题号仍会被加入列表**（仅忽略正误） |

大小写不敏感（`Correct` / `WRONG` 等同）。

## 3. 完整示例

```
# 张三的数学卷
# 列：题型 | 题号 | 小题号 | 正误

选择题   | 1 |    | 对
选择题   | 2 |    | 错
填空题   | 3 | (1) | 对
填空题   | 3 | (2) | 对
填空题   | 4 | (1) |
填空题   | 4 | (2) | 错
解答题   | 5 | (1) | 未判
解答题   | 5 | (2) |
解答题   | 5 | (3) | 错
```

## 4. 加载与优先级

```
parseFolder 阶段（每个 paper）：
  1. 读 metadata.yaml 取 paperId（不变）
  2. 若同目录存在 questions.txt：
     a. 读文本 → parseQuestionsText → { questions, judgments, errors }
     b. paper.questionList = questions
     c. paper.vlmModelId = 'questions.txt'
     d. paper.identifiedAt = 文件 lastModified（或当前时间）
     e. 暂存 paper.textJudgments = judgments（待与 localStorage 合并）
  3. paper.judgments = { ...textJudgments, ...localStorage_judgments }
     （localStorage 是用户后续操作，覆盖文本初值）

renderQuestionBar 阶段：
  - VLM 已配置：走 VLM 路径（缓存 / 全局模板 / 调用）
    即使 questions.txt 存在，VLM 结果会覆盖 paper.questionList
  - VLM 未配置 + questionList 非空（来自 questions.txt）：正常显示，
    状态文本带 "· 文本" 标识
  - VLM 未配置 + questionList 为空：隐藏题号栏（不报错）
```

## 5. 行为细节

### 5.1 题号按钮点击

文本路径产出的题号按钮，与 VLM 路径产出的按钮**行为完全一致**：点击在 `correct` ↔ `wrong` 之间切换，写入 `paper.judgments` 并持久化到 `localStorage['vlmJudgments:<paperId>']`。

### 5.2 持久化覆盖

`localStorage` 中的 judgments 与文本初值合并时，**localStorage 优先**。即：
- 用户在 UI 上把 `3(1)` 从 `对` 切到 `错` → localStorage 记 `{3(1): wrong}`
- 刷新页面 → 文本初值是 `{3(1): correct}`，localStorage 是 `{3(1): wrong}` → 合并后是 `wrong` ✅

如需重置回文本初值：
1. 在浏览器开发者工具删除 `vlmJudgments:<paperId>`
2. **重新选择文件夹**（仅 F5 刷新不会重新跑 `parseFolder`，因为文件夹选择是用户手势）

### 5.3 ↻ 刷新按钮

- VLM 已配置：清所有 paper 缓存 + 全局模板 → 重识别当前卷（覆盖 questions.txt 的题号）
- VLM 未配置：按钮禁用（题号来自静态文件，无需"刷新"）

### 5.4 导出

文本路径下导出的 `paper.json`：
- `questions` 字段会多两个属性：`type`（题型）、`sub`（小题号或 `null`）
- `judgments` 字段同 VLM 路径（合并后的最终判题）
- `vlm_model_id` = `"questions.txt"`
- `identified_at` = questions.txt 的 lastModified

`type` 和 `sub` 是**附加字段**，不破坏 `docs/schema.md` 中 v1.0 的 questions 数组契约（消费方只读 `question_no` 即可）。

## 6. 错误处理

`parseQuestionsText` 是**容错型**解析器：

| 情况 | 行为 |
|---|---|
| 空文件 | 返回 `{questions: [], judgments: {}}`，不报错 |
| 只有注释/空行 | 同上 |
| 列数 < 2 | 该行跳过，记 error |
| 题型/题号为空 | 该行跳过，记 error |
| 题号重复 | 后者覆盖前者（questions 数组里 remove + push），记 error |
| 正误值未识别 | 题号仍加入列表，但忽略正误，记 error |

所有 errors 写入浏览器 console 的 `warn`（带文件路径前缀），不阻塞加载、不弹错误提示。生产环境想看完整错误列表：

```js
parseQuestionsText('你的文本').errors
```

## 7. 与 VLM 路径的切换

| 切换动作 | 结果 |
|---|---|
| 未配置 VLM（用 questions.txt）→ 配置 VLM | 下次切图或点 ↻ → VLM 调用，覆盖 questionList；judgments 保留（按题号匹配，新题号列表里没有的会被 `pruneJudgmentsAgainstQuestions` 清掉） |
| 配置 VLM → 取消配置（清空 API key）→ 刷新 | 重新走 questions.txt 路径；judgments 是 localStorage 留存的 |

## 8. 设计取舍

- **为什么 `|` 而不是 Tab/逗号？** Tab 在文本编辑器里不可见，复制粘贴易错；逗号在中文文本里被用作语义停顿。`|` 视觉清晰，不需要转义。
- **为什么是 4 列 TSV-like 而不是 JSON/YAML？** 文件由人或简单脚本产生，行格式更友好。JSON 多一个逗号就崩，YAML 需要解析库。
- **为什么题号拼接不做归一化？** 避免引入意外副作用（`(1)` 改写为 `(1)` 还是 `1.`？）。保持纯字符串拼接，让数据生产者控制最终形态。
- **为什么 VLM 已配置时 questions.txt 被忽略？** 避免两套数据互相覆盖的不确定性。VLM 是更高优先级的"实时识别"，questions.txt 是 fallback。
