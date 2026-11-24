# Scratch API

本文记录当前 `spxbench` 使用的 Scratch HTTP 接口。它们用于设计 Scratch Live 和 Scratch Discover 的 SourceProvider。

这些接口来自当前公开服务和 `spxbench` 的实际代码，属于上游数据源依赖。正式实现必须增加超时、重试、限流、响应校验和接口变化监控。

## 1. 热门项目发现

当前 workflow 使用 Explore API 获取热门项目。

```http
GET https://api.scratch.mit.edu/explore/projects?limit={limit}&offset={offset}&language=en&mode=popular&q=*
```

当前 `spxbench` 的默认行为是：

- `limit` 使用分页大小
- `offset` 持续递增
- `mode=popular`
- `language=en`
- 使用项目 ID 去重
- 遇到 429 或 503 进行等待和重试
- 通过 `topN` 控制目标数量

该接口适合发现热门项目。它不保证覆盖所有社区项目。

## 2. 项目元数据

```http
GET https://api.scratch.mit.edu/projects/{projectID}
```

当前 `spxbench` 至少依赖以下信息：

```json
{
  "id": 123,
  "title": "project title",
  "project_token": "token used by project download"
}
```

实际响应可能包含作者、更新时间、统计数据和其他字段。SourceProvider 应保存原始 metadata，以便在上游字段变化时重新处理。

## 3. 项目 JSON

```http
GET https://projects.scratch.mit.edu/{projectID}?token={projectToken}
```

返回项目的 `project.json` 内容。Scratch 2 和 Scratch 3 的结构需要分别处理。

当前 `spxbench` 会：

1. 读取 project JSON
2. 必要时从 `assetId + dataFormat` 补齐 `md5ext`
3. 对 SB3 项目使用固定版本 Scratch VM 重新序列化
4. 收集 project JSON 中的资源引用
5. 写入展开后的项目目录

## 4. 资源下载

```http
GET https://assets.scratch.mit.edu/internalapi/asset/{md5ext}/get/
```

`{md5ext}` 类似以下形式：

```text
83a9787d4cb6f3b7632b4ddfebf74367.wav
```

资源文件名同时承担内容标识作用。下载后应校验内容 hash，并以资源 hash 做幂等键。

## 5. 项目缩略图

当前 `spxbench` 使用的缩略图地址形式为：

```http
GET https://uploads.scratch.mit.edu/get_image/project/{projectID}_480x360.png
```

缩略图可以作为仓库元数据或项目索引字段保存，不建议将每次缩略图变化都生成 Git commit。

## 6. 当前 spxbench workflow

当前 workflow 文件是：

[`.github/workflows/scratch.yml`](https://github.com/goplus/spxbench/blob/main/.github/workflows/scratch.yml)

主要步骤如下：

```text
state-pull
  -> 从 Kodo 获取 scratch-popular/state.json

cloud-sync
  -> 将已有云端归档反向同步到 state

fetch
  -> 获取热门项目、元数据、project.json 和资源

plan
  -> 根据 state 和归档生成增量计划

process-batch
  -> 上传 SB2/SB3
  -> 运行转换和测试导入

issue-sync
  -> 为失败转换创建或复用问题记录

finalize
  -> 更新项目状态和汇总

state-push
  -> 将 state.json 写回 Kodo
```

## 7. SourceProvider 建议

Scratch Live 不应把上游 HTTP 细节暴露给 xgit-server。建议封装成以下接口：

```go
type ScratchSource interface {
    DiscoverPopular(ctx context.Context, offset, limit int) ([]ProjectSummary, error)
    GetProjectMeta(ctx context.Context, projectID string) (ProjectMeta, error)
    GetProjectJSON(ctx context.Context, projectID string, token string) ([]byte, error)
    GetAsset(ctx context.Context, md5ext string) (io.ReadCloser, error)
    GetThumbnail(ctx context.Context, projectID string) (io.ReadCloser, error)
}
```

若上游能够提供项目版本列表，再增加：

```go
type ScratchHistorySource interface {
    ListVersions(ctx context.Context, projectID string, cursor string) (VersionPage, error)
    GetVersion(ctx context.Context, projectID, versionID string) (ProjectSnapshot, error)
}
```

## 8. 历史能力限制

当前可确认的接口可以读取项目元数据、项目 JSON 和资源。是否能够获取 Scratch 保存历史，需要单独验证上游接口。

如果只有当前版本接口，Scratch Mirror 只能保存从开始采集之后观察到的历史。Discover 服务应使用更新时间、内容 hash 和周期性全量对账，避免把“没有变化”误判成“没有历史”。
