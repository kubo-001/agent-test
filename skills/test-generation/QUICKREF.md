# 快速参考

## 触发命令

```
生成测试用例 / test generation / 测试用例生成
```

## 流程

```
Lead → 分析需求 → features.md
   ↓
【强制】发现至少5个待确认项
   ↓
PM → 沟通确认（如需）
   ↓
Generator × N → 并行生成测试用例
   ↓
Reviewer → 审查 → review_report.md
   ↓
Lead → 汇总 → summary.md
```

**【强制规则】Lead分析后必须发现至少5个待确认项，必须PM确认后才能进入Generator。**

## 角色职责

| 角色 | 输入 | 输出 |
|------|------|------|
| Lead | 需求文档 | features.md, 任务分配 |
| PM | 待确认事项 | 用户确认结果 |
| Generator | 功能点列表 | test_cases_{N}.md |
| Reviewer | 测试用例集 | review_report.md |

## 输出文件

```
{workdir}/output/{需求名}_{timestamp}/
├── features.md              - 功能点清单
├── test_cases_v1.md         - 第1轮测试用例
├── test_cases_v2.md         - 第2轮测试用例
├── test_cases_v3.md         - 第3轮测试用例
├── test_cases_final.md      - 最终测试用例（已去重）
├── review_report.md         - 审查报告
├── review_report_final.md    - 最终审查报告
├── iteration_log.md          - 迭代日志
└── {需求名称}.xlsx          - Excel格式测试用例（与MD同目录）
```

## 质量标准

> 详见 `rules/test-generation-workflow.md` → 配置标准

## 测试用例格式

```markdown
### TC-XXX: {测试名称}
- **前置条件**: ...
- **测试步骤**: 1. ... 2. ...
- **预期结果**: ...
- **优先级**: P0/P1/P2
- **测试类型**: 功能/边界/异常
```

## 关键路径

```
skills/test-generation/
├── SKILL.md           # 主入口
├── roles/
│   ├── lead.md
│   ├── pm.md          # 产品经理角色（新增）
│   ├── generator.md
│   └── reviewer.md
├── workflow/
│   ├── lead-execute.md
│   ├── pm-execute.md  # PM执行逻辑（新增）
│   ├── generator-execute.md
│   ├── generator-fix-execute.md
│   └── reviewer-execute.md
└── examples/
    └── usage.md

rules/
└── test-generation-workflow.md
```

## 复用检查清单

- [ ] 复制 skills/test-generation/ 到目标skills目录
- [ ] 复制 rules/test-generation-workflow.md 到目标rules目录
- [ ] 复制 tools/convert_to_excel.py 到 tools/目录
- [ ] 安装依赖: pip install python-docx openpyxl
- [ ] 配置输出目录路径
- [ ] 验证触发词生效