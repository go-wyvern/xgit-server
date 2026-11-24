# Scratch Discover

## 1. 目标

Scratch Discover 负责发现：

- 新项目
- 已有项目的潜在变化
- 需要重新抓取资源的项目
- 需要补齐历史版本的项目
- 需要更新互动指标的项目

它发现任务，不负责完整同步。

## 2. 发现来源

第一阶段使用以下来源：

- Scratch Explore popular 分页
- 手动指定项目 ID
- 已有镜像项目的更新时间和内容 hash
- 定时全量对账

后续如果 Scratch 提供可靠的事件、版本列表或搜索接口，再接入增量来源。

## 3. 状态模型

每个源项目保存一条状态。

```json
{
  "source": "scratch",
  "projectId": "123",
  "author": "alice",
  "lastMetaUpdatedAt": "2026-11-24T08:00:00Z",
  "lastProjectDigest": "sha256:...",
  "lastSourceVersion": "version-123",
  "lastCursor": "cursor-456",
  "targetRepoId": "repo_01JXG8Q6M5",
  "lastTargetCommit": "0123456789abcdef..."
}
```

状态本身需要持久化，不能只保存在 worker 本地文件。

## 4. 新项目判断

新项目判断使用 `source project ID` 作为主键。

```text
project ID 不在 state
  -> ProjectDiscovered
  -> 创建 scratch-mirror 仓库
  -> 投递 Scratch Sync
```

项目标题变化不产生新仓库，只更新仓库元数据。

## 5. 项目变更判断

优先级从高到低如下：

1. 上游提供明确的 version ID
2. 上游提供更新时间字段
3. project JSON 内容 hash 变化
4. 资源引用集合或资源内容 hash 变化
5. 定期全量对账

如果只有更新时间字段，必须保存上一次元数据快照。发现变更后再由 Sync 服务获取完整项目内容并确认内容是否真正变化。

## 6. 事件模型

```json
{
  "type": "project.changed",
  "source": "scratch",
  "projectId": "123",
  "reason": "metadata_updated",
  "sourceVersion": "version-123",
  "dedupeKey": "scratch:123:version-123"
}
```

事件类型：

- `project.discovered`
- `project.changed`
- `project.reconcile`
- `project.metrics_due`
- `project.sync_requested`

## 7. Discover API

创建一次发现任务。

```http
POST /api/v1/discovery/runs
Authorization: Bearer <control-token>
Content-Type: application/json
```

```json
{
  "source": "scratch",
  "mode": "popular",
  "topN": 50,
  "projectIds": []
}
```

查询发现任务。

```http
GET /api/v1/discovery/runs/{runId}
Authorization: Bearer <control-token>
```

响应包括：

- 扫描批次
- 发现的新项目数量
- 发现变化的项目数量
- 已发送的同步任务数量
- source cursor
- 最近错误

## 8. 低成本策略

- 热门项目使用分页和去重
- 通过 source cursor 避免重复扫描
- 资源以 md5ext 或内容 hash 去重
- 项目元数据和资源下载分离
- 单个项目失败不阻塞批次
- 429 和 503 使用指数退避
- 定期全量对账修正遗漏
- 互动指标按批次采集，不触发项目 Git commit
