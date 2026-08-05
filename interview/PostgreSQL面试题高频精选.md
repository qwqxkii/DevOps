# PostgreSQL 面试题高频精选

> 适用方向：Linux 运维、数据库运维、DevOps、SRE 初中级岗位  
> 排序依据：公开面试资料的重复度、运维岗位实际常见度及实操区分度。频率是经验分级，不是严格统计。  
> 建议：先掌握前 15 题；面试回答遵循“原理 → 命令/现象 → 排查或实践”的结构。

## 一、必问级：★★★★★

### 1. PostgreSQL 是什么？它有什么特点？

PostgreSQL 是开源的对象关系型数据库管理系统，强调 SQL 标准、事务一致性、可靠性和可扩展性。它支持 ACID、MVCC、复杂 SQL、多种索引、JSON/JSONB、扩展、自定义类型、流复制和时间点恢复等。

与只会背“开源、稳定”相比，面试中更好的回答是：PostgreSQL 适合对复杂查询、数据一致性和扩展能力要求较高的业务；运维上重点关注连接数、WAL、Vacuum、锁、复制延迟和备份恢复。

### 2. PostgreSQL 的进程架构是什么？

PostgreSQL 采用多进程架构。主进程 `postgres` 监听连接，每个客户端连接通常对应一个后端进程；此外还有 checkpointer、background writer、WAL writer、autovacuum launcher/worker、archiver 等后台进程。

- shared buffers：缓存数据页。
- WAL buffers：暂存 WAL 记录。
- 操作系统页缓存：PostgreSQL 同样依赖它，不能只看 `shared_buffers`。
- 每连接进程模型会产生内存和调度开销，因此高并发下常配合 PgBouncer。

### 3. 什么是 MVCC？它解决了什么问题？

MVCC 是多版本并发控制。PostgreSQL 更新一行时通常不会原地覆盖，而是产生新版本；每个事务依据快照判断哪些行版本可见。这样读操作通常不会阻塞写操作，写操作也通常不会阻塞普通读。

行版本包含 `xmin`、`xmax` 等事务信息。旧版本在不再被任何事务需要后，由 `VACUUM` 回收。MVCC 减少了读写冲突，但会产生死元组，因此长事务和 Vacuum 异常会导致表膨胀。

### 4. PostgreSQL 的事务隔离级别有哪些？

支持 Read Uncommitted、Read Committed、Repeatable Read、Serializable，但 PostgreSQL 的 Read Uncommitted 实际按 Read Committed 执行。

- Read Committed：默认级别，每条语句获取新快照，可能出现不可重复读和幻读。
- Repeatable Read：事务内使用稳定快照；PostgreSQL 可防止幻读，但并发更新可能报序列化失败。
- Serializable：使用 SSI 检测危险依赖，效果近似事务串行执行；应用必须能够重试失败事务。

隔离级别越高不等于越好，要在一致性、并发度和重试成本之间权衡。

### 5. 什么是 WAL？有什么作用？

WAL 是预写式日志：修改数据页前，描述该修改的 WAL 记录必须先持久化。宕机后可通过 WAL 重放恢复已提交事务，从而保证持久性并减少每次提交都刷整个数据页的成本。

WAL 还用于物理流复制、归档和 PITR。重点参数和对象包括 `wal_level`、`max_wal_size`、`archive_mode`、`archive_command`、复制槽以及 `pg_wal` 目录。不能通过直接删除 `pg_wal` 文件解决磁盘满问题。

### 6. VACUUM、VACUUM FULL 和 ANALYZE 有什么区别？

- `VACUUM`：标记并回收可复用的死元组空间，维护可见性映射，并防止事务 ID 回卷；普通 Vacuum 通常不把空间归还操作系统。
- `VACUUM FULL`：重写整张表并归还空间，但需要强排他锁，耗时且需要额外磁盘空间。
- `ANALYZE`：采样数据并更新优化器统计信息。
- `VACUUM (ANALYZE)`：同时执行清理和统计信息收集。

生产环境主要依靠 autovacuum，不能因为有 autovacuum 就完全不监控。

### 7. Autovacuum 的作用是什么？为什么会“跟不上”？

Autovacuum 自动执行 Vacuum 和 Analyze，回收死元组、更新统计信息，并避免事务 ID 回卷。跟不上常见原因包括：更新删除量过大、阈值设置不合适、worker 数不足、I/O 限速、长事务阻止旧版本回收或某张大表长期占用 worker。

常查：

```sql
SELECT relname, n_live_tup, n_dead_tup,
       last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

对于高频更新的大表，可单表调低 `autovacuum_vacuum_scale_factor`，而不是盲目全局调参。

### 8. PostgreSQL 常见索引类型及适用场景是什么？

- B-tree：默认类型，适合等值、范围、排序和前缀匹配。
- Hash：主要用于等值查询，实际使用通常少于 B-tree。
- GIN：适合数组、全文检索、JSONB 包含关系；查询快但写入和维护成本较高。
- GiST：适合几何、范围、近邻搜索等可扩展场景。
- BRIN：存储数据块摘要，适合超大且与物理顺序高度相关的表，如按时间递增的数据。
- SP-GiST：适合可分区搜索结构，如四叉树、前缀树。

索引不是越多越好，会增加写入、Vacuum、磁盘和缓存成本。

### 9. 哪些情况下索引可能不生效？

常见原因：表太小、查询返回比例过高、统计信息不准、列上使用函数但没有表达式索引、隐式类型转换、复合索引不满足最左侧条件、`LIKE '%xxx'`、排序规则或代价估算使优化器认为顺序扫描更便宜。

不要仅凭“没有走索引”判定故障，应使用：

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
```

`ANALYZE` 会真正执行语句；对 `UPDATE`、`DELETE` 等应在可回滚事务或测试环境谨慎使用。

### 10. 如何分析一条慢 SQL？

建议顺序：确认 SQL 与参数 → 看执行计划 → 检查估算行数与实际行数偏差 → 判断扫描、连接和排序方式 → 看 Buffer 命中与磁盘读取 → 检查锁等待、统计信息、表膨胀和系统 I/O。

常用手段：

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) SELECT ...;
SELECT * FROM pg_stat_activity;
SELECT * FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 10;
```

优化可能包括改写 SQL、补充或调整索引、更新统计信息、减少返回行、修复类型不匹配，最后才是调参数。`pg_stat_statements` 需预加载并创建扩展。

### 11. PostgreSQL 有哪些锁？如何排查阻塞？

常见有表级锁、行级锁、页级锁、咨询锁和内部轻量锁。普通 `SELECT` 获取 `ACCESS SHARE`；DDL 往往需要更强的锁。行锁主要用于协调并发修改，同一行上的写操作会相互等待。

排查阻塞关系：

```sql
SELECT pid, usename, state, wait_event_type, wait_event,
       pg_blocking_pids(pid) AS blocking_pids, query
FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```

先找到阻塞源和业务影响，再决定取消 SQL `pg_cancel_backend(pid)` 或终止会话 `pg_terminate_backend(pid)`；不能看到锁就直接 kill。

### 12. 什么是死锁？如何处理和避免？

死锁是多个事务形成循环等待。PostgreSQL 检测到后会中止其中一个事务，应用通常收到 `deadlock detected`。

避免方法：所有事务按相同顺序访问资源；事务尽量短；不要在事务中等待人工操作或外部接口；建立合适索引，减少锁住的行和持续时间；应用捕获异常后重试。可结合数据库日志和 `deadlock_timeout` 分析锁链。

### 13. PostgreSQL 如何进行备份？pg_dump 和物理备份有什么区别？

- 逻辑备份：`pg_dump` 备份单库对象和数据，`pg_dumpall` 可导出全局对象；跨小版本和部分大版本迁移较灵活，但大库备份恢复较慢。
- 物理备份：`pg_basebackup` 或备份工具复制整个实例的数据文件，并配合 WAL；速度适合大库，可用于搭建从库和 PITR，但版本、平台兼容限制更严格。

备份成功不代表可恢复。应定期在隔离环境做恢复演练、校验恢复时长和业务一致性。

### 14. 什么是 PITR？需要哪些条件？

PITR 是时间点恢复：以一个物理基础备份为起点，连续重放归档 WAL，恢复到指定时间、事务 ID、LSN 或恢复点。

基本条件：可用的 base backup、从备份开始到目标点连续完整的 WAL 归档、正确的恢复配置以及足够的存储和恢复演练。若中间缺一段 WAL，通常无法越过缺口继续恢复。

### 15. PostgreSQL 流复制的原理是什么？同步与异步有什么区别？

主库的 WAL sender 将 WAL 发送给备库的 WAL receiver，备库写入并重放，因此物理复制得到的是整个实例级别的块变化。

- 异步复制：主库提交不等待备库确认，性能和可用性较好，但主库故障时可能丢失尚未传到备库的事务。
- 同步复制：提交等待指定备库确认，降低数据丢失风险，但增加延迟；同步备库异常可能影响写入可用性。

流复制不等于自动故障切换，通常还需要 Patroni、repmgr 等集群管理方案和可靠的仲裁设计。

## 二、高频级：★★★★☆

### 16. 如何判断并处理复制延迟？

主库查看 `pg_stat_replication` 的 `sent_lsn`、`write_lsn`、`flush_lsn`、`replay_lsn`；备库查看 `pg_last_wal_receive_lsn()`、`pg_last_wal_replay_lsn()` 和最后重放时间。

延迟可能发生在生成、网络传输、落盘或重放阶段。原因包括主库突发大量 WAL、网络慢、备库磁盘或 CPU 瓶颈、长查询造成恢复冲突、备库参数不当。应先定位是哪一段落后，再扩容、限流或优化，不能只看“时间延迟”单一指标。

### 17. 什么是复制槽？有什么风险？

复制槽用于确保主库保留消费者尚未接收的 WAL；物理槽服务物理备库，逻辑槽服务逻辑解码。优点是避免所需 WAL 被提前回收。

风险是消费者离线或故障时 WAL 会持续积压，可能撑满主库磁盘。应监控 `pg_replication_slots` 的活跃状态和保留量，并合理设置 `max_slot_wal_keep_size`。确认不再使用后再删除槽。

### 18. 物理复制和逻辑复制有什么区别？

物理复制传输 WAL 中的块级变化，通常复制整个实例，主备版本要求较严格，适合高可用和灾备。逻辑复制按发布/订阅传输表级数据变更，可选择表并支持部分跨大版本场景，适合数据分发和在线迁移。

逻辑复制不会自动复制所有 DDL、序列状态和大对象等内容，且表通常需要合适的 replica identity；它不能简单等同于完整备份或透明 HA。

### 19. shared_buffers、work_mem、maintenance_work_mem 如何理解？

- `shared_buffers`：实例共享的数据页缓存，常从内存的约 25% 起步评估，但要以负载测试为准。
- `work_mem`：每个排序或 Hash 操作可用的内存，不是每连接只分配一次；并发查询中可能被多次使用，设得过大会导致 OOM。
- `maintenance_work_mem`：Vacuum、建索引等维护操作可用内存。

参数不能脱离并发量、SQL 计划、操作系统缓存和容器内存限制孤立调整。

### 20. 为什么 PostgreSQL 连接数不能无限增大？

每个连接通常对应一个后端进程并占用内存，过多连接会增加上下文切换、锁管理和缓存压力，严重时新连接失败或系统抖动。

处理方法包括：应用使用连接池；在数据库前部署 PgBouncer；设置合理的 `max_connections`、连接超时和保留管理连接；排查连接泄漏与 `idle in transaction`。不能只是把 `max_connections` 不断调大。

### 21. PgBouncer 的 session、transaction、statement 模式有什么区别？

- Session：客户端会话期间独占一个服务端连接，兼容性最好，复用率较低。
- Transaction：每个事务结束后归还服务端连接，最常用；依赖会话状态、某些临时表或会话级设置的业务需评估。
- Statement：每条语句后归还连接，不允许多语句事务，限制最多。

选择时应先确认应用是否使用 prepared statement、临时对象、LISTEN/NOTIFY、会话级参数等特性。

### 22. 顺序扫描一定比索引扫描差吗？

不一定。查询大比例数据或表很小时，顺序扫描可连续读取页面，可能比大量随机访问索引和堆表更快。PostgreSQL 还可能使用 Index Scan、Index Only Scan、Bitmap Index/Heap Scan。

正确判断依据是执行计划、实际耗时、Buffer 与返回数据比例，而不是看到 Seq Scan 就强制关闭 `enable_seqscan`。

### 23. 复合索引、部分索引和表达式索引分别何时使用？

- 复合索引：多个列共同过滤或排序；列顺序需结合等值、范围、排序和选择性设计。
- 部分索引：只索引满足条件的数据，如只索引 `status = 'pending'`，减少体积和维护成本；查询条件要能被优化器识别为蕴含索引谓词。
- 表达式索引：对表达式结果建索引，如 `lower(email)`，适合函数条件查询。

索引设计应匹配真实 SQL，而不是按“选择性最高永远放第一”机械套规则。

### 24. 什么是 Index Only Scan？为什么仍可能访问堆表？

当索引包含查询所需的全部列时，优化器可能选择 Index Only Scan。但 PostgreSQL 仍需确认元组对当前快照是否可见；只有可见性映射表明对应堆页面 all-visible 时，才能避免访问堆表。

因此高更新表即便有覆盖索引，也可能出现较多 Heap Fetches。及时 Vacuum 有助于维护可见性映射，但不能保证所有查询完全不访问堆。

### 25. 表膨胀和索引膨胀是怎么产生的？

MVCC 更新、删除产生死元组；正常 Vacuum 使空间可被表内部复用，但通常不缩小文件。长事务、失效的 autovacuum、高频更新和页面无法复用会造成表膨胀；索引在更新、删除和页分裂中也可能膨胀。

处理顺序：修复长事务和 autovacuum → 评估业务写入模式 → 使用 `REINDEX CONCURRENTLY`、`pg_repack` 或维护窗口内 `VACUUM FULL`。这些操作的锁、额外磁盘和 WAL 成本必须提前评估。

### 26. 什么是 HOT Update？

HOT（Heap-Only Tuple）更新在被修改列不涉及相关索引且页面有空间时，可避免为新行版本写入新的索引项，并允许在页内沿版本链查找，从而减少索引膨胀和写放大。

合理降低表的 `fillfactor` 可以为频繁更新预留页内空间，但会增大表体积；是否调整要结合更新比例和缓存命中率测试。

### 27. checkpoint 的作用是什么？过于频繁有什么影响？

Checkpoint 将此前的脏页逐步刷盘并记录一致性恢复起点，使崩溃恢复不必从无限久以前重放 WAL。过于频繁会带来写 I/O 峰值、更多 full-page writes 和 WAL。

应结合 `checkpoint_timeout`、`max_wal_size`、`checkpoint_completion_target`、日志中的 checkpoint 信息及磁盘延迟调整。单纯把间隔无限调大也会增加恢复时间和 WAL 空间需求。

### 28. 如何安全地做大表 DDL？

先确认目标版本中该 DDL 是否需要重写表、扫描表或持有强锁。设置合理的 `lock_timeout` 和 `statement_timeout`，避免排队后突然阻塞业务；检查长事务，低峰期灰度执行并监控复制延迟和磁盘空间。

建索引可考虑 `CREATE INDEX CONCURRENTLY`，删除可用 `DROP INDEX CONCURRENTLY`。并发建索引耗时更长、消耗更多资源，失败后还可能留下 invalid index，必须检查结果。

## 三、常问进阶与场景题：★★★☆☆

### 29. 数据库 CPU 突然升高，如何排查？

先确认是否真是 PostgreSQL 进程及持续时间，再查看活跃 SQL、等待事件和调用统计；区分正常流量上涨、执行计划突变、并行查询、连接风暴和后台维护。

重点查看 `pg_stat_activity`、`pg_stat_statements`、慢日志、系统 CPU/Load/I/O；对高消耗 SQL 获取执行计划并检查统计信息。不要一开始就重启数据库，因为这会丢失现场且可能造成恢复风暴。

### 30. `idle in transaction` 有什么危害？

它表示事务已开始但当前没有执行语句。长期存在会持有锁和旧快照，阻碍 Vacuum 回收死元组，增加膨胀和事务 ID 风险，也会占用连接。

应定位应用事务边界和连接池配置，设置合适的 `idle_in_transaction_session_timeout` 作为保护，并谨慎评估超时中断对业务的影响。

### 31. `pg_wal` 磁盘满了怎么办？

先保护现场并判断增长来源：归档失败、复制槽消费者停滞、备库延迟、大事务或 checkpoint/WAL 参数。检查 `pg_stat_archiver`、`pg_replication_slots`、`pg_stat_replication` 和日志。

不要手工删除 `pg_wal`。正确做法是恢复归档目标、修复或确认后移除废弃复制槽、恢复消费者、临时扩容，并处理 WAL 暴增的业务根因。

### 32. 数据目录磁盘快满时怎么处理？

先区分表/索引增长、WAL、日志、临时文件还是归档占用；确认磁盘、inode、表空间和增长趋势。可临时扩容或迁移安全的日志/归档，停止异常批任务，并处理失控复制槽或临时文件。

删除表数据后文件通常不会立即缩小。任何 `VACUUM FULL`、重建索引或迁移表空间操作都要预估额外空间、锁、WAL 和复制影响。

### 33. 如何排查“数据库连接不上”？

按链路排查：

1. 进程是否存活、端口是否监听、资源是否耗尽。
2. 网络、防火墙、DNS、负载均衡是否通。
3. `listen_addresses`、端口和 `pg_hba.conf` 是否匹配。
4. 用户、密码、数据库、SSL 与认证方式是否正确。
5. 是否达到 `max_connections`，日志中是否有明确拒绝原因。

修改 `pg_hba.conf` 后通常 reload 即可；配置顺序按首次匹配生效，排错时要特别注意。

### 34. PostgreSQL 如何做权限和安全控制？

使用角色管理登录和对象权限，通过 `GRANT`/`REVOKE` 授予最小权限；`pg_hba.conf` 控制客户端来源、数据库、用户和认证方式；网络传输可启用 TLS。应用不应使用超级用户，Schema 的 `CREATE` 权限也应审慎管理。

还要关注默认权限 `ALTER DEFAULT PRIVILEGES`、密码认证方式、审计、密钥轮换、备份加密和日志脱敏。`pg_hba.conf` 不是 SQL 对象授权的替代品。

### 35. `DELETE`、`TRUNCATE`、`DROP` 有什么区别？

- `DELETE`：逐行删除，可带条件，会产生行版本和较多 WAL，触发相应触发器。
- `TRUNCATE`：快速清空表，获取强锁，空间可立即回收，语句在 PostgreSQL 中是事务性的；触发 `ON TRUNCATE` 而非逐行 DELETE 触发器。
- `DROP`：删除对象本身及其定义。

不能简单回答“TRUNCATE 不能回滚”；在 PostgreSQL 的普通事务中它可以回滚。

### 36. PostgreSQL 分区表有什么价值和注意事项？

声明式分区可按 Range、List、Hash 将逻辑表拆为多个分区。价值包括分区裁剪、快速装卸历史数据、分区级维护和冷热数据管理。

分区不是自动提升性能：分区键不匹配查询时收益有限；分区过多会增加规划和管理成本；唯一约束通常必须包含分区键才能跨所有分区保证唯一性。索引和 autovacuum 仍需按分区关注。

### 37. JSON 和 JSONB 有什么区别？

`json` 保存输入文本，保留空格和键顺序，每次查询需解析；`jsonb` 保存分解后的二进制形式，不保留原始格式，查询和索引通常更高效，并会处理重复键。多数需要查询和索引的业务选 JSONB；只需保存原文时可选 JSON。

JSONB 支持 GIN 等索引，但把所有数据都塞入 JSONB 会削弱类型约束、关系设计和统计信息质量。

### 38. PostgreSQL 与 MySQL 如何比较？

不要回答“PostgreSQL 一定慢、MySQL 一定快”。两者都是成熟关系数据库，性能取决于版本、数据模型、SQL 和运维方式。

PostgreSQL 通常在复杂 SQL、丰富数据类型、扩展能力和标准一致性方面有优势；MySQL 在大量 Web 生态和团队使用习惯上常见。实际选型应比较一致性要求、查询复杂度、HA 方案、迁移成本、团队能力和基准测试。

### 39. 你如何设计 PostgreSQL 高可用方案？

典型方案包括一个主库、至少一个跨故障域备库、物理流复制、集群管理/选主组件、连接入口，以及独立备份和监控。关键不是画出“主从”，而是明确：

- RPO/RTO 目标及同步或异步策略。
- 脑裂防护、仲裁和 fencing。
- 故障切换后旧主如何处理，客户端如何重连。
- 备份/PITR 与 HA 独立验证。
- 定期切换和恢复演练。

### 40. PostgreSQL 日常巡检和监控哪些指标？

至少覆盖：实例存活与连接数、活跃/等待会话、慢 SQL、事务提交回滚率、锁与死锁、缓存与 I/O、表和索引大小、死元组与 autovacuum、长事务、WAL 速率与磁盘、归档状态、复制延迟和复制槽、备份成功及恢复演练结果。

告警阈值应结合趋势和业务基线，避免只监控 CPU、内存、磁盘三个系统指标。

## 四、容易追问的命令速记

```sql
-- 当前会话与等待
SELECT pid, usename, datname, state, wait_event_type, wait_event,
       xact_start, query_start, query
FROM pg_stat_activity
WHERE pid <> pg_backend_pid();

-- 长事务
SELECT pid, now() - xact_start AS xact_age, state, query
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
ORDER BY xact_start;

-- 阻塞者
SELECT pid, pg_blocking_pids(pid), query
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;

-- 表死元组
SELECT relname, n_live_tup, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;

-- 主库复制状态
SELECT application_name, client_addr, state, sync_state,
       sent_lsn, write_lsn, flush_lsn, replay_lsn
FROM pg_stat_replication;

-- 复制槽
SELECT slot_name, slot_type, active, restart_lsn,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
FROM pg_replication_slots;

-- 数据库和表大小
SELECT datname, pg_size_pretty(pg_database_size(datname))
FROM pg_database ORDER BY pg_database_size(datname) DESC;
SELECT pg_size_pretty(pg_total_relation_size('schema.table'));
```

常用系统命令：

```bash
systemctl status postgresql
pg_isready -h 127.0.0.1 -p 5432
psql -h 127.0.0.1 -U user -d database
pg_dump -Fc -d database -f database.dump
pg_restore -d target_database database.dump
pg_basebackup -h primary -D /data/standby -U repl -R -Xs -P
```

## 五、低收益题目：了解即可

以下内容不是完全不会问，而是对 Linux 运维初中级岗位投入产出比较低：

- PL/Python、PL/Perl 等过程语言的语法细节。
- 自定义数据类型、操作符和索引访问方法开发。
- GiST/SP-GiST 内部算法证明。
- PostgreSQL 源码编译调试选项。
- 冷门扩展 API、FDW 开发和 C 扩展编程。
- 单纯背所有系统目录、所有锁模式的完整列表。

## 六、面试前速记清单

1. MVCC 让读写并发，但会产生死元组；Vacuum 与长事务必须关联回答。
2. WAL 同时关联崩溃恢复、复制、归档和 PITR。
3. `VACUUM` 回收复用空间；`VACUUM FULL` 重写并强锁；`ANALYZE` 更新统计信息。
4. 慢 SQL 用 `EXPLAIN (ANALYZE, BUFFERS)`，先找估算偏差、扫描/连接方式和 I/O。
5. 顺序扫描不一定差，索引也有写入和维护成本。
6. 备份必须能恢复；HA、备份和 PITR 是三个相关但不同的问题。
7. 复制槽能防 WAL 丢失，也可能把主库磁盘撑满。
8. `idle in transaction`、长事务会持锁、保留旧快照并阻碍 Vacuum。
9. `pg_wal` 不能手工删除；先查归档、复制槽、备库和大事务。
10. 场景题按“确认影响 → 保留现场 → 定位数据库/系统/业务 → 止损 → 根因与预防”回答。

## 七、项目经验回答模板

> 我维护过 PostgreSQL 主从环境，日常关注连接、慢 SQL、锁、长事务、autovacuum、WAL、复制延迟和备份。遇到问题时先结合监控确认影响，再通过 `pg_stat_activity`、`pg_stat_statements`、执行计划和系统 I/O 定位。备份采用物理基础备份配合 WAL 归档，并定期做 PITR 恢复演练。重大 DDL 会检查锁和长事务，设置超时并在低峰期灰度执行。实际面试时应将其中内容替换为自己的真实规模、版本、工具和处理案例。

## 八、参考资料

- [PostgreSQL 官方文档](https://www.postgresql.org/docs/current/)
- [PostgreSQL：并发控制](https://www.postgresql.org/docs/current/mvcc.html)
- [PostgreSQL：索引](https://www.postgresql.org/docs/current/indexes.html)
- [PostgreSQL：常规数据库维护](https://www.postgresql.org/docs/current/maintenance.html)
- [PostgreSQL：高可用、负载均衡与复制](https://www.postgresql.org/docs/current/high-availability.html)
- [LabEx PostgreSQL 面试题及答案](https://labex.io/zh/tutorials/postgresql-postgresql-interview-questions-and-answers-593697)
- [墨天轮：52 道 PostgreSQL 面试题](https://www.modb.pro/db/534164)

> 说明：公开题库中存在过时翻译和不准确结论，本稿未照搬；技术表述按 PostgreSQL 当前官方文档校正。不同版本行为可能有差异，生产操作前应核对目标版本文档。
