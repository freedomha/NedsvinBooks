# 第8章：Accounting and Budgeting（会计与预算）

## 章节概述

本章聚焦于企业数据架构中两个核心财务领域——会计（Accounting）与预算（Budgeting）的信息需求建模。Len Silverston 指出，不同企业之间存在大量相似的会计与预算信息需求，因此通用数据模型具有很高的复用价值。

本章构建了一套完整的数据模型体系，涵盖以下核心主题：

1. **会计科目表（Chart of Accounts）**：为内部组织定义总账科目及会计期间
2. **会计交易（Accounting Transactions）**：涵盖内部交易、外部交易、债务交易和支付交易的超类型/子类型体系
3. **会计交易明细（Transaction Details）**：基于复式记账的借/贷方分录模型
4. **资产折旧（Asset Depreciation）**：固定资产折旧的计算方法追踪
5. **预算定义（Budget Definition）**：预算条目、状态、修订、审核的完整模型
6. **预算情景（Budget Scenarios）**：基于不同市场条件的多情景预算
7. **预算分配（Budget Allocations）**：预算承诺与实际支付的追踪
8. **预算与总账的映射（Budget-to-GL Xref）**：预算条目与总账科目的交叉引用

本章的核心理念是：会计与预算是独立但密切关联的两个领域。会计关注财务报告和历史交易记录，预算关注支出规划与控制。两者通过总账科目建立关联，但服务于不同的业务目的。

---

## 核心问题（Problems）

### 问题一：多内部组织的会计科目管理

企业通常包含多个内部组织（Internal Organizations），如母公司、子公司和各部门。每个内部组织可能使用不同的会计科目表（Chart of Accounts）。数据模型需要：

- 支持全局统一的总账科目定义（GENERAL LEDGER ACCOUNT）
- 允许每个内部组织从全局科目中选择适用科目
- 追踪科目在组织中的生效时间和失效时间
- 支持同一企业内不同组织使用不同的会计期间定义（例如，某组织财年为6月1日至5月31日，而母公司为1月1日至12月31日）

### 问题二：业务交易与会计交易的分离

企业在日常运营中会创建各种业务交易记录（Invoice、Payment、Inventory Item Variance等），这些业务交易最终会触发对应的会计交易。核心问题是：

- 业务交易和会计交易是否应为同一实体？
- 两者之间存在时间差：例如，一张Invoice可能已经创建但尚未过账（pending approval），此时不应产生会计影响
- 业务交易的状态（如"待经理审批"）与会计交易的过账状态是分离的
- 结论：业务交易与会计交易应当建模为独立但关联的实体

### 问题三：内部交易与外部交易的分类

会计交易可以分为两大类，具有不同的特征：

- **内部会计交易（INTERNAL ACCTG TRANS）**：仅影响单一内部组织账簿的调整交易，如折旧（Depreciation）、资本化（Capitalization）、摊销（Amortization）、库存差异调整（Item Variance Acctg Trans）
- **外部会计交易（EXTERNAL ACCTG TRANS）**：涉及外部交易对手方的交易，可进一步分为债务交易（Obligation Acctg Trans）和支付交易（Payment Acctg Trans）

内部交易只涉及一个参与方（被调整的内部组织），而外部交易涉及两方（"from"方和"to"方）。

### 问题四：复式记账的明细管理

复式记账要求每笔交易至少有两笔分录（一个借方、一个贷方），而实际业务中常需要更多分录。例如：

- **发票过账**：借方登记应收账款（Accounts Receivable），贷方登记收入（Revenue）——共2笔明细
- **提前付款并享受折扣**：贷方冲销应收账款$900，借方增加现金$882，借方增加折扣费用$18——共3笔明细
- **固定资产出售**：借方增加现金$1,000，借方冲销累计折旧$200，贷方冲销资产账面价值$800，贷方确认资本利得$400——共4笔明细

数据模型需要灵活支持每笔交易有任意数量的借/贷明细。

### 问题五：会计交易之间的关联

企业需要追踪不同会计交易之间的关联关系：

- 哪些发票（Invoice）被哪些付款（Payment）结清？哪些发票仍为未结？
- 哪些发票通过贷项通知单（Credit Memo）进行了冲减？
- 哪些付款因为与发票金额不符被退回？
- 部分付款（Partial Payment）如何追踪？
- 同一笔付款结清多张发票时如何分配金额？

简单的"付款金额等于债务金额"的推导逻辑无法处理一对多、多对一和部分付款场景。

### 问题六：账户余额——派生数据还是持久化数据？

从逻辑数据建模角度，账户余额是派生数据（可以从交易明细汇总得出），不应冗余存储。但从物理实现角度：

- 财务报表和财务报告大量依赖账户余额信息
- 管理者频繁需要直接查看某科目的当前余额，而非扫描历史交易
- 往年的交易可能已被归档，余额仍需可访问
- 余额在期间结束后很少变动（除非进行回溯调整）

这引发了数据建模中的一个经典权衡：逻辑模型的纯粹性 vs. 物理模型的实用性。

### 问题七：辅助分类账管理（Subsidiary Accounts）

总账科目的背后通常有辅助分类账（Subsidiary Ledger）提供细节：

- "应收账款"总账科目下，需要按客户追踪每位账期客户（Bill-to Customer）的欠款和还款明细
- "应付账款"总账科目下，需要按供应商追踪每家供应商（Supplier）的欠款和支付明细
- 产品收入科目可能需要按产品或产品类别进行细分

辅助分类账科目需要关联到具体的业务实体（客户、供应商、产品等）。

### 问题八：预算修订与历史追踪

预算通常经历多轮修订过程，核心问题包括：

- **简单场景**：企业将每次修订视为全新预算，通过递归关系关联各版本。优点：简单；缺点：需完整重新录入，无法追踪具体变更历史。
- **复杂场景**：企业需要按预算条目（Budget Item）级别追踪每次修订的影响（哪些条目被增加/删除/修改、修改原因、修改金额）。这种多对多关系由 BUDGET REVISION IMPACT 实体解决。

### 问题九：预算分配的追踪复杂性

企业需要同时追踪预算的两个维度：

- **预算承诺（Commitments）**：采购订单（Purchase Order）的订单条目代表对预算的承诺
- **预算支出（Expenditures）**：实际付款代表对预算的支出

关键难题：
- 付款不一定通过订单：员工可能直接到商店使用支票购买物品，没有采购订单
- 一张付款可能对应多个预算条目（例如，一笔$2,000的办公用品购买需要分配到"办公用品"和"家具"两个预算条目）
- 付款追溯回订单的路径极其复杂：Payment → Invoice Item → Shipment Item → Order Item
- 需要将付款（不仅是支出，也包括收入）分配到预算条目中来追踪预算收入

### 问题十：预算条目与总账科目的映射

部门管理者定义预算条目时使用的分类方式（如"营销"），通常与会计师使用的总账科目分类（如"广告费"、"展会费"）不一致。两者之间可能是：

- **一对一映射**：最简单，预算条目名称与总账科目名称一致
- **多对一映射**：多个预算条目（"销售总监薪资"和"销售代表薪资"）映射到同一个总账科目（"薪资费用"）
- **多对多映射**：一个预算条目（"营销"）需要按比例分配到多个总账科目（50%分配至"展会费"，50%分配至"广告费"）

### 问题十一：多情景预算

预算条目在不同市场条件下可能有不同的金额设定。例如：

- "优秀市场条件"下展会预算为$24,000（+20%）
- "一般市场条件"下展会预算为$20,000（基准）
- "较差市场条件"下展会预算为$16,000（-20%）

需要数据模型记录每种情景下的金额变化（绝对值或百分比），以及不同预算条目可能应用不同情景规则。

---

## 解决方案（Solutions）

### 方案一：会计科目表（Chart of Accounts）模型

**核心实体：**

- **GENERAL LEDGER ACCOUNT（总账科目）**：定义企业级的会计科目，包含科目ID（gl account id）和科目名称（gl account name）。这是对所有内部组织可用的标准科目池。

- **ORGANIZATION GL ACCOUNT（组织总账科目）**：将全局总账科目关联到特定内部组织，是关联总账科目与组织之间的桥梁实体。属性包括：
  - `from date`：科目对组织的生效日期
  - `thru date`：科目对组织的失效日期
  - 通过组织结构，允许追踪科目在不同时间的有效性

**数据模型关系：**
```
INTERNAL ORGANIZATION (1) ──< (N) ORGANIZATION GL ACCOUNT (N) >── (1) GENERAL LEDGER ACCOUNT
```

这个结构解决了一个关键需求：不同内部组织可以选择性地使用不同的总账科目。例如，ABC Corporation有"Trade Show Expense"（展会费）科目，而其子公司因为不参展而没有该科目。同时，通过from date和thru date属性，可以追踪科目的历史变更——例如，1997年1月1日，ABC Corporation将"Marketing Expense"科目拆分成了"Advertising Expense"和"Trade Show Expense"两个科目。

### 方案二：会计期间（Accounting Period）模型

**核心实体：**

- **ACCOUNTING PERIOD（会计期间）**：定义财务报告使用的时间期间。属性包括：
  - `from date`：期间开始日期
  - `thru date`：期间结束日期
  - `acctg period num`：期间编号（如13个会计期间的第1-13号，季度的第1-4号）

- **PERIOD TYPE（期间类型）**：定义期间的类型分类（如"fiscal year"、"fiscal quarter"、"calendar month"等）

**递归关系：**
```
ACCOUNTING PERIOD ──< within ── ACCOUNTING PERIOD
```
每月期间可以汇总到季度，季度可以汇总到年度。期间类型（fiscal vs. calendar）由PERIOD TYPE区分。

**替代设计方案讨论：**
书中讨论了一种替代方案——使用字符型的`from day`和`thru day`（如"Mar 1"、"Feb 28"）来一次性定义期间模板，避免每年重复录入。但该方案的缺点是：
- 闰年处理复杂（2月底结束的期间）
- 需要额外的日期转换逻辑来确定交易归属于哪个会计期间
- 不如显式存储日期实用

> **设计选择**：推荐使用显式的from date和thru date，虽然需要每年录入，但实现更直接、更实用。

### 方案三：会计交易（Accounting Transaction）模型

**超类型/子类型体系：**

```
ACCOUNTING TRANSACTION（会计交易，超类型）
├── INTERNAL ACCTG TRANS（内部会计交易）
│   ├── DEPRECIATION（折旧）
│   ├── CAPITALIZATION（资本化）
│   ├── AMORTIZATION（摊销）
│   ├── ITEM VARIANCE ACCTG TRANS（库存差异会计交易）
│   └── OTHER INTERNAL ACCTG TRANS（其他内部会计交易）
│
└── EXTERNAL ACCTG TRANS（外部会计交易）
    ├── OBLIGATION ACCTG TRANS（债务会计交易）
    │   ├── NOTE（票据 - Note Payable/Note Receivable）
    │   ├── CREDIT MEMO（贷项通知单）
    │   ├── TAX DUE（应付税金）
    │   ├── SALES ACCTG TRANS（销售会计交易）
    │   ├── CREDIT LINE（信用额度）
    │   └── OTHER OBLIGATION（其他债务）
    │
    └── PAYMENT ACCTG TRANS（支付会计交易）
        ├── RECEIPT ACCTG TRANS（收款会计交易 - 资金流入）
        └── DISBURSEMENT ACCTG TRANS（付款会计交易 - 资金流出）
```

**核心属性：**
- `transaction id`：交易唯一标识
- `transaction date`：交易发生日期
- `entry date`：系统录入日期
- `description`：交易说明
- 注意：超类型不包含`amount`属性，金额信息在交易明细中维护

**关联关系：**

1. **业务交易溯源**：每个 ACCOUNTING TRANSACTION 可以在业务模型中追溯到其起源：
   - SALES ACCTG TRANS 源自 INVOICE
   - PAYMENT ACCTG TRANS 源自 PAYMENT
   - ITEM VARIANCE ACCTG TRANS 源自 INVENTORY ITEM VARIANCE

2. **交易类型分类**：通过 ACCOUNTING TRANSACTION TYPE 提供更精细的分类（如"Payment Receipt for Asset Sale"、"Payment Disbursement for Purchase Order"）。

3. **参与方关系**：
   - INTERNAL ACCTG TRANS 仅关联一个 INTERNAL ORGANIZATION（被调整的组织）
   - EXTERNAL ACCTG TRANS 关联两个参与方：`from`方和`to`方
   - 一个内部组织向另一个内部组织的付款会产生两笔 PAYMENT ACCTG TRANS：付款方记录 DISBURSEMENT，收款方记录 RECEIPT

**示例数据（Table 8.3）：**

| Transaction ID | 类型 | 说明 | From | To |
|---|---|---|---|---|
| 32389 | Internal/Depreciation | 设备折旧费用 | ABC Corporation (only) | - |
| 39776 | External/Receipt | 支付发票款项，享受2%折扣 | ACME Company | ABC Corporation |
| 38948 | External/Obligation | 销售发票 | ABC Corporation | Customer X |

### 方案四：交易明细模型——复式记账的数字化实现

**核心实体：**

- **TRANSACTION DETAIL（交易明细）**：对应会计术语中的"日记账分录行"（Journal Entry Line Item），包含：
  - `transaction id`（外键）+ `trans detail seq id`：联合主键
  - `debit/credit flag`：借/贷方标识
  - `amount`：金额
  - 关联到 ORGANIZATION GL ACCOUNT（受影响的组织总账科目）

**数据模型关系：**
```
ACCOUNTING TRANSACTION (1) ──< (N) TRANSACTION DETAIL (N) >── (1) ORGANIZATION GL ACCOUNT
```

**业务规则：**
- 每笔 ACCOUNTING TRANSACTION 至少包含两条 TRANSACTION DETAIL（复式记账要求至少一借一贷）
- 每笔 TRANSACTION DETAIL 必须关联到与交易参与方一致的 ORGANIZATION GL ACCOUNT（例如，ABC Corporation的内部交易，其明细必须关联ABC Corporation的科目，不能是其他组织的科目）
- `debit/credit flag` 在物理数据库设计中常实现为金额的正/负号以简化算术运算

**递归关系：交易明细之间的关联**

TRANSACTION DETAIL 上的递归关系解决了交易关联追踪需求：

```
TRANSACTION DETAIL ──< associated with ── TRANSACTION DETAIL
```

这个递归关系可以回答：
- 哪些付款结清了哪些发票？
- 哪些贷项通知单冲减了哪些发票金额？
- 哪些付款被退回给原交易方？

**示例（Table 8.5）：多对一付款场景**

- **交易 38948**：发票，$900应收账款
- **交易 50984**：发票A，金额$X
- **交易 50999**：发票B，金额$Y
- **交易 60985**：单笔付款同时结清交易50984和50999 —— 该交易的"应收账款"贷方分录被拆分为两条明细记录，分别关联到交易50984和50999，以便追踪各Invoice被分配的具体金额

这个递归关系设计可以容纳所有会计交易关联场景：债务关联债务（Credit Memo冲减Invoice）、支付关联支付（退款）、部分付款关联债务、以及固定资产出售关联原始购买交易。

### 方案五：账户余额——逻辑模型与物理模型的权衡

**方案A（逻辑模型 - Figure 8.3a）：** 不包含账户余额实体，因为余额是派生数据，可从 TRANSACTION DETAIL 汇总得出。这是"纯粹"的数据建模实践，避免了数据冗余和同步问题。

**方案B（物理模型 - Figure 8.3b）：** 引入 ORGANIZATION GL ACCOUNT BALANCE 实体，存储在 ORGANIZATION GL ACCOUNT 和 TRANSACTION DETAIL 之间：

```
ORGANIZATION GL ACCOUNT (1) ──< (N) ORGANIZATION GL ACCOUNT BALANCE (N) >── (1) ACCOUNTING PERIOD
```

- ORGANIZATION GL ACCOUNT BALANCE 存储特定科目在特定会计期间的当前余额
- 提供对余额信息的快速直接访问
- 避免查询性能问题（无需扫描所有历史交易来汇总余额）

**设计建议：**
- 逻辑数据模型中不宜包含账户余额（避免数据同步问题）
- 物理设计中强烈建议包含（满足性能需求和实际使用场景）
- 余额虽然在期间结束后相对稳定，但回溯调整（Retroactive Adjustments）可能导致同步问题，需要在物理实现中加以控制

### 方案六：辅助分类账（Subsidiary Accounts）模型

**递归关系：**
```
ORGANIZATION GL ACCOUNT ──< comprised of ── ORGANIZATION GL ACCOUNT
```

父总账科目（如"应收账款"总账科目）可以包含多个子辅助分类账科目（如按客户分的应收账款明细）。

**关联关系：**
- SUBSIDIARY ACCOUNT（ORGANIZATION GL ACCOUNT的子类型）关联到 BIACCTO CUSTOMER（Party Role子类型）
- SUBSIDIARY ACCOUNT 关联到 SUPPLIER（Party Role子类型）
- SUBSIDIARY ACCOUNT 关联到 PRODUCT 或 PRODUCT CATEGORY（用于追踪各产品/产品线的收入）

书中讨论了将辅助科目关联到特定角色（BIACCTO CUSTOMER、SUPPLIER）还是通用 PARTY 的设计选择。通用方案更灵活（可能有其他角色的参与方欠款或被欠款），但特定角色方案更直观、约束更强。

### 方案七：资产折旧（Asset Depreciation）模型

**核心实体：**

- **DEPRECIATION**：折旧交易，是 INTERNAL ACCTG TRANS 的子类型
- **FIXED ASSET**：固定资产（第6章定义），每笔折旧对应一个固定资产
- **DEPRECIATION METHOD**：折旧方法，属性包括：
  - `description`：折旧方法描述（如"Straight-line Depreciation"、"Double-declining Balance Depreciation"）
  - `formula`：计算公式文本（如"(Purchase cost - salvage cost) / estimated life in years of the asset"）
- **FIXED ASSET DEPRECIATION METHOD**：关联实体，解决固定资产与折旧方法之间随时间变化的多对多关系：
  - `from date` / `thru date`：该折旧方法对固定资产的适用期间
  - 一个固定资产可能随时间更换折旧方法（受IRS等机构监管限制）

**示例（Table 8.6 - Pen Engraver）：**
- 1999年：使用双倍余额递减法（Double-declining balance）—— "(Purchase cost - salvage cost) * (1/estimated life in years of the asset) * 2"
- 2000年起：切换为直线折旧法（Straight-line）—— "(Purchase cost - salvage cost) / estimated life in years of the asset"

### 方案八：预算定义（Budget Definition）模型

**核心实体：**

- **BUDGET**：预算主实体
  - `budget id`：预算唯一标识
  - `description`：预算描述
  - 关联到 STANDARD TIME PERIOD（预算适用期间）和 PERIOD TYPE（期间类型）

- **BUDGET TYPE**：预算类型分类
  - OPERATING BUDGET（运营预算）：用于费用类项目
  - CAPITAL BUDGET（资本预算）：用于固定资产和长期项目
  - 可通过 BUDGET TYPE 灵活扩展其他类型

- **BUDGET ITEM**：预算条目
  - `amount`：该条目所需资金总额
  - `purpose`：该条目的目的
  - `justification`：资金需求的理由说明
  - 关联到 BUDGET ITEM TYPE（可复用的条目类型描述）
  - 递归关系：BUDGET ITEM 可包含其他 BUDGET ITEM，支持层级汇总

- **BUDGET ROLE**：预算参与方角色，PARTY 可扮演的角色包括：
  - initiator（发起人）
  - requested for（被请求方，即预算的使用部门/组织）
  - reviewer（审核人）
  - approver（审批人）

- **BUDGET STATUS**：预算状态追踪
  - 关联到 BUDGET STATUS TYPE（STATUS TYPE 的子类型）
  - `status date`：状态变更日期
  - `description` / `comment`：状态说明

**设计原则：**
预算应定义在组织的最低层级（如部门级别），以便灵活汇总到各个层级。

**示例（Table 8.7/8.8）：**

| Budget ID | Type | 部门 | 期间 | 条目 | 金额 |
|---|---|---|---|---|---|
| 29839 | Operating | Marketing Dept | 2001全年 | Trade Shows, Advertising, Direct Mail | $65,000 |
| 38576 | Operating | Administration Dept | 2002年6月 | Office Supplies, Furniture | $15,000 |
| 39908 | Capital | Manufacturing Ops | 2001全年 | 新制造设备 | - |

### 方案九：预算修订（Budget Revision）模型

书中提供了两种设计方案以满足不同的企业需求：

**方案A（简单版 - Figure 8.7 Top）：**

每次修订创建全新 BUDGET 实体，通过递归关系关联旧预算和新预算：

```
BUDGET ──< replaced by ── BUDGET
```

优点：模型简单，易于理解和实现。  
缺点：需要完整重新录入所有条目，无法追踪单个条目的变更历史。适合预算结构简单、修订次数少的企业。

**方案B（详细版 - Figure 8.7 Bottom）：**

引入 BUDGET REVISION 和 BUDGET REVISION IMPACT 实体，支持条目级别的变更追踪：

```
BUDGET (1) ──< (N) BUDGET REVISION (N) >──< (N) BUDGET ITEM
                                       └── BUDGET REVISION IMPACT（关联实体）
```

**BUDGET REVISION** 实体：
- 主键：(budget id, revision seq id)
- 关联到原始 BUDGET

**BUDGET REVISION IMPACT** 实体：
- `revised amount`：修订后的金额变化（正数增加，负数减少）
- `add delete flag`：标识该条目是被新增还是被删除
- `revision reason`：修订原因说明

通过 BUDGET REVISION IMPACT 可以回答：
- 每次修订影响了哪些预算条目？
- 每个预算条目被修订过几次？每次修订的金额变化和原因是什么？
- 预算的完整变更历史是什么？

**示例（Table 8.10 - Budget 29839 的修订过程）：**

| Revision | 受影响条目 | 金额变化 | 操作 | 原因 |
|---|---|---|---|---|
| 1.1 | Item 2 (Advertising) | -$10,000 | 修改 | 大幅削减广告预算 |
| 1.1 | Item 3 (Direct Mail) | -$7,000 | 修改 | 大幅削减直邮预算 |
| 1.1 | Item 4 (Internet Advertising) | +$5,000 | 新增 | 增加互联网广告预算 |
| 1.2 | Item 3 (Direct Mail) | -$2,000 | 修改 | 直邮预算仍需进一步削减 |

> 注意：Revision 1.1 影响三个条目（一对多），Item 3 又被 Revision 1.1 和 1.2 两次修订（多对一），验证了多对多关系的必要性。

### 方案十：预算审核（Budget Review）模型

**核心实体：**

- **BUDGET REVIEW**：预算审核记录
  - 关联到 BUDGET（每个预算可能经历多轮审核）
  - 关联到 PARTY（参与审核的个人）
  - `review date`：审核日期
  - `comment`：审核意见

- **BUDGET REVIEW RESULT TYPE**：审核结果类型（如"Accepted"、"Rejected"）

**设计决策讨论：**
- BUDGET REVIEW 关联到 BUDGET 还是 BUDGET REVISION？
  - 书中选择关联到 BUDGET（每次修订是预算的一部分，审核是针对整个预算的）
  - 审核结果通过业务规则间接影响 BUDGET STATUS，但不建立直接的数据关系（因为审核和状态各自独立存在）

**示例（Table 8.11）：**

| Budget ID | Reviewer | Review Date | Result | Comment |
|---|---|---|---|---|
| 29839 | Susan Jones | Nov 10, 2000 | Accepted | Budget seems reasonable |
| 29839 | John Smith | Nov 15, 2000 | Rejected | Budgeted amount is too high |
| 29839 | Susan Jones | Nov 22, 2000 | Accepted | Budget is OK |
| 29839 | John Smith | Nov 30, 2000 | Accepted | Budget is OK |

### 方案十一：预算情景（Budget Scenario）模型

**核心实体：**

- **BUDGET SCENARIO**：预算情景定义
  - `description`：情景类型描述（如"excellent market conditions"、"poor market conditions"、"worst case"、"best case"、"major deal signed"、"no major deal signed"）

- **BUDGET SCENARIO APPLICATION**：预算情景应用
  - 关联到 BUDGET（影响整个预算）或 BUDGET ITEM（影响特定条目）
  - `amount change`：绝对值变化金额
  - `percentage change`：百分比变化

- **BUDGET SCENARIO RULE**：预算情景规则
  - 关联到 BUDGET ITEM TYPE（按条目类型定义默认规则）
  - `amount change` / `percentage change`：标准的增减额或百分比
  - 这些规则提供默认值，但对特定预算条目可以覆写

**设计灵活性：**
- 情景可以在预算级别统一应用，也可按条目精细控制
- 规则提供标准默认值，但具体预算可以具有不同的实际值

**示例（Table 8.12）：**

| Budget Item | Base Amount | Scenario | Rule % Change | Actual % Change |
|---|---|---|---|---|
| Trade Shows | $20,000 | Excellent marketing | +20% | +20% |
| Trade Shows | $20,000 | Poor marketing | -15% | **-20%** (覆写规则) |
| Advertising | $30,000 | Excellent marketing | +25% | +25% |
| Advertising | $30,000 | Poor marketing | -15% | -15% |

注意：Trade Shows在"poor marketing conditions"下使用了-20%而非规则默认的-15%，说明特定预算条目可以覆写默认规则。

### 方案十二：预算分配——承诺与支出追踪

**承诺追踪（Commitment Tracking）：**

1. **ORDER ITEM → BUDGET ITEM**：采购订单条目直接关联到预算条目
   - 代表已建立的对预算的承诺（Commitment）
   - 同样适用于销售订单条目（追踪销售承诺与预算收入的对比）

**关键设计决策：为什么用 ORDER ITEM 而非 PRODUCT？**
- 同样的产品（如个人电脑）在不同采购场景下可能分配到不同的预算条目
- 一台PC用于系统开发项目 → 分配到项目预算
- 另一台PC用于某员工的设备更新 → 分配到计算机设备预算
- 因此，预算分配应由具体的订单条目决定，而非由产品类型决定

2. **REQUIREMENT → BUDGET ITEM**（多对多，通过 REQUIREMENT BUDGET ALLOCATION）：
   - 在正式承诺之前，需求（Requirement）可以预先分配到预算条目以验证资金可用性
   - 一个需求可以涉及多个预算条目（如项目需要从"薪资"预算和"办公用品"预算分别拨款）
   - 一个预算条目可以支撑多个需求
   - `amount` 属性记录具体的分配金额

**支出追踪（Expenditure Tracking）：**

1. **有订单的付款**：付款通过复杂的关系路径追溯到订单条目：
   ```
   PAYMENT → INVOICE PAYMENT ITEM APPLICATION → INVOICE ITEM
            → SHIPMENT ITEM（实物商品）或 ORDER ITEM（服务采购）
            → ORDER ITEM → BUDGET ITEM
   ```
   这种路径虽然复杂但是必要的：为了准确反映针对预算条目的承诺金额 vs. 已支出金额，必须将付款映射回原始订单。企业需要建立业务规则来分配付款到对应的采购订单。

2. **无订单的付款**：通过 PAYMENT BUDGET ALLOCATION 实体直接关联：
   ```
   BUDGET ITEM (N) >──< (N) PAYMENT
                     └── PAYMENT BUDGET ALLOCATION（amount）
   ```
   适用于员工直接在商店购物但需要按预算条目分配的场景：
   
   **示例（Table 8.13）：**
   | Payment | Amount | Budget ID | Budget Item | Allocation |
   |---|---|---|---|---|
   | #2903 (椅子+办公用品) | $2,000 | 38576 | Office Supplies | $500 |
   | #2903 (椅子+办公用品) | $2,000 | 38576 | Furniture | $1,500 |

**设计特点总结：**
- OBLIGATION的commitment通过ORDER ITEM到BUDGET ITEM的一对多关系追踪
- DISBURSEMENT的expenditure：
  - 有PO的：通过PAYMENT → ORDER ITEM的路径推导
  - 无PO的：通过PAYMENT BUDGET ALLOCATION显式记录
- 该模型同时支持支出预算（采购端）和收入预算（销售端），RECEIPT也可通过PAYMENT BUDGET ALLOCATION追踪

### 方案十三：预算与总账的映射模型

**核心实体：**

- **GL BUDGET XREF（总账预算交叉引用）**：关联BUDGET ITEM TYPE与GENERAL LEDGER ACCOUNT的多对多关系
  - `allocation percentage`：分配百分比（如"营销"预算条目50%分配到"展会费"科目，50%分配到"广告费"科目）
  - `from date` / `thru date`：映射关系的时间有效性（支持随时间变化的映射规则）

**数据模型关系：**
```
BUDGET ITEM TYPE (N) >──< (N) GENERAL LEDGER ACCOUNT
                        └── GL BUDGET XREF
```

**三种映射场景：**

1. **一对一**（最理想）：预算条目 "Office Supplies" ↔ 总账科目 "Office Supplies Expense"（100%）
2. **多对一**：多个预算条目映射到一个总账科目 —— "Sales Director" + "Sales Representative" → "Salaries Expense"（100% each）
3. **多对多**（最复杂）：一个预算条目映射到多个总账科目 —— "Marketing" → 50% "Trade Show Expense" + 50% "Advertising Expense"

**预算金额与总账科目的关联查询路径：**
```
BUDGET ITEM → BUDGET ITEM TYPE → GL BUDGET XREF → GENERAL LEDGER ACCOUNT → ORGANIZATION GL ACCOUNT
```

通过这个关系路径，企业可以查询"针对某个总账科目的预算额度是多少"。

**设计变体：**
如果企业的预算条目与总账科目始终为一对一关系，可以：
- 去掉 GL BUDGET XREF
- 使用 GENERAL LEDGER ACCOUNT 和 BUDGET ITEM TYPE 之间的一对一关系
- 甚至直接在 ORGANIZATION GL ACCOUNT 中添加 `budgeted amount` 字段，替代整个 BUDGET/BUDGET ITEM 结构

如果确实不需要存储其他预算信息（状态、审核、修订、情景等），则可以大幅简化模型。

---

## 关键数据模型/概念

### 概念一：业务交易与会计交易的分离（Business vs. Accounting Transactions）

这是本章最重要的设计理念之一。业务交易（如INVOICE、PAYMENT、INVENTORY ITEM VARIANCE）和会计交易（ACCOUNTING TRANSACTION及其子类型）应当是独立的实体，原因包括：

1. **时间差**：业务交易可能已创建但未过账（状态为"pending approval"）
2. **状态独立**：业务交易有自己的生命周期状态，不一定与会计过账同步
3. **职责分离**：业务系统关注运营流程，会计系统关注财务报告
4. **一对一但不是同一事物**：虽然每个业务交易对应一个会计交易，但它们代表了不同关注点的信息

这种分离使得数据模型既能满足业务运营的灵活性需求，又能保证会计信息的准确性和可控性。

### 概念二：内部交易 vs. 外部交易的参与方模型

会计交易根据参与方的不同分为两大类型，具有不同的关系模式：

| 交易类型 | 参与方数量 | 关系模式 | 示例 |
|---|---|---|---|
| INTERNAL ACCTG TRANS | 1个（被调整组织） | 单一关联到 INTERNAL ORGANIZATION | 折旧、摊销、库存调整 |
| EXTERNAL ACCTG TRANS (Obligation) | 2个（from方和to方） | 两个关联到 PARTY（债权人和债务人） | 发票、票据、贷项通知单 |
| EXTERNAL ACCTG TRANS (Payment) | 2个（from方和to方） | 两个关联到 PARTY（付款方和收款方） | 收款、付款 |

> 注意：内部组织之间的交易会产生两笔 PAYMENT ACCTG TRANS——付款方记录DISBURSEMENT，收款方记录RECEIPT。

### 概念三：交易明细递归关系的设计价值

TRANSACTION DETAIL 上的递归关系（`associated with`）是本章的一个精妙设计，它在不引入额外实体的情况下解决了复杂的交易关联追踪需求。

**支持的业务能力：**
- 债务到债务的关联（Credit Memo冲减Invoice）
- 支付到债务的关联（Payment结清Invoice）
- 支付到支付的关联（退还付款）
- 内部交易到外部交易的关联（固定资产出售关联原始折旧）

**关键数据示例：**
多发票单笔付款场景下，"应收账款"贷方分录被拆分为多条明细，每条关联到不同的原始发票。这种设计使得系统可以通过数据关系而非应用逻辑来回答"哪些发票被哪笔付款结清"这类复杂查询。

### 概念四：订单驱动 vs. 非订单驱动的预算支出追踪

本章设计了一套双轨制的预算支出追踪机制：

```
                    ┌── 有PO ── ORDER ITEM → BUDGET ITEM（通过路径推导）
PAYMENT ──┤
                    └── 无PO ── PAYMENT BUDGET ALLOCATION → BUDGET ITEM（直接记录）
```

**有采购订单路径的复杂度：**

```
PAYMENT → INVOICE PAYMENT ITEM APPLICATION → INVOICE ITEM
                                                ↓
                                         SHIPMENT ITEM（实物商品）
                                         或 ORDER ITEM（服务采购）
                                                ↓
                                          ORDER ITEM → BUDGET ITEM
```

这条路径虽然复杂（涉及多个中间实体的联结），但是对于准确反映"承诺 vs. 支出"是必要的。如果没有这条路径，$50,000的采购订单和对应的$25,000付



Let me update this task and send the results to team-lead.

<｜｜DSML｜｜tool_calls>
<｜｜DSML｜｜invoke name="TaskUpdate">
<｜｜DSML｜｜parameter name="taskId" string="true">2