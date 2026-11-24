# 运维设计

## 1. 服务状态

需要分别监控：

- Scratch API 请求成功率、429、503 和延迟
- Scratch Live materialize 延迟
- Discover cursor 延迟
- Sync 队列积压
- mirror 版本导入数量
- Git clone、fetch、push 延迟
- packfile 大小和对象桶容量
- sb2xbp 转换成功率
- spxbench 导入和测试成功率

## 2. 任务状态

每个项目至少保存：

```text
source project ID
source cursor
source version ID
mirror repo ID
mirror commit
sync status
conversion status
last error
retry count
updated at
```

## 3. 数据恢复

恢复顺序：

1. 恢复 PostgreSQL
2. 恢复对象桶
3. 检查 refs 指向的 objects
4. 重放 outbox event
5. 恢复 Discover cursor
6. 重试 Sync 和转换任务

## 4. 限流和重试

Scratch Discover 和 Scratch Sync 的上游请求必须独立限流。后台镜像不能影响 Git 客户端访问 Scratch Mirror 和 XBuilder。

429、503 和网络超时使用指数退避。单项目失败不阻塞批次，cursor 只在批次安全完成后推进。

## 5. 安全

- Scratch Live 使用独立的上游凭据
- Git token 使用短期、仓库级 scope
- 不向客户端暴露对象桶凭据
- Scratch Mirror 默认对普通用户只读
- XBuilder push 做 pack 大小、对象数量和资源类型限制
- 外部 project JSON 和资源必须经过校验
