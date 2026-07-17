# 第15章：Implementing the Universal Data Models（实施通用数据模型）

## 章节概述

本章是全书最后一章，也是最长的一章，将前14章介绍的所有通用数据模型从理论推向实践落地。核心主题是如何将通用数据模型（Universal Data Models）应用于实际的企业系统开发工作中，包括三个层面：**企业级数据模型（Enterprise Data Model）**、**特定应用的逻辑数据模型（Logical Data Model for Applications）** 和**物理数据库设计（Physical Database Design）**。本章还讨论了数据仓库架构的策略选择。全文以 XYZ 公司的联系管理（Contact Management）系统为贯穿案例，逐步展示从业务需求到物理实现的完整流程。

## 核心问题（Problems）

### 1. 企业信息需求的不确定性
即使有了通用数据模型作为跳板，每个企业仍存在独特的、超出标准模型范围的信息需求。数据建模人员必须通过访谈、小组会议和建模讨论来捕捉这些额外需求，否则模型将无法完整反映企业实情。

### 2. 业务人员不理解“数据”
大多数业务人员以业务流程的方式思考，而非以数据结构的方式思考。数据建模人员必须从业务流程的角度出发，通过业务讨论驱动数据需求的理解，这是数据建模过程中最大的挑战之一。

### 3. 逻辑模型与物理实现之间的鸿沟
逻辑数据模型代表信息需求，而物理数据库设计需要考虑性能、交易量、数据量、所选数据库管理系统（DBMS）等多重因素。如何从归一化的逻辑模型转化为性能优化的物理设计是一个核心挑战。

### 4. 子类型（Subtype/Supertype）的物理实现选择
逻辑模型中的超类型-子类型关系在物理实现时有四种不同的策略，每种策略都有各自的优劣，设计者需要在灵活性、性能、维护复杂度之间做出权衡。

### 5. 数据仓库架构的不一致问题
在将数据从操作系统转移到数据仓库时，业界存在重大争论：是先建立企业级数据仓库再分发到部门数据集市（Data Mart），还是直接从操作系统提取到部门数据仓库？不同部门各自的转换逻辑可能导致严重的数据不一致。

### 6. 数据冗余与不一致
企业的客户服务部门和销售部门可能都在使用类似的通信事件数据，但这些数据并未集成。这导致销售人员可能不知道客户刚刚向客服投诉过，反之亦然。一个供应商可能同时也是企业最好的客户，但企业可能在招标过程中完全不知道这一点。

## 解决方案（Solutions）

### 一、定制化通用数据模型以满足企业需求

**步骤：**
1. 以通用数据模型为基础（jump-start），识别哪些实体、关系和属性适用于本企业
2. 通过访谈、小组会议发现额外需求
3. 将额外需求添加到企业数据模型中
4. 将所有模型链接到一个统一的企业视图
5. 按业务概念（Subject Data Area）分割模型以便维护，如：Party、Product、Order、Shipment、Work Effort、Invoicing、Accounting、Human Resources

**XYZ公司案例（联系管理扩展）：**
- 通用数据模型已涵盖 PARTY（当事人）、PERSON（个人）、ORGANIZATION（组织）、CONTACT MECHANISM（联系机制）等实体
- 额外需求：
  - **PREFERRED CONTACT TIME**（首选联系时间）：每种联系机制可以有不同的时间偏好。例如某人可以随时接收电子邮件，但仅在工作日 9:00-17:00 通过电话联系；另一个人可能仅允许周四到周五 15:00-17:00 联系，但周末 10:00-16:00 可以打家庭电话
  - **WEB ADDRESS**、**EMAIL ADDRESS**、**IP ADDRESS** 作为 ELECTRONIC ADDRESS（电子地址）的子类型

### 二、企业数据模型解决业务问题

企业数据模型完成后能够：

1. **提供清晰的信息视图**：图形化的模型帮助业务团队获得对企业信息的共同理解，验证业务规则、概念和想法
2. **支持突破性思维（Out-of-the-Box Thinking）**：当业务人员看到信息的全貌时，能够发现多种替代方案
3. **识别潜在陷阱**：模型化信息可以提前识别效率、性能或支持方面的潜在问题
4. **发现数据冗余**：帮助识别重复的数据源和未集成的系统
5. **维护完整画像**：确保人员和组织的信息在企业的各个角色之间保持一致

**具体例子**：
- 电子邮件、邮政信件和电话本质上是联系业务实体的不同方式，属于同类信息。系统应能统一展示所有联系方式
- 通信事件（COMMUNICATION EVENT）在关系（PARTY RELATIONSHIP）的上下文中才有意义，需要追踪通信事件的跟踪情况
- 在决策支持环境中，可以分析哪种联系方式通常带来最大的销售额和成交量，以最大化收入

### 三、为特定应用构建逻辑数据模型

**双向贡献原则**：企业数据模型为特定应用提供起点，应用数据设计将洞察和学习反向反馈到企业数据模型中。

**流程：**

1. **理解业务流程（Business Process）**
   - 从业务人员的“怎么做”（How）出发，逐步引出“需要什么信息”（What）
   - 使用模板流程模型（Template Process Models）加速分析
   - 创建业务流程与相关数据的对照表

2. **XYZ公司销售团队的业务流程与数据需求对照表（Table 15.2）：**

| 业务流程 | 相关数据 |
|---------|---------|
| 检索当前联系方式 | CUSTOMER, CUSTOMER CONTACT METHOD, POSTAL ADDRESS, PHONE NUMBER, ELECTRONIC MAIL ADDRESS, WEB ADDRESS |
| 确定最有效的联系方式 | CUSTOMER CONTACT METHOD AND CUSTOMER CONTACT METHOD PURPOSE |
| 确认客户许可 | CUSTOMER, CUSTOMER CONTACT METHOD, non-solicitation ind, use permission ind, CONTACT METHOD |
| 确定最佳联系时间 | CUSTOMER CONTACT METHOD, PREFERRED CONTACT TIME |
| 进行客户联系 | CUSTOMER, CUSTOMER CONTACT METHOD, CONTACT METHOD |
| 更新客户联系与联系方法关系 | CUSTOMER, CUSTOMER CONTACT MECHANISM, CONTACT METHOD, CONTACT METHOD RELATIONSHIP |

3. **逻辑数据模型的实体增加（Figure 15.4）：**
   - **use permission ind** 属性：添加到 CUSTOMER CONTACT METHOD PURPOSE 实体，用于记录客户是否授予了特定用途（如“销售推销”）的联系许可
   - **CONTACT MECHANISM TYPE FORMAT** 实体：提供联系机制的格式验证。例如：
     - 电子邮件地址必须包含 "@" 字符
     - 国内电话号码必须以三位数字开头（区号）
   - 企业数据模型追踪所有角色的 CONTACT METHODS，而特定应用只关注 CUSTOMERS 的联系方式

### 四、物理数据库设计

#### 基本设计原则

1. **以逻辑数据模型为基础**：物理设计必须从逻辑模型出发
2. **归一化到第三范式（3NF）**：确保每个属性只存储一次，每个属性直接与主键关联（"the key, the whole key, and nothing but the key"）
3. **根据性能需求反规范化（Denormalize）**：考虑表连接数量、索引、查询频率、更新频率、插入频率
4. **DBA与数据建模师协作**：需要经验丰富的数据库管理员参与
5. **数据映射与转换**：从旧系统映射数据到新结构，规划转换逻辑和数据加载时机
6. **使用工具生成代码**：现代工具可从模型化方案生成所需代码

#### 子类型的四种物理实现策略（以 PARTY/PERSON/ORGANIZATION 为例）：

| 策略 | 说明 | 示例 |
|------|------|------|
| **策略1** | 超类型与所有子类型合并为一张表，通过查找表指示子类型 | PARTY 表 + PARTY TYPE 查找表（person / organization） |
| **策略2** | 每个子类型建立独立表，超类型属性复制到每个子类型表中 | PERSON 表 + ORGANIZATION 表，不建立 PARTY 表 |
| **策略3** | 超类型与一个子类型合并为一张表，其他子类型独立建表 | PARTY+PERSON 合并表 + 独立的 ORGANIZATION 表 |
| **策略4** | 超类型一张表，每个子类型各一张表，通过外键关联 | PARTY 表 + PERSON 表（外键指向 PARTY）+ ORGANIZATION 表（外键指向 PARTY） |

#### 物理数据库设计示例：Party Role and Relationship 模型

**背景**：基于第二章的当事人角色和关系模型（PARTY → PERSON/ORGANIZATION → CUSTOMER/EMPLOYEE/INTERNAL ORGANIZATION → CUSTOMER RELATIONSHIP/EMPLOYMENT/ORGANIZATION ROLLUP）

**方案一（Option 1）—— 分离 PERSON 和 ORGANIZATION 表**：
- PERSON 表和 ORGANIZATION 表各自独立（策略2）
- CUSTOMER、EMPLOYEE 等角色各自建表，通过 person_id 或 organization_id 关联
- CUSTOMER TYPE 表替代 BILL TO CUSTOMER 和 SHIP TO CUSTOMER 子类型（策略1）
- 共享属性（如姓名）存储在 PERSON 表中，不重复存储在角色表中
- PARTY RELATIONSHIP 子类型（EMPLOYMENT, CUSTOMER RELATIONSHIP, ORGANIZATION ROLLUP）各自独立建表，继承超类型属性和关系
- **优点**：各部门可以更容易地“拥有”和管理自己的信息；通过 person_id 和 organization_id 链接实现集成视图
- **缺点**：新角色和关系需要新建表；角色和关系表可能需要冗余属性
- **性能权衡**：从 CUSTOMER 联接到 PERSON 获取客户姓名虽然多了一次 join，但避免了同一人既是客户又是员工时姓名冗余存储的问题

**示例数据（Tables 15.3-15.9）**：
- **PERSON 表**：包含 person_id, current first name, current last name, current middle name, alias name, birthdate
- **ORGANIZATION 表**：包含 organization_id, organization name, federal tax id
- **CUSTOMER 表**：customer_id 关联到 person_id 或 organization_id，包含 last_contact_date（派生字段，用于性能优化）
- **EMPLOYEE 表**：employee_id 关联到 person_id，包含 ssn, mother's maiden name
- **CUSTOMER RELATIONSHIP 表**：链接客户与组织，包含 priority
- **EMPLOYMENT 表**：链接员工与组织，包含 from_date, thru_date
- **ORGANIZATION ROLLUP 表**：组织层级关系，如子公司→母公司、部门→分部
- **重要发现**：Joe Jones（person_id 7890）既是 XYZ 公司的员工又是客户——这类信息可用于提供更好的服务

**方案二（Option 2）—— 统一的 PARTY 表**：
- 采用 PARTY 表取代分离的 PERSON 和 ORGANIZATION 表（策略4）
- 角色和关系仍各自独立建表，通过 party_id 关联
- **优点**：
  - 人员和组织可以共享相同的数据结构（如 PARTY CONTACT MECHANISM 可用于两者）
  - AGREEMENT 和 ORDER 可以直接关联到 PARTY，无需分别关联 PERSON 和 ORGANIZATION
  - 责任分配（Responsibilities）可以直接赋给 PARTY
- **缺点**：PARTY 表可能非常庞大，需要通过大量索引或更强大的处理器来解决性能问题
- **折中方案**：在角色表中只存储 party_id 作为外键，同时冗余存储公共 PARTY 属性——这样至少企业能识别同一方向扮演多个角色（数据协调）

**方案三（Option 3）—— 通用化设计**：
- PARTY 表合并 PERSON 和 ORGANIZATION（策略1）
- 所有角色统一存储在 PARTY ROLE 表中，通过 ROLE TYPE 区分角色类型
- 所有关系统一存储在 PARTY RELATIONSHIP 表中
- **适用场景**：
  - 数据仓库或操作数据存储（ODS）
  - 数据量较小或拥有强大处理器的环境
  - 信息维护集中化管理
- **优点**：极高的灵活性，新角色和新关系无需变更数据库结构
- **缺点**：性能可能有挑战

**示例数据（Tables 15.10-15.13）**：
- PARTY 表：合并 PERSON 和 ORGANIZATION，PARTY NAME 通过拼接 first_name + last_name 生成
- PARTY ROLE 表：展示每个 PARTY 可以扮演多种角色（如 John Doe 既是 Prospect 又是 Employee）
- PARTY RELATIONSHIP 表：灵活存储各种关系

### 五、数据仓库模型的使用

#### 核心策略

**建议架构**：先建立企业级数据仓库（Enterprise-Wide Data Warehouse），再分发到部门数据集市（Departmental Data Marts）

**反对“直接到部门级”的理由：**
1. 不同部门的转换逻辑必然不同，导致管理信息不一致
2. 市场部门生成的高管决策数据可能与订单处理部门的数据不一致
3. 不一致的结果可能导致企业怀疑信息可信度，或更糟——基于错误信息做决策

**案例分析**：在一个数据转换项目中，发现 PERSON 表中同一个人的血型字段有两个不同值——一个来自系统A，一个来自系统B。企业难以判断哪个更准确，只能两个都保留。

**集中转换的优势：**
- 一次转换，全企业共享，节省时间和成本
- 识别数据不一致并统一决策如何处理
- 帮助定位运营系统中的问题，推动集成

**快速启动策略**：使用模板数据仓库设计（如本书第10章、第11章）创建原型部门数据仓库，先加载少量数据验证收益，再规划企业级数据仓库。

#### 数据仓库转型步骤（来自第10章、第11章）

转型步骤应选择性使用，不必全部执行：
- 包含派生数据（Derived Data）
- 合并表（Merge Tables）
- 创建数据数组（Create Arrays of Data）

仅在明确存在企业级需求时才进行反规范化。

## 关键数据模型/概念

### 1. PREFERRED CONTACT TIME（首选联系时间）
- 关联到 PARTY CONTACT MECHANISM
- 每个联系机制可以有多个时间偏好
- 属性：from datetime, thru datetime
- 示例：Marc Martinez 作为 ACME 客户，首选联系时间为周一至周五 9:00-17:00

### 2. CONTACT MECHANISM TYPE FORMAT（联系机制类型格式）
- 用于验证不同联系机制类型的格式正确性
- 属性：格式字符串（如 "@" 作为电子邮件的必要部分，"###" 作为电话号码的区号）

### 3. use permission ind（使用许可标识）
- 在 CUSTOMER CONTACT METHOD PURPOSE 实体中
- 标记客户是否授予了特定用途的联系许可
- 示例：客户同意某个电话号码用于“销售推销”目的

### 4. 子类型物理实现的四种策略总结（核心概念）
从上文“解决方案”部分详述，这是数据库设计者必须掌握的核心决策框架。

### 5. 企业数据模型与逻辑数据模型的双向关系
- 企业模型 → 应用模型：提供起点和上下文
- 应用模型 → 企业模型：反馈新发现的信息需求

### 6. 数据仓库集中转换原则
全企业共享一次转换的投资成果，避免各部门独立转换导致的数据不一致

## 结论/要点

### 核心设计要点（书中总结的十条）：

1. **建立企业数据模型**是信息集成的关键，帮助理解企业全局信息需求
2. **与业务专家保持沟通**以充分捕捉业务需求
3. **与业务方评审模型**，确保业务理解被正确捕获
4. **以企业数据模型为基础**构建特定应用的逻辑数据模型，确保应用与整体系统集成
5. **使用流程模型**作为开发逻辑数据模型的另一个输入来源
6. **在清晰理解业务流程的基础上**自动化业务处理
7. **基于逻辑数据模型创建物理数据库设计**，需考虑性能因素和目标数据库管理系统
8. **选择合适的子类型物理实现策略**（四种方案之一）
9. **使用企业和逻辑数据模型作为基础**，构建集成的决策支持（数据仓库）环境
10. **遵循明确定义的设计原则**，通用数据模型可以适应并构建出高质量的物理设计

### 全书总结论（Final Thoughts）

- 每个通用数据模型都有**多种实现方式**，取决于业务需求、性能要求和技术环境
- 遵循清晰的设计原则，通用数据模型可以适配并生成高质量、集成化的数据库实现
- 通用数据模型不仅可以驱动事务处理系统的开发，还可以作为数据仓库设计的基础
- 作者期望这只是更广泛的通用数据模型努力的开端，信息系统行业将继续开发更多可复用模型
- **最终目标**：缩短系统开发周期，以更低的成本生产更高质量、更好集成的信息系统

### 方法论总结

本章展示了一套完整的从理论到实践的方法论：
```
通用数据模型 (Universal Data Models)
    ↓ 定制化 + 补充企业特定需求
企业数据模型 (Enterprise Data Model)
    ↓ 提取 + 业务视角
逻辑数据模型 (Logical Data Model)
    ↓ 归一化 + 性能优化
物理数据库设计 (Physical Database Design)
    ↓ 实现 + 测试
可用的数据库应用系统
    ↓ 提取 + 转换 + 加载
企业数据仓库 → 部门数据集市
```

这套方法论的核心理念是：**不重复造轮子，但要在通用轮子的基础上适配特定的道路**——即利用已有的通用数据模型作为跳板，避免从零开始，同时充分捕捉和实现企业的独特需求。
