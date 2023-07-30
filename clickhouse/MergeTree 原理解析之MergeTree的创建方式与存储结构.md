# 简介
  表引擎是ClickHouse设计实现中的一大特色。表引擎决定了数据表拥有何种特性、数据以何种形式被存储以及如何被加载。
  根据官网介绍，Clickhouse 拥有MergeTreeFamily(合并树家族)，LogFamily(日志系列表引擎)，Integrations(集成类表引擎)和Special四大种类的。在这些所有种类的表引擎中，合并树（MergeTree）表引擎及其家族系列（*MergeTree）最为强大，在生产环境的绝大部分场景中，都会使用此系列的表引擎。**因为只有合并树系列的表引擎才支持主键索引、数据分区、数据副本和数据采样这些特性，同时也只有此系列的表引擎支持ALTER相关操作。**
  MergeTree表引擎具备了该系列其他表引擎共有的基本特征，是其他所有合并树变种的根基，所以吃透了MergeTree表引擎的原理，就能够掌握该系列引擎的精髓。**  **MergeTree家族中其他的表引擎则在MergeTree的基础之上各有所长，如ReplacingMergeTree表引擎具有删除重复数据的特性，而SummingMergeTree表引擎则会按照排序键自动聚合数据。
地位可以相当于  Mysql中的innodb 。
> MergeTree每次写入一批数据时，数据总会**以数据片段的形式写入磁盘，且数据片段不可修改。**为了避免片段过多，ClickHouse会通过后台线程，定期合并这些数据片段，属于相同分区的数据片段会被合成一个新的片段。
> **这种数据片段往复合并的特点，也正是合并树名称的由来**

# MergeTree的创建方式与存储结构
## 创建方式
创建MergeTree数据表完整的语法如下所示：
```sql
CREATE TABLE [IF NOT EXISTS] [db.]table_name [ON CLUSTER cluster]
(
    name1 [type1] [DEFAULT|MATERIALIZED|ALIAS expr1] [TTL expr1],
    name2 [type2] [DEFAULT|MATERIALIZED|ALIAS expr2] [TTL expr2],
    ...
    INDEX index_name1 expr1 TYPE type1(...) GRANULARITY value1,
    INDEX index_name2 expr2 TYPE type2(...) GRANULARITY value2
) ENGINE = MergeTree()
ORDER BY expr
[PARTITION BY expr]
[PRIMARY KEY expr]
[SAMPLE BY expr]
[TTL expr [DELETE|TO DISK 'xxx'|TO VOLUME 'xxx'], ...]
[SETTINGS name=value, ...]
```
可以看到这里将ENGINE参数声明为MergeTree()
接下来介绍几个重要的参数

- **PARTITION BY [选填]：分区键**，用于指定表数据以何种标准进行分区。
   - 分区键既可以是单个列字段，也可以通过元组的形式使 用多个列字段，同时它也支持使用列表达式。
   - 如果不声明分区键，则 ClickHouse会生成一个名为all的分区。
   - 合理使用数据分区，可以有效减少查询时数据文件的扫描范围。
- **ORDER BY [必填]：排序键**，用于指定在一个数据片段内， 数据以何种标准排序。
   - 默认情况下主键（PRIMARY KEY）与排序键相同。
   - 排序键既可以是单个列字段，例如ORDER BY CounterID，也可以通过元组的形式使用多个列字段，例如ORDER BY（CounterID,EventDate）。
- **PRIMARY KEY [选填]：主键**，声明后会依照主键字段生成一级索引，用于加速表查询。
   - 默认情况下，主键与排序键 (ORDER BY)相同，所以**通常直接使用ORDER BY代为指定主键，无须刻意通过PRIMARY KEY声明。**
   - 与其他数据库不同，MergeTree 主键允许存在重复数据（ReplacingMergeTree可以去重）。
- **SETTINGS：index_granularity [选填]：索引的粒度**，表示每间隔多少行数据才生成一条索引。
   - index_granularity对于MergeTree而言是一项非常重要的参数，MergeTree的索引在默认情况下，默认值为8192。
   - 8192是一个神奇的数字，在ClickHouse中大量数值参数都有它的影子，可以被其整除（例如最小压缩块大小min_compress_block_size:65536）。
   - 通常情况下并不需要修改此参数，但理解它的工作原理有助于我们更好地使用MergeTree
- **SETTINGS：index_granularity_bytes [选填]：自适应间隔大小，**根据每一批次写入数据的体量大小，动态划分间隔大小。
   - 默认为10M(10×1024×1024)，设置为0表示不启动自适应功能。
## 存储结构
MergeTree表引擎中的数据会按照分区目录的形式保存到磁盘之上，在 19.17.4.11版本中其完整的存储结构如下图所示
![image.png](https://cdn.nlark.com/yuque/0/2023/png/935856/1690723512411-952b310c-dbfe-4c52-85d3-7f40c649ce20.png#averageHue=%23fcfcfc&clientId=uce0afe75-17ba-4&from=paste&height=639&id=ud7492784&originHeight=1278&originWidth=1094&originalType=binary&ratio=2&rotation=0&showTitle=false&size=196568&status=done&style=none&taskId=u1855299c-b46d-4402-b112-b39dee40146&title=&width=547)
MergeTree在磁盘上的物理存储结构

- partition：分区目录，余下各类数据文件（primary.idx、[Column].mrk、[Column].bin等）都是以分区目录的形式被组织存放的
   - 属于相同分区的数据，最终会被合并到同一个分区目录，而不同分区的数据，永远不会被合并在一起。
- checksums.txt：校验文件，使用二进制格式存储。保存了余下各类文件(primary.idx、count.txt等)的size大小及size的哈希值，用于快速校验文件的完整性和正确性。
- columns.txt：列信息文件，使用明文格式存储。
- count.txt：计数文件，使用明文格式存储。用于记录当前数据分区目录下数据的总行数
- primary.idx：一级索引文件，使用二进制格式存储。
   - 用于存放稀疏索引，一张MergeTree表只能声明一次一级索引（通过ORDER BY或者PRIMARY KEY）。
   - 在数据查询的时能够更快排除主键条件范围之外的数据文件，从而有效减少数据扫描范围，加速查询速度。
- [Column].bin：数据文件，使用压缩格式存储，默认为LZ4压缩格式，用于存储某一列的数据。
   - 由于MergeTree采用列式存储，所以每一个列字段都拥有独立的.bin数据文件，并以列字段名称命名（例如CounterID.bin、EventDate.bin等）
- [Column].mrk：列字段标记文件，使用二进制格式存储。
   - 标记文件中保存了.bin文件中数据的偏移量信息。标记文件与稀疏索引对齐，又与.bin文件一一对应，所以MergeTree通过标记文件建立了primary.idx稀疏索引与.bin数据文件之间的映射关系。即首先通过稀疏索引（primary.idx）找到对应数据的偏移量信息（.mrk），再通过偏移量直接从.bin文件中读取数据。由于.mrk标记文件与.bin文件一一对应，所以MergeTree中的每个列字段都会拥有与其对应的.mrk标记文件（例如CounterID.mrk、EventDate.mrk等）
- [Column].mrk2：如果使用了自适应大小的索引间隔，则标记文件会以.mrk2命名。它的工作原理和作用与.mrk标记文件相同
- partition.dat与minmax_[Column].idx：如果使用了分区键，例如PARTITION BY EventTime，则会额外生成partition.dat与 minmax索引文件，它们均使用二进制格式存储。
- skp_idx_[Column].idx与skp_idx_[Column].mrk：如果在建表语句中声明了二级索引，则会额外生成相应的二级索引与标记文件，它们同样也使用二进制存储。二级索引在ClickHouse中又称跳数索引

# 参考文章
《ClickHouse原理解析与应用实践》之MergeTree原理解析
[MergeTree | ClickHouse Docs](https://clickhouse.com/docs/en/engines/table-engines/mergetree-family/mergetree#table_engine-mergetree-creating-a-table)
