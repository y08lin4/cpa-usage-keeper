# Number-Pool-Manager 移植分析

> 本文用于记录 `cpa-usage-keeper` 中适合移植到 Number-Pool-Manager 的设计点。
> 目标不是让 Number-Pool-Manager 依赖本服务，而是抽取同步、去重、聚合和仪表盘口径，集成到 Number-Pool-Manager 自己的 SQLite 与 WPF。

## 1. 总体结论

`cpa-usage-keeper` 已经覆盖了 CLIProxyAPI 使用量持久化的大部分关键问题：

- 从 CPA 获取使用量快照或 Redis Usage Queue 消息。
- 将上游数据规范化为 `UsageEvent`。
- 通过稳定 `EventKey` 去重。
- 使用 SQLite 保存 `SnapshotRun`、`UsageEvent`、`RedisUsageInbox`。
- 提供聚合 API 和前端仪表盘。
- 对 Redis inbox、snapshot run 和备份做清理。

Number-Pool-Manager 已经先建好对应底座：

```text
usage_sync_runs
usage_events
redis_usage_inbox
usage_sync_state
```

后续重点是把本项目的账号、池子、套餐、来源标签维度和 Keeper 的 usage event 维度合并。

## 2. 表结构对应关系

### 2.1 `SnapshotRun` -> `usage_sync_runs`

Keeper 字段：

```go
FetchedAt
CPABaseURL
ExportedAt
Version
Status
HTTPStatus
PayloadHash
RawPayload
BackupFilePath
ErrorMessage
InsertedEvents
DedupedEvents
```

Number-Pool-Manager 建议：

- `usage_sync_runs.source`：`keeper_export`、`redis_usage_queue`、`cpa_export` 等。
- `usage_sync_runs.pool_id`：映射本项目 CPA 池子。
- `fetched_count / inserted_count / skipped_count / failed_count` 对应 Keeper 的插入和去重统计。
- 原始 payload 不建议长期明文保存；如果保存，走日志/备份目录并脱敏。

### 2.2 `UsageEvent` -> `usage_events`

Keeper 核心字段：

```go
EventKey
SnapshotRunID
APIGroupKey
Model
Timestamp
Source
AuthIndex
Failed
LatencyMS
InputTokens
OutputTokens
ReasoningTokens
CachedTokens
TotalTokens
```

Number-Pool-Manager 已扩展：

- `pool_id`
- `email`
- `auth_file_name`
- `account_id`
- `plan_type_snapshot`
- `source_name_snapshot`
- `source_tags_snapshot`
- `account_cost`
- `user_cost`

扩展原因：

- 本项目主键是邮箱，账号可能在多个 CPA 池子中。
- 历史统计不能受后续来源标签修改影响，所以事件发生时要冗余套餐和来源快照。
- CPA 的 `auth_index` 是池子内索引，不适合作为本项目账号唯一标识。

### 2.3 `RedisUsageInbox` -> `redis_usage_inbox`

Keeper 已经实现的价值：

- Redis `LPOP` 后先落 inbox，避免进程崩溃导致消息丢失。
- `pending`、`processed`、`decode_failed`、`process_failed`、`discarded` 状态明确。
- 支持失败重试和清理策略。

Number-Pool-Manager 移植建议：

- Redis Queue 路线必须先写 inbox，再解析为 `usage_events`。
- `message_key` 优先使用上游消息唯一 ID；没有唯一 ID 时用 hash。
- `raw_message_json` 可能含敏感信息，UI 不直接展示。

## 3. 事件去重策略

Keeper 的 `BuildEventKey` 使用：

```text
api_group_key
model
timestamp
source
auth_index
failed
input_tokens
output_tokens
reasoning_tokens
cached_tokens
total_tokens
```

并计算 SHA-256。

Number-Pool-Manager 可以直接复用这个思路，但建议加入：

```text
source
pool_id
account_id 或 auth_file_name
```

推荐格式：

```text
sha256(source + pool_id + api_group_key + model + requested_at + auth_file_name/account_id + failed + token tuple)
```

如果上游未来提供稳定事件 ID，则优先：

```text
source + ":" + upstream_event_id
```

## 4. 同步模式选择

### 4.1 Redis Usage Queue 优先级最高

优点：

- 更接近事件流。
- 不需要反复拉全量 export。
- 可通过 inbox 防丢。

代价：

- 需要配置 Redis/RESP 地址和队列名。
- 程序不在线时可能取决于队列保留策略。

### 4.2 Legacy export 适合作为兜底

优点：

- 适合手动补齐和重建。
- 可用于和 Redis 数据校验。

代价：

- 需要稳定水位线。
- 更容易出现重复事件，因此依赖 `event_key` 去重。

### 4.3 Keeper aggregate 不作为第一优先

Keeper aggregate 适合外部仪表盘直接读，但 Number-Pool-Manager 需要把事件落到自己的 SQLite，以便和账号来源、套餐、认证历史合并。

## 5. 聚合 API 和 WPF 仪表盘可移植点

Keeper 的聚合能力可拆成几类：

1. Summary cards
   - 请求数。
   - 成功/失败数。
   - token 总量。
   - 成本。

2. Time series
   - 小范围按小时。
   - 大范围按天。

3. Dimensions
   - model。
   - source。
   - api group。

4. Request events table
   - 分页。
   - model/source/result 筛选。
   - latency 和 token 明细。

Number-Pool-Manager 的 WPF 第一阶段建议：

- 先复用 summary cards + request events table。
- 图表可以后置，先用 DataGrid 展示 daily/hourly 聚合。
- source 维度要映射到本项目 `source_name/source_tags`。

## 6. 清理策略

Keeper 已有：

- 成功 Redis inbox 清理。
- 失败 Redis inbox 保留 7 天。
- snapshot run 按本地日期保留最近窗口内每天最新一条。
- `VACUUM`。

Number-Pool-Manager 建议：

- `redis_usage_inbox`：
  - processed：保留到当天结束后清理。
  - decode/process failed：保留 7 天或 30 天。
  - pending：不自动删除。
- `usage_sync_runs`：保留至少 90 天摘要。
- `usage_events`：默认长期保留。

## 7. 推荐落地顺序

1. 完成 Number-Pool-Manager 的 `usage_events` 写入模型和 `event_key` 生成器。
2. 实现 legacy export 拉取和入库，验证 dedupe。
3. 实现 Redis inbox 拉取、解析、重试、丢弃。
4. 实现 `usage_daily_stats(source=usage_events)` 重算。
5. 在 WPF 增加精确统计页：
   - summary cards。
   - daily/hourly series DataGrid。
   - request events DataGrid。
6. 再做图表化。

## 8. 当前移植状态

已完成：

- Keeper 源码结构分析。
- Number-Pool-Manager 已新增对应 SQLite 表。
- Number-Pool-Manager 已有快照差值估算统计，作为精确统计前的观测底座。

待完成：

- 真实 CPA 使用量事件源确认。
- `event_key` 生成器。
- usage sync 定时任务。
- `usage_events` 入库。
- `usage_daily_stats(source=usage_events)` 聚合。
- WPF 精确统计仪表盘。
