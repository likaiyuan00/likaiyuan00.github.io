---
title: mcp
date: 2025-12-23 15:12:07
tags: llm
categories: python
---
# MCP 工具使用文档
https://gofastmcp.com/getting-started/installation

## 工具示例

### 基础工具定义
```python
mcp = FastMCP("Demo 🚀")
Starlette()

@mcp.tool(name='加法')
def add(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b
#fastmcp run my_server.py:mcp --transport sse --port 8000
```

### JSON数据处理工具
```python
@mcp.tool(name='从公网URL下载JSON文件并提取产品信息')
def extract_product_info(url: str) -> list[dict[str, Any]] | dict[str, str]:
    """
    从公网URL下载JSON文件并提取产品信息

    Args:
        url (str): 包含JSON数据的公网URL地址

    Returns:
        list[dict]: 包含 product_name 和 product_image 的字典列表

    Example:
        extract_product_info("https://example.com/products.json")
    """
    try:
        response = requests.get(url, timeout=10)
        response.encoding = 'utf-8'
        response.raise_for_status()

        products_name = []
        products_image = []
        lines = response.text.splitlines()

        for line in lines:
            try:
                product_data = json.loads(line.strip())
                products_name.append(product_data["product_name"])
                products_image.append(product_data["product_image"])
            except (json.JSONDecodeError, KeyError) as e:
                print(f"处理失败: {e}，行内容: {line.strip()}")

        return products_name, products_image

    except requests.exceptions.RequestException as e:
        return {"error": f"文件下载失败: {str(e)}"}
    except Exception as e:
        return {"error": f"处理失败: {str(e)}"}
```

### 音频下载工具
```python
@mcp.tool(name='下载音频文件')
def download_audio(
    url: str, 
    save_path: str = '../audio/oss_new_2min.wav',
    retries: int = 3
) -> bool:
    """
    下载远程音频文件到本地
    
    :param url: 音频文件URL
    :param save_path: 本地保存路径（包含文件名）
    :param retries: 失败重试次数
    :return: 是否下载成功
    """
    try:
        save_dir = os.path.dirname(save_path)
        Path(save_dir).mkdir(parents=True, exist_ok=True)

        if not url.startswith(('http://', 'https://')):
            print(f"无效的URL格式: {url}")
            return False

        for attempt in range(retries):
            try:
                with requests.get(url, stream=True, timeout=10, verify=False) as response:
                    response.raise_for_status()
                    total_size = int(response.headers.get('content-length', 0))
                    
                    with open(save_path, 'wb') as f:
                        downloaded = 0
                        for chunk in response.iter_content(chunk_size=8192):
                            if chunk:
                                f.write(chunk)
                                downloaded += len(chunk)
                                if total_size > 0:
                                    print(f"\r下载进度: {downloaded/total_size*100:.1f}%", end='')
                    return True
            except Exception as e:
                print(f"尝试 {attempt+1}/{retries} 失败: {str(e)}")
                time.sleep(5)
        return False
```

## MySQL-MCP 配置指南

### 环境安装
https://uv.doczh.com/getting-started/features/
```bash
# 使用 uv 安装
curl -LsSf https://astral.sh/uv/install.sh | sh
uv python install 3.12
uv init mysql-mcp
cd mysql-mcp
uv add mysql-mcp-server

# 使用 pip 安装
python -m venv mysql-mcp
source mysql-mcp/bin/activate
pip install mysql-mcp-server
pip show mysql-mcp-server
```

### 服务配置
```json
{
    "mcpServers": {
        "mysql": {
            "isActive": true,
            "command": "uv",
            "args": [
                "--directory",
                "D:/pycharm/.venv/Lib/site-packages/mysql_mcp_server",
                "run",
                "mysql_mcp_server"
            ],
            "env": {
                "MYSQL_HOST": "1.1.1.1",
                "MYSQL_PORT": "31106",
                "MYSQL_USER": "root",
                "MYSQL_PASSWORD": "123456789",
                "MYSQL_DATABASE": "app_db"
            },
            "name": "mysql"
        }
    }
}
```

### SSE 转换配置
https://github.com/supercorp-ai/supergateway
```bash
# 安装依赖
yum install -y nodejs npm
npm install -g pm2

# 设置环境变量
export MYSQL_HOST="127.0.0.1"
export MYSQL_PORT="31106"
export MYSQL_USER="root"
export MYSQL_PASSWORD="123456789"
export MYSQL_DATABASE="app_db"

# 启动服务
pm2 start --name mcp-server npx -- -y supergateway \
    --port 8951 \
    --stdio "python /root/mysql-mcp/lib/python3.12/site-packages/mysql_mcp_server/server.py"
```

### SSE 客户端配置
```json
{
    "mcpServers": {
        "pm-mysql": {
            "url": "http://服务器IP:8951/sse",
            "type": "sse"
        }
    }
}
```

> **注意事项**：
> 1. Windows路径建议使用正斜杠 `/`
> 2. 生产环境建议将密码等敏感信息存储在环境变量中
> 3. PM2进程需要保持常驻，建议配置开机启动
> 4. SSE服务需要确保防火墙开放对应端口（8951）
