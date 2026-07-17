# 第5章：Shipments（运输/发货）

## 章节概述

本章详细讨论了 Shipments（运输/发货）的��据建模，涵盖了运输的完整生命周期，包括运输类型定义、运输参与方和联系方式、运输明细、运输状态跟踪、运输与订单的关系、收货确认、出库拣货（Item Issuance）、运输文档、运输路线规划以及运输车辆管理。本章的核心是将运输流程中的各个环节抽象为可复用的数据实体，支持发货、收货、退货、调拨等多种业务场景，并与订单、库存、参与方等数据模型紧密集成。

本章涉及的实体和关系最终在图 5.8 的总览模型中展示。实施这些模型后，可以最大限度地减少冗余数据，并使参照完整性（referential integrity）维护更加简单。

## 核心问题（Problems）

1. **运输类型多样化**：企业需要处理多种运输场景——向客户发货（Customer Shipment）、从供应商收货（Purchase Shipment）、客户退货（Customer Return）、向供应商退货（Purchase Return）、内部调拨（Transfer）以及直发（Drop Shipment）。

2. **运输参与方和目的地的复杂性**：一次运输涉及发货方（shipped from party）、收货方（shipped to party）、发货地址（shipped from address）、目的地地址（shipped to address），还需要联系方式用于收货确认和运输查询。

3. **订单与运输的多对多关系**：一张订单可能分多次发货（partial shipments），一次发货也可能包含多张订单的商品（combined shipments），需要追踪每一次运输具体发货了哪些商品及其数量。

4. **运输状态生命周期管理**：运输状态会随时间变化（scheduled → shipped → in route → delivered → canceled），需要记录各状态的时间节点。

5. **收货确认与质量管理**：入库运输需要记录接收了多少、拒收了多少，拒收原因是什么，以及谁来验收。

6. **出库拣货流程**：发货前需要生成拣货单（Picklist），记录从哪些库存位置（Inventory Item）拣取了多少商品，以及拣货参与者。

7. **运输文档管理**：不同运输场景需要不同类型的文档——提货单（Bill of Lading）、装箱单（Packaging Slip）、出口文档（Export Document）、运单（Manifest）、危险品文档（Hazardous Materials Document）等。

8. **运输路线追踪**：运输可能需要经过多个路线段（Route Segment），使用不同的运输方式（Method Type），由不同的承运人（Carrier）完成。

9. **自有车队管理**：使用自有车队的企业需要追踪车辆信息、里程、油耗等，以便计算运输成本。

## 解决方案（Solutions）

### 1. 运输类型（Shipment Types）

通过 SHIPMENT 实体的子类型来区分不同的运输类型：

- **OUTGOING SHIPMENT**（外向运输）：内部组织发往外部
  - **CUSTOMER SHIPMENT**（客户发货）：向客户发货
  - **PURCHASE RETURN**（采购退货）：退回给供应商
- **INCOMING SHIPMENT**（内向运输）：外部组织发往内部
  - **PURCHASE SHIPMENT**（采购收货）：从供应商收货
  - **CUSTOMER RETURN**（客户退货）：客户退货
- **TRANSFER**（调拨）：内部组织之间的运输
- **DROP SHIPMENT**（直发）：从一个外部组织直接发往另一个外部组织

### 2. 参与方和联系方式

- SHIPMENT 与 POSTAL ADDRESS 之间存在两重关系：**发货来源地址**和**发货目的地地址**
- SHIPMENT 与 PARTY 之间有 **shipped to party** 和 **shipped from party** 两重关系
- SHIPMENT 与 CONTACT MECHANISM 之间有**收货联系号码**和**运输查询联系方式**
- 虽然这些信息在 ORDER 中也存在，但需要在 SHIPMENT 中冗余存储，因为：
  - 运输记录可能在订单创建很久之后才生成
  - 运输过程中的信息可能会被覆盖或更改

### 3. 运输明细（Shipping Detail）

- **SHIPMENT ITEM** 实体记录每次运输中的每个商品项，包含 quantity 属性
- SHIPMENT ITEM 关联到 **GOOD**（标准商品）而非 INVENTORY ITEM，因为：
  - 安排运输时可能还不知道具体的库存项（inventory item）
  - INVENTORY ITEM 通过 SHIPMENT RECEIPT 和 ITEM ISSUANCE 间接关联
- **SHIPMENT CONTENTS DESCRIPTION** 用于描述非标准商品（一次性采购）
- **SHIPMENT STATUS** 实体记录运输状态生命周期，关联 **SHIPMENT STATUS TYPE**，status_date 记录状态变更时间
- SHIPMENT ITEM 的递归关系用于追踪退货关联——例如收到商品后发现有缺陷，退回时产生关联

### 4. 运输-订单关系（Shipment-to-Order）

- **ORDER SHIPMENT** 是 SHIPMENT ITEM 和 ORDER ITEM 之间的关联实体（associative entity），解析多对多关系
- 支持**部分发货**（一个 ORDER ITEM 拆分到多个 SHIPMENT ITEM）和**合并发货**（多个 ORDER ITEM 合并到一个 SHIPMENT ITEM）
- **SHIPMENT ITEM FEATURE** 关联实体用于记录运输商品的具体特征（Product Features），独立于订货特征，因为：
  - 可能存在替代品（substitution），例如订了蓝色笔却发了黑色笔
  - 运输中可以独立于订单存在（如内部调拨）

### 5. 收货确认（Shipment Receipts）

- **SHIPMENT PACKAGE** 表示运输包裹（可能带有条形码追踪号）
- **PACKAGING CONTENT** 关联实体连接 SHIPMENT PACKAGE 和 SHIPMENT ITEM，记录每个包裹中各商品的数量
- **SHIPMENT RECEIPT** 记录收货详情：
  - datetime_received：收货时间
  - quantity_accepted：接受数量
  - quantity_rejected：拒收数量
  - item_description：非标准商品描述
- **REJECTION REASON** 记录拒收原因
- **SHIPMENT RECEIPT ROLE** 记录收货相关人员角色（签收人、质检员、入库人员、收货经理等）
- SHIPMENT RECEIPT 关联到 SHIPMENT PACKAGE（而非 SHIPMENT ITEM），因为物理上收到的是包裹，而非逻辑上的商品项

### 6. 出库拣货（Item Issuance）

- **PICKLIST** 和 **PICKLIST ITEM** 记录拣货计划：从哪些 INVENTORY ITEM 中拣取多少数量
- **ITEM ISSUANCE** 记录实际的出库操作，每个 ITEM ISSUANCE 从一个 INVENTORY ITEM 中取出商品，关联到一个 SHIPMENT ITEM
- 一个 SHIPMENT ITEM 可能关联多个 ITEM ISSUANCE（例如库存不足，分两次拣货）
- ITEM ISSUANCE 与 SHIPMENT ITEM 的关系是**可选的**，因为商品也可能为非运输目的出库（如企业内部使用）
- **ITEM ISSUANCE ROLE** 记录拣货参与者的角色

### 7. 运输文档（Shipment Documents）

- **SHIPMENT DOCUMENT** 是 **DOCUMENT** 的子类型
- 子类型包括：BILL OF LADING（提货单）、PACKAGING SLIP（装箱单）、EXPORT DOCUMENT（出口文档）、MANIFEST（运单）、PORT CHARGES DOCUMENT（港口费用单）、TAX AND TARIFF DOCUMENT（税费单）、HAZARDOUS MATERIALS DOCUMENT（危险品单）、OTHER SHIPPING DOCUMENT
- SHIPMENT DOCUMENT 可以关联到 SHIPMENT（整批运输）、SHIPMENT ITEM（运输项）或 SHIPMENT PACKAGE（包裹），取决于文档类型

### 8. 运输路线（Shipment Routing）

- **SHIPMENT ROUTE SEGMENT** 记录运输路线的每一段
- 每个 ROUTE SEGMENT 关联到：
  - **SHIPMENT METHOD TYPE**：运输方式（陆运、铁路、空运、海运等）
  - **CARRIER**：承运人（可能是内部组织或外部承运商）
  - **VEHICLE**：运输车辆（可选，适用于自有车队）
  - **FACILITY**：途经设施（可选，适用于需要追踪每个经停点的场景）
- 同一次运输中不同包裹走不同路线，建议视为两次不同的运输（这样更易于追踪和管理）

### 9. 运输车辆（Shipment Vehicle）

- **VEHICLE** 是 **FIXED ASSET** 的子类型（详见第8章）
- 追踪属性包括：start_mileage、end_mileage、fuel_used
- 通过 estimated_start_datetime 和 estimated_arrival_datetime 与实际时间的对比，可判断运输是否按计划进行
- 一辆车可以运送多次运输（one-to-many from VEHICLE to SHIPMENT ROUTE SEGMENT）

## 关键数据模型/概念

### 核心实体清单

| 实体 | 说明 |
|------|------|
| SHIPMENT | 运输主体，含子类型（Customer Shipment、Purchase Shipment、Customer Return、Purchase Return、Transfer、Drop Shipment） |
| SHIPMENT ITEM | 运输明细项，记录运输了什么商品、多少数量 |
| SHIPMENT STATUS | 运输状态（scheduled、shipped、in route、delivered、canceled） |
| SHIPMENT STATUS TYPE | 运输状态类型定义 |
| ORDER SHIPMENT | ORDER ITEM 与 SHIPMENT ITEM 的多对多关联 |
| SHIPMENT ITEM FEATURE | 运输商品的特征信息 |
| SHIPMENT PACKAGE | 运输包裹 |
| PACKAGING CONTENT | 包裹内容明细 |
| SHIPMENT RECEIPT | 收货记录 |
| REJECTION REASON | 拒收原因 |
| SHIPMENT RECEIPT ROLE | 收货参与者角色 |
| PICKLIST | 拣货单 |
| PICKLIST ITEM | 拣货单明细 |
| ITEM ISSUANCE | 出库操作记录 |
| ITEM ISSUANCE ROLE | 出库参与者角色 |
| SHIPMENT DOCUMENT | 运输文档（子类型：Bill of Lading、Packaging Slip、Export Document、Manifest 等） |
| SHIPMENT ROUTE SEGMENT | 运输路线段 |
| SHIPMENT METHOD TYPE | 运输方式类型 |
| CARRIER | 承运人 |
| VEHICLE | 运输车辆 |

### 关键关系

- **SHIPMENT ↔ ORDER**：通过 ORDER SHIPMENT 实现 SHIPMENT ITEM 与 ORDER ITEM 的多对多映射，支撑部分发货和合并发货
- **SHIPMENT ITEM ↔ GOOD**：运输项关联标准商品（非 INVENTORY ITEM，后者通过收货/出库间接关联）
- **SHIPMENT ITEM ↔ INVENTORY ITEM**：通过 SHIPMENT RECEIPT（入库方向）和 ITEM ISSUANCE（出库方向）间接关联
- **SHIPMENT ↔ POSTAL ADDRESS**：发货来源地址和目的地地址
- **SHIPMENT ↔ PARTY**：发货方和收货方
- **SHIPMENT ROUTE SEGMENT ↔ VEHICLE**：车辆与路线段的关联，支撑运输成本计算
- **SHIPMENT ITEM 递归关系**：用于追踪退货与原始运输的关联

### 重要设计决策

1. **为什么 SHIPMENT ITEM 关联 GOOD 而非 INVENTORY ITEM？**
   - 安排运输时可能还不知道具体库存项
   - INVENTORY ITEM 通过 SHIPMENT RECEIPT 和 ITEM ISSUANCE 间接关联

2. **为什么在 SHIPMENT 中冗余存储 PARTY 和 CONTACT MECHANISM（这些信息在 ORDER 中已存在）？**
   - 运输记录可能不在下单时立即创建
   - 运输信息可能在过程中被覆盖或修改

3. **为什么 SHIPMENT RECEIPT 关联 SHIPMENT PACKAGE 而非 SHIPMENT ITEM？**
   - 物理上收到的是包裹，需要对比包裹内容与预期商品数量

4. **为什么同一运输的不同包裹不建议走不同路线？**
   - 不同路线应视为不同运输，便于追踪和管理，即使它们属于同一订单

## 结论/要点

1. SHIPMENT 模型提供了从**运输计划 → 拣货出库 → 运输路线追踪 → 收货确认**的完整数据链路
2. 通过子类型设计，一套数据模型可以支撑所有运输场景（发货、收货、退货、调拨、直发）
3. 多对多关系（ORDER SHIPMENT）是模型中最为复杂但最核心的部分，它支撑了真实世界中订单与运输之间的灵活映射
4. 运输文档模型具有良好的扩展性，通过 DOCUMENT TYPE 可以支持企业自定义文档类型
5. 运输路线段（SHIPMENT ROUTE SEGMENT）的设计允许精细追踪每一次运输的完整旅程，同时保持灵活（可选关联车辆、设施等）
6. 本章模型与 ORDER（第3-4章）、INVENTORY（后续章节）、PARTY（第2章）、FIXED ASSET（第8章）等模型紧密集成，共同构成企业级数据模型体系
7. 服务（Service）的交付与商品运输不同——服务无法通过传统方式"运输"，而是通过 WORK EFFORT（工作努力）实体进行交付，这将在第6章详细讨论
