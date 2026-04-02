# 查询 K8s 环境应用日志 (Loki)

通过 logcli 查询 Loki 日志，用于排查测试/生产环境的运行时问题。

## 使用方式
/logcli <namespace> <container> <from> <to> <filter>
例如: /logcli test12 eq-alarm-service 2026-03-19T15:59:00 2026-03-19T16:01:00 NK202406131135101

## 执行步骤

### 1. 确定环境配置

根据用户提供的 namespace 确定 `org-id` 和 `addr`。**关键规律：同一个 Loki 地址下可能有多个 tenant（org-id）。**

**路由规则：**
1. 先根据 namespace 确定 **Loki 地址（addr）**
2. 再根据 namespace 确定 **租户（org-id）**

| Loki addr | org-id | 登录 addr | namespace | 说明 |
|-----------|--------|-----------|-----------|------|
| https://ops.eacon.com | **test** | https://ops.eacon.com | test01~test06, test09, test12, devdms, fbidev, fbitest, impdev, testdms | 测试租户 |
| https://ops.eacon.com | **prod** | https://ops.eacon.com | test08 | 生产租户（但在测试 Loki 上） |
| https://manage-nlt.eqfleetcmder.com | **prod** | https://manage-nlt.eqfleetcmder.com | nlt | NLT 独立 Loki |

**自动选择逻辑：**
- namespace 为 `test08` → addr=ops.eacon.com, org-id=**prod**
- namespace 为 `nlt` → addr=manage-nlt.eqfleetcmder.com, org-id=**prod**
- namespace 以 `test` 开头（非 test08）或属于其他已知测试 namespace → addr=ops.eacon.com, org-id=**test**
- 如果不确定，询问用户

### 2. 获取 Token（按集群分别缓存）

每个集群的 token 独立缓存，文件名为 `~/.claude/.logcli_token_<集群名>`，格式为 `日期|token`。每天只需登录一次。

```bash
# 根据 namespace 确定参数
NAMESPACE="test12"  # 替换为实际 namespace
if [ "$NAMESPACE" = "nlt" ]; then
  LOKI_ADDR="https://manage-nlt.eqfleetcmder.com"
  ORG_ID="prod"
  LOGIN_URL="https://manage-nlt.eqfleetcmder.com/api/v1/login"
  CLUSTER="nlt"
elif [ "$NAMESPACE" = "test08" ]; then
  LOKI_ADDR="https://ops.eacon.com"
  ORG_ID="prod"
  LOGIN_URL="https://ops.eacon.com/api/v1/login"
  CLUSTER="ops_prod"
else
  LOKI_ADDR="https://ops.eacon.com"
  ORG_ID="test"
  LOGIN_URL="https://ops.eacon.com/api/v1/login"
  CLUSTER="ops_test"
fi

TOKEN_FILE=~/.claude/.logcli_token_${CLUSTER}
TODAY=$(date +%Y-%m-%d)
CACHED_DATE="" && CACHED_TOKEN=""

# 尝试读取缓存
if [ -f "$TOKEN_FILE" ]; then
  CACHED_DATE=$(cut -d'|' -f1 "$TOKEN_FILE")
  CACHED_TOKEN=$(cut -d'|' -f2 "$TOKEN_FILE")
fi

# 如果缓存是今天的，直接使用
if [ "$CACHED_DATE" = "$TODAY" ] && [ -n "$CACHED_TOKEN" ]; then
  TOKEN=$CACHED_TOKEN
else
  # 重新登录获取 token
  TOKEN=$(curl -s "$LOGIN_URL" \
    -H 'accept: application/json' \
    -H 'content-type: application/json' \
    --data-raw '{"username":"jiayuxin","password":"8rnMTx36KxqP","role":"1"}' \
    | python3 -c "import sys,json; d=json.load(sys.stdin); print(d.get('token') or d.get('data',{}).get('token') or d.get('access_token') or d.get('jwt') or '')")
  if [ -n "$TOKEN" ]; then
    echo "${TODAY}|${TOKEN}" > "$TOKEN_FILE"
  fi
fi
```

- 缓存文件：`~/.claude/.logcli_token_ops_test`、`~/.claude/.logcli_token_ops_prod`、`~/.claude/.logcli_token_nlt`
- 每天首次查询自动登录并缓存，当天后续查询直接复用
- 如果缓存 token 导致查询返回 401/403，删除缓存文件重新获取
- 如果登录失败（返回非 JSON 或空 token），向用户索要 token 并手动写入缓存文件
- 如果登录凭据过期，提示用户提供新的凭据

### 2. 构建 logcli 查询（带重试）

执行 logcli 查询时，必须使用以下重试逻辑（最多重试 3 次，间隔 3 秒）。如果 3 次都失败，**立即停止并告知用户 Loki 服务不可用，不要继续尝试其他查询**。

```bash
logcli_query() {
  local MAX_RETRIES=3
  local RETRY_DELAY=3
  local attempt=1
  while [ $attempt -le $MAX_RETRIES ]; do
    OUTPUT=$(logcli --org-id=$ORG_ID --bearer-token=$TOKEN --addr=$LOKI_ADDR query \
      --forward --limit "$1" \
      --from="$2+08:00" --to="$3+08:00" \
      --output=raw \
      "$4" 2>&1)
    EXIT_CODE=$?
    # 检查是否为服务端错误（502/503/timeout）
    if echo "$OUTPUT" | grep -qE "502 Bad Gateway|503 Service|Query failed|run out of attempts"; then
      echo "[Attempt $attempt/$MAX_RETRIES] Loki server error, retrying in ${RETRY_DELAY}s..." >&2
      sleep $RETRY_DELAY
      attempt=$((attempt + 1))
    elif [ $EXIT_CODE -ne 0 ]; then
      # 非服务端错误（如 401/403），不重试
      echo "$OUTPUT"
      return $EXIT_CODE
    else
      echo "$OUTPUT"
      return 0
    fi
  done
  echo "ERROR: Loki 服务连续 ${MAX_RETRIES} 次请求失败，服务不可用。请稍后重试或检查 Loki 状态。" >&2
  return 1
}

# 使用方式：
# logcli_query <limit> <from> <to> '<LogQL>'
# 例如：
# logcli_query 5000 "2026-03-19T15:59:00" "2026-03-19T16:01:00" '{namespace="test12",container="eq-alarm-service"} |~ "ERROR"'
```

**重要：如果 `logcli_query` 返回失败（exit code 非 0），必须立即停止所有后续查询，直接告知用户 Loki 不可用。**

### 3. 参数说明

| 参数 | 说明 | 示例 |
|------|------|------|
| namespace | K8s namespace | test12 |
| container | 容器/服务名 | eq-alarm-service |
| from | 开始时间（北京时间） | 2026-03-19T15:59:00 |
| to | 结束时间（北京时间） | 2026-03-19T16:01:00 |
| filter | 日志过滤关键字（正则） | NK202406131135101 |

### 4. 多条件过滤

多个过滤条件使用多个 `|~` 串联（AND 关系）：

```
'{namespace="test12",container="eq-alarm-service"} |~ "NK202406131135101" |~ "matchStrategy"'
```

排除某些日志使用 `!~`：

```
'{namespace="test12",container="eq-alarm-service"} |~ "NK202406131135101" !~ "DEBUG"'
```

### 5. LogQL 正则语法陷阱

**`|~` 使用 RE2 正则引擎，有以下限制：**

1. **不支持 `.*` 跨字段匹配** — `|~ "AlarmVehicleJob.*filterIsInArea"` 看似合法但实际上在 Loki 中经常返回空结果（因为 `.` 不匹配换行，且行内匹配可能因为 RE2 优化被跳过）。
   - 错误：`|~ "AlarmVehicleJob.*filterIsInArea"`
   - 正确：`|~ "AlarmVehicleJob" |~ "filterIsInArea"`（两个 `|~` 串联，AND 关系）

2. **简单 OR 用 `|` 分隔是可以的** — `|~ "offline|Offline|matchStrategy"` 正常工作，但不要和 `.*` 混用。
   - 错误：`|~ "AlarmVehicleJob.*offline|DeviceOffline.*alarm"`
   - 正确：先用一个 `|~` 过滤主关键字，再用另一个 `|~` 过滤子关键字

3. **查询返回空时的排查顺序**：
   - 先确认是否正则语法问题（简化正则重试）
   - 再确认时间范围是否正确
   - 最后才考虑"确实没有这条日志"

## 注意事项

- 时间格式必须带时区后缀 `+08:00`（北京时间）
- 单次查询时间范围建议不超过 10 分钟，过大可能触发 Loki 502
- `--limit` 默认用 5000，如果用户需要更多可调大（最大 100000）
- `--org-id` 和 `--addr` 根据集群自动选择（test 或 nlt），新集群需用户确认配置
- 如果日志量较大，先用精确的 `|~` 过滤缩小范围再查
- 如果 token 获取失败或过期，向用户索要新的登录凭据
- 查询结果较多时，使用 context-mode 工具在沙箱中处理避免刷屏

## 已知环境与集群映射

| CLUSTER | org-id | Loki addr | 登录 addr | namespace | token 缓存文件 |
|---------|--------|-----------|-----------|-----------|---------------|
| ops_test | test | https://ops.eacon.com | https://ops.eacon.com | test01~test06, test09, test12, devdms, fbidev, fbitest, impdev, testdms | `~/.claude/.logcli_token_ops_test` |
| ops_prod | prod | https://ops.eacon.com | https://ops.eacon.com | test08 | `~/.claude/.logcli_token_ops_prod` |
| nlt | prod | https://manage-nlt.eqfleetcmder.com | https://manage-nlt.eqfleetcmder.com | nlt | `~/.claude/.logcli_token_nlt` |

**注意：** ops_test 和 ops_prod 共享同一个 Loki 地址（ops.eacon.com）和同一个登录接口，但使用不同的 org-id（tenant）。Token 可能通用，但为安全起见按 CLUSTER 分别缓存。
