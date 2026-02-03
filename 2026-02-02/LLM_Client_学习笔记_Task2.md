# 🧠 LLM Client 开发学习笔记 — Task 2：抽象基类

> 作者：Yi  
> 日期：2026-02-02  
> 阶段：Task 2 - 定义抽象层 (BaseLLM)

---

## 📚 本节核心知识点

### 1. 为什么用 `messages: List[Message]` 而不是 `prompt: str`

现代 LLM API（GPT-4、Claude 等）都基于 **Chat Completion** 接口，需要传递消息列表。

**三个关键优势：**

| 特性 | `List[Message]` | `str` |
|------|-----------------|-------|
| 多轮对话 | ✅ 天然支持上下文记忆 | ❌ 需要手动拼接 |
| System Prompt | ✅ `role="system"` 设定 AI 人设 | ❌ 无法实现 |
| 角色区分 | ✅ 明确标记 user/assistant | ❌ 混乱不清 |

**示例：**
```python
messages = [
    Message(role="system", content="你是一个 Python 专家"),
    Message(role="user", content="什么是装饰器？"),
    Message(role="assistant", content="装饰器是..."),
    Message(role="user", content="给我一个例子")  # AI 能理解这是在问装饰器的例子
]
```

---

### 2. 为什么要独立传递 `GenerationConfig`

**核心思想：解耦 = 调用方和配置方分离**

不是为了"方便"，而是为了**预定义 + 运行时切换**：

```python
# 预定义多套配置
creative_config = GenerationConfig(temperature=0.9, top_p=0.95)
coding_config = GenerationConfig(temperature=0.2, top_p=0.1)
strict_config = GenerationConfig(temperature=0.0, top_p=0.1)

# 运行时根据场景切换，不用改任何调用代码
await client.generate(messages, creative_config)   # 写小说
await client.generate(messages, coding_config)     # 写代码
await client.generate(messages, strict_config)     # 做数学题
```

**好处：**
- 改配置不用改调用代码
- 改调用逻辑不用管配置细节
- 配置可复用、可版本管理

---

### 3. 抽象基类 (ABC) 的作用

**抽象基类 = 契约 = 规范**

```python
from abc import ABC, abstractmethod

class BaseLLM(ABC):
    @abstractmethod
    async def generate(self, messages, config) -> LLMResponse:
        pass

    @abstractmethod
    async def stream(self, messages, config) -> AsyncIterator[str]:
        pass
```

**为什么直接实例化会报错是"成功"的标志？**

```bash
>>> BaseLLM()
TypeError: Can't instantiate abstract class BaseLLM with abstract methods generate, stream
```

这是 **feature，不是 bug**！

**好处：**
1. **机器强制执行规范** —— 比文档靠谱一万倍
2. **防止遗漏实现** —— 如果子类忘了实现 `stream()`，代码直接跑不起来
3. **团队协作保障** —— 所有人必须按同一个接口写代码

---

### 4. AsyncIterator 与流式输出

`AsyncIterator[str]` 用于实现"打字机效果"：

```python
async def stream(self, messages, config) -> AsyncIterator[str]:
    full_text = "Hello World"
    for char in full_text:
        await asyncio.sleep(0.1)  # 模拟网络延迟
        yield char                 # 逐字输出
```

**使用方式：**
```python
async for chunk in client.stream(messages, config):
    print(chunk, end="", flush=True)
```

---

## ⚠️ 易错点总结

### 错误 1：子类使用 `@abstractmethod`

```python
# ❌ 错误
class MockLLM(BaseLLM):
    @abstractmethod  # 子类不应该有这个！
    async def generate(self, ...):
        pass

# ✅ 正确
class MockLLM(BaseLLM):
    async def generate(self, ...):  # 直接实现，不加装饰器
        return LLMResponse(...)
```

**记住：**
> 基类用 `@abstractmethod` **声明**规范  
> 子类直接**实现**方法体，不加任何装饰器

---

### 错误 2：Pydantic/Dataclass 实例化方式

```python
# ❌ 错误 - 这只是拿到类本身
obj = LLMResponse
obj.content = "..."  # 修改的是类属性！

# ✅ 正确 - 调用构造函数创建实例
obj = LLMResponse(
    content="This is a mock response.",
    model="mock",
    usage={"prompt_tokens": 0, "completion_tokens": 0}
)
```

---

### 错误 3：忘记导入必要模块

```python
# ❌ 用了 asyncio.sleep 但没导入
await asyncio.sleep(0.1)  # NameError!

# ✅ 记得导入
import asyncio
```

---

## 📝 完整代码示例：MockLLM

```python
import asyncio
from src.llm.base import BaseLLM
from src.llm.schemas import LLMResponse, Message, GenerationConfig
from typing import List, AsyncIterator


class MockLLM(BaseLLM):
    """用于测试的模拟 LLM 客户端"""

    async def generate(
        self, 
        messages: List[Message], 
        config: GenerationConfig
    ) -> LLMResponse:
        """非流式：直接返回固定响应"""
        return LLMResponse(
            content="This is a mock response.",
            model="mock",
            usage={"prompt_tokens": 0, "completion_tokens": 0}
        )

    async def stream(
        self, 
        messages: List[Message], 
        config: GenerationConfig
    ) -> AsyncIterator[str]:
        """流式：逐字输出"""
        full_text = "Hello World"
        for char in full_text:
            await asyncio.sleep(0.1)
            yield char
```

---

## 🎯 本节关键收获

| 概念 | 一句话总结 |
|------|-----------|
| `List[Message]` | 支持多轮对话 + System Prompt + 角色区分 |
| `GenerationConfig` | 解耦 = 预定义配置 + 运行时切换 |
| 抽象基类 (ABC) | 机器强制执行的接口契约 |
| `@abstractmethod` | 只在基类用，子类直接实现 |
| `AsyncIterator` | 流式输出的核心，配合 `yield` 使用 |

---

## 🚀 下一步

Task 3：实现具体的 LLM Client（OpenAI / DeepSeek）

将 `BaseLLM` 这个"插座"接上真正的"电器"！
