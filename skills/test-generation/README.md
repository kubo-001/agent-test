# 多Agent测试用例生成系统

> 基于蚁群仿生架构的智能测试用例生成工具 - 可被其他AI系统复用

---

## 目录结构

```
test-generation/
├── SKILL.md                      # 主入口（用户输入触发词即可激活）
├── README.md                     # 本文件
├── QUICKREF.md                   # 快速参考
├── roles/                        # Agent角色Prompt定义
│   ├── lead.md                   # Lead角色 - 需求分析、任务分配、汇总
│   ├── generator.md              # Generator角色 - 测试用例编写
│   └── reviewer.md               # Reviewer角色 - 审查、质量把控
├── workflow/                     # 可执行workflow
│   ├── lead-execute.md           # Lead执行逻辑
│   ├── generator-execute.md       # Generator执行逻辑
│   ├── generator-fix-execute.md  # Generator修复逻辑（迭代时使用）
│   └── reviewer-execute.md        # Reviewer执行逻辑
├── templates/                    # 输出模板
│   ├── test-case.md              # 测试用例模板
│   └── review-report.md          # 审查报告模板
└── examples/                     # 使用示例
    └── usage.md                  # 完整示例
```

---

## 核心流程：Review迭代循环

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

**关键特性**：
- 审查发现问题 → 自动进入修复迭代
- 最多3轮迭代
- 无Critical问题且覆盖率达标 → 最终交付

---

## 触发方式

用户输入以下关键词即可触发：

| 触发词 | 说明 |
|--------|------|
| `生成测试用例` | 中文触发 |
| `test generation` | 英文触发 |
| `测试用例生成` | 同上 |
| `编写测试` | 手工触发 |
| `test agent` | 多Agent场景 |

---

## 输出目录结构

**核心原则**：每次生成测试用例时，创建独立的项目文件夹，不覆盖历史数据。

```
{workdir}/output/
└── {需求名称}_{YYYY-MM-DD_HHmmss}/     # 每次生成创建新文件夹
    ├── features.md              # 功能点清单
    ├── test_cases_v1.md         # 第1轮测试用例
    ├── test_cases_v2.md         # 第2轮测试用例
    ├── test_cases_v3.md         # 第3轮测试用例
    ├── test_cases_final.md      # 最终测试用例（已去重）
    ├── review_report.md          # 审查报告（最新版）
    ├── review_report_final.md    # 最终审查报告
    ├── iteration_log.md         # 迭代日志
    └── {需求名称}.xlsx           # Excel格式测试用例（必须生成）
```

**去重机制**：合并v1/v2/v3版本时，每个TC编号只保留最新版本（编号最大的）。

---

## 质量标准

| 指标 | 目标 | 低于目标处理 |
|------|------|--------------|
| 功能覆盖 | 100% | Critical |
| 边界覆盖 | ≥20% | Warning |
| 异常覆盖 | ≥10% | Warning |
| P0占比 | ≥25% | Warning |

---

## 测试用例格式

### Markdown格式 (test_cases_final.md)

```markdown
## F8: 首页付费专属资料

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

---

## 复用方式

### 方法一：完整复制

```
复制整个 test-generation/ 目录到目标系统的 skills/ 目录
复制 test-generation-workflow.md 到目标系统的 rules/ 目录
复制 tools/convert_to_excel.py 到 tools/ 目录
```

### 方法二：按需引用

单独复制需要的组件：
- 只需要Lead → 复制 `workflow/lead-execute.md`
- 只需要Generator → 复制 `workflow/generator-execute.md`
- 只需要Workflow规则 → 复制 `rules/test-generation-workflow.md`

---

## 环境配置

### 依赖安装

```bash
pip install python-docx openpyxl
```

### 路径配置

convert_to_excel.py 支持相对路径和绝对路径：
- 相对路径：`python convert_to_excel.py ./test_cases_final.md`
- 绝对路径：`python convert_to_excel.py /path/to/test_cases_final.md`

输出目录默认与输入文件同一目录，也可通过第二个参数指定。

---

## 自定义扩展

### 修改执行逻辑
编辑 `workflow/` 目录下的文件。

### 修改模板
编辑 `templates/` 目录下的文件。

### 修改质量标准
编辑 `workflow/reviewer-execute.md` 或 `rules/test-generation-workflow.md` 中的覆盖率目标。

---

## 依赖

- Python 3.10+
- openpyxl (用于生成Excel)

```bash
pip install openpyxl
```

---

## 版本历史

| 版本 | 说明 |
|------|------|
| v1 | 初始生成 |
| v2 | 第1次修复迭代（覆盖不足时触发） |
| v3 | 第2次修复迭代（仍有问题时触发） |
| final | 最终交付（已去重，可能118条用例） |

---

## License

MIT