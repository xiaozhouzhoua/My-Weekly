# Redis 持久化：RDB 与 AOF

## 概述

Redis 是内存数据库，重启即失数据。要落盘，Redis 提供两种机制：**RDB 快照** 与
**AOF 日志**，还可以两者并用（混合持久化）。

| 机制 | 原理 | 优点 | 缺点 |
|------|------|------|------|
| RDB | 定时把内存快照写成二进制文件 | 文件小、恢复快 | 两次快照间的数据可能丢失 |
| AOF | 把每条写命令追加到日志 | 丢数据少、可读 | 文件大、恢复慢 |
| 混合 | AOF 头部嵌 RDB + 增量日志 | 兼顾两者 | 结构稍复杂 |

## RDB 快照

### 触发方式

```bash
# 手动触发（阻塞，生产慎用）
SAVE

# 后台子进程生成快照（推荐）
BGSAVE

# 满足条件自动触发（默认配置示例）
save 900 1      # 900 秒内至少 1 次写
save 300 10     # 300 秒内至少 10 次写
save 60 10000   # 60 秒内至少 10000 次写
```

### 相关配置

```
# 快照文件名与目录
dbfilename dump.rdb
dir /var/lib/redis

# 生成失败时是否拒绝写入（防止以为有备份实际没有）
stop-writes-on-bgsave-error yes

# 是否压缩（CPU 换体积）
rdbcompression yes
```

### 优缺点

- 优点：紧凑、恢复快，适合做冷备、灾备、迁移
- 缺点：两次快照间隔内的写入会丢（取决于 `save` 频率）

## AOF 追加日志

默认关闭，开启：

```
appendonly yes
appendfilename "appendonly.aof"
```

### 写入策略

```
# 每次写命令都 fsync，最安全也最慢
appendfsync always

# 每秒 fsync 一次，最多丢 1 秒数据（官方推荐）
appendfsync everysec

# 交给操作系统，最快，崩溃时丢失不可控
appendfsync no
```

### AOF 重写

AOF 会无限增长，需要压缩成等价的最小命令集：

```
# 触发条件：AOF 比上次重写后增长一倍、且超过 64MB
auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb
```

手动触发：`BGREWRITEAOF`（子进程完成，不阻塞主线程）。

### 恢复与修复

AOF 尾部损坏时，用自带工具修复（会截断损坏部分）：

```bash
redis-check-aof --fix appendonly.aof
```

## 混合持久化（RDB + AOF）

```
aof-use-rdb-preamble yes
```

重写后的 AOF 文件：开头是一段 RDB 快照，后面跟增量 AOF 命令。
既保留了 RDB 恢复快的优点，又保留 AOF 丢数据少的优点。Redis 4.0+ 推荐开启。

## 选型建议

| 场景 | 建议 |
|------|------|
| 纯缓存、可重建、丢了无所谓 | 关闭持久化 |
| 需要持久化、能容忍分钟级丢失 | 只用 RDB |
| 丢数据代价高、要求秒级 | AOF `everysec` |
| 生产通用 | **混合持久化 + `everysec`** |

## 小结

- RDB 是「快照」，AOF 是「流水账」，混合是「快照 + 增量流水」
- 生产默认推荐：`appendonly yes` + `aof-use-rdb-preamble yes` + `appendfsync everysec`
- 别只看配置，定期演练一次**从备份恢复**，才知道真的能救回来
