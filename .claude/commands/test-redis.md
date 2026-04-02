# 查询测试环境 Redis

用户指定测试环境和 Redis 命令，自动连接对应环境的 Redis 执行命令。

## 使用方式
/test-redis <环境名> [redis类型] <Redis命令>
例如:
- /test-redis test06 GET alarm:alarm_vehicle
- /test-redis test06 cluster HGETALL vehicleAlarm:VIN001
- /test-redis test06 cluster-device KEYS alarm:*

## Redis 类型说明
- **redis**（默认）: 单机 Redis，每个环境独立
- **cluster**: Redis 集群，大部分环境共享 test06 节点（10.30.41.31:6279），test04/test09 独立
- **cluster-device**: Redis 集群 device 实例，共享规则同 cluster

## 执行步骤

1. 解析用户输入，提取环境名、Redis 类型（默认 redis）和 Redis 命令
2. 从配置文件 `~/Work/test.env.config.toml` 中读取连接信息：
   - redis 类型: 对应 `[testXX.redis]` 段的 `external_addr` 和 `password`
   - cluster 类型: 对应 `[testXX.redis_cluster]` 段的 `external_addr` 和 `password`
   - cluster-device 类型: 对应 `[testXX.redis_cluster_device]` 段
3. 使用 redis-cli 执行：
   ```bash
   # 单机
   redis-cli -h <host> -p <port> -a <password> <command>
   # 集群
   redis-cli -h <host> -p <port> -a <password> -c <command>
   ```
4. 如果结果较多，使用 context-mode 工具处理
5. 将结果摘要返回给用户

## 注意事项
- 默认使用 external_addr 连接
- 拒绝 FLUSHALL/FLUSHDB/DEL（批量通配）等危险操作，除非用户明确要求
- 如果用户未指定环境，询问目标环境
- cluster/cluster-device 连接时加 `-c` 参数启用集群模式
