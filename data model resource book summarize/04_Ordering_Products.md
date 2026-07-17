# 第4章：Ordering Products（产品订购）

## 章节概述

本章深入探讨了产品订购领域的数据模型设计，涵盖订单（销售订单和采购订单）、订单项、订单参与方、订单角色、订单调整、订单状态与条款、订单项关联，同时还介绍了可选模型：需求（Requirements）、请求（Requests）、报价（Quotes）以及协议（Agreements）。核心目标是从"我方"视角的简单订单模型，演进到一个灵活的、支持多角色、多参与方、多联系方式的通用订单数据模型，能够同时处理销售和采购两种视角，既适用于商品也适用于服务。

---

## 核心问题（Problems）

### 1. 销售订单与采购订单的数据结构割裂

传统的且普遍的做法是为销售订单（Sales Order）和采购订单（Purchase Order）各自建立独立的数据模型（如图4.1所示）。这带来了以下问题：

- **信息冗余**：销售订单和采购订单的属性和结构非常相似，实际上"订单只是订单"（an order is an order），区别仅在于视角——卖方视角 vs 买方视角。
- **无法复用**：由于两者数据结构不同，开发者无法创建通用程序来处理共同的逻辑，例如查询订单条款、监控订单状态与预计交付日期、计算订单项价格扩展等。
- **额外的差异不必要**：少数属性差异（如销售订单中需要记录销售人员的佣金，而采购方一般无权知晓）完全可以通过子类型来处理。

### 2. "我方"视角的局限性

传统模型隐含一个假设：订单中只涉及一个组织——销售订单模型只有 Customer，采购订单模型只有 Supplier。这是一种"以我为出发点"（"I" perspective）的设计，即不需要明确谁来接收销售订单或谁来下采购订单，因为"显然就是我们自己"。

**问题表现**：
- 企业内部可能有多个组织（子公司、部门、分支机构）会接收销售订单或下达采购订单，需要记录下来
- 即使是小型企业，也可能有经纪人（broker）代为接单，或未来会发展出子公司等内部组织
- 同一订单可能涉及多个参与方：下单客户（Placing Customer）、收货客户（Ship-to Customer）、账单客户（Bill-to Customer）、安装客户（Installation Customer）、接单人（Person Taking the Order）、登记订单的组织等，采购订单也存在类似的多角色需求

### 3. 联系方式（Contact Mechanism）信息缺失

订单涉及多种联系方式，传统模型未覆盖：
- 订单应开票到哪里？
- 订单应发运到哪里？
- 订单源自哪个联系方式（特定电子邮件、电话号码）？
- 接收订单的位置或联系方式是什么？

### 4. 订单调整（Order Adjustments）的表达

传统"order line items"概念不仅表示订购的产品，还包含折扣、附加费、税费、运费、说明等。如何建模：

- 是否将调整作为 ORDER ITEM 的子类型？
- 这需要 ORDER ITEM 的自递归关系来表示调整适用于哪个订单项
- 调整既可能作用于整个 ORDER，也可能作用于某个 ORDER ITEM
- 调整本质上是不同于"被订购的东西"的概念

### 5. 订单状态与条款的动态管理

订单随时间会经历多种状态（已接收、已批准、已取消等），需要跟踪状态的历史。订单及订单项可能关联多种条款（如取消费用、退换政策、履约罚金等），有些条款是订单级别的，有些是订单项级别的。

### 6. 订单项之间的关联

存在销售订单项与采购订单项之间的直接关联需求：
- 经销商接到销售订单后库存不足，向供应商下采购订单来补货
- 一笔采购订单项可能满足多笔销售订单项，反之亦然
- 销售方可能需要记录买方对应的采购订单号（corresponding PO ID）

### 7. 需求管理（可选）

订单源于需求。有些企业需要跟踪客户需求（Customer Requirements）或内部需求（Internal Requirements）：

- 需求可以是产品需求（Product Requirement）或工作需求（Work Requirement）
- 需求可能有层次结构（递归关系）
- 需求需要多种角色参与、状态跟踪
- 需求与订单之间的多对多关系（一个需求可能被多个订单项满足，一个订单项可能满足多个需求）

### 8. 请求与报价流程（可选）

在某些场景下，不直接下单，而是先走请求（Request）和报价（Quote）流程：

- 请求用于向供应商征集出价/报价/响应（RFP、RFQ、RFI）
- 报价是请求的响应
- 一个需求可能关联多次请求，一次请求可能有多个报价
- 报价项可能与请求要求的产品不完全一致（更具体的产品）

### 9. 协议管理（可选）

协议（Agreement）定义了双方或多方长期合作的条款和条件，与订单的关键区别在于是长期承诺而非一次性承诺。需要建模的方面：

- 多种协议类型（销售协议、采购协议、雇佣协议、合伙协议等）
- 协议的参与方角色
- 协议项（Agreement Items）及其层次结构
- 协议的条款（Agreement Terms）
- 协议的定价（Agreement Pricing）
- 协议的补充/修订（Addendum）
- 协议的适用范围（地理区域、产品、组织）
- 协议与订单的关系

---

## 解决方案（Solutions）

### 解决方案1：统一的 ORDER 模型（Orders and Order Items）

**核心设计（图4.2）**：

```
ORDER（订单）
├── 子类型：SALES ORDER（销售订单）
├── 子类型：PURCHASE ORDER（采购订单）
└── 包含 ORDER ITEM（订单项）
    ├── 子类型：PURCHASE ORDER ITEM
    ├── 子类型：SALES ORDER ITEM
    └── 关联 PRODUCT（产品）或 PRODUCT FEATURE（产品特性）
```

**关键属性**：
- `order date`：订单接收/下达日期
- `entry date`：订单录入系统日期
- ORDER ITEM 的 `quantity`：数量（商品）或工时/天数（服务）
- `unit price`：单价，允许覆盖通过定价模型计算的价格（协商价）
- `estimated delivery date`：预计交付日期
- `shipping instructions`：发运说明
- `comments`：备注
- `item description`：用于非标准项或非产品类项（如工作成果、专业服务工时）

**为什么 UNIT PRICE 不是派生字段**：允许用户用实际协商价格覆盖系统计算价格，基础价格、折扣和附加费都可以作为与某产品关联的订单项出现。

**为什么叫 ORDER ITEM 而非 ORDER LINE ITEM**："order line items"一词暗示订单表单上的物理行，而订单表单上的行可能包含更多内容（备注、调整、税费等）。ORDER ITEM 专门表示已订购的产品或服务。

**产品特性（PRODUCT FEATURE）的建模**（表4.4、递归关系 ordered with）：
- 产品特性可用于定制订单（如颜色、尺寸、软件特性）
- 使用递归关系：一个 ORDER ITEM（for 某 product feature）关联到另一个 ORDER ITEM（for 订购的产品）
- 而非在同一个 ORDER ITEM 中同时维护产品和特性——因为同一产品可能在两个不同订单项上以不同特性组合被订购

### 解决方案2：订单参与方与联系方式

#### 2.1 销售订单参与方与联系方式（图4.3）

**订单级别的关键角色和联系方式**：
- **placed by** → PLACING CUSTOMER（下单客户）：可由个人、组织或经纪人/代理下单
- **placed using** → CONTACT MECHANISM（下订单使用的联系方式）
- **taken by** → INTERNAL ORGANIZATION（接单的内部组织）：特定子公司、部门、门店、或通过网站URL
- **taken via** → CONTACT MECHANISM（接单的联系方式）
- **designated to be billed to** → CONTACT MECHANISM（开票地址）
- **requested bill to** → BILL TO CUSTOMER（付款方）

**订单项级别的收货信息**：
- **designated to be shipped to** → SHIP TO CUSTOMER（收货方）——在每个 ORDER ITEM 级别，而非 ORDER 级别
  - 原因：一个订单的不同商品可能发往不同收货方，如同一个订单5000箱可乐分别发往5个区域门店
- **designated to be shipped to** → CONTACT MECHANISM（收货地址）

**为什么收货信息在订单层面而非发货层面**：发货（SHIPMENT，第5章）可以将来自多个订单的订单项合并为一次派送，因此在订单阶段就需要记录期望的收货信息以决定如何组合发货。

**订单角色与联系方式的独立性**：
- 订单直接关联 PARTY 和 CONTACT MECHANISM 是独立的关系，而非通过 PARTY CONTACT MECHANISM —— 因为可能只知道某一方面不清楚另外一方面：
  - 知道谁下单但不知道联系方式
  - 互联网订单可能知道网址但不知道是谁下的单

#### 2.2 采购订单参与方与联系方式（图4.4）

与销售订单结构相同，但角色名称不同：
- **placed by** → PLACING PARTY（下单方）
- **taken by** → SUPPLIER（供应商核收）
- **requested bill to** → BILL TO PURCHASER（开票采购方）
- **designated to be shipped to** → SHIP TO BUYER（收货采购方）

#### 2.3 通用订单角色与联系方式（图4.5）

当希望数据模型更加灵活、能够适应业务规则变化时，使用通用模型：

```
ORDER
└── ORDER ROLE（可由 ORDER ROLE TYPE 描述）
    └── 关联 PARTY

ORDER
└── ORDER CONTACT MECHANISM（可由 CONTACT MECHANISM PURPOSE TYPE 描述）
    └── 关联 CONTACT MECHANISM

ORDER ITEM
└── ORDER ITEM ROLE（可由 ORDER ITEM ROLE TYPE 描述）
    └── 关联 PARTY

ORDER ITEM
└── ORDER ITEM CONTACT MECHANISM（可由 CONTACT MECHANISM PURPOSE TYPE 描述）
    └── 关联 CONTACT MECHANISM
```

**通用模型 vs 特定模型的权衡**：
- **特定模型优点**：明确表达业务规则、易于阅读和理解（如 ORDER 关联 BILL TO CUSTOMER 比 ORDER 有多个 ORDER ROLE 其中一个 type 是 "bill-to customer" 更直观）
- **通用模型优点**：灵活性强，业务规则变化时（如从"一个开票客户"变为"多个开票客户"）基础数据结构不变
- **建议**：如果业务规则稳定且不预期变化，使用特定建模方法；如果预期可能变化，使用通用方法

**ORDER ROLE TYPE 的示例值**：placing party、bill-to customer、internal organization taking order、placing customer、placing buyer、supplier、bill to purchaser、order entry person、order salesperson、order authorizer

**CONTACT MECHANISM PURPOSE TYPE 的示例值**：ship to、bill to、confirmation、placing、taken via

### 解决方案3：人员订单角色（Person Roles for Orders）

在图4.3中通过 ORDER ROLE 机制存储参与者在订单中的各种角色（表4.6）：

| 角色示例 | 说明 |
|---------|------|
| Salesperson（销售人员） | 完成销售的人员，含 `percent contribution` 属性用于佣金计算 |
| Processor（处理人） | 负责录入数据的人员 |
| Reviewer（审核人） | 负责审核订单信息的人员 |
| Authorizer（授权人） | 批准订单有效承诺的人员 |

**角色分配**：即使该功能尚未执行，也可以预先指定某方承担该角色。

### 解决方案4：订单调整（Order Adjustments — 图4.6）

**设计决策**：ORDER ADJUSTMENT 作为独立实体，而非 ORDER ITEM 的子类型。

**理由**：调整（折扣、附加费、手续费、运费）并不是被"订购"的东西，与表示被订购产品的 ORDER ITEM 在概念上不同。

**ORDER ADJUSTMENT 的子类型**：
- **DISCOUNT ADJUSTMENT**（折扣调整）：按金额或百分比，可作用于整个 ORDER 或单个 ORDER ITEM
- **SURCHARGE ADJUSTMENT**（附加费调整）：如"非常规区域的派送附加费$10.00"
- **SALES TAX**（销售税）：订单或订单项级别的税费
- **SHIPPING AND HANDLING CHARGE**（发运处理费）
- **FEE**（费用）：如订单处理费、管理费
- **MISCELLANEOUS CHARGE**（杂项费用）：如"调整错误"以修正之前的订单

**ORDER ADJUSTMENT TYPE** 实体用于将调整进一步细分为详细类别。

**SALES TAX LOOKUP**（销售税查询）实体：存储税率，税率可因 GEOGRAPHIC BOUNDARY（县、市、州）和 PRODUCT CATEGORY（如食品类可能有不同税率、某些产品可免税）而异。

### 解决方案5：订单状态与条款（Order Status and Terms — 图4.7）

**ORDER STATUS**：
- 跟踪 ORDER 和/或 ORDER ITEM 的状态历史
- `status datetime` 属性记录每种状态发生的时间
- **ORDER STATUS TYPE** 维护可能的状态值：received（已接收）、approved（已批准）、canceled（已取消）等
- **设计说明**：不将 shipped、completed、backordered、invoiced 作为订单状态——这些可从发货项、发票项、订单项关联等关系中推导得出（逻辑模型中派生，物理模型中可直接存储方便访问）

**ORDER TERM**（订单条款）：
- 订单和订单项均可关联条款
- **TERM TYPE** 维护可能的条款类型
- `term value` 属性的含义取决于条款类型
- 示例（表4.9）：
  - 若购买方在10天内取消，收取25%取消费
  - 商品一旦交付不接受退换
  - 若超过预计交付日期30天，供应商支付5%履约罚金

### 解决方案6：订单项关联（Order Item Association — 图4.8）

**ORDER ITEM ASSOCIATION**：
- 多对多关系连接 SALES ORDER ITEM 和 PURCHASE ORDER ITEM
- 典型场景：经销商接到销售订单后库存不足，向供应商下采购订单补货（即"backordered"）
- 一个采购订单项可满足多个销售订单项，一个销售订单项也可由多个采购订单项满足

**corresponding PO ID** 属性：
- 定义在 SALES ORDER ITEM 上
- 用于记录买方的对应采购订单号
- 定义为订单项级别而非订单头级别——一个销售订单的不同项可能对应不同采购订单

### 解决方案7：需求管理（Requirements — 图4.9）

**REQUIREMENT（需求）实体**：
- 子类型：CUSTOMER REQUIREMENT（客户需求）、INTERNAL REQUIREMENT（内部需求）
- 也可是：PRODUCT REQUIREMENT（产品需求）、WORK REQUIREMENT（工作需求）
- **递归关系**：一个需求可包含多个子需求（如"采购办公用品"分解为"买纸、买铅笔、买笔、买光盘"）

**关键属性**：
- `description`：需求描述
- `requirement creation date`：创建日期
- `required by date`：需要满足的截止日期
- `estimated budget`：预算金额
- `quantity`：需要的数量
- `reason`：需要的原因

**PRODUCT REQUIREMENT**：
- 可选关联 PRODUCT（产品）
- 可关联 DESIRED FEATURE（期望特性），如颜色"蓝色"、`optional ind` 表示是"期望"还是"必须"

**REQUIREMENT ROLEs**：owner（所有者）、originator（发起人）、manager（管理者）、authorizer（授权人）、implementor（实施人）。`from date` 和 `thru date` 定义角色有效时间段。

**REQUIREMENT STATUS**：active（活跃）、on-hold（暂停）、inactive（不活跃）及其历史记录。

**ORDER REQUIREMENT COMMITMENT**：
- 多对多关系连接 REQUIREMENT 和 ORDER ITEM
- `quantity` 属性记录从某个订单项分配了多少数量满足某需求

### 解决方案8：请求管理（Requests — 图4.10）

**REQUEST（请求）实体**：
- 子类型：
  - **RFP**（Request for Proposal 征求建议书）：要求供应商对需求提出解决方案
  - **RFQ**（Request for Quote 征求报价）：要求供应商对特定产品进行报价
  - **RFI**（Request for Information 征求信息）：通常在RFP/RFQ之前，用于筛查合格供应商

**属性**：`request date`（创建日期）、`response required date`（回复截止日期）、`description`（描述）

**REQUEST ROLEs**：originator（发起方）、preparer（准备人）、manager（管理者）、quality assurer（质量保证人）

**RESPONDING PARTY**（回复方）：被要求响应请求的一方，主要用于外发请求，因为接收的请求通常只有一个回复方。

**REQUEST ITEM**（请求项）：
- `quantity`：多少数量
- `required by date`：需要交付的日期
- `maximum amount`：最高价格上限
- 可选关联 PRODUCT（对于RFQ类请求）或通过 description 描述问题（对于RFP类请求）

**REQUIREMENT REQUEST**（需求-请求关联）：
- 多对多关系：一个需求可能关联多个请求（如先发RFI再发RFQ），一个请求项可能合并多个需求

### 解决方案9：报价管理（Quote Definition — 图4.11）

**QUOTE 实体**：报价/投标/建议书的统称

**子类型**：
- **PROPOSAL**（方案建议书）：更详细，包含需求陈述、方案描述、收益、成本论证、所需资源等
- **PRODUCT QUOTE**（产品报价）：较简单，仅记录产品条款和价格

**属性**：`issue date`（签日期）、`valid from date` / `valid thru date`（有效期）、`description`

**QUOTE ROLEs**：quoted by（报价人）、reviewed by（审核人）、approved by（批准人）。此外显式标出 issued by 和 given to 两方。

**QUOTE ITEM**（报价项）：
- 关联 PRODUCT（报价产品可能与请求产品不同，更具体）
- 必须与单个 REQUEST ITEM 关联（报价是对请求的响应）
- 一个 REQUEST ITEM 可能有多个 QUOTE ITEM（多家供应商报价）
- 一个 QUOTE ITEM 可能产生多个 ORDER ITEM（可被多次订购）

**QUOTE TERM**：报价和报价项均可以有关联条款

### 解决方案10：协议管理（Agreement Definition — 图4.12/4.13/4.14/4.15/4.16）

#### 10.1 协议核心结构（图4.12）

**AGREEMENT 实体**：
- 子类型：PRODUCT AGREEMENT → SALES AGREEMENT / PURCHASE AGREEMENT、EMPLOYMENT AGREEMENT、PARTNERSHIP AGREEMENT 等
- 属性：`agreement date`、`from date`、`thru date`、`description`、`text`（协议文本）

**AGREEMENT ROLE 与 PARTY RELATIONSHIP 的区别**：
- PARTY RELATIONSHIP 记录非正式关系（如 customer relationship）
- AGREEMENT ROLE 记录正式协议/合同中的角色
- 两者可以并存（非正式关系存在 + 正式协议存在），也可以只有其一
- 协议角色示例：supplier、customer、licensee、licensor、contracting firm、employer、partner、legal council、approver、guarantor、entered by

**ADDENDUM**（补充协议/修正）：
- 关联 AGREEMENT 或 AGREEMENT ITEM
- 属性：`addendum creation date`、`addendum effective date`、`addendum text`
- 例如：时间延期补充协议，更新 AGREEMENT 的 thru date 同时记录 ADDENDUM 历史

#### 10.2 协议项（Agreement Items — 图4.13）

**AGREEMENT ITEM（协议项）**：
- 子类型：
  - **SUB AGREEMENT**（子协议）：协议中的协议，如总体专业服务协议中包含的保密协议
  - **AGREEMENT SECTION**（协议章节）：拆分出的章节，可关联特定组织、产品或地理区域
  - **AGREEMENT PRICING PROGRAM**（协议定价方案）：约定的产品价格
  - **AGREEMENT EXHIBIT**（协议附件）
- **递归关系**：子协议 → 章节 → 条款的多层组合
- 属性：`agreement text`（协议文本）、`agreement image`（图表）

**适用范围实体**：
- **AGREEMENT GEOGRAPHICAL APPLICABILITY**（地理适用范围）
- **AGREEMENT PRODUCT APPLICABILITY**（产品适用范围）
- **AGREEMENT ORGANIZATION APPLICABILITY**（组织适用范围）
- 使协议项可以为特定地理区域、产品或组织定制

#### 10.3 协议条款（Agreement Terms — 图4.14）

**AGREEMENT TERM**（协议条款）：
- 可用于 AGREEMENT 级别或 AGREEMENT ITEM 级别
- `term value` 的语义取决于 TERM TYPE
- 条款类型示例：
  - 法律条款（legal terms）
  - 财务条款（financial terms）
  - 激励条款（incentives）
  - 门槛条款（thresholds）
  - 续签条款（renewals）
  - 终止条款（agreement termination）
  - 赔偿条款（indemnification）
  - 竞业禁止条款（non-competition）
  - 排他性关系条款（exclusive relationship provisions）

#### 10.4 协议定价（Agreement Pricing — 图4.15）

**协议定价模型**：复用第3章的 PRICE COMPONENTS 模型

- AGREEMENT PRICING PROGRAM（AGREEMENT ITEM 子类型）关联多个 PRICE COMPONENT
- PRICE COMPONENT 可以是：BASE PRICE、DISCOUNT COMPONENT、SURCHARGE COMPONENT、MANUFACTURER SUGGESTED PRICE
- 定价可随 GEOGRAPHIC BOUNDARY、PRODUCT CATEGORY、QUANTITY BREAK、ORDER VALUE、SALE TYPE 变化
- 与产品定价的区别：协议定价针对特定 PARTY，不需要 PARTY TYPE 关系

**定价优先级（三条路径）**：
1. 标准产品价格（standard product price）
2. 预先签订的协议价格（agreement price，覆盖标准价格）
3. 订单层面的特定协商价格（negotiated order price，覆盖协议价格）

#### 10.5 协议与订单的关系（图4.16）

- 协议与订单之间通常没有直接的数据关系
- 订单处理程序通过比较以下信息来判断适用哪个协议：
  - 订单日期 vs 协议有效期（from date / thru date）
  - 下单方 vs 协议签约方
  - ORDER ROLEs vs AGREEMENT ROLEs
- 订单的单据受协议的定价和条款约束，但在特殊情况下允许覆盖（如因上次交期延误而给予诚意折扣）

---

## 关键数据模型/概念

### 核心概念

| 概念 | 英文名称 | 说明 |
|------|---------|------|
| 订单 | ORDER | 核心实体，子类型为 SALES ORDER 和 PURCHASE ORDER |
| 订单项 | ORDER ITEM | 表示订购的特定产品或服务，关联 PRODUCT |
| 产品特性 | PRODUCT FEATURE | 产品可选特性（颜色、尺寸等），通过递归关系与 ORDER ITEM 关联 |
| 订单角色 | ORDER ROLE | 参与方在订单中扮演的角色，由 ORDER ROLE TYPE 描述 |
| 订单联系方式 | ORDER CONTACT MECHANISM | 订单相关的联系信息，由 CONTACT MECHANISM PURPOSE TYPE 描述 |
| 订单调整 | ORDER ADJUSTMENT | 折扣、附加费、税、运费等非产品类调整 |
| 订单状态 | ORDER STATUS | 订单/订单项的状态历史跟踪 |
| 订单条款 | ORDER TERM | 订单/订单项的条款条件 |
| 订单项关联 | ORDER ITEM ASSOCIATION | 销售订单项与采购订单项的多对多关联 |
| 需求 | REQUIREMENT | 客户或内部的商品/工作需求 |
| 请求 | REQUEST | 向供应商发出的征求（RFP/RFQ/RFI） |
| 报价 | QUOTE | 对请求的响应（PROPOSAL / PRODUCT QUOTE） |
| 协议 | AGREEMENT | 双方/多方长期合作的正式约定 |
| 协议项 | AGREEMENT ITEM | 协议的组成部分（子协议、章节、定价方案、附件） |
| 补充协议 | ADDENDUM | 对协议或协议项的修改 |

### 主要数据模型图示

| 图号 | 内容 |
|------|------|
| 图4.1 | 标准订单模型（问题模型） |
| 图4.2 | 统一订单与订单项模型 |
| 图4.3 | 销售订单参与方与联系方式 |
| 图4.4 | 采购订单参与方与联系方式 |
| 图4.5 | 通用订单角色与联系方式 |
| 图4.6 | 订单调整 |
| 图4.7 | 订单状态与条款 |
| 图4.8 | 订单项关联 |
| 图4.9 | 需求管理 |
| 图4.10 | 请求管理 |
| 图4.11 | 报价管理 |
| 图4.12 | 协议定义 |
| 图4.13 | 协议项 |
| 图4.14 | 协议条款 |
| 图4.15 | 协议定价 |
| 图4.16 | 协议与订单的关系 |
| 图4.17 | 整体订单模型总览 |

### 整体订单流程

```
REQUIREMENT（需求）
    │
    ├──→ 直接 → ORDER（订单）
    │
    └──→ REQUEST（请求）→ QUOTE（报价）→ ORDER（订单）
                                              │
                                    ┌─────────┴─────────┐
                                    ↓                    ↓
                            Chapter 5: SHIPMENT    Chapter 6: WORK EFFORT
                            （商品交付）            （服务交付）

AGREEMENT（协议）—— 横向影响整个流程中的条款和定价
```

---

## 结论/要点

1. **销售订单和采购订单本质上是同一件事**，应从统一的数据结构出发设计，通过子类型（SALES ORDER / PURCHASE ORDER）来区分，从而最大化代码复用并消除冗余。

2. **订单建模必须超越"我方"视角**，明确记录所有参与方及其角色（下单方、收单方、收货方、开票方、安装方、销售人员、审批人等），同时提供通用化机制（ORDER ROLE + ORDER ROLE TYPE）以应对未来业务变化。

3. **联系方式与参与方应独立建模**，因为可能只知道一方而不知道另一方（如匿名网络订单），且订单专用的联系方式可能与参与方主数据中的默认联系方式不同。

4. **收货信息应定位于 ORDER ITEM 级别**，而非 ORDER 级别，因为同一订单的不同商品可能发往不同目的地，而发货（SHIPMENT）可将来自多个订单的订单项合并为一次配送。

5. **订单调整（折扣、附加费、税、费等）应作为独立实体**（ORDER ADJUSTMENT），而非 ORDER ITEM 的子类型，因为调整不是被"订购"的实物。

6. **订单状态应记录历史**（status datetime），但 shipped/completed/backordered/invoiced 等状态应从发货项、发票项等关系中推导，而非作为直接状态值。

7. **订单条款可同时作用于订单级别和订单项级别**，条款类型可灵活扩展。

8. **销售订单项与采购订单项之间存在多对多的关联**，这是实现"backorder"等业务场景的基础，同时也需要记录对应的采购订单号（corresponding PO ID）。

9. **可选模型（需求、请求、报价、协议）应根据企业实际需要决定是否采用**：
   - 专业服务类企业重视需求跟踪
   - 需要招标流程的企业需要请求和报价管理
   - 有长期合作关系的企业需要协议管理

10. **定价具有三级优先级**：产品标准价 < 协议价 < 订单协商价。协议通过 AGREEMENT PRICING PROGRAM 复用 PRICE COMPONENTS 模型来实现灵活定价。

11. **协议的通用模型设计**使其能够适应各种类型的协议（销售、采购、雇佣、合伙等），通过 AGREEMENT TYPE 和 AGREEMENT ROLE TYPE 灵活扩展，通过 AGREEMENT ITEM 和递归关系支持多层次的协议结构。

12. **第4章是承上启下的关键章节**：订单模型连接了第3章的产品模型，同时为第5章的发货模型（商品交付）和第6章的工作成果模型（服务交付）奠定了基础。
