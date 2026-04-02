# 查询测试环境 PostgreSQL

用户指定测试环境和 SQL，自动连接对应环境的 PostgreSQL 执行查询。

## 使用方式
/test-pg <环境名> <SQL语句>
例如: /test-pg test06 SELECT * FROM tb_risk_info LIMIT 5

## 执行步骤

1. 解析用户输入，提取环境名（如 test01~test12，不含 test08）和 SQL 语句
2. 从配置文件 `~/Work/test.env.config.toml` 中读取对应环境的 PostgreSQL 连接信息：
   - 使用 `external_addr`（格式 host:port）
   - `username` 和 `password`
   - 数据库名固定为 `lbs`
3. 使用 psql 执行查询：
   ```bash
   PGPASSWORD=<password> psql -h <host> -p <port> -U <username> -d lbs -c "<SQL>"
   ```
4. 如果查询结果较多，使用 context-mode 工具在沙箱中执行并索引结果，避免刷屏
5. 将结果摘要返回给用户

## 注意事项
- 默认使用 external_addr 连接（需 VPN 或办公网络）
- 只执行 SELECT 查询，拒绝 DROP/DELETE/TRUNCATE/UPDATE/INSERT 等写操作，除非用户明确要求
- 如果用户未指定环境，询问目标环境
- 如果连接失败，提示检查 VPN 或网络连接
