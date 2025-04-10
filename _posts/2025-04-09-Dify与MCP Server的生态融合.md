---
title: Dify与MCP Server的生态融合
categories: [技术工具]
tags: [dify]
date: 2025-04-09
media_subpath: '/posts/2025/04/09'
---


MCP ([Model Context Protocol](https://zhida.zhihu.com/search?content_id=255975296&content_type=Article&match_order=1&q=Model+Context+Protocol&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NDQzNjAyMTIsInEiOiJNb2RlbCBDb250ZXh0IFByb3RvY29sIiwiemhpZGFfc291cmNlIjoiZW50aXR5IiwiY29udGVudF9pZCI6MjU1OTc1Mjk2LCJjb250ZW50X3R5cGUiOiJBcnRpY2xlIiwibWF0Y2hfb3JkZXIiOjEsInpkX3Rva2VuIjpudWxsfQ.94mqxeT90ZnCTv0lwYX_Z9K1jzllhs0qIfILNo7aCDw&zhida_source=entity)) 是一个开放协议，用于标准化应用程序如何向 [LLM](https://zhida.zhihu.com/search?content_id=255975296&content_type=Article&match_order=1&q=LLM&zd_token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJ6aGlkYV9zZXJ2ZXIiLCJleHAiOjE3NDQzNjAyMTIsInEiOiJMTE0iLCJ6aGlkYV9zb3VyY2UiOiJlbnRpdHkiLCJjb250ZW50X2lkIjoyNTU5NzUyOTYsImNvbnRlbnRfdHlwZSI6IkFydGljbGUiLCJtYXRjaF9vcmRlciI6MSwiemRfdG9rZW4iOm51bGx9.GiCl-FFsTuR9ubhyX5Tyy9IUZ0L95y_eyO-5dXMOJVs&zhida_source=entity) 提供上下文。可以将 MCP 想象成 AI 应用程序的 USB-C 接口。就像 USB-C 为设备连接各种外设和配件提供标准化方式一样，MCP 为 AI 模型连接不同的数据源和工具提供了标准化的方式。

从本质上讲，MCP 遵循客户端-服务器架构，其中主机应用程序可以连接到多个服务器：

![1744187813827.png](2b257dae80e17b1b0ef9b4444aff57fa.png)

**MCP 主机**: 像 Claude Desktop、IDE 或 AI 工具等想要通过 MCP 访问数据的程序

**MCP 客户端**: 与服务器保持 1:1 连接的协议客户端

**MCP 服务器**: 通过标准化的模型上下文协议暴露特定功能的轻量级程序

**本地数据源**: MCP 服务器可以安全访问的计算机文件、数据库和服务

**远程服务**: MCP 服务器可以连接的通过互联网提供的外部系统（例如通过 API）

为了大家更好的理解这个概念，我会在本地搭建一个MCP服务器，然后通过Dify的SSE协议插件连接该服务。在Dify的Agent节点中配置支持MCP工具的策略，将MCP Proxy地址设置为本地服务的SSE端点（例如http://192.168.0.231:8066/sse）。最终由Dify调度DeepSeek大模型完成对MCP服务的调用与结果解析。

MCP服务器就用官网上天气预报的例子。可以获取天气预报和严重天气警报。

MCP服务端提供两个工具：get-alerts和get-forecast。

**一、效果演示**

为了大家更形象的理解，不从搭建讲起，从效果开始看。

咱们测试一下两个工具的能力。

**1.1 天气警报测试**

首先测试一下天气警报的工具。

问题：德克萨斯州有哪些活跃的天气警报？

可以看到dify调用了名为weather的MCP服务。

调用的具体工具是get\_alerts，这个工具需要一个参数，为每一个州的代码。

最后MCP Server返回天气预报接口的数据。

也许有人会问，干嘛多此一举呢？

好处就是我们可以用自然语言调用这些工具，可以看到本来工具的参数是美国各州的代码简写，咱们肯定记不住这么繁琐的数据，大模型可以直接识别我们的用途，自动给出这个参数，去调用MCP中的工具，返回我们想要的结果。

<img src="dca33bf2f74f7128dfa672982f52a46b.png" alt="截图" style="zoom:50%;" />

最后由dify中配置的大模型DeepSeek来汇总数据：

<img src="981f1371a811b9e710eb3f686a58df4c.png" alt="截图" style="zoom:50%;" />

**1.2 天气预报测试**

测试一下天气预报的工具。

问题：萨克拉门托的天气如何？

可以看到dify调用了名为weather的MCP服务。

调用的具体工具是get\_forecast，这个工具需要两个参数，为经纬度。

工具返回接口的数据。

<img src="8aced352b20ad241380d8b079e9533ae.png" alt="截图" style="zoom:50%;" />

大模型汇总的结果：

<img src="e91007afe81523863445b85ff543dd6e.png" alt="截图" style="zoom:50%;" />

**二、Dify环境准备**

**2.1 部署dify**

dify部署不是本文的重点，详情参考 [部署文档](https://blog.wsleepybear.cn/posts/dify%E9%83%A8%E7%BD%B2%E4%B8%8E%E5%8D%87%E7%BA%A7/)

<br/>

**2.2 下载dify所需的插件**

点击dify的工具，在工具中找到"通过SSE发现和调用MCP工具",并且安装

<img src="a9cb12c51cdfcaf84d3d342b584ba9d1.png" alt="截图" style="zoom:50%;" />

**三、 搭建服务端**

看到效果后咱们看看怎么在本地搭建一个MCP Server。我用的是python代码来搭建的MCP Server。

** 1.1 安装uv** 

UV命令通常指的是在Python项目管理中使用的uv工具的命令。uv是一个由Astral团队开发的现代化Python工具链，旨在提高Python项目的开发和管理效率。它提供了比传统工具如pip更快的包安装速度、快速创建虚拟环境、原生支持pyproject.toml等功能。

我这里是Linux环境，命令：

```sh
curl -LsSf https://astral.sh/uv/install.sh | sh
```

此脚本会自动下载并安装最新版 uv，完成后会将可执行文件添加到用户目录的 `.cargo/bin` 或 `.local/bin` 中。若提示找不到命令，需将安装路径加入 `PATH` 环境变量：

```sh
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc  # 或 ~/.zshrc
source ~/.bashrc
```

安装完成后，执行以下命令检查版本：

```sh
uv --version  # 应输出类似 uv 0.1.0 的信息
```

**1.2 创建项目**

我这里用vscode远程连接服务器。打开一个空的文件夹test。

<img src="b30e33d0e15102c89d6d08e38e85044e.png" alt="截图" style="zoom:50%;" />

**1.2.1 为项目创建一个新目录**

使用uv init weather来初始化一个名为 **weather** 的项目。

执行完命令后会看到test文件夹内多了一个weather的目录，自动生成了一些文件。

![截图](f0f597b8e87f7f4e1c1f95a2be20ee62.png)

** 1.2.2 创建虚拟环境并激活它** 

使用 `uv venv` 创建虚拟环境并激活。在Linux环境中，使用 `activate` 激活虚拟环境，命令如下：

```sh
source .venv/bin/activate
```

** 1.2.3 安装依赖** 

使用以下命令安装依赖：

```sh
uv add "mcp[cli]" httpx
```

** 1.2.4 建服务器文件 weather.py** 

制作一个MCP服务器的关键是利用MCP库（如Python的FastMCP）定义并注册工具函数，使大模型可以通过标准协议调用外部服务。

本示例为使用Python实现的天气信息查询服务。代码主要包含两个核心功能：获取天气预警信息和获取天气预报。它通过调用美国国家气象局(NWS)的API来获取数据，使用FastMCP框架提供服务接口。包含了错误处理和数据格式化功能，同时使用了异步编程来提高性能。`get_alerts` 函数可以获取指定州的天气预警，而 `get_forecast` 函数则可以根据经纬度获取详细的天气预报信息。

### 详细步骤

1. **选择开发语言和库**
使用Python作为开发语言，并安装FastMCP库。通过以下命令安装依赖，确保开发环境准备就绪：

```sh
pip install fastmcp
```

2. **初始化MCP服务器**
在代码中导入FastMCP并创建一个服务器实例。
3. **定义工具函数**
编写两个工具函数：`get_alerts`（获取天气警报）和 `get_forecast`（获取天气预报）。这些函数需要使用 `@mcp.tool()` 装饰器注册为MCP工具，并且是异步的。
4. **处理API请求**
使用 `httpx` 库异步调用外部API（如美国国家气象服务API）。例如：
   - 在 `get_alerts` 中，请求指定州的警报数据并解析JSON响应。
   - 在 `get_forecast` 中，根据经纬度请求预报数据并提取关键信息。
5. **运行MCP服务器**

```sh
mcp.run(transport='SSE')
```

** 完整代码** 

```
from typing import Any
import httpx
from mcp.server.fastmcp import FastMCP

# Initialize FastMCP server
mcp = FastMCP("weather")

# Constants
NWS_API_BASE = "https://api.weather.gov"
USER_AGENT = "weather-app/1.0"

async def make_nws_request(url: str) -> dict[str, Any] | None:
    """Make a request to the NWS API with proper error handling."""
    headers = {
        "User-Agent": USER_AGENT,
        "Accept": "application/geo+json"
    }
    async with httpx.AsyncClient() as client:
        try:
            response = await client.get(url, headers=headers, timeout=30.0)
            response.raise_for_status()
            return response.json()
        except Exception:
            return None

def format_alert(feature: dict) -> str:
    """Format an alert feature into a readable string."""
    props = feature["properties"]
    return f"""
Event: {props.get('event', 'Unknown')}
Area: {props.get('areaDesc', 'Unknown')}
Severity: {props.get('severity', 'Unknown')}
Description: {props.get('description', 'No description available')}
Instructions: {props.get('instruction', 'No specific instructions provided')}
"""

@mcp.tool()
async def get_alerts(state: str) -> str:
    """Get weather alerts for a US state.

    Args:
        state: Two-letter US state code (e.g. CA, NY)
    """
    url = f"{NWS_API_BASE}/alerts/active/area/{state}"
    data = await make_nws_request(url)

    if not data or "features" not in data:
        return "Unable to fetch alerts or no alerts found."

    if not data["features"]:
        return "No active alerts for this state."

    alerts = [format_alert(feature) for feature in data["features"]]
    return "\n---\n".join(alerts)

@mcp.tool()
async def get_forecast(latitude: float, longitude: float) -> str:
    """Get weather forecast for a location.

    Args:
        latitude: Latitude of the location
        longitude: Longitude of the location
    """
    # First get the forecast grid endpoint
    points_url = f"{NWS_API_BASE}/points/{latitude},{longitude}"
    points_data = await make_nws_request(points_url)

    if not points_data:
        return "Unable to fetch forecast data for this location."

    # Get the forecast URL from the points response
    forecast_url = points_data["properties"]["forecast"]
    forecast_data = await make_nws_request(forecast_url)

    if not forecast_data:
        return "Unable to fetch detailed forecast."

    # Format the periods into a readable forecast
    periods = forecast_data["properties"]["periods"]
    forecasts = []
    for period in periods[:5]:  # Only show next 5 periods
        forecast = f"""
{period['name']}:
Temperature: {period['temperature']}°{period['temperatureUnit']}
Wind: {period['windSpeed']} {period['windDirection']}
Forecast: {period['detailedForecast']}
"""
        forecasts.append(forecast)

    return "\n---\n".join(forecasts)

if __name__ == "__main__":
    # Initialize and run the server
    mcp.run(transport='sse')
```

**1.2.5 运行服务器**

运行uv run weather.py以确认一切正常工作。

<img src="4b221d5d67df6175ef774f669fae7a3b.png" alt="截图" style="zoom:50%;" />

**四、在dify中配置MCP Server**

** 4.1 配置MCP Server** 

打开dify，点击”通过SEE发现和调用MCP工具“。

<img src="46581a7cf73d39e62110b947038a1dd5.png" alt="截图" style="zoom:50%;" />

其中的路径可以替换为你自己的真实路径。如果dify 在docker内部可以写docker回路ip地址，或者其他局域网地址。

```json
{
  "weather": {
    "url": "http://127.0.0.1:8000/sse",
    "headers": {},
    "timeout": 60,
    "sse_read_timeout": 300
  }
}
```

**4.2 配置agent**

在dify中新建一个agent，并且配置agent

![截图](ab6985d53b97e524d1b9762319964703.png)

填入对应的提示词即可。

**五、测试**

测试：德克萨斯州有哪些活跃的天气警报？

<img src="dca33bf2f74f7128dfa672982f52a46b.png" alt="截图" style="zoom:50%;" />

最后由dify中配置的大模型DeepSeek来汇总数据：

<img src="981f1371a811b9e710eb3f686a58df4c.png" alt="截图" style="zoom:50%;" />

**六、流程分析**

![Image 22](https://pic1.zhimg.com/v2-ed981a122c83b3fc702dd8cf58011958_1440w.jpg)

客户端发起请求时，会调用工具函数（如 get\_alerts），并传递必要的参数（如州的代码TX）。

在 MCP 传输层，数据通过标准输入输出（stdio）进行传输，同时对消息进行序列化和反序列化处理，并维护通信协议以确保数据传输的稳定性。

天气服务器接收到请求后，会解析请求内容，执行对应的工具函数（get\_alerts），并准备响应数据以供后续处理。

最后，服务器与天气 API 进行交互，发送 HTTP 请求以获取天气数据，接收返回的天气信息，并对结果进行格式化处理，以便最终返回给客户端。