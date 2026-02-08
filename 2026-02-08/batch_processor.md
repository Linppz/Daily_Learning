# "死不崩溃"的批量处理器笔记

## 核心问题

`asyncio.gather` 默认行为：一个任务失败 → 取消其他所有任务 → 整体崩溃，拿不到任何结果。

## 解决方案

```python
result = await asyncio.gather(*tasks, return_exceptions=True)
```

加上 `return_exceptions=True` 后，失败的任务不会炸掉整体，而是把 Exception 对象放进结果列表。

## 结果清洗

```python
for i, res in enumerate(result, 1):
    if isinstance(res, Exception):
        print(f"第{i}个任务失败了")
    else:
        # res 是正常结果，收集起来
```

## gather 为什么比 wait 更适合这里？

- `gather` 返回结果的**顺序和传入顺序一致**，第 N 个结果对应第 N 个任务
- `wait` 返回的是集合，顺序不保证，做数据清洗时不知道哪个结果对应哪个任务

## 实验对比

| 场景 | 表现 |
|------|------|
| 无 return_exceptions | 任务14失败 → 其他任务被取消 → CancelledError → 全崩 |
| 有 return_exceptions=True | 任务14失败 → 其他19个正常完成 → 最后报告"第14个失败了" |
