# Kubernetes 面试题高频精选

> 适用方向：SRE、DevOps、Kubernetes 平台、云原生运维开发、AI Infra 社招  
> 目标级别：月薪 25K～40K、3～7 年经验的中高级岗位  
> 排序依据：四份牛客资料的重复度、生产岗位常见度、追问深度和故障处理区分度。星级是经验分级，不是严格统计。  
> 推荐答法：**先说结论 → 讲控制链路或内核机制 → 给出验证方法 → 补充生产取舍和防复发措施**。

---

## 先看结论：这类 Kubernetes 题怎么准备

面向 25K～40K 的岗位，面试官通常不会满足于“会部署集群、会写 YAML”。真正有区分度的是：

1. 能否讲清 API Server、etcd、Scheduler、Controller、kubelet 之间如何协作。
2. 能否从声明式 API、控制循环和最终一致性的角度解释资源变化。
3. 能否定位 Pod Pending、CrashLoopBackOff、节点 NotReady、Service 不通、DNS 异常、OOM、PVC Pending 等生产事故。
4. 是否做过高可用、备份恢复、升级、容量治理、权限和安全加固。
5. 是否理解 CNI、CSI、CRI、Linux 网络、cgroup、conntrack 等边界，而不是把所有问题都归给 Kubernetes。

本题库共 44 道：15 道必问、15 道高频、12 道生产场景、2 道 AI Infra 扩展。你已有 CKA，复习时应优先把“操作题”升级成“原理 + 事故案例”。

---

## 一、必问级：★★★★★

### 1. Kubernetes 整体架构是什么？各组件分别做什么？

Kubernetes 集群由控制面和工作节点组成，核心思路是通过 API 保存期望状态，再由各控制循环不断把实际状态收敛到期望状态。

- `kube-apiserver`：统一 API 入口，负责认证、鉴权、准入、校验及对 etcd 的读写。
- `etcd`：保存 Kubernetes API 数据的强一致键值数据库。
- `kube-scheduler`：为尚未绑定节点的 Pod 选择合适节点。
- `kube-controller-manager`：运行 Deployment、ReplicaSet、Node、EndpointSlice 等控制器。
- `cloud-controller-manager`：可选，连接云厂商的节点、路由和负载均衡能力。
- `kubelet`：节点代理，持续同步分配给本节点的 Pod，通过 CRI 驱动容器运行时。
- 容器运行时：常见为 containerd、CRI-O，负责镜像和容器生命周期。
- `kube-proxy`：可选，在传统实现中同步 Service 转发规则；使用某些 eBPF 数据面时可能被替代。
- CoreDNS、CNI、CSI、监控和日志属于常见插件或扩展，不是所有集群都以同样方式部署。

高分回答要强调：组件主要通过 API Server 交互，而不是互相随意直连；控制器和调度器通常通过 watch/informer 获取变化。

### 2. 从提交 YAML 到 Pod 运行，完整流程是什么？

可以按“接收、调度、节点执行、进入服务”四段回答：

1. `kubectl` 把请求发送到 API Server。
2. API Server 完成认证、鉴权、准入控制、默认值填充和对象校验，随后写入 etcd。
3. Scheduler 发现没有 `spec.nodeName` 的 Pod，经过过滤、评分和插件处理选择节点，并写入绑定结果。
4. 目标节点 kubelet watch 到 Pod，执行 Pod 同步。
5. kubelet 通过 CRI 请求运行时创建 Pod sandbox；运行时配置 network namespace，并调用 CNI 分配 IP、设置路由。
6. kubelet准备 Secret、ConfigMap 和卷；需要持久化存储时通过 CSI 完成 attach／mount 流程。
7. 运行时拉取镜像并启动 init container，成功后再启动应用容器和原生 sidecar。
8. kubelet持续上报 Pod、容器状态并执行探针。
9. Pod Ready 且匹配 Service selector 后，EndpointSlice 控制器把它纳入服务端点，流量转发规则随之更新。

面试官常追问：CNI 是谁调用的、调度器是否直接启动容器、Pod Running 是否等于可接流量。答案分别是运行时在 CRI sandbox 流程中调用 CNI；调度器只负责绑定；Running 不等于 Ready。

### 3. Kubernetes 为什么叫声明式系统？Controller 如何工作？

用户提交的是期望状态，例如 Deployment 希望有 10 个副本；控制器读取期望状态和实际状态，计算差异并执行动作，直到收敛。

典型控制循环是：

1. 通过 informer 的 list/watch 获得对象变化并维护本地缓存。
2. 把待处理 key 放入 work queue。
3. worker 执行 reconcile，对比 desired state 与 actual state。
4. 创建、更新或删除下游对象；失败时限速重试。
5. 更新 status 和 condition，等待下一次事件或周期性重同步。

这种模型通常是最终一致，而不是每次 API 请求都同步完成全部动作。控制器应当幂等，因为同一对象可能被重复处理。

### 4. Pod 生命周期、容器状态和 `restartPolicy` 怎么理解？

Pod phase 主要有 `Pending`、`Running`、`Succeeded`、`Failed`、`Unknown`；容器状态是 `Waiting`、`Running`、`Terminated`。两者不能混为一谈。

- `restartPolicy` 取值为 `Always`、`OnFailure`、`Never`，由 kubelet在原 Pod 内决定容器是否重启。
- Deployment、DaemonSet、StatefulSet 的 Pod 通常使用 `Always`。
- Job 常见 `OnFailure` 或 `Never`。
- `CrashLoopBackOff` 不是 Pod phase，而是容器反复失败后进入指数退避时看到的状态原因。
- 容器重启不会创建新 Pod；控制器替换失败 Pod 时才会产生新的 Pod UID。

排障不能只看 STATUS 列，要同时看 `status.phase`、container state、`lastState`、退出码、reason、events 和前一次日志。

### 5. liveness、readiness、startup probe 有什么区别？如何避免探针造成事故？

- liveness：判断容器是否需要被重启；失败后 kubelet终止容器，再按重启策略处理。
- readiness：判断是否可以接收流量；失败时 Pod 不 Ready，并从匹配 Service 的可用 EndpointSlice 端点中移出。
- startup：给慢启动应用独立启动窗口；成功前会抑制 liveness 和 readiness，避免应用尚未启动就被杀死。

探针支持 HTTP、TCP、gRPC 和 exec 等方式。生产上应注意：

1. liveness 只检查“重启能修复”的故障，不能把下游数据库可用性直接作为存活条件，否则下游故障会触发集体重启。
2. readiness 可以包含关键依赖，但要避免检查过重造成雪崩。
3. 设置合理的 `timeoutSeconds`、`periodSeconds`、`failureThreshold`。
4. 慢启动程序使用 startup probe，不要把 liveness 初始延迟设置得无限大。
5. 监控探针失败率和重启次数，发布前做压力验证。

### 6. Deployment 滚动更新是怎么实现的？怎样做到尽量无损？

Deployment 创建新 ReplicaSet，逐步扩大新 RS、缩小旧 RS。`maxSurge` 控制最多额外创建多少 Pod，`maxUnavailable` 控制最多允许多少不可用；百分比计算时前者向上取整、后者向下取整。

无损发布不仅是两个参数：

- 正确配置 readiness 和 startup probe。
- 应用捕获 `SIGTERM`，先停止接新请求，再完成在途请求。
- 合理配置 `terminationGracePeriodSeconds`，必要时使用 `preStop`。
- 使用 `minReadySeconds` 防止刚 Ready 就抖动。
- 通过 `progressDeadlineSeconds`、监控和流水线判断发布是否失败。
- 保留足够 `revisionHistoryLimit`，镜像使用不可变版本或 digest。
- 对长连接、消息消费、注册中心摘流和数据库变更单独设计。

`kubectl rollout undo` 只能回滚 Deployment 模板，不能自动回滚数据库、外部配置和不可逆数据变更。

### 7. Service、EndpointSlice 和 kube-proxy 是什么关系？

Service 提供稳定的服务发现和虚拟入口，后端 Pod 可以动态变化。带 selector 的 Service 由控制器根据标签生成 EndpointSlice；EndpointSlice 保存实际后端 IP、端口和就绪等条件。

传统数据面中，每个节点的 kube-proxy watch Service 和 EndpointSlice，并把 ClusterIP／NodePort 等虚拟入口转换为发往某个后端端点的规则。流量通常不会经过 API Server，也不会真的存在一个监听 ClusterIP 的普通进程。

常见 Service 类型：

- ClusterIP：集群内部虚拟 IP，默认类型。
- NodePort：在节点端口暴露，同时仍包含 ClusterIP 语义。
- LoadBalancer：依赖云控制器或 MetalLB 等实现外部负载均衡。
- ExternalName：返回外部域名的 CNAME，不做四层转发。
- Headless Service：`clusterIP: None`，通常通过 DNS 直接返回端点，常用于 StatefulSet。

当前应优先查看 EndpointSlice；旧 Endpoints API 已被标记弃用，并存在大规模端点截断等限制。

### 8. kube-proxy 的 iptables、IPVS、nftables 模式有什么区别？

- iptables：使用 netfilter/iptables 规则完成转发，成熟且兼容性广；规则规模很大时同步和遍历成本较高。
- IPVS：使用内核 IPVS 和部分 iptables 规则，历史上在大量 Service 下性能较好并支持多种调度算法，但 Kubernetes Service 语义与 IPVS 并不完全匹配。
- nftables：使用 nftables API，规则更新和大规模性能更好，Kubernetes 1.33 起稳定；要求较新的内核，并需确认 CNI 和现有防火墙兼容性。

版本纠错：Kubernetes 1.35 已将 IPVS kube-proxy 模式标记为弃用，官方更推荐以 nftables 替代；不能再机械回答“生产一定用 IPVS”。现网迁移前要检查内核、CNI、NodePort 绑定地址、防火墙和 conntrack 行为。

使用 Cilium 等 eBPF 数据面时，Service 转发可能由 eBPF 实现并替代 kube-proxy，需要结合具体集群回答。

### 9. Kubernetes 网络模型是什么？CNI 负责什么？

核心网络模型是：每个 Pod 拥有独立集群 IP；同一 Pod 内的容器共享 network namespace，可以通过 localhost 通信；在未被策略限制时，Pod 间应能直接通信，不要求应用自行做 NAT。

CNI 是网络插件规范，负责在 Pod sandbox 创建或删除时配置网络，例如：

- 创建和移动 veth、分配 Pod IP。
- 设置路由、网桥、隧道或 eBPF 程序。
- 返回网络配置结果并在删除时清理。

不同实现的数据路径不同：Calico 可用路由、VXLAN、IP-in-IP 或 eBPF；Cilium 以 eBPF 为核心。面试时不要把“CNI”直接等同于某个产品，也不要把 Service 转发和 Pod 网络混为一谈。

### 10. requests、limits、QoS、OOM 和 CPU throttling 有什么关系？

- scheduler 主要根据 requests 判断节点能否容纳 Pod，不按实时使用率调度。
- CPU request 影响调度和竞争时的权重；CPU limit 常通过 CFS quota 产生节流，不会像内存超限那样直接 OOM。
- memory request 影响调度和驱逐排序；memory limit 是硬边界之一，超限可能触发 cgroup OOM Kill。
- 退出码 137 只说明进程收到 `SIGKILL`，还需结合 `OOMKilled`、内核日志和 cgroup 指标确认。

QoS 常见三类：

- Guaranteed：每个容器都设置 CPU 和内存 request/limit，且对应值相等。
- Burstable：不满足 Guaranteed，但至少有资源 request 或 limit。
- BestEffort：未设置 CPU、内存 request 和 limit。

节点压力驱逐不仅看 QoS，还会综合使用量是否超过 request、Pod Priority 和超出程度。OOM Kill 与 kubelet eviction 是不同路径。

### 11. Scheduler 调度 Pod 的核心流程是什么？Pod Pending 怎么定位到具体插件？

调度大体分为：队列 → 过滤 → 评分 → 预留／许可等扩展点 → 绑定。

- PreFilter／Filter：检查资源、节点选择器、污点容忍、亲和性、卷拓扑和端口等硬条件。
- Score：对候选节点按资源、亲和性、拓扑分布等维度评分。
- Reserve／Permit／PreBind：为高级插件提供预留、等待和绑定前处理。
- Bind：把 Pod 的 `spec.nodeName` 写回 API。

排查 Pending 先看 events 中的 `FailedScheduling`，不要直接猜：

```bash
kubectl describe pod POD -n NS
kubectl get events -n NS --sort-by=.lastTimestamp
kubectl describe node NODE
kubectl get pvc -n NS
```

再把提示映射到资源不足、taint、affinity、PVC 拓扑、hostPort、配额或调度器扩展插件。必要时查看 scheduler 日志和指标。

### 12. Deployment、StatefulSet、DaemonSet、Job、CronJob 怎么选？

- Deployment：无状态、可替换副本，支持滚动更新和回滚。
- StatefulSet：需要稳定网络身份、有序管理或稳定存储声明的副本；不代表应用天然高可用。
- DaemonSet：每个符合条件的节点运行一个 Pod，常用于日志、监控、网络和设备插件。
- Job：执行到成功完成的批处理任务，可设置并行度、完成数和失败策略。
- CronJob：按计划创建 Job；要考虑并发策略、时区、错过调度和任务幂等。

判断标准是工作负载语义，而不是“有数据库就一定 StatefulSet”。数据库的复制、一致性、备份和故障切换仍由数据库或 Operator 负责。

### 13. PV、PVC、StorageClass、CSI 的关系是什么？

- PV：集群级存储资源对象，描述一块可用存储。
- PVC：命名空间级存储需求，声明容量、访问模式和 StorageClass。
- StorageClass：描述动态供给的 provisioner、参数、回收策略和绑定模式。
- CSI：存储插件接口，控制器侧负责 provision/attach/snapshot 等，节点侧负责 stage/publish/mount 等。

动态供给时，PVC 触发 CSI provisioner 创建后端卷和 PV，再完成绑定。拓扑受限存储建议使用 `WaitForFirstConsumer`，把供给或绑定延迟到调度时，避免卷先建在 Pod 无法运行的可用区。

`Retain`、`Delete` 是 PV 回收策略；它们不等于业务级备份。生产数据库仍需应用一致性备份、恢复演练和跨故障域设计。

### 14. etcd 在 Kubernetes 中起什么作用？为什么必须做快照恢复演练？

etcd 保存 Kubernetes API 对象，是控制面的事实数据源。它使用 Raft 在成员间复制日志并依赖多数派形成 quorum，因此常用奇数成员；增加偶数成员不会增加可容忍故障数。

生产关注点：

- 低延迟、稳定的磁盘 fsync 和成员间网络。
- 定期快照、快照状态和哈希校验。
- 定期 compaction、按需 defrag、空间配额与告警。
- 监控 leader 变化、proposal、WAL fsync、backend commit 和 DB 大小。
- 客户端与 peer TLS、最小网络暴露。

只备份不恢复等于没有验证。恢复会创建新的逻辑 etcd 集群；Kubernetes 场景还要考虑 revision 回退对 informer/watch 缓存的影响，按当前 etcd 官方流程使用 revision bump。恢复后必须检查 API 对象、控制器、工作负载和证书／加密配置。

### 15. Kubernetes 高可用控制面怎么设计？

典型方案是：多个 control-plane 节点 + API Server 前置四层负载均衡／虚拟 IP + 3 或 5 节点 etcd + 控制组件 leader election。

- API Server 基本无状态，可多实例并行服务。
- Scheduler 和 Controller Manager 多实例运行，但通常由 leader 执行关键控制逻辑。
- stacked etcd 部署简单但故障域与控制面重合；external etcd 隔离更强、运维更复杂。
- 节点应跨宿主机、机架或可用区，避免共同电源和网络故障。
- 负载均衡健康检查要验证 API 可用性，不只检查端口。
- 备份、证书、时间同步、版本偏差和容量必须纳入日常管理。

高可用不是“部署三个 Master”就结束，还要验证失去一个 API Server、一个 etcd 成员、一个可用区时，读写、调度和已有业务分别有什么影响。

---

## 二、高频级：★★★★☆

### 16. 删除一个 Pod 时发生什么？怎样优雅终止？

正常删除时 API 对象设置 `deletionTimestamp` 和宽限期；kubelet看到终止状态后执行 `preStop`，让运行时向主进程发送 TERM；同时控制面把端点标记为终止且不再作为常规就绪后端。宽限期结束后仍未退出的进程会收到 KILL，随后清理 sandbox、网络和卷，API 对象最终删除。

默认宽限期通常是 30 秒。`preStop` 消耗的是同一段宽限期，不是额外时间。强制删除会让 API 对象先消失，但节点上的进程可能短时间仍存在，存在重复实例或数据损坏风险。

### 17. ConfigMap 和 Secret 怎样更新到 Pod？为什么改了配置应用没生效？

- 环境变量在容器启动时注入，更新 ConfigMap／Secret 后不会自动刷新，通常需要滚动重启。
- 普通 projected volume 会由 kubelet周期性更新，存在传播延迟；应用还必须支持重新读取或热加载。
- 使用 `subPath` 挂载的单文件不会收到自动更新。
- Secret 默认只是 base64 编码，不等于加密；应启用 etcd 静态加密、严格 RBAC、审计和外部密钥管理。

生产发布常把配置哈希写入 Pod template annotation，让配置变化触发 Deployment 滚动更新。

### 18. RBAC、ServiceAccount 和 Admission 分别控制什么？

API 请求大体经过认证、鉴权和准入：

- ServiceAccount 给 Pod 提供工作负载身份，优先使用短期 projected token。
- Role／ClusterRole 描述允许的 API verbs 和 resources。
- RoleBinding／ClusterRoleBinding 把权限绑定给用户、组或 ServiceAccount。
- Admission 在鉴权后对对象做变更或校验，例如默认值、资源策略、镜像策略和 Pod Security Admission。

最小权限要同时限制 verb、resource、namespace 和身份。能在 Pod 内拿到 token 不代表一定有权限；最终以 API Server 的 RBAC 决策为准。

### 19. NetworkPolicy 如何工作？为什么写了策略却没有效果？

NetworkPolicy 通过 Pod／Namespace selector 描述 ingress、egress 允许规则。当某方向被策略选中后，该方向进入默认拒绝，只放行规则匹配流量；策略是叠加允许，不是按顺序执行的防火墙 ACL。

无效常见原因：

- CNI 不支持 NetworkPolicy 或某些字段。
- selector／namespace label 写错。
- 只限制 ingress，遗漏 egress，或反过来。
- 忘记放行 DNS、健康检查和必要控制面流量。
- `ipBlock` 与 Service NAT 前后地址的匹配行为依赖数据路径实现。

变更前应从源、目标双向验证，并保留回滚路径，避免把自己锁出集群。

### 20. Service、Ingress、Ingress Controller、Gateway API 有什么区别？

- Service 是四层服务发现和暴露抽象。
- Ingress 是 HTTP／HTTPS 路由 API 对象，本身不转发流量，必须有 Ingress Controller 实现。
- Ingress Controller 运行实际代理或配置云负载均衡。
- Gateway API 是更角色化、可扩展的流量 API，常见稳定资源包括 GatewayClass、Gateway、HTTPRoute；仍需要对应 controller。

Ingress 不是 Service 类型，也不等于 Nginx。新平台设计可评估 Gateway API，但要看控制器成熟度、现有注解能力、迁移成本和团队维护能力。

### 21. HPA 如何工作？为什么 CPU HPA 不扩容？

HPA controller 周期性读取资源指标、custom metrics 或 external metrics，根据当前值与目标值计算期望副本，并受容忍区间、稳定窗口和伸缩策略影响。

CPU utilization 通常是相对 CPU request 计算；没设 request、metrics-server 异常、指标缺失或 Pod 尚未 Ready 都可能导致计算异常。还要检查：

- `minReplicas`／`maxReplicas`。
- 扩缩容行为策略和 stabilization window。
- 新副本是否因资源、PVC、GPU 或拓扑约束 Pending。
- 下游瓶颈是否导致扩容无效。

HPA 解决工作负载副本数，Node Autoscaler 解决节点供给，两者不是同一层。

### 22. PDB、cordon、drain 分别有什么作用？

- `cordon`：把节点标为不可调度，不迁移已有 Pod。
- `drain`：通过 Eviction API 驱逐可驱逐 Pod，常用于升级或维护。
- PDB：限制自愿中断期间同时不可用的副本数，用 `minAvailable` 或 `maxUnavailable` 表达。

PDB 不防节点宕机、OOM、节点压力驱逐等非自愿中断，也不保证业务一定可用。配置过严可能让 drain 卡住；单副本服务设置 `minAvailable: 1` 会阻止正常自愿驱逐。

### 23. nodeSelector、亲和性、污点容忍和 topology spread 如何选择？

- nodeSelector：简单硬约束。
- node affinity：支持 required／preferred 和更丰富表达式。
- pod affinity／anti-affinity：按其他 Pod 的标签和拓扑位置约束。
- taint：节点排斥 Pod；toleration 只允许 Pod 被考虑，不保证一定调度到该节点。
- topology spread：按 zone、hostname 等拓扑域均匀分布，适合可用性和容量平衡。

生产中避免大量硬 anti-affinity 导致无节点可选；关键副本通常使用跨可用区 topology spread + 合理软／硬约束，并结合真实节点标签和容量验证。

### 24. init container、sidecar 和 ephemeral container 分别用在哪里？

- init container：应用容器前按顺序运行并成功完成，适合初始化、等待依赖或生成配置。
- 原生 sidecar：以 init container 的 `restartPolicy: Always` 表达，可在主容器前启动并由 kubelet按特定顺序终止。
- ephemeral container：临时注入已运行 Pod 做调试，不参与正常重启和资源声明，不应作为常驻业务方案。

如果生产镜像没有 shell，可以使用 `kubectl debug` 的临时容器或节点调试，而不是长期给业务镜像塞满排障工具。

### 25. Static Pod 和 Mirror Pod 是什么？

Static Pod 由某个节点上的 kubelet直接管理，常用于 kubeadm 控制面组件。kubelet可在 API Server 中创建对应 Mirror Pod 便于观察，但控制面不能像普通 Pod 那样调度或接管它。

删除 Mirror Pod 不能停止真正的 Static Pod；应修改或移除节点上的静态清单。排查控制面时要检查 `/etc/kubernetes/manifests`、kubelet 日志和容器运行时，而不只是 `kubectl logs`。

### 26. ownerReferences、Finalizer 和级联删除是什么关系？

ownerReferences 描述对象所有权，垃圾回收器据此处理 dependents；前台／后台／孤儿级联策略决定删除方式。Finalizer 是删除前必须完成的清理任务。

对象有 `deletionTimestamp` 却长期 Terminating，常见原因是 finalizer 对应 controller 不工作、外部资源清理失败或权限不足。不能把“手工删 finalizer”当第一步，先确认会不会遗留负载均衡器、云盘、网络或业务数据。

### 27. `imagePullPolicy` 和镜像标签怎样影响发布？

- 显式配置优先；未配置时，创建对象时会根据标签／digest 设置默认策略。
- `Always` 仍可能复用节点上已有的相同 digest，并不等于每次完整下载镜像层。
- `IfNotPresent` 适合不可变版本，但同一 tag 被覆盖时可能造成节点间内容不一致。
- `Never` 只适合已预置镜像的受控环境。

生产应使用唯一版本或 digest，配置镜像仓库凭据、并发和退避监控；不要把 `latest` 当作发布版本管理。

### 28. Kubernetes 集群升级应怎样做？

升级前先看版本偏差策略、API 废弃、CNI／CSI／Ingress／监控兼容矩阵，并完成 etcd 快照和恢复验证。以 kubeadm 为例，不支持跳过 minor 版本升级。

推荐流程：

1. 在测试集群和一小部分节点演练。
2. 检查控制面、etcd、节点和关键工作负载健康。
3. 先升级第一个控制面，再升级其他控制面。
4. 逐批 cordon、drain、升级 worker 的 kubeadm／kubelet／kubectl，再 uncordon。
5. 验证 CoreDNS、kube-proxy、CNI、CSI、Admission、CRD 和业务 SLO。
6. 保留停止条件和恢复方案，记录实际耗时和容量余量。

升级不只是替换二进制；API 兼容、webhook 和第三方控制器往往是实际风险点。

### 29. 怎样监控 Kubernetes 控制面、节点和工作负载？

至少分三层：

- 控制面：API 请求延迟／错误／限流、etcd leader 与 fsync、scheduler pending、controller queue。
- 节点：CPU、内存、磁盘／inode、网络、conntrack、kubelet、runtime、CNI、证书和 Node condition。
- 工作负载：副本可用性、重启、OOM、throttling、Pending、探针、请求 RED 指标、依赖和 SLO。

常见组合是 Prometheus、Alertmanager、Grafana、kube-state-metrics、node exporter、日志系统和 OpenTelemetry。告警应指向用户影响和可执行动作，避免仅按“Pod 重启一次”制造噪声。

### 30. Kubernetes 中如何做容量规划和成本治理？

先区分 requests、limits、实际峰值和业务 SLO。持续统计 P50／P95／P99 使用、周期性、发布峰值和故障冗余，再调整资源：

- 纠正明显过高或过低的 requests／limits。
- 使用 HPA／VPA 建议、Node Autoscaler 和合适实例规格。
- 为系统组件预留资源，保留升级、节点故障和突发流量余量。
- 用 PriorityClass、Quota、LimitRange 和命名空间成本归属控制多租户。
- 批处理、在线服务和 GPU 任务采用不同调度及超卖策略。

不能只追求平均利用率。集群压得太满会导致发布失败、故障迁移失败和扩容来不及，最终成本更高。

---

## 三、生产场景题：★★★★☆～★★★★★

### 31. Pod 一直 Pending，怎么排查？

先用 events 定位阶段，再分支处理：

1. `FailedScheduling`：资源 requests、taint、node affinity、topology spread、hostPort、Pod 数量上限、Priority 或配额。
2. `FailedMount`／PVC Pending：StorageClass、CSI、访问模式、容量、拓扑和后端存储。
3. `ImagePullBackOff`：镜像名、tag、仓库网络、证书、凭据、限流和节点磁盘。
4. Pod 已绑定但初始化慢：CNI sandbox、volume mount、init container 和 runtime。

不要第一反应就给节点加资源；先确认是容量不足还是硬约束互相冲突。

### 32. Pod 出现 CrashLoopBackOff，怎么排查？

```bash
kubectl describe pod POD -n NS
kubectl logs POD -n NS -c CONTAINER --previous
kubectl get pod POD -n NS -o yaml
```

重点检查：退出码和 `lastState`、应用日志、命令／参数、环境变量、挂载、权限、依赖、探针、OOM 和架构不匹配。若镜像过于精简，使用 ephemeral container 调试；必要时复制 Pod 并改入口进行隔离验证。

处理顺序是先判断是否影响服务并回滚错误发布，再定位原因。不要为了“消除 CrashLoop”盲目延长探针或无限增加内存。

### 33. 节点变成 NotReady，怎么处理？

先确认影响范围和副本是否已迁移，必要时 cordon 节点。排查链路：

1. `kubectl describe node` 查看 Ready、MemoryPressure、DiskPressure、PIDPressure、NetworkUnavailable 和事件。
2. 检查 Node Lease 是否更新、API Server 到节点和节点到 API Server 的网络。
3. 查看 kubelet 状态、日志、证书、时间同步和 cgroup 配置。
4. 检查 containerd／CRI、CNI、磁盘／inode、内存、PID、conntrack 和内核异常。
5. 确认机器、电源、虚拟化或云平台故障。

恢复后不能只 uncordon，要验证 Pod sandbox、DNS、Service、卷和节点监控；无法快速恢复时按业务和 PDB 安全迁移。

### 34. Service 有 ClusterIP，但集群内访问不通，怎么排查？

按数据路径逐层缩小范围：

1. Service 名称、端口、协议、`targetPort` 是否正确。
2. selector 是否匹配，EndpointSlice 是否有 ready 端点。
3. 绕过 Service 直接访问 Pod IP:port，确认应用监听地址和容器端口。
4. 检查 NetworkPolicy、CNI 路由和跨节点通信。
5. 根据数据面检查 kube-proxy 日志与 iptables／nftables／IPVS，或 Cilium eBPF 状态。
6. 查看 conntrack 表、反向路由、rp_filter、MTU 和节点防火墙。

同节点能通、跨节点不通，优先查 CNI、路由／隧道和 MTU；Pod IP 能通、Service IP 不通，优先查 Service 规则和 EndpointSlice。

### 35. Pod DNS 偶发超时，怎么排查？

先区分解析失败、连接 CoreDNS 失败和上游 DNS 失败：

- 检查 Pod `/etc/resolv.conf`、`ndots` 和 search domain，确认是否产生大量无效查询。
- 直接查询 CoreDNS Service IP，再查询 CoreDNS Pod IP，比较 UDP 和 TCP。
- 检查 CoreDNS 副本、CPU throttling、延迟、错误、缓存、上游转发和日志。
- 检查 kube-proxy／Service 规则、CNI、conntrack 表容量和 UDP 丢包。
- 确认节点是否存在路由、MTU、iptables 竞争更新或 conntrack race。

高 QPS 集群可评估 NodeLocal DNSCache、合理缓存和减少搜索域扩展，但必须先找到瓶颈，不能用加副本掩盖网络问题。

### 36. Pod 跨节点小包正常，大包失败或偶发超时，怎么定位？

优先怀疑 MTU、隧道封装、分片／PMTUD、路由和安全策略。操作顺序：

1. 比较同节点与跨节点、Pod IP 与 Service IP。
2. 使用 `ping -M do -s` 或等价方法逐步探测有效 MTU。
3. 抓取 Pod veth、隧道接口、物理网卡两端数据包。
4. 检查 VXLAN／IP-in-IP／WireGuard 开销、ICMP fragmentation-needed 是否被丢弃。
5. 检查双向路由、rp_filter、NetworkPolicy、conntrack 和 NIC offload。

修复应统一节点、CNI 和底层网络 MTU，并加入跨节点大包探测，而不是只在应用层无限重试。

### 37. Pod 被 OOMKilled 或节点发生 Evicted，怎么区分和治理？

OOMKilled 是内核／cgroup 内存回收路径，常见 container reason 为 `OOMKilled`；Evicted 是 kubelet因节点 MemoryPressure、DiskPressure、inode 等压力主动终止 Pod，Pod reason 会显示驱逐原因。

排查：

- 看 `lastState`、exit code、events、node condition 和 kubelet／内核日志。
- 对比 working set、RSS、page cache、request、limit 和节点可用内存。
- 检查 `emptyDir`、容器日志、镜像垃圾、inode 和 ephemeral-storage。
- 判断内存泄漏、瞬时峰值、并发变化还是 limit 不合理。

治理包括修复应用、设置合理 requests/limits、容量告警、节点预留、临时存储限制和压力演练。不能把所有 137 都直接写成 OOM。

### 38. Pod CPU 使用率不高，但请求延迟很高，可能是什么原因？

平均 CPU 低不代表没有 CPU 问题。重点检查：

- CFS throttling：CPU limit 太紧，短时间突发被节流。
- 单线程或少数核打满，但指标被多核平均。
- run queue、上下文切换、steal、NUMA 和宿主机噪声。
- GC、锁竞争、I/O、DNS、网络重传、连接池和下游延迟。
- readiness 抖动、负载不均、冷启动或 HPA 指标滞后。

结合容器 throttled seconds、per-CPU、应用 profile、RED 指标和分布式追踪定位。简单删除 CPU limit 有时能缓解，但会扩大邻居干扰，应基于业务 SLO 和节点隔离做取舍。

### 39. Deployment 发布卡住或新版本大量 5xx，怎么处理？

先止损：暂停发布、回滚镜像／配置、切回稳定流量，保护核心业务。再检查：

- `rollout status`、Deployment condition、新旧 ReplicaSet 和 events。
- 新 Pod 的 image、配置、Secret、init、探针和资源。
- `maxSurge` 是否造成资源不足，`maxUnavailable` 是否过大。
- readiness 是否过早、优雅终止是否缺失、连接是否排空。
- 数据库迁移、缓存格式、接口兼容和下游限流。

恢复后补充自动验证、金丝雀、指标门禁、不可变制品、配置版本和回滚演练。事故复盘要解释为什么监控或发布系统没有提前阻止。

### 40. PVC 一直 Pending，或者 Pod 挂载卷失败，怎么排查？

PVC Pending 重点看 StorageClass、provisioner、容量、access mode、selector、配额和 `volumeBindingMode`。Pod 已调度后挂载失败，再看 CSI controller／node plugin、VolumeAttachment、节点权限、设备、文件系统和云盘区域。

常用路径：

```bash
kubectl describe pvc PVC -n NS
kubectl get pv,storageclass
kubectl describe pod POD -n NS
kubectl get volumeattachment
```

还要查 CSI sidecar 和 node plugin 日志。多可用区环境中，Immediate 绑定可能把卷建在错误区域；使用 `WaitForFirstConsumer` 可让存储拓扑参与调度。

### 41. API Server 很慢或 etcd 延迟升高，怎么处理？

先确认用户影响、请求类型和是否存在变更风暴，必要时暂停高频 controller、批量任务或异常客户端。然后分层检查：

- API Server 请求延迟、错误码、inflight、APF 限流、审计和 webhook 延迟。
- etcd leader 变化、WAL fsync、backend commit、proposal、DB 大小、配额和网络 RTT。
- 磁盘 IOPS／延迟、CPU steal、内存压力和文件系统。
- 大对象、全量 list、过多 watch、事件风暴和失控 Operator。

etcd compaction 和 defrag 要按官方流程安排，不能在高峰期盲目执行。修复后应给 API 客户端增加限速、分页、watch 和退避，给 webhook 设置超时及合理 failurePolicy。

### 42. 线上某节点磁盘或 inode 即将耗尽，怎样安全处理？

先判断是 nodefs、imagefs 还是容器／Pod 卷，并确认业务是否正在写数据。检查：

- 容器日志、已删除但仍被进程占用的文件。
- unused image、停止容器、emptyDir、hostPath 和本地 PV。
- inode 数量、镜像层、runtime snapshot 和 kubelet pod 目录。
- kubelet image GC、container log rotation 和 eviction 阈值。

可以先 cordon，保护业务副本并逐步清理明确可回收内容。不要直接递归删除 `/var/lib/kubelet` 或 `/var/lib/containerd`。长期要设置日志轮转、ephemeral-storage requests/limits、磁盘与 inode 告警、独立数据盘和容量趋势预测。

---

## 四、AI Infra／机器学习运维扩展：★★★★☆

### 43. Kubernetes 如何调度 GPU？Device Plugin 和 GPU Operator 分别做什么？

GPU 通常通过 Device Plugin 暴露为扩展资源，例如 `nvidia.com/gpu`。kubelet的 device manager 接收插件注册并把可用设备上报到 Node；scheduler 按 Pod 的扩展资源请求选择节点；kubelet在启动容器时调用插件 Allocate，把设备、环境变量或挂载交给运行时。

NVIDIA GPU Operator 用 Operator 管理 Driver、Container Toolkit、Device Plugin、DCGM Exporter、MIG Manager 等组件的部署与生命周期。它不等于调度器，也不会自动解决拓扑、碎片和多租户隔离。

生产上还需设计节点 taint／label、RuntimeClass、驱动与 CUDA 兼容、GPU 健康、MIG、配额、监控和故障下线。

### 44. GPU Pod 一直 Pending，或已运行却看不到 GPU，怎么排查？

Pending 时先看 `FailedScheduling`：

- GPU 扩展资源是否出现在 Node capacity／allocatable。
- Pod 请求的资源名、数量是否正确。
- device plugin 是否健康，是否有 Xid 故障导致设备下线。
- taint／toleration、node selector、affinity、MIG profile 和资源碎片。
- GPU 是否已被其他 Pod 分配，Cluster Autoscaler 是否能扩 GPU 节点组。

已运行看不到 GPU，再检查 Driver、Container Toolkit、runtime 配置、Allocate 结果、设备文件和 CUDA／driver 兼容。训练性能差还要继续看 NUMA、PCIe／NVLink、RDMA／NCCL、存储吞吐和 CPU 数据加载，不能只看 `nvidia-smi` 利用率。

---

## 五、面试官常见连续追问

### 追问链 1：你说做过 Kubernetes 生产集群

1. 集群多少节点、多少 Pod、峰值 QPS 和日常变更量？
2. control plane、etcd 和入口怎么做高可用？
3. 用什么 CNI、CSI 和运行时，为什么这样选？
4. 升级过哪些版本，发生过什么兼容问题？
5. 最严重事故是什么，影响多久，如何止损和防复发？

不要用“几十个节点左右”模糊回答。至少准备真实数量级、拓扑、SLO、职责边界和一项自己主导的改进。

### 追问链 2：你说 Service 不通时会排查

1. Pod IP 能通、ClusterIP 不通说明什么？
2. 同节点能通、跨节点不通说明什么？
3. EndpointSlice 有地址，但还是不通怎么查？
4. iptables／nftables／eBPF 数据面分别看什么？
5. conntrack 满、MTU 不一致和反向路由异常如何验证？

### 追问链 3：你说资源治理做得好

1. request 根据平均值还是 P99 设置？
2. 为什么 CPU limit 会增加尾延迟？
3. HPA 和 VPA 能否同时修改 CPU／内存？
4. 节点故障、升级和扩容延迟需要留多少余量？
5. 降本后怎样证明 SLO 没退化？

### 追问链 4：你说做过无损发布

1. readiness 成功后立即接流量会有什么风险？
2. 删除 Pod 时 EndpointSlice 和 SIGTERM 谁先完成？
3. 长连接、gRPC、消息消费者如何排空？
4. 数据库 schema 变更如何兼容回滚？
5. 发布系统依据哪些指标自动停止？

### 追问链 5：你说做过 etcd 备份

1. 多久备份一次，RPO／RTO 是多少？
2. 快照保存在哪里，怎样校验？
3. 最近一次恢复演练用了多久？
4. 恢复后的 revision 回退怎样处理？
5. 除 etcd 外还要保存哪些证书、加密配置和外部资源信息？

---

## 六、已删除或降级的题目

以下题目不作为 25K～40K 主干，但基础不能完全不会：

- “Kubernetes 是什么”“常用 kubectl 命令有哪些”：区分度太低，只用于热身。
- 标签如何添加、修改、删除、查看：合并进资源管理基础，不单独背多道命令题。
- 镜像有哪些“状态”：定义不严谨，改为 imagePullPolicy 和 ImagePullBackOff 场景。
- Deployment 回滚命令：命令本身价值低，重点改为可回滚发布设计。
- ReplicationController 与 ReplicaSet 的细碎比较：RC 已是遗留知识，知道 RS 支持 set-based selector 即可。
- PodSecurityPolicy：已从 Kubernetes 1.25 移除，改学 Pod Security Admission 和策略控制器。
- dockershim 工作原理：已从 Kubernetes 1.24 移除；保留 CRI、containerd、runc 的关系。
- “生产环境一定使用 IPVS”：已过时；Kubernetes 1.35 已将 IPVS 模式标记为弃用。
- Heapster、PetSet、Federation v1：过时或已被替代，不投入主要复习时间。
- 只问 `kubectl scale`、`expose`、`label` 的操作题：CKA 已覆盖，面试重点应升级到原理和故障。

---

## 七、面试前 10 分钟速记

1. Pod 创建：API → etcd → scheduler bind → kubelet → CRI → CNI／CSI → probe → EndpointSlice。
2. 控制器：list/watch + informer cache + work queue + idempotent reconcile。
3. Running 不等于 Ready；CrashLoopBackOff 不是 Pod phase。
4. liveness 决定重启，readiness 决定流量，startup 保护慢启动。
5. Service 先查 selector、EndpointSlice、targetPort，再查数据面。
6. CNI 管 Pod 网络；kube-proxy／eBPF 数据面管 Service 转发；CoreDNS 管服务发现。
7. scheduler 看 request；CPU limit 会 throttling；memory limit 可能 OOM。
8. OOMKilled、Evicted、137 不能混为一谈。
9. PVC Pending 先看 StorageClass、CSI、拓扑和 WaitForFirstConsumer。
10. etcd 备份必须恢复演练；关注 fsync、leader、DB 大小和 revision。
11. 升级不能跳 minor；先兼容评估、快照、灰度，再逐节点 drain。
12. IPVS 模式已弃用；回答现网时要带 Kubernetes 版本和数据面实现。

---

## 八、项目经验回答模板

建议准备一个 3 分钟 Kubernetes 项目回答：

> 我负责的生产集群大约有 **X 个集群、Y 个节点、Z 个 Pod**，承载的是 **业务类型和峰值**。控制面采用 **拓扑**，运行时是 **containerd**，网络使用 **CNI 及模式**，存储使用 **CSI／后端**。我主要负责 **升级、监控、资源治理、故障处理或平台开发**。其中一次比较有代表性的事故是 **现象**，当时影响 **范围和时长**。我先通过 **止损动作** 恢复服务，再利用 **events／metrics／logs／抓包／内核指标** 定位到 **根因**，最终通过 **修复措施** 解决，并补充 **告警、发布门禁、自动化、演练或容量策略** 防止复发。该改进把 **MTTR、资源利用率、发布失败率或成本** 从多少改善到多少。

面试前务必把 X、Y、Z、影响时长、恢复时长和改进结果换成真实数据。无法公开业务数据时可以说数量级或比例，但不要编造经历。

---

## 九、参考资料与校正依据

### 用户提供的面经

- [小红书基础平台研发一面：Pod 创建与 cgroup](https://www.nowcoder.com/feed/main/detail/a6625b64fbe74aa6bc2804c76625c226)
- [Kubernetes 面试题汇总（一）](https://www.nowcoder.com/discuss/387352444697669632)
- [Kubernetes 面试题汇总（二）](https://www.nowcoder.com/discuss/374641235858911232)
- [杭州谐云 Go 实习岗面经](https://www.nowcoder.com/feed/main/detail/eacfd1155f1048fdb5a27254d4c48456)

其中两篇汇总内容高度重复，主要用于提取基础高频，不按两份独立频次重复计权；实习面经只作为“控制面、核心对象、容器异常仍常被问”的旁证。

### 官方校正资料

- [Kubernetes Components](https://kubernetes.io/docs/concepts/overview/components/)
- [Pod Lifecycle](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/)
- [Deployments](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/)
- [Service 与 EndpointSlice](https://kubernetes.io/docs/concepts/services-networking/service/)
- [Virtual IPs and Service Proxies](https://kubernetes.io/docs/reference/networking/virtual-ips/)
- [Kubernetes 网络模型](https://kubernetes.io/docs/concepts/services-networking/)
- [Resource Management](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [StorageClass](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- [Node-pressure Eviction](https://kubernetes.io/docs/concepts/scheduling-eviction/node-pressure-eviction/)
- [Debug Services](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/)
- [kubeadm 集群升级](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-upgrade/)
- [etcd Disaster Recovery](https://etcd.io/docs/v3.5/op-guide/recovery/)

> 版本说明：官方资料以 2026 年 8 月可见的 Kubernetes 1.36 文档为主要校正依据。面试时仍要以目标公司的现网版本、CNI／CSI 和托管集群实现为准。

---

## 十、复习优先级

如果时间有限，按下面顺序：

1. 先掌握 1～15，要求不看答案能完整口述并接住两轮追问。
2. 重点练 31～42，每题都按“现象 → 止损 → 定位 → 根因 → 修复 → 防复发”回答。
3. 从自己真实经历中准备 Pod、网络、资源、发布、节点或存储事故至少 3 个。
4. 再补 16～30 的生产细节和平台治理。
5. 面向 AI Infra 岗位，最后强化 43～44，并继续学习 GPU、RDMA、NCCL 和训练／推理平台。

对你当前阶段，最重要的不是再背一遍 CKA 操作，而是让面试官相信：**你能解释 Kubernetes 为什么这样工作，也能在生产事故中快速缩小故障域、恢复业务并建立防复发机制。**
