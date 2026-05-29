---
name: test-generation-workflow
description: 多Agent测试用例生成workflow自动触发规则。包含Lead→Generator→Reviewer的迭代循环，直到生成最终合格用例。
parent: ccg-skills.md
---

# 测试用例生成工作流规则 (迭代增强版)

## 核心流程：Review循环

```
需求文档 → Lead分析 → Generator并行 → Reviewer审查
                                      ↓
                               有问题?
                              ↓是↓  ↓否
                           修复迭代  最终交付
                              ↓
                         再次Review
                              ↓
                         循环直到通过
```

**关键改进**：审查后发现问题，自动进入修复迭代流程，直到生成合格用例。

---

## 配置标准（权威来源）

> 以下配置为系统唯一标准，其他文件引用此处，不再重复定义

### 质量标准

| 维度 | 目标 | 低于目标处理 |
|------|------|--------------|
| 功能覆盖 | 100% | Critical |
| 边界覆盖 | ≥20% | Critical |
| 异常覆盖 | ≥10% | Critical |
| P0占比 | ≥25% | Warning |

### 文件命名

| 文件 | 命名 |
|------|------|
| 测试用例v1 | test_cases_v1.md |
| 测试用例v2 | test_cases_v2.md |
| 测试用例v3 | test_cases_v3.md |
| 最终用例 | test_cases_final.md |
| Excel | {需求名称}.xlsx |

### 目录结构

```
{workdir}/output/{需求名称}_{YYYY-MM-DD_HHmmss}/
```

> **Excel文件位置**: `output/{需求名称}_{日期时间}/{需求名称}.xlsx`（与MD文件在同一时间戳文件夹）

### Excel格式字段

| 列号 | 字段名 |
|------|--------|
| 1 | ID |
| 2 | 标题 |
| 3 | 目录 |
| 4 | 前置条件 |
| 5 | 步骤描述 |
| 6 | 预期结果 |
| 7 | 负责人 |
| 8 | 优先级 |

---

## 触发条件

当用户提到以下关键词时，自动激活测试用例生成workflow：

| 触发词 | 含义 |
|--------|------|
| `生成测试用例` | 需要为需求/功能点生成测试用例 |
| `test generation` | 英文触发 |
| `测试用例生成` | 同上 |
| `编写测试` | 手工触发 |
| `test agent` | 多Agent测试场景 |

---

## 阶段定义

### Phase 1: Lead分析

**触发**: 用户输入包含触发词

**执行**:
1. 识别需求文档路径（用户输入或新建）
2. **文档格式检测与转换**:
   - 如果是 `.docx` / `.doc` 格式 → 使用 python-docx 转换为临时 `.md` 文件
   - 如果是 `.md` 格式 → 直接使用，无需转换
   - 如果是其他格式 → 尝试转换或提示用户
3. 读取转换后的需求文档
4. 分析并提取功能点
5. **【强制】发现并列出至少5个待确认项**（见下方"待确认项清单"）
6. 分配任务给Generator

**待确认项清单（强制要求）**：

Lead分析完需求后，**必须**生成至少5个待确认项，格式如下：

```markdown
## Lead待确认项清单

| 序号 | 类别 | 待确认内容 | 重要性 |
|------|------|------------|--------|
| 1 | 需求模糊 | [具体问题描述] | P0/P1 |
| 2 | 边界条件 | [具体问题描述] | P0/P1 |
| 3 | 技术风险 | [具体问题描述] | P0/P1 |
| 4 | 影响范围 | [具体问题描述] | P0/P1 |
| 5 | 优先级 | [具体问题描述] | P0/P1 |
```

**待确认项类别**：
- **需求模糊**：功能描述不清晰、有歧义
- **边界条件**：边界值、异常场景未明确
- **技术风险**：实现方案存在不确定性
- **影响范围**：对现有系统的影响范围不明确
- **优先级**：功能点优先级需要确认
- **用户体验**：交互细节、UI表现未明确
- **数据边界**：数据量、并发量、性能要求未明确
- **兼容性**：多端兼容、版本兼容问题

**必须调用PM与用户沟通确认后，才能进入Generator阶段**。

**文档转换命令** (docx → md):
```python
from docx import Document
import re

def docx_to_markdown(docx_path, output_path):
    """将Word文档转换为Markdown格式"""
    doc = Document(docx_path)
    lines = []

    for para in doc.paragraphs:
        text = para.text.strip()
        if text:
            lines.append(text)

    # 处理表格
    for table in doc.tables:
        for row in table.rows:
            row_data = [cell.text.strip() for cell in row.cells]
            if any(row_data):
                lines.append('| ' + ' | '.join(row_data) + ' |')
        lines.append('')

    with open(output_path, 'w', encoding='utf-8') as f:
        f.write('\n'.join(lines))
```

**输出**: `features.md`

---

### Phase 2: Generator并行编写 (第1轮)

**触发**: Lead完成功能点分配

**执行**:
1. 启动多个Generator Agent
2. 每个Generator分配独立功能点子集
3. 并行执行，互不阻塞

**输出**: `test_cases_v1.md`

---

### Phase 3: Reviewer审查 (第1轮)

**触发**: Generator完成

**执行**:
1. 收集所有Generator输出的测试用例
2. 启动Reviewer Agent审查
3. 输出结构化审查报告

**输出**: `review_report.md`

---

### Phase 4: 迭代循环 (关键！)

**触发**: Reviewer审查后发现以下任一Critical问题

| 维度 | 触发阈值 | 严重级别 |
|------|----------|----------|
| 功能覆盖 | <100% | Critical |
| 边界覆盖 | <20% | Critical |
| 异常覆盖 | <10% | Critical |

**执行循环**:

```
while (还有Critical问题 && 循环次数 < 3):
    │
    ├── Lead分析审查报告
    ├── 识别需修复的用例
    ├── 分配修复任务给Generator
    │
    ├── Generator修复/补充用例
    │
    ├── 更新版本: v{N} → v{N+1}
    │
    └── Reviewer再次审查
         │
         └── 判断: 还有Critical问题?
```

**循环退出条件**:
- ✓ 无Critical问题
- ✓ 覆盖率达标（功能100%、边界≥20%、异常≥10%）
- ✓ (最多3轮)

**⚠️ 强制规则**：覆盖率不达标时不得跳过迭代直接交付

**输出**: `test_cases_v{N}.md`, `review_report.md` (更新)

---

### Phase 5: 最终交付

**触发**: 循环退出条件满足

**执行**:
1. 汇总所有版本 - **关键：必须去重，每个TC编号只保留最新版本**
2. 输出最终测试用例 (`test_cases_final.md`)
3. 生成迭代日志 (`iteration_log.md`)
4. **生成Excel格式测试用例** - 使用 `convert_to_excel.py` 转换最终用例

**版本合并规则（重要）**:
```
合并v1/v2/v3时，对于重复的TC编号：
- 只保留编号最大的版本（最新迭代的用例）
- 例如：TC-001 在 v1、v2、v3 中都存在 → 只保留 v3 的版本
- 原因：每次迭代可能修改同一用例，后面的版本更完善
```

**去重算法**:
```python
# 伪代码：合并时去重
seen_tcs = set()
new_lines = []
for line in lines:
    if line starts with "### TC-XXX:":
        if tc_id in seen_tcs:
            continue  # 跳过旧版本
        seen_tcs.add(tc_id)
    new_lines.append(line)
```

**Excel生成命令**:
```bash
python {workdir}/tools/convert_to_excel.py {output_dir}/test_cases_final.md
```

**说明**: `{workdir}` 为工作目录根路径，`{output_dir}` 为测试用例输出目录（如 `output/【需求名】_YYYY-MM-DD_HHmmss/`）

**输出**:
- `test_cases_final.md` - 最终测试用例(Markdown)，已去重
- `{需求名称}.xlsx` - **最终测试用例(Excel格式)** - 必须生成
- `review_report_final.md` - 最终审查报告
- `iteration_log.md` - 迭代日志

---

## 迭代管理

### 版本命名规则

```
第1轮生成: test_cases_v1.md
第2轮修复: test_cases_v2.md (可能新增/修改TC编号)
第3轮修复: test_cases_v3.md (可能新增/修改TC编号)
最终版本: test_cases_final.md (合并v1~v3，必须去重)
```

**重要**：Generator在每次迭代时应使用不同的TC编号段避免冲突：
- v1: TC-001 ~ TC-099
- v2: TC-101 ~ TC-199 (或新增 TC-201~TC-299)
- v3: TC-301 ~ TC-399 (或新增 TC-401~TC-499)

或采用更简单的方案：**直接覆盖**，只保留最新版，TC编号保持连续。

### 循环次数限制

| 次数 | 行为 |
|------|------|
| 1-2次 | 正常迭代，满足则提前结束 |
| 3次 | 强制交付，标注剩余问题 |
| >3次 | 停止，输出当前版本 + 问题说明 |

### 提前结束条件

第2轮迭代完成后，如果满足以下条件则**提前结束**，不再进行第3轮：
- ✓ 功能覆盖 = 100%
- ✓ 边界覆盖 ≥20%
- ✓ 异常覆盖 ≥10%
- ✓ P0占比 ≥25%

### 迭代日志格式

```markdown
# 迭代日志

## v1 → v2 迭代
- 迭代原因: 边界覆盖率不足
- 修复内容:
  - 新增TC-XXX, TC-YYY (边界测试)
  - 修改TC-ZZZ (步骤不清晰)
- 修复后覆盖率: 边界 15% → 45%

## v2 → v3 迭代
- 迭代原因: Critical问题未完全修复
- 修复内容:
  - 新增TC-AAA (售价边界)
  - 新增TC-BBB (批量导入边界)
- 修复后覆盖率: 边界 45% → 72%

## 最终状态
- 最终版本: v3
- Critical问题: 0
- 覆盖率: 全部达标
```

---

## 质量门禁

| 阶段 | 检查项 | 失败动作 |
|------|--------|----------|
| Lead分析 | 功能点≥5 | 要求用户补充需求 |
| Generator编写 | 用例数≥功能点×2 | 警告，提示覆盖不足 |
| Review审查 | Critical问题=0? | **进入修复迭代循环** |
| Review审查 | 覆盖率达标? | **进入修复迭代循环** |
| 迭代(3次后) | 仍有问题 | 强制交付+问题说明 |

> **强制规则**：Review阶段发现Critical覆盖率问题，必须进入修复迭代循环，不得跳过迭代直接交付

---

## 质量标准

| 维度 | 目标 | 低于目标处理 |
|------|------|--------------|
| 功能覆盖 | 100% | Critical |
| 边界覆盖 | ≥20% | Critical |
| 异常覆盖 | ≥10% | Critical |
| P0占比 | ≥25% | Warning |

---

## 问题处理规则

### Critical问题 (必须修复)

1. **覆盖率不达标** - 功能<100%, 边界<20%, 异常<10%
2. **缺少核心功能测试** - P0功能点无测试
3. **用例不可执行** - 步骤模糊或结果不明确
4. **用例重复** - 冗余用例未清理

### Warning问题 (建议修复)

1. **边界覆盖不足** - 缺少边界值测试（但边界≥20%才为Critical）
2. **异常场景遗漏** - 重要异常未覆盖（但异常≥10%才为Critical）
3. **优先级偏差** - P0占比<25%

### Info问题 (可选修复)

1. 措辞优化
2. 前置条件完善
3. 测试步骤细化

---

## 任务状态管理

使用Task工具跟踪进度：

```markdown
TaskCreate:
  - Lead主任务: 分析需求
  - Generator子任务: 编写用例
  - Reviewer子任务: 审查用例
  - Generator-Fix子任务: 修复用例 (迭代时创建)

TaskUpdate:
  - 各Agent更新任务状态
  - metadata记录信息素(discovery/progress/warning)

TaskList:
  - Lead监控整体进度
  - 判断是否进入迭代
```

---

## 错误处理

| 错误 | 处理 |
|------|------|
| Generator失败 | Lead重新分配任务 |
| Reviewer发现问题 | 进入修复迭代流程 |
| 覆盖率不达标 | 进入修复迭代流程 |
| 3次迭代后仍有问题 | 强制交付+问题说明 |

---

## 输出目录结构

**核心原则**：每次生成测试用例时，创建独立的项目文件夹，不覆盖历史数据。

### 目录命名规则
```
{需求名称}_{YYYY-MM-DD_HHmmss}/
```

### 示例
```
{workdir}/output/
├── 【阿虎医考App】v9.3.6 - POP付费资料_2026-05-14_143000/
│   ├── features.md              # 功能点清单
│   ├── test_cases_v1.md       # 第1轮测试用例
│   ├── test_cases_v2.md        # 第2轮(修复后)
│   ├── test_cases_v3.md        # 第3轮(修复后)
│   ├── test_cases_final.md     # 最终测试用例
│   ├── review_report.md         # 审查报告(最新版)
│   ├── review_report_final.md   # 最终审查报告
│   └── iteration_log.md        # 迭代日志
│
├── 【阿虎医考App】v9.3.6 - POP付费资料_2026-05-15_090000/
│   └── ... (同上结构)
│
└── 【订单模块】_2026-05-16_110000/
    └── ... (同上结构)
```

**说明**：`{workdir}` 为执行时的当前工作目录，输出目录可配置。

### 执行流程

1. **识别需求名称**：从需求文档名或用户描述中提取
2. **检查是否存在**：如果 `{需求名称}_*` 文件夹已存在，在其下创建 `{需求名称}_{日期时间}` 子文件夹
3. **创建新文件夹**：如果不存在，创建 `{需求名称}_{日期时间}` 文件夹
4. **所有输出写入该文件夹**

### 文件命名规则
| 文件 | 命名 | 说明 |
|------|------|------|
| 功能点清单 | features.md | 需求分析结果 |
| 测试用例v1 | test_cases_v1.md | 第1轮生成 |
| 测试用例v2 | test_cases_v2.md | 第2轮修复 |
| 测试用例v3 | test_cases_v3.md | 第3轮修复 |
| 最终用例 | test_cases_final.md | 合并v1~v3 |
| **Excel用例** | `{需求名称}.xlsx` | **动态名称，与需求一致** |
| 审查报告 | review_report.md | 每次审查输出 |
| 最终审查 | review_report_final.md | 最终审查报告 |
| 迭代日志 | iteration_log.md | 迭代过程记录 |
| 临时md | `{需求名}_temp.md` | docx转换后的临时文件 |

---

## 测试用例Excel模板格式

**必须生成Excel格式测试用例，字段与模板一致：**

| 列号 | 字段名 | 说明 |
|------|--------|------|
| 1 | ID | 测试用例ID，格式：`TC-XXX` |
| 2 | 标题 | 测试用例标题，不含TC-XXX前缀 |
| 3 | 目录 | 功能模块分组，如：`F8-首页付费专属资料` |
| 4 | 前置条件 | 测试前需满足的条件 |
| 5 | 步骤描述 | 详细操作步骤，每步一行 |
| 6 | 预期结果 | 期望的系统行为 |
| 7 | 负责人 | 留空或填写测试工程师姓名 |
| 8 | 优先级 | P0/P1/P2 |

### 示例数据
| ID | 标题 | 目录 | 前置条件 | 步骤描述 | 预期结果 | 负责人 | 优先级 |
|------|------|------|----------|----------|--------|--------|--------|
| TC-001 | 已购用户显示入口 | F8-首页付费专属资料 | 用户已完成POP资料购买，权益已开通 | 1. 用户登录App 2. 进入首页 | 首页展示「付费专属资料」模块 | | P0 |
| TC-002 | 未购买用户不显示 | F8-首页付费专属资料 | 用户未购买任何POP资料 | 1. 用户登录App 2. 进入首页 | 首页不展示「付费专属资料」模块 | | P0 |

### 输出要求
1. **必须生成Excel文件** - 文件名与需求名称一致，如 `{需求名称}.xlsx`
2. Sheet名: `testcase items`
3. 使用 `openpyxl` 库生成
4. 表头行冻结
5. 列宽自适应

---

## 完整执行示例

```
用户: 为订单模块生成测试用例

AI自动执行:
1. Lead读取需求文档
2. Lead提取功能点（订单创建、支付、取消等）
3. Lead分配任务：
   - Generator-1: 订单创建、订单查询
   - Generator-2: 订单支付、订单取消
4. 并行生成 → test_cases_v1.md
5. Reviewer审查 → review_report.md
   └─ 发现: 边界覆盖不足, Critical
6. Lead分配修复任务
7. Generator修复 → test_cases_v2.md
8. Reviewer再次审查 → review_report.md (更新)
   └─ 通过: 覆盖率达标, 无Critical
9. 最终交付: test_cases_final.md + 迭代日志
```

---

## 复用方式

其他AI系统可通过以下方式复用：

1. **复制skill文件**到目标系统的skills目录
2. **复制rule文件**到目标系统的rules目录
3. **按需修改**角色Prompt中的细节参数