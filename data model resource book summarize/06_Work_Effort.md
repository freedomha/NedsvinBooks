# 第6章：Work Effort（工作成果/项目工作）

## 章节概述

本章深入探讨企业中用于跟踪和管理工作成果（Work Effort）的数据模型。从工作的需求（Work Requirement）开始，经过承诺（Work Order Item），到实际工作的执行和跟踪（Work Effort），以及资源的分配（人员、库存、固定资产），最后到工作成果的记录。本章涵盖了一个完整的工作生命周期管理模型，涉及需求管理、项目分解、资源调度、时间跟踪、费率管理、成本核算以及工作标准等核心领域。

该模型适用于制造业的生产运行、内部项目的管理、维护维修任务、专业服务交付等各种业务场景。

---

## 核心问题（Problems）

企业在管理工作成果时需要回答以下关键问题：

1. **工作需求来源多样**：工作需求可能来自客户订单、内部预测需求、设备维护、内部项目等多种渠道，如何统一建模？
2. **需求与承诺的关系**：Work Requirement（需求）和 Order（订单/承诺）有何区别？Requisition（申请单）属于哪一类？
3. **工作分解结构（WBS）**：如何将大型工作（Program、Project）逐层分解为 Phase、Activity、Task，并管理它们之间的依赖关系？
4. **资源分配**：如何将人员（Party）、库存物料（Inventory）和固定资产（Fixed Asset）分配到各级工作，并跟踪其使用情况？
5. **时间跟踪与计费**：如何记录工作人员的时间条目（Time Entry），并与工作关联，用于薪资和客户计费？
6. **费率管理**：工作费率可能因人、职位或具体工作分配而异，且随时间变化，如何灵活建模？
7. **工作标准与资源预测**：如何建立不同类型工作的标准（所需技能、物料、设备），用于规划和资源预测？
8. **工作成果记录**：如何记录工作完成后实际产出的交付物（Deliverable）、生产出的库存（Inventory Item Produced）或修复的资产？

---

## 解决方案（Solutions）

### 1. 工作需求（Work Requirement）

Work Requirement 代表对完成某项工作的**需求**，而非承诺。它与 Order Item 的区别在于：Requirement 表示需求，Order 表示承诺。

**WORK REQUIREMENT 关键属性**：
- `description` — 需求描述
- `requirement creation date` — 创建日期
- `required by date` — 需求截止日期
- `estimated budget` — 估计预算
- `quantity` — 所需数量
- `reason` — 需求原因

**REQUIREMENT TYPE 子类型**：
| 类型 | 关联实体 | 典型场景 |
|------|----------|----------|
| Production Run | PRODUCT + quantity | 根据预测需求量产特定产品 |
| Maintenance / Repair | FIXED ASSET | 维修雕刻机等设备 |
| Internal Project | DELIVERABLE | 制定销售与市场计划 |

模型使用 **Exclusive Arc**（互斥弧）约束：一个 Work Requirement 只能关联三者之一（PRODUCT、DELIVERABLE 或 FIXED ASSET）。

**预期需求（Anticipated Demand）**：不仅客户订单会触发生产需求，企业的预测/趋势分析也可以生成 Work Requirement，以便提前备货。预期需求仅适用于库存产品，不适用于服务（服务无法预制）。

### 2. 工作需求的角色（Work Requirement Roles）

与订单角色类似，Work Requirement 也需要跟踪相关参与方角色。

核心实体：
- **WORK REQUIREMENT ROLE TYPE** — 定义有效角色类型
- **PARTY WORK REQUIREMENT ROLE** — 关联 PARTY、WORK REQUIREMENT 和 WORK REQUIREMENT ROLE TYPE

常见角色：`Created for`（为谁创建）、`Created by`（创建者）、`Responsible for`（负责人）、`Authorized by`（审批人）。

模型支持同一Party对同一Work Requirement在不同时间段拥有相同角色（通过 `from date` 纳入主键），也支持角色变更历史。

### 3. 工作成果的生成（Work Effort Generation）

完整的工作流转路径：

```
REQUIREMENT ──(ORDER REQUIREMENT COMMITMENT)──→ WORK ORDER ITEM ──(WORK ORDER ITEM FULFILLMENT)──→ WORK EFFORT
```

- **WORK REQUIREMENT** → 需要做某事
- **WORK ORDER ITEM** → 承诺完成某项工作（Order Item 的子类型）
- **WORK EFFORT** → 跟踪实际工作的执行

三者之间均为多对多关系，提供高度灵活性：
- 多个需求可以合并为一个工作成果
- 一个需求可以拆分为多个工作成果
- 需求可以直接通过 WORK REQUIREMENT FULFILLMENT 关联到 WORK EFFORT

**WORK EFFORT 来源场景**：
1. 内部工作需求
2. 客户订购需要制造的产品
3. 已售出服务的履行
4. 客户维修/服务请求

### 4. 工作成果类型与用途类型（Work Effort Type & Purpose Type）

**WORK EFFORT TYPE 子类型（按层级）：**
- **PROGRAM** — 项目群
- **PROJECT** — 项目
- **PHASE** — 阶段
- **ACTIVITY** — 活动
- **TASK** — 任务

每个 Work Effort Type 可记录 `standard work hours`（标准工时估算）。

**WORK EFFORT PURPOSE TYPE 子类型：**
- **MAINTENANCE** — 维护保养
- **PRODUCTION RUN** — 生产运行（含 quantity to produce 和 quantity produced）
- **WORK FLOW** — 工作流分配
- **RESEARCH** — 研究调查

**Work Effort 属性**：name、description、scheduled start date、scheduled completion date、estimated hours、actual start datetime、actual completion datetime、actual hours、special terms、total dollars allowed、total hours allowed。

每个 Work Effort 可以关联一个 **FACILITY**（物理设施）。对于没有记录的次级工作，默认使用父级 Work Effort 的 Facility。

### 5. 工作成果分解与依赖（Work Effort Associations & Dependencies）

**WORK EFFORT ASSOCIATION** — 多对多递归关联，实现灵活的工作分解：
- 一个 Program 包含多个 Project
- 一个 Project 包含多个 Activity
- 一个 Task 可以参与多个父级 Work Effort（如"设置生产线"同时参与两个生产运行）

**WORK EFFORT DEPENDENCY 子类型：**
- **WORK EFFORT PRECEDENCY** — 前置依赖（B 必须在 A 完成后才能开始）
- **WORK EFFORT CONCURRENCY** — 并行依赖（两个任务需同时进行）

### 6. 工作成果的人员分配（Work Effort Party Assignment）

核心实体：
- **WORK EFFORT PARTY ASSIGNMENT** — 将 Party 分配到 Work Effort（可以是个人或组织）
- **WORK EFFORT ROLE TYPE** — 角色类型（如 project manager、team member、contractor）
- **PARTY SKILL / SKILL TYPE** — 人员和组织的技能信息

**PARTY SKILL 属性**：skill type、years of experience、rating。技能不仅关联到个人（Person），也关联到组织（Organization），用于评估外包供应商的能力。

**WORK EFFORT STATUS** — 跟踪工作状态（如 in progress、started、completed、pending）。每个 Work Effort 可有多个历史状态记录。

**分配的可选 Facility**：随着远程办公的增加，个人分配级可关联不同的 FACILITY。

### 7. 时间跟踪（Work Effort Time Tracking）

时间跟踪模型层次：

```
TIMESHEET（考勤表）
  ├── WORKER（Party Role，如 EMPLOYEE 或 CONTRACTOR）
  ├── TIMESHEET ROLE（需要由 TIMESHEET ROLE TYPE 定义，如 approver、manager）  
  └── TIME ENTRY（时间条目）
        ├── from datetime
        ├── thru datetime
        ├── hours
        └── WORK EFFORT（归集到哪个工作）
```

TIME ENTRY 不直接关联 WORK EFFORT PARTY ASSIGNMENT，而是通过 TIMESHEET 确定 WORKER（Party），再关联到 WORK EFFORT。这样避免了冗余存储 Party 信息。

如需其他计量单位（如天数），可将 `hours` 改为 `quantity` 并关联 UNIT OF MEASURE。

### 8. 工作成果费率（Work Effort Rates）

费率可能来自三个维度：

| 维度 | 实体 | 说明 |
|------|------|------|
| 个人/组织 | PARTY RATE | John Smith 的费率是 $180/小时 |
| 职位 | POSITION RATE | 高级顾问的标准费率 $250/小时 |
| 工作分配 | WORK EFFORT ASSIGNMENT RATE | 特定工作分配的费率 |

每个费率实体都有 `from date` / `thru date` 支持历史费率变更。

**RATE TYPE** 分类：billing rate（计费费率）、cost（成本费率）、overtime rate（加班费率）、regular pay（正常薪资）等。

业务规则需要确定：多费率共存时的优先级、加班判定规则。

### 9. 库存分配（Inventory Assignments）

**WORK EFFORT INVENTORY ASSIGNMENT** — 记录工作执行过程中消耗的库存物料：

| WORK EFFORT | INVENTORY ITEM | QUANTITY |
|-------------|---------------|----------|
| 组装铅笔零件 | Pencil cartridges | 100 |
| 组装铅笔零件 | Erasers | 100 |
| 组装铅笔零件 | Labels | 100 |

使用时需要通过业务流程同步更新库存。

### 10. 固定资产分配（Fixed Asset Assignments）

**FIXED ASSET 子类型**：PROPERTY（房产）、VEHICLE（车辆）、EQUIPMENT（设备）、OTHER FIXED ASSET。

**FIXED ASSET 属性**：date acquired（折旧计算）、date last serviced、date next service、production capacity + UOM。

**FIXED ASSET TYPE** — 支持递归分类：
```
Equipment → Pencil-making machine
Vehicle → Truck → Mac truck-18 wheels
```

**WORK EFFORT FIXED ASSET ASSIGNMENT** — 记录资产在工作中的使用：
- from date / thru date — 分配时间段
- allocated cost — 分配成本
- WORK EFF ASSET ASSIGN STATUS TYPE — 分配状态（requested / assigned）

**PARTY FIXED ASSET ASSIGNMENT** — 资产借出给个人（如笔记本电脑、工具套件）。

### 11. 工作成果类型标准（Work Effort Type Standards）

用于资源规划的标准实体：

| 实体 | 关键属性 | 用途 |
|------|----------|------|
| WORK EFFORT SKILL STANDARD | estimated num people, estimated duration, estimated cost | 预测所需技能和人力 |
| WORK EFFORT GOOD STANDARD | estimated quantity, estimated cost | 预测所需物料 |
| WORK EFFORT FIXED ASSET STANDARD | estimated quantity, estimated duration, estimated cost | 预测所需设备 |

此外还有：
- **WORK EFFORT TYPE BREAKDOWN** — 标准的工作类型分解结构
- **WORK EFFORT TYPE DEPENDENCY** — 标准的依赖关系

每个 Work Effort Type 可以关联其修复的 FIXED ASSET TYPE 或生产的 DELIVERABLE/PRODUCT。

### 12. 工作成果结果（Work Effort Results）

记录实际产生的结果：

- **WORK EFFORT DELIVERABLE PRODUCED** — 产出的交付物
- **WORK EFFORT INVENTORY PRODUCED** — 生产出的库存物品
- **FIXED ASSET 修复** — 维修工作的结果

结果建议在较低层级（如 Task/Activity）记录，以便汇总到整体 Project。

---

## 关键数据模型/概念

| 概念 | 英文实体名 | 中文说明 |
|------|-----------|----------|
| 工作需求 | WORK REQUIREMENT | 执行某项工作的需求 |
| 需求类型 | REQUIREMENT TYPE | Production Run / Maintenance / Internal Project |
| 工作订单项 | WORK ORDER ITEM | 承诺执行工作（Order Item 子类型） |
| 工作成果 | WORK EFFORT | 实际工作的执行和跟踪 |
| 工作成果类型 | WORK EFFORT TYPE | Program / Project / Phase / Activity / Task |
| 工作成果用途 | WORK EFFORT PURPOSE TYPE | Maintenance / Production Run / Work Flow / Research |
| 工作成果关联 | WORK EFFORT ASSOCIATION | 多对多递归工作分解 |
| 工作成果依赖 | WORK EFFORT DEPENDENCY | Precedency（前置）/ Concurrency（并行） |
| 工作成果人员分配 | WORK EFFORT PARTY ASSIGNMENT | 分配 Party 到工作 |
| 工作成果角色类型 | WORK EFFORT ROLE TYPE | 分配的角色 |
| 工作成果状态 | WORK EFFORT STATUS | 跟踪工作状态变化 |
| 技能 | PARTY SKILL / SKILL TYPE | 人员和组织的技能及评级 |
| 时间条目 | TIME ENTRY | 记录工作时间 |
| 考勤表 | TIMESHEET | 时间条目集合 |
| 工作成果分配费率 | WORK EFFORT ASSIGNMENT RATE | 工作分配的具体费率 |
| 费率类型 | RATE TYPE | billing rate / cost / overtime |
| 库存分配 | WORK EFFORT INVENTORY ASSIGNMENT | 工作消耗的库存 |
| 固定资产 | FIXED ASSET | 设备、车辆、房产等 |
| 固定资产类型 | FIXED ASSET TYPE | 递归分类 |
| 固定资产分配 | WORK EFFORT FIXED ASSET ASSIGNMENT | 工作中使用的资产 |
| 工作成果技能标准 | WORK EFFORT SKILL STANDARD | 标准技能需求 |
| 工作成果物料标准 | WORK EFFORT GOOD STANDARD | 标准物料需求 |
| 工作成果固定资产标准 | WORK EFFORT FIXED ASSET STANDARD | 标准设备需求 |
| 工作成果交付物 | WORK EFFORT DELIVERABLE PRODUCED | 工作产出的交付物 |
| 工作成果库存产出 | WORK EFFORT INVENTORY PRODUCED | 工作产出的库存 |
| 设施 | FACILITY | 工作发生的地点 |

### 整体模型架构图

整体模型（Figure 6.13）覆盖了以下完整流程：

```
Work Requirements（需求）
    ↓
Work Order Items（承诺）
    ↓
Work Efforts（执行）
    ├── Work Effort Breakdown（分解/递归）
    │   ├── Work Effort Association（关联）
    │   └── Work Effort Dependency（依赖）
    ├── Resource Assignments（资源分配）
    │   ├── Party Assignment（人员）
    │   ├── Inventory Assignment（库存）
    │   └── Fixed Asset Assignment（固定资产）
    ├── Time Tracking（时间跟踪）
    │   └── Time Entry / Timesheet
    ├── Rates（费率）
    │   ├── Party Rate
    │   ├── Position Rate
    │   └── Work Effort Assignment Rate
    ├── Standards（标准）
    │   ├── Skill Standard
    │   ├── Good Standard
    │   └── Fixed Asset Standard
    └── Results（成果）
        ├── Deliverable Produced
        ├── Inventory Produced
        └── Fixed Asset Repaired
```

---

## 结论/要点

1. **Requirement 与 Order 的区分至关重要**：Requirement 代表需求/需要，Order 代表承诺/契约。Work Requirement 通过 RE UIIREMENT TYPE 子类型化来支持多种业务场景（生产运行、维护、内部项目）。

2. **多层级工作分解灵活建模**：通过 WORK EFFORT 的递归关联（WORK EFFORT ASSOCIATION），可以动态构建任意深度的工作分解结构（WBS），避免了硬编码层级限制。

3. **统一资源分配模型**：人员（Party）、物料（Inventory Item）、设备（Fixed Asset）三种资源通过各自的 Assignment 实体统一关联到 WORK EFFORT，模型一致且可扩展。

4. **时间跟踪与费率分离**：TIME ENTRY 记录实际工时，WORK EFFORT ASSIGNMENT RATE 独立管理费率，支持计费、成本、加班等多种费率类型，且通过 from date/thru date 支持历史变更。

5. **标准驱动规划**：WORK EFFORT TYPE STANDARDS 系列实体允许企业为不同类型的工作预定义标准资源需求（技能、物料、设备），为项目规划和资源预测提供数据基础。

6. **结果追踪闭环**：工作完成后通过 WORK EFFORT DELIVERABLE PRODUCED、WORK EFFORT INVENTORY PRODUCED、FIXED ASSET Repair 记录实际产出，与标准形成对比，完成 Plan-Do-Check 的管理闭环。

7. **Party 而非 Person**：资源分配面向 Party（包括个人和组织），支持外包场景——整个任务可以分配给外部咨询公司而不指定具体执行人。

8. **互斥弧（Exclusive Arc）**：Work Requirement 对 PRODUCT、DELIVERABLE、FIXED ASSET 使用互斥弧约束，确保需求类型的语义一致性。
