# Scratch Live

## 1. 目标

Scratch Live 将当前 Scratch 后端包装成只读 Git Smart HTTP 服务。

```text
Scratch API
  -> Scratch Source Adapter
  -> Snapshot Materializer
  -> ReadOnly Git Backend
  -> Git Smart HTTP
```

它不负责持久化 Scratch Mirror，也不负责 XBP 转换。

## 2. 服务边界

建议独立仓库维护：

```text
xgit-scratch-live/
├── cmd/xgit-scratch-live
├── internal/scratchapi
├── internal/snapshot
├── internal/gitbackend
└── pkg/config
```

独立维护可以隔离：

- Scratch 后端凭据
- 上游接口变化
- 429/503 重试策略
- Scratch project JSON 兼容逻辑
- 独立发布和回滚周期

公共 Git 协议包和接口契约可以由 xgit-server 发布，Scratch Live 不依赖 xgit-server 的 `internal` 包。

## 3. Git remote

```text
https://scratch-live.io/{author}/{projectID}.git
```

建议默认暴露：

```text
refs/heads/main
```

`main` 指向当前可读取的项目快照。

如果上游提供稳定历史版本，可以额外暴露：

```text
refs/tags/source/{versionID}
refs/versions/{versionID}
```

Scratch Live 默认只读，不接受 `git-receive-pack`。

## 4. Git 协议复用

Git 协议引擎依赖通用的只读 backend：

```go
type GitRepositoryBackend interface {
    GetMetadata(ctx context.Context, repoID string) (RepositoryMetadata, error)
    ReadRefs(ctx context.Context, repoID string) ([]Ref, error)
    ReadObject(ctx context.Context, repoID string, oid ObjectID) (io.ReadCloser, error)
    IsReadOnly(ctx context.Context, repoID string) (bool, error)
}
```

Scratch Live 将 Scratch Source Adapter 适配成这个接口，再复用 `pkg/gitprotocol`。

## 5. 读取流程

### `info/refs`

1. 根据作者名和项目 ID 定位 Scratch 项目
2. 请求 Scratch 元数据
3. 获取当前 project JSON 和资源引用
4. 规范化项目快照
5. 生成稳定的 tree 和 commit
6. 将对象写入 Live 缓存
7. 返回 `main` ref

### `git-upload-pack`

1. 读取缓存的 refs
2. 解析客户端 want/have
3. 从缓存或对象临时存储读取 Git objects
4. 生成 packfile
5. 流式返回给 Git 客户端

缓存未命中时可以同步生成，也可以先创建 materialize job，再等待结果。

## 6. 稳定 commit

相同 Scratch 项目版本必须生成相同 commit。commit 生成应使用：

- source project ID
- source version ID 或内容 hash
- 规范化后的 project JSON
- 资源 hash
- 稳定的 author 和 committer 元数据

不能把当前请求时间直接写进 commit，否则同一项目每次访问都会产生不同 SHA。

## 7. 与 Scratch Sync 的关系

Scratch Sync 有两种读取方式。

### Git 模式

```text
Scratch Sync
  -> git fetch scratch-live
  -> git push scratch-mirror
```

### Source 模式

```text
Scratch Sync
  -> Scratch Source API
  -> Snapshot Materializer
  -> xgit-server Native Repository
```

Git 模式适合验证协议链路。Source 模式更适合生产镜像，因为可以保留 source version、cursor 和原始元数据。
