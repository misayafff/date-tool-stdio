# date-tool-stdio

一个基于 [MCP（Model Context Protocol）](https://modelcontextprotocol.io/) 的极简工具服务，通过 **stdio** 传输向大模型提供“获取今天日期”的能力。

## 功能

| 工具名 | 说明 | 返回值 |
| --- | --- | --- |
| `get_today` | 返回当前的年月日 | 字符串，格式 `YYYY-MM-DD`，例如 `2026-11-26` |

## 目录结构

```
mcp发布/
├── mcp_server_stdio.py   # MCP 服务端实现
├── requirements.txt      # 依赖清单
└── README.md             # 本说明文件
```

## 环境要求

- Python 3.10 及以上
- 依赖：`mcp>=1.28.1`

## 安装

```powershell
pip install -r requirements.txt
```

## 运行

该服务使用 stdio 传输，必须由 MCP 客户端拉起：

```powershell
python mcp_server_stdio.py
```

> 注意：**直接在终端运行不会有任何输出，也不会监听端口，这是 stdio 传输的正常表现。** 服务从标准输入读取 JSON-RPC 消息，终端没有输入时会静默退出。要验证它是否可用，请用下面的客户端方式连接。

## 客户端配置

### 方式一：MCP 客户端配置文件

以常见的 `mcpServers` 配置为例（Trae、Claude Desktop 等客户端通用）：

```json
{
  "mcpServers": {
    "date-tool-stdio": {
      "command": "python",
      "args": ["C:/path/to/mcp发布/mcp_server_stdio.py"]
    }
  }
}
```

如果 `python` 不在系统 `PATH` 中，请把 `command` 写成解释器绝对路径，例如 `D:/python/python.exe`。

### 方式二：LangChain 客户端

```python
import asyncio

from langchain_mcp_adapters.client import MultiServerMCPClient

client = MultiServerMCPClient(
    {
        "date-tool-stdio": {
            "transport": "stdio",
            "command": r"D:/python/python.exe",
            "args": [r"C:/path/to/mcp发布/mcp_server_stdio.py"],
        },
    }
)


async def main():
    tools = await client.get_tools()
    print([t.name for t in tools])          # ['get_today']
    print(await tools[0].ainvoke({}))       # 例如 '2026-11-26'


asyncio.run(main())
```

## 代码说明

```python
from datetime import datetime

from mcp.server.fastmcp import FastMCP

mcp = FastMCP("date-tool-stdio")

@mcp.tool()
def get_today() -> str:
    """返回当前的年月日，格式 YYYY-MM-DD。"""
    return datetime.now().strftime("%Y-%m-%d")

if __name__ == "__main__":
    mcp.run()
```

- `FastMCP("date-tool-stdio")`：创建服务实例，名称用于客户端识别。
- `@mcp.tool()`：把普通 Python 函数注册为 MCP 工具，函数的类型注解会转成工具参数结构，docstring 会作为工具描述提供给模型。
- `mcp.run()`：默认使用 stdio 传输启动服务。若需要常驻的 HTTP 服务，可改为 `mcp.run(transport="sse")` 或 `mcp.run(transport="streamable-http")`，并在 `FastMCP(...)` 中指定 `host` 与 `port`。

## 扩展新工具

新增工具只需再加一个被 `@mcp.tool()` 装饰的函数：

```python
@mcp.tool()
def get_time() -> str:
    """返回当前时间，格式 HH:MM:SS。"""
    return datetime.now().strftime("%H:%M:%S")
```

客户端重新拉取工具列表后即可调用。

## 常见问题

**运行时报 `ModuleNotFoundError: No module named 'mcp'`**

当前使用的解释器没有安装依赖，请确认安装到了同一个 Python 环境：

```powershell
python -m pip install -r requirements.txt
```

**客户端连接时报 `WinError 2`（系统找不到指定的文件）**

配置中的 `command` 或 `args` 路径不正确，或命令不在 `PATH` 中，请改用绝对路径。

**直接运行程序没有任何反应，也没有报错**

这是 stdio 服务的正常行为，参见上方“运行”一节的说明。
