# Azure OpenAI gpt-image-2 首个请求 429 排障与体验优化手册

日期: 2026-05-14  
对象: Azure OpenAI / Microsoft Foundry `gpt-image-2`、`GlobalStandard` / `DataZoneStandard` 图像生成部署

## 结论摘要

现实里确实会出现“明明有 10 RPM, 第一个请求就 429”的现象。这里的“第一个请求”通常只是某个客户端、某个 worker、某个用户会话看到的第一个请求, 不是 Azure 后端在该 deployment 上看到的第一个请求。

常见根因通常是这些机制叠加:

1. Azure OpenAI 的 RPM 不是只按整分钟总数判断, 还会用 1 秒或 10 秒级的小窗口评估 burst。
2. 限流是 deployment 级别的, 多个进程、用户、SDK 自动重试、超时后仍在服务端处理的请求会共享同一个窗口。
3. `Standard` / `GlobalStandard` 是共享资源池。即使 quota 没变, 也可能出现系统容量型 429, 或临时有效限额调整。
4. 429 类型不同: `RateLimitReached` 通常带 `retry-after` / `x-ratelimit-reset-requests`; `EngineOverloaded` 可能没有 retry header。
5. OpenAI SDK / Azure SDK 默认会重试。业务日志里的一次“请求”可能已经在底层放大为多次 HTTP 调用。

实测在一个 `GlobalStandard`、2 RPM 的 `gpt-image-2` deployment 上观察到了两类 429:

- `RateLimitReached`: `retry-after` 从 60 线性下降到 33, `x-ratelimit-limit-requests` 保持为 2。
- `EngineOverloaded`: 没有 `retry-after`, 需要客户端自己做指数退避和随机抖动。

## 官方依据

### 1. gpt-image-2 默认 RPM

Azure OpenAI quota 文档列出 `gpt-image-2` 默认 quota:

- `GlobalStandard`: 9 RPM （实际只能开出到2）
- `DataZoneStandard`: 3 RPM

参考: [Azure OpenAI quotas and limits](https://learn.microsoft.com/en-us/azure/foundry/openai/quotas-limits)

注意: 默认 quota 是每个 subscription、region、model/deployment type 的 quota 池。实际某个 deployment 的 RPM 取决于你分配到该 deployment 的 capacity, 不等于“账号里所有区域的总和”。

### 2. RPM 会按小窗口评估

Azure 文档说明: RPM rate limits 期望请求在一分钟内均匀分布。如果平均流量没有维持, 即使整分钟总量没超过 quota, 也可能 429。Azure 会在较小时间段评估 incoming request rate, 通常是 1 秒或 10 秒。

参考: [Manage Azure OpenAI quota - Understanding rate limits](https://learn.microsoft.com/en-us/azure/foundry-classic/openai/how-to/quota?tabs=rest#understanding-rate-limits)

对 `gpt-image-2` 这种低 RPM 模型尤其重要:

| Deployment RPM | 平滑间隔建议 | 含 20% 安全余量 |
|---:|---:|---:|
| 2 RPM | 30s / request | 36s / request |
| 9 RPM | 6.7s / request | 8s / request |
| 10 RPM | 6s / request | 7.2s / request |

如果 10 RPM 的 deployment 在 1 秒内打 2 到 3 个请求, 从整分钟看似没超, 但小窗口可能已经超了。

### 3. 失败请求也会影响限流

Azure 文档和 OpenAI Cookbook 都强调: 不成功的请求仍然可能计入 per-minute rate limit。连续无退避重试会浪费请求预算, 让吞吐变差。

参考:

- [Azure OpenAI quota - rate limit best practices](https://learn.microsoft.com/en-us/azure/foundry-classic/openai/how-to/quota?tabs=rest#rate-limit-best-practices)
- [OpenAI Cookbook - How to handle rate limits](https://developers.openai.com/cookbook/examples/how_to_handle_rate_limits)

### 4. 共享池和临时有效限额调整

Azure 文档将 429 分为多类, 包括:

- Rate limit exceeded
- System capacity throttling
- Temporary rate limit adjustment
- Token budget exceeded by request parameters

其中临时有效限额调整是 `Standard` / `GlobalStandard` 共享资源池的保护机制: 配置 quota 没变, 但 response header 里的有效 limit 可能低于你配置的 quota, 通常会在流量稳定后恢复。

参考: [Understanding 429 throttling errors and what to do](https://learn.microsoft.com/en-us/azure/foundry-classic/openai/how-to/quota?tabs=rest#understanding-429-throttling-errors-and-what-to-do)

### 5. SDK 自动重试会放大请求

Azure 文档说明 Azure OpenAI Python SDK 默认会对 429 和 transient errors 自动重试, 默认 2 次。若再套外层重试库, 要把 SDK `max_retries=0`, 否则一次业务调用会放大成更多 HTTP 请求。

参考: [Azure OpenAI quota - rate limit best practices](https://learn.microsoft.com/en-us/azure/foundry-classic/openai/how-to/quota?tabs=rest#rate-limit-best-practices)

## 本地实测记录

测试对象:

```text
Azure CLI profile: isolated profile for the target subscription
Subscription: <subscription-id>
Resource group: <resource-group>
Account: <azure-openai-account>
Region: <region>
Deployment: gpt-image-2
SKU: GlobalStandard
Capacity/RPM: 2
API: /openai/deployments/gpt-image-2/images/generations?api-version=2025-04-01-preview
Client: raw httpx, no SDK retry
```

### Test 1: burst + probe

先打 6 个并发, 再每秒 probe 12 次。

结果:

```text
200 = 2
429 = 16
```

典型 header:

```text
burst-1   retry-after=60   x-ratelimit-reset-requests=60   limit=2
probe-1   retry-after=59   x-ratelimit-reset-requests=59   limit=2
probe-2   retry-after=58   x-ratelimit-reset-requests=58   limit=2
probe-12  retry-after=44   x-ratelimit-reset-requests=44   limit=2
```

观察:

- `retry-after` 和 `x-ratelimit-reset-requests` 按窗口剩余秒数递减。
- `x-ratelimit-limit-requests` 一直是 2, 没有低于配置 capacity。
- 出现过 `EngineOverloaded` 429, 这类没有 `retry-after`。

### Test 2: capacity + 1 后停止, 再恢复 probe

3 个并发请求, 即 `capacity + 1`:

```text
200 = 2
429 EngineOverloaded = 1
```

等待 75 秒后单 probe:

```text
200
x-ratelimit-limit-requests = 2
x-ratelimit-remaining-requests = 1
```

观察: 该轮没有因为一个 429 导致长时间不可恢复。

### Test 3: 更长的连续 429

8 并发 burst + 30 次连续 probe, interval 0.5 秒。

结果:

```text
200 = 9
429 = 34
```

`RateLimitReached` header:

```text
b-probe-1   retry-after=60   reset=60   limit=2
b-probe-10  retry-after=51   reset=51   limit=2
b-probe-20  retry-after=42   reset=42   limit=2
b-probe-30  retry-after=33   reset=33   limit=2
```

等待 65 秒后单 probe:

```text
200
x-ratelimit-limit-requests = 2
x-ratelimit-remaining-requests = 1
```

观察:

- 连续 429 没有把可见 `limit` 压低。
- 窗口结束后可以恢复。
- 但连续 429 仍然是坏策略, 因为它浪费请求预算、干扰业务队列, 并可能在更高流量场景触发共享池保护。

## 为什么会出现“第一个 request 就 429”

### 场景 A: 你看到的第一个, 不是 deployment 看到的第一个

多进程、多 pod、多用户、多脚本共用一个 deployment。某个 worker 的第一条请求发出时, deployment 的请求窗口可能已经被其他 worker 占满。

这也是最常见原因。解决方式不是在每个 worker 本地做 rate limit, 而是做跨进程的共享 token bucket。

### 场景 B: 上一个请求超时了, 但服务端仍在处理

图像生成请求可能 30 到 80 秒才返回。客户端 20 秒超时并重试时, 原请求可能已经被服务端接收并计入限流窗口。于是“新的第一个请求”会直接 429。

排查方式:

- 每个请求记录 client request id。
- 记录 client timeout 和 server response latency。
- 对用户点击做幂等去重, 不要超时后立刻盲重试。

### 场景 C: SDK 自动重试隐藏了真实请求次数

OpenAI / Azure OpenAI SDK 默认可能自动重试。业务代码里一次调用, 底层可能已经发了 2 到 3 次。你日志里的“first request”可能是最后一次失败。

排查方式:

- 打开 SDK debug logging。
- 统一在 HTTP transport 层记录 attempt number。
- 如果外层使用重试库, 设置 SDK `max_retries=0`。

### 场景 D: 小窗口 burst 限流

10 RPM 等于平均 6 秒一个请求, 不是允许一秒内打 10 个请求。冷启动、队列恢复、多个用户同时点击都会制造 burst。

排查方式:

- 记录每个 deployment 每秒进入的请求数。
- 对低 RPM deployment 不要使用“每分钟释放 N 个 token 后瞬间消费”的粗粒度限流。
- 用 leaky bucket 平滑出队。

### 场景 E: 共享池容量压力或 EngineOverloaded

实测中出现过:

```json
{
  "code": "EngineOverloaded",
  "message": "We are currently servicing too many requests at the moment. Please wait and try again later."
}
```

这类 429 可能没有 `retry-after`。它不是你的 RPM 配置一定超了, 而是共享后端当时过载。

排查方式:

- 看 error code 是否是 `EngineOverloaded` / system capacity。
- 看是否没有 `retry-after`。
- 换 region/deployment 做同样单 probe。
- 如果长期高频出现, 考虑 PTU 或支持工单。

### 场景 F: 临时有效限额调整

Azure 文档说明共享池可能临时降低有效 rate limit。判断标准是 response header 里的 `x-ratelimit-limit-*` 低于配置 quota。

我们这次实测没有触发, `x-ratelimit-limit-requests` 始终等于 2。但生产排障时要保留这个检查。

## 排障 Runbook

### Step 1: 每个请求必须记录这些字段

最小日志字段:

```text
timestamp_utc
deployment
region
model
status_code
error.code
error.message prefix
retry-after
retry-after-ms
x-ratelimit-limit-requests
x-ratelimit-remaining-requests
x-ratelimit-reset-requests
x-ratelimit-limit-tokens
x-ratelimit-remaining-tokens
x-ratelimit-reset-tokens
x-ms-region
x-request-id
apim-request-id
client_attempt
client_timeout_s
queue_wait_ms
inflight_count
```

没有这些字段, 很难区分 quota、burst、共享池过载、SDK 重试放大。

### Step 2: 区分 429 类型

| 类型 | 常见特征 | 优先处理 |
|---|---|---|
| `RateLimitReached` | 带 `retry-after` / reset header | 按 header 暂停该 deployment |
| `EngineOverloaded` | 可能没有 retry header, message 是 too many requests / service busy | 指数退避 + 随机抖动, 快速切换其他 region |
| 临时有效限额调整 | header 里的 effective limit 低于配置 quota | 降载、等恢复、分流、必要时开工单 |
| SDK 重试放大 | 应用 1 次, HTTP 多次 | 关闭双重重试, 统一重试层 |
| 小窗口 burst | minute 总量没超但秒级集中 | 平滑节流, 分散队列释放 |

### Step 3: 查配置 quota 与部署 capacity

```bash
az cognitiveservices account deployment show \
    -g <resource-group> \
    -n <azure-openai-account> \
  --deployment-name gpt-image-2 \
  --query "{deployment:name,model:properties.model.name,sku:sku.name,capacity:sku.capacity,state:properties.provisioningState}" \
  -o table

az cognitiveservices usage list -l <region> \
  --query "[?name.value=='OpenAI.GlobalStandard.gpt-image-2'].{name:name.value,current:currentValue,limit:limit,unit:unit}" \
  -o table
```

### Step 4: 做安全复现实验

不要暴力撞 429。推荐:

1. 冷却 `max(retry-after, 70s)`。
2. 单请求 probe, 确认是否健康。
3. `capacity + 1` 并发一次, 一旦出现 429 立刻停。
4. 记录 headers。
5. 等 `max(retry-after, 70s)` 后单请求 probe。

如果想验证连续 429 对恢复窗口的影响, 控制在一个低 RPM deployment 上, 不要打所有区域。生产系统不应该依赖反复触发 429 来判断可用性。

### Step 5: 判断是否要升级或开工单

开工单前准备:

- UTC timestamp
- `x-request-id`
- `apim-request-id`
- region / deployment / model / SKU / capacity
- 429 error code 和 response headers
- 同一时间窗口的发送速率、in-flight、重试次数
- Azure Monitor 429/latency/usage 截图或导出

需要升级/开工单的信号:

- 单请求 probe 多次 `EngineOverloaded`。
- `x-ratelimit-limit-*` 长时间低于配置 quota。
- 按 `retry-after` 等待后仍长期 429。
- 多个 region 同时出现系统容量型 429。

## 生产预防措施

### 1. 每个 deployment 使用共享 token bucket

不要每个进程各自按 10 RPM 放行。要用 Redis / durable queue 做全局 bucket。

建议速率:

```text
2 RPM deployment: 1 request / 36s
9 RPM deployment: 1 request / 8s
10 RPM deployment: 1 request / 7.2s
```

这里包含约 20% 安全余量, 用来吸收时钟偏差、网络延迟、SDK retry、请求耗时波动。

### 2. 用 leaky bucket 平滑出队

Token bucket 允许短时 burst, 对 Azure OpenAI 低 RPM deployment 不友好。更推荐 leaky bucket 或 paced queue, 固定间隔出队。

### 3. deployment 级 circuit breaker

收到 429 后不要继续打同一个 deployment:

```text
if retry-after or retry-after-ms exists:
    mark deployment unavailable until now + retry_after + jitter
else:
    retry delay: 2s, 4s, 8s, 16s, max 60s + jitter
```

同时把请求转给其他健康 region。

### 4. 多区域 health-aware routing

一个常见做法是把 `gpt-image-2` 分散到多个 region。例如 5 个 region、每个 2 RPM 时, 总计可以形成 10 RPM 的全局池:

```text
eastus2: 2 RPM
westus3: 2 RPM
polandcentral: 2 RPM
swedencentral: 2 RPM
uaenorth: 2 RPM
```

不要简单 round-robin。应该按 deployment 的 `next_available_at`、in-flight、最近 429 类型、p95 latency 做路由。

### 5. 用户体验策略

图像生成天然慢, 直接让用户反复点击会制造重复请求。建议:

- 请求入队后立即返回 job id。
- UI 显示排队位置和预计时间。
- 用户重复点击时返回同一个 job id, 不新建请求。
- 429 时显示“排队中/正在重试”, 不暴露原始错误。
- 支持取消任务, 但取消不保证服务端已接收请求不计入限流。

### 6. 重试层只能有一个

如果用 SDK 自动重试:

```python
client = AzureOpenAI(..., max_retries=5)
```

外层不要再套 Tenacity。

如果要自己控制队列、熔断、跨 region fallback:

```python
client = AzureOpenAI(..., max_retries=0)
```

然后在自己的调度层做 retry。

### 7. 对 EngineOverloaded 单独处理

`EngineOverloaded` 不一定给 `retry-after`。它应该触发:

- 当前 deployment 短暂熔断。
- 立即切到其他 region。
- 指数退避重试。
- 若多 region 同时出现, 降低全局出队速率。

### 8. 考虑 PTU 或更高保障方案

如果业务是 latency-sensitive 或 mission-critical, `GlobalStandard` 的共享池特性无法完全避免抖动。Azure 文档建议对一致吞吐和 latency SLA 有要求的生产工作负载考虑 Provisioned Throughput。

## 建议的调度器伪代码

```python
def choose_deployment(deployments):
    healthy = [d for d in deployments if now() >= d.next_available_at]
    healthy.sort(key=lambda d: (d.inflight, d.p95_latency_ms, d.last_429_at or 0))
    return healthy[0] if healthy else min(deployments, key=lambda d: d.next_available_at)

def on_response(deployment, response):
    headers = response.headers

    if response.status_code == 200:
        deployment.consecutive_429 = 0
        deployment.limit_requests = int(headers.get("x-ratelimit-limit-requests", deployment.configured_rpm))
        deployment.remaining_requests = int(headers.get("x-ratelimit-remaining-requests", 0))
        return

    if response.status_code == 429:
        deployment.consecutive_429 += 1
        retry_after = parse_retry_after(headers)

        if retry_after is not None:
            deployment.next_available_at = now() + retry_after + random_jitter(0.5, 2.0)
        else:
            delay = min(60, 2 ** min(deployment.consecutive_429, 6))
            deployment.next_available_at = now() + delay + random_jitter(0, delay * 0.2)

        if response.error_code == "EngineOverloaded":
            deployment.capacity_pressure_score += 1
```

## 可复用探针脚本

这个脚本不会打印 key。它会用 Azure CLI 读取 endpoint/key, 发 `capacity + 1` 请求, 然后记录 headers。

```python
import asyncio
import json
import subprocess
import time
import httpx

AZ = "az"
RG = "<resource-group>"
ACCOUNT = "<azure-openai-account>"
DEPLOYMENT = "gpt-image-2"
API_VERSION = "2025-04-01-preview"

def az_json(args):
    return json.loads(subprocess.check_output([AZ] + args, text=True))

def az_tsv(args):
    return subprocess.check_output([AZ] + args, text=True).strip()

def pick_headers(resp):
    names = [
        "retry-after", "retry-after-ms",
        "x-ratelimit-limit-requests", "x-ratelimit-remaining-requests", "x-ratelimit-reset-requests",
        "x-ms-region", "x-request-id", "apim-request-id",
    ]
    lower = {k.lower(): v for k, v in resp.headers.items()}
    return {k: lower.get(k) for k in names if lower.get(k) is not None}

async def one(client, url, headers, label):
    body = {
        "prompt": f"A tiny test shape on a white background. {label}",
        "n": 1,
        "size": "1024x1024",
        "quality": "low",
    }
    start = time.time()
    resp = await client.post(url, headers=headers, json=body, timeout=180)
    row = {
        "label": label,
        "elapsed_s": round(time.time() - start, 3),
        "status": resp.status_code,
        "headers": pick_headers(resp),
    }
    if resp.status_code != 200:
        try:
            row["error"] = resp.json().get("error", {})
        except Exception:
            row["body_prefix"] = resp.text[:200]
    print(json.dumps(row, ensure_ascii=False))

async def main():
    endpoint = az_json(["cognitiveservices", "account", "show", "-g", RG, "-n", ACCOUNT, "--query", "properties.endpoint", "-o", "json"]).rstrip("/")
    key = az_tsv(["cognitiveservices", "account", "keys", "list", "-g", RG, "-n", ACCOUNT, "--query", "key1", "-o", "tsv"])
    url = f"{endpoint}/openai/deployments/{DEPLOYMENT}/images/generations?api-version={API_VERSION}"
    headers = {"api-key": key, "Content-Type": "application/json"}

    async with httpx.AsyncClient(http2=False) as client:
        tasks = [one(client, url, headers, f"capacity-plus-one-{i+1}") for i in range(3)]
        await asyncio.gather(*tasks)

asyncio.run(main())
```

## 一句话建议

不要把 `RPM=10` 理解成“任意时刻可以并发 10 个”。对 gpt-image-2 这种低 RPM 图像模型, 应把它当作慢速队列服务: 每个 deployment 做全局平滑节流, 按 429 header 熔断, 对无 header 的容量型 429 做指数退避和跨 region fallback。这样比单纯堆重试更能提升用户体验。
