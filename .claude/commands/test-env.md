# 查看测试环境信息

查看指定测试环境的所有中间件连接信息，或列出所有可用环境。

## 使用方式
/test-env [环境名]
例如:
- /test-env           -- 列出所有可用环境
- /test-env test06    -- 显示 test06 的所有中间件连接信息

## 执行步骤

1. 读取配置文件 `~/Work/test.env.config.toml`
2. 如果用户未指定环境，列出所有可用环境名称及简要说明
3. 如果指定了环境，展示该环境的所有中间件连接信息：
   - PostgreSQL: host, port, username, password
   - Redis: host, port, password
   - Redis Cluster: host, port, password（标注是否共享 test06）
   - Nacos: 地址, username, password（标注不同实例 nacos/nacosa/nacosb）
   - MinIO: web地址, S3地址, 凭证
   - Kafka（如有）: broker 地址

## 环境概览
- test01~test07, test09~test12（test08 不存在）
- test04, test09: 独立 Redis 集群
- 其余环境: 共享 test06 节点的 Redis 集群（10.30.41.31:6279）
- test09: Nacos 密码为 nacos2024
- test11: 临时环境
