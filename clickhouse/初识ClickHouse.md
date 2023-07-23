# clickhouse 是什么？
ClickHouse是一款MPP架构的列式存储数据库，它是一个 OLAP 数据库同时也是关系型数据库。与传统的数据库的最大区别是其采用的是列式存储，这种结构在做统计分析，聚类分析的时候有天然的优势，。按列存储与按行存储相比，前者可以有效减少查询时所需扫描的数据量，这一点可以用一个示例简单说明。假设一张数据表A拥有60个字段A1～A60，以及1000行数据。现在需要查询前5个字段并进行数据分析，则可以用如下SQL实现：
```sql
SELECT A1，A2，A3，A4，A5 FROM A
```
如果数据按行存储，数据库首先会逐行扫描，并获取每行数据的所有60个字段，再从每一行数据中返回A1～A5这5个字段。如果数据按列存储，数据库可以直接获取A1～A5这5列的数据，从而避免了多余的数据扫描。  
# 为什么要叫 clickhouse？
Clickhouse初始设计目标是服务Yandex公司的一款名叫Yandex.Metrica的Web流量分析工具，该工具基于前方探针采集行为数据，然后进行一系列的数据分析。在采集数据的过程中，一次页面click（点击），会产生一个event（事件），即基于页面的点击事件流，面向数据仓库进行OLAP分析。所以ClickHouse的全称是Click Stream，Data WareHouse，简称ClickHouse。

# OLTP与 OLAP
上面说到了 OLAP 数据库，这里介绍一下OLTP 与 OLAP 的区别

- OLTP(Online transaction processing)
   - 在线/联机事务处理。典型的OLTP类操作都比较简单，主要是对数据库中的数据进行增删改查，操作主体一般是产品的用户。
   - 单次OLTP处理的数据量比较小，所涉及的表非常有限，一般仅几张表。
   - 是传统的关系型数据库，主要操作增删改查，强调事务一致性，比如银行系统、电商系统。常见的数据库：MySQL、PostgreSQL、Oracle等。
- OLAP(Online analytical processing)
   - 指联机分析处理。通过分析数据库中的数据来得出一些结论性的东西。比如数据报表，用户行为统计，不同维度的汇总分析结果等等。操作主体一般是运营、销售和市场等团队人员。
   - 为了从大量的数据中找出规律/总结性的东西，经常用到count()、sum()和avg()等聚合方法，用于了解现状并为将来的计划/决策提供数据支撑，所以对多张表的数据进行连接汇总非常普遍。
   - 为了表示跟OLTP的数据库（database）在数据量和复杂度上的不同，一般称OLAP的操作对象为数据仓库（data warehouse），简称数仓。数据库仓库中的数据，往往来源于多个数据库，以及相应的业务日志。
   - 常见的OLAP数据库有：Hive、Greenplum、HBase、ClickHouse等
- 还有一种尝试统一两大类型的HATP（Hybird Analyze Transaction Process）系统，例如TiDB、OceanBase等。
- 两者区别总结
|  | OLAP | OLTP |
| --- | --- | --- |
| 用途 | 数据仓库 | 事务数据库 |
| 数据容量 | 大，PB级 | 小，GB级，部分能达到TB级 |
| 事务能力 | 弱（或无） | 强 |
| 分析能力 | 强 | 弱，只能做简单的分析 |
| 并发数 | 低 | 高 |
| 数据质量 | 相对低 | 高 |
| 数据来源 | 各业务数据库 | 各业务系统 |

# MPP 架构
MPP，全称为Massively Parallel Processor，翻译过来就是大规模并行处理。MPP系统是由许多松耦合的处理单元组成的。每个处理单元内的CPU都有自己私有的资源，如总线，内存，硬盘等，且都有操作系统和管理数据库的实例副本。这种结构最大的特点在于不共享资源(share-nothing)。
![image.png](https://cdn.nlark.com/yuque/0/2023/png/935856/1690118242856-a2d56f49-79c0-4f94-9f71-037786907296.png#averageHue=%23fbe2c9&clientId=u26d23f03-24d3-4&from=paste&height=340&id=u89eb7ad0&originHeight=521&originWidth=701&originalType=binary&ratio=2&rotation=0&showTitle=false&size=24065&status=done&style=none&taskId=u15941a5f-e461-4c3d-914f-5aadee0377a&title=&width=457.5)
# ClickHouse特性

- 完备的DBMS（Database Management System，数据库管理系统）功能
   - 拥有完备的管理功能，如 DDL，DML，权限控制，数据备份与恢复和分布式管理
- 列式存储与数据压缩
   - 通过列式存储减少数据扫描范围
   - 通过数据压缩减少数据传输时的大小
- 向量化执行引擎
   - 向量化执行这项寄存器硬件层面的特性，为上层应用程序的性能带来了指数级的提升，要实现向量化执行，需要利用CPU的SIMD指令
   - SIMD的全称是Single Instruction Multiple Data，即用单条指令操作多条数据。现代计算机系统概念中，它是通过数据并行以提高性能的一种实现方式（其他的还有指令级并行和线程级并行），它的原理是在CPU寄存器层面实现数据的并行操作
- 关系模型与SQL查询
   - 使用关系模型描述数据并提供了传统数据库的概念（数据库、表、视图和函数等）并完全使用SQL作为查询语言（支持GROUP BY、ORDER BY、JOIN、IN等大部分标准SQL）
- 多样化的表引擎
   - ClickHouse像MySQL那样也将存储部分进行了抽象，把存储引擎作为一层独立的接口
   - 支持MergeTree，日志， 集成引擎和其他特殊引擎等四大类引擎
- 多线程与分布式
   - 多线程处理通过线程级并行的方式实现了性能的提升，线程级并行通常由更高层次的软件层面控制
- 多主架构
   - 采用Multi-Master多主架构，集群中的每个节点角色对等，客户端访问任意一个节点都能得到相同的效果，它天然规避了单点故障的问题，非常适合用于多数据中心、异地多活的场景。
- 在线查询
   - 90%的查询在1秒内返回
   - 即便是在复杂查询的场景下，它也能够做到极快响应，且无须对数据进行任何预处理加工
- 数据分片与分布式查询
   - 支持分片，而分片则依赖集群。每个集群由1到多个分片组成，而每个分片则对应了ClickHouse的1个服务节点。分片的数量上限取决于节点数量（1个分片只能对应1个服务节点）
   - 提供了本地表（Local Table）与分布式表（Distributed Table）的概念。一张本地表等同于一份数据的分片。而分布式表本身不存储任何数据，它是本地表的访问代理，其作用类似分库中间件。
# 适用与不适用场景

- 适用场景
   - 胜任各种数据分析类的场景，并且随着数据体量的增大，它的优势也会变得越为明显
   - 适用于商业智能领域（也就是我们所说的BI领域），除此之外，它也能够被广泛应用于广告流量、Web、App流量等
- 不适用场景
   - 不应该把它用于任何OLTP事务性操作的场景，因为其不支持事务
# 性能对比
直接使用 ClickHouse 官网提供的 6600w 数据集来做对比测试，在 MySQL、InfluxDB、ClickHouse 同样分配 4c16g 资源的情况下，ClickHouse 无论是导入速度、磁盘占用、查询性能都完全碾压 MySQL 和 InfluxDB，具体对比指标如以下表格：
![image.png](https://cdn.nlark.com/yuque/0/2023/png/935856/1690121788214-7c426ba2-1ebf-49c6-bf8b-36de0940087c.png#averageHue=%23f5f5f5&clientId=u26d23f03-24d3-4&from=paste&height=343&id=ub86ec9f5&originHeight=686&originWidth=1548&originalType=binary&ratio=2&rotation=0&showTitle=false&size=120412&status=done&style=none&taskId=u4ffd4375-02f6-4615-986e-4c2edd644be&title=&width=774)
性能对比结果参考自[记一次 ClickHouse 性能测试_clickhouse入库耗时_劼哥stone的博客-CSDN博客](https://blog.csdn.net/shi0090/article/details/126339159)
# 参考文章
《ClickHouse原理解析与应用实践》ClickHouse 的前世今生
[OLAP与OLTP的区别？_olap和oltp的区别_Gasing.C的博客-CSDN博客](https://blog.csdn.net/weixin_44087159/article/details/124477313)
[OLAP和OLTP的本质区别，一篇文章讲明白-olap oltp区别](https://www.51cto.com/article/743908.html)
[MPP架构_迷路剑客的博客-CSDN博客](https://blog.csdn.net/baichoufei90/article/details/84328666)
[ClickHouse官方文档](https://clickhouse.com/docs/zh)
[记一次 ClickHouse 性能测试_clickhouse入库耗时_劼哥stone的博客-CSDN博客](https://blog.csdn.net/shi0090/article/details/126339159)
