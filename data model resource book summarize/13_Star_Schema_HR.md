# 第13章：Star Schema Designs for Human Resources（人力资源星型模式设计）

## 章节概述

本章以ABC公司的EEOC（Equal Employment Opportunity Commission，平等就业机会委员会）合规部门为实际案例，展示了如何基于企业数据仓库构建人力资源分析数据集市。EEOC合规部门在听闻东部区域销售经理对第12章所建销售分析数据集市赞誉有加后——该数据集市大幅缩短了获取战略报告的时间和精力，借助合适的DSS（Decision Support System）工具还能相当便利地分析趋势——主动联系IS（Information Services）部门，请求为其部门构建一个小型数据仓库。

IS经理对数据集市概念的传播感到欣喜，随即委派人员投入此项任务。经过一系列深入访谈，IS团队明确了EEOC分析师最关心的问题，并基于企业数据仓库中的数据设计了两个互补的星型模式（Star Schema）：

1. **Figure 13.1 — HUMAN_RESOURCES_FACT（明细级模式）**：以员工月度统计快照为中心的细粒度星型模式，支持8个维度的任意组合查询。
2. **Figure 13.2 — HUMAN_RESOURCES_SUMMARY_FACT（汇总级模式）**：更高粒度的汇总星型模式，为跨组织平均指标对比和联邦政府报告优化。

本章强调了两条重要原则：数据仓库设计必须以终端用户的实际业务需求为驱动，而成功的部门级数据集市案例是推动企业数据仓库战略最有效的推广手段。

## 核心问题（Problems）

- **EEOC合规报告的维度复杂性**：EEOC合规部门需要按职位类型（Position Type）、种族/族裔（EEOC Category）、性别（Gender）、薪资等级（Pay Grade）、服务年限（Length of Service）、雇佣状态（Status）等多个维度组合来分析员工的薪资公平性，以满足联邦政府（Federal Government）的合规报告要求。传统操作型系统难以高效支撑这种多维度即席聚合查询。

- **公平就业机会的量化分析需求**：EEOC分析师需要回答以下典型问题：
  - 有多少程序员/分析师是非裔美国人或西班牙裔？他们的年薪与白人同行相比如何？
  - 女性员工与男性员工的平均薪资是否存在显著差异？
  - 是否存在某个群体薪资增长率系统性高于或低于其他群体的现象？
  - 按职位类型看，薪资与工作经验年限或公司任职年限之间的关联性如何？
  - 少数族裔员工的总数及其占比是多少？
  - 不同雇佣状态的员工（如兼职员工）人数及其平均年薪是多少？
  - 不同职位类型、不同服务年限段的员工分布如何？服务超过10年、20年的员工有多少？

- **时间维度的趋势追踪**：分析人员需要观察各维度组合下的薪资、年龄、经验等指标随时间的变化趋势，但这不需要每日级别的粒度——月度快照足以捕捉有意义的趋势变化。

- **两种粒度的查询需求并存**：同一部门内部存在两类分析场景：一类需要细粒度查询（可下钻到具体部门、具体岗位类型的差异分析），另一类需要粗粒度的快速汇总报表（按职位类型、EEOC类别、性别跨组织比较平均年薪和平均年龄）。查询性能和数据的受控可见性均是驱动多层粒度设计的因素。

- **组织层次结构的查询处理**：企业分析需要从部门（Department）→ 分部（Division）→ 区域（Region）→ 子公司（Subsidiary）→ 公司（Corporation）的灵活汇总路径。传统的规范化层级结构需要在SQL中写递归查询，这在星型模式的即席查询场景中对用户极不友好。

- **部门推广和协作障碍**：组织内经常出现一个部门观望另一个部门的成功后才愿意投入资源的情况。IS部门需要证明数据集市方法论的可靠性——不仅要交付数据集市，还要交付满意且积极的用户。

- **数据安全与访问控制**：人力资源数据涉及员工个人隐私（年龄、性别、种族、薪资等），不同角色的人员应只能访问其被授权看到的数据粒度级别。

## 解决方案（Solutions）

### 方案一：明细级人力资源星型模式（Figure 13.1 — HUMAN_RESOURCES_FACT）

IS部门设计了以HUMAN_RESOURCES_FACT事实表为核心的明细级星型模式，如图13.1所示。此模式按月装载数据，为分析师提供了任意维度组合下的员工统计快照。

**事实表度量设计（5个度量值）**：

- **number_of_employees**：满足指定维度条件组合的员工计数。这是一个可加性度量，用于回答诸如"某个部门有多少员工？某个职位类型下有多少西班牙裔女性员工？"等问题。
- **average_age**：满足指定维度条件的员工平均年龄。
- **average_years_experience**：员工在行业内（包含ABC公司内外的全部工作履历）的平均工作年限。这个度量可以帮助回答"有15年以上行业经验的数据建模岗位员工平均薪资是多少？"等问题。分析师可以通过将average_years_experience除以number_of_employees来计算给定维度下所有员工的总经验年数。
- **average_years_employed**：员工在ABC公司的平均任职年数。这是一个计算度量——基于企业数据仓库中员工历任职位（Positions）的from_date和thru_date记录汇总得到的总任职时长。月度快照机制确保了这个度量的历史可追溯性。
- **average_annual_pay**：截至当月月末、按员工当前薪资率（Pay Rate）折算的年化薪资数额。在快照时刻点，这是假设当前薪资率保持不变的情况下员工一年可获得的金额。该度量支持诸如"按职位类型和部门看，薪资与15年以上经验/任职年限之间的关联如何？"等问题。

**事实表设计关键点**：

- **复合主键规则**：第12章曾显式标注过事实表的主键组成，本章及之后章节不再显式书写——读者应理解事实表的主键恒为所有维度表主键的组合。这是星型模式设计的核心约定。
- **月度快照（Monthly Snapshot）机制**：month_id作为时间维度外键被包含在事实表键中。虽然员工的年龄、经验值、薪资率在持续变化，但month_id的设计使得每次月度快照成为一个独立的历史版本——分析师可以追踪这些度量在任意历史时点的值，例如比较"2023年1月与2024年1月55-60岁员工的平均薪资"。
- **数据源与装载周期**：每月月末从企业数据仓库执行一次ETL批量装载。装载对象主要包括EMPLOYEES表和POSITIONS表的相关数据。

**8个维度表的详细设计**：

**1. Organizations Dimension（组织维度）**

数据源自企业数据仓库中的INTERNAL_ORG_ADDRESSES表。由于EEOC数据集市不需要地理分析维度，仅提取了organization_id和organization_name。

核心设计特性——**组织层级反规范化（Denormalized Org Hierarchy）**：
- 维度表通过多级org_type列（level1_org_type、level2_org_type、level3_org_type、level4_org_type、level5_org_type）以及对应的多级name列（level1_name、level2_name、level3_name、level4_name、level5_name），将企业数据仓库中规范化存储的组织层次关系"展开"为扁平结构。
- level1始终代表与事实表度量记录直接关联的最低层组织单元（如Accounting Department）。level2是其直接的上级组织（如Finance Division），level3再上一级（如Eastern Region），level4（ABC Subsidiary），level5（ABC Corporation）。
- 每个维度记录由organization_id唯一定义——这个ID对应的是level1级别的最底层组织。更高的层级（level2、level3等）不包含独立的organization_id，仅以名称和类型列的形式存在。
- ABC公司的组织定义中刚好有5个层级。其他企业可根据自身情况增加或减少层级列。

这种扁平化设计的优势在于：
- 分析师无需编写递归SQL即可按照任意层级进行汇总。"显示Eastern Region下所有部门的平均薪资"这样的查询只需在WHERE子句中对level3_name进行简单过滤。
- 查询执行计划极其简单，优化器可以直接利用索引和分区裁剪，性能大幅优于递归查询。
- 终端用户——尤其是非技术背景的EEOC分析师——的操作难度显著降低。

Table 13.1提供了该维度表的数据示例（为简洁仅展示到level3）：

| ORGANIZATION ID | LEVEL 1 NAME | LEVEL 1 ORG_TYPE | LEVEL 2 NAME | LEVEL 2 ORG_TYPE | LEVEL 3 NAME | LEVEL 3 ORG_TYPE |
|---|---|---|---|---|---|---|
| 10929 | Accounting | Department | Finance | Division | Eastern | Region |
| 23948 | Investments | Department | Finance | Division | Eastern | Region |
| 29039 | Sales | Department | Marketing | Division | Western | Region |

**2. Position Types Dimension（职位类型维度）**

从企业数据仓库的POSITIONS表直接提取，对应第9章逻辑数据模型中的POSITION（职位）和POSITION TYPE（职位类型）实体。

该维度包含两个分析列：
- **position_type**：具体的职位类型描述，如"数据建模师"、"程序员/分析师"等。
- **position_class**：更高层的职位分类，源自POSITION TYPE CLASSIFICATION（职位类型分类）实体。

这种两层嵌套分类结构的设计类似于第12章PRODUCTS维度中product_description和category_description的并列设计。分析师可以在position_type（细粒度）和position_class（粗粒度）两个层次之间灵活切换分析视角。

设计前提假设：一个具体的职位在其整个生命周期内（through time）始终对应且仅对应一个position_type。

**3. Genders Dimension（性别维度）**

用于分析不同性别在岗位分布、薪资水平、经验水平等方面的差异——这是EEOC合规分析的核心维度之一。

包含5种性别分类描述值：
- **male** — 男性
- **female** — 女性
- **male to female** — 男转女（跨性别）
- **female to male** — 女转男（跨性别）
- **not provided** — 未提供

这种5分类设计满足了许多人力资源企业维护多元化报告的合规标准需求。

**4. Length of Services Dimension（服务年限维度）**

该维度允许分析师按预定义的服务年限区间进行数据切片。例如：
- "服务10-15年的员工有多少人？他们的平均年薪是多少？"
- "服务20年以上的员工按职位类型和部门分布情况如何？"

Length of Services维度是一个典型的**区间维度（Banding Dimension）**，将连续的任职年限离散化为分析区间。该维度可以与连续型度量average_years_employed配合使用：分析师可先按服务年限区间（如5-9年）过滤，再进一步查看该区间内员工的精确average_years_employed值，实现"区间粗筛→度量精查"的分析流程。

**5. Statuses Dimension（状态维度）**

按员工的雇佣状态（Employment Status）进行分组分析。可能的状态值包括：
- **part time** — 兼职
- **full time** — 全职
- **exempt** — 豁免人员（不受加班费条款约束的职位）
- **temporary** — 临时雇员

通过此维度，企业可以回答："临时雇员有多少人？他们的平均年薪与全职员工相比如何？"

**6. Pay Grades Dimension（薪资等级维度）**

对应第9章图9.7中的PAY GRADE（薪资等级）实体。允许分析师按企业定义的薪资等级（如"pay grade 5"）对员工进行分组并查询对应的人数和平均年薪。

可扩展性说明：如果业务需要更细粒度的分析（如SALARY STEP，即薪资阶梯），可以将其作为此维度的额外层级进行扩展。

典型分析场景："薪资等级5的员工有多少人？他们的平均年薪是多少？不同职位类型的员工在薪资等级上的分布是否有差异？"

**7. EEOC Types Dimension（EEOC类型维度）**

从企业数据仓库的EMPLOYEES表中提取所有唯一的eeoc_type_id及其对应的description值。

此维度被置于核心位置，因为该数据集市项目由EEOC合规部门直接赞助——种族/族裔分类分析是合规报告的基本要求，而非可选的附加项。

典型分类值包括：white（白人）、Hispanic（西班牙裔/拉丁裔）、African-American（非裔美国人）、Asian（亚裔）、Native American（美洲原住民）等。

**8. Time_By_Month Dimension（时间按月维度）**

与第12章销售分析数据集市共用同一张TIME_BY_MONTH表。month_id逐月唯一，代表一个会计年度（fiscal year）和月份的组合。

此维度的关键价值：
- 分析师不仅可以按月分析，还能按季度（quarter）和年度（year）一级向上汇总。
- 表中数据应覆盖HUMAN_RESOURCES_FACT事实表所涵盖的完整时间段。
- 与销售数据集市共用同一时间维度表的做法，意味着两个数据集市在对齐时间维度上具有一致性——这是数据仓库架构中**符合维度（Conformed Dimension）**的最佳实践。

### 方案二：汇总级人力资源星型模式（Figure 13.2 — HUMAN_RESOURCES_SUMMARY_FACT）

访谈中，EEOC合规人员明确表示还需要跨组织的汇总对比指标，用于定期向联邦政府提交合规报告。例如："按职位类型、EEOC类别、性别看，全公司的平均年薪"——这类查询只关心宏观对比，不需要部门明细。

IS部门因此基于HUMAN_RESOURCES_SUMMARY_FACT事实表构建了第二个更高粒度的星型模式（图13.2）。

**汇总模式的核心度量与明细模式相同**（number_of_employees、average_age、average_years_experience、average_years_employed、average_annual_pay），但有两个关键区别：
- 维度组合更少——按organization_id、position_type_id、eeoc_type_id、gender_id进行更宽范围的分配（不包含LENGTH_OF_SERVICES、STATUSES、PAY_GRADES），从而得到更高层次的聚合值。
- 每条记录代表跨多个下位维度的汇总数据，查询扫描的数据量大幅减少。

**装载策略**：汇总数据可以从两个来源装载：
1. 直接从企业数据仓库汇总生成。
2. 从HUMAN_RESOURCE_FACT明细表汇总生成——这是推荐方式。

采用方案2的优势：
- 两个模式为同一部门使用，且装载周期相同（月末）。
- 明���表在月度ETL完成后已经完成了数据的预筛选和清晰化（preselected），无需再次扫描整个企业数据仓库。
- ETL逻辑简洁：从明细表选取该月数据，提取organization_id、position_type_id、eeoc_type_id、gender_id四个维度，对number_of_employees求和，对其他度量取平均——关系型DBMS通常有内置的AVG()等聚合函数，实现非常简单。

通过此汇总模式，EEOC分���师可以高效回答如下问题：
- 西班牙裔女性数据建模师的平均薪资与白人男性数据建模师相比如何？
- 女性主管与男性主管在平均年龄和平均薪资上的差异如何？
- 在特定部门内，工作经验年限、年化薪资和EEOC类别之间是否存在显著的趋势关系？

### 为什么需要第二个汇总视图？——设计动机分析

本章专门设有一节讨论此问题，其重要性不容忽视。理论上，图13.2汇总视图能回答的任何问题，图13.1的明细视图也都能回答——只需在明细表上做一次额外的聚合查询。那么，为什么还要创建和维护另一个星型模式？

两个核心理由：

**1. 查询性能**

汇总模式针对EEOC合规部门的特定报表需求进行了**预聚合优化**。明细表可能包含数百万行员工月度快照记录，每次生成一份联邦政府报告就需要扫描大量数据并实时计算聚合。汇总表将这一计算提前完成——以有限的存储空间换取每次查询的即时响应。对于需要定期产出的高频报表场景，这种以空间换时间的设计是必要的。

实际上，一个企业可能有多个部门的多个不同粒度的汇总星型模式——每个部门只需要其特定范围的汇总数据。

**2. 数据安全**

人力资源信息是高度敏感的数据，包含员工的年龄、种族、性别、薪资等个人信息。汇总视图作为一种**数据访问控制手段**：

- 某些部门或用户角色可能仅被授权查看粗粒度的人力资源汇总信息——例如，只能看到"跨部门按职位类型和EEOC类别统计的平均薪资"，但不能下钻到具体部门甚至具体员工的薪资数据。
- 明细级模式则仅向EEOC合规部门的授权分析师开放。
- 通过提供不同粒度的**重叠数据视图**（overlapping views），系统管理员可以在保护隐私和满足分析需求之间找到平衡点——限制敏感数据访问的同时，不同受众仍能获取其工作所需的汇总信息。

## 关键数据模型/概念

### 事实表设计模式

- **HUMAN_RESOURCES_FACT（明细事实表）**：以organization、position_type、gender、length_of_service、status、pay_grade、eeoc_type、month为8维组合的月度员工统计快照表。所有度量值均为聚合类型（average或count），事实表复合主键是所有维度表主键的组合。
- **HUMAN_RESOURCES_SUMMARY_FACT（汇总事实表）**：粒度更粗但度量语义相同的汇总快照。区别于明细表：维度数更少（4个维度），每条记录对应更宽泛的分组，查询扫描量大幅减少。

### 维度设计模式

- **org_type层级列技术（Level Columns）**：将规范化组织树反规范化为多组levelN_org_type和levelN_name列。以存储空间换取查询简便性和性能增益，避免递归查询。层级数量可按企业组织深度灵活定制（ABC公司为5层）。

- **分类实体嵌套（Classification Nesting）**：Position Types维度同时包含position_type（细粒度）和position_class（粗粒度），形成两个分析层级。此模式在第12章的产品维度（product_description + category_description）中也出现过，是处理层级分类维度的统一设计模式。

- **区间维度（Banding Dimension）vs. 连续度量（Continuous Measure）**：Length of Services维度和Pay Grades维度都属于区间维度，将连续变量离散化为分组区间。其中Length of Services维度还可与连续度量average_years_employed交叉使用，提供"先按区间粗分、再查精确度量"的双重分析能力。

- **符合维度（Conformed Dimension）**：Time_By_Month维度跨Chapter 12和Chapter 13共用，使不同数据集市的查询结果可以在统一时间维度上对齐。这是Kimball维度建模理论的核心概念。

### 粒度与汇总策略

- **多层粒度（Multiple Granularities）**是数据仓库设计的标准原则，第12章销售分析中也出现了类似的多层模式。
- **两层设计**：
  1. 明细层（HUMAN_RESOURCES_FACT）：全面支持8维任意组合分析。
  2. 汇总层（HUMAN_RESOURCES_SUMMARY_FACT）：为定期合规报告优化，查询快但维度组合受限。
- 汇总层的两个设计驱动：性能优化（预聚合加速）、数据安全（限制可见粒度）。

### 度量计算与快照

- 所有5个度量（number_of_employees、average_age、average_years_experience、average_years_employed、average_annual_pay）均为聚合值，属于**非加性（non-additive）或半加性（semi-additive）度量**——不能简单跨月份直接累加（因为不同月份间员工重叠），但在单一月份内可以跨其他维度聚合。例如，某月各部门的员工总数不是各部门employee count的直接加和（因为一个员工只属于一个部门）。
- average_years_employed的推算逻辑：从EMPLOYEE POSITION记录中的from_date和thru_date计算每个人的总任职天数，再按维度分组取平均。月度快照机制确保每年每月保留一份该值的历史版本。
- **快照事实表 vs. 事务事实表**：HUMAN_RESOURCES_FACT是一个**快照事实表**（Snapshot Fact Table）而非事务事实表（Transaction Fact Table）。它记录的是每个维度组合在每个月的"状态"而非"事件"。

### ETL装载架构

- 装载节奏：月末批量（Batch Load），无需每日增量。
- 数据流：企业数据仓库 → HUMAN_RESOURCES_FACT → HUMAN_RESOURCES_SUMMARY_FACT。
- 明细表作为汇总表的唯一数据源（推荐方案），避免重复扫描企业数据仓库，遵循"一次ETL、多层派生"的架构原则。

### 方法论与团队协作

- **终端用户访谈是不可替代的**：部门数据集市的结构和粒度不是由技术团队预先设想，而是在多轮用户访谈中逐步明确的。EEOC分析师需要回答什么具体问题、需要什么粒度的数据（日/月）、需要哪些可选的维度组合——这些最终由用户场景决定。
- **成功案例驱动组织变革**：IS部门最初为东部区域销售部门建数据集市时，其他部门持观望态度。当销售数据集市证明其价值后（报表时间大幅减少、用户满意度提升），EEOC合规部门才主动签约。这说明数据仓库策略的推广不仅是技术问题，更是组织文化和管理说服力的游戏。

## 结论/要点

1. **一个数据集市可以有多个模式的组合**：本章为同一部门设计了两个不同粒度的星型模式——明细模式服务于灵活的下钻分析，汇总模式服务于定期合规报表的快速产出。当两者共享相同的维度键和装载周期时，汇总表可直接从明细表派生，减少ETL总体开销。

2. **组织层次结构的扁平化反规范化是星型模式的实用核心技术**：通过levelN_org_type扁平列预先展开层级关系，替代了递归SQL查询。这条设计原则在第14章的Inventory、Purchase Order、Shipment等星型模式中也会再次出现并推广应用。

3. **Time_By_Month维度是跨数据集市的符合维度**：与销售数据集市共用同一时间维度表，不仅保证了时间维度上的一致性，也简化了时间维度表的维护——这是数据仓库架构中应当优先复用的基础维度。

4. **月度快照设计为时变属性提供了历史追溯能力**：员工的年龄、经验、薪资等属性随时间持续变化，每月一份快照将这类缓慢变化维度（SCD, Slowly Changing Dimension）属性的历史值固化在事实表中，使趋势分析成为可能——无需额外的SCD管理机制。

5. **汇总视图既是性能优化工具，也是数据安全手段**：粗粒度预聚合加速查询性能、减少扫描数据量。同时，数据安全并非只能通过数据库权限实现——不同粒度的数据视图本身就是一种"门控"机制，让不同角色的人员只能看到合适级别的汇总数据，而HR个人敏感信息得到妥善保护。

6. **数据集市设计应由终端用户问题驱动**：星型模式的维度选择、度量定义、粒度设计都源于对分析师实际问题的应答——"工程师需要回答什么问题"才是设计的原点，技术手段只是实现方式。

7. **一个部门的成功是另一个部门参与的理由**：数据集市的推广往往遵循"先试点验证、再横向复制"的路径。方法论和技术被一个部门的满意用户验证过后，自然吸引其他部门加入。

8. **本章所有设计均可按企业实际定制**：组织层级数量（5级）、维度选择和扩展（如SALARY STEP）、装载频率（月末 vs. 其他周期）均可调整。附录C提供了完整的表和字段清单，配套CD-ROM包含可在多种数据库平台上运行的SQL建表脚本（需单独授权）。
