# 查询测试环境 Nacos 配置

用户指定测试环境和配置项，自动连接对应环境的 Nacos 查询配置。

## 使用方式
/test-nacos <环境名> [nacos实例] <dataId> [group]
例如:
- /test-nacos test06 eq-alarm-service.properties
- /test-nacos test06 eq-alarm-service.properties DEFAULT_GROUP
- /test-nacos test04 nacosa eq-alarm-service.properties

## Nacos 实例说明
- **nacos**（默认）: 主 Nacos，端口 8848
- **nacosa**: Unita Nacos，端口 18848（部分环境有）
- **nacosb**: Unitb Nacos，端口 28848（部分环境有）

## 执行步骤

### 1. 解析输入与读取连接信息

解析用户输入，提取环境名、nacos 实例（默认 nacos）、dataId 和 group（默认 DEFAULT_GROUP）。

从配置文件 `~/Work/test.env.config.toml` 中读取连接信息：
- 使用 `external_addr`（完整 URL 格式）
- `username` 和 `password`

**地址处理**：`external_addr` 可能包含 `/nacos/`、`/nacos/#/` 等后缀，API 调用时统一截取到 `/nacos` 基路径：
```python
import re
base_url = re.sub(r'/nacos.*', '/nacos', external_addr)
```

### 2. 认证（带降级）

先尝试获取 accessToken，如果认证失败则降级为无认证模式：

```bash
# 尝试登录
RESULT=$(curl -s -X POST '<nacos_addr>/v1/auth/login' -d 'username=<user>&password=<pass>')

# 检查是否成功获取 token
TOKEN=$(echo "$RESULT" | python3 -c "import sys,json;
try:
    d=json.load(sys.stdin)
    print(d.get('accessToken',''))
except:
    print('')" 2>/dev/null)

# 如果 token 为空（返回 'unknown user!' 或其他错误），降级为无认证
if [ -z "$TOKEN" ]; then
    ACCESS_PARAM=""   # 不传 accessToken
else
    ACCESS_PARAM="&accessToken=${TOKEN}"
fi
```

**已知特殊认证**：
- test09 的 nacos 密码是 `nacos2024`，其他环境都是 `nacos`
- 部分低版本 Nacos（如 1.4.3）不需要认证，登录返回 `unknown user!` 但直接查询可用

### 3. dataId 模糊搜索与自动补全

**关键规则：当用户提供的 dataId 直接查询返回空时，自动执行模糊搜索找到正确的 dataId。**

```bash
# 第一步：直接查询
RESULT=$(curl -s "${NACOS_ADDR}/v1/cs/configs?dataId=${DATAID}&group=${GROUP}&tenant=${ACCESS_PARAM}")

# 如果返回空或 "config data not exist"
if [ -z "$RESULT" ] || echo "$RESULT" | grep -q "config data not exist"; then
    # 第二步：模糊搜索（在所有 group 中搜索包含用户关键字的 dataId）
    SEARCH_RESULT=$(curl -s "${NACOS_ADDR}/v1/cs/configs?dataId=*${DATAID}*&group=&search=blur&pageNo=1&pageSize=20${ACCESS_PARAM}")
    # 展示匹配列表让用户确认，或自动选择唯一匹配
fi
```

**典型场景**：
- 用户输入 `AlarmThresholdConfig` → 直接查询失败 → 模糊搜索找到 `com.eq.alarm.config.AlarmThresholdConfig`（group=eq-alarm-service）→ 自动使用正确的 dataId 和 group 重新查询
- 用户输入 `eq-alarm-service.properties` → 直接查询成功 → 直接返回

### 4. group 自动发现

**当用户未指定 group 且默认 DEFAULT_GROUP 查不到时，自动搜索正确的 group：**

搜索顺序：
1. 先用用户指定的 group（或默认 DEFAULT_GROUP）查
2. 查不到则用 `group=` (空) 做模糊搜索，找到实际 group
3. 用找到的 group 重新查询

**本项目常用 group**：
- `eq-alarm-service` — 告警服务专用配置（AlarmThresholdConfig 等）
- `DEFAULT_GROUP` — 通用配置

### 5. 查询与返回

```bash
# 精确查询
curl -s "${NACOS_ADDR}/v1/cs/configs?dataId=${DATAID}&group=${GROUP}&tenant=${ACCESS_PARAM}"

# 模糊搜索配置列表
curl -s "${NACOS_ADDR}/v1/cs/configs?dataId=${PATTERN}&group=&search=blur&pageNo=1&pageSize=20${ACCESS_PARAM}"
```

使用 context-mode 工具索引大结果，搜索用户关心的配置项，将配置内容或摘要返回给用户。

## 常用场景
- 查看报警阈值配置: `/test-nacos test04 AlarmThresholdConfig` → 自动补全为 `com.eq.alarm.config.AlarmThresholdConfig`
- 查看服务配置: `/test-nacos test06 eq-alarm-service.properties`
- 查看所有服务配置列表: `/test-nacos test06 *` (模糊搜索)
- 对比不同环境配置差异

## 注意事项
- 只读查询，不执行配置修改，除非用户明确要求
- 大配置（>5KB）使用 context-mode 索引后按需搜索，避免刷屏
