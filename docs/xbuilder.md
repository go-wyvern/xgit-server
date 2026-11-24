# XBuilder 集成

## 1. 目标

Scratch Mirror 的主要消费者是 XBuilder 测试链路。

```text
Scratch commit
  -> sb2xbp
  -> XBP
  -> XBuilder 项目或测试输入
  -> spxbench
```

## 2. XBuilder 版本关系

Git commit 是技术版本。XBuilder Project Revision 和 Release 是业务版本。

```text
Scratch Mirror commit
  -> sb2xbp conversion
  -> XBP artifact
  -> XBuilder Project
  -> XBuilder Release
  -> spxbench test case
```

XBuilder 业务数据库保存：

```text
source_project_id
source_version_id
mirror_repo_id
mirror_commit
xbuilder_project_id
xbuilder_revision
xbuilder_release
conversion_status
```

## 3. 转换任务

转换任务从 Git commit 导出 SB2/SB3，而不是读取 worker 本地目录。

```text
GET mirror repo at commit
  -> export project archive
  -> sb2xbp
  -> validate XBP
  -> upload XBP
  -> update XBuilder metadata
```

这样可以保证转换结果与历史 commit 一一对应。

## 4. 资源和缓存

Git Mirror 保存项目源文件和历史。XBP、SPX Replay、截图和 pHash 等测试产物可以继续使用独立对象桶。

spxbench 的运行缓存不应成为 Git Mirror 的事实来源。

## 5. 失败处理

转换失败不回滚 Scratch Mirror commit。

转换状态保存：

```text
commit
source project ID
source version ID
converter version
error message
log artifact
retry count
```

当 `sb2xbp` 或 XBuilder 版本变化时，可以对同一个 Git commit 重新转换，比较不同转换器版本的结果。
