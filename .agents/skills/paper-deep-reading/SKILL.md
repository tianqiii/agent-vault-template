---
name: paper-deep-reading
description: 深读 `raw/02-papers/` 中的论文 PDF，抽取关键图表到 `assets/`，在 `wiki/` 中沉淀关键公式 / 公式空位 / 结构化证据；若论文明确给出代码仓库链接，尝试发起子代理总结仓库代码对照，未给出则跳过代码对照总结。默认执行器：本地 `pdf_tool.py + paper_deep_read.py`。
user-invocable: true
---

# paper-deep-reading 技能

## 目标与触发

目标：深读论文 PDF，不只写摘要；沉淀可检索的论文证据层，供后续 `/ingest`、`/query`、`/query-with-code` 复用。

触发：
- `/paper-deep-reading <pdf路径>`
- 用户要求“深读论文 / 抽图和公式 / 为代码对照准备资料”
- `ingest` 处理论文时会默认调用本流程；显式调用本技能表示先做“论文证据层”。

产物：
- 图片：`assets/papers/{paper-slug}/figure-01.png`、`table-01.png`、`equation-region-01.png`
- 文本缓存：`.cache/agents/papers/{paper-slug}/source_text.txt`
- Wiki：`wiki/sources/摘要-{slug}.md`，必要时补充 `entities/`、`concepts/`

## 工作流

### 0. 路径解析

```bash
python ".agents/scripts/router.py" paper-deep-reading "<pdf路径或论文标识>"
```

读取 `workspace_root`、`wiki_dir`、`raw_dir`、`index_path`、`log_path`。输入优先级：
1. `raw/02-papers/*.pdf`
2. 用户明确给出的本地 PDF
3. 已归档论文对应的 `wiki/sources/` 页面（只补全，不回读 `raw/09-archived/`）

边界：禁止修改 `raw/` PDF 正文；图片只能写 `assets/`；知识产物只能写 `wiki/`。

### 1. 默认执行

规则直出：

```bash
python ".agents/scripts/paper_deep_read.py" "<pdf>"
```

复杂版面 / 中文文献 / caption 含 `图1/表1` / 自动排序可疑时，走两阶段：

```bash
python ".agents/scripts/paper_deep_read.py" "<pdf>" --selection-mode agent
python ".agents/scripts/paper_deep_read.py" "<pdf>" --selection-mode agent \
  --selected-slot figure-01 \
  --selected-slot figure-03 \
  --selected-slot table-02
```

第一步只返回 `candidate_pool`、`recommended_slots`、`selection_deficit`；第二步才落盘图片并更新 source/index/log。

默认选择口径：
- `pdf_tool.py` 负责定位、预览、裁图；`score` 只表示截图质量。
- `paper_deep_read.py` 负责正文候选召回、内容价值排序，输出 `value_bucket`、`value_score`、`selection_reason`。
- 候选范围默认只看主论文正文，排除 `References`、`Appendix/Appendices`、`Supplementary/Supplemental` 之后内容。
- caption 支持 `Figure/Fig./Table/Tab./图/表`，支持阿拉伯数字与罗马数字。
- 软配额为 `2 图 + 1 表`；缺少高价值表格时用 `selection_deficit` 报告，不用低价值表格补 filler。

### 2. 本地工具速查

```bash
python ".agents/scripts/pdf_tool.py" extract-text <pdf>
python ".agents/scripts/pdf_tool.py" find <pdf> "Figure 1" --mode auto
python ".agents/scripts/pdf_tool.py" snapshot-query-preview <pdf> "Table II" --preset table --mode auto
python ".agents/scripts/pdf_tool.py" render-page <pdf> --page 3 --output /tmp/page-3.png
python ".agents/scripts/pdf_tool.py" snapshot-query <pdf> "Figure 1" --output "assets/papers/{slug}/figure-01.png" --preset figure --mode auto
python ".agents/scripts/pdf_tool.py" snapshot-rect <pdf> --page 3 --rect x0,y0,x1,y1 --output "assets/papers/{slug}/figure-01.png" --preset exact
```

Preset：
- `figure`：caption 在下方的常规图。
- `figure-column`：双栏页面中避免卷入邻栏。
- `table`：标题在上、主体在下的表格。
- `theorem`：定理/命题块，目标是不截 `Proof`。
- `exact`：自动裁图不干净时人工精裁。

### 3. 截图策略

优先保留解释方法的图表，再补效果、效率、鲁棒性或权衡证据。

理论 / 方法核心锚点：
- `Theorem`、`Lemma`、`Proposition`、`Corollary`、`Assumption`、`Property`、`Remark`
- `Algorithm`、`Objective`、`Problem`、`System Model`、`Architecture`、`Framework`、`Loss`、`Optimization`

图 / 表锚点：
- `Fig.`、`Figure`、`Table`、`Tab.`、`图`、`表`

不确定时向用户确认：
- 先用 `find` / `snapshot-query-preview` / `render-page` 自行判断；只有页码、真实 caption、单列/双列布局仍不确定时才问用户。
- 提问只收集必要信息：图/表编号、所在页、占一列还是两列；拿到答案后用 `--page N` 和合适 preset 裁图。

Caption 消歧：
1. 先用 `find` 看全部命中。
2. 区分正文交叉引用（如 `shown in Fig. 4`）与真实 caption（通常以 `Fig. N.` / `Figure N.` 开头）。
3. 多命中时选真实 caption 页，并显式加 `--page N`。
4. 临时输出后验图，再覆盖 `assets/`。
5. source 证据锚点写真实 caption 页码。

必须重裁的情况：
- theorem / method block 缺首尾行，或混入 `Proof` / 下一个块。
- figure / table 混入邻近正文、相邻图表，或只截到部分子图。
- 坐标轴、图例、标题、列名、底线、最后几行缺失。
- 自动裁图过小导致符号、轴标签不可读。

重裁顺序：`render-page` → 观察边界 → `snapshot-rect --preset exact`。

## 抽取要求

### 图片

优先抽：模型结构图、总体流程图、训练/推理流程图、核心机制图、关键实验表格，保存其中最高价值的3张图片即可。Wiki 中统一用 Obsidian embed：

```markdown
![[papers/{paper-slug}/figure-01.png]]
```

命名按角色编号：`figure-01.png`、`table-01.png`、`theorem-01.png`、`technical-core-01.png`；不要用时间戳或把中文作者名塞进文件名。

### 公式

优先抽：总体目标函数 / loss、关键模块打分函数、训练约束、推理异常分数 / 检索分数 / 融合公式，保存其中最高价值的3-5个公式。

保存原则：
- 能转写 LaTeX 时写成 `$$ ... $$`。
- 无法稳定识别时，至少保留公式空位、原文定位、自然语言解释；可附局部截图。
- 不只堆公式，必须解释其训练/推理作用。
- 若论文给出代码仓库链接，再补代码对照线索；否则跳过。

### 代码仓库与代码对照总结

先在论文正文、首页脚注、实验设置、附录、metadata 中查找 GitHub / GitLab / project page / code URL。

- 有明确仓库链接：尝试发起子代理读取仓库入口文件、模型/训练/评估目录，总结 `loss`、`score`、`module`、`metric` 的待核验代码位置，写入 `## 代码对照线索`。
- 无明确仓库链接：跳过代码对照总结；禁止猜仓库、猜文件名、编造实现路径。

## Wiki 落盘结构

`wiki/sources/摘要-{slug}.md` 优先包含：

```markdown
## 核心摘要
- 问题 / 方法 / 结论 / 局限

## 关键图示
- ![[papers/{paper-slug}/figure-01.png]]
- 图示含义；证据：第 N 页 Fig. X caption。

## 表格速览
### Table N — English Title（第 N 页）
| 原文列名 | ... |
|---|---|
| ... | ... |
- **出处**: 第 N 页 Table N。

## 关键公式
### 公式 1：训练目标
- 原文定位：第 N 页附近的 loss/objective/equation 段落。
- 原文描述：正文中的作用说明。
$$
...
$$
- 解释：该式对应训练或推理中的什么步骤。

## 代码对照线索（可选）
- 仅当论文给出代码仓库链接：仓库链接 -> 子代理总结的 loss / score / module / metric 待核验位置。

## 关联连接
- [[EntityName]]
- [[ConceptName]]
```

Entity / concept 页只追加高复用公式或图示，且必须回链 source 页。

## 与 ingest / query-with-code 联动

- 先执行 `/paper-deep-reading` 后再 `/ingest`：`ingest` 应复用结构化结论，不重新从零读论文。
- 直接 `/ingest` 论文：`ingest -> paper-deep-reading -> ingest 收口`。
- `/query-with-code` 优先读取 `## 关键公式`、`## 关键图示`、`## 代码对照线索`（若存在）。

## 索引、日志与回归

若新增或明显增强 wiki 页面，必须同步：

```bash
python ".agents/scripts/write_index.py" \
  --index-path "<index_path>" \
  --section "Sources" \
  --page "摘要-{slug}" \
  --description "论文核心摘要一句话。"
```

刷新 JdocMunch：

```python
jdocmunch_index_local(
  path="<wiki_dir>",
  name="<jdocmunch_repo>",
  incremental=True,
  use_ai_summaries=False,
  use_embeddings="auto"
)
```

`<jdocmunch_repo>`：取 `workspace_root` 目录名，小写，非字母数字转 `-`，去首尾 `-`，追加 `-wiki`；禁止写死 vault 绝对路径或索引名。默认 `incremental=True`；只有索引失真、目录结构大改或 section 检索连续异常才全量重建。

`wiki/log.md` append-only 记录变更与冲突：

```markdown
## [YYYY-MM-DD] ingest | 深读论文并补充关键图示与公式
- **变更**: 更新 [[摘要-{slug}]]；新增 `assets/papers/{paper-slug}/...`
- **冲突**: 无
```

自动回归至少覆盖：
1. 完整命中：高价值方法图、补充图、高价值表格稳定输出 `2 图 + 1 表`。
2. 缺表：不生成低价值 `table-01.png`，用 `selection_deficit.missing.table` 报告。
3. 罗马数字 caption：`Table II`、`Fig. IV` 可召回、排序、选择。
4. 中文 caption：`图1/图 1/表1/表 1` 可召回，并生成中英 query 变体。
5. 确定性重跑：同一 PDF 连跑两次，规则模式下 `query/page_number/kind/value_bucket/selection_rank` 一致。

## 强制约束

- 禁止修改 `raw/` PDF 正文；禁止读取 `raw/09-archived/` PDF 重新深读。
- 禁止把图片写回 `raw/`；图片只能写 `assets/`。
- 禁止只保存截图而不保存可检索的公式文本或公式空位。
- 禁止只保存公式而不解释训练/推理作用。
- 禁止普通相对路径图片链接；统一 `![[...]]`。
- 禁止为低价值装饰图创建 assets。
- 没有代码仓库链接时，禁止输出代码对照总结。
