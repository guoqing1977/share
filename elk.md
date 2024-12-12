# ELK 运维文档

## Logstash

### Monitoring API

Logstash 提供了以下 4 个 Monitoring API 用于查看节点级别的信息:

1. **Node Info API**: 用于查看节点的基本信息,可选参数有 `pipelines`、`os` 和 `jvm`。示例:

   ```bash
   curl 127.0.0.1:9600/_node/os,jvm
   ```

   返回结果包含 pipeline 的默认配置、OS 和 JVM 的基本信息。其中 `status` 字段展示了 Logstash 的当前健康状态。

2. **Plugins Info API**: 用于查看已安装插件的版本信息:

   ```bash
   curl 127.0.0.1:9600/_node/plugins
   ```

3. **Node Stats API**: 用于查看 Logstash 的运行时状态,可选参数有 `jvm`、`process`、`events`、`flow`、`pipelines` 和 `reloads`。这些参数可以提供 JVM 内存和 GC 情况、进程指标、事件处理情况等详细信息。

4. **Hot Threads API**: 用于查看 Logstash 的热点线程信息,可以确定哪些线程占用了较高的 CPU 时间。

   ```bash
   curl 127.0.0.1:9600/_node/hot_threads
   ```

### Logstash Exporter 指标

Logstash Exporter 会采集上述 Monitoring API 的指标,包括 `jvm`、`events`、`process` 和 `reloads` 等维度的信息。

### 插件管理

**离线安装插件**:

```bash
bin/logstash-plugin install file:///path/to/logstash-offline-plugins-8.6.2.zip
```

**更新插件**:

```bash
bin/logstash-plugin update                       # 更新所有插件
bin/logstash-plugin update logstash-input-github # 更新特定插件
```

**移除插件**:

```bash
bin/logstash-plugin remove logstash-output-kafka
```

**使用私有 Gem 库**:

Logstash 插件管理器默认连接 `http://rubygems.org` 仓库。可以通过修改 Gemfile 中的 `source` 行指向自己的私有仓库地址。

### 性能调优

Logstash 提供了以下 3 个参数来调优 pipeline 的性能:

1. `pipeline.workers`: 设置处理 filter 和 output 的线程数。可根据 CPU 利用率适当调大该值。
2. `pipeline.batch.size`: 设置单个 worker 线程在执行 filter 和 output 前采集的事件总数。通常 batch 越大,处理效率越高,但会增大内存开销。
3. `pipeline.batch.delay`: 该配置通常无需调整。

Logstash 中 inflight 事件的总数与 `pipeline.workers` 和 `pipeline.batch.size` 的配置有关。inflight 事件过多会导致 GC 和 CPU 出现突刺,合理的 inflight 事件场景下 GC 和 CPU 曲线会较为平滑。

### Troubleshooting

1. **如何校验 Logstash 配置文件**:

   ```bash
   $ logstash --config.test_and_exit -f <path_to_config_file>
   ```

   如果返回 `Configuration OK`，说明配置文件正确。

2. **Logstash 可能出现的问题**:

   - Logstash 比较耗内存,可以检查 JVM 内存利用率。
   - Logstash input 和 filter/output 处理效率不匹配,可以通过 `/_node/stats/pipelines` 接口获取各阶段的详细信息,适当调整 `pipeline.workers` 和 `pipeline.batch.size`。
   - Logstash 处理能力不足,可以根据 CPU 和内存使用情况调整上述两个参数。

3. **如何保证事件不丢失**:

   Logstash 默认使用内存队列来缓存各个阶段的事件。可以使用 `persistent queue` 机制防止事件丢失,它位于 input 和 filter 阶段之间。事件成功写入 PQ 后,input 就可以向事件源返回确认。

4. **是否可以保证消息处理顺序**:

   Logstash 默认不保证消息处理顺序,可能会出现乱序。可以启动单个 Logstash 实例并设置 `pipeline.ordered => true` 来保证顺序处理。

5. **Logstash 如何退出**:

   Logstash 收到 SIGTERM 信号后会: 1) 停止所有 input/filter/output 插件 2) 处理完所有未完成的事件 3) 结束进程。该过程可能会受input/filter/output 速度的影响。可以使用 `--pipeline.unsafe_shutdown` 参数强制退出,但可能会导致事件丢失。

## Elasticsearch (基于 ES 8)

### 安装事项

1. **内核参数设置**: 
   - `vm.max_map_count` 至少设置为 262144
   - 设置打开文件句柄数: `ulimit -n 65535`
   - 设置可创建线程数: `ulimit -u 4096`

2. **JVM 配置**:
   - 推荐使用 `/usr/share/elasticsearch/config/jvm.options.d` 来设置 JVM 参数
   - 不推荐使用 `ES_JAVA_OPTS` 环境变量

3. **Keystore 管理**:
   - 使用 `elasticsearch-keystore` 命令创建和管理密钥
   - 可以通过 `elasticsearch-keystore list` 和 `elasticsearch-keystore show` 查看内容

### 初始化重要配置

1. **路径配置**:
   - `path.data`: 保存索引数据和 data stream 数据
   - `path.logs`: 保存集群状态和操作数据

2. **集群名称**:
   一个 ES 集群由配置了相同 `cluster.name` 的节点组成,默认为 `elasticsearch`。

3. **节点名称**:
   通过 `node.name` 配置节点名称,默认为机器主机名。

4. **网络主机配置**:
   - `network.host`: 设置 ES 的绑定地址,默认为 `127.0.0.1` 和 `[::1]`
   - `http.port`: HTTP 客户端通信端口,默认 `9200-9300`
   - `transport.port`: 节点间通信端口,默认 `9300-9400`

5. **节点发现和 Master 节点选举**:
   - `discovery.seed_hosts`: 集群中 master-eligible 节点的地址列表,用于新节点加入集群
   - `cluster.initial_master_nodes`: 首次集群引导时使用的 master-eligible 节点列表

6. **JVM 配置**:
   - `Xms` 和 `Xmx` 建议不超过总内存的 50%
   - 设置 circuit breaker 的内存上限为 85% 节点内存

### 节点类型

1. **Master-eligible 节点**:
   - 角色为 `master`
   - 负责集群范围内的轻量工作,如创建/删除索引、探测节点健康等
   - 可被选举为 master 节点
   - 需要配置 `path.data` 目录保存集群元数据

2. **Voting-only 节点**:
   - 角色为 `voting_only`
   - 只参与 master 选举,不会成为 master

3. **Data 节点**:
   - 角色为 `data`
   - 负责数据相关的操作,如 CRUD、查找、聚合等
   - 当监测到 I/O、内存和 CPU 资源过载时,需要添加新的 Data 节点
   - 在多层架构中,还可以配置 `data_content`、`data_hot`、`data_warm`、`data_cold` 或 `data_frozen` 等角色

### 集群引导和管理

1. **集群引导**:
   - 通过 `discovery.seed_hosts` 配置已有集群中的 master-eligible 节点地址
   - 使用 `cluster.initial_master_nodes` 配置首次集群引导时的 master-eligible 节点

2. **Master 选举**:
   - Master 节点负责集群元数据的维护和更新
   - Master-eligible 节点通过 Zen Discovery 机制进行 master 选举

3. **集群健康状态**:
   - 红色(Red): 主分片不可用,需要人工介入修复
   - 黄色(Yellow): 副本分片不可用,集群可以提供服务但存在单点故障风险
   - 绿色(Green): 所有主分片和副本分片都可用

4. **集群状态发布**:
   - 集群状态变化后,会通过 Discovery 模块告知集群内其他节点
   - 各节点根据集群状态进行分片分配和路由

5. **增删节点**:
   - 添加节点: 配置 `discovery.seed_hosts` 即可
   - 移除 master-eligible 节点: 先转移 master 角色,然后从 `cluster.initial_master_nodes` 中移除

### 分片分配和路由

1. **分片分配设置**:
   - `cluster.routing.allocation.enable`: 控制分片分配的总开关
   - `cluster.routing.allocation.node_initial_primaries_recoveries`: 初次分配主分片时的并发数
   - `cluster.routing.allocation.node_concurrent_recoveries`: 节点间分片恢复的并发数

2. **分片均衡设置**:
   - `cluster.routing.rebalance.enable`: 控制分片 rebalance 的总开关
   - `cluster.routing.allocation.allow_rebalance`: 什么情况下允许 rebalance

3. **基于磁盘的分配配置**:
   - `cluster.routing.allocation.disk.threshold_enabled`: 是否基于磁盘使用率进行分片分配
   - `cluster.routing.allocation.disk.watermark.*`: 设置磁盘使用率的高低水位

4. **基于节点属性的分配**:
   - `node.attr.*`: 自定义节点属性
   - `index.routing.allocation.include/exclude/require.*`: 根据节点属性控制索引分片的分配

5. **分片限制**:
   - `cluster.max_shards_per_node`: 单个节点上的最大分片数
   - `cluster.routing.allocation.total_shards_per_node`: 单个节点上分片总数的软限制

### 读写流程

1. **写流程**:
   - Client 发送写请求至 coordinating node
   - coordinating node 确定目标分片,转发请求至对应 primary shard
   - primary shard 完成写操作,并将请求同步复制至副本分片

2. **Refresh 和 Flush**:
   - Refresh: 使新写入的文档可搜索,默认 1 秒刷新一次
   - Flush: 将内存缓冲区的数据刷新至磁盘,减少 Lucene segment 文件

3. **读流程**:
   - Client 发送读请求至 coordinating node
   - coordinating node 确定目标分片,并将请求转发至相应的 replica shard
   - 返回读取结果

### 故障处理

1. **故障类型**:
   - 单 shard 故障: 通过 replica 完成故障转移
   - 单 node 故障: 通过 rebalance 迁移分片至健康节点
   - 整个集群故障: 通过 snapshot/restore 机制恢复数据

2. **故障处理方案**:
   - 检查集群健康状态、节点状态、分片分配情况
   - 如主分片丢失,需要手动修复
   - 如节点下线,ES 会自动进行分片迁移和负载均衡
   - 如整个集群故障,可通过 snapshot 恢复数据

### 索引管理

1. **索引生命周期管理(ILM)**:
   - 定义 Lifecycle Policy 管理索引的生命周期
   - 包括 Hot/Warm/Cold/Frozen 等阶段

2. **Data Stream**:
   - 自动管理索引生命周期,无需手动创建索引
   - 支持滚动写入新数据

### 集群监控

1. **集群级监控指标**:
   - 集群健康状态、节点状态、分片分配情况
   - 集群 CPU、内存、I/O 等资源使用情况
   - 查询、索引等性能指标

2. **节点级监控指标**:
   - 节点 CPU、内存、磁盘使用情况
   - JVM 内存和 GC 情况
   - 线程池使用情况

3. **度量收集和展示**:
   - 使用 Prometheus 等监控系统收集 ES 指标
   - 在 Grafana 等可视化平台上展示监控大盘

### 集群运维

1. **集群重启**:
   - 遵循 rolling restart 的原则
   - 先停止数据节点,最后停止 master 节点

2. **索引运维**:
   - 列出集群中的所有索引
   - 查看分片分配情况,并进行手动 rebalance
   - 分裂(split)和收缩(shrink)索引

3. **集群快照管理**:
   - 创建快照仓库
   - 手动或定期自动创建快照
   - 恢复集群或指定索引

4. **安全与认证**:
   - 启用 x-pack security 模块
   - 配置用户、角色和权限
   - 开启 TLS 加密

5. **集群升级**:
   - 遵循 blue/green 部署的原则
   - 先升级 client 节点,再升级 data 节点,最后升级 master 节点

## Kibana

### 安装配置

1. **安装路径**:
   - Linux 下默认安装路径为 `/usr/share/kibana`
   - Windows 下默认安装路径为 `C:\Program Files\Kibana`

2. **配置文件**:
   - 主配置文件为 `config/kibana.yml`
   - 可以通过环境变量 `KIBANA_PATH_CONF` 指定配置文件目录

3. **与 Elasticsearch 集群的连接**:
   - 通过 `elasticsearch.hosts` 配置 Elasticsearch 节点地址
   - 如果 Elasticsearch 启用了认证,需要配置 `elasticsearch.username` 和 `elasticsearch.password`

4. **监听地址和端口**:
   - `server.host`: Kibana 服务的监听地址,默认为 `localhost`
   - `server.port`: Kibana 服务的监听端口,默认为 `5601`

### 基本操作

1. **Index Pattern 管理**:
   - 配置 Kibana 如何与 Elasticsearch 索引进行交互
   - 可通过 Management > Index Patterns 进行管理

2. **Discover 页面**:
   - 查看和分析 Elasticsearch 索引中的数据
   - 支持丰富的查询语法和聚合分析

3. **Visualize 页面**:
   - 创建各种可视化图表,如折线图、柱状图、饼图等
   - 支持在可视化中使用聚合查询

4. **Dashboard 页面**:
   - 将不同的可视化组件集成到仪表盘中
   - 可以保存和共享仪表盘

5. **Machine Learning 功能**:
   - 通过机器学习模型发现数据异常
   - 支持异常检测和预测分析

6. **Alerting 功能**:
   - 基于监控规则设置告警
   - 当满足告警条件时,Kibana 会发送通知

7. **Monitoring 功能**:
   - 监控 Elasticsearch 集群和 Kibana 自身的健康状况
   - 提供集群运行状态、资源使用情况等指标

### 安全与认证

1. **认证配置**:
   - 启用 `xpack.security.enabled` 选项
   - 配置 `elasticsearch.username` 和 `elasticsearch.password`

2. **角色与权限管理**:
   - 在 Kibana 中管理用户、角色和权限
   - 可以为不同用户分配不同的访问权限

3. **SSL/TLS 加密**:
   - 开启 `server.ssl.enabled` 选项
   - 配置 `server.ssl.certificate` 和 `server.ssl.key` 指定证书文件

4. **Audit Logging**:
   - 开启 `xpack.security.audit.enabled` 选项
   - 审计日志记录用户的各种操作行为

### 集成与扩展

1. **Reporting 功能**:
   - 将 Kibana 仪表盘以 PDF 或 PNG 格式导出
   - 通过 UI 界面或 API 方式生成报告

2. **Canvas 功能**:
   - 创建交互式的数据可视化页面
   - 支持自定义图表、图像和文本等元素

3. **APM 集成**:
   - 集成 Elastic APM 监控应用程序性能
   - 在 Kibana 中查看应用程序拓扑和性能指标

4. **Maps 功能**:
   - 在地图上可视化地理位置数据
   - 支持丰富的地图图层和交互功能

5. **infra 功能**:
   - 监控服务器、容器和Kubernetes集群的运行状况
   - 在 Kibana 中查看系统级别的指标和事件

总之,ELK 是一套强大的日志分析和监控解决方案,在实际应用过程中需要根据业务需求合理配置和管理这些组件,确保系统的稳定性和可观测性。
