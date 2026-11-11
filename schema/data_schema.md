# 数据 Schema · 有限元错误分析闭环数据集

> 版本 v1.0 · 定义"错误—诊断—修复—验证—反馈"闭环数据的完整字段结构。
> 全量数据按此 schema 组织；样例数据包为该 schema 的最小可用实例。

## 目录结构（全量数据）

```
cases/
└── {case_id}/                     # 案例唯一标识，如 drone_thermal_error_001
    ├── error/
    │   ├── model.inp              # 报错模型（原始提交版）
    │   └── logs.txt               # 报错日志（.msg/.dat/.sta 摘录合并）
    ├── diagnosis/
    │   ├── findings.json          # 判案记录（见 findings 字段）
    │   └── report.md              # 诊断报告（机理级）
    ├── repair/
    │   ├── fixed.inp              # 修复后模型
    │   └── actions.json           # 修复动作记录（见 actions 字段）
    ├── verify/
    │   └── result.json            # 重新求解验证结果（见 verify 字段）
    └── meta.json                  # 案例元信息 + 质量标签 + 反馈
```

## 字段定义

### meta.json —— 案例元信息与质量标签

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| case_id | string | 案例唯一标识 |
| title | string | 案例名称（如"四旋翼无人机热-力耦合失效"） |
| error_type | string | 求解终止类型（TOO MANY ATTEMPTS / ZERO PIVOT / OVERCONSTRAINT …） |
| solver | string | 求解器（ABAQUS 等） |
| element_types | string[] | 单元类型（C3D8T / C3D8R …） |
| ndof | int | 模型自由度 |
| created_at | string | 入库时间 |
| source | enum | `synthetic`（缺陷注入合成）/ `real`（用户真实提交）/ `internal`（内部验证） |
| quality | object | 质量标签（见下） |
| feedback | object | 用户反馈（见下） |

**quality（质量标签）**：

```json
{
  "verified": true,              // 是否经重新求解验证
  "verify_status": "COMPLETED",  // COMPLETED / FAILED
  "consistency": "passed",       // 一致性校验：passed / error
  "benchmark": "passed",         // 基准回归：passed / failed
  "checker_version": "v2.1"
}
```

**feedback（用户反馈）**：

```json
{
  "rating": "up",                // up / down
  "comment": "温度奇异定位准确，修复文件跑通了"
}
```

### findings.json —— 诊断记录（机理标签）

```json
{
  "findings": [
    {
      "type": "temperature_unconstrained",   // 缺陷类型
      "severity": "critical",                // critical / warning
      "location": {"node": 10, "dof": 11},   // 位置（节点/自由度）
      "mechanism": {
        "physical": "温度场存在刚体平移零能模态，与结构刚体模态同构",
        "program": "C3D8T 单元激活 DOF 11，但分析步内无温度边界条件"
      }
    }
  ]
}
```

**缺陷类型枚举（12+ 类）**：
`rigid_translation` / `rigid_rotation` / `isolated_nodes` / `contact_gap` /
`contact_master_slave_suspect` / `tie_overconstraint` / `undefined_surface` /
`temperature_unconstrained` / `unit_inconsistency` / `boundary_invalid` / …

### actions.json —— 修复动作记录（Agent 轨迹）

```json
{
  "actions": [
    {
      "type": "add_boundary_temperature",   // 白名单干预动作
      "scope": "coupled_step",              // 作用范围
      "target": "Motors",                   // 作用对象（节点集/集合）
      "detail": "耦合分析步内为 Motors 补充 DOF 11 = 100.0 温度边界，后续步重置机械边界"
    }
  ]
}
```

### verify/result.json —— 求解验证记录（闭环标签）

```json
{
  "fixed_inp": "fixed.inp",
  "solver": "ABAQUS",
  "status": "COMPLETED",          // COMPLETED / FAILED
  "steps_completed": 5,
  "steps_total": 5,
  "notes": "修复后重新提交求解，5 个分析步全部完成"
}
```

## 数据分层（合规）

| 层级 | 来源 | 授权 | 可否对外 |
| --- | --- | --- | --- |
| L1 合成数据 | 缺陷注入器批量生成 | 完全自有 | ✅ 可售卖/开放 |
| L2 用户数据 | 用户诊断脱敏后 | 用户授权（服务条款） | ⚠️ 脱敏+授权后可用 |
| L3 涉敏数据 | 军工/涉密客户 | 数据不出域 | 🔒 仅本地使用 |

## 质量保障流程

```
采集 ──▶ 判案标注 ──▶ 修复 ──▶ 求解验证 ──▶ 一致性校验 ──▶ 基准回归 ──▶ 入库
                ▲                                          │
                └────────── 用户反馈（RLHF 偏好） ──────────┘
```
