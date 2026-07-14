# 批改图片标注工具

四个基于浏览器的本地静态网页工具，构成完整的「标注 → 查看 → 分析」工作流。所有数据均在浏览器本地处理，**不会上传到任何服务器**。

## 在线使用

直接点击链接打开，无需安装：

| 工具 | 用途 | 链接 |
|------|------|------|
| **主观题批改工具** | 在卷面上标记错题、挂错误原因、导出结果 | [demo-subjective.html](https://shiren23.github.io/grading-image-annotation-tool/demo-subjective.html) |
| **主观题批改结果查看器** | 只读浏览批改结果 + AI 对话分析 | [result-viewer-v2.html](https://shiren23.github.io/grading-image-annotation-tool/result-viewer-v2.html) |
| **整卷正确率统计工具** | 由欣懿制作 | [grading-tool.html](https://octavia-11.github.io/grading--tool/question-counter.html) |
| 已弃用工具 |  | |
| **图片批阅标注工具** | 用矩形框框选错误位置，导出 ZIP | [annotation-tool.html](https://shiren23.github.io/grading-image-annotation-tool/annotation-tool.html) |
| **标注结果查看工具** | 浏览 bbox 标注结果，支持编辑 | [result-viewer.html](https://shiren23.github.io/grading-image-annotation-tool/result-viewer.html) |


> 推荐 Chrome / Edge / Safari，不需要后端服务。

---

## 新手入门

### 我该用哪个工具？

```
你的数据长什么样？
│
├─ "留痕与详情" 文件夹（含 task_*.json + 卷面图片）
│   │
│   ├─ 我想标记错题、批改 → 「主观题批改工具」
│   │
│   └─ 我想查看别人批改完的结果 → 「主观题批改结果查看器」
│
└─ 普通文件夹（只有图片 + metadata.yaml）
    │
    ├─ 我想框选错误位置 → 「图片批阅标注工具」
    │
    └─ 我想查看已导出的 ZIP 标注 → 「标注结果查看工具」
```

### 环境要求

- 现代浏览器（推荐 **Chrome / Edge**）
- 不需要安装任何软件
- 不需要联网（AI 对话功能除外）

---

## 主观题批改工具（demo-subjective.html）

### 什么是"留痕与详情"文件夹？

这是 VLM 后端自动生成的批阅数据目录，结构如下：

```
留痕与详情/
├── 参考卷/                     ← 可选，自动识别作对照用
│   ├── 1.jpeg ~ 4.jpeg        ← 卷面扫描图
│   └── task_<uuid>.json       ← VLM 返回的题目结构 + 判定结果
└── 未匹配/                     ← 学生卷子
    ├── 1/
    │   ├── 1.jpg, 2.jpg
    │   └── task_<uuid>.json
    ├── 2/ ...
    └── N/ ...
```

每个含 `task_*.json` + 图片的子目录会被识别为一份卷子。

### 使用步骤

**第 1 步：打开工具**

访问 https://shiren23.github.io/grading-image-annotation-tool/demo-subjective.html

**第 2 步：导入数据**

1. 点击右上角 **📁 导入**
2. 选择"留痕与详情"目录（整个文件夹，不是单个子目录）
3. 工具自动解析，默认显示第一份卷子

> 也可以本地双击 `demo-subjective.html` 打开（浏览器允许 file:// 读取 webkitdirectory 选中的文件）。

**第 3 步：标记错题**

三种标记方式，数据互通：

| 方式 | 操作 |
|------|------|
| **点答题线** | 直接点击卷面上的答题线区域，切换对/错 |
| **点题号 chip** | 点击右侧题号栏的数字方块 |
| **键盘** | `↑` `↓` 切空 → `Enter` 切换对错 |

**第 4 步：挂错误原因**

选中错题后，在中间详情面板：
1. 点选 4 大类之一（切题 / OCR / 解题 / 判题）
2. 再点选具体子类
3. 可选：填写备注

> 快捷键：`1`-`4` 先选大类，再按 `1`-`4` 选子类，自动标记为错。

**第 5 步：导出结果**

- **📦 导出 JSON** — 导出批改结果（含 user_marks + error_reasons），供查看器使用
- **📊 导出 Excel** — 导出统计表

### 快捷键速查

| 快捷键 | 功能 |
|--------|------|
| `↑` `↓` | 切换到上/下一个空（末尾自动切下一学生） |
| `←` `→` | 上一张 / 下一张卷面图 |
| `,` | 下一张图，末张则切下一学生 |
| `Enter` / `空格` | 切换当前选中空的对错 |
| `1` `2` `3` `4` | 选中空后：先选错误大类，再选子类 |
| `0` | 缩放适应宽度 |
| `1` | 100% 缩放（未选中空时） |
| `+` `−` | 放大 / 缩小 |
| `Esc` | 取消选中 |

### Excel 大题分组

导出 Excel 时，按以下顺序识别"大题"：

1. **手动输入**（推荐）：页面右上角填写，如 `1-8,9-11,12-16`
2. 自动读取上次保存的分组
3. 自动识别题干中的 `一、` `二、` 等大题标题
4. 退化：每个小题单独一组

---

## 主观题批改结果查看器（result-viewer-v2.html）

### 使用步骤

**第 1 步：导入两份数据**（都需要）

| 按钮 | 导入什么 | 获取什么 |
|------|----------|----------|
| **📁 导入文件夹** | "留痕与详情"目录 | 卷面图片 + 题目位置 |
| **📄 导入 JSON** | 批改工具导出的 `主观题批改结果_*.json` | 批改标记 + 错误原因 |

工具按 `task_id` 自动匹配合并。

**第 2 步：浏览**

- **左侧**：试卷列表，红点=有错误，绿点=全对
- **卷面图**：只高亮用户标记为「错」的答题区（红色），点击查看详情
- **详情面板**：只读展示题号、题型、判定、题干、答案、错误原因
- **题号栏**：绿=对，红=后端判错，橙=用户手标错，灰=未判定

**第 3 步：AI 对话分析**（可选）

1. 在页面右上角输入 **DeepSeek API Key**（本地存储，不上传）
2. 点击右下角 **💬** 按钮打开对话框
3. AI 能看到所有卷子的汇总统计，可以问：
   - "整体情况如何？"
   - "哪些题错误率最高？"
   - "第 5 题的主要错误原因是什么？"

> 模型：`deepseek-v4-pro`，端点 `api.deepseek.com`

### 快捷键

| 快捷键 | 功能 |
|--------|------|
| `←` `→` | 上一张 / 下一张卷面图 |
| `0` | 缩放适应 |
| `1` | 100% 缩放 |
| `+` `−` | 放大 / 缩小 |
| 滚轮 | 以光标为中心缩放 |
| 双击 | 切换 fit / 100% |
| 拖拽 | 平移卷面 |
| `Esc` | 取消选中 |

---

## 图片批阅标注工具（annotation-tool.html）

### 使用步骤

**第 1 步：准备文件夹**

```
批阅任务文件夹/
├── 未匹配/
│   ├── 1/
│   │   ├── 1.jpg
│   │   └── metadata.yaml      ← 包含 task_ids
│   └── ...
└── 学生卷/
    └── 张三/
        ├── 张三.jpg
        └── metadata.yaml
```

**第 2 步：导入并标注**

1. 打开 [标注工具](https://shiren23.github.io/grading-image-annotation-tool/annotation-tool.html)
2. 点击「📁 选择文件夹」，选择最外层目录
3. 在左侧选择错误类型（如「切题」→「答非所问」）
4. 在图片上**左键拖拽**框选错误位置
5. 在左侧填写 comment
6. 「✓ 保存标注」→ 自动跳到下一张

**第 3 步：导出与查看**

1. 点击「📦 导出结果」下载 ZIP
2. 解压（或直接选解压目录）
3. 打开 [结果查看工具](https://shiren23.github.io/grading-image-annotation-tool/result-viewer.html)
4. 选择解压后的目录
5. 工具自动加载，可浏览/编辑错误详情

### 快捷键

| 快捷键 | 功能 |
|--------|------|
| `←` `→` | 上一张 / 下一张图片 |
| `Z` | 撤销最后一个 bbox |
| `Delete` | 删除选中的 bbox |
| `Esc` | 取消绘制中 |
| `S` / `Enter` | 保存 / 跳过 |

---

## 错误分类法

主观题批改工具和图片标注工具共用 4 大类错误分类：

| 大类 | 颜色 | 子类 |
|------|------|------|
| **切题** | 🔴 红 | 未切出此题、切题不完整、未切出图例 |
| **OCR** | 🔵 蓝 | 单字识别错、漏字、公式错、格式/标点 |
| **解题** | 🟠 橙 | 逻辑错、计算错、公式错、步骤缺失 |
| **判题** | 🟣 紫 | 对错结论错、分数错、漏判、标签错误 |

图片标注工具的分类法由 `taxonomy.json` / `taxonomy.js` 定义，修改后两个文件需保持同步（`.js` 是 `window.TAXONOMY = {...}` 包装版）。

---

## 部署方式

### 方式一：在线使用（推荐）

直接打开上面的 GitHub Pages 链接。

### 方式二：本地打开

1. 下载 `annotation-tool.html`、`result-viewer.html`、`taxonomy.js`（三个文件必须放同一目录）
2. 浏览器双击打开

### 方式三：本地 HTTP 服务器

```bash
python3 -m http.server 8080
# 访问 http://localhost:8080/annotation-tool.html
```

### 方式四：部署到静态托管

GitHub Pages / Vercel / Netlify / Nginx 均可。

---

## 常见问题（FAQ）

### Q：导入后什么都没显示？

检查文件夹结构：
- 主观题工具：每个子目录必须有 `task_*.json` + 至少一张图片（jpg/png）
- 标注工具：每个子目录必须有 `metadata.yaml` + 图片

### Q：数据会上传到服务器吗？

**不会。** 所有图片和数据均在浏览器本地处理。图片用 Blob URL 在内存中加载，不经过任何服务器。

### Q：刷新页面后数据还在吗？

- **主观题批改工具**：批改结果累积在内存中，刷新会丢失未导出的数据。请及时导出 JSON。
- **图片标注工具**：使用 localStorage 增量持久化，刷新不丢，启动时提示恢复。
- **结果查看器**：只读工具，每次打开需要重新导入数据。API Key 存 localStorage。

### Q：主观题查看器的 AI 对话报错？

1. 确认右上角已输入正确的 DeepSeek API Key
2. 确认网络可以访问 `api.deepseek.com`
3. API Key 存在浏览器 localStorage，不会上传到本工具的服务器

### Q：Excel 导出的大题分组不对？

在导出前，手动在页面右上角填写大题范围，如 `1-8,9-11,12-16`，表示第一大题为第 1-8 题，第二大题为第 9-11 题。填一次后会记住，下次自动使用。

### Q：file:// 双击打开有什么限制？

- `annotation-tool.html` 的 sha256 计算（SubtleCrypto）可能不可用，`source_hash` 字段会置 null，不影响正常使用
- `taxonomy.js` 通过 `<script>` 标签加载，必须与 HTML 文件在同一目录

### Q：旧数据（marked_*.jpg + error_info.txt）能用吗？

结果查看工具自动兼容旧格式（双路解析）。如需升级到新格式，用迁移脚本：

```bash
python3 migrate_legacy_zip.py 批阅标注结果_xxx/         # 就地迁移
python3 migrate_legacy_zip.py 批阅标注结果_xxx.zip --output result_v2.zip  # 输出新 ZIP
python3 migrate_legacy_zip.py 批阅标注结果_xxx/ --dry-run  # 只看 diff 不写盘
```

---

## 浏览器兼容性

- ✅ **Chrome / Edge**（推荐，支持 File System Access API 直接覆盖原文件）
- ✅ Safari
- ✅ Firefox（部分 API 降级）

---

## 文件说明

| 文件 | 说明 |
|------|------|
| `demo-subjective.html` | **主观题批改工具** |
| `result-viewer-v2.html` | **主观题批改结果查看器** |
| `annotation-tool.html` | **图片批阅标注工具** |
| `result-viewer.html` | **标注结果查看工具** |
| `taxonomy.json` / `taxonomy.js` | 错误分类法（数据 + file:// 兼容包装） |
| `docs/schema.md` | annotation.json 字段定义 |
| `migrate_legacy_zip.py` | 旧格式迁移脚本 |
| `test_*.py` | Playwright 自动化测试 |

---

## 许可证

MIT License
