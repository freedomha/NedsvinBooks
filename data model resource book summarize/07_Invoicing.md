# 第7章：Invoicing（发票/计费）

## 章节概述

本章提供了发票（Invoice）和发票行项目（Invoice Item）的全面数据模型，涵盖产品计费、产品特性计费、调整项、发货、工作成果、时间录入和订单的计费关系。此外还讨论了发票角色（Invoice Roles）、计费账户（Billing Account）、发票状态与条款（Invoice Status and Terms）、发票支付（Invoice Payments）以及金融账户的存款和取款（Financial Account Deposits and Withdrawals）。

**核心主题**：发票代表一种支付请求（request for payment），是连接订单、发货、服务交付和实际资金流动的关键枢纽。模型设计强调灵活性，能够同时处理销售发票（Sales Invoice）和采购发票（Purchase Invoice），支持多种计费来源（发货、工作成果、时间录入、订单）的任意组合。

---

## 核心问题（Problems）

### 问题1：发票行项目的灵活分类

企业需要为不同类型的费用开具发票，不仅包括产品本身，还包括：
- 产品特性（Product Feature）的附加费用——例如在同一产品上附加特殊光泽纸的特性
- 运费（Freight charges）
- 手续费（Handling charges）
- 销售税（Sales tax）
- 折扣调整（Discount adjustments）
- 附加费（Surcharge adjustments）
- 杂费（Miscellaneous charges）

传统做法是在 INVOICE 实体上添加各种属性（如 tax_amount, freight_charge 等），但这种方式缺乏灵活性——每当需要跟踪新的调整类型时，就必须修改表结构。

### 问题2：发票行项目之间的关联

当产品特性需要单独计费时，需要能够关联该特性费用与它所附加的产品费用。例如，客户购买了"Johnson fine grade 8 1/2 by 11 bond paper"，同时还购买了这个产品上的"extra glossy finish"特性。需要记录这个特性费用是在哪个产品上下文中的。

### 问题3：发票的参与方与角色

发票涉及多个参与方（Party），标准模型通常只记录客户地址和默认的企业开票方地址（"I"模型）。但在多地点、多公司的组织中，需要更灵活的结构来处理：
- 发票发送给谁（billed to）
- 发票由谁发出（billed from）
- 录入人（entered by）
- 审批人（approver）
- 发送人（sender）
- 接收人（receiver）

### 问题4：发票发送方式的多样性

在电子商务时代，发票可以通过多种方式发送和接收：
- 邮政地址（Postal Address）
- 电子邮件（Electronic Address）
- 传真（Telecommunications Number / Fax）

传统系统只能处理物理邮寄地址，无法适应电子发票的需求。

### 问题5：计费账户管理

某些行业（银行、信用卡公司、电信行业）需要能够将不同类型的费用分组到不同的账户中：
- 一个客户可能需要一个办公用品账户和一个家具购买账户
- 一个账户可能有多方负责支付（主要付款人、次要付款人）
- 需要跟踪各方在账户中的角色及有效期

### 问题6：发票来源的多样性

发票可以基于多种来源生成：
- 发货项（Shipment Items）——货物交付后需要付款
- 工作成果（Work Efforts）——按项目进度计费
- 时间录入（Time Entries）——按服务时间计费
- 订单项（Order Items）——基于订单直接计费

每种来源都存在多对多的关系：
- 一张发票行可能聚合多个发货项
- 一个发货项可能被多次计费（如部分计费或信用调整）

### 问题7：部分支付与支付追踪

- 一笔发票可能被多次部分支付
- 一笔支付可能同时支付多张发票
- 需要支持"按账户支付"（on account payment），即收款但不明确指定支付哪张发票
- 复杂的发票可能需要追踪到发票行项目级别的支付

### 问题8：发票状态管理

发票在生命周期中状态会发生变化，例如"已审批"、"已发送"、"已作废"。需要注意，"已支付"不应作为发票状态，因为它可以通过支付交易推断出来。

### 问题9：发票条款管理

发票可能有各种条款（Terms）：
- 付款期限（如：净30天）
- 滞纳金比例
- 催款罚金
- 某些行项目不可退款

这些条款可能适用于整张发票，也可能仅适用于特定的发票行项目。

### 问题10：资金流转管理

收到付款后，需要将款项存入金融账户（银行账户、投资账户等）。需要跟踪：
- 多笔收款（Receipt）如何组成一笔存款（Deposit）
- 付款（Disbursement）如何对应取款（Withdrawal）

---

## 解决方案（Solutions）

### 方案1：发票行项目统一模型（Invoice Item）

**核心设计**：所有开票内容——无论是产品、特性还是调整项——都统一使用 INVOICE ITEM 实体表示。

**关键实体**：
- **INVOICE**：发票头实体，包含发票编号和发票日期
- **INVOICE ITEM**：发票行项目实体，包含序号（sequence id）、数量、金额和应纳税标志
- **INVOICE ITEM TYPE**：发票行项目类型，描述行项目的性质

**设计要点**：
1. 产品信息（如计量单位）通过 PRODUCT 到 UNIT OF MEASURE 的关系获取（见第3章），不需要在发票模型中冗余存储
2. 扩展价格（extended price = 数量 × 单价）是可推导信息，不存储为属性
3. 应纳税标志（taxable flag）存储在发票行项目上，因为某项目是否应纳税取决于多种情况（发货来源地、目的地、购买方的纳税状态等），不能仅由项目类型决定
4. 具体的税务计算规则依赖于各地的法律法规，不包含在此模型中

**与订单调整的区别**：
在订单章节中，ORDER ADJUSTMENT 是独立实体，因为调整项（如税金、手续费）不是"订购的物品"。而在发票中，调整项是实际产生的费用，因此作为 INVOICE ITEM 的实例存储。

**调整项的处理**：
- INVOICE ITEM TYPE 可以包括各种调整类型：杂费（miscellaneous charge）、销售税（sales tax）、折扣调整（discount adjustment）、运费和手续费（shipping and handling charges）、附加费调整（surcharge adjustment）、费用（fee）
- 如果需要，可以在 INVOICE ITEM 上添加百分比属性（percentage）来存储调整比例（如销售税 0.07）

**递归关系的应用**：
INVOICE ITEM 支持递归关系（recursive relationship）来处理以下场景：

1. **产品特性与产品的关联**：特性费用可以通过递归关系关联到对应的产品发票行项目
2. **发票纠错**：如果原始发票行录入数量有误（如10件应为8件），可以创建一个后续发票行（数量为 -2），通过递归关系关联到原始行项目。这样做的好处是保留审计线索，而不是直接修改原始发票

**替代模型（Figure 7.1b）**：
提供更细化的子类型：
- **INVOICE ADJUSTMENT**：表示各种调整项的子类型
- **INVOICE ACQUIRING ITEM**：表示实际获得的物品（产品、特性、工作成果或时间）
- 使用具体的关系替代递归关系：INVOICE PRODUCT ITEM 到 INVOICE PRODUCT FEATURE ITEM，以及 INVOICE ACQUIRING ITEM 到 INVOICE ADJUSTMENT

### 方案2：发票角色模型（Invoice Roles）

**核心设计**：INVOICE 可以与任何 PARTY 建立"billed to"和"billed from"关系，同时支持通过 INVOICE ROLE 记录额外的角色。

**关键实体**：
- **INVOICE**：发票实体，通过 billed from PARTY 和 billed to PARTY 记录两个主要角色
- **INVOICE ROLE**：发票角色关联实体，连接 INVOICE、PARTY 和 INVOICE ROLE TYPE
- **INVOICE ROLE TYPE**：发票角色类型，如"entered by"、"approver"、"sender"、"receiver"
- **datetime**：记录角色执行的日期和时间

**发送/接收方式的灵活性**：
发票需要记录发送到哪个联系机制（CONTACT MECHANISM）以及从哪个联系机制发出。CONTACT MECHANISM 作为超类型，支持三种子类型：
- **POSTAL ADDRESS**：传统邮寄地址
- **ELECTRONIC ADDRESS**：电子地址（如电子邮件）
- **TELECOMMUNICATIONS NUMBER**：电信地址（如传真号码）

这使系统能够处理现代电子商务多种发票传递方式——从传统邮寄到电子邮件到传真。

**验证增强**：
PARTY CONTACT MECHANISM PURPOSE（见第2章）可用于额外的业务规则验证，例如确保发票关联的联系机制确实标记为"接收发票用途"。

### 方案3：计费账户（Billing Account）

**核心设计**：BILLING ACCOUNT 为特定类型的企业提供了除直接发送到 PARTY 之外的另一种计费方式。这是一种可选机制。

**关键实体**：
- **BILLING ACCOUNT**：计费账户实体，包含 from date（生效日期）、thru date（失效日期）和 description（描述）
- **BILLING ACCOUNT ROLE**：计费账户角色关联实体，连接 PARTY、BILLING ACCOUNT 和 BILLING ACCOUNT ROLE TYPE
- **BILLING ACCOUNT ROLE TYPE**：角色类型，如"primary payer"（主要付款人）、"secondary payer"（次要付款人）、"customer service representative"、"manager"、"sales representative"
- **from date** 和 **thru date**：记录各方在账户中的活跃时间段

**工作原理**：
- INVOICE 可以直接 billed to PARTY，也可以 billed to BILLING ACCOUNT
- BILLING ACCOUNT 最终必须关联到 CONTACT MECHANISM，因为所有发票最终都要发送到某个物理或电子位置
- 如果企业使用计费账户，第4章的订单模型也应该包含到 BILLING ACCOUNT 的关系

**扩展设计**：
- 可以为账户发行卡片（如银行卡、信用卡），通过添加 CARD 或 MEDIA 实体实现
- CARD 与 BILLING ACCOUNT 和 PARTY 的关系可以是一对多或多对多，取决于业务规则

**典型行业应用**：
- **银行业**和**信用卡公司**：允许客户将不同类型的费用分到不同账户
- **电信行业**：标准电话服务放在一个账户，专线网络服务放在另一个账户；电话卡可以关联到账户，允许客户将通话费计入账户

### 方案4：发票特定角色模型（Invoice Specific Roles）

**核心设计**：作为方案2的替代，直接关联到具体的角色实体，表达更明确的业务规则。

**关键实体**：
- **SALES INVOICE**：销售发票子类型
- **PURCHASE INVOICE**：采购发票子类型
- **BILL TO CUSTOMER**：客户角色（PARTY ROLE 的子类型）
- **INTERNAL ORGANIZATION**：内部组织角色（PARTY ROLE 的子类型）
- **SUPPLIER**：供应商角色（PARTY ROLE 的子类型）

**业务规则**：
- SALES INVOICE 必须 billed to BILL TO CUSTOMER 或 BILLING ACCOUNT
- SALES INVOICE 必须 billed from INTERNAL ORGANIZATION
- PURCHASE INVOICE 必须 billed to INTERNAL ORGANIZATION 或 BILLING ACCOUNT
- PURCHASE INVOICE 必须 billed from 一个且仅一个 SUPPLIER

**利弊分析**：
- **优势**：传达更具体的业务规则；清晰地表达了各方在销售和采购发票中需要扮演的角色
- **劣势**：灵活性较差，因为固定的业务规则可能随时间变化。例如，未来代理方（Agent）可能需要作为发票发送方
- **使用指南**：当实体间关系非常稳定、不太可能变化时，使用更具体的模型；如果需要灵活性以适应未来的变化，则使用通用模型

### 方案5：发票状态管理（Invoice Status）

**核心设计**：通过 INVOICE STATUS 关联实体追踪发票状态的时间序列变化。

**关键实体**：
- **INVOICE STATUS**：发票状态关联实体，连接 INVOICE 和 INVOICE STATUS TYPE
- **INVOICE STATUS TYPE**：状态类型（STATUS TYPE 的子类型），值如"sent"、"void"、"approved"
- **status date**：状态生效日期

**设计要点**：
- "Paid"（已支付）不是有效的发票状态，因为它可以通过支付交易推断出来
- 当前状态可通过查找最近日期的状态记录来确定
- 如果状态类型数量有限，可以考虑在物理模型中反规范化（denormalization），将 approved_date、sent_date、void_date 作为 INVOICE 实体的属性而非 INVOICE STATUS 的记录

### 方案6：发票条款管理（Invoice Terms）

**核心设计**：通过 INVOICE TERM 实体管理发票及行项目级别的条款。

**关键实体**：
- **INVOICE TERM**：发票条款实体，关联 INVOICE 或 INVOICE ITEM
- **TERM TYPE**：条款类型（如"Payment—net days"、"Late fee—percent"、"Penalty for collection agency—percent"）
- **term value**：条款值（如"30"表示30天付款期）
- **description**：条款描述

**条款层级**：
- 发票级别条款：适用于整张发票（如付款期限、滞纳金比例）
- 行项目级别条款：适用于特定行项目（如某行项目不可退款）

### 方案7：基于发货的计费（Billing for Shipment Items）

**核心设计**：通过 SHIPMENT ITEM BILLING 关联实体处理 INVOICE ITEM 与 SHIPMENT ITEM 的多对多关系。

**关键实体**：
- **SHIPMENT ITEM BILLING**：发货计费关联实体
- **INVOICE ITEM** → SHIPMENT ITEM BILLING → **SHIPMENT ITEM**

**两种计费模式**：
1. **一个发货项对应多个发票行**：例如原始发货1000件，客户发现10件损坏，随后发出信用发票行（数量-10）关联同一发货项，用于审计追踪
2. **多个发货项对应一个发票行**：例如三个批次发货的组件被合并为一个组装件计费行；或者三次不同日期的发货按预订协议合并为一张发票

**设计要点**：
- SHIPMENT INVOICE 实体上不存储数量属性；数量来自 INVOICE ITEM
- 如果企业需要部分计费场景，可以添加数量属性
- 并非所有发货都会产生计费记录（如内部转移发货）

### 方案8：基于工作成果和时间的计费（Billing for Work Efforts and Time Entries）

**核心设计**：服务可以通过工作成果（WORK EFFORT）或时间录入（TIME ENTRY）两种方式计费。

**关键实体**：
- **WORK EFFORT BILLING**：工作成果计费关联实体，连接 INVOICE ITEM 和 WORK EFFORT
- **TIME ENTRY BILLING**：时间录入计费关联实体，连接 INVOICE ITEM 和 TIME ENTRY
- **percentage** 属性：记录该发票行计费了工作成果的百分比

**工作成果计费场景**：
- **进度付款**：咨询公司可能按项目进度计费——启动时30%，初次交付30%，完成后30天40%
- **多工作合并**：律师事务所可能将三项不同的工作打包为固定费用，计为一张发票行
- 采用多对多关系而非递归关系的原因是：一个发票行可能聚合了多个时间录入（不同日期），如果需要纠正，需要知道具体关联到哪个时间录入

**时间录入计费场景**：
- **信用调整**：5小时咨询服务被投诉，发出2小时信用发票，导致一个时间录入对应两个发票行
- **多时间合并**：会计师周一2小时、周三3小时、周五4小时，合并为一个9小时发票行，附加时间表

### 方案9：基于订单的计费（Billing for Order Items）

**核心设计**：通过 ORDER ITEM BILLING 关联实体处理 ORDER ITEM 与 INVOICE ITEM 的多对多关系。

**关键实体**：
- **ORDER ITEM BILLING**：订单计费关联实体
- **amount** 属性：记录从订单项分配到发票行的金额
- **quantity** 属性：记录从订单项分配到发票行的数量

**典型场景**：
1. **一个订单项对应多个发票行**：采购订单价值$120,000/年，供应商按月开票$10,000，每次发票行对应同一订单项。通过 ORDER ITEM BILLING 可以追踪原始承诺中有多少已被计费，避免超额计费
2. **多个订单项对应一个发票行**：三张独立采购订单各40小时技术支持（共120小时），供应商第一张发票计费100小时，第二张计费剩余20小时

**设计意义**：
- 企业无法控制服务供应商如何开票，因此模型需要足够灵活以处理各种计费方式
- 支持销售和采购订单项的双向计费追踪

### 方案10：支付管理（Payments）

**核心设计**：通过 PAYMENT、PAYMENT APPLICATION 处理支付与发票的多对多关系。

**关键实体**：
- **PAYMENT**：支付实体，表示资金的转移
  - **RECEIPT**（收款）：企业内部组织收到的款项
  - **DISBURSEMENT**（付款）：企业内部组织发出的款项
- **PAYMENT APPLICATION**：支付应用关联实体，处理多对多关系
- **PAYMENT METHOD TYPE**：支付方式类型（electronic、cash、certified check、personal check、credit card）
- **payment ref num**：支付参考号（支票号、电子转账标识）
- **effective date**：生效日期
- **comment**：注释

**三种支付应用模式**：

1. **PAYMENT → INVOICE**（Figure 7.8a）：按发票级别追踪支付
   - 一笔支付可以应用于多张发票
   - 一张发票可以被多笔支付部分付款
   - 支持"按账户支付"——支付先应用于 BILLING ACCOUNT，以后再分配到具体发票

2. **PAYMENT → INVOICE ITEM**（Figure 7.8b）：按发票行项目级别追踪支付
   - 适用于复杂发票场景（如大型咨询公司的一张账单包含多名顾问费用）
   - 付款方可能只支付某些行项目
   - 银行业贷款场景：部分还款时按规则优先偿还费用、利息、本金

3. **混合模式**：支付可以应用于 INVOICE 或 INVOICE ITEM
   - 灵活性最高，但可能导致程序混乱

**内部组织间支付**：一个内部组织向另一个内部组织付款会产生两条记录——付款方记录 DISBURSEMENT，收款方记录 RECEIPT。

### 方案11：金融账户管理（Financial Accounts, Deposits, and Withdrawals）

**核心设计**：通过 FINANCIAL ACCOUNT、DEPOSIT、WITHDRAWAL 管理资金在金融账户中的流转。

**关键实体**：
- **FINANCIAL ACCOUNT**：金融账户实体
  - **BANK ACCOUNT**：银行账户子类型
  - **INVESTMENT ACCOUNT**：投资账户子类型
- **FINANCIAL ACCOUNT TYPE**：账户类型（如 checking account、savings account、IRA account、mutual fund account）
- **DEPOSIT**：存款（FINANCIAL ACCOUNT TRANSACTION 的子类型）
- **WITHDRAWAL**：取款（FINANCIAL ACCOUNT TRANSACTION 的子类型）
- **FINANCIAL ACCOUNT TRANSACTION**：金融账户交易超类型

**资金流动路径**：
- 多笔 RECEIPT → 组成一笔 DEPOSIT → 影响一个 FINANCIAL ACCOUNT
- 每一笔 DISBURSEMENT → 对应一笔 WITHDRAWAL → 影响一个 FINANCIAL ACCOUNT

---

## 关键数据模型/概念

### 整体模型结构（Figure 7.10 - Overall Invoice Model）

完整的第7章模型整合了以下所有概念，形成了一个连接产品计费、订单、发货、工作成果、时间、支付和金融账户的统一框架：

1. **INVOICE** —— 发票核心实体
   - 关联 PARTY（billed to / billed from）
   - 关联 BILLING ACCOUNT（可选）
   - 关联 CONTACT MECHANISM（发送/接收地址）
   - 关联 INVOICE STATUS（状态追踪）
   - 关联 INVOICE TERM（条款管理）

2. **INVOICE ITEM** —— 发票行项目
   - 关联 PRODUCT / PRODUCT FEATURE（产品计费）
   - 关联 SHIPMENT ITEM（通过 SHIPMENT ITEM BILLING）
   - 关联 WORK EFFORT（通过 WORK EFFORT BILLING）
   - 关联 TIME ENTRY（通过 TIME ENTRY BILLING）
   - 关联 ORDER ITEM（通过 ORDER ITEM BILLING）
   - 支持递归关系（行项目关联）
   - 关联 INVOICE TERM（行项目级别条款）

3. **PAYMENT** —— 支付
   - RECEIPT / DISBURSEMENT 子类型
   - 通过 PAYMENT APPLICATION 关联 INVOICE 或 INVOICE ITEM
   - 支持"按账户支付"（支付应用于 BILLING ACCOUNT）

4. **FINANCIAL ACCOUNT** —— 金融账户
   - DEPOSIT 聚合多笔 RECEIPT
   - WITHDRAWAL 对应单笔 DISBURSEMENT

### 关键设计原则

1. **灵活性优先**：所有计费内容统一使用 INVOICE ITEM 而非独立的实体类型，当需要新的调整类型时只需新增 INVOICE ITEM TYPE，无需改表
2. **多对多关联**：计数来源（发货、工作、时间、订单）与发票行之间采用多对多关系，适应复杂的现实业务场景
3. **可推导不存储**：扩展价格（extended price）、支付状态等信息是推导信息，不存储为属性，避免了数据冗余和不一致
4. **审计线索**：通过递归关系和关联实体实现纠错，而非直接修改原始记录
5. **通用与专用的平衡**：提供通用模型（PARTY 角色）和专用模型（BILL TO CUSTOMER / INTERNAL ORGANIZATION / SUPPLIER）两种选择，让企业根据业务稳定性来选择
6. **分层支付追蹤**：支持发票级别和行项目级别的支付追踪，企业可根据需求选择
7. **资金完整链条**：从支付到存款/取款到金融账户，形成完整的资金流动记录

### 关键实体一览

| 实体名 | 说明 | 所属Figure |
|--------|------|-----------|
| INVOICE | 发票实体（请求支付） | Fig 7.1a |
| INVOICE ITEM | 发票行项目（每种费用） | Fig 7.1a |
| INVOICE ITEM TYPE | 行项目类型（产品/调整/税等） | Fig 7.1a |
| INVOICE ADJUSTMENT | 发票调整子类型 | Fig 7.1b (替代) |
| INVOICE ACQUIRING ITEM | 获得物项子类型 | Fig 7.1b (替代) |
| INVOICE PRODUCT ITEM | 产品行项目 | Fig 7.1b (替代) |
| INVOICE PRODUCT FEATURE ITEM | 产品特性行项目 | Fig 7.1b (替代) |
| INVOICE ROLE | 发票附加角色 | Fig 7.2 |
| INVOICE ROLE TYPE | 角色类型 | Fig 7.2 |
| BILLING ACCOUNT | 计费账户 | Fig 7.3a |
| BILLING ACCOUNT ROLE | 账户角色 | Fig 7.3a |
| BILLING ACCOUNT ROLE TYPE | 账户角色类型 | Fig 7.3a |
| BILL TO CUSTOMER | 客户角色(Party Role子类型) | Fig 7.3b |
| INTERNAL ORGANIZATION | 内部组织角色(Party Role子类型) | Fig 7.3b |
| SUPPLIER | 供应商角色(Party Role子类型) | Fig 7.3b |
| SALES INVOICE | 销售发票子类型 | Fig 7.3b |
| PURCHASE INVOICE | 采购发票子类型 | Fig 7.3b |
| INVOICE STATUS | 发票状态（时间线） | Fig 7.4 |
| INVOICE STATUS TYPE | 状态类型 | Fig 7.4 |
| INVOICE TERM | 发票条款 | Fig 7.4 |
| SHIPMENT ITEM BILLING | 发货计费关联 | Fig 7.5 |
| WORK EFFORT BILLING | 工作成果计费关联 | Fig 7.6 |
| TIME ENTRY BILLING | 时间录入计费关联 | Fig 7.6 |
| ORDER ITEM BILLING | 订单计费关联 | Fig 7.7 |
| PAYMENT | 支付实体 | Fig 7.8a |
| RECEIPT | 收款子类型 | Fig 7.8a |
| DISBURSEMENT | 付款子类型 | Fig 7.8a |
| PAYMENT APPLICATION | 支付应用关联 | Fig 7.8a/7.8b |
| PAYMENT METHOD TYPE | 支付方式类型 | Fig 7.8a |
| FINANCIAL ACCOUNT | 金融账户 | Fig 7.9 |
| BANK ACCOUNT | 银行账户子类型 | Fig 7.9 |
| INVESTMENT ACCOUNT | 投资账户子类型 | Fig 7.9 |
| FINANCIAL ACCOUNT TYPE | 金融账户类型 | Fig 7.9 |
| FINANCIAL ACCOUNT TRANSACTION | 金融账户交易 | Fig 7.9 |
| DEPOSIT | 存款子类型 | Fig 7.9 |
| WITHDRAWAL | 取款子类型 | Fig 7.9 |

### 与其他章节的关联

- **第2章（Parties）**：PARTY、PARTY ROLE、CONTACT MECHANISM、PARTY CONTACT MECHANISM PURPOSE
- **第3章（Products）**：PRODUCT、PRODUCT FEATURE、UNIT OF MEASURE
- **第4章（Orders）**：ORDER ITEM、ORDER ADJUSTMENT、BILLING ACCOUNT 也应加入订单模型
- **第5章（Shipments）**：SHIPMENT ITEM
- **第6章（Work Efforts）**：WORK EFFORT、TIME ENTRY
- **第8章（Accounting）**：INVOICE 和 PAYMENT 将作为会计交易的来源

---

## 结论/要点

### 核心收获

1. **UNIFIED INVOICE ITEM 设计**：所有计费内容（产品、特性、调整项、税、运费、手续费）统一处理为 INVOICE ITEM，由 INVOICE ITEM TYPE 区分。这种设计比在 INVOICE 实体上添加属性字段要灵活得多，任何新的调整类型只需新增 INVOICE ITEM TYPE 即可，无需修改数据库结构。

2. **MULTI-SOURCE BILLING**：发票行项目可以来源于四种不同的业务交易——Shipment Items（实物交付）、Work Efforts（工作成果）、Time Entries（服务时间）和 Order Items（直接订单计费）。每种来源都通过专门的关联实体（SHIPMENT ITEM BILLING、WORK EFFORT BILLING、TIME ENTRY BILLING、ORDER ITEM BILLING）管理与 INVOICE ITEM 的多对多关系。

3. **ADJUSTMENT AS ITEM**：发票调整项（税、运费、手续费、折扣、附加费）作为 INVOICE ITEM 的实例存储，而非独立的实体。这与订单模型（ORDER ADJUSTMENT 为独立实体）的设计理念不同，因为发票行代表的是"实际产生的费用"而订单不请求调整项。

4. **CORRECTIVE INVOICE ITEMS**：发票纠错通过创建反向（负数）发票行并关联到原始行项目来实现，而非直接修改原始发票。这种审计友好的设计是成熟企业系统的标志。

5. **FLEXIBLE ROLE MODELS**：提供两种发票参与方模型——通用的 PARTY 角色模型（更灵活）和专用的角色实体模型（业务规则更明确）。选择指南：业务关系稳定时用专用模型，需要灵活应对未来变化时用通用模型。

6. **MULTI-CHANNEL DELIVERY**：发票发送/接收可通过多种 CONTACT MECHANISM（邮政地址、电子邮件、传真），适应电子商务环境。在这个层面使用超类型设计是前瞻性的。

7. **BILLING ACCOUNT 可选扩展**：计费账户为银行、信用卡、电信等特定行业提供分组计费能力，支持多个付款方共同负责一个账户。

8. **LAYERED PAYMENT TRACKING**：支付追踪可以在发票级别（PAYMENT → INVOICE）或行项目级别（PAYMENT → INVOICE ITEM）进行。行项目级别提供更精确的追踪，但增加数据量和维护工作。

9. **COMPLETE MONEY FLOW**：模型覆盖了从支付到金融账户交易的完整资金链条——RECEIPT → DEPOSIT → FINANCIAL ACCOUNT 和 DISBURSEMENT → WITHDRAWAL → FINANCIAL ACCOUNT。

10. **DERIVED NOT STORED**：坚持"可推导信息不存储"原则——扩展价格（计算值）、支付状态（可通过交易推断）均不存储为属性，保证数据一致性。

### 章节位置与模型索引

本章（第7章 Invoicing）位于原书第233-258页，共包含以下数据模型图示：
- **Figure 7.1a**：Invoice and Invoice Items（发票与发票行项目）——基本模型
- **Figure 7.1b**：Invoice and Invoice Items—alternate model（替代模型，含子类型）
- **Figure 7.2**：Invoice Parties（发票参与方）
- **Figure 7.3a**：Billing Account（计费账户）
- **Figure 7.3b**：Invoice Specific Party Roles（发票特定参与方角色）
- **Figure 7.4**：Invoice Status and Terms（发票状态与条款）
- **Figure 7.5**：Billing for Shipment Items（发货项计费）
- **Figure 7.6**：Billing of Time Entries and Work Efforts（时间录入与工作成果计费）
- **Figure 7.7**：Billing for Order Items（订单项计费）
- **Figure 7.8a**：Invoice Payments（发票支付——发票级别）
- **Figure 7.8b**：Invoice Item Payments（发票行项目支付）
- **Figure 7.9**：Financial Accounts, Withdrawals, and Deposits（金融账户、取款与存款）
- **Figure 7.10**：Overall Invoice Model（整体发票模型——全部实体关系概览）
