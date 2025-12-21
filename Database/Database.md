# 数据库技术体系

## 文档导读

本文档系统性地介绍数据库技术体系，从分布式理论到各类数据库实现，涵盖以下核心内容：

| 章节 | 内容 | 适合读者 |
|------|------|----------|
| **分布式常用算法** | Quorum、Raft、Paxos 共识算法 | 所有人 |
| **Database（关系型）** | MySQL、PostgreSQL、AlloyDB、Aurora 架构对比 | 开发/DBA |
| **NoSQL 数据库** | 键值、文档、列族数据库核心概念 | 开发/架构师 |
| **键值 vs 文档数据库** | Redis vs MongoDB vs DynamoDB 深度对比 | 架构师 |
| **OLTP & OLAP & HTAP** | 事务型、分析型、混合型数据库分类 | 架构师 |
| **NoSQL 数据库实现** | DynamoDB、MongoDB、Cassandra 架构详解 | 开发/架构师 |
| **搜索引擎数据库** | Elasticsearch、OpenSearch 架构与调优 | 开发/运维 |
| **时序数据库** | InfluxDB、TimescaleDB、Prometheus 对比 | 运维/SRE |
| **图数据库** | Neo4j、Neptune 图模型与查询 | 开发/架构师 |
| **数据库理论** | CAP、ACID、BASE、MVCC 理论基础 | 所有人 |
| **NewSQL** | TiDB、CockroachDB、Spanner 分布式事务 | 架构师 |
| **Big Data** | EMR 组件、大数据生态概览 | 数据工程师 |

---

## 分布式常用算法

Quorum、Raft和Paxos都是分布式系统中用于保证数据一致性和高可用性的机制或算法，但它们在设计理念、实现复杂度及应用场景上有显著区别：

### 核心算法对比

#### 1. Quorum（法定人数机制）

- **核心思想**：对读写操作设置最小投票数W（写）和R（读），保证满足 W+R>N*W*+*R*>*N*（N为节点总数），使得读写操作集存在交集，从而保证一致性。
- **特点**：没有固定Leader，所有节点地位平等，灵活调整读写性能与一致性之间的平衡。
- **适用场景**：适合对强一致性要求不那么严格，但需要高可用性的场景。实现简单，但一致性保证相对弱一些。

#### 2. Paxos（一致性算法）

- **核心思想**：通过多轮提案和投票，确保分布式系统中所有节点就某个值达成共识。基于严格的多数派原则和提案编号排序。
- **特点**：不依赖固定Leader，可以容忍多节点故障。算法复杂，理解和实现较难。Multi-Paxos引入Leader角色提升效率。
- **适用场景**：对严格一致性有高要求，且能够接受系统复杂性的场景。

#### 3. Raft（一致性算法）

- **核心思想**：将共识过程分为选举、日志复制、安全性三个阶段，设计了明确的Leader、Follower、Candidate角色。
- **特点**：相比Paxos更易理解和实现；Leader负责日志复制和客户端请求处理；通过任期号选举领导者，避免活锁。
- **适用场景**：广泛应用于实际分布式系统，兼顾可理解性和一致性。

#### 总结比较

| 特性        | Quorum        | Paxos                 | Raft           |
|:--------- |:------------- |:--------------------- |:-------------- |
| 是否有Leader | 无             | 无（Multi-Paxos有Leader） | 有              |
| 一致性保证     | 可配置（R+W>N时强一致，否则最终一致） | 强                     | 强              |
| 复杂度       | 低             | 高                     | 中              |
| 容错能力      | 多数节点正常即可      | 多数节点正常即可              | 多数节点正常即可       |
| 性能        | 灵活调节读写参数，性能可调 | Multi-Paxos与Raft相当，Basic Paxos较低 | 较高（Leader集中处理） |
| 理解实现难度    | 简单            | 复杂                    | 相对简单           |

简言之，Quorum是一种投票表决机制，Raft和Paxos是具体的分布式共识算法，Raft从Paxos简化而来，以提高实用性和易用性。

---

## Database


### 常见数据库术语映射与结构对照表

| 类型      | 代表实例                      | 命名空间            | 表/集合          | 行/记录       | 列/字段         | 核心索引/存储要点                                                                                     | 一致性/事务语义                                                                                           | 典型访问模式/最佳实践                                                                                               | 一句话避坑提示                                                                                                                                                                                                                                                                                                      |
| ------- | ------------------------- | --------------- | ------------- | ---------- | ------------ | --------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 关系型     | **MySQL**                 | Database        | Table         | Row        | Column       | B-Tree、二级索引；<br>✅Redo Log(WAL)                                                                | ACID、强一致、事务、MVCC(InnoDB)                                                                           | OLTP 点查/范围、联表、事务                                                                                          | **索引设计与最左前缀；<br>避免大事务（操作数据量大，Undo Log膨胀）+<br>长事务（持有锁时间长，阻塞其他事务）**                                                                                                                                                                                                                                            |
| 关系型     | PostgreSQL                | Database        | Table         | Row        | Column       | B-Tree/GiST/GIN/BRIN；<br>✅WAL                                                                 | ACID、MVCC                                                                                          | 复杂查询、全文/地理、OLTP/OLAP 混合                                                                                   | 统计过期导致慢查询；合理用分区表与并行                                                                                                                                                                                                                                                                                          |
| 关系型     | **AlloyDB(GCP)**          | Database        | Table         | Row        | Column       | B-Tree + 列式引擎；<br>智能缓存；<br>✅WAL                                                               | ACID、MVCC；<br>PostgreSQL 兼容                                                                        | OLTP + 轻量 OLAP、混合负载                                                                                       | **①成本高**：比 Cloud SQL 贵 2-3 倍；<br>**②锁定 GCP**：仅 GCP 托管，无自建选项；<br>**③分析查询优化**：列式引擎加速聚合，但不如专用 OLAP                                                                                                                                                                                                              |
| NewSQL  | **TiDB**                  | Database        | Table         | Row        | Column       | 分布式事务 + 二级索引；<br>✅Raft Log(WAL)                                                               | 分布式 ACID、MVCC(Percolator)                                                                          | 横向扩展 OLTP、HTAP                                                                                            | 热点行/自增主键需打散；合理 TiFlash/统计                                                                                                                                                                                                                                                                                    |
| NewSQL  | **Spanner(GCP)**          | Database        | Table         | Row        | Column       | 分布式事务 + 二级索引；<br>TrueTime 时钟同步；<br>✅Paxos Log(WAL)                                            | 分布式 ACID、MVCC(TrueTime)；<br>External Consistency（外部一致性）                                            | 全球分布式事务、金融/广告                                                                                             | **①TrueTime 依赖**：需要 GPS+原子钟硬件支持；<br>**②跨区延迟**：全球一致性以延迟为代价（跨洲写入可达100-500ms）；<br>**③热点键**：单调递增主键导致热点（原理同 TiDB）                                                                                                                                                                                                 |
| 文档型     | **MongoDB**               | Database        | Collection    | Document   | Field        | B-Tree、多键/稀疏/TTL；<br>✅Journal(WAL)                                                            | 单文档原子；集合级、MVCC(WiredTiger)                                                                         | 文档点查、聚合管道、灵活 schema                                                                                       | **①高基数字段+缺索引→爆表**：_id 是主键(ObjectId自动生成)，user_id/order_id 等业务字段需手动建索引，否则全表扫描；<br>**②Shard Key 热点**：自增ID/时间戳/低基数字段作分片键导致数据写入集中在单个Shard，应选高基数字段或哈希分片（**原理同 HBase RowKey/DynamoDB Partition Key 热点**）                                                                                                            |
| 文档型     | **Firestore Native(GCP)** | Database        | Collection    | Document   | Field        | 自动索引；Megastore 存储                                                                             | 强一致（Paxos）；Entity Group 事务；MVCC(多版本)                                                               | 实时订阅、移动/Web 应用、离线支持                                                                                       | **①Entity Group 限制**：事务限制在 25 个实体/秒；<br>**②查询能力受限**：无 JOIN、无复杂聚合，不适合后端复杂业务；<br>**③成本模型**：按读写次数计费，高频访问成本高                                                                                                                                                                                                     |
| 键值      | Redis                     | DB(逻辑)          | -             | KV         | -            | 内存结构+跳表等；<br>✅AOF(WAL)                                                                        | 最终/主从；Lua 原子                                                                                       | 极低延迟缓存、计数、队列                                                                                              | 不要当永久存储；内存淘汰与持久化取舍                                                                                                                                                                                                                                                                                           |
| 键值/文档   | **DynamoDB**              | -               | Table         | Item       | Attribute    | 主键+GSI/LSI                                                                                    | 可调一致；分区容量                                                                                          | 单表大规模、高可用、事件驱动                                                                                            | **Partition Key 热点**：自增ID/时间戳作分区键导致写入集中在单分区，应选高基数字段或复合键（**原理同 HBase RowKey 热点**）；GSI 写放大与费用                                                                                                                                                                                                                  |
| 列族宽表    | Cassandra                 | Keyspace        | Table         | Row(分区/聚簇) | Column       | LSM+SSTable；<br>分区键+聚簇键，<br>**列族必须预定义(<10)**；列动态；<br>✅CommitLog(WAL)                          | 可调一致(ONE/QUORUM/ALL)                                                                               | 按分区键点查/时间序                                                                                                | 每分区建议<100MB、<100k 项；<br>**Partition Key 热点**：低基数/单调递增键导致数据倾斜（**原理同 HBase RowKey 热点**）；<br>二级索引用慎                                                                                                                                                                                                             |
| 列族宽表    | **HBase**                 | Namespace       | Table         | Row        | CF/Qualifier | 行键顺序；<br>**列族必须预定义(<10)**；列动态；<br>✅HLog(WAL)                                                  | 单行原子；无跨行事务                                                                                         | 行键点查/前缀/范围扫描                                                                                              | **列族尽量少(<10)，列可以多(数千)**；<br>**RowKey 热点**：时间戳/自增ID作RowKey导致写入集中在单个Region，应使用前缀散列或反转（**原理同 DynamoDB Partition Key/MongoDB Shard Key 热点**）；<br>二级索引需外部（**Phoenix 查询引擎**）                                                                                                                                       |
| 列族宽表    | **Bigtable(GCP)**         | -               | Table         | Row        | CF/Qualifier | 行键有序分片；<br>**列族必须预定义**；列动态；<br>**多版本**（每个Cell保存历史）；<br>**TTL**（自动过期删除）；<br>✅Colossus Log(WAL) | **单行原子**（依赖Colossus同步复制）；<br>**单集群强一致**（读写同一副本集）；<br>**跨集群最终一致**（异步复制）；<br>**无跨行事务**（无Paxos/2PC协调） | **键值访问**：RowKey点查/前缀/范围扫描；<br>**多版本读取**：查询历史版本数据；<br>**TTL自动清理**：日志/时序数据自动过期；<br>**时序/日志/明细**：操作型查询（非分析型） | **列族预定义(建议<10)，列动态(数千)**；<br>**Row Key 热点**：单调递增键导致写入集中在单个Tablet，应使用域散列或反转（**原理同 HBase RowKey 热点**）；<br>无内建二级索引<br>**场景说明**：①时序=高频写入+按时间范围检索原始数据（非聚合）；②日志=实时追加+按ID/时间查询近期日志；③明细=存储原始事件+按主键快速检索。**Row Key有序存储使范围查询无需索引**<br>**版本/TTL说明**：多版本=每个单元格保存多个时间戳版本（如传感器历史读数），可查询历史数据；TTL=按列族配置自动过期时间（如日志保留7天），无需手动清理 |
| 图       | **Neo4j**                 | Database        | 图(按 Label)    | 节点/关系      | 属性           | 属性/全文/空间索引                                                                                    | 事务                                                                                                 | 图遍历、路径、推荐                                                                                                 | **关系爆炸要分层/聚合；避免超大超级节点**                                                                                                                                                                                                                                                                                      |
| OLAP 列式 | **ClickHouse**            | Database        | Table         | Row(列式)    | Column       | 列式+排序键(稀疏索引)；<br>❄️不可变Part；<br>🔴无WAL                                                         | ⚠️轻量事务(v22.8+)：单表INSERT原子性；<br>🔴无MVCC；🔴无OLTP事务；<br>最终一致（副本异步）                                    | 大扫描/聚合、按排序/分区裁剪                                                                                           | **①主键=排序键**：ORDER BY 决定数据物理排序（非唯一约束），必须按查询模式设计，否则全表扫描；<br>**②批量写入**：单行写入创建大量Part文件导致查询慢（10ms→10s），建议10000+行/批次或用Buffer表                                                                                                                                                                                      |
| OLAP 列式 | **Apache Doris**          | Database        | Table         | Row(列式)    | Column       | 列式+前缀索引+倒排索引；<br>🔄可变(Delete Bitmap)；✅RocksDB WAL                                             | ⚠️轻量事务：批量导入原子性；<br>🔴无MVCC；🔴无OLTP事务；<br>最终一致                                                      | HTAP、实时报表、高效UPDATE/DELETE                                                                                 | **①Delete Bitmap开销**：频繁更新导致Compaction压力；<br>**②主键表限制**：主键表(Primary Key)支持更新但写入性能降低30-50%                                                                                                                                                                                                                     |
| OLAP 列式 | **Snowflake**             | Account         | Table         | Row(列式)    | Column       | 微分片+统计裁剪、聚簇键；<br>❄️不可变微分片；Metadata Log                                                        | ✅ACID事务(仓库语义)；<br>✅MVCC(快照隔离)；<br>Time Travel                                                      | ELT、分析、半结构化                                                                                               | 无传统索引；合理分区/聚簇与仓库大小                                                                                                                                                                                                                                                                                           |
| OLAP/时序 | Druid                     | -               | Datasource    | Row        | 维度/度量        | 列式+倒排/位图；时间分区                                                                                 | 最终一致                                                                                               | 实时/近实时聚合、roll-up                                                                                          | 维度基数过高影响内存；分段与保留策略                                                                                                                                                                                                                                                                                           |
| 搜索      | **Elasticsearch**         | -               | Index(≈Table) | Document   | Field        | 倒排索引+列存字典；<br>✅Translog(WAL)                                                                  | 最终一致                                                                                               | 关键词/过滤/聚合、日志搜索                                                                                            | **①Mapping爆炸**：Mapping=字段结构定义，动态字段无限增长导致集群OOM（如日志中error_code_1001/1002...动态key），需禁用dynamic或用flattened类型；<br>**②高基数聚合**：对user_id等高基数字段聚合消耗大量内存，建议用composite聚合或预聚合                                                                                                                                             |
| 时序      | InfluxDB                  | Database/Bucket | Measurement   | Point      | Field/Tag    | TSM+TSI；<br>✅WAL                                                                              | 最终一致                                                                                               | 时间线写入、按 tag 过滤                                                                                            | tag 设计决定可查性；保留策略/压缩                                                                                                                                                                                                                                                                                          |
| 列式仓库    | **BigQuery(GCP)**         | Dataset         | Table         | Row(列式)    | Column       | 列式+列压缩；分区/聚簇；<br>❄️不可变Capacitor文件；🔴无WAL                                                      | ✅ ACID 事务<br>（多语句事务支持）                                                                       | 海量扫描/SQL 分析                                                                                               | **①费用按扫描量**：SELECT * 或未分区表全表扫描成本暴增（1TB=$5），必须用分区裁剪；<br>**②分区/聚簇设计**：按日期分区+高基数字段聚簇（如user_id），否则扫描量大10-100倍                                                                                                                                                                                                    |

#### 核心观察：WAL 与存储架构的关系

**有 WAL 的数据库（需要保护内存数据）：**
- **关系型/NewSQL**：MySQL(Redo Log)、PostgreSQL(WAL)、TiDB(Raft Log)、Spanner(Paxos Log)
- **文档型**：MongoDB(Journal)、Firestore(Paxos Log)
- **键值**：Redis(AOF)、DynamoDB(内部 WAL)
- **列族宽表**：Cassandra(CommitLog)、HBase(HLog)、Bigtable(Colossus Log)
- **OLAP**：Apache Doris(RocksDB WAL)、StarRocks(RocksDB WAL)、Snowflake(Metadata Log)
- **搜索/时序**：Elasticsearch(Translog)、InfluxDB(WAL)

**无 WAL 的数据库（直接写文件或依赖外部保障）：**
- **ClickHouse**：无 MemTable，直接生成不可变 Part，依赖客户端重试和副本
- **BigQuery**：托管服务，依赖 Google 基础设施保障
- **Druid**：依赖外部消息队列(Kafka)保障数据可靠性

**关键结论：**
- ✅ **有 MemTable（内存缓冲） → 必须有 WAL**（保护内存数据不丢失）
- ✅ **LSM-Tree 架构 ≠ 无 WAL**（大多数 LSM-Tree 都有 WAL）
- ✅ **不可变存储 ≠ 无 WAL**（HBase/Cassandra 都是不可变存储但有 WAL）
- ⚠️ **无 WAL 的代价**：牺牲单次写入可靠性，适合批量导入和可重试场景


#### LSM-Tree 更新/删除机制对比

**传统 LSM-Tree（HBase/Cassandra/ClickHouse）：**
```
更新/删除流程:
1. 写入 WAL（如果有）
2. 写入 MemTable（追加新版本或删除标记）
3. Flush → 生成不可变 SSTable/Part
4. 读取时合并多个文件，跳过删除标记
5. Compaction 时物理删除旧版本和删除标记

特点:
- ❄️ 数据文件完全不可变
- 🔴 更新/删除需要重写整个文件（Compaction 时）
- ⚠️ 读取需要合并多个文件（读放大）
- ✅ 写入性能极高（纯追加）
```

**改进版 Delete Bitmap（Apache Doris/StarRocks）：**
```
更新/删除流程:
1. 写入 WAL (RocksDB WAL)
2. 标记 Delete Bitmap（逻辑删除旧行）
3. 写入 MemTable（新数据）
4. Flush → 生成新 Segment + 持久化 Delete Bitmap
5. 读取时应用 Bitmap 过滤
6. Compaction 时根据 Bitmap 物理删除

特点:
- ❄️ Segment 文件不可变
- 🔄 Delete Bitmap 可变（逻辑删除）
- ✅ 更新/删除无需立即重写文件
- ⚠️ 读取需要应用 Bitmap 过滤（轻量级）
- ✅ 支持高效 UPDATE/DELETE（HTAP 场景）
```

**关键差异对比：**

| 维度 | 传统 LSM-Tree | Delete Bitmap 改进版 |
|------|--------------|---------------------|
| **数据文件** | 不可变 SSTable/Part | 不可变 Segment |
| **删除方式** | 写入删除标记到新文件 | 标记 Delete Bitmap |
| **更新方式** | 写入新版本到新文件 | Bitmap 标记 + 新 Segment |
| **读取开销** | 合并多个文件 | 应用 Bitmap 过滤 |
| **Compaction 频率** | 必须频繁执行 | 可延迟执行 |
| **UPDATE/DELETE 性能** | 慢（需等 Compaction） | 快（立即生效） |
| **适用场景** | OLAP 批量导入 | HTAP 混合负载 |
| **代表数据库** | HBase/Cassandra/ClickHouse | Apache Doris/StarRocks |

**为什么 Apache Doris/StarRocks 需要这个改进？**
- HTAP 场景需要频繁更新/删除
- 传统 LSM-Tree 的 Compaction 延迟太高
- Delete Bitmap 实现了"逻辑删除 + 延迟物理删除"
- 权衡：读取时需要额外的 Bitmap 过滤开销

#### 重要概念澄清：什么是"宽列"？

> **核心结论**：HBase/Bigtable/Cassandra 是"列族（Column Family）数据库"，这里的"宽列"并不是"有很多列族"的意思，而是"在同一列族下可以有极多、动态的列（列限定符，Qualifier），而且表是稀疏的，没写入的列不占空间"。因此"列族尽量少（通常＜10）"与"宽列表"并不矛盾：**少列族、每个列族里可以有很多动态列，才是推荐模型**。

##### 为何列族要少？

**列族是物理聚簇与资源管理单元**，每个列族都有：
- 独立的 MemStore（内存缓冲）
- 独立的 HFile/SSTable（磁盘文件）
- 独立的 BlockCache 策略
- 独立的压缩/编码配置
- 独立的 TTL/版本控制
- 独立的 Block 大小设置

**列族一多会导致**：
- ✅ flush/compaction/IO 调度的开销与复杂度显著上升
- ✅ 读写放大：一次写入涉及多个列族时，触发多路内存刷盘与后续合并
- ✅ Region 层面的放大：每个 Region 持有该表所有列族的 Store，列族越多，管理和合并的组合成本上升

##### 什么是"列族"和"列"？

**列族（Column Family, CF）**：
- 表设计时**必须预先定义**，稳定且少变
- 决定物理聚簇与一组存储策略（压缩、TTL、版本、Block 大小、缓存等）
- 可把"经常一起访问、生命周期相近"的列放在一个列族
- **建议数量**：1-3 个列族就够用，很少超过 5-8 个

**列（列限定符，Qualifier）**：
- **不需要预定义**，按需出现，天然支持"动态列""稀疏列"
- 命名通常是"cf:qualifier"，例如 info:name、info:age
- **宽列能力体现在这里**：同一行里可有非常多的 qualifier，未写入的列不占空间

##### 为何叫"宽列表"？

强调"同一张表里可能存在非常多的列（qualifier），并且每行拥有的列集合可以不同"。这种"宽且稀疏"的列集合，与关系型的固定列数完全不同。

**宽列是"列族之下的列丰富度"，不是"列族数量很多"**。工程上恰恰反过来：**列族应少，列可以多**。

##### 类比理解：

```
列族 = 文件夹（少）
列 = 文件夹里的文件（多）

HBase 表
├─ 列族1: info（文件夹1）
│  ├─ name（文件）
│  ├─ age（文件）
│  ├─ email（文件）
│  └─ ... 可以有数千个列（文件）
│
└─ 列族2: tags（文件夹2）
   ├─ tag:golang（文件）
   ├─ tag:python（文件）
   ├─ tag:java（文件）
   └─ ... 可以有数千个列（文件）

建议：文件夹少（<10个），但每个文件夹里可以有很多文件
```

##### 实际例子：

```
电商用户行为分析表（只有 2 个列族）

业务场景：
- 需要存储用户基本信息（固定几个字段）
- 需要记录用户浏览过的所有商品（数量不确定，可能几个，可能几千个）
- 每个用户浏览的商品不同（稀疏）

RowKey: user#001
├─ 列族: profile（基础信息 - 固定字段，经常一起读取）
│  ├─ profile:name = "Alice"
│  ├─ profile:age = 25
│  ├─ profile:city = "Beijing"
│  └─ profile:vip_level = "Gold"
│
└─ 列族: browsed_products（浏览记录 - 动态字段，每个用户不同）
   ├─ browsed_products:iphone15 = "2025-11-10 10:30:00"
   ├─ browsed_products:macbook_pro = "2025-11-10 11:15:00"
   ├─ browsed_products:airpods = "2025-11-10 12:00:00"
   └─ ... 用户可能浏览了 1000+ 个商品

RowKey: user#002
├─ 列族: profile
│  ├─ profile:name = "Bob"
│  ├─ profile:age = 30
│  └─ profile:city = "Shanghai"
│
└─ 列族: browsed_products（Bob 只浏览了 3 个商品）
   ├─ browsed_products:nike_shoes = "2025-11-09 14:20:00"
   ├─ browsed_products:adidas_jacket = "2025-11-09 15:00:00"
   └─ browsed_products:puma_bag = "2025-11-09 16:30:00"

这就是"宽列"的实际应用：
- 只有 2 个列族（少）：profile 和 browsed_products
- 但 browsed_products 列族下可以有数千个动态列（宽）：每个商品是一列
- 每个用户浏览的商品不同（稀疏）：Alice 浏览了 1000+ 个，Bob 只浏览了 3 个
- 不需要预定义所有商品列：新商品上架后，用户浏览时自动创建新列
- 未浏览的商品不占存储空间：Bob 没浏览 iphone15，就不存储这一列

为什么这样设计？
- profile 列族：4 个固定字段，经常一起读取，放在一个列族
- browsed_products 列族：动态增长，每个用户不同，TTL 可以设置为 30 天自动过期
- 查询效率：读取用户基本信息时，只访问 profile 列族，不会读取大量浏览记录
- 存储效率：稀疏存储，只存储实际浏览的商品，不浪费空间
```

---

**与传统关系型数据库的对比：**

```
传统 MySQL（关系型数据库）：

users 表：
+--------+-------+-----+----------+-----------+
| user_id| name  | age | city     | vip_level |
+--------+-------+-----+----------+-----------+
| 001    | Alice | 25  | Beijing  | Gold      |
| 002    | Bob   | 30  | Shanghai | NULL      |
+--------+-------+-----+----------+-----------+

browsed_products 表（需要单独的关联表）：
+--------+------------------+---------------------+
| user_id| product_id       | browsed_time        |
+--------+------------------+---------------------+
| 001    | iphone15         | 2025-11-10 10:30:00 |
| 001    | macbook_pro      | 2025-11-10 11:15:00 |
| 001    | airpods          | 2025-11-10 12:00:00 |
| 002    | nike_shoes       | 2025-11-09 14:20:00 |
| 002    | adidas_jacket    | 2025-11-09 15:00:00 |
+--------+------------------+---------------------+

问题：
- 需要 2 张表
- 查询用户浏览记录需要 JOIN
- 每个浏览记录都要存储 user_id（冗余）
- 如果用户浏览了 1000 个商品，就有 1000 行记录


HBase（宽列数据库）：

只需要 1 张表，所有数据在一行里：
RowKey: user#001（相当于 MySQL 的主键 user_id）
├─ 列族: profile（相当于 users 表的列）
│  ├─ profile:name = "Alice"
│  ├─ profile:age = 25
│  ├─ profile:city = "Beijing"
│  └─ profile:vip_level = "Gold"
│
└─ 列族: browsed_products（相当于 browsed_products 表，但在同一行里）
   ├─ browsed_products:iphone15 = "2025-11-10 10:30:00"
   ├─ browsed_products:macbook_pro = "2025-11-10 11:15:00"
   └─ browsed_products:airpods = "2025-11-10 12:00:00"

优势：
- 只需要 1 张表
- 不需要 JOIN（所有数据在一行里）
- 不需要重复存储 user_id
- 1000 个浏览记录就是 1000 个列，不是 1000 行
```

**核心概念对应关系：**

| 传统数据库概念             | HBase 概念                  | 说明                           |
| :------------------ | :------------------------ | :--------------------------- |
| **表（Table）**        | **表（Table）**              | 相同                           |
| **行（Row）**          | **行（Row）**                | 相同，但 HBase 的一行可以包含多个"子表"（列族） |
| **主键（Primary Key）** | **RowKey**                | 相同，唯一标识一行                    |
| **列（Column）**       | **列族:列限定符（CF:Qualifier）** | HBase 的列是两层结构                |
| **关联表（JOIN）**       | **列族（Column Family）**     | HBase 用列族把关联表"压扁"到一行里        |

**列族的本质：把多张关联表合并到一行里**

```
MySQL 需要 2 张表 + JOIN：
users 表 + browsed_products 表

HBase 只需要 1 张表的 1 行：
user#001 行 = profile 列族（users 表） + browsed_products 列族（browsed_products 表）
```

**为什么叫"列族"？**

因为它是一"族"列的集合：
- profile 列族 = {name, age, city, vip_level} 这一族列
- browsed_products 列族 = {iphone15, macbook_pro, airpods, ...} 这一族列

**一句话理解：**
- **RowKey** = MySQL 的主键（标识哪一行）
- **列族** = MySQL 的关联表（但不需要 JOIN，直接在一行里）
- **列限定符** = MySQL 的列名（但可以动态添加，不需要预定义）

**实际查询对比：**

```sql
-- MySQL（需要 JOIN）
SELECT u.name, u.age, b.product_id, b.browsed_time
FROM users u
LEFT JOIN browsed_products b ON u.user_id = b.user_id
WHERE u.user_id = '001';

-- HBase（不需要 JOIN，直接读一行）
Get 'user#001'
  - 列族 profile: 返回 {name, age, city, vip_level}
  - 列族 browsed_products: 返回 {iphone15, macbook_pro, airpods, ...}
```

**关键点总结：**
1. **RowKey** = 主键（标识哪个用户）
2. **列族** = 把关联表"压扁"到一行里（不需要 JOIN）
3. **列限定符** = 动态的列名（可以有数千个）
4. **宽列的"宽"** = 一个列族下可以有数千个动态列

---

##### 如何落地设计？

**列族分组**：
- 按"访问共性"和"生命周期共性"来分组
- 例如把冷热、TTL 不同或一起读取的列放到各自列族中
- 通常 1-3 个列族就够用，很少超过 5-8 个

**行键设计**：
- 防热点：加盐/散列前缀、复合键
- 保障相邻访问局部性：按时间分段放在后缀、或"高基数维度在前、低基数维度在后"

**列使用**：
- 在同一列族里灵活新增 qualifier，不必迁表
- 用版本/TTL 管理时间序保留

**二级索引**：
- HBase 原生没有通用二级索引
- 需要外部方案（如 Phoenix）或冗余写（反向索引表）

##### 一句话记忆：

- **列族**：少而稳（物理与策略维度）
- **列（qualifier）**：多而活（逻辑/动态维度）
- **宽列的"宽"**：指的是"列族之下的动态列很多"，不是"列族很多"

---

### 数据库本质特性与选型速查表

| 类型      | 代表实例                      | 典型延迟/吞吐侧重点       | Schema 与稀疏      | 范围扫描能力                        | 前缀扫描支持                                                    | 分片策略                                | 二级索引能力                   | 事务与一致性级别                                                                                           | 典型存储布局         | 费用/资源模型            | 常见应用场景                                                                                                    | 反模式示例                                                                                                                                                                                                                                                                                                        |
| ------- | ------------------------- | ---------------- | --------------- | ----------------------------- | --------------------------------------------------------- | ----------------------------------- | ------------------------ | -------------------------------------------------------------------------------------------------- | -------------- | ------------------ | --------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| 关系型     | **MySQL**                 | 低延迟 OLTP         | 强 schema        | 支持基于索引的范围                     | ✅ LIKE 'prefix%'<br>(B-Tree 索引) <br>✅ 索引前缀匹配              | 单机（无分片）                             | 强（B-Tree 等）              | 强事务、强一致、MVCC(InnoDB)                                                                               | 行式             | 计算与存储耦合（自管/云RDS）   | 交易系统、订单、财务                                                                                                | 大事务/长事务锁、全表扫描、无索引模糊匹配                                                                                                                                                                                                                                                                                        |
| 关系型     | PostgreSQL                | 低延迟 OLTP/混合      | 强 schema        | 索引范围与高级索引裁剪                   | ✅ LIKE 'prefix%'<br>(B-Tree 索引) <br>✅ 索引前缀匹配              | 单机（无分片）                             | 很强（B-Tree/GiST/GIN/BRIN） | 强事务、MVCC                                                                                           | 行式             | 计算与存储耦合            | 复杂查询、地理/全文、分析+事务                                                                                          | 统计过期致优化器误判、跨库外键、高并发大 join 未优化                                                                                                                                                                                                                                                                                |
| 关系型     | **AlloyDB(GCP)**          | 低延迟 OLTP + 快速分析  | 强 schema        | 索引范围 + 列式加速                   | ✅ LIKE 'prefix%'<br>(继承 PG) <br>✅ 索引前缀匹配                  | 单机（主从复制）                            | 很强（B-Tree + 列式引擎）        | 强事务、MVCC；PostgreSQL 兼容                                                                             | 行式 + 列式混合      | 计算存储分离（托管）         | 混合负载、OLTP + 轻量 OLAP                                                                                       | **①成本高**：比 Cloud SQL 贵 2-3 倍；<br>**②锁定 GCP**：仅 GCP 托管，无自建选项；<br>**③分析查询优化**：列式引擎加速聚合，但不如专用 OLAP                                                                                                                                                                                                              |
| NewSQL  | **TiDB**                  | 低延迟 OLTP + 分布式吞吐 | 强 schema        | 索引范围、分布式执行                    | ✅ 索引前缀<br>(分布式) <br>✅ 支持索引前缀                              | **Range**（默认，自动分裂）；<br>支持：Range     | 强（分布式二级索引）               | 分布式 ACID、MVCC(Percolator)                                                                          | 行式为主（可叠加列存）    | 计算存储分离（组件化）        | HTAP、在线交易+近实时分析                                                                                           | 热点行/自增主键需打散；合理 TiFlash/统计                                                                                                                                                                                                                                                                                    |
| NewSQL  | **Spanner(GCP)**          | 低延迟 OLTP + 全球分布  | 强 schema        | 索引范围、全球分布式执行                  | ✅ 索引前缀<br>(分布式) <br>✅ 支持索引前缀                              | **Range**（自动分裂）                     | 强（分布式二级索引）               | 分布式 ACID、MVCC(TrueTime)；External Consistency                                                       | 行式             | 计算存储分离（托管）         | 全球分布式事务、金融/广告                                                                                             | **①TrueTime 依赖**：需要 GPS+原子钟硬件支持；<br>**②跨区延迟**：全球一致性以延迟为代价（跨洲写入可达100-500ms）；<br>**③热点键**：单调递增主键导致热点（原理同 TiDB）                                                                                                                                                                                                 |
| 文档型     | **MongoDB**               | 低延迟 OLTP（文档）     | 弱/灵活 schema     | 依赖索引的范围/前缀                    | ✅ 索引前缀 <br>✅ B-Tree 索引支持                                  | **Hash**（默认）；<br>支持：Hash/Range/Zone | 强（多键/稀疏/TTL）             | 单文档原子；集合级、MVCC(WiredTiger)                                                                         | 行式文档 + B-Tree  | 计算与存储耦合（托管可分）      | 文档点查、聚合管道、灵活 schema                                                                                       | **①高基数字段+缺索引→爆表**：_id 是主键(ObjectId自动生成)，user_id/order_id 等业务字段需手动建索引，否则全表扫描；<br>**②Shard Key 热点**：自增ID/时间戳/低基数字段作分片键导致数据写入集中在单个Shard，应选高基数字段或哈希分片（**原理同 HBase RowKey/DynamoDB Partition Key 热点**）                                                                                                            |
| 文档型     | **Firestore Native(GCP)** | 低延迟 + 实时推送       | 弱/灵活 schema     | 依赖索引的范围                       | ✅ 自动索引前缀 <br>✅ 自动索引支持                                     | 自动分区（透明）                            | 强（自动索引）                  | 强一致（Paxos）；Entity Group 事务；MVCC(多版本)                                                               | 文档 + Megastore | 计算存储分离（Serverless） | 实时订阅、移动/Web 应用、离线支持                                                                                       | **①Entity Group 限制**：事务限制在 25 个实体/秒；<br>**②查询能力受限**：无 JOIN、无复杂聚合，不适合后端复杂业务；<br>**③成本模型**：按读写次数计费，高频访问成本高                                                                                                                                                                                                     |
| 键值      | Redis                     | 极低延迟             | 无 schema        | 不擅长范围（少数组件支持）                 | ⚠️ SCAN + 模式匹配 <br>⚠️ 非索引，性能差                             | 单机（无分片）；<br>Cluster：Hash（槽位）        | 无通用二级索引                  | 最终/主从；Lua 原子                                                                                       | 内存结构（跳表/哈希等）   | 内存为主；计算与存储耦合       | 极低延迟缓存、计数、队列                                                                                              | 不要当永久存储；内存淘汰与持久化取舍                                                                                                                                                                                                                                                                                           |
| 键值/文档   | **DynamoDB**              | 低延迟 + 高吞吐        | 弱/灵活 schema     | 分区+排序键范围                      | 🔴 Partition Key<br>✅ Sort Key (begins_with)，只支持 Sort Key | **Hash**（Partition Key，固定）          | 有（GSI/LSI）但需成本权衡         | 可调一致；分区容量                                                                                          | LSM 风格存储       | 计算存储分离（Serverless） | 单表大规模、高可用、事件驱动                                                                                            | **Partition Key 热点**：自增ID/时间戳作分区键导致写入集中在单分区，应选高基数字段或复合键（**原理同 HBase RowKey 热点**）；GSI 写放大与费用                                                                                                                                                                                                                  |
| 列族宽表    | Cassandra                 | 低延迟写入 + 高吞吐      | 半灵活（宽表稀疏）       | 分区内按聚簇键强范围                    | 🔴 Partition Key<br>✅ Clustering Key， 只支持 Clustering Key  | **Hash**（Partition Key，固定）          | 有，但谨慎（场景受限）              | 可调一致(ONE/QUORUM/ALL)                                                                               | LSM + SSTable  | 计算存储耦合（集群）         | 按分区键点查/时间序                                                                                                | 每分区建议<100MB、<100k 项；<br>**Partition Key 热点**：低基数/单调递增键导致数据倾斜（**原理同 HBase RowKey 热点**）；<br>二级索引用慎                                                                                                                                                                                                             |
| 列族宽表    | **HBase**                 | 低延迟点查/范围         | 半灵活（CF 固定、列动态）  | 行键字典序强范围                      | ✅ Row Key 前缀<br>(Range 分片) <br>✅ 原生支持                     | **Range**（Row Key，自动分裂）             | 原生无通用二级索引                | 单行原子；无跨行事务                                                                                         | 列族聚簇 + LSM     | 计算存储分离（HDFS）       | 行键点查/前缀/范围扫描                                                                                              | **列族尽量少(<10)，列可以多(数千)**；<br>**RowKey 热点**：时间戳/自增ID作RowKey导致写入集中在单个Region，应使用前缀散列或反转（**原理同 DynamoDB Partition Key/MongoDB Shard Key 热点**）；<br>二级索引需外部（**Phoenix 查询引擎**）                                                                                                                                       |
| 列族宽表    | **Bigtable(GCP)**         | 低延迟 + 稳定吞吐       | 半灵活（CF 固定、列动态）  | 行键有序强范围<br>**（Row Key 物理连续）** | ✅ Row Key 前缀<br>(Range 分片) <br>✅ 原生支持                     | **Range**（Row Key，自动分裂）             | 无内建二级索引                  | **单行原子**（依赖Colossus同步复制）；<br>**单集群强一致**（读写同一副本集）；<br>**跨集群最终一致**（异步复制）；<br>**无跨行事务**（无Paxos/2PC协调） | 列族聚簇 + SSTable | 计算存储分离（托管）         | **键值访问**：RowKey点查/前缀/范围扫描；<br>**多版本读取**：查询历史版本数据；<br>**TTL自动清理**：日志/时序数据自动过期；<br>**时序/日志/明细**：操作型查询（非分析型） | **列族预定义(建议<10)，列动态(数千)**；<br>**Row Key 热点**：单调递增键导致写入集中在单个Tablet，应使用域散列或反转（**原理同 HBase RowKey 热点**）；<br>无内建二级索引<br>**场景说明**：①时序=高频写入+按时间范围检索原始数据（非聚合）；②日志=实时追加+按ID/时间查询近期日志；③明细=存储原始事件+按主键快速检索。**Row Key有序存储使范围查询无需索引**<br>**版本/TTL说明**：多版本=每个单元格保存多个时间戳版本（如传感器历史读数），可查询历史数据；TTL=按列族配置自动过期时间（如日志保留7天），无需手动清理 |
| 图       | **Neo4j**                 | 低延迟遍历            | 灵活 schema       | 图遍历（非键范围）                     | ✅ 属性索引前缀 <br>✅ 支持字符串前缀                                    | 单机（无分片）；<br>企业版：图分区                 | 属性/全文/空间索引               | 事务                                                                                                 | 面向图的邻接存储       | 计算存储耦合/可托管         | 图遍历、路径、推荐                                                                                                 | **关系爆炸要分层/聚合；避免超大超级节点**                                                                                                                                                                                                                                                                                      |
| 搜索      | **Elasticsearch**         | 低延迟检索 + 高吞吐写入    | 弱/灵活 schema（映射） | 不支持键序范围；倒排过滤/聚合               | ✅ prefix 查询<br>(倒排索引) <br>✅ 专门的 prefix 查询                 | **Hash**（默认，文档ID）；<br>支持：自定义路由      | 强（倒排/聚合结构）               | 最终一致                                                                                               | 倒排 + 列式字典      | 计算存储耦合/冷热分层        | 关键词/过滤/聚合、日志搜索                                                                                            | **①Mapping爆炸**：Mapping=字段结构定义，动态字段无限增长导致集群OOM（如日志中error_code_1001/1002...动态key），需禁用dynamic或用flattened类型；<br>**②高基数聚合**：对user_id等高基数字段聚合消耗大量内存，建议用composite聚合或预聚合                                                                                                                                             |
| 时序      | InfluxDB                  | 低延迟写入 + 快速查询     | 强 schema（列式）    | 时间线写入、按 tag 过滤                | ✅ Tag 前缀 <br> ✅ 支持 Tag 前缀                                 | 时间分段（自动）                            | 倒排/位图                    | 最终一致                                                                                               | TSM + TSI      | 存算分离（多进程角色）        | 时间线写入、按 tag 过滤                                                                                            | tag 设计决定可查性；保留策略/压缩                                                                                                                                                                                                                                                                                          |
| OLAP 列式 | **ClickHouse**            | 批量高吞吐 + 快速聚合     | 强 schema        | 分区/排序裁剪、索引标记                  | ✅ ORDER BY 前缀<br>(排序键优化) <br>✅ 依赖排序键设计                    | **Hash**（默认）；<br>支持：Hash/Range/手动   | 索引为稀疏/数据跳跃标记             | ⚠️轻量事务(v22.8+)：单表INSERT原子性；<br>🔴无MVCC；🔴无OLTP事务；<br>最终一致（副本异步）                                    | 列式 + 向量化       | 计算存储耦合/可解耦         | 大扫描/聚合、按排序/分区裁剪                                                                                           | **①主键=排序键**：ORDER BY 决定数据物理排序（非唯一约束），必须按查询模式设计，否则全表扫描；<br>**②批量写入**：单行写入创建大量Part文件导致查询慢（10ms→10s），建议10000+行/批次或用Buffer表                                                                                                                                                                                      |
| OLAP 列式 | **Apache Doris**          | 批量高吞吐 + 快速聚合     | 强 schema        | 分区/排序裁剪、索引标记                  | ✅ 前缀索引 <br>✅ 前缀索引支持                                       | **Hash**（默认）；<br>支持：Hash/Range      | 索引为稀疏/数据跳跃标记             | ⚠️轻量事务：批量导入原子性；<br>🔴无MVCC；🔴无OLTP事务；<br>最终一致                                                      | 列式 + 向量化       | 计算存储耦合/可解耦         | HTAP、实时报表、高效UPDATE/DELETE                                                                                 | **①Delete Bitmap开销**：频繁更新导致Compaction压力；<br>**②主键表限制**：主键表(Primary Key)支持更新但写入性能降低30-50%                                                                                                                                                                                                                     |
| OLAP 列式 | **Snowflake**             | 弹性吞吐 + 快速聚合      | 强 schema        | 微分片统计裁剪、聚簇键                   | ⚠️ CLUSTER BY 优化<br>(非传统前缀) <br>⚠️ 间接支持                   | 微分片（自动，透明）                          | 无传统二级索引                  | ✅ACID事务(仓库语义)；<br>✅MVCC(快照隔离)；<br>Time Travel                                                      | 列式 + 存算分离      | 存算分离/弹性计费          | ELT、分析、半结构化                                                                                               | 无传统索引；合理分区/聚簇与仓库大小                                                                                                                                                                                                                                                                                           |
| OLAP/时序 | Druid                     | 近实时摄取 + 快速聚合     | 强 schema（列式）    | 时间分段 + 倒排/位图裁剪                | ✅ 维度前缀 <br>✅ 倒排索引支持                                       | 时间分段（自动）                            | 倒排/位图                    | 最终一致                                                                                               | 列式 + 倒排/位图     | 存算分离（多进程角色）        | 实时/近实时聚合、roll-up                                                                                          | 维度基数过高影响内存；分段与保留策略                                                                                                                                                                                                                                                                                           |
| 列式仓库    | **BigQuery(GCP)**         | 批量高吞吐 + 快速聚合     | 强 schema        | 分区/聚簇裁剪、列式压缩                  | ⚠️ CLUSTER BY 优化<br>(非传统前缀) <br>⚠️ 间接支持                   | 微分片（自动，透明）                          | 无传统二级索引                  | ✅ ACID 事务<br>（多语句事务支持）                                                                       | 列式 + 存算分离      | 存算分离/按扫描量计费        | 海量扫描/SQL 分析                                                                                               | **①费用按扫描量**：SELECT * 或未分区表全表扫描成本暴增（1TB=$5），必须用分区裁剪；<br>**②分区/聚簇设计**：按日期分区+高基数字段聚簇（如user_id），否则扫描量大10-100倍                                                                                                                                                                                                    |

**前缀扫描支持：**
- ✅ 完全支持: HBase, Bigtable, MySQL, PostgreSQL, ClickHouse
- ⚠️ 部分支持: DynamoDB (仅 Sort Key), Cassandra (仅 Clustering Key)
- 🔴 不支持: DynamoDB Partition Key, Cassandra Partition Key


#### 前缀扫描支持的底层机制差异

**核心原理：分片策略决定前缀扫描能力**

**✅ 支持前缀扫描的数据库（Range 分片 + 有序存储）：**

| 数据库 | 分片策略 | 存储顺序 | 前缀扫描机制 | 性能特点 |
|--------|---------|---------|------------|---------|
| **HBase** | Range（Row Key 自动分裂） | 字典序 | 1. 根据前缀定位 Region<br>2. 在 Region 内顺序扫描<br>3. 遇到不匹配前缀停止 | O(log N) 定位 + O(M) 扫描 |
| **Bigtable** | Range（Row Key 自动分裂） | 字典序 | 同 HBase，数据物理连续 | 小范围查询极快 |
| **ClickHouse** | Hash/Range（可选） | ORDER BY 列 | 1. 稀疏索引定位 Block<br>2. 按排序键前缀过滤<br>3. 跳过不匹配的 Part | 依赖排序键设计 |
| **MySQL** | 单机（无分片） | B-Tree 索引 | LIKE 'prefix%' 利用索引 | 索引扫描，选择性决定性能 |

**🔴 不支持前缀扫描的数据库（Hash 分片）：**

| 数据库 | 分片策略 | 为什么不支持 | 替代方案 |
|--------|---------|------------|---------|
| **Cassandra** | Hash（Partition Key） | 前缀无法映射到特定节点<br>hash("addr_1") ≠ hash("addr_2") | ✅ Clustering Key 支持前缀<br>（分区内有序） |
| **DynamoDB** | Hash（Partition Key） | 同 Cassandra | ✅ Sort Key 支持 begins_with<br>（分区内有序） |
| **Elasticsearch** | Hash（文档 ID） | 倒排索引不基于 Key 顺序 | 使用 prefix 查询（倒排索引） |

**关键差异示例：**

```
场景: 查询 Row Key 前缀 "address_hash_001#"

HBase/Bigtable (Range 分片):
  Region 1: [address_hash_001#...] ← 前缀定位到这里
  Region 2: [address_hash_500#...]
  Region 3: [address_hash_999#...]
  
  执行: 只扫描 Region 1，高效 ✅

Cassandra/DynamoDB (Hash 分片):
  Node A: hash("address_hash_001#tx_1") = 0x1a2b
  Node B: hash("address_hash_001#tx_2") = 0x9f8e
  Node C: hash("address_hash_001#tx_3") = 0x3c4d
  
  执行: 必须扫描所有节点，低效 ❌
```

**设计权衡：**

| 特性       | Range 分片        | Hash 分片    |
| -------- | --------------- | ---------- |
| **前缀扫描** | ✅ 支持            | 🔴 不支持     |
| **范围查询** | ✅ 高效            | 🔴 全表扫描    |
| **负载均衡** | ⚠️ 需要设计（避免热点）   | ✅ 自动均衡     |
| **热点问题** | 🔴 单调递增 Key 易热点 | ✅ 哈希天然打散   |
| **扩容**   | ⚠️ 需要分裂/迁移      | ✅ 一致性哈希简单  |
| **适用场景** | 时序/日志/需要范围查询    | 高并发写入/全球分布 |

**结论：**
- **前缀扫描 = Range 分片 + 有序存储**
- HBase/Bigtable 选择 Range 分片，牺牲负载均衡换取查询能力
- Cassandra/DynamoDB 选择 Hash 分片，牺牲查询能力换取负载均衡
- 这是**架构设计的根本权衡**，不是功能缺失

#### MVCC 实现机制深度对比表

| 数据库                 | MVCC 支持          | 版本存储位置                    | 版本标识                    | 可见性判断             | 垃圾回收机制               | 隔离级别支持               | 读写特性             | 典型性能影响         | 适用场景            |
| ------------------- | ---------------- | ------------------------- | ----------------------- | ----------------- | -------------------- | -------------------- | ---------------- | -------------- | --------------- |
| **MySQL (InnoDB)**  | ✅ 有              | Undo Log（独立表空间）           | 事务 ID（DB_TRX_ID）        | Read View + 版本链遍历 | Purge 线程异步清理         | RC/RR                | 读写不阻塞            | 长事务→Undo Log膨胀 | OLTP 高并发读写      |
| **PostgreSQL**      | ✅ 有              | 数据页内（Tuple）               | xmin/xmax（事务ID）         | 事务快照（xmin/xmax比较） | VACUUM 清理死元组         | RC/RR/Serializable   | 读写不阻塞            | 长事务→表膨胀        | OLTP/OLAP 混合    |
| **AlloyDB(GCP)**    | ✅ 有（继承 PG）       | 数据页内（继承 PG） + 列式缓存        | xmin/xmax（继承 PG）        | 事务快照（继承 PG）       | VACUUM（继承 PG） + 智能清理 | RC/RR/Serializable   | 读写不阻塞            | 列式引擎加速分析查询     | OLTP + 轻量 OLAP  |
| **Oracle**          | ✅ 有              | Undo 表空间                  | SCN（系统变更号）              | Undo 段查询          | 自动 Undo 管理（AUM）      | RC/Serializable      | 读写不阻塞            | Undo 表空间管理     | 企业级 OLTP        |
| **SQL Server**      | ✅ 有              | tempdb 版本存储               | 事务序列号                   | 版本链遍历             | 自动清理 tempdb          | RC/Snapshot          | 读写不阻塞            | tempdb 压力      | 企业级 OLTP        |
| **TiDB**            | ✅ 有 (Percolator) | RocksDB 多版本               | 时间戳（start_ts/commit_ts） | Percolator 快照隔离   | GC Worker 清理旧版本      | SI（快照隔离）             | 分布式读写不阻塞         | 跨节点版本协调开销      | 分布式 HTAP        |
| **Spanner**         | ✅ 有              | Tablet 独立存储               | TrueTime 时间戳            | 时间戳比较（ts <= 读时间戳） | 基于时间（如保留7天）          | External Consistency | 全球一致读            | Tablet 级别扩展    | 全球分布式事务         |
| **MongoDB**         | ✅ 有 (WiredTiger) | Update Chain（链表）          | 时间戳                     | 快照隔离              | 自动清理旧版本              | Snapshot/RC          | 文档级读写不阻塞         | 链表遍历开销         | 文档型 OLTP        |
| **Greenplum**       | ✅ 有 (继承 PG)      | 数据页内（继承 PG）               | xmin/xmax               | 事务快照              | VACUUM（继承 PG）        | RC/RR                | 读写不阻塞            | 表膨胀（继承 PG）     | MPP 数据仓库        |
| **Snowflake**       | ✅ 有（快照隔离）        | S3 不可变文件 + 元数据            | 文件版本 + 元数据              | 元数据版本查询           | Time Travel 过期清理     | Snapshot             | Time Travel 历史查询 | 存储成本（保留历史）     | 云数仓 Time Travel |
| **ClickHouse**      | 🔴 无             | 不可变 Part                  | Part 版本号                | 查询时去重（保留最新）       | 后台 Merge             | 无（批量写）               | 无并发控制            | 更新需重写 Part     | OLAP 批量分析       |
| **Apache Doris**   | 🔴 无             | Delete Bitmap            | 标记删除                    | 查询时过滤删除标记         | Compaction           | 轻量事务                 | 原地更新             | Compaction 开销  | HTAP 实时分析       |
| **StarRocks**      | 🔴 无             | Primary Key              | 标记删除                    | 查询时过滤删除标记         | Compaction           | 轻量事务                 | 原地更新             | Compaction 开销  | HTAP 实时分析       |
| **HBase**           | 🔴 无             | 多版本 Cell（可选）              | 时间戳                     | 读取指定版本            | TTL/版本数限制            | 单行原子                 | 无跨行事务            | 版本数过多影响读       | 大规模 KV 存储       |
| **Cassandra**       | 🔴 无             | 时间戳标记                     | 时间戳                     | 最新时间戳优先           | Compaction           | 可调一致性                | 最终一致             | Tombstone 累积   | 高可用写入           |
| **DynamoDB**        | 🔴 无             | 无多版本                      | 无                       | 单版本               | 无                    | 可调一致性                | 条件写入（乐观锁）        | 热点分区           | Serverless KV   |
| **Redis**           | 🔴 无             | 单版本内存                     | 无                       | 单版本               | 无                    | ⚠️ 轻量事务<br>（MULTI/EXEC，无回滚）                  | Lua 脚本原子性        | 内存限制           | 缓存/队列           |

**MVCC vs 不可变存储 vs 其他机制对比：**

| 机制 | 代表数据库 | 存储类型 | 更新方式 | 并发控制 | 适用场景 |
|------|-----------|---------|---------|---------|---------|
| **MVCC（多版本）** | MySQL, PostgreSQL, TiDB, Spanner | 行式 | 保留多版本，事务可见性判断 | 读写不阻塞 | OLTP 高并发 |
| **MVCC + 不可变文件** | Snowflake | 列式 | S3 不可变文件 + 元数据版本 | Time Travel | 云数仓 |
| **不可变 Part** | ClickHouse, BigQuery | 列式 | 重写整个数据块（Mutation） | 无锁（无并发写） | OLAP 批量分析 |
| **Delete Bitmap** | Apache Doris, StarRocks | 列式 | 标记删除 + 原地更新 | 轻量锁 | HTAP 实时分析 |
| **多版本 Cell** | HBase, Cassandra | 列族 | 时间戳版本，保留多版本 | 无跨行事务 | 大规模 KV 存储 |
| **单版本** | Redis, DynamoDB | KV | 直接覆盖 | 乐观锁/条件写 | 缓存/Serverless |

**核心差异：**
- **MVCC（行式）**：适合高并发事务，读写不阻塞，需要垃圾回收
- **不可变 Part（列式）**：适合批量写入，更新代价高，无需并发控制
- **Delete Bitmap（列式）**：兼顾更新性能和列式优势，适合 HTAP
- **多版本 Cell（列族）**：适合时序数据，版本数可控
- **单版本（KV）**：最简单，适合缓存和简单 KV 场景

> 详细的可变/不可变存储对比见 BigData.md 第 4161 行表格

---

**关键洞察：**
- **MVCC 核心价值**：读写不阻塞，高并发性能，事务隔离
- **OLTP 标配**：MySQL/PostgreSQL/Oracle/SQL Server 都实现完整 MVCC
- **分布式 MVCC**：TiDB（Percolator）、Spanner（TrueTime）解决跨节点一致性
- **OLAP 罕见**：仅 Snowflake/Greenplum 支持，主要用于 Time Travel 而非并发控制
- **NoSQL 多数无 MVCC**：依赖最终一致性、乐观锁、单行原子性
- **版本存储差异**：Undo Log（MySQL）vs 数据页内（PostgreSQL）vs 独立存储（Spanner）
- **性能权衡**：长事务影响（Undo Log 膨胀/表膨胀）vs 无 MVCC 的并发限制

---

#### 列族宽表 vs 列式存储：核心特性对比表

| 维度                | 列族宽表（HBase/Cassandra/Bigtable）                                                                                         | OLAP 列式（ClickHouse/Apache Doris/StarRocks/Snowflake/BigQuery）                                                                                                                                                      |
| ----------------- | ---------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **存储模型**          | 按列族分文件，文件内按行组织                                                                                                         | 每列独立文件，按列组织                                                                                                                                                                                                 |
| **物理布局**          | RowKey → [CF1: col1, col2] [CF2: col3]                                                                                 | col1.bin, col2.bin, col3.bin                                                                                                                                                                                |
| **读取粒度**          | 整个列族（无法只读单列）                                                                                                           | 单列（可精确读取）                                                                                                                                                                                                   |
| **二级索引**          | 🚫 原生不支持（HBase/Bigtable）<br>⚠️ 有但谨慎（Cassandra）                                                                         | ⚠️ 稀疏索引（ClickHouse）<br>⚠️ 倒排索引（Apache Doris/StarRocks）<br>🚫 无传统索引（Snowflake/BigQuery）                                                                                                                             |
| **索引替代**          | Row Key 设计、Phoenix（外部）、应用层索引表                                                                                          | **ClickHouse**：排序键 + 分区裁剪<br>**Apache Doris/StarRocks**：前缀索引 + Rollup<br>**Snowflake/BigQuery**：统计裁剪 + 聚簇                                                                                                          |
| **点查询（主键）**       | ✅ 极快（1-5ms）<br>二分查找 + Bloom Filter + Cache                                                                             | **ClickHouse**：⚠️ 较慢（10-50ms）<br>**Apache Doris/StarRocks**：✅ 快（5-20ms，主键表）<br>**Snowflake/BigQuery**：⚠️ 较慢（50-200ms）                                                                                              |
| **范围查询（主键）**      | **HBase/Bigtable**：✅ 快（10-100ms，跨分区）<br>**Cassandra**：⚠️ 仅分区内（Clustering Key）<br>原因：Range 分片 vs Hash 分片                | **ClickHouse**：✅ 快（排序键优化）<br>**Apache Doris/StarRocks**：✅ 快（前缀索引）<br>**Snowflake/BigQuery**：✅ 快（分区裁剪）                                                                                                              |
| **前缀查询（Row Key）** | **HBase/Bigtable**：✅ 支持（原生前缀扫描）<br>**Cassandra**：🔴 不支持（必须完整 Partition Key）<br>**DynamoDB**：🔴 不支持（必须完整 Partition Key） | **ClickHouse**：✅ 支持（排序键前缀优化）<br>**Apache Doris/StarRocks**：✅ 支持（前缀索引）<br>**BigQuery**：⚠️ 间接支持（CLUSTER BY 优化）                                                                                                       |
| **非主键查询**         | 🔴 慢（秒-分钟级）<br>全表扫描，无索引                                                                                                | **ClickHouse**：✅ 快（列式扫描）<br>**Apache Doris/StarRocks**：✅ 快（倒排索引）<br>**Snowflake/BigQuery**：✅ 快（列式扫描）                                                                                                               |
| **聚合查询**          | 🔴 很慢（分钟级）<br>非列式，无向量化                                                                                                 | **ClickHouse**：✅ 极快（秒级，SIMD）<br>**Apache Doris/StarRocks**：✅ 极快（秒级，物化视图）<br>**Snowflake/BigQuery**：✅ 极快（秒级，并行）                                                                                                     |
| **写入性能**          | ✅ 极快（百万行/秒）<br>纯追加，无需更新索引                                                                                              | **不可变**（ClickHouse/BigQuery/Snowflake）：<br>✅ 批量快（10万+行/秒）<br>🔴 单行慢（10-100ms/行）<br>**可变**（Apache Doris/StarRocks）：<br>⚠️ 中等（1万-10万行/秒）<br>**原因：不可变需后台 Merge，可变需更新 Delete Bitmap**                                  |
| **更新/删除**         | ✅ 快（直接修改）<br>LSM 追加标记                                                                                                  | **不可变**（ClickHouse/BigQuery/Snowflake）：<br>🔴 慢（重写整个 Part/文件）<br>**可变**（Apache Doris/StarRocks）：<br>✅ 快（Delete Bitmap 标记）                                                                                          |
| **压缩率**           | ⚠️ 中等（列族内混合类型）                                                                                                         | ✅ 高（同类型数据连续）                                                                                                                                                                                                |
| **SIMD 优化**       | 🚫 不支持（数据不连续）                                                                                                          | ✅ 支持（向量化执行）                                                                                                                                                                                                 |
| **存储引擎**          | **HBase/Bigtable**：LSM-Tree（❄️不可变 HFile/SSTable + WAL）<br>**Cassandra**：LSM-Tree（❄️不可变 SSTable + CommitLog）            | **ClickHouse**：MergeTree（❄️不可变 Part，无 WAL）<br>**Apache Doris/StarRocks**：Segment + Tablet（🔄可变，RocksDB WAL + Delete Bitmap）<br>**BigQuery**：Capacitor（❄️不可变列式文件，无 WAL）<br>**Snowflake**：PAX（❄️不可变微分片，Metadata Log） |
| **分片策略**          | **HBase/Bigtable**：Range（自动分裂）<br>**Cassandra**：Hash（一致性哈希）                                                            | **ClickHouse**：Hash/Range/手动<br>**Apache Doris/StarRocks**：Hash/Range<br>**Snowflake/BigQuery**：微分片（自动）                                                                                                            |
| **一致性**           | **HBase/Bigtable**：单行原子<br>**Cassandra**：可调一致（ONE/QUORUM/ALL）                                                          | **ClickHouse**：最终一致（副本异步）<br>**Apache Doris/StarRocks**：最终一致<br>**Snowflake/BigQuery**：事务（仓库语义）                                                                                                                    |
| **适用场景**          | ✅ 高速写入 + Row Key 查询<br>✅ 时序/日志/IoT/用户画像<br>✅ 查询模式固定                                                                    | **ClickHouse**：实时分析、日志分析<br>**Apache Doris/StarRocks**：HTAP、实时报表<br>**Snowflake/BigQuery**：离线分析、BI 报表                                                                                                              |
| **不适用场景**         | 🔴 多维度查询<br>🔴 复杂聚合<br>🔴 临时查询                                                                                         | 🔴 高频点查询<br>🔴 频繁更新（不可变存储）<br>🔴 OLTP 事务                                                                                                                                                                    |

#### 可变 vs 不可变存储机制

**不可变存储（❄️）：**
1. **HBase/Bigtable**：不可变 HFile/SSTable
2. **Cassandra**：不可变 SSTable
3. **ClickHouse**：不可变 Part
4. **BigQuery**：不可变列式文件
5. **Snowflake**：不可变微分片

**可变存储（🔄）：**
1. **Apache Doris/StarRocks**：可变（Delete Bitmap 机制）

**关键洞察：**

**为什么 LSM-Tree 都是不可变的？**
- LSM-Tree 的核心思想就是：写入不可变文件 + 后台 Compaction 合并
- 更新操作通过追加新版本实现，不是原地修改
- 优势：写入快（顺序写）、无碎片、易于备份

**为什么 Apache Doris/StarRocks 是可变的？**
- 为了支持 HTAP 场景的高效 UPDATE/DELETE
- 使用 Delete Bitmap 标记删除，支持原地更新
- 这是对传统 OLAP 不可变存储的改进
- 权衡：更新快，但 Compaction 开销更大

---

#### 为什么列族宽表和列式数据库基本不支持传统二级索引？

| 数据库类型       | 代表         | 二级索引支持         | 替代机制        |
| :---------- | :--------- | :------------- | :---------- |
| **列族宽表**    | Cassandra  | ⚠️ 有，但谨慎（场景受限） | Row Key 设计  |
| **列族宽表**    | HBase      | 🚫 原生无通用二级索引   | Phoenix（外部） |
| **列族宽表**    | Bigtable   | 🚫 无内建二级索引     | Row Key 设计  |
| **OLAP 列式** | ClickHouse | ⚠️ 稀疏索引/跳数索引   | 分区裁剪、排序键    |
| **OLAP 列式** | Snowflake  | 🚫 无传统二级索引     | 微分片统计裁剪     |
| **OLAP 列式** | BigQuery   | 🚫 无传统二级索引     | 分区/聚簇裁剪     |
| **OLAP/时序** | Druid      | ✅ 倒排/位图索引      | 非传统索引       |

#### 查询性能实测对比（1000 万行数据）

| 查询类型                   | HBase/Bigtable | Cassandra | ClickHouse | MySQL（有索引）    | 性能差距              |
| ---------------------- | -------------- | --------- | ---------- | ------------- | ----------------- |
| **点查询（主键）**            | 1-5ms          | 1-5ms     | 10-50ms    | 1-2ms         | 列族宽表最快            |
| **前缀查询（Row Key 前缀）**   | 10-50ms        | 🔴 不支持    | 20-100ms   | 50-200ms      | HBase/Bigtable 最快 |
| **范围查询（主键 1000 行）**    | 50-100ms       | 50-100ms  | 50-100ms   | 100-200ms     | 相当                |
| **非主键过滤（age=25）**      | 40-50秒         | 40-50秒    | 1-3秒       | 0.5秒          | 列式快 20-50 倍       |
| **聚合查询（AVG/GROUP BY）** | 4-6分钟          | 0.5-2秒    | 30秒        | 列式快 100-600 倍 |                   |
| **多列聚合**               | 5-10分钟         | 1-3秒      | 1-2分钟      | 列式快 100-300 倍 |                   |

**关键结论：**
- ✅ 列族宽表：主键查询极快，其他查询极慢
- ✅ 列式存储：聚合查询极快，点查询较慢
- ✅ 关系型数据库：有索引时综合性能最均衡

#### 实际查询场景对比

**场景 1：统计平均年龄（SELECT AVG(age) FROM users）**

| 数据库        | 磁盘读取               | 执行时间   | 原因                    |
| ---------- | ------------------ | ------ | --------------------- |
| HBase      | ~50GB（所有列）         | 5-10分钟 | 🔴 读取所有列族，无法跳过不需要的列   |
| ClickHouse | ~40MB（只有 age 列）    | 0.5-2秒 | ✅ 只读 age.bin，SIMD 向量化 |
| MySQL      | ~200MB（age 列 + 索引） | 30秒    | ⚠️ 需要扫描索引或全表          |

**场景 2：过滤查询（SELECT name, city WHERE age > 30）**

| 数据库        | 磁盘读取         | 网络传输         | 执行时间  | 原因             |
| ---------- | ------------ | ------------ | ----- | -------------- |
| HBase      | ~50GB（所有列）   | ~15GB（30%数据） | 3-5分钟 | 🔴 全表扫描，读取所有列  |
| ClickHouse | ~120MB（3列）   | ~40MB（结果）    | 1-3秒  | ✅ 只读 3 列，向量化过滤 |
| MySQL      | ~200MB（索引扫描） | ~50MB（结果）    | 0.5秒  | ✅ age 列有索引     |

**场景 3：点查询（SELECT * WHERE id = 'user_5000000'）**

| 数据库 | 执行时间 | 原因 |
|--------|---------|------|
| HBase | 2ms | ✅ 二分查找 + Bloom Filter + Cache |
| ClickHouse | 10-50ms | ⚠️ 稀疏索引 + 需读取多个列文件 |
| MySQL | 1ms | ✅ B+Tree 索引精确定位 |

**场景 4：前缀查询（SELECT * WHERE row_key LIKE 'address_hash#%'）**

| 数据库            | 执行时间     | 原因                              |
| -------------- | -------- | ------------------------------- |
| HBase/Bigtable | 10-50ms  | ✅ 原生前缀扫描，Range 分片优化             |
| Cassandra      | 🔴 不支持   | 🔴 Hash 分片，必须提供完整 Partition Key |
| ClickHouse     | 20-100ms | ✅ 排序键前缀优化（ORDER BY 第一列）         |
| MySQL          | 50-200ms | ⚠️ 索引前缀扫描（取决于选择性）               |

### 列族宽表不支持二级索引的原因

**核心原因：架构设计的根本冲突**

1. **数据分布模型冲突**
   ```
   列族宽表：数据按 RowKey 分片
   Shard 1: RowKey [0000-3333]
   Shard 2: RowKey [3334-6666]
   
   二级索引：需要按索引列分片
   → 查询时需要跨 Shard 扫描
   → 破坏分布式架构的局部性
   ```

2. **写入性能优化 vs 查询灵活性**
   ```
   列族宽表设计目标：极致写入性能
   - 纯追加写入（Append-Only）
   - 无需查找现有数据
   - 无需更新索引
   - 吞吐量：百万行/秒
   
   二级索引需求：
   - 每次写入需更新索引
   - 需要读取旧值
   - 可能触发索引重组
   → 写入性能下降 50-80%
   ```

3. **LSM-Tree 架构限制**
   ```
   LSM-Tree 特点：
   - 数据分散在多个 SSTable
   - 需要后台 Compaction 合并
   - 同一行可能有多个版本
   
   索引困境：
   - 索引指向 SSTable + Offset → Compaction 后失效
   - 索引指向 RowKey → 需要二次查询，性能更差
   ```

4. **分布式一致性代价**
   ```
   全局二级索引挑战：
   - 强一致性（2PC）→ 延迟 10 倍，吞吐量下降 100 倍
   - 最终一致性 → 索引滞后，查询结果不准确
   ```

---

### OLAP 列式数据库不支持传统二级索引的原因

**核心原因：列式存储本身就是"索引"**

1. **列式存储的天然优势**
   ```
   传统行式 + 二级索引：
   Table: [Row1, Row2, Row3, ...]
   Index: age → [Row1, Row5, Row9]
   查询 WHERE age=25：先查索引 → 再回表读行
   
   列式存储：
   Column_age: [25, 30, 25, 40, 25, ...]
   查询 WHERE age=25：直接扫描 age 列（无需索引）
   → 列式压缩 + 向量化扫描，速度极快
   ```

2. **批量扫描 vs 点查询**
   ```
   OLAP 查询模式：
   SELECT AVG(age) FROM users WHERE city='Beijing'
   → 扫描 city 列 + age 列（只读 2 列，不读其他列）
   → 列式存储天然高效，不需要索引
   
   OLTP 查询模式：
   SELECT * FROM users WHERE email='user@example.com'
   → 点查询，需要索引快速定位
   ```

3. **替代机制更高效**
   ```
   ClickHouse：
   - 稀疏索引（每 8192 行一个标记）
   - 分区裁剪（按日期分区，跳过无关分区）
   - 排序键（ORDER BY 决定物理排序）
   
   BigQuery/Snowflake：
   - 微分片（自动分片，每个分片有统计信息）
   - 统计裁剪（Min/Max/Count，跳过无关分片）
   - 聚簇键（相关数据物理聚集）
   
   Druid：
   - 倒排索引（维度列）
   - 位图索引（低基数列）
   - 时间分段（按时间自动分区）
   ```

4. **写入性能考虑**
   ```
   OLAP 写入模式：批量导入
   - 一次写入百万行
   - 维护传统索引成本极高
   - 不可变数据（Immutable），无需更新索引
   
   替代方案：
   - 写入时排序（按 ORDER BY 排序）
   - 写入时分区（按时间/地区分区）
   - 写入后统计（记录 Min/Max/Count）
   ```

---

#### 关于列族宽表，列式存储的实际解决方案对比

**列族宽表的解决方案：**

| 方案 | 实现方式 | 优点 | 缺点 |
|------|---------|------|------|
| **应用层索引表** | 手动维护多张表 | 灵活可控 | 需要应用层保证一致性 |
| **Phoenix（HBase）** | 全局二级索引 | 透明使用 | 写入性能下降 |
| **Cassandra 本地索引** | 每个节点独立索引 | 写入快 | 查询需扫描所有节点 |
| **外部索引系统** | HBase + Elasticsearch | 功能强大 | 架构复杂，数据同步 |

**OLAP 列式的解决方案：**

| 方案 | 实现方式 | 优点 | 缺点 |
|------|---------|------|------|
| **分区裁剪** | 按时间/地区分区 | 跳过无关数据 | 需要合理设计分区键 |
| **排序键优化** | ORDER BY 设计 | 范围查询快 | 只能一个排序键 |
| **物化视图** | 预聚合结果 | 查询极快 | 存储成本高 |
| **外部索引** | ClickHouse + ES | 全文搜索 | 架构复杂 |

---

#### 选型建议

**选择列族宽表（无二级索引）的场景**：
- ✅ 高速写入 + 简单查询（按 Row Key）
- ✅ 时序数据、日志数据、IoT 数据
- ✅ 查询模式固定且可预测
- 🔴 不适合：多维度查询、临时查询

**选择 OLAP 列式（无传统索引）的场景**：
- ✅ 批量扫描 + 聚合查询
- ✅ 分析报表、BI 看板
- ✅ 数据仓库、数据湖
- 🔴 不适合：高频点查询、实时更新

**需要灵活查询的场景**：
- ✅ MongoDB/PostgreSQL（有二级索引）
- ✅ 混合架构（HBase + Elasticsearch）
- ✅ TiDB（分布式二级索引）

---

#### 列族宽表只有一级索引（RowKey）为什么查询仍然快？

**核心答案：快是有前提的 - 仅限于按 RowKey 查询！**

##### 1. 存储引擎层面的优化

**HBase 存储引擎（基于 LSM-Tree）：**

```
物理存储结构：
Region Server 1:
  Region: [rowkey_0000 ~ rowkey_3333]
    ├─ MemStore（内存）
    │   └─ 最新写入的数据，按 RowKey 排序
    │
    └─ HFile（磁盘，SSTable 格式）
        ├─ HFile_001: [rowkey_0000 ~ rowkey_1000]
        │   ├─ Data Block（4KB-64KB）
        │   │   ├─ rowkey_0001: CF:info → name=Alice, age=25
        │   │   ├─ rowkey_0002: CF:info → name=Bob, age=30
        │   │   └─ ...
        │   │
        │   ├─ Index Block（索引块）
        │   │   ├─ rowkey_0001 → offset: 0
        │   │   ├─ rowkey_0100 → offset: 4096
        │   │   └─ rowkey_0200 → offset: 8192
        │   │
        │   └─ Bloom Filter（布隆过滤器）
        │       └─ 快速判断 RowKey 是否存在
        │
        └─ HFile_002: [rowkey_1001 ~ rowkey_2000]

关键优化：
1. RowKey 字典序排序 → 二分查找 O(log N)
2. Index Block → 快速定位到 Data Block
3. Bloom Filter → 避免无效磁盘读取
4. Block Cache → 热点数据缓存
```

**Cassandra 存储引擎（基于 LSM-Tree）：**

```
物理存储结构：
Node 1:
  Partition: [partition_key_hash_range]
    ├─ MemTable（内存）
    │   └─ 按 Partition Key + Clustering Key 排序
    │
    └─ SSTable（磁盘）
        ├─ SSTable_001
        │   ├─ Data.db（数据文件）
        │   │   ├─ Partition: user_001
        │   │   │   ├─ Clustering Key: 2024-01-01 → login_count=10
        │   │   │   ├─ Clustering Key: 2024-01-02 → login_count=15
        │   │   │   └─ ...
        │   │   └─ Partition: user_002
        │   │
        │   ├─ Index.db（分区索引）
        │   │   ├─ user_001 → offset: 0
        │   │   ├─ user_002 → offset: 2048
        │   │   └─ ...
        │   │
        │   ├─ Summary.db（索引摘要，内存常驻）
        │   │   └─ 每 128 个分区一个采样点
        │   │
        │   └─ Filter.db（布隆过滤器）
        │       └─ 快速判断 Partition Key 是否存在
        │
        └─ SSTable_002

关键优化：
1. Partition Key 哈希分片 → 直接定位节点
2. Clustering Key 排序 → 范围查询快
3. Summary.db 常驻内存 → 减少磁盘 I/O
4. Bloom Filter → 避免读取不存在的分区
```

**Bigtable 存储引擎（基于 SSTable）：**

```
物理存储结构：
Tablet Server 1:
  Tablet: [rowkey_range]
    ├─ MemTable（内存）
    │   └─ 按 RowKey 排序的跳表
    │
    └─ SSTable（GFS/Colossus 存储）
        ├─ SSTable_001
        │   ├─ Data Block（64KB）
        │   │   └─ 按 RowKey 排序的 KV 对
        │   │
        │   ├─ Index Block（多级索引）
        │   │   ├─ Level 1: 每 64KB 一个索引项
        │   │   └─ Level 2: 索引的索引
        │   │
        │   └─ Bloom Filter
        │       └─ 每个 SSTable 一个
        │
        └─ SSTable_002

关键优化：
1. 多级索引 → 大规模数据快速定位
2. Block Cache（内存）→ 热点数据缓存
3. Bloom Filter → 减少无效读取
4. 存储计算分离 → 扩展性好
```

##### 2. 查询引擎层面的优化

**场景 1：点查询（Get）- 极快**

```
查询：get 'users', 'user_12345'

HBase 查询路径：
1. 客户端 → ZooKeeper
   └─ 查询 Meta 表：user_12345 在哪个 Region？
   └─ 返回：Region Server 3, Region [user_10000 ~ user_20000]
   
2. 客户端 → Region Server 3
   └─ 查询 Region 的 MemStore（内存）
      ├─ 找到？→ 直接返回（1-2ms）
      └─ 未找到？→ 继续查询 HFile
   
3. 查询 HFile（磁盘）
   └─ 步骤 3.1：Bloom Filter 检查
      ├─ 不存在？→ 跳过该 HFile
      └─ 可能存在？→ 继续
   
   └─ 步骤 3.2：Index Block 二分查找
      └─ user_12345 在 Data Block 5（offset: 20480）
   
   └─ 步骤 3.3：读取 Data Block 5
      ├─ 检查 Block Cache（内存）
      │   ├─ 命中？→ 直接返回（2-5ms）
      │   └─ 未命中？→ 从磁盘读取（5-20ms）
      └─ 在 Block 内二分查找 user_12345

总耗时：
- 内存命中：1-5ms
- 磁盘读取：5-20ms
- 跨网络：+1-5ms

为什么快？
✅ RowKey 有序 → 二分查找 O(log N)
✅ Bloom Filter → 避免 90% 无效读取
✅ Block Cache → 热点数据内存命中率 >80%
✅ 直接定位 Region → 无需扫描其他节点
```

**场景 2：范围查询（Scan）- 快**

```
查询：scan 'users', {STARTROW => 'user_10000', STOPROW => 'user_10100'}

HBase 查询路径：
1. 定位起始 Region
   └─ user_10000 在 Region Server 3
   
2. 顺序扫描（Sequential Scan）
   └─ MemStore: user_10000 ~ user_10050（内存）
   └─ HFile_001: user_10051 ~ user_10080（磁盘顺序读）
   └─ HFile_002: user_10081 ~ user_10100（磁盘顺序读）
   
3. 跨 Region 扫描（如果需要）
   └─ Region Server 3: user_10000 ~ user_19999
   └─ Region Server 4: user_20000 ~ user_10100（跨节点）

总耗时：
- 100 行：10-50ms
- 1000 行：50-200ms
- 10000 行：200-1000ms

为什么快？
✅ RowKey 有序 → 顺序读取，无需随机 I/O
✅ 预读（Prefetch）→ 提前加载下一个 Block
✅ 批量返回 → 减少网络往返
✅ 并行扫描 → 多个 Region 并行读取
```

**场景 3：非 RowKey 查询（Filter）- 慢！**

```
查询：scan 'users', {FILTER => "SingleColumnValueFilter('info', 'age', =, 'binary:25')"}

HBase 查询路径：
1. 全表扫描（Full Table Scan）
   └─ Region Server 1: 扫描所有行
   └─ Region Server 2: 扫描所有行
   └─ Region Server 3: 扫描所有行
   └─ ...
   
2. 每一行都需要：
   └─ 读取 RowKey
   └─ 读取 CF:info 列族
   └─ 检查 age 列是否等于 25
   └─ 符合条件？→ 返回
   
3. 无法利用任何索引
   └─ 无法跳过不符合条件的行
   └─ 无法跳过不需要的列族

总耗时：
- 1 万行：1-5 秒
- 10 万行：10-50 秒
- 100 万行：1-5 分钟
- 1000 万行：10-50 分钟

为什么慢？
❌ 全表扫描 → O(N) 复杂度
❌ 读取所有列族 → 即使只需要 age 列
❌ 无法并行优化 → 受限于磁盘 I/O
❌ 网络传输大 → 所有数据都要传输到客户端过滤
```

##### 3. 三种列族宽表的查询性能对比

| 查询类型                 | HBase    | Cassandra | Bigtable | 为什么快/慢                        |
| -------------------- | -------- | --------- | -------- | ----------------------------- |
| **点查询（RowKey）**      | 1-5ms    | 1-5ms     | 1-5ms    | ✅ 二分查找 + Bloom Filter + Cache |
| **范围查询（RowKey）**     | 10-100ms | 10-100ms  | 10-100ms | ✅ 顺序扫描 + 预读                   |
| **分区内范围（Cassandra）** | N/A      | 5-50ms    | N/A      | ✅ Clustering Key 有序           |
| **非 RowKey 过滤**      | 秒-分钟级    | 秒-分钟级     | 秒-分钟级    | 🔴 全表扫描，无索引                   |
| **聚合查询**             | 分钟级      | 分钟级       | 分钟级      | 🔴 非列式存储，无向量化                 |
| **跨分区查询（Cassandra）** | N/A      | 很慢        | N/A      | 🔴 需要查询所有节点                   |

##### 4. 为什么列族存储是"行式存储的变种"而不是真正的"列式存储"？

**核心区别：数据的物理存储单位**

**重要说明：理解 HBase 的层级结构**

```
HBase 完整层级：
Namespace: default
  └─ Table: users  ← 表（逻辑概念）
      ├─ Column Family: info  ← 列族（逻辑分组）
      │   ├─ Column: name, age, city
      │   └─ 物理存储: HFile_info（物理文件）
      │
      └─ Column Family: stats  ← 列族（逻辑分组）
          ├─ Column: login_count, last_login
          └─ 物理存储: HFile_stats（物理文件）

关键理解：
- Table（表）= 逻辑概念，包含所有列族
- Column Family（列族）= 逻辑分组，按列族分文件存储
- HFile = 物理文件，每个列族对应一个或多个 HFile
- 同一行的数据通过 RowKey 关联（分散在不同 HFile 中）

下面的图展示的是 users 表的物理存储结构：
```

---

**4.1 三种存储模型的本质差异**

**传统行式存储（MySQL InnoDB）：**
```
Table: users（一个表，一个文件）

物理存储（磁盘上的实际布局）：
users.ibd:
  Page 1 (16KB):
    ┌─────────────────────────────────────────────────────────┐
    │ Row 1: [id=1][name=Alice][age=25][city=Beijing]         │
    │ Row 2: [id=2][name=Bob][age=30][city=Shanghai]          │
    │ Row 3: [id=3][name=Carol][age=28][city=Guangzhou]       │
    │ ...                                                      │
    └─────────────────────────────────────────────────────────┘

存储单位：整行
读取粒度：整行（即使只需要 age 列）
优势：点查询快（一次 I/O 读取完整行）
劣势：列查询慢（需要读取所有列）
```

**列族存储（HBase/Cassandra）：**
```
Table: users（一个表，多个文件，按列族分开）

物理存储（磁盘上的实际布局）：

关键：按列族分文件，但每个文件内按行组织

HFile_info（列族 info 的物理文件）：
  ┌─────────────────────────────────────────────────────────┐
  │ RowKey: user_001 ← 同一行的所有 info 列在一起            │
  │   ├─ info:name → Alice                                   │
  │   ├─ info:age → 25                                       │
  │   └─ info:city → Beijing                                 │
  ├─────────────────────────────────────────────────────────┤
  │ RowKey: user_002 ← 同一行的所有 info 列在一起            │
  │   ├─ info:name → Bob                                     │
  │   ├─ info:age → 30                                       │
  │   └─ info:city → Shanghai                                │
  ├─────────────────────────────────────────────────────────┤
  │ RowKey: user_003 ← 同一行的所有 info 列在一起            │
  │   ├─ info:name → Carol                                   │
  │   ├─ info:age → 28                                       │
  │   └─ info:city → Guangzhou                               │
  └─────────────────────────────────────────────────────────┘
  ↑
  注意：name、age、city 按行聚集，不是按列聚集！

HFile_stats（列族 stats 的物理文件）：
  ┌─────────────────────────────────────────────────────────┐
  │ RowKey: user_001 ← 同一行的所有 stats 列在一起           │
  │   ├─ stats:login_count → 100                             │
  │   └─ stats:last_login → 2024-01-01                       │
  ├─────────────────────────────────────────────────────────┤
  │ RowKey: user_002 ← 同一行的所有 stats 列在一起           │
  │   ├─ stats:login_count → 50                              │
  │   └─ stats:last_login → 2024-01-02                       │
  ├─────────────────────────────────────────────────────────┤
  │ RowKey: user_003 ← 同一行的所有 stats 列在一起           │
  │   ├─ stats:login_count → 80                              │
  │   └─ stats:last_login → 2024-01-03                       │
  └─────────────────────────────────────────────────────────┘
  ↑
  注意：login_count、last_login 按行聚集，不是按列聚集！

完整的一行数据（通过 RowKey 关联）：
RowKey: user_001
  ├─ 在 HFile_info 中: name=Alice, age=25, city=Beijing
  └─ 在 HFile_stats 中: login_count=100, last_login=2024-01-01

存储单位：列族（按列族分文件，但文件内按行组织）
读取粒度：列族级别（读取某列时，必须读取该行的整个列族）

读取示例：
查询 user_001 的 age：
  1. 打开 HFile_info（跳过 HFile_stats）
  2. 定位到 user_001
  3. 读取该行的所有 info 列：name、age、city（一起读取！）
  4. 提取 age = 25
  ❌ 无法只读取 age，必须读取整个列族

优势：可以只读取需要的列族（跳过 stats），跳过其他列族
劣势：无法只读取单个列（age），仍需读取整个列族的所有列（name、age、city）
```

**真正的列式存储（ClickHouse/Parquet）：**
```
Table: users（一个表，每列一个文件）

物理存储（磁盘上的实际布局）：

关键：每一列独立存储，所有行的同一列在一起

name.bin（所有行的 name 列）：
  ┌─────────────────────────────────────────────────────────┐
  │ [Alice][Bob][Carol][David][Eve][Frank]...               │
  │   ↑     ↑     ↑      ↑      ↑     ↑                     │
  │   行1   行2   行3    行4    行5   行6                     │
  └─────────────────────────────────────────────────────────┘
  ↑
  注意：所有行的 name 连续存储！

age.bin（所有行的 age 列）：
  ┌─────────────────────────────────────────────────────────┐
  │ [25][30][28][35][22][40]...                              │
  │  ↑   ↑   ↑   ↑   ↑   ↑                                  │
  │  行1 行2 行3 行4 行5 行6                                  │
  └─────────────────────────────────────────────────────────┘
  ↑
  注意：所有行的 age 连续存储！

city.bin（所有行的 city 列）：
  ┌─────────────────────────────────────────────────────────┐
  │ [Beijing][Shanghai][Guangzhou][Shenzhen]...             │
  │    ↑        ↑          ↑           ↑                    │
  │   行1      行2        行3         行4                     │
  └─────────────────────────────────────────────────────────┘

login_count.bin（所有行的 login_count 列）：
  ┌─────────────────────────────────────────────────────────┐
  │ [100][50][80][120][30][90]...                            │
  │   ↑   ↑   ↑   ↑    ↑   ↑                                │
  │  行1 行2 行3 行4  行5 行6                                 │
  └─────────────────────────────────────────────────────────┘

完整的一行数据（通过行号关联）：
行1: name.bin[0]=Alice, age.bin[0]=25, city.bin[0]=Beijing, login_count.bin[0]=100

存储单位：单列（每一列独立存储）
读取粒度：单列（只读取需要的列）

读取示例：
查询所有用户的 age：
  1. 只打开 age.bin 文件
  2. 顺序读取所有 age 值：[25, 30, 28, 35, ...]
  3. 完全不需要读取 name.bin、city.bin 等
  ✅ 只读取需要的列，跳过所有其他列

优势：列查询极快，压缩率高（同类型数据），聚合查询快（向量化）
劣势：点查询需要读取多个文件（如查询某一行的所有列）
```

**ClickHouse .bin 文件内部结构（底层物理布局）：**

**首先，理解一个完整的表结构：**

```sql
-- 创建表
CREATE TABLE users (
    id UInt64,
    name String,
    age UInt8,
    city String,
    login_count UInt32
) ENGINE = MergeTree()
ORDER BY id;

-- 插入数据（假设插入了 100 万行）
INSERT INTO users VALUES
    (1, 'Alice', 25, 'Beijing', 100),
    (2, 'Bob', 30, 'Shanghai', 50),
    (3, 'Carol', 28, 'Guangzhou', 80),
    ...
    (1000000, 'Zoe', 35, 'Shenzhen', 120);
```

**这个表在磁盘上的物理存储结构：**

```
/var/lib/clickhouse/data/default/users/20241106_1_1_0/  ← Part 目录
├── checksums.txt           # 校验和
├── columns.txt             # 列信息
├── count.txt               # 行数：1000000
│
├── primary.idx             # 稀疏索引（每 8192 行一个索引项）
│   ├─ Entry 0: id=1      → Granule 0 (行 0-8191)
│   ├─ Entry 1: id=8193   → Granule 1 (行 8192-16383)
│   ├─ Entry 2: id=16385  → Granule 2 (行 16384-24575)
│   └─ ... (共 122 个索引项，1000000 / 8192 ≈ 122)
│
├── id.bin                  # id 列的数据文件（压缩存储）
├── id.mrk2                 # id 列的标记文件
│
├── name.bin                # name 列的数据文件（压缩存储）
├── name.mrk2               # name 列的标记文件
│
├── age.bin                 # age 列的数据文件（压缩存储）
├── age.mrk2                # age 列的标记文件
│
├── city.bin                # city 列的数据文件（压缩存储）
├── city.mrk2               # city 列的标记文件
│
├── login_count.bin         # login_count 列的数据文件（压缩存储）
└── login_count.mrk2        # login_count 列的标记文件
```

**关键理解：一行数据是如何存储的？**

```
逻辑上的一行数据：
Row 10000: id=10000, name='Alice', age=25, city='Beijing', login_count=100

物理上分散存储在 5 个文件中：
├─ id.bin        → Granule 1 的第 1808 个位置 → 值: 10000
├─ name.bin      → Granule 1 的第 1808 个位置 → 值: 'Alice'
├─ age.bin       → Granule 1 的第 1808 个位置 → 值: 25
├─ city.bin      → Granule 1 的第 1808 个位置 → 值: 'Beijing'
└─ login_count.bin → Granule 1 的第 1808 个位置 → 值: 100

通过"行号"关联：所有列的第 10000 行（在 Granule 1 的第 1808 个位置）组成完整的一行
```

**现在深入理解单个 .bin 文件的内部结构：**

```
age.bin 文件不是简单的连续存储，而是分层组织：

age.bin（物理文件，大小约 1MB，压缩后）：
  ├─ Compressed Block 0（压缩块，包含多个 Granule）
  │  ├─ Granule 0: [25, 30, 28, 22, 35, ...] (行 0-8191，共 8192 个 age 值)
  │  │   ↑   ↑   ↑   ↑   ↑
  │  │   行0 行1 行2 行3 行4 的 age 值
  │  │
  │  ├─ Granule 1: [25, 40, 33, ...] (行 8192-16383，共 8192 个 age 值)
  │  │   ↑
  │  │   行 8192 的 age 值（对应 id=8193 的那一行）
  │  │
  │  └─ Granule 2: [45, 50, 33, ...] (行 16384-24575)
  │
  ├─ Compressed Block 1（压缩块）
  │  ├─ Granule 3: [...] (行 24576-32767)
  │  └─ Granule 4: [...] (行 32768-40959)
  └─ ...

age.mrk2（标记文件，记录每个 Granule 在 age.bin 中的位置）：
  Mark 0: offset_in_compressed_file=0,    offset_in_decompressed_block=0
          ↑ 指向 Compressed Block 0 的起始位置
  Mark 1: offset_in_compressed_file=0,    offset_in_decompressed_block=32768
          ↑ 仍在 Compressed Block 0 中，但偏移量是 32768 字节
  Mark 2: offset_in_compressed_file=0,    offset_in_decompressed_block=65536
  Mark 3: offset_in_compressed_file=4096, offset_in_decompressed_block=0
          ↑ 指向 Compressed Block 1 的起始位置（文件偏移 4096 字节）
  ...

primary.idx（稀疏索引，每 8192 行一个索引项）：
  Index Entry 0: id=1      → Granule 0 (行 0-8191)
  Index Entry 1: id=8193   → Granule 1 (行 8192-16383)
  Index Entry 2: id=16385  → Granule 2 (行 16384-24575)
  ...
```

**完整查询示例：SELECT * FROM users WHERE id = 10000**

```
步骤 1: 在 primary.idx 中二分查找
  - 10000 > 8193  → 在 Granule 1 或之后
  - 10000 < 16385 → 在 Granule 1 中
  - 结论：id=10000 在 Granule 1（行 8192-16383）

步骤 2: 读取 id 列的 Granule 1
  - 查找 id.mrk2 的 Mark 1 → offset_in_compressed_file=0, offset_in_decompressed_block=32768
  - 读取 id.bin 的 Compressed Block 0 → 解压
  - 提取 Granule 1 的数据（8192 个 id 值）：[8193, 8194, 8195, ..., 10000, ..., 16384]
  - 在 Granule 1 中线性扫描找到 id=10000
  - 找到！位置是 Granule 1 的第 1808 个元素（相对行号 = 8192 + 1808 = 10000）

步骤 3: 读取其他列的同一行（第 10000 行）
  - 查找 name.mrk2 的 Mark 1 → 读取 name.bin 的 Granule 1 → 提取第 1808 个元素 → 'Alice'
  - 查找 age.mrk2 的 Mark 1 → 读取 age.bin 的 Granule 1 → 提取第 1808 个元素 → 25
  - 查找 city.mrk2 的 Mark 1 → 读取 city.bin 的 Granule 1 → 提取第 1808 个元素 → 'Beijing'
  - 查找 login_count.mrk2 的 Mark 1 → 读取 login_count.bin 的 Granule 1 → 提取第 1808 个元素 → 100

步骤 4: 组装完整的一行数据
  Row 10000: id=10000, name='Alice', age=25, city='Beijing', login_count=100

性能分析：
  - 索引查找：O(log N)，N = 索引项数量（1000000 / 8192 ≈ 122 项）
  - Granule 扫描：O(8192)，最多扫描 8192 行
  - 读取 5 个列文件：5 × (解压 + 提取) ≈ 10-50ms
  - 总复杂度：O(log N + 8192)，远小于全表扫描 O(1000000)
  - 但比 MySQL B+Tree 的 O(log N) 慢，因为需要扫描整个 Granule
```

**关键概念总结：**
- **Granule**：最小读取单位（默认 8192 行），不可再分
- **Compressed Block**：压缩单位（包含多个 Granule）
- **.mrk2 文件**：记录每个 Granule 在 .bin 文件中的位置
- **稀疏索引**：每 8192 行一个索引项（不是每行都有索引）
- **行号关联**：所有列的第 N 行通过"行号"关联，组成完整的一行

**这就是为什么 ClickHouse 点查询慢的原因：**
- 即使只查 1 行，也要读取整个 Granule（8192 行）
- 而 MySQL B+Tree 可以精确定位到单行（密集索引）
- 但 ClickHouse 的优势在于聚合查询：只读需要的列，SIMD 向量化处理

---

**三种存储模型对比总结：**

| 特征 | 行式存储（MySQL） | 列族存储（HBase） | 列式存储（ClickHouse） |
|------|-----------------|-----------------|---------------------|
| **表结构** | 1个表 = 1个文件 | 1个表 = 多个文件（按列族） | 1个表 = 多个文件（按列） |
| **文件组织** | 按行 | 按列族分文件，文件内按行 | 按列分文件 |
| **数据关联** | 行内所有列在一起 | 通过 RowKey 关联不同列族 | 通过行号关联不同列 |
| **存储单位** | 整行 | 列族 | 单列 |
| **读取粒度** | 整行 | 整个列族 | 单列 |

---

**4.2 为什么说列族存储是"行式的变种"？**

**关键证据 1：存储组织方式**

```
HBase 的 KeyValue 结构（实际存储格式）：

单个 KeyValue 对象：
┌──────────────────────────────────────────────────────────────┐
│ Key Length (4 bytes)                                         │
│ Value Length (4 bytes)                                       │
│ ┌────────────────────────────────────────────────────────┐   │
│ │ Key:                                                    │   │
│ │   ├─ Row Length (2 bytes)                               │   │
│ │   ├─ Row Key (variable): "user_001"                     │   │
│ │   ├─ Column Family Length (1 byte)                      │   │
│ │   ├─ Column Family: "info"                              │   │
│ │   ├─ Column Qualifier: "age"                            │   │
│ │   ├─ Timestamp (8 bytes): 1704067200000                 │   │
│ │   └─ Key Type (1 byte): Put                             │   │
│ └────────────────────────────────────────────────────────┘   │
│ ┌────────────────────────────────────────────────────────┐   │
│ │ Value: "25"                                             │   │
│ └────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘

HFile 中的存储顺序（按 RowKey 排序）：
[user_001, info:age, ts1] → 25
[user_001, info:city, ts1] → Beijing
[user_001, info:name, ts1] → Alice
[user_001, stats:login_count, ts1] → 100
[user_002, info:age, ts1] → 30
[user_002, info:city, ts1] → Shanghai
[user_002, info:name, ts1] → Bob
...

关键点：
❌ 同一行的所有列（同一列族）物理上相邻存储
❌ 按 RowKey 组织，不是按列组织
✅ 这就是"行式存储"的特征！
```

**关键证据 2：读取行为**

```
场景：查询 user_001 的 age

HBase 读取过程：
1. 定位到 RowKey = user_001 的起始位置
2. 顺序读取该行的所有 KeyValue：
   ├─ [user_001, info:age, ts1] → 25      ← 目标数据
   ├─ [user_001, info:city, ts1] → Beijing ← 也被读取了！
   ├─ [user_001, info:name, ts1] → Alice   ← 也被读取了！
   └─ [user_001, stats:login_count, ts1] → 100 ← 如果跨列族，也可能被读取

为什么会读取额外数据？
因为它们在磁盘上是连续存储的（同一个 Block）！

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

ClickHouse 读取过程：
1. 只打开 age.bin 文件
2. 读取 age 列的所有值：[25, 30, 28, ...]
3. 过滤出 age=25 的行号：[0, 5, 12, ...]
4. 如果需要其他列，再打开对应文件

完全不同的读取模式！
```

**关键证据 3：Block 内部结构**

```
HBase Data Block（64KB）内部布局：

Block Header
├─ Magic Number
├─ Compression Codec
└─ ...

Block Data（按 RowKey 顺序）:
┌─────────────────────────────────────────────────────────────┐
│ [user_001, info:age] → 25                                   │
│ [user_001, info:city] → Beijing                             │
│ [user_001, info:name] → Alice                               │
│ [user_001, stats:login_count] → 100                         │
│ ─────────────────────────────────────────────────────────── │
│ [user_002, info:age] → 30                                   │
│ [user_002, info:city] → Shanghai                            │
│ [user_002, info:name] → Bob                                 │
│ [user_002, stats:login_count] → 50                          │
│ ─────────────────────────────────────────────────────────── │
│ [user_003, info:age] → 28                                   │
│ [user_003, info:city] → Guangzhou                           │
│ ...                                                         │
└─────────────────────────────────────────────────────────────┘

Block Index
└─ user_001 → offset 0
└─ user_002 → offset 256
└─ user_003 → offset 512

关键观察：
❌ 同一行的数据聚集在一起
❌ 不同行的同一列（如 age）分散在 Block 各处
✅ 这是典型的"行式存储"特征！

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

ClickHouse Data Block（64KB）内部布局：

age.bin 的一个 Block:
┌─────────────────────────────────────────────────────────────┐
│ [25][30][28][35][22][40][33][27][29][31][26][24]...         │
│  ↑   ↑   ↑   ↑   ↑   ↑   ↑   ↑   ↑   ↑   ↑   ↑             │
│  所有行的 age 值连续存储                                       │
└─────────────────────────────────────────────────────────────┘

name.bin 的一个 Block:
┌─────────────────────────────────────────────────────────────┐
│ [Alice][Bob][Carol][David][Eve][Frank]...                   │
│   ↑     ↑     ↑      ↑      ↑     ↑                         │
│   所有行的 name 值连续存储                                      │
└─────────────────────────────────────────────────────────────┘

完全不同的组织方式！
```

---

**4.3 实际查询对比：为什么列族存储无法享受列式存储的优势**

**查询 1：统计平均年龄**
```sql
SELECT AVG(age) FROM users;
```

**HBase 执行过程：**
```
1. 扫描所有 Region
2. 每个 Region 读取所有 HFile
3. 每个 HFile 的每个 Block：
   ├─ 读取 Block（64KB）
   ├─ 解析所有 KeyValue 对象
   ├─ 过滤出 info:age 列
   ├─ 累加 age 值
   └─ 丢弃其他列（name, city, login_count 等）
   
4. 汇总结果

问题：
❌ 读取了大量不需要的数据（name, city, login_count）
❌ 需要解析每个 KeyValue 对象（CPU 开销大）
❌ 无法利用 SIMD 向量化（数据不连续）
❌ 压缩率低（混合类型数据）

1000 万行数据：
- 磁盘读取：~50GB（包含所有列）
- 执行时间：5-10 分钟
```

**ClickHouse 执行过程：**
```
1. 只打开 age.bin 文件
2. 读取所有 age 值（连续存储）
3. 向量化计算平均值：
   ├─ SIMD 指令批量处理（一次处理 8-16 个值）
   ├─ CPU Cache 友好（数据连续）
   └─ 高压缩率（同类型数据）
   
4. 返回结果

优势：
✅ 只读取需要的列（age）
✅ 数据连续，SIMD 加速
✅ 压缩率高（整数压缩）
✅ 并行处理

1000 万行数据：
- 磁盘读取：~40MB（只有 age 列，压缩后）
- 执行时间：0.5-2 秒

性能差距：150-600 倍！
```

**查询 2：过滤 age > 30 的用户**
```sql
SELECT name, city FROM users WHERE age > 30;
```

**HBase 执行过程：**
```
1. 全表扫描（无法利用索引）
2. 每个 Block：
   ├─ 读取所有数据（包括 age, name, city, login_count 等）
   ├─ 逐行检查 age > 30
   ├─ 符合条件的返回 name 和 city
   └─ 不符合的丢弃
   
问题：
❌ 读取了所有列（即使只需要 age, name, city）
❌ 无法提前过滤（必须读取后才能判断）
❌ 网络传输大（所有数据都要传输）

1000 万行数据（假设 30% 符合条件）：
- 磁盘读取：~50GB
- 网络传输：~15GB
- 执行时间：3-5 分钟
```

**ClickHouse 执行过程：**
```
1. 读取 age.bin，过滤 age > 30
   └─ 得到行号列表：[5, 12, 18, 25, ...]
   
2. 根据行号读取 name.bin 和 city.bin
   └─ 只读取符合条件的行
   
3. 返回结果

优势：
✅ 只读取 3 个列（age, name, city）
✅ 提前过滤（减少后续读取）
✅ 向量化过滤（SIMD 加速）
✅ 压缩率高

1000 万行数据（30% 符合条件）：
- 磁盘读取：~120MB（3 列，压缩后）
- 网络传输：~40MB（只传输结果）
- 执行时间：1-3 秒

性能差距：100-300 倍！
```

---

**4.4 总结：列族存储为什么是"行式的变种"**

| 特征          | 行式存储      | 列族存储        | 列式存储      |
| ----------- | --------- | ----------- | --------- |
| **存储单位**    | 整行        | 列族（同一行的一组列） | 单列        |
| **物理布局**    | 按行组织      | 按行组织（列族分组）  | 按列组织      |
| **读取粒度**    | 整行        | 整个列族        | 单列        |
| **点查询**     | ✅ 快       | ✅ 快         | ⚠️ 需读多个文件 |
| **列查询**     | 🔴 慢（读整行） | 🔴 慢（读整个列族） | ✅ 快（只读该列） |
| **聚合查询**    | 🔴 慢      | 🔴 慢        | ✅ 极快      |
| **压缩率**     | 低         | 中（列族内混合类型）  | 高（同类型数据）  |
| **SIMD 优化** | 🔴 不支持    | 🔴 不支持      | ✅ 支持      |

**核心结论：**
1. ✅ 列族存储按 **RowKey** 组织数据，不是按列组织
2. ✅ 同一行的数据（同一列族）物理上 **相邻存储**
3. ✅ 读取时无法 **跳过不需要的列**（列族内）
4. ✅ 无法利用 **向量化执行**（数据不连续）
5. ✅ 这些都是 **行式存储的特征**！

**列族存储的本质：**
- 它是在行式存储的基础上，增加了"列族"的概念
- 可以跳过不需要的列族，但无法跳过列族内的列
- 相比纯行式有改进，但远不如真正的列式存储
- 因此称为"行式存储的变种"

---

**列族存储 vs 列式存储（详细对比）：**


```
HBase 列族存储（行式的变种）：
Region:
  RowKey: user_001
    └─ CF:info → [name=Alice, age=25, city=Beijing]（一起存储）
    └─ CF:stats → [login_count=100, last_login=2024-01-01]（一起存储）
  
  RowKey: user_002
    └─ CF:info → [name=Bob, age=30, city=Shanghai]
    └─ CF:stats → [login_count=50, last_login=2024-01-02]

查询 age=25：
❌ 需要读取整个 CF:info 列族
❌ 无法只读取 age 列
❌ 无法跳过 name 和 city 列

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

ClickHouse 列式存储（真正的列式）：
Part_001:
  ├─ name.bin:  [Alice, Bob, Carol, ...]  ← 独立存储
  ├─ age.bin:   [25, 30, 28, ...]         ← 独立存储
  ├─ city.bin:  [Beijing, Shanghai, ...]  ← 独立存储
  └─ login_count.bin: [100, 50, 80, ...]  ← 独立存储

查询 age=25：
✅ 只读取 age.bin 文件
✅ 快速过滤出行号 [0, 5, 12, ...]
✅ 再读取其他需要的列
✅ 压缩率高（同类型数据）
```

##### 5. 实际性能测试对比

**测试场景：1000 万行用户数据**

```
表结构：
RowKey: user_id
CF:info → name, age, city, email
CF:stats → login_count, last_login, total_amount

测试 1：点查询
Query: get 'users', 'user_5000000'

HBase:     2ms   ✅
Cassandra: 3ms   ✅
Bigtable:  2ms   ✅
MySQL:     1ms   ✅（有索引）

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

测试 2：范围查询（1000 行）
Query: scan 'users', {STARTROW => 'user_5000000', STOPROW => 'user_5001000'}

HBase:     50ms   ✅
Cassandra: 60ms   ✅
Bigtable:  45ms   ✅
MySQL:     100ms  ⚠️（索引范围扫描）

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

测试 3：非 RowKey 过滤
Query: SELECT * WHERE age = 25

HBase:     45秒   ❌（全表扫描）
Cassandra: 50秒   ❌（全表扫描）
Bigtable:  40秒   ❌（全表扫描）
MySQL:     0.5秒  ✅（age 列有索引）
ClickHouse: 2秒   ✅（列式存储 + 向量化）

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

测试 4：聚合查询
Query: SELECT age, COUNT(*) GROUP BY age

HBase:     5分钟   ❌（全表扫描 + 应用层聚合）
Cassandra: 6分钟   ❌（全表扫描 + 应用层聚合）
Bigtable:  4分钟   ❌（全表扫描 + 应用层聚合）
MySQL:     30秒    ⚠️（索引扫描 + 聚合）
ClickHouse: 0.5秒  ✅（列式 + 向量化 + 并行）
```

##### 6. 总结

**列族宽表查询快的真相：**

| 场景               | 是否快           | 原因                          |
| ---------------- | ------------- | --------------------------- |
| **按 RowKey 点查**  | ✅ 极快（1-5ms）   | 二分查找 + Bloom Filter + Cache |
| **按 RowKey 范围查** | ✅ 快（10-100ms） | 顺序扫描 + 预读 + 并行              |
| **按非 RowKey 查询** | 🔴 慢（秒-分钟）    | 全表扫描，无索引                    |
| **聚合查询**         | 🔴 很慢（分钟级）    | 非列式存储，无向量化                  |

**关键认知：**
1. ✅ 列族宽表的"快"是有前提的 - **仅限于 RowKey 查询**
2. 🔴 它们**不是真正的列式存储** - 无法享受列式存储的优势
3. ✅ 底层优化（LSM-Tree + Bloom Filter + Cache）让 RowKey 查询极快
4. 🔴 缺少二级索引导致非 RowKey 查询性能差
5. ✅ 适合场景：**已知主键的高速读写**（时序数据、日志、用户画像）
6. 🔴 不适合场景：**多维度查询、复杂聚合分析**


### NoSQL 数据库分类

```
NoSQL 数据库
├─ 键值数据库（Key-Value）
│  └─ Redis, DynamoDB, Memcached
│
├─ 文档数据库（Document）
│  └─ MongoDB, Firestore Native, DocumentDB
│
├─ 宽列存储 / 列族型（Wide-Column）← ✅ Bigtable/HBase/Cassandra 属于这里
│  └─ Bigtable, Cassandra, HBase
│
└─ 图数据库（Graph）
   └─ Neo4j, Neptune
```

**重要区分：宽列存储（列族型）vs 列式数据仓库**

**一句话本质区别：**
- **宽列存储（Bigtable/HBase/Cassandra）= 行式存储 + 动态列，按行读写，NoSQL 数据库**
- **列式数据仓库（BigQuery/Redshift）= 列式存储 + 固定列，按列读写，SQL 数据仓库**

**核心概念：什么是"动态宽列"？**

动态宽列 = 动态（随时添加列）+ 宽（可以有数千列）+ 列（独立键值对）

```
传统 SQL（固定列）：
CREATE TABLE users (id INT, name VARCHAR, age INT);
- 必须预定义列
- 所有行有相同的列
- 通常几十列

Bigtable（动态宽列）：
Row Key: user:001
  profile:name = "Alice"
  profile:age = 30
  tag:golang = true      ← 动态添加
  tag:kubernetes = true  ← 可以有数千个 tag 列（受行大小限制）
  
Row Key: user:002
  profile:name = "Bob"
  extra:hobby = "reading"  ← 每行可以有不同的列

特点：
✅ 无需预定义 Schema（列限定符动态）
✅ 每行可以有不同的列
✅ 列族需预定义且建议很少（Bigtable/HBase: <10）
✅ 列限定符可以很多（受行大小和分区大小限制）
✅ 稀疏存储（只存储有值的列）
```

**宽列存储 / 列族型数据库（Wide-Column Family Store）：**

- 代表：Bigtable, Cassandra, HBase
- 本质：**列族聚簇存储（LSM-Tree）+ 行键有序 + 动态列 + 稀疏表**
- 数据模型：
  - **Bigtable/HBase**：Row Key（有序）+ Column Family + Column Qualifier + Timestamp
  - **Cassandra**：Partition Key（散列）+ Clustering Columns（分区内有序）+ Columns
- 物理存储：
  - **Bigtable/HBase**：按列族聚簇存储在 SSTable，不同列族在不同文件
  - **Cassandra**：按分区键散列分布，分区内按聚簇键有序，无物理列族概念
- 查询方式：通过 Row Key/Partition Key 做点查与范围扫描（强键值特性）
- 🔴 不支持：SQL（原生）、JOIN、复杂聚合、向量化扫描
- 适合：时序数据、IoT、用户标签、日志存储、低延迟单行/范围读写、多版本数据
- 性能：极高吞吐（100万+ TPS），低延迟（5-10ms）
- 列数量限制：
  - **Bigtable**：列族建议个位数（上限约100），列限定符动态（受行大小约100MB限制）
  - **HBase**：列族强烈建议<10，列限定符动态（受行大小和版本限制）
  - **Cassandra**：每行理论上限约20亿列，但强烈建议每分区<100k列、<100MB

**列式数据仓库（Columnar Database for OLAP）：**

- 代表：Redshift, BigQuery, ClickHouse, Parquet
- 本质：**按列物理存放，面向分析型批量扫描**
- 数据模型：关系型表（固定列）
- 物理存储：**按列连续存储**，同一列的数据物理相邻，压缩向量化执行
- 查询方式：SQL 查询，支持任意列查询、聚合、JOIN
- 支持：GROUP BY、聚合函数、向量化扫描、列式压缩
- 适合：OLAP 分析、BI 报表、数据仓库、批量扫描
- 性能：列式扫描快，聚合查询优化，高压缩比

**核心区别对比表：**

| 维度         | 宽列存储（Bigtable/Cassandra/HBase）                                                                                                    | 列式数据仓库（BigQuery/Redshift/ClickHouse） |
| ---------- | --------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------ |
| **本质**     | 列族聚簇存储 + 行键有序 + 动态列                                                                                                                       | 按列物理存放（向量化扫描）                      |
| **物理存储**   | 按列族聚簇（Column Family内数据连续）<br>行键有序（RowKey字典序）                                                                                                                     | 按列连续（一列数据连续）                         |
| **键值特性**   | ✅ 强键值特性（Row Key点查/范围扫描）                                                                                                           | 🔴 无键值特性（SQL查询）                      |
| **查询引擎**   | 原生API（无SQL引擎）<br>**HBase**: Phoenix（SQL→HBase API）<br>**Cassandra**: CQL引擎（类SQL，无JOIN）<br>**Bigtable**: 原生API（可通过Dataflow/Presto） | SQL引擎（标准或方言）                         |
| **客户端支持**  | 多语言支持<br>Bigtable: gRPC（Java/Python/Go等）<br>HBase: Java最成熟，支持多语言<br>Cassandra: 原生协议（多语言）                                          | JDBC/ODBC + 多语言SDK                   |
| **列定义**    | 列族需预定义（建议很少）<br>列限定符动态（Qualifier）<br>Cassandra: 无列族，列动态                                                                                                                        | 固定列（需要 Schema）                       |
| **列数量**    | 列族：Bigtable/HBase建议<10<br>列限定符：数千到数万（受行/分区大小限制）<br>Cassandra: 理论20亿列/行，建议<100k列/分区                                                                                                                        | 通常几十到几百列                             |
| **稀疏性**    | 稀疏（只存储有值的列）                                                                                                                       | Schema固定，支持NULL高效压缩（RLE/字典）                         |
| **存储布局**   | Bigtable/HBase: 按列族聚簇的SSTable（不同列族不同文件）<br>Cassandra: 按分区键散列+聚簇键有序的SSTable                                                                                                                    | 按列分别存储的文件（Parquet/ORC）               |
| **多版本**    | ✅ 原生支持（Timestamp + TTL）                                                    | 🔴 不支持（需应用层处理）                       |
| **查询模式**   | Row Key/Partition Key 点查/范围扫描                                                                                                                   | SQL 任意列查询/聚合                         |
| **读取一行**   | 快（每个列族一次I/O）<br>Cassandra: 取决于SSTable数量和缓存                                                                                                                         | 慢（需要从多个列文件读取）                        |
| **读取一列**   | 慢（需要扫描所有行的该列）<br>可用列过滤器优化                                                                                                                        | 快（一次 I/O 读取整列）                       |
| **聚合查询**   | 慢（需要扫描相关列）<br>列族隔离可减少I/O<br>通常借助外部计算引擎（Spark/Dataflow）                                                                                                                    | 快（只读取需要的列，向量化执行）                     |
| **访问模式**   | 低延迟在线查询（OLTP-like）                                                                                                                     | 批量分析扫描（OLAP）                         |
| **SQL 支持** | 🔴 原生不支持（需Phoenix/Presto）                                                                                                                            | ✅ 完整支持                               |
| **向量化执行**  | 🔴 不支持                                                                                                                            | ✅ 支持                                 |
| **典型场景**   | 时序数据、IoT、用户标签、多版本数据                                                                                                                     | BI 报表、数据分析                           |

**物理存储对比示例：**

```
Bigtable/HBase（列族聚簇存储）：
磁盘文件（按列族分离的SSTable）：

SSTable-profile:
[Row: user:001][profile:name=Alice][profile:age=30]
[Row: user:002][profile:name=Bob][profile:age=25]
[Row: user:003][profile:name=Charlie]

SSTable-tag:
[Row: user:001][tag:golang=true]
[Row: user:003][tag:python=true]

特点：
- 按列族聚簇，不同列族在不同文件
- 同一列族内的列连续存储
- 每行可以有不同的列（稀疏）
- 读取一行需要访问涉及的列族文件（多列族=多次I/O）
- 读取一列需要扫描所有行的该列

Cassandra（分区内聚簇存储）：
磁盘文件（SSTable）：
Partition: user_id=001
  [clustering_key=profile][name=Alice][age=30]
  [clustering_key=tag][golang=true]

Partition: user_id=002
  [clustering_key=profile][name=Bob][age=25]

特点：
- 按分区键散列分布到不同节点
- 分区内按聚簇键有序
- 无物理列族概念
- 读取一行取决于SSTable数量和缓存命中

BigQuery（列式存储）：
磁盘文件：
Column: name
  [Alice][Bob][Charlie]

Column: age
  [30][25][NULL]

Column: tag:golang
  [true][NULL][NULL]

特点：
- 一列的所有值连续存储
- 所有行有相同的列（固定列）
- 读取一列快，读取一行慢
```

**Bigtable 的准确定义：**

- 官方描述："稀疏、分布式、可扩展到数十亿行和数千列的表"
- 分类：**列族型宽表数据库（Wide-Column Family Store）**
- 特性：行式存储 + 动态宽列 + **强键值特性**
- 实现：表是"有序键到多版本列映射"的大键值表

**总结：**
- Bigtable = 行式存储 + 动态宽列，适合在线低延迟场景
- BigQuery = 列式存储 + 固定列，适合离线分析场景


### 数据库概念对应关系

#### 关系型数据库（MySQL/PostgreSQL）

**示例：**
```
MySQL 实例
└─ ecommerce (Database)
   └─ users (Table)
      ├─ Row 1: {id: 1, name: "Alice", age: 25}
      ├─ Row 2: {id: 2, name: "Bob", age: 30}
      └─ Row 3: {id: 3, name: "Charlie", age: 35}
```

**说明与修正：**
- **特点**：严格 Schema、ACID 事务、强一致性；丰富的二级索引与 SQL 能力
- **最佳实践**：主键、二级索引与分区表设计决定性能；控制长事务与锁等待；避免无索引全表扫描
- **备注**：
  - MySQL 的 ACID 取决于存储引擎，InnoDB 为默认且提供完整 ACID
  - PostgreSQL 原生 MVCC、索引类型更丰富（B-Tree、GiST、GIN、BRIN 等）

---

#### 文档型数据库（MongoDB）

**示例：**
```
MongoDB 实例
└─ ecommerce (Database)
   └─ users (Collection)
      ├─ Document 1: {_id: 1, name: "Alice", age: 25}
      ├─ Document 2: {_id: 2, name: "Bob", email: "bob@x.com"}
      └─ Document 3: {_id: 3, name: "Charlie", address: {city: "NYC"}}
```

**对应关系：**
- Collection ≈ Table
- Document ≈ Row
- Field ≈ Column

**说明与修正：**
- **特点**：灵活 Schema、嵌套结构、聚合管道；支持单文档原子操作与多文档事务（现代版本）
- **最佳实践**：为高基数/数组字段建立合适索引；谨慎选择分片键以均衡写入与查询；控制文档体积与字段膨胀
- **重要修正**：现代 MongoDB 采用 WiredTiger 引擎，不支持集合级别切换到 LSM；也不使用 "type=file/type=lsm" 这种对外可选配置

---

#### 键值型数据库（Redis）

**示例：**
```
Redis 实例
└─ Database 0
   ├─ "user:1" → "{name: 'Alice', age: 25}"
   ├─ "user:2" → "{name: 'Bob', age: 30}"
   └─ "session:abc" → "user_id=123"
```

**对应关系：**
- 无表概念，常通过 key 前缀模拟集合（如 user:*）

**说明与修正：**
- **特点**：极低延迟内存结构（字符串、哈希、列表、集合、有序集合等）；脚本原子执行
- **最佳实践**：把 Redis 当缓存/会话/限流/队列用；设计好过期策略与内存淘汰；避免阻塞式大 key/大集合操作
- **备注**：不是 B-Tree；不要当长期持久数据库使用，持久化策略（RDB/AOF）与复制一致性需权衡

---

#### 列族型数据库（Cassandra）

**示例（使用现代术语）：**
```
Cassandra 集群
└─ ecommerce (Keyspace)
   └─ users (Table)
      ├─ Partition Key: (region='us', user_id=1)
      │   └─ Clustering: ts DESC → {name: "Alice", age: 25, email: "alice@x.com"}
      └─ Partition Key: (region='us', user_id=2)
          └─ Clustering: ts DESC → {name: "Bob", age: 30}
```

**对应关系：**
- Keyspace ≈ 命名空间（类似 Database）
- Table = 表（早期术语"Column Family"现在基本等价于 Table）

**说明与修正：**
- **特点**：半灵活稀疏宽表；LSM/SSTable；按分区键分布、分区内按聚簇键有序；一致性可调（ONE/QUORUM/ALL）
- **最佳实践**：查询必须以分区键为前提；单分区建议控制在 <100MB、条目 <10 万；二级索引谨慎使用
- **备注**：与 HBase/Bigtable 同属宽列家族，但布局/一致性模型不同（Dynamo 风格分布与可调一致性）

---

#### 列族型数据库（HBase）

**示例：**
```
HBase 集群
└─ ecommerce (Namespace)
   └─ users (Table)
      ├─ RowKey: user#1
      │   └─ CF:info → {name:"Alice", age:25}
      └─ RowKey: user#2
          └─ CF:info → {name:"Bob"}
```

**对应关系：**
- 列族（Column Family, CF）为物理聚簇单元
- 列限定符（Qualifier）动态出现
- 未写入列不占空间

**说明与修正：**
- **特点**：行键字典序、范围扫描/前缀扫描；多版本/TTL；单行原子写
- **最佳实践**：列族尽量少（通常 <10）；rowkey 防热点（前缀散列/复合键）；避免全表扫描；二级索引需 Phoenix 或自建方案
- **备注**：与传统 OLAP "列式存储"不同，HBase 是列族宽表，面向在线读写与范围扫描

---

#### 列族型数据库（Bigtable, GCP）

**示例：**
```
Bigtable 实例
└─ users (Table)
   ├─ RowKey: user#1
   │   └─ cf1:name → "Alice"
   └─ RowKey: user#2
       └─ cf1:age → 30
```

**说明：**
- **特点**：托管化列族宽表；行键有序范围切分；过滤器体系；版本/TTL；单行原子，跨行无事务
- **最佳实践**：行键设计决定延迟与成本；控制单值大小与单行总量；无原生二级索引（用冗余写/外部系统）

---

#### 搜索引擎（Elasticsearch）

**示例：**
```
ES 集群
└─ users (Index)
   ├─ Document 1: {"name":"Alice","age":25}
   └─ Document 2: {"name":"Bob","age":30}
```

**说明与修正：**
- **对应关系**：Index ≈ Table（现代版本）；Document ≈ Row。ES 没有与 MySQL Database 完全等价的层
- **特点**：倒排索引、全文检索、过滤与聚合；最终一致
- **最佳实践**：控制动态字段与字段基数；为聚合类型选择合适数据类型（keyword、numeric）；规划刷新/合并策略
- **备注**：早期的 _type 已弃用，不再作为"表"的类比层次

---

#### OLAP 列式（ClickHouse）

**示例：**
```
ClickHouse
└─ ecommerce (Database)
   └─ sales (Table, 列式)
      ├─ Row 1: (ts, user_id, sku, price)
      └─ Row N: ...
```

**说明：**
- **特点**：列式存储、向量化执行；主键=排序键（稀疏索引）；分区/排序/采样键驱动裁剪；批量写入
- **架构**：Shared-Nothing（开源版），计算存储耦合；Cloud 版采用 SharedMergeTree 存算分离
- **扩容限制**：🔴 开源版无自动 Rebalance，数据移动代价昂贵；需手动迁移或配置写入权重
- **最佳实践**：
  - 设计好分区与排序键；批量导入（避免频繁单行写）
  - 利用物化视图/聚合表提速
  - 提前规划分片数，扩容时优先增加副本而非分片
  - 使用 Distributed 表 + 随机分布避免数据倾斜

---

#### OLAP 列式（Snowflake）

**示例：**
```
Snowflake
└─ ANALYTICS_DB (Database)
   └─ SALES (Table, 列式)
```

**说明：**
- **特点**：存算分离、微分片与统计裁剪；可设置聚簇键；支持半结构化（VARIANT）
- **最佳实践**：根据查询分布选择分区/聚簇列；控制扫描量；治理长期未使用的表和物化视图

---

#### OLAP/时序（Druid）

**示例：**
```
Druid 集群
└─ sales (Datasource)
   ├─ event-time 分段
   └─ 维度/度量 列式存储 + 倒排/位图
```

**说明：**
- **特点**：实时/近实时摄取；时间分段；倒排/位图索引；roll-up 降采样
- **最佳实践**：控制高基数维度；合理段大小与保留策略；roll-up 规划

---

#### 列式仓库（BigQuery, GCP）

**示例：**
```
BigQuery
└─ dataset: analytics
   └─ table: sales (列式)
```

**说明：**
- **特点**：列式、分区/聚簇、列压缩与裁剪；按扫描量计费；与对象存储生态协同
- **最佳实践**：必须设计分区/聚簇列；避免 select *；用物化视图与分区过滤减少扫描

---

#### 图数据库（Neo4j）

**示例：**
```
Neo4j 实例
└─ neo4j (Database)
   ├─ Node: User {id: 1, name: "Alice"}
   ├─ Node: Product {id: 101, name: "iPhone"}
   └─ Relationship: (User{id:1})-[:BOUGHT {ts: 2025-11-10}]→(Product{id:101})
```

**对应与特点：**
- 无"表"概念，通过 Label 给节点/关系打标签
- Node/Relationship 相当于一行实体和一条边，但具备图语义
- 强项是模式匹配与遍历（Cypher MATCH），对关系跳数、路径查询、推荐等非常高效
- 支持属性索引、全文与空间索引

**最佳实践：**
- 为常用过滤属性建立索引（CREATE INDEX FOR (n:User) ON (n.id)）
- 对关系密集的超级节点进行分层或引入中间节点，避免遍历风暴
- 用 MERGE 保证幂等写入
- 为时间序按月/日拆分关系或引入时间节点，降低边集合规模

**一句话避坑：**
- 不要把需要海量聚合/宽表扫描的数仓类查询丢给 Neo4j；它擅长遍历，不擅长大规模列式聚合
- 超级节点需建模控制（如分桶/分层/反范式边），否则单点会成为性能瓶颈

---

#### 关于"存储引擎选择粒度"的修正

- **MySQL**：表级选择存储引擎（现代生产基本用 InnoDB；MyISAM 不再推荐用于事务场景）
- **PostgreSQL**：没有"切换存储引擎"的概念；只有"为表建立不同类型索引"（B-Tree/GiST/GIN/BRIN 等）与分区/存储参数
- **MongoDB**：现代版本统一 WiredTiger；不支持集合级切换到 LSM，也不存在 "type=file/type=lsm" 用法

---

#### Schema 灵活性对比

- **关系型**：严格 Schema，强约束与 ACID；适合事务、一致性强的结构化数据
- **文档型**：灵活 Schema，嵌套/半结构化；用索引保障可查性与性能，支持事务（现代版本）
- **键值型**：无表 Schema，key 约定即模型；极简高性能，但查询表达力有限
- **列族宽表**：半灵活（列族固定、列动态/稀疏）；擅长按键点查/前缀/范围与时序明细
- **图数据库**：灵活 Schema；关系为一等公民；擅长路径/遍历/图算法，弱于聚合类大扫描

---

### Cloud Native Database

| 类型      | 使用场景                 | AWS                               | GCP                                             | Azure                         | 架构模式                                            | 关键特性                                      |
| :------ | :------------------- | :-------------------------------- | :---------------------------------------------- | :---------------------------- | :---------------------------------------------- | :---------------------------------------- |
| 关系型数据库  | 结构化数据，事务处理，<br>企业级应用 | RDS <br>(MySQL/PostgreSQL/Oracle) | Cloud SQL, <br>**AlloyDB(PostgreSQL 兼容，HTAP)**,<br>**Cloud Spanner(全球分布式数据库)**      | Azure SQL Database            | AWS: Primary/Standby<br>GCP: 分布式（Spanner）/主从（AlloyDB）       | 强事务支持（ACID），数据完整性<br>与一致性保证，SQL 标准查询      |
| 键值数据库   | 高速缓存，简单数据存取          | DynamoDB, <br>ElastiCache         Memorystore | Azure Cache for Redis         | AWS: 分布式<br>GCP: 分布式                            | 简单键值存储，高性能（微秒级延迟），<br>分布式支持，水平扩展          |
| 文档数据库   | 半结构化数据，灵活模式          | DocumentDB                        | **Firestore (Native mode)**                     | Cosmos DB <br>(MongoDB API)   | AWS: Primary/Replicas<br>GCP: 分布式               | JSON/BSON 文档存储，灵活 Schema，<br>嵌套查询支持，索引自动化 |
| 宽列存储数据库 | 海量数据，时序数据，<br>高吞吐写入  | Keyspaces (Cassandra)             | **Bigtable**                                    | Cosmos DB <br>(Cassandra API) | AWS: 分布式<br>GCP: 分布式                            | 宽列存储（Wide-Column），<br>PB 级扩展，高吞吐写入，仅主键查询  |
| 图数据库    | 关系网，社交网络，推荐系统        | Neptune                           | 无原生服务<br>（可用 Neo4j on GCP）                      | Cosmos DB <br>(Gremlin API)   | AWS: Primary/Replicas                           | 高效图关系存储与查询，<br>支持 Gremlin/Cypher，多跳关系遍历   |
| 时序数据库   | 交易数据，基于时间的数据         | Timestream                        | 无原生服务<br>（可用 InfluxDB on GCP）                   | Azure Data Explorer           | AWS: 分布式                                        | 时间序列优化，高压缩比，<br>时间窗口聚合，IoT/监控场景           |
| 列式数据仓库  | OLAP 分析，BI 报表        | **Redshift**                      | **BigQuery**                                    | Synapse Analytics             | AWS: MPP 集群<br>GCP: 无服务器分布式                     | 列式存储，MPP 架构，SQL 分析，<br>PB 级数据处理，与 BI 工具集成 |
| 全球分布式   | 多区域强一致，全球应用          | Aurora Global Database            | **Cloud Spanner**                               | Cosmos DB                     | AWS: Primary + 跨区域 Replicas<br>GCP: 全球分布式 Paxos | 全球分布式强一致性，多区域自动复制，<br>99.999% SLA，外部一致性快照 |


---

**架构哲学差异：**
 - **GCP**：偏好天生分布式架构（Spanner、Firestore、Bigtable），源自 Google 内部系统（搜索、广告、Gmail）的全球化需求
 - **AWS**：提供多种选择，RDS/DocumentDB 采用传统 Primary/Replicas 架构（兼容性优先），DynamoDB 采用分布式架构（云原生）
 - **Azure**：Cosmos DB 统一多模型分布式架构

**GCP 的分布式优势：GFE（Google Front End）统一入口**

GCP 几乎所有服务都使用 **GFE 作为统一入口**，这是 GCP 分布式架构的基石：

| 服务类型      | GCP 服务        | 使用 GFE | AWS 对应服务               | AWS 架构                    |
| --------- | ------------- | ------ | ---------------------- | ------------------------- |
| **数据库**   | Cloud Spanner | ✅ 默认   | Aurora Global          | 需配置 Global Accelerator    |
|           | Firestore     | ✅ 默认   | DynamoDB Global Tables | 需配置 Route 53 + CloudFront |
|           | Bigtable      | ✅ 默认   | Keyspaces              | 区域 Load Balancer          |
|           | Cloud SQL     | ✅ 默认   | RDS                    | 区域 Load Balancer          |
| **计算**    | Cloud Run     | ✅ 默认   | Fargate + ALB          | 需配置 ALB + CloudFront      |
|           | GKE           | ✅ 默认   | EKS + ALB              | 需配置 ALB/NLB               |
| **存储**    | Cloud Storage | ✅ 默认   | S3                     | 需配置 CloudFront            |
| **数据仓库**  | BigQuery      | ✅ 默认   | Redshift               | 区域 Load Balancer          |
| **AI/ML** | Vertex AI     | ✅ 默认   | SageMaker              | 区域 Endpoint               |

**GFE 核心特性：**
- **Anycast 路由**：全球任意位置访问同一 IP，自动路由到最近的接入点（延迟降低 50-80%）
- **边缘 DDoS 防护**：攻击流量在边缘拦截，不进入数据中心，利用 Google 全球网络吸收攻击
- **TLS 终止**：统一证书管理，支持最新 TLS 协议，减轻后端服务负担
- **全球负载均衡**：自动选择最佳后端（地理位置、健康检查、负载），无需手动配置跨区域负载均衡
- **零配置**：默认启用，开发者只需部署服务，GFE 自动处理所有网络复杂度

**为什么 GFE 特别适合分布式：**
```
传统架构（AWS 风格）：
客户端 → 区域 Load Balancer → 服务
问题：需要组合 CloudFront + ALB + Route 53 + Global Accelerator

GCP 架构（GFE 风格）：
客户端 → GFE（全球统一入口）→ 服务
         ↑
    Anycast IP（全球同一 IP，自动路由到最近节点）
优势：统一入口，零配置，自动优化
```

**GFE 支撑的分布式场景：**
- Spanner 全球分布：美国/欧洲/亚洲用户自动路由到最近节点，同一 IP 全球访问
- Firestore 实时同步：边缘智能路由，优化跨区域延迟
- Bigtable 高吞吐：流量智能分流，故障自动切换

这是 Google 20+ 年分布式系统经验的结晶，也是 GCP 相比 AWS 的重要架构优势。

---

**Cloud SQL vs Cloud Spanner 的核心区别：**

| 维度      | Cloud SQL                                                            | Cloud Spanner     |
| ------- | -------------------------------------------------------------------- | ----------------- |
| 定位      | 托管的传统关系型数据库                                                          | 全球分布式 NewSQL 数据库  |
| 数据库引擎   | MySQL、PostgreSQL、SQL Server                                          | Google 自研（类 SQL）  |
| 架构      | 单区域主从复制                                                              | 全球多区域分布式          |
| 一致性     | 最终一致性（跨区域）                                                           | 全球强一致性（外部一致性）     |
| 扩展性     | 垂直扩展（升级实例）                                                           | 水平扩展（自动分片）        |
| 延迟      | 单区域：毫秒级<br>跨区域：秒级                                                    | 全球：单数字毫秒级         |
| 事务      | ACID（单区域）                                                            | ACID（全球分布式）       |
| 适用场景    | 传统应用迁移、单区域应用                                                         | 全球应用、金融、供应链       |
| 价格      | 低（按实例计费）                                                             | 高（按节点 + 存储计费）     |
| SLA     | 99.95%（高可用配置）                                                        | 99.999%（5 个 9）    |
| SQL 兼容性 | 完全兼容 MySQL/PostgreSQL                                                | 类 SQL（有差异）        |
| 最大数据库大小 | PostgreSQL：64 TB<br>MySQL：64 TB（8.0）/ 30 TB（5.7）<br>SQL Server：16 TB | 无限制（PB 级，实际受配额限制） |

## NoSQL 数据库

### 核心概念速查：理解键值与文档数据库的6大维度

> **目标**：建立系统性认知框架，从底层原理到应用场景全面理解 DynamoDB、DocumentDB、Firestore Native、MongoDB 的核心机制与差异。
> 
> **注意**：Bigtable（宽列存储数据库）将在后续章节单独讨论。

#### 概念层 1：数据模型层

| 数据库                  | 数据模型 | 主键设计                             | Schema灵活性    | 嵌套支持            | 数组支持            | 支持的数据类型                                                                                      |
| :------------------- | :--- | :------------------------------- | :----------- | :-------------- | :-------------- | :------------------------------------------------------------------------------------------- |
| **DynamoDB**         | 键值   | Partition Key + Sort Key（可选）     | 无Schema（黑盒）  | 🔴 可存储但不可查询     | 🔴 可存储但不可查询     | String, Number, Binary, Boolean, Null<br>List, Map, Set<br>🔴 无Date类型（需转换为Number/String） |
| **MongoDB**          | 文档   | _id（自动生成或自定义）                    | 灵活Schema（透明） | ✅ 任意层级          | ✅ 原生支持          | String, Number, Boolean, Date, ObjectId<br>Array, Object, Binary, Null, Regex<br>✅ Decimal128（高精度） |
| **DocumentDB**       | 文档   | _id（兼容MongoDB）                   | 灵活Schema（透明） | ✅ 任意层级          | ✅ 原生支持          | 兼容MongoDB类型<br>🔴 不支持：Decimal128, ClientSession<br>⚠️ 部分支持：某些聚合操作符                      |
| **Firestore Native** | 文档   | 自动生成ID或自定义                       | 灵活Schema（透明） | ✅ 嵌套Map         | ✅ 原生支持          | String, Number, Boolean, Timestamp<br>Array, Map, GeoPoint, Reference<br>🔴 无Binary类型         |

**核心洞察**：
- **键值模型（DynamoDB）**：Value是黑盒，可以存储嵌套对象和数组，但无法查询内部字段，只能通过主键访问整个Value
- **文档模型（MongoDB/DocumentDB/Firestore）**：Value透明可查询，支持复杂嵌套结构和数组，可对任意字段建索引和查询

**数据类型差异说明**：
- **DynamoDB**：无原生Date类型，需要用Number（Unix时间戳）或String（ISO 8601）存储
- **MongoDB**：支持最丰富的数据类型，包括高精度Decimal128（金融场景）
- **DocumentDB**：兼容MongoDB大部分类型，但不支持Decimal128（金融场景需注意）
- **Firestore Native**：有原生Timestamp类型，但无Binary类型（需Base64编码）

---

#### 概念层 2：存储引擎层

| 数据库 | 存储引擎 | 写优化 | 读优化 | 压缩能力 | 存储特性 |
|:---|:---|:---|:---|:---|:---|
| **DynamoDB** | 类LSM-Tree* | ✅ 高吞吐写入 | ⚠️ 需索引优化 | 自动压缩 | 写放大4-10x |
| **MongoDB** | WiredTiger（B+Tree/LSM可选） | ⚠️ 默认B+Tree | ✅ B+Tree读写平衡 | 2-7x压缩 | 文档级锁+MVCC |
| **DocumentDB** | Aurora存储层 | ⚠️ 日志优先写入 | ✅ 共享存储 | 自动压缩 | 6副本跨AZ、MVCC(类MongoDB) |
| **Firestore Native** | Megastore | ⚠️ Paxos写入 | ✅ 强一致读 | 自动压缩 | Entity Group限制、MVCC(多版本) |

> *DynamoDB存储引擎未公开，基于性能特征推测为LSM-Tree架构

**核心洞察**：
- **LSM-Tree**：写入先到内存（MemTable）再批量刷盘（SSTable），写优化但有写放大
- **B+Tree**：原地更新，读写平衡，MongoDB的WiredTiger支持两种引擎切换
- **存储与索引分离**：索引用B+Tree查找位置，存储用LSM-Tree/B+Tree持久化数据

---

#### 概念层 3：分布式架构层

| 数据库                  | 架构模式           | 分区策略                           | 复制模式                | 一致性模型           | 跨区域           |
| :------------------- | :------------- | :----------------------------- | :------------------ | :-------------- | :------------ |
| **DynamoDB**         | 分布式            | Hash分区（Partition Key）          | 主从复制（3副本跨AZ）        | 最终一致/强一致可选      | Global Tables |
| **MongoDB**          | 副本集+分片         | Hash/Range/Zone分片（Shard Key策略） | Raft共识（副本集）         | 可调一致性           | 跨区域副本集        |
| **DocumentDB**       | 主从架构（Aurora存储） | 无分片（单集群）                       | 6副本同步（存储层）+异步（只读副本） | 强一致（主）/最终一致（只读） | 跨区域只读副本       |
| **Firestore Native** | 分布式            | 自动分区                           | Paxos共识             | 强一致             | 多区域自动         |

**核心洞察**：
- **分区策略**：Hash分区（均匀分布）vs Range分区（范围查询优化）
- **复制模式**：异步复制（低延迟）vs 共识算法（强一致但高延迟）
- **一致性权衡**：DynamoDB最终一致延迟1-3ms，Firestore强一致延迟5-10ms（Paxos代价）

---

#### 概念层 4：索引与查询层

| 数据库                  | 主键索引                     | 二级索引    | 复合索引    | 查询能力        | 全文搜索     |
| :------------------- | :----------------------- | :------ | :------ | :---------- | :------- |
| **DynamoDB**         | Partition Key + Sort Key | GSI/LSI | ✅ 复合键   | 仅主键+GSI     | 🔴 不支持   |
| **MongoDB**          | _id                      | ✅ 任意字段  | ✅ 多字段复合 | 任意字段+聚合     | ✅ Text索引 |
| **DocumentDB**       | _id                      | ✅ 任意字段  | ✅ 多字段复合 | 任意字段+聚合     | ⚠️ 有限支持  |
| **Firestore Native** | 自动ID                     | ✅ 自动索引  | ✅ 复合索引  | 任意字段（无JOIN） | 🔴 不支持   |

**核心洞察**：
- **键值数据库**：只能通过主键或预定义GSI查询，无法临时查询任意字段
- **文档数据库**：可对任意字段建索引并查询，支持复杂过滤和聚合
- **索引结构**：都用B+Tree索引（快速查找），但存储引擎可能是 LSM-Tree

---

#### 概念层 5：事务与并发层

| 数据库                  | 事务范围                | 隔离级别                                  | 并发控制      | 锁粒度          | MVCC           |
| :------------------- | :------------------ | :------------------------------------ | :-------- | :----------- | :------------- |
| **DynamoDB**         | 单分区/跨分区（性能较低）       | 可串行化（事务）<br>读已提交（非事务）                | 乐观锁（条件写入） | 项级别          | 🔴 无           |
| **MongoDB**          | 跨分片/跨文档             | 快照隔离（默认）<br>可调：读未提交/读已提交/可重复读/线性化 | MVCC      | 文档级别         | ✅ Update Chain |
| **DocumentDB**       | 跨文档（4.0+）          | 快照隔离（默认）<br>兼容MongoDB隔离级别            | MVCC      | 文档级别         | ✅ 类MongoDB     |
| **Firestore Native** | Entity Group内（有限制） | 可串行化（强制）                             | Paxos共识   | Entity Group | ✅ 多版本          |

**核心洞察**：
- **MVCC机制**：多版本并发控制，读不阻塞写，MongoDB 用 Update Chain，PostgreSQL 用 Undo Log
- **锁粒度**：文档级锁（MongoDB/DocumentDB）优于表锁，支持高并发
- **事务限制**：Firestore的 Entity Group限制（25个实体/秒），DynamoDB 跨分区事务性能较低

---

#### 概念层 6：性能特征层

| 数据库                  | P99延迟                      | 吞吐模型   | 热点处理  | 扩展性       | 成本模型   |
| :------------------- | :------------------------- | :----- | :---- | :-------- | :----- |
| **DynamoDB**         | 1-3ms（最终一致）                | 预配置/按需 | 自适应容量 | 无限水平扩展    | 按请求+存储 |
| **MongoDB**          | 5-10ms（点查）<br>50-500ms（聚合） | 实例配置   | 手动分片  | 水平分片      | 按实例    |
| **DocumentDB**       | 5-15ms                     | 实例配置   | 读副本扩展 | 垂直扩展（无分片） | 按实例    |
| **Firestore Native** | 5-10ms（强一致）                | 自动扩展   | 自动分区  | 无限水平扩展    | 按读写+存储 |

**核心洞察**：
- **延迟差异**：DynamoDB最低（异步复制），Firestore较高（Paxos共识代价）
- **热点问题**：DynamoDB需设计好Partition Key避免热点
- **成本权衡**：DynamoDB按需模式适合不可预测负载，MongoDB实例模式适合稳定负载

---

### 选型决策树：基于核心机制的数据库选择

#### 决策流程图

```
开始选型
    │
    ├─ 需要查询Value内部字段？
    │   │
    │   ├─ 否 → 键值数据库
    │   │   │
    │   │   ├─ 需要极致性能（1-3ms）？
    │   │   │   └─ 是 → DynamoDB ✅
    │   │   │
    │   │   │
    │   │   └─ 遗留App Engine项目？
    │   │
    │   └─ 是 → 文档数据库
    │       │
    │       ├─ 需要实时订阅推送？
    │       │   │
    │       │   ├─ 是 + 移动/Web应用
    │       │   │   └─ Firestore Native 🌟
    │       │   │
    │       │   └─ 是 + 后端应用
    │       │       └─ MongoDB ✅
    │       │
    │       ├─ 需要MongoDB兼容性？
    │       │   │
    │       │   ├─ 是 + AWS生态
    │       │   │   └─ DocumentDB ✅
    │       │   │
    │       │   └─ 是 + 完整功能
    │       │       └─ MongoDB Atlas ✅
    │       │
    │       └─ 通用文档存储
    │           │
    │           ├─ AWS生态 → DocumentDB ✅
    │           ├─ GCP生态 → Firestore Native 🌟
    │           └─ 多云/自建 → MongoDB ✅
```

---

#### 场景化选型矩阵

| 场景 | 首选方案 | 备选方案 | 不推荐 | 核心原因 |
|:---|:---|:---|:---|:---|
| **会话存储** | DynamoDB | Redis | MongoDB | 简单KV，无需查询Value，极致性能 |
| **用户配置** | DynamoDB | Firestore Native  KV模型足够，DynamoDB延迟最低 |
| **商品目录** | MongoDB | DocumentDB | DynamoDB | 需要多字段查询（价格、分类、标签） |
| **订单系统** | MongoDB | DocumentDB | Firestore Native | 需要复杂聚合（统计、报表） |
| **移动App** | Firestore Native | MongoDB | DynamoDB | 实时订阅+离线支持+自动同步 |
| **IoT时序数据**  DynamoDB | MongoDB | 宽列模型，高吞吐写入，时间范围查询 |
| **日志存储**  DynamoDB | DocumentDB | 追加写入，按时间范围查询 |
| **内容管理** | MongoDB | Firestore Native | DynamoDB | 富文本、嵌套结构、全文搜索 |
| **社交Feed** | DynamoDB  MongoDB | 按用户ID查询，高并发读写 |
| **实时聊天** | Firestore Native | MongoDB | DynamoDB | 实时推送，离线消息同步 |

---

#### 核心机制决策要点

##### 1. 数据模型层决策

**选择键值数据库（DynamoDB）的条件**：
- ✅ 访问模式简单（只通过主键查询）
- ✅ 需要极致性能（毫秒级延迟）
- ✅ 数据结构扁平（无复杂嵌套）
- 🔴 big 不需要查询Value内部字段

**选择文档数据库（MongoDB/DocumentDB/Firestore）的条件**：
- ✅ 需要多字段查询（姓名、年龄、城市等）
- ✅ 数据结构复杂（嵌套对象、数组）
- ✅ Schema灵活变化（字段动态增减）
- ✅ 需要聚合分析（统计、分组）

---

##### 2. 存储引擎层决策

**选择LSM-Tree架构（DynamoDB）的条件**：
- ✅ 写多读少（写入吞吐 > 读取吞吐）
- ✅ 可接受写放大（4-10x磁盘写入）
- ✅ 顺序扫描为主（时序数据）
- ⚠️ 需要注意Compaction对性能的影响

**选择B+Tree架构（MongoDB WiredTiger默认）的条件**：
- ✅ 读写平衡（读写吞吐相近）
- ✅ 随机读取为主（点查询）
- ✅ 需要原地更新（减少写放大）
- ✅ 需要范围查询（B+Tree天然有序）

---

##### 3. 分布式架构层决策

**选择最终一致性（DynamoDB）的条件**：
- ✅ 可接受短暂数据不一致（秒级）
- ✅ 需要极致性能（1-3ms延迟）
- ✅ 跨区域部署（Global Tables）
- ✅ 高可用优先于一致性

**选择强一致性（Firestore Native）的条件**：
- ✅ 必须保证数据一致性（金融、库存）
- ⚠️ 可接受较高延迟（5-10ms，Paxos代价）
- ✅ 需要事务保证（Entity Group内）
- ✅ 简化应用逻辑（无需处理最终一致性）

**选择可调一致性（MongoDB）的条件**：
- ✅ 不同场景需要不同一致性级别
- ✅ 需要灵活权衡性能与一致性
- ✅ 复杂业务逻辑（部分强一致，部分最终一致）

---

##### 4. 索引与查询层决策

**选择仅主键索引（DynamoDB）的条件**：
- ✅ 访问模式固定且可预测
- ✅ 可以通过GSI覆盖所有查询（DynamoDB）
- ✅ 不需要临时查询（Ad-hoc Query）
- ⚠️ 需要提前设计好所有访问路径

**选择任意字段索引（MongoDB/DocumentDB/Firestore）的条件**：
- ✅ 访问模式多样且不可预测
- ✅ 需要临时查询和数据探索
- ✅ 需要复杂过滤条件（多字段组合）
- ✅ 需要聚合分析（GROUP BY、SUM等）

---

##### 5. 事务与并发层决策

**选择单分区事务（DynamoDB）的条件**：
- ✅ 事务范围小（同一Partition Key）
- ✅ 需要极致性能（跨分区事务慢）
- ✅ 可以通过设计避免跨分区事务
- ⚠️ 跨分区事务延迟高（10-50ms）

**选择跨分片事务（MongoDB）的条件**：
- ✅ 需要跨多个文档的ACID事务
- ✅ 复杂业务逻辑（转账、库存扣减）
- ⚠️ 可接受事务性能开销
- ✅ 需要快照隔离级别

**选择Entity Group事务（Firestore Native）的条件**：
- ✅ 事务范围可控（25个实体/秒限制）
- ✅ 需要强一致性保证
- ⚠️ 可接受Entity Group限制
- ✅ 移动/Web应用（事务需求简单）

---

##### 6. 性能特征层决策

**选择预配置容量（DynamoDB）的条件**：
- ✅ 负载可预测（稳定流量）
- ✅ 需要成本优化（预留容量折扣）
- ✅ 可接受容量规划（需要监控调整）

**选择按需容量（DynamoDB/Firestore）的条件**：
- ✅ 负载不可预测（突发流量）
- ✅ 开发测试环境（低成本）
- ✅ 无需容量规划（自动扩展）
- ⚠️ 成本较高（按请求计费）

**选择实例模式（MongoDB/DocumentDB）的条件**：
- ✅ 负载稳定（持续高吞吐）
- ✅ 需要成本可控（固定月费）
- ✅ 需要完整数据库功能（管理员权限）

---

#### 反模式警告

##### ❌ 不要用MongoDB做简单KV存储
- **问题**：过度设计，浪费资源
- **原因**：MongoDB的文档解析、索引维护、MVCC开销对简单KV是浪费
- **替代**：DynamoDB（持久化KV）或Redis（缓存KV）

##### ❌ 不要用DynamoDB做复杂查询
- **问题**：需要创建大量GSI，成本高且维护复杂
- **原因**：DynamoDB只能通过主键+GSI查询，无法临时查询任意字段
- **替代**：MongoDB或DocumentDB

##### ❌ 不要用Firestore Native做后端复杂业务
- **问题**：查询能力受限（无JOIN、无聚合）
- **原因**：Firestore定位移动/Web，不支持复杂查询
- **替代**：MongoDB Atlas

##### ❌ 不要用DocumentDB做实时订阅
- **问题**：不支持Change Streams
- **原因**：DocumentDB是MongoDB兼容层，未实现Change Streams
- **替代**：MongoDB Atlas或Firestore Native

---

#### 迁移路径建议

##### 从关系型数据库迁移

**MySQL/PostgreSQL → MongoDB/DocumentDB**：
- ✅ 适合：复杂嵌套数据（JSON字段多）
- ✅ 适合：Schema频繁变化
- ⚠️ 注意：需要重新设计数据模型（反范式化）
- ⚠️ 注意：事务语义差异（快照隔离 vs 读已提交）

**MySQL/PostgreSQL → DynamoDB**：
- ✅ 适合：简单KV访问模式
- ✅ 适合：需要无限扩展
- ⚠️ 注意：需要彻底重新设计（主键设计、GSI规划）
- ⚠️ 注意：无法支持复杂JOIN和聚合

##### 从NoSQL数据库迁移

**MongoDB → DocumentDB**：
- ✅ 兼容性高（API兼容）
- ⚠️ 注意：部分功能缺失（Change Streams、某些聚合操作）
- ⚠️ 注意：性能差异（需要测试验证）

**DynamoDB → MongoDB**：
- ✅ 适合：需要更灵活的查询能力
- ⚠️ 注意：需要重新设计索引策略
- ⚠️ 注意：性能特征差异（延迟可能增加）

---

### 总结：核心机制驱动的选型原则

1. **数据模型决定查询能力**：需要查询Value内部字段就选文档数据库，否则选键值数据库
2. **存储引擎决定性能特征**：写多读少选LSM-Tree，读写平衡选B+Tree
3. **分布式架构决定一致性**：强一致选Paxos/Raft，高性能选异步复制
4. **索引能力决定灵活性**：访问模式固定选主键索引，多样化选任意字段索引
5. **事务范围决定复杂度**：简单事务选单分区/单文档，复杂事务选跨分片
6. **成本模型决定经济性**：稳定负载选实例模式，突发流量选按需模式

**最终建议**：
- **AWS生态 + 简单KV** → DynamoDB
- **AWS生态 + 文档存储** → DocumentDB
- **GCP生态 + 移动/Web** → Firestore Native
- **多云/完整功能** → MongoDB Atlas

---

## 键值数据库 vs 文档数据库

### 核心区别：查询能力（文档数据库可以查询任何嵌套的 key 或 value，键值数据库只能根据 key 查询）

**键值数据库（Key-Value Database）**
- 只能通过 **主键** 查询
- 无法查询 Value 内部的字段
- Value 对数据库来说是 **黑盒**（不透明）
- 代表：DynamoDB、Redis

**文档数据库（Document Database）**
- 可以通过 **任意字段** 查询
- 可以查询嵌套字段、数组元素
- Value 对数据库来说是 **透明的**（可解析）
- 代表：MongoDB、Firestore Native mode、DocumentDB

### 键值数据库 vs 文档数据库深度解析

#### 1. 产品分类与核心差异

**产品对比：**

| 数据库                  | 类型  | 云厂商 | Value可见性 | 查询能力       | 索引能力     | 实时订阅             | 推荐度         | 适用场景      |
| :------------------- | :-- | :-- | :------- | :--------- | :------- | :--------------- | :---------- | :-------- |
| **DynamoDB**         | 键值  | AWS | 🔴 黑盒    | 只能主键查询     | 仅主键+GSI  | ⚠️ Streams       | ✅ 推荐        | 高性能KV存储   |
| **MongoDB**          | 文档  | 开源  | ✅ 透明     | **任意字段查询** | **任意字段** | ✅ Change Streams | ✅ 推荐        | 通用文档存储    |
| **DocumentDB**       | 文档  | AWS | ✅ 透明     | **任意字段查询** | **任意字段** | 🔴 不支持           | ✅ 推荐        | MongoDB兼容 |
| **Firestore Native** | 文档  | GCP | ✅ 透明     | **任意字段查询** | **任意字段** | ✅ 原生支持           | 🌟 **强烈推荐** | 移动/Web应用  |

> **重要说明**：
> - **Firestore Native**：GCP 主推产品，所有新项目应优先选择，功能最强（实时订阅、离线支持、自动索引）
> - 同一 GCP 项目只能选择一种 Firestore 模式（Native 或 Datastore），**选择后无法切换**

**核心差异：Value 的可见性决定查询能力**

```
键值数据库（Key-Value）：
┌─────────────────────────────────────┐
│  数据库视角                           │
│  Key: "user:123"                    │
│  Value: [0x1A2B3C4D...]  ← 黑盒      │
│                                     │
│  特点：                              │
│  • 数据库不解析 Value 内部结构         │
│  • 只能通过 Key 查询                  │
│  • 无法对 Value 内部字段建索引         │
│  • 查询：GetItem(key)                │
│  • 性能：极高（O(1)查询）              │
└─────────────────────────────────────┘

文档数据库（Document）：
┌─────────────────────────────────────┐
│  数据库视角                           │
│  {                                  │
│    "_id": "user:123",               │
│    "name": "Alice",      ← 透明      │
│    "age": 30,            ← 透明      │
│    "address": {          ← 透明      │
│      "city": "Beijing"   ← 透明      │
│    }                                │
│  }                                  │
│                                     │
│  特点：                              │
│  • 数据库解析所有字段                  │
│  • 可以通过任意字段查询                │
│  • 可以对嵌套字段建索引                │ 
│  • 查询：find({age: {$gt: 25}})      │
│  • 性能：高（O(log N)索引查询）        │
└─────────────────────────────────────┘
```

---

### 数据库架构对比

#### 键值数据库架构

##### DynamoDB 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         客户端层                                 │
│  AWS SDK (Java/Python/Node.js) │ AWS CLI │ Console │ PartiQL    │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      API Gateway 层                             │
│  ├─ 请求路由（根据 Table 路由）                                    │
│  ├─ 认证授权（IAM）                                               │
│  ├─ 限流控制（Throttling）                                        │
│  └─ 请求验证（Schema Validation）                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      查询处理层                                   │
│  ├─ PartiQL 解析器（SQL → DynamoDB API）                          │
│  ├─ 查询优化器（索引选择：主键 vs GSI）                              │
│  └─ 执行计划生成                                                  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      请求路由层                                   │
│  ├─ 一致性哈希（Consistent Hashing）                              │
│  ├─ Partition Key → 计算哈希值 → 定位存储节点                       │
│  ├─ 负载均衡（Load Balancing）                                    │
│  └─ 自动分区（Auto Partitioning）                                 │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      存储节点层（类 LSM-Tree 架构）                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Storage Node │  │ Storage Node │  │ Storage Node │          │
│  │ Partition 1  │  │ Partition 2  │  │ Partition 3  │          │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │          │
│  │ │内存缓冲层  │ │  │ │内存缓冲层 │  │  │ │内存缓冲层 │ │          │
│  │ │(写入优化)  │ │  │ │(写入优化) │ │  │ │(写入优化) │ │          │
│  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │          │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │          │
│  │ │ B+Tree   │ │  │ │ B+Tree   │ │  │ │ B+Tree   │ │          │
│  │ │ Index    │ │  │ │ Index    │ │  │ │ Index    │ │          │
│  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │          │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │          │
│  │ │磁盘持久层  │ │  │ │磁盘持久层 │ │  │ │磁盘持久层  │ │          │
│  │ │(分层存储) │ │  │ │(分层存储) │  │  │ │(分层存储) │ │          │
│  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      持久化层                                    │
│  ├─ WAL（Write-Ahead Log）- 顺序写入、崩溃恢复                      │
│  └─ AWS 分布式存储（SSD）- 3 副本跨 3 个 AZ                         │
└─────────────────────────────────────────────────────────────────┘
```

##### 2.2 文档数据库架构

###### 2.2.1 MongoDB 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         客户端层                                 │
│  MongoDB Driver (Java/Python/Node.js) │ Mongo Shell │ Compass   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      Mongos 路由层（分片集群）                     │
│  ├─ 查询路由（根据 Shard Key 路由）                                │
│  ├─ 查询合并（聚合多个 Shard 结果）                                 │
│  └─ 负载均衡                                                     │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      Shard 层（数据分片）                         │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │   Shard 1    │  │   Shard 2    │  │   Shard 3    │           │
│  │ (Replica Set)│  │ (Replica Set)│  │ (Replica Set)│           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                  单个 Shard 内部（Replica Set）                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Primary    │  │  Secondary   │  │  Secondary   │          │
│  │ ┌──────────┐ │  │              │  │              │          │
│  │ │查询处理层  │ │  │              │  │              │          │
│  │ │Parser    │ │  │              │  │              │          │
│  │ │Optimizer │ │  │              │  │              │          │
│  │ │Executor  │ │  │              │  │              │          │
│  │ └──────────┘ │  │              │  │              │          │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │          │
│  │ │WiredTiger│ │  │ │WiredTiger│ │  │ │WiredTiger│ │          │
│  │ │(B+Tree)  │ │  │ │(B+Tree)  │ │  │ │(B+Tree)  │ │          │
│  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │          │
│  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │          │
│  │ │ Indexes  │ │  │ │ Indexes  │ │  │ │ Indexes  │ │          │
│  │ │(B+Tree)  │ │  │ │(B+Tree)  │ │  │ │(B+Tree)  │ │          │
│  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│         ↓                  ↓                  ↓                 │
│      写入复制（Raft 协议）                                         │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      持久化层                                    │
│  ├─ Journal（WAL）- 崩溃恢复                                      │
│  └─ 数据文件（BSON）- 压缩存储（Snappy/zlib/zstd）                  │
└─────────────────────────────────────────────────────────────────┘
```

###### 2.2.2 DocumentDB 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         客户端层                                 │
│  MongoDB Driver (兼容 3.6/4.0) │ Mongo Shell │ Compass           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      MongoDB 兼容层                              │
│  ├─ Wire Protocol 转换                                           │
│  ├─ BSON 解析                                                    │
│  ├─ 查询语法转换（MongoDB → SQL）                                  │
│  └─ 聚合管道模拟                                                  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      查询引擎层                                   │
│  ├─ 查询优化器                                                    │
│  ├─ 执行计划生成                                                  │
│  └─ 分布式查询协调                                                │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      Aurora 存储架构                             │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  计算节点（Primary + Replicas）                              │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │ │
│  │  │   Primary    │  │   Replica 1  │  │   Replica 2  │      │ │
│  │  │   (读写)      │  │   (只读)     │  │   (只读)      │      │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘      │ │
│  └────────────────────────────────────────────────────────────┘ │
│                              ↓                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  存储层（6 副本跨 3 个 AZ）                                   │ │
│  │  ┌────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐  ┌────┐            │ │
│  │  │ AZ1│  │ AZ1│  │ AZ2│  │ AZ2│  │ AZ3│  │ AZ3│            │ │
│  │  │副本1│  │副本2│ │副本3│  │副本4│  │副本5│ │副本6│            │ │
│  │  └────┘  └────┘  └────┘  └────┘  └────┘  └────┘            │ │
│  │  写入：4/6 确认    读取：任意副本                              │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      持久化层                                    │
│  ├─ Redo Log（分布式 WAL）                                       │
│  ├─ 计算存储分离                                                  │
│  └─ 自动故障恢复（< 30秒）                                         │
└─────────────────────────────────────────────────────────────────┘
```

###### 2.2.3 Firestore Native 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         客户端层                                 │
│  Firebase SDK (Web/iOS/Android) │ Admin SDK │ REST API          │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      GFE 层（Google Front End）                  │
│  ├─ 全球负载均衡                                                  │
│  ├─ TLS 终止                                                     │
│  └─ 认证授权（Firebase Auth）                                     │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      Firestore 服务层                            │
│  ├─ 查询解析器（Query Parser）                                    │
│  ├─ 查询优化器（索引选择、查询计划）                                 │
│  ├─ 查询引擎（支持复合查询）                                        │
│  ├─ 实时订阅引擎（WebSocket）                                     │
│  ├─ 事务协调器（多文档 ACID）                                      │
│  └─ 索引管理（自动 + 手动）                                        │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      存储层（Megastore）                          │
│  ├─ 分布式存储（跨区域复制）                                        │
│  ├─ Paxos 复制（强一致性）                                         │
│  └─ 自动分片                                                      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      持久化层                                    │
│  └─ Colossus（Google 分布式文件系统）                              │
└─────────────────────────────────────────────────────────────────┘
```


##### 2.3 架构层次对比表（5个数据库）

| 层级       | DynamoDB<br>✅ 推荐 | Cloud Datastore<br>✅ 推荐 | MongoDB<br>✅ 推荐 | DocumentDB<br>✅ 推荐 | Firestore Native<br>🌟 推荐 |
| -------- | ---------------- | ------------------------ | --------------- | ------------------ | ------------------------- |
| **类型**   | 键值数据库            | 键值数据库                    | 文档数据库           | 文档数据库              | 文档数据库                     |
| **客户端层** | AWS SDK          | Cloud Datastore Client | MongoDB Driver     | MongoDB Driver（兼容）        | Firebase SDK   |
| **网关层**  | API Gateway      | GFE                    | Mongos（分片）         | MongoDB兼容层                | GFE            |
| **路由层**  | 一致性哈希            | Key路由                  | Shard Key路由        | 查询引擎                      | 自动路由           |
| **存储引擎** | 类LSM-Tree        | Megastore              | B+Tree（WiredTiger） | Aurora存储                  | Megastore      |
| **索引结构** | B+Tree（仅Key）     | B+Tree（仅Key）           | B+Tree（任意字段）       | B+Tree（任意字段）              | B+Tree（自动索引）   |
| **复制协议** | 3副本（跨AZ）         | Paxos（Entity Group）    | Replica Set（Raft）  | 6副本（跨AZ）                  | Paxos（跨Region） |
| **持久化**  | WAL + SSD        | Colossus               | Journal + BSON     | Redo Log（分布式）             | Colossus       |
| **分片方式** | Partition Key哈希  | Entity Group           | Shard Key范围/哈希     | 自动分片                      | 自动分片           |
| **事务支持** | 单项ACID           | Entity Group事务         | 多文档ACID            | 多文档ACID                   | 多文档ACID        |
| **一致性**  | 最终一致（可选强）        | 强一致（Entity Group内）     | 强一致（Primary）       | 强一致                       | 强一致（Paxos）     |
| **跨区域**  | ✅ Global Tables  | ⚠️ 手动配置                | ⚠️ 手动配置            | ✅ 原生支持                    | ✅ 原生支持         |

**Firestore Native 匹配移动端的原因**

1. ✅ 实时推送/订阅：WebSocket 长连接，自动推送变化
2. ✅ 离线支持：本地缓存，网络不稳定时仍可用
3. ✅ 简单查询：快速响应，适合移动端场景

| 维度   | 移动端需求       | Firestore Native | 后端需求       | Firestore Native |
| ---- | ----------- | ---------------- | ---------- | ---------------- |
| 实时推送 | ✅ 核心需求      | ✅ 完美支持           | 🔴 不需要     | ⚠️ 浪费资源          |
| 离线支持 | ✅ 必需（网络不稳定） | ✅ 自动缓存           | 🔴 不需要     | ⚠️ 无意义           |
| 简单查询 | ✅ 主要场景      | ✅ 快速响应           | ⚠️ 部分场景    | ✅ 支持             |
| 复杂查询 | 🔴 很少需要     | 🔴 不支持           | ✅ 核心需求     | 🔴 不支持           |
| 聚合计算 | 🔴 不需要      | 🔴 不支持           | ✅ 常用       | 🔴 不支持           |
| 批量操作 | 🔴 不需要      | ⚠️ 有限支持          | ✅ 常用       | 🔴 成本高           |
| 数据量  | ⚠️ 小（KB-MB） | ✅ 适合             | ✅ 大（GB-TB） | ⚠️ 成本高           |

**说明：**
- **存储引擎 vs 索引结构**：
  - `存储引擎`：数据如何组织和存储（LSM-Tree、B+Tree）
  - `索引结构`：如何快速查找数据（B+Tree索引）
  - DynamoDB：存储引擎用LSM-Tree（写优化），索引用B+Tree（查询优化）
- **B+Tree vs B-Tree**：
  - 实际数据库多使用B+Tree（数据仅在叶子节点，叶子节点有链表）
  - B+Tree范围查询更快，适合数据库场景
- **索引能力**：
  - `B+Tree（仅Key）`：键值数据库只能索引主键
  - `B+Tree（任意字段）`：文档数据库可索引任意字段，包括嵌套字段和数组
  - `B+Tree（自动索引）`：Firestore Native自动为查询字段创建索引
- **事务支持**：
  - `单项ACID`：仅支持单个Item的原子操作
  - `Entity Group事务`：仅在Entity Group内支持ACID事务
  - `多文档ACID`：支持跨Collection/Document的完整ACID事务
- **跨区域**：
  - `✅ 原生支持`：开箱即用的跨区域复制
  - `⚠️ 手动配置`：需要手动配置Replica Set或Entity Group的跨区域分布

---

#### 3. 核心机制深度对比

##### 3.1 存储引擎对比

**LSM-Tree vs B+Tree 架构差异：**

```
LSM-Tree（DynamoDB）：
写入流程：
┌─────────────────────────────────────┐
│  1. WAL（顺序写）        ← 极快       │
│  2. MemTable（内存）     ← 极快       │
│  3. 返回成功             ← < 1ms     │
│  4. 后台 Flush 到 SSTable            │
└─────────────────────────────────────┘

读取流程：
┌─────────────────────────────────────┐
│  1. 查 MemTable                     │
│  2. 查 Level 0 SSTable              │
│  3. 查 Level 1 SSTable              │
│  4. ...                             │
│  读放大：可能查询多个文件               │
└─────────────────────────────────────┘

B+Tree（MongoDB）：
写入流程：
┌─────────────────────────────────────┐
│  1. Journal（顺序写）                │
│  2. 更新 B+Tree 节点（随机写）         │
│  3. 可能触发页分裂                    │
│  4. 返回成功             ← 5-10ms    │
└─────────────────────────────────────┘

读取流程：
┌─────────────────────────────────────┐
│  1. 从根节点开始                      │
│  2. 二分查找到叶子节点                 │
│  3. 返回数据                         │
│  读放大：O(log N)                    │
└─────────────────────────────────────┘
```

**存储引擎对比表：**

| 维度        | DynamoDB        | Cloud Datastore    | MongoDB            | DocumentDB         | Firestore Native  |
| --------- | --------------- | ------------------ | ------------------ | ------------------ | ----------------- |
| **存储结构**  | LSM-Tree        | Megastore（类B+Tree） | B+Tree（WiredTiger） | Aurora（类B+Tree）    | Megastore/Spanner |
| **写入性能**  | ✅ 极高（顺序写）   | ✅ 高                | ⚠️ 高（随机写）          | ✅ 高              | ✅ 高               |
| **读取性能**  | ✅ 高（需查多层）   | ✅ 高                | ✅ 高（O(log N)）      | ✅ 极高（缓存）         | ✅ 高               |
| **空间利用率** | ✅ 高（压缩）     | ✅ 高                | ⚠️ 中（页分裂碎片）        | ✅ 高（压缩）          | ✅ 高               |
| **写放大**   | ⚠️ 高（写优化架构） | ⚠️ 中               | ✅ 低                | ✅ 低              | ⚠️ 中              |
| **读放大**   | ⚠️ 中（多层存储）  | ✅ 低                | ✅ 低（单次查找）          | ✅ 低              | ✅ 低               |
| **后台合并**  | ⚠️ 推测需要     | 🔴 不需要             | 🔴 不需要             | 🔴 不需要           | 🔴 不需要            |
| **数据格式**  | Blob（黑盒）    | Blob（黑盒）           | 文档格式（透明）           | 文档格式（透明）         | 文档格式（透明）          |
| **适用场景**  | 写密集型        | 事务型                | 读写平衡               | 读密集型             | 实时应用              |

> **注释**：
> - 标注 * 的 DynamoDB 特性基于性能表现推断，AWS 未公开具体实现细节
> - **数据格式说明**：
>   - `Blob（黑盒）`：键值数据库将 Value 视为不透明的二进制数据，无法解析内部字段，只能通过主键查询
>   - `文档格式（透明）`：文档数据库可以解析所有字段（MongoDB/DocumentDB 使用 BSON，Firestore 使用内部格式），支持任意字段查询、嵌套字段和数组查询

---

##### 3.2 索引机制对比

**GSI vs 多键索引架构：**

```
DynamoDB GSI（全局二级索引）：
┌─────────────────┐         ┌─────────────────┐
│   主表 (Users)   │         │GSI (email-index)│
│ userId | email  │  异步    │ email | userId  │
│ ─────────────── │  ────→  │ ─────────────── │
│ u1     | a@x.com│  复制    │ a@x.com | u1    │
│ u2     | b@x.com│         │ b@x.com | u2    │
└─────────────────┘         └─────────────────┘

特点：
• 独立的表（有自己的分区）
• 最终一致性（异步复制）
• 最多 20 个 GSI
• 只能索引顶层字段

MongoDB 多键索引：
┌─────────────────────────────────────┐
│  Collection: users                  │
│  {                                  │
│    "_id": "u1",                     │
│    "email": "a@x.com",              │
│    "address": {                     │
│      "city": "Beijing"  ← 可索引     │
│    },                               │
│    "tags": ["A", "B"]   ← 可索引     │
│  }                                  │
└─────────────────────────────────────┘
         ↓
┌─────────────────────────────────────┐
│  索引：                              │
│  • email 索引（单字段）               │
│  • address.city 索引（嵌套字段）      │
│  • tags 索引（数组，多键索引）         │
│  • {email, age} 复合索引             │
└─────────────────────────────────────┘

特点：
• 无数量限制
• 强一致性
• 支持嵌套字段、数组
• 支持复合索引
```

**索引机制对比表：**

| 维度         | DynamoDB     | MongoDB | Firestore Native |
| ---------- | ------------ | ------- | ---------------- |
| **单字段索引**  | GSI（最多 20 个） | 无限制     | 自动创建             |
| **复合索引**   | GSI（最多 20 个） | 无限制     | 手动创建             |
| **嵌套字段索引** | 🔴 不支持       | ✅ 支持    | ✅ 支持             |
| **数组索引**   | 🔴 不支持       | ✅ 多键索引  | ✅ array-contains |
**索引能力对比表：**

| 维度 | DynamoDB | Cloud Datastore | MongoDB | DocumentDB | Firestore Native |
|------|----------|-----------------|---------|------------|------------------|
| **主键索引** | ✅ Partition Key + Sort Key | ✅ Key | ✅ _id | ✅ _id | ✅ Document ID |
| **二级索引** | ✅ GSI（最多20个） | 🔴 不支持 | ✅ 无限制 | ✅ 无限制 | ✅ 自动+手动 |
| **复合索引** | ✅ Sort Key | 🔴 不支持 | ✅ 支持 | ✅ 支持 | ✅ 支持 |
| **嵌套字段索引** | 🔴 仅顶层 | 🔴 不支持 | ✅ address.city | ✅ address.city | ✅ 支持 |
| **数组索引** | 🔴 不支持 | 🔴 不支持 | ✅ 多键索引 | ✅ 多键索引 | ✅ 数组包含 |
| **全文索引** | 🔴 不支持 | 🔴 不支持 | ✅ Text Index | ✅ Text Index | 🔴 不支持 |
| **地理索引** | 🔴 不支持 | 🔴 不支持 | ✅ 2dsphere | ✅ 2dsphere | ✅ GeoPoint |
| **索引结构** | B+Tree | B+Tree | B+Tree | B+Tree | B+Tree |
| **一致性** | 最终一致性（GSI） | 强一致性 | 强一致性 | 强一致性 | 强一致性 |
| **创建方式** | 手动 | N/A | 手动 | 手动 | 自动+手动 |
| **索引限制** | 最多20个GSI | 仅主键 | 无限制 | 无限制 | 200个复合索引 |

---

##### 3.3 查询引擎对比

**Key 查询 vs 文档查询流程：**

```
DynamoDB Key 查询：
hash(Partition Key) → Partition → B-Tree 查找 → 返回 Value

限制：
• 只能通过 Partition Key 查询
• Sort Key 支持范围查询
• 非主键字段需要 GSI 或全表扫描

MongoDB 文档查询：
解析查询条件 → 选择索引 → B-Tree 查找 → 返回文档

能力：
• 任意字段查询
• 嵌套字段查询（address.city）
• 数组查询（tags: "A"）
• 聚合管道（$match, $group, $sort）
```

**查询能力对比表：**

| 维度 | DynamoDB | Cloud Datastore | MongoDB | DocumentDB | Firestore Native |
|------|----------|-----------------|---------|------------|------------------|
| **查询模型** | Key查询 | Key查询 | 任意字段查询 | 任意字段查询 | 任意字段查询 |
| **查询语言** | PartiQL（有限） | GQL | MQL | MQL（兼容） | Firebase Query |
| **嵌套查询** | 🔴 不支持 | 🔴 不支持 | ✅ address.city | ✅ address.city | ✅ 支持 |
| **数组查询** | 🔴 不支持 | 🔴 不支持 | ✅ tags: "A" | ✅ tags: "A" | ✅ array-contains |
| **OR 查询** | 🔴 不支持 | 🔴 不支持 | ✅ $or | ✅ $or | ⚠️ 需多次查询 |
| **聚合查询** | 🔴 不支持 | 🔴 不支持 | ✅ 完整（Pipeline） | ✅ 完整（Pipeline） | 🔴 不支持 |
| **JOIN** | 🔴 不支持 | 🔴 不支持 | ✅ $lookup | ✅ $lookup | 🔴 不支持 |
| **全文搜索** | 🔴 不支持 | 🔴 不支持 | ✅ Text Index | ✅ Text Index | 🔴 不支持 |
| **实时订阅** | ⚠️ Streams | 🔴 不支持 | ✅ Change Streams | 🔴 不支持 | ✅ 原生支持 |
| **查询优化器** | ✅ 简单 | ✅ 简单 | ✅ 复杂 | ✅ 复杂 | ✅ 自动 |
| **查询延迟** | 1-3ms | 5-10ms | 5-10ms | 5-10ms | 10-20ms |
| **扫描能力** | ✅ Scan | ✅ Query | ✅ find() | ✅ find() | ✅ get() |
| **复杂查询支持** | 🔴 不支持 | 🔴 不支持 | ✅ 完整支持 | ✅ 完整支持 | 🔴 基本不支持 |

---

##### 3.4 事务机制对比

| 维度 | DynamoDB | Cloud Datastore | MongoDB | DocumentDB | Firestore Native |
|------|----------|-----------------|---------|------------|------------------|
| **事务类型** | 单项ACID | Entity Group事务 | 多文档ACID | 多文档ACID | 多文档ACID |
| **事务范围** | 单个Item | Entity Group内 | 跨Collection | 跨Collection | 跨Collection |
| **ACID支持** | ✅ 单项 | ✅ Entity Group内 | ✅ 完整 | ✅ 完整 | ✅ 完整 |
| **跨分片事务** | 🔴 不支持 | 🔴 不支持 | ✅ 2PC | ✅ 2PC | ✅ Paxos |
| **事务延迟** | < 5ms | 10-50ms | 10-50ms | 10-50ms | 20-100ms |
| **隔离级别** | Read Committed | Serializable | Snapshot Isolation | Snapshot Isolation | Serializable |
| **事务超时** | 无限制 | 30秒 | 60秒 | 60秒 | 270秒 |

---

##### 3.5 复制与分片对比

| 维度       | DynamoDB        | Cloud Datastore   | MongoDB           | DocumentDB        | Firestore Native |
| -------- | --------------- | ----------------- | ----------------- | ----------------- | ---------------- |
| **复制协议** | 3副本（AWS内部）      | Paxos（Entity Group） | Replica Set（Raft） | 6副本（Aurora）       | Paxos（跨Region）   |
| **副本数量** | 3（固定）           | 3（Entity Group）     | 3-7（可配置）          | 6（固定）       | 5（跨Region）       |
| **一致性**  | 最终一致（默认）        | 强一致（EG内）            | 强一致（Primary）      | 强一致         | 强一致（Paxos）       |
| **故障转移** | 自动（秒级）          | 自动（秒级）              | 自动（秒级）            | 自动（<30秒）    | 自动（秒级）           |
| **跨区域**  | ✅ Global Tables | ⚠️ 手动配置             | ⚠️ 手动配置           | ✅ 原生支持      | ✅ 原生支持           |
| **分片策略** | Partition Key哈希 | Entity Group        | Shard Key范围/哈希    | 自动分片        | 自动分片             |
| **分片数量** | 自动扩展            | 手动管理                | 手动管理              | 自动扩展        | 自动扩展             |
| **读写分离** | 🔴 不支持          | 🔴 不支持              | ✅ Secondary读      | ✅ Replica读  | 🔴 不支持           |

---

#### 4. 性能与适用场景对比

##### 4.1 性能指标对比表

| 操作             | DynamoDB  | Cloud Datastore | MongoDB  | DocumentDB | Firestore Native |
| -------------- | --------- | --------------- | -------- | ---------- | ---------------- |
| **Get by Key** | 1-3ms     | 5-10ms          | 5-10ms   | 3-5ms      | 10-20ms          |
| **复杂查询**       | 🔴 不支持    | 🔴 不支持   | 50-500ms   | 50-500ms         | 100-1000ms |
| **聚合查询**       | 🔴 不支持    | 🔴 不支持   | 100-1000ms | 100-1000ms       | 🔴 不支持     |
| **写入延迟**       | 1-5ms     | 10-20ms  | 5-10ms     | 5-10ms           | 10-20ms    |
| **吞吐量**        | 100万+ TPS | 10万+ TPS | 10万+ TPS   | 10万+ TPS         | 1万+ TPS    |
| **扩展性**        | ✅ 无限      | ✅ 高      | ⚠️ 手动分片    | ✅ 自动             | ✅ 自动       |
| **存储上限**       | 无限        | 无限       | 无限         | 128TB            | 无限         |
| **单文档大小**      | 400KB     | 1MB      | 16MB       | 16MB             | 1MB        |

**延迟差异说明：**

> **为什么 Datastore 延迟比 DynamoDB 高（5-10ms vs 1-3ms）？**
> 
> 虽然两者都有优秀的网络层（DynamoDB 用 API Gateway，Datastore 用 GFE），但**存储层架构差异**导致延迟不同：
> 
> **DynamoDB（1-3ms）：性能优先**
> - 异步复制：写入 Primary 后立即返回，不等待副本确认
> - 类 LSM-Tree：内存缓冲（MemTable）+ B+Tree 索引，读取极快
> - 最终一致性：默认模式，牺牲一致性换取性能
> - 延迟分解：网络 < 1ms + 存储 1-2ms = 1-3ms
> 
> **Datastore（5-10ms）：一致性优先**
> - Paxos 共识：需要多数派确认（2/3），多轮消息交换
> - Entity Group 事务：所有操作都需要检查锁和事务日志
> - 强一致性：保证读取到最新数据，需要额外开销
> - 延迟分解：网络 < 1ms + Paxos 2-3ms + Megastore 2-3ms + Colossus 1-2ms = 5-10ms
> 
> **权衡：**
> - DynamoDB：极致性能，适合高吞吐场景（购物车、IoT），但默认最终一致性
> - Datastore：强一致性 + 事务保证，适合需要 ACID 的场景，但延迟更高
> 
> 这是 CAP 定理的经典权衡：**性能 vs 一致性**，无法同时兼得。

---

##### 4.2 使用场景对比表

| 场景          | DynamoDB    | Cloud Datastore | MongoDB  | DocumentDB       | Firestore Native |
| ----------- | ----------- | --------------- | -------- | ---------------- | ---------------- |
| **电商购物车**   | ✅ 极致性能      | ✅ 事务保证          | ✅ 灵活查询   | ✅ 高性能            | ⚠️ 成本较高          |
| **用户画像**    | ⚠️ 查询受限     | ⚠️ 查询受限  | ✅ 灵活扩展           | ✅ 灵活扩展           | ⚠️ 查询限制 |
| **内容管理**    | 🔴 不支持复杂查询  | ⚠️ 查询受限  | ✅ 最佳选择           | ✅ 最佳选择           | ⚠️ 查询限制 |
| **IoT数据采集** | ✅ 高吞吐写入     | ✅ 事务保证   | ✅ 灵活存储           | ✅ 高性能            | ⚠️ 成本较高 |
| **实时聊天**    | ⚠️ 需Streams | 🔴 无实时订阅 | ✅ Change Streams | 🔴 无实时订阅         | ✅ 原生支持  |
| **游戏排行榜**   | ✅ 低延迟       | ✅ 事务     | ✅ 灵活排序           | ✅ 高性能            | ✅ 实时更新  |
| **日志存储**    | ✅ 高吞吐       | ✅ 批量写入   | ✅ TTL索引          | ✅ 高吞吐            | ⚠️ 成本高  |
| **移动应用**    | ⚠️ 需SDK     | ⚠️ 需SDK  | ✅ 灵活             | ✅ 灵活             | ✅ 最佳选择  |

**选择建议：**

```
✅ 选择 DynamoDB：
• 已知访问模式（通过Key查询）
• 需要极致性能（1-3ms）
• 需要无限扩展
• 高吞吐写入（IoT、游戏）
❌ 不适合：复杂查询、未知访问模式

✅ 选择 MongoDB：
• 需要复杂查询（聚合、全文搜索）
• 未知访问模式（灵活查询）
• 需要强大的查询能力
• 灵活Schema
❌ 不适合：极致性能要求

✅ 选择 DocumentDB：
• 需要MongoDB兼容性
• AWS生态集成
• 需要Aurora级别性能
• 计算存储分离
❌ 不适合：需要最新MongoDB特性

✅ 选择 Firestore Native：
• 移动/Web应用
• 需要实时订阅
• 需要离线支持
• 快速开发（BaaS）
❌ 不适合：复杂聚合、成本敏感
```

---

#### 5. 核心机制精简示例

##### 5.1 DynamoDB核心机制

**存储架构说明：**

> **注意**：AWS 官方文档未公开 DynamoDB 内部存储引擎的具体实现细节。以下描述基于性能特征推断，采用类 LSM-Tree 架构：
> 
> - **内存缓冲层**：接收写入请求，提供 1-3ms 低延迟（类似 LSM-Tree 的 MemTable）
> - **B+Tree 索引**：用于快速查找主键和 GSI
> - **磁盘持久层**：分层存储，后台合并压缩（类似 LSM-Tree 的 SSTable）
> 
> 这种架构特点：
> - 写入优化：顺序写入，延迟极低
> - 读取优化：B+Tree 索引 + 内存缓冲
> - 存储高效：压缩存储，自动清理过期数据

**一致性哈希分区：**
```python
# Partition Key → 哈希 → 存储节点
partition = hash(partition_key) % num_partitions
node = partition_map[partition]
```

**GSI查询：**
```python
# 主表查询
response = table.get_item(Key={'userId': 'u123'})

# GSI查询（email索引）
response = table.query(
    IndexName='email-index',
    KeyConditionExpression=Key('email').eq('user@example.com')
)
```

##### 5.2 MongoDB 代表性功能

> MongoDB 作为通用文档数据库，提供完整的查询和实时功能，以下是两个代表性特性：

**聚合管道（Aggregation Pipeline）- 复杂数据处理：**
```javascript
// 统计每个用户的订单总额并排序
db.orders.aggregate([
  { $match: { status: "paid" } },                    // 过滤
  { $group: { _id: "$userId", total: { $sum: "$amount" } } },  // 聚合
  { $sort: { total: -1 } }                           // 排序
])

特点：
✅ 支持复杂的数据转换和聚合
✅ 多阶段管道处理（$match, $group, $lookup, $unwind 等）
✅ 适合数据分析和报表生成
```

**Change Streams - 实时变更监听：**
```javascript
// 监听集合的实时变更
const changeStream = collection.watch();
changeStream.on('change', (change) => {
  console.log(change); // 实时监听变更
});

特点：
✅ 实时监听数据变更（insert, update, delete）
✅ 支持过滤和转换
✅ 适合实时同步和事件驱动架构
```

##### 5.3 DocumentDB 核心特性

> DocumentDB 是 AWS 的 MongoDB 兼容服务，核心特性是 MongoDB 兼容层 + Aurora 存储。

**MongoDB 兼容层 - 无缝迁移：**
```javascript
// MongoDB 查询语法完全兼容
db.users.find({ age: { $gt: 25 } })

// 内部转换为 Aurora 存储的查询
// 用户无需修改代码
```

**Aurora 存储 - 高性能高可用：**
```
写入流程：
Primary → 6 副本（跨 3 AZ）
确认：4/6 副本确认即返回

读取流程：
任意副本读取（低延迟）
自动故障转移（< 30 秒）

优势：
✅ 6 副本高可用
✅ 计算存储分离
✅ 自动扩展（最大 128 TB）
```

##### 5.4 Firestore Native 核心特性

> Firestore Native 是面向移动/Web 的实时数据库，核心特性是实时监听和离线支持。

**实时监听（Real-time Listener）- 自动推送：**
```javascript
// 监听集合变化，自动推送到客户端
db.collection('messages').onSnapshot((snapshot) => {
  snapshot.docChanges().forEach((change) => {
    if (change.type === 'added') {
      console.log('New:', change.doc.data());
    }
  });
});

特点：
✅ WebSocket 长连接，自动推送变化
✅ 增量更新（只推送变化的文档）
✅ 自动重连（网络断开恢复）
✅ 适合聊天、协作、实时通知
```

**离线支持：**
```javascript
// 自动缓存，离线可用
db.enablePersistence();
```

---

### 文档数据库

#### MongoDB

##### MongoDB 单节点详细架构

> **说明**：前面 2.2.1 节展示了 MongoDB 的**分片集群架构**（Mongos → Shard → Replica Set），本节展示**单个 mongod 节点的内部详细架构**，重点是查询处理、索引管理和存储引擎层。

```
┌─────────────────────────────────────────────────────────────────┐
│                         客户端层                                 │
│  MongoDB Driver (Java/Python/Node.js) │ mongosh │ Compass       │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      mongod 服务层（单节点内部）                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              查询处理层                                      │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                  │ │
│  │  │ Parser   │→ │ Optimizer│→ │ Executor │                  │ │
│  │  │ (解析器)  │  │ (优化器)  │  │ (执行器)  │                  │ │
│  │  └──────────┘  └──────────┘  └──────────┘                  │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              索引管理层                                      │ │
│  │  ├─ B+Tree 索引（任意字段）                                   │ │
│  │  ├─ 全文索引（Text Index）                                   │ │
│  │  ├─ 地理位置索引（2dsphere）                                  │ │
│  │  ├─ 哈希索引（Hashed Index）                                 │ │
│  │  └─ 索引选择器（Index Selector）                             │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │              存储引擎层（WiredTiger）                        │ │
│  │  ├─ BSON 解析器（解析文档结构）                               │ │
│  │  ├─ Cache（缓存热点数据）                                    │ │
│  │  ├─ B+Tree 数据存储（默认，存储完整文档）                       │ │
│  │  ├─ LSM-Tree 数据存储（可选，写优化场景）                      │ │
│  │  ├─ 压缩（Snappy/zlib/zstd）                                │ │
│  │  └─ MVCC（多版本并发控制）                                    │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      持久化层                                    │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Journal（WAL）                                             │ │
│  │  - 操作日志                                                  │ │
│  │  - 崩溃恢复                                                  │ │
│  │  - Group Commit（批量提交）                                  │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  数据文件（BSON 格式）                                        │ │
│  │  - Collection 数据                                          │ │
│  │  - 索引数据                                                  │ │
│  │  - Checkpoint（定期快照）                                    │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  副本集（Replica Set）                                       │ │
│  │  - Primary（主节点）                                         │ │
│  │  - Secondary（从节点）                                       │ │
│  │  - Oplog（操作日志复制）                                      │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**两个架构图的区别：**

| 维度 | 2.2.1 分片集群架构 | 本节单节点详细架构 |
|------|------------------|------------------|
| **视角** | 分布式集群视角 | 单节点内部视角 |
| **重点** | Mongos 路由 + Shard 分片 + Replica Set | mongod 内部：查询处理 + 索引 + 存储引擎 |
| **详细程度** | 简略（4 层） | 详细（5 层，展开 mongod 内部） |
| **适合理解** | 分片集群如何工作 | 单个查询如何执行 |
| **索引层** | 未展开 | 详细展开（5 种索引类型） |
| **存储引擎** | 简单（WiredTiger B+Tree） | 详细（B+Tree/LSM-Tree + MVCC + 压缩） |

---

###### MongoDB 存储引擎（WiredTiger）

**核心概念：索引 vs 存储（必读）**

在深入 MongoDB 存储引擎之前，必须理解**索引结构**和**数据存储结构**的区别，两者都可能使用 B+Tree，但作用完全不同： 
 **1. 索引结构（Index Structure）- 用于查找**

 作用：快速定位数据在磁盘上的位置
 存储内容：Key → 位置的映射关系
 位置：索引管理层
 ```
例如：age 字段的 B+Tree 索引
 ┌─────────────────────┐
 │  age: 25 → 位置 A    │
 │  age: 30 → 位置 B    │
 │  age: 35 → 位置 C    │
 └─────────────────────┘
```
特点：
- 只存储 Key 和位置指针
- 不存储完整文档
- 查询速度：O(log N)

 **2. 数据存储结构（Storage Structure）- 用于保存数据**

 作用：存储完整的 BSON 文档
 存储内容：完整的文档数据
 位置：存储引擎层（WiredTiger）
  ```
 例如：B+Tree 数据存储
 ┌─────────────────────────────────────┐
 │  位置 A → { _id: 1, age: 25, ... }  │
 │  位置 B → { _id: 2, age: 30, ... }  │
 │  位置 C → { _id: 3, age: 35, ... }  │
 └─────────────────────────────────────┘
 ``` 
 特点：
 - 存储完整的 BSON 文档
 - 按 _id 有序存储
 - 支持范围扫描
 
 **3. 查询流程（两者协同工作）**

> 查询：db.users.find({ age: 25 })
> 
> 步骤 1：索引查找
> ├─ 使用 age 字段的 B+Tree 索引
> ├─ 找到 age=25 的所有位置：[位置 A, 位置 D, 位置 F]
> └─ 速度：O(log N)
> 
> 步骤 2：数据读取
> ├─ 根据位置，从 B+Tree 数据存储中读取完整文档
> ├─ 位置 A → { _id: 1, name: "Alice", age: 25 }
> ├─ 位置 D → { _id: 4, name: "David", age: 25 }
> └─ 位置 F → { _id: 6, name: "Frank", age: 25 }
> 
> 总延迟：O(log N) + O(M)，M = 结果数量


**4. WiredTiger 的两种数据存储模式**

MongoDB 的 WiredTiger 存储引擎支持两种数据存储结构：
 
 B+Tree 数据存储（默认）：
 - 读写平衡
 - 有序存储（按 _id）
 - 适合大多数场景
 
 LSM-Tree 数据存储（可选）：
 - 写入优化（顺序写）
 - 适合日志、时序数据
 - 读取稍慢（需要查多层）
 
 注意：这里说的是数据存储结构，不是索引结构！

 **关键要点：**
 - ✅ **索引**：B+Tree 索引，用于快速查找位置
 - ✅ **存储**：B+Tree/LSM-Tree 数据存储，用于保存完整文档
 - ✅ 两者都可能用树结构，但**作用完全不同**
 - ✅ 查询时：先用索引找位置，再从存储读数据

---

###### WiredTiger 架构

**WiredTiger vs 其他存储引擎：**

WiredTiger 是 MongoDB 3.2+ 的默认存储引擎，相比其他主流存储引擎有显著优势：

| 存储引擎           | 数据库     | 数据结构                 | 锁粒度         | 压缩比  | 核心特点     |
| -------------- | ------- | -------------------- | ----------- | ---- | -------- |
| **WiredTiger** | MongoDB | B+Tree + LSM-Tree 可选 | 文档级锁 + MVCC | 2-7x | **灵活性强** |
| **InnoDB**     | MySQL   | B+Tree               | 行级锁 + MVCC  | 2-3x | 事务型      |
| **RocksDB**    | 多种      | LSM-Tree             | 无锁（MVCC）    | 2-5x | 写密集型     |
| **MMAPv1** | MongoDB（旧） | 内存映射 | 集合级锁 | 不支持 | 已废弃 |

**数据结构选择：**

| 场景            | 推荐          | 原因                       |
| ------------- | ----------- | ------------------------ |
| OLTP 通用场景     | B-Tree (默认) | 高频点查（单行查询）               |
| 写密集 + OLAP 扫描 | LSM-Tree    | 顺序写 + 大范围扫描（百万行级别）        |
| OLTP 读密集      | B-Tree      | 点查性能好（O(log N) 单次查找）      |
| 时序数据 + 分析     | LSM-Tree    | 顺序写入 + 时间范围扫描（如日志、监控指标） |

**WiredTiger 的核心优势：**

**1. 灵活的存储结构（独特优势）**
```javascript
// B+Tree 存储（默认）- 读写平衡
db.createCollection("users", {
  storageEngine: { wiredTiger: { configString: "type=file" }}
})

// LSM-Tree 存储（可选）- 写入优化
db.createCollection("logs", {
  storageEngine: { wiredTiger: { configString: "type=lsm" }}
})

对比：
- InnoDB：只有 B+Tree
- RocksDB：只有 LSM-Tree
- WiredTiger：两者都支持！
```

**2. 文档级锁 + MVCC（高并发）**
- 锁粒度：文档级（比 InnoDB 行级更细）
- MVCC：读写不阻塞，快照隔离
- 并发性能：比 MMAPv1 提升 10x

**3. 高压缩比（低成本）**
- Snappy（默认）：2-3x，快速
- zlib：5-7x，高压缩比
- zstd：3-5x，平衡
- 存储成本：比 InnoDB 降低 50-80%

**4. Checkpoint 机制（快速恢复）**
- 每 60 秒创建一致性快照
- 崩溃恢复：秒级（vs InnoDB 分钟级）
- 无碎片化：自动压缩，空间利用率 90-95%

**5. 可配置的缓存**
```yaml
storage:
  wiredTiger:
    engineConfig:
      cacheSizeGB: 8  # 默认 50% 内存 - 1GB
```

**为什么 MongoDB 选择 WiredTiger：**
- MongoDB 3.2 之前：MMAPv1（集合级锁，性能差）
- MongoDB 3.2+：WiredTiger（默认）
- 性能提升：5-10x
- 存储成本：降低 50-80%
- 并发能力：提升 10x

```
WiredTiger 存储引擎：
┌─────────────────────────────────────────────────────────────────┐
│                      内存层（Cache）                              │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Cache（默认 50% 系统内存 - 1GB）                             │ │
│  │  ├─ 热点数据页（Page）                                       │ │
│  │  ├─ 索引页                                                  │ │
│  │  ├─ LRU 淘汰策略                                            │ │
│  │  └─ Eviction（驱逐）机制                                     │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      B-Tree 层                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  B-Tree 结构（默认）                                         │ │
│  │                                                            │ │
│  │              [Root Page]                                   │ │
│  │             /            \                                 │ │
│  │    [Internal Page]  [Internal Page]                        │ │
│  │    /      |      \      /      |      \                    │ │
│  │ [Leaf] [Leaf] [Leaf] [Leaf] [Leaf] [Leaf]                  │ │
│  │                                                            │ │
│  │  特点：                                                     │ │
│  │  - 有序存储（按 _id 或索引键排序）                             │ │
│  │  - 支持范围查询                                              │ │
│  │  - 读写平衡                                                 │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      压缩层                                      │
│  ├─ Snappy（默认，快速压缩）                                       │
│  ├─ zlib（高压缩比）                                              │
│  ├─ zstd（平衡压缩比和速度）                                       │
│  └─ 压缩比：2-10x                                                │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      磁盘层                                      │
│  ├─ 数据文件（.wt）                                               │
│  ├─ 索引文件（.wt）                                               │
│  └─ Journal 文件（.wt）                                          │
└─────────────────────────────────────────────────────────────────┘
```

###### BSON 格式

BSON（Binary JSON）是 MongoDB 的数据存储格式，相比 JSON 有以下优势：

```
JSON vs BSON：

JSON（文本格式）：
{
  "name": "Alice",
  "age": 30,
  "address": {
    "city": "Beijing"
  }
}
大小：~60 字节（文本）
解析：需要完整解析

BSON（二进制格式）：
┌─────────────────────────────────────────┐
│ 文档长度: 4 字节 (60)                     │
├─────────────────────────────────────────┤
│ 字段: name                               │
│   类型: 0x02 (String)                    │
│   长度: 4 字节 (5)                       │
│   值: "Alice\0"                         │
├─────────────────────────────────────────┤
│ 字段: age                                │
│   类型: 0x10 (Int32)                     │
│   值: 30 (4 字节二进制)                   │
├─────────────────────────────────────────┤
│ 字段: address                            │
│   类型: 0x03 (Document)                  │
│   嵌套文档:                               │
│     ├─ 字段: city                        │
│     │   类型: 0x02 (String)              │
│     │   值: "Beijing\0"                 │
└─────────────────────────────────────────┘
大小：~55 字节（二进制）
解析：可跳过不需要的字段
```

BSON 优势：
✅ 类型丰富（Date、ObjectId、Binary、Decimal128）
✅ 遍历速度快（记录长度，可跳过）
✅ 修改效率高（原地更新）
✅ 支持嵌套文档和数组


---

###### MongoDB 主键：_id 与 ObjectId

**核心概念：MongoDB 的主键不是业务字段**

```
MongoDB vs 关系型数据库主键对比：

MySQL（关系型）：
CREATE TABLE orders (
  order_id VARCHAR(50) PRIMARY KEY,  ← 业务字段作为主键
  user_id VARCHAR(50),
  amount DECIMAL(10,2)
);

MongoDB（文档型）：
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),  ← 系统主键（自动生成）
  "order_id": "order_67890",                    ← 业务字段（普通字段）
  "user_id": "user_12345",                      ← 业务字段（普通字段）
  "amount": 99.99
}
```

关键区别：
✅ MongoDB 主键固定为 _id (不可更改字段名)
✅ _id 自动生成 ObjectId（分布式友好）
🔴 业务字段（order_id、user_id）不是主键，需手动建索引


**ObjectId 结构：分布式唯一ID**

```
ObjectId("507f1f77bcf86cd799439011")
         └─────┬─────┘└┬┘└┬┘└──┬──┘
               │       │  │    │
               │       │  │    └─ 3字节：计数器（Counter）
               │       │  └────── 2字节：进程ID（Process ID）
               │       └───────── 2字节：机器ID（Machine ID）
               └───────────────── 4字节：时间戳（Timestamp，秒级）

总长度：12 字节 = 24 个十六进制字符

生成过程：
┌─────────────────────────────────────────┐
│ 1. 时间戳（4字节）                        │
│    当前时间（秒）：0x507f1f77              │
│    → 可从 ObjectId 提取创建时间           │
├─────────────────────────────────────────┤
│ 2. 机器ID（2字节）                        │
│    机器标识：0xbcf8                       │
│    → 不同机器生成不同ID                    │
├─────────────────────────────────────────┤
│ 3. 进程ID（2字节）                        │
│    进程标识：0x6cd7                       │
│    → 同一机器不同进程生成不同ID             │
├─────────────────────────────────────────┤
│ 4. 计数器（3字节）                        │
│    递增序列：0x99439011                   │
│    → 同一秒内生成多个ID时递增               │
└─────────────────────────────────────────┘

特性：
✅ 全局唯一（时间+机器+进程+计数器）
✅ 无需中心协调（每个节点独立生成）
✅ 趋势递增（时间戳在前）
✅ 包含时间信息（可提取创建时间）
✅ 高性能（10万+/秒生成速度）
```

**为什么不用自增ID？**

```
自增ID在分布式环境的问题：

问题1：需要中心化协调
┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Shard 1    │  │  Shard 2    │  │  Shard 3    │
│  INSERT     │  │  INSERT     │  │  INSERT     │
└──────┬──────┘  └──────┬──────┘  └──────┬──────┘
       │                │                │
       └────────────────┼────────────────┘
                        ↓
              ┌─────────────────┐
              │ ID生成器（中心节点）│
              │ 分配ID：1,2,3... │
              └─────────────────┘
❌ 中心节点成为性能瓶颈
❌ 单点故障风险
❌ 网络延迟开销
❌ 吞吐量：1000-5000/秒

问题2：写入热点
Shard 1: ID [1 ~ 100000]      ← 旧数据，不再写入
Shard 2: ID [100001 ~ 200000] ← 旧数据，不再写入
Shard 3: ID [200001 ~ 300000] ← 所有新数据写这里！热点！

❌ 所有新数据集中在最后一个分片
❌ 其他分片闲置
❌ 无法利用分布式优势
```

**ObjectId vs 自增ID 对比**

| 维度       | ObjectId（MongoDB默认） | 自增ID（传统方式）     |
| :------- | :------------------ | :------------- |
| **生成方式** | 每个节点独立生成            | 需要中心协调         |
| **性能**   | 10万+插入/秒            | 1000-5000插入/秒  |
| **单点故障** | 无                   | 有（中心节点）        |
| **网络开销** | 无                   | 每次插入需请求中心节点    |
| **分片友好** | ✅ 是（哈希分布）           | 🔴 否（热点问题）     |
| **连续性**  | 趋势递增（非严格连续）         | 严格连续（1,2,3...） |
| **时间信息** | ✅ 包含（可提取）           | 🔴 无           |
| **长度**   | 12字节                | 4-8字节          |

**业务字段索引的必要性**

```javascript
// MongoDB 文档结构
{
  "_id": ObjectId("..."),           // ← 主键，自动有索引
  "user_id": "user_12345",          // ← 业务字段，无索引！
  "order_id": "order_67890",        // ← 业务字段，无索引！
  "email": "user@example.com",      // ← 业务字段，无索引！
  "created_at": ISODate("2024-01-01")
}

// 查询1：按 _id 查询（快）
db.orders.find({ _id: ObjectId("...") });
// ✅ 使用主键索引，0.01秒

// 查询2：按 user_id 查询（慢！）
db.orders.find({ user_id: "user_12345" });
// ❌ 全表扫描，30秒（1亿条记录）

// 查询3：按 order_id 查询（慢！）
db.orders.find({ order_id: "order_67890" });
// ❌ 全表扫描，30秒

// ✅ 正确做法：为业务字段建索引
db.orders.createIndex({ user_id: 1 });
db.orders.createIndex({ order_id: 1 });
db.orders.createIndex({ email: 1 });

// 现在查询很快
db.orders.find({ user_id: "user_12345" });
// ✅ 使用索引，10-50ms
```

**高基数字段为什么需要索引？**

```
高基数 = 字段值的唯一性高

示例：100万条用户记录

低基数字段 - 性别（gender）：
- 只有2种值：男、女
- 男：50万条
- 女：50万条
→ 基数 = 2

高基数字段 - 用户ID（user_id）：
- 有100万种不同的值
- user_001：1条
- user_002：1条
- ...
- user_1000000：1条
→ 基数 = 100万

查询性能对比：

查询：db.orders.find({ user_id: "user_12345" })

无索引：
┌─────────────────────────────────────┐
│ 全表扫描 100万条记录                   │
│ ├─ user_001 ✗                       │
│ ├─ user_002 ✗                       │
│ ... 扫描 99.9999% 的无关数据          │
│ └─ user_12345 ✓ 找到！               │
└─────────────────────────────────────┘
耗时：30秒

有索引：
┌─────────────────────────────────────┐
│ 索引直接定位                          │
│ user_12345 → 第12345行               │
│ 只读取这1条记录                       │
└─────────────────────────────────────┘
耗时：0.01秒

性能差距：3000倍！

结论：
✅ 高基数字段：索引效果显著（3000倍提升）
⚠️ 低基数字段：索引效果有限（1.25倍提升）
```

**分布式ID生成方案对比**

```javascript
// 方案1：ObjectId（MongoDB默认，推荐）
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "user_id": "user_12345"
}
// ✅ 无需配置，开箱即用
// ✅ 性能最高（10万+/秒）
// ✅ 分布式友好

// 方案2：Snowflake（需要额外依赖）
const Snowflake = require('snowflake-id');
const snowflake = new Snowflake({ mid: 1 });
const id = snowflake.generate();
db.orders.insertOne({ _id: id, ... });
// ✅ 趋势递增（更友好）
// ✅ 高性能（5万+/秒）
// ⚠️ 需要配置机器ID

// 方案3：计数器集合（不推荐）
db.counters.findAndModify({
  query: { _id: "order_id" },
  update: { $inc: { seq: 1 } },
  new: true
});
// ❌ 性能瓶颈（1000-5000/秒）
// ❌ 中心化协调
// ❌ 分片环境下问题多

// 方案4：分段自增（可选）
// Shard 1：生成 1,4,7,10...（步长=3）
// Shard 2：生成 2,5,8,11...（步长=3）
// Shard 3：生成 3,6,9,12...（步长=3）
// ✅ 无需中心协调
// ⚠️ 不是严格连续
// ⚠️ 扩容时需重新配置
```

**最佳实践**

```javascript
// 1. 使用默认的 ObjectId（推荐）
db.orders.insertOne({
  // _id 自动生成，无需指定
  order_id: "order_67890",  // 业务ID
  user_id: "user_12345",
  amount: 99.99
});

// 2. 为业务字段建索引（必须）
db.orders.createIndex({ order_id: 1 });
db.orders.createIndex({ user_id: 1 });

// 3. 复合索引（支持多字段查询）
db.orders.createIndex({ user_id: 1, created_at: 1 });

// 4. 如果需要自定义 _id（可选）
db.orders.insertOne({
  _id: "custom_id_12345",  // 自定义主键
  order_id: "order_67890",
  user_id: "user_12345"
});
// ⚠️ 需要自己保证全局唯一性
// ⚠️ 需要考虑分布式冲突问题
```

---

###### 写入流程

```
MongoDB 写入流程：
┌─────────────────────────────────────────┐
│  1. 客户端发起写入请求                     │
│     db.users.insertOne({...})           │
└─────────────────────────────────────────┘
                ↓
┌─────────────────────────────────────────┐
│  2. 写入 Journal（WAL）                   │
│     - 顺序写入磁盘                        │
│     - Group Commit（批量提交，100ms）     │
│     - 保证持久性                          │
└─────────────────────────────────────────┘
                ↓
┌─────────────────────────────────────────┐
│  3. 写入 WiredTiger Cache                │
│     - 更新 B-Tree                        │
│     - 标记为脏页（Dirty Page）            │
└─────────────────────────────────────────┘
                ↓
┌─────────────────────────────────────────┐
│  4. 返回成功响应（writeConcern: 1）        │
│     - 默认：写入 Primary 即返回            │
│     - majority：写入多数派才返回           │
└─────────────────────────────────────────┘
                ↓
┌─────────────────────────────────────────┐
│  5. 后台 Checkpoint（60 秒一次）          │
│     - 将脏页刷写到磁盘                    │
│     - 创建一致性快照                      │
│     - 清理 Journal                       │
└─────────────────────────────────────────┘
                ↓
┌─────────────────────────────────────────┐
│  6. 副本集复制（Oplog）                   │
│     - Primary 写入 Oplog                 │
│     - Secondary 拉取 Oplog               │
│     - Secondary 重放操作                 │
└─────────────────────────────────────────┘

崩溃恢复：
1. 读取最后一个 Checkpoint
2. 重放 Journal 日志（Checkpoint 之后的操作）
3. 恢复到崩溃前的状态
```

###### MVCC（多版本并发控制）

**核心机制：与传统 SQL 数据库相同**

MongoDB 的 MVCC 与 PostgreSQL/MySQL InnoDB 的核心思想相同：多版本并发控制，读写不阻塞，快照隔离。

```
MVCC 实现：
┌─────────────────────────────────────────┐
│  文档的多个版本：                          │
│                                          │
│  _id: 1                                  │
│  ├─ Version 1 (ts=100): {name: "Alice"}  │
│  ├─ Version 2 (ts=200): {name: "Bob"}    │
│  └─ Version 3 (ts=300): {name: "Charlie"}│
└─────────────────────────────────────────┘

事务 A（ts=150）：
- 读取 Version 1（ts=100）
- 看不到 Version 2 和 3

事务 B（ts=250）：
- 读取 Version 2（ts=200）
- 看不到 Version 3

事务 C（ts=350）：
- 读取 Version 3（ts=300）
- 最新版本

优势：
✅ 读写不阻塞
✅ 快照隔离（Snapshot Isolation）
✅ 无锁读取

垃圾回收：
- 定期清理旧版本
- 保留活跃事务需要的版本
```

**MongoDB vs 传统 SQL 数据库 MVCC 对比：**

| 维度 | MongoDB (WiredTiger) | PostgreSQL | MySQL InnoDB |
|------|---------------------|-----------|--------------|
| **核心机制** | ✅ 多版本并发控制 | ✅ 多版本并发控制 | ✅ 多版本并发控制 |
| **版本存储** | Update Chain（链表） | 表中多行 | Undo Log |
| **版本标识** | 时间戳（Timestamp） | 事务 ID（xmin/xmax） | 事务 ID（DB_TRX_ID） |
| **最新版本** | 链表头部 | 表中最新行 | 表中当前行 |
| **历史版本** | 链表后续节点 | 表中旧行 | Undo Log 中 |
| **垃圾回收** | 后台线程自动清理 | VACUUM 进程 | Purge 线程 |
| **隔离级别** | Snapshot Isolation | 4 种（含 Serializable） | 4 种（含 Serializable） |

**实现细节差异：**

```
版本存储位置：

MongoDB（WiredTiger Update Chain）：
_id: 1 → [Version 3] → [Version 2] → [Version 1]
         (ts=300)      (ts=200)      (ts=100)
特点：最新版本在链表头部，优化读取

PostgreSQL（表中多行）：
tuple_1: {xmin=100, xmax=200, name="Alice"}
tuple_2: {xmin=200, xmax=300, name="Bob"}
tuple_3: {xmin=300, xmax=∞, name="Charlie"}
特点：版本作为独立行存储，需要扫描

MySQL InnoDB（Undo Log）：
当前行: {name="Charlie", DB_TRX_ID=300, DB_ROLL_PTR=→}
Undo Log: [Version 2] → [Version 1]
特点：当前版本在表中，历史版本在 Undo Log
```

**核心相同点：**
- ✅ 读写不阻塞（读不阻塞写，写不阻塞读）
- ✅ 快照隔离（事务看到一致的数据快照）
- ✅ 版本链管理（维护多个版本）
- ✅ 垃圾回收（清理旧版本）

**主要差异点：**
- 版本存储位置不同（链表 vs 表中 vs Undo Log）
- 版本标识方式不同（时间戳 vs 事务 ID）
- MongoDB 隔离级别较少（只有 Snapshot Isolation）

---

###### MongoDB 索引引擎

###### B-Tree 索引

MongoDB 使用 B-Tree 作为索引结构，支持任意字段索引。

```
B-Tree 索引结构：

                    [Root Node]
                   /           \
          [Internal Node]   [Internal Node]
          /       \              /       \
    [Leaf Node] [Leaf Node] [Leaf Node] [Leaf Node]
    ├─ age: 25 → Doc1
    ├─ age: 30 → Doc2
    └─ age: 35 → Doc3

特点：
- 有序存储（支持范围查询）
- 平衡树（查找 O(log N)）
- 叶子节点存储文档位置（指针）
```

**索引类型：**

**1. 单字段索引**
```javascript
// 创建索引
db.users.createIndex({ age: 1 });  // 1: 升序, -1: 降序

// 查询使用索引
db.users.find({ age: 30 });

// 索引结构
age: 25 → Doc1
age: 30 → Doc2
age: 35 → Doc3
```

**2. 复合索引**
```javascript
// 创建复合索引
db.users.createIndex({ age: 1, city: 1 });

// 查询使用索引
db.users.find({ age: 30, city: "Beijing" });

// 索引结构（多级排序）
age: 25, city: "Beijing" → Doc1
age: 25, city: "Shanghai" → Doc2
age: 30, city: "Beijing" → Doc3
age: 30, city: "Shanghai" → Doc4

索引前缀规则：
✅ { age: 30 } → 使用索引
✅ { age: 30, city: "Beijing" } → 使用索引
❌ { city: "Beijing" } → 不使用索引（缺少前缀）
```

**3. 嵌套字段索引**
```javascript
// 文档结构
{
  _id: 1,
  name: "Alice",
  address: {
    city: "Beijing",
    district: "Chaoyang"
  }
}

// 创建嵌套字段索引
db.users.createIndex({ "address.city": 1 });

// 查询使用索引
db.users.find({ "address.city": "Beijing" });

// 索引结构
address.city: "Beijing" → Doc1
address.city: "Shanghai" → Doc2
```

**4. 数组索引（多键索引）**
```javascript
// 文档结构
{
  _id: 1,
  name: "Alice",
  tags: ["vip", "premium", "active"]
}

// 创建数组索引
db.users.createIndex({ tags: 1 });

// 索引结构（每个数组元素都创建索引）
tags: "active" → Doc1
tags: "premium" → Doc1
tags: "vip" → Doc1

// 查询使用索引
db.users.find({ tags: "vip" });  // 查找包含 "vip" 的文档
db.users.find({ tags: { $in: ["vip", "premium"] } });
```

**5. 全文索引**
```javascript
// 创建全文索引
db.articles.createIndex({ 
    title: "text", 
    content: "text" 
});

// 全文搜索
db.articles.find({ 
    $text: { $search: "MongoDB database" } 
});

// 索引结构（倒排索引）
"MongoDB" → [Doc1, Doc3, Doc5]
"database" → [Doc1, Doc2, Doc4]
"NoSQL" → [Doc3, Doc5]

特点：
- 支持多语言（中文、英文等）
- 支持词干提取（running → run）
- 支持停用词过滤（the, a, an）
- 支持权重（title 权重 > content 权重）
```

**6. 地理位置索引**
```javascript
// 文档结构
{
  _id: 1,
  name: "Starbucks",
  location: {
    type: "Point",
    coordinates: [116.4074, 39.9042]  // [经度, 纬度]
  }
}

// 创建地理位置索引
db.places.createIndex({ location: "2dsphere" });

// 查询附近的地点（5km 内）
db.places.find({
    location: {
        $near: {
            $geometry: {
                type: "Point",
                coordinates: [116.4, 39.9]
            },
            $maxDistance: 5000  // 米
        }
    }
});

// 索引结构（GeoHash）
- 将地球划分为网格
- 每个网格有唯一编码
- 支持快速范围查询
```

###### 索引选择器

```
查询优化器的索引选择过程：

查询：db.users.find({ age: 30, city: "Beijing" })

步骤 1：查找可用索引
├─ 索引 1: { age: 1 }
├─ 索引 2: { city: 1 }
├─ 索引 3: { age: 1, city: 1 }
└─ 索引 4: { city: 1, age: 1 }

步骤 2：评估每个索引的成本
├─ 索引 1: 扫描 300,000 行（age=30），过滤 city
├─ 索引 2: 扫描 200,000 行（city=Beijing），过滤 age
├─ 索引 3: 扫描 10,000 行（age=30 AND city=Beijing）✅
└─ 索引 4: 扫描 200,000 行（city=Beijing），过滤 age

步骤 3：选择最优索引
- 选择索引 3（复合索引）
- 扫描行数最少
- 无需额外过滤

步骤 4：执行查询
- 使用索引 3 查找
- 返回 10,000 个文档
- 查询时间：~10ms

查看执行计划：
db.users.find({ age: 30, city: "Beijing" }).explain("executionStats");

输出：
{
  "executionStats": {
    "executionTimeMillis": 10,
    "totalKeysExamined": 10000,
    "totalDocsExamined": 10000,
    "executionStages": {
      "stage": "FETCH",
      "inputStage": {
        "stage": "IXSCAN",  // 索引扫描
        "indexName": "age_1_city_1",
        "keysExamined": 10000
      }
    }
  }
}
```

---

##### MongoDB 持久化层

###### Journal（WAL）

```
Journal 机制：
┌─────────────────────────────────────────┐
│  Journal 文件（预写日志）                  │
│  ├─ journal.0000001                     │
│  ├─ journal.0000002                     │
│  └─ journal.0000003                     │
└─────────────────────────────────────────┘

写入流程：
1. 操作写入 Journal（顺序写）
2. 操作写入 Cache（内存）
3. 返回客户端成功
4. 后台 Checkpoint 刷盘

Journal 记录格式：
┌─────────────────────────────────────────┐
│  Sequence Number（序列号）                │
├─────────────────────────────────────────┤
│  Operation（操作类型）                    │
│  - insert / update / delete             │
├─────────────────────────────────────────┤
│  Namespace（数据库.集合）                 │
│  - mydb.users                           │
├─────────────────────────────────────────┤
│  Document（BSON 文档）                    │
├─────────────────────────────────────────┤
│  Timestamp（时间戳）                      │
└─────────────────────────────────────────┘

Group Commit（批量提交）：
- 默认：100ms 批量提交一次
- 减少磁盘 I/O
- 提高吞吐量

配置：
storage:
  journal:
    enabled: true
    commitIntervalMs: 100  // 提交间隔
```

###### Checkpoint

```
Checkpoint 机制：
┌─────────────────────────────────────────┐
│  Checkpoint（一致性快照）                 │
│  - 默认：60 秒一次                        │
│  - 将内存中的脏页刷写到磁盘                 │
│  - 创建一致性快照                         │
└─────────────────────────────────────────┘

Checkpoint 流程：
1. 标记当前时间点
2. 将所有脏页写入磁盘
3. 更新 Checkpoint 元数据
4. 清理旧的 Journal 文件

崩溃恢复：
1. 加载最后一个 Checkpoint
2. 重放 Journal（Checkpoint 之后的操作）
3. 恢复到崩溃前的状态

时间线：
T0: Checkpoint 1
T1: 写入操作 A
T2: 写入操作 B
T3: 崩溃
T4: 恢复
    - 加载 Checkpoint 1
    - 重放操作 A 和 B
    - 恢复完成
```

###### Oplog（操作日志）

```
Oplog 用于副本集复制：
┌─────────────────────────────────────────┐
│  Oplog（Capped Collection）              │
│  - 固定大小（默认 5% 磁盘空间）             │
│  - 循环覆盖（FIFO）                       │
│  - 幂等操作（可重复执行）                   │
└─────────────────────────────────────────┘

Oplog 记录格式：
{
  "ts": Timestamp(1704067200, 1),  // 时间戳
  "t": NumberLong(1),               // Term（选举轮次）
  "h": NumberLong("123456789"),     // 哈希值
  "v": 2,                           // Oplog 版本
  "op": "i",                        // 操作类型（i=insert, u=update, d=delete）
  "ns": "mydb.users",               // 命名空间
  "o": {                            // 操作内容
    "_id": ObjectId("..."),
    "name": "Alice",
    "age": 30
  }
}

复制流程：
┌─────────────┐
│  Primary    │  ← 写入操作
└──────┬──────┘
       ↓ 写入 Oplog
┌─────────────┐
│  Oplog      │
└──────┬──────┘
       ↓ 拉取 Oplog
┌─────────────┐
│  Secondary  │  ← 重放操作
└─────────────┘

特点：
- 异步复制（默认）
- 可配置同步复制（writeConcern: majority）
- 支持延迟复制（延迟从节点）
```

##### MongoDB 复制与分片

###### 副本集（Replica Set）

```
副本集架构：
┌─────────────────────────────────────────────────────────────────┐
│                      副本集（3 节点）                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │  Primary     │  │  Secondary   │  │  Secondary   │           │
│  │  (主节点)     │  │  (从节点)     │  │  (从节点)     │           │
│  │              │  │              │  │              │           │
│  │  读写操作     │  │  只读操作      │  │  只读操作     │           │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘           │
│         │                  │                  │                 │
│         └──────────────────┼──────────────────┘                 │
│                            ↓                                    │
│                      Oplog 复制                                  │
└─────────────────────────────────────────────────────────────────┘

选举机制（Raft 协议）：
- Primary 故障 → 自动选举新 Primary
- 多数派投票（2/3）
- 选举时间：通常 < 12 秒
```

###### 分片（Sharding）

```
分片架构：
┌─────────────────────────────────────────────────────────────────┐
│                      mongos（路由层）                             │
└─────────────────────────────────────────────────────────────────┘
                              ↓
        ┌─────────────────────┼─────────────────────┐
        ↓                     ↓                     ↓
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│  Shard 1     │      │  Shard 2     │      │  Shard 3     │
│  (副本集)     │      │  (副本集)     │      │  (副本集)     │
│              │      │              │      │              │
│  Key: A-M    │      │  Key: N-T    │      │  Key: U-Z    │
└──────────────┘      └──────────────┘      └──────────────┘

分片键选择：
✅ 高基数（Cardinality）
✅ 查询频繁
✅ 均匀分布
❌ 避免单调递增（_id、timestamp）
```

---

##### MongoDB 实战场景

###### 典型应用场景

| 场景 | 业务描述 | 数据量 | 并发量 | 关键指标 | 技术要点 |
|:---|:---|:---|:---|:---|:---|
| **内容管理系统** | 文章、评论、标签 | 千万级 | 5000 QPS | 灵活 Schema、复杂查询 | 嵌套文档、数组索引 |
| **用户画像** | 用户属性、行为数据 | 亿级 | 1万 QPS | 灵活扩展、快速查询 | 动态字段、聚合管道 |
| **日志存储** | 应用日志、审计日志 | 亿级 | 10万 TPS | 高吞吐写入、TTL | Capped Collection、TTL 索引 |
| **实时分析** | 用户行为分析、报表 | 千万级 | 1000 QPS | 复杂聚合、实时计算 | 聚合管道、Change Streams |
| **移动应用后端** | 用户数据、消息、通知 | 千万级 | 5万 QPS | 灵活 Schema、高可用 | 副本集、地理位置索引 |

---

###### 常见问题与解决方案

##### 问题 1：慢查询

**现象：**
```javascript
// 查询耗时 5 秒
db.users.find({ age: { $gt: 25 }, city: "Beijing" });
```

**解决方案：**
```javascript
// 1. 创建复合索引
db.users.createIndex({ age: 1, city: 1 });

// 2. 使用 explain 分析
db.users.find({ age: { $gt: 25 }, city: "Beijing" }).explain("executionStats");

// 3. 避免全集合扫描
// ❌ 错误
db.users.find({ $where: "this.age > 25" });  // 全集合扫描

// ✅ 正确
db.users.find({ age: { $gt: 25 } });  // 使用索引
```

##### 问题 2：内存不足

##### 问题 3：副本集延迟

**现象：**
```
场景：从节点数据延迟，读取到旧数据
影响：数据不一致，用户体验差
```

**原因分析：**
```
1. 网络延迟（跨区域部署）
2. 从节点负载过高
3. Oplog 窗口不足
4. 写入压力过大
```

**解决方案：**

**方案 1：使用 Read Concern**
```javascript
// ❌ 错误：默认读取（可能读到旧数据）
db.users.find({ userId: "user123" });

// ✅ 正确：使用 majority（读取多数派确认的数据）
db.users.find({ userId: "user123" }).readConcern("majority");

// ✅ 正确：使用 linearizable（线性一致性，最强）
db.users.find({ userId: "user123" }).readConcern("linearizable");

Read Concern 级别：
- local：读取本地数据（默认，最快，可能不一致）
- available：读取可用数据（分片集群）
- majority：读取多数派确认的数据（强一致）
- linearizable：线性一致性（最强，最慢）
```

**方案 2：增加 Oplog 大小**
```javascript
// 查看 Oplog 状态
db.getReplicationInfo();

输出：
{
  "logSizeMB": 5000,  // Oplog 大小
  "usedMB": 4500,     // 已使用
  "timeDiff": 86400,  // 时间窗口（秒）
  "timeDiffHours": 24 // 时间窗口（小时）
}

// 如果时间窗口 < 24 小时，需要增加 Oplog
// 修改配置文件
replication:
  oplogSizeMB: 10000  // 增加到 10GB

建议：
- 高写入场景：Oplog 至少 24 小时窗口
- 跨区域部署：Oplog 至少 48 小时窗口
```

**方案 3：监控副本集延迟**
```javascript
// 查看副本集状态
rs.status();

// 查看延迟
rs.printReplicationInfo();
rs.printSecondaryReplicationInfo();

// 关键指标
{
  "members": [
    {
      "name": "primary:27017",
      "stateStr": "PRIMARY",
      "optime": { "ts": Timestamp(1704067200, 1) }
    },
    {
      "name": "secondary1:27017",
      "stateStr": "SECONDARY",
      "optime": { "ts": Timestamp(1704067195, 1) },
      "optimeDate": ISODate("..."),
      "syncSourceHost": "primary:27017",
      "syncSourceId": 0,
      "replicationLag": 5  // 延迟 5 秒
    }
  ]
}

告警阈值：
- 延迟 > 10 秒：警告
- 延迟 > 60 秒：严重
```

---

##### 问题 4：分片键选择不当

**现象：**
```
场景：数据分布不均，某些分片负载过高
影响：热点分片，性能下降
```

**原因分析：**
```
❌ 错误的分片键选择：
1. 单调递增键（_id、timestamp）
   - 所有写入集中在最后一个分片
   - 其他分片空闲

2. 低基数键（status、type）
   - 只有几个不同的值
   - 无法均匀分布

3. 查询不包含分片键
   - 需要广播到所有分片
   - 性能差
```

**解决方案：**

**方案 1：选择合适的分片键**
```javascript
// ❌ 错误：单调递增
db.orders.createIndex({ _id: 1 });
sh.shardCollection("mydb.orders", { _id: 1 });
// 所有写入集中在最后一个分片

// ✅ 正确：哈希分片
db.orders.createIndex({ _id: "hashed" });
sh.shardCollection("mydb.orders", { _id: "hashed" });
// 均匀分布

// ✅ 正确：复合分片键
db.orders.createIndex({ userId: 1, orderTime: 1 });
sh.shardCollection("mydb.orders", { userId: 1, orderTime: 1 });
// userId 分散，orderTime 排序

分片键选择原则：
✅ 高基数（Cardinality）：不同值多
✅ 查询频繁：大部分查询包含分片键
✅ 均匀分布：避免热点
✅ 不可变：分片键不能修改
```

**方案 2：预分片**
```javascript
// 创建空 Chunk，避免初期热点
for (var i = 0; i < 100; i++) {
    db.adminCommand({
        split: "mydb.orders",
        middle: { userId: "user" + i }
    });
}

// 手动移动 Chunk
db.adminCommand({
    moveChunk: "mydb.orders",
    find: { userId: "user50" },
    to: "shard0001"
});
```

**方案 3：监控分片分布**
```javascript
// 查看分片状态
sh.status();

// 查看数据分布
db.orders.getShardDistribution();

输出：
Shard shard0000 at shard0000/localhost:27018
  data: 10GB docs: 1000000 chunks: 50
  estimated data per chunk: 200MB
  estimated docs per chunk: 20000

Shard shard0001 at shard0001/localhost:27019
  data: 5GB docs: 500000 chunks: 25
  estimated data per chunk: 200MB
  estimated docs per chunk: 20000

// 不均匀！需要重新平衡
```

---

##### 问题 5：连接池耗尽

**现象：**
```
错误：MongoServerSelectionError: connection pool is full
影响：应用无法连接数据库
```

**原因分析：**
```
1. 连接池配置过小
2. 连接泄漏（未关闭）
3. 并发请求过多
4. 慢查询占用连接
```

**解决方案：**

**方案 1：优化连接池配置**
```javascript
// Node.js 示例
const { MongoClient } = require('mongodb');

const client = new MongoClient('mongodb://localhost:27017', {
    maxPoolSize: 100,        // 最大连接数（默认 100）
    minPoolSize: 10,         // 最小连接数（默认 0）
    maxIdleTimeMS: 30000,    // 空闲连接超时（30 秒）
    waitQueueTimeoutMS: 5000,// 等待连接超时（5 秒）
    serverSelectionTimeoutMS: 5000  // 服务器选择超时
});

// Java 示例
MongoClientSettings settings = MongoClientSettings.builder()
    .applyConnectionString(new ConnectionString("mongodb://localhost:27017"))
    .applyToConnectionPoolSettings(builder ->
        builder.maxSize(100)
               .minSize(10)
               .maxWaitTime(5, TimeUnit.SECONDS)
               .maxConnectionIdleTime(30, TimeUnit.SECONDS))
    .build();

配置建议：
- Web 应用：maxPoolSize = 100-200
- 微服务：maxPoolSize = 50-100
- 批处理：maxPoolSize = 10-20
```

**方案 2：避免连接泄漏**
```javascript
// ❌ 错误：未关闭游标
const cursor = db.collection('users').find({});
await cursor.forEach(doc => {
    // 处理文档
});
// 游标未关闭，连接泄漏

// ✅ 正确：使用 try-finally
const cursor = db.collection('users').find({});
try {
    await cursor.forEach(doc => {
        // 处理文档
    });
} finally {
    await cursor.close();  // 确保关闭
}

// ✅ 更好：使用 toArray（自动关闭）
const docs = await db.collection('users').find({}).toArray();
```

**方案 3：监控连接池**
```javascript
// 监控连接池状态
client.on('connectionPoolCreated', event => {
    console.log('连接池创建:', event);
});

client.on('connectionPoolClosed', event => {
    console.log('连接池关闭:', event);
});

client.on('connectionCheckedOut', event => {
    console.log('连接签出:', event);
});

client.on('connectionCheckedIn', event => {
    console.log('连接签入:', event);
});

// 查看连接池指标
const stats = await db.admin().serverStatus();
console.log('连接数:', stats.connections);

输出：
{
  "connections": {
    "current": 85,      // 当前连接数
    "available": 15,    // 可用连接数
    "totalCreated": 150 // 总创建连接数
  }
}

告警阈值：
- available < 10：警告
- available < 5：严重
```

**现象：**
```
错误：Executor error during find command: OperationFailed: Sort operation used more than the maximum 33554432 bytes of RAM
```

**解决方案：**
```javascript
// 1. 使用索引排序
db.users.createIndex({ age: 1 });
db.users.find().sort({ age: 1 });  // 使用索引排序

// 2. 增加排序内存限制
db.users.find().sort({ age: 1 }).allowDiskUse(true);  // 使用磁盘排序

// 3. 分页查询
db.users.find().sort({ age: 1 }).limit(100);
```

---

#### Firestore Native

##### 架构概述

###### Firestore Native Mode 说明

> **注意**：Firestore 有两种模式：
> - **Firestore Native Mode**（推荐）：功能完整的文档数据库，支持实时订阅、离线支持、自动索引
> - **Firestore Datastore Mode**（不推荐）：遗留模式，仅用于 App Engine 遗留项目兼容，功能受限（无实时订阅、仅主键查询）
> - 同一 GCP 项目只能选择一种模式，**选择后无法切换**，所有新项目应使用 Native Mode

特点：
- 实时订阅（WebSocket）
- 离线支持（本地缓存）
- 自动索引（单字段）
- 基于 Megastore 架构
- 底层使用 Bigtable

适用场景：
- 移动应用（聊天、协作）
- Web 应用（实时仪表板）
- 需要实时订阅和离线支持

限制：
- 单文档最大 1MB
- 不支持复杂查询（OR、不等于）
- 需要手动创建复合索引

---

#### DocumentDB

##### 架构详解

DocumentDB 是 AWS 托管的 MongoDB 兼容服务，但底层架构与 MongoDB 不同。

```
DocumentDB 架构：
┌─────────────────────────────────────────────────────────────────┐
│                         客户端层                                 │
│  MongoDB Driver (兼容 3.6/4.0/5.0 API)                           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      MongoDB 兼容层                              │
│  ├─ Wire Protocol（MongoDB 协议）                                │
│  ├─ 查询解析器（兼容 MongoDB 查询语法）                             │
│  ├─ 聚合管道（兼容 MongoDB 聚合）                                  │
│  └─ 索引管理（兼容 MongoDB 索引）                                  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      计算层（实例）                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │  Primary     │  │  Replica 1   │  │  Replica 2   │           │
│  │  (读写)       │  │  (只读)       │  │  (只读)       │          │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      存储层（Aurora 架构）                         │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  共享存储（6 副本，跨 3 个 AZ）                                │ │
│  │  - 自动复制                                                  │ │
│  │  - 自动修复                                                  │ │
│  │  - 持续备份到 S3                                             │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**DocumentDB vs MongoDB 对比：**

| 维度                 | DocumentDB     | MongoDB    | 说明               |
| :----------------- | :------------- | :--------- | :--------------- |
| **架构**             | 存算分离（类 Aurora） | 存算一体       | DocumentDB 共享存储  |
| **存储引擎**           | 自研（类 Aurora）   | WiredTiger | 底层完全不同           |
| **副本集**            | 最多 15 个副本      | 最多 50 个副本  | DocumentDB 限制更多  |
| **分片**             | 🔴 不支持         | ✅ 支持       | DocumentDB 无分片   |
| **Change Streams** | 🔴 不支持         | ✅ 支持       | DocumentDB 无实时订阅 |
| **事务**             | ✅ 支持（4.0+）     | ✅ 支持       | 都支持 ACID         |
| **索引**             | ✅ 兼容           | ✅ 完整       | 基本兼容             |
| **聚合管道**           | ✅ 兼容           | ✅ 完整       | 基本兼容             |
| **API 兼容性**        | 3.6/4.0/5.0    | 最新版本       | DocumentDB 有延迟   |
| **扩展性**            | 垂直扩展           | 水平扩展（分片）   | DocumentDB 受限    |
| **存储容量**           | 128 TB         | 无限制（分片）    | DocumentDB 单集群限制 |
| **备份**             | 自动备份到 S3       | 手动备份       | DocumentDB 更方便   |
| **高可用**            | 99.99% SLA     | 需自己配置      | DocumentDB 托管    |
| **成本**             | 按实例 + 存储       | 自建成本低      | DocumentDB 更贵    |

**不支持的 MongoDB 特性：**

```javascript
// ❌ 不支持 Change Streams
const changeStream = db.collection('users').watch();
// DocumentDB 不支持

// ❌ 不支持分片
sh.shardCollection("mydb.users", { userId: 1 });
// DocumentDB 不支持

// ❌ 不支持某些聚合操作符
db.users.aggregate([
    { $graphLookup: {...} },  // 不支持
    { $facet: {...} }         // 不支持
]);

// ❌ 不支持某些索引类型
db.users.createIndex({ name: "text" });  // 全文索引不支持
db.users.createIndex({ location: "2dsphere" });  // 地理位置索引部分支持
```

**适用场景：**

```
选择 DocumentDB：
✅ 需要 MongoDB 兼容性
✅ 需要托管服务（低运维）
✅ 需要高可用（99.99% SLA）
✅ 数据量 < 128 TB
✅ 不需要分片
✅ 不需要 Change Streams
✅ 预算充足

选择 MongoDB：
✅ 需要完整的 MongoDB 特性
✅ 需要分片（水平扩展）
✅ 需要 Change Streams（实时订阅）
✅ 数据量 > 128 TB
✅ 需要最新版本
✅ 成本敏感（自建）
```

**迁移注意事项：**

```javascript
// 从 MongoDB 迁移到 DocumentDB

1. 检查兼容性
   - 使用 AWS Schema Conversion Tool
   - 检查不支持的特性
   - 测试应用兼容性

2. 数据迁移
   // 使用 mongodump/mongorestore
   mongodump --uri="mongodb://source:27017/mydb"
   mongorestore --uri="mongodb://documentdb:27017/mydb" dump/

   // 或使用 AWS DMS（Database Migration Service）
   - 支持持续复制
   - 最小化停机时间

3. 修改应用代码
   // 连接字符串
   // MongoDB
   mongodb://user:pass@mongodb:27017/mydb

   // DocumentDB（需要 TLS）
   mongodb://user:pass@documentdb:27017/mydb?tls=true&tlsCAFile=rds-combined-ca-bundle.pem

4. 性能调优
   - DocumentDB 的查询优化器与 MongoDB 不同
   - 需要重新测试和优化查询
   - 索引策略可能需要调整
```

**性能对比（实测）：**

| 操作 | DocumentDB | MongoDB（自建） | 说明 |
|:---|:---|:---|:---|
| **单文档查询** | 5-10ms | 3-5ms | DocumentDB 略慢 |
| **批量插入** | 1000 docs/s | 5000 docs/s | DocumentDB 写入慢 |
| **聚合查询** | 100-500ms | 50-200ms | DocumentDB 慢 2-3 倍 |
| **索引查询** | 10-20ms | 5-10ms | DocumentDB 略慢 |
| **故障转移** | 30-60 秒 | 10-30 秒 | DocumentDB 自动 |

---

#### Bigtable

##### 整体架构

```
┌─────────────────────────────────────────────────────────────────────┐
│  GCP Bigtable 分布式架构                                              │
│                                                                     │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  客户端层 (Client Layer)                                        │ │
│  │  ├─ HBase API（兼容 HBase 客户端）                               │ │
│  │  ├─ gRPC API（原生 Bigtable API）                               │ │
│  │  ├─ 客户端缓存（元数据缓存）                                       │ │
│  │  └─ 负载均衡（自动路由到最优节点）                                  │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              ↓                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  前端层 (Frontend Layer)                                        │ │
│  │  ├─ 全球负载均衡（Google Front End）                             │ │
│  │  ├─ 请求路由（根据 Row Key 路由到对应 Tablet Server）              │ │
│  │  ├─ 认证与授权（IAM）                                            │ │
│  │  └─ 限流与配额管理                                               │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              ↓                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  元数据管理层 (Metadata Layer)                                   │ │
│  │  ├─ Chubby（分布式锁服务，类似 ZooKeeper）                        │ │
│  │  │  ├─ 存储 Tablet 分配信息                                      │ │
│  │  │  ├─ Master 选举                                              │ │
│  │  │  └─ Schema 元数据                                            │ │
│  │  └─ Master Server（集群管理）                                    │ │
│  │     ├─ Tablet 分配与负载均衡                                      │ │
│  │     ├─ 表创建/删除                                               │ │
│  │     ├─ 垃圾回收                                                  │ │
│  │     └─ 不参与数据读写（无单点瓶颈）                                 │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              ↓                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  计算层 (Tablet Server Layer) - 无状态                           │ │
│  │                                                                │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │ │
│  │  │ Tablet Server│  │ Tablet Server│  │ Tablet Server│          │ │
│  │  │      1       │  │      2       │  │      N       │          │ │
│  │  │              │  │              │  │              │          │ │
│  │  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │          │ │
│  │  │ │ Tablet A │ │  │ │ Tablet C │ │  │ │ Tablet E │ │          │ │
│  │  │ │          │ │  │ │          │ │  │ │          │ │          │ │
│  │  │ │Commit Log│ │  │ │Commit Log│ │  │ │Commit Log│ │          │ │
│  │  │ │(Colossus)│ │  │ │(Colossus)│ │  │ │(Colossus)│ │          │ │
│  │  │ │    ↓     │ │  │ │    ↓     │ │  │ │    ↓     │ │          │ │
│  │  │ │ MemTable │ │  │ │ MemTable │ │  │ │ MemTable │ │          │ │
│  │  │ │ (内存)    │ │  │ │ (内存)   │ │  │ │ (内存)    │ │          │ │
│  │  │ │    +     │ │  │ │    +     │ │  │ │    +     │ │          │ │
│  │  │ │  Cache   │ │  │ │  Cache   │ │  │ │  Cache   │ │          │ │
│  │  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │          │ │
│  │  │ ┌──────────┐ │  │ ┌──────────┐ │  │ ┌──────────┐ │          │ │
│  │  │ │ Tablet B │ │  │ │ Tablet D │ │  │ │ Tablet F │ │          │ │
│  │  │ │          │ │  │ │          │ │  │ │          │ │          │ │
│  │  │ │Commit Log│ │  │ │Commit Log│ │  │ │Commit Log│ │          │ │
│  │  │ │(Colossus)│ │  │ │(Colossus)│ │  │ │(Colossus)│ │          │ │
│  │  │ │    ↓     │ │  │ │    ↓     │ │  │ │    ↓     │ │          │ │
│  │  │ │ MemTable │ │  │ │ MemTable │ │  │ │ MemTable │ │          │ │
│  │  │ │ (内存)    │ │  │ │ (内存)   │ │  │ │ (内存)    │ │          │ │
│  │  │ │    +     │ │  │ │    +     │ │  │ │    +     │ │          │ │
│  │  │ │  Cache   │ │  │ │  Cache   │ │  │ │  Cache   │ │          │ │
│  │  │ └──────────┘ │  │ └──────────┘ │  │ └──────────┘ │          │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘          │ │
│  │                                                                │ │
│  │  核心职责：                                                      │ │
│  │  • 处理读写请求                                                  │ │
│  │  • 管理 MemTable（内存表）                                       │ │
│  │  • 管理 Commit Log（共享预写日志，存储在 Colossus）                 │ │
│  │  • 管理 Block Cache（读缓存）                                    │ │
│  │  • Compaction（合并 SSTable）                                   │ │
│  │  • 无状态（MemTable 可重建，Commit Log 和 SSTable 在 Colossus）    │ │
│  └────────────────────────────────────────────────────────────────┘ │
│                              ↓                                      │
│  ┌────────────────────────────────────────────────────────────────┐ │
│  │  存储层 (Storage Layer) - Colossus                              │ │
│  │                                                                │ │
│  │  ┌────────────────────────────────────────────────────────────┐│ │
│  │  │  Colossus（Google 分布式文件系统，GFS 继任者）                 │ │ │
│  │  │                                                            │ │ │
│  │  │  ┌──────────┐  ┌──────────┐  ┌──────────┐                  │ │ │
│  │  │  │ SSTable  │  │ SSTable  │  │ SSTable  │                  │ │ │
│  │  │  │  File 1  │  │  File 2  │  │  File N  │                  │ │ │
│  │  │  │          │  │          │  │          │                  │ │ │
│  │  │  │ (不可变)  │  │ (不可变)  │  │ (不可变)  │                  │ │ │
│  │  │  └──────────┘  └──────────┘  └──────────┘                  │ │ │
│  │  │       ↓              ↓              ↓                      │ │ │
│  │  │  ┌──────────────────────────────────────┐                  │ │ │
│  │  │  │  自动 3 副本同步复制（跨数据中心）        │                  │ │ │
│  │  │  │  • 副本 1（Zone A）                   │                  │ │ │
│  │  │  │  • 副本 2（Zone B）                   │                  │ │ │
│  │  │  │  • 副本 3（Zone C）                   │                  │ │ │
│  │  │  └──────────────────────────────────────┘                  │ │ │
│  │  │                                                            │ │ │
│  │  │  核心特性：                                                  │ │ │
│  │  │  • 存储计算分离                                              │ │ │
│  │  │  • 自动分片                                                 │ │ │
│  │  │  • 自动副本管理                                              │ │ │
│  │  │  • 高可用（99.9% SLA）                                      │ │ │
│  │  │  • 自动扩展                                                 │ │ │
│  │  └────────────────────────────────────────────────────────────┘│ │
│  └────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────┘
```

**Bigtable 核心架构特点：**

| 层级        | 组件                 | 特点                                   | 对比 HBase                     |
| --------- | ------------------ | ------------------------------------ | ---------------------------- |
| **客户端层**  | gRPC/HBase API     | 兼容 HBase 客户端                         | 相同                           |
| **元数据管理** | Chubby + Master    | Chubby（类似 ZooKeeper）<br>Master 不参与读写 | HBase 使用 ZooKeeper + HMaster |
| **计算层**   | Tablet Server（无状态） | 无状态，可快速扩缩容<br>数据存储在 Colossus         | HBase RegionServer 有状态       |
| **存储层**   | Colossus           | 完全托管<br>自动副本<br>存储计算分离               | HBase 使用 HDFS                |
| **副本协议**  | Colossus 同步复制      | 文件系统级复制（非 Paxos）                     | HDFS 副本                      |
| **一致性**   | 单实例强一致<br>跨实例最终一致  | 单实例：Colossus 同步复制（3 副本）<br>跨实例：异步复制  | 强一致                          |
| **事务支持**  | 单行原子性              | 无跨行事务                                | 单行原子性                        |
| **扩展性**   | 自动扩展               | 完全托管，无需手动扩容                          | 手动扩容                         |
| **运维复杂度** | 极低                 | 完全托管                                 | 高（需要管理 HDFS + ZooKeeper）     |

**数据流向：**

```
写入流程（包含 Commit Log）：
Client → Tablet Server 
       ↓
   1. 写入 Commit Log（Colossus，同步）
       ↓
   2. 写入 MemTable（内存）
       ↓
   3. 返回客户端（ACK）
       ↓
   4. MemTable 满时 Flush 到 SSTable（Colossus）
       ↓
   5. 定期 Compaction（合并 SSTable）

Commit Log（共享日志）特点：
• 持久化保证：写入 MemTable 前先写 Commit Log
• 共享机制：多个 Tablet Server 可以共享同一个 Commit Log
• 存储位置：存储在 Colossus（3 副本同步）
• 崩溃恢复：Tablet Server 崩溃后，从 Commit Log 重建 MemTable
• 清理时机：MemTable Flush 到 SSTable 后，Commit Log 可删除
• 与传统 WAL 的区别：
  - 传统 WAL：每个节点独立的日志文件
  - Bigtable Commit Log：共享的、存储在 Colossus 的日志
  - 优势：Tablet 可以快速迁移到其他 Tablet Server

读取流程：
Client → Tablet Server 
       ↓
   1. Block Cache（缓存命中）
       ↓（缓存未命中）
   2. MemTable（内存）
       ↓（未找到）
   3. Colossus（读取 SSTable）
       ↓
   4. Bloom Filter 过滤 + 二分查找
```

**一致性模型详解：**

```
场景 1：单实例强一致性（Single Instance Strong Consistency）
┌─────────────────────────────────────────────────────────┐
│  Bigtable Instance（单个实例，如 us-central1）             │
│                                                         │
│  写入流程：                                               │
│  Client → Tablet Server → Colossus                      │
│                          ↓                              │
│                    同步复制到 3 个副本                     │
│                    ├─ Zone A（副本 1）                   │
│                    ├─ Zone B（副本 2）                   │
│                    └─ Zone C（副本 3）                   │
│                          ↓                              │
│                    所有副本写入成功                        │
│                          ↓                              │
│                    返回客户端（ACK）                      │
│                                                         │
│  读取流程：                                               │
│  Client → Tablet Server → 从任意副本读取                  │
│                          ↓                              │
│                    总是读到最新数据                        │
│                                                         │
│  ✅ 强一致性保证：                                        │
│  • 写入必须等待所有 3 个副本同步完成                         │
│  • 读取总是能看到最新写入的数据                              │
│  • 无延迟，无数据丢失风险                                   │
└─────────────────────────────────────────────────────────┘

场景 2：跨实例最终一致性（Cross-Instance Eventual Consistency）
┌─────────────────────────────────────────────────────────┐
│  跨区域复制（Replication）                                │
│                                                         │
│  Instance A（us-central1）                              │
│       ↓ 写入（同步，强一致）                               │
│  Colossus（3 副本）                                      │
│       ↓ 异步复制（有延迟）                                 │
│  Instance B（europe-west1）                             │
│       ↓                                                 │
│  Colossus（3 副本）                                      │
│                                                         │
│  ⚠️ 最终一致性：                                          │
│  • Instance A 写入后立即返回                              │
│  • Instance B 异步接收数据（延迟：秒级到分钟级）              │
│  • 短时间内 Instance B 可能读到旧数据                       │
│  • 最终会同步到最新状态                                     │
└─────────────────────────────────────────────────────────┘

对比总结：
┌──────────────┬─────────────┬─────────────┬──────────┐
│ 场景          │ 一致性级别    │ 延迟        │ 适用场景  │
├──────────────┼─────────────┼─────────────┼──────────┤
│ 单实例        │ 强一致       │ 5-10ms      │ 大多数应用 │
│ 跨实例复制     │ 最终一致     │ 秒级-分钟级   │ 容灾备份  │
└──────────────┴─────────────┴─────────────┴──────────┘

关键点：
• 99% 的用户使用单实例 → 享受强一致性
• 跨实例复制主要用于容灾，不用于读写分离
• 如果需要全球强一致性，应该使用 Spanner（非 Bigtable）
```

**Tablet 分片机制：**

```
表 (Table)
  ↓
按 Row Key 范围分片
  ↓
┌─────────────┬─────────────┬─────────────┐
│  Tablet 1   │  Tablet 2   │  Tablet 3   │
│ Row: A-F    │ Row: G-M    │ Row: N-Z    │
│ (100-200MB) │ (100-200MB) │ (100-200MB) │
└─────────────┴─────────────┴─────────────┘
       ↓              ↓              ↓
  Tablet Server  Tablet Server  Tablet Server
       ↓              ↓              ↓
    Colossus       Colossus       Colossus
    (SSTable)      (SSTable)      (SSTable)

特点：
• 自动分裂（Tablet 超过阈值自动分裂）
• 自动负载均衡（Master 自动迁移 Tablet）
• Range 分区（适合范围查询）
```

---

##### 核心概念速查

**数据库类型**：宽列存储数据库（Wide-Column Store），不是键值数据库，也不是专用时序数据库

**核心特征**：
- **数据模型**：Row Key + Column Family + Column = Value（简单值：字符串或数字）
- **存储引擎**：LSM-Tree（Tablet），写优化，高吞吐
- **分区策略**：Range分区（按Row Key范围），适合时间范围查询
- **一致性**：单集群强一致（Colossus同步复制），跨集群最终一致（异步复制）
- **索引能力**：仅Row Key索引，无二级索引
- **事务范围**：单行原子性
- **典型延迟**：5-10ms

---

##### 一致性与事务机制

**核心问题：为什么 Bigtable 只支持单行原子性，不支持跨行事务？**

###### 1. 底层架构决定一致性模型

**Bigtable 不使用 Paxos 协议**（与 Spanner 的关键区别）

```
Bigtable 架构：

Client
  ↓
Tablet Server（无状态，无事务协调能力）
  ↓
Colossus（分布式文件系统，自动3副本同步复制）
  ↓
物理磁盘（多个数据中心）

一致性保证机制：
1. 单行写入 → Tablet Server 写入 Colossus（同步）
2. Colossus 自动复制到 3 个副本（同步，类似 RAID）
3. 所有副本写入成功 → 返回客户端（强一致性）
4. 单行读取 → 从任意副本读取（强一致，因为同步复制）

关键点：
✅ 依赖 Colossus 的文件系统级复制（非 Paxos 共识）
✅ 单个 SSTable 文件的强一致性
❌ 没有跨 Tablet 的分布式事务协调器
❌ 没有 2PC（两阶段提交）机制
❌ 没有全局时钟（TrueTime）
```

**对比 Spanner 架构（支持跨行事务）**

```
Spanner 架构：

Client
  ↓
事务协调器（Transaction Manager）
  ├─ 2PC 协调
  ├─ TrueTime 时间戳
  └─ 跨 Paxos 组协调
  ↓
Spanserver（有状态）
  ├─ Paxos Group（复制组）
  │  ├─ Leader + Followers
  │  ├─ 同步复制（多数派）
  │  └─ 强一致性
  ↓
Colossus（分布式文件系统）

跨行事务流程：
1. Client → 事务协调器
2. 协调器 → Paxos Leader 1（Prepare Row A）
3. 协调器 → Paxos Leader 2（Prepare Row B）
4. 两个 Leader 都确认 → 协调器发起 Commit
5. TrueTime 等待（确保外部一致性）
6. 协调器 → Client（成功）

关键差异：
✅ Spanner 有事务协调器
✅ Spanner 使用 Paxos 保证副本一致性
✅ Spanner 有 TrueTime 全局时钟
✅ Spanner 支持 2PC 跨行事务
```

###### 2. 单行原子性的实现机制

**Bigtable 单行写入流程**

```
写入流程（单行原子性保证）：

1. Client 发起写入请求
   PUT row_key="user_001"
     cf:name="Alice"
     cf:age=25

2. Tablet Server 接收请求
   ├─ 写入 MemTable（内存）
   ├─ 写入 WAL（预写日志，持久化到 Colossus）
   └─ 返回成功（此时已持久化）

3. Colossus 同步复制
   WAL 文件 → 3 个副本（同步写入）
   ├─ 副本1（数据中心A）
   ├─ 副本2（数据中心B）
   └─ 副本3（数据中心C）

4. 后台异步刷盘
   MemTable → SSTable（Colossus）

原子性保证：
✅ WAL 写入是原子的（Colossus 文件系统保证）
✅ 单行的所有列一起写入（不会部分成功）
✅ 崩溃恢复：从 WAL 重放未刷盘的数据
```

**单行读取流程（强一致性）**

```
读取流程：

1. Client 发起读取请求
   GET row_key="user_001"

2. Tablet Server 处理
   ├─ 查询 MemTable（内存）
   ├─ 查询 SSTable（Colossus）
   └─ 合并多个版本（按 Timestamp）

3. Colossus 读取
   从任意副本读取（因为同步复制，所有副本一致）

强一致性保证：
✅ Colossus 同步复制确保所有副本一致
✅ 读取任意副本都能看到最新写入
✅ 无需 Quorum 读（与 Cassandra 不同）
```

###### 3. 为什么不支持跨行事务？

**原因 1：无分布式事务协调器**

```
跨行事务场景：

BEGIN TRANSACTION
  UPDATE row_key='Alice' SET balance=balance-100  -- Tablet 1
  UPDATE row_key='Bob' SET balance=balance+100    -- Tablet 2
COMMIT

问题：
❌ Tablet 1 和 Tablet 2 在不同的 Tablet Server
❌ 没有事务协调器协调两个 Tablet
❌ 无法保证原子性（可能部分成功）

Bigtable 的设计选择：
- 优化单行低延迟（1-10ms）
- 牺牲跨行事务能力
- 适合高吞吐、低延迟场景（IoT、时序数据）
```

**原因 2：无 Paxos 共识协议**

```
Bigtable vs Spanner 复制机制：

Bigtable（Colossus 复制）：
- 文件系统级复制（类似 RAID）
- 同步复制到 3 个副本
- 单文件强一致性
- ❌ 无法协调跨文件（跨 Tablet）操作

Spanner（Paxos 复制）：
- 每个 Paxos Group 管理一个数据分片
- Leader 协调写入，Followers 同步复制
- 多数派确认（2/3）
- ✅ 可以协调跨 Paxos Group 的事务（2PC）
```

**原因 3：无全局时钟（TrueTime）**

```
Spanner 的 TrueTime：
- GPS + 原子钟提供全球时钟同步
- 不确定性 < 10ms
- 保证事务全局顺序（外部一致性）

Bigtable 没有 TrueTime：
- 无法保证跨 Tablet 的事务顺序
- 无法实现外部一致性
- 只能保证单行的时间戳顺序
```

**原因 4：设计目标不同**

| 维度 | Bigtable | Spanner |
|:---|:---|:---|
| **设计目标** | 高吞吐、低延迟、大规模 | 全球分布式事务 |
| **典型延迟** | 1-10ms | 10-50ms（跨区域） |
| **吞吐量** | 百万 TPS | 数万 TPS |
| **一致性** | 单行强一致 | 全局强一致（外部一致性） |
| **事务** | 单行原子 | 跨行/跨表 ACID |
| **适用场景** | IoT、时序、日志 | 金融、支付、全球业务 |
| **成本** | 中（按节点） | 高（按节点+复杂度） |

###### 4. 一致性级别详解

**单集群强一致性**

```
单个 Bigtable 集群（同一区域）：

写入：
Client → Tablet Server → Colossus（同步3副本）→ 返回成功

读取：
Client → Tablet Server → Colossus（任意副本）→ 返回数据

保证：
✅ 读取立即看到最新写入（强一致性）
✅ 无需 Quorum 读写
✅ 延迟：1-10ms

原理：
Colossus 同步复制确保所有副本一致
类似于单机数据库的 ACID 保证
```

**跨集群最终一致性**

```
多个 Bigtable 集群（跨区域复制）：

主集群（us-central1）：
Client → 写入 → 立即返回

从集群（europe-west1）：
异步复制（延迟：秒-分钟级）

保证：
⚠️ 最终一致性（有延迟）
⚠️ 可能读到旧数据
✅ 高可用（主集群故障时切换）

使用场景：
- 灾难恢复（DR）
- 跨区域读取加速
- 不适合强一致性需求
```

###### 5. 实际应用建议

**如果需要跨行事务，怎么办？**

**方案 1：应用层实现（补偿事务）**

```python
# Saga 模式
def transfer_money(from_user, to_user, amount):
    try:
        # 步骤1：扣款
        deduct_balance(from_user, amount)
        
        # 步骤2：加款
        add_balance(to_user, amount)
        
    except Exception as e:
        # 补偿：回滚扣款
        add_balance(from_user, amount)
        raise
```

**方案 2：使用 Spanner（原生支持）**

```sql
-- Spanner 跨行事务
BEGIN TRANSACTION;
  UPDATE accounts SET balance = balance - 100 WHERE user_id = 'Alice';
  UPDATE accounts SET balance = balance + 100 WHERE user_id = 'Bob';
COMMIT;
```

**方案 3：数据建模优化（避免跨行）**

```
Bigtable 设计（单行事务）：

Row Key: transaction_12345
  ├─ from_user: Alice
  ├─ to_user: Bob
  ├─ amount: 100
  └─ status: pending/completed

异步处理：
1. 写入事务记录（单行原子）
2. 后台任务处理转账
3. 更新账户余额
4. 更新事务状态
```

###### 6. 总结

**Bigtable 一致性模型**

| 维度 | 单集群 | 跨集群 |
|:---|:---|:---|
| **一致性级别** | 强一致性 | 最终一致性 |
| **复制机制** | Colossus 同步复制（3副本） | 异步复制 |
| **读写延迟** | 1-10ms | 写入快，读取可能旧 |
| **事务范围** | 单行原子 | 单行原子 |
| **适用场景** | 生产环境主集群 | 灾难恢复、读加速 |

**为什么不支持跨行事务？**

1. 🔴 无分布式事务协调器（无 Transaction Manager）
2. 🔴 无 Paxos 共识协议（依赖 Colossus 文件系统复制）
3. 🔴 无全局时钟（无 TrueTime API）
4. 🔴 无 2PC 实现（无两阶段提交）
5. ✅ 设计目标：优化单行低延迟，非事务场景

**Bigtable 的定位**

```
Bigtable = 高吞吐、低延迟的 NoSQL 数据库

适合：
✅ 时序数据（IoT、监控）
✅ 日志存储（应用日志、审计日志）
✅ 用户行为（点击流、事件流）
✅ 大规模稀疏数据（TB/PB 级）

不适合：
❌ 需要跨行事务（金融转账、订单系统）
❌ 复杂查询（JOIN、聚合）
❌ 小数据量 OLTP（成本高）
```

---

##### 适用场景深度解析

###### 为什么无二级索引却适合时序/日志/明细场景？

**核心答案：Row Key 有序存储 + LSM-Tree 追加优化 = 无需索引的高效范围查询**

##### 1. 时序数据场景

**场景描述**：IoT 传感器每秒产生百万条数据，需要：
- 高吞吐实时写入（>100万 TPS）
- 按设备 ID + 时间范围查询原始数据
- 低延迟点查（获取最新读数）

**Row Key 设计**：
```
Row Key: {device_id}#{timestamp}

示例：
sensor_001#20250110220000  → temperature=25.3, humidity=60
sensor_001#20250110220100  → temperature=25.5, humidity=61
sensor_001#20250110220200  → temperature=25.7, humidity=62
sensor_002#20250110220000  → temperature=22.1, humidity=55
```

**为什么不需要二级索引？**

```
查询："获取 sensor_001 在 22:00-23:00 的所有数据"

Bigtable 执行过程：
1. 定位起始 Row Key: sensor_001#20250110220000
2. 顺序扫描到结束 Row Key: sensor_001#20250110230000
3. 数据物理连续存储，一次磁盘顺序读取

性能：10-50ms（扫描1000-10000行）

对比 ClickHouse（需要这个场景）：
- 如果查询"计算所有传感器过去1小时平均温度" → Bigtable 需全表扫描（慢）
- ClickHouse 列式存储 + 向量化聚合 → 秒级完成

结论：Bigtable 适合"检索原始数据"，不适合"聚合分析"
```

**实际架构模式**：
```
实时层（Bigtable）：
- 写入：传感器数据实时追加
- 查询：按设备 ID + 时间范围检索原始记录
- 保留：最近 7 天热数据

分析层（ClickHouse/BigQuery）：
- 写入：每小时从 Bigtable 批量导出
- 查询：跨设备聚合、趋势分析、异常检测
- 保留：历史数据长期存储
```

##### 2. 日志数据场景

**场景描述**：应用日志每秒产生数十万条，需要：
- 实时追加写入（无更新）
- 按应用 ID + 时间查询最近日志
- 按日志 ID 快速定位

**Row Key 设计**：
```
Row Key: {app_id}#{reverse_timestamp}#{log_id}

示例（reverse_timestamp = MAX_TIMESTAMP - actual_timestamp，使最新日志排在前面）：
nginx#99999999999999-1736515665#abc123  → level=ERROR, msg="Connection timeout"
nginx#99999999999999-1736515664#def456  → level=INFO, msg="Request processed"
nginx#99999999999999-1736515663#ghi789  → level=WARN, msg="Slow query"
```

**为什么不需要二级索引？**

```
查询 1："获取 nginx 最近 1 小时的 ERROR 日志"

Bigtable 执行：
1. 起始 Row Key: nginx#99999999999999-{1小时前timestamp}
2. 结束 Row Key: nginx#99999999999999-{当前timestamp}
3. 顺序扫描 + 应用层过滤 level=ERROR

性能：50-200ms（扫描数千行，过滤数百行）

查询 2："按日志 ID 精确查询"
Row Key: nginx#*#abc123
使用 Row Key 前缀扫描 → 5-10ms

对比 Elasticsearch（更适合这个场景）：
- 如果需要"全文搜索日志内容" → Bigtable 无倒排索引（慢）
- Elasticsearch 倒排索引 + 分词 → 毫秒级全文检索

结论：Bigtable 适合"按时间/ID检索日志"，不适合"全文搜索"
```

##### 3. 明细数据场景

**场景描述**：电商订单明细，需要：
- 按订单 ID 快速查询订单详情
- 按用户 ID 查询历史订单
- 高并发读写

**Row Key 设计方案 1（按订单查询优化）**：
```
Row Key: order#{order_id}

order#20250110001  → user_id=123, amount=99.99, status=paid
order#20250110002  → user_id=456, amount=199.99, status=shipped
```

**Row Key 设计方案 2（按用户查询优化）**：
```
Row Key: user#{user_id}#{order_id}

user#123#20250110001  → amount=99.99, status=paid
user#123#20250109005  → amount=49.99, status=delivered
user#456#20250110002  → amount=199.99, status=shipped
```

**为什么不需要二级索引？**

```
方案 1 查询：
- 按订单 ID 查询 → 点查，1-5ms ✅
- 按用户 ID 查询 → 全表扫描，秒-分钟级 ❌

方案 2 查询：
- 按用户 ID 查询 → 范围扫描，10-50ms ✅
- 按订单 ID 查询 → 全表扫描，秒-分钟级 ❌

解决方案：双写策略
Table 1: order#{order_id}  → 订单详情查询
Table 2: user#{user_id}#{order_id}  → 用户订单列表查询

对比 MySQL（有二级索引）：
- MySQL 可以在单表上建立 order_id 和 user_id 索引
- Bigtable 需要应用层维护多个表

结论：Bigtable 适合"单一访问模式"，多访问模式需双写或外部索引
```

##### 4. 核心技术原理总结

**为什么 Row Key 有序存储能替代索引？**

```
传统数据库（MySQL）：
数据存储：无序（按插入顺序或聚簇索引）
范围查询：依赖 B-Tree 索引 → 索引查找 + 回表

Bigtable：
数据存储：按 Row Key 字典序物理排序
范围查询：直接顺序扫描物理连续的数据块

示例对比：
查询："device_001 在 10:00-11:00 的数据"

MySQL：
1. 在 (device_id, timestamp) 索引上查找范围
2. 获取主键列表
3. 回表读取完整行（随机 I/O）
性能：100-500ms（索引 + 回表开销）

Bigtable：
1. 定位 Row Key: device_001#20250110100000
2. 顺序读取到 device_001#20250110110000
3. 数据物理连续（顺序 I/O）
性能：10-50ms（纯顺序扫描）
```

**LSM-Tree 为什么适合时序/日志？**

```
时序/日志特点：
- 写入：追加为主（很少更新）
- 读取：按时间范围查询（很少随机点查）

LSM-Tree 优势：
1. 写入优化：
   - 数据先写入内存（MemTable）
   - 批量刷盘到 SSTable（顺序写）
   - 写入吞吐：100万+ TPS

2. 范围查询优化：
   - SSTable 按 Row Key 排序
   - 范围查询 = 顺序扫描多个 SSTable
   - Bloom Filter 跳过不相关的 SSTable

3. 压缩优化：
   - 时序数据重复度高（如传感器类型、单位）
   - 列族压缩率：5-10倍

对比 B-Tree（MySQL）：
- 写入：随机 I/O，需要维护索引
- 范围查询：索引扫描 + 回表（随机 I/O）
- 写入吞吐：数万 TPS
```

##### 5. Bigtable vs ClickHouse/BigQuery 场景对比

| 维度       | Bigtable           | ClickHouse/BigQuery |
| :------- | :----------------- | :------------------ |
| **数据写入** | 实时追加，每秒百万行         | 批量写入，每批数千-数万行       |
| **查询类型** | 点查/范围查询（按 Row Key） | 聚合/分析查询（任意列）        |
| **查询延迟** | 1-50ms             | 秒-分钟级               |
| **典型查询** | "获取设备123过去10分钟数据"  | "计算所有设备过去1年平均值"     |
| **数据保留** | 热数据（天-周）           | 冷数据（月-年）            |
| **成本**   | 按节点计费（固定成本）        | 按扫描量/存储计费（变动成本）     |
| **适合场景** | 操作型查询（OLTP-style）  | 分析型查询（OLAP-style）   |

**实际案例：IoT 数据平台架构**

```
数据流：
传感器 → Bigtable（实时写入）→ Dataflow（ETL）→ BigQuery（分析）
         ↓
      实时查询 API
      （按设备 ID + 时间范围）

查询分工：
1. 实时监控："设备123当前温度是多少？"
   → Bigtable 点查，2ms

2. 近期数据："设备123过去1小时温度曲线"
   → Bigtable 范围查询，20ms

3. 历史分析："所有设备过去1年温度趋势"
   → BigQuery 列式聚合，5秒

4. 异常检测："哪些设备温度超过阈值？"
   → BigQuery WHERE + 聚合，10秒
```

**总结：Bigtable 的"时序/日志/明细"定位**

```
✅ 适合 Bigtable：
- 高频实时写入（>10万 TPS）
- 按主键/时间范围检索原始记录
- 低延迟点查（<10ms）
- 数据保留周期短（天-周）

❌ 不适合 Bigtable：
- 复杂聚合（SUM/AVG/GROUP BY）
- 任意字段过滤（WHERE age > 25）
- 全文搜索
- JOIN 多表关联

推荐架构：
Bigtable（热数据层）+ ClickHouse/BigQuery（分析层）
```

---

###### Bigtable 的键值特性底层机制

**为什么说 Bigtable 有"强键值特性"？**

```
Bigtable 本质上是一个有序的、分布式的、多维的键值映射表：

数据模型：
Bigtable = Map<RowKey, Map<ColumnFamily:Column, Map<Timestamp, Value>>>

简化理解：
Bigtable[RowKey][ColumnFamily:Column][Timestamp] = Value

这就是一个多维的 KV 结构！
```

**1. 数据访问完全依赖 Row Key**

```
底层存储结构（SSTable）：

┌─────────────────────────────────────────┐
│ Row Key: user#001                       │
│   cf:name[t1] = "Alice"                 │
│   cf:age[t1] = 25                       │
├─────────────────────────────────────────┤
│ Row Key: user#002                       │
│   cf:name[t1] = "Bob"                   │
│   cf:age[t1] = 30                       │
├─────────────────────────────────────────┤
│ Row Key: user#003                       │
│   cf:name[t1] = "Carol"                 │
└─────────────────────────────────────────┘

关键特点：
✅ Row Key 是唯一的访问路径
✅ Row Key 按字典序排序存储
✅ 所有查询必须指定 Row Key 或 Row Key 范围
❌ 无法通过 Column 值查询（如：WHERE age=25）
```

**2. LSM-Tree 存储引擎（典型 KV 架构）**

```
写入流程：

1. 写入 MemTable（内存）
   RowKey → ColumnFamily:Column → Value
   ┌─────────────────────────────────┐
   │ MemTable（内存，按 RowKey 排序） │
   │ user#001 → cf:name = "Alice"    │
   │ user#002 → cf:name = "Bob"      │
   └─────────────────────────────────┘

2. MemTable 满后刷盘到 SSTable
   ┌─────────────────────────────────┐
   │ SSTable（磁盘，不可变）          │
   │ 按 RowKey 排序存储               │
   └─────────────────────────────────┘

3. 多个 SSTable 定期合并（Compaction）
   SSTable_1 + SSTable_2 → SSTable_3（合并后）

读取流程：

1. 根据 RowKey 查找对应的 Tablet
2. 在 Tablet 内二分查找 RowKey
3. 读取该 RowKey 的所有列
4. 合并多个版本（Timestamp）

时间复杂度：
- 点查询：O(log N) - 二分查找
- 范围查询：O(log N + K) - 二分查找 + 顺序扫描 K 行
```

**3. 分区策略基于 Row Key（典型 KV 分区）**

```
Bigtable 的分区（Tablet）：

Table: users
├─ Tablet 1: [user#000000 ~ user#099999]  ← Tablet Server 1
├─ Tablet 2: [user#100000 ~ user#199999]  ← Tablet Server 2
├─ Tablet 3: [user#200000 ~ user#299999]  ← Tablet Server 3
└─ Tablet 4: [user#300000 ~ user#399999]  ← Tablet Server 4

查询：Get("user#150000")
→ 路由到 Tablet 2
→ 在 Tablet 2 内二分查找
→ 返回结果

这就是典型的 KV 数据库分区策略！
```

**4. SSTable 文件结构（KV 存储格式）**

```
SSTable 文件结构（按 Row Key 组织）：

┌─────────────────────────────────────────┐
│ Data Block 1                            │
│ ├─ RowKey: user#001                     │
│ │  ├─ cf:name[t1] = "Alice"             │
│ │  └─ cf:age[t1] = 25                   │
│ ├─ RowKey: user#002                     │
│ │  ├─ cf:name[t1] = "Bob"               │
│ │  └─ cf:age[t1] = 30                   │
└─────────────────────────────────────────┘
┌─────────────────────────────────────────┐
│ Index Block（稀疏索引）                   │
│ ├─ user#001 → Data Block 1, offset=0   │
│ ├─ user#100 → Data Block 1, offset=1024│
│ └─ user#200 → Data Block 2, offset=0   │
└─────────────────────────────────────────┘
┌─────────────────────────────────────────┐
│ Bloom Filter                            │
│ 快速判断 RowKey 是否存在                  │
└─────────────────────────────────────────┘

查询流程：
1. Bloom Filter 判断 RowKey 是否存在
2. Index Block 二分查找定位 Data Block
3. Data Block 内顺序扫描找到 RowKey
4. 返回该 RowKey 的所有列

这就是典型的 KV 数据库查询流程！
```

**5. 与纯 KV 数据库的区别**

| 维度            | Bigtable                    | Redis                | DynamoDB        |
| :------------ | :-------------------------- | :------------------- | :-------------- |
| **Value 结构**  | 多维（CF + Column + Timestamp） | 简单（String/Hash/List） | 半结构化（Attribute） |
| **Value 可见性** | 部分可见（可按 Column 过滤）          | 不可见（整体读写）            | 不可见（整体读写）       |
| **多版本**       | ✅ 原生支持（Timestamp）           | 🔴 不支持               | 🔴 不支持          |
| **稀疏性**       | ✅ 稀疏（只存有值的列）                | 🔴 固定结构              | ⚠️ 部分支持         |
| **范围查询**      | ✅ 强（Row Key 有序）             | ⚠️ 弱（需特殊结构）          | ✅ 有（Sort Key）   |

**6. 典型 KV 使用场景**

```java
// 场景1：用户会话存储（典型 KV 操作）
// Row Key = session_id
table.put(
  "session_abc123",
  "session_data:user_id", "user_001",
  "session_data:login_time", "2024-01-01T10:00:00",
  "session_data:ip", "192.168.1.1"
);

// 读取（KV 查询）
Row row = table.get("session_abc123");
String userId = row.getValue("session_data:user_id");

// 为什么快？
// ✅ 直接通过 session_id（Row Key）定位
// ✅ 一次 I/O 读取所有会话数据
// ✅ 延迟：1-5ms

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

// 场景2：时序数据（KV 范围扫描）
// Row Key = device_id#timestamp（复合键）
table.put(
  "device_001#20240101100000",
  "metrics:temperature", "25.5",
  "metrics:humidity", "60",
  "metrics:pressure", "1013"
);

// 范围查询（KV 数据库的范围扫描）
Scan scan = new Scan()
  .setStartRow("device_001#20240101000000")
  .setStopRow("device_001#20240101235959");

// 为什么快？
// ✅ Row Key 有序，范围扫描高效
// ✅ 顺序读取，预读优化
// ✅ 延迟：10-100ms（扫描1000行）

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

// 场景3：用户画像（宽表 KV）
// Row Key = user_id
table.put(
  "user_001",
  "profile:name", "Alice",
  "profile:age", "25",
  "tags:golang", "true",      // 动态添加
  "tags:kubernetes", "true",  // 动态添加
  "behavior:last_login", "2024-01-01"
);

// 读取（KV 查询）
Row row = table.get("user_001");
// 只读取有值的列，不读取空列

// 为什么适合？
// ✅ 每个用户的标签不同（稀疏）
// ✅ 可以动态添加新标签（无需修改 Schema）
// ✅ 通过 user_id 快速查询（KV 特性）
```

**总结：Bigtable 的键值特性**

```
Bigtable = 具有宽列特性的 KV 数据库

核心机制：
1. 数据模型：Map<RowKey, Map<Column, Value>>
2. 存储结构：LSM-Tree + SSTable，按 RowKey 排序
3. 查询方式：只能通过 RowKey 查询（点查/范围查询）
4. 分区策略：按 RowKey 范围分区（Tablet）
5. 索引机制：稀疏索引 + Bloom Filter（典型 KV 优化）
6. 无二级索引：无法通过 Column 值查询

为什么叫"宽列存储"而不是"KV数据库"？
- Value 不是简单的 Blob，而是结构化的 Column Family + Column
- 支持稀疏列（每行可以有不同的列）
- 支持多版本（Timestamp）
- 但查询方式仍然是 KV 模式（必须通过 Row Key）
```

---

**适用场景**：
- ✅ IoT时序数据（传感器数据、设备日志）
- ✅ 应用日志存储（追加写入，按时间查询）
- ✅ 监控指标（高吞吐写入，时间范围扫描）
- ✅ 金融交易历史（大规模数据，顺序访问）
- ✅ 大规模稀疏数据（动态列，TB/PB级）

**不适用场景**：
- 🔴 小数据量OLTP（成本高，最低3节点）
- 🔴 复杂查询（无二级索引，无JOIN）
- 🔴 强一致性需求（仅最终一致）
- 🔴 需要嵌套对象/数组（仅支持简单值）

**Row Key 设计最佳实践**：
```
时序数据：sensor_id#timestamp
  - 优点：按传感器分区，时间范围查询高效
  - 注意：避免单调递增timestamp（热点问题）

日志数据：app_id#reverse_timestamp#log_id
  - 优点：最新日志在前，查询最近日志快
  - reverse_timestamp = Long.MAX_VALUE - timestamp

用户行为：user_id#event_type#timestamp
  - 优点：按用户分区，支持事件类型过滤
```

**与其他数据库对比**：

| 维度          | Bigtable    | DynamoDB   | MongoDB      |
| :---------- | :---------- | :--------- | :----------- |
| **数据模型**    | 宽列（稀疏列）     | 键值（黑盒）     | 文档（透明）       |
| **Value类型** | 简单值（字符串/数字） | 任意类型（不可查询） | 嵌套对象/数组（可查询） |
| **二级索引**    | 🔴 不支持      | ✅ GSI/LSI  | ✅ 任意字段       |
| **查询能力**    | 仅Row Key    | 主键+GSI     | 任意字段+聚合      |
| **时序数据**    | ✅ 原生优化      | ⚠️ 需设计GSI  | ⚠️ 需索引优化     |
| **成本模型**    | 按节点+存储      | 按请求+存储     | 按实例          |
| **最低成本**    | 高（3节点起）     | 低（按需模式）    | 中（单实例）       |

**成本优化建议**：
- 节点数优化：最低3节点，根据吞吐需求扩展
- HDD存储：冷数据使用HDD（成本降低50%）
- 数据压缩：启用压缩（节省30-50%存储）
- TTL策略：自动删除过期数据（Column Family级别）

**反模式警告**：
- 🔴 不要用Bigtable做小数据量OLTP（成本高，功能受限）
- 🔴 不要用单调递增Row Key（写热点问题）
- 🔴 不要存储大Value（>10MB，影响性能）
- 🔴 不要期望复杂查询（无二级索引，需应用层处理）

---

### Spanner

##### 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         客户端层                                 │
│  gcloud CLI │ 应用程序 │ Client Libraries (Java/Go/Python/Node)  │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│              全球前端层（GFE - Google Front End）                 │
│            负载均衡 │ TLS 终止 │ DDoS 防护 │ 请求路由               │
│                                                                 │
│  类似 AWS Global Accelerator（非 CDN）：                           │
│  • Anycast IP：全球单一入口                                       │
│  • 智能路由：路由到最近的 Spanserver                               │
│  • Google 骨干网：优化跨区域延迟                                   │
│  • 无缓存：直接访问数据库（动态数据，强一致性）                        │
│  • 与 CDN 区别：不缓存内容，实时查询，全局事务                        │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      Spanner API 前端                            │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ SQL 解析器    │  │ 查询优化器     │  │ 事务协调器     │          │
│  │ (Parser)     │  │ (Optimizer)  │  │ (Coordinator)│           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    Spanserver 层（核心）                          │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Region 1 (us-central1) - 美国                              │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │ │
│  │  │ Spanserver 1 │  │ Spanserver 2 │  │ Spanserver 3 │      │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘      │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Region 2 (europe-west1) - 欧洲                             │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │ │
│  │  │ Spanserver 4 │  │ Spanserver 5 │  │ Spanserver 6 │      │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘      │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Region 3 (asia-east1) - 亚洲                               │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │ │
│  │  │ Spanserver 7 │  │ Spanserver 8 │  │ Spanserver 9 │      │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘      │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│              单个 Spanserver 内部结构（基于论文）                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Tablet 1 (Key Range: A-M)                                 │ │
│  │  ┌──────────────────────────────────────────────────────┐  │ │
│  │  │  Paxos Group (5 副本跨 Region)                        │  │ │
│  │  │  ┌────────┐  ┌────────┐  ┌────────┐                  │  │ │
│  │  │  │ Leader │  │Replica │  │Replica │  ...             │  │ │
│  │  │  │(美国)   │  │(欧洲)  │  │(亚洲)   │                  │  │ │
│  │  │  └────────┘  └────────┘  └────────┘                  │  │ │
│  │  │  ┌──────────────┐  ┌──────────────┐                  │  │ │
│  │  │  │ Lock Table   │  │ Transaction  │                  │  │ │
│  │  │  │ (2PL 锁)     │  │ Manager(2PC) │                  │  │ │
│  │  │  └──────────────┘  └──────────────┘                  │  │ │
│  │  └──────────────────────────────────────────────────────┘  │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Tablet 2 (Key Range: N-Z)                                 │ │
│  │  └─ Paxos Group (5 副本跨 Zone) ...                         │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    底层存储层（Colossus）                         │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Google 分布式文件系统（GFS 继任者）                           │ │
│  │  - 数据分片存储                                              │ │
│  │  - 自动副本管理                                              │ │
│  │  - 跨区域持久化                                              │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    TrueTime 层（全球时钟同步）                     │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  GPS 时钟 + 原子钟（每个数据中心）                             │ │
│  │  - TT.now() 返回时间区间 [earliest, latest]                  │ │
│  │  - 不确定性窗口：                                            │ │
│  │    • 论文（2012）：平均 4ms，最大 7ms                          │ │
│  │    • 现代 Spanner（2020+）：平均 1-2ms，最大 5ms               │ │
│  │    • 同一数据中心：< 1ms                                     │ │
│  │    • 同一区域不同数据中心：1-2ms                              │ │
│  │    • 跨区域：2-5ms                                          │ │
│  │  - commit-wait：等待不确定性消散                              │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      数据分片与路由层                              │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Split（动态分裂）                                           │ │
│  │  - 按主键范围自动分裂与合并                                    │ │
│  │  - 基于大小 + 负载动态调整                                    │ │
│  │  - 支持手动预分裂（Pre-split，有配额限制）                      │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Directory（论文概念）                                       │ │
│  │  - 管理副本与局部性的抽象                                      │ │
│  │  - 数据移动的最小单元                                         │ │
│  │  - 可细分为多个片段以实现负载均衡                               │ │
│  └────────────────────────────────────────────────────────────┘ │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │  Interleave（产品实践）                                      │ │
│  │  - 表间物理共置（父表与子表存于同一 Split）                      │ │
│  │  - 减少网络跳转与远程存取                                      │ │
│  │  - ⚠️ 一旦生效不可直接撤销，需重建数据                          │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

**核心组件详解：**

| 层级               | 组件                         | 职责          | 关键点                                |
| ---------------- | -------------------------- | ----------- | ---------------------------------- |
| **客户端层**         | Client Libraries           | 应用程序接入      | 支持多语言 SDK                          |
| **全球前端层**        | GFE                        | 负载均衡与安全     | 全球统一入口，仅网络层转发，不处理 SQL              |
| **API 前端**       | SQL 解析/优化/协调               | SQL 处理与事务协调 | 分布式查询优化；**Coordinator 协调器包含执行器功能** |
| **Spanserver 层** | Tablet + Paxos Group       | 数据分片与复制     | 每个 Tablet 5 副本跨区域                  |
| **存储层**          | Colossus                   | 分布式文件系统     | 自动副本管理                             |
| **TrueTime 层**   | GPS + 原子钟                  | 全球时钟同步      | 不确定性几毫秒                            |
| **分片路由层**        | Split/Directory/Interleave | 数据分片与局部性    | 动态分裂，负载均衡                          |

**关键设计：**

| 设计 | 说明 | 优势 |
|------|------|------|
| **多主架构** | 每个区域都可写入，无需转发 | 低延迟，高可用 |
| **外部一致性** | 全球强一致性（比线性一致性更强） | 事务全局有序 |
| **TrueTime** | GPS + 原子钟，时钟不确定性几毫秒 | 保证全球事务顺序 |
| **Paxos 复制** | 每个 Tablet 5 副本跨区域 | 高可用，自动故障转移 |
| **动态 Split** | 按大小 + 负载自动分裂 | 负载均衡，避免热点 |
| **Interleave** | 表间物理共置 | 减少跨网络查询 |

**数据流转示例（写事务）：**
```
客户端发送 INSERT
    ↓
GFE 路由到最近的 Spanner API 前端
    ↓
SQL 解析器解析 SQL
    ↓
查询优化器生成执行计划
    ↓
事务协调器定位 Tablet Leader
    ↓
Tablet Leader 执行两阶段锁（2PL）
    ↓
TrueTime 分配时间戳
    ↓
Paxos Group 同步复制（多数派确认）
    ↓
commit-wait 等待 TrueTime 不确定性消散
    ↓
事务提交，返回响应
    ↓
Colossus 持久化数据
```

**Spanner 多主架构（Multi-Master）：**
```
为什么能实现多主写入？
1. 数据按 Key Range 分片（Tablet）
2. 每个 Tablet 有独立的 Leader（可在任何区域）
3. 写入路由到对应 Tablet 的 Leader
4. TrueTime 保证全球事务顺序
5. Paxos 保证副本一致性

延迟取决于 Tablet Leader 位置：
- Leader 在本地区域 → 低延迟
- Leader 在远程区域 → 高延迟（跨区域网络）
```

---

##### Spanner 核心概念

##### 0. 全球前端层（GFE）底层机制与原理

**GFE 是什么？**

GFE（Google Front End）是 Spanner 的全球网络接入层，类似 AWS Global Accelerator，为数据库访问提供全球加速和智能路由。

**GFE 完整功能架构：**

```
┌─────────────────────────────────────────┐
│           GFE 功能层次                   │
├─────────────────────────────────────────┤
│                                         │
│  1. Anycast IP（入口层）                  │
│     • 全球统一 IP 地址                    │
│     • BGP 自动路由到最近节点               │
│                                         │
│  2. 边缘处理层                            │
│     • TLS 终止（边缘解密）                 │
│     • DDoS 防护（流量清洗）                │
│     • 连接池管理                          │
│                                         │
│  3. 负载均衡层                            │
│     • 健康检查                           │
│     • 流量分发                           │
│     • 故障转移                           │
│                                         │
│  4. 智能路由层                            │
│     • 定位最近的 Spanner API 前端          │
│     • 选择最优网络路径                     │
│     • 利用 Google 骨干网                  │
│                                         │
└─────────────────────────────────────────┘
```

**核心机制详解：**

| 机制 | 底层技术 | 作用 | 延迟优化 |
|------|----------|------|----------|
| **Anycast IP** | BGP 协议广播同一 IP | 用户自动连接到最近的 GFE 节点 | 减少 50-100ms |
| **TLS 终止** | 边缘节点 SSL/TLS 处理 | 避免跨区域 TLS 握手 | 减少 100-200ms |
| **DDoS 防护** | 流量清洗与限流 | 过滤恶意请求 | 保护可用性 |
| **负载均衡** | 健康检查 + 权重分配 | 分发到多个后端 | 避免单点过载 |
| **Google 骨干网** | 专用光纤网络 | 跨区域高速传输 | 比公网快 30-50% |

**智能路由底层机制详解：**

GFE 的"**选择最优网络路径**"依赖两层路由机制：

```
┌─────────────────────────────────────────────────────────────┐
│              GFE 智能路由双层架构                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  第一层：Anycast 路由（网络层）                                 │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  • 技术：BGP 协议                                       │  │
│  │  • 原理：基于网络拓扑距离（AS 跳数）                       │  │
│  │  • 作用：将用户路由到最近的 GFE 边缘节点                   │  │
│  │  • 特点：自动、静态、基于网络层                            │  │
│  └───────────────────────────────────────────────────────┘  │
│                          ↓                                  │
│  第二层：Espresso SDN 智能路由（应用层）                        │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  • 技术：Google Espresso SDN + Maglev 负载均衡器         │  │
│  │  • 原理：实时网络遥测 + 动态路径选择                       │  │
│  │  • 作用：从 GFE 到后端 Spanner 选择最优路径               │  │
│  │  • 特点：动态、智能、基于实时状态                          │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**第二层路由机制对比（GCP vs AWS）：**

| 维度 | Google Espresso SDN | AWS Global Accelerator |
|------|---------------------|------------------------|
| **路由数据源** | 实时网络遥测（延迟、丢包、拥塞） | 路由表（IP 前缀 → Region 延迟映射） |
| **数据结构** | 分布式状态机 + 时序数据 | KV 数据库（类似 DynamoDB） |
| **更新频率** | 毫秒级实时更新 | 秒级/分钟级更新 |
| **决策因素** | 延迟 + 丢包 + 拥塞 + 负载 | 主要基于延迟 |
| **路径调整** | 动态切换（故障自动规避） | 相对静态（需手动配置） |
| **实现方式** | SDN 控制器集中决策 | 边缘节点查表决策 |

**Espresso SDN 工作原理：**

```
步骤 1：实时网络遥测
┌─────────────────────────────────────────┐
│  Google 骨干网每条链路持续监控：            │
│  • 延迟（RTT）：每秒测量                    │
│  • 丢包率：实时统计                        │
│  • 带宽利用率：拥塞检测                    │
│  • 链路健康状态：故障检测                   │
└─────────────────────────────────────────┘
              ↓
步骤 2：集中式 SDN 控制器
┌─────────────────────────────────────────┐
│  Espresso 控制器收集全球网络状态：          │
│  • 构建实时网络拓扑图                      │
│  • 计算最优路径（Dijkstra 算法变种）        │
│  • 考虑多维度权重：                        │
│    - 延迟权重：40%                        │
│    - 丢包权重：30%                        │
│    - 负载权重：20%                        │
│    - 成本权重：10%                        │
└─────────────────────────────────────────┘
              ↓
步骤 3：动态路由决策
┌─────────────────────────────────────────┐
│  GFE 节点查询 Espresso 控制器：            │
│  • 请求：从 GFE(美国) 到 Spanner(亚洲)     │
│  • 响应：最优路径 + 备用路径                │
│  • 缓存：本地缓存路由决策（TTL 1-5秒）       │
└─────────────────────────────────────────┘
              ↓
步骤 4：流量转发
┌─────────────────────────────────────────┐
│  通过 Google 骨干网转发：                  │
│  • 使用 MPLS/Segment Routing 标签        │
│  • 沿最优路径转发数据包                    │
│  • 实时监控，异常时切换备用路径              │
└─────────────────────────────────────────┘
```

**AWS Global Accelerator 的 KV 路由表机制：**

```
路由表结构（简化示例）：
┌──────────────────┬─────────────┬──────────┐
│  IP 前缀          │ 目标 Region │  延迟(ms) │
├──────────────────┼─────────────┼──────────┤
│  1.0.0.0/8       │  ap-east-1  │    5     │
│  1.0.0.0/8       │  us-west-1  │   150    │
│  203.0.113.0/24  │  eu-west-1  │    8     │
│  203.0.113.0/24  │  us-east-1  │   120    │
└──────────────────┴─────────────┴──────────┘

决策逻辑：
1. 根据客户端 IP 前缀查表
2. 选择延迟最低的 Region
3. 定期（每分钟）更新延迟数据
4. 故障时切换到次优 Region
```

**关键差异总结：**

| 特性 | Google Espresso | AWS KV 路由表 |
|------|----------------|--------------|
| **智能程度** | 高（实时动态） | 中（定期更新） |
| **复杂度** | 高（SDN 控制器） | 低（查表） |
| **响应速度** | 毫秒级切换 | 分钟级切换 |
| **成本** | 高（需专用硬件） | 低（软件实现） |
| **适用场景** | 全球分布式数据库 | 应用加速 |

**为什么 Spanner 需要 Espresso 而非简单 KV 表？**

1. **数据库对延迟敏感**：毫秒级延迟差异影响事务性能
2. **跨区域事务**：需要实时选择最优 Paxos Leader 位置
3. **故障快速切换**：链路故障时需秒级切换路径
4. **负载均衡**：避免热点 Region 过载

**客户端访问完整流程：**

```
步骤 1：客户端发起请求
应用程序 → spanner.googleapis.com
↓

步骤 2：DNS 解析
DNS 返回 Anycast IP（如 142.250.x.x）
↓

步骤 3：Anycast 路由（自动）
用户请求 → 最近的 GFE 节点
• 美国用户 → GFE（美国）
• 欧洲用户 → GFE（欧洲）
• 亚洲用户 → GFE（亚洲）
↓

步骤 4：GFE 边缘处理
GFE 节点执行：
├─ TLS 握手（边缘完成，无需跨区域）
├─ 身份验证
├─ DDoS 检测
└─ 连接建立
↓

步骤 5：智能路由到后端
GFE → Google 骨干网 → Spanner API 前端
• 选择最近的 Spanner 区域
• 通过专用网络传输
↓

步骤 6：数据库处理
Spanner API → Spanserver → 返回结果
```

**关键点：**

✅ **Anycast IP 是入口**：让用户快速接入最近的 GFE 
✅ **GFE 是完整服务**：提供安全、负载均衡、路由等功能 
✅ **不仅仅是转发**：在边缘节点处理 TLS、安全等 

**类比：**
- **Anycast IP** = 机场的地址（让你找到最近的机场）
- **GFE** = 整个机场（安检、登机、航班调度）

**与 AWS Global Accelerator 的相似性：**

| 特性 | GFE | AWS AGA |
|------|-----|---------|
| **Anycast IP** | ✅ 有 | ✅ 有 |
| **智能路由** | ✅ 有 | ✅ 有 |
| **专用骨干网** | ✅ Google 骨干网 | ✅ AWS 骨干网 |
| **边缘接入** | ✅ 全球边缘节点 | ✅ 全球边缘位置 |
| **无缓存** | ✅ 实时转发 | ✅ 实时转发 |
| **服务类型** | 数据库服务 | 应用加速 |

**关键优势：**

1. **全球单一入口**：用户无需关心数据库部署位置
2. **自动最优路由**：BGP 自动选择最近接入点
3. **骨干网加速**：跨区域延迟比公网低 30-50%
4. **透明性**：应用层无需感知网络优化

**延迟示例：**

```
场景：美国用户访问 Spanner

无 GFE（直接访问）：
用户（美国）→ 公网 → Spanner（亚洲）
延迟：200-300ms

有 GFE：
用户（美国）→ GFE（美国）→ Google 骨干网 → Spanner（亚洲）
延迟：100-150ms（减少 50%）

最优情况（Leader 在本地）：
用户（美国）→ GFE（美国）→ Spanner（美国）
延迟：< 10ms
```

---

##### 1. 查询执行流程底层机制

**为什么 Spanner 使用"事务协调器"而非传统"执行器"？**

Spanner 是分布式数据库，查询执行涉及多个 Spanserver 节点的协调，因此采用"事务协调器（Transaction Coordinator）"这一术语，但它实际上包含了传统数据库执行器的功能。

**传统单机数据库 vs Spanner 分布式架构：**

| 阶段 | 传统 OLTP（如 MySQL） | Spanner |
|------|----------------------|---------|
| **1. 解析** | SQL Parser | SQL Parser（API 前端） |
| **2. 优化** | Query Optimizer | Query Optimizer（API 前端） |
| **3. 执行** | Executor（单机执行） | Transaction Coordinator（分布式协调） |
| **4. 存储** | Storage Engine（本地） | Spanserver（跨区域分布） |

**Spanner 查询执行完整流程：**

```
┌─────────────────────────────────────────────────────────────┐
│  步骤 1：客户端发送 SQL                                        │
│  SELECT * FROM users WHERE region = 'US'                    │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  步骤 2：GFE 网络层转发（不处理 SQL）                           │
│  • Anycast 路由到最近的 GFE 节点                              │
│  • TLS 终止、负载均衡                                         │
│  • 转发到 Spanner API 前端                                    │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  步骤 3：SQL Parser（解析）                                   │
│  • 词法分析：识别关键字（SELECT, FROM, WHERE）                  │
│  • 语法分析：构建抽象语法树（AST）                               │
│  • 语义分析：验证表、列是否存在                                  │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  步骤 4：Query Optimizer（优化）                              │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  传统优化（与 OLTP/OLAP 相同）                           │  │
│  │  • 生成多个执行计划                                      │  │
│  │  • 基于统计信息选择最优计划                               │  │
│  │  • 选择索引 vs 全表扫描                                  │  │
│  │  • JOIN 算法选择（Nested Loop/Hash/Merge）              │  │
│  └───────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  分布式特殊优化（Spanner 独有）                           │  │
│  │  ✅ 数据局部性优化                                      │  │
│  │     • 查询涉及哪些 Tablet？                             │  │
│  │     • Tablet Leader 在哪个区域？                        │  │
│  │     • 是否使用 Interleave 表共置？                       │  │
│  │  ✅ 网络成本优化                                        │  │
│  │     • 跨区域查询成本（50-200ms）                         │  │
│  │     • 数据移动成本 vs 计算成本                            │  │
│  │     • 最小化网络传输量                                   │  │
│  │  ✅ 分布式 JOIN 优化                                    │  │
│  │     • Broadcast JOIN（小表广播）                        │  │
│  │     • Shuffle JOIN（大表重分布）                        │  │
│  │     • Collocated JOIN（同 Tablet 本地 JOIN）            │  │
│  │  ✅ 查询下推优化                                        │  │
│  │     • 过滤条件下推到 Spanserver                         │  │
│  │     • 聚合计算下推（减少数据传输）                         │  │
│  │  ✅ 并行度优化                                          │  │
│  │     • 根据 Tablet 数量决定并行度                         │  │
│  │     • 避免单个 Spanserver 过载                          │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  步骤 5：Transaction Coordinator（执行/协调）                  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  5.1 定位数据分片                                       │  │
│  │  • 根据 WHERE 条件确定涉及的 Tablet                      │  │
│  │  • region = 'US' → Tablet_US                          │  │
│  └───────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  5.2 路由到 Spanserver                                 │  │
│  │  • 查找 Tablet_US 的 Leader 位置                        │  │
│  │  • 发送查询请求到对应 Spanserver                         │  │
│  └───────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  5.3 并行执行（如果跨多个 Tablet）                        │  │
│  │  • 多个 Spanserver 并行处理                             │  │
│  │  • Coordinator 收集结果                                │  │
│  └───────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  5.4 事务管理                                          │  │
│  │  • 读事务：获取 TrueTime 时间戳                          │  │
│  │  • 写事务：两阶段锁（2PL）+ 两阶段提交（2PC）               │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  步骤 6：Spanserver 执行                                      │
│  • Tablet Leader 读取数据                                    │
│  • 应用过滤条件（WHERE region = 'US'）                        │
│  • 返回结果给 Coordinator                                    │
└─────────────────────────────────────────────────────────────┘
                         ↓
┌─────────────────────────────────────────────────────────────┐
│  步骤 7：结果聚合与返回                                        │
│  • Coordinator 合并多个 Spanserver 的结果                     │
│  • 排序、去重（如需要）                                        │
│  • 通过 GFE 返回给客户端                                      │
└─────────────────────────────────────────────────────────────┘
```

**关键点说明：**

| 组件 | 传统数据库角色 | Spanner 特殊性 |
|------|---------------|---------------|
| **SQL Parser** | 解析 SQL | 相同，但需处理分布式扩展语法 |
| **Query Optimizer** | 生成执行计划 | 需考虑数据分布、网络延迟、跨区域成本、数据局部性 |
| **Transaction Coordinator** | Executor（执行器） | 不仅执行，还需协调多个 Spanserver |
| **Spanserver** | Storage Engine | 分布式存储节点，自带 Paxos 复制 |

**Spanner 优化器 vs 传统优化器对比：**

| 优化维度 | 传统 OLTP/OLAP | Spanner |
|---------|---------------|---------|
| **优化目标** | 单机执行效率（CPU/内存/磁盘） | 分布式执行效率 + 网络成本最小化 |
| **成本模型** | 磁盘 I/O > CPU > 内存 | 网络传输 > 磁盘 I/O > CPU |
| **JOIN 策略** | Nested Loop / Hash / Merge | + Broadcast / Shuffle / Collocated |
| **数据移动** | 不考虑（本地数据） | 核心考虑（跨区域 50-200ms） |
| **并行度** | 基于 CPU 核数 | 基于 Tablet 数量和分布 |
| **查询下推** | 存储引擎层下推 | Spanserver 层下推（跨网络） |
| **统计信息** | 表/索引统计 | + Tablet 分布、Leader 位置 |

**Spanner 优化器的成本计算示例：**

```
查询：SELECT * FROM orders WHERE user_id = 123

传统优化器成本：
├─ 索引扫描：100 次磁盘 I/O × 10ms = 1000ms
└─ 全表扫描：10000 次磁盘 I/O × 10ms = 100000ms
选择：索引扫描（成本更低）

Spanner 优化器成本：
├─ 本地 Tablet 查询：10ms（Leader 在本地区域）
├─ 跨区域 Tablet 查询：150ms（Leader 在远程区域）
└─ 跨多个 Tablet 并行查询：max(10ms, 150ms) = 150ms
选择：优先路由到本地 Tablet，避免跨区域查询
```

**为什么叫"协调器"而非"执行器"？**

1. **分布式特性**：查询可能涉及多个 Spanserver，需要协调而非单机执行
2. **事务管理**：跨区域事务需要两阶段提交（2PC）协调
3. **并行执行**：多个 Spanserver 并行处理，Coordinator 负责结果聚合
4. **网络感知**：需要考虑网络延迟、数据局部性等分布式因素

**读写事务的执行差异：**

```
读事务（Read-Only Transaction）：
1. Coordinator 获取 TrueTime 时间戳
2. 路由到相关 Spanserver
3. Spanserver 读取对应时间戳的数据（MVCC）
4. 无需锁，无需等待
5. 返回结果

写事务（Read-Write Transaction）：
1. Coordinator 分配事务 ID
2. 两阶段锁（2PL）：获取涉及行的锁
3. 执行写操作（缓存在内存）
4. 两阶段提交（2PC）：
   • Prepare 阶段：所有 Spanserver 准备提交
   • Commit 阶段：TrueTime 分配时间戳
5. commit-wait：等待 TrueTime 不确定性消散
6. Paxos 复制到多数派副本
7. 释放锁，返回结果
```

**性能优化机制：**

| 优化 | 说明 | 效果 |
|------|------|------|
| **数据局部性** | Interleave 表共置，减少跨网络查询 | 延迟降低 50-80% |
| **并行执行** | 多个 Spanserver 并行处理 | 吞吐量提升 N 倍 |
| **查询下推** | 过滤条件在 Spanserver 执行 | 减少网络传输 |
| **批量操作** | Batch API 减少往返次数 | 延迟降低 10-100 倍 |

---

##### 2. TrueTime API

**TrueTime 是什么？**

TrueTime 是 Google 自研的全球时钟同步 API，通过 GPS 时钟和原子钟提供高精度时间，是 Spanner 实现外部一致性的核心技术。

**TrueTime API 接口：**

| API | 返回值 | 说明 |
|-----|--------|------|
| TT.now() | TTinterval: [earliest, latest] | 返回当前时间的区间（不确定性窗口） |
| TT.after(t) | true if t < TT.now().earliest | 判断时间 t 是否已经过去 |
| TT.before(t) | true if t > TT.now().latest | 判断时间 t 是否还未到来 |

**时钟不确定性：**

```
TT.now() 返回时间区间：
[earliest, latest]
    ↑         ↑
    |         |
    |         +-- 最晚可能的当前时间
    +------------ 最早可能的当前时间

不确定性 = latest - earliest
论文中：< 7ms（平均 4ms）
实际部署：几毫秒量级
```

**TrueTime 实现原理：**

```
每个数据中心部署：
┌─────────────────────────────────────┐
│  Time Master 服务器（多台）           │
│  ┌──────────┐  ┌──────────┐         │
│  │ GPS 接收器│  │ 原子钟    │         │
│  │ (多数)    │  │ (少数)   │         │
│  └──────────┘  └──────────┘         │
│       ↓              ↓              │
│  时间源互相校验，检测异常               │
└─────────────────────────────────────┘
            ↓
┌─────────────────────────────────────┐
│  Spanserver（每台机器）               │
│  - 定期与 Time Master 同步            │
│  - 本地维护时钟偏移                    │
│  - 计算不确定性窗口                    │
└─────────────────────────────────────┘

时钟不确定性来源：
1. 网络延迟（与 Time Master 通信）
2. 时钟漂移（本地时钟与标准时间的偏差）
3. 同步间隔（定期同步的时间间隔）
```

**commit-wait 机制：**

```
写事务提交流程：
1. 事务准备提交，获取 TT.now() = [t1, t2]
2. 选择提交时间戳 s = t2（最晚时间）
3. commit-wait：等待直到 TT.after(s) = true
   - 即等待到 TT.now().earliest > s
   - 确保 s 已经成为"过去"
4. 事务提交，对外可见

为什么需要 commit-wait？
- 保证事务时间戳的全局顺序
- 确保后续事务看到已提交的事务
- 实现外部一致性（External Consistency）

等待时间 ≈ 不确定性窗口 ≈ 几毫秒
```

**外部一致性（External Consistency）：**

```
定义：如果事务 T1 在事务 T2 开始前提交，
     则 T1 的时间戳 < T2 的时间戳

比线性一致性更强：
- 线性一致性：单个对象的操作有序
- 外部一致性：跨对象的事务有序

实现方式：
1. TrueTime 提供全球统一的时间基准
2. commit-wait 确保时间戳的全局顺序
3. Paxos 保证副本一致性

结果：全球任意两个事务都有明确的先后顺序
```

##### 2. 数据分片与复制

**Tablet 的两层概念：横向分片 + 纵向复制**

Spanner 通过 Tablet 实现两个维度的数据组织：

```
┌─────────────────────────────────────────────────────────────┐
│                    数据库全部数据                             │
└─────────────────────────────────────────────────────────────┘
                         ↓
        横向维度：数据分片（Tablet 之间）
┌──────────────┬──────────────┬──────────────┬──────────────┐
│  Tablet_1    │  Tablet_2    │  Tablet_3    │  Tablet_4    │
│  [A-E]       │  [F-J]       │  [K-O]       │  [P-Z]       │
└──────────────┴──────────────┴──────────────┴──────────────┘
       ↓              ↓              ↓              ↓
   纵向维度：数据复制（Tablet 内部）
   
Tablet_1 的 5 个副本：
┌────────────┐
│  Leader    │ ← 美国（处理读写）
├────────────┤
│  Follower  │ ← 美国（参与复制）
├────────────┤
│  Follower  │ ← 欧洲（参与复制）
├────────────┤
│  Follower  │ ← 亚洲（参与复制）
├────────────┤
│  Follower  │ ← 亚洲（参与复制）
└────────────┘
这 5 个副本存储相同数据（100% 复制）
```

**关键理解：**

| 维度 | 说明 | 目的 |
|------|------|------|
| **横向分片** | 不同 Tablet 存储不同 Key Range 的数据 | 水平扩展，分散负载 |
| **纵向复制** | 同一 Tablet 的 5 个副本存储相同数据 | 高可用，容灾备份 |
| **数据完整性** | 所有 Tablet 的 Leader 数据加起来 = 数据库全部数据 | 无数据丢失 |

**实际示例：**

```
假设 Spanner 数据库有 1 亿用户数据，分成 4 个 Tablet：

Tablet_1: user_id [0-25M]        → 2500 万用户
  ├─ Leader (美国)               ← 这 2500 万用户的主副本
  └─ 4 个 Followers (全球分布)    ← 这 2500 万用户的备份

Tablet_2: user_id [25M-50M]      → 2500 万用户
  ├─ Leader (欧洲)
  └─ 4 个 Followers

Tablet_3: user_id [50M-75M]      → 2500 万用户
  ├─ Leader (亚洲)
  └─ 4 个 Followers

Tablet_4: user_id [75M-100M]     → 2500 万用户
  ├─ Leader (美国)
  └─ 4 个 Followers

数据库全部数据 = Tablet_1 Leader + Tablet_2 Leader + 
                Tablet_3 Leader + Tablet_4 Leader
              = 2500万 + 2500万 + 2500万 + 2500万
              = 1 亿用户
```

---

**Tablet（数据分片）：**

| 概念 | 说明 |
|------|------|
| **Tablet** | 按主键范围（Key Range）分片的数据单元，由 5 个副本整体构成 |
| **Paxos Group** | 每个 Tablet 对应一个 Paxos Group（5 个副本） |
| **Leader** | 每个 Paxos Group 有 1 个 Leader（主副本，处理读写） |
| **Follower** | 其他 4 个 Follower（从副本），支持两种分布：<br>• Multi-Region：跨 Region 分布（全球高可用）<br>• Regional：同一 Region 跨 Zone 分布（区域低延迟） |
| **副本关系** | 5 个副本存储相同数据（100% 复制），非数据切分 |
| **Split** | Tablet 的动态分裂与合并（对外抽象） |

**Tablet 副本结构示例：**

```
Tablet_1 (Key Range: user_id [0-1000万])
├─ Leader Replica    → Spanserver（美国 us-central1）     ← 处理读写
├─ Follower Replica  → Spanserver（美国 us-west1）        ← 参与复制
├─ Follower Replica  → Spanserver（欧洲 europe-west1）    ← 参与复制
├─ Follower Replica  → Spanserver（亚洲 asia-east1）      ← 参与复制
└─ Follower Replica  → Spanserver（亚洲 asia-northeast1） ← 参与复制

关键点：
• 这 5 个副本共同构成 Tablet_1
• 每个副本存储完整数据（100%），非 1/5 分片
• 写入需要多数派（3/5）确认
• Paxos 协议保证 5 个副本数据一致
```

**Split 动态分裂：**

```
初始状态：
Tablet 1: [A-Z]  (1GB, 1000 QPS)

负载增长：
Tablet 1: [A-Z]  (10GB, 10000 QPS)  ← 触发分裂

自动分裂：
Tablet 1: [A-M]  (5GB, 5000 QPS)
Tablet 2: [N-Z]  (5GB, 5000 QPS)

分裂条件（系统自动判断）：
- 大小：通常在 4-8 GB 时触发分裂（非固定值，系统动态调整）
- 负载：CPU 利用率 > 65% 或 QPS 持续过高
- 热点：某个 Key Range 的 CPU 使用率显著高于其他 Split
- Split 数量：每个节点建议 < 5000 个 Split（过多影响性能）

手动预分裂（Pre-split）：
- 建表时指定分裂点
- 避免初期热点
- 有配额限制（默认每天 5 次 DDL 操作）
- 需要提前规划分裂点
- 不当的预分裂可能导致空 Split 浪费资源
```

**Paxos 复制：**

Spanner 支持两种副本分布配置，根据创建实例时的配置决定：

**配置 1：Multi-Region（多区域配置）**

```
Tablet 的 Paxos Group（5 副本跨 3 个 Region）：

Region 1 (us-central1 - 美国中部):
  ┌────────┐  ┌────────┐
  │ Leader │  │Replica │
  └────────┘  └────────┘

Region 2 (europe-west1 - 欧洲西部):
  ┌────────┐  ┌────────┐
  │Replica │  │Replica │
  └────────┘  └────────┘

Region 3 (asia-east1 - 亚洲东部):
  ┌────────┐
  │Replica │
  └────────┘

特点：
• 跨地理区域分布（全球高可用）
• 写入延迟：50-200ms（跨区域 Paxos 复制）
• 读取延迟：< 10ms（本地 Leader）或 50-200ms（远程 Leader）
• 容灾能力：区域级故障不影响
• 成本：较高
• 适用场景：全球应用、金融交易、关键业务
```

**配置 2：Regional（区域配置）**

```
Tablet 的 Paxos Group（5 副本跨同一 Region 的 3 个 Zone）：

Region: us-central1
├─ Zone A (us-central1-a):
│    ┌────────┐  ┌────────┐
│    │ Leader │  │Replica │
│    └────────┘  └────────┘
├─ Zone B (us-central1-b):
│    ┌────────┐  ┌────────┐
│    │Replica │  │Replica │
│    └────────┘  └────────┘
└─ Zone C (us-central1-c):
     ┌────────┐
     │Replica │
     └────────┘

特点：
• 同一 Region 内跨 Zone 分布（区域高可用）
• 写入延迟：< 10ms（同 Region Paxos 复制）
• 读取延迟：< 5ms（本地 Leader）
• 容灾能力：Zone 级故障不影响
• 成本：较低
• 适用场景：区域应用、低延迟需求
```

**配置对比：**

| 维度 | Multi-Region | Regional |
|------|-------------|----------|
| **副本分布** | 跨 3 个 Region | 同一 Region 跨 3 个 Zone |
| **写入延迟** | 50-200ms | < 10ms |
| **读取延迟** | < 10ms（本地）/ 50-200ms（远程） | < 5ms |
| **容灾级别** | 区域级 | Zone 级 |
| **成本** | 高 | 低 |
| **适用场景** | 全球应用 | 区域应用 |

##### Paxos 协议的作用域

⚠️ **关键理解：Paxos 只作用于单个 Tablet 内部**

**Spanner 架构中的作用域层次：**

```
全局层面（跨 Tablet）：
├─ 2PC：协调多个 Tablet 的事务
├─ TrueTime：全球时间同步
└─ Transaction Coordinator：事务协调器

Tablet 层面（单个 Tablet 内部）：
└─ Paxos：管理 1 个 Leader + 4 个 Followers
    ├─ 副本复制
    └─ Leader 选举
```

**Paxos 的作用范围：**

```
┌─────────────────────────────────────┐
│  Tablet_1 (user_id [0-50M])         │
│  ┌───────────────────────────────┐  │
│  │  Paxos Group 1（5 个副本）      │  │
│  │  ├─ Leader (美国)              │  │
│  │  ├─ Follower (美国)            │  │
│  │  ├─ Follower (欧洲)            │  │
│  │  ├─ Follower (亚洲)            │  │
│  │  └─ Follower (亚洲)            │  │
│  │                               │  │
│  │  Paxos 只在这 5 个副本间工作     │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
         ↕ 跨 Tablet 协调由 2PC 负责
┌─────────────────────────────────────┐
│  Tablet_2 (user_id [50M-100M])      │
│  ┌───────────────────────────────┐  │
│  │  Paxos Group 2（5 个副本）      │  │
│  │  独立的 Paxos 协议              │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘

Paxos 负责：
✅ 单个 Tablet 的 5 个副本之间的数据复制
✅ 单个 Tablet 的 Leader 选举
❌ 不负责跨 Tablet 的事务协调（那是 2PC）
❌ 不负责全局时间顺序（那是 TrueTime）
```

**Paxos 写入流程（两种配置相同）：**

```
1. 客户端写入请求路由到 Leader
2. Leader 提议（Propose）写入
3. 多数派（3/5）确认
4. Leader 应用（Apply）写入
5. 返回客户端成功
```

**Paxos 在 Spanner 中的两个核心作用：**

1. Tablet 的 Replica 复制
2. Tablet 的 Leader 选举

```
作用 1：副本复制（数据一致性）
┌─────────────────────────────────────┐
│  Tablet Leader 写入数据              │
│         ↓                           │
│  Paxos Propose（提议）               │
│         ↓                           │
│  5 个副本投票                         │
│  ├─ Replica 1: Accept               │
│  ├─ Replica 2: Accept               │
│  ├─ Replica 3: Accept  ← 3/5 多数派  │
│  ├─ Replica 4: Timeout              │
│  └─ Replica 5: Timeout              │
│         ↓                           │
│  Leader 应用写入                     │
│  返回客户端成功                       │
└─────────────────────────────────────┘

作用 2：Leader 选举（高可用性）
┌─────────────────────────────────────┐
│  Tablet Leader 故障                  │
│         ↓                           │
│  Followers 检测到 Leader 失联         │
│         ↓                           │
│  Paxos 选举新 Leader                 │
│  ├─ Replica 1: 提议自己为 Leader      │
│  ├─ Replica 2: 投票给 Replica 1      │
│  ├─ Replica 3: 投票给 Replica 1      │
│  └─ 3/5 多数派同意                    │
│         ↓                           │
│  Replica 1 成为新 Leader（秒级）      │
│  继续处理读写请求                      │
└─────────────────────────────────────┘

关键点：
• Paxos 只在单个 Tablet 内部使用
• 不参与跨 Tablet 的事务协调（那是 2PC 的工作）
• 不参与全局时间顺序（那是 TrueTime 的工作）
• 保证单个 Tablet 的副本一致性和高可用性
```

**Leader 选举详细机制：**

| 维度 | 说明 |
|------|------|
| **触发条件** | Leader 故障、网络分区、手动切换 |
| **选举时间** | 秒级（通常 1-5 秒） |
| **选举策略** | 优先选择低延迟区域的副本 |
| **多数派要求** | 需要 3/5 副本同意 |
| **脑裂避免** | 奇数副本 + 多数派机制 |
| **数据完整性** | 新 Leader 必须有最新数据 |

**副本分布策略：**
- Multi-Region：跨区域高可用（区域故障不影响）
- Regional：跨 Zone 高可用（Zone 故障不影响）
- 奇数副本：避免脑裂

**Directory 与 Interleave：**

**Directory（论文概念）：**
- 管理副本与局部性的抽象
- 数据移动的最小单元
- 可包含多个 Tablet
- 系统可将 Directory 细分为多个片段

**Interleave（表交错存储）详解：**

**什么是 Interleave？**

Interleave 是 Spanner 的数据局部性优化特性，让父表和子表的相关数据物理上存储在同一个 Tablet 中，避免跨网络查询。

**核心概念：**

```
传统存储（没有 Interleave）：
┌─────────────────────────────────┐
│  Tablet_1 (美国)                 │
│  Users 表：                      │
│  ├─ user_id=1, name='Alice'     │
│  └─ user_id=2, name='Bob'       │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│  Tablet_2 (欧洲)                 │
│  Orders 表：                     │
│  ├─ order_id=101, user_id=1     │
│  ├─ order_id=102, user_id=1     │
│  └─ order_id=201, user_id=2     │
└─────────────────────────────────┘

问题：
❌ JOIN 查询需要跨 Tablet（美国 → 欧洲）
❌ 延迟：150-200ms（跨 Region 网络）
❌ 事务涉及两个 Tablet，需要 2PC

使用 Interleave：
┌─────────────────────────────────┐
│  Tablet_1 (美国)                 │
│  物理存储顺序：                   │
│  ├─ Users: user_id=1, name='Alice'       ← 父表
│  ├─ Orders: user_id=1, order_id=101      ← 子表（紧邻父表）
│  ├─ Orders: user_id=1, order_id=102      ← 子表（紧邻父表）
│  ├─ Users: user_id=2, name='Bob'         ← 父表
│  └─ Orders: user_id=2, order_id=201      ← 子表（紧邻父表）
└─────────────────────────────────┘

优势：
✅ JOIN 查询在同一 Tablet（本地查询）
✅ 延迟：< 10ms（无跨网络）
✅ 事务只涉及一个 Tablet，无需 2PC
```

**SQL 语法示例：**

```sql
-- 1. 创建父表
CREATE TABLE Users (
  user_id STRING(36) NOT NULL,
  name STRING(100),
  email STRING(100),
) PRIMARY KEY (user_id);

-- 2. 创建子表，使用 INTERLEAVE IN PARENT
CREATE TABLE Orders (
  user_id STRING(36) NOT NULL,  -- ⚠️ 必须包含父表主键
  order_id STRING(36) NOT NULL,
  amount FLOAT64,
  status STRING(20),
  created_at TIMESTAMP,
) PRIMARY KEY (user_id, order_id),  -- ⚠️ 子表主键必须以父表主键开头
  INTERLEAVE IN PARENT Users ON DELETE CASCADE;
  
-- 3. 查询（无跨网络）
SELECT u.name, o.amount
FROM Users u
JOIN Orders o ON u.user_id = o.user_id
WHERE u.user_id = '123';

-- 结果：所有数据在同一 Tablet，延迟 < 10ms
```

**性能对比：**

| 场景 | 没有 Interleave | 使用 Interleave | 性能提升 |
|------|----------------|----------------|---------|
| **JOIN 查询** | 150-200ms（跨 Region） | < 10ms（本地） | **15-20 倍** |
| **事务操作** | 需要 2PC（跨 Tablet） | 本地事务（单 Tablet） | **10-15 倍** |
| **网络传输** | 需要跨 Region 传输 | 无网络传输 | **节省 100%** |
| **并发能力** | 受 2PC 限制 | 无 2PC 开销 | **提升 2-3 倍** |

**适用场景：**

```
✅ 适合使用 Interleave：
1. 父子关系明确
   • 用户 → 订单
   • 博客 → 评论
   • 客户 → 发票

2. 经常 JOIN 查询
   • SELECT * FROM Users JOIN Orders

3. 事务同时操作父子表
   • 创建用户 + 创建订单
   • 删除用户 + 级联删除订单

4. 子表数据量适中
   • 每个父记录对应 < 1000 个子记录

❌ 不适合使用 Interleave：
1. 子表独立查询频繁
   • 经常单独查询 Orders 表

2. 子表数据量远大于父表
   • 每个用户有 100 万条日志

3. 需要灵活调整表结构
   • Interleave 一旦设置，难以撤销

4. 多对多关系
   • 学生 ↔ 课程（需要中间表）
```

**实际案例：**

```
案例 1：电商系统（适合 Interleave）

父表：Users (100 万用户)
子表：Orders (每个用户平均 50 个订单)

没有 Interleave：
• 查询用户订单：150ms（跨 Region）
• 创建订单事务：200ms（2PC）

使用 Interleave：
• 查询用户订单：8ms（本地）
• 创建订单事务：15ms（本地事务）

性能提升：18 倍

案例 2：日志系统（不适合 Interleave）

父表：Users (100 万用户)
子表：Logs (每个用户 100 万条日志)

问题：
❌ 子表数据量过大（1000 亿条）
❌ 导致 Tablet 过大，无法分裂
❌ 热点问题严重

建议：不使用 Interleave，独立分片
```

**关键限制：**

| 限制 | 说明 | 影响 |
|------|------|------|
| **主键要求** | 子表主键必须以父表主键开头 | 设计灵活性降低 |
| **不可逆** | 一旦设置，无法直接撤销 | 需要重建表并迁移数据 |
| **级联删除** | 删除父记录会级联删除子记录 | 需要谨慎操作 |
| **Tablet 大小** | 子表数据过多会导致 Tablet 过大 | 影响分裂和负载均衡 |
| **深度限制** | 最多 7 层嵌套 | 复杂层级关系受限 |

**最佳实践：**

```
1. 评估数据量
   ✅ 每个父记录 < 1000 个子记录
   ❌ 避免子表数据量过大

2. 分析查询模式
   ✅ 经常 JOIN 父子表
   ❌ 子表独立查询频繁

3. 考虑未来扩展
   ✅ 数据增长可预测
   ❌ 不确定未来需求

4. 测试性能
   ✅ 在测试环境验证性能提升
   ❌ 不要盲目使用

5. 监控 Tablet 大小
   ✅ 定期检查 Tablet 大小
   ❌ 避免单个 Tablet 过大（> 8GB）
```

**物理存储示例：**

```
父表：Users
  id (PK)  name
  1        Alice
  2        Bob

子表：Orders (Interleave in Users)
  user_id (PK)  order_id (PK)  amount
  1             101            100
  1             102            200
  2             201            150

物理存储（同一 Tablet）：
┌─────────────────────────────────────────┐
│  Tablet_1                               │
│  ┌───────────────────────────────────┐  │
│  │ [Users: id=1, name='Alice']       │  │ ← 父记录
│  │ [Orders: user_id=1, order_id=101] │  │ ← 子记录（紧邻）
│  │ [Orders: user_id=1, order_id=102] │  │ ← 子记录（紧邻）
│  │ [Users: id=2, name='Bob']         │  │ ← 父记录
│  │ [Orders: user_id=2, order_id=201] │  │ ← 子记录（紧邻）
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘

优势：
✅ 父子数据物理相邻
✅ JOIN 查询无需跨网络
✅ 事务操作同一 Tablet，延迟低
```

---

##### 3. 事务与一致性

**事务类型：**

| 事务类型 | 特点 | 延迟 | 使用场景 |
|---------|------|------|---------|
| **读写事务（RW）** | 两阶段锁（2PL）+ commit-wait | 较高（需等待 TrueTime） | 需要修改数据 |
| **只读事务（RO）** | MVCC，无锁 | 低（无 commit-wait） | 只读查询 |
| **快照读（Snapshot）** | 指定时间戳读取历史数据 | 低 | 历史数据查询 |

**事务隔离级别：**

Spanner 的隔离级别与传统数据库不同：

| 数据库 | 支持的隔离级别 | 默认级别 | 说明 |
|--------|--------------|---------|------|
| **Spanner** | • Serializable（读写事务）<br>• Snapshot Isolation（只读事务） | Serializable | 只有 2 个级别，默认最强隔离 |
| **MySQL** | • Read Uncommitted<br>• Read Committed<br>• Repeatable Read<br>• Serializable | Repeatable Read | 4 个级别，可灵活选择 |
| **PostgreSQL** | • Read Uncommitted<br>• Read Committed<br>• Repeatable Read<br>• Serializable | Read Committed | 4 个级别，可灵活选择 |

**关键差异：**

```
Spanner 的设计哲学：
✅ 默认 Serializable（最强隔离级别）
✅ 全球 Serializable（跨区域）
❌ 不支持 Read Uncommitted / Read Committed
❌ 无法降低隔离级别换取性能

原因：
• Spanner 为全球分布式设计，必须保证强一致性
• TrueTime + commit-wait 保证全局顺序
• 降低隔离级别无法显著提升性能（瓶颈在网络）

传统数据库的灵活性：
✅ 可以降低隔离级别换取性能
✅ 适合单机场景的性能优化
❌ 无法保证跨区域一致性
```

**事务性能对比：**

| 场景                 | Spanner (Regional) | Spanner (Multi-Region)                   | Aurora/RDS | 说明                   |
| ------------------ | ------------------ | ---------------------------------------- | ---------- | -------------------- |
| **单表事务**           | 5-10ms             | 5-10ms（本地 Leader）<br>50-200ms（远程 Leader） | 5-10ms     | 性能相当                 |
| **跨表事务（同 Tablet）** | 10-20ms            | 10-20ms（本地）<br>50-200ms（远程）              | 10-20ms    | 性能相当                 |
| **跨表事务（跨 Tablet）** | 10-20ms            | 150-600ms（跨 Region 2PC）                  | 10-20ms    | Spanner 跨 Region 延迟高 |
| **跨区域事务**          | 不适用                | 150-600ms                                | 🔴 不支持     | Spanner 独有能力         |

**性能结论：**

```
单区域场景：
Aurora/RDS ≥ Spanner (Regional)
• 性能相当或 Aurora 略优
• Aurora 针对单区域优化
• Spanner 有 TrueTime 和 Paxos 开销

跨区域场景：
Spanner >>> Aurora/RDS
• Aurora/RDS 不支持跨区域强一致事务
• 需要应用层协调（复杂且不可靠）
• Spanner 原生支持，自动处理

选择建议：
✅ 单区域应用 → Aurora/RDS（性能更好，成本更低）
✅ 跨区域应用 → Spanner（唯一选择）
✅ 需要全球强一致性 → Spanner
✅ 预算有限 → Aurora/RDS
```

**全局一致性机制：Tablet 级锁 + 2PC + TrueTime + Paxos**

Spanner 不使用传统的全局锁，而是通过多层机制实现全局一致性：

1. 第一层：Tablet 级锁（**并发控制**）
2. 第二层：2PC 协调（**跨 Tablet 原子性**）
3. 第三层：TrueTime（**全局时间顺序**）
4. 第四层：Paxos 复制（Tablet 内部**副本一致性**）

```
┌─────────────────────────────────────────────────────────────┐
│           Spanner 全局一致性机制                              │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  第一层：Tablet 级锁（避免全局锁瓶颈）                        │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  每个 Tablet Leader 独立管理自己的锁                   │  │
│  │  • Tablet_1 Leader (美国): 锁 user_id=100            │  │
│  │  • Tablet_2 Leader (欧洲): 锁 order_id=500           │  │
│  │  • 不同 Tablet 的锁互不影响，可并行加锁                │  │
│  └───────────────────────────────────────────────────────┘  │
│                          ↓                                  │
│  第二层：2PC 协调（跨 Tablet 事务一致性）                     │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  协调器协调多个 Tablet Leader                          │  │
│  │  • Prepare: 所有 Tablet 确认持有锁                    │  │
│  │  • Commit: 所有 Tablet 应用写入                       │  │
│  │  • 原子性：要么全部成功，要么全部回滚                   │  │
│  └───────────────────────────────────────────────────────┘  │
│                          ↓                                  │
│  第三层：TrueTime（全局时间顺序）                            │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  通过全球时钟同步保证事务全局顺序                        │  │
│  │  • 分配全局唯一时间戳                                  │  │
│  │  • commit-wait: 等待不确定性消散                       │  │
│  │  • 保证外部一致性（比线性一致性更强）                    │  │
│  └───────────────────────────────────────────────────────┘  │
│                          ↓                                  │
│  第四层：Paxos 复制（Tablet 内部副本一致性）                  │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  每个 Tablet 通过 Paxos 复制到多数派副本                │  │
│  │  • Tablet_1: Leader → 5 副本（3/5 确认）              │  │
│  │  • Tablet_2: Leader → 5 副本（3/5 确认）              │  │
│  │  • 保证数据持久化和高可用                              │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**为什么不用全局锁？** 🔴🔴🔴

| 对比维度            | 全局锁方案               | Spanner 方案        |
| --------------- | ------------------- | ----------------- |
| **性能瓶颈**        | 🔴 全局锁是单点瓶颈         | ✅ Tablet 级锁，无单点   |
| **并发能力**        | 🔴 所有事务串行等待         | ✅ 不同 Tablet 并行加锁  |
| **跨 Region 延迟** | 🔴 每次都需跨 Region 获取锁 | ✅ 本地 Tablet 本地加锁  |
| **可扩展性**        | 🔴 无法水平扩展           | ✅ 随 Tablet 数量线性扩展 |
| **一致性保证**       | ✅ 通过全局锁             | ✅ 通过 TrueTime 时间戳 |

**实际案例对比：**

```
场景：两个并发事务

传统全局锁方案：
T1: 获取全局锁 → 修改 user_id=100 → 释放全局锁 (100ms)
T2: 等待全局锁 → 修改 order_id=500 → 释放全局锁 (100ms)
总延迟：200ms（串行）

Spanner 方案：
T1: Tablet_1 加锁 → 修改 user_id=100 → 释放锁 (10ms)
T2: Tablet_2 加锁 → 修改 order_id=500 → 释放锁 (10ms)
总延迟：10ms（并行，因为不同 Tablet）

关键：TrueTime 保证 T1 和 T2 的全局顺序，无需全局锁
```

**三层机制如何协同工作：**

```
步骤 1：Tablet 级加锁 🟢（第一层）
事务 T1 → Tablet_1 Leader: 🔒 加锁 user_id=100
事务 T1 → Tablet_2 Leader: 🔒 加锁 order_id=500

步骤 2：2PC 协调 🟢（第二层）
协调器 → Tablet_1 Leader: Prepare?
协调器 → Tablet_2 Leader: Prepare?
Tablet_1 Leader → 协调器: OK（持有锁）
Tablet_2 Leader → 协调器: OK（持有锁）

步骤 3：TrueTime 分配时间戳 🟢（第三层）
协调器获取 TT.now() = [10:00:00.000, 10:00:00.007]
选择时间戳 s = 10:00:00.007
commit-wait: 等待到 10:00:00.007

步骤 4：提交 + Paxos 复制 🟢
协调器 → Tablet_1 Leader: Commit with ts=10:00:00.007
协调器 → Tablet_2 Leader: Commit with ts=10:00:00.007
    ↓
Tablet_1 Leader 应用写入 → Paxos 复制到 5 个副本（3/5 确认）
Tablet_2 Leader 应用写入 → Paxos 复制到 5 个副本（3/5 确认）
    ↓
🔓 释放锁

结果：
• 无需全局锁
• 不同 Tablet 并行加锁
• TrueTime 保证全局顺序
• Paxos 保证副本一致性
```

**读写事务流程：**

**锁的范围说明：**
- ⚠️ Spanner 没有全局锁（避免性能瓶颈）
- 🔒 锁是 Tablet 级别的：**每个 Tablet Leader 管理自己的锁**
- 🌍 跨 Tablet 事务：**通过 2PC 协调多个 Tablet 的锁**
- ⏰ 全局一致性：通过 TrueTime + commit-wait 保证，而非全局锁

```
1. 事务开始
   - 客户端发起事务
   - 协调器分配事务 ID

2. 读阶段（两阶段锁 - 2PL）
   - 🔒 对读取的行加读锁（Shared Lock）← 在涉及的 Tablet Leader 上加锁
   - 🔒 对修改的行加写锁（Exclusive Lock）← 在涉及的 Tablet Leader 上加锁
   - 锁在事务提交前不释放

3. 写阶段
   - 缓存写操作（不立即写入）← 🟢 在 Tablet Leader 节点内存中缓存
   - 检测冲突（与其他事务的锁）

4. 提交阶段（两阶段提交 - 2PC）
   - Prepare：协调器向所有参与者发送 Prepare
   - ⚠️ 参与者确认持有锁并返回 Prepared（锁已在步骤 2 获取）
   - Commit：协调器获取 TT.now()，选择时间戳 s
   - commit-wait：等待 TT.after(s) = true
   - 协调器向所有参与者发送 Commit
   - ✅ 参与者应用写入（此时触发 Paxos 复制）
   - ✅ 每个 Tablet Leader 通过 Paxos 复制到多数派副本（3/5）
   - ✅ Paxos 复制完成后，🔓 释放锁（释放步骤 2 的锁）

5. 事务完成
   - 返回客户端成功
```

**时序说明：**

```
步骤 1：2PC Prepare
协调器 → Tablet Leaders: Prepare?
Tablet Leaders → 协调器: OK（Paxos 未参与）

步骤 2：commit-wait
等待 TrueTime 不确定性消散（1-7ms）

步骤 3：2PC Commit + Paxos 复制（并行）
协调器 → Tablet Leaders: Commit
  ↓
Tablet_1 Leader → Paxos Group 1（5 副本）← Paxos 开始
Tablet_2 Leader → Paxos Group 2（5 副本）← Paxos 开始
  ↓
等待 Paxos 多数派（3/5）确认
  ↓
释放锁

步骤 4：返回客户端
事务成功
```

**2PC 和 Paxos 的两层协议关系：**

```
事务层（2PC）：协调多个 Tablet 的事务
    ↓
Tablet_1 Leader ← 2PC Prepare → Tablet_2 Leader
    ↓                               ↓
复制层（Paxos）：每个 Tablet 内部副本复制
    ↓                               ↓
Paxos Group 1                  Paxos Group 2
(5 副本)                       (5 副本)
```

**锁的机制详解：**

```
锁的范围（Tablet 级别，非全局锁）：

单 Tablet 事务：
┌─────────────────────────────┐
│  Tablet_1 Leader (美国)      │
│  🔒 锁：user_id=100 的行     │
│  范围：只在这个 Leader 上     │
└─────────────────────────────┘

跨 Tablet 事务：
┌─────────────────────────────┐
│  Tablet_1 Leader (美国)      │
│  🔒 锁：user_id=100 的行     │
└─────────────────────────────┘
         ↓ 2PC 协调
┌─────────────────────────────┐
│  Tablet_2 Leader (欧洲)      │
│  🔒 锁：order_id=500 的行    │
└─────────────────────────────┘

关键点：
• 每个 Tablet Leader 独立管理自己的锁
• 不同 Tablet 的锁互不影响
• 2PC 协调多个 Tablet 的锁，而非使用全局锁
• TrueTime + commit-wait 保证全局一致性

为什么不用全局锁？
✅ 避免单点瓶颈：全局锁会成为性能瓶颈
✅ 提高并发：不同 Tablet 可以并行加锁
✅ 降低延迟：无需跨 Region 获取全局锁
✅ TrueTime 替代：通过时间戳保证全局顺序
```

**跨 Tablet 事务延迟分析：**

```
单 Region 跨 Tablet：
- 2PC Prepare：< 5ms（本地网络）
- Paxos 复制：< 5ms（同 Region）
- 2PC Commit：< 5ms（本地网络）
- commit-wait：1-7ms（TrueTime）
总延迟：10-20ms

跨 Region 跨 Tablet：
- 2PC Prepare：50-200ms（跨 Region 网络）
- Paxos 复制：50-200ms（跨 Region 副本）
- 2PC Commit：50-200ms（跨 Region 网络）
- commit-wait：1-7ms（TrueTime）
总延迟：150-600ms

关键点：
• 2PC 延迟取决于 Tablet Leader 位置
• Paxos 延迟取决于副本分布（Multi-Region vs Regional）
• 网络中断时 Paxos 自动重试，2PC 会超时回滚
• 延迟 = max(各 Tablet 延迟) + 2PC 开销 + commit-wait
```

**只读事务流程：**

```
1. 事务开始
   - 客户端指定只读事务
   - 协调器分配时间戳 s = TT.now().latest

2. 读取数据
   - 使用 MVCC 读取时间戳 s 的快照
   - 无需加锁
   - 可以从任意副本读取（不一定是 Leader）

3. 事务完成
   - 返回结果
   - 无需 commit-wait

优势：
✅ 无锁，高并发
✅ 无 commit-wait，低延迟
✅ 可从本地副本读取
✅ 全局一致性快照
```

**MVCC 实现：**

```
每行数据维护多个版本：
Key: user_id=1
Versions:
  [ts=100, name='Alice', age=25]  ← 最新版本
  [ts=90,  name='Alice', age=24]
  [ts=80,  name='Alice', age=23]

只读事务（ts=95）：
- 读取 ts <= 95 的最新版本
- 结果：[ts=90, name='Alice', age=24]

垃圾回收：
- 定期清理旧版本
- 保留最近的版本（支持 Time Travel）
- 配置保留时间（如 7 天）
```

**Spanner MVCC vs 传统 MVCC：**

| 维度           | Spanner MVCC          | 传统 MVCC (MySQL/PostgreSQL) |
| ------------ | --------------------- | -------------------------- |
| **版本标识**     | TrueTime 时间戳（全球唯一）    | 事务 ID（本地递增）                |
| **时间戳来源**    | GPS + 原子钟（硬件）         | 数据库内部计数器（软件）               |
| **MVCC 作用域** | ⚠️ **Tablet 级别**（分布式） | ⚠️ **实例级别**（单机）            |
| **可见性判断**    | 简单：ts <= 读时间戳         | 复杂：需判断事务状态（活跃/提交/回滚）       |
| **跨节点一致性**   | ✅ 全球一致（外部一致性）         | 🔴 单机一致（无法跨节点）             |
| **时间旅行**     | ✅ 精确到毫秒（任意历史时间点）      | ⚠️ 有限支持（依赖 Undo Log 保留）    |
| **垃圾回收**     | 基于时间（如保留 7 天）         | 基于事务完成（Undo Log 清理）        |
| **存储位置**     | 每个版本独立存储              | Undo Log（回滚段）              |

**⚠️ 关键差异：MVCC 作用域级别**

这是 Spanner 与传统数据库最重要的架构差异之一：

```
传统数据库（MySQL/PostgreSQL/Aurora）：
┌─────────────────────────────────────────┐
│  Primary 实例（单机）                     │
│  ┌───────────────────────────────────┐  │
│  │  MVCC 在整个实例级别管理            │  │
│  │  ├─ 全局事务 ID 计数器              │  │
│  │  ├─ 全局 Undo Log                  │  │
│  │  └─ 所有表共享 MVCC 机制            │  │
│  └───────────────────────────────────┘  │
│                                         │
│  所有数据的 MVCC 版本在同一个实例中      │
└─────────────────────────────────────────┘

Spanner（分布式）：
┌─────────────────────────────────────────┐
│  Tablet_1 (user_id [0-50M])             │
│  ┌───────────────────────────────────┐  │
│  │  独立的 MVCC 版本管理               │  │
│  │  ├─ user_id=1: [ts=100, ts=90]    │  │
│  │  └─ user_id=2: [ts=105]           │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
┌─────────────────────────────────────────┐
│  Tablet_2 (user_id [50M-100M])          │
│  ┌───────────────────────────────────┐  │
│  │  独立的 MVCC 版本管理               │  │
│  │  ├─ user_id=60M: [ts=110]         │  │
│  │  └─ user_id=70M: [ts=115, ts=105] │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘

每个 Tablet 独立管理 MVCC，但时间戳全球统一
```

**MVCC 作用域对比：**

| 数据库类型                | MVCC 作用域     | 架构   | 扩展性               |
| -------------------- | ------------ | ---- | ----------------- |
| **MySQL/PostgreSQL** | 实例级别         | 单机   | 🔴 无法水平扩展 MVCC    |
| **Aurora**           | 实例级别         | 单主多读 | 🔴 写入无法扩展         |
| **RDS Read Replica** | 实例级别（每个副本独立） | 主从复制 | ⚠️ 副本有延迟，MVCC 不一致 |
| **Spanner**          | Tablet 级别    | 分布式  | ✅ 随 Tablet 水平扩展   |

**为什么这个差异很重要？**

```
场景：1 亿用户数据

传统数据库（实例级别 MVCC）：
┌─────────────────────────────────────┐
│  Primary 实例                        │
│  • 1 亿用户的所有 MVCC 版本            │
│  • 单个 Undo Log                     │
│  • 单点瓶颈                          │
│  • 无法水平扩展                       │
└─────────────────────────────────────┘

问题：
❌ Undo Log 过大（几十 GB）
❌ 垃圾回收压力大
❌ 长事务影响所有表
❌ 无法分散负载

Spanner（Tablet 级别 MVCC）：
┌─────────────────────────────────────┐
│  Tablet_1: 2500 万用户               │
│  • 独立的 MVCC 版本                   │
│  • 独立的垃圾回收                     │
└─────────────────────────────────────┘
┌─────────────────────────────────────┐
│  Tablet_2: 2500 万用户               │
│  • 独立的 MVCC 版本                   │
│  • 独立的垃圾回收                     │
└─────────────────────────────────────┘
┌─────────────────────────────────────┐
│  Tablet_3: 2500 万用户               │
│  • 独立的 MVCC 版本                   │
│  • 独立的垃圾回收                     │
└─────────────────────────────────────┘
┌─────────────────────────────────────┐
│  Tablet_4: 2500 万用户               │
│  • 独立的 MVCC 版本                   │
│  • 独立的垃圾回收                     │
└─────────────────────────────────────┘

优势：
✅ MVCC 版本分散存储
✅ 垃圾回收并行执行
✅ 长事务只影响相关 Tablet
✅ 随 Tablet 数量线性扩展
```

**实际影响：**

| 操作 | 传统数据库（实例级别） | Spanner（Tablet 级别） |
|------|---------------------|---------------------|
| **长事务** | 影响整个实例的所有表 | 只影响涉及的 Tablet |
| **垃圾回收** | 单线程，阻塞所有表 | 并行，每个 Tablet 独立 |
| **MVCC 版本数** | 所有表累加（可能几十 GB） | 分散在各 Tablet（每个几百 MB） |
| **扩展性** | 无法扩展 | 随 Tablet 线性扩展 |
| **热点问题** | 影响整个实例 | 只影响单个 Tablet |

**关键差异详解：**

```
传统 MVCC（MySQL InnoDB）：
┌─────────────────────────────────────┐
│  当前版本（数据页）                   │
│  user_id=1, name='Alice', age=25    │
│  txn_id=100                         │
└─────────────────────────────────────┘
         ↓ 指向 Undo Log
┌─────────────────────────────────────┐
│  Undo Log（回滚段）                  │
│  ├─ [txn_id=90, age=24]             │
│  └─ [txn_id=80, age=23]             │
└─────────────────────────────────────┘

可见性判断：
• 读取事务 T1（txn_id=95）
• 检查当前版本 txn_id=100 > 95 → 不可见
• 沿 Undo Log 链查找 txn_id <= 95
• 找到 txn_id=90 → 可见

Spanner MVCC：
┌─────────────────────────────────────┐
│  所有版本（独立存储）                 │
│  ├─ [ts=100, name='Alice', age=25]  │
│  ├─ [ts=90,  name='Alice', age=24]  │
│  └─ [ts=80,  name='Alice', age=23]  │
└─────────────────────────────────────┘

可见性判断：
• 读取事务 T1（ts=95）
• 直接查找 ts <= 95 的最新版本
• 找到 ts=90 → 返回

优势：
✅ 无需复杂的事务状态判断
✅ 全球时间戳保证跨节点一致性
✅ 支持精确的时间旅行查询
```

**时间旅行查询示例：**

```sql
-- Spanner 支持精确到毫秒的历史查询
SELECT * FROM users 
FOR SYSTEM_TIME AS OF '2024-01-01 10:00:00.123'
WHERE user_id = 1;

-- 传统数据库（MySQL）
-- ❌ 不支持或需要额外配置（如 Flashback Query）
```

##### 4. 延迟特征与优化

**不同场景的写入延迟对比：**

| 场景 | 配置 | 延迟范围 | 主要开销 |
|------|------|---------|---------|
| **单 Tablet 单 Region** | Regional | 5-10ms | Paxos 复制（同 Region）+ commit-wait（1-7ms） |
| **跨 Tablet 单 Region** | Regional | 10-20ms | 2PC 协调 + Paxos 复制 + commit-wait |
| **单 Tablet 跨 Region** | Multi-Region | 50-200ms | Paxos 复制（跨 Region）+ commit-wait |
| **跨 Tablet 跨 Region** | Multi-Region | **150-600ms** | 2PC 协调（跨 Region）+ Paxos 复制 + commit-wait |

**延迟组成详解：**

```
单 Region 写入（5-10ms）：
├─ 2PL 加锁：< 1ms
├─ Paxos 复制（同 Region 3/5 副本）：2-5ms
├─ commit-wait（TrueTime 不确定性）：1-7ms
└─ 释放锁：< 1ms

跨 Region 写入（150-600ms）：
├─ 2PL 加锁：< 1ms
├─ 2PC Prepare（跨 Region）：50-200ms  ← 主要开销
├─ Paxos 复制（跨 Region 3/5 副本）：50-200ms  ← 主要开销
├─ 2PC Commit（跨 Region）：50-200ms  ← 主要开销
├─ commit-wait（TrueTime 不确定性）：1-7ms
└─ 释放锁：< 1ms

关键点：
• 单 Region 延迟主要是 commit-wait（1-7ms）
• 跨 Region 延迟主要是网络传输（50-200ms × 3 次往返）
• 2PC 需要 3 次跨 Region 网络往返（Prepare + Commit + Ack）
• Paxos 需要多数派（3/5）确认，跨 Region 时延迟显著
```

**延迟影响因素：**

| 因素                   | 影响                 | 优化方法                        |
| -------------------- | ------------------ | --------------------------- |
| **Tablet Leader 位置** | Leader 在远程区域 → 高延迟 | 数据局部性配置，让 Leader 靠近用户       |
| **TrueTime 不确定性**    | commit-wait 等待时间   | 无法优化（硬件限制）                  |
| **跨 Tablet 事务**      | 2PC 协调开销           | 使用 Interleave 减少跨 Tablet 查询 |
| **网络延迟**             | 跨区域通信              | 多区域部署，就近访问                  |
| **Paxos 复制**         | 多数派确认              | 副本放置优化                      |

**延迟监测与分析：**

```
Cloud Monitoring 提供的延迟指标：
1. API 延迟（端到端）
   - P50 / P95 / P99
   - 按 API 方法分类（Query / DML / Commit）

2. 延迟分解：
   ┌─────────────────────────────────┐
   │ 客户端往返延迟                    │
   │  - 网络延迟                      │
   │  - 客户端处理                    │
   ├─────────────────────────────────┤
   │ GFE 延迟                         │
   │  - 负载均衡                      │
   │  - TLS 终止                      │
   ├─────────────────────────────────┤
   │ Spanner API 前端延迟             │
   │  - SQL 解析                      │
   │  - 查询优化                      │
   ├─────────────────────────────────┤
   │ Spanner 后端延迟                 │
   │  - Tablet 查找                   │
   │  - 锁等待                        │
   │  - Paxos 复制                    │
   │  - commit-wait                  │
   └─────────────────────────────────┘

3. 热点检测：
   - CPU 利用率（按 Split）
   - QPS（按 Split）
   - 扫描行数

4. 慢查询分析：
   - 查询计划
   - 锁等待时间
   - 扫描行数
```

**优化建议：**

```
1. 主键设计：
   ❌ 避免：单调递增 ID（热点）
   ✅ 推荐：UUID / 哈希前缀 / 复合主键

2. 数据局部性：
   ✅ 使用 Interleave 减少跨 Tablet 查询
   ✅ 配置 Leader 位置靠近用户

3. 查询优化：
   ✅ 使用二级索引
   ✅ 避免全表扫描
   ✅ 使用只读事务（无 commit-wait）

4. 事务优化：
   ✅ 减少事务范围（涉及的行数）
   ✅ 避免长事务（持有锁时间）
   ✅ 批量操作使用 Mutation API

5. 预分裂：
   ✅ 建表时指定分裂点
   ✅ 避免初期热点
```

---

##### Spanner 实战场景

##### 典型应用场景

| 场景 | 业务描述 | 数据量 | 并发量 | 关键指标 | 技术要点 |
|:---|:---|:---|:---|:---|:---|
| **全球支付系统** | 跨国转账、支付处理 | PB 级 | 10 万+ TPS | 强一致性、低延迟 | Interleave 父子表、数据局部性配置、只读事务优化 |
| **全球游戏排行榜** | 实时排名、跨区域同步 | TB 级 | 5 万+ QPS | 全球一致性、实时性 | 复合主键避免热点、二级索引、快照读 |
| **金融交易系统** | 股票交易、订单撮合 | TB 级 | 1 万+ TPS | ACID、外部一致性 | 读写事务、2PC、commit-wait |
| **全球供应链** | 库存管理、订单跟踪 | TB 级 | 1 万+ QPS | 多区域写入、强一致性 | 多主架构、Paxos 复制、预分裂 |
| **广告计费系统** | 点击计费、实时扣费 | PB 级 | 10 万+ QPS | 精确计费、不丢失 | 批量 Mutation、只读事务、MVCC |
| **SaaS 多租户平台** | 全球客户数据隔离 | PB 级 | 5 万+ QPS | 数据隔离、合规性 | 按租户 ID 分片、Interleave、加密 |

##### 场景深入分析

**场景 1：全球支付系统（如 Stripe、PayPal）**

```
业务需求：
- 用户在任何区域发起转账
- 全球强一致性（不能重复扣款）
- 低延迟（用户体验）
- 高可用（99.999% SLA）

数据模型：
-- 用户表
CREATE TABLE Users (
  user_id STRING(36) NOT NULL,  -- UUID
  region STRING(10),
  balance NUMERIC,
  created_at TIMESTAMP,
) PRIMARY KEY (user_id);

-- 交易表（Interleave in Users）
CREATE TABLE Transactions (
  user_id STRING(36) NOT NULL,
  transaction_id STRING(36) NOT NULL,
  amount NUMERIC,
  status STRING(10),
  created_at TIMESTAMP,
) PRIMARY KEY (user_id, transaction_id),
  INTERLEAVE IN PARENT Users ON DELETE CASCADE;

技术要点：
1. Interleave：用户与交易数据物理相邻，JOIN 无跨网络
2. 数据局部性：配置 Leader 位置
   - 美国用户 → Leader 在 us-central1
   - 欧洲用户 → Leader 在 europe-west1
   - 亚洲用户 → Leader 在 asia-east1
3. 读写事务：保证扣款原子性
   BEGIN;
   UPDATE Users SET balance = balance - 100 WHERE user_id = 'xxx';
   INSERT INTO Transactions (...) VALUES (...);
   COMMIT;
4. 只读事务：查询余额（无 commit-wait）
   SELECT balance FROM Users WHERE user_id = 'xxx';

性能优化：
- 主键使用 UUID（避免热点）
- 预分裂：按 region 预分裂
- 批量操作：使用 Mutation API
- 监控：P99 延迟 < 100ms

实际效果（基于 Google Cloud 官方数据）：

本地区域写入（Leader 在本地）：
- P50: 5-10ms
- P95: 15-25ms
- P99: 30-50ms
- 包含：网络往返 + SQL 解析 + Paxos 复制 + commit-wait

跨区域写入（Leader 在远程）：
- P50: 50-100ms（美国 ↔ 欧洲）
- P50: 100-200ms（美国 ↔ 亚洲）
- P99: 200-500ms
- 主要开销：跨区域网络延迟

只读事务（本地副本）：
- P50: 1-3ms
- P95: 5-10ms
- P99: 10-20ms
- 无 commit-wait，可从本地副本读取

影响因素：
- Tablet Leader 位置（最关键）
- TrueTime 不确定性（1-5ms）
- 网络延迟（区域间差异大）
- 查询复杂度（扫描行数）
- 事务范围（涉及的 Tablet 数量）

可用性：99.999%
```

**场景 2：全球游戏排行榜（如 PUBG、王者荣耀）**

```
业务需求：
- 全球玩家实时排名
- 跨区域一致性（不能出现排名错乱）
- 高并发读取（百万玩家查询）
- 定期更新（每局游戏结束）

数据模型：
-- 玩家表
CREATE TABLE Players (
  player_id STRING(36) NOT NULL,
  region STRING(10),
  username STRING(50),
  total_score INT64,
  rank INT64,
  updated_at TIMESTAMP,
) PRIMARY KEY (player_id);

-- 排行榜索引
CREATE INDEX idx_rank ON Players(total_score DESC, player_id);

技术要点：
1. 复合主键避免热点：
   - ❌ 错误：PRIMARY KEY (rank)  -- 单调递增，热点
   - ✅ 正确：PRIMARY KEY (player_id)  -- UUID，分散
2. 二级索引：按 total_score 排序
3. 只读事务：查询排行榜（高并发）
   SELECT player_id, username, total_score, rank
   FROM Players
   ORDER BY total_score DESC
   LIMIT 100;
4. 快照读：查询历史排名
   SELECT * FROM Players
   FOR SYSTEM_TIME AS OF TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR);

性能优化：
- 读写分离：排名更新用读写事务，查询用只读事务
- 缓存：热门排行榜缓存到 Redis（1 分钟 TTL）
- 分页：LIMIT + OFFSET 避免全表扫描
- 预聚合：定期计算 rank 字段

实际效果：
- 排名更新：50-100ms
- 排行榜查询：10-20ms（只读事务）
- 并发：10 万+ QPS
- 一致性：全球实时同步
```

**场景 3：金融交易系统（如证券交易所）**

```
业务需求：
- 订单撮合（买卖匹配）
- 严格 ACID（不能丢单、重复成交）
- 外部一致性（交易顺序全球一致）
- 审计追溯（监管要求）

数据模型：
-- 订单表
CREATE TABLE Orders (
  order_id STRING(36) NOT NULL,
  user_id STRING(36),
  symbol STRING(10),  -- 股票代码
  type STRING(4),     -- BUY/SELL
  price NUMERIC,
  quantity INT64,
  status STRING(10),  -- PENDING/FILLED/CANCELLED
  created_at TIMESTAMP,
) PRIMARY KEY (order_id);

-- 成交表
CREATE TABLE Trades (
  trade_id STRING(36) NOT NULL,
  buy_order_id STRING(36),
  sell_order_id STRING(36),
  price NUMERIC,
  quantity INT64,
  created_at TIMESTAMP,
) PRIMARY KEY (trade_id);

技术要点：
1. 读写事务：订单撮合
   BEGIN;
   -- 查找匹配订单
   SELECT * FROM Orders
   WHERE symbol = 'AAPL' AND type = 'SELL' AND price <= 150
   ORDER BY price ASC, created_at ASC
   LIMIT 1
   FOR UPDATE;  -- 加锁
   
   -- 更新订单状态
   UPDATE Orders SET status = 'FILLED' WHERE order_id IN (...);
   
   -- 插入成交记录
   INSERT INTO Trades (...) VALUES (...);
   COMMIT;

2. 外部一致性：TrueTime 保证交易顺序
   - 交易 A 在交易 B 前提交 → ts(A) < ts(B)
   - 全球任意节点查询都看到相同顺序

3. MVCC：历史数据查询（审计）
   SELECT * FROM Trades
   FOR SYSTEM_TIME AS OF '2025-01-01 00:00:00';

4. 二级索引：加速订单查询
   CREATE INDEX idx_symbol_price ON Orders(symbol, price, created_at);

性能优化：
- 热点优化：按 symbol 预分裂
- 批量处理：批量撮合订单
- 只读副本：审计查询走只读事务
- 监控：锁等待时间、commit-wait 时间

实际效果：
- 订单撮合：20-50ms
- 查询订单：5-10ms
- 吞吐量：1 万+ TPS
- 一致性：外部一致性保证
```

##### 最佳实践

**1. 主键设计**

```
❌ 避免的模式：
- 单调递增 ID（热点）
  PRIMARY KEY (id)  -- 所有写入集中在最后一个 Split

- 时间戳前缀（热点）
  PRIMARY KEY (TIMESTAMP, id)  -- 同一时间写入集中

✅ 推荐的模式：
- UUID（分散）
  PRIMARY KEY (UUID())

- 哈希前缀（分散）
  PRIMARY KEY (FARM_FINGERPRINT(user_id), user_id)

- 复合主键（业务相关）
  PRIMARY KEY (region, user_id)  -- 按 region 分片

- 反转时间戳（分散）
  PRIMARY KEY (MAX_TIMESTAMP - TIMESTAMP, id)
```

**2. Interleave 使用**

```
✅ 适合 Interleave：
- 父子关系明确（用户-订单、博客-评论）
- 经常 JOIN 查询
- 事务操作同时涉及父子表

❌ 不适合 Interleave：
- 子表独立查询频繁
- 子表数据量远大于父表（不平衡）
- 需要灵活调整表结构

示例：
-- 适合
CREATE TABLE Users (...) PRIMARY KEY (user_id);
CREATE TABLE Orders (
  user_id STRING(36),
  order_id STRING(36),
  ...
) PRIMARY KEY (user_id, order_id),
  INTERLEAVE IN PARENT Users;

-- 不适合
CREATE TABLE Logs (...)  -- 日志表数据量巨大，不适合 Interleave
```

**3. 事务优化**

```
✅ 使用只读事务（无 commit-wait）：
-- 查询操作
SELECT * FROM Users WHERE user_id = 'xxx';

✅ 减少事务范围：
-- ❌ 错误：大事务
BEGIN;
UPDATE Users SET ... WHERE region = 'US';  -- 影响百万行
COMMIT;

-- ✅ 正确：小事务
BEGIN;
UPDATE Users SET ... WHERE user_id = 'xxx';  -- 影响单行
COMMIT;

✅ 批量操作使用 Mutation API：
-- 比事务更高效
mutations = [
  spanner.insert('Users', columns, values),
  spanner.update('Orders', columns, values),
]
database.batch_write(mutations)

✅ 避免长事务：
-- 持有锁时间短
BEGIN;
SELECT ... FOR UPDATE;  -- 加锁
-- 快速处理
UPDATE ...;
COMMIT;  -- 尽快释放锁
```

**4. 查询优化**

```
✅ 使用二级索引：
CREATE INDEX idx_email ON Users(email);
SELECT * FROM Users WHERE email = 'xxx';  -- 使用索引

✅ 避免全表扫描：
-- ❌ 错误
SELECT * FROM Users;  -- 扫描所有 Split

-- ✅ 正确
SELECT * FROM Users WHERE region = 'US' LIMIT 1000;

✅ 使用 EXPLAIN 分析：
EXPLAIN SELECT * FROM Users WHERE email = 'xxx';
-- 查看是否使用索引、扫描行数

✅ 分页查询：
SELECT * FROM Users
WHERE user_id > 'last_user_id'
ORDER BY user_id
LIMIT 100;  -- 避免 OFFSET（性能差）
```

**5. 监控与告警**

```
关键指标：
1. 延迟指标
   - API 延迟 P50/P95/P99
   - 按操作类型（Query/DML/Commit）

2. CPU 利用率
   - 高优先级 CPU：< 65%（推荐）
   - 低优先级 CPU：< 90%

3. 存储利用率
   - < 80%（推荐）

4. 热点检测
   - Split CPU 利用率
   - Split QPS
   - 锁等待时间

5. 慢查询
   - 扫描行数 > 10000
   - 延迟 > 1s

告警阈值：
- P99 延迟 > 100ms
- CPU 利用率 > 65%
- 热点 Split CPU > 80%
- 慢查询数量 > 10/min
```

---

##### Spanner 与其他数据库对比

**Spanner 的核心优势（独特价值）：**

Spanner 是唯一同时满足以下四个特性的数据库：

```
┌─────────────────────────────────────────────────────────────┐
│              Spanner 的四大核心优势                            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  1️⃣ 全球强一致性（外部一致性）                               │
│     • 比线性一致性更强                                       │
│     • 跨区域事务全局有序                                     │
│     • 无需应用层处理冲突                                     │
│     ✅ 优势：金融交易、库存管理等场景的数据准确性保证          │
│                                                             │
│  2️⃣ 全球分布 + 低延迟读写                                    │
│     • 多区域多主写入（无需转发）                              │
│     • 数据局部性优化（Leader 靠近用户）                       │
│     • 读取延迟 < 10ms（本地 Leader）                         │
│     ✅ 优势：全球用户都能获得低延迟体验                        │
│                                                             │
│  3️⃣ 水平扩展 + SQL 支持                                      │
│     • 自动分片（Tablet）                                     │
│     • 标准 SQL 查询（JOIN、事务）                            │
│     • 无需应用层分库分表                                     │
│     ✅ 优势：既有 NoSQL 的扩展性，又有 SQL 的易用性           │
│                                                             │
│  4️⃣ 99.999% SLA（5 个 9）                                   │
│     • Paxos 自动故障转移（秒级）                             │
│     • 跨区域副本（区域故障不影响）                            │
│     • 无需手动运维                                          │
│     ✅ 优势：关键业务的高可用保证                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**与其他数据库的核心差异：**

| 数据库类型                              | 缺少的 Spanner 优势                           | 适用场景差异           |
| ---------------------------------- | ---------------------------------------- | ---------------- |
| **传统 RDBMS**<br>(MySQL/PostgreSQL) | 🔴 无法水平扩展<br>🔴 单区域部署<br>🔴 可用性 < 99.99% | 单区域应用，数据量 < TB 级 |
| **NewSQL**<br>(CockroachDB/TiDB)   | ⚠️ 无 TrueTime（一致性较弱）<br>⚠️ 跨区域延迟更高       | 多区域应用，但对延迟要求不高   |
| **NoSQL**<br>(DynamoDB/Cassandra)  | 🔴 最终一致性<br>🔴 无 SQL 支持<br>🔴 无全局事务      | 高吞吐写入，可接受最终一致性   |
| **HTAP**<br>(TiDB)                 | ⚠️ 无 TrueTime<br>✅ 有 OLAP 能力             | 需要实时分析的场景        |

**Spanner 的独特技术：**

```
TrueTime（全球唯一）：
├─ GPS + 原子钟硬件
├─ 不确定性 < 7ms
├─ 实现外部一致性
└─ 无需应用层时钟同步

其他数据库的替代方案：
├─ CockroachDB/YugabyteDB: HLC（混合逻辑时钟）
│   └─ 软件实现，精度较低，只能保证线性一致性
├─ TiDB: TSO（时间戳服务）
│   └─ 中心化服务，可能成为瓶颈
└─ 传统数据库: 无全球时钟
    └─ 无法保证跨区域一致性
```

**横向对比（同类产品）：**

| 优势维度        | Spanner        | CockroachDB  | TiDB       | Aurora             |
| ----------- | -------------- | ------------ | ---------- | ------------------ |
| **全球强一致性**  | ✅ 外部一致性        | ⚠️ 线性一致性     | ⚠️ 线性一致性   | 🔴 单区域             |
| **跨区域延迟**   | ✅ 最优（TrueTime） | ⚠️ 较高（HLC）   | ⚠️ 较高（TSO） | 🔴 不支持             |
| **SQL 兼容性** | ⚠️ 类 SQL       | ✅ PostgreSQL | ✅ MySQL    | ✅ MySQL/PostgreSQL |
| **托管服务**    | ✅ GCP 原生       | ⚠️ 第三方       | ⚠️ 第三方     | ✅ AWS 原生           |
| **成本**      | ❌ 最高           | ⚠️ 中等        | ⚠️ 中等      | ⚠️ 中等              |

**纵向对比（不同类型）：**

| 特性          | Spanner  | 传统 RDBMS | NoSQL   | HTAP     |
| ----------- | -------- | -------- | ------- | -------- |
| **扩展性**     | ✅ 水平扩展   | 🔴 垂直扩展  | ✅ 水平扩展  | ✅ 水平扩展   |
| **一致性**     | ✅ 全球强一致  | ✅ 单机强一致  | 🔴 最终一致 | ✅ 强一致    |
| **SQL 支持**  | ✅ 标准 SQL | ✅ 完全兼容   | 🔴 有限支持 | ✅ 标准 SQL |
| **全球分布**    | ✅ 原生支持   | 🔴 需要中间件 | ✅ 原生支持  | ⚠️ 部分支持  |
| **OLAP 能力** | 🔴 无     | 🔴 无     | 🔴 无    | ✅ 有      |

##### Spanner vs 传统关系数据库

| 维度 | Spanner | MySQL/PostgreSQL | 说明 |
|------|---------|------------------|------|
| **扩展性** | 水平扩展（自动分片） | 垂直扩展（单机） | Spanner 可扩展到 PB 级 |
| **一致性** | 全球强一致性（外部一致性） | 单机 ACID | Spanner 跨区域强一致 |
| **可用性** | 99.999%（5 个 9） | 99.95%（主从） | Spanner 自动故障转移 |
| **延迟** | 取决于 Leader 位置（几毫秒到几百毫秒） | 毫秒级（单机） | Spanner 跨区域延迟较高 |
| **成本** | 高（按节点 + 存储计费） | 低（按实例计费） | Spanner 适合大规模场景 |
| **SQL 兼容性** | 类 SQL（有差异） | 完全兼容 | Spanner 不支持部分特性 |
| **事务** | 分布式事务（2PC + TrueTime） | 本地事务（2PL + MVCC） | Spanner 全球事务 |
| **复制** | Paxos（同步复制） | 主从复制（异步/半同步） | Spanner 强一致复制 |

##### Spanner vs NewSQL 数据库

| 维度 | Spanner | CockroachDB | TiDB | YugabyteDB |
|------|---------|-------------|------|------------|
| **时钟同步** | TrueTime（GPS + 原子钟） | HLC（混合逻辑时钟） | TSO（时间戳服务） | HLC（混合逻辑时钟） |
| **一致性** | 外部一致性 | 线性一致性 | 线性一致性 | 线性一致性 |
| **复制协议** | Paxos | Raft | Raft | Raft |
| **部署** | 托管服务（GCP） | 自部署 / 托管 | 自部署 / 托管 | 自部署 / 托管 |
| **成本** | 高 | 中 | 中 | 中 |
| **SQL 兼容性** | 类 SQL | PostgreSQL 兼容 | MySQL 兼容 | PostgreSQL 兼容 |
| **特色** | TrueTime 全球时钟 | 开源，多云 | HTAP（TiFlash） | 多 API（SQL + Cassandra） |

##### Spanner vs NoSQL 数据库

| 维度 | Spanner | DynamoDB | Cassandra | MongoDB |
|------|---------|----------|-----------|---------|
| **数据模型** | 关系型（SQL） | KV + 文档 | 宽列 | 文档 |
| **一致性** | 强一致性 | 最终一致性（可配置强一致） | 最终一致性（可调） | 最终一致性（可配置） |
| **事务** | 全局 ACID | 单项 ACID | 轻量事务 | 多文档事务 |
| **查询** | SQL | KV 查询 + GSI | CQL | MQL |
| **扩展性** | 自动分片 | 自动分片 | 手动分片 | 自动分片 |
| **适用场景** | 全球事务应用 | 高吞吐 KV | 高可用写入 | 灵活文档存储 |

##### 选择建议

```
选择 Spanner：
✅ 全球分布式应用（多区域写入）
✅ 需要强一致性（金融、支付）
✅ 需要 SQL 查询（复杂关联）
✅ 需要 99.999% SLA
✅ 预算充足

选择 MySQL/PostgreSQL：
✅ 单区域应用
✅ 预算有限
✅ 需要完全 SQL 兼容
✅ 数据量 < 1TB

选择 CockroachDB/TiDB：
✅ 需要开源方案
✅ 多云部署
✅ 需要 PostgreSQL/MySQL 兼容
✅ 预算中等

选择 DynamoDB/Cassandra：
✅ KV 访问模式
✅ 高吞吐写入
✅ 最终一致性可接受
✅ 简单查询
```

---

##### 数据库选型决策树

```
开始
  ↓
需要 SQL 查询？
  ├─ 是 → 需要全球分布式？
  │       ├─ 是 → 需要强一致性？
  │       │       ├─ 是 → **Spanner**
  │       │       └─ 否 → **CockroachDB / TiDB**
  │       └─ 否 → 数据量 > 1TB？
  │               ├─ 是 → **Aurora / Cloud SQL（高配）**
  │               └─ 否 → **Cloud SQL / RDS**
  │
  └─ 否 → 需要复杂查询（嵌套、数组）？
          ├─ 是 → 需要实时订阅？
          │       ├─ 是 → **Firestore Native**
          │       └─ 否 → **MongoDB**
          │
          └─ 否 → 访问模式已知（Key 查询）？
                  ├─ 是 → 需要极致性能？
                  │       ├─ 是 → **DynamoDB**
                  │
                  └─ 否 → **MongoDB**（灵活查询）
```

**快速选择指南：**

| 场景 | 推荐数据库 | 理由 |
|------|-----------|------|
| 全球支付系统 | Spanner | 强一致性、ACID、全球分布 |
| 电商网站（单区域） | Cloud SQL + Redis | 成本低、SQL 兼容、缓存加速 |
| 移动应用（聊天） | Firestore Native | 实时订阅、离线支持、BaaS |
| IoT 数据采集 | DynamoDB | 高吞吐写入、自动扩展 |
| 内容管理系统 | MongoDB | 灵活 Schema、复杂查询 |
| 游戏排行榜 | Spanner / Redis | 全球一致性 / 极致性能 |
| 日志分析 | BigQuery | 列式存储、PB 级分析 |

---

##### Spanner 常见问题与解决方案

##### 问题 1：热点问题（Hotspot）

**现象：**
```
某个 Split 的 CPU 利用率持续 > 80%
其他 Split CPU < 20%
写入延迟 P99 > 500ms
```

**原因分析：**

**底层原因：Spanner 按主键范围分片（Tablet）**

```
Spanner 的数据分片机制：
┌─────────────────────────────────────────────────────────┐
│  按主键范围自动分片到不同 Tablet                          │
├─────────────────────────────────────────────────────────┤
│  Tablet_1: id [0 - 100万]                               │
│  Tablet_2: id [100万 - 200万]                           │
│  Tablet_3: id [200万 - 300万]                           │
│  Tablet_4: id [300万 - 400万]  ← 当前最大 ID 范围        │
└─────────────────────────────────────────────────────────┘

单调递增 ID 的写入模式：
INSERT id=400001  → Tablet_4  ← 所有新数据
INSERT id=400002  → Tablet_4  ← 都写入这里
INSERT id=400003  → Tablet_4  ← 形成热点
INSERT id=400004  → Tablet_4

结果：
🔴 Tablet_4 CPU 100%（过载）
✅ Tablet_1/2/3 CPU < 10%（闲置）
🔴 无法利用分布式并行能力
```

**三种常见热点原因：**

```
1. 单调递增主键（所有写入集中在最后一个 Tablet）
   CREATE TABLE Users (
     id INT64 NOT NULL,  -- 自增 ID，热点！
     ...
   ) PRIMARY KEY (id);
   
   问题：新数据永远写入"最大 ID 所在的 Tablet"

2. 时间戳前缀（同一时间写入集中在同一个 Tablet）
   PRIMARY KEY (TIMESTAMP, user_id)
   
   问题：同一秒的数据都写入同一个 Tablet
   
   示例：
   2024-01-01 10:00:00, user_1  → Tablet_X
   2024-01-01 10:00:00, user_2  → Tablet_X  ← 同一 Tablet
   2024-01-01 10:00:00, user_3  → Tablet_X  ← 同一 Tablet

3. 热门数据访问（明星用户、热门商品）
   SELECT * FROM Users WHERE user_id = 'celebrity_id';
   
   问题：某个特定 Tablet 的读取压力过大
```

**解决方案：**

**方案 1：主键设计优化**
```sql
-- ❌ 错误：单调递增
CREATE TABLE Users (
  id INT64 NOT NULL,
  ...
) PRIMARY KEY (id);

-- ✅ 方案 A：UUID（分散）
CREATE TABLE Users (
  id STRING(36) NOT NULL DEFAULT (GENERATE_UUID()),
  ...
) PRIMARY KEY (id);

-- ✅ 方案 B：哈希前缀（分散）
CREATE TABLE Users (
  id STRING(36) NOT NULL,
  shard_id INT64 AS (MOD(FARM_FINGERPRINT(id), 100)) STORED,
  ...
) PRIMARY KEY (shard_id, id);

-- ✅ 方案 C：反转时间戳（分散）
CREATE TABLE Events (
  event_id STRING(36) NOT NULL,
  reverse_timestamp INT64 AS (9223372036854775807 - UNIX_MICROS(event_time)) STORED,
  event_time TIMESTAMP,
  ...
) PRIMARY KEY (reverse_timestamp, event_id);
```

**方案 2：预分裂（Pre-split）**
```sql
-- 建表时指定分裂点
CREATE TABLE Users (
  user_id STRING(36) NOT NULL,
  region STRING(10),
  ...
) PRIMARY KEY (user_id);

-- 手动预分裂（假设 UUID 前缀分布均匀）
ALTER TABLE Users ADD ROW DELETION POLICY (OLDER_THAN(TIMESTAMP "2099-01-01", event_time));

-- 使用 gcloud 命令预分裂
gcloud spanner databases ddl update my-database \
  --instance=my-instance \
  --ddl="ALTER TABLE Users ADD ROW DELETION POLICY (OLDER_THAN(TIMESTAMP '2099-01-01', created_at))"
```

**方案 3：读热点优化**
```
1. 使用缓存（Redis）
   - 热门数据缓存 1 分钟
   - 减少 Spanner 读取压力

2. 使用只读事务
   - 无锁，可从任意副本读取
   - 降低 Leader 压力

3. 数据复制
   - 将热门数据复制到多个 Split
   - 分散读取压力
```

**监控与告警：**
```
关键指标：
- Split CPU 利用率 > 65%（告警）
- Split QPS > 10000（告警）
- 锁等待时间 > 100ms（告警）

查询热点 Split：
SELECT
  split_id,
  avg_cpu_utilization,
  qps
FROM
  SPANNER_SYS.SPLIT_STATS
WHERE
  avg_cpu_utilization > 0.65
ORDER BY
  avg_cpu_utilization DESC;
```

---

##### 问题 2：跨区域延迟高

**现象：**
```
用户在亚洲，写入延迟 > 300ms
用户在美国，写入延迟 < 50ms
```

**原因分析：**
```
Tablet Leader 在美国区域
亚洲用户写入需要跨太平洋网络
网络延迟：亚洲 -> 美国 ≈ 150-200ms
```

**解决方案：**

**方案 1：数据局部性配置（Leader 位置）**
```sql
-- 按区域分片
CREATE TABLE Users (
  region STRING(10) NOT NULL,  -- 'US', 'EU', 'ASIA'
  user_id STRING(36) NOT NULL,
  ...
) PRIMARY KEY (region, user_id);

-- 配置 Leader 位置（通过 Interleave 或手动配置）
-- 美国用户 → Leader 在 us-central1
-- 欧洲用户 → Leader 在 europe-west1
-- 亚洲用户 → Leader 在 asia-east1
```

**方案 2：多区域配置优化**
```
配置实例：
- 区域 1：us-central1（美国用户主区域）
- 区域 2：europe-west1（欧洲用户主区域）
- 区域 3：asia-east1（亚洲用户主区域）

副本配置：
- 每个 Tablet 5 副本
- 跨 3 个区域分布
- Leader 优先选择本地区域
```

**方案 3：读写分离**
```sql
-- 写入：路由到 Leader（可能跨区域）
INSERT INTO Users (user_id, name) VALUES ('xxx', 'Alice');

-- 读取：使用只读事务（本地副本）
SELECT * FROM Users WHERE user_id = 'xxx';
-- 只读事务可从本地副本读取，延迟低
```

**方案 4：应用层优化**
```
1. 异步写入
   - 非关键数据异步写入
   - 用户无需等待

2. 批量写入
   - 使用 Mutation API
   - 减少网络往返次数

3. 缓存
   - 读多写少的数据缓存到 Redis
   - 减少跨区域读取
```

**延迟监测：**
```
按区域监测延迟：
- us-central1 → P99 < 50ms
- europe-west1 → P99 < 50ms
- asia-east1 → P99 < 50ms

跨区域延迟：
- asia-east1 → us-central1 → P99 > 200ms
```

---

##### 问题 3：事务冲突与死锁

**现象：**
```
错误信息：
ABORTED: Transaction aborted due to concurrent modification
或
DEADLINE_EXCEEDED: Transaction exceeded deadline

高并发场景下事务失败率 > 5%
```

**原因分析：**
```
1. 多个事务同时修改同一行
   事务 A：UPDATE Users SET balance = balance - 100 WHERE user_id = 'xxx';
   事务 B：UPDATE Users SET balance = balance + 50 WHERE user_id = 'xxx';
   → 冲突，一个事务被 ABORT

2. 长事务持有锁时间过长
   BEGIN;
   SELECT ... FOR UPDATE;  -- 加锁
   -- 业务逻辑处理 5 秒
   UPDATE ...;
   COMMIT;
   → 其他事务等待超时

3. 死锁
   事务 A：锁定行 1 → 等待行 2
   事务 B：锁定行 2 → 等待行 1
   → 死锁
```

**解决方案：**

**方案 1：减少事务范围**
```sql
-- ❌ 错误：大事务
BEGIN;
UPDATE Users SET balance = balance - 100 WHERE region = 'US';  -- 影响百万行
COMMIT;

-- ✅ 正确：小事务
BEGIN;
UPDATE Users SET balance = balance - 100 WHERE user_id = 'xxx';  -- 影响单行
COMMIT;
```

**方案 2：避免长事务**
```sql
-- ❌ 错误：长事务
BEGIN;
SELECT * FROM Users WHERE user_id = 'xxx' FOR UPDATE;
-- 调用外部 API（5 秒）
UPDATE Users SET status = 'processed' WHERE user_id = 'xxx';
COMMIT;

-- ✅ 正确：短事务
-- 1. 先查询（无锁）
SELECT * FROM Users WHERE user_id = 'xxx';
-- 2. 调用外部 API
-- 3. 快速更新
BEGIN;
UPDATE Users SET status = 'processed' WHERE user_id = 'xxx';
COMMIT;
```

**方案 3：使用乐观锁**
```sql
-- 添加版本号字段
CREATE TABLE Users (
  user_id STRING(36) NOT NULL,
  balance NUMERIC,
  version INT64,
  ...
) PRIMARY KEY (user_id);

-- 乐观锁更新
BEGIN;
-- 1. 读取当前版本
SELECT balance, version FROM Users WHERE user_id = 'xxx';
-- balance = 1000, version = 5

-- 2. 更新（检查版本）
UPDATE Users 
SET balance = 900, version = 6
WHERE user_id = 'xxx' AND version = 5;
-- 如果 version 不匹配，UPDATE 影响 0 行，事务失败

COMMIT;
```

**方案 4：重试机制**
```python
def update_with_retry(user_id, amount, max_retries=3):
    for attempt in range(max_retries):
        try:
            with database.batch() as batch:
                batch.update(
                    table='Users',
                    columns=['user_id', 'balance'],
                    values=[(user_id, balance - amount)]
                )
            return True
        except Aborted:
            if attempt == max_retries - 1:
                raise
            time.sleep(0.1 * (2 ** attempt))  # 指数退避
    return False
```

**方案 5：避免死锁**
```sql
-- 统一锁定顺序
-- ❌ 错误：不同顺序
事务 A：锁定 user_id='aaa' → 锁定 user_id='bbb'
事务 B：锁定 user_id='bbb' → 锁定 user_id='aaa'

-- ✅ 正确：统一顺序（按 user_id 排序）
事务 A：锁定 user_id='aaa' → 锁定 user_id='bbb'
事务 B：锁定 user_id='aaa' → 锁定 user_id='bbb'
```

**监控：**
```
关键指标：
- 事务 ABORT 率 > 1%（告警）
- 锁等待时间 > 100ms（告警）
- 事务延迟 P99 > 500ms（告警）
```

---

##### 问题 4：查询性能慢

**现象：**
```
查询耗时 > 5 秒
扫描行数 > 100 万行
CPU 利用率飙升
```

**原因分析：**
```
1. 缺少索引（全表扫描）
   SELECT * FROM Users WHERE email = 'xxx';
   -- 没有 email 索引，扫描全表

2. 索引选择不当
   SELECT * FROM Users WHERE region = 'US' AND status = 'active';
   -- 只有 region 索引，status 需要过滤

3. 大范围扫描
   SELECT * FROM Orders WHERE create_time > '2020-01-01';
   -- 扫描 4 年数据

4. 复杂 JOIN
   SELECT * FROM Orders o
   JOIN Users u ON o.user_id = u.user_id
   JOIN Products p ON o.product_id = p.product_id;
   -- 跨多个 Split 的 JOIN
```

**解决方案：**

**方案 1：添加二级索引**
```sql
-- 创建索引
CREATE INDEX idx_email ON Users(email);

-- 查询自动使用索引
SELECT * FROM Users WHERE email = 'xxx';

-- 复合索引
CREATE INDEX idx_region_status ON Users(region, status);

-- 覆盖索引（避免回表）
CREATE INDEX idx_email_name ON Users(email) STORING (name);
SELECT name FROM Users WHERE email = 'xxx';  -- 无需回表
```

**方案 2：使用 EXPLAIN 分析**
```sql
-- 查看执行计划
EXPLAIN SELECT * FROM Users WHERE email = 'xxx';

-- 输出示例：
-- Scan Table: Users
-- Index: idx_email
-- Rows Scanned: 1
-- Execution Time: 5ms
```

**方案 3：优化查询条件**
```sql
-- ❌ 错误：大范围扫描
SELECT * FROM Orders WHERE create_time > '2020-01-01';

-- ✅ 正确：缩小范围
SELECT * FROM Orders 
WHERE create_time BETWEEN '2024-01-01' AND '2024-01-31'
LIMIT 1000;

-- ✅ 正确：分页查询
SELECT * FROM Orders 
WHERE order_id > 'last_order_id'
ORDER BY order_id
LIMIT 100;
```

**方案 4：Interleave 优化 JOIN**
```sql
-- 父表
CREATE TABLE Users (
  user_id STRING(36) NOT NULL,
  ...
) PRIMARY KEY (user_id);

-- 子表（Interleave）
CREATE TABLE Orders (
  user_id STRING(36) NOT NULL,
  order_id STRING(36) NOT NULL,
  ...
) PRIMARY KEY (user_id, order_id),
  INTERLEAVE IN PARENT Users;

-- JOIN 查询（无跨网络）
SELECT u.name, o.amount
FROM Users u
JOIN Orders o ON u.user_id = o.user_id
WHERE u.user_id = 'xxx';
```

**方案 5：使用只读事务**
```sql
-- 只读事务（无锁，低延迟）
SELECT * FROM Users WHERE user_id = 'xxx';

-- 快照读（历史数据）
SELECT * FROM Users 
FOR SYSTEM_TIME AS OF TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)
WHERE user_id = 'xxx';
```

**监控：**
```
关键指标：
- 查询延迟 P99 > 1s（告警）
- 扫描行数 > 10 万（告警）
- CPU 利用率 > 65%（告警）

慢查询分析：
SELECT
  query_text,
  avg_latency_seconds,
  rows_scanned
FROM
  SPANNER_SYS.QUERY_STATS
WHERE
  avg_latency_seconds > 1
ORDER BY
  avg_latency_seconds DESC;
```

---

##### 问题 5：成本优化

**现象：**
```
月账单 > $10,000
CPU 利用率 < 30%（资源浪费）
存储利用率 < 50%
```

**原因分析：**
```
1. 过度配置（节点数过多）
   配置：10 个节点
   实际：CPU 利用率 < 30%

2. 存储浪费（未清理历史数据）
   存储：10 TB
   活跃数据：2 TB
   历史数据：8 TB（未清理）

3. 跨区域流量（数据传输费用）
   跨区域查询频繁
   数据传输费用高
```

**解决方案：**

**方案 1：节点数优化**
```
CPU 利用率建议：
- 高优先级 CPU：< 65%（推荐）
- 低优先级 CPU：< 90%

节点数计算：
当前：10 个节点，CPU 30%
优化：4 个节点，CPU 75%
节省：60% 成本

监控：
- 持续监控 CPU 利用率
- 根据负载动态调整节点数
```

**方案 2：数据清理策略**
```sql
-- 方案 A：TTL（Time To Live）
CREATE TABLE Events (
  event_id STRING(36) NOT NULL,
  event_time TIMESTAMP,
  ...
) PRIMARY KEY (event_id),
  ROW DELETION POLICY (OLDER_THAN(event_time, INTERVAL 90 DAY));

-- 方案 B：手动归档
-- 1. 导出历史数据到 GCS
gcloud spanner databases execute-sql my-database \
  --instance=my-instance \
  --sql="SELECT * FROM Events WHERE event_time < '2023-01-01'" \
  --format=csv > events_archive.csv

-- 2. 删除历史数据
DELETE FROM Events WHERE event_time < '2023-01-01';

-- 方案 C：分区表（未来支持）
-- Spanner 目前不支持分区表，可通过应用层实现
```

**方案 3：减少跨区域流量**
```
1. 数据局部性
   - 用户数据存储在用户所在区域
   - 减少跨区域查询

2. 缓存
   - 热门数据缓存到 Redis
   - 减少 Spanner 查询

3. 只读副本
   - 读取本地副本
   - 避免跨区域读取
```

**方案 4：使用 Autoscaler**
```
Cloud Spanner Autoscaler：
- 根据 CPU 利用率自动扩缩容
- 高峰期：自动增加节点
- 低峰期：自动减少节点

配置示例：
- 目标 CPU 利用率：65%
- 最小节点数：2
- 最大节点数：10
- 扩容阈值：CPU > 70%（持续 5 分钟）
- 缩容阈值：CPU < 50%（持续 30 分钟）
```

**方案 5：使用 Committed Use Discounts**
```
承诺使用折扣：
- 1 年承诺：37% 折扣
- 3 年承诺：55% 折扣

适用场景：
- 稳定负载
- 长期使用
```

**成本监控：**
```
关键指标：
- 月成本 > 预算（告警）
- CPU 利用率 < 30%（资源浪费）
- 存储增长率 > 10%/月（告警）

成本分析：
- 节点成本：$0.90/小时/节点
- 存储成本：$0.30/GB/月
- 数据传输成本：$0.12/GB（跨区域）
```

---

## OLTP & OLAP & HTAP

### 数据库分类与对比

#### 常见 OLTP & OLAP & HTAP 数据库

**术语解释：**

- **OLAP（Online Analytical Processing）**：在线分析处理，面向复杂查询和聚合分析，读多写少
- **OLTP（Online Transaction Processing）**：在线事务处理，面向高并发事务，读写均衡，强一致性
- **HTAP（Hybrid Transaction/Analytical Processing）**：混合事务/分析处理，同时支持 OLTP 和 OLAP
- **ROLAP（Relational OLAP）**：关系型 OLAP，基于关系数据库（SQL）实现 OLAP，通过 SQL 查询和物化视图加速分析
- **MOLAP（Multidimensional OLAP）**：多维 OLAP，预计算多维数据立方体（Cube），查询极快但灵活性低
- **MPP（Massively Parallel Processing）**：大规模并行处理，将查询分解到多个节点并行执行

| 产品                       | 架构定位                   | 存储模型                                       | 查询接口                         | 实时性与事务                                       | 多表关联性能                      | 运维复杂度                                     | 云厂商/开源                  |
| :----------------------- | :--------------------- | :----------------------------------------- | :--------------------------- | :------------------------------------------- | :--------------------------- | :---------------------------------------- | :---------------------- |
| ClickHouse               | 纯 OLAP MPP 列式数据库       | MergeTree 系列列式引擎 + <br>自动压缩                | SQL<br>（ClickHouse 方言）       | 最佳批量写入吞吐<br>⚠️ 轻量事务 (v22.8+)<br>📥 批量 INSERT | **中等**<br>（大宽表优先，JOIN 优化提升） | **需手动分片，<br>🔴 开源版无自动 Rebalance，<br>维护高** | 开源<br>（ClickHouse Inc.） |
| TiDB<br>（TiKV + TiFlash） | HTAP 混合行列存储            | TiKV 行存储 + <br>TiFlash 列存储 + <br>自动压缩      | SQL<br>（MySQL 兼容）            | ✅ 强一致事务支持<br>📥 支持实时写入                       | **优**<br>（TiFlash MPP 模式）   | TiUP 工具简化运维，<br>中等                        | 开源（PingCAP）            |
| Apache Doris             | MPP OLAP 数据库           | 列式存储 + <br>自动压缩，<br>支持 Unique/Aggregate 模型 | SQL<br>（MySQL 兼容）            | ⚠️ 基础事务支持<br>📥 Stream Load 实时导入             | **优**<br>（Runtime Filter/Join 优化） | **低**<br>（FE+BE 极简架构，不依赖 ZK）              | 开源（Apache）             |
| StarRocks                | MPP OLAP 列式数据库         | 列式存储 + <br>分区分桶 + <br>自动压缩                 | SQL<br>（MySQL 兼容）            | ⚠️ 轻量事务支持<br>📥 Stream Load 实时导入             | **最优**<br>（强大的 CBO 优化器）      | **中/低**<br>（FE+BE 架构，部署简单）                | 开源<br>（Linux Foundation） |
| Greenplum                | MPP 数据库（基于 PostgreSQL） | 行存储 + 列存储（AO 表）+ <br>可选压缩                  | SQL<br>（PostgreSQL 兼容）       | ✅ 支持事务（MVCC）<br>📥 支持 INSERT/批量导入            | PostgreSQL 兼容，关联性能好         | 需手动分布键设计，<br>维护中等                         | 开源（VMware）             |
| AWS Redshift             | 云原生 MPP 数据仓库           | 列式存储 + <br>自动压缩                            | SQL<br>（PostgreSQL 兼容）       | ✅ 支持 ACID 事务<br>📥 COPY 从 S3 批量导入（推荐）        | 星型模型优化，关联性能优              | 全托管，自动扩缩容，<br>维护低                         | AWS 托管                  |
| GCP BigQuery             | Serverless MPP 数据仓库    | Capacitor 列式存储 + <br>自动压缩                  | SQL<br>（标准 SQL）              | ✅ 支持 ACID 事务<br>📥 Streaming/Load Job        | 自动优化 JOIN，性能优             | 全托管 Serverless，<br>维护极低                   | GCP 托管                  |
| Azure Synapse            | 云原生 MPP 数据仓库           | 列式存储 + <br>PolyBase 外部表 + <br>自动压缩         | SQL<br>（T-SQL）               | ✅ 支持事务（专用池）<br>📥 PolyBase/COPY 批量导入         | 星型模型优化，关联性能优              | 全托管，自动扩缩容，<br>维护低                         | Azure 托管                |
| Snowflake                | 云原生 MPP 数据仓库           | 列式存储 + <br>微分区 + <br>自动压缩                  | SQL<br>（标准 SQL + 扩展）         | ✅ 支持 ACID 事务<br>📥 支持小批量写入/COPY INTO         | 自动优化 JOIN，性能优             | 全托管，存储计算分离，<br>维护极低                       | 多云托管                    |
| Apache Druid             | 实时 OLAP 平台             | 列式存储 + <br>倒排索引 + <br>自动压缩                 | SQL + <br>Native Query（JSON） | 实时摄入（秒级）<br>📥 Kafka/HTTP 流式摄入               | **弱**<br>（单表聚合优异，JOIN 有限）  | **高**<br>（组件多，依赖 ZK/Deep Storage）         | 开源（Apache）             |
| Presto/Trino             | 分布式 SQL 查询引擎           | 无存储（查询联邦）                                  | SQL<br>（ANSI SQL）            | 🔴 不支持事务<br>🚫 不支持写入                         | 跨数据源关联，性能中等               | 需配置连接器，<br>维护中等                           | 开源（Meta/Trino）         |

**OLAP（主流）基本都采用 MPP 架构，也都基本支持 SQL 查询方式**
	• ClickHouse
	• Apache Doris
	• StarRocks
	• Greenplum
	• Redshift
	• BigQuery
	• Snowflake
	• Azure Synapse
	• Presto/Trino

#### 数据库核心机制对比

##### 0. 核心写入流程（OLTP vs OLAP）

##### OLAP 存储可变性（核心概念）

OLAP 数据库的存储单元可以分为两类：

| 维度          | 可变存储（Mutable）                      | 不可变存储（Immutable）          |
| ----------- | ---------------------------------- | ------------------------- |
| **更新方式**    | 原地更新（In-Place Update）              | 重写文件（Rewrite）             |
| **DELETE**  | 标记删除或原地删除                          | 标记删除 + 后台合并               |
| **UPDATE**  | 直接修改数据                             | 插入新版本 + 删除旧版本 + **原子替换** |
| **性能**      | UPDATE 快                           | 查询快（无碎片）                  |
| **更新机制**    | Delete Bitmap / Primary Key        | Mutation / 重写 Part         |
| **适用场景**    | 频繁更新（用户画像、CDC）                     | 追加写入（日志、时序）               |
| **代表产品**    | Apache Doris、StarRocks、Redshift、Greenplum | ClickHouse、Druid、BigQuery |

**特殊情况：Snowflake**
- **存储**：不可变（Micro-partition 不可变）
- **MVCC**：✅ 有（快照隔离）- OLAP 中的罕见特例
- **实现方式**：通过 S3 不可变文件 + 元数据版本实现 MVCC
- **Time Travel**：保留历史版本文件（天到周级别）
- **更新机制**：重写 Micro-partition，旧版本保留用于 Time Travel
- **说明**：Snowflake 是少数支持 MVCC 的 OLAP 数据库，主要用于 Time Travel 功能，而非传统的事务隔离

**MVCC 在 OLTP vs OLAP 中的差异：**

| 维度         | OLTP 的 MVCC      | OLAP 的 MVCC（Snowflake 特例）         |
| ---------- | ---------------- | --------------------------------- |
| **主要目的**   | 事务隔离、并发控制        | Time Travel、历史查询                  |
| **实现方式**   | Undo Log / 数据页内  | S3 不可变文件 + 元数据版本                  |
| **版本保留时间** | 短期（事务结束后清理）      | 长期（天到周级别）                         |
| **适用场景**   | 高并发读写、事务一致性      | 历史数据查询、数据恢复                       |
| **代表产品**   | MySQL、PostgreSQL | Snowflake（Greenplum 继承 PG MVCC）   |
| **是否常见**   | ✅ OLTP 标配        | 🔴 OLAP 罕见（仅 Snowflake、Greenplum） |

**注意：Apache Doris、StarRocks 的 Delete Bitmap / Primary Key 不是 MVCC**
- Delete Bitmap / Primary Key 只是更新机制，用于标记删除和主键去重
- 不提供事务隔离和多版本并发控制
- 不支持读取历史版本数据

#####  OLTP & OLAP 写入流程

**OLTP 写入流程：**
```
Buffer Pool 数据页修改 => Undo Log（磁盘）=> Redo Log（磁盘）=> 事务提交 => 数据页（磁盘，异步）
```
- **特点**：小事务、高并发、需要快速回滚、ACID 保证
- **适用**：MySQL、PostgreSQL、Oracle、SQL Server

**OLAP 写入流程（4 种组合模式）：**

**模式 1：WAL + 可变存储**
```
内存批量数据 => WAL（磁盘）=> 原地更新 Tablet => 多副本同步
```
- **适用**：StarRocks、Apache Doris、Greenplum
- **特点**：支持高效 UPDATE/DELETE，适合 HTAP 场景
- **更新机制**：Delete Bitmap（Apache Doris）、Primary Key（StarRocks）
- **MVCC**：通过 Delete Bitmap 或 Primary Key 实现

**模式 2：不可变文件 + 无 WAL**
```
内存批量数据 => 不可变 Part/Segment（磁盘，同步写入）=> 后台合并 => 多副本同步
```
- **适用**：ClickHouse、Druid
- **特点**：查询性能极致，UPDATE 需要重写 Part
- **更新机制**：Mutation（ClickHouse）、重新摄入（Druid）
- **MVCC**：无
- **数据安全保证**：
  - ClickHouse：批量写入直接持久化到 Part 文件（同步写入），写入成功后才返回
  - 多副本机制：数据同时写入多个副本（通过 ZooKeeper 协调）
  - 崩溃恢复：依赖副本恢复，未返回成功的数据由客户端重试
  - ⚠️ 风险：单行写入或小批量写入可能导致大量 Part 文件，影响查询性能
  - ✅ 推荐：批量写入 10000+ 行，摊销磁盘 I/O 成本

**模式 3：云存储 + 可变**
```
内存批量数据 => Transaction Log => S3 + 本地缓存 => 原地更新
```
- **适用**：Redshift
- **特点**：云存储持久化，支持原地更新
- **更新机制**：列式存储原地更新
- **MVCC**：无

**模式 4：云存储 + 不可变 + MVCC（Snowflake 特例）**
```
内存批量数据 => S3 不可变文件 => 元数据更新 => 保留历史版本
```
- **适用**：Snowflake、BigQuery（BigQuery 无 MVCC）
- **特点**：完全托管，存储计算分离，更新重写文件
- **更新机制**：重写 Micro-partition（Snowflake）、重写数据（BigQuery）
- **MVCC**：Snowflake 通过版本文件实现，BigQuery 无 MVCC

**核心差异：**

| 维度               | OLTP        | OLAP                |
| ---------------- | ----------- | ------------------- |
| **写入粒度**         | 小事务（行级）     | 批量导入（百万行）           |
| **Undo Log**     | ✅ 必需（频繁回滚）  | 🔴 不需要（失败重试）        |
| **Redo Log/WAL** | ✅ 必需（崩溃恢复）  | ⚠️ 部分需要（或用其他机制）     |
| **崩溃恢复**         | Redo + Undo | WAL 重放 / 副本恢复 / 云存储 |
| **高可用**          | 主从复制（可选）    | 多副本（核心）             |

---

##### 1. 存储与事务机制

**Redo Log vs Undo Log 核心概念：**
- **Redo Log（重做日志）**：崩溃恢复的**核心机制**，**重做已提交但未写入磁盘**的事务，保证 🟢持久性（Durability）
- **Undo Log（回滚日志）**：事务回滚 + MVCC 的**辅助机制**，**回滚未提交**的事务，保证 🟢原子性（Atomicity）
- **崩溃恢复流程**：Redo Log 重做已提交事务 → Undo Log 回滚未提交事务
- **OLAP 为什么不需要 Undo Log**：
  - 依赖**分片/副本机制**进行崩溃恢复（多副本 + 分布式存储如 S3/HDFS）
  - **批量写入**为主，失败后重新导入，很少需要回滚单个事务
  - 采用**不可变数据**设计（Immutable Data），更新通过标记删除 + 插入新版本实现
  - 只需 **Redo Log/WAL** 保证持久性，不需要复杂的事务回滚机制

**OLTP vs OLAP 的崩溃恢复**

| 维度       | OLTP               | OLAP              |
| -------- | ------------------ | ----------------- |
| Redo Log | ✅ 必需（重做已提交事务）      | ✅ 必需（重做已提交写入）     |
| Undo Log | ✅ 必需（回滚未提交事务）      | 🔴 不需要（很少回滚）      |
| 副本机制     | ⚠️ 可选（主从复制）        | ✅ 核心（多副本 + 分布式存储） |
| 崩溃恢复     | Redo + Undo + 主从切换 | WAL + 副本恢复        |
| 事务粒度     | 小事务（毫秒级）           | 批量导入（分钟级）         |
| 回滚频率     | 高（业务逻辑失败）          | 低（导入失败重试）         |

| 数据库                | 类型        | MVCC                | WAL/Redo Log        | Undo Log               | 事务支持                  | 存储模型                 | 持久化机制                    |
| ------------------ | --------- | ------------------- | ------------------- | ---------------------- | --------------------- | -------------------- | ------------------------ |
| **MySQL (InnoDB)** | OLTP      | ✅ 有                 | ✅ Redo Log          | ✅ Undo Log             | ✅ ACID                | 行存储                  | Redo Log + DWB           |
| **PostgreSQL**     | OLTP      | ✅ 有                 | ✅ WAL               | ✅ 数据页内（Tuple）          | ✅ ACID                | 行存储                  | WAL + DWB                |
| **Oracle**         | OLTP      | ✅ 有                 | ✅ Redo Log          | ✅ Undo 表空间             | ✅ ACID                | 行存储                  | Redo + Undo              |
| **SQL Server**     | OLTP      | ✅ 有                 | ✅ Transaction Log   | ✅ tempdb 版本存储          | ✅ ACID                | 行存储                  | WAL                      |
| **============**   | **=====** | **===============** | **===============** | **==================** | **=================** | **================** | **====================** |
| **TiDB**           | HTAP      | ✅ 有 (Percolator)    | ✅ Raft Log          | ✅ MVCC 版本链             | ✅ ACID                | 行存储 (TiKV)           | Raft + RocksDB WAL       |
| **TiFlash**        | HTAP      | ✅ 共享 TiDB MVCC      | ✅ Delta Log         | ✅ 共享 TiDB              | ✅ 读一致性                | 列存储                  | Delta + Stable           |
| **============**   | **=====** | **===============** | **===============** | **==================** | **=================** | **================** | **====================** |
| **ClickHouse**     | OLAP      | 🔴 无                | 🔴 无 WAL（直接写 Part）          | 🔴 无                   | ⚠️ 轻量事务 (v22.8+)      | 列存储 (MergeTree)      | 副本 + ZooKeeper 协调<br>**批量写入直接持久化**              |
| **StarRocks**      | OLAP      | 🔴 无                | ✅ RocksDB WAL（Tablet 级别）       | 🔴 无                   | ⚠️ 轻量事务               | 列存储                  | RocksDB WAL + Tablet     |
| **Doris**          | OLAP      | 🔴 无                | ✅ RocksDB WAL（Tablet 级别）       | 🔴 无                   | ⚠️ 轻量事务               | 列存储                  | RocksDB WAL + Tablet     |
| **Greenplum**      | OLAP      | ✅ 有 (继承 PG)         | ✅ WAL               | ✅ 数据页内（继承 PG）          | ✅ ACID                | 行/列混合                | WAL (继承 PG)              |
| **Redshift**       | OLAP      | 🔴 无                | ✅ Transaction Log   | 🔴 无                   | ✅ ACID 事务               | 列存储                  | S3 + 本地缓存                |
| **BigQuery**       | OLAP      | 🔴 无                | 🔴 无（Colossus 原子写入）      | 🔴 无                   | ✅ ACID 事务               | 列存储 (Capacitor)      | Colossus 分布式存储           |
| **Snowflake**      | OLAP      | ✅ 有 (快照隔离)          | ✅ Metadata Log      | ✅ 快照版本                 | ✅ ACID                | 列存储 (微分区)            | S3 + 元数据服务               |
| **Druid**          | OLAP      | 🔴 无                | 🔴 无                | 🔴 无                   | 🔴 无事务                | 列存储 + 倒排索引           | Deep Storage (S3/HDFS)   |
| **============**   | **=====** | **===============** | **===============** | **==================** | **=================** | **================** | **====================** |
| **Cassandra**      | NoSQL     | 🔴 无                | ✅ CommitLog         | 🔴 无                   | ⚠️ 轻量事务               | 宽列存储                 | CommitLog + SSTable      |
| **HBase**          | NoSQL     | 🔴 无                | ✅ WAL               | 🔴 无                   | 🔴 无事务                | 列族存储 (LSM)           | WAL + HFile              |
| **MongoDB**        | NoSQL     | ✅ 有 (WiredTiger)    | ✅ Journal           | ✅ WiredTiger 版本        | ✅ ACID (4.0+)         | 文档存储                 | Journal + Oplog          |
| **Redis**          | NoSQL     | 🔴 无                | ✅ AOF/RDB           | 🔴 无                   | ⚠️ 轻量事务<br>（MULTI/EXEC，无回滚）                | 内存键值                 | AOF + RDB 快照             |
| **DynamoDB**       | NoSQL     | 🔴 无                | ✅ 内部 Log            | 🔴 无                   | ⚠️ 有限事务               | 键值/文档                | 分布式日志                    |
| **============**   | **=====** | **===============** | **===============** | **==================** | **=================** | **================** | **====================** |
| **Neo4j**          | 图数据库      | 🔴 无                | ✅ Transaction Log   | 🔴 无                   | ✅ ACID                | 图存储                  | WAL                      |
| **InfluxDB**       | 时序        | 🔴 无                | ✅ WAL               | 🔴 无                   | 🔴 事务                 | TSM (列式)             | WAL + TSM                |
| **Prometheus**     | 时序        | 🔴 无                | ✅ WAL               | 🔴 无                   | 🔴 无事务                | 时序块                  | WAL + TSDB               |

**关键发现：**
- **MVCC**：主要存在于 OLTP 和 HTAP 数据库，**OLAP 数据库通常不需要 MVCC**
- **WAL**：几乎所有数据库都有某种形式的 WAL，用于持久化和恢复
  - **例外**：ClickHouse 和 BigQuery 无 WAL
  - **ClickHouse**：批量写入直接持久化到 Part 文件 + 多副本机制
  - **BigQuery**：依赖 Colossus 分布式文件系统的原子写入保证
- **事务支持**：OLTP 和 HTAP 提供完整 ACID，OLAP 通常不支持或仅支持轻量事务（ClickHouse v22.8+ 支持轻量事务，主要用于单表多行插入的原子性）

**持久化机制说明：**
- **MySQL Redo Log vs Binlog**：
  - **Redo Log**（InnoDB 引擎层）：崩溃恢复的核心，保证持久性（Durability），事务提交前写入
  - **Binlog**（MySQL Server 层）：用于主从复制、数据恢复、审计，事务提交后写入，**不是主要持久化机制**
  - MySQL 持久化依赖 Redo Log，Binlog 是复制和备份机制
- **StarRocks/Apache Doris Binlog**：
  - 这里的 Binlog **不是 MySQL 的 Binlog**，是 **FE（Frontend）的元数据变更日志**
  - 用于 FE 节点间同步元数据（表结构、分区信息等），不是数据持久化机制
  - 数据持久化依赖底层 **RocksDB WAL**（每个 Tablet 分片的 LSM-Tree 引擎）

**Undo Log 机制说明：**

**Undo Log 的作用：**
1. **事务回滚**：当事务执行失败或主动回滚时，使用 Undo Log 恢复数据到事务开始前的状态
2. **MVCC（多版本并发控制）**：通过 Undo Log 构建数据的历史版本，实现读写不阻塞
3. **一致性读**：在 Repeatable Read 隔离级别下，通过 Undo Log 读取事务开始时的快照数据

**不同数据库的 Undo 实现：**

| 数据库 | Undo 实现 | 存储位置 | 清理机制 |
|--------|----------|----------|----------|
| **MySQL** | Undo Log | 独立的 Undo 表空间（ibdata1 或独立文件） | Purge 线程异步清理 |
| **PostgreSQL** | 数据页内（Tuple） | 旧版本直接存储在数据页中（dead tuple） | VACUUM 清理死元组 |
| **Oracle** | Undo 表空间 | 独立的 Undo 表空间 | 自动 Undo 管理（AUM） |
| **SQL Server** | tempdb 版本存储 | tempdb 数据库（Version Store） | 自动清理 tempdb |
| **TiDB** | MVCC 版本链 | TiKV 中的多版本数据（RocksDB） | GC Worker 清理旧版本 |
| **Greenplum** | 数据页内（继承 PG） | 数据页中（继承 PostgreSQL） | VACUUM（继承 PG） |
| **Snowflake** | 快照版本 | S3 中的不可变文件 + 元数据 | Time Travel 过期清理 |

**为什么 OLAP 数据库通常不需要 Undo Log：**

1. **写入模式不同**
   - **OLTP**：频繁的小事务（INSERT/UPDATE/DELETE），需要随时回滚
   - **OLAP**：批量导入（COPY/LOAD），很少回滚，失败后重新导入即可

2. **并发模式不同**
   - **OLTP**：高并发读写混合，**需要 MVCC 避免读写锁冲突**
   - **OLAP**：读多写少，写入通常是离线批量导入，可以用分区版本化或不可变数据

3. **查询模式不同**
   - **OLTP**：点查询（WHERE id = 1），需要最新数据，需要隔离级别保证
   - **OLAP**：全表扫描或大范围聚合，可以接受快照读（查询开始时的数据状态）

4. **性能优先**
   - 维护 Undo Log 会增加写入开销（每次更新需要保存旧版本）
   - OLAP 追求查询性能和写入吞吐，牺牲事务能力换取性能

5. **数据不可变性**
   - 很多 OLAP 数据库采用**不可变数据**设计（Immutable Data）
   - 更新操作通过**标记删除 + 插入新版本**实现（如 ClickHouse 的 Mutation）
   - 不需要传统的 Undo Log，通过版本化文件实现类似功能

**例外情况：**
- **Greenplum**：继承 PostgreSQL，保留了完整的 MVCC 和 Undo 机制（数据页内）
- **Snowflake**：支持 ACID 事务和 Time Travel，通过 S3 不可变文件 + 元数据实现快照隔离
- **TiFlash**：HTAP 架构，共享 TiDB 的 MVCC 机制，支持一致性读

---

##### 2. 查询处理机制

| 数据库            | 语法解析树               | AST  | 查询优化器   | 执行引擎  | 并行处理               | 向量化执行      |
| -------------- | ------------------- | ---- | ------- | ----- | ------------------ | ---------- |
| **MySQL**      | ✅ 有                 | ✅ 有  | ✅ CBO   | 行执行   | ⚠️ 有限              | 🔴 无       |
| **PostgreSQL** | ✅ 有                 | ✅ 有  | ✅ CBO   | 行/向量化 | ✅ 并行查询             | ⚠️ 部分      |
| **Oracle**     | ✅ 有                 | ✅ 有  | ✅ CBO   | 行执行   | ✅ 并行查询             | 🔴 无       |
| **SQL Server** | ✅ 有                 | ✅ 有  | ✅ CBO   | 行/批处理 | ✅ 并行查询             | ⚠️ 批处理模式   |
| **TiDB**       | ✅ 有                 | ✅ 有  | ✅ CBO   | 向量化   | ✅ MPP (TiFlash)    | ✅ 有        |
| **ClickHouse** | ✅ 有                 | ✅ 有  | ✅ CBO   | 向量化   | ✅ MPP              | ✅ 有 (SIMD) |
| **StarRocks**  | ✅ 有                 | ✅ 有  | ✅ CBO   | 向量化   | ✅ MPP              | ✅ 有        |
| **Doris**      | ✅ 有                 | ✅ 有  | ✅ CBO   | 向量化   | ✅ MPP              | ✅ 有        |
| **Greenplum**  | ✅ 有                 | ✅ 有  | ✅ CBO   | 行执行   | ✅ MPP              | 🔴 无       |
| **Redshift**   | ✅ 有                 | ✅ 有  | ✅ CBO   | 向量化   | ✅ MPP              | ✅ 有        |
| **BigQuery**   | ✅ 有                 | ✅ 有  | ✅ CBO   | 向量化   | ✅ Dremel           | ✅ 有        |
| **Snowflake**  | ✅ 有                 | ✅ 有  | ✅ CBO   | 向量化   | ✅ MPP              | ✅ 有        |
| **Druid**      | ✅ 有                 | ✅ 有  | ✅ 简化优化  | 列扫描   | ✅ 分布式              | ⚠️ 部分      |
| **Cassandra**  | ✅ 有 (CQL)           | ✅ 有  | ⚠️ 简单优化 | 行执行   | ✅ 分布式              | 🔴 无       |
| **HBase**      | 🔴 API 调用           | 🔴 无 | 🔴 无    | 行扫描   | ✅ 分布式              | 🔴 无       |
| **MongoDB**    | ✅ 有 (MQL)           | ✅ 有  | ✅ 查询计划  | 文档扫描  | ✅ 分片               | 🔴 无       |
| **Redis**      | 🔴 命令解析             | 🔴 无 | 🔴 无    | 直接执行  | ⚠️ 多线程 I/O (v6.0+) | 🔴 无       |
| **Neo4j**      | ✅ 有 (Cypher)        | ✅ 有  | ✅ CBO   | 图遍历   | ✅ 并行遍历             | 🔴 无       |
| **InfluxDB**   | ✅ 有 (InfluxQL/Flux) | ✅ 有  | ✅ 查询计划  | 列扫描   | ✅ 分布式              | ⚠️ 部分      |

**关键发现：**
- **AST**：所有支持复杂查询语言的数据库都有 AST，Redis 等简单命令式数据库除外
- **向量化执行**：OLAP 数据库普遍支持，OLTP 数据库较少支持
- **并行处理**：OLAP 数据库都支持 MPP，OLTP 数据库支持有限（Redis 6.0+ 支持多线程 I/O，但命令执行仍是单线程）

---

##### 3. 索引与存储优化

| 数据库            | 主要索引类型              | 二级索引                 | 数据压缩          | 分区支持    | 物化视图 | 列式存储                |
| -------------- | ------------------- | -------------------- | ------------- | ------- | ---- | ------------------- |
| **MySQL**      | B-Tree              | ✅ 有                  | ⚠️ 有限         | ✅ 有     | 🔴 无 | 🔴 无                |
| **PostgreSQL** | B-Tree, GiST, GIN   | ✅ 有                  | ✅ TOAST       | ✅ 有     | ✅ 有  | 🔴 无                |
| **Oracle**     | B-Tree, Bitmap      | ✅ 有                  | ✅ 高级压缩        | ✅ 有     | ✅ 有  | ⚠️ 混合列存储            |
| **SQL Server** | B-Tree, Columnstore | ✅ 有                  | ✅ 页压缩         | ✅ 有     | ✅ 有  | ✅ Columnstore Index |
| **TiDB**       | B-Tree (LSM)        | ✅ 有                  | ✅ Snappy/Zstd | ✅ 有     | 🔴 无 | 🔴 无 (TiKV)         |
| **TiFlash**    | 列式索引                | ✅ 有                  | ✅ LZ4/Zstd    | ✅ 有     | 🔴 无 | ✅ 列存储               |
| **ClickHouse** | 稀疏索引                | ✅ 跳数索引               | ✅ 多种算法        | ✅ 有     | ✅ 有  | ✅ 列存储               |
| **StarRocks**  | 前缀索引                | ✅ Bitmap/BloomFilter | ✅ LZ4/Zstd    | ✅ 有     | ✅ 有  | ✅ 列存储               |
| **Doris**      | 前缀索引                | ✅ Bitmap/BloomFilter | ✅ LZ4/Zstd    | ✅ 有     | ✅ 有  | ✅ 列存储               |
| **Greenplum**  | B-Tree              | ✅ 有                  | ✅ 压缩          | ✅ 有     | ✅ 有  | ⚠️ AO 表             |
| **Redshift**   | 分布键                 | ✅ 排序键                | ✅ 自动压缩        | ✅ 有     | ✅ 有  | ✅ 列存储               |
| **BigQuery**   | 聚簇                  | 🔴 无传统索引             | ✅ 自动压缩        | ✅ 有     | ✅ 有  | ✅ 列存储               |
| **Snowflake**  | 微分区                 | 🔴 无传统索引             | ✅ 自动压缩        | ✅ 有     | ✅ 有  | ✅ 列存储               |
| **Druid**      | 时间索引                | ✅ 倒排索引               | ✅ LZ4         | ✅ 有     | 🔴 无 | ✅ 列存储               |
| **Cassandra**  | 分区键                 | ✅ 二级索引               | ✅ LZ4/Snappy  | ✅ 有     | ✅ 有  | 🔴 宽列               |
| **HBase**      | 行键                  | 🔴 无                 | ✅ Snappy/LZO  | ✅ 有     | 🔴 无 | 🔴 列族               |
| **MongoDB**    | B-Tree              | ✅ 有                  | ✅ Snappy/Zstd | ✅ 分片    | 🔴 无 | 🔴 文档               |
| **Redis**      | Hash                | 🔴 无                 | ⚠️ LZF        | 🔴 无    | 🔴 无 | 🔴 内存               |
| **Neo4j**      | 标签索引                | ✅ 属性索引               | ✅ 压缩          | 🔴 无    | 🔴 无 | 🔴 图                |
| **InfluxDB**   | 时间索引                | ✅ Tag 索引             | ✅ 高压缩比        | ✅ Shard | 🔴 无 | ✅ TSM               |

**关键发现：**
- **列式存储**：OLAP 数据库标配，OLTP 数据库通常不支持
- **物化视图**：OLAP 和部分 OLTP 支持，用于查询加速
- **数据压缩**：OLAP 数据库压缩率更高（10:1 到 100:1）

---

##### 4. 高可用与分布式

| 数据库            | 副本机制       | 一致性协议       | 分片/分区     | 故障转移        | 全球分布                | 读写分离          |
| -------------- | ---------- | ----------- | --------- | ----------- | ------------------- | ------------- |
| **MySQL**      | 主从复制       | 🔴 无 (异步)   | ⚠️ 手动分片   | ⚠️ 手动       | 🔴 无                | ✅ 有           |
| **PostgreSQL** | 流复制        | 🔴 无 (异步)   | ⚠️ 手动分片   | ⚠️ 手动       | 🔴 无                | ✅ 有           |
| **Oracle**     | Data Guard | ⚠️ Redo 同步  | ⚠️ 手动分片   | ✅ 自动        | ✅ Active Data Guard | ✅ 有           |
| **SQL Server** | Always On  | ✅ WSFC      | ⚠️ 手动分片   | ✅ 自动        | ✅ 可用性组              | ✅ 有           |
| **TiDB**       | Raft       | ✅ Raft      | ✅ 自动      | ✅ 自动        | ✅ 支持                | ✅ 有 (TiFlash) |
| **ClickHouse** | 副本表        | ✅ ZooKeeper | ✅ 手动分片    | 🔴 无自动 Rebalance<br>（开源版）       | 🔴 无                | ✅ 有           |
| **StarRocks**  | 副本         | ✅ Quorum    | ✅ 自动      | ✅ 自动        | 🔴 无                | ✅ 有           |
| **Doris**      | 副本         | ✅ Quorum    | ✅ 自动      | ✅ 自动        | 🔴 无                | ✅ 有           |
| **Greenplum**  | 镜像         | 🔴 无        | ✅ 手动分片    | ⚠️ 手动       | 🔴 无                | ✅ 有           |
| **Redshift**   | 快照         | 🔴 无        | ✅ 自动      | ✅ 自动        | ✅ 跨区域               | 🔴 无          |
| **BigQuery**   | 自动副本       | ✅ Colossus  | ✅ 自动      | ✅ 自动        | ✅ 多区域               | 🔴 无          |
| **Snowflake**  | 自动副本       | ✅ 内部协议      | ✅ 自动      | ✅ 自动        | ✅ 多云                | ✅ 有           |
| **Druid**      | 副本         | ✅ ZooKeeper | ✅ 自动      | ✅ 自动        | 🔴 无                | ✅ 有           |
| **Cassandra**  | 副本         | ✅ Quorum    | ✅ 自动      | ✅ 自动        | ✅ 支持                | 🔴 无          |
| **HBase**      | HDFS 副本    | ✅ ZooKeeper | ✅ 自动      | ✅ 自动        | 🔴 无                | ✅ 有           |
| **MongoDB**    | 副本集        | ✅ Raft-like | ✅ 分片      | ✅ 自动        | ✅ 支持                | ✅ 有           |
| **Redis**      | 主从复制       | 🔴 无 (异步)   | ✅ Cluster | ⚠️ Sentinel | 🔴 无                | ✅ 有           |
| **DynamoDB**   | 副本         | ✅ Quorum    | ✅ 自动      | ✅ 自动        | ✅ 全球表               | 🔴 无          |
| **Spanner**    | Paxos      | ✅ Paxos     | ✅ 自动      | ✅ 自动        | ✅ 原生                | ✅ 有           |
| **Neo4j**      | 副本         | ✅ Raft      | ⚠️ 手动分片   | ✅ 自动        | ✅ 联邦                | ✅ 有           |
| **InfluxDB**   | 副本         | ✅ Raft      | ✅ 自动      | ✅ 自动        | 🔴 无                | ✅ 有           |

**关键发现：**
- **一致性协议**：分布式数据库普遍采用 Raft/Paxos/Quorum，传统数据库多为异步复制
- **自动分片**：云原生和分布式数据库支持自动分片，传统数据库需要手动
- **全球分布**：云数据库（Spanner, DynamoDB, Cosmos DB）和部分分布式数据库支持

---

##### 5. 核心机制总结

**OLTP 数据库特征：**
- ✅ MVCC（多版本并发控制）
- ✅ WAL（Write-Ahead Log）
- ✅ 完整 ACID 事务
- ✅ B-Tree 索引
- 🔴 通常不支持列式存储
- 🔴 有限的并行查询能力

**OLAP 数据库特征：**
- 🔴 通常无 MVCC
- ✅ 某种形式的 WAL/Binlog
- ❌ 不支持或轻量事务
- ✅ 列式存储
- ✅ 向量化执行引擎
- ✅ MPP 并行处理
- ✅ 高压缩比
- ✅ 物化视图

**HTAP 数据库特征：**
- ✅ MVCC（行存储部分）
- ✅ WAL + 列式存储
- ✅ 完整 ACID 事务
- ✅ 行列混合存储
- ✅ 支持 OLTP 和 OLAP 负载

**NoSQL 数据库特征：**
- ⚠️ 部分支持 MVCC（MongoDB）
- ✅ 某种形式的持久化日志
- ⚠️ 有限或无事务支持（HBase 需要 Phoenix 提供事务）
- ✅ 灵活的数据模型
- ✅ 水平扩展能力
- ✅ 最终一致性（部分支持强一致性，如 HBase 强一致读写）

---

#### 技术选型参考

- 纯 OLAP 分析、超大表快速聚合时首选 ClickHouse，其向量化引擎对宽表性能极佳.
- 既要 OLTP 强一致又要 OLAP 查询时考虑 TiDB，通过 TiFlash 提供列存分析能力.
- 需要低运维成本、支持物化视图加速常见聚合且多表关联场景多时优选 Apache Doris.
- 对多表关联、复杂 SQL 和高并发分析有极致需求时推荐 StarRocks，其多表 join 性能优于 ClickHouse.

**行级与列式写入区别**

MySQL 写入数据过程：

- MySQL 属于关系型数据库，数据以行方式存储。写入时通过 SQL 的 INSERT INTO 操作，将一行数据整体写入到表中。
- 数据直接写入磁盘存储的表文件，支持事务和 ACID 特性，保证数据一致性。
- 写入时可能涉及多次磁盘 I/O 操作，同时需要维护索引和日志文件，如重做日志（Redo Log），以保证事务安全.

HBase 写入数据过程：

- HBase 是列族型分布式数据库，数据写入先写入 WAL（Write Ahead Log，预写日志）用于防止数据丢失。
- 数据接着写入内存中的 MemStore（类似缓存），写入速度快。
- 当 MemStore 达到阈值，会合并批量刷新到存储在 HDFS 上的 HFile 文件中，HFile 是列族聚簇存储格式。
- 这种写路径设计减少了磁盘随机写，采用顺序写入日志和批量写大文件以提升写入性能.

**为什么 HBase 查询速度快及磁盘存储和 I/O 机制区别：**

- HBase 采用列族聚簇存储，磁盘上的数据是按列族聚簇存放在 HFile 中，查询时可以只读取需要的列族，显著减少无关数据的 I/O 读取。
- 使用了内存缓存（BlockCache）保存热点数据块，减少磁盘访问。
- 采用 LSM（Log-Structured Merge Tree）结构，把写操作先存入内存，再批量合并写磁盘，减少随机写，优化磁盘 I/O。
- 传统 MySQL 是行存储，读时必须载入整行数据，进行 I/O 量较大；写时需要随机写多个数据页，造成磁盘负载较高。
- HBase 通过顺序写 WAL 和批量刷写 HFile，有效降低磁盘寻址和随机写，提高大数据环境下的读写性能和扩展性.

**行式存储与列式存储查询区别**

例：数据包含三个字段：name、address、phone

MySQL 由于是行式存储，三个字段在磁盘上是连续存储的，查询时需要一次性读取整行数据。

ClickHouse 由于是列式存储，三个字段分别独立存储，每列的所有行数据连续存放在一起。

| 查询场景      | MySQL 查询效率 | ClickHouse 查询效率     |
|:--------- |:---------- |:-------------- |
| 查询所有列     | 较高（整行读取）   | 较低（需读取多个列文件） |
| 查询部分列（单列） | 较低（需读整行数据） | 极高（只读单列文件，向量化扫描）    |

| 内容      | MySQL（行式存储）            | ClickHouse（列式存储）                        |
|:------- |:---------------------- |:---------------------------------- |
| 数据写入    | 直接写整行到磁盘文件，支持事务和日志     | 按列批量写入，每列独立文件 |
| 存储结构    | 按行组织，数据行连续存储           | 按列组织，每列所有行数据连续存储              |
| 查询时 I/O | 需读取整行，读取不相关列数据造成额外 I/O | 只读取查询所需列文件，跳过其他列                    |
| 磁盘写入方式  | **随机写，磁盘负载大**          | 批量顺序写，按列压缩              |
| 缓存机制    | 页面缓存                   | 列数据缓存，向量化处理                 |
| 适用场景    | 事务处理，OLTP 应用             | 分析查询，OLAP 场景，聚合统计                   |

#### MySQL

| 隔离级别                   | 简称          | 允许的异常现象       | 备注         |
|:---------------------- |:----------- |:------------- | ---------- |
| 读未提交（Read Uncommitted） | RU（最低）      | 脏读、不可重复读、幻读   |            |
| 读已提交（Read Committed）   | RC          | 不可重复读、幻读      |            |
| 可重复读（Repeatable Read）  | RR（MySQL默认） | 仅幻读           | 使用 MVCC 实现 |
| 串行化（Serializable）      | SS（最高）      | 无异常事务现象（完全隔离） |            |

脏读：一个事务读取了另一个未提交事务修改的数据。

不可重复读：一个事务多次读同一数据，结果不一致。

幻读：一个事务多次执行查询，数据集合发生变化（新增或删除了行）。

- InnoDB还通过**间隙锁（Gap Lock）和 Next-Key锁** 解决事务中的幻读问题。

##### 查询优化器底层工作机制

- MySQL 查询优化器负责将SQL语句转换为高效的执行计划。
- **解析**：SQL语句经过词法和语法分析**生成解析树（Parse Tree）**。
- **预处理**：**抽象语法树 AST（Abstract Syntax Tree）**做简单改写，比如解析别名、视图展开。
- **查询优化**：优化器进行多种策略决策，如访问路径选择（全表扫描、索引扫描）、连接算法（嵌套循环、哈希连接）、连接顺序优化。
- 使用代价模型估算不同执行方案的成本，选择最低代价方案。
- 利用统计信息（表和索引数据分布）辅助决策。
- 支持执行计划缓存，加快相同SQL的再执行。

**解析数和抽象语法树主要区别**

	SQL 语句
	    ↓ 词法分析（Lexer）
	Token 流
	    ↓ 语法分析（Parser）
	解析树（Parse Tree）← 完整的语法结构
	    ↓ 简化/抽象
	语法树（AST）← 去掉冗余，保留语义
	    ↓ 预处理
	改写后的 AST
	    ↓ 优化
	优化后的执行计划

| 维度   | 解析树（Parse Tree） | 语法树（AST）     |
| ---- | --------------- | ------------ |
| 生成阶段 | 语法分析（Parsing）   | 语法分析后的简化     |
| 节点数量 | 多（包含所有语法规则）     | 少（只保留语义节点）   |
| 冗余信息 | 包含（如括号、分号等）     | 去除           |
| 用途   | 验证语法正确性         | 后续优化和执行      |
| 是否抽象 | 具体（Concrete）    | 抽象（Abstract） |

```
解析树（Parse Tree）

定义： 词法和语法分析的直接产物，完整反映语法规则

SQL: SELECT id FROM users WHERE age > 18

解析树（Parse Tree）：
          SELECT语句
         /    |    \
    SELECT   FROM   WHERE
      |       |       |
     id     users   条件
                    /  |  \
                  age  >  18

特点：
- 包含所有语法细节
- 包含所有终结符和非终结符
- 结构冗余，节点多
  
  
2. 语法树（Syntax Tree / AST - Abstract Syntax Tree）

定义： 解析树的简化版本，去掉冗余信息

SQL: SELECT id FROM users WHERE age > 18

语法树（AST）：
      Query
     /  |  \
  SELECT FROM WHERE
    |     |     |
   id   users  >
              / \
            age  18

特点：
- 去掉了冗余的语法节点
- 只保留语义相关的节点
- 结构简洁，便于后续处理  
```


**MySQL 8.0 删除查询缓存，推荐用应用层缓存（如Redis）替代**

- MySQL早期支持查询缓存，缓存查询结果，后续相同查询直接返回结果。写操作会导致缓存失效，带来性能开销。

- MySQL 删除 Query Cache 主要因为**全局锁机制造成的并发瓶颈**
  
  - 全局互斥锁（Global Mutex）：
    
    - 查询缓存时：加锁
    - 写入缓存时：加锁
    - 失效缓存时：加锁
    
    高并发场景：
    线程 1: SELECT ... (等待锁)
    线程 2: SELECT ... (等待锁)
    线程 3: INSERT ... (持有锁，清空缓存)
    线程 4: SELECT ... (等待锁)
    ...
    
    → 所有线程排队等待
    → 并发性能急剧下降

##### 索引

**索引数据结构对比**

| 数据结构       | 查找时间复杂度         | 范围查询   | 优点                       | 缺点          | 适用场景         |
| :--------- | :-------------- | :----- | :----------------------- | :---------- | :----------- |
| **B+Tree** | O(log N)        | ✅ 支持   | 多路平衡树，叶子节点链表，3-4层支持千万级数据 | 维护成本高       | **MySQL 默认** |
| Hash       | O(1)            | 🔴 不支持 | 等值查询极快                   | 无法范围查询，无法排序 | Memory 引擎    |
| 二叉树        | O(log N) ~ O(N) | ✅ 支持   | 简单                       | 极端情况退化为链表   | 不适用          |

**B+Tree 索引原理**

```
B+Tree 结构（3层，每个节点3个键）：

                    [10, 20]              ← 根节点（非叶子节点）
                   /    |    \
                  /     |     \
         [3, 7]      [13, 17]   [23, 27]  ← 中间节点（非叶子节点）
        /  |  \      /  |  \     /  |  \
       /   |   \    /   |   \   /   |   \
     [1,2] [5,6] [8,9] [11,12] [15,16] [18,19] [21,22] [25,26] [28,29]  ← 叶子节点（存储数据）
       ↔     ↔     ↔      ↔       ↔       ↔       ↔       ↔       ↔
                    双向链表（支持范围查询）

特点：
1. 非叶子节点只存储键（索引），不存储数据
2. 叶子节点存储所有数据，并通过双向链表连接
3. 所有叶子节点在同一层（平衡树）
4. 多路（每个节点多个子节点），减少树高度

性能分析：
- 树高度 = log_m(N)，m = 每个节点键数量，N = 总记录数
- InnoDB 默认页大小 16KB，每个节点约 1000 个键
- 3层 B+Tree：1000 × 1000 × 1000 = 10亿条记录
- 查询次数 = 树高度 = 3 次磁盘 I/O
```

**主键索引 vs 二级索引**

| 维度 | 主键索引（聚簇索引） | 二级索引（非聚簇索引） |
|:---|:---|:---|
| 叶子节点存储 | 完整行数据 | 主键值 |
| 查询性能 | 快（直接返回数据） | 慢（需要回表） |
| 存储空间 | 大 | 小 |
| 数据组织 | 按主键排序 | 按索引列排序 |

```
主键索引（id）：
叶子节点：[id=1, name='Tom', age=25] → [id=2, name='Jane', age=30] → ...

二级索引（name）：
叶子节点：[name='Jane', id=2] → [name='Tom', id=1] → ...
                    ↓ 回表
          主键索引查找 id=2 的完整数据
```

**回表查询示例**

```sql
-- 使用二级索引（name）
SELECT * FROM users WHERE name = 'Tom';

执行流程：
1. 在 name 索引的 B+Tree 中查找 'Tom'
   - 找到叶子节点：[name='Tom', id=1]
2. 回表：使用 id=1 在主键索引中查找
   - 找到完整数据：[id=1, name='Tom', age=25]
3. 返回结果

性能：2次 B+Tree 查找（二级索引 + 主键索引）
```

**覆盖索引（避免回表）**

```sql
-- 创建复合索引
CREATE INDEX idx_name_age ON users(name, age);

-- 查询只需要索引列
SELECT name, age FROM users WHERE name = 'Tom';

执行流程：
1. 在 idx_name_age 索引中查找 'Tom'
   - 叶子节点：[name='Tom', age=25, id=1]
2. 索引已包含 name 和 age，无需回表
3. 直接返回结果

性能：1次 B+Tree 查找（无回表）
```

**索引下推（Index Condition Pushdown, ICP）**

```sql
-- 复合索引：idx_name_age (name, age)
SELECT * FROM users WHERE name LIKE 'T%' AND age > 20;

无索引下推（MySQL 5.6 之前）：
1. 使用 name 索引查找 'T%'，返回所有匹配的主键
2. 回表获取完整数据
3. 在 Server 层过滤 age > 20

有索引下推（MySQL 5.6+）：
1. 使用 name 索引查找 'T%'
2. 在存储引擎层直接过滤 age > 20（索引下推）
3. 只回表符合条件的记录

性能提升：减少回表次数
```

**最左前缀原则**

```sql
-- 复合索引：idx_abc (a, b, c)

✅ 可以使用索引：
WHERE a = 1
WHERE a = 1 AND b = 2
WHERE a = 1 AND b = 2 AND c = 3
WHERE a = 1 AND c = 3  （只使用 a）

❌ 无法使用索引：
WHERE b = 2
WHERE c = 3
WHERE b = 2 AND c = 3

原因：B+Tree 按 (a, b, c) 顺序排序，跳过 a 无法定位
```

复合索引

- 多列组合的索引，对多条件查询优化效果显著
- 建议按照查询中字段的使用频率和过滤性顺序设计

前缀索引

- 对字符串列只索引开头部分（如前10个字符），减少索引空间
- 切记前缀过短可能导致选择性差，影响效果

覆盖索引

- 索引包含查询所需的所有字段，不访问表数据，提升查询效率
- 适用于频繁查询且涉及少量字段的场景

##### 慢查询分析

- **开启慢查询日志**，记录执行时间超过阈值的SQL。
- 分析慢查询可关注：完全扫描、索引缺失、JOIN顺序、锁等待等。
- 用`EXPLAIN`查看执行计划，优化索引和查询写法。
- 对大表可拆分、分区或使用分页等技术避免全表扫描。
- 调整服务器配置（缓存大小、连接数限制）协同优化

##### MVCC（多版本并发控制）

**MVCC 核心原理**

MVCC 通过保存数据的多个版本，实现读写不阻塞，提高并发性能。

**隐藏字段**

InnoDB 为每行记录添加 3 个隐藏字段：

| 字段 | 长度 | 说明 |
|:---|:---|:---|
| DB_TRX_ID | 6 字节 | 最后修改该行的事务 ID |
| DB_ROLL_PTR | 7 字节 | 回滚指针，指向 Undo Log 中的上一个版本 |
| DB_ROW_ID | 6 字节 | 隐藏主键（无主键时使用） |

```
实际存储的行数据：
[id=1, name='Tom', age=25, DB_TRX_ID=100, DB_ROLL_PTR=0x1234, DB_ROW_ID=1]
```

**Undo Log 版本链**

```
当前数据（最新版本）：
[id=1, name='Tom', age=30, DB_TRX_ID=103, DB_ROLL_PTR → Undo Log]
                                                    ↓
Undo Log 版本链：
[id=1, name='Tom', age=25, DB_TRX_ID=102, DB_ROLL_PTR → 更早版本]
                                                    ↓
[id=1, name='Tom', age=20, DB_TRX_ID=101, DB_ROLL_PTR → NULL]

版本链形成过程：
1. 事务 101：INSERT (id=1, name='Tom', age=20)
2. 事务 102：UPDATE age=25（旧版本写入 Undo Log）
3. 事务 103：UPDATE age=30（旧版本写入 Undo Log）
```

**Read View（读视图）**

Read View 决定事务能看到哪些版本的数据。

| 字段 | 说明 |
|:---|:---|
| m_ids | 当前活跃事务 ID 列表 |
| min_trx_id | 最小活跃事务 ID |
| max_trx_id | 下一个要分配的事务 ID |
| creator_trx_id | 创建 Read View 的事务 ID |

**可见性判断规则**

```
判断版本是否可见（按顺序判断）：

1. DB_TRX_ID == creator_trx_id
   → 可见（自己修改的数据）

2. DB_TRX_ID < min_trx_id
   → 可见（事务开始前已提交）

3. DB_TRX_ID >= max_trx_id
   → 不可见（事务开始后才启动的事务）

4. min_trx_id <= DB_TRX_ID < max_trx_id
   → 如果 DB_TRX_ID 在 m_ids 中：不可见（未提交）
   → 如果 DB_TRX_ID 不在 m_ids 中：可见（已提交）

如果当前版本不可见，沿着 DB_ROLL_PTR 查找 Undo Log 中的上一个版本，重复判断
```

**RC vs RR 的 Read View 生成时机**

| 隔离级别 | Read View 生成时机 | 效果 |
|:---|:---|:---|
| **RC（读已提交）** | 每次查询生成新的 Read View | 能读到其他事务已提交的修改（不可重复读） |
| **RR（可重复读）** | 事务开始时生成一次 Read View | 整个事务期间读到的数据一致（可重复读） |

**MVCC 示例**

```sql
-- 初始数据：id=1, name='Tom', age=20

时间线：
T1: 事务 A 开始（TRX_ID=100）
T2: 事务 B 开始（TRX_ID=101）
T3: 事务 A 执行：UPDATE users SET age=25 WHERE id=1;
T4: 事务 A 提交
T5: 事务 B 执行：SELECT age FROM users WHERE id=1;

RC 隔离级别：
- T5 时刻，事务 B 生成新的 Read View
- min_trx_id=102（事务 A 已提交）
- 事务 A 的修改（age=25）可见
- 结果：age=25（不可重复读）

RR 隔离级别：
- T2 时刻，事务 B 生成 Read View
- m_ids=[100, 101]，min_trx_id=100
- T5 时刻，事务 A（TRX_ID=100）在 m_ids 中，不可见
- 沿着 Undo Log 查找旧版本（age=20）
- 结果：age=20（可重复读）
```

**MVCC 优势**

✅ 读写不阻塞：读操作不加锁，写操作只锁定当前版本
✅ 提高并发：多个事务可以同时读取不同版本
✅ 解决不可重复读：RR 隔离级别下保证事务内一致性

**MVCC 局限**

🔴 无法完全解决幻读：需要配合 Next-Key Lock
🔴 Undo Log 占用空间：长事务导致版本链过长
🔴 只支持 RC 和 RR：串行化不使用 MVCC

##### Lock 机制

**锁粒度对比**

| 锁类型 | 适用引擎 | 粒度 | 作用 | 主要问题 |
|:---|:---|:---|:---|:---|
| 表级锁 | MyISAM、MEMORY | 粗 | 简化加锁，阻塞整个表操作 | 影响并发性能 |
| 行级锁 | InnoDB | 细 | 高并发，锁定具体行 | 资源消耗，可能死锁 |
| 间隙锁 | InnoDB | 间隙 | 防止范围内数据插入，解决幻读问题 | 可能导致锁升级，影响性能 |

**InnoDB 锁算法**

| 锁算法 | 锁定范围 | 作用 | 隔离级别 |
|:---|:---|:---|:---|
| **Record Lock** | 单条索引记录 | 锁定具体行 | RC、RR |
| **Gap Lock** | 索引记录之间的间隙 | 防止插入，解决幻读 | RR |
| **Next-Key Lock** | Record Lock + Gap Lock | 锁定记录 + 前面的间隙 | RR（默认） |

**Record Lock（记录锁）**

```sql
-- 表结构：id 主键，age 普通索引
-- 数据：id=1,5,10,15,20

-- 事务 A
BEGIN;
UPDATE users SET name='Tom' WHERE id=10;  -- 锁定 id=10 这一行
-- 持有锁：Record Lock on id=10

-- 事务 B
UPDATE users SET name='Jane' WHERE id=5;  -- ✅ 成功（不同行）
UPDATE users SET name='Bob' WHERE id=10;  -- ❌ 阻塞（同一行）
```

**Gap Lock（间隙锁）**

```sql
-- 数据：id=1,5,10,15,20
-- 间隙：(-∞,1), (1,5), (5,10), (10,15), (15,20), (20,+∞)

-- 事务 A（RR 隔离级别）
BEGIN;
SELECT * FROM users WHERE id BETWEEN 5 AND 15 FOR UPDATE;
-- 锁定间隙：(5,10), (10,15)

-- 事务 B
INSERT INTO users (id, name) VALUES (7, 'Tom');   -- ❌ 阻塞（间隙 5-10）
INSERT INTO users (id, name) VALUES (12, 'Jane'); -- ❌ 阻塞（间隙 10-15）
INSERT INTO users (id, name) VALUES (3, 'Bob');   -- ✅ 成功（间隙 1-5）
```

**Next-Key Lock（记录锁 + 间隙锁）**

```sql
-- 数据：id=1,5,10,15,20

-- 事务 A
BEGIN;
SELECT * FROM users WHERE id >= 10 FOR UPDATE;
-- Next-Key Lock 锁定：
-- 1. Record Lock: id=10, 15, 20
-- 2. Gap Lock: (5,10), (10,15), (15,20), (20,+∞)

-- 事务 B
UPDATE users SET name='Tom' WHERE id=10;  -- ❌ 阻塞（Record Lock）
INSERT INTO users (id, name) VALUES (12, 'Jane'); -- ❌ 阻塞（Gap Lock）
INSERT INTO users (id, name) VALUES (25, 'Bob');  -- ❌ 阻塞（Gap Lock）
```

**锁退化规则**

| 查询条件 | 锁类型 | 说明 |
|:---|:---|:---|
| 唯一索引等值查询（记录存在） | Record Lock | 退化为记录锁 |
| 唯一索引等值查询（记录不存在） | Gap Lock | 退化为间隙锁 |
| 唯一索引范围查询 | Next-Key Lock | 标准 Next-Key Lock |
| 非唯一索引查询 | Next-Key Lock | 标准 Next-Key Lock |

**死锁检测**

```sql
-- 死锁场景：交叉锁定

-- 事务 A
BEGIN;
UPDATE users SET name='Tom' WHERE id=1;  -- 持有 id=1 的锁
-- 等待 id=2 的锁...
UPDATE users SET name='Tom' WHERE id=2;  -- ❌ 阻塞

-- 事务 B
BEGIN;
UPDATE users SET name='Jane' WHERE id=2; -- 持有 id=2 的锁
-- 等待 id=1 的锁...
UPDATE users SET name='Jane' WHERE id=1; -- ❌ 死锁！

-- InnoDB 死锁检测：
-- 1. 构建 Wait-for Graph（等待图）
-- 2. 检测到循环等待：A → B → A
-- 3. 自动回滚代价最小的事务（Undo Log 最少）
-- 4. 事务 B 回滚，事务 A 继续执行
```

**死锁预防**

```sql
-- 1. 按相同顺序访问资源
-- ✅ 正确：事务都按 id 升序更新
UPDATE users SET name='Tom' WHERE id=1;
UPDATE users SET name='Tom' WHERE id=2;

-- 2. 缩短事务时间
BEGIN;
UPDATE users SET name='Tom' WHERE id=1;
COMMIT;  -- 尽快提交

-- 3. 降低隔离级别（RC 无 Gap Lock）
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 4. 使用索引避免锁升级
-- ❌ 全表扫描，锁定所有行
UPDATE users SET name='Tom' WHERE age=25;
-- ✅ 使用索引，只锁定匹配行
CREATE INDEX idx_age ON users(age);
```

**查看锁信息**

```sql
-- MySQL 8.0+
SELECT * FROM performance_schema.data_locks;
SELECT * FROM performance_schema.data_lock_waits;

-- 查看死锁日志
SHOW ENGINE INNODB STATUS\G
-- 查看 LATEST DETECTED DEADLOCK 部分
```

##### 查询执行流程

**SQL 执行完整流程**

```
客户端
  ↓
连接器（验证身份、管理连接）
  ↓
查询缓存（MySQL 8.0 已删除）
  ↓
解析器（词法分析、语法分析）
  ↓
预处理器（语义检查、权限验证）
  ↓
优化器（生成执行计划）
  ↓
执行器（调用存储引擎接口）
  ↓
存储引擎（InnoDB/MyISAM）
  ↓
返回结果
```

**1. 连接器**

```sql
-- 建立连接
mysql -h host -u user -p

-- 连接器工作：
-- 1. 验证用户名密码
-- 2. 查询权限表
-- 3. 分配连接 ID
-- 4. 维护连接状态（空闲/活跃）

-- 查看连接
SHOW PROCESSLIST;

-- 连接超时（默认 8 小时）
wait_timeout = 28800
```

**2. 解析器**

```sql
SELECT id, name FROM users WHERE age > 20;

-- 词法分析：
-- Token: [SELECT] [id] [,] [name] [FROM] [users] [WHERE] [age] [>] [20]

-- 语法分析：
-- 生成语法树（AST）
SELECT
├─ SELECT_LIST
│  ├─ id
│  └─ name
├─ FROM
│  └─ users
└─ WHERE
   └─ age > 20
```

**3. 预处理器**

```sql
-- 语义检查
SELECT id, name FROM users WHERE age > 20;

-- 检查项：
-- 1. 表 users 是否存在
-- 2. 列 id, name, age 是否存在
-- 3. 当前用户是否有 SELECT 权限
-- 4. 解析别名、视图展开
```

**4. 优化器（核心）**

```sql
SELECT * FROM orders o
JOIN users u ON o.user_id = u.id
WHERE u.age > 30;

-- 优化器决策：
-- 1. 索引选择
--    - orders 表：主键索引 vs user_id 索引
--    - users 表：主键索引 vs age 索引

-- 2. JOIN 顺序
--    方案 A：先扫描 orders，再 JOIN users
--    方案 B：先过滤 users (age>30)，再 JOIN orders ✅

-- 3. JOIN 算法
--    - Nested Loop Join（小表驱动大表）
--    - Block Nested Loop Join（无索引时）
--    - Index Nested Loop Join（有索引时）✅

-- 4. 成本计算
--    - I/O 成本：读取页数 × 1.0
--    - CPU 成本：记录数 × 0.2
--    - 选择成本最低的方案
```

**EXPLAIN 执行计划分析**

```sql
EXPLAIN SELECT * FROM users WHERE age > 20;

-- 关键字段：
-- id: 查询序列号（越大越先执行）
-- select_type: 查询类型（SIMPLE/PRIMARY/SUBQUERY）
-- table: 访问的表
-- type: 访问类型（性能从好到差）
-- possible_keys: 可能使用的索引
-- key: 实际使用的索引
-- rows: 预计扫描行数
-- Extra: 额外信息
```

**type 字段（访问类型）**

| type | 说明 | 性能 |
|:---|:---|:---|
| system | 表只有一行（系统表） | ⭐⭐⭐⭐⭐ |
| const | 主键或唯一索引等值查询 | ⭐⭐⭐⭐⭐ |
| eq_ref | 唯一索引扫描（JOIN） | ⭐⭐⭐⭐ |
| ref | 非唯一索引扫描 | ⭐⭐⭐⭐ |
| range | 索引范围扫描 | ⭐⭐⭐ |
| index | 索引全扫描 | ⭐⭐ |
| ALL | 全表扫描 | ⭐ |

```sql
-- const（最优）
EXPLAIN SELECT * FROM users WHERE id = 1;
-- type: const, rows: 1

-- ref（良好）
EXPLAIN SELECT * FROM users WHERE age = 25;
-- type: ref, rows: 100

-- range（可接受）
EXPLAIN SELECT * FROM users WHERE age > 20;
-- type: range, rows: 5000

-- ALL（最差）
EXPLAIN SELECT * FROM users WHERE name LIKE '%Tom%';
-- type: ALL, rows: 100000
```

**Extra 字段（重要信息）**

| Extra                 | 说明          | 性能   |
| :-------------------- | :---------- | :--- |
| Using index           | 覆盖索引，无需回表   | ✅ 好  |
| Using where           | 使用 WHERE 过滤 | ✅ 正常 |
| Using index condition | 索引下推        | ✅ 好  |
| Using filesort        | 文件排序（无索引）   | 🔴 差 |
| Using temporary       | 使用临时表       | 🔴 差 |

```sql
-- Using index（覆盖索引）
CREATE INDEX idx_age ON users(age);
EXPLAIN SELECT age FROM users WHERE age > 20;
-- Extra: Using index

-- Using filesort（需要优化）
EXPLAIN SELECT * FROM users ORDER BY age;
-- Extra: Using filesort
-- 优化：CREATE INDEX idx_age ON users(age);

-- Using temporary（需要优化）
EXPLAIN SELECT age, COUNT(*) FROM users GROUP BY age;
-- Extra: Using temporary
-- 优化：CREATE INDEX idx_age ON users(age);
```

**5. 执行器**

```sql
SELECT * FROM users WHERE age > 20;

-- 执行器流程：
-- 1. 检查权限（是否有 SELECT 权限）
-- 2. 调用存储引擎接口
--    - 打开表：handler->open()
--    - 读取第一行：handler->read_first()
--    - 循环读取：handler->read_next()
--    - 判断条件：age > 20
--    - 符合条件加入结果集
-- 3. 返回结果给客户端
-- 4. 记录慢查询日志（如果超时）
```

**优化建议**

```sql
-- 1. 避免 SELECT *
-- ❌ 差
SELECT * FROM users WHERE id = 1;
-- ✅ 好
SELECT id, name FROM users WHERE id = 1;

-- 2. 使用覆盖索引
CREATE INDEX idx_name_age ON users(name, age);
SELECT name, age FROM users WHERE name = 'Tom';

-- 3. 避免函数操作索引列
-- ❌ 索引失效
SELECT * FROM users WHERE YEAR(create_time) = 2024;
-- ✅ 使用索引
SELECT * FROM users WHERE create_time >= '2024-01-01' AND create_time < '2025-01-01';

-- 4. 避免隐式类型转换
-- ❌ 索引失效（phone 是 VARCHAR）
SELECT * FROM users WHERE phone = 13800138000;
-- ✅ 使用索引
SELECT * FROM users WHERE phone = '13800138000';

-- 5. 使用 LIMIT 限制返回行数
SELECT * FROM users WHERE age > 20 LIMIT 100;
```

##### 存储引擎对比

**InnoDB vs MyISAM**

| 维度   | InnoDB                | MyISAM    |
| :--- | :-------------------- | :-------- |
| 事务   | ✅ 支持 ACID             | 🔴 不支持    |
| 锁粒度  | 行锁                    | 表锁        |
| 外键   | ✅ 支持                  | 🔴 不支持    |
| MVCC | ✅ 支持                  | 🔴 不支持    |
| 崩溃恢复 | ✅ Redo Log + Undo Log | 🔴 无自动恢复  |
| 全文索引 | ✅ 支持（5.6+）            | ✅ 支持      |
| 存储空间 | 较大（事务日志）              | 较小        |
| 适用场景 | OLTP、高并发写入            | OLAP、只读查询 |

**InnoDB 崩溃恢复**

```
写入流程：
1. 写 Undo Log（回滚日志）
2. 更新 Buffer Pool（内存）
3. 写 Redo Log（重做日志）
4. 提交事务
5. 后台刷盘（异步）

崩溃恢复：
1. 读取 Redo Log
2. 重做已提交事务（前滚）
3. 读取 Undo Log
4. 回滚未提交事务（回滚）
5. 数据恢复完成
```

**存储文件对比**

```
InnoDB：
├─ .frm（表结构，MySQL 8.0 已废弃）
├─ .ibd（表数据 + 索引）
└─ ib_logfile0/1（Redo Log）

MyISAM：
├─ .frm（表结构）
├─ .MYD（表数据）
└─ .MYI（索引）
```

**其他锁类型**

| 锁类型 | 适用引擎 | 粒度 | 作用 | 主要问题 |
|:---|:---|:---|:---|:---|
| 临键锁 | InnoDB         | 行+间隙 | 防止**幻读**，是隐式锁    | 锁范围扩大，影响并发性      |
| 意向锁 | InnoDB         | 表    | 协调表锁和行锁          | 本身无阻塞作用，仅辅助其他锁判断 |
| 自增锁 | InnoDB, MyISAM | 事务级  | 控制自增列安全          | 影响批量写入性能         |


**死锁及解决**

- 多个事务相互等待，形成循环依赖，导致系统无法继续执行。
- MySQL自动检测死锁，选择一个事务回滚解除死锁。

**ShardingSphere 分库分表局限性**

- 跨分片 JOIN（性能差）
- 跨分片聚合（性能差）
- 分布式事务（性能差）
- 扩容困难
- 不支持全局二级索引

#### 基于数据库的  Parser/Planner/Optimizer 通用架构进行对比

| 数据库        | Parser | Planner | Optimizer | Executor | 复杂度 | 特点       |
| ---------- | ------ | ------- | --------- | -------- | --- | -------- |
| MySQL      | ✅      | ✅       | ✅（中等）     | ✅        | 中   | 单机，基于成本  |
| PostgreSQL | ✅      | ✅       | ✅（强）      | ✅        | 高   | 单机，优化器强大 |
| Presto     | ✅      | ✅       | ✅（很强）     | ✅        | 很高  | 分布式，流式处理 |
| MongoDB    | ✅      | ✅       | ✅（简单）     | ✅        | 低   | 主要是索引选择  |
| HBase      | ✅      | 🔴      | 🔴        | ✅        | 极低  | 直接定位，无优化 |
| Redis      | ✅      | 🔴      | 🔴        | ✅        | 极低  | 哈希查找，无优化 |
| Cassandra  | ✅      | ✅       | ✅（简单）     | ✅        | 低   | 分区路由     |
| ClickHouse | ✅      | ✅       | ✅（强）      | ✅        | 高   | 列式存储，优化强 |

```
关系型数据库:
• 支持复杂 Join
• 支持子查询
• 支持聚合
• 需要复杂优化

NoSQL:
• 不支持或限制 Join
• 查询路径固定
• 优化空间小

优化器复杂度（从高到低）:
1. Presto/Spark SQL（分布式 + 多数据源）
2. PostgreSQL（单机，优化器强）
3. MySQL（单机，优化器中等）
4. ClickHouse（列式存储，优化强）
5. MongoDB（文档数据库，简单优化）
6. Cassandra（分区路由）
7. HBase（直接定位）
8. Redis（哈希查找）
```

**为什么 NoSQL 优化器更简单？SQL 和 NoSQL 设计目标不同**

- **查询模式简单**
  - 关系型数据库复杂，优化器需要考虑：选择哪个索引？Join 顺序？Join 算法（Nested Loop/Hash/Merge）？是否需要临时表？排序算法？
  - NoSQL，直接使用 Get
- **设计目标不同**
  - 关系型数据库，目标：灵活的查询，代价：复杂的优化器
  - NoSQL，目标：高性能、高可用，代价：查询能力受限，优势：简单、快速
- 数据模型限制
  - 关系型数据库，支持复杂 Join，支持子查询，支持聚合，需要复杂优化
  - NoSQL，不支持或限制 Join，查询路径固定，优化空间小

#### 系统文件文件大小

```
磁盘存储层次（从上到下 - 应用到硬件）

┌─────────────────────────────────────────────────────┐
│          应用程序层  (数据库、文件编辑器等)              │
└─────────────────────────────────────────────────────┘
                        ↓
              4. 页 (Page) - 应用层
                 ├─ 大小：4KB - 16KB（数据库）
                 ├─ 特点：数据库/应用程序 I/O 单位
                 └─ 层级：应用程序层
                        ↓
┌─────────────────────────────────────────────────────┐
│           文件系统层 (NTFS, ext4, XFS 等)             │
└─────────────────────────────────────────────────────┘
                        ↓
              3. 簇 (Cluster) - 文件系统层
                 ├─ 大小：4KB - 64KB（可配置）
                 ├─ 特点：文件系统分配空间的最小单位
                 └─ 层级：文件系统层
                        ↓
┌─────────────────────────────────────────────────────┐
│           操作系统层 (Linux, Windows 等)              │
└─────────────────────────────────────────────────────┘
                        ↓
              2. 块 (Block) - 操作系统层
                 ├─ 大小：4KB（Linux 默认）
                 ├─ 特点：操作系统 I/O 的基本单位
                 └─ 层级：OS 文件系统层
                        ↓
┌─────────────────────────────────────────────────────┐
│           磁盘驱动层 (HDD, SSD 控制器)                 │
└─────────────────────────────────────────────────────┘
                        ↓
              1. 扇区 (Sector) - 物理层
                 ├─ 大小：512B（传统）或 4KB（高级格式化）
                 ├─ 特点：磁盘的最小物理读写单位
                 └─ 层级：硬件层
                        ↓
┌─────────────────────────────────────────────────────┐
│            物理磁盘层 (磁盘盘片、闪存芯片)               │
└─────────────────────────────────────────────────────┘
```

##### WAL 和 DWB 数据库对比表

- **WAL 保证事务持续性，DWB 解决部分写问题**

- 是否需要 DWB，需要考虑数据是否是不可变数据（Imuutable Data），数据库页是否等于系统 OS 页，或采用了 Full Page Write，Checksum 等其他方案

| 数据库          | 默认页大小    | OS 页大小 | 有 WAL       | 有 DWB | 部分写解决方案              |
| ------------ | -------- | ------ | ----------- | ----- | -------------------- |
| MySQL InnoDB | 16KB     | 4KB    | ✅           | ✅     | DWB (Double Write)   |
| PostgreSQL   | 8KB      | 4KB    | ✅           | 🔴    | **Full Page Writes** |
| Oracle       | 8KB      | 4KB    | ✅           | 🔴    | Redo Log + Checksum  |
| SQL Server   | 8KB      | 4KB    | ✅           | 🔴    | Torn Page Detection  |
| MongoDB      | 32KB     | 4KB    | ✅           | 🔴    | Journaling           |
| SQLite       | 4KB      | 4KB    | ✅           | 🔴    | 无需（页大小相同）            |
| ClickHouse   | 64KB-1MB | 4KB    | 🔴          | 🔴    | MergeTree 不可变        |
| Cassandra    | -        | 4KB    | ✅           | 🔴    | CommitLog + SSTable  |
| Redis        | -        | 4KB    | ✅ (AOF，RDB) | 🔴    | 内存数据库                |

**MySQL 故障切换细节**

| 场景           | WAL 处理                                                                               | 故障检测机制                                                                                                                                                                     | 故障切换时间                                                                                                                                                                                                                                                                                   |
| ------------ | ------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 自建 MySQL     | 手动配置 DWB、page size、fsync                                                             |                                                                                                                                                                            |                                                                                                                                                                                                                                                                                          |
| RDS 单实例      | AWS 自动优化，使用 EBS 存储，<br>优化 fsync 性能，保证原子写入特性                                          |                                                                                                                                                                            |                                                                                                                                                                                                                                                                                          |
| RDS Multi-AZ | AWS 自动同步复制，<br>Primary 的 WAL 自动同步复制到 Standby                                         | 1. 实例级监控检测（数据库进行是否响应，是否执行简单 SQL 查询）<br />2. 系统级检查（操作系统是否响应，网络连接是否正常，网络连接阈值一般是 60-120s）<br />3. 存储检查（EBS 是否可访问，I/O 操作是否正常）<br />4. 跨 AZ 心跳检测（Primary 与 Standby 心跳检测，复制延迟监控） | Primary 实例故障，<br />1. RDS 检测到故障（60-120s），<br />2. Standby 使用 WAL 完成崩溃恢复（10-30s），<br />3. DNS 启动切换到 Standby（10-20s），<br />4. 应用重连（5-10s）。<br />总切换时间 2-3分钟                                                                                                                                |
| Aurora MySQL | **分布式 WAL 存储，<br>6副本，3AZ，<br>Quorum 机制 4/6 成功，<br>不需要传统 binlog 同步，<br>故障恢复从分钟降到秒级别** | 1. 存储层检查（**毫米级**，6个存储副本持续监控，Quorum 成功，自动检测修复损坏副本）<br />2. 计算层检查（**秒级**，数据库实例心跳检测，SQL 查询响应检查，网络连接状态检查，跨 AZ 检查）<br />3. 集群健康管理（持续监控所有实例，自动维护实例优先级，预先准备切换路径）                  | Aurora 的 Primary 和 Read Replic 底层共享存储，<br />1. 所以发生故障时，故障检测 10-15s，<br />2. 选举 Primary（1-2s），<br />3. 只需要切换存储层访问权限（**只读改为读写**），<br>数据库角色切换（**READ_REPLICA 切换为 PRIMARY**），<br>并**通知其他 Read Replica**，前边时间消耗 10-15s，<br />4. 然后再执行 DNS 切换（10-20s）即可，<br />5. 应用重连（5-10s）。<br />最短总切换时间 30s |

**MySQL 数据最安全配置**

```sql
innodb_flush_log_at_trx_commit = 1  -- 每次事务提交都刷盘
sync_binlog = 1                     -- 每次事务提交都同步 binlog

配置说明：
• = 1: 最安全，性能稍低（RDS Multi-AZ 默认）
• = 2: 每秒刷盘，性能更好，但可能丢失 1 秒数据
• = 0: 性能最好，但崩溃可能丢失数据（不推荐生产环境）
```

**AWS RDS Standby 和 Replica 区别**

| 特性    | Multi-AZ Standby | Read Replica                         |
| ----- | ---------------- | ------------------------------------ |
| 用途    | 高可用（HA）          | 读扩展 + 灾难恢复（DR）                       |
| 复制方式  | **同步复制（事务级）**    | **异步复制（binlog）**                     |
| 可访问性  | 不可访问（仅待命）        | 可读取（分担读负载）                           |
| 数据一致性 | **强一致性（0 延迟）**   | **最终一致性（有延迟）**                       |
| 故障切换  | 自动切换（2-3 分钟）     | 手动提升为 Primary<br />（因为最终一致性，可能会丢失数据） |
| 跨 AZ  | **必须不同 AZ**      | 可同 AZ 或跨 Region                      |
| 计费    | 包含在 Multi-AZ 费用  | 独立计费（按实例）                            |

**Multi AZ 和 Aurora 故障切换详细流程**

| 维度           | RDS Multi-AZ      | Aurora                    |
| ------------ | ----------------- | ------------------------- |
| Standby 可访问性 | 🔴 不可访问           | ✅ Read Replica 可读         |
| 故障检测时间       | 60-120 秒          | 5-10 秒                    |
| WAL 恢复时间     | 10-30 秒           | 0 秒（共享存储）**不需要考虑 WAL 问题** |
| 总切换时间        | 2-3 分钟            | 30 秒                      |
| 数据一致性        | 强一致性              | 强一致性                      |
| 读扩展能力        | 需要额外 Read Replica | **Read Replica 同时用于 HA**  |
| 成本           | 双倍实例费用            | 更高效（一份存储）                 |

```
场景 1: Multi-AZ（1 Primary + 1 Standby）
正常状态：
Primary (AZ-A) ──同步复制──> Standby (AZ-B) [不可访问]
                              ↓
                          仅用于故障切换

故障切换：
Primary 故障 → Standby 提升为 Primary → DNS 切换 → 完成（2-3 分钟）


场景 2: Multi-AZ + Read Replicas（1 Primary + 1 Standby + 3 Read Replicas）
正常状态：
Primary (AZ-A) ──同步复制──> Standby (AZ-B) [不可访问]
       ↓
    异步复制
       ↓
Read Replica 1 (AZ-A) ──可读──> 应用读请求
Read Replica 2 (AZ-B) ──可读──> 应用读请求
Read Replica 3 (AZ-C) ──可读──> 应用读请求

故障切换流程：
1. Primary 故障
2. Standby 自动提升为新 Primary（2-3 分钟）
3. Read Replicas 自动重新指向新 Primary
4. 应用写请求重连到新 Primary
5. 应用读请求继续访问 Read Replicas（可能短暂中断）

实际切换时间分解

Multi-AZ 故障切换详细时间线：
00:00 - Primary 实例故障
00:01 - RDS 检测到故障（健康检查失败）
00:60 - 确认故障（多次检查失败，避免误判）
01:00 - 开始故障切换流程
01:10 - Standby 应用剩余 WAL 日志（崩溃恢复）
01:40 - Standby 提升为 Primary
01:50 - DNS 记录更新（A 记录更新）
02:00 - 应用重连成功
02:30 - 完全恢复正常

总时间：2-3 分钟（取决于 WAL 恢复量）


关键点：
• **Standby 优先级最高**：故障切换永远是 Standby → Primary
• **Read Replica 不参与自动切换**：它们只是读副本，不能自动变成 Primary
• **手动提升 Read Replica**：如果 Primary 和 Standby 都故障，可以手动提升 Read Replica（但会丢失未复制的数据）


Aurora 故障切换时间线：

00:00 - Primary 实例故障
00:01 - 集群管理器检测到故障
00:05 - 选择优先级最高的 Read Replica（评估每个 Read Replica 优先级的健康状态，复制延迟时间）
00:10 - 提升为新 Primary（无需 WAL 恢复！），该过程存储层权限被选举出来的 Read Replica 会更改存储层权限从 “只读” 改为读写，数据库角色切换到     
                                         PRIMARY，并更新集群拓扑（更新集群元数据，通知其他 Read Replica，更新监控指标）
00:15 - DNS 切换（A 记录更新），geng
00:30 - 应用重连完成

总时间：30 秒左右
```

**WAL（Write-Ahead Logging，预写日志）和Double Write Buffer（双写缓冲区）在MySQL InnoDB存储引擎中是协同工作的两种机制，共同保证数据的安全性和一致性。**

##### WAL 与 Double Write Buffer 的关系

WAL 机制要求在修改数据页之前，先将对应的修改日志（Redo Log）写入磁盘，这样即使系统崩溃，也可以通过日志进行恢复。Redo Log 只记录变化的日志，用于恢复事务未完成时的数据。

然而，仅有Redo Log不足以解决在数据页实际写进磁盘过程中的“部分写入”(partial write)问题。因为数据页大小通常是16KB，而操作系统页面是**4KB**，如果写盘过程中断，可能只写入部分数据页，**导致数据页残缺**，**无法通过Redo Log正确恢复**。

Double Write Buffer机制解决了这个问题。其做法是：在刷写脏页到磁盘时，先将页面数据复制到内存中的Double Write Buffer，再一次性顺序地写入磁盘上的Double Write文件，这一步称为第一次写，确保写入是完整且连续的；接着再将数据页写回其真正的数据文件位置（第二次写），这次是离散写。这样，若写入数据文件时出现断电或系统崩溃，可以从Double Write Buffer中恢复完整页面，确保数据页不损坏。

总结来说，WAL保证的是日志先写，确保事务的持久性；Double Write Buffer保证的是数据页写入过程的完整性，防止“脏页”写磁盘时的断电损坏导致数据丢失或不一致。两者相辅相成，构成InnoDB的可靠持久化保障机制。

 **Double Write Buffer（双写缓冲）**

**双写原因：**

- MySQL InnoDB 使用默认 16KB 页大小，磁盘操作通常是以 **4KB 或 8KB** 为最小写入单位（操作系统磁盘块大小）。
- 当写入一个 16KB 页面时，底层可能分为多个块（4KB 或 8KB）分批写入磁盘。
- 如果在写入过程中，写了一半（比如4KB或8KB），系统发生崩溃，导致该页面只写入部分数据，另一部分残留旧数据或垃圾。
- 这会导致恢复时该页面变得“损坏”，产生数据丢失或出现不一致。

**双写机制**：

- 刷脏页时，先将脏页完整复制到内存中的 Double Write Buffer（大约2MB，连续存储多个页）。
- Double Write Buffer 中数据再以顺序写方式刷到磁盘共享表空间（共享表空间中的连续物理页）。
- 等 Double Write 写成功后，再写数据文件中对应的页。
- 若写数据文件时崩溃，恢复时可用共享表空间中的完整页副本进行还原，避免数据丢失。

该设计用额外一次写操作换取了页写入的原子性和完整性，规避了操作系统最小写入单位对大页写入风险的影响，保证崩溃恢复时页数据安全。

**MySQL 日志**

| 日志类型       | 主要作用               | 典型应用场景     | 是否必须开启         |
|:---------- |:------------------ |:---------- |:-------------- |
| 错误日志       | 记录服务器启动及运行错误       | 故障排查       | 必须             |
| 通用查询日志     | 记录所有客户端SQL请求       | 调试、审计      | 不建议常开，调试时开启    |
| 慢查询日志      | 记录执行时间长的查询         | 性能优化       | 建议开启           |
| 二进制日志      | 记录修改数据操作，支持复制与恢复   | 数据复制、恢复、审计 | 复制环境必须开启       |
| 中继日志       | 从服务器保存主服务器binlog事件 | 复制环境       | 复制环境从服务器使用     |
| 事务日志（Redo） | 事务日志，保证崩溃恢复可靠性     | 崩溃恢复、事务持久性 | InnoDB存储引擎自动管理 |
| 撤销日志（Undo） | 支持事务回滚和MVCC读一致性    | 事务回滚、隔离性实现 | InnoDB存储引擎自动管理 |
| DDL日志      | 记录DDL操作元数据         | 结构变更审计与恢复  | MySQL8.0及以上支持  |

**Redo 与 Undo 详细区别**

这两种日志协同工作，保障InnoDB事务的正确执行和数据安全。

| 特性   | Redo Log         | Undo Log          |
|:---- |:---------------- |:----------------- |
| 记录内容 | 数据修改后的物理变化（新值）   | 数据修改前的旧值          |
| 主要目标 | 保障事务持久性，崩溃恢复     | 保障事务原子性，实现回滚和MVCC |
| 写入时机 | 修改数据后，事务提交前同步写磁盘 | 修改数据前，事务执行时写入     |
| 日志类型 | 物理日志             | 逻辑日志              |
| 应用场景 | 系统崩溃后恢复          | 事务回滚、快照读等         |
| 生命周期 | 事务提交后保留，支持恢复     | 事务结束后可清理          |

Redo Log（重做日志）**记录数据页中按时间顺序记录每次修改的值，属于每次数据变更产生的增量日志**

- 作用
  - 记录**数据修改后的物理变化**，确保事务的**持久性**。
  - 即使系统崩溃，已提交的事务也可以通过重做日志将修改重做，恢复数据一致性。

- 工作原理
  - 在数据页被修改之前，先将修改内容写入Redo Log缓冲区，并持久化到磁盘。
  - 这是一种物理日志，记录的是数据的“新值”。
  - 事务提交时，必须确保Redo Log已经写入磁盘。
  - 崩溃恢复时，利用Redo Log进行恢复。

- 应用场景
  - 系统崩溃重启后的数据恢复。
  - 保证数据一致性和事务的持久性。

Undo Log（回滚日志）**记录每次数据修改前的旧值**

- 作用
  - 记录**数据修改前的旧值**，支持事务回滚和一致性读。
  - 保证事务的**原子性**和**隔离性**。

- 工作原理
  - 在执行修改操作之前，InnoDB将被修改行的旧数据写入Undo Log。
  - 如果事务回滚，可以利用Undo Log恢复数据状态。
  - 通过Undo Log结合Read View实现多版本并发控制（MVCC），保证一致性读（快照读）。

- 应用场景
  - 事务执行失败或回滚时恢复数据。
  - MVCC查询时，根据事务的快照读取数据的历史版本。

##### RPO & RTO

RPO（Recovery Point Objective）恢复点目标，**业务能容忍丢失的数据量**

- RPO 关注的是**数据的完整性和恢复的时间点**。

- 容忍最大数据丢失时间，RPO=4，发生故障时最多允许丢失过去 4 小时内数据

RTO（Recovery Time Objective）恢复时间目标，**业务能容忍停机时间**

- RTO 关注的是**业务的连续性和恢复速度**。

- 恢复到可用状态的所需最大时间，RTO=3，发生故障时系统必须在3消失内恢复

**什么策略能够同时减少 RPO 与 RTO？**

- 增量备份与差异备份结合
- 主从复制，RPO 接近零
- 快照
- DR（备份，信号灯，暖备，多活）

##### AWS RDS for MySQL 和 Aurora 关于 Double Write Buffer 的优化

- AWS RDS for MySQL 引入了“RDS Optimized Writes”功能，开启后可以跳过传统的 Double Write Buffer 机制，只需写一次数据页到持久存储。这得益于底层**AWS Nitro系统的硬件支持，使得16KB页的写入成为原子操作**，避免了传统MySQL中因写入中断导致的“半写页”问题
  - https://docs.aws.amazon.com/zh_cn/AmazonRDS/latest/UserGuide/rds-optimized-writes.html
- Aurora MySQL底层架构**依赖分布式存储，多副本写入和冗余机制提高容错性**。Aurora设计上天然减少了对Double Write Buffer的依赖，通过其多副本一致性协议和物理存储的原子写支持，实现了对数据页写入的安全保障和高性能。

##### AWS Aurora

**传统数据库设计原则，数据库的一切都与 I/O 相关**

- 提高 I/O 带宽
- 减少 I/O 数量

**Aurora 架构描述：**

- 计算层：三个 AZ 中有 Master，Slave（Master 推送 Redo Log 到共享存储，Slave 直接访问 Redo Log）
  - 每个实例分为 SQL，Transactions，Caching 三层

- 存储层：第一层 Shared storage volume（Master 和 Slave 共享存储层，主要存储 Redo Log），第二层 Storage node with SSDs（6 副本跨 3AZ，6写4成功的 Quorum 模型）
  - **Aurora 使用 Segment Storage，每个段为 10GB，在 10Gbps 网络下，1分钟内（通常10秒）可以恢复 10GB**

- Aurora Caching 独立于 DB 进程，数据库重启，Cache 数据还是热数据

- Aurora 的检查点是下放到了存储层，使用定期检查点
 ![[image-20250901165610444.png]]

![[image-20250901204954573.png]]
**Aurora 是一种云原生，采用存算分离，日志即数据库。**

1. Aurora底层存储层共享设计
   
   1. Aurora的存储层为**分布式共享存储**，主实例和所有副本**共享同一个底层存储卷**（分布在多个可用区的多个存储节点）。
   2. 主实例负责数据的写入，直接写入存储层，**副本访问相同存储卷读取数据，无需传统的物理或逻辑复制流程**。

2. 旁路（bypass）同步并非传统复制
   
   1. 旁路同步指的是**绕过副本自身的重放日志过程**，主实例将写入操作直接同步到底层存储副本节点。
   2. 副本只需从本地共享存储读取数据，**不需要执行过多复制回放**步骤，减少延迟和资源消耗。

3. 保持数据一致性
   
   1. 主实例直接写入记录Redo日志到存储层的所有节点，确保所有副本的存储数据在时间轴上的一致。
   2. 如果允许副本自己同步Redo日志，网络和同步复杂度增加，且无法充分利用共享存储的优势。

4. 降低延迟，提升性能
   
   1. 通过共享存储，数据同步几乎是同步完成，副本读取延迟极低，一般不到100ms。
   2. 传统基于逻辑或物理复制的多副本同步存在较大延迟瓶颈。

5. 故障恢复和高可用
   
   1. 主实例故障时，副本相比传统复制能更快切换，利用存储层最新的数据版本无缝接管。
   2. 这种架构简化了副本与主实例之间的同步流程，提高可用性和切换速度。

**副本自身日志重放过程**

副本自身的“重放日志过程”指传统数据库复制中，副本节点（Replica）接收到主节点传来的**二进制日志**（binlog）或**重做日志**（redo log）后，需要执行（重放）这些日志，将日志中描述的数据修改逐条应用到本地数据库中，以保持与主节点的数据一致。

```
传统副本复制流程：
主节点提交事务 → 生成日志 → 发送到副本 → 副本逐条重放日志 → 数据同步

Aurora复制流程：
主节点直接写入共享存储 → 副本通过共享存储读取数据，无需日志重放
```

---

#### MySQL 实战场景

##### 典型应用场景

| 场景 | 业务描述 | 数据量 | 并发量 | 关键指标 | 技术要点 |
|:---|:---|:---|:---|:---|:---|
| **电商订单系统** | 订单创建、支付、发货、退款 | 千万-亿级 | 1万 TPS | 强一致性、ACID 事务 | 分库分表、读写分离 |
| **用户中心** | 用户注册、登录、资料管理 | 亿级 | 10万 QPS | 读多写少、高可用 | 缓存、主从复制 |
| **内容管理系统** | 文章、评论、点赞、收藏 | 百万-千万级 | 5000 QPS | 复杂查询、JOIN | 索引优化、分页 |
| **金融交易** | 转账、余额、流水、对账 | 千万级 | 5000 TPS | 强一致性、审计 | 事务、日志、备份 |
| **库存系统** | 库存扣减、预占、释放 | 百万级 | 2万 TPS | 高并发写入、防超卖 | 乐观锁、分布式锁 |

---

##### MySQL 单表数据量上限指南

| 数据量级别 | 行数 | 存储大小 | 查询性能 | DDL耗时 | 备份恢复 | 性能瓶颈 | 推荐方案 |
|:---|:---|:---|:---|:---|:---|:---|:---|
| **推荐** | **2000万-5000万** | **20-50GB** | 10-50ms | 5-10分钟 | 10-30分钟 | 无明显瓶颈 | 标准索引设计即可 |
| **可接受** | 5000万-1亿 | 50-100GB | 50-200ms | 30分钟-1小时 | 1-2小时 | 索引深度增加<br>回表开销大 | 分区表<br>索引优化<br>缓存预热 |
| **极限** | 1亿-5亿 | 100-500GB | 200ms-2秒 | 数小时 | 数小时 | B-Tree页分裂频繁<br>锁竞争严重<br>缓冲池命中率低 | 分库分表（必须）<br>读写分离<br>冷热分离 |
| **不可用** | >5亿 | >500GB | 秒-分钟级 | 数天 | 数天 | 全表扫描超时<br>DDL锁表过久<br>备份恢复困难 | 迁移到分布式数据库<br>（TiDB/Spanner）<br>或OLAP数据库<br>（ClickHouse/BigQuery） |

**关键决策点：**

| 场景 | 数据量阈值 | 必须采取的行动 |
|:---|:---|:---|
| **OLTP 高并发** | > 2000万行 | 分库分表 |
| **读多写少** | > 5000万行 | 分区表 + 读写分离 |
| **历史数据** | > 1亿行 | 冷热分离（归档到 Hive/ClickHouse） |
| **分析查询** | > 5000万行 | 迁移到 OLAP 数据库 |

---

##### 常见问题与解决方案

##### 问题 1：慢查询优化

**现象：**
```sql
-- 查询耗时 5 秒，影响用户体验
SELECT * FROM orders 
WHERE user_id = 123 
AND status = 'pending' 
AND create_time > '2024-01-01'
ORDER BY create_time DESC;
```

**原因分析：**
```sql
-- 使用 EXPLAIN 分析
EXPLAIN SELECT * FROM orders 
WHERE user_id = 123 
AND status = 'pending' 
AND create_time > '2024-01-01';

-- 结果：
-- type: ALL（全表扫描）
-- rows: 1000000（扫描 100 万行）
-- Extra: Using where; Using filesort

问题：
1. 缺少复合索引，导致全表扫描
2. SELECT * 返回不必要的字段
3. ORDER BY 无法使用索引，需要文件排序
```

**解决方案：**
```sql
-- 方案 1：创建复合索引（最左前缀原则）
CREATE INDEX idx_user_status_time 
ON orders(user_id, status, create_time);

-- 方案 2：优化查询（只返回需要的字段）
SELECT order_id, amount, create_time 
FROM orders 
WHERE user_id = 123 
AND status = 'pending' 
AND create_time > '2024-01-01'
ORDER BY create_time DESC
LIMIT 20;

-- 方案 3：使用覆盖索引（避免回表）
CREATE INDEX idx_user_status_time_amount 
ON orders(user_id, status, create_time, order_id, amount);

-- 验证优化效果
EXPLAIN SELECT order_id, amount, create_time 
FROM orders 
WHERE user_id = 123 
AND status = 'pending' 
AND create_time > '2024-01-01'
ORDER BY create_time DESC
LIMIT 20;

-- 优化后结果：
-- type: ref（索引查询）
-- rows: 100（只扫描 100 行）
-- Extra: Using index（覆盖索引，无需回表）
-- 查询时间：5s → 10ms（500 倍提升）
```

**最佳实践：**
- ✅ 遵循最左前缀原则设计复合索引
- ✅ 避免 SELECT *，只查询需要的字段
- ✅ 使用覆盖索引避免回表
- ✅ 定期分析慢查询日志：`SHOW VARIABLES LIKE 'slow_query%'`
- ✅ 使用 EXPLAIN 验证执行计划
- ✅ 索引字段顺序：等值查询 > 范围查询 > 排序字段

---

##### 问题 2：死锁问题

**现象：**
```
ERROR 1213 (40001): Deadlock found when trying to get lock; 
try restarting transaction
```

**原因分析：**
```sql
-- 事务 A（10:00:00.000）
BEGIN;
UPDATE orders SET status='paid' WHERE order_id=1;  -- 持有锁 1
-- 等待 1 秒
UPDATE orders SET status='paid' WHERE order_id=2;  -- 等待锁 2
COMMIT;

-- 事务 B（10:00:00.500，同时执行）
BEGIN;
UPDATE orders SET status='paid' WHERE order_id=2;  -- 持有锁 2
-- 等待 1 秒
UPDATE orders SET status='paid' WHERE order_id=1;  -- 等待锁 1 → 死锁！
COMMIT;

死锁原因：
1. 两个事务交叉持有和等待锁
2. 形成循环等待：A 等待 B，B 等待 A
3. InnoDB 检测到死锁，自动回滚代价小的事务
```

**解决方案：**
```sql
-- 方案 1：统一访问顺序（推荐）
-- 所有事务按主键升序访问资源
BEGIN;
UPDATE orders SET status='paid' 
WHERE order_id IN (1, 2) 
ORDER BY order_id;  -- 强制按 ID 升序
COMMIT;

-- 方案 2：使用 SELECT ... FOR UPDATE 预先加锁
BEGIN;
-- 先按顺序锁定所有行
SELECT * FROM orders 
WHERE order_id IN (1, 2) 
ORDER BY order_id 
FOR UPDATE;

-- 再执行更新
UPDATE orders SET status='paid' WHERE order_id IN (1, 2);
COMMIT;

-- 方案 3：降低事务粒度（拆分事务）
-- 事务 1
BEGIN;
UPDATE orders SET status='paid' WHERE order_id=1;
COMMIT;

-- 事务 2
BEGIN;
UPDATE orders SET status='paid' WHERE order_id=2;
COMMIT;

-- 方案 4：使用乐观锁（version 字段）
BEGIN;
UPDATE orders 
SET status='paid', version=version+1 
WHERE order_id=1 AND version=10;  -- 检查版本号
-- 如果 affected_rows=0，说明被其他事务修改，重试
COMMIT;

-- 方案 5：设置锁等待超时
SET innodb_lock_wait_timeout = 5;  -- 5 秒超时
```

**死锁检测和分析：**
```sql
-- 查看最近的死锁日志
SHOW ENGINE INNODB STATUS\G

-- 关键信息：
-- *** (1) TRANSACTION:
--   TRANSACTION 12345, ACTIVE 2 sec starting index read
--   mysql tables in use 1, locked 1
--   LOCK WAIT 2 lock struct(s), heap size 1136, 1 row lock(s)
--   MySQL thread id 10, OS thread handle 140123456789, query id 100 localhost root updating
--   UPDATE orders SET status='paid' WHERE order_id=2
--   
-- *** (2) TRANSACTION:
--   TRANSACTION 12346, ACTIVE 1 sec starting index read
--   mysql tables in use 1, locked 1
--   3 lock struct(s), heap size 1136, 2 row lock(s)
--   MySQL thread id 11, OS thread handle 140123456790, query id 101 localhost root updating
--   UPDATE orders SET status='paid' WHERE order_id=1
--   *** (2) HOLDS THE LOCK(S):
--   RECORD LOCKS space id 58 page no 3 n bits 72 index PRIMARY of table `mydb`.`orders` trx id 12346 lock_mode X locks rec but not gap
--   *** (2) WAITING FOR THIS LOCK TO BE GRANTED:
--   RECORD LOCKS space id 58 page no 3 n bits 72 index PRIMARY of table `mydb`.`orders` trx id 12346 lock_mode X locks rec but not gap waiting
--   *** WE ROLL BACK TRANSACTION (2)
```

**最佳实践：**
- ✅ 统一资源访问顺序（按主键升序）
- ✅ 缩短事务时间，减少锁持有时间
- ✅ 使用合适的索引避免锁升级（行锁 → 表锁）
- ✅ 避免大事务，拆分为小事务
- ✅ 使用乐观锁代替悲观锁（高并发场景）
- ✅ 监控死锁日志：`SHOW ENGINE INNODB STATUS`
- ✅ 设置合理的锁等待超时：`innodb_lock_wait_timeout`
- ✅ 业务层实现重试机制

---

##### 问题 3：主从延迟

**现象：**
```
场景：用户下单后立即查询订单，显示"订单不存在"
原因：主库写入成功，从库复制延迟 1-5 秒
```

**原因分析：**
```sql
-- 主从复制流程
主库（Master）：
1. 执行 SQL：INSERT INTO orders ...
2. 写入 Binlog
3. 返回客户端成功

从库（Slave）：
1. IO 线程读取主库 Binlog
2. 写入 Relay Log
3. SQL 线程执行 Relay Log
4. 数据可见（延迟 1-5 秒）

延迟原因：
1. 主从异步复制（默认）
2. 从库负载过高（慢查询、大事务）
3. 网络延迟
4. 从库硬件性能差
5. 单线程复制（MySQL 5.6 之前）
```

**解决方案：**
```sql
-- 方案 1：强制读主库（关键业务）
-- 写入后立即读取，直接查主库
// 伪代码
$db->master()->query("INSERT INTO orders ...");
$order = $db->master()->query("SELECT * FROM orders WHERE order_id = ?");

-- 方案 2：半同步复制（MySQL 5.5+）
-- 主库等待至少 1 个从库确认后才返回成功

-- 主库配置
INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';
SET GLOBAL rpl_semi_sync_master_enabled = 1;
SET GLOBAL rpl_semi_sync_master_timeout = 1000;  -- 1 秒超时

-- 从库配置
INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';
SET GLOBAL rpl_semi_sync_slave_enabled = 1;

-- 验证半同步状态
SHOW STATUS LIKE 'Rpl_semi_sync%';
-- Rpl_semi_sync_master_status: ON
-- Rpl_semi_sync_master_clients: 2

-- 方案 3：并行复制（MySQL 5.7+）
-- 从库多线程并行执行 Relay Log

-- 从库配置
STOP SLAVE;
SET GLOBAL slave_parallel_type = 'LOGICAL_CLOCK';
SET GLOBAL slave_parallel_workers = 4;  -- 4 个并行线程
START SLAVE;

-- 方案 4：应用层延迟补偿
-- 写入后等待 100ms 再读取
$db->master()->query("INSERT INTO orders ...");
usleep(100000);  // 等待 100ms
$order = $db->slave()->query("SELECT * FROM orders WHERE order_id = ?");

-- 方案 5：使用缓存
-- 写入时同时更新缓存
$db->master()->query("INSERT INTO orders ...");
$redis->setex("order:{$orderId}", 300, json_encode($order));
// 读取时优先从缓存读
$order = $redis->get("order:{$orderId}") ?: $db->slave()->query(...);

-- 方案 6：监控延迟并路由
-- 检查从库延迟，超过阈值则读主库
SHOW SLAVE STATUS\G
-- Seconds_Behind_Master: 2（延迟 2 秒）

// 伪代码
$delay = $db->slave()->query("SHOW SLAVE STATUS")->Seconds_Behind_Master;
if ($delay > 1) {
    // 延迟超过 1 秒，读主库
    $order = $db->master()->query(...);
} else {
    // 延迟可接受，读从库
    $order = $db->slave()->query(...);
}
```

**监控主从延迟：**
```sql
-- 从库执行
SHOW SLAVE STATUS\G

-- 关键指标：
-- Slave_IO_Running: Yes（IO 线程正常）
-- Slave_SQL_Running: Yes（SQL 线程正常）
-- Seconds_Behind_Master: 2（延迟 2 秒）
-- Master_Log_File: mysql-bin.000003
-- Read_Master_Log_Pos: 12345
-- Relay_Log_File: relay-bin.000002
-- Relay_Log_Pos: 6789

-- 计算延迟
-- 延迟 = 当前时间 - 从库执行的最后一个事务的时间戳
```

**最佳实践：**
- ✅ 关键业务读主库（订单、支付、库存）
- ✅ 启用半同步复制（牺牲少量性能换一致性）
- ✅ 启用并行复制（MySQL 5.7+）
- ✅ 监控主从延迟，设置告警阈值（> 5 秒）
- ✅ 从库硬件配置不低于主库
- ✅ 避免从库执行慢查询（影响复制）
- ✅ 使用缓存减少数据库压力
- ✅ 读写分离中间件：ProxySQL、MaxScale
- ✅ 定期检查主从数据一致性：`pt-table-checksum`

---

##### 问题 4：大表分页查询慢

**现象：**
```sql
-- 深度分页，耗时 10 秒
SELECT * FROM orders 
ORDER BY create_time DESC 
LIMIT 1000000, 20;

-- 用户翻到第 50000 页时，查询超时
```

**原因分析：**
```sql
-- MySQL 执行流程
1. 扫描索引，找到前 1000020 行
2. 加载这 1000020 行的完整数据
3. 丢弃前 1000000 行
4. 返回最后 20 行

-- EXPLAIN 分析
EXPLAIN SELECT * FROM orders 
ORDER BY create_time DESC 
LIMIT 1000000, 20;

-- 结果：
-- type: index
-- rows: 1000020（扫描 100 万行）
-- Extra: Using index（只扫描索引，但仍需回表 100 万次）

问题：
1. 大量无用数据被加载和丢弃
2. 回表操作次数 = LIMIT offset + count
3. offset 越大，性能越差
```

**解决方案：**
```sql
-- 方案 1：子查询优化（覆盖索引）
-- 先通过索引找到 ID，再回表
SELECT * FROM orders 
WHERE order_id >= (
  SELECT order_id FROM orders 
  ORDER BY create_time DESC 
  LIMIT 1000000, 1
)
ORDER BY create_time DESC 
LIMIT 20;

-- 性能提升：10s → 2s

-- 方案 2：记录上次查询位置（推荐）
-- 第一次查询
SELECT order_id, create_time, amount 
FROM orders 
ORDER BY create_time DESC, order_id DESC 
LIMIT 20;

-- 返回最后一条：order_id=123456, create_time='2024-01-01 10:00:00'

-- 第二次查询（下一页）
SELECT order_id, create_time, amount 
FROM orders 
WHERE create_time < '2024-01-01 10:00:00'
   OR (create_time = '2024-01-01 10:00:00' AND order_id < 123456)
ORDER BY create_time DESC, order_id DESC 
LIMIT 20;

-- 性能：始终保持 10ms 以内

-- 方案 3：使用 Elasticsearch 做分页
-- MySQL 只存储数据，Elasticsearch 做搜索和分页
// 伪代码
$es->search([
    'index' => 'orders',
    'body' => [
        'from' => 1000000,
        'size' => 20,
        'sort' => [['create_time' => 'desc']]
    ]
]);

-- 方案 4：限制最大页数
-- 前端只允许翻到前 100 页
if ($page > 100) {
    throw new Exception('最多只能查看前 100 页');
}

-- 方案 5：使用延迟关联
SELECT * FROM orders 
INNER JOIN (
    SELECT order_id FROM orders 
    ORDER BY create_time DESC 
    LIMIT 1000000, 20
) AS tmp USING(order_id);
```

**最佳实践：**
- ✅ 避免深度分页（LIMIT 10000, 20）
- ✅ 使用游标分页（记录上次位置）
- ✅ 大数据量搜索用 Elasticsearch
- ✅ 前端限制最大页数（如 100 页）
- ✅ 使用覆盖索引减少回表
- ✅ 考虑业务场景：用户很少翻到第 1000 页

---

##### 问题 5：在线 DDL 锁表

**现象：**
```sql
-- 添加索引，锁表 30 分钟，业务中断
ALTER TABLE orders ADD INDEX idx_user_id(user_id);

-- 所有写入操作被阻塞
INSERT INTO orders ... -- 等待中
UPDATE orders ... -- 等待中
```

**原因分析：**
```
MySQL 5.5 及之前：
1. 创建临时表
2. 复制数据到临时表
3. 删除原表
4. 重命名临时表
5. 全程锁表（LOCK TABLE）

MySQL 5.6+ Online DDL：
1. 支持部分 DDL 不锁表
2. 但仍有短暂的元数据锁（MDL）
```

**解决方案：**
```sql
-- 方案 1：使用 Online DDL（MySQL 5.6+）
ALTER TABLE orders 
ADD INDEX idx_user_id(user_id), 
ALGORITHM=INPLACE, 
LOCK=NONE;

-- ALGORITHM 选项：
-- COPY: 创建临时表（锁表）
-- INPLACE: 原地修改（不锁表或短暂锁表）

-- LOCK 选项：
-- NONE: 不锁表（推荐）
-- SHARED: 允许读，不允许写
-- EXCLUSIVE: 完全锁表

-- 方案 2：使用 pt-online-schema-change（推荐）
-- Percona Toolkit 工具，零停机 DDL
pt-online-schema-change \
  --alter "ADD INDEX idx_user_id(user_id)" \
  --execute \
  D=mydb,t=orders

-- 工作原理：
-- 1. 创建新表（带索引）
-- 2. 创建触发器同步数据
-- 3. 分批复制数据（不阻塞业务）
-- 4. 原子切换表名

-- 方案 3：使用 gh-ost（GitHub 开源）
gh-ost \
  --user="root" \
  --password="password" \
  --host="localhost" \
  --database="mydb" \
  --table="orders" \
  --alter="ADD INDEX idx_user_id(user_id)" \
  --execute

-- 方案 4：业务低峰期执行
-- 凌晨 2-4 点执行 DDL
-- 配合主从切换，减少影响

-- 方案 5：先在从库执行，再主从切换
-- 步骤：
-- 1. 从库执行 DDL（不影响主库）
-- 2. 验证从库正常
-- 3. 主从切换（从库提升为主库）
-- 4. 原主库执行 DDL
```

**监控 DDL 进度：**
```sql
-- MySQL 5.7+
SELECT * FROM performance_schema.events_stages_current
WHERE EVENT_NAME LIKE 'stage/innodb/alter%'\G

-- 关键字段：
-- WORK_COMPLETED: 已完成的工作量
-- WORK_ESTIMATED: 预估总工作量
-- 进度 = WORK_COMPLETED / WORK_ESTIMATED * 100%
```

**最佳实践：**
- ✅ 使用 pt-online-schema-change 或 gh-ost
- ✅ 业务低峰期执行 DDL
- ✅ 先在从库测试，再在主库执行
- ✅ 监控 DDL 进度和锁等待
- ✅ 设置 DDL 超时：`lock_wait_timeout`
- ✅ 大表 DDL 分批执行（如按月分区）
- ✅ 提前规划索引，避免频繁 DDL

---

##### 问题 6：分库分表策略

**现象：**
```
单表数据量：1 亿行
表大小：100GB
查询变慢：全表扫描 30 秒
写入变慢：索引维护耗时
备份困难：全量备份 8 小时
```

**分库分表方案：**
```sql
-- 方案 1：垂直分库（按业务模块）
-- 原始：单库
mydb
├── users（用户表）
├── orders（订单表）
├── products（商品表）
└── payments（支付表）

-- 垂直分库：按业务拆分
user_db
└── users

order_db
├── orders
└── order_items

product_db
└── products

payment_db
└── payments

-- 优点：
-- ✅ 业务隔离，互不影响
-- ✅ 扩展灵活
-- 缺点：
-- ❌ 无法解决单表数据量大的问题
-- ❌ 跨库 JOIN 困难

-- 方案 2：水平分表（按数据量）
-- 单库多表
mydb
├── orders_0（0-999 万）
├── orders_1（1000-1999 万）
├── orders_2（2000-2999 万）
└── orders_3（3000-3999 万）

-- 路由规则：
-- table_index = order_id % 4

-- 优点：
-- ✅ 单表数据量减少
-- ✅ 查询性能提升
-- 缺点：
-- ❌ 单库压力仍然大
-- ❌ 跨表查询困难

-- 方案 3：水平分库分表（推荐）
-- 多库多表
order_db_0
├── orders_0
└── orders_1

order_db_1
├── orders_2
└── orders_3

-- 路由规则：
-- db_index = order_id / 2 % 2
-- table_index = order_id % 2

-- 示例：
-- order_id=5 → db_index=1, table_index=1 → order_db_1.orders_1
-- order_id=8 → db_index=0, table_index=0 → order_db_0.orders_0
```

**分片键选择：**
```sql
-- 分片键选择原则：
-- 1. 高基数（Cardinality）：值的种类多
-- 2. 查询频繁：大部分查询都带分片键
-- 3. 数据均匀：避免数据倾斜

-- 常见分片键：
-- ✅ user_id：用户维度查询多
-- ✅ order_id：订单维度查询多
-- ❌ status：基数低，数据倾斜
-- ❌ create_time：时间递增，热点问题

-- 示例：订单表分片
-- 分片键：user_id（用户维度查询多）
-- 路由规则：db_index = user_id % 4

-- 查询示例：
-- ✅ 单分片查询（高效）
SELECT * FROM orders WHERE user_id = 123;
-- 路由到：order_db_3.orders_x

-- ❌ 跨分片查询（低效）
SELECT * FROM orders WHERE status = 'pending';
-- 需要查询所有分片，再聚合结果
```

**使用 ShardingSphere：**
```yaml
# sharding-jdbc 配置
spring:
  shardingsphere:
    datasource:
      names: ds0,ds1
      ds0:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/order_db_0
      ds1:
        type: com.zaxxer.hikari.HikariDataSource
        driver-class-name: com.mysql.cj.jdbc.Driver
        jdbc-url: jdbc:mysql://localhost:3306/order_db_1
    
    rules:
      sharding:
        tables:
          orders:
            actual-data-nodes: ds$->{0..1}.orders_$->{0..1}
            database-strategy:
              standard:
                sharding-column: user_id
                sharding-algorithm-name: db-inline
            table-strategy:
              standard:
                sharding-column: user_id
                sharding-algorithm-name: table-inline
        
        sharding-algorithms:
          db-inline:
            type: INLINE
            props:
              algorithm-expression: ds$->{user_id % 2}
          table-inline:
            type: INLINE
            props:
              algorithm-expression: orders_$->{user_id % 2}
```

**分库分表的挑战：**
```sql
-- 1. 跨分片 JOIN
-- ❌ 无法直接 JOIN
SELECT o.*, u.name 
FROM orders o 
JOIN users u ON o.user_id = u.user_id;

-- ✅ 应用层 JOIN
-- 1. 查询 orders
-- 2. 提取 user_id 列表
-- 3. 查询 users（IN 查询）
-- 4. 应用层合并

-- 2. 分布式事务
-- ❌ 跨库事务无法保证 ACID
BEGIN;
INSERT INTO order_db_0.orders ...;
INSERT INTO order_db_1.order_items ...;
COMMIT;

-- ✅ 使用分布式事务框架
-- Seata、TCC、Saga

-- 3. 全局唯一 ID
-- ❌ 自增 ID 会重复
-- ✅ 雪花算法（Snowflake）
-- ✅ UUID
-- ✅ 数据库号段模式

-- 4. 扩容困难
-- 2 个分片 → 4 个分片
-- 需要数据迁移和重新路由
```

**最佳实践：**
- ✅ 提前规划分片数量（2^n，便于扩容）
- ✅ 选择合适的分片键（高基数、查询频繁）
- ✅ 使用中间件：ShardingSphere、Vitess
- ✅ 避免跨分片 JOIN 和事务
- ✅ 使用全局唯一 ID（雪花算法）
- ✅ 冷热数据分离（历史数据归档）
- ✅ 监控各分片负载均衡
- ✅ 预留扩容空间（如 4 个分片预留到 8 个）

---

## NoSQL 数据库实现

### 数据库实现详解

#### 键值数据库

##### Redis

##### Redis 整体架构

```
┌─────────────────────────────────────────────────────────────────┐
│                         客户端层                                 │
│  Redis CLI │ 应用程序 │ Jedis/Lettuce │ redis-py │ ioredis       │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                    协议层（RESP Protocol）                        │
│         请求解析 │ 命令序列化 │ 响应格式化 │ Pipeline 支持           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      网络层（IO 多路复用）                         │
│       epoll (Linux) / kqueue (macOS) / select (Windows)         │
│              单线程（6.0 前）│ 多线程 IO（6.0+）                    │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        事件处理层                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ 文件事件处理器  │  │ 时间事件处理器 │  │ 事件调度器     │           │
│  │ (连接/读/写)   │  │ (定时任务)    │  │ (aeEventLoop)│           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        命令处理层                                 │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ 命令表         │ │ 命令执行器     │  │ Lua 脚本引擎  │           │
│  │(Command Table)│ │ (单线程)      │  │(EVAL/EVALSHA)│           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      数据库层（DB 0-15）                          │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  键空间 (dict)：存储所有 key-value                          │   │
│  │  过期字典 (expires)：存储 key 的过期时间                     │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                         内存数据结构层                            │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  数据类型：String │ List │ Hash │ Set │ Sorted Set │ Stream│   │
│  └──────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────┐   │
│  │  底层编码：SDS │ ZipList │ QuickList │ HashTable │ SkipList│   │ 
│  │            IntSet │ Listpack │ Radix Tree                │   │
│  └──────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                         内存管理层                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ 淘汰策略      │  │ 过期删除       │  │ 内存碎片整理   │          │
│  │ (8 种策略)    │  │ (惰性/定期)    │  │(activedefrag)│          │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        后台线程层（BIO）                          │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ 异步删除      │  │ AOF 重写      │  │ RDB 持久化    │           │
│  │ (UNLINK)     │  │(BGREWRITEAOF)│  │ (BGSAVE)     │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        持久化层                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ RDB (快照)    │  │ AOF (日志)   │  │ 混合持久化     │           │
│  │ dump.rdb     │  │appendonly.aof│  │ (RDB+AOF)    │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                        高可用层                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ 主从复制      │  │ 哨兵 (Sentinel)│ │ 集群 (Cluster)│           │
│  │Master-Replica│  │ 自动故障转移    │ │ 分片 + 高可用  │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│                      辅助功能层                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ 发布订阅      │  │ 事务          │  │ 慢查询日志     │           │
│  │ (Pub/Sub)    │  │ (MULTI/EXEC) │  │ (Slowlog)    │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

**核心组件详解：**

| 层级        | 组件             | 职责     | 关键点             |
| --------- | -------------- | ------ | --------------- |
| **协议层**   | RESP           | 序列化协议  | 简单文本协议，易于解析     |
| **网络层**   | IO 多路复用        | 高并发连接  | 单线程非阻塞 IO       |
| **事件处理层** | aeEventLoop    | 事件驱动   | Reactor 模式      |
| **命令处理层** | 命令表 + 执行器      | 命令分派执行 | 单线程保证原子性        |
| **数据库层**  | 键空间 + 过期字典     | 数据存储   | 16 个 DB，默认 DB 0 |
| **数据结构层** | 9 种数据类型        | 数据抽象   | 自动编码转换优化        |
| **内存管理层** | 淘汰 + 过期 + 碎片整理 | 内存优化   | 8 种淘汰策略         |
| **后台线程层** | BIO 线程池        | 异步任务   | 避免阻塞主线程         |
| **持久化层**  | RDB/AOF        | 数据持久化  | 混合持久化最优         |
| **高可用层**  | 主从/哨兵/集群       | 高可用    | 自动故障转移          |
| **辅助功能层** | Pub/Sub/事务/慢查询 | 扩展功能   | 增强功能性           |

**关键设计：**

| 设计 | 说明 | 优势 |
|------|------|------|
| **单线程命令执行** | 所有命令在主线程串行执行 | 避免锁竞争，保证原子性 |
| **IO 多路复用** | 一个线程处理多个连接 | 高并发，低资源消耗 |
| **事件驱动** | Reactor 模式 | 非阻塞，高性能 |
| **多层数据结构** | 抽象类型 + 底层编码 | 自动优化，节省内存 |
| **后台线程** | 异步处理耗时操作 | 不阻塞主线程 |
| **COW 机制** | fork 子进程持久化 | 不影响主进程服务 |

**数据流转示例：**
```
客户端发送 SET key value
    ↓
RESP 协议解析
    ↓
IO 多路复用接收
    ↓
文件事件处理器读取命令
    ↓
命令表查找 SET 命令
    ↓
命令执行器执行（单线程）
    ↓
写入键空间（dict）
    ↓
AOF 缓冲区追加日志
    ↓
返回响应给客户端
    ↓
后台线程异步刷盘（AOF）
```

**Redis 单线程模型（6.0 前）：**
```
为什么单线程还这么快？
1. 纯内存操作（ns 级别）
2. IO 多路复用（避免阻塞）
3. 高效的数据结构
4. 避免线程切换和锁竞争
5. 简单的命令（大部分 O(1)）
```

**Redis 多线程 IO（6.0+）：**
```
主线程：命令执行（保持单线程）
IO 线程：网络读写（多线程加速）
后台线程：持久化、删除（异步处理）
```

---

##### Redis 数据类型与底层实现

| #   | 数据类型                 | 底层编码                               | 使用场景          | 时间复杂度           | 特点                                  |
| --- | -------------------- | ---------------------------------- | ------------- | --------------- | ----------------------------------- |
| 1   | **String（字符串）**      | SDS（Simple Dynamic String）         | 缓存、计数器、分布式锁   | O(1)            | 二进制安全，最大 512MB，支持整数操作               |
| 2   | **List（列表）**         | QuickList（LinkedList + ZipList）    | 消息队列、时间线、栈/队列 | 头尾 O(1)，中间 O(N) | 双向链表，支持 LPUSH/RPUSH/LPOP/RPOP       |
| 3   | **Hash（哈希）**         | ZipList（小数据）/ HashTable（大数据）       | 对象存储、购物车      | O(1)            | 类似 Map，节省内存（vs 多个 String key）       |
| 4   | **Set（集合）**          | IntSet（整数）/ HashTable（字符串）         | 标签、共同好友、去重    | O(1)            | 无序不重复，支持交集/并集/差集                    |
| 5   | **Sorted Set（有序集合）** | ZipList（小数据）/ SkipList + HashTable | 排行榜、延时队列、范围查询 | O(log N)        | 按 score 排序，支持范围查询                   |
| 6   | **Bitmap（位图）**       | String + 位运算                       | 签到、在线状态、布隆过滤器 | O(1)            | 本质是 String，通过位运算操作二进制位，1 亿用户仅需 12MB |
| 7   | **HyperLogLog**      | 稀疏/密集编码                            | UV 统计、基数估算    | O(1)            | 0.81% 误差，固定 12KB 内存                 |
| 8   | **Geospatial（地理位置）** | Sorted Set（底层）                     | 附近的人、地理围栏     | O(log N)        | 基于 GeoHash 编码                       |
| 9   | **Stream（流）**        | Radix Tree + Listpack              | 消息队列、日志采集     | O(1) 追加         | 支持消费者组、持久化、ACK 机制                   |

**底层数据结构详解：**

| # | 底层结构 | 特点 | 使用条件 | 优势 |
|---|---------|------|---------|------|
| 1 | **SDS** | 动态字符串，记录长度 | 所有 String 类型 | O(1) 获取长度，二进制安全，减少内存分配 |
| 2 | **ZipList** | 连续内存块，紧凑存储 | 元素少且小（< 64 字节，< 512 个）| 节省内存，缓存友好 |
| 3 | **QuickList** | ZipList 的双向链表 | List 类型 | 平衡内存和性能 |
| 4 | **HashTable** | 哈希表，渐进式 rehash | Hash/Set 大数据 | O(1) 查询，动态扩容 |
| 5 | **IntSet** | 有序整数数组 | Set 全是整数且 < 512 个 | 节省内存，二分查找 |
| 6 | **SkipList** | 跳表，多层索引 | Sorted Set | O(log N) 查询/插入，支持范围查询 |
| 7 | **Listpack** | ZipList 改进版 | Stream、小 Hash/ZSet（Redis 7.0+）| 避免连锁更新，更高效 |

**编码转换规则（自动优化）：**

```
Hash:
  ZipList → HashTable（元素 > 512 或单个值 > 64 字节）

Set:
  IntSet → HashTable（非整数或元素 > 512）

Sorted Set:
  ZipList → SkipList + HashTable（元素 > 128 或单个值 > 64 字节）

List:
  始终使用 QuickList（Redis 3.2+ 统一实现）
```

**Redis 事务**

Redis 提供事务机制支持原子操作。事务通过 MULTI、EXEC、WATCH 等命令实现，在一个事务块中多个命令被缓存，执行 EXEC 时依次执行，保证命令的全部成功或全部失败。**Redis 事务不支持回滚**，但可以通过 **WATCH 监控键变化来实现乐观锁**，保证事务的可靠性。

**重要说明：**

**1. 淘汰策略配置（只能选择一种）：**

```bash
# redis.conf - 全局配置，只能设置一个策略
maxmemory 2gb
maxmemory-policy allkeys-lru  # 只能选择一种策略

# 运行时修改
CONFIG SET maxmemory-policy allkeys-lfu
```

**🔴 常见误区：**
- 不能同时使用多种淘汰策略
- 淘汰策略是全局配置，作用于整个 Redis 实例

**✅ 如果需要不同策略：**
```
方案 1：使用多个 Redis 实例
  ├─ 实例 1（6379）：allkeys-lru（通用缓存）
  ├─ 实例 2（6380）：volatile-ttl（会话存储）
  └─ 实例 3（6381）：noeviction（重要数据）

方案 2：应用层控制
  ├─ 为不同类型的 key 设置不同的 TTL
  └─ 使用 key 前缀区分重要性
```

**策略选择指南：**

| 场景 | 推荐策略 | 原因 |
|------|---------|------|
| 纯缓存 | allkeys-lru | 淘汰最少使用的，保留热点数据 |
| 热点数据 | allkeys-lfu | 淘汰访问频率低的（Redis 4.0+）|
| 部分可淘汰 | volatile-lru | 只淘汰设置了过期时间的 key |
| 不允许淘汰 | noeviction | 内存满时拒绝写入 |
| 会话存储 | volatile-ttl | 优先淘汰即将过期的 |

**2. 集群模式下的事务限制：**

**🔴 跨 Slot 无法保证事务一致性：**

```
┌─────────────────────────────────────────────────────────────┐
│              Redis 集群 Slot 分布                            │
└─────────────────────────────────────────────────────────────┘

节点 A                节点 B                节点 C
Slot 0-5460          Slot 5461-10922       Slot 10923-16383
    │                     │                      │
user:1000 ───────────────>│                      │
(Slot 5798)               │                      │
                          │                      │
order:2000 ─────────────────────────────────────>│
(Slot 12182)              │                      │

# 跨 Slot 事务失败
MULTI
SET user:1000 "Alice"   # 节点 B
SET order:2000 "..."    # 节点 C
EXEC
# 错误：CROSSSLOT Keys in request don't hash to the same slot
```

**原因：**

| 原因 | 说明 |
|------|------|
| 分布式架构 | 不同 Slot 在不同节点，无法在单个节点上原子执行 |
| 无分布式事务 | Redis 集群不支持分布式事务（2PC、3PC）|
| 网络分区 | 节点间通信可能失败，无法保证一致性 |
| 独立执行 | 每个节点独立执行命令，无全局协调器 |

**✅ 解决方案：**

**方案 1：使用 Hash Tag 强制同 Slot**
```bash
# Hash Tag：只计算 {} 内的部分
MULTI
SET {user:1000}:profile "Alice"   # Slot 5798
SET {user:1000}:order "..."       # Slot 5798（同一个 Slot）
EXEC
# 成功：两个 key 在同一个节点

# Hash Tag 原理
CRC16({user:1000}:profile) mod 16384 = 5798
CRC16({user:1000}:order) mod 16384 = 5798
                ↑
        只计算 {} 内的部分
```

**方案 2：Lua 脚本 + Hash Tag**
```lua
-- 在同一个节点上原子执行
local balance = redis.call('GET', KEYS[1])
if tonumber(balance) >= 100 then
    redis.call('DECRBY', KEYS[1], 100)
    redis.call('SET', KEYS[2], 'order_data')
    return 1
else
    return 0
end
```

```java
// 使用 Hash Tag 确保在同一个 Slot
redis.eval(script, 
    Arrays.asList("{user:1000}:balance", "{user:1000}:order"),
    args);
```

**方案 3：Redisson 锁 + Lua 脚本（推荐）**
```java
// Redisson 锁保证并发控制
RLock lock = redisson.getLock("{user:1000}:lock");
lock.lock();
try {
    // Lua 脚本保证原子性
    String script = "...";
    redis.eval(script, 
        Arrays.asList("{user:1000}:balance", "{user:1000}:order"),
        args);
} finally {
    lock.unlock();
}
```

**方案 4：业务设计避免跨 Slot**
```
设计原则：
✅ 相关数据使用相同的 Hash Tag
✅ 聚合根模式（DDD）
✅ 最终一致性代替强一致性

示例：
用户相关数据：
  {user:1000}:profile
  {user:1000}:balance
  {user:1000}:orders
  {user:1000}:sessions
  
订单相关数据：
  {order:2000}:detail
  {order:2000}:items
  {order:2000}:status
```

**事务支持对比：**

| 场景              | 是否支持事务 | 解决方案                                                  |
| --------------- | ------ | ----------------------------------------------------- |
| **单节点 Redis**   | ✅ 支持   | MULTI/EXEC                                            |
| **集群 - 同 Slot** | ✅ 支持   | Hash Tag + MULTI/EXEC                                 |
| **集群 - 跨 Slot** | 🔴 不支持 | 1. Hash Tag 强制同 Slot<br>2. 业务设计避免跨 Slot<br>3. 使用数据库事务 |
| **跨节点强一致性**     | 🔴 不支持 | 使用 MySQL 等关系型数据库                                      |

**Redisson 锁 vs 事务一致性：**

| 机制 | 保证的内容 | 不保证的内容 | 适用场景 |
|------|-----------|-------------|---------|
| **Redis 事务** | 命令顺序执行 | 回滚、跨节点 | 单节点、简单操作 |
| **Redisson 锁** | 并发控制、串行执行 | 原子性、回滚 | 业务逻辑互斥 |
| **Lua 脚本** | 原子性（单节点）| 跨节点 | 复杂原子操作 |
| **Redisson 锁 + Lua** | 并发控制 + 原子性 | 跨节点 | ✅ 推荐方案 |

**最佳实践：**

```java
// ✅ 推荐：Redisson 锁 + Hash Tag + Lua 脚本
RLock lock = redisson.getLock("{user:1000}:lock");
lock.lock();
try {
    String script = 
        "local balance = redis.call('GET', KEYS[1]) " +
        "if tonumber(balance) >= 100 then " +
        "  redis.call('DECRBY', KEYS[1], 100) " +
        "  redis.call('SET', KEYS[2], ARGV[1]) " +
        "  return 1 " +
        "else " +
        "  return 0 " +
        "end";
    
    Long result = redis.eval(script,
        Arrays.asList("{user:1000}:balance", "{user:1000}:order"),
        Arrays.asList("order_data"));
    
    if (result == 1) {
        // 成功
    } else {
        // 余额不足
    }
} finally {
    lock.unlock();
}
```

**权衡建议：**

```
如果必须跨 Slot 且需要强一致性：
  ❌ Redis 集群不适合
  ✅ 使用 MySQL 等关系型数据库
  ✅ Redis 作为缓存，不作为主存储
  ✅ 使用 Saga 模式实现最终一致性
```

---

##### 一、Redis 核心机制

###### 1. 内存管理与淘汰策略

**maxmemory 配置：**
```bash
# 设置最大内存（例如 2GB）
maxmemory 2gb
```

**8 种淘汰策略：**

| #   | 策略                  | 作用范围        | 淘汰规则       | 适用场景        |
| --- | ------------------- | ----------- | ---------- | ----------- |
| 1   | **noeviction**      | -           | 不淘汰，写入返回错误 | 不允许数据丢失     |
| 2   | **allkeys-lru**     | 所有 key      | 淘汰最近最少使用   | 通用缓存（推荐）    |
| 3   | **allkeys-lfu**     | 所有 key      | 淘汰最不经常使用   | 热点数据缓存      |
| 4   | **allkeys-random**  | 所有 key      | 随机淘汰       | 数据访问均匀      |
| 5   | **volatile-lru**    | 设置过期时间的 key | 淘汰最近最少使用   | 部分数据可淘汰     |
| 6   | **volatile-lfu**    | 设置过期时间的 key | 淘汰最不经常使用   | 热点数据 + 过期控制 |
| 7   | **volatile-random** | 设置过期时间的 key | 随机淘汰       | 过期数据随机清理    |
| 8   | **volatile-ttl**    | 设置过期时间的 key | 淘汰 TTL 最短的 | 优先清理即将过期数据  |

**LRU vs LFU：**
- **LRU（Least Recently Used）**：淘汰最久未访问的数据，适合时间局部性强的场景
- **LFU（Least Frequently Used）**：淘汰访问频率最低的数据，适合热点数据场景（Redis 4.0+）

**内存碎片整理：**
```bash
# 启用自动碎片整理（Redis 4.0+）
activedefrag yes
active-defrag-ignore-bytes 100mb        # 碎片达到 100MB 才整理
active-defrag-threshold-lower 10        # 碎片率 > 10% 触发
active-defrag-cycle-min 5               # CPU 最小使用 5%
active-defrag-cycle-max 75              # CPU 最大使用 75%
```

###### 2. 过期键删除策略

**三种删除机制：**

| 策略 | 触发时机 | 优点 | 缺点 |
|------|---------|------|------|
| **惰性删除** | 访问 key 时检查 | CPU 友好 | 可能占用内存 |
| **定期删除** | 每 100ms 随机抽查 | 平衡内存和 CPU | 可能有漏网之鱼 |
| **主动删除** | 内存不足时触发淘汰策略 | 保证内存可用 | 可能影响性能 |

**定期删除算法：**
```
每 100ms 执行一次：
1. 随机抽取 20 个设置过期时间的 key
2. 删除其中已过期的 key
3. 如果过期 key 比例 > 25%，重复步骤 1
4. 单次执行时间不超过 25ms
```

###### 3. 事件驱动模型

**什么是 IO 多路复用？**

**传统阻塞 IO 模型（BIO - Blocking IO）：**

```
┌─────────────────────────────────────────────────────────────┐
│                    传统阻塞 IO 模型                           │
└─────────────────────────────────────────────────────────────┘

客户端 1                线程 1                    内核
   │                      │                        │
   │─────连接请求─────────>│                        │
   │                      │────创建 socket─────────>│
   │                      │                        │
   │                      │────read() 阻塞等待─────>│
   │                      │        ⏸️              │
   │                      │     (线程阻塞)          │
   │─────发送数据─────────>│                        │
   │                      │<───数据到达─────────────│
   │                      │        ▶️              │
   │                      │     (线程唤醒)          │
   │<────返回数据───────── │                        │


客户端 2                线程 2                    内核
   │                      │                        │
   │─────连接请求─────────>│                        │
   │                      │────创建 socket─────────>│
   │                      │                        │
   │                      │────read() 阻塞等待─────>│
   │                      │        ⏸️              │
   │                      │     (线程阻塞)          │


客户端 3                线程 3                     内核
   │                      │                        │
   │─────连接请求─────────>│                         │
   │                      │────创建 socket─────────>│
   │                      │────read() 阻塞等待─────> │
   │                      │        ⏸️               │

问题：
❌ 1 个连接 = 1 个线程
❌ 10000 个连接 = 10000 个线程
❌ 每个线程占用 1MB 栈空间 = 10GB 内存
❌ 大量线程切换开销
❌ 大部分时间线程都在阻塞等待
```

**IO 多路复用模型（Multiplexing IO）：**

```
┌─────────────────────────────────────────────────────────────┐
│                    IO 多路复用模型                            │
└─────────────────────────────────────────────────────────────┘

客户端 1 ─┐
客户端 2 ─┤
客户端 3 ─┤                单个线程                    内核
客户端 4 ─┼────连接请求────>│                          │
客户端 5 ─┤                │                          │
  ...   ─┤                │                          │
客户端 N ─┘                │                          │
                           │                          │
                           │──epoll_create()─────────>│
                           │<─返回 epoll_fd───────────│
                           │                          │
                           │──epoll_ctl(ADD, fd1)────>│
                           │──epoll_ctl(ADD, fd2)────>│
                           │──epoll_ctl(ADD, fd3)────>│
                           │       ...                │
                           │──epoll_ctl(ADD, fdN)────>│
                           │                          │
                           │──epoll_wait()───────────>│
                           │      (等待事件)           │
                           │                          │
客户端 1 ─────发送数据──────>│                          │
                           │                          │
客户端 5 ─────发送数据──────>│                          │
                           │                          │
                           │<─返回就绪列表────────────│
                           │  [fd1, fd5]              │
                           │                          │
                           │──read(fd1)──────────────>│
                           │<─返回数据────────────────│
                           │                          │
                           │──read(fd5)──────────────>│
                           │<─返回数据────────────────│
                           │                          │
                           │──epoll_wait()───────────>│
                           │      (继续等待)           │

优势：
✅ 1 个线程监听 N 个连接
✅ 10000 个连接 = 1 个线程
✅ 内存占用：1MB（vs 10GB）
✅ 无线程切换开销
✅ 只处理就绪的连接（非阻塞）
✅ epoll_wait() 设置 -1 永久阻塞，0 立即范围，N > 0 超时等待 N 毫秒，超时/有时间都返回。Redis 设置 N，该 N 是动态计算，一次结束会再次调用 epoll_wait()
```

**三种 IO 多路复用实现对比：**

**1. select 模型：**
```
┌─────────────────────────────────────────────────────────────┐
│                      select 模型                             │
└─────────────────────────────────────────────────────────────┘

应用程序                                              内核
    │                                                  │
    │──select(fd_set)──────────────────────────────>│
    │   传递所有 fd 的集合（最多 1024 个）              │
    │                                                  │
    │                                    遍历所有 fd   │
    │                                    检查是否就绪   │
    │                                    O(n) 复杂度   │
    │                                                  │
    │<─返回就绪的 fd 数量───────────────────────────────│
    │                                                  │
    │   应用程序再次遍历所有 fd                         │
    │   找出哪些 fd 就绪（O(n)）                        │
    │                                                  │

缺点：
❌ 最多监听 1024 个连接（FD_SETSIZE 限制）
❌ 每次调用都要传递整个 fd_set（内核态/用户态拷贝）
❌ 内核需要遍历所有 fd（O(n)）
❌ 应用程序需要再次遍历找出就绪的 fd（O(n)）
```

**2. poll 模型：**
```
┌─────────────────────────────────────────────────────────────┐
│                       poll 模型                              │
└─────────────────────────────────────────────────────────────┘

应用程序                                              内核
    │                                                  │
    │──poll(pollfd[], nfds)────────────────────────>   │
    │   传递 pollfd 数组（无数量限制）                     │
    │                                                  │
    │                                    遍历所有 fd    │
    │                                    检查是否就绪    │
    │                                    O(n) 复杂度    │
    │                                                  │
    │<─返回就绪的 fd 数量─────────────────────────────── │
    │                                                  │
    │   应用程序遍历 pollfd 数组                          │
    │   找出哪些 fd 就绪（O(n)）                          │
    │                                                  │

改进：
✅ 无连接数限制（vs select 的 1024）
缺点：
❌ 每次调用都要传递整个数组（内核态/用户态拷贝）
❌ 内核需要遍历所有 fd（O(n)）
❌ 应用程序需要再次遍历找出就绪的 fd（O(n)）
```

**3. epoll 模型（Redis 在 Linux 上使用）：**
```
┌─────────────────────────────────────────────────────────────┐
│                      epoll 模型                              │
└─────────────────────────────────────────────────────────────┘

应用程序                                              内核
    │                                                  │
    │──epoll_create()──────────────────────────────>│
    │                                                  │
    │<─返回 epoll_fd───────────────────────────────────│
    │                                                  │
    │                                    创建红黑树     │
    │                                    创建就绪列表   │
    │                                                  │
    │──epoll_ctl(ADD, fd1)─────────────────────────>│
    │──epoll_ctl(ADD, fd2)─────────────────────────>│
    │──epoll_ctl(ADD, fd3)─────────────────────────>│
    │                                                  │
    │                                    fd1 加入红黑树│
    │                                    fd2 加入红黑树│
    │                                    fd3 加入红黑树│
    │                                                  │
    │──epoll_wait()────────────────────────────────>│
    │                                                  │
    │                                    等待事件...   │
    │                                                  │
    │                          fd1 数据到达            │
    │                          ↓                      │
    │                          回调函数触发            │
    │                          ↓                      │
    │                          fd1 加入就绪列表        │
    │                                                  │
    │<─返回就绪列表 [fd1]──────────────────────────────│
    │   (只返回就绪的 fd，无需遍历)                     │
    │                                                  │
    │──read(fd1)───────────────────────────────────>│
    │<─返回数据────────────────────────────────────────│
    │                                                  │

优势：
✅ 无连接数限制
✅ 只返回就绪的 fd（O(1)）
✅ 无需每次传递所有 fd（内核维护红黑树）
✅ 事件驱动（回调机制）
✅ 支持边缘触发（ET）和水平触发（LT）
```

**epoll 的内部数据结构：**

```
┌─────────────────────────────────────────────────────────────┐
│                   epoll 内部结构                             │
└─────────────────────────────────────────────────────────────┘

                    epoll 实例
                        │
        ┌───────────────┴───────────────┐
        │                               │
   红黑树（存储所有 fd）          就绪列表（双向链表）
        │                               │
    ┌───┴───┐                      ┌────┴────┐
    │  fd1  │                      │   fd3   │ ← 有数据可读
    ├───────┤                      ├─────────┤
    │  fd2  │                      │   fd7   │ ← 有数据可读
    ├───────┤                      └─────────┘
    │  fd3  │ ─────事件到达────────>  加入就绪列表
    ├───────┤
    │  fd4  │
    ├───────┤
    │  fd5  │
    ├───────┤
    │  fd6  │
    ├───────┤
    │  fd7  │ ─────事件到达────────>  加入就绪列表
    └───────┘

查找 fd：O(log n)（红黑树）
返回就绪 fd：O(1)（直接返回就绪列表）
```

**Redis 使用 epoll 的完整流程：**

```
┌─────────────────────────────────────────────────────────────┐
│              Redis 使用 epoll 处理客户端请求                  │
└─────────────────────────────────────────────────────────────┘

1. 初始化阶段：
   Redis 启动
      ↓
   创建 epoll 实例（epoll_create）
      ↓
   创建监听 socket（bind + listen）
      ↓
   将监听 socket 加入 epoll（epoll_ctl ADD）
      ↓
   进入事件循环

2. 接受连接：
   客户端连接到达
      ↓
   epoll_wait 返回（监听 socket 可读）
      ↓
   调用 accept() 创建客户端 socket
      ↓
   将客户端 socket 加入 epoll（epoll_ctl ADD）
      ↓
   注册读事件（EPOLLIN）

3. 读取命令：
   客户端发送命令
      ↓
   epoll_wait 返回（客户端 socket 可读）
      ↓
   调用 read() 读取数据
      ↓
   解析 RESP 协议
      ↓
   执行命令（单线程）
      ↓
   将响应写入输出缓冲区

4. 发送响应：
   输出缓冲区有数据
      ↓
   注册写事件（EPOLLOUT）
      ↓
   epoll_wait 返回（客户端 socket 可写）
      ↓
   调用 write() 发送数据
      ↓
   取消写事件（避免频繁触发）
      ↓
   继续等待下一个命令

5. 关闭连接：
   客户端断开连接
      ↓
   epoll_wait 返回（socket 关闭事件）
      ↓
   从 epoll 移除（epoll_ctl DEL）
      ↓
   关闭 socket
```

**性能对比总结：**

| 模型         | 连接数限制 | 查找就绪 fd | 内核拷贝     | 适用场景        |
| ---------- | ----- | ------- | -------- | ----------- |
| **select** | 1024  | O(n)    | 每次都拷贝    | 少量连接        |
| **poll**   | 无限制   | O(n)    | 每次都拷贝    | 中等连接        |
| **epoll**  | 无限制   | O(1)    | **无需拷贝** | 大量连接（Redis） |

传统的阻塞 IO 模型中，每个连接需要一个线程处理：
```
客户端 1 → 线程 1 → 阻塞等待数据
客户端 2 → 线程 2 → 阻塞等待数据
客户端 3 → 线程 3 → 阻塞等待数据
...
客户端 10000 → 线程 10000 → 阻塞等待数据

问题：
- 线程数量 = 连接数量
- 大量线程切换开销
- 内存消耗巨大（每个线程 1MB 栈空间）
```

**IO 多路复用：一个线程监听多个连接**
```
                    ┌─ 客户端 1（可读）
                    ├─ 客户端 2（无事件）
单个线程 → IO 多路复用 ┼─ 客户端 3（可写）
                    ├─ 客户端 4（无事件）
                    └─ 客户端 5（可读）

优势：
- 一个线程处理所有连接
- 无线程切换开销
- 内存占用小
```

**三种 IO 多路复用实现：**

| 实现 | 平台 | 最大连接数 | 时间复杂度 | 特点 |
|------|------|-----------|-----------|------|
| **select** | 跨平台 | 1024（FD_SETSIZE）| O(n) | 最古老，性能最差 |
| **poll** | Linux/Unix | 无限制 | O(n) | 改进 select，无连接数限制 |
| **epoll** | Linux | 无限制 | O(1) | 最高效，Redis 在 Linux 上首选 |
| **kqueue** | macOS/BSD | 无限制 | O(1) | macOS 上的高效实现 |

**epoll 工作原理（Redis 在 Linux 上使用）：**

```c
// 1. 创建 epoll 实例
int epfd = epoll_create(1024);

// 2. 注册监听的文件描述符（socket）
struct epoll_event ev;
ev.events = EPOLLIN;  // 监听可读事件
ev.data.fd = client_fd;
epoll_ctl(epfd, EPOLL_CTL_ADD, client_fd, &ev);

// 3. 等待事件发生（非阻塞）
struct epoll_event events[MAX_EVENTS];
int nfds = epoll_wait(epfd, events, MAX_EVENTS, timeout);

// 4. 处理就绪的事件
for (int i = 0; i < nfds; i++) {
    if (events[i].events & EPOLLIN) {
        // 有数据可读，处理读事件
        handle_read(events[i].data.fd);
    }
}
```

**epoll 的三种触发模式：**

| 模式 | 触发时机 | 特点 | Redis 使用 |
|------|---------|------|-----------|
| **水平触发（LT）** | 只要有数据就一直通知 | 简单，不易丢事件 | ✅ 默认使用 |
| **边缘触发（ET）** | 数据到达时只通知一次 | 高效，但需小心处理 | 可选 |

**什么是 Reactor 模式？**

Reactor 是一种事件驱动的设计模式，用于处理并发 IO：

```
┌─────────────────────────────────────────────────────────┐
│                    Reactor 模式                          │
└─────────────────────────────────────────────────────────┘

1. 注册阶段：
   应用程序向 Reactor 注册感兴趣的事件（读/写/连接）

2. 事件循环：
   Reactor 通过 IO 多路复用等待事件发生

3. 事件分派：
   事件发生时，Reactor 调用对应的事件处理器

4. 事件处理：
   事件处理器执行具体的业务逻辑
```

**Reactor 模式的核心组件：**

| 组件 | 职责 | Redis 中的实现 |
|------|------|---------------|
| **Reactor（反应器）** | 事件循环，等待事件 | aeEventLoop |
| **Handle（句柄）** | 标识事件源 | 文件描述符（socket fd）|
| **Event Handler（事件处理器）** | 处理事件的回调函数 | acceptTcpHandler、readQueryFromClient |
| **Synchronous Event Demultiplexer** | IO 多路复用 | epoll/kqueue/select |

**Redis 的 Reactor 实现（aeEventLoop）：**

```c
// Redis 事件循环结构
typedef struct aeEventLoop {
    int maxfd;                      // 最大文件描述符
    aeFileEvent *events;            // 注册的文件事件数组
    aeFiredEvent *fired;            // 就绪的事件数组
    aeTimeEvent *timeEventHead;     // 时间事件链表
    void *apidata;                  // IO 多路复用的私有数据（epoll_fd）
} aeEventLoop;

// 文件事件结构
typedef struct aeFileEvent {
    int mask;                       // 事件类型（AE_READABLE/AE_WRITABLE）
    aeFileProc *rfileProc;          // 读事件处理函数
    aeFileProc *wfileProc;          // 写事件处理函数
    void *clientData;               // 客户端数据
} aeFileEvent;
```

**Redis 事件驱动流程：**

```
┌─────────────────────────────────────────────────────────┐
│                  Redis 事件循环                          │
└─────────────────────────────────────────────────────────┘

while (server.running) {
    // 1. 计算最近的时间事件触发时间
    shortest = getShortestTimeEvent();
    
    // 2. 调用 IO 多路复用等待事件（最多等待 shortest 时间）
    numevents = aeApiPoll(eventLoop, shortest);
    
    // 3. 处理文件事件（网络 IO）
    for (i = 0; i < numevents; i++) {
        aeFileEvent *fe = &eventLoop->events[fired[i].fd];
        
        if (fe->mask & AE_READABLE) {
            fe->rfileProc(eventLoop, fd, fe->clientData, mask);
        }
        
        if (fe->mask & AE_WRITABLE) {
            fe->wfileProc(eventLoop, fd, fe->clientData, mask);
        }
    }
    
    // 4. 处理时间事件（定时任务）
    processTimeEvents(eventLoop);
}
```

**Redis 中的事件类型：**

**1. 文件事件（File Event）：**

| 事件              | 触发条件      | 处理器                 | 作用                  |
| --------------- | --------- | ------------------- | ------------------- |
| **AE_READABLE** | socket 可读 | acceptTcpHandler    | 接受新连接（监听 socket）    |
| **AE_READABLE** | socket 可读 | readQueryFromClient | 读取客户端命令（客户端 socket） |
| **AE_WRITABLE** | socket 可写 | sendReplyToClient   | 发送响应给客户端            |

**说明：**
- **AE_READABLE** 事件根据 socket 类型调用不同的处理器：
  - 监听 socket 可读 → 有新连接 → acceptTcpHandler
  - 客户端 socket 可读 → 有命令到达 → readQueryFromClient

**2. 时间事件（Time Event）：**

| 时间事件 | 触发频率 | 作用 |
|---------|---------|------|
| **serverCron** | 每 100ms | 清理过期 key、rehash、持久化检查 |

**完整的请求处理流程：**

```
1. 客户端连接到达
   ↓
2. epoll_wait 返回，监听 socket 可读
   ↓
3. Reactor 调用 acceptTcpHandler
   ↓
4. 创建客户端 socket，注册 AE_READABLE 事件
   ↓
5. 客户端发送命令
   ↓
6. epoll_wait 返回，客户端 socket 可读
   ↓
7. Reactor 调用 readQueryFromClient
   ↓
8. 读取命令，解析协议
   ↓
9. 执行命令（单线程）
   ↓
10. 将响应写入输出缓冲区
   ↓
11. 注册 AE_WRITABLE 事件
   ↓
12. epoll_wait 返回，客户端 socket 可写
   ↓
13. Reactor 调用 sendReplyToClient
   ↓
14. 发送响应给客户端
   ↓
15. 取消 AE_WRITABLE 事件（避免频繁触发）
```

**为什么 Redis 使用单线程 + Reactor 模式？**

| 优势 | 说明 |
|------|------|
| **无锁设计** | 单线程执行命令，无需加锁 |
| **简单** | 代码逻辑简单，易于维护 |
| **高性能** | 纯内存操作 + IO 多路复用，性能足够 |
| **原子性** | 命令执行天然原子性 |
| **可预测** | 无线程调度的不确定性 |

**单线程的性能瓶颈：**

| 瓶颈 | 说明 | 解决方案 |
|------|------|---------|
| **CPU 密集型命令** | KEYS、SORT 等阻塞主线程 | 禁用或用 SCAN 替代 |
| **网络 IO** | 大量连接时，读写数据占用 CPU | Redis 6.0+ 多线程 IO |
| **持久化** | BGSAVE/BGREWRITEAOF | fork 子进程异步处理 |
| **大 key 删除** | DEL 阻塞主线程 | 使用 UNLINK 异步删除 |

**单线程模型（Redis 6.0 前）：**
```
主线程：
  ├─ 网络 IO（读写）
  ├─ 命令解析
  ├─ 命令执行
  └─ 响应发送

后台线程（BIO）：
  ├─ AOF 重写
  ├─ RDB 持久化
  └─ 异步删除
```

**多线程 IO（Redis 6.0+）：**
```bash
# 启用多线程 IO
io-threads 4                    # IO 线程数（建议 CPU 核心数）
io-threads-do-reads yes         # 读操作也用多线程
```

**多线程 IO 架构：**
```
主线程：
  ├─ 命令解析
  ├─ 命令执行（仍然单线程）
  └─ 事件分派

IO 线程池（4 个线程）：
  ├─ IO 线程 1：读取客户端 1-250 的数据
  ├─ IO 线程 2：读取客户端 251-500 的数据
  ├─ IO 线程 3：读取客户端 501-750 的数据
  └─ IO 线程 4：读取客户端 751-1000 的数据

后台线程（BIO）：
  ├─ AOF 重写
  ├─ RDB 持久化
  └─ 异步删除
```

**多线程 IO 的工作流程：**
```
1. 主线程通过 epoll_wait 获取就绪的 socket
2. 主线程将读任务分配给 IO 线程池
3. IO 线程并发读取数据到各自的缓冲区
4. 主线程等待所有 IO 线程完成
5. 主线程串行解析和执行命令（保持单线程）
6. 主线程将写任务分配给 IO 线程池
7. IO 线程并发发送响应
```

**性能对比：**

| 场景 | 单线程 | 多线程 IO | 提升 |
|------|-------|----------|------|
| 少量连接（< 100）| 10 万 QPS | 10 万 QPS | 无提升 |
| 大量连接（> 1000）| 10 万 QPS | 15-20 万 QPS | 50-100% |
| 大 value（> 1KB）| 5 万 QPS | 10 万 QPS | 100% |

**总结：**

| 概念 | 核心思想 | Redis 实现 |
|------|---------|-----------|
| **IO 多路复用** | 一个线程监听多个连接 | epoll/kqueue/select |
| **Reactor 模式** | 事件驱动，非阻塞 IO | aeEventLoop |
| **单线程模型** | 命令执行单线程 | 简单、无锁、原子性 |
| **多线程 IO** | 网络 IO 多线程 | 提升高并发性能 |

---

###### 4. 持久化机制详解

**RDB（快照）：**

| 特性 | 说明 |
|------|------|
| **触发方式** | SAVE（阻塞）/ BGSAVE（fork 子进程）/ 自动（save 配置）|
| **文件格式** | 二进制压缩文件（dump.rdb）|
| **COW 机制** | fork 子进程，父进程修改数据时 Copy-On-Write |
| **优点** | 文件小、恢复快、对性能影响小 |
| **缺点** | 可能丢失最后一次快照后的数据 |

**配置示例：**
```bash
# 自动触发条件（满足任一即触发）
save 900 1          # 900 秒内至少 1 次写操作
save 300 10         # 300 秒内至少 10 次写操作
save 60 10000       # 60 秒内至少 10000 次写操作

# RDB 文件
dbfilename dump.rdb
dir /var/lib/redis

# 压缩
rdbcompression yes
rdbchecksum yes
```

**AOF（追加日志）：**

| 特性 | 说明 |
|------|------|
| **记录内容** | 每个写命令（RESP 协议格式）|
| **文件格式** | 文本文件（appendonly.aof）|
| **同步策略** | always / everysec / no |
| **优点** | 数据安全性高、可读性强 |
| **缺点** | 文件大、恢复慢 |

**三种同步策略：**

| 策略 | 同步时机 | 性能 | 安全性 | 适用场景 |
|------|---------|------|--------|---------|
| **always** | 每个命令都 fsync | 慢 | 最高（最多丢 1 条）| 金融场景 |
| **everysec** | 每秒 fsync 一次 | 快 | 高（最多丢 1 秒）| 推荐（默认）|
| **no** | 由操作系统决定 | 最快 | 低（可能丢几十秒）| 可接受数据丢失 |

**AOF 重写机制：**

```bash
# 自动触发条件（两个条件都要满足，AND 关系）
auto-aof-rewrite-percentage 100    # AOF 文件增长 100% 触发
auto-aof-rewrite-min-size 64mb     # AOF 文件至少 64MB

# 触发逻辑
if (当前 AOF 大小 >= 64MB) AND (增长率 >= 100%) {
    触发 AOF 重写
}

# 示例
上次重写后：64MB，当前：128MB
├─ 增长率：(128-64)/64 = 100% ✅
├─ 大小：128MB >= 64MB ✅
└─ 结果：触发重写

上次重写后：10MB，当前：20MB
├─ 增长率：(20-10)/10 = 100% ✅
├─ 大小：20MB >= 64MB ❌
└─ 结果：不触发（文件太小）

# 手动触发
BGREWRITEAOF
```

**AOF 的完整生命周期：**

```
┌─────────────────────────────────────────────────────────────┐
│              AOF 追加与重写的循环过程                          │
└─────────────────────────────────────────────────────────────┘

AOF 重写 1                                      AOF 重写 2
(重写完成)                                      (再次重写)
    ↓                                               ↓
AOF: 10MB                                       AOF: 20MB
    │                                               │
    │←──────── 正常追加模式 ────────────────────────→ │
    │                                               │
    │  SET key1 "v1"    → 追加到 AOF                 │
    │  INCR counter     → 追加到 AOF                 │
    │  LPUSH list "a"   → 追加到 AOF                 │
    │  ...（每个写命令都追加）                         │
    │                                               │
    │  文件持续增长：10MB → 15MB → 20MB               │
    │                                               │
    │  达到触发条件：                                 │
    │  ├─ 文件 >= 64MB                              │
    │  └─ 增长 >= 100%                              │
    │                                               │
    └──────────────────────────────────────────────→│
                                                    │
                                            fork 子进程
                                            生成新 AOF
                                            rename 替换
                                                    │
                                            AOF: 12MB
                                                    │
                                            继续追加...
```

**关键点：**
- ✅ 正常模式：每个写命令追加到 AOF 文件
- ✅ 文件增长：持续追加导致文件变大
- ✅ 触发重写：达到条件后 fork 子进程
- ✅ 原子替换：rename 系统调用替换旧文件
- ✅ 循环往复：重写后继续追加

**重写原理：**
```
1. fork 子进程
2. 子进程遍历内存数据，生成新 AOF 文件
3. 父进程继续处理命令，写入 AOF 重写缓冲区
4. 子进程完成后，父进程追加缓冲区数据到新文件
5. 原子替换旧 AOF 文件（rename 系统调用）
```

**重要说明：AOF vs RDB 的本质区别**

| 特性 | AOF | RDB |
|------|-----|-----|
| **记录内容** | 所有写操作命令（操作日志）| 数据的最终状态（快照）|
| **文件格式** | 文本（RESP 协议）| 二进制 |
| **文件结构（< Redis 7.0）** | 单个大文件（`appendonly.aof`）| 单个大文件（`dump.rdb`）|
| **文件结构（≥ Redis 7.0）** | 多个文件（Base + Incremental + Manifest）| 单个大文件（`dump.rdb`）|
| **正常模式** | 追加每个写命令 | 定期生成快照 |
| **重写后** | 压缩为最终状态的命令 | - |
| **恢复方式** | 重放命令 | 直接加载数据 |

**Redis 7.0+ Multi-Part AOF 文件结构：**

```bash
/var/lib/redis/appendonlydir/
├── appendonly.aof.1.base.rdb      # Base AOF（RDB 格式，重写后的完整数据）
├── appendonly.aof.1.incr.aof      # Incremental AOF（重写后的新写入）
├── appendonly.aof.2.incr.aof      # Incremental AOF（继续追加）
└── appendonly.aof.manifest        # Manifest 文件（记录 AOF 文件列表）
```

**AOF 重写的关键理解：**

```
重写前（记录所有操作）：
SET counter 1
INCR counter    # counter = 2
INCR counter    # counter = 3
INCR counter    # counter = 4
INCR counter    # counter = 5
文件大小：5 条命令

重写后（只保留最终状态）：
SET counter 5
文件大小：1 条命令

✅ 恢复结果相同：counter = 5
❌ 操作历史丢失（这是设计目标，不是 bug）
```

**为什么重写会"丢失"操作历史？**

| 原因 | 说明 |
|------|------|
| **设计目的** | AOF 用于数据恢复，不是审计日志 |
| **性能考虑** | 保留所有历史会导致文件无限增长 |
| **恢复需求** | 只需要最终状态，不需要操作过程 |

**如果需要操作历史：**
```
✅ 应用层日志：在代码中记录操作
✅ Redis Streams：XADD 记录操作历史
✅ 数据库审计表：MySQL 等存储审计日志
❌ 不要依赖 AOF：AOF 不是审计系统
```

**混合持久化 Redis 6.0 vs 7.0 持久化配置建议：**

| Redis 版本       | AOF 配置                                         | 文件结构                                   | 推荐场景                |
| -------------- | ---------------------------------------------- | -------------------------------------- | ------------------- |
| **Redis 6.0**  | `appendonly yes`<br>`aof-use-rdb-preamble yes` | 单个 AOF 文件<br>（`appendonly.aof`）        | 生产环境标准配置<br>混合持久化   |
| **Redis 7.0+** | `appendonly yes`<br>`aof-use-rdb-preamble yes` | Multi-Part AOF<br>（Base + Incremental） | 生产环境推荐<br>更好的性能和可靠性 |

**Redis 6.0 配置示例：**
```bash
# redis.conf
appendonly yes
appendfilename "appendonly.aof"
aof-use-rdb-preamble yes
appendfsync everysec

# RDB 配置（作为备份）
save 900 1
save 300 10
save 60 10000
```

**Redis 7.0+ 配置示例：**
```bash
# redis.conf
appendonly yes
appenddirname "appendonlydir"
aof-use-rdb-preamble yes
appendfsync everysec

# Multi-Part AOF 自动启用
# 文件会自动生成在 appendonlydir/ 目录下

# RDB 配置（作为备份）
save 900 1
save 300 10
save 60 10000
```

**升级建议：**
- **从 Redis 6.0 升级到 7.0**：
  - 无需修改配置，Redis 7.0 会自动将旧的单文件 AOF 转换为 Multi-Part AOF
  - 首次启动时会在 `appendonlydir/` 目录下生成新的文件结构
  - 旧的 `appendonly.aof` 文件会被保留（可手动删除）

- **生产环境推荐**：
  - Redis 7.0+ 优先，Multi-Part AOF 性能更好
  - 如果使用 Redis 6.0，确保开启混合持久化（`aof-use-rdb-preamble yes`）
  - 定期备份 RDB 文件到远程存储

Redis 支持多种持久化方式：

- RDB（快照）：定时生成数据库数据快照，存为二进制文件，适合备份和灾难恢复。
- AOF（追加日志）：记录每个写操作日志，重启时通过重放日志恢复数据，持久性更强但恢复速度较慢。

###### 5. 主从复制机制

**架构：**
```
Master（主节点）
    ↓ 复制
Replica 1（从节点）
Replica 2（从节点）
Replica 3（从节点）
```

**复制流程：**

| 阶段 | 步骤 | 说明 |
|------|------|------|
| **1. 建立连接** | Replica 发送 PSYNC | 携带 replication ID 和 offset |
| **2. 全量同步** | Master 执行 BGSAVE | 生成 RDB 文件 |
| | Master 发送 RDB | 同时缓冲新写命令到复制缓冲区 |
| | Replica 加载 RDB | 清空旧数据，加载新数据 |
| **3. 增量同步** | Master 发送缓冲区命令 | 追赶主节点状态 |
| **4. 命令传播** | 持续同步写命令 | 保持主从一致 |

**PSYNC2 优化（Redis 4.0+）：**

| 场景 | Redis 2.8 | Redis 4.0+ PSYNC2 |
|------|-----------|-------------------|
| Replica 重启 | 全量同步 | 增量同步（如果 offset 在 backlog 内）|
| 主从切换 | 全量同步 | 增量同步（新 Master 继承 replication ID）|
| 网络闪断 | 可能全量 | 增量同步（backlog 足够大）|

**复制积压缓冲区（Replication Backlog）：**
```bash
# 配置缓冲区大小（默认 1MB）
repl-backlog-size 10mb

# 缓冲区保留时间（从节点断开后）
repl-backlog-ttl 3600
```

**无盘复制（Diskless Replication）：**
```bash
# 启用无盘复制（直接通过网络发送 RDB，不写磁盘）
repl-diskless-sync yes
repl-diskless-sync-delay 5    # 等待 5 秒，让更多 Replica 一起同步
```

**适用场景：**
- 磁盘 IO 慢（如网络存储）
- 多个 Replica 同时同步

**主从配置：**
```bash
# Replica 配置
replicaof <master-ip> <master-port>
masterauth <master-password>

# 只读模式（默认）
replica-read-only yes

# 优先级（数字越小优先级越高，0 表示永不提升为 Master）
replica-priority 100
```

###### 6. 哨兵机制（Sentinel）

**架构：**
```
Sentinel 1 ─┐
Sentinel 2 ─┼─ 监控 ─> Master
Sentinel 3 ─┘              ↓ 复制
                      Replica 1
                      Replica 2
```

**核心功能：**
1. **监控**：检测 Master 和 Replica 是否正常
2. **通知**：通过 API 通知管理员或应用
3. **自动故障转移**：Master 故障时自动提升 Replica
4. **配置提供**：客户端通过 Sentinel 获取当前 Master 地址

**故障检测：**

| 类型 | 判断条件 | 影响 |
|------|---------|------|
| **主观下线（SDOWN）** | 单个 Sentinel 认为节点下线 | 仅该 Sentinel 认为下线 |
| **客观下线（ODOWN）** | 多数 Sentinel 认为节点下线 | 触发故障转移 |

**判断流程：**
```
1. Sentinel 每秒向 Master 发送 PING
2. 超过 down-after-milliseconds 无响应 → 主观下线
3. Sentinel 询问其他 Sentinel 是否同意下线
4. 超过 quorum 数量同意 → 客观下线
5. 触发故障转移
```

**故障转移流程：**

| 步骤 | 说明 |
|------|------|
| **1. 选举 Leader Sentinel** | 使用 Raft 算法，多数派投票 |
| **2. 选择新 Master** | 根据优先级、复制偏移量、运行 ID 选择 |
| **3. 提升 Replica** | 向选中的 Replica 发送 SLAVEOF NO ONE |
| **4. 更新其他 Replica** | 让其他 Replica 复制新 Master |
| **5. 更新配置** | 通知客户端新 Master 地址 |
| **6. 降级旧 Master** | 旧 Master 恢复后变为 Replica |

**选择新 Master 的规则（优先级从高到低）：**
```
1. replica-priority 最小（0 除外）
2. 复制偏移量最大（数据最新）
3. 运行 ID 最小（字典序）
```

**Sentinel 配置：**
```bash
# sentinel.conf
port 26379

# 监控 Master（quorum=2 表示 2 个 Sentinel 同意才客观下线）
sentinel monitor mymaster 127.0.0.1 6379 2

# 认证
sentinel auth-pass mymaster <password>

# 判断下线时间（30 秒无响应）
sentinel down-after-milliseconds mymaster 30000

# 故障转移超时
sentinel failover-timeout mymaster 180000

# 并行同步的 Replica 数量
sentinel parallel-syncs mymaster 1
```

**启动 Sentinel：**
```bash
redis-sentinel /path/to/sentinel.conf
# 或
redis-server /path/to/sentinel.conf --sentinel
```

**客户端连接：**
```python
from redis.sentinel import Sentinel

sentinel = Sentinel([
    ('sentinel1', 26379),
    ('sentinel2', 26379),
    ('sentinel3', 26379)
], socket_timeout=0.1)

# 获取 Master
master = sentinel.master_for('mymaster', socket_timeout=0.1)
master.set('key', 'value')

# 获取 Replica（读操作）
replica = sentinel.slave_for('mymaster', socket_timeout=0.1)
value = replica.get('key')
```

---

##### 二、Redis 集群深入

###### Redis 锁机制

**Redisson 分布式锁机制**

1. **可重入锁（Reentrant Lock），适合同一业务流程中，上下游逻辑处理，例如：调用库存锁和支付锁，支付锁由调用库存的检查**
   
   - **机制**：**允许同一个线程（客户端）多次获得同一把锁**，每次获得锁都增加重入次数，释放锁时减少重入次数，直至归零才真正释放。
   
   - **实现**：在 Redis 中通过键值存储锁信息，键对应锁标识，值中存储锁持有者线程标识和重入计数，操作用 Lua 脚本保证原子性。
   
   - **看门狗**：支持自动续期，看门狗机制会周期性延长锁的过期时间，避免业务执行时间过长锁自动释放，保证锁有效性。

2. **公平锁（Fair Lock）**
   
   - **机制**：保证加锁请求按照请求顺序公平地获得锁，避免线程饥饿。
   
   - **实现**：内部维护一个等待队列（Redis 列表），客户端请求加锁时进入队列，只有排在队首的客户端能尝试获得锁。
   
   - **看门狗**：公平锁同样支持看门狗自动续期，确保锁的持有期间可续期。

3. **联锁（MultiLock）**
   
   - **机制**：实现多个锁的组合加锁，即只有当所有指定锁都成功获得时，联锁才算获得成功。
   
   - **实现**：内部依次尝试对多个独立锁加锁，失败则释放已经加的锁，避免部分获得导致死锁。
   
   - **看门狗**：因底层依赖可重入锁，这些锁均支持看门狗续期，因此联锁隐含支持看门狗。

4. **红锁（RedLock），适合电商秒杀，保证一条消息只有一个消费者**
   
   - **机制**：基于 Redis 官方推荐的 Redlock 算法（底层使用红黑树）实现，解决单节点 Redis 宕机导致锁失效问题，适合高可靠分布式环境。
   
   - **实现**：客户端向多个独立 Redis 节点并发加锁，只有在大多数节点成功获得锁时才认为加锁成功。
   
   - **看门狗**：每个节点的锁都支持看门狗自动续期，保证锁的可靠性和有效期。

5. **读写锁（ReadWriteLock）**
   
   - **机制**：实现多线程读-写互斥锁，允许多个读锁并发，同时写锁独占。
   
   - **实现**：在 Redis 中维护两个键分别存储读锁和写锁状态，读锁计数累积，写锁检查是否安全加锁，所有操作原子执行。
   
   - **看门狗**：读锁和写锁均支持看门狗自动续期，保证锁长时间持有时不失效。

6. **信号量（Semaphore），适合限制并发数，访问许可**
   
   - **机制**：实现信号量控制，允许一定数量的客户端获得许可访问资源，超过数量则阻塞或失败。
   
   - **实现**：利用 Redis 的集合结构维护许可数量，客户端尝试获取许可时从集合中添加唯一标识，释放时移除，操作原子。
   
   - **看门狗**：信号量的单个许可也支持自动续期（类似看门狗），防止持有许可客户端异常退出导致许可一直占用。

###### 1. 哈希槽机制

**哈希槽分配**

- Redis 集群将整个键空间分成 16384 个哈希槽（hash slots）。每个 key 通过 CRC16 校验算法计算哈希值，然后对 16384 取模得到对应的槽号。集群中的每个节点负责一定范围的槽位，节点之间通过分配不同数量的槽位来均衡存储负载。槽位分配可以根据节点硬件资源（如硬盘大小）灵活调整，不完全平均，方便管理数据分布。

**Redis 分片原理**

每个键根据其槽号定位到具体节点。客户端访问集群时，根据槽位映射表直达对应节点。这样避免了传统一致性哈希中节点变动导致大量缓存失效的问题。**哈希槽的划分类似硬盘分区**，节点之间转移的是槽位而非直接数据，便于管理和重分配。

**槽位计算：**
```
HASH_SLOT = CRC16(key) mod 16384
```

**Hash Tag（强制同槽）：**

Hash Tag 通过 `{}` 语法强制相关 key 分配到同一个槽位，解决 Redis Cluster 跨槽事务限制问题。

**槽位计算规则：**
```bash
# 普通 key：对整个 key 计算哈希
HASH_SLOT = CRC16("user:1000:profile") mod 16384

# Hash Tag key：只对 {} 内的内容计算哈希
HASH_SLOT = CRC16("user:1000") mod 16384

# 示例
SET {user:1000}:profile "..."     # 槽位由 "user:1000" 决定
SET {user:1000}:orders "..."      # 槽位由 "user:1000" 决定
SET {user:1000}:cart "..."        # 槽位由 "user:1000" 决定
# 以上三个 key 保证在同一个槽位，支持事务和批量操作
```

**跨槽事务限制：**
```bash
# ❌ 错误：跨槽事务会失败
MULTI
SET user:1000:profile "..."   # 可能在槽 5460
SET user:1000:orders "..."    # 可能在槽 8923
EXEC
# (error) CROSSSLOT Keys in request don't hash to the same slot

# ✅ 正确：使用 Hash Tag
MULTI
SET {user:1000}:profile "..."  # 都在同一个槽
SET {user:1000}:orders "..."   # 都在同一个槽
EXEC
# OK
```

**关键注意事项：**

1. **必须从初始设计使用**：Hash Tag 必须在首次 SET 操作时就使用，不能后期添加
   ```bash
   # 已存在的数据
   SET user:1000:profile "..."  # 槽 5460
   
   # ❌ 无法通过改名添加 Hash Tag
   RENAME user:1000:profile {user:1000}:profile
   # 这会创建新 key，可能分配到不同槽位
   
   # ✅ 正确做法：数据迁移
   # 1. 读取旧数据
   # 2. 写入新 key（带 Hash Tag）
   # 3. 删除旧 key
   ```

2. **集群重平衡时的行为**：
   - 重平衡不会自动触发，需要手动执行 `redis-cli --cluster rebalance`
   - 同一 Hash Tag 的所有 key 作为一个单元一起迁移
   - 迁移过程中保持原子性，不会拆散相关 key

3. **智能客户端自动处理**：
   ```python
   # redis-py-cluster 示例
   from rediscluster import RedisCluster
   
   startup_nodes = [{"host": "127.0.0.1", "port": "7000"}]
   rc = RedisCluster(startup_nodes=startup_nodes, decode_responses=True)
   
   # 客户端自动维护槽位映射
   rc.set("{user:1000}:profile", "data")  # 自动路由到正确节点
   rc.set("{user:1000}:orders", "data")   # 自动路由到相同节点
   
   # 集群扩容后，客户端自动更新映射，业务代码无需修改
   ```

4. **常见客户端库支持**：
   - Python: `redis-py-cluster`
   - Java: `Jedis`, `Lettuce`
   - Node.js: `ioredis`
   - Go: `go-redis/redis`
   
   这些客户端会自动：
   - 维护槽位到节点的映射表
   - 处理 MOVED/ASK 重定向
   - 在节点变更时更新路由信息
   - **无需业务代码感知集群拓扑变化**

**Gossip 协议：**

| 消息类型     | 作用      | 频率              |
| -------- | ------- | --------------- |
| **PING** | 心跳检测    | 每秒随机 PING 5 个节点 |
| **PONG** | 响应 PING | 立即响应            |
| **MEET** | 加入集群    | 新节点加入时          |
| **FAIL** | 标记节点下线  | 检测到故障时广播        |

**故障检测：**

| 阶段 | 条件 | 状态 |
|------|------|------|
| **疑似下线（PFAIL）** | 单个节点超时未响应 | 主观判断 |
| **确认下线（FAIL）** | 超过半数节点标记 PFAIL | 客观判断，广播 FAIL 消息 |

**自动故障转移：**
```
1. 检测到 Master 下线（FAIL）
2. 该 Master 的 Replica 发起选举
3. 其他 Master 投票（每个 Master 一票）
4. 获得多数票的 Replica 提升为 Master
5. 新 Master 接管槽位
6. 广播 PONG 消息更新集群配置
```

**MOVED vs ASK 重定向：**

| 重定向类型 | 场景 | 客户端行为 | 槽位状态 |
|-----------|------|-----------|---------|
| **MOVED** | 槽位已迁移完成 | 更新本地槽位映射，后续直接访问新节点 | 稳定 |
| **ASK** | 槽位正在迁移 | 仅本次请求访问新节点，不更新映射 | 迁移中 |

**示例：**
```bash
# MOVED 重定向
127.0.0.1:7000> GET key1
(error) MOVED 3999 127.0.0.1:7001

# ASK 重定向
127.0.0.1:7000> GET key2
(error) ASK 3999 127.0.0.1:7001
# 客户端需要先发送 ASKING 命令
127.0.0.1:7001> ASKING
127.0.0.1:7001> GET key2
```

**集群脑裂问题：**

**场景：**
```
网络分区
    ↓
Master A（少数派）继续接受写入
Master B（多数派）被选举为新 Master
    ↓
网络恢复
    ↓
Master A 的数据丢失（降级为 Replica）
```

**解决方案：**
```bash
# 要求至少 N 个 Replica 在线才接受写入
min-replicas-to-write 1

# 要求 Replica 延迟不超过 N 秒
min-replicas-max-lag 10
```

**槽位迁移原子性保证：**
```
1. 源节点标记槽位为 MIGRATING
2. 目标节点标记槽位为 IMPORTING
3. 逐个迁移 key（MIGRATE 命令）
4. 客户端访问时：
   - 源节点：key 存在返回，不存在返回 ASK
   - 目标节点：收到 ASKING 后可访问迁移中的 key
5. 迁移完成后更新槽位映射
```

Redis 槽位重平衡过程中，底层会进行以下操作：

- 源节点把槽状态标记为“**MIGRATING**”，将对应数据迁移给目标节点。
- 目标节点标记该槽为“导入”状态。
- 客户端根据 MOVED 和 ASK 重定向命令，将请求切换至新节点。
- 槽迁移完成后，更新槽状态并生效。

槽位重平衡 

- ```
  redis-cli --cluster rebalance 集群内任一主节点IP:端口
  ```

**手动重分片，手动精确控制迁移的槽位数量**

```bash
redis-cli --cluster reshard 集群节点IP:端口
```

---

##### 三、Redis 高级特性

###### 8. 事务与 Lua 脚本

**事务特性：**
- 命令打包执行（MULTI/EXEC）
- 不支持回滚
- WATCH 实现乐观锁

**Lua 脚本优势：**
1. **原子性**：脚本执行期间不会插入其他命令
2. **减少网络开销**：多个命令一次执行
3. **复用**：脚本可缓存

**EVAL vs EVALSHA：**
```bash
# EVAL：每次发送完整脚本
EVAL "return redis.call('SET', KEYS[1], ARGV[1])" 1 mykey myvalue

# EVALSHA：使用脚本 SHA1 哈希（节省带宽）
SCRIPT LOAD "return redis.call('SET', KEYS[1], ARGV[1])"
# 返回：c686f316aaf1eb01d5a4de1b0b63cd233010e63d
EVALSHA c686f316aaf1eb01d5a4de1b0b63cd233010e63d 1 mykey myvalue
```

**分布式限流示例（令牌桶）：**
```lua
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local current = tonumber(redis.call('GET', key) or "0")

if current + 1 > limit then
    return 0  -- 超过限流
else
    redis.call('INCR', key)
    redis.call('EXPIRE', key, 1)
    return 1  -- 允许通过
end
```

###### 9. 发布订阅（Pub/Sub）

**基本用法：**
```bash
SUBSCRIBE channel1        # 订阅频道
PUBLISH channel1 "Hello"  # 发布消息
PSUBSCRIBE news.*         # 模式订阅（通配符）
```

**频道订阅 vs 模式订阅：**

| 类型 | 语法 | 匹配规则 | 性能 |
|------|------|---------|------|
| **频道订阅** | SUBSCRIBE channel | 精确匹配 | 快 |
| **模式订阅** | PSUBSCRIBE pattern | 通配符匹配（*、?、[]）| 慢 |

**Pub/Sub vs Stream：**

| 特性 | Pub/Sub | Stream |
|------|---------|--------|
| **持久化** | 无 | 有 |
| **消息历史** | 无 | 有（可回溯）|
| **消费者组** | 无 | 有 |
| **ACK 机制** | 无 | 有 |
| **适用场景** | 实时通知、聊天 | 消息队列、事件溯源 |

**消息丢失问题：**
- 订阅者离线时消息会丢失（无持久化）
- 发布者不关心是否有订阅者

###### 10. Pipeline 与慢查询

**Pipeline 批量操作：**
```python
# 减少 RTT（Round-Trip Time）
pipe = redis.pipeline()
pipe.set('key1', 'value1')
pipe.set('key2', 'value2')
pipe.get('key1')
results = pipe.execute()  # 一次网络往返
```

**Pipeline vs 事务：**

| 特性 | Pipeline | 事务（MULTI/EXEC）|
|------|---------|------------------|
| **原子性** | 无 | 有（命令打包执行）|
| **网络优化** | 有（减少 RTT）| 有 |
| **回滚** | 不支持 | 不支持 |
| **适用场景** | 批量操作 | 需要原子性的操作 |

**慢查询配置：**
```bash
# 超过 10ms 记录为慢查询（单位：微秒）
slowlog-log-slower-than 10000

# 最多保留 128 条慢查询记录
slowlog-max-len 128
```

**查看慢查询：**
```bash
SLOWLOG GET 10    # 查看最近 10 条
SLOWLOG LEN       # 查看数量
SLOWLOG RESET     # 清空日志
```

**性能分析建议：**
- 避免 O(N) 命令：KEYS、FLUSHALL、FLUSHDB
- 大 key 拆分：避免单个 key 过大
- 使用 SCAN 代替 KEYS

---

##### 四、AWS ElastiCache for Redis

###### 11. 架构与部署

**部署模式：**

| 模式 | 节点数 | 高可用 | 扩展性 | 适用场景 |
|------|-------|--------|--------|---------|
| **单节点** | 1 | 无 | 垂直扩展 | 开发/测试 |
| **主从复制** | 1 Master + N Replica | 有（自动故障转移）| 读扩展 | 生产环境 |
| **集群模式** | 多分片 | 有 | 水平扩展 | 大数据量 |

**多 AZ 部署：**
```
AZ-1a: Master
AZ-1b: Replica 1
AZ-1c: Replica 2
```

**自动故障转移：**
- 检测到 Master 故障（健康检查失败）
- 自动提升 Replica 为新 Master
- 更新 DNS 端点（30-60 秒）
- 应用自动重连

**Global Datastore（跨区域复制）：**
```
Primary Region (us-east-1)
    ↓ 异步复制（< 1 秒延迟）
Secondary Region (eu-west-1)
```

**优势：**
- 灾难恢复（RPO < 1 秒）
- 低延迟读取（就近访问）
- 跨区域故障转移

###### 12. 监控与告警

**CloudWatch 关键指标：**

| 指标 | 说明 | 告警阈值建议 |
|------|------|-------------|
| **CPUUtilization** | 主线程 CPU 使用率 | > 90% |
| **EngineCPUUtilization** | Redis 引擎 CPU | > 90% |
| **DatabaseMemoryUsagePercentage** | 内存使用率 | > 80% |
| **Evictions** | 淘汰的 key 数量 | > 0（持续）|
| **CurrConnections** | 当前连接数 | 接近 maxclients |
| **NetworkBytesIn/Out** | 网络吞吐量 | 接近实例限制 |
| **ReplicationLag** | 复制延迟 | > 10 秒 |
| **CacheHitRate** | 缓存命中率 | < 80% |

**告警配置示例：**
```bash
aws cloudwatch put-metric-alarm \
  --alarm-name redis-high-cpu \
  --metric-name CPUUtilization \
  --namespace AWS/ElastiCache \
  --statistic Average \
  --period 300 \
  --threshold 90 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2
```

###### 13. 安全配置

**传输加密（TLS）：**
```bash
aws elasticache create-replication-group \
  --replication-group-id my-redis \
  --transit-encryption-enabled \
  --auth-token "MyStrongPassword123!"
```

**静态加密：**
```bash
--at-rest-encryption-enabled \
--kms-key-id arn:aws:kms:region:account:key/xxx
```

**Redis ACL（6.0+）：**
```bash
# 创建用户
ACL SETUSER alice on >password ~cached:* +get +set

# 权限说明：
# on: 启用用户
# >password: 设置密码
# ~cached:*: 只能访问 cached: 前缀的 key
# +get +set: 只能执行 GET 和 SET 命令
```

**VPC 隔离：**
```
VPC
  ├─ Private Subnet (ElastiCache)
  ├─ Public Subnet (NAT Gateway)
  └─ Security Group
      ├─ Inbound: 6379 from App SG
      └─ Outbound: All
```

###### 14. 扩展与优化

**垂直扩展（在线）：**
```bash
aws elasticache modify-replication-group \
  --replication-group-id my-redis \
  --cache-node-type cache.r7g.xlarge \
  --apply-immediately
```

**水平扩展（集群模式）：**
```bash
# 添加分片
aws elasticache increase-replica-count \
  --replication-group-id my-redis \
  --new-replica-count 2 \
  --apply-immediately
```

**Data Tiering（r6gd/r7gd）：**
```
内存层（热数据）
    ↓ 自动分层
SSD 层（温数据）
```

**优势：**
- 成本降低 60%+
- 容量提升（内存 + SSD）
- 透明访问（应用无感知）

**参数组调优：**

| 参数 | 默认值 | 推荐值 | 说明 |
|------|-------|--------|------|
| **maxmemory-policy** | volatile-lru | allkeys-lru | 淘汰策略 |
| **timeout** | 0 | 300 | 空闲连接超时（秒）|
| **tcp-keepalive** | 300 | 60 | TCP keepalive 间隔 |
| **slowlog-log-slower-than** | 10000 | 10000 | 慢查询阈值（微秒）|

###### 15. 迁移策略

**从自建 Redis 迁移到 ElastiCache：**

**方案 1：在线迁移（DMS）**
```bash
aws dms create-replication-task \
  --replication-task-identifier redis-migration \
  --source-endpoint-arn arn:aws:dms:...:endpoint/source \
  --target-endpoint-arn arn:aws:dms:...:endpoint/target \
  --migration-type full-load-and-cdc
```

**方案 2：RDB 文件导入**
```bash
# 1. 生成 RDB 文件
redis-cli --rdb /tmp/dump.rdb

# 2. 上传到 S3
aws s3 cp /tmp/dump.rdb s3://my-bucket/

# 3. 导入到 ElastiCache
aws elasticache create-snapshot \
  --snapshot-name imported-data \
  --replication-group-id my-redis \
  --source-snapshot-name s3://my-bucket/dump.rdb
```

**方案 3：双写方案（零停机）**
```python
# 同时写入源和目标
def set_key(key, value):
    source_redis.set(key, value)
    target_redis.set(key, value)

# 切换读流量
def get_key(key):
    return target_redis.get(key)  # 切换到新集群
```

---

##### 五、Redis 生产最佳实践

###### 16. 缓存设计模式

| 模式              | 谁负责缓存 | 特点                                      |
| --------------- | ----- | --------------------------------------- |
| Cache-Aside（旁路） | 应用程序  | 读：先查缓存，miss 则查 DB 并回填<br>写：先更新 DB，再删除缓存 |
| Read-Through    | 缓存层   | 应用只访问缓存，缓存自动从 DB 加载                     |
| Write-Through   | 缓存层   | 应用写缓存，缓存自动同步写 DB                        |
| Write-Behind    | 缓存层   | 应用写缓存，缓存异步批量写 DB                        |

**Cache-Aside（旁路缓存）：**
```python
def get_user(user_id):
    # 1. 查缓存
    user = redis.get(f"user:{user_id}")
    if user:
        return user
    
    # 2. 查数据库
    user = db.query("SELECT * FROM users WHERE id = ?", user_id)
    
    # 3. 写缓存
    redis.setex(f"user:{user_id}", 3600, user)
    return user

def update_user(user_id, data):
    # 1. 更新数据库
    db.execute("UPDATE users SET ... WHERE id = ?", user_id)
    
    # 2. 删除缓存（而不是更新）
    redis.delete(f"user:{user_id}")
```

**缓存问题解决方案：**

| 问题       | 原因          | 解决方案                                                       |
| -------- | ----------- | ---------------------------------------------------------- |
| **缓存穿透** | 查询不存在的数据    | 1. 缓存空值（TTL 短）<br>2. 布隆过滤器**(最佳实践)**                       |
| **缓存击穿** | 热点 key 过期   | 1. 热点 key 永不过期（最佳）<br>2. 互斥锁重建（减轻但不能完全避免）<br>3. **提前异步刷新** |
| **缓存雪崩** | 大量 key 同时过期 | 1. 过期时间加随机值<br>2. 多级缓存                                     |

**布隆过滤器示例：**
```python
from redisbloom.client import Client

rb = Client()
rb.bfCreate('users', 0.01, 1000000)  # 1% 误判率，100 万容量

# 添加
rb.bfAdd('users', 'user:1000')

# 检查
if rb.bfExists('users', 'user:9999'):
    # 可能存在，查缓存/数据库
else:
    # 一定不存在，直接返回
```

###### 17. 常见问题解决

**热点 Key 问题：**

**识别：**
```bash
# 使用 redis-cli --hotkeys
redis-cli --hotkeys

# 使用 MONITOR（生产慎用）
redis-cli MONITOR | head -n 10000 | awk '{print $4}' | sort | uniq -c | sort -rn | head -n 10
```

**解决方案：**

1. **本地缓存：**
```python
from cachetools import TTLCache

local_cache = TTLCache(maxsize=1000, ttl=60)

def get_hot_key(key):
    if key in local_cache:
        return local_cache[key]
    
    value = redis.get(key)
    local_cache[key] = value
    return value
```

2. **多副本：**
```python
import random

# 将热点 key 复制多份
def set_hot_key(key, value):
    for i in range(10):
        redis.set(f"{key}:copy:{i}", value)

def get_hot_key(key):
    copy_id = random.randint(0, 9)
    return redis.get(f"{key}:copy:{copy_id}")
```

**大 Key 问题：**

**识别：**
```bash
# 扫描大 key
redis-cli --bigkeys

# 查看 key 大小
MEMORY USAGE mykey
```

**拆分策略：**
```python
# Hash 拆分
HSET user:1000:0 field1 value1 ... field100 value100
HSET user:1000:1 field101 value101 ... field200 value200

# List 按时间分片
LPUSH messages:2025-01-01 msg1
LPUSH messages:2025-01-02 msg2
```

**渐进式删除：**
```bash
# 使用 UNLINK 代替 DEL（异步删除）
UNLINK big_key

# 分批删除 Hash
HSCAN big_hash 0 COUNT 100
```

###### 18. 性能调优

**禁用危险命令：**
```bash
# redis.conf
rename-command KEYS ""
rename-command FLUSHALL ""
rename-command FLUSHDB ""
rename-command CONFIG "CONFIG_ADMIN_ONLY"
```

**使用 SCAN 代替 KEYS：**
```python
# 错误：阻塞服务器
keys = redis.keys("user:*")

# 正确：游标迭代
cursor = 0
while True:
    cursor, keys = redis.scan(cursor, match="user:*", count=100)
    for key in keys:
        process(key)
    if cursor == 0:
        break
```

**连接池优化：**
```python
from redis import ConnectionPool, Redis

pool = ConnectionPool(
    host='localhost',
    port=6379,
    max_connections=50,
    socket_keepalive=True,
    socket_connect_timeout=5,
    socket_timeout=5,
    retry_on_timeout=True
)

redis_client = Redis(connection_pool=pool)
```

**命令时间复杂度：**

| 命令          | 复杂度        | 建议         |
| ----------- | ---------- | ---------- |
| GET/SET     | O(1)       | ✅ 安全       |
| HGET/HSET   | O(1)       | ✅ 安全       |
| LPUSH/RPUSH | O(1)       | ✅ 安全       |
| LRANGE      | O(N)       | ⚠️ 限制范围    |
| KEYS        | O(N)       | 🔴 禁用      |
| SMEMBERS    | O(N)       | ⚠️ 用 SSCAN |
| SORT        | O(N log N) | ⚠️ 慎用      |

---

##### Redis 实战场景

###### 典型应用场景

| 场景       | 业务描述           | 数据结构              | 并发量      | 关键指标      | 技术要点              |
| :------- | :------------- | :---------------- | :------- | :-------- | :---------------- |
| **缓存系统** | 热点数据缓存、查询结果缓存  | String、Hash       | 10万 QPS  | 命中率 > 95% | 过期策略、淘汰策略         |
| **会话存储** | 用户登录态、购物车      | String、Hash       | 5万 QPS   | 高可用、持久化   | 主从复制、AOF          |
| **分布式锁** | 秒杀、库存扣减、防重复提交  | String            | 1万 TPS   | 原子性、可靠性   | Redisson、RedLock  |
| **排行榜**  | 游戏积分、热门文章、销量排名 | Sorted Set        | 5000 QPS | 实时更新、范围查询 | ZADD、ZRANGE       |
| **消息队列** | 异步任务、事件通知      | List、Stream       | 1万 TPS   | 可靠性、顺序性   | LPUSH/RPOP、Stream |
| **限流**   | API 限流、防刷      | String、Sorted Set | 10万 QPS  | 精确控制、高性能  | 令牌桶、滑动窗口          |
| **计数器**  | 点赞数、浏览量、库存     | String            | 5万 TPS   | 原子性、高并发   | INCR、DECR         |
| **地理位置** | 附近的人、配送范围      | Geo               | 1000 QPS | 距离计算、范围查询 | GEOADD、GEORADIUS  |

---

###### 常见问题与解决方案

##### 问题 1：数据一致性（缓存与数据库）

**现象：**
```
场景：更新数据库后，缓存未更新，导致读取到旧数据
影响：数据不一致，用户看到过期信息
```

**原因分析：**

缓存一致性的根本问题：应用代码手动维护缓存，容易出现删除失败、并发回填等问题。

**✅ 完美方案：Binlog/CDC（推荐）**

通过监听数据库Binlog自动删除缓存，应用代码无需关心缓存维护：

```
流程：
1. 应用只更新数据库（无需操作缓存）
2. MySQL 写入 Binlog
3. Canal/Debezium 监听 Binlog
4. 自动删除对应缓存

优势：
✅ 完全解耦（应用无需关心缓存）
✅ 可靠（Binlog 保证不丢失）
✅ 无并发问题（基于事务日志）
✅ 自动重试
✅ 无侵入（不修改业务代码）
```

**⚠️ 传统方案（应用代码手动维护，都有问题）**

如果无法使用Binlog/CDC，传统方案都存在可靠性问题：

```
策略 1：先更新数据库，再删除缓存
问题：
❌ 删除缓存失败 → 缓存中是旧数据
❌ 并发读写可能导致不一致

策略 2：先删除缓存，再更新数据库
问题：
❌ 删除缓存后，并发读取会回填旧数据
❌ 不一致窗口期长
❌ 更新数据库失败，缓存已被删除

策略 3：先更新缓存，再更新数据库
问题：
❌ 更新数据库失败 → 缓存和数据库不一致
❌ 违反"数据库是权威数据源"原则

结论：传统方案都无法完美解决一致性问题
```

**传统方案的补救措施（治标不治本）**

```python
# ✅ 方案 1：订阅 Binlog/CDC（最佳实践）
# 使用 Canal/Debezium 监听 MySQL Binlog
# 
# 流程：
# 1. 应用更新数据库
# 2. MySQL 写入 Binlog
# 3. Canal 监听 Binlog 变更
# 4. Canal 删除对应缓存（普通数据）
#    或 删除后立即更新缓存（热点数据）
# 5. 下次读取时，缓存未命中，从DB读取并回填
# 
# 删除 vs 更新缓存的选择：
# 
# 普通数据：删除缓存（推荐）
# ✅ 避免并发更新导致缓存数据错乱
# ✅ 避免复杂查询的缓存更新逻辑
# ✅ 懒加载，只缓存真正被访问的数据
# 
# 热点数据（hot key）：删除后立即更新（必要）
# ✅ 避免缓存击穿（大量请求同时回源DB）
# ✅ 保持缓存始终有效
# ⚠️ 需要识别热点数据（访问频率统计）
# ⚠️ 更新逻辑需要从Binlog解析或重新查询DB
# 
# 示例：
# - 普通用户信息 → 删除缓存
# - 热门商品详情 → 删除后立即更新
# - 秒杀商品库存 → 删除后立即更新
# 
# 优势：
# ✅ 完全解耦（应用无需关心缓存）
# ✅ 可靠（Binlog 保证不丢失）
# ✅ 自动重试
# ✅ 无侵入（不修改业务代码）
# ✅ 同时解决删除失败和并发回填问题
# ✅ 可针对热点数据特殊处理

# ⚠️ 方案 2：异步消息队列（有问题）
def update_user(user_id, data):
    db.update("UPDATE users SET ... WHERE id = ?", user_id)
    mq.send("cache_delete_queue", {"user_id": user_id})

def cache_delete_consumer():
    while True:
        msg = mq.receive("cache_delete_queue")
        try:
            redis.delete(f"user:{msg['user_id']}")
            mq.ack(msg)
        except Exception:
            mq.nack(msg)

# 问题：
# ❌ 无法解决并发回填（删除后仍可能被回填旧数据）
# ❌ 消息可能丢失（应用崩溃在发送MQ之前）
# ❌ 消息乱序（多次更新同一数据）

# ⚠️ 方案 3：重试机制（有限可靠性）
def update_user(user_id, data):
    db.update("UPDATE users SET ... WHERE id = ?", user_id)
    
    max_retries = 3
    for i in range(max_retries):
        try:
            redis.delete(f"user:{user_id}")
            break
        except Exception as e:
            if i == max_retries - 1:
                log_failed_cache_delete(user_id)  # 仍可能失败
            time.sleep(0.1 * (i + 1))

# 问题：重试 3 次后仍可能失败，无法保证最终一致性

# ❌ 方案 4：延迟双删（不推荐）
def update_user(user_id, data):
    redis.delete(f"user:{user_id}")  # 第一次删除
    db.update("UPDATE users SET ... WHERE id = ?", user_id)
    time.sleep(0.5)
    redis.delete(f"user:{user_id}")  # 第二次删除

# 问题：
# ❌ 如果两次删除都失败，仍然不一致
# ❌ 延迟时间难以确定（500ms 可能不够）
# ❌ 阻塞线程，影响性能
# ❌ 只解决并发回填，不解决删除失败
```

**辅助措施（仅用于传统方案，治标不治本）**

⚠️ **重要：辅助措施只能缓解问题，无法解决根本问题**

```python
# 措施 1：设置过期时间（兜底）
redis.setex(f"user:{user_id}", 300, data)  # 5 分钟过期
# 缓解效果：最多 5 分钟后自动修复
# 根本问题：5 分钟内仍然不一致
# 注：Binlog/CDC 方案也建议配置过期时间作为双重保险

# 措施 2：分布式锁（防止并发）
lock = redis.lock(f"lock:user:{user_id}", timeout=5)
if lock.acquire():
    try:
        db.update(...)
        redis.delete(f"user:{user_id}")
    finally:
        lock.release()
# 缓解效果：防止并发导致的不一致
# 根本问题：无法解决删除失败、消息丢失、消息乱序
```

**为什么辅助措施无法解决根本问题？**

| 传统方案问题  | 过期时间能解决？      | 分布式锁能解决？ | 根本原因          |
| ------- | ------------- | -------- | ------------- |
| 删除缓存失败  | ⚠️ 缓解（5分钟后修复） | 🔴 不能    | 应用代码手动维护缓存不可靠 |
| 消息丢失    | ⚠️ 缓解         | 🔴 不能    | 应用崩溃在发送MQ之前   |
| 消息乱序    | 🔴 不能         | 🔴 不能    | MQ无法保证严格顺序    |
| 并发回填旧数据 | ⚠️ 缓解         | ✅ 能      | 删除和回填之间有时间窗口  |

**结论：只有 Binlog/CDC 能从根本上解决问题（基于事务日志，不依赖应用代码）**

---

**方案对比：**

| 方案               | 解决删除失败    | 解决并发回填 | 需要辅助措施 | 推荐度  |
| ---------------- | --------- | ------ | ------ | ---- |
| **✅ Binlog/CDC** | ✅ 完全      | ✅ 完全   | 🔴 不需要 | 唯一推荐 |
| **⚠️ 消息队列**      | ⚠️ 部分     | 🔴     | ✅ 需要   | 有问题  |
| **⚠️ 重试机制**      | ⚠️ 部分（3次） | 🔴     | ✅ 需要   | 有问题  |
| **🔴 延迟双删**      | 🔴        | ⚠️ 部分  | ✅ 需要   | 不推荐  |

**最佳实践：**

```
✅ 唯一推荐：
Binlog/CDC + 过期时间

其他方案都有可靠性问题，不推荐使用
```

---

##### 问题 2：延时队列实现

**现象：**
```
场景：订单 30 分钟未支付自动取消
需求：延时执行任务
```

**解决方案：**
```python
# ✅ 方案 1：Redisson DelayedQueue（推荐）
from redisson import Redisson

redisson = Redisson.create()
queue = redisson.get_delayed_queue("delay_queue")

# 添加延时任务
queue.offer("task_123", 30, TimeUnit.MINUTES)

# 消费任务
while True:
    task = queue.poll()
    if task:
        process_task(task)

# 优势：
# ✅ 封装完善，开箱即用
# ✅ 自动处理重试、持久化
# ✅ 支持分布式
# ✅ 高可靠性

# 方案 2：Redis Stream + 消费者组（大规模场景）
# 添加任务
r.xadd("delay_stream", {"task_id": "123", "delay": 1800})

# 消费任务（带 ACK）
while True:
    messages = r.xreadgroup("group1", "consumer1", {"delay_stream": ">"}, count=10, block=1000)
    for stream, msgs in messages:
        for msg_id, data in msgs:
            if time.time() >= float(data["delay"]):
                process_task(data["task_id"])
                r.xack("delay_stream", "group1", msg_id)

# 优势：高吞吐量、消费者组支持
# 劣势：需要自己实现延时逻辑

# ⚠️ 方案 3：Sorted Set + 定时轮询（简单场景）
import time
import redis

r = redis.Redis()

# 添加延时任务
def add_delay_task(task_id, delay_seconds):
    execute_time = time.time() + delay_seconds
    r.zadd("delay_queue", {task_id: execute_time})

# 消费延时任务
def consume_delay_tasks():
    while True:
        now = time.time()
        tasks = r.zrangebyscore("delay_queue", 0, now, start=0, num=10)
        
        for task_id in tasks:
            if r.zrem("delay_queue", task_id):
                process_task(task_id)
        
        time.sleep(1)  # 每秒轮询

# 优势：实现简单
# 劣势：轮询开销、需手动处理失败
```

**方案对比：**

| 方案 | 可靠性 | 性能 | 复杂度 | 推荐度 |
|------|--------|------|--------|---------|
| **✅ Redisson DelayedQueue** | ✅ 高（自动重试、持久化） | ✅ 高 | 低 | ✅ 最佳实践 |
| **Sorted Set + 轮询** | ⚠️ 中（需手动处理失败） | ⚠️ 中（轮询开销） | 低 | 简单场景 |
| **Redis Stream** | ✅ 高（消费者组、ACK） | ✅ 高 | 中 | 大规模场景 |
| **RabbitMQ Delayed Plugin** | ✅ 高 | ✅ 高 | 高 | 需要MQ时 |

**最佳实践：**

```
✅ 推荐方案（按优先级）：

1. Redisson DelayedQueue（✅ 最佳实践）
   - 封装完善，开箱即用
   - 自动处理重试、持久化
   - 支持分布式

2. Redis Stream + 消费者组（大规模场景）
   - 高吞吐量
   - 消费者组支持
   - 需要自己实现延时逻辑

3. Sorted Set + 轮询（简单场景）
   - 实现简单
   - 适合小规模
   - 需要手动处理失败

不推荐：
❌ 数据库轮询（性能差）
❌ JDK DelayQueue（单机，不支持分布式）
```

---

##### 问题 3：限流方案

**现象：**
```
场景：API 限流，每秒最多 100 次请求
需求：防止恶意刷接口
```

**解决方案：**
```python
# ✅ 方案 1：Redisson RateLimiter（推荐）
from redisson import Redisson

redisson = Redisson.create()
limiter = redisson.get_rate_limiter("api_limiter")
limiter.try_set_rate(RateType.OVERALL, 100, 1, RateIntervalUnit.SECONDS)

# 尝试获取许可
if limiter.try_acquire():
    process_request()
else:
    return "Too Many Requests"

# 优势：
# ✅ 封装完善，开箱即用
# ✅ 支持多种限流算法（令牌桶、漏桶）
# ✅ 分布式支持
# ✅ 高性能

# 方案 2：滑动窗口（精确限流）
def sliding_window_limiter(user_id, limit=100, window=1):
    key = f"limiter:{user_id}"
    now = time.time()
    
    # 删除过期记录
    r.zremrangebyscore(key, 0, now - window)
    
    # 统计当前窗口请求数
    count = r.zcard(key)
    
    if count < limit:
        r.zadd(key, {str(now): now})
        r.expire(key, window)
        return True
    return False

# 优势：精确限流，无临界问题
# 劣势：内存占用较高（存储每次请求时间戳）

# 方案 3：令牌桶 Lua 脚本（高性能）
lua_script = """
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local current = tonumber(redis.call('GET', key) or "0")

if current + 1 > limit then
    return 0
else
    redis.call('INCR', key)
    redis.call('EXPIRE', key, 1)
    return 1
end
"""

def token_bucket_limiter(user_id, limit=100):
    return r.eval(lua_script, 1, f"limiter:{user_id}", limit)

# 优势：高性能，原子操作
# 劣势：需要自己实现令牌桶逻辑

# ⚠️ 方案 4：固定窗口（简单但不精确）
def fixed_window_limiter(user_id, limit=100):
    key = f"limiter:{user_id}:{int(time.time())}"
    count = r.incr(key)
    r.expire(key, 1)
    return count <= limit

# 优势：实现简单
# 劣势：临界问题（窗口边界可能超限）
```

**方案对比：**

| 方案                         | 精确度        | 性能   | 复杂度 | 推荐度    |
| -------------------------- | ---------- | ---- | --- | ------ |
| **✅ Redisson RateLimiter** | ✅ 高        | ✅ 高  | 低   | ✅ 最佳实践 |
| **滑动窗口**                   | ✅ 高        | ⚠️ 中 | 中   | 精确限流场景 |
| **Lua 脚本**                 | ✅ 高        | ✅ 高  | 中   | 自定义逻辑  |
| **⚠️ 固定窗口**                | 🔴 低（临界问题） | ✅ 高  | 低   | 简单场景   |

**最佳实践：**

```
✅ 推荐方案（按优先级）：

1. Redisson RateLimiter（✅ 最佳实践）
   - 封装完善，支持多种算法
   - 分布式支持
   - 高性能

2. 滑动窗口（精确限流场景）
   - 无临界问题
   - 适合需要精确控制的场景

3. Lua 脚本（自定义逻辑）
   - 高性能
   - 灵活可定制

不推荐：
❌ 固定窗口（有临界问题，窗口边界可能超限2倍）
```

---

##### 问题 4：会话存储方案

**现象：**
```
场景：用户登录后，多台服务器共享会话
需求：高可用、快速访问
```

**⚠️ 重要：容器化时代推荐无状态架构**

**✅ 现代方案：JWT + 无状态（推荐）**

```python
# 方案 1：JWT Token（✅ 最佳实践）
import jwt
from datetime import datetime, timedelta

def generate_token(user_id, user_data):
    payload = {
        'user_id': user_id,
        'username': user_data['username'],
        'exp': datetime.utcnow() + timedelta(hours=24)
    }
    return jwt.encode(payload, SECRET_KEY, algorithm='HS256')

def verify_token(token):
    try:
        return jwt.decode(token, SECRET_KEY, algorithms=['HS256'])
    except jwt.ExpiredSignatureError:
        return None

# 优势：
# ✅ 无状态，容器可随意扩缩容
# ✅ 无需 Redis 存储会话
# ✅ 跨域友好
# ✅ 减少服务器压力

# 劣势：
# ⚠️ 无法主动失效（需配合黑名单）
# ⚠️ Token 较大（包含用户信息）
```

**传统方案：Redis 会话存储（遗留系统）**

仅在以下场景使用：
- 遗留系统无法改造为无状态
- 需要主动踢出用户
- 需要实时更新会话数据

```python
# 方案 2：Hash 存储（传统方案）
def set_session_hash(session_id, user_data, expire=3600):
    key = f"session:{session_id}"
    r.hset(key, mapping=user_data)
    r.expire(key, expire)

def get_session_hash(session_id):
    return r.hgetall(f"session:{session_id}")

# 方案 3：String 存储（简单场景）
import json

def set_session(session_id, user_data, expire=3600):
    r.setex(f"session:{session_id}", expire, json.dumps(user_data))

def get_session(session_id):
    data = r.get(f"session:{session_id}")
    return json.loads(data) if data else None
```

**方案对比：**

| 方案                 | 无状态  | 扩展性           | 复杂度 | 推荐度    |
| ------------------ | ---- | ------------- | --- | ------ |
| **✅ JWT**          | ✅ 是  | ✅ 高（容器友好）     | 低   | ✅ 最佳实践 |
| **Redis Hash**     | 🔴 否 | ⚠️ 中（依赖Redis） | 中   | 遗留系统   |
| **Redis String**   | 🔴 否 | ⚠️ 中          | 低   | 遗留系统   |
| **Spring Session** | 🔴 否 | ⚠️ 中          | 低   | 遗留系统   |

**最佳实践：**

```
✅ 容器化时代推荐：
JWT + 无状态架构

传统方案（不推荐）：
Redis 会话存储（仅用于遗留系统）

混合方案：
JWT + Redis 黑名单（需要主动踢出用户时）
```

---

##### 问题 5：排行榜实现

**现象：**
```
场景：游戏积分排行榜，实时更新
需求：高性能、支持范围查询
```

**解决方案：**
```python
# ✅ 方案 1：Sorted Set（✅ 最佳实践）
# 更新分数
def update_score(user_id, score):
    r.zadd("leaderboard", {user_id: score})

# 获取排行榜（Top 10）
def get_top_10():
    return r.zrevrange("leaderboard", 0, 9, withscores=True)

# 获取用户排名
def get_user_rank(user_id):
    rank = r.zrevrank("leaderboard", user_id)
    return rank + 1 if rank is not None else None

# 获取用户周围排名
def get_nearby_ranks(user_id, range=5):
    rank = r.zrevrank("leaderboard", user_id)
    if rank is None:
        return []
    start = max(0, rank - range)
    end = rank + range
    return r.zrevrange("leaderboard", start, end, withscores=True)

# 优势：
# ✅ O(log N) 时间复杂度
# ✅ 支持范围查询、排名查询
# ✅ 自动排序
# ✅ 内存高效

# 方案 2：多维度排行榜（扩展场景）
# 日榜
r.zadd("leaderboard:daily:2024-01-01", {user_id: score})

# 周榜
r.zadd("leaderboard:weekly:2024-W01", {user_id: score})

# 月榜
r.zadd("leaderboard:monthly:2024-01", {user_id: score})

# 定时任务清理过期榜单
def cleanup_old_leaderboards():
    old_date = (datetime.now() - timedelta(days=7)).strftime("%Y-%m-%d")
    r.delete(f"leaderboard:daily:{old_date}")

# 方案 3：分段排行榜（千万级数据量）
# 按分数段分片，减少单个 Sorted Set 大小
def get_shard_key(score):
    return f"leaderboard:shard:{score // 1000}"

def update_score_sharded(user_id, score):
    shard_key = get_shard_key(score)
    r.zadd(shard_key, {user_id: score})

# 适用场景：数据量 > 1000万
```

**方案对比：**

| 方案 | 适用场景 | 性能 | 复杂度 | 推荐度 |
|------|---------|------|--------|---------|
| **✅ Sorted Set** | 通用排行榜 | ✅ 高（O(log N)） | 低 | ✅ 最佳实践 |
| **多维度榜单** | 日/周/月榜 | ✅ 高 | 中 | 扩展场景 |
| **分段榜单** | 千万级数据 | ⚠️ 中（需合并） | 高 | 超大规模 |

**最佳实践：**

```
✅ 通用场景：
Sorted Set（Redis 原生支持，性能最优）

扩展场景：
- 多维度榜单（日/周/月）
- 定期清理过期数据

超大规模（> 1000万）：
- 分段存储
- 缓存 Top 100
```
- ✅ 异步更新排行榜（减少延迟）

---

##### 问题 6：内存优化策略

**现象：**
```
场景：Redis 内存占用过高，接近 maxmemory
影响：触发淘汰策略，性能下降
```

**优化步骤（按顺序执行）：**

```python
# 步骤 1：设置过期时间（必须，第一步）
# 所有 key 都必须设置过期时间，防止内存无限增长
r.setex("key", 3600, "value")
r.expire("existing_key", 3600)

# 检查没有过期时间的 key
# redis-cli --scan --pattern '*' | xargs -L 1 redis-cli TTL | grep -c "^-1"

# 步骤 2：使用内存分析工具（找出问题）
# 找出占用内存最多的 key
# redis-cli --bigkeys

# 分析内存使用详情
# redis-cli --memkeys

# 步骤 3：优化数据结构（节省内存）
# ❌ 差：大量 String key
for i in range(1000000):
    r.set(f"user:{i}:name", "Tom")
    r.set(f"user:{i}:age", 25)
# 内存占用：~200MB

# ✅ 好：使用 Hash
for i in range(1000000):
    r.hset(f"user:{i}", mapping={"name": "Tom", "age": 25})
# 内存占用：~100MB（节省 50%）

# 步骤 4：启用内存淘汰策略（兜底）
# redis.conf
maxmemory 2gb
maxmemory-policy allkeys-lru  # 或 volatile-lru

# 淘汰策略选择：
# - allkeys-lru: 所有 key 中淘汰最少使用的（推荐）
# - volatile-lru: 只淘汰设置了过期时间的 key
# - allkeys-lfu: 所有 key 中淘汰访问频率最低的（Redis 4.0+）

# 步骤 5：数据分层存储（架构优化）
# 热数据：Redis（高频访问）
# 温数据：Redis + 短过期时间（1小时）
# 冷数据：MySQL/MongoDB（低频访问）

def get_user_data(user_id):
    # 先查 Redis
    data = r.get(f"user:{user_id}")
    if data:
        return json.loads(data)
    
    # Redis 未命中，查数据库
    data = db.query("SELECT * FROM users WHERE id = ?", user_id)
    
    # 回填 Redis（短过期时间）
    r.setex(f"user:{user_id}", 3600, json.dumps(data))
    return data

# 步骤 6：压缩数据（可选，大对象场景）
import json
import zlib

def set_compressed(key, data, expire=3600):
    compressed = zlib.compress(json.dumps(data).encode())
    r.setex(key, expire, compressed)

def get_compressed(key):
    compressed = r.get(key)
    return json.loads(zlib.decompress(compressed)) if compressed else None

# 适用场景：单个 value > 1KB
# 压缩率：通常 50-70%
```

**优化效果对比：**

| 优化步骤 | 内存节省 | 实施难度 | 优先级 |
|---------|---------|---------|--------|
| **1. 设置过期时间** | 30-50% | 低 | ✅ 必须（第一步） |
| **2. 内存分析** | - | 低 | ✅ 必须（找问题） |
| **3. 优化数据结构** | 40-60% | 中 | ✅ 推荐 |
| **4. 淘汰策略** | - | 低 | ✅ 必须（兜底） |
| **5. 数据分层** | 60-80% | 高 | ✅ 推荐 |
| **6. 压缩数据** | 50-70% | 中 | 可选（大对象） |

**执行顺序：**

```
1. 设置过期时间（立即执行，防止内存泄漏）
   ↓
2. 内存分析（找出大 key 和热点数据）
   ↓
3. 优化数据结构（Hash 代替多个 String）
   ↓
4. 启用淘汰策略（兜底保护）
   ↓
5. 数据分层存储（架构级优化）
   ↓
6. 压缩数据（可选，针对大对象）
```

**监控指标：**
- ✅ 内存使用率 < 80%
- ✅ 淘汰 key 数量（evicted_keys）
- ✅ 过期 key 数量（expired_keys）
- ✅ 大 key 数量和大小

---

##### 问题 7：集群扩容方案

**现象：**
```
场景：Redis 集群容量不足，需要扩容
挑战：数据迁移、槽位重分配
```

**解决方案：**
```bash
# 方案 1：添加新节点（主节点）
# 1. 启动新节点
redis-server --port 7006 --cluster-enabled yes

# 2. 加入集群
redis-cli --cluster add-node 127.0.0.1:7006 127.0.0.1:7000

# 3. 重新分配槽位
redis-cli --cluster reshard 127.0.0.1:7000
# 输入要迁移的槽位数量
# 输入目标节点 ID
# 输入源节点 ID（all 表示所有节点）

# 4. 验证槽位分配
redis-cli --cluster check 127.0.0.1:7000

# 方案 2：添加副本节点
redis-cli --cluster add-node 127.0.0.1:7007 127.0.0.1:7000 --cluster-slave --cluster-master-id <master-id>

# 方案 3：在线扩容（零停机）
# 使用 redis-cli --cluster reshard
# 数据迁移过程中，集群仍可正常服务
# 客户端自动处理 MOVED/ASK 重定向

# 方案 4：缩容（删除节点）
# 1. 迁移槽位到其他节点
redis-cli --cluster reshard 127.0.0.1:7000 --cluster-from <node-id> --cluster-to <target-id> --cluster-slots <count>

# 2. 删除节点
redis-cli --cluster del-node 127.0.0.1:7000 <node-id>

# 方案 5：自动化扩容（AWS ElastiCache）
aws elasticache modify-replication-group \
  --replication-group-id my-redis \
  --cache-node-type cache.r7g.xlarge \
  --apply-immediately
```

**最佳实践：**
- ✅ 提前规划分片数量（2^n）
- ✅ 使用 redis-cli --cluster 工具
- ✅ 扩容时监控数据迁移进度
- ✅ 避免高峰期扩容
- ✅ 使用智能客户端（自动处理重定向）
- ✅ 云环境使用托管服务（ElastiCache）
- ✅ 定期演练扩容流程

---

##### 列族宽表数据库

##### Cassandra

##### Cassandra 核心架构

**分布式特点**

| 维度 | Cassandra | MongoDB | MySQL |
|:---|:---|:---|:---|
| 架构 | 无主节点（P2P） | 主从复制 | 主从复制 |
| 一致性 | 可调一致性 | 强一致性 | 强一致性 |
| 可用性 | 极高（AP） | 高（CP） | 中（CP） |
| 扩展性 | 线性扩展 | 水平扩展 | 垂直扩展 |
| 写性能 | 极高 | 高 | 中 |
| 适用场景 | IoT、时序、高写入 | 文档存储 | OLTP |

##### Cassandra 数据模型

**宽列存储**

```sql
-- 创建 Keyspace（类似数据库）
CREATE KEYSPACE myapp WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'datacenter1': 3
};

-- 创建表
CREATE TABLE users (
  user_id UUID,
  name TEXT,
  age INT,
  email TEXT,
  created_at TIMESTAMP,
  PRIMARY KEY (user_id)
);

-- 宽列表（时序数据）
CREATE TABLE sensor_data (
  sensor_id UUID,
  timestamp TIMESTAMP,
  temperature DOUBLE,
  humidity DOUBLE,
  PRIMARY KEY (sensor_id, timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC);
```

**分区键 vs 聚簇键**

```sql
-- PRIMARY KEY = (分区键, 聚簇键)
CREATE TABLE orders (
  user_id UUID,        -- 分区键（决定数据分布到哪个节点）
  order_date DATE,     -- 聚簇键（决定分区内数据排序）
  order_id UUID,       -- 聚簇键
  amount DECIMAL,
  PRIMARY KEY ((user_id), order_date, order_id)
);

-- 数据分布：
-- 分区键 user_id 决定数据存储在哪个节点
-- 聚簇键 (order_date, order_id) 决定分区内排序

-- 高效查询（单分区查询）
SELECT * FROM orders WHERE user_id = ? AND order_date = ?;

-- 低效查询（跨分区查询，需要扫描所有节点）
SELECT * FROM orders WHERE order_date = ?;  -- ❌ 缺少分区键
```

##### Cassandra 一致性级别

**可调一致性**

| 级别 | 说明 | 读写节点数 | 一致性 | 性能 |
|:---|:---|:---|:---|:---|
| ONE | 1 个节点响应 | 1 | 弱 | 最快 |
| QUORUM | 多数节点响应 | N/2 + 1 | 强 | 中 |
| ALL | 所有节点响应 | N | 最强 | 最慢 |
| LOCAL_QUORUM | 本地数据中心多数 | 本地 N/2 + 1 | 强（本地） | 快 |

```sql
-- 写入（QUORUM 级别）
INSERT INTO users (user_id, name) VALUES (?, ?)
USING CONSISTENCY QUORUM;

-- 读取（QUORUM 级别）
SELECT * FROM users WHERE user_id = ?
USING CONSISTENCY QUORUM;

-- 强一致性保证：
-- W + R > N（W=写节点数，R=读节点数，N=副本数）
-- 例如：W=2, R=2, N=3 → 2 + 2 > 3 ✅ 强一致性
```

**一致性哈希**

```
一致性哈希环（Token Ring）：

        Token: 0
           ↓
    ┌──────────────┐
    │              │
Node A          Node B
(Token: 0)    (Token: 85)
    │              │
    └──────────────┘
           ↑
      Token: 170

数据分布：
- Key 哈希后映射到环上
- 顺时针找到第一个节点
- 例如：Key 哈希值 = 50 → 存储在 Node B

节点加入/离开：
- 只影响相邻节点
- 数据迁移量最小
```

##### Cassandra 写入流程

**写入路径**

```
写入流程（极快）：
1. 写入 CommitLog（顺序写，持久化）
2. 写入 MemTable（内存）
3. 返回成功（异步刷盘）

┌─────────────────────────────────┐
│  1. 写入 CommitLog（磁盘）        │
│     ├─ 顺序写入                  │
│     └─ 持久化保证                │
└────────────┬────────────────────┘
             ↓
┌─────────────────────────────────┐
│  2. 写入 MemTable（内存）         │
│     ├─ 排序存储                  │
│     └─ 快速写入                  │
└────────────┬────────────────────┘
             ↓
┌─────────────────────────────────┐
│  3. MemTable 满时刷盘            │
│     ├─ 写入 SSTable（磁盘）       │
│     └─ 不可变文件                 │
└─────────────────────────────────┘

写入性能：
- 顺序写 CommitLog（极快）
- 内存写 MemTable（极快）
- 无需读取旧数据（无回表）
```

**SSTable 合并（Compaction）**

```
Compaction 策略：

1. SizeTieredCompactionStrategy（STCS）
   - 合并相似大小的 SSTable
   - 适合写多读少场景

2. LeveledCompactionStrategy（LCS）
   - 分层合并（类似 LSM-Tree）
   - 适合读多写少场景

3. TimeWindowCompactionStrategy（TWCS）
   - 按时间窗口合并
   - 适合时序数据
```

##### Cassandra 读取流程

**读取路径**

```
读取流程：
1. 查询 MemTable（内存）
2. 查询 Bloom Filter（判断 SSTable 是否包含数据）
3. 查询 SSTable（磁盘）
4. 合并结果（多版本数据）

┌─────────────────────────────────┐
│  1. 查询 MemTable                │
│     └─ 最新数据                  │
└────────────┬────────────────────┘
             ↓
┌─────────────────────────────────┐
│  2. Bloom Filter 过滤            │
│     ├─ 快速判断 SSTable 是否包含   │
│     └─ 减少磁盘 I/O              │
└────────────┬────────────────────┘
             ↓
┌─────────────────────────────────┐
│  3. 查询 SSTable                 │
│     ├─ 索引查找                  │
│     └─ 读取数据块                 │
└────────────┬────────────────────┘
             ↓
┌─────────────────────────────────┐
│  4. 合并多版本数据                │
│     └─ 返回最新版本               │
└─────────────────────────────────┘
```

##### Cassandra 数据中心复制

**多数据中心架构**

```
┌─────────────────────────────────────────────┐
│         数据中心 1（北京）                     │
│  ┌──────┐  ┌──────┐  ┌──────┐               │
│  │Node A│  │Node B│  │Node C│               │
│  └──────┘  └──────┘  └──────┘               │
└────────────────┬────────────────────────────┘
                 │ 跨数据中心复制
┌────────────────▼────────────────────────────┐
│         数据中心 2（上海）                     │
│  ┌──────┐  ┌──────┐  ┌──────┐               │
│  │Node D│  │Node E│  │Node F│               │
│  └──────┘  └──────┘  └──────┘               │
└─────────────────────────────────────────────┘

复制策略：
CREATE KEYSPACE myapp WITH replication = {
  'class': 'NetworkTopologyStrategy',
  'datacenter1': 3,  // 北京 3 副本
  'datacenter2': 3   // 上海 3 副本
};
```

##### Cassandra 性能优化

**分区键设计**

```sql
-- ❌ 差：热点分区
CREATE TABLE logs (
  date DATE,           -- 分区键（所有数据在同一天会写入同一节点）
  timestamp TIMESTAMP,
  message TEXT,
  PRIMARY KEY (date, timestamp)
);

-- ✅ 好：分散分区
CREATE TABLE logs (
  date DATE,
  bucket INT,          -- 添加 bucket 分散数据
  timestamp TIMESTAMP,
  message TEXT,
  PRIMARY KEY ((date, bucket), timestamp)
);
```

**批量操作**

```sql
-- 批量写入（同一分区）
BEGIN BATCH
  INSERT INTO users (user_id, name) VALUES (?, ?);
  INSERT INTO users (user_id, name) VALUES (?, ?);
APPLY BATCH;

-- ⚠️ 避免跨分区批量（性能差）
BEGIN BATCH
  INSERT INTO users (user_id, name) VALUES (uuid1, ?);  -- 分区 1
  INSERT INTO users (user_id, name) VALUES (uuid2, ?);  -- 分区 2
APPLY BATCH;
```

---

##### Cassandra 实战场景

###### 典型应用场景

| 场景 | 业务描述 | 数据量 | 并发量 | 关键指标 | 技术要点 |
|:---|:---|:---|:---|:---|:---|
| **IoT 数据** | 传感器数据、设备状态 | 亿级 | 10万 TPS | 高写入、时序查询 | 时间分区、TTL |
| **时序数据** | 监控指标、日志 | 亿级 | 5万 TPS | 高写入、范围查询 | 复合分区键 |
| **消息存储** | 聊天记录、通知 | 亿级 | 5万 TPS | 高可用、最终一致性 | AP 架构 |

---

###### 常见问题与解决方案

##### 问题 1：分区键设计

**现象：**
```cql
-- 热点分区，写入集中在单个节点
CREATE TABLE sensor_data (
  sensor_id UUID,
  timestamp TIMESTAMP,
  value DOUBLE,
  PRIMARY KEY (sensor_id, timestamp)
);
-- 问题：某个传感器数据量大，导致单分区过大
```

**解决方案：**
```cql
-- 方案 1：添加 bucket 分散数据
CREATE TABLE sensor_data (
  sensor_id UUID,
  bucket INT,  -- 0-9，按小时取模
  timestamp TIMESTAMP,
  value DOUBLE,
  PRIMARY KEY ((sensor_id, bucket), timestamp)
);

-- 写入时计算 bucket
INSERT INTO sensor_data (sensor_id, bucket, timestamp, value)
VALUES (?, hour % 10, ?, ?);

-- 方案 2：时间分区
CREATE TABLE sensor_data (
  date DATE,
  sensor_id UUID,
  timestamp TIMESTAMP,
  value DOUBLE,
  PRIMARY KEY ((date, sensor_id), timestamp)
) WITH CLUSTERING ORDER BY (timestamp DESC);

-- 查询时指定日期
SELECT * FROM sensor_data 
WHERE date = '2024-01-01' AND sensor_id = ?;
```

**最佳实践：**
- ✅ 避免单调递增分区键
- ✅ 使用复合分区键分散数据
- ✅ 单分区大小 < 100MB
- ✅ 监控分区大小：`nodetool cfstats`

---

##### 问题 2：一致性级别选择

**现象：**
```cql
-- 场景：金融交易，需要强一致性
-- 问题：如何选择一致性级别？
```

**解决方案：**
```cql
-- 方案 1：强一致性（W + R > N）
-- 3 副本，QUORUM 读写
CONSISTENCY QUORUM;
INSERT INTO accounts (id, balance) VALUES (?, ?);
SELECT * FROM accounts WHERE id = ?;

-- W=2, R=2, N=3 → 2+2>3 ✅ 强一致性

-- 方案 2：最终一致性（高性能）
CONSISTENCY ONE;
INSERT INTO logs (id, message) VALUES (?, ?);

-- 方案 3：本地一致性（多数据中心）
CONSISTENCY LOCAL_QUORUM;
-- 只要求本地数据中心多数派确认

-- 方案 4：动态调整
-- 写入用 ONE（快速）
CONSISTENCY ONE;
INSERT INTO sensor_data ...;

-- 读取用 QUORUM（一致）
CONSISTENCY QUORUM;
SELECT * FROM sensor_data ...;
```

**最佳实践：**
- ✅ 金融交易：QUORUM/ALL
- ✅ 日志写入：ONE
- ✅ 监控数据：LOCAL_QUORUM
- ✅ 根据业务选择一致性级别

---

##### 问题 3：数据建模

**现象：**
```cql
-- 场景：用户订单查询
-- 需求：按用户查询订单、按订单 ID 查询
```

**解决方案：**
```cql
-- 方案 1：查询驱动设计（推荐）
-- 表 1：按用户查询
CREATE TABLE orders_by_user (
  user_id UUID,
  order_date DATE,
  order_id UUID,
  amount DECIMAL,
  PRIMARY KEY ((user_id), order_date, order_id)
) WITH CLUSTERING ORDER BY (order_date DESC);

-- 表 2：按订单 ID 查询
CREATE TABLE orders_by_id (
  order_id UUID PRIMARY KEY,
  user_id UUID,
  order_date DATE,
  amount DECIMAL
);

-- 写入时同时写入两张表
BEGIN BATCH
  INSERT INTO orders_by_user ...;
  INSERT INTO orders_by_id ...;
APPLY BATCH;

-- 方案 2：物化视图（自动同步）
CREATE MATERIALIZED VIEW orders_by_date AS
  SELECT * FROM orders
  WHERE order_date IS NOT NULL AND order_id IS NOT NULL
  PRIMARY KEY (order_date, order_id);
```

**最佳实践：**
- ✅ 一个查询一张表
- ✅ 避免二级索引（性能差）
- ✅ 使用物化视图自动同步
- ✅ 接受数据冗余

---

##### HBase

##### HBase 核心架构

**组件架构**

```
┌─────────────────────────────────────────────┐
│              ZooKeeper                      │
│  ├─ Master 选举                              │
│  ├─ RegionServer 注册                        │
│  └─ 元数据管理                                │
└────────────────┬────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────┐
│           HMaster（主节点）                   │
│  ├─ Region 分配                              │
│  ├─ 负载均衡                                  │
│  ├─ DDL 操作                                 │
│  └─ 故障恢复                                 │
└────────────────┬────────────────────────────┘
                 ↓
┌────────────────┴────────────────────────────┐
│         RegionServer（数据节点）              │
│  ┌──────────────────────────────────────┐   │
│  │  Region 1                            │   │
│  │  ├─ MemStore（内存）                   │  │
│  │  ├─ BlockCache（读缓存）               │   │
│  │  └─ HFile（磁盘，HDFS）                │   │
│  └──────────────────────────────────────┘   │
│  ┌──────────────────────────────────────┐   │
│  │  Region 2                            │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────┐
│              HDFS（存储层）                   │
│  ├─ HFile 存储                               │
│  ├─ WAL 日志                                 │
│  └─ 3 副本                                   │
└─────────────────────────────────────────────┘
```

**数据模型**

```
HBase 表结构（宽列存储）：

RowKey | Column Family: info | Column Family: data
       | name    | age      | value1  | value2
-------+----------+----------+---------+---------
row1   | Tom     | 25       | 100     | 200
row2   | Jane    | 30       | 150     | 250

特点：
- RowKey：主键，字典序排序
- Column Family：列族，物理存储单元
- Column：列，动态添加
- Timestamp：多版本，默认保留 3 个版本
```

##### HBase 写入流程

**写入路径**

```
写入流程：
1. 写入 WAL（Write-Ahead Log，HDFS）
2. 写入 MemStore（内存）
3. 返回成功
4. MemStore 满时刷盘到 HFile

┌─────────────────────────────────┐
│  1. 写入 WAL（HDFS）             │
│     ├─ 持久化保证                │
│     └─ 故障恢复                  │
└────────────┬────────────────────┘
             ↓
┌─────────────────────────────────┐
│  2. 写入 MemStore（内存）         │
│     ├─ 排序存储（RowKey 排序）     │
│     └─ 快速写入                   │
└────────────┬────────────────────┘
             ↓
┌─────────────────────────────────┐
│  3. MemStore 刷盘（128MB）       │
│     ├─ 生成 HFile（HDFS）        │
│     └─ 不可变文件                │
└─────────────────────────────────┘

写入性能：
- 顺序写 WAL（快）
- 内存写 MemStore（快）
- 批量刷盘（高效）
```

##### HBase Compaction

**合并策略**

```
Minor Compaction：
- 合并小 HFile
- 不删除过期数据
- 频繁执行

Major Compaction：
- 合并所有 HFile
- 删除过期数据和删除标记
- 定期执行（默认 7 天）

┌─────────────────────────────────┐
│  HFile 1 (10MB)                 │
│  HFile 2 (15MB)                 │
│  HFile 3 (20MB)                 │
└────────────┬────────────────────┘
             ↓ Minor Compaction
┌─────────────────────────────────┐
│  HFile 4 (45MB)                 │
└────────────┬────────────────────┘
             ↓ Major Compaction
┌─────────────────────────────────┐
│  HFile 5 (40MB，删除过期数据）     │
└─────────────────────────────────┘
```

##### HBase RowKey 设计

**RowKey 设计原则**

```
1. 避免热点（单调递增）
❌ 差：timestamp_userid
   - 所有写入集中在最新的 Region
   - 无法利用分布式写入

✅ 好：hash(userid)_timestamp
   - 数据分散到多个 Region
   - 充分利用分布式写入

2. 长度控制
- 建议 10-100 字节
- 过长影响性能（索引大）

3. 可读性
- 使用分隔符：userid_timestamp
- 便于调试和查询

4. 预分区
// 创建表时预分区
create 'mytable', 'cf', SPLITS => ['row10', 'row20', 'row30']
```

**RowKey 设计示例**

```java
// 时序数据 RowKey 设计
// ❌ 差：timestamp_sensorid
String rowKey = timestamp + "_" + sensorId;

// ✅ 好：sensorid_reverse_timestamp
String rowKey = sensorId + "_" + (Long.MAX_VALUE - timestamp);
// 优势：
// 1. 按 sensorId 分区（避免热点）
// 2. 反转时间戳（最新数据在前，范围查询友好）
```

##### HBase 性能优化

**读优化**

```java
// 1. 使用 Scan 缓存
Scan scan = new Scan();
scan.setCaching(1000);  // 每次 RPC 返回 1000 行
scan.setBatch(100);     // 每行返回 100 列

// 2. 使用 Bloom Filter
HColumnDescriptor cf = new HColumnDescriptor("cf");
cf.setBloomFilterType(BloomType.ROW);  // 行级 Bloom Filter

// 3. 使用 BlockCache
scan.setCacheBlocks(true);  // 缓存数据块
```

**写优化**

```java
// 1. 批量写入
List<Put> puts = new ArrayList<>();
for (int i = 0; i < 1000; i++) {
  Put put = new Put(Bytes.toBytes("row" + i));
  put.addColumn(Bytes.toBytes("cf"), Bytes.toBytes("col"), Bytes.toBytes("value"));
  puts.add(put);
}
table.put(puts);

// 2. 禁用 WAL（高风险）
put.setDurability(Durability.SKIP_WAL);  // 跳过 WAL，极快但可能丢数据

// 3. 预分区
admin.createTable(tableDescriptor, Bytes.toBytes("row00"), Bytes.toBytes("row99"), 10);
```

---

##### HBase 实战场景

###### 典型应用场景

| 场景 | 业务描述 | 数据量 | 并发量 | 关键指标 | 技术要点 |
|:---|:---|:---|:---|:---|:---|
| **大数据存储** | 用户行为、日志、爬虫数据 | PB 级 | 5万 TPS | 海量存储、随机读写 | RowKey 设计、预分区 |
| **实时查询** | 用户画像、实时推荐 | TB 级 | 1万 QPS | 低延迟、高并发 | BlockCache、Bloom Filter |
| **日志分析** | 应用日志、审计日志 | PB 级 | 10万 TPS | 高写入、范围查询 | 时间反转、Compaction |

---

###### 常见问题与解决方案

##### 问题 1：RowKey 设计

**现象：**
```java
// ❌ 差：单调递增 RowKey
String rowKey = timestamp + "_" + userId;
// 问题：所有写入集中在最新的 Region，热点问题
```

**解决方案：**
```java
// 方案 1：Hash 前缀（推荐）
String rowKey = MD5(userId).substring(0, 4) + "_" + timestamp + "_" + userId;
// 优点：数据分散到多个 Region
// 缺点：范围查询需要扫描所有 Region

// 方案 2：反转时间戳
String rowKey = userId + "_" + (Long.MAX_VALUE - timestamp);
// 优点：最新数据在前，范围查询友好
// 缺点：按用户分区，可能热点

// 方案 3：Salting（加盐）
int salt = userId.hashCode() % 10;  // 0-9
String rowKey = salt + "_" + userId + "_" + timestamp;
// 优点：数据均匀分布
// 缺点：查询时需要扫描 10 个 Region

// 方案 4：预分区
byte[][] splits = new byte[10][];
for (int i = 0; i < 10; i++) {
    splits[i] = Bytes.toBytes(String.format("%02d", i));
}
admin.createTable(tableDescriptor, splits);
```

**最佳实践：**
- ✅ 避免单调递增 RowKey
- ✅ 使用 Hash 前缀分散数据
- ✅ RowKey 长度 10-100 字节
- ✅ 预分区避免热点
- ✅ 监控 Region 负载

---

##### 问题 2：热点问题

**现象：**
```java
// 场景：明星用户数据访问量大
// 问题：单个 Region 负载过高
```

**解决方案：**
```java
// 方案 1：RowKey 加盐
String rowKey = (userId.hashCode() % 10) + "_" + userId;
// 数据分散到 10 个 Region

// 方案 2：多版本存储
Put put = new Put(Bytes.toBytes(rowKey));
put.addColumn(family, qualifier, timestamp, value);
// 保留多个版本，分散读压力

// 方案 3：BlockCache 优化
HColumnDescriptor cf = new HColumnDescriptor("cf");
cf.setBlockCacheEnabled(true);
cf.setCacheDataOnWrite(true);
// 热点数据缓存在内存

// 方案 4：读写分离
// 写入 HBase
// 热点数据同步到 Redis
redis.set(userId, data);
// 读取优先从 Redis
```

**最佳实践：**
- ✅ RowKey 加盐分散热点
- ✅ 启用 BlockCache
- ✅ 热点数据缓存到 Redis
- ✅ 监控 Region 热点：`hbase hbck`

**热点监控与诊断：**
```bash
# 1. 查看 Region 负载分布
hbase hbck -details

# 2. 监控 RegionServer 指标
# Web UI: http://regionserver:16030
# 关注指标：
# - requestsPerSecond（每秒请求数）
# - readRequestCount / writeRequestCount
# - storeFileCount（HFile 数量）

# 3. 查看热点 Region
echo "status 'detailed'" | hbase shell
# 输出：Region 的读写请求数、MemStore 大小

# 4. JMX 监控
curl http://regionserver:16030/jmx?qry=Hadoop:service=HBase,name=RegionServer,sub=Regions
```

**具体业务场景 RowKey 设计：**

```java
// 场景 1：订单系统（查询用户订单）
// 需求：按用户查询订单，按时间倒序
String rowKey = userId + "_" + (Long.MAX_VALUE - orderTime) + "_" + orderId;
// 查询：Scan startRow=userId_0, stopRow=userId_MAX

// 场景 2：IoT 时序数据（百万设备上报）
// 需求：高并发写入，按设备查询时间范围
int bucket = deviceId.hashCode() % 100;
String rowKey = String.format("%02d_%s_%d", bucket, deviceId, timestamp);
// 查询：并行扫描 100 个 bucket

// 场景 3：社交关系（明星用户热点）
// 需求：明星用户粉丝列表，避免热点
int shard = followerId.hashCode() % 10;
String rowKey = userId + "_" + shard + "_" + followerId;
// 存储：明星数据分散到 10 个 shard
// 查询：扫描 userId_0 到 userId_9

// 场景 4：日志存储（按日期查询）
// 需求：按日期范围查询，避免写入热点
String date = "20250109";
int hash = logId.hashCode() % 20;
String rowKey = date + "_" + String.format("%02d", hash) + "_" + logId;
// 写入：分散到 20 个 Region
// 查询：Scan startRow=20250109_00, stopRow=20250109_99
```

**Region 分裂与合并策略：**

```java
// 1. 预分区（建表时）
byte[][] splits = new byte[20][];
for (int i = 1; i < 20; i++) {
    splits[i-1] = Bytes.toBytes(String.format("%02d", i));
}
admin.createTable(tableDescriptor, splits);

// 2. 禁用自动分裂（大表场景）
HTableDescriptor desc = new HTableDescriptor(TableName.valueOf("mytable"));
desc.setMaxFileSize(Long.MAX_VALUE);  // 禁用自动分裂
admin.modifyTable(TableName.valueOf("mytable"), desc);

// 3. 手动分裂热点 Region
admin.split(TableName.valueOf("mytable"), Bytes.toBytes("rowkey_split_point"));

// 4. 合并小 Region
admin.mergeRegionsAsync(region1, region2, false);

// 5. Region 负载均衡
admin.balancer();  // 触发负载均衡
admin.setBalancerRunning(true, true);  // 启用自动均衡
```

**Region 管理最佳实践：**
```bash
# 查看 Region 分布
echo "status 'detailed'" | hbase shell

# 移动热点 Region 到其他 RegionServer
hbase org.apache.hadoop.hbase.util.RegionMover.rb unload <hostname>

# 查看 Region 分裂历史
hbase hbck -details | grep -i split

# 监控 Region 数量（建议单表 < 1000 个 Region）
echo "list" | hbase shell | wc -l
```

---

##### 问题 3：Compaction 优化

**现象：**
```
场景：大量小文件，读性能下降
原因：HFile 过多，查询需要扫描多个文件
```

**解决方案：**
```java
// 方案 1：手动触发 Major Compaction
admin.majorCompact(TableName.valueOf("mytable"));

// 方案 2：调整 Compaction 策略
HTableDescriptor desc = new HTableDescriptor(TableName.valueOf("mytable"));
desc.setConfiguration("hbase.hstore.compaction.min", "3");  // 最少 3 个文件才合并
desc.setConfiguration("hbase.hstore.compaction.max", "10"); // 最多合并 10 个文件

// 方案 3：定时 Compaction（避开高峰）
// 凌晨 2-4 点执行 Major Compaction
crontab -e
0 2 * * * hbase org.apache.hadoop.hbase.util.CompactionTool -major mytable

// 方案 4：禁用自动 Compaction（手动控制）
desc.setConfiguration("hbase.hstore.compaction.min", "999999");
```

**最佳实践：**
- ✅ 定期执行 Major Compaction
- ✅ 避开业务高峰期
- ✅ 监控 HFile 数量
- ✅ 调整 Compaction 阈值

---

##### 问题 4：性能调优

**现象：**
```java
// 查询慢，耗时 5 秒
Scan scan = new Scan();
scan.setStartRow(Bytes.toBytes("row000"));
scan.setStopRow(Bytes.toBytes("row999"));
ResultScanner scanner = table.getScanner(scan);
```

**解决方案：**
```java
// 方案 1：使用 Bloom Filter
HColumnDescriptor cf = new HColumnDescriptor("cf");
cf.setBloomFilterType(BloomType.ROW);  // 行级 Bloom Filter
// 减少不必要的磁盘 I/O

// 方案 2：设置 Scan 缓存
Scan scan = new Scan();
scan.setCaching(1000);  // 每次 RPC 返回 1000 行
scan.setBatch(100);     // 每行返回 100 列

// 方案 3：使用 Filter
Scan scan = new Scan();
scan.setFilter(new SingleColumnValueFilter(
    Bytes.toBytes("cf"),
    Bytes.toBytes("age"),
    CompareOp.GREATER,
    Bytes.toBytes(25)
));

// 方案 4：并行 Scan
ExecutorService executor = Executors.newFixedThreadPool(10);
List<Future<List<Result>>> futures = new ArrayList<>();

for (int i = 0; i < 10; i++) {
    final int partition = i;
    futures.add(executor.submit(() -> {
        Scan scan = new Scan();
        scan.setStartRow(Bytes.toBytes("row" + partition + "00"));
        scan.setStopRow(Bytes.toBytes("row" + partition + "99"));
        return scanResults(scan);
    }));
}
```

**最佳实践：**
- ✅ 启用 Bloom Filter
- ✅ 设置合理的 Scan 缓存
- ✅ 使用 Filter 减少数据传输
- ✅ 并行 Scan 提升性能
- ✅ 监控 RegionServer 负载

---

##### 问题 5：数据迁移

**现象：**
```
场景：从 MySQL 迁移到 HBase
挑战：数据格式转换、停机时间、数据一致性
```

**解决方案：**
```java
// 方案 1：使用 Sqoop 导入（批量迁移）
sqoop import \
  --connect jdbc:mysql://localhost:3306/mydb \
  --table orders \
  --hbase-table orders \
  --column-family cf \
  --hbase-row-key order_id \
  --hbase-create-table

// 方案 2：使用 BulkLoad（大数据量）
// 步骤 1：生成 HFile
Configuration conf = HBaseConfiguration.create();
Job job = Job.getInstance(conf);
job.setMapperClass(MyMapper.class);
job.setReducerClass(MyReducer.class);
HFileOutputFormat2.configureIncrementalLoad(job, table, regionLocator);

// 步骤 2：加载 HFile 到 HBase
LoadIncrementalHFiles loader = new LoadIncrementalHFiles(conf);
loader.doBulkLoad(new Path("/output/hfiles"), admin, table, regionLocator);

// 方案 3：实时同步（Canal + Kafka + HBase）
// MySQL Binlog → Canal → Kafka → Flink → HBase
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
DataStream<String> stream = env.addSource(new FlinkKafkaConsumer<>(...));
stream.map(new MapFunction<String, Put>() {
    @Override
    public Put map(String value) {
        JSONObject json = JSON.parseObject(value);
        Put put = new Put(Bytes.toBytes(json.getString("id")));
        put.addColumn(Bytes.toBytes("cf"), Bytes.toBytes("name"), 
                     Bytes.toBytes(json.getString("name")));
        return put;
    }
}).addSink(new HBaseSinkFunction());

// 方案 4：双写验证
// 1. 应用同时写入 MySQL 和 HBase
// 2. 对比数据一致性
// 3. 切换读流量到 HBase
// 4. 停止写入 MySQL
```

**最佳实践：**
- ✅ 大数据量使用 BulkLoad（比 Put 快 10 倍）
- ✅ 实时同步使用 CDC（Canal/Debezium）
- ✅ 迁移前预分区，避免热点
- ✅ 验证数据一致性（行数、抽样对比）
- ✅ 灰度切流，逐步迁移

---

##### 问题 6：备份与恢复

**现象：**
```
场景：误删数据、集群故障、数据恢复
需求：定期备份、快速恢复、跨集群复制
```

**解决方案：**
```bash
# 方案 1：Snapshot 快照（推荐）
# 创建快照
hbase snapshot create -n snapshot_20250109 -t mytable

# 查看快照
hbase snapshot list

# 恢复快照
hbase snapshot restore -s snapshot_20250109

# 导出快照到 HDFS/S3
hbase org.apache.hadoop.hbase.snapshot.ExportSnapshot \
  -snapshot snapshot_20250109 \
  -copy-to hdfs://backup-cluster/hbase/snapshots

# 方案 2：Export/Import（表级备份）
# 导出表
hbase org.apache.hadoop.hbase.mapreduce.Export \
  mytable /backup/mytable 1

# 导入表
hbase org.apache.hadoop.hbase.mapreduce.Import \
  mytable /backup/mytable

# 方案 3：CopyTable（跨集群复制）
hbase org.apache.hadoop.hbase.mapreduce.CopyTable \
  --peer.adr=backup-cluster:2181:/hbase \
  --new.name=mytable_backup \
  mytable

# 方案 4：Replication（实时备份）
# 主集群配置
add_peer '1', CLUSTER_KEY => 'backup-cluster:2181:/hbase'
enable_table_replication 'mytable'

# 查看复制状态
status 'replication'
```

**备份策略：**
```bash
# 1. 每日增量快照
0 2 * * * hbase snapshot create -n daily_$(date +\%Y\%m\%d) -t mytable

# 2. 每周全量备份
0 3 * * 0 hbase org.apache.hadoop.hbase.snapshot.ExportSnapshot \
  -snapshot weekly_$(date +\%Y\%m\%d) \
  -copy-to s3://backup-bucket/hbase/

# 3. 保留策略（保留 30 天）
hbase snapshot delete -n snapshot_20241210
```

**最佳实践：**
- ✅ 使用 Snapshot（快速、无锁、增量）
- ✅ 定期导出到 S3/HDFS（异地备份）
- ✅ 启用 Replication（实时容灾）
- ✅ 测试恢复流程（定期演练）
- ✅ 监控备份任务（失败告警）

---

##### 问题 7：集群扩容

**现象：**
```
场景：数据量增长，需要扩容 RegionServer
挑战：数据迁移、负载均衡、停机时间
```

**解决方案：**
```bash
# 方案 1：在线扩容（推荐）
# 步骤 1：添加新 RegionServer
# 启动新节点
hbase-daemon.sh start regionserver

# 步骤 2：触发负载均衡
hbase balancer

# 步骤 3：监控 Region 迁移
echo "status 'detailed'" | hbase shell

# 方案 2：手动迁移 Region
# 移动指定 Region
hbase org.apache.hadoop.hbase.util.RegionMover.rb \
  unload old-regionserver

# 方案 3：优雅下线旧节点
# 步骤 1：停止接收新 Region
hbase-daemon.sh stop regionserver

# 步骤 2：迁移 Region 到其他节点
hbase org.apache.hadoop.hbase.util.RegionMover.rb \
  unload old-regionserver new-regionserver

# 方案 4：调整 Region 分布策略
# 配置 hbase-site.xml
<property>
  <name>hbase.master.loadbalancer.class</name>
  <value>org.apache.hadoop.hbase.master.balancer.StochasticLoadBalancer</value>
</property>
```

**扩容检查清单：**
```bash
# 1. 检查集群健康
hbase hbck

# 2. 检查 HDFS 空间
hdfs dfsadmin -report

# 3. 检查 Region 分布
echo "status 'detailed'" | hbase shell

# 4. 检查负载均衡状态
hbase balancer_enabled

# 5. 监控迁移进度
watch -n 5 'echo "status" | hbase shell'
```

**最佳实践：**
- ✅ 扩容前备份数据
- ✅ 避开业务高峰期
- ✅ 逐步添加节点（每次 1-2 台）
- ✅ 监控 Region 迁移进度
- ✅ 验证数据完整性

---

##### 问题 8：Phoenix SQL 层集成

**现象：**
```
场景：HBase 只支持 Key-Value API，缺少 SQL 支持
需求：使用 SQL 查询 HBase，支持 JOIN、聚合
```

**解决方案：**
```sql
-- 方案 1：安装 Phoenix
-- 下载 Phoenix 并配置
wget https://phoenix.apache.org/download.html
cp phoenix-server.jar $HBASE_HOME/lib/

-- 启动 Phoenix
sqlline.py localhost:2181

-- 方案 2：创建 Phoenix 表（映射 HBase 表）
CREATE TABLE orders (
    order_id VARCHAR PRIMARY KEY,
    user_id VARCHAR,
    amount DECIMAL,
    create_time TIMESTAMP
) COLUMN_ENCODED_BYTES=0;

-- 方案 3：使用 SQL 查询
SELECT user_id, SUM(amount) as total
FROM orders
WHERE create_time > CURRENT_DATE() - 30
GROUP BY user_id
ORDER BY total DESC
LIMIT 10;

-- 方案 4：创建二级索引
CREATE INDEX idx_user_id ON orders(user_id);

-- 方案 5：使用视图（映射已有 HBase 表）
CREATE VIEW hbase_orders (
    pk VARCHAR PRIMARY KEY,
    "cf"."user_id" VARCHAR,
    "cf"."amount" DECIMAL
);
```

**Phoenix vs HBase API：**

| 维度 | HBase API | Phoenix SQL |
|:---|:---|:---|
| 查询方式 | Java API（Get/Scan） | SQL |
| 学习成本 | 高 | 低 |
| 复杂查询 | 困难（需要代码实现） | 简单（SQL） |
| 性能 | 最优 | 略低（SQL 解析开销） |
| 二级索引 | 不支持 | 支持 |
| JOIN | 不支持 | 支持 |
| 聚合 | 需要 Coprocessor | 原生支持 |

**最佳实践：**
- ✅ 简单查询用 HBase API（性能最优）
- ✅ 复杂查询用 Phoenix SQL（开发效率高）
- ✅ 创建二级索引加速查询
- ✅ 使用 Phoenix 连接池
- ✅ 监控 Phoenix Query Server 性能

---

###### 监控与运维

**关键监控指标：**

```bash
# 1. RegionServer 指标
# Web UI: http://regionserver:16030
- requestsPerSecond（每秒请求数）
- readRequestCount / writeRequestCount
- storeFileCount（HFile 数量）
- memStoreSize（MemStore 大小）
- blockCacheHitRatio（缓存命中率）

# 2. Master 指标
# Web UI: http://master:16010
- numRegionServers（RegionServer 数量）
- numDeadRegionServers（宕机节点）
- averageLoad（平均负载）
- ritCount（Region in Transition）

# 3. HDFS 指标
hdfs dfsadmin -report
- Capacity（总容量）
- DFS Used（已用空间）
- DFS Remaining（剩余空间）

# 4. ZooKeeper 指标
echo stat | nc localhost 2181
- Connections（连接数）
- Latency（延迟）
```

**告警规则：**
```yaml
# Prometheus 告警规则
groups:
  - name: hbase
    rules:
      - alert: RegionServerDown
        expr: hbase_regionserver_up == 0
        for: 1m
        annotations:
          summary: "RegionServer {{ $labels.instance }} is down"

      - alert: HighMemStoreSize
        expr: hbase_memstore_size_mb > 1024
        for: 5m
        annotations:
          summary: "MemStore size > 1GB on {{ $labels.instance }}"

      - alert: LowBlockCacheHitRatio
        expr: hbase_blockcache_hit_ratio < 0.8
        for: 10m
        annotations:
          summary: "BlockCache hit ratio < 80% on {{ $labels.instance }}"
```

**日常运维命令：**
```bash
# 1. 检查集群状态
hbase hbck
hbase hbck -details

# 2. 查看表信息
echo "describe 'mytable'" | hbase shell
echo "status 'detailed'" | hbase shell

# 3. 手动触发 Compaction
echo "major_compact 'mytable'" | hbase shell

# 4. 查看 Region 分布
hbase org.apache.hadoop.hbase.util.RegionSplitter \
  -D split.algorithm=HexStringSplit \
  -c 20 -f cf mytable

# 5. 修复元数据
hbase hbck -repair

# 6. 清理过期快照
hbase snapshot list | grep 2024 | xargs -I {} hbase snapshot delete -n {}
```

---

###### 故障排查手册

**问题 1：RegionServer 宕机**
```bash
# 症状
- Web UI 显示 Dead RegionServers
- 客户端报错：RetriesExhaustedException

# 排查步骤
1. 查看 RegionServer 日志
tail -f /var/log/hbase/hbase-regionserver.log

2. 检查 JVM 内存
jstat -gcutil <pid> 1000

3. 检查 HDFS 健康
hdfs dfsadmin -report

4. 检查 ZooKeeper 连接
echo stat | nc localhost 2181

# 解决方案
- 重启 RegionServer
- 调整 JVM 参数（增加堆内存）
- 修复 HDFS 损坏的块
```

**问题 2：Region 长时间 RIT（Region in Transition）**
```bash
# 症状
- Region 一直处于 OPENING/CLOSING 状态
- 表无法访问

# 排查步骤
echo "status 'detailed'" | hbase shell | grep RIT

# 解决方案
# 方案 1：手动 assign
echo "assign 'region_name'" | hbase shell

# 方案 2：强制关闭
hbase hbck -fixAssignments

# 方案 3：重启 Master
hbase-daemon.sh restart master
```

**问题 3：写入慢**
```bash
# 症状
- Put 操作耗时 > 100ms
- MemStore flush 频繁

# 排查步骤
1. 检查 WAL 写入延迟
grep "Slow sync" /var/log/hbase/hbase-regionserver.log

2. 检查 HDFS 写入性能
hdfs dfs -put test.file /tmp/

3. 检查 MemStore 大小
# Web UI: http://regionserver:16030

# 解决方案
- 增加 MemStore 大小
- 调整 WAL 刷盘策略
- 优化 HDFS 配置
```

**问题 4：读取慢**
```bash
# 症状
- Get/Scan 操作耗时 > 1s
- BlockCache 命中率低

# 排查步骤
1. 检查 BlockCache 命中率
# Web UI: blockCacheHitRatio

2. 检查 HFile 数量
echo "status 'detailed'" | hbase shell

3. 检查 Bloom Filter
echo "describe 'mytable'" | hbase shell

# 解决方案
- 启用 Bloom Filter
- 增加 BlockCache 大小
- 执行 Major Compaction
- 优化 RowKey 设计
```

---

## 搜索引擎数据库

### OpenSearch / Elasticsearch

#### OpenSearch

**AWS OpenSearch 没有 ZooKeeper，Control Plane 概念。**

![OpenSearch-HighLevel-Architecture](https://api.contentstack.io/v2/assets/578fc0aae4d30ee1302140e2/download?uid=blt3fa2156574584bc7)

![OpenSearch-Architecture](https://miro.medium.com/v2/resize:fit:4800/format:webp/1*b0vgJt_UjmzRTzOfwAR7wQ.png)

##### Elasticsearch 核心架构

**集群架构**

```
┌─────────────────────────────────────────────┐
│              客户端                          │
└────────────────┬────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────┐
│      Coordinating Node（协调节点）           │
│  ├─ 接收请求                                │
│  ├─ 路由查询                                │
│  ├─ 聚合结果                                │
│  └─ 返回响应                                │
└────────┬────────────────────┬───────────────┘
         ↓                    ↓
┌────────────────┐    ┌────────────────┐
│  Master Node   │    │  Master Node   │
│  （主节点）     │    │  （候选）       │
│  ├─ 集群管理    │    └────────────────┘
│  ├─ 索引管理    │
│  ├─ 分片分配    │
│  └─ 元数据管理  │
└────────────────┘
         ↓
┌────────┴────────────────────┴───────────────┐
│              Data Node（数据节点）            │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ Shard 0  │  │ Shard 1  │  │ Shard 2  │  │
│  │ Primary  │  │ Primary  │  │ Primary  │  │
│  └──────────┘  └──────────┘  └──────────┘  │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  │
│  │ Shard 0  │  │ Shard 1  │  │ Shard 2  │  │
│  │ Replica  │  │ Replica  │  │ Replica  │  │
│  └──────────┘  └──────────┘  └──────────┘  │
└─────────────────────────────────────────────┘
```

**节点角色**

| 角色 | 说明 | 配置 |
|:---|:---|:---|
| Master Node | 集群管理、索引创建/删除、分片分配 | node.master: true |
| Data Node | 存储数据、执行查询 | node.data: true |
| Coordinating Node | 路由请求、聚合结果 | node.master: false<br>node.data: false |
| Ingest Node | 数据预处理（Pipeline） | node.ingest: true |

---

##### 正向索引 vs 倒排索引对比

**传统数据库的正向索引（MySQL B-Tree）**

```
表数据：
CREATE TABLE articles (
    id INT PRIMARY KEY,
    title VARCHAR(200),
    content TEXT
);

INSERT INTO articles VALUES
(1, 'Learn MySQL', 'MySQL is a relational database'),
(2, 'Learn Redis', 'Redis is a key-value store'),
(3, 'Learn MySQL Advanced', 'MySQL supports transactions');

物理存储（articles.ibd）：
Page 1 (16KB):
┌─────────────────────────────────────────────────────┐
│ Row 1: id=1, title='Learn MySQL',                   │
│        content='MySQL is a relational database'     │
├─────────────────────────────────────────────────────┤
│ Row 2: id=2, title='Learn Redis',                   │
│        content='Redis is a key-value store'         │
├─────────────────────────────────────────────────────┤
│ Row 3: id=3, title='Learn MySQL Advanced',          │
│        content='MySQL supports transactions'        │
└─────────────────────────────────────────────────────┘

B-Tree 主键索引：
        [2]
       /   \
    [1]     [3]
     ↓       ↓       ↓
  Row1   Row2   Row3

查询1：根据ID查询（快）
SELECT * FROM articles WHERE id = 2;
→ B-Tree查找：O(log N)
→ 直接定位到Row 2

查询2：全文搜索（慢）
SELECT * FROM articles WHERE content LIKE '%MySQL%';
→ 全表扫描：O(N × M)
  N = 总行数
  M = 每行content的平均长度
→ 执行过程：
  1. 读取Row 1: 'MySQL is...' → 包含'MySQL' ✓
  2. 读取Row 2: 'Redis is...' → 不包含'MySQL' ✗
  3. 读取Row 3: 'MySQL supports...' → 包含'MySQL' ✓
→ 返回Row 1和Row 3

问题：
❌ 必须读取每一行的完整内容
❌ 必须对每行进行字符串匹配
❌ 无法利用索引加速
```

**Elasticsearch 的倒排索引（基于 Lucene）**

```
同样的数据在 Elasticsearch 中：

PUT /articles/_doc/1
{"id": 1, "title": "Learn MySQL", "content": "MySQL is a relational database"}

PUT /articles/_doc/2
{"id": 2, "title": "Learn Redis", "content": "Redis is a key-value store"}

PUT /articles/_doc/3
{"id": 3, "title": "Learn MySQL Advanced", "content": "MySQL supports transactions"}

分词和标准化：
Doc 1 content: "MySQL is a relational database"
→ 分词: [mysql, is, a, relational, database]
→ 去除停用词: [mysql, relational, database]

Doc 2 content: "Redis is a key-value store"
→ 分词: [redis, is, a, key, value, store]
→ 去除停用词: [redis, key, value, store]

Doc 3 content: "MySQL supports transactions"
→ 分词: [mysql, supports, transactions]

倒排索引结构：
Term Dictionary（词典）：
┌──────────────────────────────────────┐
│ Term: "database"                     │
│   → Postings List offset: 100        │
├──────────────────────────────────────┤
│ Term: "key"                          │
│   → Postings List offset: 200        │
├──────────────────────────────────────┤
│ Term: "mysql"                        │
│   → Postings List offset: 300        │
├──────────────────────────────────────┤
│ Term: "redis"                        │
│   → Postings List offset: 400        │
└──────────────────────────────────────┘

Postings List（倒排列表）：
offset=300 (Term: "mysql"):
┌─────────────────────────────────────┐
│ Doc Frequency: 2                    │
│ Doc IDs: [1, 3]                     │
│ Doc 1: TF=1, Position=[0]           │
│ Doc 3: TF=1, Position=[0]           │
└─────────────────────────────────────┘

查询：搜索包含"MySQL"的文档
GET /articles/_search
{
  "query": {"match": {"content": "MySQL"}}
}

执行过程：
1. 分词："MySQL" → "mysql"
2. 在FST中查找："mysql" → offset=300
3. 读取Postings List：Doc IDs [1, 3]
4. 计算评分（TF-IDF/BM25）
5. 返回结果：[Doc 1, Doc 3]

时间复杂度：O(log T + K)
  T = 词项总数
  K = 包含该词的文档数
```

**性能对比**

| 场景 | MySQL 正向索引 | Elasticsearch 倒排索引 |
|:---|:---|:---|
| **根据ID查询** | O(log N) ✅ | O(1) ✅ |
| **全文搜索单词** | O(N × M) ❌ | O(log T + K) ✅ |
| **多词搜索** | O(N × M) ❌ | O(log T + K1 + K2) ✅ |
| **存储开销** | 小（只存原始数据） | 大（原始数据+倒排索引） |

```
实际存储示例（1000万行，每行1KB）：

MySQL：
- 表数据：10GB
- 主键索引：200MB
- 全文搜索：需要扫描10GB

Elasticsearch：
- 原始文档：10GB
- 倒排索引：2GB
  - Term Dictionary: 500MB
  - Postings List: 1.5GB（压缩后）
  - FST（内存）: 50MB
- 全文搜索：只需查询FST + 读取相关Postings List
```

---

##### Lucene 和 Elasticsearch 的关系

**Lucene 是什么？**

```
Lucene = Apache Lucene
• Java编写的全文搜索引擎库（不是独立应用）
• 由Doug Cutting于1999年创建（Hadoop之父）
• 提供了倒排索引的核心实现
• 是一个库（Library），不是服务器

类比理解：
Lucene        ≈ MySQL的InnoDB存储引擎
               （底层索引和存储实现）

Elasticsearch ≈ MySQL Server
               （对外提供服务的完整系统）
```

**架构层次**

```
┌─────────────────────────────────────────────────┐
│           Elasticsearch                         │
│  (分布式搜索引擎，对外提供REST API)                 │
│                                                 │
│  功能：                                          │
│  - 集群管理（节点发现、分片分配）                    │
│  - RESTful API（HTTP接口）                       │
│  - 查询DSL（JSON查询语言）                        │
│  - 聚合分析（Aggregations）                       │
│  - 分布式协调（分片、副本）                         │
│  - 监控和管理（Kibana集成）                        │
└─────────────────────────────────────────────────┘
                    ↓ 调用
┌─────────────────────────────────────────────────┐
│              Apache Lucene                      │
│  (全文搜索引擎库，核心索引和搜索实现)                 │
│                                                 │
│  功能：                                          │
│  - 倒排索引（Inverted Index）                     │
│  - 分词器（Tokenizer/Analyzer）                   │
│  - 查询解析（Query Parser）                       │
│  - 评分算法（TF-IDF/BM25）                        │
│  - 索引文件格式（Segment Files）                  │
│  - FST（Finite State Transducer）               │
│  - Postings List（倒排列表）                      │
└─────────────────────────────────────────────────┘
                    ↓ 使用
┌─────────────────────────────────────────────────┐
│              文件系统                            │
│  (磁盘上的索引文件)                                │
│                                                 │
│  /data/elasticsearch/nodes/0/indices/           │
│    └─ my_index/0/index/                         │
│        ├─ _0.cfs (Compound File)                │
│        ├─ _0.tim (Term Dictionary)              │
│        ├─ _0.tip (Term Index - FST)             │
│        ├─ _0.doc (Postings List)                │
│        ├─ _0.pos (Position)                     │
│        └─ segments_1 (Segment Info)             │
└─────────────────────────────────────────────────┘
```

**Lucene Segment 完整文件列表**

```
一个 Segment 包含的所有文件：

_0.cfs   - Compound File（可选，将多个文件合并）
_0.cfe   - Compound File Entries（索引.cfs的内容）

或者分离的文件：
_0.tim   - Term Dictionary（词典）
_0.tip   - Term Index（FST，词典索引）
_0.doc   - Postings List（文档ID列表）
_0.pos   - Positions（词项位置）
_0.pay   - Payloads（额外信息）
_0.nvd   - Norms Data（标准化因子）
_0.nvm   - Norms Metadata
_0.dvd   - Doc Values Data（列式存储）
_0.dvm   - Doc Values Metadata
_0.fdx   - Field Index（字段索引）
_0.fdt   - Field Data（原始文档）
_0.fnm   - Field Names（字段名称）
_0.si    - Segment Info（段信息）

segments_N - Segment Metadata（所有段的元数据）

Lucene Index 结构：
Elasticsearch Index: "articles"
├─ Shard 0 → Lucene Index 0
├─ Shard 1 → Lucene Index 1
└─ Shard 2 → Lucene Index 2

Lucene Index 内部：
├─ Segment 0 (不可变)
│  ├─ _0.tim, _0.tip, _0.doc, _0.pos...
├─ Segment 1 (不可变)
│  ├─ _1.tim, _1.tip, _1.doc, _1.pos...
└─ Segment 2 (不可变)
   └─ _2.tim, _2.tip, _2.doc, _2.pos...
```

**Elasticsearch 在 Lucene 之上的增强**

| 功能       | Lucene      | Elasticsearch    |
| :------- | :---------- | :--------------- |
| **分布式**  | 🔴 单机库      | ✅ 分布式集群          |
| **API**  | Java API    | RESTful HTTP API |
| **查询语言** | Java代码      | JSON DSL         |
| **聚合分析** | 基础支持        | 强大的Aggregations  |
| **实时性**  | 需要手动refresh | 自动refresh（1秒）    |
| **高可用**  | 无           | 自动故障转移           |
| **监控**   | 无           | Kibana集成         |

---

##### 倒排索引原理

**正排索引 vs 倒排索引**

```
正排索引（MySQL）：
Doc ID → Content
1      → "Elasticsearch is fast"
2      → "Elasticsearch is scalable"
3      → "Fast search engine"

倒排索引（Elasticsearch）：
Term          → Doc IDs
elasticsearch → [1, 2]
fast          → [1, 3]
scalable      → [2]
search        → [3]
engine        → [3]
```

**倒排索引结构**

```
倒排索引 = 词典（Term Dictionary） + 倒排列表（Posting List）

┌─────────────────────────────────────────────┐
│           词典（Term Dictionary）            │
│  ├─ elasticsearch → Posting List 1          │
│  ├─ fast → Posting List 2                   │
│  └─ scalable → Posting List 3               │
└────────────────┬────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────┐
│        倒排列表（Posting List）               │
│  Posting List 1:                            │
│  ├─ Doc 1: position [0], frequency 1        │
│  └─ Doc 2: position [0], frequency 1        │
│                                             │
│  Posting List 2:                            │
│  ├─ Doc 1: position [2], frequency 1        │
│  └─ Doc 3: position [0], frequency 1        │
└─────────────────────────────────────────────┘
```

**分词（Analysis）**

```
原始文本：
"Elasticsearch is a fast search engine"

分词流程：
1. Character Filter（字符过滤）
   - HTML 标签去除
   - 特殊字符处理

2. Tokenizer（分词器）
   - Standard Tokenizer: ["Elasticsearch", "is", "a", "fast", "search", "engine"]

3. Token Filter（词元过滤）
   - Lowercase: ["elasticsearch", "is", "a", "fast", "search", "engine"]
   - Stop Words: ["elasticsearch", "fast", "search", "engine"]  // 去除 "is", "a"
   - Stemming: ["elasticsearch", "fast", "search", "engin"]     // 词干提取

最终索引：
elasticsearch → [Doc 1]
fast → [Doc 1]
search → [Doc 1]
engin → [Doc 1]
```

---

##### Doc Values（列存字典）原理

**为什么需要 Doc Values？**

```
倒排索引的局限性：

倒排索引：词 → 文档列表
elasticsearch → [Doc1, Doc3, Doc5]
fast → [Doc1, Doc2, Doc4]

✅ 擅长：全文搜索
查询：WHERE message LIKE '%elasticsearch%'
→ 直接查倒排索引，快速定位文档

❌ 不擅长：聚合和排序
查询：SELECT city, COUNT(*) GROUP BY city
→ 需要遍历所有词，然后统计，效率低

问题：
1. 倒排索引是"词 → 文档"的映射
2. 聚合需要"文档 → 字段值"的映射
3. 需要反向遍历，性能差
```

**Doc Values 的解决方案**

```
Elasticsearch 的双存储架构：

写入文档：
{
  "user_id": "user_123",
  "city": "Beijing",
  "age": 25
}

存储方式1：倒排索引（用于搜索）
┌─────────────────────────────────┐
│ 词 → 文档列表                     │
├─────────────────────────────────┤
│ "user_123" → [Doc1]              │
│ "beijing"  → [Doc1, Doc3, ...]   │
│ "25"       → [Doc1, Doc5, ...]   │
└─────────────────────────────────┘
用途：全文搜索、过滤
查询：WHERE city = 'Beijing'

存储方式2：Doc Values（用于聚合/排序）
┌─────────────────────────────────┐
│ 文档 → 字段值（列式存储）          │
├─────────────────────────────────┤
│ Doc1: user_id=user_123, city=Beijing, age=25 │
│ Doc2: user_id=user_456, city=Shanghai, age=30│
│ Doc3: user_id=user_789, city=Beijing, age=28 │
└─────────────────────────────────┘

实际存储（按列组织）：
user_id.docvalues: [user_123, user_456, user_789, ...]
city.docvalues:    [Beijing, Shanghai, Beijing, ...]
age.docvalues:     [25, 30, 28, ...]

用途：聚合、排序、统计
查询：GROUP BY city, AVG(age)
```

**字典编码（Dictionary Encoding）**

```
原始数据（重复多）：
city: [Beijing, Shanghai, Beijing, Shanghai, Beijing, Guangzhou, Beijing, ...]
存储：每个字符串 ~10 字节 × 1000万 = 100MB

字典编码后：
字典表：
0 → Beijing
1 → Shanghai
2 → Guangzhou

编码数据：
city.docvalues: [0, 1, 0, 1, 0, 2, 0, ...]
存储：每个数字 1 字节 × 1000万 = 10MB

压缩比：10倍！

查询：GROUP BY city
1. 扫描编码数据：[0, 1, 0, 1, 0, 2, 0, ...]
2. 统计：0出现4次，1出现2次，2出现1次
3. 解码：Beijing(4), Shanghai(2), Guangzhou(1)
```

---

**底层存储机制详解**

**1. 倒排索引的物理存储**

```
Lucene Segment 文件结构（Elasticsearch 底层）：

索引目录：
/data/nodes/0/indices/logs/0/index/
├─ segments_N                    ← Segment 元数据
├─ _0.cfs                        ← Compound File（复合文件）
│  ├─ _0.tim                     ← Term Dictionary（词典）
│  ├─ _0.tip                     ← Term Index（词典索引）
│  ├─ _0.doc                     ← Posting List（倒排列表）
│  ├─ _0.pos                     ← Position（词位置）
│  ├─ _0.pay                     ← Payload（额外数据）
│  └─ _0.dvd / _0.dvm            ← Doc Values 数据/元数据
└─ _0.si                         ← Segment Info

核心文件说明：

.tim（Term Dictionary）：
┌─────────────────────────────────┐
│ 词典（FST 有限状态机）             │
│ elasticsearch → Term ID: 1      │
│ fast → Term ID: 2               │
│ search → Term ID: 3             │
└─────────────────────────────────┘
存储：FST 压缩，内存占用小

.doc（Posting List）：
┌─────────────────────────────────┐
│ Term ID: 1 (elasticsearch)      │
│ ├─ Doc 1: freq=1, pos=[0]       │
│ ├─ Doc 3: freq=1, pos=[0]       │
│ └─ Doc 5: freq=2, pos=[0,10]    │
└─────────────────────────────────┘
存储：Delta 编码 + Roaring Bitmap 压缩

Delta 编码示例：
原始 Doc IDs: [1, 3, 5, 8, 12, 15]
Delta 编码:   [1, 2, 2, 3, 4, 3]  ← 存储差值
压缩比：~50%

Roaring Bitmap 压缩：
稀疏数据：[1, 100, 1000, 10000]
→ 使用 Bitmap：00000001...00000001...
密集数据：[1, 2, 3, 4, 5, 6, 7, 8]
→ 使用 Array：[1,2,3,4,5,6,7,8]
自动选择最优压缩方式
```

**2. Doc Values 的物理存储**

```
Doc Values 文件结构：

.dvd（Doc Values Data）：
┌─────────────────────────────────┐
│ 字段：city（keyword 类型）        │
│                                 │
│ 字典表（Dictionary）：             │
│ 0 → Beijing                     │
│ 1 → Shanghai                    │
│ 2 → Guangzhou                   │
│                                 │
│ 编码数据（Ordinals）：             │
│ Doc 0: 0 (Beijing)              │
│ Doc 1: 1 (Shanghai)             │
│ Doc 2: 0 (Beijing)              │
│ Doc 3: 1 (Shanghai)             │
│ Doc 4: 0 (Beijing)              │
│ Doc 5: 2 (Guangzhou)            │
│ ...                             │
└─────────────────────────────────┘

.dvm（Doc Values Metadata）：
┌─────────────────────────────────┐
│ 字段元数据                        │
│ ├─ 字段名：city                  │
│ ├─ 字段类型：SORTED              │
│ ├─ 字典大小：3                   │
│ ├─ 文档数量：1000万               │
│ └─ 数据偏移量：12345678           │
└─────────────────────────────────┘

存储格式（按字段类型）：

SORTED（keyword 类型）：
┌─────────────────────────────────┐
│ 字典：[Beijing, Shanghai, ...]   │
│ 序号：[0, 1, 0, 1, 0, 2, ...]    │
└─────────────────────────────────┘
压缩：字典编码 + 序号压缩

NUMERIC（long/integer 类型）：
┌─────────────────────────────────┐
│ 原始值：[25, 30, 28, 35, ...]    │
│ 压缩后：Delta + BitPacking       │
└─────────────────────────────────┘
压缩：Delta 编码 + Bit Packing

BINARY（text 类型，如果启用）：
┌─────────────────────────────────┐
│ 原始值：[value1, value2, ...]    │
│ 存储：直接存储字节数组             │
└─────────────────────────────────┘
压缩：LZ4 压缩
```

**3. 数值类型的 Bit Packing 压缩**

```
原始数据（age 字段）：
[25, 30, 28, 35, 22, 40, 33, 27, 29, 31]

步骤1：计算最大值
max = 40
需要的位数 = log2(40) = 6 bits

步骤2：Bit Packing
原始存储：每个数字 32 bits（4 bytes）
10 个数字 = 320 bits = 40 bytes

Bit Packing：每个数字 6 bits
10 个数字 = 60 bits = 8 bytes

压缩比：40 / 8 = 5倍！

实际存储（二进制）：
011001 011110 011100 100011 010110 101000 100001 011011 011101 011111
  25     30     28     35     22     40     33     27     29     31

读取时：
1. 读取 6 bits
2. 转换为整数
3. 返回值
```

**4. 倒排索引 vs Doc Values 的磁盘布局对比**

```
同一个字段（city）的两种存储：

倒排索引（.tim + .doc）：
┌─────────────────────────────────┐
│ Term Dictionary (.tim)          │
│ Beijing → Posting List Offset   │
│ Shanghai → Posting List Offset  │
│ Guangzhou → Posting List Offset │
└────────┬────────────────────────┘
         ↓
┌─────────────────────────────────┐
│ Posting List (.doc)             │
│ Beijing → [Doc0, Doc2, Doc4]    │
│ Shanghai → [Doc1, Doc3]         │
│ Guangzhou → [Doc5]              │
└─────────────────────────────────┘

存储特点：
✅ 按词组织（词 → 文档列表）
✅ 适合搜索：快速找到包含某个词的文档
❌ 不适合聚合：需要遍历所有词

Doc Values (.dvd)：
┌─────────────────────────────────┐
│ Dictionary                      │
│ 0 → Beijing                     │
│ 1 → Shanghai                    │
│ 2 → Guangzhou                   │
└────────┬────────────────────────┘
         ↓
┌─────────────────────────────────┐
│ Ordinals（按文档顺序）            │
│ Doc0 → 0 (Beijing)              │
│ Doc1 → 1 (Shanghai)             │
│ Doc2 → 0 (Beijing)              │
│ Doc3 → 1 (Shanghai)             │
│ Doc4 → 0 (Beijing)              │
│ Doc5 → 2 (Guangzhou)            │
└─────────────────────────────────┘

存储特点：
✅ 按文档组织（文档 → 字段值）
✅ 适合聚合：顺序扫描，统计每个值的出现次数
❌ 不适合搜索：需要扫描所有文档
```

**5. 查询时的底层执行流程**

```
查询1：全文搜索
GET /logs/_search
{
  "query": {
    "match": { "message": "database error" }
  }
}

底层执行：
1. 分词：database, error
2. 查询 Term Dictionary (.tim)：
   - database → Term ID: 123
   - error → Term ID: 456
3. 读取 Posting List (.doc)：
   - Term 123 → [Doc1, Doc5, Doc10]
   - Term 456 → [Doc1, Doc3, Doc10]
4. 求交集：[Doc1, Doc10]
5. 返回结果

磁盘读取：
- Term Dictionary：内存缓存（FST）
- Posting List：磁盘读取（顺序读）
- 总读取：~1KB

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

查询2：聚合统计
GET /logs/_search
{
  "aggs": {
    "by_city": {
      "terms": { "field": "city" }
    }
  }
}

底层执行：
1. 打开 Doc Values 文件 (.dvd)
2. 读取 Dictionary：
   - 0 → Beijing
   - 1 → Shanghai
   - 2 → Guangzhou
3. 顺序扫描 Ordinals：
   - [0, 1, 0, 1, 0, 2, 0, ...]
4. 统计每个值的出现次数：
   - 0 (Beijing): 4次
   - 1 (Shanghai): 2次
   - 2 (Guangzhou): 1次
5. 返回结果

磁盘读取：
- Dictionary：内存缓存
- Ordinals：磁盘顺序读（列式存储）
- 总读取：~10MB（1000万文档）

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

查询3：排序
GET /logs/_search
{
  "sort": [
    { "timestamp": "desc" }
  ]
}

底层执行：
1. 打开 timestamp 字段的 Doc Values (.dvd)
2. 读取所有文档的 timestamp 值：
   - [1704096000, 1704096060, 1704096120, ...]
3. 排序（使用外部排序算法）
4. 返回排序后的文档ID
5. 根据文档ID获取完整文档

磁盘读取：
- Doc Values：磁盘顺序读
- 文档数据：随机读
- 总读取：~100MB（1000万文档）
```

**6. 内存映射（mmap）机制**

```
Elasticsearch 使用 mmap 读取 Doc Values：

传统文件读取：
┌─────────────┐
│ 应用程序     │
│ read()      │
└──────┬──────┘
       ↓ 系统调用
┌─────────────┐
│ 内核缓冲区   │
│ 复制数据     │
└──────┬──────┘
       ↓ 复制
┌─────────────┐
│ 应用程序内存 │
└─────────────┘
问题：数据复制两次，效率低

mmap 文件映射：
┌─────────────┐
│ 应用程序     │
│ 直接访问     │
└──────┬──────┘
       ↓ 内存映射
┌─────────────┐
│ 页缓存       │
│ （内核管理）  │
└──────┬──────┘
       ↓ 按需加载
┌─────────────┐
│ 磁盘文件     │
└─────────────┘
优势：
✅ 零拷贝（直接访问页缓存）
✅ 操作系统自动管理缓存
✅ 多进程共享内存

Doc Values 使用 mmap：
1. 打开 .dvd 文件
2. mmap 映射到虚拟内存
3. 访问数据时，操作系统自动加载页
4. 热数据留在页缓存中
5. 冷数据自动换出

内存占用：
- 虚拟内存：文件大小（如 10GB）
- 物理内存：实际访问的页（如 2GB）
- 操作系统自动管理
```

**7. 为什么 Doc Values 比倒排索引占用更多内存？**

```
对比：1000万文档，city 字段（3个唯一值）

倒排索引：
Term Dictionary:
  Beijing → Posting List (3MB)
  Shanghai → Posting List (2MB)
  Guangzhou → Posting List (1MB)
总大小：~6MB

内存占用：
- Term Dictionary：FST 压缩，~100KB
- Posting List：按需加载，~6MB
- 总内存：~6MB

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Doc Values:
Dictionary:
  0 → Beijing
  1 → Shanghai
  2 → Guangzhou
Ordinals:
  [0,1,0,1,0,2,0,1,0,1,...] × 1000万
总大小：~10MB（字典编码后）

内存占用：
- Dictionary：~100 bytes
- Ordinals：mmap 映射，~10MB
- 总内存：~10MB

为什么更大？
- 倒排索引：只存储"词 → 文档列表"
- Doc Values：存储"每个文档的字段值"
- 文档数量 >> 唯一词数量
- 所以 Doc Values 更大
```

**8. 总结：倒排索引 vs Doc Values 底层对比**

| 维度 | 倒排索引 | Doc Values |
|:---|:---|:---|
| **文件格式** | .tim + .doc | .dvd + .dvm |
| **数据结构** | FST + Posting List | Dictionary + Ordinals |
| **压缩算法** | Delta + Roaring Bitmap | 字典编码 + Bit Packing |
| **内存管理** | 部分缓存 | mmap 全映射 |
| **磁盘读取** | 随机读（按词） | 顺序读（按文档） |
| **压缩比** | 5-10x | 2-5x |
| **内存占用** | 小（只存词典） | 大（存所有值） |

---

**使用场景对比**

```
场景1：全文搜索
GET /logs/_search
{
  "query": {
    "match": { "message": "database error" }
  }
}
→ 使用倒排索引 ✅
→ 快速定位包含"database"和"error"的文档

场景2：精确过滤
GET /logs/_search
{
  "query": {
    "term": { "level": "ERROR" }
  }
}
→ 使用倒排索引 ✅
→ 查找 level 字段值为"ERROR"的文档

场景3：聚合统计
GET /logs/_search
{
  "aggs": {
    "by_level": {
      "terms": { "field": "level" }
    }
  }
}
→ 使用 Doc Values ✅
→ 统计每个 level 的文档数量

场景4：排序
GET /logs/_search
{
  "sort": [
    { "timestamp": "desc" }
  ]
}
→ 使用 Doc Values ✅
→ 按 timestamp 字段排序

场景5：脚本计算
GET /logs/_search
{
  "script_fields": {
    "age_in_days": {
      "script": "doc['timestamp'].value"
    }
  }
}
→ 使用 Doc Values ✅
→ 在脚本中访问字段值
```

**字段类型与存储**

| 字段类型             | 倒排索引 | Doc Values | 说明           |
| :--------------- | :--- | :--------- | :----------- |
| **text**         | ✅ 有  | 🔴 无（默认）   | 全文搜索字段，不支持聚合 |
| **keyword**      | ✅ 有  | ✅ 有        | 精确匹配字段，支持聚合  |
| **long/integer** | ✅ 有  | ✅ 有        | 数值字段，支持聚合    |
| **date**         | ✅ 有  | ✅ 有        | 日期字段，支持聚合    |
| **boolean**      | ✅ 有  | ✅ 有        | 布尔字段，支持聚合    |
| **ip**           | ✅ 有  | ✅ 有        | IP地址字段，支持聚合  |
| **geo_point**    | ✅ 有  | ✅ 有        | 地理位置字段，支持聚合  |

**禁用 Doc Values（优化存储）**

```json
// 场景：只需要搜索，不需要聚合/排序
PUT /logs
{
  "mappings": {
    "properties": {
      "message": {
        "type": "text"  // text 类型默认无 Doc Values
      },
      "user_id": {
        "type": "keyword",
        "doc_values": false  // ← 禁用 Doc Values
      },
      "timestamp": {
        "type": "date"  // 保留 Doc Values（需要排序）
      }
    }
  }
}

// 优势：
// ✅ 减少磁盘占用（约30-50%）
// ✅ 减少索引时间

// 劣势：
// ❌ 无法对 user_id 进行聚合
// ❌ 无法对 user_id 进行排序
// ❌ 无法在脚本中访问 user_id
```

**Elasticsearch vs ClickHouse 列存对比**

| 维度 | Elasticsearch Doc Values | ClickHouse 列存 |
|:---|:---|:---|
| **存储架构** | 倒排索引 + 列存（双份） | 只有列存（单份） |
| **主要用途** | 辅助存储（聚合/排序） | 主存储（所有查询） |
| **搜索能力** | 依赖倒排索引（极快） | 依赖稀疏索引（较慢） |
| **聚合能力** | 快（列存字典） | 极快（列存+向量化） |
| **存储开销** | 大（双份存储） | 小（单份存储） |
| **压缩比** | 中等（2-5x） | 极高（5-20x） |
| **适用场景** | 全文搜索 + 聚合 | 分析查询 |

**性能对比示例**

```
数据：1亿条日志，10个字段

查询1：全文搜索
SELECT * FROM logs WHERE message LIKE '%error%'

Elasticsearch: 100ms ✅（倒排索引）
ClickHouse:    5秒 ⚠️（列扫描）

查询2：聚合统计
SELECT level, COUNT(*) FROM logs GROUP BY level

Elasticsearch: 500ms ✅（Doc Values）
ClickHouse:    50ms ✅✅（列存+向量化）

查询3：复杂聚合
SELECT city, AVG(response_time), MAX(response_time) 
FROM logs 
GROUP BY city

Elasticsearch: 2秒 ✅（Doc Values）
ClickHouse:    200ms ✅✅（列存+向量化+并行）

结论：
- 需要全文搜索 → Elasticsearch
- 需要分析查询 → ClickHouse
- 两者都要 → Elasticsearch（搜索）+ ClickHouse（分析）
```

---

##### Mapping 爆炸问题详解

**什么是 Mapping？**

```
Mapping = Elasticsearch 的字段结构定义（类似 MySQL 的表结构）

MySQL:
CREATE TABLE users (
  id INT,           ← 字段定义
  name VARCHAR(100), ← 字段定义
  age INT           ← 字段定义
);

Elasticsearch:
PUT /users
{
  "mappings": {
    "properties": {
      "id": { "type": "integer" },      ← 字段定义
      "name": { "type": "text" },       ← 字段定义
      "age": { "type": "integer" }      ← 字段定义
    }
  }
}
```

**Mapping 爆炸 = 字段数量失控**

```
核心问题：Elasticsearch 默认会自动为新字段创建 Mapping

第1条日志：
{
  "timestamp": "2024-01-01T10:00:00",
  "level": "INFO",
  "message": "User login"
}
→ 自动创建 3 个字段的 Mapping ✅

第2条日志（包含动态字段）：
{
  "timestamp": "2024-01-01T10:01:00",
  "level": "ERROR",
  "message": "Database error",
  "error_details": {
    "error_code_1001": "Connection timeout",  ← 动态 key！
    "error_code_1002": "Query failed"         ← 动态 key！
  }
}
→ 自动创建 2 个新字段的 Mapping
→ 总字段数：5 个 ✅

第3条日志（更多动态字段）：
{
  "timestamp": "2024-01-01T10:02:00",
  "level": "ERROR",
  "message": "Database error",
  "error_details": {
    "error_code_1003": "Deadlock",      ← 又一个新 key！
    "error_code_1004": "Disk full"      ← 又一个新 key！
  }
}
→ 自动创建 2 个新字段的 Mapping
→ 总字段数：7 个 ✅

继续写入 1000 条日志，每条都有不同的 error_code...
→ 总字段数：1000+ 个 ❌

继续写入 10000 条日志...
→ 总字段数：10000+ 个 ❌❌❌
→ 集群 OOM（内存溢出）
→ 集群崩溃！
```

**为什么会导致 OOM？**

```
每个字段的 Mapping 需要存储：
- 字段名
- 字段类型
- 倒排索引结构
- Doc Values 结构
- 字段统计信息

10000 个字段 × 每个字段 1MB = 10GB 内存

而且：
- Mapping 存储在每个节点的内存中
- 无法释放（除非删除索引）
- 查询时需要加载所有字段的 Mapping
```

**真实案例**

```json
// 案例1：应用日志（最常见）
{
  "timestamp": "2024-01-01T10:00:00",
  "app": "web-server",
  "custom_fields": {
    "user_123_login_time": "10:00:00",      // ← 每个用户一个字段
    "user_456_login_time": "10:01:00",      // ← 每个用户一个字段
    "user_789_login_time": "10:02:00",      // ← 每个用户一个字段
    // ... 100万个用户 = 100万个字段！
  }
}

// 案例2：监控指标
{
  "timestamp": "2024-01-01T10:00:00",
  "metrics": {
    "server_1_cpu": 80,                     // ← 每个服务器一个字段
    "server_2_cpu": 70,                     // ← 每个服务器一个字段
    "server_3_cpu": 90,                     // ← 每个服务器一个字段
    // ... 10000个服务器 = 10000个字段！
  }
}
```

**解决方案**

```json
// 方案1：禁用动态 Mapping
PUT /logs
{
  "mappings": {
    "dynamic": "strict",  // ← 禁止自动创建字段
    "properties": {
      "timestamp": { "type": "date" },
      "level": { "type": "keyword" },
      "message": { "type": "text" }
    }
  }
}

// 写入包含未定义字段的文档会报错
POST /logs/_doc
{
  "timestamp": "2024-01-01T10:00:00",
  "level": "INFO",
  "message": "test",
  "unknown_field": "value"  // ← 报错！
}

// 方案2：使用 flattened 类型
PUT /logs
{
  "mappings": {
    "properties": {
      "timestamp": { "type": "date" },
      "level": { "type": "keyword" },
      "message": { "type": "text" },
      "error_details": { 
        "type": "flattened"  // ← 关键！整个对象作为一个字段
      }
    }
  }
}

// 写入任意动态字段
POST /logs/_doc
{
  "timestamp": "2024-01-01T10:00:00",
  "level": "ERROR",
  "message": "Database error",
  "error_details": {
    "error_code_1001": "Connection timeout",
    "error_code_1002": "Query failed",
    "error_code_1003": "Deadlock",
    // ... 任意多个字段
  }
}

// Mapping 不会爆炸！
// error_details 始终只占用 1 个字段
// 无论里面有多少个 key

// 方案3：重新设计数据结构
// ❌ 错误设计（会导致 Mapping 爆炸）
{
  "timestamp": "2024-01-01T10:00:00",
  "metrics": {
    "server_1_cpu": 80,
    "server_2_cpu": 70,
    "server_3_cpu": 90
  }
}

// ✅ 正确设计（固定字段）
{
  "timestamp": "2024-01-01T10:00:00",
  "server_id": "server_1",  // ← 固定字段
  "metric_name": "cpu",     // ← 固定字段
  "metric_value": 80        // ← 固定字段
}

// 多个服务器 = 多个文档，而不是多个字段
```

---

##### Elasticsearch 分片和副本

**分片策略**

```
索引 = 多个分片（Shard）

┌─────────────────────────────────────────────┐
│              Index: products                │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Shard 0  │  │ Shard 1  │  │ Shard 2  │   │
│  │ Primary  │  │ Primary  │  │ Primary  │   │
│  │ Doc 0,3,6│  │ Doc 1,4,7│  │ Doc 2,5,8│   │
│  └──────────┘  └──────────┘  └──────────┘   │
│       ↓              ↓              ↓       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Shard 0  │  │ Shard 1  │  │ Shard 2  │   │
│  │ Replica  │  │ Replica  │  │ Replica  │   │
│  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────┘

文档路由：
shard_num = hash(document_id) % number_of_primary_shards

特点：
- Primary Shard 数量创建后不可变
- Replica Shard 数量可动态调整
```

**副本机制**

```
写入流程：
1. 客户端 → Coordinating Node
2. Coordinating Node → Primary Shard
3. Primary Shard 写入成功
4. Primary Shard → Replica Shard（同步复制）
5. Replica Shard 写入成功
6. 返回客户端

读取流程：
1. 客户端 → Coordinating Node
2. Coordinating Node 选择 Shard（Primary 或 Replica，轮询负载均衡）
3. Shard 返回结果
4. Coordinating Node 聚合结果
5. 返回客户端
```

##### Elasticsearch 查询流程

**查询阶段（Query Phase）**

```
查询：GET /products/_search?q=laptop

步骤 1：Query Phase
┌─────────────────────────────────────────────┐
│      Coordinating Node                      │
│  1. 接收查询请求                              │
│  2. 广播到所有 Shard（Primary 或 Replica）     │
└────────┬────────────────────┬───────────────┘
         ↓                    ↓
┌────────────────┐    ┌────────────────┐
│  Shard 0       │    │  Shard 1       │
│  1. 执行查询    │    │  1. 执行查询     │
│  2. 返回 Doc ID │    │ 2. 返回 Doc ID │
│     + Score    │    │     + Score    │
└────────────────┘    └────────────────┘

步骤 2：Fetch Phase
┌─────────────────────────────────────────────┐
│      Coordinating Node                      │
│  1. 合并所有 Shard 的结果                     │
│  2. 排序（按 Score）                          │
│  3. 选择 Top N 的 Doc ID                     │
│  4. 向相关 Shard 请求完整文档                  │
└────────┬────────────────────┬───────────────┘
         ↓                    ↓
┌────────────────┐    ┌────────────────┐
│  Shard 0       │    │  Shard 1       │
│  返回完整文档    │    │  返回完整文档    │
└────────────────┘    └────────────────┘
         ↓                    ↓
┌─────────────────────────────────────────────┐
│      Coordinating Node                      │
│  返回最终结果给客户端                          │
└─────────────────────────────────────────────┘
```

**评分机制（TF-IDF）**

```
TF-IDF = TF（词频） × IDF（逆文档频率）

TF（Term Frequency）：
- 词在文档中出现的频率
- TF = 词在文档中出现次数 / 文档总词数

IDF（Inverse Document Frequency）：
- 词的稀有程度
- IDF = log(文档总数 / 包含该词的文档数)

示例：
查询："elasticsearch fast"

Doc 1: "Elasticsearch is fast"
- elasticsearch: TF=1/3, IDF=log(3/2)=0.18 → Score=0.06
- fast: TF=1/3, IDF=log(3/2)=0.18 → Score=0.06
- Total Score = 0.12

Doc 2: "Fast search engine"
- fast: TF=1/3, IDF=log(3/2)=0.18 → Score=0.06
- Total Score = 0.06

结果：Doc 1 排名更高
```

##### Elasticsearch 聚合查询

**聚合类型**

| 聚合类型 | 说明 | 示例 |
|:---|:---|:---|
| Metric Aggregation | 指标聚合（求和、平均、最大、最小） | avg, sum, max, min |
| Bucket Aggregation | 桶聚合（分组） | terms, range, date_histogram |
| Pipeline Aggregation | 管道聚合（基于其他聚合结果） | derivative, cumulative_sum |

**聚合示例**

```json
// 按类别统计商品数量和平均价格
GET /products/_search
{
  "size": 0,
  "aggs": {
    "category_stats": {
      "terms": {
        "field": "category.keyword",
        "size": 10
      },
      "aggs": {
        "avg_price": {
          "avg": {
            "field": "price"
          }
        }
      }
    }
  }
}

// 结果：
{
  "aggregations": {
    "category_stats": {
      "buckets": [
        {
          "key": "Electronics",
          "doc_count": 1000,
          "avg_price": { "value": 500.0 }
        },
        {
          "key": "Books",
          "doc_count": 800,
          "avg_price": { "value": 25.0 }
        }
      ]
    }
  }
}
```

##### Elasticsearch 索引管理

**索引生命周期（ILM）**

```
Hot → Warm → Cold → Delete

┌─────────────────────────────────────────────┐
│  Hot Phase（热阶段）                          │
│  ├─ 高频写入和查询                            │
│  ├─ SSD 存储                                │
│  └─ 保留 7 天                                │
└────────────────┬────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────┐
│  Warm Phase（温阶段）                         │
│  ├─ 只读，低频查询                             │
│  ├─ 合并 Segment                             │
│  ├─ 迁移到 HDD                               │
│  └─ 保留 30 天                               │
└────────────────┬────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────┐
│  Cold Phase（冷阶段）                         │
│  ├─ 极少查询                                 │
│  ├─ 快照备份                                 │
│  └─ 保留 90 天                               │
└────────────────┬────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────┐
│  Delete Phase（删除阶段）                     │
│  └─ 删除索引                                 │
└─────────────────────────────────────────────┘
```

**索引模板**

```json
// 创建索引模板
PUT _index_template/logs_template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1
    },
    "mappings": {
      "properties": {
        "timestamp": { "type": "date" },
        "message": { "type": "text" },
        "level": { "type": "keyword" }
      }
    }
  }
}

// 创建索引时自动应用模板
PUT logs-2024-01-01
// 自动应用 logs_template 配置
```

##### Elasticsearch 性能优化

**写入优化**

```json
// 1. 批量写入（Bulk API）
POST _bulk
{ "index": { "_index": "products", "_id": "1" } }
{ "name": "Laptop", "price": 1000 }
{ "index": { "_index": "products", "_id": "2" } }
{ "name": "Mouse", "price": 20 }

// 2. 调整 Refresh Interval
PUT /products/_settings
{
  "index": {
    "refresh_interval": "30s"  // 默认 1s，增大可提升写入性能
  }
}

// 3. 禁用副本（写入时）
PUT /products/_settings
{
  "index": {
    "number_of_replicas": 0  // 写入完成后再设置为 1
  }
}
```

**查询优化**

```json
// 1. 使用 Filter Context（不计算评分，可缓存）
GET /products/_search
{
  "query": {
    "bool": {
      "must": [
        { "match": { "name": "laptop" } }  // Query Context（计算评分）
      ],
      "filter": [
        { "range": { "price": { "gte": 500 } } }  // Filter Context（不计算评分）
      ]
    }
  }
}

// 2. 使用 _source 过滤
GET /products/_search
{
  "query": { "match_all": {} },
  "_source": ["name", "price"]  // 只返回需要的字段
}

// 3. 使用分页（避免深度分页）
// ❌ 差：深度分页
GET /products/_search?from=10000&size=10

// ✅ 好：Scroll API
POST /products/_search?scroll=1m
{
  "size": 100,
  "query": { "match_all": {} }
}

// ✅ 好：Search After
GET /products/_search
{
  "size": 10,
  "query": { "match_all": {} },
  "search_after": [1000, "doc_id"],
  "sort": [
    { "price": "asc" },
    { "_id": "asc" }
  ]
}
```

**集群优化**

```yaml
# elasticsearch.yml

# 1. JVM 堆内存（不超过 32GB）
-Xms16g
-Xmx16g

# 2. 分片数量控制
# 单个分片大小：20-40GB
# 单节点分片数：< 20 个/GB 堆内存

# 3. 副本数量
# 生产环境：至少 1 个副本
# 读多写少：增加副本数

# 4. 禁用 Swap
bootstrap.memory_lock: true

# 5. 文件描述符
# ulimit -n 65535
```

---

##### Elasticsearch 实战场景

##### 典型应用场景

| 场景 | 业务描述 | 数据量 | 并发量 | 关键指标 | 技术要点 |
|:---|:---|:---|:---|:---|:---|
| **日志分析** | 应用日志、访问日志、错误日志 | TB-PB 级 | 5万 TPS | 全文搜索、聚合分析 | ILM、索引模板 |
| **全文搜索** | 商品搜索、文档搜索 | 亿级 | 1万 QPS | 相关性、高亮 | 分词器、评分 |
| **实时监控** | 系统指标、业务指标 | TB 级 | 2万 TPS | 实时聚合、告警 | Rollup、聚合 |
| **安全分析** | 安全日志、审计日志 | PB 级 | 10万 TPS | 异常检测、关联分析 | Machine Learning |

---

##### 常见问题与解决方案

##### 问题 1：索引设计

**现象：**
```json
// 场景：日志索引，每天 1TB 数据
// 问题：单个索引过大，查询慢
```

**解决方案：**
```json
// 方案 1：按时间分索引（推荐）
PUT logs-2024-01-01
PUT logs-2024-01-02
PUT logs-2024-01-03

// 使用索引模板自动创建
PUT _index_template/logs_template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "refresh_interval": "30s"
    },
    "mappings": {
      "properties": {
        "timestamp": { "type": "date" },
        "level": { "type": "keyword" },
        "message": { "type": "text" }
      }
    }
  }
}

// 方案 2：使用别名查询
POST _aliases
{
  "actions": [
    { "add": { "index": "logs-2024-01-*", "alias": "logs-current-month" } }
  ]
}

// 查询别名
GET logs-current-month/_search

// 方案 3：ILM 自动管理
PUT _ilm/policy/logs_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "50GB",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": { "number_of_shards": 1 },
          "forcemerge": { "max_num_segments": 1 }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "freeze": {}
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```

**最佳实践：**
- ✅ 按时间分索引（日/周/月）
- ✅ 使用索引模板
- ✅ 启用 ILM 自动管理
- ✅ 单个分片大小 20-40GB
- ✅ 使用别名简化查询

---

##### 问题 2：深度分页

**现象：**
```json
// 深度分页，耗时 30 秒
GET logs/_search
{
  "from": 10000,
  "size": 10,
  "query": { "match_all": {} }
}
// 问题：需要加载前 10010 条数据，内存占用大
```

**解决方案：**
```json
// 方案 1：Scroll API（推荐 - 导出数据）
POST logs/_search?scroll=1m
{
  "size": 1000,
  "query": { "match_all": {} }
}

// 继续滚动
POST _search/scroll
{
  "scroll": "1m",
  "scroll_id": "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAAD4WYm9laVYtZndUQlNsdDcwakFMNjU1QQ=="
}

// 方案 2：Search After（推荐 - 实时分页）
GET logs/_search
{
  "size": 10,
  "query": { "match_all": {} },
  "sort": [
    { "timestamp": "desc" },
    { "_id": "asc" }
  ]
}

// 下一页
GET logs/_search
{
  "size": 10,
  "query": { "match_all": {} },
  "search_after": [1609459200000, "doc_id_123"],
  "sort": [
    { "timestamp": "desc" },
    { "_id": "asc" }
  ]
}

// 方案 3：限制最大分页深度
PUT logs/_settings
{
  "index.max_result_window": 10000
}
```

**最佳实践：**
- ✅ 导出数据：Scroll API
- ✅ 实时分页：Search After
- ✅ 限制最大分页深度
- ✅ 前端限制页数（如 100 页）

---

##### 问题 3：聚合查询优化

**现象：**
```json
// 聚合查询慢，耗时 10 秒
GET logs/_search
{
  "size": 0,
  "aggs": {
    "by_level": {
      "terms": { "field": "level" },
      "aggs": {
        "by_host": {
          "terms": { "field": "host" }
        }
      }
    }
  }
}
```

**解决方案：**
```json
// 方案 1：使用 doc_values
PUT logs/_mapping
{
  "properties": {
    "level": {
      "type": "keyword",
      "doc_values": true  // 默认开启
    }
  }
}

// 方案 2：限制聚合桶数量
GET logs/_search
{
  "size": 0,
  "aggs": {
    "by_level": {
      "terms": {
        "field": "level",
        "size": 10  // 只返回 Top 10
      }
    }
  }
}

// 方案 3：使用 Composite Aggregation（分页聚合）
GET logs/_search
{
  "size": 0,
  "aggs": {
    "my_buckets": {
      "composite": {
        "size": 100,
        "sources": [
          { "level": { "terms": { "field": "level" } } },
          { "host": { "terms": { "field": "host" } } }
        ]
      }
    }
  }
}

// 方案 4：预聚合（Rollup）
PUT _rollup/job/logs_rollup
{
  "index_pattern": "logs-*",
  "rollup_index": "logs_rollup",
  "cron": "0 */1 * * * ?",
  "page_size": 1000,
  "groups": {
    "date_histogram": {
      "field": "timestamp",
      "interval": "1h"
    },
    "terms": {
      "fields": ["level", "host"]
    }
  },
  "metrics": [
    {
      "field": "response_time",
      "metrics": ["avg", "max", "min"]
    }
  ]
}
```

**最佳实践：**
- ✅ 使用 doc_values
- ✅ 限制聚合桶数量
- ✅ 大数据量使用 Composite Aggregation
- ✅ 预聚合热点数据（Rollup）
- ✅ 避免嵌套过深的聚合

---

##### 问题 4：集群容量规划

**现象：**
```
场景：日志系统，每天 1TB 数据，保留 90 天
问题：需要多少节点？多少分片？
```

**解决方案：**
```
// 容量计算
原始数据：1TB/天 × 90 天 = 90TB
副本数：1（1 主 + 1 副本）
总存储：90TB × 2 = 180TB
压缩率：50%（Elasticsearch 压缩）
实际存储：180TB × 0.5 = 90TB

// 节点规划
单节点存储：2TB（SSD）
节点数：90TB / 2TB = 45 个节点

// 分片规划
单个分片大小：20-40GB（推荐 30GB）
总分片数：90TB / 30GB = 3000 个分片
每天索引分片数：1TB / 30GB = 34 个分片
每天索引数：34 / 3（每个索引 3 个分片）= 12 个索引

// 配置示例
PUT _index_template/logs_template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "number_of_replicas": 1,
      "refresh_interval": "30s"
    }
  }
}

// 硬件配置
节点配置：
- CPU：16 核
- 内存：64GB（JVM 堆 31GB）
- 磁盘：2TB SSD
- 网络：10Gbps

// 角色分配
- Master 节点：3 个（专用）
- Data 节点：45 个
- Coordinating 节点：3 个（可选）
```

**最佳实践：**
- ✅ 单个分片 20-40GB
- ✅ 单节点分片数 < 20 个/GB 堆内存
- ✅ JVM 堆内存 < 32GB
- ✅ 预留 20% 存储空间
- ✅ 使用 SSD 磁盘

---

##### 问题 5：数据建模

**现象：**
```json
// 场景：电商商品搜索
// 问题：如何设计 Mapping？
```

**解决方案：**
```json
// 方案 1：合理设计 Mapping
PUT products
{
  "mappings": {
    "properties": {
      "title": {
        "type": "text",
        "analyzer": "ik_max_word",  // 中文分词
        "fields": {
          "keyword": {
            "type": "keyword"  // 精确匹配
          }
        }
      },
      "price": {
        "type": "double"
      },
      "category": {
        "type": "keyword"  // 不分词
      },
      "tags": {
        "type": "keyword"  // 数组
      },
      "description": {
        "type": "text",
        "index": false  // 不索引，节省空间
      },
      "create_time": {
        "type": "date"
      }
    }
  }
}

// 方案 2：嵌套对象 vs 父子文档
// 嵌套对象（推荐 - 性能好）
{
  "title": "iPhone 15",
  "specs": [
    { "name": "颜色", "value": "黑色" },
    { "name": "容量", "value": "256GB" }
  ]
}

// 父子文档（灵活 - 性能差）
// 父文档
PUT products/_doc/1
{
  "title": "iPhone 15"
}

// 子文档
PUT products/_doc/2?routing=1
{
  "spec_name": "颜色",
  "spec_value": "黑色",
  "product_join": {
    "name": "spec",
    "parent": 1
  }
}
```

**最佳实践：**
- ✅ text 用于全文搜索
- ✅ keyword 用于精确匹配、聚合、排序
- ✅ 不需要搜索的字段设置 index: false
- ✅ 优先使用嵌套对象
- ✅ 避免动态 Mapping

---

## 时序数据库

### 时序数据库实现

#### InfluxDB

**时序数据库特点**

| 维度 | InfluxDB | MySQL | Elasticsearch |
|:---|:---|:---|:---|
| 数据模型 | 时间序列 | 关系表 | 文档 |
| 写入性能 | 极高（百万/秒） | 中 | 高 |
| 压缩率 | 极高（10-100x） | 低 | 中 |
| 查询类型 | 时间范围、聚合 | 复杂 JOIN | 全文搜索 |
| 适用场景 | 监控、IoT、金融 | OLTP | 日志分析 |

**TSM 存储引擎**

```
TSM（Time-Structured Merge Tree）存储结构：

┌─────────────────────────────────────────────┐
│              WAL（预写日志）                  │
│  ├─ 顺序写入                                 │
│  ├─ 持久化保证                               │
│  └─ 故障恢复                                 │
└────────────┬────────────────────────────────┘
             ↓
┌─────────────────────────────────────────────┐
│           Cache（内存缓存）                   │
│  ├─ 最新数据                                 │
│  ├─ 排序存储                                 │
│  └─ 快速查询                                 │
└────────────┬────────────────────────────────┘
             ↓ 达到阈值
┌─────────────────────────────────────────────┐
│         TSM File（磁盘文件）                  │
│  ├─ 不可变文件                                │
│  ├─ 列式存储                                 │
│  ├─ 高压缩率                                 │
│  └─ 时间索引                                 │
└─────────────────────────────────────────────┘
             ↓ 定期合并
┌─────────────────────────────────────────────┐
│         Compaction（合并）                   │
│  ├─ 合并小文件                                │
│  ├─ 删除过期数据                              │
│  └─ 优化查询性能                              │
└─────────────────────────────────────────────┘
```

**数据模型**

```
InfluxDB 数据模型：

Measurement（表）
  ├─ Tags（标签，索引）
  │  ├─ host=server01
  │  └─ region=us-west
  ├─ Fields（字段，值）
  │  ├─ cpu_usage=80.5
  │  └─ memory_usage=60.2
  └─ Timestamp（时间戳）
     └─ 2024-01-01T00:00:00Z

示例数据：
cpu,host=server01,region=us-west value=80.5 1609459200000000000
│   │                           │     │
│   └─ Tags（索引）              └─ Field  └─ Timestamp（纳秒）
└─ Measurement
```

**Retention Policy（数据保留策略）**

```sql
-- 创建保留策略
CREATE RETENTION POLICY "one_week"
ON "mydb"
DURATION 7d
REPLICATION 1
DEFAULT;

-- 数据自动过期
-- 7 天后自动删除

-- 多级保留策略
CREATE RETENTION POLICY "raw_data" ON "mydb" DURATION 7d REPLICATION 1;
CREATE RETENTION POLICY "downsampled" ON "mydb" DURATION 90d REPLICATION 1;

-- Continuous Query（连续查询）自动降采样
CREATE CONTINUOUS QUERY "cq_downsample"
ON "mydb"
BEGIN
  SELECT mean(value) AS value
  INTO "downsampled"."cpu"
  FROM "raw_data"."cpu"
  GROUP BY time(1h), *
END;
```

**查询语言（InfluxQL）**

```sql
-- 基本查询
SELECT value FROM cpu WHERE time > now() - 1h;

-- 聚合查询
SELECT mean(value) FROM cpu
WHERE time > now() - 1d
GROUP BY time(1h), host;

-- 结果：
time                 host      mean
----                 ----      ----
2024-01-01T00:00:00Z server01  75.5
2024-01-01T01:00:00Z server01  80.2
2024-01-01T00:00:00Z server02  65.3

-- 窗口函数
SELECT moving_average(value, 5) FROM cpu
WHERE time > now() - 1h;
```

**性能优化**

```sql
-- 1. 使用 Tags 而不是 Fields 作为过滤条件
-- ✅ 好（Tags 有索引）
SELECT value FROM cpu WHERE host='server01';

-- ❌ 差（Fields 无索引）
SELECT value FROM cpu WHERE value > 80;

-- 2. 限制时间范围
SELECT value FROM cpu
WHERE time > now() - 1h  -- 必须指定时间范围
AND host='server01';

-- 3. 使用降采样数据
SELECT mean(value) FROM "downsampled"."cpu"
WHERE time > now() - 30d;
```

---

##### InfluxDB 实战场景

**典型场景：** IoT 传感器数据、系统监控、性能指标

**常见问题与解决方案：**

##### 问题 1：数据建模（Tags vs Fields）

**现象：** 查询慢，内存占用高

**原因：** Tags 和 Fields 选择错误
- Tags：索引字段，用于过滤和分组
- Fields：数值字段，不索引

**解决方案：**
```sql
-- ✅ 正确设计
-- Tags: host, region（高基数，用于过滤）
-- Fields: cpu_usage, memory（数值，用于计算）
INSERT cpu,host=server01,region=us-west cpu_usage=80.5,memory=60.2

-- ❌ 错误：数值字段用 Tags
INSERT cpu,cpu_usage=80.5 host="server01"  -- cpu_usage 不应该是 Tag
```

**最佳实践：**
- ✅ Tags：字符串、低基数（< 10万）、用于 WHERE/GROUP BY
- ✅ Fields：数值、高基数、用于聚合计算
- ✅ 避免高基数 Tags（如 UUID、时间戳）

---

##### 问题 2：降采样策略

**现象：** 历史数据查询慢，存储空间大

**原因：** 保留所有原始数据

**解决方案：**
```sql
-- Continuous Query 自动降采样
CREATE CONTINUOUS QUERY "cq_1h" ON "mydb"
BEGIN
  SELECT mean(cpu_usage) AS cpu_usage
  INTO "mydb"."autogen"."cpu_1h"
  FROM "mydb"."autogen"."cpu"
  GROUP BY time(1h), *
END

-- 原始数据：保留 7 天
-- 1 小时聚合：保留 90 天
```

**最佳实践：**
- ✅ 原始数据：1 秒粒度，保留 7 天
- ✅ 1 分钟聚合：保留 30 天
- ✅ 1 小时聚合：保留 1 年

---

##### 问题 3：保留策略

**现象：** 磁盘空间不足

**原因：** 未设置自动删除策略

**解决方案：**
```sql
-- 创建保留策略
CREATE RETENTION POLICY "7days" ON "mydb" DURATION 7d REPLICATION 1 DEFAULT

-- 自动删除 7 天前的数据
```

**最佳实践：**
- ✅ 根据业务需求设置保留时间
- ✅ 使用多级保留策略（原始 + 聚合）
- ✅ 监控磁盘使用率

---

#### Prometheus

**监控架构**

```
┌─────────────────────────────────────────────┐
│           Prometheus Server                 │
│  ┌──────────────────────────────────────┐   │
│  │  Retrieval（数据采集）                 │   │
│  │  ├─ Pull 模式（主动拉取）               │   │
│  │  └─ Service Discovery（服务发现）      │   │
│  └────────────┬─────────────────────────┘   │
│               ↓                             │
│  ┌──────────────────────────────────────┐   │
│  │  TSDB（时序数据库）                    │   │
│  │  ├─ 本地存储                          │   │
│  │  ├─ 2小时 Block                      │   │
│  │  └─ 压缩合并                          │   │
│  └────────────┬─────────────────────────┘   │ 
│               ↓                             │
│  ┌──────────────────────────────────────┐   │
│  │  PromQL（查询引擎）                    │   │
│  └──────────────────────────────────────┘   │
└────────────────┬────────────────────────────┘
                 ↓
┌────────────────┴────────────────────────────┐
│         Alertmanager（告警管理）              │
│  ├─ 告警规则                                 │
│  ├─ 告警分组                                 │
│  └─ 告警通知                                 │
└─────────────────────────────────────────────┘
```

**数据模型**

```
Prometheus 指标格式：

http_requests_total{method="GET", status="200"} 1234 @1609459200

│                   │                          │    │
└─ Metric Name      └─ Labels（标签）           │    └─ Timestamp
                                               └─ Value

指标类型：
1. Counter（计数器）：只增不减
   - http_requests_total
   - errors_total

2. Gauge（仪表盘）：可增可减
   - cpu_usage
   - memory_usage

3. Histogram（直方图）：分布统计
   - http_request_duration_seconds

4. Summary（摘要）：分位数统计
   - http_request_duration_seconds_summary
```

**PromQL 查询语言**

```promql
# 即时查询
http_requests_total

# 范围查询（过去 5 分钟）
http_requests_total[5m]

# 速率计算（每秒请求数）
rate(http_requests_total[5m])

# 聚合查询
sum(rate(http_requests_total[5m])) by (method)

# 结果：
{method="GET"}  100
{method="POST"} 50

# 告警规则
- alert: HighErrorRate
  expr: rate(errors_total[5m]) > 0.05
  for: 10m
  labels:
    severity: critical
  annotations:
    summary: "Error rate > 5%"
```

**服务发现**

```yaml
# prometheus.yml
scrape_configs:
  # 静态配置
  - job_name: 'static'
    static_configs:
      - targets: ['localhost:9090']

  # Kubernetes 服务发现
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true

  # Consul 服务发现
  - job_name: 'consul'
    consul_sd_configs:
      - server: 'localhost:8500'
```

---

##### Prometheus 实战场景

**典型场景：** 服务监控、告警系统、容量规划

**常见问题与解决方案：**

##### 问题 1：指标设计

**现象：** 指标类型选择错误，无法正确计算

**原因：** Counter、Gauge、Histogram 使用场景不同

**解决方案：**
```promql
# Counter（只增不减）：请求数、错误数
rate(http_requests_total[5m])  # 每秒请求数

# Gauge（可增可减）：CPU、内存、连接数
avg_over_time(cpu_usage[5m])  # 5 分钟平均 CPU

# Histogram（分布统计）：响应时间
histogram_quantile(0.95, rate(http_duration_bucket[5m]))  # P95 延迟
```

**最佳实践：**
- ✅ Counter：累计值（请求数、字节数）
- ✅ Gauge：瞬时值（温度、内存、队列长度）
- ✅ Histogram：分布分析（延迟、大小）

---

##### 问题 2：告警规则

**现象：** 告警风暴，误报频繁

**原因：** 告警阈值设置不合理

**解决方案：**
```yaml
# 避免瞬时抖动
- alert: HighErrorRate
  expr: rate(errors_total[5m]) > 0.05
  for: 10m  # 持续 10 分钟才告警
  annotations:
    summary: "错误率 > 5%"

# 使用 avg_over_time 平滑
- alert: HighCPU
  expr: avg_over_time(cpu_usage[5m]) > 80
  for: 15m
```

**最佳实践：**
- ✅ 使用 `for` 避免瞬时抖动
- ✅ 设置合理的阈值（基于历史数据）
- ✅ 告警分级（warning/critical）

---

##### 问题 3：存储优化

**现象：** 本地存储空间不足

**原因：** Prometheus 本地存储有限（默认 15 天）

**解决方案：**
```yaml
# 远程存储（Thanos）
remote_write:
  - url: "http://thanos:19291/api/v1/receive"

# 或使用 Cortex
remote_write:
  - url: "http://cortex:9009/api/v1/push"
```

**最佳实践：**
- ✅ 本地存储：15-30 天
- ✅ 远程存储：长期数据（Thanos、Cortex、VictoriaMetrics）
- ✅ 使用 Recording Rules 减少存储

---

##### 问题 4：高可用

**现象：** 单点故障，监控中断

**原因：** 单实例部署

**解决方案：**
```yaml
# 联邦集群（多个 Prometheus 实例）
# 实例 1 和实例 2 采集相同目标
# Alertmanager 去重告警

# 或使用 Thanos Sidecar 实现高可用
```

**最佳实践：**
- ✅ 部署多个 Prometheus 实例
- ✅ 使用 Alertmanager 集群去重
- ✅ 使用 Thanos 实现全局视图

---

#### TimescaleDB

**Hypertable（超表）**

```sql
-- 创建普通表
CREATE TABLE sensor_data (
  time TIMESTAMPTZ NOT NULL,
  sensor_id INTEGER,
  temperature DOUBLE PRECISION,
  humidity DOUBLE PRECISION
);

-- 转换为 Hypertable
SELECT create_hypertable('sensor_data', 'time');

-- 自动分区（Chunk）
-- 每个 Chunk 默认 7 天数据
┌─────────────────────────────────────────────┐
│           Hypertable: sensor_data           │
│                                             │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Chunk 1  │  │ Chunk 2  │  │ Chunk 3  │   │
│  │ 1-7 天   │  │ 8-14 天   │  │ 15-21 天 │   │
│  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────┘
```

**时间分区**

```sql
-- 自定义分区间隔
SELECT create_hypertable('sensor_data', 'time',
  chunk_time_interval => INTERVAL '1 day');

-- 空间分区（多维分区）
SELECT create_hypertable('sensor_data', 'time',
  partitioning_column => 'sensor_id',
  number_partitions => 4);

-- 查询自动路由到相关 Chunk
SELECT * FROM sensor_data
WHERE time > now() - INTERVAL '1 day'
AND sensor_id = 123;
-- 只扫描相关 Chunk，性能极高
```

**压缩策略**

```sql
-- 启用压缩
ALTER TABLE sensor_data SET (
  timescaledb.compress,
  timescaledb.compress_segmentby = 'sensor_id',
  timescaledb.compress_orderby = 'time DESC'
);

-- 自动压缩策略
SELECT add_compression_policy('sensor_data',
  INTERVAL '7 days');

-- 压缩效果：
-- 原始数据：1GB
-- 压缩后：100MB（10x 压缩率）
```

**连续聚合（Continuous Aggregate）**

```sql
-- 创建连续聚合视图
CREATE MATERIALIZED VIEW sensor_data_hourly
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', time) AS hour,
       sensor_id,
       avg(temperature) AS avg_temp,
       max(temperature) AS max_temp
FROM sensor_data
GROUP BY hour, sensor_id;

-- 自动刷新策略
SELECT add_continuous_aggregate_policy('sensor_data_hourly',
  start_offset => INTERVAL '3 hours',
  end_offset => INTERVAL '1 hour',
  schedule_interval => INTERVAL '1 hour');

---

##### TimescaleDB 实战场景

**典型场景：** 金融数据、传感器数据、日志分析

**常见问题与解决方案：**

##### 问题 1：Hypertable 设计

**现象：** 分区过多或过少，影响性能

**原因：** 分区间隔设置不合理

**解决方案：**
```sql
-- 根据数据量选择分区间隔
-- 高频数据（每天 > 1GB）：1 天分区
SELECT create_hypertable('sensor_data', 'time', chunk_time_interval => INTERVAL '1 day');

-- 低频数据（每天 < 100MB）：7 天分区
SELECT create_hypertable('logs', 'time', chunk_time_interval => INTERVAL '7 days');
```

**最佳实践：**
- ✅ 单个 Chunk 大小：10-100GB
- ✅ 高频数据：1 天分区
- ✅ 低频数据：7 天分区

---

##### 问题 2：压缩策略

**现象：** 存储空间占用大

**原因：** 未启用压缩

**解决方案：**
```sql
-- 启用压缩（10x 压缩率）
ALTER TABLE sensor_data SET (
  timescaledb.compress,
  timescaledb.compress_segmentby = 'sensor_id'
);

-- 自动压缩 7 天前的数据
SELECT add_compression_policy('sensor_data', INTERVAL '7 days');
```

**最佳实践：**
- ✅ 压缩率：10-100x
- ✅ 压缩时机：7 天后（冷数据）
- ✅ segmentby：高基数字段

---

##### 问题 3：连续聚合

**现象：** 历史数据聚合查询慢

**原因：** 每次查询都实时计算

**解决方案：**
```sql
-- 创建连续聚合（100x 性能提升）
CREATE MATERIALIZED VIEW sensor_hourly
WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', time) AS hour,
       sensor_id,
       avg(temperature) AS avg_temp
FROM sensor_data
GROUP BY hour, sensor_id;

-- 自动刷新
SELECT add_continuous_aggregate_policy('sensor_hourly',
  start_offset => INTERVAL '3 hours',
  end_offset => INTERVAL '1 hour',
  schedule_interval => INTERVAL '1 hour');
```

**最佳实践：**
- ✅ 1 小时聚合：保留 1 年
- ✅ 1 天聚合：保留 5 年
- ✅ 自动刷新：每小时

---

##### 问题 4：保留策略

**现象：** 历史数据占用空间

**原因：** 未设置自动删除

**解决方案：**
```sql
-- 自动删除 90 天前的数据
SELECT add_retention_policy('sensor_data', INTERVAL '90 days');
```

**最佳实践：**
- ✅ 原始数据：30-90 天
- ✅ 聚合数据：1-5 年
- ✅ 监控存储使用率

---

  start_offset => INTERVAL '3 hours',
  end_offset => INTERVAL '1 hour',
  schedule_interval => INTERVAL '1 hour');

-- 查询自动使用聚合视图
SELECT * FROM sensor_data_hourly
WHERE hour > now() - INTERVAL '1 day';
-- 查询速度提升 100x
```

**数据保留策略**

```sql
-- 自动删除旧数据
SELECT add_retention_policy('sensor_data',
  INTERVAL '90 days');

-- 分层存储
-- 1. 最近 7 天：原始数据（未压缩）
-- 2. 7-30 天：压缩数据
-- 3. 30-90 天：聚合数据
-- 4. 90 天后：自动删除
```

**性能对比**

| 操作 | PostgreSQL | TimescaleDB | 提升 |
|:---|:---|:---|:---|
| 插入（百万行/秒） | 5 万 | 50 万 | 10x |
| 时间范围查询 | 10 秒 | 0.1 秒 | 100x |
| 聚合查询 | 30 秒 | 0.3 秒 | 100x |
| 存储空间 | 100GB | 10GB | 10x |

---

## 图数据库

### 图数据库实现

#### Neo4j

**图数据库特点**

| 维度 | Neo4j | MySQL | MongoDB |
|:---|:---|:---|:---|
| 数据模型 | 图（节点+关系） | 表（行列） | 文档 |
| 关系查询 | 极快（原生图） | 慢（JOIN） | 慢（$lookup） |
| 多跳查询 | O(1) | O(n^m) | O(n^m) |
| 适用场景 | 社交网络、推荐、知识图谱 | OLTP | 文档存储 |

**图模型**

```
图 = 节点（Node） + 关系（Relationship） + 属性（Property）

┌─────────────────────────────────────────────┐
│              社交网络图示例                   │
│                                             │
│  (Alice:Person {age:25})                    │
│       │                                     │
│       │ [:FOLLOWS {since:2020}]             │
│       ↓                                     │
│  (Bob:Person {age:30})                      │
│       │                                     │
│       │ [:LIKES]                            │
│       ↓                                     │
│  (Post:Content {title:"Hello"})             │
│       ↑                                     │
│       │ [:CREATED]                          │
│       │                                     │
│  (Charlie:Person {age:28})                  │
└─────────────────────────────────────────────┘

节点（Node）：
- 标签（Label）：Person, Post
- 属性（Property）：age, title

关系（Relationship）：
- 类型（Type）：FOLLOWS, LIKES, CREATED
- 方向（Direction）：单向
- 属性（Property）：since
```

**Cypher 查询语言**

```cypher
-- 创建节点
CREATE (alice:Person {name: 'Alice', age: 25})
CREATE (bob:Person {name: 'Bob', age: 30})

-- 创建关系
MATCH (alice:Person {name: 'Alice'})
MATCH (bob:Person {name: 'Bob'})
CREATE (alice)-[:FOLLOWS {since: 2020}]->(bob)

-- 查询：Alice 关注的人
MATCH (alice:Person {name: 'Alice'})-[:FOLLOWS]->(friend)
RETURN friend.name

-- 查询：Alice 的二度好友（朋友的朋友）
MATCH (alice:Person {name: 'Alice'})-[:FOLLOWS]->()-[:FOLLOWS]->(fof)
RETURN DISTINCT fof.name

-- 查询：最短路径
MATCH path = shortestPath(
  (alice:Person {name: 'Alice'})-[:FOLLOWS*]-(bob:Person {name: 'Bob'})
)
RETURN length(path)

-- 查询：推荐好友（共同好友最多）
MATCH (alice:Person {name: 'Alice'})-[:FOLLOWS]->(friend)-[:FOLLOWS]->(fof)
WHERE NOT (alice)-[:FOLLOWS]->(fof) AND alice <> fof
RETURN fof.name, count(*) AS common_friends
ORDER BY common_friends DESC
LIMIT 10
```

**图算法**

```cypher
-- 1. PageRank（网页排名）
CALL gds.pageRank.stream('myGraph')
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).name AS name, score
ORDER BY score DESC
LIMIT 10

-- 2. 社区发现（Louvain）
CALL gds.louvain.stream('myGraph')
YIELD nodeId, communityId
RETURN communityId, collect(gds.util.asNode(nodeId).name) AS members

-- 3. 中心性分析（Betweenness Centrality）
CALL gds.betweenness.stream('myGraph')
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).name AS name, score
ORDER BY score DESC

-- 4. 相似度计算（Jaccard Similarity）
MATCH (p1:Person {name: 'Alice'})-[:FOLLOWS]->(common)<-[:FOLLOWS]-(p2:Person)
WITH p1, p2, count(common) AS intersection
MATCH (p1)-[:FOLLOWS]->(p1_follows)
WITH p1, p2, intersection, count(p1_follows) AS p1_count
MATCH (p2)-[:FOLLOWS]->(p2_follows)
WITH p1, p2, intersection, p1_count, count(p2_follows) AS p2_count
RETURN p2.name, 
       toFloat(intersection) / (p1_count + p2_count - intersection) AS jaccard
ORDER BY jaccard DESC
```

**索引和性能优化**

```cypher
-- 创建索引
CREATE INDEX person_name FOR (p:Person) ON (p.name)
CREATE INDEX person_age FOR (p:Person) ON (p.age)

-- 复合索引
CREATE INDEX person_name_age FOR (p:Person) ON (p.name, p.age)

-- 全文索引
CALL db.index.fulltext.createNodeIndex(
  'personFulltext',
  ['Person'],
  ['name', 'bio']
)

-- 使用全文索引
CALL db.index.fulltext.queryNodes('personFulltext', 'Alice~')
YIELD node, score
RETURN node.name, score

-- 查询优化
-- ✅ 好：使用索引
MATCH (p:Person {name: 'Alice'})
RETURN p

-- ❌ 差：全表扫描
MATCH (p:Person)
WHERE p.age > 25
RETURN p

-- ✅ 好：限制深度
MATCH (p:Person {name: 'Alice'})-[:FOLLOWS*1..3]->(friend)
RETURN friend.name

-- ❌ 差：无限深度（可能遍历整个图）
MATCH (p:Person {name: 'Alice'})-[:FOLLOWS*]->(friend)
RETURN friend.name
```

**实际应用场景**

```cypher
-- 1. 社交网络：推荐好友
MATCH (user:Person {name: 'Alice'})-[:FOLLOWS]->(friend)-[:FOLLOWS]->(fof)
WHERE NOT (user)-[:FOLLOWS]->(fof) AND user <> fof
WITH fof, count(*) AS common_friends
ORDER BY common_friends DESC
LIMIT 10
MATCH (fof)-[:LIKES]->(content)
RETURN fof.name, common_friends, collect(content.title) AS interests

-- 2. 知识图谱：实体关系查询
MATCH (entity:Entity {name: 'Apple'})-[r]->(related)
RETURN type(r) AS relationship, related.name
LIMIT 20

-- 3. 欺诈检测：环路检测
MATCH path = (account:Account)-[:TRANSFER*3..5]->(account)
WHERE ALL(r IN relationships(path) WHERE r.amount > 10000)
RETURN path

-- 4. 推荐系统：协同过滤
MATCH (user:User {id: 123})-[:PURCHASED]->(product)<-[:PURCHASED]-(other)
MATCH (other)-[:PURCHASED]->(recommendation)
WHERE NOT (user)-[:PURCHASED]->(recommendation)
RETURN recommendation.name, count(*) AS score
ORDER BY score DESC
LIMIT 10
```

---

##### Neo4j 实战场景

##### 典型应用场景

| 场景 | 业务描述 | 数据量 | 并发量 | 关键指标 | 技术要点 |
|:---|:---|:---|:---|:---|:---|
| **社交网络** | 好友关系、推荐好友 | 亿级节点 | 5000 QPS | 多跳查询、路径查找 | 索引、Cypher 优化 |
| **推荐系统** | 协同过滤、关联推荐 | 千万级 | 1万 QPS | 图算法、实时计算 | PageRank、相似度 |
| **知识图谱** | 实体关系、语义搜索 | 亿级 | 5000 QPS | 复杂查询、推理 | 图建模、索引 |
| **欺诈检测** | 关系分析、环路检测 | 千万级 | 1000 QPS | 实时检测、模式匹配 | 图算法、规则引擎 |

---

##### 常见问题与解决方案

##### 问题 1：图建模

**现象：**
```cypher
-- 场景：电商订单系统
-- 问题：如何设计图模型？
```

**解决方案：**
```cypher
-- 方案 1：节点设计
CREATE (u:User {id: 1, name: "Tom"})
CREATE (p:Product {id: 101, name: "iPhone"})
CREATE (o:Order {id: 1001, amount: 5999})

-- 方案 2：关系设计
CREATE (u)-[:PLACED]->(o)
CREATE (o)-[:CONTAINS {quantity: 1}]->(p)

-- 方案 3：查询优化
// ❌ 差：笛卡尔积
MATCH (u:User), (p:Product)
WHERE u.id = 1 AND p.id = 101
RETURN u, p

// ✅ 好：使用关系
MATCH (u:User {id: 1})-[:PLACED]->(o)-[:CONTAINS]->(p:Product {id: 101})
RETURN u, o, p
```

**最佳实践：**
- ✅ 节点表示实体，关系表示连接
- ✅ 关系可以有属性
- ✅ 避免笛卡尔积查询
- ✅ 使用索引加速查询

---

##### 问题 2：Cypher 优化

**现象：**
```cypher
-- 查询慢，耗时 10 秒
MATCH (u:User)-[:FOLLOWS*2..3]->(friend)
WHERE u.name = 'Tom'
RETURN friend.name
```

**解决方案：**
```cypher
-- 方案 1：创建索引
CREATE INDEX user_name FOR (u:User) ON (u.name)

-- 方案 2：限制深度和数量
MATCH (u:User {name: 'Tom'})-[:FOLLOWS*1..2]->(friend)
RETURN friend.name
LIMIT 100

-- 方案 3：使用 PROFILE 分析
PROFILE
MATCH (u:User {name: 'Tom'})-[:FOLLOWS]->(friend)
RETURN friend.name

-- 方案 4：避免全图扫描
// ❌ 差
MATCH (u:User)-[:FOLLOWS]->(friend)
RETURN u.name, friend.name

// ✅ 好
MATCH (u:User {name: 'Tom'})-[:FOLLOWS]->(friend)
RETURN u.name, friend.name
```

**最佳实践：**
- ✅ 创建索引（节点属性）
- ✅ 限制遍历深度（1-3 跳）
- ✅ 使用 LIMIT 限制结果
- ✅ 使用 PROFILE 分析性能

---

##### 问题 3：性能调优

**现象：**
```
场景：社交网络，查询二度好友慢
问题：如何优化？
```

**解决方案：**
```cypher
-- 方案 1：预计算（物化视图）
// 定时任务计算二度好友
MATCH (u:User {id: 1})-[:FOLLOWS]->()-[:FOLLOWS]->(fof)
WHERE NOT (u)-[:FOLLOWS]->(fof) AND u <> fof
WITH u, fof, count(*) AS common
CREATE (u)-[:RECOMMENDED {score: common, updated: timestamp()}]->(fof)

// 查询预计算结果
MATCH (u:User {id: 1})-[r:RECOMMENDED]->(fof)
RETURN fof.name, r.score
ORDER BY r.score DESC
LIMIT 10

-- 方案 2：使用图算法
// PageRank
CALL gds.pageRank.stream('myGraph')
YIELD nodeId, score
RETURN gds.util.asNode(nodeId).name, score
ORDER BY score DESC

-- 方案 3：缓存热点数据
// 应用层缓存
redis.set("user:1:friends", json.dumps(friends))
```

**最佳实践：**
- ✅ 预计算复杂查询结果
- ✅ 使用图算法库（GDS）
- ✅ 缓存热点数据到 Redis
- ✅ 定期更新预计算结果

---

## 数据库理论

### 核心理论

#### CAP 定理

**CAP 三角形**

```
        Consistency（一致性）
              ▲
             / \
            /   \
           /     \
          /  CA   \
         /         \
        /    CAP    \
       /             \
      /      CP       \
     /                 \
    /        AP         \
   /_____________________\
Availability          Partition Tolerance
（可用性）              （分区容错性）

定理：分布式系统最多只能同时满足 CAP 中的两项

实际选择：
- 网络分区（P）不可避免，必须选择
- 只能在 C 和 A 之间权衡
- CP：牺牲可用性，保证一致性
- AP：牺牲一致性，保证可用性
```

**各数据库的 CAP 选择**

| 数据库 | CAP 选择 | 说明 | 适用场景 |
|:---|:---|:---|:---|
| **MySQL** | CA | 单机强一致性，网络分区时不可用 | 传统 OLTP |
| **MongoDB** | CP | 主节点故障时不可用，保证一致性 | 文档存储 |
| **Cassandra** | AP | 最终一致性，高可用 | IoT、时序数据 |
| **Redis** | AP | 主从异步复制，最终一致性 | 缓存、会话 |
| **HBase** | CP | 强一致性，Region 故障时不可用 | 大数据存储 |
| **Elasticsearch** | AP | 最终一致性，高可用 | 日志分析 |
| **Spanner** | CP | 全球强一致性，TrueTime | 金融、全球业务 |

**一致性级别**

```
强一致性（Strong Consistency）
- 读操作立即看到最新写入
- 例如：MySQL、MongoDB（默认）

最终一致性（Eventual Consistency）
- 读操作可能看到旧数据
- 一段时间后达到一致
- 例如：Cassandra、Redis

因果一致性（Causal Consistency）
- 有因果关系的操作保证顺序
- 例如：评论必须在帖子之后

会话一致性（Session Consistency）
- 同一会话内保证一致性
- 例如：用户看到自己的写入
```

#### BASE 理论

**BASE vs ACID**

| 维度 | ACID（传统数据库） | BASE（分布式系统） |
|:---|:---|:---|
| 一致性 | 强一致性 | 最终一致性 |
| 可用性 | 可能不可用 | 高可用 |
| 性能 | 较低 | 高 |
| 适用场景 | 金融、订单 | 社交、推荐 |

**BASE 三要素**

```
1. Basically Available（基本可用）
   - 系统保证基本可用
   - 允许部分功能降级
   - 例如：双11 限流、降级非核心功能

2. Soft State（软状态）
   - 允许中间状态存在
   - 数据可以有短暂不一致
   - 例如：库存扣减后，订单状态异步更新

3. Eventually Consistent（最终一致性）
   - 经过一段时间后达到一致
   - 不保证实时一致
   - 例如：Redis 主从复制延迟
```

#### 分布式事务

**2PC（两阶段提交）**

```
协调者（Coordinator）
    ↓
┌───────────────────────────────────────┐
│  阶段 1：准备阶段（Prepare Phase）       │
│                                       │
│  Coordinator → Participant A: Prepare │
│  Coordinator → Participant B: Prepare │
│                                       │
│  Participant A → Coordinator: Yes     │
│  Participant B → Coordinator: Yes     │
└───────────────┬───────────────────────┘
                ↓
┌───────────────────────────────────────┐
│  阶段 2：提交阶段（Commit Phase）        │
│                                       │
│  Coordinator → Participant A: Commit  │
│  Coordinator → Participant B: Commit  │
│                                       │
│  Participant A → Coordinator: ACK     │
│  Participant B → Coordinator: ACK     │
└───────────────────────────────────────┘

优点：
✅ 强一致性
✅ 实现简单

缺点：
❌ 同步阻塞（性能差）
❌ 单点故障（Coordinator 故障）
❌ 数据不一致（Commit 阶段部分失败）
```

**3PC（三阶段提交）**

```
阶段 1：CanCommit（询问）
阶段 2：PreCommit（预提交）
阶段 3：DoCommit（提交）

改进：
✅ 增加超时机制
✅ 减少阻塞时间

缺点：
❌ 仍然有数据不一致风险
❌ 实现复杂
```

**TCC（Try-Confirm-Cancel）**

```java
// TCC 模式示例：转账

// Try 阶段：预留资源
public void tryTransfer(String from, String to, int amount) {
    // 冻结 from 账户金额
    accountService.freeze(from, amount);
    // 预增加 to 账户金额（标记为冻结）
    accountService.preAdd(to, amount);
}

// Confirm 阶段：确认提交
public void confirmTransfer(String from, String to, int amount) {
    // 扣减 from 账户冻结金额
    accountService.deduct(from, amount);
    // 确认增加 to 账户金额
    accountService.add(to, amount);
}

// Cancel 阶段：回滚
public void cancelTransfer(String from, String to, int amount) {
    // 解冻 from 账户金额
    accountService.unfreeze(from, amount);
    // 取消 to 账户预增加
    accountService.cancelPreAdd(to, amount);
}

优点：
✅ 无锁，高性能
✅ 业务可控

缺点：
❌ 业务侵入性强
❌ 需要实现 Try、Confirm、Cancel 三个接口
```

**Saga 模式**

```
长事务拆分为多个本地事务

正向流程：
T1 → T2 → T3 → T4 → 成功

补偿流程（T3 失败）：
T1 → T2 → T3 ✗ → C2 → C1

示例：订单流程
┌─────────────────────────────────────┐
│  T1: 创建订单                        │
│  T2: 扣减库存                        │
│  T3: 扣减余额                        │
│  T4: 发送通知                        │
└─────────────────────────────────────┘

如果 T3 失败：
┌─────────────────────────────────────┐
│  C2: 恢复库存                        │
│  C1: 取消订单                        │
└─────────────────────────────────────┘

优点：
✅ 长事务支持
✅ 无锁，高性能

缺点：
❌ 需要实现补偿逻辑
❌ 无隔离性（中间状态可见）
```

**分布式事务对比**

| 方案 | 一致性 | 性能 | 复杂度 | 适用场景 |
|:---|:---|:---|:---|:---|
| 2PC | 强一致性 | 低 | 低 | 小规模、强一致性要求 |
| 3PC | 强一致性 | 低 | 中 | 改进 2PC |
| TCC | 最终一致性 | 高 | 高 | 高并发、业务可控 |
| Saga | 最终一致性 | 高 | 中 | 长事务、微服务 |
| 本地消息表 | 最终一致性 | 高 | 中 | 异步场景 |

#### 一致性协议

**Paxos 协议**

```
角色：
- Proposer（提议者）：提出提案
- Acceptor（接受者）：投票
- Learner（学习者）：学习结果

两阶段：
1. Prepare 阶段
   - Proposer 发送 Prepare(n)
   - Acceptor 承诺不接受编号 < n 的提案

2. Accept 阶段
   - Proposer 发送 Accept(n, value)
   - Acceptor 接受提案
   - 多数派接受则达成共识

特点：
✅ 理论完备
❌ 实现复杂
❌ 难以理解
```

**Raft 协议**

```
角色：
- Leader（领导者）：处理所有请求
- Follower（跟随者）：被动接收
- Candidate（候选者）：选举中

选举流程：
1. Follower 超时 → 变为 Candidate
2. Candidate 发起投票
3. 获得多数票 → 变为 Leader
4. Leader 发送心跳维持地位

日志复制：
┌─────────────────────────────────────┐
│  Leader                             │
│  ├─ 接收客户端请求                    │
│  ├─ 写入本地日志                      │
│  ├─ 复制到 Follower                  │
│  ├─ 多数派确认                        │
│  └─ 提交并返回客户端                  │
└─────────────────────────────────────┘

特点：
✅ 易于理解
✅ 易于实现
✅ 工程化友好
```

**ZAB 协议（ZooKeeper）**

```
类似 Raft，但有区别：

1. 崩溃恢复模式
   - Leader 选举
   - 数据同步

2. 消息广播模式
   - Leader 接收请求
   - 广播到 Follower
   - 多数派确认

特点：
✅ 专为 ZooKeeper 设计
✅ 保证顺序一致性
```

**协议对比**

| 协议 | 复杂度 | 性能 | 使用场景 |
|:---|:---|:---|:---|
| Paxos | 高 | 中 | 理论研究 |
| Raft | 低 | 高 | etcd、TiKV、Consul |
| ZAB | 中 | 高 | ZooKeeper |

#### 数据库选型决策

**选型矩阵**

```
┌─────────────────────────────────────────────┐
│              数据库选型决策树                │
│                                             │
│  需要事务？                                  │
│  ├─ 是 → 需要分布式？                        │
│  │  ├─ 是 → TiDB、Spanner、CockroachDB      │
│  │  └─ 否 → MySQL、PostgreSQL               │
│  └─ 否 → 数据模型？                          │
│     ├─ 文档 → MongoDB                       │
│     ├─ 宽列 → Cassandra、HBase              │
│     ├─ 图 → Neo4j                           │
│     ├─ 时序 → InfluxDB、Prometheus          │
│     ├─ 搜索 → Elasticsearch                 │
│     └─ 缓存 → Redis                         │
└─────────────────────────────────────────────┘
```

**场景选型**

| 场景 | 推荐数据库 | 理由 |
|:---|:---|:---|
| 电商订单 | MySQL + Redis | 强一致性 + 高性能缓存 |
| 社交网络 | Neo4j + Redis | 图关系 + 缓存 |
| 日志分析 | Elasticsearch | 全文搜索 + 聚合 |
| 监控告警 | Prometheus + InfluxDB | 时序数据 + 告警 |
| 实时分析 | ClickHouse | 列式存储 + 高性能 |
| 全球业务 | Spanner + DynamoDB | 全球一致性 + 高可用 |
| IoT 数据 | Cassandra + InfluxDB | 高写入 + 时序 |
| 推荐系统 | Neo4j + Redis | 图算法 + 缓存 |

---

## NewSQL

### NewSQL 数据库实现

#### TiDB

**HTAP 架构**

```
┌─────────────────────────────────────────────┐
│              TiDB Server（SQL 层）           │
│  ├─ SQL 解析                                 │
│  ├─ 查询优化                                  │
│  └─ 分布式执行                                │
└────────┬────────────────────┬───────────────┘
         ↓                    ↓
┌────────────────┐    ┌────────────────┐
│  TiKV（行存储） │    │ TiFlash（列存储）│
│  OLTP 负载     │    │  OLAP 负载      │
│  ├─ Raft 副本  │    │  ├─ 列式存储    │
│  ├─ RocksDB    │    │  ├─ 实时同步    │
│  └─ 事务支持   │    │  └─ 向量化执行  │
└────────────────┘    └────────────────┘
         ↓                    ↓
┌─────────────────────────────────────────────┐
│              PD（调度中心）                   │
│  ├─ 元数据管理                               │
│  ├─ 负载均衡                                 │
│  └─ 时间戳分配（TSO）                         │
└─────────────────────────────────────────────┘
```

**分布式事务（Percolator）**

```
两阶段提交 + MVCC：

1. Prewrite 阶段
   - 写入所有 Key 的锁
   - 写入数据（带时间戳）
   - 选择一个 Primary Key

2. Commit 阶段
   - 提交 Primary Key
   - 异步提交其他 Key
   - 清理锁

示例：转账事务
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;

执行流程：
┌─────────────────────────────────────────────┐
│  Prewrite 阶段                               │
│  ├─ Lock(id=1) + Write(balance=900, ts=10)  │
│  └─ Lock(id=2) + Write(balance=1100, ts=10) │
└────────────────┬────────────────────────────┘
                 ↓
┌─────────────────────────────────────────────┐
│  Commit 阶段                                 │
│  ├─ Commit(id=1, ts=11) ← Primary Key       │
│  └─ Commit(id=2, ts=11) ← 异步提交           │
└─────────────────────────────────────────────┘

特点：
✅ 分布式 ACID
✅ 快照隔离（SI）
✅ 无单点故障
```

**Raft 协议实现**

```
TiKV 使用 Raft 保证数据一致性：

Region（数据分片）：
┌─────────────────────────────────────────────┐
│  Region 1: [a, m)                           │
│  ├─ Leader（TiKV-1）                        │
│  ├─ Follower（TiKV-2）                      │
│  └─ Follower（TiKV-3）                      │
└─────────────────────────────────────────────┘

写入流程：
1. 客户端 → Leader
2. Leader 写入 Raft Log
3. Leader 复制到 Follower
4. 多数派确认（2/3）
5. Leader 应用到状态机
6. 返回客户端

Region 分裂：
- 当 Region 大小超过 96MB 时自动分裂
- 分裂成 2 个 Region
- 保持负载均衡
```

**HTAP 查询路由**

```sql
-- OLTP 查询（自动路由到 TiKV）
SELECT * FROM users WHERE id = 123;
-- 执行时间：<10ms

-- OLAP 查询（自动路由到 TiFlash）
SELECT region, COUNT(*), AVG(amount)
FROM orders
WHERE date >= '2024-01-01'
GROUP BY region;
-- 执行时间：<1s（列式存储 + 向量化）

-- 强制使用 TiFlash
SELECT /*+ READ_FROM_STORAGE(TIFLASH[orders]) */
  region, SUM(amount)
FROM orders
GROUP BY region;
```

**性能对比**

| 场景 | MySQL | TiDB（TiKV） | TiDB（TiFlash） |
|:---|:---|:---|:---|
| 点查询 | 1ms | 5ms | - |
| 事务写入 | 1000 TPS | 5000 TPS | - |
| 聚合查询 | 30s | 20s | 2s |
| 扩展性 | 垂直 | 水平 | 水平 |

---

##### TiDB 实战场景

**典型应用场景**

| 场景 | 业务描述 | 数据量 | 并发量 | 关键指标 | 技术要点 |
|:---|:---|:---|:---|:---|:---|
| **HTAP 分析** | 实时 OLTP + OLAP | TB 级 | 1万 TPS | 事务 + 分析 | TiKV + TiFlash |
| **分布式事务** | 跨分片事务 | TB 级 | 5000 TPS | 强一致性 | Percolator |
| **实时报表** | 业务报表、BI | TB 级 | 5000 QPS | 低延迟聚合 | TiFlash 列存储 |

---

**常见问题与解决方案**

##### 问题 1：HTAP 查询路由

**现象：**
```sql
-- OLAP 查询慢，影响 OLTP 性能
SELECT region, SUM(amount) FROM orders GROUP BY region;
```

**解决方案：**
```sql
-- 方案 1：自动路由到 TiFlash
-- TiDB 自动识别 OLAP 查询，路由到 TiFlash

-- 方案 2：强制使用 TiFlash
SELECT /*+ READ_FROM_STORAGE(TIFLASH[orders]) */
  region, SUM(amount)
FROM orders
GROUP BY region;

-- 方案 3：创建 TiFlash 副本
ALTER TABLE orders SET TIFLASH REPLICA 1;

-- 方案 4：资源隔离
-- 使用资源组隔离 OLTP 和 OLAP
CREATE RESOURCE GROUP oltp_group RU_PER_SEC = 1000;
CREATE RESOURCE GROUP olap_group RU_PER_SEC = 500;
```

**最佳实践：**
- ✅ 为 OLAP 表创建 TiFlash 副本
- ✅ 使用资源组隔离负载
- ✅ 监控 TiKV 和 TiFlash 负载
- ✅ 大查询使用 TiFlash

---

##### 问题 2：热点问题

**现象：**
```sql
-- 自增 ID 导致写入热点
CREATE TABLE orders (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  ...
);
-- 所有写入集中在最新的 Region
```

**解决方案：**
```sql
-- 方案 1：使用 SHARD_ROW_ID_BITS
CREATE TABLE orders (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  ...
) SHARD_ROW_ID_BITS = 4;
-- 将数据分散到 16 个 Region

-- 方案 2：使用 UUID
CREATE TABLE orders (
  id VARCHAR(36) PRIMARY KEY DEFAULT (UUID()),
  ...
);

-- 方案 3：使用 AUTO_RANDOM
CREATE TABLE orders (
  id BIGINT AUTO_RANDOM PRIMARY KEY,
  ...
);
-- TiDB 自动生成分散的 ID

-- 方案 4：预分裂 Region
SPLIT TABLE orders BETWEEN (0) AND (1000000) REGIONS 10;
```

**最佳实践：**
- ✅ 避免自增 ID（使用 AUTO_RANDOM）
- ✅ 使用 SHARD_ROW_ID_BITS 分散数据
- ✅ 预分裂 Region
- ✅ 监控热点 Region

---

##### 问题 3：扩容策略

**现象：**
```
场景：数据量增长，需要扩容
问题：如何在线扩容？
```

**解决方案：**
```bash
# 方案 1：添加 TiKV 节点（存储扩容）
# 1. 启动新 TiKV 节点
tiup cluster scale-out my-cluster scale-out.yaml

# 2. PD 自动均衡 Region
# 无需手动操作，PD 自动迁移数据

# 方案 2：添加 TiDB 节点（计算扩容）
tiup cluster scale-out my-cluster tidb-scale-out.yaml

# 方案 3：添加 TiFlash 节点（OLAP 扩容）
tiup cluster scale-out my-cluster tiflash-scale-out.yaml

# 方案 4：监控扩容进度
# Grafana 监控 Region 迁移进度
```

**最佳实践：**
- ✅ 使用 TiUP 工具在线扩容
- ✅ 避开业务高峰期
- ✅ 监控 Region 均衡进度
- ✅ 扩容后验证性能

---

#### CockroachDB

**全球分布式架构**

```
┌─────────────────────────────────────────────┐
│         美国西部（us-west）                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Node 1   │  │ Node 2   │  │ Node 3   │   │
│  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────┘
         ↓ 跨区域复制
┌─────────────────────────────────────────────┐
│         美国东部（us-east）                   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Node 4   │  │ Node 5   │  │ Node 6   │   │
│  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────┘
         ↓ 跨区域复制
┌─────────────────────────────────────────────┐
│         欧洲（eu-west）                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │ Node 7   │  │ Node 8   │  │ Node 9   │   │
│  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────┘

特点：
✅ 全球强一致性
✅ 自动故障转移
✅ 地理分区（Geo-Partitioning）
```

**分布式事务**

```sql
-- 全球分布式事务
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;  -- 美国
UPDATE accounts SET balance = balance + 100 WHERE id = 2;  -- 欧洲
COMMIT;

-- 使用 Raft 协议保证一致性
-- 跨区域延迟：50-200ms
```

**地理分区**

```sql
-- 按地理位置分区数据
ALTER TABLE users PARTITION BY LIST (region) (
  PARTITION us VALUES IN ('us-west', 'us-east'),
  PARTITION eu VALUES IN ('eu-west', 'eu-central'),
  PARTITION asia VALUES IN ('asia-east', 'asia-south')
);

-- 数据就近存储，降低延迟
-- 美国用户数据存储在美国节点
-- 欧洲用户数据存储在欧洲节点
```

---


## Big Data

以下表格列出了 Amazon EMR 支持的核心开源组件及其主要功能与典型使用场景：

| 组件                              | 功能                                                    | 使用场景                        |
|:------------------------------- |:----------------------------------------------------- |:--------------------------- |
| Apache Hadoop (YARN/HDFS)       | 提供分布式存储与资源管理，支持 MapReduce 作业                          | 批量数据处理、日志分析、大规模数据仓库         |
| Apache Spark                    | 基于内存的通用计算引擎，包含 Spark SQL、Spark Streaming、MLlib、GraphX | 快速批处理、交互式查询、实时流式计算、机器学习、图计算 |
| Apache Flink                    | 流式处理引擎，支持事件时间语义、一次性精确保障、回压控制                          | 高频实时分析、事件检测、复杂事件处理          |
| Apache Hive                     | 基于 SQL 的数据仓库和分析工具，底层执行 MapReduce                      | 大规模数据仓库、ETL 任务、定时报表、日志聚合    |
| Presto                          | 分布式 ANSI SQL 查询引擎，针对低延迟交互式查询优化                        | Ad hoc 分析、BI 报表查询、多数据源联合查询  |
| Apache HBase                    | 列族型 NoSQL 数据库，提供低延迟随机读写与列族聚簇存储                           | 实时访问大规模稀疏数据、物联网和用户画像场景      |
| Apache Hudi                     | 面向数据湖的增量数据管理框架，支持 CDC、写时合并                            | 记录级插入/更新/删除、数据隐私合规、流式摄取场景   |
| Apache Phoenix                  | 在 HBase 上提供 ACID 事务与二级索引的 SQL 层                       | 低延迟 SQL 查询、事务处理、实时 OLTP 场景  |
| EMR Studio / Jupyter / Zeppelin | 浏览器 IDE 与交互式笔记本，支持 Python、R、Scala、SparkSQL            | 数据探索、可视化、调试、协作式开发           |
| Hue                             | 面向 Hadoop 的 Web 界面，支持 Hive 查询、HDFS 文件管理、Pig 脚本        | 非技术用户自助查询、文件管理与脚本调试         |
| Apache Oozie                    | 支持有向无环图工作流调度与基于时间或事件的触发                               | ETL 管道编排、依赖控制、批量作业调度        |


#### AWS Redshift 与开源对应

Amazon Redshift 是一种全托管的大规模并行处理 (MPP) 云数据仓库，采用列式存储和大规模并行查询以实现高性能.
常见开源替代方案：

- Greenplum：基于 PostgreSQL 的开源 MPP 数据仓库，支持 SQL 查询和弹性并行处理，适合低成本自建场景.
- ClickHouse：高性能列式 OLAP 数据库，擅长实时分析和交互式查询，可作为 Redshift 的轻量级替代.

---
