# asyncio.Semaphore 并发控制笔记

## 核心概念

- **Total Requests（总请求数）**：一共要发多少个请求，比如 20 个
- **Concurrency（并发数）**：同一时刻最多允许几个请求在跑，比如 6 个

## 关键原则

Semaphore **不要写在 Client 内部**，要写在**业务编排层**（调用方），这样不同场景可以传不同的并发数，灵活性更好。

## 代码结构

```python
async def singletask(client, task_id, prompt, sem):
    async with sem:  # 获取信号量，超过限制的会在这里等待
        result = await client.generate(prompt, GenerationConfig())

async def main():
    sem = asyncio.Semaphore(6)  # 在编排层创建，通过参数传入
    client = LLMFactory.get_client()

    box = []
    for i in range(1, 21):
        box.append(singletask(client, i, prompt, sem))

    await asyncio.gather(*box)  # gather 接收协程列表，并发执行
```

## 运行效果对比

| 场景 | 表现 |
|------|------|
| 无 Semaphore | 20 个请求瞬间全部发出，同时跑 |
| Semaphore(6) | 先跑 6 个，一个完成一个补上，始终最多 6 个在跑 |

## 踩过的坑

1. `asyncio.gather` 需要**协程对象**，不是字符串或 Message 对象
2. `prompt: list` 是类型注解，不是创建列表，要写 `prompt: list = []`
3. `asyncio.run(main())` 要加括号调用，不能传函数本身
4. `result = await client.generate(...)` —— await 在等号右边
5. `client.generate()` 的第一个参数是 `List[Message]`，不是字符串
