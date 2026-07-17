# 第2章：People and Organizations（人员与组织）

## 章节概述
本章是全书数据模型的基础章节，建立了通用的、规范化的 Party 数据模型。作者指出大多数企业系统中的组织与人员信息存在大量冗余（如 customer、supplier、employee 等实体各自独立存储），导致数据不一致和维护困难。本章通过引入 PARTY 超级实体（包含 PERSON 和 ORGANIZATION 两个子类型）、PARTY ROLE（角色）、PARTY RELATIONSHIP（关系）、以及灵活的 Postal Address、Contact Mechanism 和 Communication Event 模型，构建了一套高度灵活、可扩展的统一人员与组织信息管理框架，后续章节均以此为基础进行扩展。

---

## 核心问题（Problems）

### 1. 组织数据的冗余问题（Organization Redundancy）
- 传统数据模型为每种业务场景创建独立实体：**customer（客户）**、**supplier（供应商）**、**department（部门）** 等
- 同一个组织可能在多个系统中扮演多种角色。例如，物业管理部既是产品销售的**供应商**，也是产品销售的**客户**
- 当组织信息变更（如地址变化）时，往往只在某一个系统中更新，导致企业内信息不一致，无法生成准确的邮件列表

### 2. 人员数据的冗余问题（Person Redundancy）
- 系统通常为不同类型的角色建立独立的实体：employee、contractor、supplier contact、customer contact 等
- 一个人可能在职业生涯中变换多种角色（如从客户变为合同工再变为雇员），每个系统都会保存一份记录，但像姓名、性别、出生日期、技能等基础信息保持不变
- 同一个人可能**同时**扮演多种角色。例如：Shirley Jones 同时是运输部的 employee、供应部的 customer 和出版部的 supplier，若拆分为三条记录则造成冗余

### 3. 人与组织分别建模导致的复杂性（Person vs. Organization Separation Problem）
- 人和组织有许多共同特征：信用评级、地址、电话、传真、电子邮件等
- 人和组织可以扮演相似的角色：合同方、买家、卖家、责任人、组织成员
- 如果将 Person 和 Organization 作为完全独立的实体，每个涉及任一方的事务（如合同、订单、会员资格）都需要建立**两条互斥的关系**（exclusive arc），增加模型复杂度

### 4. 客户关系管理（CRM）系统中缺失关系实体（Missing Relationship Entity in CRM）
- 许多 CRM 系统仅有一个 **contact** 实体来记录关联方的信息，并将 status、priority、notes 和日期等属性直接关联到 contact 上
- 问题在于：这些信息（状态、优先级、备注）实际属于**两个参与方之间的某个具体关系**，而非属于 contact 本身
- 例如：三位销售人员分别与同一位客户联系人 Marc Martinez 有不同关系，一位销售记录状态为 "very active"，另一位可能记录为 "inactive"，直接关联到 contact 会导致相互覆盖

### 5. 任职关系的多对多问题（Employment Many-to-Many Problem）
- 一个人可能随时间先后或同时隶属于多个内部组织（Internal Organization）
- 一个内部组织也可能拥有多名雇员

### 6. 组织层级结构变更问题（Organization Hierarchy Changes Over Time）
- 部门可能随时间变更所属的上级组织单位（division），需要跟踪历史变化

### 7. 联系人信息的局限性（Contact Information Limitations）
- 传统模型将联系信息作为实体的属性字段（如 address_line1, address_line2, home_phone, office_phone, office_fax）
- **数量限制问题**：如果某个人的工作电话多于一个国家，而数据库中仅有一个字段，那么多余的号码只能被挤到 "comment" 备注字段中，不利于搜索
- **类型扩展问题**：随着通信技术发展，新的联系方式不断出现，固定属性模型难以扩展
- **信息冗余问题**：如果没有独立建模，每个联系方式和地址都可能有自己的关联信息（如 directions 指引、nonsolicitation 标识、最佳通话时间）。若地址作为属性存储，地址指引可能被重复记录多次

### 8. 地址重复问题（Address Duplication）
- 同一个地址可能被多个组织使用（如共用办公设施的子公司、总部地址）
- 同一个地址可能被多个人员使用（如同一个设施内的多名员工）
- 同一个人或组织可能有多个地址（家庭地址、工作地址、度假地址等）
- 如果不共用地址记录，将产生冗余并导致 update anomaly

### 9. 联系机制的用途不明确（Contact Mechanism Purpose Ambiguity）
- 同一个地址可能有多重用途：总部地址、账单查询地址、服务地址等
- 大多数系统为每种用途创建单独的记录，即使地址信息完全相同
- 同一条电话线路可能同时用于语音和传真

### 10. 通信事件的跟踪需求（Communication Event Tracking）
- 销售人员需要了解对谁、何时、以何种目的进行过联系，以便进行后续跟进
- 通信事件需要关联到具体的 PARTY RELATIONSHIP（如双方同时存在客户关系和雇佣关系，不同关系下的通信应有不同记录）
- 通信事件可能涉及多方（如会议、研讨会），需要记录每方的角色

### 11. 物理设施的联系信息（Facility vs. Contact Mechanism）
- 联系信息可能绑定到人（如个人手机号），也可能绑定到物理位置（如工厂电话号码、塔台电话号码）
- 这些物理设施不是 postal address，也不是 party，需要独立的实体来描述

---

## 解决方案（Solutions）

### 1. 统一组织实体（ORGANIZATION Entity）
- 建立单一的 **ORGANIZATION** 实体，存储所有组织的基础信息（名称、联邦税号等），消除冗余
- 子类型划分：
  - **LEGAL ORGANIZATION（合法组织）**：如 CORPORATION（公司）、GOVERNMENT AGENCY（政府机构）
  - **INFORMAL ORGANIZATION（非正式组织）**：如 FAMILY（家庭）、TEAM（团队）、OTHER INFORMAL ORGANIZATION
- 只有合法组织才能作为合同方

### 2. 统一人员实体（PERSON Entity）
- 建立单一的 **PERSON** 实体，存储个人的基础信息，与具体的工作或角色解耦
- 属性包括：current_last_name, current_first_name, current_middle_name, current_personal_title, current_suffix, current_nickname, gender, birth_date, height, weight, mothers_maiden_name, marital_status, social_security_no, current_passport_no, current_passport_expire_date, total_years_work_experience, comment

### 3. Person 备用模型（Person — Alternate Model）
- 将重复属性拆分为独立实体，支持历史追踪：
  - **MARITAL STATUS**：婚姻状态历史（single, married, divorced, widowed），通过 MARITAL STATUS TYPE 维护类型值
  - **PHYSICAL CHARACTERISTICS**：身体特征历史（身高、体重、血压等），通过 PHYSICAL CHARACTERISTIC TYPE 维护特征类型
  - **PERSON NAME**：姓名历史及别名（aliases），通过 PERSON NAME TYPE 区分姓、名等类型；适用于矫正机构等需要追踪改名的场景
  - **CITIZENSHIP** 和 **PASSPORT**：支持多国籍和多本护照的历史记录
  - **GENDER TYPE**：支持更多性别分类（male, female, male_to_female, female_to_male, not_provided），如需历史可加入 PERSON GENDER 关联实体

### 4. Party 超级实体（PARTY Superentity）
- 建立 **PARTY** 超级实体，**PERSON** 和 **ORGANIZATION** 为其子类型
- 所有涉及人或组织的事务只需关联到 PARTY 即可，避免 exclusive arc
- 通过 **PARTY CLASSIFICATION**（分类体系）对各方进行分类：
  - **ORGANIZATION CLASSIFICATION**（组织分类）：
    - INDUSTRY CLASSIFICATION（行业分类）：telecommunications, government institute, manufacturer
    - SIZE CLASSIFICATION（规模分类）：small, medium, large, national account
    - MINORITY CLASSIFICATION（少数群体分类）：minority-owned business, 8A business, woman-owned business
  - **PERSON CLASSIFICATION**（人员分类）：
    - EEOC CLASSIFICATION（平等就业分类）：african american, native american, asian or pacific islander, hispanic, white non-hispanic
    - INCOME CLASSIFICATION（收入分类）：less than $20,000, $20,001 to $50,000, $50,001 to $250,000, over $250,000
  - **PARTY TYPE** 维护所有其他可能的分类值
- 每个分类带 from_date 和 thru_date 以支持历史追踪（如公司从 8A program 毕业的情况）

### 5. Party Role 角色模型（PARTY ROLE）
- 建立 **PARTY ROLE** 实体，描述参与方在当前语境下扮演的角色，与 PARTY 解耦
- 角色带有 from_date 和 thru_date 属性以表示有效时间段
- **PERSON ROLE（人员角色）**：
  - EMPLOYEE（正式雇员）
  - CONTRACTOR（合同工）
  - FAMILY MEMBER（家庭成员）
  - CONTACT（联系人/代表，可以是销售代表、支持代表、客户代表、供应商代表等）
- **ORGANIZATION ROLE（组织角色）**：
  - DISTRIBUTION CHANNEL（分销渠道），子类型包括 AGENT（代理，不购买和持有产品）和 DISTRIBUTOR（分销商，先购买再转售）
  - COMPETITOR（竞争对手）
  - PARTNER（合作伙伴）
  - REGULATORY AGENCY（监管机构）
  - HOUSEHOLD（住户）
  - ASSOCIATION（协会）
  - SUPPLIER（供应商）
  - ORGANIZATION UNIT（组织单位），子类型包括：PARENT ORGANIZATION（母公司）、SUBSIDIARY（子公司）、DEPARTMENT（部门）、DIVISION（事业部）、OTHER ORGANIZATION UNIT
  - INTERNAL ORGANIZATION（内部组织，指被建模的企业自身）
- **通用角色（适用于 Person 或 Organization）**：
  - CUSTOMER（客户），细分为 BILL TO CUSTOMER（付款客户）、SHIP TO CUSTOMER（收货客户）、END USER CUSTOMER（最终用户）
  - SHAREHOLDER（股东）
  - PROSPECT（潜在客户）
- **PARTY ROLE TYPE**：以声明式维护角色类型值，覆盖所有未在子类型中列出的角色（如 "placing customer", "installation customer", "notary", "doctor" 等）

### 6. 角色是否应于事务发生时定义？（Roles at Transaction Time）
- 讨论：角色可以从事务（Order、Invoice、Shipment）中推导，为什么还需要显式声明角色？
- 结论：
  - 对于**与事务无关的角色**（如 prospect 潜在客户、notary 公证人、doctor 医生），没有关联事务可供推导
  - 从**实用性角度**：企业需要快速查询谁是 customer、supplier、employee，而不必每次都去搜索关联事务
  - 角色可以在事务发生时即时创建，不必提前创建

### 7. 统一角色类型架构（ROLE TYPE Hierarchy）
- 所有角色类型采用统一架构，**ROLE TYPE** 为超级类型
- 子类型包括：PARTY ROLE TYPE、ORDER ROLE TYPE、AGREEMENT ROLE TYPE、REQUIREMENT ROLE TYPE、SHIPMENT ROLE TYPE、INVOICE ROLE TYPE 等
- 每个角色都关联到一个 PARTY、一个 ROLE TYPE 和相应的事务

### 8. Party Relationship 关系模型（PARTY RELATIONSHIP）
- 建立 **PARTY RELATIONSHIP** 实体，记录两个参与方之间的具体关系
- 每个关系由 **两个 PARTY ROLE** 及其各自的角色定义
- 属性：from_date 和 thru_date，标记关系起止时间
- 子类型示例：
  - **CUSTOMER RELATIONSHIP（客户关系）**：定义某个 CUSTOMER 是哪个 INTERNAL ORGANIZATION 的客户（多对多：一个客户可以属于多个内部组织，反之亦然）
  - **EMPLOYMENT（雇佣关系）**：定义某个 EMPLOYEE 属于哪个 INTERNAL ORGANIZATION（多对多：一个人可能先后为多个内部组织工作）
  - **ORGANIZATION ROLLUP（组织层级）**：定义组织单位之间的从属关系（多对多而非一对多：部门可能随组织结构调整而变更上级）
- **PARTY RELATIONSHIP TYPE**：定义关系类型的名称和描述，例如 "customer relationship" 的描述为 "顾客购买了或使用了内部组织的产品"
- **PARTY RELATIONSHIP TYPE 与 PARTY ROLE TYPE 的关联**：每个关系类型仅对特定的角色对有效（如 "customer relationship" 仅对 "customer"/"internal organization" 角色对有效）

### 9. Party Relationship 附属信息（Party Relationship Information）
- **PRIORITY TYPE（优先级）**：标识关系的相对重要性（very high, high, medium, low 或 first, second, third 等）
- **PARTY RELATIONSHIP STATUS TYPE（关系状态）**：标识关系的当前状态（active, inactive, pursuing more involvement）
- 状态类型采用统一的 **STATUS TYPE** 超级类型架构，子类型还包括 ORDER STATUS、SHIPMENT STATUS、WORK EFFORT STATUS 等

### 10. Postal Address 地址模型（Postal Address Information）
- 建立 **POSTAL ADDRESS** 实体，集中存储所有地址信息
- 属性：address1, address2, directions（路标指引）
- 通过 **PARTY POSTAL ADDRESS** 交集实体维护 PARTY 与 POSTAL ADDRESS 之间的多对多关系，带 from_date 和 thru_date 追踪地址历史
- **GEOGRAPHIC BOUNDARY（地理边界）**：描述各种地理范围（包括 CITY, STATE, COUNTRY, POSTAL CODE, PROVINCE, TERRITORY, SALES TERRITORY, SERVICE TERRITORY, REGION 等）
- GEOGRAPHIC BOUNDARY 具有**自递归关系**（如一个 SALES TERRITORY 可以包含若干 CITY、STATE 或 COUNTRY）
- 通过 **POSTAL ADDRESS BOUNDARY** 交集实体关联地址与地理边界

### 11. 联系人机制模型（Party Contact Mechanism）
- 建立 **CONTACT MECHANISM** 实体，通过 **CONTACT MECHANISM TYPE** 维护可用的联系机制类型
- 子类型：
  - **TELECOMMUNICATIONS NUMBER（电信号码）**：电话、传真、调制解调器、寻呼机、手机
  - **ELECTRONIC ADDRESS（电子地址）**：Internet 地址、电子邮件、Web URL
- 通过 **PARTY CONTACT MECHANISM** 交集实体维护多对多关系
- **non-solicitation_ind** 属性：标识该联系方式不可用于推销目的（法律合规考虑）

### 12. 联系人机制扩展模型（Party Contact Mechanism — Expanded）
- 将 **POSTAL ADDRESS** 作为 CONTACT MECHANISM 的子类型，统一管理所有联络方式
- **PARTY CONTACT MECHANISM PURPOSE**：表示每个联系机制对每个参与方的用途。例如：同一个地址可能兼作总部地址、服务地址、账单查询地址；同一条线路可能兼作语言和传真
- 通过 **CONTACT MECHANISM PURPOSE TYPE** 维护可用用途值（primary home address, summer home address, main office number, secondary fax number, billing inquiries, headquarters number, emergency only 等）
- **PARTY CONTACT MECHANISM 与 PARTY ROLE TYPE 的可选关联**：表示联络信息仅针对特定的角色有效（如某个地址仅在组织作为 customer 时使用）
- **CONTACT MECHANISM LINK（自递归）**：用于表示联系机制之间的关联。例如：忙时自动转接到寻呼机或手机，传真号码自动关联到电子邮件地址

### 13. 物理设施模型（Facility vs. Contact Mechanism）
- 建立 **FACILITY** 实体，描述物理结构（子类型包括 WAREHOUSE（仓库）、PLANT（工厂）、BUILDING（建筑）、ROOM（房间）、OFFICE（办公室））
- **FACILITY TYPE** 支持额外设施类型
- 设施具有**自递归关系**：建筑包含房间，楼层包含房间等
- 属性如 square_footage（面积）
- **FACILITY ROLE**：记录哪些 PARTY 对设施有何种关系（使用、租赁、拥有等），通过 FACILITY ROLE TYPE 维护角色类型
- **FACILITY CONTACT MECHANISM**：设施与联系机制的多对多关联（一个设施可能有多个邮政地址、多个电话等）

### 14. 通信事件模型（Party Communication Event）
- 建立 **COMMUNICATION EVENT** 实体，记录参与方之间发生或将要发生的通信交互
- 属性：datetime_started, datetime_ended, note
- 通信事件通常发生在某个 PARTY RELATIONSHIP 的上下文中（两个人之间可能存在多个关系，不同关系下的通信分别记录）
- 对于涉及多方的通信（如会议、研讨会），通过 **COMMUNICATION EVENT ROLE** 记录各方及其角色（facilitator 主持人、participant 参与人、note taker 记录人等）
- **COMMUNICATION EVENT PURPOSE**：记录通信事件的目的，通过 COMMUNICATION EVENT PURPOSE TYPE 维护。示例：
  - INQUIRY（咨询）
  - SUPPORT CALL（支持电话）
  - CUSTOMER SERVICE CALL（客服电话）
  - MEETING（会议）
  - SALES FOLLOW-UP（销售跟进）
  - CONFERENCE（大会）
  - SEMINAR（研讨会）
  - ACTIVITY REQUEST（活动请求）
  - 其他如 initial sales call, service repair call, demonstration, sales lunch appointment, telephone solicitation
- 通过 **COMMUNICATION EVENT STATUS TYPE** 维护事件状态（scheduled, in_progress, completed）
- 事件必须关联一个 **CONTACT MECHANISM TYPE**（电话、传真、邮件、面谈等）
- **VALID CONTACT MECHANISM ROLE**：定义何种 COMMUNICATION EVENT ROLE TYPE 对何种 CONTACT MECHANISM TYPE 有效（如 caller 和 receiver 对 phone 有效，facilitator 和 participant 对 face-to-face 有效）

### 15. 通信事件后续跟进模型（Communication Event Follow-Up）
- 建立 **CASE** 实体，将关于同一问题的多个关联通信事件归组
- 每个 CASE 可以有多个 **CASE ROLE**，记录谁负责该案例、谁检查服务质量、谁是客户等
- **WORK EFFORT（工作项）**：通信事件可能触发一个或多个工作项（如客户报告问题后需要发送软件补丁）
- 一个工作项可能与多个通信事件关联（如：请求创建的工作项 → 跟进查询进度 → 回复完成通知）
- 具体的 WORK EFFORT 模型在第六章详述

---

## 关键数据模型/概念

| 实体/概念 | 说明 |
|-----------|------|
| **ORGANIZATION** | 统一组织实体，子类型为 LEGAL ORGANIZATION 和 INFORMAL ORGANIZATION |
| **PERSON** | 统一人员实体，与具体角色解耦 |
| **PARTY** | 超级实体，PERSON 和 ORGANIZATION 为其子类型 |
| **PARTY CLASSIFICATION** | 分类体系（行业、规模、EEOC、收入等） |
| **PARTY ROLE** | 参与方角色（customer, supplier, employee, internal organization 等），独立于 PARTY 存在 |
| **PARTY ROLE TYPE** | 角色类型声明系统 |
| **PARTY RELATIONSHIP** | 双方之间的关系定义，由两个 PARTY ROLE 构成 |
| **PARTY RELATIONSHIP TYPE** | 关系类型，关联到特定的角色对 |
| **PRIORITY TYPE / PARTY RELATIONSHIP STATUS TYPE** | 关系优先级和状态 |
| **POSTAL ADDRESS** | 集中式地址记录，避免冗余 |
| **GEOGRAPHIC BOUNDARY** | 地理边界（含自递归），涵盖城市、国家、邮政编码、销售区域等 |
| **CONTACT MECHANISM** | 统一联系机制（电信号码、电子地址、邮政地址） |
| **CONTACT MECHANISM PURPOSE** | 联系机制的用途分类 |
| **CONTACT MECHANISM LINK** | 联系机制之间的递归关联 |
| **FACILITY** | 物理设施，与 PARTY/CONTACT MECHANISM 解耦 |
| **COMMUNICATION EVENT** | 通信事件历史记录 |
| **CASE** | 关联通信事件的案例管理 |
| **WORK EFFORT** | 通信事件后续需要执行的工作 |
| **ROLE TYPE** | 全书统一的超级类型，贯穿所有事务的角色类型 |

---

## 结论/要点

1. **消除冗余是本章的核心动机**：通过在单一位置（PARTY、POSTAL ADDRESS、CONTACT MECHANISM）维护基础信息，可以大幅减少数据冗余和更新异常，提升企业信息一致性。

2. **PARTY 超级实体是全书最重要的架构设计**：通过将 Person 和 Organization 统一为 Party，避免了 exclusive arc 问题，使所有后续的事务模型（订单、合同、发货等）都可以统一关联到 Party。

3. **角色（Role）和关系（Relationship）是两个独立但互补的维度**：PARTY ROLE 定义一方"是什么"（如 customer），PARTY RELATIONSHIP 定义两方"之间的关系是什么"（如 ACME 是 ABC Subsidiary 的 customer）。这种分离使得模型能精准表达复杂的现实世界业务关系。

4. **灵活性与可扩展性是设计核心**：通过 TYPE 模式（PARTY ROLE TYPE、CONTACT MECHANISM TYPE 等），企业可以随时添加新的分类值、新的联系机制类型，适应未来业务变化。

5. **状态、优先级、备注属于关系而非实体本身**：这是客户关系管理领域的一个重要洞察，能有效避免多角色参与方数据相互覆盖的问题。

6. **联系方式的集中化建模**：将邮政地址、电信号码和电子地址统一视为 CONTACT MECHANISM，结合 PURPOSE 分类，构建了高度灵活的联系管理系统。

7. **整个模型以 PARTY 为中心向外辐射**：本章的各种实体（PARTY ROLE、PARTY RELATIONSHIP、PARTY POSTAL ADDRESS、PARTY CONTACT MECHANISM、COMMUNICATION EVENT、CASE）均围绕 PARTY 展开，构成了后续章节（产品、订单、工作管理等）的基础骨架。
