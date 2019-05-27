### Graph Analytics for Healthcare Fraud Risk Estimation

**论文整体流程**

1. 介绍医疗保险数据集与数据集处理
2. 利用Neo4j构建图
3. 在图上抽取为每个provider抽取11个特征
4. 抽取特征后将provider划分为train和test，在WEKA上跑决策树模型预测provider是否欺诈





**数据集：**

- **LEIE**（List of Excluded Individuals/Entities）

  该数据集包含部分已被识别的医保欺诈者（个人或机构）的信息。

- **PUF**（Medicare Provider Utilization and Payment Data: Physician and Other Supplier）

  | 列名                             | 描述                                                         |
  | :------------------------------- | :----------------------------------------------------------- |
  | NPI                              | National Provider Identifier医疗服务提供方唯一标识           |
  | NPPES\_PROVIDER\_\*\_NAME        | Provider的名字                                               |
  | NPPES\_CREDENTIALS               | Provider学历                                                 |
  | NPPES\_PROVIDER\_GENDER          | Provider性别                                                 |
  | NPPES\_ENTITY\_CODER             | Provider类别，‘I’为个人，‘O’为组织                           |
  | NPPES\_\*                        | Provider地址邮编等                                           |
  | PROVIDER_TYPE                    | Provider所申请报销的服务在特殊服务列表中的最大服务数         |
  | MEDICARE_PARTICAPATION_INDICATOR | 医保允许额度是否被Provider接受标记                           |
  | PLACE_OF_SERVICE                 | 报销申请地机构性质标识                                       |
  | HCPCS\_CODER                     | Treatment唯一标识                                            |
  | HCPCS\_DESCRIPTION               | Treatment描述                                                |
  | HCPCS\_DRUG\_INDICATOR           | Treatment编码是否是属于Medicare Part B Drug Average Sales Price (ASP) file上的编码 |
  | LINE_SRVC_CNT                    | 服务次数                                                     |
  | BENE_UNIQUE_CNT                  | 服务受益人数（去除多次获取同一服务的人次）                   |
  | BENE_DAY_SRVC_CNT                | 服务受益人数（去除同一天内获取同一服务的人次）               |
  | AVERAGE_MEDICARE_ALLOWED_AMT     | 平均医保允许赔偿额度                                         |
  | AVERAGE_SUBMITED_CHRG_AMT        | 平均Treament收费                                             |
  | AVERAGE_MEDICARE_PAYMENT_AMT     | 平均实际赔偿额                                               |
  | AVERAGE_MEDICARE_STANDARD_AMT    | 数据标准化后（消除不同地区病人收入、支付能力、消费意愿不同的影响）的平均实际赔偿额 |

  该数据集包含2012年、2013年和2014年的“医疗保险提供者使用和支付数据:医生和其他提供者”数据

- **Part-D**（Medicare Provider Utilization and Payment Data: Part D Prescriber）

  该数据集包含医生对患者开具的处方药信息。

- **NPIs**（National Provider Identifiers）

  该数据集包含医院或机构的唯一标识符。





**Part-D数据图构建**

数据集特征说明（中文）：

> 药物的总成本包含：药物总成本包括药物的成分成本，配药费，销售税和任何适用的管理费，并且基于Part-D计划，Medicare受益人，政府补贴和任何其他第三方支付者支付的金额。

| 特征名                        | 特征说明                                           |
| ----------------------------- | -------------------------------------------------- |
| npi                           | 执行索赔的执行者的唯一标识（共计893160个）         |
| nppes_provider_last_org_name  | 个体（I）的姓或组织（O）的名称（个体共计893140个） |
| nppes_provider_first_name     | 个体（I）的名或组织（O）为空（组织共计20个）       |
| nppes_provider_city           | 注册城市（共计12130个）                            |
| nppes_provider_state          | 注册者所在的州（缩写）（共计61个）                 |
| specialty_description         | 索赔描述（共计）                                   |
| description_flag              | 描述的标识（S、T）                                 |
| drug_name                     | 药物名称：品牌名称或者非专利名称（共计2779种）     |
| generic_name                  | 药物的化学名称                                     |
| bene_count                    | 医生所开此药物的受益人的数量，小于11的为nan        |
| total_claim_count             | 一年内此药物的医疗保险的索赔数量                   |
| total_30_day_fill_count       | 一年内开此药的天数 / 30                            |
| total_day_supply              | 一年内开的药物总量                                 |
| total_drug_cost               | 一年内所有相关索赔支付的总药物费用                 |
| bene_count_ge65               | 65岁及以上的医疗保险唯一受益人的总人数             |
| bene_count_ge65_suppress_flag | bene_count_ge65被限制的标识                        |
| total_claim_count_ge65        | 65岁及以上的医疗保险索赔数量                       |
| ge65_suppress_flag            | total_claim_count_ge65被限制的标识                 |
| total_30_day_fill_count_ge65  | 一年内65岁及以上的开此药数量 / 30                  |
| total_day_supply_ge65         | 一年内为65岁及以上的受益人分发这种药物的总日供应量 |
| total_drug_cost_ge65          | 一年内为65岁及以上受益人的所有相关索赔支付的总药费 |

#### 图构建

- **Nodes：**provider，drug，treatment code (HCPCS)，locations，exclusion codes（each corresponding to a possible reason for being excluded），generic drugs，
- **Edges：**
  - **PRESCRIBED：**每个provider与其开的每种药之间有一条边，边属性来自PartD数据集。
  - **CHARGE_OF：**每个provider与其每次治疗（HCPCS）之间有一条边，边属性来自PUF数据集。
  - **LOCATED_AT：**每个provider与其所在的地址之间有一条边。address转换为latitude与longitude拼接起来作为唯一标识.

> 整个图包含173M条边，5.05M个节点。下图是整个图的一个例子

![1558923342757](../pics/1558923342757.png)

#### Graph Analytics

包含三个任务：

- Similarity functions between pairs of entities based on structural similarity
- Attribute estimation based on network propagation
  - can be used to predict cascades or epidemics or to estimate centrality, status, or contagion
- Complex structure detection and analysis

**Behavior-Vector Similarity**

假设provider的报销与开药行为会影响其欺诈风险程度。

#### Experimental

包含`12000`个欺诈provider作为负样本，每个provider至少存在一条边。另外随机选择`12000`非欺诈的provider作为正样本。分别选择`6000`个作为训练集与测试集。每个provider包含11个特征