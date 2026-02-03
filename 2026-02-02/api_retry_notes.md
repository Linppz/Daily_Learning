# API 重试机制学习笔记

## 一、核心概念

### 1.1 为什么需要重试机制？

调用外部 API（如 LLM API）时，可能遇到各种临时性错误：
- 网络波动（`ConnectionError`、`TimeoutError`）
- 服务器过载（`503 Service Unavailable`）
- 限流（`429 Too Many Requests`）

重试机制能自动处理这些临时错误，提高系统稳定性。

### 1.2 核心代码结构

```python
import logging
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    retry_if_exception,
    before_sleep_log
)

logger = logging.getLogger("llm_client")

def is_retryable_error(exception: Exception) -> bool:
    """判断错误是否值得重试"""
    status_code = getattr(exception, "status_code", None)

    if status_code is not None:
        if status_code == 429 or status_code >= 500:
            return True
        return False
    return True

api_retry = retry(
    wait=wait_exponential(multiplier=1, min=1, max=10),
    stop=stop_after_attempt(3),
    retry=retry_if_exception(is_retryable_error),
    before_sleep=before_sleep_log(logger, logging.WARNING)
)
```

---

## 二、重试策略详解

### 2.1 哪些错误应该重试？

| HTTP 状态码 | 是否重试 | 原因 |
|-------------|----------|------|
| `429` Rate Limit | ✅ 是 | 限流是暂时的，等一下就好 |
| `5xx` Server Error | ✅ 是 | 服务端问题，可能很快恢复 |
| `400` Bad Request | ❌ 否 | 请求格式错误，重试无用 |
| `401` Unauthorized | ❌ 否 | 认证失败，重试无用 |
| `403` Forbidden | ❌ 否 | 权限不足，重试无用 |
| 无 status_code（网络错误） | ✅ 是 | 网络波动通常是暂时的 |

**工业界铁律**：不要重试 4xx 错误（除了 429）！

### 2.2 指数退避（Exponential Backoff）

```python
wait=wait_exponential(multiplier=1, min=1, max=10)
```

等待时间按指数增长：

| 重试次数 | 等待时间 |
|----------|----------|
| 第 1 次重试前 | 1 秒 |
| 第 2 次重试前 | 2 秒 |
| 第 3 次重试前 | 4 秒 |
| 第 4 次重试前 | 8 秒 |
| 第 5 次重试前 | 10 秒（达到 max 上限） |

### 2.3 重试次数

```python
stop=stop_after_attempt(3)
```

**注意**：`stop_after_attempt(3)` 表示**总共尝试 3 次**，即：
- 1 次初始调用 + 2 次重试 = 3 次总调用

---

## 三、关键函数解析

### 3.1 `getattr(exception, "status_code", None)`

**功能**：安全地获取对象属性，不存在时返回默认值而非抛异常。

```python
class MyError(Exception):
    pass

err = MyError("error")

# ❌ 直接访问 - 抛 AttributeError
err.status_code  # AttributeError!

# ❌ 两参数 getattr - 也抛异常
getattr(err, "status_code")  # AttributeError!

# ✅ 三参数 getattr - 安全返回默认值
getattr(err, "status_code", None)  # 返回 None
```

**为什么需要它？**

不同来源的异常结构不同：
- `httpx`、`openai` 的异常有 `status_code`
- Python 原生的 `ConnectionError`、`TimeoutError` 没有

用 `getattr(..., None)` 可以统一处理。

### 3.2 `logging.getLogger("llm_client")`

**功能**：创建或获取一个命名的 logger 对象。

```python
logger = logging.getLogger("llm_client")
```

- `"llm_client"` 是自定义名称，用于标识日志来源
- 相同名称返回同一个 logger 实例（单例模式）
- 配合 `before_sleep_log` 在每次重试前输出日志

**日志输出示例**：

```
WARNING:llm_client:Retrying call_api in 1.0 seconds as it raised HTTPStatusError with status 429.
WARNING:llm_client:Retrying call_api in 2.0 seconds as it raised HTTPStatusError with status 500.
```

**没有 logger 的缺点**：
- 调试困难：不知道重试了几次
- 问题定位难：不知道是什么错误触发的重试
- 无法监控：无法统计重试频率

### 3.3 `retry_if_exception` 的作用

```python
# 写法 A：直接传函数（能工作，但不推荐）
retry=is_retryable_error

# 写法 B：用 retry_if_exception 包装（推荐）
retry=retry_if_exception(is_retryable_error)
```

**为什么推荐写法 B？**

可以与其他条件**组合使用**：

```python
from tenacity import retry_if_exception, retry_if_result

# 异常时重试 OR 返回值为 None 时重试
retry=retry_if_exception(is_retryable_error) | retry_if_result(lambda x: x is None)
```

---

## 四、进阶问题

### 4.1 重试风暴（Thundering Herd）

**场景**：100 个用户同时调用 API，服务器返回 503。

**问题**：
- 最坏情况：100 × 3 = 300 次调用
- 所有重试几乎同时发生
- 对已过载的服务器造成更大压力（雪崩效应）

**解决方案 —— 添加 Jitter（随机抖动）**：

```python
from tenacity import wait_exponential, wait_random

api_retry = retry(
    wait=wait_exponential(multiplier=1, min=1, max=10) + wait_random(0, 2),
    # 原本：1s, 2s, 4s
    # 加抖动后：1~3s, 2~4s, 4~6s（错开请求时间）
    ...
)
```

### 4.2 幂等性问题

**场景**：创建订单的请求实际成功了，但客户端没收到响应，触发重试。

**问题**：重复创建订单，用户被扣两次钱。

**解决方案 —— 幂等键（Idempotency Key）**：

```python
import uuid

def create_order(user_id, amount):
    idempotency_key = str(uuid.uuid4())  # 客户端生成唯一标识
    
    response = api.post("/orders", {
        "user_id": user_id,
        "amount": amount,
        "idempotency_key": idempotency_key
    })
    # 服务端逻辑：
    # if 数据库已存在这个 key:
    #     return 已存在的订单（不重复创建）
    # else:
    #     创建新订单
```

---

## 五、易错点总结

### 5.1 拼写错误

```python
# ❌ 错误
stop_after_attemp(3)

# ✅ 正确
stop_after_attempt(3)  # 注意最后的 t
```

### 5.2 比较运算符

```python
# ❌ 错误：漏掉了 500 本身
if status_code > 500:

# ✅ 正确
if status_code >= 500:
```

### 5.3 logger 命名规范

```python
# ⚠️ 不推荐（有空格）
logger = logging.getLogger("llm client")

# ✅ 推荐（用下划线）
logger = logging.getLogger("llm_client")
```

### 5.4 getattr 参数个数

```python
# ❌ 两参数 - 属性不存在时抛异常
getattr(exception, "status_code")

# ✅ 三参数 - 属性不存在时返回默认值
getattr(exception, "status_code", None)
```

### 5.5 重试次数理解

```python
stop=stop_after_attempt(3)
```

- **不是**重试 3 次
- **而是**总共尝试 3 次（1 次初始 + 2 次重试）

### 5.6 logging 不输出

```python
# ❌ 只创建 logger，但没有配置，日志不会输出
logger = logging.getLogger("llm_client")

# ✅ 需要配置 logging 才能看到输出
import logging
logging.basicConfig(level=logging.WARNING)
logger = logging.getLogger("llm_client")
```

---

## 六、完整示例代码

```python
import logging
import httpx
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    wait_random,
    retry_if_exception,
    before_sleep_log
)

# 配置 logging
logging.basicConfig(level=logging.WARNING)
logger = logging.getLogger("llm_client")

TIMEOUT = 5.0

def is_retryable_error(exception: Exception) -> bool:
    """判断错误是否值得重试"""
    # 处理超时错误
    if isinstance(exception, (httpx.TimeoutException, TimeoutError)):
        return True
    
    status_code = getattr(exception, "status_code", None)
    
    if status_code is None:
        # 网络错误，值得重试
        if isinstance(exception, httpx.RequestError):
            return True
        return False
    
    # 401/403 不重试
    if status_code in (401, 403):
        return False
    
    # 429 或 5xx 重试
    if status_code == 429 or status_code >= 500:
        return True
    
    return False

api_retry = retry(
    wait=wait_exponential(multiplier=1, min=1, max=10) + wait_random(0, 2),  # 加 jitter
    stop=stop_after_attempt(3),
    retry=retry_if_exception(is_retryable_error),
    before_sleep=before_sleep_log(logger, logging.WARNING)
)

@api_retry
def call_llm_api(prompt: str) -> str:
    """调用 LLM API，自动重试"""
    with httpx.Client(timeout=TIMEOUT) as client:
        response = client.post(
            "https://api.example.com/v1/chat",
            json={"prompt": prompt}
        )
        response.raise_for_status()
        return response.json()["content"]
```

---

## 七、知识点速查表

| 概念 | 说明 |
|------|------|
| 指数退避 | 等待时间按 2^n 增长，避免频繁请求 |
| Jitter | 随机抖动，错开并发重试时间 |
| 幂等键 | 唯一标识，防止重复操作 |
| `getattr(obj, attr, default)` | 安全获取属性，不存在返回默认值 |
| `stop_after_attempt(n)` | 总共尝试 n 次（非重试 n 次） |
| `retry_if_exception` | 包装判断函数，支持条件组合 |
| `before_sleep_log` | 重试前记录日志 |
