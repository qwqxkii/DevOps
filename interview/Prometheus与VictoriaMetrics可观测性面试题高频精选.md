# Prometheus 与 VictoriaMetrics 可观测性面试题高频精选

> 适用方向：SRE、DevOps、Kubernetes 平台、云原生运维开发、AI Infra／MLOps 平台  
> 目标级别：月薪 25K～40K、3～7 年经验的中高级岗位  
> 能力定位：Prometheus 基础必须扎实，VictoriaMetrics 与监控平台达到生产应用水平，可观测性具备系统设计意识  
> 排序依据：四份参考资料中的重复度、实际岗位常见度、连续追问深度和生产故障区分度。星级是经验分级，不是严格统计。  
> 推荐答法：**先说结论 → 解释数据链路或计算语义 → 给出验证指标/查询 → 说明生产取舍与防复发**。

---

## 一、先看结论：25K～40K 岗位到底考什么

这个薪资档不会满足于“会装 Prometheus、会导入 Grafana 面板”。真正有区分度的是：

1. 能讲清采集、写入、存储、查询、规则计算、告警通知的完整链路。
2. 能正确写出错误率、P95/P99、CPU、内存、可用率等常用 PromQL，并解释结果为什么可能失真。
3. 能治理高基数、查询慢、内存高、磁盘满、Remote Write 积压和告警风暴。
4. 能设计 Prometheus/Alertmanager 高可用，理解它们优先可用性而非强一致或 exactly-once。
5. 能说清 VictoriaMetrics 为什么引入、组件职责、数据分片、副本、去重、部分响应和扩缩容风险。
6. 能从用户体验和 SLO 设计监控，而不是只对 CPU、内存设固定阈值。
7. 能把 Metrics、Logs、Traces、Profiles 串成一条故障定位链路。
8. 能监控 Kubernetes、GPU 训练任务和在线推理服务，而不只是主机。

### 推荐复习优先级

| 优先级 | 范围 | 要求 |
|---|---|---|
| P0 | 第 1～15 题 | 必须能够独立口述并写出核心 PromQL |
| P1 | 第 16～30 题 | 决定能否达到中高级监控/SRE深度 |
| P1 | 第 31～45 题 | 必须按事故处理链路回答 |
| P2 | 第 46～50 题 | SRE、AI Infra 和平台岗加分项 |

---

# 第一部分：必问基础与核心原理（1～15）

## 1. 监控和可观测性有什么区别？★★★★★

**口述答案：**

监控主要回答“已知问题是否发生”，通过预先定义的指标、阈值和告警观察系统状态；可观测性强调仅根据系统输出，推断内部发生了什么，尤其用于分析事先没有预定义的未知问题。

监控是可观测性的一部分。一个平台有很多指标和大屏，不代表它可观测；如果数据没有统一标签、请求上下文和跨信号关联，发生复杂故障时仍然无法定位。

生产上通常组合：

- Metrics：低成本发现趋势、聚合和告警。
- Logs：查看离散事件和详细上下文。
- Traces：还原请求跨服务的因果路径。
- Profiles：定位代码级 CPU、内存等资源热点。

**常见追问：**“有了日志为什么还要指标？”——日志适合细节检索，指标适合低成本聚合、长期趋势和规则计算；两者不能互相完全替代。

---

## 2. 四个黄金信号、RED 和 USE 分别是什么？★★★★★

**口述答案：**

- 四个黄金信号：Latency、Traffic、Errors、Saturation。
- RED：Rate、Errors、Duration，适合面向请求的在线服务。
- USE：Utilization、Saturation、Errors，适合 CPU、内存、磁盘、网络等资源。

实践中先监控用户路径和 SLI，再下钻依赖与资源。例如支付服务先看请求量、错误率、P99 和成功率，再看连接池、线程池、数据库、CPU 和磁盘，而不是一开始只盯主机负载。

**易错点：**CPU 使用率高不是必然故障；若没有排队、延迟上升和错误，可能只是资源被有效利用。

---

## 3. Prometheus 的核心组件和数据链路是什么？★★★★★

**口述答案：**

Prometheus Server 通过静态配置或服务发现获得目标，按周期 Pull `/metrics`；抓取前使用 `relabel_configs` 处理目标，抓取后使用 `metric_relabel_configs` 处理样本；样本进入本地 TSDB，同时可通过 Remote Write 发送至远端存储。

Prometheus 周期性执行 Recording Rules 和 Alerting Rules。告警状态由 Prometheus 计算，发送给 Alertmanager；Alertmanager 负责分组、路由、抑制、静默、去重和通知。Grafana 通过查询 API 获取数据并展示。

**一句话链路：**

> 服务发现 → 拉取 → Relabel → TSDB/Remote Write → PromQL/规则计算 → Alertmanager → 通知渠道。

**易错点：**Alertmanager 不负责判断 CPU 是否超阈值；判断条件由 Prometheus 或 `vmalert` 执行。

---

## 4. Prometheus 的数据模型是什么？什么是时间序列？★★★★★

**口述答案：**

一条时间序列由指标名和完整标签集合唯一确定：

```text
http_requests_total{service="order",method="GET",code="200"}
```

只要任意标签值变化，就会产生新序列。样本由时间戳和值组成。Prometheus 的灵活性来自多维标签，主要风险也来自标签组合数爆炸。

**高基数估算：**

如果一个指标包含 100 个服务、20 个实例、10 个接口、8 个状态码，理论上可达到：

```text
100 × 20 × 10 × 8 = 160000 条序列
```

若再加入 `user_id`、完整 URL、trace_id，序列数可能失控。

---

## 5. Counter、Gauge、Histogram、Summary 如何选择？★★★★★

**口述答案：**

- Counter：只增不减，进程重启可归零；用于请求数、错误数、处理字节数，通常配合 `rate` 或 `increase`。
- Gauge：可增可减；用于温度、队列长度、并发数、当前内存。
- Histogram：服务端按 bucket 累计，产生 `_bucket/_sum/_count`；可跨实例聚合后计算分位数。
- Summary：客户端直接计算 quantile；分位数通常不能正确跨实例聚合，计算成本在客户端。

延迟、大小等通常优先 Histogram。bucket 要围绕 SLO 设计，例如 SLO 为 300ms，应在 300ms 附近设置有意义的边界。

**易错点：**不要平均多个实例的 Summary quantile，那在统计上通常没有意义。

---

## 6. Prometheus 为什么默认使用 Pull？Pushgateway 适合什么场景？★★★★★

**口述答案：**

Pull 让采集端控制抓取频率、超时和目标状态，便于服务发现，也能通过 `up` 区分目标可达性。它的代价是跨网络边界、短生命周期任务和大规模采集需要额外设计。

Pushgateway 主要适合**服务级、短生命周期批处理任务**，例如执行时间短于抓取周期的 CronJob。它不应成为所有长期服务的默认入口，因为：

- 指标生命周期与原进程脱钩，任务消失后旧序列可能仍存在。
- Pushgateway 成为瓶颈和单点。
- 失去 Prometheus 直接通过 `up` 观察实例健康的能力。
- 机器级批任务指标可能因 `instance` 等标签难以管理。

长期服务通常仍暴露端点供拉取；跨网络可使用 vmagent、Prometheus Agent 或 OpenTelemetry Collector 转发。

---

## 7. `relabel_configs` 和 `metric_relabel_configs` 有什么区别？★★★★★

**口述答案：**

- `relabel_configs` 在抓取前处理 target labels，可决定抓谁、地址和路径是什么，也可丢弃目标。
- `metric_relabel_configs` 在抓取后、写入前处理样本，可丢弃某些指标或标签。
- `write_relabel_configs` 在 Remote Write 发送前处理要发往远端的数据。

如果希望减少无用目标带来的网络和抓取开销，应在 target relabel 阶段过滤；如果目标必须抓，但部分指标无用，则在 metric relabel 阶段丢弃。

**常见追问：**丢弃 `__name__` 匹配的大量无用指标可以降低存储和序列数，但抓取网络流量已经发生。

---

## 8. `rate`、`irate`、`increase` 有什么区别？★★★★★

**口述答案：**

- `rate(counter[5m])`：用区间内样本估算每秒速率，能处理 Counter reset，适合告警和趋势图。
- `irate(counter[5m])`：只根据最后两个样本计算瞬时速率，波动大，适合观察短时尖峰，不适合大多数告警。
- `increase(counter[1h])`：估算区间内增长量，本质上相当于 `rate × 区间秒数`，适合表达一段时间发生多少次。

窗口通常至少覆盖多个抓取点，常用经验是大于等于抓取间隔的 4 倍；窗口太短会因抓取抖动或一次失败出现空值。

```promql
sum by (service) (
  rate(http_requests_total{code=~"5.."}[5m])
)
/
sum by (service) (
  rate(http_requests_total[5m])
)
```

**易错点：**先 `rate` 再聚合，才能正确识别各序列 reset：

```promql
sum by (service) (rate(http_requests_total[5m]))
```

---

## 9. PromQL 的 Instant Vector、Range Vector、Scalar 是什么？★★★★☆

**口述答案：**

- Instant Vector：每条序列在某个时刻的一个样本，例如 `up`。
- Range Vector：每条序列在一段时间内的一组样本，例如 `http_requests_total[5m]`。
- Scalar：单个数值。
- String：存在但实际使用较少。

`rate`、`increase` 等函数接收 Range Vector，聚合操作符通常处理 Instant Vector。Grafana 范围查询会在多个时间点重复执行瞬时表达式，从而画出曲线。

---

## 10. PromQL 中 `by`、`without` 和向量匹配如何理解？★★★★★

**口述答案：**

`sum by (service)` 保留 `service` 维度；`sum without (instance,pod)` 删除指定维度并保留其他维度。二元运算时，左右两边必须按标签正确匹配，否则会得到空结果或 many-to-many 错误。

```promql
sum by (service) (rate(http_requests_total{code=~"5.."}[5m]))
/
sum by (service) (rate(http_requests_total[5m]))
```

当一侧比另一侧多维度时，使用 `on(...)`／`ignoring(...)` 指定匹配标签，并在明确一对多关系时使用 `group_left` 或 `group_right`。

**易错点：**`group_left` 不是“修复查询的万能开关”；必须先证明关系确实是一对多，避免重复或错误放大。

---

## 11. 如何正确计算 P95/P99？★★★★★

经典 Histogram 示例：

```promql
histogram_quantile(
  0.99,
  sum by (le, service) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

先对每个 bucket 的 Counter 求 `rate`，再保留 `le` 聚合，最后计算分位数。结果是基于 bucket 分布插值的估计值，不是精确排序结果；bucket 设计不合理时误差会很大。

平均延迟为：

```promql
sum by (service) (rate(http_request_duration_seconds_sum[5m]))
/
sum by (service) (rate(http_request_duration_seconds_count[5m]))
```

**易错点：**P99 不保证一定大于平均值，极端长尾可能把平均值拉得高于 P99。

---

## 12. Prometheus TSDB 的 WAL、Head、Block 和 Compaction 是什么？★★★★★

**口述答案：**

新样本先进入 Head，同时写 WAL 保证崩溃后可重放；Head 中的 chunk 会映射到磁盘。初始持久块通常覆盖 2 小时，包含 chunks、index、meta 和可能的 tombstones，后台再把小块压成更大块。

WAL 是未压缩成正常 block 的近期原始数据，空间和恢复时间都需要监控。Prometheus 本地存储不提供内建集群复制，不建议使用 NFS/EFS 这类非受支持文件系统作为 TSDB 数据目录。

**常见追问：**删掉 WAL 能不能修复启动？——只能作为最后手段，会丢失 WAL 覆盖的近期数据；先备份目录并确认损坏范围。

---

## 13. Recording Rule 和 Alerting Rule 有什么区别？★★★★☆

**口述答案：**

- Recording Rule：定期计算表达式，把结果写成新时间序列，用于加速大盘、复用复杂查询和统一口径。
- Alerting Rule：定期计算条件并维护 Pending/Firing 状态，将告警发送给 Alertmanager。

规则组内通常按顺序执行，同一组的规则共享评估周期；复杂大查询应拆分和预计算，但 Recording Rule 会增加序列与存储，命名和标签必须治理。

规则上线前应执行：

```bash
promtool check rules rules.yml
promtool test rules tests.yml
```

---

## 14. 告警规则中的 `for` 和 `keep_firing_for` 是什么？★★★★★

**口述答案：**

表达式首次为真时告警进入 Pending；持续满足 `for` 才进入 Firing，主要用于过滤短暂抖动。`keep_firing_for` 让表达式恢复后继续保持 Firing 一段时间，可减少数据短暂缺失或抖动导致的反复恢复/触发。

告警总延迟至少包含：采集间隔 + 规则评估等待 + `for` + Alertmanager 的 `group_wait` + 通知渠道耗时。

**易错点：**`for: 5m` 不是“五分钟检查一次”，而是条件需连续成立五分钟；规则评估间隔是另一个配置。

---

## 15. Alertmanager 的 grouping、inhibition、silence、deduplication 分别是什么？★★★★★

**口述答案：**

- Grouping：把同类告警合并成一条通知，例如按 `cluster/namespace/alertname` 分组。
- Inhibition：某个源告警存在时，抑制相关目标告警，例如集群网络故障时抑制大量节点失联告警。
- Silence：人工按标签和时间窗口静默，常用于已知维护。
- Deduplication：根据告警标签集合与通知日志避免重复通知。

`group_wait` 等待同组首批告警；`group_interval` 控制同组新增/变化后的再次通知；`repeat_interval` 控制持续未恢复告警的重复提醒。

**易错点：**静默不会让 Prometheus 停止计算告警；它只影响 Alertmanager 通知。

---

# 第二部分：架构、VictoriaMetrics 与告警治理（16～30）

## 16. Prometheus 如何做高可用和水平扩展？★★★★★

**口述答案：**

高可用通常让两个 Prometheus 副本以 active-active 方式抓取相同目标并独立计算规则，而不是依赖主备选举。两者把告警发到 Alertmanager 集群；查询与长期存储可接入 VictoriaMetrics、Thanos、Mimir 等。

扩展方式要按瓶颈选择：

- 按集群、地域、业务或目标哈希做采集分片。
- Federation 汇总少量聚合指标，适合分层监控，不等同通用远程存储。
- Remote Write 将数据写入可扩展后端。
- 全局查询层聚合多个数据源并处理 HA 副本去重。

**易错点：**Prometheus 本地 TSDB 没有原生分布式复制，但整体监控系统仍可通过双采集、远程存储和查询层实现高可用。

---

## 17. Alertmanager 集群如何实现 HA？为什么仍可能重复通知？★★★★★

Alertmanager 节点通过 gossip 同步 Silence 和 Notification Log，实例独立接收、路由和发送通知。设计目标是 at-least-once 和 fail-open：网络分区时宁可重复，也不要漏掉关键告警。

Prometheus 应配置所有 Alertmanager 实例，而不是只把告警交给一个普通负载均衡入口。网络分区、gossip 未收敛、时钟偏差或通知日志同步异常都会造成重复通知。

**回答加分点：**Alertmanager 不是 Raft 集群，没有 leader 和 quorum；奇数节点不具有共识集群那种特殊意义。

---

## 18. 为什么选 VictoriaMetrics？它与 Prometheus 是什么关系？★★★★★

**口述答案：**

Prometheus 是采集、规则、查询和本地 TSDB 一体化的监控系统；VictoriaMetrics 更偏高性能、可扩展的时序数据库与监控组件栈，兼容 Prometheus 抓取、Remote Write API、查询 API 和大部分 PromQL 使用方式。

常见引入原因：

- 需要更长保留周期、更高写入量或统一多集群查询。
- Prometheus 本地存储达到单机容量或运维边界。
- 希望采集、规则计算、查询和存储分别扩缩容。
- 需要更好的压缩比、较低资源开销或多租户能力。

**选型不能只说性能：**还要比较团队经验、对象存储诉求、生态兼容、容灾方式、查询语义差异、成本和运维复杂度。小规模场景优先考虑 VictoriaMetrics 单机版或单个 Prometheus，避免为了“分布式”过度设计。

---

## 19. VictoriaMetrics 单机版和集群版如何选择？★★★★★

单机版是一个进程完成写入、存储和查询，部署简单、资源效率高，也可以通过多个独立实例和上游复制实现 HA。集群版拆成 `vminsert`、`vmstorage`、`vmselect`，可独立扩展和多租户，更适合超大规模或不同读写资源需要独立伸缩的场景。

官方文档明确建议：写入速率低于每秒一百万数据点时优先评估单机版，因为更容易配置和运维；这个数字是选型提示，不是机械红线，最终要用实际活跃序列、写入速率、查询负载、保留周期和故障域压测决定。

---

## 20. `vminsert`、`vmstorage`、`vmselect` 分别做什么？★★★★★

- `vminsert`：接收写入，按指标名和完整标签的一致性哈希将数据分散到 `vmstorage`。
- `vmstorage`：持久化原始数据，并按标签过滤与时间范围返回数据。
- `vmselect`：接收查询，向所配置的 `vmstorage` 获取数据并完成查询处理与合并。

三者可独立扩缩容。`vmstorage` 是有状态节点；它们采用 shared-nothing 架构，彼此不通信也不共享磁盘。`vminsert` 和 `vmselect` 前通常放置 vmauth 或负载均衡。

**易错点：**不能把 VictoriaMetrics 集群说成“所有组件都无状态”。

---

## 21. `vmagent`、`vmalert`、`vmauth` 分别做什么？★★★★★

- `vmagent`：抓取 Prometheus 目标或接收多种写入协议，进行 relabel、缓冲、分片/复制并 Remote Write 到后端；可作为边缘采集代理。
- `vmalert`：从数据源查询并执行 Recording/Alerting Rules，把记录结果写回远端，把告警发送给 Alertmanager。
- `vmauth`：位于入口进行认证、授权和基于 URL/用户的路由，也可在多个后端间转发请求。

**核心纠错：**`vmalert` 不替代 Alertmanager，它替代的是规则评估部分；告警通知的分组、抑制、静默和路由仍由 Alertmanager 完成。`vmagent` 也不只是“等于 Remote Write”，它还可负责抓取、协议接入、磁盘缓冲、复制和流式聚合。

---

## 22. VictoriaMetrics 如何实现分片、副本和去重？★★★★★

默认情况下，`vminsert` 按序列哈希把数据分散到不同 `vmstorage`，这是分片，不等于复制。配置 replication factor 后，同一数据会写入多个不同存储节点，提高数据耐久性，但会增加写放大、磁盘与网络开销。

如果两个 vmagent 或 Prometheus HA 副本采集相同目标，后端可能收到重复样本。应配置稳定一致的标签和正确的去重窗口，确保抓取间隔匹配；去重不是任意数据清洗，时间戳或标签不一致时可能无法按预期合并。

**生产取舍：**副本保护硬件/节点故障，跨 AZ 容灾更适合独立集群 + vmagent 向多个目标复制，避免把一个集群的强耦合组件横跨高延迟、易抖动链路。

---

## 23. VictoriaMetrics 的部分响应是什么？可用性和一致性如何取舍？★★★★☆

当部分 `vmstorage` 不可用时，`vmselect` 默认仍可返回从健康节点获得的数据，并标记 `isPartial: true`。这是优先可用性的设计，但历史查询可能缺失部分序列。

若业务要求宁可失败也不能返回不完整数据，可以启用拒绝部分响应的配置或查询参数。配置 replication factor 后，`vmselect` 可在不超过副本容忍范围时把响应视为完整。

**面试回答重点：**大盘可接受带标识的部分结果，计费、审计或严格 SLO 报表可能更应失败；需要按查询用途决定，而不是全局一刀切。

---

## 24. MetricsQL 和 PromQL 有什么关系？★★★★☆

MetricsQL 以 PromQL 兼容为目标，并增加了额外函数、语法和对缺失点、窗口推断等行为的改进。大多数 PromQL 可以直接运行，但不能声称 100% 语义完全相同。

迁移时重点验证：

- `rate`、`increase` 和窗口推断结果。
- 空结果、staleness 与 lookback 行为。
- Histogram、标签匹配和 subquery。
- Recording/Alerting Rules 的最终结果。
- Grafana 中依赖 `$__rate_interval` 的查询。

迁移应使用同一时间范围和输入数据做双读对比，尤其是告警表达式和 SLO 报表。

---

## 25. Remote Write 的链路和可靠性如何？★★★★★

Prometheus 的每个 Remote Write 目标拥有队列，从 WAL 读取样本，分配给多个 shard 的内存队列，再批量发送到远端；失败会退避重试并自动调整分片数。

重点监控：待发送样本、失败/重试、发送速率、最旧未发送样本时间、shard 数、远端延迟和返回码。Remote Write 会增加 CPU、网络和内存，尤其是序列 churn 较高时。

如果远端持续不可用超过 WAL 可覆盖时间，未发送数据仍可能丢失。`vmagent` 可为每个远端配置磁盘缓冲，但磁盘也必须做容量和水位告警。

---

## 26. 如何做 Prometheus/VictoriaMetrics 容量规划？★★★★★

至少收集这些输入：

- active series 数量与增长率。
- 每秒写入样本数：目标数 × 每目标序列数 ÷ 抓取间隔。
- 标签 churn、新序列创建率。
- 保留时间、压缩率和副本因子。
- 查询 QPS、时间范围、并发数和重查询比例。
- Recording Rules 新增序列与 Remote Write 写放大。
- 故障时剩余节点需要承接的额外负载。

Prometheus 本地磁盘可粗估：

```text
保留秒数 × 每秒样本数 × 每样本字节数 + WAL/Head/索引/安全余量
```

官方给出的常见压缩范围约 1～2 byte/sample，但不能用它替代压测。生产应保留磁盘水位余量，Prometheus 使用基于大小的 retention 时通常不要吃满整个卷。

---

## 27. 什么是高基数和高 churn？如何治理？★★★★★

高基数是同一指标/标签组合产生大量活跃序列；高 churn 是序列频繁创建和消失。Kubernetes 中 Pod UID、容器 ID、临时任务和滚动发布很容易制造 churn。

治理顺序：

1. 找出序列最多的 metric、label pair、tenant 和抓取 job。
2. 禁止 `user_id`、trace_id、完整 URL、随机 ID、错误原文等无界标签。
3. 把 URL 归一化为 route template，把错误归类为有限 error code。
4. 在应用埋点、relabel 或采集代理处尽早丢弃无用维度。
5. 为租户设置写入/序列限制并监控增长率。
6. 通过 Recording Rule 聚合查询热点，但不要把原始高基数无限保留。

**易错点：**增加磁盘只能缓解容量，无法解决索引、内存、查询和 churn 成本。

---

## 28. Kubernetes 中 node-exporter、cAdvisor、kube-state-metrics 各负责什么？★★★★★

- node-exporter：节点 OS 和硬件资源，如 CPU、内存、磁盘、网络、文件系统。
- cAdvisor/kubelet 指标：容器和 Pod 的 CPU、内存、网络、文件系统使用情况。
- kube-state-metrics：读取 Kubernetes API 对象并暴露其声明/状态，如 Deployment 副本、Pod phase、资源 requests/limits；不采集容器真实资源使用量。

还要监控 API Server、etcd、Scheduler、Controller Manager、kubelet、CoreDNS、CNI，以及应用自身 RED 指标。

**易错点：**`kube_pod_container_resource_requests` 是资源声明，不是实际使用量。

---

## 29. 如何设计一条“有用”的告警？★★★★★

有用告警应满足：

- 对用户、SLO、数据安全或容量风险有明确影响。
- 收到的人能够采取动作，严重级别与通知渠道匹配。
- 包含环境、集群、服务、影响、当前值、Dashboard 和 Runbook。
- 有持续时间和恢复条件，避免抖动。
- 能通过分组、抑制和依赖关系避免一因多报。
- 有负责人、定期回顾命中率、误报率、重复率和处理结果。

页面告警优先围绕症状，例如错误预算快速燃烧、成功率下降；CPU 高等原因类指标更适合作为下钻或工单告警，除非已经证明其会快速造成用户影响。

---

## 30. SLI、SLO、SLA、Error Budget 和 Burn Rate 是什么？★★★★★

- SLI：实际测量值，如成功请求占比、P99 小于阈值的请求占比。
- SLO：内部可靠性目标，如 30 天可用率 99.9%。
- SLA：对外承诺及可能的赔偿条款。
- Error Budget：允许失败的比例，99.9% SLO 对应 0.1% 预算。
- Burn Rate：错误预算被消耗的速度；1 倍表示按当前速度恰好在窗口末用完。

成熟告警使用多窗口、多燃烧率：短窗口高 burn rate 发现急性故障，长窗口较低 burn rate 发现慢性退化，同时降低瞬时噪音。

---

# 第三部分：生产故障场景题（31～45）

> 统一回答顺序：**现象 → 止损 → 定位 → 根因 → 修复 → 防复发**。  
> 面试官在意的不只是命令，而是你如何缩小范围、保护用户和验证恢复。

## 31. Prometheus target 显示 Down，怎么排查？★★★★★

1. **现象确认：**看 Targets 页/API 的 `lastError`、状态码、超时和最后成功时间，确认单点、单 job 还是全局。
2. **链路分层：**服务发现是否发现 target；relabel 后地址是否正确；DNS、路由、NetworkPolicy、TLS/RBAC、端口和 `/metrics` 是否正常。
3. **从采集端验证：**在 Prometheus 或 vmagent 所在网络命名空间中执行 DNS、TCP、HTTP/TLS 请求，避免只在自己电脑测试。
4. **目标侧检查：**Exporter 进程、监听地址、认证、响应耗时、指标大小及日志。
5. **修复与验证：**观察 `up`、scrape duration、samples scraped 和业务指标恢复。
6. **防复发：**统一 ServiceMonitor、证书和 NetworkPolicy 模板，对抓取超时、发现异常和 exporter 自身建立告警。

---

## 32. 大盘突然出现数据断点，但业务正常，怎么定位？★★★★★

先判断是采集缺失、写入缺失、查询错误还是可视化问题：

- 检查 `up`、抓取错误、样本数量和时间同步。
- 检查 Prometheus 重启、WAL replay、磁盘、Remote Write pending/failures。
- 在后端直接查询原始序列，排除 Grafana datasource、变量、step 和时间范围。
- 检查 relabel、服务发现、标签变化造成“旧序列消失、新序列出现”。
- VictoriaMetrics 查询检查是否为 partial response、某个 vmstorage 不可用。

短时缺点不要通过“填 0”掩盖；缺失和真实的 0 在告警语义上不同。

---

## 33. Prometheus 内存持续上涨或 OOM，怎么排查？★★★★★

**止损：**限制重查询并发、暂停异常大盘/规则、临时增加资源；避免立刻删除数据目录。

**定位：**

- active series、Head series、每秒新增序列和 churn 是否增加。
- 某次发布是否加入无界 label 或抓取了巨大 metrics endpoint。
- Remote Write series cache 和 pending 是否增长。
- 查询并发、查询时间范围、regex、join、subquery 是否异常。
- rule evaluation duration 和失败。
- 使用 TSDB status、query log、运行时指标和 pprof 确认内存归属。

**修复：**删除高基数标签/指标、分片采集、优化规则与大盘、限制查询、合理缩短 retention 或迁移远端存储。

**防复发：**对序列数、churn、采样率、查询成本和租户配额做预算及告警。

---

## 34. PromQL/Grafana 查询很慢，怎么排查？★★★★★

先确认是数据源慢、网络慢还是 Grafana 渲染慢。后端关注查询耗时、并发、扫描序列/样本、缓存、CPU、内存、磁盘 IO 和部分节点状态。

常见问题：

- 时间范围过大、step 过小。
- 宽泛正则或负匹配扫描大量序列。
- 没先过滤就做高基数 `group by`、join、subquery。
- Dashboard 同时发出大量重复查询。
- 规则和交互查询抢资源。
- vmselect/vmstorage 资源不足或某存储节点变慢。

修复可使用 Recording Rule、缩小标签与时间范围、调整 step、限制并发/最大样本、拆分查询读路径、缓存热点，并从源头治理高基数。

---

## 35. Prometheus 磁盘快满或已经满了，怎么处理？★★★★★

**止损：**确认文件系统和 inode，释放与 TSDB 无关的安全空间；若服务仍运行，降低 retention size/time 并等待后台清理，同时控制异常写入。

**定位：**区分 block、WAL、chunks_head、日志还是临时文件增长；检查样本率、序列数、WAL/Remote Write 积压、compaction 失败和 tombstone。

**注意：**不能在运行中随意 `rm` block/WAL。严重损坏时先停止实例、备份目录，再根据官方恢复流程处理；删除 WAL 会损失近期数据。

**防复发：**按实际样本率估算容量，设置 size retention 和 15%～20% 以上余量，监控磁盘预测耗尽时间、compaction 与 WAL 增速。

---

## 36. Remote Write 积压，如何处理？★★★★★

看远端返回码和延迟、pending samples、发送/失败/重试速率、shard 数、WAL 时间跨度、Prometheus CPU/内存/网络以及远端入口容量。

- 4xx 通常先修认证、租户、标签限制或协议问题，盲目重试无效。
- 429/5xx/超时通常是远端过载或网络问题，先保护后端并确认限流策略。
- 可调整 shard、batch、capacity 和 backoff，但增大并发可能压垮远端并增加本地内存。
- 降低无用指标、增加抓取间隔或流式聚合可以从源头减量。

恢复后关注 backlog 清空速度，避免“补发洪峰”再次打垮后端。

---

## 37. 告警风暴发生时怎么处理？★★★★★

**止损：**保留根因级关键告警，对受影响范围建立有截止时间和责任人的 Silence；不要直接关闭整个告警系统。

**定位：**找到最早告警与共同依赖，判断是基础设施根因、标签变化、规则错误还是通知通道重试。检查 Alertmanager 的 group key、route、inhibition、通知失败和集群状态。

**修复：**按服务/集群合理分组，建立父子告警抑制，加入 `for` 和恢复稳定窗口，删除不可行动告警，SLO 告警与资源告警分级。

**防复发：**统计每规则通知量、误报、无人处理和重复率；告警规则走代码评审、测试、灰度与复盘。

---

## 38. 指标已经超过阈值，但一直没有收到通知，怎么排查？★★★★★

沿完整链路检查：

1. 原始指标是否存在，查询时间与标签是否正确。
2. Rule 页面中规则是否加载、评估成功，当前为 Inactive/Pending/Firing 哪种状态。
3. `for` 是否满足，评估是否因查询错误/超时而失败。
4. Prometheus/vmalert 是否成功把告警发到所有 Alertmanager。
5. Alertmanager 是否被 Silence、Inhibition、time interval 或路由条件过滤。
6. Receiver 是否发送失败，Webhook/邮件/IM 是否限流。
7. 通知模板是否渲染失败。

防复发要做 Dead Man's Switch/Watchdog 和端到端合成告警，不能只监控组件进程存活。

---

## 39. 同一告警收到多次重复通知，怎么排查？★★★★☆

先比较每条通知的完整标签集合和 group key；只要 replica、pod、instance 等标签不同，就可能被视为不同告警。再检查 Alertmanager gossip 成员、网络分区、通知日志同步、时钟和 `repeat_interval`。

Prometheus HA 副本应保证告警身份一致，对仅用于区分采集副本的标签在相应去重层正确处理。Alertmanager 网络分区时重复通知是 fail-open 的预期行为，目标是避免漏报，不承诺 exactly-once。

---

## 40. VictoriaMetrics 某个 `vmstorage` 宕机，会发生什么？★★★★★

默认情况下：

- `vminsert` 会把新数据重路由到健康存储节点；健康节点负载会上升。
- `vmselect` 仍可查询，但可能返回带 `isPartial` 标记的不完整结果。
- 没有副本的数据在故障节点恢复前无法完整查询；如果磁盘永久损坏且无副本/备份则会丢失。

处置先确保剩余节点容量足够，限制重查询和异常写入，再修复节点。不要立即把坏节点从配置中永久删除并反复扩缩容，因为路由变化会造成数据分布变化和负载抖动。

---

## 41. VictoriaMetrics 集群扩容后数据会自动均衡吗？★★★★☆

不能简单回答“会自动搬迁所有历史数据”。新增 `vmstorage` 后，新的写入会按新拓扑分配，但历史数据仍主要留在原节点；查询层需要同时访问新旧节点。负载会随新数据逐步变化，历史磁盘不会立刻均衡。

扩缩容前要评估一致性哈希变化、剩余节点峰值、历史查询、备份和回滚。需要重新均衡时按照官方支持的迁移/再平衡工具和步骤执行，不能直接搬数据目录或复制 block 假设节点可互换。

---

## 42. 指标量突然暴涨，如何确认是哪个业务或标签导致的？★★★★★

先观察 active series、new series、ingestion rate 与 tenant/job 的变化时间点，再从 TSDB cardinality 统计中找：

- Top metric names。
- 每个 label name/value 的数量。
- 某 metric 的标签组合数。
- 新发布前后的差异。

常见根因是将请求 URL、异常消息、Pod UID、订单号、用户 ID、模型 ID 的非受控值作为 label。先通过 write/metric relabel 或入口限额止血，再修应用埋点；不能只在 Grafana 隐藏它。

---

## 43. Kubernetes Pod CPU/内存与 `kubectl top`、Grafana 对不上，怎么解释？★★★★★

先对齐指标定义、单位、窗口和容器范围：

- CPU Counter 应使用 `rate(container_cpu_usage_seconds_total[窗口])`，结果单位是核，不是百分比；除以 limit/节点核数才能得到相应比例。
- `kubectl top` 来自 Metrics API 的近期聚合值，采样和窗口与 Prometheus 查询不一定一致。
- 内存有 RSS、working set、usage、cache 等口径；working set 不是严格等于“不可回收内存”。
- 是否排除 pause/POD 容器、是否按 Pod 聚合、Pod 是否重建都会影响结果。
- requests/limits 是声明值，不是当前使用量。

回答时应给出业务要解决的问题，再选择口径，不要争论哪一个数字绝对“正确”。

---

## 44. P99 告警突然升高，但日志中大部分请求很快，怎么排查？★★★★★

检查总请求量与 bucket 增量，低流量时少数慢请求就会显著改变 P99。确认：

- bucket 是否覆盖真实延迟区间，边界是否太粗。
- 是否把不同接口、状态码、地域混合聚合。
- 查询是否保留了 `le`，是否先 `rate` 再 `sum`。
- 是否存在抓取缺失、Counter reset 或窗口过短。
- 慢请求是否集中在某个实例、依赖或冷启动阶段。

用 Trace exemplar 或相同时间窗口的 trace/log 下钻验证，不要只因“多数日志很快”否定长尾指标。

---

## 45. 监控系统自身故障，如何避免“监控瞎了却没人知道”？★★★★★

建立 Monitoring of Monitoring：

- 抓取成功率、规则评估失败/耗时、TSDB/WAL、Remote Write、查询错误与延迟。
- VictoriaMetrics 各组件可用性、写入错误、partial response、磁盘与集群成员。
- Alertmanager gossip、通知失败、队列与接收端可用性。
- 从独立故障域执行外部黑盒探测和 Heartbeat/Watchdog。
- 关键告警至少有一个独立通知链路或云外探测。

关键原则是避免监控平台、被监控业务和唯一通知渠道共享完全相同的故障域。

---

# 第四部分：可观测性与 AI Infra 扩展（46～50）

## 46. Metrics、Logs、Traces、Profiles 如何关联？★★★★☆

推荐路径是：

1. Metrics 告警发现哪个服务、地域和时间窗口异常。
2. 从面板 exemplar 或标签跳转到代表性 Trace。
3. Trace 找到最慢/错误 Span 和下游依赖。
4. 通过 trace_id/span_id 查询结构化日志获取业务上下文。
5. 如果是资源或代码热点，再看持续剖析 Profiles。

统一 `service.name`、environment、cluster、region 等资源属性，日志写入 trace_id/span_id；同时控制高基数，不能把 trace_id 直接变成 Prometheus 常规 label。

---

## 47. OpenTelemetry 解决什么问题？Collector 有什么作用？★★★★☆

OpenTelemetry 提供厂商中立的 API、SDK、语义约定和 Collector，用于生成、接收、处理和导出 traces、metrics、logs 等遥测数据，本身通常不是最终存储和可视化后端。

Collector 可承担：

- receiver 接收 OTLP、Prometheus 等协议。
- processor 做 batch、memory limiter、采样、属性清洗和资源检测。
- exporter 发往不同后端。
- 在 Agent 或 Gateway 模式部署。

生产上要防止 Collector 成为统一单点，并对队列、丢弃、导出失败、内存和 backpressure 做自监控。

---

## 48. 如何监控 GPU 集群和训练任务？★★★★☆

基础设施层通过 DCGM Exporter/NVML 等关注：GPU 利用率、显存、温度、功耗、时钟、XID/ECC、PCIe/NVLink、MIG 实例和节点健康。

训练任务还要关注：

- job/worker 状态、重启数、排队时间。
- step time、samples/tokens per second、loss 基本趋势。
- DataLoader/CPU/磁盘/对象存储吞吐。
- NCCL/RDMA 网络、集合通信耗时和 straggler。
- checkpoint 成功率、耗时、大小和最近成功时间。

GPU 利用率低只是现象，可能是数据读取、CPU 预处理、通信、同步等待、batch 太小或故障重试造成，不能直接得出“GPU 不够忙所以扩容”的结论。

---

## 49. 如何监控大模型在线推理服务？★★★★☆

用户层关注成功率、TTFT（首 Token 延迟）、TPOT/ITL、端到端延迟、tokens/s、请求/Token 吞吐和超时率；调度层关注排队时间、队列长度、并发、batch size、preemption；资源层关注 GPU/显存/KV Cache、CPU、网络与模型加载。

指标维度应控制模型、版本、量化方式、实例和状态码等有限集合，不应把 prompt、用户 ID 或 request ID 放入 label。告警优先围绕成功率、TTFT/P99 和 SLO burn rate，再用 GPU 与队列指标定位原因。

---

## 50. 设计一个多集群 Kubernetes 可观测性平台，怎么回答？★★★★★

**建议架构：**

1. 每个集群部署 vmagent 或 Prometheus Agent，使用 ServiceMonitor/PodMonitor 发现目标，采集节点、K8s 状态、控制面和应用指标。
2. 边缘采集端完成 relabel、限流、磁盘缓冲和必要聚合，并为数据附加稳定的 cluster/region/environment 标签。
3. 数据 Remote Write 到中心 VictoriaMetrics 单机或集群；跨 AZ 关键场景写入两个独立后端，而不是把一个 VM 集群跨高延迟链路强行拉开。
4. vmalert 执行规则并发往 Alertmanager HA；Alertmanager 做分组、抑制、静默和多渠道通知。
5. Grafana 提供统一查询，vmauth/网关实现认证、租户路由和权限边界。
6. 日志和 Trace 使用 OpenTelemetry Collector 接入独立后端，通过统一资源属性与 trace_id 关联。
7. 对平台自身做外部监控、容量预算、租户配额、规则测试、变更灰度、备份与恢复演练。

**面试官继续追问时要主动给出：**

- 当前活跃序列、samples/s、保留周期、查询 QPS 和增长率。
- 选 Prometheus、VictoriaMetrics 单机或集群的量化依据。
- 单集群、单 AZ、中心网络和通知渠道故障时系统如何降级。
- 如何处理重复样本、部分查询、高基数、租户隔离与成本归属。
- 如何证明告警链路真的可用，而不是组件都显示 Running。

---

# 第五部分：必须练熟的 PromQL

## 1. 实例可用率

```promql
avg_over_time(up{job="api"}[30d])
```

注意：`up` 只说明 Prometheus 能否抓取，不等于业务请求成功率，业务 SLI 应使用真实请求或黑盒探测。

## 2. 5xx 错误率

```promql
sum by (service) (rate(http_requests_total{code=~"5.."}[5m]))
/
sum by (service) (rate(http_requests_total[5m]))
```

## 3. P99 延迟

```promql
histogram_quantile(
  0.99,
  sum by (le, service) (
    rate(http_request_duration_seconds_bucket[5m])
  )
)
```

## 4. 节点 CPU 使用率

```promql
100 * (
  1 - avg by (instance) (
    rate(node_cpu_seconds_total{mode="idle"}[5m])
  )
)
```

## 5. 节点可用内存比例

```promql
100 * node_memory_MemAvailable_bytes
/
node_memory_MemTotal_bytes
```

Linux 可用内存优先看 `MemAvailable`，不要简单使用 `MemFree`。

## 6. 文件系统预测 24 小时内写满

```promql
predict_linear(
  node_filesystem_avail_bytes{fstype!~"tmpfs|overlay"}[6h],
  24 * 3600
) < 0
```

同时限制当前使用率和数据质量，避免刚挂载、清理任务或非线性增长造成误报。

## 7. Pod CPU 使用核数

```promql
sum by (cluster, namespace, pod) (
  rate(container_cpu_usage_seconds_total{container!="",image!=""}[5m])
)
```

## 8. Pod CPU 相对 requests 的比例

```promql
sum by (cluster, namespace, pod) (
  rate(container_cpu_usage_seconds_total{container!="",image!=""}[5m])
)
/
sum by (cluster, namespace, pod) (
  kube_pod_container_resource_requests{resource="cpu"}
)
```

真实环境先核对指标版本、单位和重复采集，再处理分母为 0 的 Pod。

## 9. Deployment 可用副本不足

```promql
kube_deployment_status_replicas_available
<
kube_deployment_spec_replicas
```

## 10. 规则评估接近超时/周期

```promql
prometheus_rule_group_last_duration_seconds
/
prometheus_rule_group_interval_seconds
> 0.8
```

不同版本和组件的自监控指标名可能不同，上线前必须以实际 `/metrics` 为准。

---

# 第六部分：项目经历回答模板

面试官问“你们监控平台怎么做的”，不要只报组件名称。建议按以下顺序回答：

## 1. 背景与规模

- 管理多少集群、节点、Pod、服务和租户。
- active series、samples/s、保留周期、日增量、查询 QPS。
- 过去的痛点：Prometheus OOM、查询慢、数据孤岛、告警噪音或成本高。

## 2. 架构与选型

- 采集：Prometheus、Prometheus Agent 或 vmagent。
- 存储：Prometheus 本地、VictoriaMetrics 单机/集群，为什么。
- 查询与展示：Grafana、统一数据源、租户和权限。
- 规则与通知：Prometheus/vmalert + Alertmanager。
- 日志/Trace：如何通过公共资源属性和 trace_id 关联。

## 3. 关键取舍

- Pull 与跨网络采集如何解决。
- HA 是双采集、副本还是双集群；如何去重。
- 一致性与部分响应如何选择。
- 高基数、查询成本、租户限额和数据保留如何治理。
- 告警为什么基于 SLO/症状，如何减少噪音。

## 4. 一次真实事故

严格使用：

> 现象 → 影响范围 → 止损 → 证据链 → 根因 → 修复 → 验证 → 防复发。

尽量给数字，例如“序列数从 300 万升至 900 万”“Remote Write 延迟从 20 秒涨到 40 分钟”“P99 在 15 分钟内从 200ms 升到 2s”，而不是只说“很多、很慢、优化了”。

## 5. 最终结果

量化前后变化：告警量、MTTA/MTTR、查询 P95、资源成本、数据完整率、序列数和故障恢复时间。

---

# 第七部分：原始资料中降级或删除的内容

以下内容不作为主干复习：

- “Prometheus 是什么”“Grafana 是什么”这类一句话定义题，只保留为回答开场。
- 罗列所有 exporter、所有函数、所有端口和所有命令，区分度低且容易过时。
- Kubernetes 旧版非认证只读端口、Docker experimental metrics 等历史配置。
- 把 `vmalert` 说成 Alertmanager 替代品。
- 把 `vmagent` 简化成“Prometheus Remote Write”。
- 把 Prometheus HA 说成主备选举，或认为本地 TSDB 没集群就无法构建 HA 系统。
- 认为 Alertmanager 集群能严格保证 exactly-once。
- 认为 VictoriaMetrics 增加节点后会自动把全部历史数据立刻均衡。
- 把 Summary quantile 跨实例平均，或认为 `histogram_quantile` 是精确分位数。
- 只根据 CPU/内存固定阈值做页面告警，不解释用户影响和处置动作。

---

# 第八部分：面试前 10 分钟速记

1. Prometheus：发现 → 拉取 → Relabel → TSDB/Remote Write → Rules → Alertmanager。
2. Counter 用 `rate`；告警优先 `rate`，尖峰观察才考虑 `irate`。
3. Histogram 可聚合；Summary quantile 通常不可跨实例聚合。
4. P99 是估算值，正确查询必须保留 `le`，bucket 应围绕 SLO 设计。
5. 高基数重点查无界 label；高 churn 重点查临时 Pod、任务和滚动发布。
6. Prometheus HA 通常 active-active；Alertmanager 是 gossip、at-least-once、fail-open。
7. `vminsert` 写入分发，`vmstorage` 存储，`vmselect` 查询合并。
8. `vmagent` 负责采集/转发/缓冲；`vmalert` 执行规则；Alertmanager 负责通知治理。
9. VictoriaMetrics 默认分片不等于复制；部分存储故障可能返回 partial response。
10. 场景题永远回答：现象 → 止损 → 定位 → 根因 → 修复 → 防复发。

---

# 参考资料

## 用户提供的资料

- [知乎：参考文章](https://zhuanlan.zhihu.com/p/1900526724631988167)
- [博客园：高可用 Prometheus 问题集锦](https://www.cnblogs.com/uglyliu/p/12733763.html)
- [阿里云开发者社区：Prometheus 核心概念与实践高频问答](https://developer.aliyun.com/article/1239814)
- [Prometheus 面试题整理](https://devopsz.top/docs/2025-2-28-prometheus%E9%A2%98%E7%9B%AE/)

## 用于版本纠错的官方资料

- [Prometheus Storage](https://prometheus.io/docs/prometheus/latest/storage/)
- [Prometheus Remote Write tuning](https://prometheus.io/docs/practices/remote_write/)
- [Prometheus Alerting Rules](https://prometheus.io/docs/prometheus/latest/configuration/alerting_rules/)
- [Alertmanager High Availability](https://prometheus.io/docs/alerting/latest/high_availability/)
- [Prometheus Histograms and Summaries](https://prometheus.io/docs/practices/histograms/)
- [VictoriaMetrics Cluster Architecture](https://docs.victoriametrics.com/cluster-victoriametrics/)
- [VictoriaMetrics vmagent](https://docs.victoriametrics.com/victoriametrics/vmagent/)
- [VictoriaMetrics vmalert](https://docs.victoriametrics.com/victoriametrics/vmalert/)
- [VictoriaMetrics MetricsQL](https://docs.victoriametrics.com/victoriametrics/metricsql/)
- [OpenTelemetry Observability Primer](https://opentelemetry.io/docs/concepts/observability-primer/)

> 说明：参考文章用于判断题目出现频率和实践关注点；涉及当前组件行为、配置语义和版本变化的答案，以官方文档为准。
