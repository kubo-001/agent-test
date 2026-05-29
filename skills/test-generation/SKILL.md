---
name: test-generation
description: 多Agent测试用例生成系统。触发词：生成测试用例、test generation、测试用例生成、编写测试。Lead分析需求→Generator编写→Reviewer审查→修复迭代→最终交付。
trigger: 生成测试用例, test generation, 测试用例生成, 编写测试, test agent, multi-agent test
user-invocable: true
---

# 多Agent测试用例生成系统

> 蚁群仿生架构 + Review循环：Lead分析 → **PM确认（如需）** → Generator编写 → Reviewer审查 → 修复迭代 → 最终交付

---

## 核心流程：Review循环

```
需求文档 → Lead分析
              ↓
     【强制】发现至少5个待确认项
              ↓
        PM沟通确认
              ↓
           确认完成 → 启动Generator并行
                          ↓
                      Reviewer审查
                          ↓
                    [修复迭代] → 最终交付
```

**关键特性**：
- **【强制】Lead分析后必须发现至少5个待确认项**
- Lead发现待确认项 → PM介入沟通 → 用户确认后继续
- 审查发现问题 → 自动进入修复迭代
- 最多3轮迭代，第2轮达标可提前结束
- 无Critical问题且覆盖率达标 → 最终交付

---

## 执行流程

### Phase 1: Lead分析需求

**执行者**: 当前Agent (Lead)

**步骤**:
1. 识别需求文档路径（用户输入或新建）
2. **文档格式检测与转换**:
   - `.docx` / `.doc` → 用python-docx转换为`.md`
   - `.md` → 直接使用，无需转换
3. 读取转换后的需求文档
4. 分析需求，识别功能点
5. **【强制】发现并列出至少5个待确认项**
6. 调用PM与用户沟通确认
7. 将功能点分组分配给Generator

**待确认项类别（至少选5类）**：
- 需求模糊 - 功能描述不清晰、术语未定义
- 边界条件 - 最大/最小数量限制、超时时间
- 技术风险 - 实现方案不确定性、依赖外部系统
- 影响范围 - 对现有功能的影响、兼容性
- 优先级 - 功能点优先级排序
- 用户体验 - 交互细节、UI表现
- 数据边界 - 数据量预估、性能指标
- 兼容性 - 多端兼容、版本兼容

**输出**: `features.md`, `待确认项清单`

---

### Phase 2: Generator并行编写

**执行者**: Agent (subagent parallel)

**步骤**:
1. Lead启动两个Generator Agent并行执行
2. 每个Generator分配独立的功能点子集
3. 每个Generator输出测试用例

**输出**: `test_cases_v1.md`

---

### Phase 3: Reviewer审查

**执行者**: Agent (subagent)

**步骤**:
1. Lead启动Reviewer Agent
2. 审查覆盖率、正确性、冗余性
3. 输出结构化审查报告

**输出**: `review_report.md`

---

### Phase 4: 迭代循环 (关键！)

**条件**: Review发现Critical问题 或 覆盖率不达标

**循环**:
```
while (还有问题 && 循环次数 < 3):
    Lead分析审查报告 → 分配修复任务 → Generator修复 → 再次Review
```

**每次迭代**:
- 修复/补充测试用例
- 版本号+1 (v1 → v2 → v3)
- 记录迭代日志

---

### Phase 5: 最终交付

**条件**: 无Critical问题 + 覆盖率达标

**输出**:
- `test_cases_final.md` - 最终测试用例
- `{需求名称}.xlsx` - Excel格式测试用例
- `review_report_final.md` - 最终审查报告
- `iteration_log.md` - 迭代日志

---

## 质量标准

> 质量标准统一在 `rules/test-generation-workflow.md` → 配置标准 中定义

---

## 版本管理

```
v1: 初始生成
v2: 第1次修复迭代
v3: 第2次修复迭代
final: 最终交付（已去重）
```

**去重机制**: 合并v1/v2/v3时，每个TC编号只保留最新版本

---

## 输出文件结构

```
{workdir}/output/{需求名称}_{YYYY-MM-DD_HHmmss}/
├── features.md              # 功能点清单
├── test_cases_v1.md         # 第1轮测试用例
├── test_cases_v2.md         # 第2轮测试用例
├── test_cases_v3.md         # 第3轮测试用例
├── test_cases_final.md      # 最终测试用例（已去重）
├── review_report.md         # 审查报告
├── review_report_final.md   # 最终审查报告
├── iteration_log.md         # 迭代日志
└── {需求名称}.xlsx          # Excel格式测试用例（与MD同目录）
```

---

## 关键文件

| 文件 | 路径 |
|------|------|
| Lead执行逻辑 | `workflow/lead-execute.md` |
| PM执行逻辑 | `workflow/pm-execute.md` |
| Generator执行逻辑 | `workflow/generator-execute.md` |
| Generator修复逻辑 | `workflow/generator-fix-execute.md` |
| Reviewer执行逻辑 | `workflow/reviewer-execute.md` |
| 转换工具 | `tools/convert_to_excel.py` |

---

## 环境依赖

```bash
pip install python-docx openpyxl
```

---

## 示例

```
用户: 为订单模块生成测试用例

AI自动执行:
1. Lead读取需求文档（docx自动转md）
2. 分析提取功能点
3. 分配: Generator-1处理[F1,F2,F3], Generator-2处理[F4,F5,F6]
4. 并行生成测试用例 → v1
5. Reviewer审查
   └─ 发现边界覆盖不足 (Critical)
6. 修复迭代 → v2
7. 再次Review
   └─ 通过
8. 最终交付 → test_cases_final.md + {需求名称}.xlsx
```