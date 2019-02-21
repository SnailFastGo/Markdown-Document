# ClickHouse最佳实践



**ClickHouse是什么**

>ClickHouse（CK）是由俄罗斯Yandex开源的用于联机分析处理（OLAP）的列式数据库管理系统。
    
**ClickHouse的特点**

 - 列式存储
 - 分布式
 - MPP（Massively Parallel Processing）
 - 异步复制
 - 线性扩展
 - 数据最终一致
 - 超高性能
 - 不支持事务
 
**应用场景**
    
 - PB级存储
 - 百万级TPS
 - 以分区字段作为Where条件的Ad-hoc
 - 每次查询部分表的部分列
 - 聚合统计
 - 数据副本
 - 允许最终一致

**核心概念**

 - 表引擎  
    表引擎决定了数据在文件系统中的存储方式，常用的也是官方推荐的存储引擎是MergeTree系列，如果需要数据副本的话可以使用ReplicatedMergeTree系列，相当于MergeTree的副本版本。读取集群数据需要使用分布式表引擎Distribute。  
 - 表分区  
    表中的数据可以按照指定的字段分区存储，每个分区在文件系统中都是都以目录的形式存在。常用时间字段作为分区字段，数据量大的表可以按照小时分区，数据量小的表可以在按照天分区或者月分区，查询时，使用分区字段作为Where条件，可以有效的过滤掉大量非结果集数据。
 - 分片  
    一个分片就是一个CK实例，既可以单独对外提供服务，也可以加入到集群中，作为集群的一部分。
 - 集群  
    可以使用多个CK实例组成一个集群，并统一对外提供服务。

**集群配置**

>集群配置分为有副本的集群配置和无副本的集群配置，如下图所示：
![cluster_no_replica](https://github.com/SnailFastGo/Markdown-Document/blob/master/blob/pic/ck/cluster_no_replica.png)
![cluster_replica](https://github.com/SnailFastGo/Markdown-Document/blob/master/blob/pic/ck/cluster_replica.png)

>图中shard标签是CK实例的配置，shard标签上层是集群名字，shard标签里面的CK实例互为副本。

**读写方式**

- 创建本地表  
![create_local_table](https://github.com/SnailFastGo/Markdown-Document/blob/master/blob/pic/ck/create_local_table.png)
- 创建分布式表  
![create_distribute_table](https://github.com/SnailFastGo/Markdown-Document/blob/master/blob/pic/ck/create_distribute_table.png)
- 写本地表  
    数据写入时，可以由客户端控制数据分布，直接写入集群中CK实例的本地表。
- 写分布式表  
    数据写入时，可以先写入集群中的分布式表，再由分布式表将Insert语句分发到集群各个节点上执行，分布式表不存储实际数据。
- 数据副本  
    在集群配置中，shard标签里面配置的replica互为副本。可以将internal_replication设置成true，通过Zookeeper异步复制数据，相同ZK路径下的表会相互复制。
- 读分布式表  
使用Distribute表引擎作为集群的统一访问入口，当客户端查询分布式表时，CK会将查询分发到集群中各个节点上执行，并将各个节点的返回结果在分布式表所在节点上进行汇聚，将汇聚结果作为最终结果返回给客户端。

**跨中心透明访问**

>在实际应用中，往往会将某种类型的业务数据存储在多个CK集群中，当查询时，又要对客户端透明，好像所有数据存储在一个节点上。比如用户的通话记录，按照通话时用户所在地区存储到该地区的集群中，当查询某个用户所有的通话记录时，就需要同时查询多个集群中的数据。

>CK可以实现跨中心透明访问，将所有集群的分片都添加到一个CK实例中，并在该CK实例上创建分布式表，作为客户端查询的统一入口。当客户端查询该分布式表时，CK会将查询分发到所有分片上，并将各个分片的返回结果在分布式表所在节点上进行汇聚，将汇聚结果作为最终结果返回给客户端。

**负载均衡&高可用**
>使用使用一个CK实例的分布式表作为集群的统一访问入口，一旦该CK实例出现故障无法对外提供服务，那么整个集群的数据都将无法访问，这样会造成单点故障。或者当查询比较频繁时，所有查询请求都到一个CK实例上，也会造成该CK实例负载过高，拖慢查询的响应时间。

>可以使用多个CK实例的分布式表作为集群的访问入口，然后用Nginx代理这些实例，客户端直接访问Nginx，由Nginx将请求路由到代理的CK实例，这样既将请求分摊开，又避免了单点故障，同时实现了负载均衡和高可用。

**优化项**

 - 硬件配置  
 
 CPU核数 | CPU频率 | Mem | Harddisk | NIC
:-: | :-: | :-: | :-: | :-:
40core | 1200MHZ | 256G | 48T | 10GB| 


 - 软件版本  
 
 软件名称 | 版本号 
:-: | :-: 
ClickHouse | 1.1.54380 


 - 参数调整

 参数名称 | 默认值 | 调整后的值 | 参数说明 | 参数所在配置文件
:-: | :-: | :-: | :-: | :-:
max_memory_usage_for_all_queries | 0 | 200G |  单台服务器上所有查询的内存使用量，默认没有限制 | users.xml |  
max_memory_usage | 10G| 100G | 一个查询在单台服务器的最大内存使用量，默认是10GB | users.xml|  
max_execution_time | 0 | 300 | 单次查询耗时的最长时间，单位为秒。默认没有限制 | users.xml|  
distributed_product_mode | deny | local | 默认SQL中的子查询不允许使用分布式表，修改为local表示将子查询中对分布式表的查询转换为对应的本地表 | users.xml|  
background_pool_size | 16 | 32 | 后台用于merge的线程池大小 | users.xml|  
log_queries | 0 | 1 | system.query_log表的开关。默认值为0，不存在该表。修改为1，系统会自动创建system.query_log表，并记录每次query的日志信息 | users.xml|  
skip_unavailable_shards | 0 | 1 | 当通过分布式表查询时，遇到无效的shard是否跳过。默认值为0表示不跳过，抛异常。设置值为1表示跳过无效shard | users.xml|  
keep_alive_timeout | 10 | 600 | 服务端与客户端保持长连接的时长，单位为秒 | config.xml|  
max_concurrent_queries | 100 | 150 | 最大支持的Query数量 | config.xml|  
session_timeout_ms | 3000 | 120000 | ClickHouse服务和Zookeeper保持的会话时长，超过该时间Zookeeper还收到不ClickHouse的心跳信息，会将与ClickHouse的Session断开 | metrika.xml|  

 
 **监控**
>项目中使用Grafana监控ClickHouse的各项性能指标，以达到数据统计和提前预警的目的，如下图所示：
![mornitor_ck_fs](https://github.com/SnailFastGo/Markdown-Document/blob/master/blob/pic/ck/monitor_fs.png)

>使用Grafana监控CK有两种方式：

 >-  ClickHouse Exporter + Prometheus + Grafana
 

 >- Grafana自己的ClickHouse插件
