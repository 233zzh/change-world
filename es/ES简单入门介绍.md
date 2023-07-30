# ElasticSearch简单介绍

## 概述

ES是一个开源分布式搜索分析引擎，以Java语言为基础(基于Lucene开发进行拓展)，支持分布式，可以水平拓展，使用RESTful接口可支持多种语言集成。目前GitHub、stack overflow等公司都在使用ES

### 主要功能

- 提供数据分布式存储以及集群管理能力
- 海量数据近实时搜索与分析(聚合)

### 应用场景

数据储存搜索、日志管理分析、安全分析等

集中基础应用的解决方案

（1）数据存储查询

由于事务等一些原因，通常都是使用MySQL进行存储，之后同步到es提供查询功能

![es提供存储功能解决方案](https://markdown-img-ct.oss-cn-beijing.aliyuncs.com/img/es%E6%8F%90%E4%BE%9B%E5%AD%98%E5%82%A8%E5%8A%9F%E8%83%BD%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88.png)

（2）日志收集

Beats收集日志文件，为了避免过大数据量造成的压力，Beats的数据传入到 Kafka 中，这样日志量激增也可以保护 Logstash 和 Elasticsearch 免受此类数据突发攻击，Kafka将日志传入到 Logstash 中，最终导入到 Elasticsearch中。

![Untitled Diagram-es日志收集解决方案](https://markdown-img-ct.oss-cn-beijing.aliyuncs.com/img/es%E6%97%A5%E5%BF%97%E6%94%B6%E9%9B%86%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88.png)

## 基础概念

### 文档(doc)

ES面向文档，文档是可搜索数据的最小单元，在ES中文档会被设置成json格式，每个文档都有自己的唯一id，每个文档在关系型数据库中相当于是一个表格的一行数据，例如订单表(es中的索引)的某个订单数据为一个文档。doc的元数据：

| 元数据   | 说明               |
| -------- | ------------------ |
| _index   | 文档所属的索引名   |
| _type    | 文档所属类型名     |
| _id      | 文档唯一id         |
| _source  | 文档的原始json数据 |
| _version | 文档的版本信息     |
| _score   | 相关性打分         |

### 索引(index)

ES中一类文档的集合，是文档的容器，索引中的数据分散在Shard(分片)上，(索引是一个逻辑概念，物理概念上数据分散存放在Shard上)，索引具有Mapping以及Setting定义，Mapping定义索引字段的存储类型、分词方式、是否存储等信息，Setting定义集群中对索引的定义信息，例如索引的分片数、副本数等等。

以下为索引movies的定义 `GET /movies`

```json
{
  "movies" : {
    "aliases" : { },
    "mappings" : {
      "properties" : {
        "@version" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "genre" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "id" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "title" : {
          "type" : "text",
          "fields" : {
            "keyword" : {
              "type" : "keyword",
              "ignore_above" : 256
            }
          }
        },
        "year" : {
          "type" : "long"
        }
      }
    },
    "settings" : {
      "index" : {
        "creation_date" : "1690469826044",
        "number_of_shards" : "1",
        "number_of_replicas" : "1",
        "uuid" : "VDSIhh1iQLqsX-dYKlhLnQ",
        "version" : {
          "created" : "7010099"
        },
        "provided_name" : "movies"
      }
    }
  }
}
```

### 节点(Node)

节点是ES的一个实例，为一个Java进程，每个节点有自己的名称以及UID。节点存储数据以及提供集群索引以及搜索功能，一个节点可以有不同的角色，ES节点常见的角色有：

| 角色                 | 说明                                                         |
| -------------------- | ------------------------------------------------------------ |
| Master Node          | 主节点                                                       |
| Master-eligible Node | 每个节点启动默认为Master-eligible Node，可以参与选主         |
| Data Node            | 保存数据的结点，负责保存分片数据                             |
| Coordinating Node    | 负责接受Client的请求，将请求分发到合适节点，最后进行汇集，每个节点默认都是Coordinating Node |

### 分片(Shards)

索引存放在节点上，用于水平拓展以及单节点空间限制等。索引数据被设计成存放在各个分片上，分片分为主分片和副本，副本是主分片的备份，一个分片存在于单个节点，一个节点可以包含多个分片。

> 主分片：解决数据水平拓展问题 副本：数据备份，解决数据高可用

```json
{
  "settings" :{
      "number_of_shards": 3,
      "number_of_replicas": 2
  },
  ...
}
```

分片数量以及副本数量在索引的setting中进行设置，(主分片数量)number_of_shards/(一个分片下副本数量)number_of_replicas，索引创建之后就不能进行更改了，后续更改只能通过reindex的方式

> **分片如何设定** a. 分片数量少：后续无法增加节点进行水平拓展，只能reindex；单分片数据量可能过大，数据重新分配耗时 b. 分片数量多：影响搜索结果相关性打分影响统计解决准确性；资源浪费，影响性能（sharding）

### 集群(Cluster)

多个安装了ES节点的服务集合，为一个ES集群，每个集群都自己的名称，是集群的唯一标识

集群中的节点健康状态有三种：

- green：健康状态，主副分配正常；
- yellow：警告状态，主分片全部正常分配，存在副本分片没有正常分配的情况；
- red：异常状态，存在主分片没有正常分配的情况。

### 存储结构

一个索引在ES集群中的存储示例

index:teachers_idx

number_of_shards=2/number_of_replicas=1

![es存储结构](https://markdown-img-ct.oss-cn-beijing.aliyuncs.com/img/es%E5%AD%98%E5%82%A8%E7%BB%93%E6%9E%84.png)