# 多Agent协作测试用例生成系统

> —— 基于蚁群仿生架构的实践指南

---

## 一、项目概述

本项目实现了一个多Agent协作的测试用例生成系统，通过模拟蚁群觅食行为，实现了从需求分析到测试用例生成、审查、迭代优化的完整流程。

### 1.1 核心特性

- **Lead分析**：读取需求文档，提取功能点，构建功能清单
- **Generator编写**：并行编写测试用例，最大化效率
- **Reviewer审查**：检查覆盖率、正确性、冗余性
- **迭代优化**：发现问题自动进入修复迭代，最多3轮

### 1.2 技术架构

系统采用多Agent协作架构，每个Agent有明确的角色分工：

- **Lead（主Agent）**：负责任务分配、进度协调、质量把控
- **Generator（生成Agent）**：负责编写测试用例
- **Reviewer（审查Agent）**：负责质量审查和反馈

---

## 二、核心工作流

### 2.1 Review循环机制

系统采用Review循环机制，确保测试用例质量：

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

### 2.2 完整执行流程

#### Phase 1: Lead分析需求

- 识别需求文档路径（用户输入或新建）
- 文档格式检测与转换（.docx → .md）
- 读取转换后的需求文档
- 分析需求，识别功能点
- 构建功能点清单（features.md）
- 将功能点分组分配给Generator

#### Phase 2: Generator并行编写

- Lead启动两个Generator Agent并行执行
- 每个Generator分配独立的功能点子集
- 每个Generator输出测试用例

#### Phase 3: Reviewer审查

- Lead启动Reviewer Agent
- 审查覆盖率、正确性、冗余性
- 输出结构化审查报告

#### Phase 4: 迭代循环

- Review发现Critical问题或覆盖率不达标
- Lead分析审查报告，分配修复任务
- Generator修复/补充用例
- 版本号+1 (v1 → v2 → v3)
- Reviewer再次审查

#### Phase 5: 最终交付

- 汇总所有版本（去重）
- 输出test_cases_final.md
- 生成迭代日志
- 生成Excel格式测试用例

---

## 三、质量标准

质量标准统一在rules文件中定义，确保全局一致性：

| 维度 | 目标 | 低于目标处理 |
|------|------|--------------|
| 功能覆盖 | 100% | Critical |
| 边界覆盖 | ≥20% | Warning |
| 异常覆盖 | ≥10% | Warning |
| P0占比 | ≥25% | Warning |

**说明**：
- Critical：必须修复，否则无法通过
- Warning：建议修复，影响质量评分

---

## 四、文件结构

项目采用标准化的目录结构：

```
{workdir}/
├── skills/ccg/test-generation/     # Agent技能定义
│   ├── SKILL.md                  # 主入口
│   ├── roles/                    # Agent角色定义
│   │   ├── lead.md              # Lead角色
│   │   ├── generator.md         # Generator角色
│   │   └── reviewer.md          # Reviewer角色
│   ├── workflow/                 # 执行流程
│   │   ├── lead-execute.md
│   │   ├── generator-execute.md
│   │   └── reviewer-execute.md
│   └── templates/                # 模板文件
│
├── rules/                        # 工作流规则
│   └── test-generation-workflow.md
│
├── tools/                        # 工具目录
│   └── convert_to_excel.py       # Markdown转Excel
│
└── output/                        # 输出目录
    └── {需求名称}_{YYYY-MM-DD_HHmmss}/
        ├── features.md
        ├── test_cases_v1.md
        ├── test_cases_v2.md
        ├── test_cases_v3.md
        ├── test_cases_final.md
        ├── review_report.md
        ├── review_report_final.md
        ├── iteration_log.md
        └── {需求名称}.xlsx
```

---

## 五、实战案例

### 5.1 项目背景

- **需求**：阿虎医考App v9.3.6 - POP付费资料
- **目标**：为该需求生成完整的测试用例集

### 5.2 执行过程

#### Step 1: 需求分析（Lead）

Lead读取原始需求文档（.docx格式），提取16个功能点：

- F1-F6: 教务后台相关（POP资料库、POP资料商品）
- F7-F11: 平台相关（三方配置、三方订单）
- F12-F16: App相关（首页入口、资料列表、PDF浏览、POP弹窗）

#### Step 2: 并行生成（Generator）

两个Generator并行工作：

- **Generator-1**：F1-F6（教务后台），生成20个用例
- **Generator-2**：F7-F16（App+平台），生成34个用例

#### Step 3: 审查（Reviewer）

Reviewer审查v1版本，发现问题：

- 边界覆盖率不足（9.3% < 20%）
- F12无异常测试
- TC-003/TC-010分类错误

#### Step 4: 迭代修复

- 进入v2迭代，补充边界用例，修复问题
- 进入v3迭代，修正分类错误，补充F16用例

### 5.3 最终结果

| 版本 | 用例数 | 边界覆盖 | P0占比 |
|------|--------|----------|--------|
| v1 | 54 | 9.3% | 33.3% |
| v2 | 63 | 20.6% | 28.6% |
| v3 | 65 | 21.5% | 28.6% |
| **目标** | - | ≥20% | ≥25% |

---

## 六、关键技术

### 6.1 去重机制

合并v1/v2/v3版本时，系统自动去重：

- 每个TC编号只保留最新版本
- 例如：TC-001在v1/v2/v3中都存在，只保留v3的版本
- 原因：每次迭代可能修改同一用例，后面版本更完善

### 6.2 迭代管理

迭代版本规则：

- **v1**：初始生成
- **v2**：第1次修复迭代
- **v3**：第2次修复迭代
- **final**：最终交付（已去重）

### 6.3 Excel生成

使用convert_to_excel.py工具，将Markdown转换为Excel格式：

- 输入：test_cases_final.md
- 输出：{需求名称}.xlsx
- 格式：ID、标题、目录、前置条件、步骤描述、预期结果、负责人、优先级

---

## 七、复用指南

### 7.1 复用步骤

1. 复制skills/ccg/test-generation/到目标skills目录
2. 复制rules/test-generation-workflow.md到目标rules目录
3. 复制tools/convert_to_excel.py到目标tools目录
4. 安装依赖：pip install python-docx openpyxl

### 7.2 触发使用

在AI对话中输入以下触发词：

- 生成测试用例
- test generation
- 测试用例生成
- 编写测试

---

## 八、总结

本项目展示了多Agent协作的强大能力：

- **分工明确**：Lead、Generator、Reviewer各司其职
- **并行高效**：两个Generator同时工作
- **质量保障**：Review循环确保用例质量
- **迭代优化**：自动修复覆盖率等问题
- **易于复用**：标准化结构便于迁移

通过蚁群仿生架构，系统模拟了真实的协作流程，显著提升了测试用例生成的效率和质量。

---

> 文档生成时间: 2026-05-15