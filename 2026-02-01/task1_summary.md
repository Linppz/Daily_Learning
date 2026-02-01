# 子任务 1：项目初始化与配置管理

## 📁 项目结构 - Src Layout

### 最终目录结构

```
llm-client/
├── .env                  # 环境变量配置文件
├── .gitignore
├── pyproject.toml        # 项目配置（依赖、元数据）
├── uv.lock
├── src/                  # 源代码目录
│   ├── __init__.py
│   ├── core/             # 核心配置模块
│   │   ├── __init__.py
│   │   └── config.py     # 配置管理
│   └── llm/              # LLM 相关模块
│       ├── __init__.py
│       └── schemas.py    # 数据结构定义
├── tests/                # 测试目录
└── verify_task1.py       # 验证脚本
```

### 为什么使用 Src Layout？

1. **防止导入混乱**：强制通过包名导入，避免「开发时能跑，安装后跑不了」的问题
2. **隔离清晰**：源代码、测试、配置文件各司其职
3. **打包友好**：符合 Python 打包工具的最佳实践

### `__init__.py` 的作用

- 告诉 Python 解释器该目录是一个 **Python 包**
- 任何 `import` 该包时都会执行 `__init__.py`
- Python 3.3+ 虽然支持无 `__init__.py` 的 namespace package，但显式创建更安全

---

## ⚙️ 配置管理 - Pydantic Settings

### 核心代码：`src/core/config.py`

```python
from pydantic import Field, SecretStr
from pydantic_settings import BaseSettings, SettingsConfigDict
from typing import Literal

class Settings(BaseSettings):
    # 允许的模型厂商（枚举限制）
    LLM_PROVIDER: Literal["openai", "deepseek", "anthropic"] = "openai"
    
    # API Keys（安全字符串）
    OPENAI_API_KEY: SecretStr | None = None
    DEEPSEEK_API_KEY: SecretStr | None = None
    ANTHROPIC_API_KEY: SecretStr | None = None
    
    # 通用设置
    LLM_TIMEOUT: int = Field(default=30, description="Global timeout in seconds")
    LLM_MAX_RETRIES: int = Field(default=3, description="Max retry attempts")

    # 读取 .env 文件
    model_config = SettingsConfigDict(
        env_file=".env", 
        env_file_encoding="utf-8",
        extra="ignore"  # 忽略 .env 中未定义的变量
    )

# 单例模式
settings = Settings()
```

### 工作原理

```
┌─────────────────┐
│     .env 文件    │    LLM_PROVIDER=deepseek
│                 │    LLM_TIMEOUT=60
└────────┬────────┘
         │ 1. 读取文件，解析成键值对（全是字符串）
         ▼
┌─────────────────┐
│   环境变量字典   │    {"LLM_PROVIDER": "deepseek", "LLM_TIMEOUT": "60"}
└────────┬────────┘
         │ 2. 实例化 Settings() 时触发
         ▼
┌─────────────────┐
│  Pydantic 验证  │    遍历 Settings 类的字段定义
│                 │    按字段名去字典里找对应的值
│                 │    根据类型注解做转换和验证
└────────┬────────┘
         ▼
┌─────────────────┐
│  Settings 实例  │    settings.LLM_PROVIDER = "deepseek" (str)
│                 │    settings.LLM_TIMEOUT = 60 (int，已转换)
└─────────────────┘
```

**关键点：是「类定义驱动」的，Pydantic 根据你定义的字段去找环境变量，不是反过来。**

### 重要类型说明

| 类型 | 作用 | 示例 |
|------|------|------|
| `Literal["a", "b"]` | 限制值只能是指定的几个 | 传入其他值会抛出 `ValidationError` |
| `SecretStr` | 安全字符串，防止意外打印泄露 | 打印显示 `**********`，需要 `.get_secret_value()` 获取真实值 |
| `Field(default=...)` | 带元数据的默认值 | 可添加 description、ge、le 等约束 |

### SecretStr 使用示例

```python
print(settings.OPENAI_API_KEY)                      # 输出: **********
print(settings.OPENAI_API_KEY.get_secret_value())   # 输出: sk-proj-xxx...

# 如果 key 未设置，值是 None，不是 SecretStr
print(settings.DEEPSEEK_API_KEY)                    # 输出: None
```

---

## 📦 数据结构 - Pydantic Schemas

### 核心代码：`src/llm/schemas.py`

```python
from pydantic import BaseModel, Field
from typing import Optional, Dict, Any
from enum import Enum

class Role(str, Enum):
    SYSTEM = "system"
    USER = "user"
    ASSISTANT = "assistant"

class Message(BaseModel):
    role: Role
    content: str

class TokenUsage(BaseModel):
    prompt_tokens: int = 0
    completion_tokens: int = 0
    total_tokens: int = 0

class LLMResponse(BaseModel):
    content: str
    raw_response: Dict[str, Any] = {}
    usage: TokenUsage
    model_name: str
    finish_reason: Optional[str] = None

class GenerationConfig(BaseModel):
    temperature: float = Field(default=0.7, ge=0.0, le=2.0)
    max_tokens: Optional[int] = Field(default=1000)
    top_p: Optional[float] = 1.0
```

### Pydantic 自动类型转换

Pydantic 会自动转换兼容的类型：

```python
# 字典 → Pydantic Model ✅ 自动转换
response = LLMResponse(
    content="Hello",
    usage={"total_tokens": 100},  # 自动转成 TokenUsage 对象
    model_name="gpt-4"
)

# 字符串 → int ✅ 自动转换
# .env 里 LLM_TIMEOUT=60 (字符串) → settings.LLM_TIMEOUT = 60 (int)

# 不兼容类型 ❌ 报错
# .env 里 LLM_TIMEOUT=abc → ValidationError
```

---

## 🔧 环境变量配置 `.env`

```ini
LLM_PROVIDER=openai
OPENAI_API_KEY=sk-proj-your-key-here
LLM_TIMEOUT=30
```

**注意事项：**
- `.env` 文件不要提交到 Git（已在 `.gitignore` 中）
- 所有值都是字符串，类型转换由 Pydantic 处理
- 未定义的变量会被忽略（因为设置了 `extra="ignore"`）

---

## ✅ 验证脚本

```python
# verify_task1.py
from src.core.config import settings
from src.llm.schemas import LLMResponse, TokenUsage

def main():
    print("--- 1. 验证配置加载 ---")
    print(f"当前 Provider: {settings.LLM_PROVIDER}")
    print(f"OpenAI Key (Safe): {settings.OPENAI_API_KEY}")
    
    print("\n--- 2. 验证 DTO 结构 ---")
    res = LLMResponse(
        content="测试成功",
        usage=TokenUsage(total_tokens=10),
        model_name="gpt-4o"
    )
    print(f"响应内容: {res.content}")
    print("✅ 子任务 1 全部完成！")

if __name__ == "__main__":
    main()
```

运行命令：`uv run verify_task1.py`

---

## 📚 核心概念速记

1. **Src Layout** = 源代码放 `src/` 下，防止导入混乱
2. **`__init__.py`** = 标记目录为 Python 包
3. **Pydantic Settings** = 类定义驱动，自动从 `.env` 读取并转换类型
4. **SecretStr** = 防呆设计，默认隐藏值，需显式调用才能获取
5. **Literal** = 枚举限制，值不在列表中会报错
6. **自动类型转换** = 字典→Model、字符串→int 等，不兼容则报错
