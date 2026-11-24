# Scratch Sync

## 1. 目标

Scratch Sync 将 Scratch Live 的项目写入 xgit-server 的 Scratch Mirror。

```text
Scratch Live
  -> Scratch Sync
  -> xgit-server Native Repository
  -> Scratch Mirror
```

它同时负责将镜像版本交给 `sb2xbp` 和后续测试系统。

## 2. 新项目同步

用户描述的新项目流程是：

```text
1. 从 scratch-live git clone
2. 在 xgit-server 创建 scratch-mirror 仓库
3. 将项目 push 到 scratch-mirror
4. 记录 source project ID、source version 和 target commit
5. 调用 sb2xbp 转换指定 commit
6. 将 XBP 提交给 XBuilder 或 spxbench
```

Git 模式命令示例：

```bash
git clone "$SCRATCH_LIVE_REMOTE" "$WORKTREE"
git -C "$WORKTREE" push "$SCRATCH_MIRROR_REMOTE" \
  'refs/heads/main:refs/heads/main' \
  --tags
```

生产实现不应依赖本地工作区，可以使用 Git library 或内部 Source 模式直接写入 Native Repository。

## 3. 已有项目同步

```text
1. 从 source state 读取 last source version 或 cursor
2. git fetch scratch-live，或调用 SourceProvider 列出新版本
3. 判断是否有新 commit 或新资源
4. 将新版本写入 scratch-mirror
5. 更新 target main 或 version ref
6. 保存 source version、cursor 和 target commit
7. 触发 sb2xbp 和 spxbench 后续任务
```

Git 模式可以表现为 pull 后 push：

```bash
git fetch "$SCRATCH_LIVE_REMOTE"
git push "$SCRATCH_MIRROR_REMOTE" \
  'refs/remotes/source/main:refs/heads/main'
```

## 4. Source 模式

Source 模式的内部步骤如下。

```text
ListVersions(cursor)
  -> GetSnapshot(version)
  -> GetAsset(asset)
  -> SB3/SB2 normalize
  -> Materialize Git tree and commit
  -> Put objects
  -> UpdateRefsCAS
  -> Advance cursor
```

Source 模式需要保留：

- source project ID
- source version ID
- source cursor
- 原始 metadata
- 抓取时间
- 目标 commit
- 失败资源和重试状态

## 5. Sync API

创建单项目同步任务。

```http
POST /api/v1/sync/projects/{projectId}
Authorization: Bearer <control-token>
Content-Type: application/json
```

```json
{
  "source": "scratch",
  "mode": "incremental",
  "targetRepoId": "repo_01JXG8Q6M5",
  "sourceVersion": "version-123"
}
```

创建批量同步任务。

```http
POST /api/v1/sync/runs
Authorization: Bearer <control-token>
Content-Type: application/json
```

```json
{
  "source": "scratch",
  "mode": "incremental",
  "projectIds": ["123", "456"]
}
```

查询任务。

```http
GET /api/v1/sync/runs/{runId}
Authorization: Bearer <control-token>
```

```json
{
  "runId": "run_01JXG8Q6M5",
  "state": "completed",
  "projectsDiscovered": 50,
  "projectsSynced": 47,
  "versionsImported": 129,
  "assetsImported": 820,
  "lastError": null
}
```

## 6. 同步幂等

推荐幂等键：

```text
source + projectID + sourceVersionID
```

如果上游没有 version ID，则使用：

```text
source + projectID + normalizedProjectDigest + assetDigest
```

同一个幂等键再次同步时，复用已有 commit，不重复更新 refs。

## 7. sb2xbp 和 spxbench 投影

Git 镜像成功后发布事件：

```json
{
  "type": "mirror.commit.created",
  "repoId": "repo_01JXG8Q6M5",
  "projectId": "123",
  "commit": "0123456789abcdef...",
  "format": "sb3"
}
```

下游转换任务：

```text
mirror.commit.created
  -> 导出指定 commit 的 SB2/SB3
  -> sb2xbp 转换 XBP
  -> 上传 XBP 或导入 XBuilder
  -> 通知 spxbench
```

XBP 转换失败不回滚 Git 镜像。失败状态应关联到 source version 和 target commit，并支持重试。

## 8. 失败恢复

- 下载失败保留当前 cursor，不推进后续版本
- 单个资源失败只重试该资源
- commit 已写入但 ref 更新失败时重试 CAS
- ref 已更新但事件发布失败时依靠 outbox 重放
- 单个项目失败不阻塞批次
- 任务完成后再推进 batch cursor
