# xgit-server 设计

## 1. 目标

xgit-server 是 Scratch Mirror 和 XBuilder 的统一 Git Smart HTTP 存储服务。

它不负责发现 Scratch 项目，也不直接访问 Scratch 后端。它接收 Scratch Sync 写入的 Git 仓库，并为 Git 客户端提供标准 remote。

## 2. 仓库类型

### Scratch Mirror

```text
https://scratch-mirror.io/{author}/{projectID}.git
```

由 Scratch Sync 写入，默认只读给普通用户。

### XBuilder

```text
https://xbuilder.io/{author}/{projectID}.git
```

支持用户或 Agent push。push 成功后异步更新 XBuilder Project、Revision、Release 和预览。

## 3. Git Smart HTTP

```text
GET  /git/{namespace}/{repo}.git/info/refs?service=git-upload-pack
POST /git/{namespace}/{repo}.git/git-upload-pack
POST /git/{namespace}/{repo}.git/git-receive-pack
```

协议响应是 packet-line 和 packfile 二进制数据。

建议协议引擎依赖通用接口：

```go
type GitRepositoryBackend interface {
    GetMetadata(ctx context.Context, repoID string) (RepositoryMetadata, error)
    ReadRefs(ctx context.Context, repoID string) ([]Ref, error)
    ReadObject(ctx context.Context, repoID string, oid ObjectID) (io.ReadCloser, error)
    IsReadOnly(ctx context.Context, repoID string) (bool, error)
}
```

xgit-server 的 `NativeRepositoryProvider` 实现完整读写能力。xgit-scratch-live 可以实现只读 backend，并复用同一协议包。

## 4. 存储

```text
PostgreSQL
  repository
  refs
  HEAD
  token
  ACL
  sync state
  outbox event

Object Bucket
  Git objects
  packfile
  pack index
  project snapshot
  source metadata
```

典型对象路径：

```text
repos/{repoID}/objects/{oid}
repos/{repoID}/packs/{packID}.pack
repos/{repoID}/packs/{packID}.idx
repos/{repoID}/snapshots/{commit}/manifest.json
```

objects 先写入对象桶，再通过事务 CAS 更新 refs。失败对象由 GC 清理。

## 5. 模块

```text
xgit-server/
├── gitprotocol/       # Git Smart HTTP 协议核心
├── repository/        # Native Repository、refs 和权限
├── storage/           # PostgreSQL、对象桶和缓存
├── snapshot/          # 项目快照与 Git tree/commit 转换
├── sync/              # Source 到 Native Repository 的同步编排
├── scratch/format/    # SB2/SB3 展开、规范化和校验
├── xbuilder/          # XBuilder 文件映射和业务投影
├── jobs/              # 重试、repack、GC 和投影任务
└── metrics/           # 互动数据时间序列
```

Scratch Mirror 与 XBuilder 共用 Native Repository，不实现两套 Git 存储 provider。

## 6. 控制面 API

创建仓库：

```http
POST /api/v1/namespaces/{namespace}/repos
```

```json
{
  "name": "project-123",
  "kind": "scratch-mirror",
  "readOnly": true,
  "defaultBranch": "main",
  "source": {
    "provider": "scratch",
    "projectId": "123"
  }
}
```

签发 Git token：

```http
POST /api/v1/repos/{repoId}/tokens
```

```json
{
  "scope": "read",
  "expiresIn": 3600
}
```

查询 refs：

```http
GET /api/v1/repos/{repoId}/refs
```

## 7. push 流程

```text
认证
  -> 读取 refs
  -> 创建 push lease
  -> 接收 packfile
  -> 校验 objects 和 ref update
  -> 写入对象桶
  -> CAS 更新 refs
  -> 写入 outbox event
  -> 异步项目投影
```

Scratch Mirror 的写入 token 只授予 Scratch Sync。普通 Git 客户端使用 read token。

## 8. 部署

第一阶段可以部署：

```text
xgit-api
  控制面、Git Smart HTTP、Native Repository

xgit-worker
  Scratch Sync、repack、GC、sb2xbp 投影任务

xgit-scratch-live
  独立的 Scratch Live Git adapter

PostgreSQL + 对象桶
```

xgit-scratch-live 如果需要复用协议核心，使用 xgit-server 发布的公共 Git protocol package 和 contracts package。
