# Docker 面试题高频精选

> 适用方向：SRE、DevOps、Kubernetes 平台、Linux 运维、AI Infra 社招  
> 目标级别：月薪 25K～40K、3～7 年经验的中高级岗位  
> 排序依据：四份参考资料的重复度、当前岗位常见度、生产实战区分度。星级是经验分级，不是严格统计。  
> 推荐答法：**先说结论 → 解释内核或运行时原理 → 给出验证命令 → 结合生产案例和预防措施**。

---

## 先看结论：这类 Docker 题怎么准备

25K～40K 的 SRE／DevOps 面试不会满足于“会 `docker run`”。面试官更关心：

1. 你是否理解容器本质：进程、namespace、cgroup、联合文件系统，而不是“小型虚拟机”。
2. 你能否写出安全、精简、可复现、可缓存的 Dockerfile。
3. 你是否能定位容器启动失败、OOM、CPU 限流、网络不通、磁盘占满等生产故障。
4. 你是否理解 Docker 与 containerd、runc、OCI、Kubernetes 的关系。
5. 你是否具备镜像供应链、最小权限和运行时安全意识。

本题库共 38 道，基础命令只保留面试真正会追问的部分。

---

## 一、必问级：★★★★★

### 1. Docker 容器与虚拟机有什么区别？

容器本质上是宿主机上的普通进程，通过 namespace 获得隔离视图，通过 cgroup 做资源统计和限制，并共享宿主机内核。虚拟机通过 Hypervisor 虚拟硬件，每个虚拟机通常运行独立的 Guest OS 和内核。

- 容器启动快、密度高、镜像小，适合应用交付和弹性扩缩。
- 虚拟机隔离边界通常更强，可以运行不同内核和操作系统。
- 容器不是绝对安全边界；内核漏洞、错误权限和挂载都可能扩大风险。
- 生产环境经常是“虚拟机上运行容器”，两者不是互斥关系。

不要回答“容器没有操作系统”。更准确地说，镜像可以包含用户态文件和工具，但容器共享宿主机内核。

### 2. Docker 的整体架构是什么？一次 `docker run` 发生了什么？

Docker 采用客户端／服务端架构。CLI 调用 Docker Engine API，`dockerd` 负责镜像、网络、卷和容器管理，并通过 containerd 管理容器生命周期，containerd 再调用 OCI 运行时（常见为 runc）创建容器进程。

`docker run` 可以拆成：

1. 解析镜像名和运行参数，本地无镜像时从 Registry 拉取 manifest 和缺失层。
2. 基于镜像创建容器可写层，准备 OCI runtime 配置。
3. 创建 namespace、cgroup、挂载点、网络接口及安全策略。
4. 启动容器入口进程，接入日志、标准输入输出和健康检查。
5. 容器主进程退出后，容器进入停止状态；是否重启由重启策略或编排系统决定。

### 3. namespace 和 cgroup 分别解决什么问题？

namespace 解决“看见什么”，cgroup 解决“能用多少、用了多少”。

- PID namespace：进程编号和进程树隔离。
- Network namespace：网卡、路由、端口、协议栈隔离。
- Mount namespace：挂载点和文件系统视图隔离。
- UTS、IPC、User namespace：主机名、进程间通信和 UID/GID 映射隔离。
- cgroup：对 CPU、内存、I/O、PIDs 等资源做统计、优先级和限制。

它们提供的是隔离和控制，不等于虚拟出一套新内核。

### 4. 镜像、容器和镜像层是什么关系？

镜像是不可变的只读层集合及其配置元数据；容器是镜像加一个可写层后运行起来的进程环境。多个容器可以共享同一组只读镜像层，但拥有各自的可写层。

镜像层通常按内容寻址，可被缓存和复用。容器删除后，可写层中的数据会丢失；需要保留的数据应写入 volume、bind mount 或外部存储。

严格来说，镜像不只是“容器模板”，还包括入口命令、环境变量、工作目录等配置。

### 5. Docker 镜像为什么能分层？OverlayFS 的写时复制怎么工作？

Docker 常用 `overlay2` 存储驱动把多个只读 lowerdir 和一个可写 upperdir 组合成统一 merged 视图。

- 读取未修改文件时直接从下层读取。
- 首次修改下层文件时先 copy-up 到可写层，再修改副本。
- 删除下层文件时用 whiteout 标记遮蔽，而不是修改原层。
- 每条 Dockerfile 指令不一定都产生文件系统层，但会影响构建历史和缓存。

因此在容器可写层写大量随机数据、频繁修改大文件可能产生额外开销，数据库数据更适合放到独立存储。

### 6. 如何优化 Dockerfile 和镜像？

重点不是机械地减少指令条数，而是降低体积、漏洞面、构建时间并保证可复现：

1. 选择可信且精简的基础镜像，生产镜像不携带编译器和无关工具。
2. 使用多阶段构建，只复制最终二进制或运行所需文件。
3. 使用 `.dockerignore` 排除 `.git`、日志、密钥、依赖缓存等无关内容。
4. 把变化少的依赖文件放在前面，源码放后面，提高缓存命中率。
5. 安装软件后在同一层清理包索引和临时文件。
6. 使用非 root 用户，明确 `WORKDIR`、`USER`、`ENTRYPOINT`。
7. 在 CI 中使用 BuildKit 缓存、漏洞扫描、SBOM 和镜像签名。
8. 发布时优先用不可变 digest 或唯一版本标签，不依赖漂移的 `latest`。

### 7. `CMD` 和 `ENTRYPOINT` 有什么区别？shell 格式和 exec 格式又有什么区别？

- `ENTRYPOINT` 定义容器主要可执行程序。
- `CMD` 提供默认命令，或作为 `ENTRYPOINT` 的默认参数。
- `docker run IMAGE 参数` 会覆盖 `CMD`；覆盖 `ENTRYPOINT` 通常要显式使用 `--entrypoint`。

推荐 exec 格式：

```dockerfile
ENTRYPOINT ["/app/server"]
CMD ["--config", "/etc/app/config.yaml"]
```

shell 格式会经过 `/bin/sh -c`，shell 可能成为 PID 1，导致信号转发和退出码行为与预期不同。需要变量展开或管道时可以用 shell，但应正确使用 `exec`。

### 8. `COPY` 和 `ADD` 有什么区别？

`COPY` 只负责把构建上下文中的文件复制到镜像，语义清晰，通常优先使用。`ADD` 还有额外行为，例如对本地 tar 自动解包，并可接受特定远程源；额外语义容易造成不可预测和缓存问题。

下载远程文件更推荐在构建步骤中使用明确工具，并校验哈希；如果确实需要自动解包本地归档，再考虑 `ADD`。二者都不能复制 `.dockerignore` 已排除或构建上下文之外的任意文件。

### 9. 为什么容器主进程结束后容器就退出？PID 1 有什么特殊问题？

容器生命周期绑定其入口进程，而不是绑定某个“完整操作系统”。入口进程退出，容器就停止。

Linux 中 PID 1 需要正确处理终止信号并回收孤儿子进程。应用若不转发 `SIGTERM`、不执行 `wait`，可能导致优雅退出失败或僵尸进程积累。常见解决方式：

- 让应用直接作为 PID 1，并正确处理信号和子进程。
- 启动脚本最后使用 `exec` 替换 shell。
- 必要时使用 `docker run --init` 引入轻量 init。

### 10. `docker stop` 和 `docker kill` 有什么区别？如何实现优雅退出？

`docker stop` 默认先向 PID 1 发送 `SIGTERM`，等待超时后再发 `SIGKILL`；`docker kill` 默认直接发送 `SIGKILL`，也可以指定其他信号。

优雅退出需要：

1. 应用捕获 `SIGTERM`，停止接收新请求。
2. 等待进行中的请求、事务或任务完成。
3. 关闭连接、刷盘并退出。
4. 停止超时要覆盖业务最大合理处理时间，但不能无限等待。

如果 Dockerfile 使用了错误的 shell 入口，即使应用写了退出逻辑，也可能收不到信号。

### 11. Docker bridge 网络如何通信？`-p 8080:80` 的流量路径是什么？

容器通常拥有独立 network namespace，一端 veth 在容器内，另一端接到宿主机网桥。容器出网通过宿主机路由和 SNAT/MASQUERADE；发布端口时，Docker 配置 DNAT／转发规则，把宿主机 `8080` 的流量转到容器 IP 的 `80`。

排查时按路径检查：

```bash
docker inspect CONTAINER
docker network inspect NETWORK
ip link
ip route
ss -lntp
iptables-save
# 使用 nftables 后端时同时检查 nft 规则
nft list ruleset
```

`-p 8080:80` 默认可能绑定宿主机所有地址；只需本机访问时应指定 `127.0.0.1:8080:80`，减少暴露面。

### 12. 为什么容器里访问 `127.0.0.1` 连不到宿主机或另一个容器？容器 DNS 怎么工作？

每个 network namespace 都有自己的 loopback，容器内的 `127.0.0.1` 只指向该容器。访问其他容器应加入同一个用户自定义网络并使用服务名；访问宿主机应使用明确的宿主机地址或平台提供的 host gateway 方案。

用户自定义 bridge 和 Compose 网络通常由 Docker 内置 DNS 提供容器名／服务名解析。排查顺序：

```bash
cat /etc/resolv.conf
getent hosts SERVICE
ip addr
ip route
ss -lntp
```

还要确认应用监听的是 `0.0.0.0`，而不是仅监听容器内 `127.0.0.1`。

### 13. Volume、Bind Mount 和 tmpfs 有什么区别？

- Volume：由 Docker 管理生命周期和路径，适合持久化应用数据，便于迁移和备份。
- Bind Mount：把宿主机指定路径直接挂入容器，适合配置、开发源码或必须访问的宿主资源，但与宿主目录结构耦合且风险更高。
- tmpfs：数据只在宿主机内存中，不写入容器可写层，容器停止后消失，适合临时敏感数据或缓存。

生产中还要考虑 UID/GID、只读挂载、SELinux 标签、备份一致性和存储性能。不要把“挂载成功”等同于“应用有权限读写”。

### 14. Docker 如何限制 CPU 和内存？OOM 时发生什么？

- `--memory` 设置内存上限。
- `--memory-reservation` 可表达软性预留或压力下的目标。
- `--cpus` 或 CFS quota 限制可使用的 CPU 时间。
- `--cpu-shares` 是竞争发生时的相对权重，不是固定核数保证。
- `--pids-limit` 防止进程数量失控。

容器达到 cgroup 内存上限时可能触发 cgroup OOM，内核选择进程杀死；容器常见退出码是 137，但 137 只表示收到 `SIGKILL`，不能单凭它断定 OOM。应结合 `docker inspect`、内核日志和 cgroup 事件确认。

### 15. Kubernetes 已不使用 dockershim，Docker 还有什么价值？

Kubernetes 从 1.24 起移除了内置 dockershim，kubelet 通过 CRI 对接 containerd、CRI-O 等运行时。这不代表 Docker 镜像不能在 Kubernetes 使用，也不代表 Docker 失去价值。

- Docker／BuildKit 仍常用于本地开发、CI 构建、调试和 Compose 环境。
- Docker 构建出的 OCI／兼容镜像可以由 containerd、CRI-O 拉取和运行。
- 面试时需要把 Docker Engine、containerd、runc、CRI 和 OCI 区分清楚。

对 SRE 岗位，Docker 是容器基础；生产 Kubernetes 运行时和 CRI 才是更需要深入的下一层。

---

## 二、高频级：★★★★☆

### 16. `docker run`、`create`、`start`、`exec`、`attach` 有什么区别？

- `run`：本质上是 create + start，并可同时配置网络、挂载和资源限制。
- `create`：只创建容器，不启动入口进程。
- `start`：启动已创建或已停止的容器，沿用原配置。
- `exec`：在运行中的容器里启动一个新进程，常用于排障。
- `attach`：连接容器主进程的标准输入输出，不会创建新进程，误操作可能影响主进程。

生产排障优先使用 `exec`，但精简镜像可能没有 shell 和调试工具。

### 17. 镜像 tag 和 digest 有什么区别？为什么生产不建议依赖 `latest`？

tag 是可变的人类可读引用，例如 `app:1.2`；digest 是由镜像内容计算出的不可变标识，例如 `sha256:...`。同一 tag 可以被重新推送并指向不同内容，导致回滚和审计困难。

生产发布应使用唯一版本号、Git SHA，必要时锁定 digest。`latest` 只是普通 tag，不代表最新、不保证自动更新，也不能证明镜像内容未被替换。

### 18. 什么是 BuildKit？怎样提高构建速度和可复现性？

BuildKit 支持并行构建、改进缓存、按需传输上下文、缓存挂载和构建期 secret 等能力。

实践要点：

- 合理排列 Dockerfile，让依赖层稳定、源码层后置。
- 使用 registry cache 或 CI 远程缓存。
- 用 `RUN --mount=type=cache` 缓存包管理器目录。
- 用 `RUN --mount=type=secret` 注入构建期密钥，避免写入层和构建参数。
- 锁定依赖版本、基础镜像 digest，并输出 SBOM／构建证明。

### 19. Docker Compose 的 `depends_on` 能否保证依赖服务已经可用？

仅指定启动顺序不等于服务已经就绪。现代 Compose 可以结合依赖服务的 `healthcheck` 和 `condition: service_healthy` 等条件，但应用仍应具备连接重试、超时和退避能力。

数据库进程启动不代表已经完成恢复、迁移或可以接受业务请求。生产就绪判断应基于真实健康信号，而不是 `sleep 10`。

### 20. `HEALTHCHECK` 有什么用？它会自动重启不健康容器吗？

`HEALTHCHECK` 周期性执行探测并把容器标记为 `starting`、`healthy` 或 `unhealthy`，便于观测和编排决策。探测应轻量、带超时，并验证关键依赖但避免把所有外部依赖都硬绑定进去。

Docker Engine 的重启策略主要根据容器进程退出判断，单纯变成 `unhealthy` 不会自动重启容器。是否摘流、重建或重启，需要 Compose 周边机制、Swarm、Kubernetes 或监控自动化处理。

### 21. Docker 重启策略有哪些？什么时候会失效？

常见策略有 `no`、`on-failure[:max-retries]`、`always`、`unless-stopped`。

- `on-failure` 只对非零退出码生效，不处理守护进程整体宕机等所有情况。
- `always` 和 `unless-stopped` 在 daemon 重启后的行为不同，后者会尊重显式停止状态。
- 重启策略不能替代健康检查，也不能修复错误配置；无限重启会掩盖根因并制造日志风暴。

### 22. `docker logs` 的日志来自哪里？如何防止日志撑满磁盘？

`docker logs` 读取容器主进程写到 stdout／stderr、再由 Docker logging driver 收集的内容，并不会自动读取容器内任意日志文件。

常见事故是默认 `json-file` 日志无限增长。应配置日志轮转，例如限制 `max-size`、`max-file`，或使用适合轮转的本地驱动；生产再接入 Fluent Bit、Loki、Elasticsearch 等集中系统。日志采集端也要有限速、缓冲、背压和磁盘保护。

### 23. 如何观察 Docker 容器资源和运行状态？

常用入口：

```bash
docker ps -a
docker inspect CONTAINER
docker stats --no-stream
docker top CONTAINER
docker logs --tail 200 CONTAINER
docker events --since 30m
docker system df -v
```

`stats` 适合快速观察，精确分析要继续查看宿主机进程、cgroup v2 指标、内核日志和应用监控。`inspect` 中的配置是证据，不能只依赖启动命令记忆。

### 24. 容器为什么不建议默认以 root 运行？rootless 和 user namespace 有什么作用？

默认情况下，容器内 root 权限虽受 namespace、capabilities 和 seccomp 约束，但错误挂载、内核漏洞或附加权限仍可能扩大为宿主机风险。

- Dockerfile 使用 `USER` 让应用以非 root 运行。
- User namespace 可把容器内 UID 映射到宿主机非特权 UID。
- Rootless Docker 让 daemon 和容器都由普通用户运行，进一步缩小 daemon 被攻破后的权限。

落地时要处理低端口、cgroup、存储驱动、宿主目录 UID/GID 和网络能力差异。

### 25. `--privileged`、capabilities、seccomp、AppArmor／SELinux 分别做什么？

`--privileged` 会显著放宽设备和内核能力限制，接近把宿主机控制权交给容器，生产中应极力避免。

- Linux capabilities：把传统 root 权限拆分成可独立授予的能力；应默认 drop，再按需 add。
- seccomp：限制可调用的系统调用。
- AppArmor／SELinux：通过强制访问控制限制进程对文件、能力等对象的访问。
- 只读根文件系统、非 root、受限挂载共同形成纵深防御。

### 26. 为什么挂载 `/var/run/docker.sock` 风险很高？

能访问 Docker socket 的进程通常可以调用 daemon 创建特权容器、挂载宿主机根目录、读取其他容器 secret，实际接近宿主机 root 权限。

不要因为 CI 或管理工具方便就直接挂载 socket。可使用隔离构建器、受限代理、最小权限 API、独立构建节点或无守护进程构建方案，并对访问进行审计。

### 27. 镜像安全应该怎么做？

完整链路包括：

1. 只用可信基础镜像，固定版本或 digest。
2. 最小化包、用户和攻击面，不把密钥写入 ENV、ARG 或镜像层。
3. 构建时生成 SBOM，扫描 OS 包和应用依赖漏洞。
4. 对镜像签名并在部署入口验证来源和策略。
5. 定期重建镜像以纳入基础镜像补丁，而不是进入运行中容器手工打补丁。
6. 运行时采用非 root、只读文件系统、capability drop、seccomp 和网络隔离。

### 28. `docker save/load` 与 `docker export/import` 有什么区别？

- `save/load` 面向镜像，保留镜像层、标签和历史，适合离线搬运镜像。
- `export/import` 面向容器文件系统快照，会扁平化文件系统，不保留原有镜像层和大部分镜像配置。

常规迁移和备份优先使用 registry 或 `save/load`；不要用 `export/import` 替代规范镜像构建。

---

## 三、生产场景题：★★★★☆～★★★★★

### 29. 容器启动后立即退出，怎么排查？

先查退出码、入口命令和日志：

```bash
docker ps -a --no-trunc
docker inspect CONTAINER
docker logs --tail 200 CONTAINER
```

重点检查：

- 主进程是否正常执行完毕，是否错误地把服务放到后台。
- `ENTRYPOINT`／`CMD` 是否被覆盖，脚本是否有执行权限、shebang 和正确换行。
- 配置、环境变量、挂载文件、DNS、依赖和端口是否正确。
- 退出码：126 常见为不可执行，127 常见为命令不存在，137 表示 SIGKILL，139 常见为段错误，143 表示 SIGTERM。

需要交互排查时可覆盖入口启动 shell；distroless 镜像没有 shell，应使用独立调试容器或在宿主机进入相应 namespace。

### 30. 容器发生 OOM 或频繁退出码 137，如何定位？

1. 用 `docker inspect` 看 `OOMKilled`、退出时间和资源配置。
2. 看 `dmesg`／journal 和 cgroup 的 memory events，确认系统 OOM 还是 cgroup OOM。
3. 对比应用 RSS、匿名内存、Page Cache、共享内存和限制值。
4. 检查是否有内存泄漏、缓存无上限、并发增长、JVM／Go 运行时未感知 cgroup 限制。
5. 先止损：降并发、扩容、合理调高限制或回滚；再修复内存模型和容量规划。

不要直接把 limit 调到很大，否则可能把容器级问题升级为节点级 OOM。

### 31. 容器 CPU 使用不高，但请求延迟明显上升，怎么排查？

要考虑 CPU quota throttling。容器可能在允许的时间窗口内用满配额后被节流，平均 CPU 看起来不高，P99 延迟却升高。

检查 cgroup 的 CPU throttling 指标、宿主机 run queue、单核热点、上下文切换和应用线程池；同时确认 `--cpus`、quota、shares 与实际并发模型。对延迟敏感服务，应结合峰值并发设置资源，而不是只看日均 CPU。

### 32. 宿主机磁盘被 Docker 占满，如何安全处理？

先确定占用来源，不直接执行全局 prune：

```bash
df -h
df -i
docker system df -v
du -xhd1 /var/lib/docker
```

常见来源包括容器日志、未使用镜像和构建缓存、停止容器、匿名卷、容器可写层。处理原则：

1. 先轮转或治理日志。
2. 明确哪些镜像、缓存、容器和卷不再使用。
3. 备份并确认卷归属后再删除。
4. 建立容量告警、保留策略和定期清理机制。

`docker system prune -a --volumes` 可能删除仍有价值的数据，生产环境不能无确认执行。

### 33. 容器端口已经 `-p` 发布，但外部访问不通，怎么排查？

按层验证：

1. 容器是否运行、应用是否监听正确端口和 `0.0.0.0`。
2. `docker port`／`inspect` 中 HostIP、HostPort 是否符合预期。
3. 宿主机 `ss`、本机 curl 是否成功。
4. Docker bridge、veth、路由、DNAT／转发规则是否存在。
5. 宿主机防火墙、安全组、云 LB、ACL 是否放行。
6. 是否存在端口冲突、反向代理 upstream 配错、IPv4／IPv6 不一致。

抓包可同时在宿主机入口网卡、bridge 和容器 namespace 观察报文在哪一段消失。

### 34. 容器能访问 IP，但域名解析失败，怎么排查？

这通常是 DNS 链路问题：

```bash
cat /etc/resolv.conf
getent hosts example.com
docker inspect CONTAINER
docker network inspect NETWORK
```

检查 Docker 内置 DNS、宿主机上游 DNS、UDP/TCP 53、防火墙、conntrack、搜索域和 `ndots`。若只在高并发下偶发，重点看 conntrack、DNS 超时重试、上游容量和网络丢包。不要把宿主机 `/etc/resolv.conf` 直接覆盖进所有容器而忽略 Docker DNS 转发逻辑。

### 35. 容器跨主机通信偶发超时或大包失败，如何定位？

优先怀疑 MTU、隧道封装、分片、PMTUD 和防火墙丢弃 ICMP。表现可能是小包正常、TLS 握手或大响应卡住。

```bash
ip link
ip route
ping -M do -s SIZE TARGET
tcpdump -ni any host TARGET
ss -ti
```

对比宿主机、Docker bridge、VPN/VXLAN 等链路 MTU，检查 TCP 重传和 MSS。修复应统一底层与容器网络 MTU，或正确配置 MSS clamp，而不是仅靠应用重试掩盖问题。

### 36. 镜像构建非常慢、镜像又很大，怎么优化？

先用构建日志和镜像历史定位慢步骤及大层，再处理：

- `.dockerignore` 缩小上下文。
- 多阶段构建分离编译和运行环境。
- 依赖清单前置，源码后置，减少缓存失效。
- 使用 BuildKit cache mount 和远程缓存。
- 避免先写入大文件、再在后续层删除；删除不会让旧层变小。
- 检查基础镜像架构、包源、代理和 registry 延迟。

优化后应比较构建耗时、传输字节、最终镜像体积和漏洞数量，而不只是看层数。

### 37. “本机能运行，发布到服务器就失败”，你怎么处理？

比较环境差异而不是盲目重建：

1. 镜像 digest、CPU 架构、内核能力和 Docker／containerd 版本。
2. 环境变量、secret、配置文件、时区、证书和 DNS。
3. Volume 内容、UID/GID、只读文件系统、SELinux／AppArmor。
4. 端口、网络出口、代理和外部依赖。
5. 资源限制、文件描述符、共享内存和磁盘空间。

改进方式是让 CI 构建一次并按 digest 晋级各环境，配置外置且版本化，启动前校验配置，并建立与生产相近的预发布验证。

### 38. 线上容器内需要紧急改配置或打补丁，应该怎么做？

原则上不要把运行中容器当长期服务器维护。容器内手工修改只存在于该容器可写层，重建后会丢失，也无法审计和规模化复制。

正确流程是：

1. 先用回滚、切流或临时受控配置止损。
2. 在代码、配置仓库或 Dockerfile 中修复并评审。
3. CI 重新构建、扫描并发布新镜像。
4. 灰度验证后滚动替换，保留旧 digest 以便回滚。
5. 复盘为什么发布链路不能快速完成这一过程。

只有在明确记录、限定范围且随后会被正式版本覆盖时，才考虑临时进入容器操作。

---

## 四、面试官常见连续追问

### 追问链 1：你说容器很轻量

1. 轻量具体轻在哪里？
2. 既然共享内核，隔离由什么实现？
3. root 进程能否看到宿主机进程？
4. `--privileged` 后隔离发生了什么变化？
5. 容器逃逸可能经过哪些边界？

### 追问链 2：你说做过镜像优化

1. 原镜像多大，优化后多大？
2. 最大的一层是什么，如何确认？
3. 为什么后续 `rm` 文件，镜像仍然没有变小？
4. 如何兼顾调试便利和生产镜像攻击面？
5. 如何保证基础镜像升级后可以复现和回滚？

### 追问链 3：你说处理过容器网络故障

1. 报文从客户端到容器经历哪些设备和规则？
2. 如何进入容器 network namespace 抓包？
3. 为什么容器内访问 `localhost` 不到宿主机？
4. 只有大包失败为什么要怀疑 MTU？
5. conntrack 满了会有什么现象，如何确认？

### 追问链 4：你说容器被 OOM Kill

1. 如何证明是 cgroup OOM，不是人工 `kill -9`？
2. 容器内 `free` 和宿主机指标为什么可能不一致？
3. Page Cache 是否计入容器内存？
4. 为什么只调高 limit 可能让节点更危险？
5. 在 Kubernetes 中 requests、limits、QoS 又如何影响结果？

---

## 五、已删除或降级的题目

以下题目在参考资料中出现过，但不建议占用主复习时间：

| 题目 | 处理 | 原因 |
|---|---|---|
| Docker 能运行在哪些旧版 Linux／云平台 | 删除 | 列表明显过时，几乎没有岗位区分度 |
| Compose 能不能使用 JSON | 删除 | 边缘语法题，实际价值低 |
| 如何一次停止／删除全部容器 | 删除 | 命令可随时查询，不能体现中高级能力 |
| 如何退出交互终端而不停止容器 | 删除 | 初级操作题 |
| Docker 容器究竟有“几种状态” | 降级 | 容易陷入枚举；应会用 inspect 根据状态排障 |
| 如何修改已创建容器的端口／目录映射配置文件 | 删除 | 直接改 Docker 内部元数据风险高；应声明式重建 |
| 用 `docker commit` 保存线上修改 | 删除 | 不可复现、不可审计，不是规范交付流程 |
| Docker Hub／官方镜像默认端口 | 删除 | 题意本身混乱，不具备通用答案 |
| Windows／非 Linux 如何运行 Docker | 降级 | 除非岗位明确涉及 Windows 容器或桌面开发 |
| Hypervisor 的分类 | 降级 | 属于虚拟化基础，不是 Docker 专题重点 |
| Docker Swarm 详细命令与调度 | 降级 | 当前目标岗位更重 Kubernetes；只需知道定位和基本特性 |
| 孤儿卷的定义及一条命令删除 | 合并 | 已并入磁盘占满与安全清理场景 |
| `docker cp` 等零散命令 | 删除 | 现场可查，不足以作为 25K～40K 主问题 |

参考资料里“直接修改容器配置文件”“用 commit 固化修改”“healthy 是运行状态之一”等表述容易误导生产实践，本题库未原样保留。

---

## 六、面试前 10 分钟速记

- 容器 = 普通进程 + namespace 隔离视图 + cgroup 资源控制 + 分层文件系统。
- 镜像只读层共享，容器增加可写层；持久数据不要放可写层。
- PID 1 要接收信号、回收子进程；启动脚本最后用 `exec`。
- `stop` 先 TERM 后 KILL；137 不是 OOM 的唯一证据。
- `ENTRYPOINT` 是主程序，`CMD` 是默认命令或默认参数。
- `COPY` 默认优先；多阶段构建、`.dockerignore`、BuildKit 缓存是镜像优化重点。
- `latest` 是可变 tag；发布用唯一版本或 digest。
- bridge 网络核心是 network namespace、veth、网桥、路由和 NAT。
- 容器里的 `127.0.0.1` 只代表自己；应用对外服务通常监听 `0.0.0.0`。
- Volume 由 Docker 管理；Bind Mount 与宿主路径耦合；tmpfs 不落盘。
- CPU shares 是竞争权重，quota 才是时间上限；延迟高要关注 throttling。
- `HEALTHCHECK` 标记不健康，不等于 Docker 一定自动重启。
- Docker socket 基本等价于宿主机高权限入口；慎用 privileged。
- Docker 磁盘满先查日志、镜像、构建缓存、可写层和卷，再定向清理。
- K8s 移除的是 dockershim，不是 OCI 镜像，也不是所有 Docker 工具价值。

---

## 七、项目经验回答模板

不要说：

> 我负责 Docker 部署，写 Dockerfile，用 Compose 启动服务。

建议说：

> 我负责把一组传统部署服务容器化并接入 CI/CD。最初镜像包含完整构建环境，单个约 X GB，构建平均 X 分钟；我通过多阶段构建、依赖缓存、`.dockerignore` 和基础镜像治理，把镜像降到 X MB、构建时间降到 X 分钟。运行侧统一使用非 root、只读挂载、CPU／内存限制和日志轮转，并把退出码、OOM、重启次数、磁盘和健康检查接入监控。上线后曾遇到一次 X 故障，我先通过 X 止损，再用 inspect、cgroup 指标和抓包定位为 X，最后通过 X 修复，并增加 X 告警／准入策略防止复发。

回答中尽量给出真实数字：镜像大小、构建时间、发布耗时、容器数量、故障恢复时间和资源节省比例。

---

## 八、参考资料与校正依据

用户提供资料：

1. [掘金：排名前 20 的 Docker 面试问题（附答案）](https://juejin.cn/post/7088894767047639054)
2. [ApacheCN：Docker 面试题（爪哇程序员）](https://github.com/apachecn/baguwen-wiki/blob/master/docs/Docker-%E9%9D%A2%E8%AF%95%E9%A2%98%EF%BC%88%E7%88%AA%E5%93%87%E7%A8%8B%E5%BA%8F%E5%91%98%EF%BC%89.md)
3. [牛客：Docker 面试题总结（配答案）](https://www.nowcoder.com/discuss/352905402784264192)
4. [牛客：Docker 面试题](https://www.nowcoder.com/discuss/829817359517888512)

机制与时效性校正优先参考：

- [Docker Engine 官方文档](https://docs.docker.com/engine/)
- [Dockerfile 官方参考](https://docs.docker.com/reference/dockerfile/)
- [Docker 网络官方文档](https://docs.docker.com/engine/network/)
- [Docker 存储官方文档](https://docs.docker.com/engine/storage/)
- [Docker 安全官方文档](https://docs.docker.com/engine/security/)
- [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec)
- [Kubernetes：Dockershim 移除 FAQ](https://kubernetes.io/blog/2022/02/17/dockershim-faq/)

---

## 九、复习优先级

如果时间有限，按以下顺序：

1. 先熟练回答 1～15，尤其是 namespace/cgroup、镜像层、PID 1、网络、资源限制。
2. 再把 29～38 的场景题练成“现象 → 止损 → 定位 → 根因 → 修复 → 防复发”。
3. 从自己的生产经历中准备至少 3 个容器案例：镜像优化、资源故障、网络或磁盘故障。
4. 最后复习 16～28 的工具、安全和工程化细节。

对于你的目标岗位，Docker 专题不需要背 100 道题。把这 38 道讲深，并能自然衔接 containerd、Kubernetes、Linux 网络和 SRE 故障处理，价值更高。
