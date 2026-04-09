# Debezium CDC + RocketMQ 实战学习分享（debezuim-cdc-rocketmq）

这篇文章是我对项目 `debezuim-cdc-rocketmq` 的一次完整学习梳理，目标是回答三个问题：

1. 这个项目解决了什么问题？
2. 它的核心架构为什么这样设计？
3. 如果我要在生产里落地，应该重点关注哪些点？

项目地址：
https://github.com/doublej9999/debezuim-cdc-rocketmq

---

## 一、项目定位：把数据库变更稳定送到 RocketMQ

这个项目基于：
- Java 21
- Spring Boot 3.4
- Debezium 3.0（Embedded Engine）
- RocketMQ 4.9
- PostgreSQL（逻辑复制）

核心目标是：
- 监听 PostgreSQL 表变更（INSERT/UPDATE/DELETE）
- 将 CDC 事件转换后发送到 RocketMQ
- 支持多数据源/多配置并发
- 提供可观测、可治理、可恢复的工程能力

这不是“只能跑 Demo 的 CDC 代码”，而是面向持续运行场景做了不少生产化增强。

---

## 二、我认为最有价值的 8 个设计点

### 1）多配置并发：每个配置一条 CDC 管道

`MultiConfigCdcPipelineManager` 负责按配置启动/停止/重启 Debezium 管道，并且使用 Java 21 虚拟线程承载。

价值：
- 多租户/多业务线可并行采集
- 单条管道故障不拖垮全局
- 配置开关控制灵活（启停粒度到 configId）

### 2）异步解耦：Debezium 与 RocketMQ 发送拆开

项目没有在 Debezium 回调里直接同步发 MQ，而是先进入 `AsyncEventSenderService` 队列，再由发送线程池批量/逐条投递。

价值：
- RocketMQ 瞬时抖动时不立即反压 Debezium 引擎
- 引擎侧更专注于“采集”，发送侧更专注于“投递”
- 更容易做削峰、重试、监控

### 3）事件落库：event_log 作为可靠性中间层

每条事件在入队前先落 `event_log`，状态机包含 PENDING / SENT / RETRY / FAILED。
支持：
- 自动重试
- 手动重试
- 幂等（通过 configId + lsn 进行去重判断）

价值：
- 具备“可追踪、可补偿、可回放”的基础能力
- 出问题时不是“日志里找不到”，而是有结构化记录可查

### 4）顺序性策略：按 key 分片路由队列

异步发送阶段根据 key（或 configId）路由到分片队列，同 key 事件进入同一分片，有助于保持业务顺序。

价值：
- 在提高吞吐的同时，兼顾关键业务对象的顺序性

### 5）Watchdog 自动巡检与重启

项目内置定时巡检（`cdc.watchdog.interval.ms`），发现管道未运行可自动触发重启，并且带冷却机制避免抖动重启风暴。

价值：
- 降低“人工盯盘+人工重启”的维护成本

### 6）Replication Slot 生命周期治理

围绕 PostgreSQL 逻辑复制 slot 做了比较多治理能力（监控、清理、优化建议），并强调 slot 与 publication 的维护。

价值：
- 这是 CDC 生产事故高发区，能提前防坑
- 有助于控制 WAL 膨胀风险

### 7）可观测性：不仅有状态接口，还有指标

项目有 API 级状态查看，也在异步发送服务中接入了 Micrometer Counter/Gauge/Timer（入队量、成功量、失败量、队列深度、发送时延）。

价值：
- 可以更快判断瓶颈在“采集、队列、发送、下游”哪一层

### 8）安全细节：前端到后端密码加密 + 后端存储加密

数据源密码在传输链路使用 RSA-OAEP（前端加密），并在后端进行字段级处理。

价值：
- 对“内网就不加密”的常见误区有明显改善

---

## 三、从代码里读到的核心数据流

简化后大致是：

PostgreSQL WAL
→ Debezium Engine（每配置一套）
→ 解析事件（提取 lsn / key / payload）
→ event_log 落库 + 异步入队
→ 分片发送线程池
→ RocketMQ topic/tag/key
→ 成功/失败回写 event_log 状态

重点在于：
- 采集与发送职责分离
- 事件先持久化再异步投递
- 重试与状态管理闭环

---

## 四、落地时最该先做的 6 件事

### 1）先把 PostgreSQL 逻辑复制参数和权限校准

至少确认：
- `wal_level=logical`
- `max_replication_slots`、`max_wal_senders` 充足
- 复制权限与网络连通性 OK

### 2）按吞吐压测调 async 参数

重点参数：
- `async.event.queue.size`
- `async.event.sender.threads`
- `async.event.sender.batch.size`
- `async.event.sender.batch.timeout.ms`

建议做“峰值 + 抖动”双场景压测，不要只测平均值。

### 3）监控 WAL 留存与 slot 活跃情况

必须把 slot 健康做成常驻监控，否则很容易在低峰期被忽略、高峰期爆雷。

### 4）明确消息幂等边界

虽然有 lsn 维度的幂等保护，但消费端仍应有业务幂等机制，避免重复投递导致脏写。

### 5）区分“可自动重试”和“需人工介入”错误

建议把失败原因分层：
- 可重试：网络抖动、短暂不可用
- 不可重试：权限错误、配置错误、数据格式硬错误

### 6）给 event_log 做生命周期策略

项目提供了清理逻辑，实际部署要结合审计需求制定保留策略，避免无限增长。

---

## 五、我认为项目还能继续增强的方向

1. 更细粒度告警模板
   - 比如“队列持续增长 N 分钟”“失败率超过阈值”直接告警。

2. DLQ（死信队列）与补偿工作流可视化
   - 失败事件进入专门处理通道，减少人工 SQL 操作。

3. 多环境配置与密钥管理进一步标准化
   - 与 KMS/配置中心联动，避免敏感配置散落。

4. 端到端压测基线文档
   - 给出不同数据量级下的建议参数区间，帮助新团队快速落地。

---

## 六、总结

`debezuim-cdc-rocketmq` 的价值，不只是“把 Debezium 接上 RocketMQ”，而是把 CDC 在真实运行中会遇到的关键问题（稳定性、可观测、重试补偿、slot 治理）做了系统化处理。

如果你正在做：
- 数据库变更事件驱动
- 多库到 MQ 的变更分发
- 低侵入的数据同步架构

这个项目非常值得作为工程参考样本。

---

如果后续你希望，我可以再补一篇：
- 《debezuim-cdc-rocketmq 部署与调优清单（生产版）》
- 按“启动前检查 / 运行中观测 / 事故应急”三个阶段给出可执行 checklist。
