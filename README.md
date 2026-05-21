# AI-Test-Generator

多Agent测试用例生成系统 - 基于蚁群仿生架构的智能测试用例生成工具

## 概述

本项目实现了一个多Agent协作的测试用例生成系统，通过模拟蚁群觅食行为，实现：
- **Lead** 分析需求 → **Generator** 编写用例 → **Reviewer** 审查 → **迭代优化**
- 支持最多3轮迭代，自动修复覆盖率不足等问题
- 最终输出Markdown和Excel两种格式的测试用例

## 目录结构

```
{workdir}/                          # 工作目录（可配置）
├── skills/                          # AI Agent技能定义
│   └── ccg/
│       └── test-generation/         # 测试用例生成相关技能
│           ├── SKILL.md            # 主入口
│           ├── roles/               # Agent角色定义
│           │   ├── lead.md
│           │   ├── generator.md
│           │   └── reviewer.md
│           ├── workflow/           # 执行流程定义
│           │   ├── lead-execute.md
│           │   ├── generator-execute.md
│           │   ├── generator-fix-execute.md
│           │   └── reviewer-execute.md
│           └── templates/          # 模板文件
│
├── rules/                           # 工作流规则
│   └── test-generation-workflow.md  # 测试用例生成工作流规则
│
├── tools/                           # 工具目录
│   └── convert_to_excel.py          # Markdown转Excel工具
│
├── output/                          # 输出目录（可配置）
│   └── {需求名称}_{YYYY-MM-DD_HHmmss}/  # 每次生成创建独立文件夹
│       ├── features.md
│       ├── test_cases_v1.md
│       ├── test_cases_v2.md
│       ├── test_cases_v3.md
│       ├── test_cases_final.md
│       ├── review_report.md
│       ├── review_report_final.md
│       ├── iteration_log.md
│       └── {需求名称}.xlsx
│
└── README.md
```

## 核心流程

```
需求文档 → Lead分析 → Generator并行 → Reviewer审查
                                      ↓
                               有问题?
                              ↓是↓  ↓否
                           修复迭代  最终交付
                              ↓
                         再次Review
                              ↓
                         循环直到通过（最多3轮）
```

## 触发方式

在AI对话中输入以下关键词即可触发：

| 触发词 | 说明 |
|--------|------|
| `生成测试用例` | 中文触发 |
| `test generation` | 英文触发 |
| `测试用例生成` | 同上 |
| `编写测试` | 手工触发 |
| `test agent` | 多Agent场景 |

## 质量标准

> 质量标准统一在 `rules/test-generation-workflow.md` → 配置标准 中定义

## 输出格式

### Markdown格式 (test_cases_final.md)

```markdown
### TC-001: 已购用户显示入口
- **前置条件**: 用户已完成POP资料购买，权益已开通
- **测试步骤**:
  1. 用户登录App
  2. 进入首页
- **预期结果**: 首页展示「付费专属资料」模块
- **优先级**: P0
- **测试类型**: 功能
```

### Excel格式 ({需求名称}.xlsx)

| ID | 标题 | 目录 | 前置条件 | 步骤描述 | 预期结果 | 负责人 | 优先级 |
|----|------|------|----------|----------|----------|--------|--------|
| TC-001 | 已购用户显示入口 | F8-首页付费专属资料 | 用户已完成POP资料购买... | 1. 用户登录App 2. 进入首页 | 首页展示... | | P0 |

- Sheet名: `testcase items`
- 表头行冻结
- 列宽自适应

## 使用方法

### 1. 准备需求文档

将需求文档（Word、Markdown等格式）准备好，AI会自动读取分析。

### 2. 触发生成

在AI对话中输入：
```
为{需求名称}生成测试用例
```

### 3. 自动执行流程

1. **Lead分析** - 读取需求文档，提取功能点，生成features.md
2. **Generator并行** - 2个Generator同时编写测试用例
3. **Reviewer审查** - 检查覆盖率、正确性、冗余性
4. **迭代修复** - 如有问题，自动进入第2轮、第3轮迭代
5. **最终交付** - 输出test_cases_final.md和Excel文件

### 3. 单独使用Excel转换工具

```bash
python tools/convert_to_excel.py <输入md路径> [输出目录]

# 示例
python tools/convert_to_excel.py ./test_cases_final.md
# 默认输出到 output/需求名称.xlsx
python tools/convert_to_excel.py ./test_cases_final.md ./output/
# 指定输出目录
```

## 去重机制

合并v1/v2/v3版本时，系统会自动去重：
- 每个TC编号只保留最新版本（编号最大的）
- 例如：TC-001在v1/v2/v3中都存在，只保留v3的版本

## 版本历史

| 版本 | 说明 |
|------|------|
| v1 | 初始生成 |
| v2 | 第1次修复迭代 |
| v3 | 第2次修复迭代 |
| final | 最终交付（已去重） |

## 复用方式

将skills、rules和tools复制到其他项目即可复用：

1. 复制 `skills/test-generation/` 到目标skills目录
2. 复制 `rules/test-generation-workflow.md` 到目标rules目录
3. 复制 `tools/convert_to_excel.py` 到目标tools目录

> **输出目录**: `output/` 文件夹会在首次生成时**自动检测**——如果存在则直接使用，如果不存在则自动创建。
>
> **质量标准、文件命名、目录结构**统一在 `rules/test-generation-workflow.md` 中管理，修改一处全局生效。
>
> 如需自定义Agent行为（如任务分配策略、审查严格程度），可修改 `skills/test-generation/roles/` 下的角色文件：
> - `lead.md` - Lead任务分配逻辑
> - `generator.md` - Generator用例编写规范
> - `reviewer.md` - Reviewer审查标准

## 依赖

- Python 3.10+
- python-docx (用于读取Word文档)
- openpyxl (用于生成Excel)

```bash
pip install python-docx openpyxl
```

```bash
pip install python-docx openpyxl
```

## 示例输出

实际生成结果位于 `output/【阿虎医考App】v9.3.6 - POP付费资料_2026-05-14_100000/` 目录，包含118个测试用例。