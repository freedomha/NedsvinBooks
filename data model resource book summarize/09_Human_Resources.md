# 第9章：Human Resources（人力资源）

## 章节概述

本章讨论了人力资源管理的数据建模，涵盖从职位定义、职位授权与分类、职位履行与跟踪、汇报结构、薪酬确定与历史、福利定义与跟踪、工资单信息、求职申请、技能与资格管理、员工绩效评价到员工离职全生命周期。模型基于 PARTY/EMPLOYMENT 关系模型（第2章）和 BUDGET 模型（第8章），通过 POSITION 与 PERSON 的分离设计，实现了灵活的组织结构管理和历史追踪。

本章的总体模型如图 9.14 所示，将所有人力资源相关的实体和关系统一呈现。

## 核心问题（Problems）

1. **职位与人员的混淆**：许多数据模型直接将员工属性（如状态、薪资、汇报关系）关联到 EMPLOYEE 实体，而非 POSITION 实体。"已辞职"是工作关系（EMPLOYMENT）的状态，而非该人员的状态——同一个人可能从一个子公司辞职后又被另一个子公司聘用。

2. **汇报结构的动态变化**：传统模型用 EMPLOYEE 的递归关系表示管理者，在大规模组织重组时需要更新大量记录。实际上汇报结构是组织结构的函数，而非人员的函数——当主管升职后，下属仍向同一个 POSITION 汇报，只是换了另一个填补该 POSITION 的人。

3. **薪资与职位类型脱钩**：实际支付给个人的薪资可能与其职位的标准费率不同（如员工被赋予更多职责但未及时加薪），职位共享场景下两人占据同一职位但可能有不同薪资水平。

4. **简历/福利/薪资应关联到 EMPLOYMENT 而非 PERSON**：这些信息属于员工与雇主之间的工作关系，不应直接放在 PERSON 实体中。将福利关联到 EMPLOYEE + ORGANIZATION 可避免承包商用错福利。

5. **组织结构的历史追踪**：大多数系统只维护当前组织结构快照，重组和晋升历史丢失，当面临 EEOC 诉讼等问题时无法还原历史。

6. **多雇主场景**：在大型多公司企业中，员工可能在不同时间在不同子公司工作，福利、薪资等需要按 EMPLOYMENT 实例分别追踪。

7. **工资支付方式的多样性**：现代企业需要支持现金、支票、电子转账，员工可能要求部分工资以不同方式分配给不同银行账户，还需要处理定期扣款。

8. **职位共享（Job Sharing）���战**：两人共享一个 FTE（Full Time Equivalent）职位，HEAD COUNT 应关联到 POSITION 而非 PERSON。

9. **绩效评价的敏感性**：绩效记录属于高度敏感信息，需要专门的权限控制和处理方式。

10. **职位的多维度分类**：职位可能需要按多种维度分类（计算机类/技术类/管理类），且分类可能随时间变化。

11. **薪酬体系的多样性**：不同企业采用不同薪酬体系——有的使用 pay grade/salary step 体系（如政府），有的使用工资等级范围，有的完全灵活。

## 解决方案（Solutions）

### 1. 职位定义（Position Definition，图 9.3）

- **POSITION** 实体：表示企业中的一个职位空缺（job slot），可被多人随时间占用
  - 包含 estimated_from_date / estimated_thru_date（计划时间）
  - 包含 actual_from_date / actual_thru_date（实际时间）
  - 包含 salaried / exempt / full_time / temporary 等标志位
- **POSITION TYPE**：定义同类职位的共同特征（职位描述、标准职称、福利百分比）
- **BUDGET ITEM** 关联：职位通过预算项授权和资金支持
- **RESPONSIBILITY TYPE / VALID RESPONSIBILITY / POSITION RESPONSIBILITY**：三级职责管理体系
  - RESPONSIBILITY TYPE：所有可能的职责类型
  - VALID RESPONSIBILITY：某 POSITION TYPE 下合法的职责（含 from/thru date）
  - POSITION RESPONSIBILITY：特定 POSITION 实际分配的职责

### 2. 职位类型分类（Position Type Definition，图 9.4）

- **POSITION TYPE CLASS**：多对多交叉实体，连接 POSITION TYPE 和 POSITION CLASSIFICATION TYPE
- 支持分类随时间变化的追踪（如 Business Analyst 从 Computer 类重新分类为 MIS 类）
- 可通过分类类型存储薪资方式信息（hourly/salary/exempt/non-exempt）
- 关联 UNION 实体（ORGANIZATION ROLE 子类型）追踪工会信息

### 3. 职位履行与追踪（Position Fulfillment，图 9.5）

- **POSITION FULFILLMENT**：多人对多职位的历史追踪实体
  - 复合主键：position_id + party_id(PERSON) + from_date
  - 支持多人同时占用同一职位（职位共享）的记录
  - 关联 PERSON 而非 EMPLOYEE，允许记录合同工等外部人员
  - 支持完整职业发展路径追踪
- **POSITION STATUS TYPE**：追踪职位状态生命周期（planned/active/open/inactive/closed）
- HIRING ORGANIZATION 跟踪职位的所属组织

### 4. 职位汇报结构（Position Reporting Structure，图 9.6）

- **POSITION REPORTING STRUCTURE**：POSITION 的递归关系
  - 包含 from_date / thru_date 支持组织重组历史追踪
  - 包含 primary_flag 支持矩阵式汇报结构（一人向多名管理者汇报）
  - 重组时只需更改职位的汇报关系，无需修改每个人的记录

### 5. 薪酬确定与历史（Salary Determination，图 9.7）

- **POSITION TYPE RATE**（扩展自第6章）：记录职位类型的标准费率
  - 关联 RATE TYPE（highest/lowest/average/standard pay rate）
  - 关联 PERIOD TYPE（per year/week/month）
  - 支持 from_date/thru_date 追踪费率变更历史
- **PAY GRADE / SALARY STEP** 体系支持：
  - PAY GRADE 包含多个 SALARY STEP（含 amount 属性）
  - POSITION TYPE RATE 引用 PAY GRADE + SALARY STEP 替代直接金额
  - 薪资涨幅直接更新 SALARY STEP 的 amount，无需更新 POSITION TYPE RATE 记录
- **PAY HISTORY**：记录实际支付薪资
  - 关联到 EMPLOYMENT（EMPLOYEE + INTERNAL ORGANIZATION），非 POSITION
  - 必须填写 amount（不接受间接引用），确保历史准确性
  - 一个时间段只能有一条 PAY HISTORY 记录

### 6. 福利定义与追踪（Benefits Tracking，图 9.8）

- **PARTY BENEFIT**：关联到 EMPLOYMENT（非 PERSON），避免福利给错人
  - 包含 cost、actual_employer_paid_percent、available_time
  - 关联 PERIOD TYPE 提供金额的周期上下文
- **BENEFIT TYPE**：福利类型定义（health/vacation/sick leave/401k）
  - 标准 employer_paid_percent 作为默认值
- 三层雇主付费百分比优先级规则：
  1. PARTY BENEFIT 的 actual_employer_paid_percent（最高优先级）
  2. POSITION TYPE 的 benefit_percent（次优先级）
  3. BENEFIT TYPE 的 employer_paid_percent（默认）

### 7. 工资单信息（Payroll Information，图 9.9）

- **PAYCHECK**：DISBURSEMENT 的子类型（继承第7章支付模型）
  - 继承 payment_id、payment_ref_num、effective_date、amount
  - 关联 EMPLOYEE 和 INTERNAL ORGANIZATION
- **PAYMENT METHOD TYPE**：支付方式（cash/check/electronic）
- **PAYROLL PREFERENCE**：员工支付偏好
  - 支持多种分配方式（百分比或固定金额）
  - 包含 routing_number、account_number、bank_name（电子转账）
  - 关联 PERIOD TYPE 和 DEDUCTION TYPE（定期扣款）
  - from_date/thru_date 支持偏好变更历史
- **DEDUCTION / DEDUCTION TYPE**：扣款记录和类型
  - DEDUCTION TYPE：federal tax/FICA/state tax/401k/retirement/insurance
  - DEDUCTION：每张工资单的实际扣款金额

### 8. 求职申请（Employment Application，图 9.10）

- **EMPLOYMENT APPLICATION**：关联 POSITION（可选）和 PERSON（候选人）
- **EMPLOYMENT APPLICATION STATUS TYPE**：申请状态（received/reviewed/filed/rejected）
- **EMPLOYMENT APPLICATION SOURCE TYPE**：来源（newspaper/personal referral/Internet）
- 关联 referring PERSON 追踪推荐人

### 9. 技能与资格（Skills and Qualifications，图 9.11）

- **PARTY SKILL / PARTY QUALIFICATION**：关联到 PARTY（不仅是 PERSON，组织也有技能和资格）
- **PERSON TRAINING**：个人参加的培训项目追踪
- **RESUME**：关联 PARTY（组织也可能有简历/资质描述）

### 10. 员工绩效（Employee Performance，图 9.12a/9.12b）

- **EMPLOYEE PERFORMANCE REVIEW**：
  - receiver：被评审员工
  - manager_：负责评审的管理者
  - 由 PERFORMANCE REVIEW ITEM 组成（问题 + rating + comment）
  - 关联 RATING TYPE 提供评分标准
  - 可关联 PAY HISTORY（加薪/降薪）、PAYCHECK（奖金）、POSITION（晋升/降职）
- **PERFORMANCE NOTE**：绩效笔记，记录非正式绩效事件
- 替代方案 **9.12b**：将绩效作为 COMMUNICATION EVENT 的子类型处理

### 11. 员工离职（Employee Termination，图 9.13）

- **PARTY RELATIONSHIP STATUS TYPE**：记录 EMPLOYMENT 的 "terminated" 状态
- **TERMINATION TYPE**：离职类型（resignation/firing/retirement）
- **TERMINATION REASON**：离职原因（insubordination/new job/non-performance/moved）
- **UNEMPLOYMENT CLAIM**：失业索赔追踪
  - 包含 claim_date、description
  - 关联 UNEMPLOYMENT CLAIM STATUS TYPE（filed/pending/accepted/rejected）
- 关键概念：EMPLOYMENT 被终止，而非员工被终止；PARTY STATUS 用于记录与关系无关的 party 状态（如 "deceased"）

## 关键数据模型/概念

| 实体/概念 | 说明 |
|-----------|------|
| POSITION | 职位空缺（job slot），可被多人占用 |
| POSITION TYPE | 同类职位的共性定义 |
| POSITION FULFILLMENT | 人员-职位 多对多历史追踪 |
| POSITION REPORTING STRUCTURE | 职位间的汇报关系（递归）|
| POSITION RESPONSIBILITY | 职位实际分配的职责 |
| POSITION TYPE RATE | 职位类型的标准薪资费率 |
| PAY GRADE / SALARY STEP | 结构化薪资等级体系 |
| PAY HISTORY | 人员实际薪资历史（关联 EMPLOYMENT）|
| PARTY BENEFIT | 人员福利（关联 EMPLOYMENT）|
| PAYROLL PREFERENCE | 员工支付偏好配置 |
| PAYCHECK | 工资单（DISBURSEMENT 子类型）|
| DEDUCTION | 工资扣款记录 |
| EMPLOYMENT APPLICATION | 求职申请追踪 |
| PARTY SKILL / QUALIFICATION | 技能和资格（关联 PARTY）|
| EMPLOYEE PERFORMANCE REVIEW | 绩效评审 |
| PERFORMANCE NOTE | 绩效笔记 |
| TERMINATION TYPE / REASON | 离职类型和原因 |
| UNEMPLOYMENT CLAIM | 失业索赔 |

## 结论/要点

1. **POSITION 与 PERSON 是分离的实体**：汇报结构是 POSITION 的函数，薪资和福利是 EMPLOYMENT（PERSON + ORGANIZATION 关系）的函数——将这些属性混入 PERSON 实体会导致数据不一致和难以追踪历史。

2. **EMPLOYMENT 而非 PERSON**：PAY HISTORY、PARTY BENEFIT、PAYCHECK 等均应关联到 EMPLOYMENT（EMPLOYEE + INTERNAL ORGANIZATION），以确保正确的业务规则执行和多雇主场景支持。

3. **职位履行的灵活性**：POSITION FULFILLMENT 的三部分复合主键支持职位共享（job sharing）和历史追踪，关联 PERSON（非 EMPLOYEE）支持合同工等外部人员。

4. **汇报结构通过职位管理**：重组时只需修改 POSITION REPORTING STRUCTURE 的记录，无需逐人修改，大幅减少更新并保留完整历史。

5. **薪酬体系适配**：模型同时支持灵活薪资（直接金额）和结构化薪资等级（PAY GRADE/SALARY STEP），通过 POSITION TYPE RATE 统一。

6. **三层雇主付费百分比优先级**：PARTY BENEFIT → POSITION TYPE → BENEFIT TYPE，灵活满足不同精细度的福利成本计算需求。

7. **没有真正"终止"的员工**：EMPLOYMENT 终止不等于 PERSON 终止；PARTY RELATIONSHIP STATUS TYPE 追踪雇佣关系终止，PARTY STATUS 追踪人员本身状态（如 deceased）。

8. **绩效评价的两种建模方案**：独立实体方案（9.12a）适合高敏感性场景，COMMUNICATION EVENT 子类型方案（9.12b）适合需要统一追踪的通用场景。
