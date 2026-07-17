# 第3章：Products（产品）

## 章节概述

本章聚焦于**产品（Products）**的数据模型，产品包括**商品（Goods）**和**服务（Services）**两大类。Goods 是有形的物理商品，Services 是专业人员的时间与专业知识的销售（如咨询服务、法律服务等）。本章的数据模型覆盖企业自身产品、供应商产品以及竞争对手产品的信息需求，具体包括：产品定义、产品分类、产品标识码、产品特性（Features）、计量单位、供应商与制造商、库存物品存储、产品定价、产品成本核算以及产品间关联关系。

---

## 核心问题（Problems）

### 1. 产品定义与分类问题

- 产品既可以是有形商品也可以是无形的服务，需要一个统一的模型同时容纳两者。
- 产品需要按多种维度分类（产品线、型号、品级、行业细分等），不同分类可能服务于不同的业务目的（目录组织、销售分析、清单分组等）。
- 产品可能同时属于多个类别（如磁盘既属于"办公用品"又属于"计算机耗材"），这在聚合分析时会导致重复计数，造成销售报告的夸大。
- 产品分类会随时间变化，需要记录时间有效性。

### 2. 产品标识码多样性问题

- 同一商品可能有多个标准标识码：制造商编号（Manufacturer's ID）、SKU（库存单位）、UPCA（美国通用产品码）、UPCE（欧洲通用产品码）、ISBN（国际标准书号）等。
- 需要支持一个商品对应多个不同类型的标识码。

### 3. 产品特性（Features）与定制化问题

- 产品特性（也可称为特征、选项、变体、修饰符）的可变性：同一特性在某个产品中可能是定义的一部分（required），在另一个产品中则可能是一个可选选项（optional）。
- 某些特性的适用性依赖于其他已选特性（例如笔记本电脑选择了内置DVD就不能同时选择内置可擦写CD-ROM）。
- 产品特性涉及多种类型：质量品级（Quality）、颜色（Color）、尺寸（Dimension）、大小（Size）、品牌名称（Brand Name）、软件特性（Software Feature）、硬件特性（Hardware Feature）、计费特性（Billing Feature）等。不同类型的企业可能需要不同的特性子类型。

### 4. 计量单位与转换问题

- 同一类型的产品可能以不同的计量单位出售（如"每支"、"小盒"、"大盒"），需要通过统一的计量单位来计算总库存、成本和销售额。
- 流程制造业涉及升、加仑、吨等多种度量衡，需要支持度量单位之间的换算（如夸脱与加仑之间的4倍转换因子）。

### 5. 供应商与制造商管理问题

- 同一产品可能由多个供应商提供，需要记录每个供应商的供货时间范围、优先级（Preference Type）、评级（Rating Type）和标准交货时间（Lead Time）。
- 再订购指南（Reorder Guideline）可能因地理区域、设施或内部组织的不同而不同。

### 6. 库存物品存储问题

- 需要区分目录商品（GOOD）和物理库存物品（INVENTORY ITEM）：前者是标准化的可购买项目，后者是位于特定位置的具体实物。
- 需要支持序列化库存物品（Serialized Inventory Item，每个物品单独跟踪序列号）和非序列化库存物品（Non-Serialized Inventory Item，按组跟踪、维护现有数量）。
- 需要通过批次（Lot）跟踪库存来源（生产批次或供应商发货批次），以便在需要召回时进行追溯。
- 库存物品可能在设施（Facility）层面或更细粒度的容器（Container，如货架、抽屉、箱子）层面跟踪。
- 需要维护库存物品状态（Status Type），如"良好"、"待修"、"轻度损坏"、"缺陷"、"报废"。
- 需要记录盘点差异（Inventory Item Variance），包括差异数量、发现日期、原因和备注，作为审计追踪。

### 7. 产品定价的复杂性问题

- 定价由多个组件（Price Component）构成：基础价格（Base Price）、折扣组件（Discount Component）、附加费组件（Surcharge Component）、制造商建议价（Manufacturer's Suggested Price）。
- 费用类型多样化：一次性收费（One Time Charge）、周期性收费（Recurring Charge，基于时间频率）、使用量收费（Utilization Charge，基于使用量）。
- 定价可能受多种因素组合影响：地理区域（Geographic Boundary）、参与方类型（Party Type）、产品类别（Product Category）、数量折扣（Quantity Break）、订单价值（Order Value）、销售类型（Sale Type）。
- 不同货币的国际定价需求。
- 产品特性（Features）可能独立定价，也可能与产品关联定价。
- 同一产品对不同组织（包括竞争对手、供应商）可能有不同价格。

### 8. 产品成本核算问题

- 实际产品成本散布在多个数据实体中（采购订单、货运、工时表、设备排产记录、管理费用等），难以统一获取。
- 需要支持估计成本（Estimated Cost），以便产品分析师预测未来成本趋势，而不仅依赖历史数据。
- 成本组件包括：估计材料成本（Estimated Materials Cost）、估计人工成本（Estimated Labor Cost）、估计其他成本（Estimated Other Cost，如制造设备使用费、损耗费、运输费、销售佣金、行政管理费）。
- 成本可能随时间、地理位置和组织的不同而变化。

### 9. 产品间关联关系问题

- 产品组件化（Product Component）：一个产品由其他多个产品组成（如办公文具套装包含笔、铅笔、日历等），组件可能随时间变化。
- 产品替代（Product Substitute）：哪些产品可以替代其他产品，替代可能涉及数量关系。
- 产品淘汰（Product Obsolescence）：产品何时被新版本取代。
- 产品互补（Product Complement）：哪些产品适合配合使用（如桌垫和桌垫替换纸）。
- 产品不兼容（Product Incompatibility）：哪些产品不能与其他产品一起使用。

### 10. 产品与零部件的区分问题

- 办公用品、备件、原材料与主流对外销售产品的区别处理。
- 存在两种建模思路：将 Good 子类化为 Finished Good / Raw Material / Subassembly；或者将 Part（物理存在的实际物品）与 Product（市场销售的包装品）分离建模。

---

## 解决方案（Solutions）

### 1. 产品定义与分类模型（Figure 3.1, 3.2）

- **PRODUCT** 作为超类型实体，通过 `product_subtype` 属性区分 Good 和 Service。
- **PRODUCT CATEGORY** 实体用于对产品进行灵活分组。通过关联实体 **PRODUCT CATEGORY CLASSIFICATION** 实现产品与类别的多对多关系，包含 `from_date` 和 `thru_date` 属性以跟踪时间有效性。
- **PRODUCT CATEGORY ROLLUP** 递归实体实现产品类别的层级结构（一个类别可以包含多个子类别，一个子类别可以属于多个父类别）。
- 使用 `primary_flag` 属性标记主分类，避免多分类导致的分析重复计数问题。若主分类因应用场景不同而变化，可引入 **PRODUCT CLASSIFICATION TYPE** 实体提供更灵活的方案。
- **MARKET INTEREST** 实体链接 PARTY TYPE 和 PRODUCT CATEGORY，帮助企业记录特定类型的参与方对哪些产品类别感兴趣，支持销售预测和目标客户发掘。同样包含 `from_date` 和 `thru_date` 时间属性。

### 2. 产品标识码模型（Figure 3.3）

- **GOOD IDENTIFICATION** 实体存储商品的各类标识码值（`id_value`），作为超类型。
- 子类型包括：
  - **MANUFACTURER'S ID NO**（制造商编号）
  - **SKU**（库存单位）
  - **UPCA**（美国通用产品码）
  - **UPCE**（欧洲通用产品码）
  - **ISBN**（国际标准书号）
- 一个 Good 可以拥有多个标识码。

### 3. 产品特性模型（Figure 3.4）

- **PRODUCT FEATURE** 实体统一定义产品的所有可变特性。同一特性在不同产品中可以是必需特征、标准特征或可选特征，因此不需要分离成两个实体。
- **PRODUCT FEATURE APPLICABILITY** 关联实体维护哪些产品可以使用哪些特性，子类型包括：
  - **STANDARD FEATURE**（标准特征，可取消选择）
  - **REQUIRED FEATURE**（必需特征，不可取消）
  - **SELECTABLE FEATURE**（必选特征，如颜色）
  - **OPTIONAL FEATURE**（可选特征）
- **PRODUCT FEATURE INTERACTION** 实体存储特性之间的交互规则：
  - **SELECTION INTERACTION INCOMPATIBILITY**（选择互斥）
  - **FEATURE INTERACTION DEPENDENCY**（特性依赖）
- PRODUCT FEATURE 的子类型可按企业定制：
  - **PRODUCT QUALITY**（品级，如A级/B级，专家级/初级）
  - **COLOR**（颜色）
  - **DIMENSION**（尺寸，关联 Unit of Measure）
  - **SIZE**（大小，如XL/L/M/S，服装行业常用）
  - **BRAND NAME**（品牌名称，可能与制造商不同）
  - **SOFTWARE FEATURE**（软件特性）
  - **HARDWARE FEATURE**（硬件特性）
  - **BILLING FEATURE**（计费特性，如月付/季付）

### 4. 计量单位模型（Figure 3.4）

- **UNIT OF MEASURE** 实体定义产品的计量方式，可定义产品如何库存和销售（如按"箱"或"个"），对于服务产品则是"小时"或"天"。
- **UNIT OF MEASURE CONVERSION** 关联实体实现多对多的递归关系，通过 `conversion_factor` 实现不同计量单位之间的换算（如1小盒=12支，1大盒=24支；1加仑=4夸脱）。这使得企业能够用通用计量单位来计算总库存。
- Unit of Measure 不作为 Product Feature 的子类型，因为以不同单位出售的同一产品实质上是不同的产品。

### 5. 供应商与制造商模型（Figure 3.5）

- **SUPPLIER PRODUCT** 关联实体记录哪个组织（供应商）提供哪个产品，包含：
  - `available_from_date` 和 `available_thru_date`（供货时间范围）
  - `standard_lead_time`（标准交货时间）
- **PREFERENCE TYPE** 实体跟踪供应商优先级（第一选择、第二选择等）。
- **RATING TYPE** 实体对每个供应商产品的整体表现进行评级。
- **REORDER GUIDELINE** 实体定义再订购指南，仅适用于 Good（服务一般不按此方式再订购）：
  - `reorder_level`（再订购触发数量）
  - `reorder_quantity`（推荐订购数量）
  - 可按 **GEOGRAPHIC BOUNDARY**、**FACILITY**、**INTERNAL ORGANIZATION** 分别定义。
- 一个产品由一个制造商（MANUFACTURER）制造。即使制造商将生产分包给其他组织，原始制造商仍被视为制造方。

### 6. 库存物品存储模型（Figure 3.6）

- **INVENTORY ITEM** 实体表示物理库存物品，与 GOOD（目录商品）区分。子类型包括：
  - **SERIALIZED INVENTORY ITEM**（序列化库存，跟踪每个物品的序列号）
  - **NON-SERIALIZED INVENTORY ITEM**（非序列化库存，跟踪数量 `quantity_on_hand`）
- **LOT** 实体表示生产批次，INVENTORY ITEM 属于一个 LOT（多对一），支持问题追溯和召回。
- **FACILITY** 实体表示仓库、厂房等物理结构；**CONTAINER** 实体表示更细粒度的位置（货架、文件抽屉、箱子、桶、房间等），通过 **CONTAINER TYPE** 定义容器类型。
- **INVENTORY ITEM STATUS TYPE** 维护物品状态（良好/待修/损坏/缺陷/报废）。如需历史状态跟踪，可增加关联实体 **INVENTORY ITEM STATUS** 并附加状态日期。
- **ITEM VARIANCE** 实体记录盘点差异：
  - `physical_inventory_date`（盘点日期）
  - `quantity`（差异数量，正数为多出，负数为短少）
  - **REASON** 实体提供标准差异原因（偷盗、损耗、不明差异、损坏等）
  - `comment` 属性提供补充说明
  - 差异数据可用于调整 INVENTORY ITEM 的现有数量，作为非运输类交易的审计追踪。

### 7. 产品定价模型（Figure 3.7）

- **PRICE COMPONENT** 实体作为价格信息的核心，包含：
  - `from_date` 和 `thru_date`（价格有效期）
  - `price`（金额）和 `percent`（百分比）——每个 PRICE COMPONENT 只存储其中一个
  - `comment`（说明，如"促销折扣"）
- 第一组子类型（定价类型）：
  - **BASE PRICE**（基础价格）
  - **DISCOUNT COMPONENT**（折扣组件）
  - **SURCHARGE COMPONENT**（附加费组件）
  - **MANUFACTURER'S SUGGESTED PRICE**（制造商建议零售价）
- 第二组子类型（收费模式）：
  - **ONE TIME CHARGE**（一次性收费）
  - **RECURRING CHARGE**（周期性收费，基于 Time Frequency Measure）
  - **UTILIZATION CHARGE**（使用量收费，基于某种 Unit of Measure，如网络点击数）
- PRICE COMPONENT 可关联到 PRODUCT、PRODUCT FEATURE 或两者（特性价格：可能是产品上下文内的价格，也可能是独立于产品的价格）。
- PRICE COMPONENT 可关联到 ORGANIZATION（用于存储竞争对手和供应商的价格）。
- **定价因素（Pricing Factors）**通过可选关系支持灵活组合：
  - **GEOGRAPHIC BOUNDARY**（地理区域定价）
  - **PARTY TYPE**（参与方类型定价，如少数民族企业优惠）
  - **PRODUCT CATEGORY**（产品类别定价，如所有纸制品9月促销折扣5%）
  - **QUANTITY BREAK**（数量折扣，通过 `from_quantity` 和 `thru_quantity` 定义区间）
  - **ORDER VALUE**（订单价值定价）
  - **SALE TYPE**（销售类型定价，如网络销售vs零售vs目录销售）
- **国际定价**：通过 CURRENCY MEASURE 关系支持多币种定价。
- **价格优先级**：当多个价格条件同时满足时，企业需要建立业务规则确定哪个价格组件优先。

### 8. 产品成本核算模型（Figure 3.8）

- **ESTIMATED PRODUCT COST** 实体维护每个产品的估计成本，关联到 PRODUCT 或 PRODUCT FEATURE。
- **COST COMPONENT TYPE** 实体指定成本类型，子类型包括：
  - **ESTIMATED MATERIALS COST**（估计材料成本）
  - **ESTIMATED LABOR COST**（估计人工成本）
  - **ESTIMATED OTHER COST**（估计其他成本，包括：制造设备成本、损耗成本、运输成本、销售成本如佣金/经纪费、行政管理费用）
- 成本关联可选的因素：
  - **GEOGRAPHIC BOUNDARY**（不同地理位置成本不同，如某国制造更便宜）
  - **ORGANIZATION**（如跟踪比较多个供应商的成本）
- `from_date` 和 `thru_date` 属性记录成本的有效时间段。
- 采用估计成本而非实际成本的优势：产品分析师可以结合市场理解和未来趋势预测成本，而不仅依赖历史数据。

### 9. 产品间关联关系模型（Figure 3.9a, 3.9b）

**Figure 3.9a 独立实体模型：**
- **PRODUCT COMPONENT**（产品组件）：递归多对多关系
  - `quantity_used`（组件使用数量）
  - `instruction`（组装说明）和 `comment`（备注）
  - `from_date` 和 `thru_date`（组件随时间变化的历史）
  - 适用于制造业的BOM，也适用于分销商的套装组装（如美容套装）和服务组织的服务打包
- **PRODUCT SUBSTITUTE**（产品替代）：递归多对多+数量
  - `from_date` 和 `thru_date`（替代有效期）
  - `quantity`（替代数量关系，如1小盒铅笔=12支铅笔）
  - `comment`（替代备注）
- **PRODUCT OBSOLESCENCE**（产品淘汰）：多对多递归
  - 新产品可以取代多个旧产品，旧产品可以被多个新产品取代
  - 如软件新版本合并多个旧功能为一个产品，或反之将旧功能拆分为多个新产品
- **PRODUCT COMPLEMENT**（产品互补）：多对多递归
  - 如桌垫替换纸是桌垫的互补品，建议作为配件购买
- **PRODUCT INCOMPATIBILITY**（产品不兼容）：多对多递归
  - 如某品牌的笔芯与另一种笔不兼容，需要在订单时提醒客户
  - 如果大部分组合都兼容，可改用 PRODUCT COMPATIBILITY 实体反向建模

**Figure 3.9b 替代模型：**
- 将上述所有产品关联关系通过 **PRODUCT ASSOCIATION** 超类型实体统一管理
- **PRODUCT ASSOCIATION TYPE** 实体存储关联类型，可随时扩展新的关联类型
- 适用于属性结构和关系结构类似的情况，模型更灵活

### 10. 产品与零部件模型（Figure 3.10a, 3.10b）

**Figure 3.10a 简化模型：**
- GOOD 子类化为：
  - **FINISHED GOOD**（成品，可直接发货）
  - **RAW MATERIAL**（原材料，最低级别组件，企业未对其加工）
  - **SUBASSEMBLY**（半成品，部分完成状态，一般不售卖给客户）
- 使用 PRODUCT COMPONENT 维护BOM关系

**Figure 3.10b 分离模型（适用于制造业等零部件信息非常重要的场景）：**
- **PART** 实体代表物理存在的实际物品，**PRODUCT** 实体代表市场营销的包装品
- 同一物理物品可作为不同产品出售（如电信公司同一条电话线可售为"住宅线路"或"商业线路"，取决于客户类型）
- PART 子类化为 RAW MATERIAL / SUBASSEMBLY / FINISHED GOOD
- **PART BOM** 实体维护零部件之间的BOM关系
- **PRODUCT COMPONENT**（或命名为 MARKETING PACKAGE）维护产品打包关系
- PART 需要关联到 INVENTORY ITEM、PRICE COMPONENT、REORDER GUIDELINE 和 SUPPLIER OFFERING
- 若 PART 和 PRODUCT 分开管理，SUPPLIER PRODUCT 应重命名为 **SUPPLIER OFFERING**（因为它代表提供零件或产品的参与方）

---

## 关键数据模型/概念

| 模型 | 核心实体 | 关键点 |
|------|---------|--------|
| 产品定义 (Fig 3.1) | PRODUCT (Good/Service) | 商品和服务统一为PRODUCT超类型 |
| 产品分类 (Fig 3.2) | PRODUCT CATEGORY, PRODUCT CATEGORY CLASSIFICATION, PRODUCT CATEGORY ROLLUP, MARKET INTEREST | 多层级灵活分类，primary_flag避免重复计数，分类与市场兴趣关联 |
| 产品标识 (Fig 3.3) | GOOD IDENTIFICATION, Manufacturer ID, SKU, UPCA, UPCE, ISBN | 一个商品可有多个不同类型的标识码 |
| 产品特性 (Fig 3.4) | PRODUCT FEATURE, PRODUCT FEATURE APPLICABILITY, PRODUCT FEATURE INTERACTION | 统一管理必需/标准/可选特性，支持特性间交互规则 |
| 计量单位 (Fig 3.4) | UNIT OF MEASURE, UNIT OF MEASURE CONVERSION | 支持单位换算，统一计算总库存 |
| 供应商与制造商 (Fig 3.5) | SUPPLIER PRODUCT, PREFERENCE TYPE, RATING TYPE, REORDER GUIDELINE | 多供应商优先级、评级、标准交货时间、按位置/设施的再订购策略 |
| 库存存储 (Fig 3.6) | INVENTORY ITEM (Serialized/Non-Serialized), LOT, FACILITY, CONTAINER, ITEM VARIANCE | 物品跟踪粒度灵活，批次追溯，状态与差异审计 |
| 定价 (Fig 3.7) | PRICE COMPONENT (Base/Discount/Surcharge/MSRP, OneTime/Recurring/Utilization) + 6种Pricing Factors + Currency Measure | 极度灵活的定价引擎，支持多维度多组合、多币种、特性定价 |
| 成本核算 (Fig 3.8) | ESTIMATED PRODUCT COST, COST COMPONENT TYPE | 估计成本支持趋势预测，按地理/组织/时间变化 |
| 产品关联 (Fig 3.9a/b) | PRODUCT COMPONENT/SUBSTITUTE/OBSOLESCENCE/COMPLEMENT/INCOMPATIBILITY → PRODUCT ASSOCIATION | 全面的产品间关系建模，替代方案更灵活可扩展 |
| 产品与零部件 (Fig 3.10a/b) | PART, FINISHED GOOD, RAW MATERIAL, SUBASSEMBLY, PART BOM | 区分物理物品与营销包装，适用不同企业类型 |

---

## 结论/要点

1. **产品定义统一性**：Goods 和 Services 应作为 PRODUCT 的子类型统一建模，共享基础属性，又保留各自特有的信息需求。

2. **分类的灵活性与一致性**：产品分类需要支持多层级、多重归属和时间有效性。`primary_flag` 是解决多分类导致的聚合重复计数的关键设计。若主分类要求因场景变化，可进一步引入 PRODUCT CLASSIFICATION TYPE。

3. **特性即灵活性**：将产品特性（Feature）统一建模为 NEEDED FEATURE 而非拆分为"定义特征"和"可选选项"两个实体，因为同一特性在不同产品中可能扮演不同角色。特性交互（互斥/依赖）是复杂产品配置的关键支持。

4. **定价需要引擎化思维**：PRICE COMPONENT 模型设计如此灵活（6个定价因素可任意组合、3种收费模式、多币种、可关联产品或产品特性或皆不关联），是因为定价是企业最频繁变化的业务规则。企业必须制定清晰的业务规则（如哪个折扣优先级更高、数量折扣是基础价的一部分还是独立折扣）才能有效使用该模型。

5. **库存跟踪粒度随需而定**：INVENTORY ITEM 从 Facility 到 Container 的多级定位、Serialized 与 Non-Serialized 的区分、Lot 的批次追溯、以及 ITEM VARIANCE 的差异审计，构成了完整的库存生命周期管理。

6. **产品间关系是增值信息**：Component（组件化）、Substitute（替代）、Obsolescence（淘汰）、Complement（互补）、Incompatibility（不兼容）五种关系的建模，为企业提供了超越基本产品信息的战略性数据能力。替代模型（PRODUCT ASSOCIATION）提供了更好的可扩展性。

7. **Product vs Part 的取舍**：对于大多数企业，简单的 GOOD 子类型化（Finished Good / Raw Material / Subassembly）足够；但对于以零部件为核心的制造业，将 Part（实物）和 Product（营销品）分离建模，并使用 PART BOM 管理制造BOM、MARKETING PACKAGE 管理销售打包，能更好地反映业务本质。

8. **成本估计优于实际成本**：虽然实际成本可从多种来源推导，但使用 ESTIMATED PRODUCT COST 让产品分析师能够结合市场洞察预测未来趋势，而非仅依赖历史数据做出定价决策。

9. **模型覆盖范围全面**：本章模型不仅覆盖企业自身产品，还覆盖供应商产品和竞争对手产品（通过 Organization 关联），使单一模型可服务于采购、销售和竞争分析等多个业务场景。
