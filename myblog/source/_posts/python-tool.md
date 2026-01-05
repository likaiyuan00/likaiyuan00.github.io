---
title: python_tool
date: 2026-01-05 14:15:20
tags:
categories: python
---
# Python 多环境管理综合指南

## 1. 源码编译安装 Python

### 操作步骤

```bash
#!/bin/bash

# 用法：./install_python.sh <版本号> [安装目录]
# 示例：./install_python.sh 3.8.12 /opt/python3.8

set -euo pipefail

if [ $# -lt 1 ] || [ $# -gt 2 ]; then
    echo "错误：参数数量不正确"
    echo "用法：$0 <python_version> [安装目录]"
    echo "示例：$0 3.8.12 /opt/python3.8"
    exit 1
fi


START_TIME=$(date +"%Y.%m.%d %H:%M:%S")

VERSION="$1"
INSTALL_DIR="${2:-/usr/local/python${VERSION}}"
LOG_FILE="$PWD/python_${VERSION}_install.log"
PYTHON_URL="https://www.python.org/ftp/python/${VERSION}/Python-${VERSION}.tar.xz"
TAR_FILE="Python-${VERSION}.tar.xz"
SOURCE_DIR="Python-${VERSION}"


# 验证版本号格式
if [[ ! "$VERSION" =~ ^[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
    echo "错误：无效的版本号格式 '$VERSION'，格式应为 3.8.12" | tee -a "$LOG_FILE"
    exit 1
fi

# 日志初始化（带标准化时间戳）
echo "=== 开始安装 Python $VERSION 到 ${INSTALL_DIR} [${START_TIME}] ===" | tee -a "$LOG_FILE"
echo "完整日志记录到：$(pwd)/${LOG_FILE}" | tee -a "$LOG_FILE"

execute_step() {
    local step_name="$1"
    local command="$2"
    
    echo -e "\n[$(date +'%Y.%m.%d %H:%M:%S')] $step_name" | tee -a "$LOG_FILE"
    
    # echo "命令: $command" >> "$LOG_FILE"
    # eval "$command" >> "$LOG_FILE" 2>&1 
    echo "命令: $command" | tee -a "$LOG_FILE"
    eval "$command" 2>&1 | tee -a "$LOG_FILE"
    
    # 错误处理
    local status=$?
    if [ $status -ne 0 ]; then
        echo -e "\n[$(date +'%Y.%m.%d %H:%M:%S')] 错误：步骤 '$step_name' 失败，状态码 $status" | tee -a "$LOG_FILE"
        exit $status
    fi
}


install_build_deps() {
    if command -v apt-get >/dev/null 2>&1; then
        echo -e "\n[$(date +'%Y.%m.%d %H:%M:%S')] === 检测到APT包管理器 ===" | tee -a "$LOG_FILE"
        execute_step "更新软件源" "sudo apt update -y"
        execute_step "安装编译工具链" "sudo apt install -y build-essential checkinstall libreadline-gplv2-dev libncursesw5-dev libssl-dev libsqlite3-dev tk-dev libgdbm-dev libbz2-dev libffi-dev zlib1g-dev"
    
    elif command -v yum >/dev/null 2>&1; then
        echo -e "\n[$(date +'%Y.%m.%d %H:%M:%S')] === 检测到YUM包管理器 ===" | tee -a "$LOG_FILE"
        execute_step "安装开发工具组" "sudo yum groupinstall -y 'Development Tools'"
        execute_step "安装额外依赖" "sudo yum install -y python3-devel openssl-devel bzip2-devel libffi-devel sqlite-devel zlib-devel"
    
    else
        echo -e "\n[$(date +'%Y.%m.%d %H:%M:%S')] 错误：不支持的包管理器" | tee -a "$LOG_FILE"
        exit 1
    fi
}

install_build_deps
execute_step "创建安装目录" "mkdir -p '$INSTALL_DIR'"
execute_step "下载源码" "wget --no-check-certificate -O '$TAR_FILE' '$PYTHON_URL' || (rm -f '$TAR_FILE' && exit 1)"
execute_step "解压源码" "rm -rf '$SOURCE_DIR' && tar -xf '$TAR_FILE'"
cd "$SOURCE_DIR" || (echo "[$(date +'%Y.%m.%d %H:%M:%S')] 无法进入目录 $SOURCE_DIR" | tee -a "$LOG_FILE"; exit 1)

execute_step "配置" "./configure --prefix='$INSTALL_DIR' --enable-optimizations"
execute_step "编译" "make -j$(nproc)"
execute_step "安装" "make altinstall"

# 完成
echo -e "\n[$(date +'%Y.%m.%d %H:%M:%S')] === Python $VERSION 安装成功 ===" | tee -a "$LOG_FILE"
echo "安装路径: $INSTALL_DIR/bin/python${VERSION%.*}"
echo "请将以下内容添加到 ~/.bashrc 或 ~/.zshrc:"
echo "export PATH=\"$INSTALL_DIR/bin:\$PATH\""
exit 0
```

### 日志示例
```log
[2023.10.20 14:30:45] === 开始安装 Python 3.8.12 ===
[2023.10.20 14:30:46] 安装开发工具组
[2023.10.20 14:31:22] 下载源码...
[2023.10.20 14:35:12] 编译完成
```

### 1.1 使用安装脚本
```bash
# 赋予执行权限
chmod +x install_python.sh

# 执行安装（默认安装到 /usr/local/pythonX.X）
./install_python.sh 3.12.0

# 指定安装目录
./install_python.sh 3.12.0 /opt/python3.12

#创建
python3 -m venv vllm
#激活
source vllm/bin/activate
#导出依赖
pip freeze > requirements.txt
#退出
deactivate
```

### 1.2 脚本特性
- ✅ 自动检测包管理器（APT/YUM）
- ✅ 完整依赖安装
- ✅ 编译优化（--enable-optimizations）
- ✅ 全流程日志记录（python_3.12.0_install.log）
- ✅ 错误中断机制（set -euo pipefail）

### 1.3 环境变量配置
```bash
# 添加以下内容到 ~/.bashrc 或 ~/.zshrc
export PATH="/opt/python3.12/bin:$PATH"

# 立即生效
source ~/.bashrc
```

---

## 2. pyenv 版本管理

### 2.1 安装与配置
```bash
# 安装依赖
sudo yum groupinstall -y "Development Tools"
sudo yum install -y openssl-devel bzip2-devel readline-devel sqlite-devel libffi-devel xz-devel

# 安装 pyenv
git clone https://gitee.com/mirrors/pyenv.git ~/.pyenv

# 配置环境变量
echo 'export PATH="$HOME/.pyenv/bin:$PATH"' >> ~/.bashrc
echo 'eval "$(pyenv init --path)"' >> ~/.bashrc
echo 'eval "$(pyenv init -)"' >> ~/.bashrc
source ~/.bashrc

# 验证安装
pyenv --version
```

### 2.2 常用操作
```bash
# 查看可安装版本
pyenv install -l | grep -E '^\s*3\.[0-9]+\.[0-9]+$'

# 加速编译（使用多核）
export MAKE_OPTS="-j$(nproc)"

# 安装指定版本
pyenv install 3.12.1

# 版本管理
pyenv global 3.12.1    # 全局默认版本
pyenv local 3.12.1     # 当前目录使用指定版本
pyenv versions         # 查看已安装版本
```

---

## 3. Miniconda 环境管理

### 3.1 安装与初始化
```bash
# 下载安装包
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh

# 执行安装（按提示操作）
bash Miniconda3-latest-Linux-x86_64.sh

# 初始化
~/miniconda3/bin/conda init bash
source ~/.bashrc

# 禁用自动激活 base 环境
conda config --set auto_activate_base false
```

### 3.2 环境操作
```bash
# 创建环境
conda create -n demo python=3.12

# 激活环境
conda activate demo

# 安装包
conda install numpy pandas

# 退出环境
conda deactivate

# 删除环境
conda remove -n demo --all
```

---

## 4. uv 极速环境管理

### 4.1 安装与初始化
```bash
# 一键安装
curl -LsSf https://astral.sh/uv/install.sh | sh

# 配置环境变量
echo 'export PATH="$HOME/.cargo/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### 4.2 使用示例
```bash
# 项目级环境管理
uv python install 3.12     # 安装指定 Python 版本
uv init my-project         # 初始化项目
cd my-project
uv add requests numpy      # 添加依赖
uv sync                    # 同步环境

# 传统 venv 模式
uv venv --python=3.12 uv-demo
source uv-demo/bin/activate
uv pip install -r requirements.txt
```

---

## 5. 环境优先级配置建议

### 5.1 PATH 顺序配置
```bash
# ~/.bashrc 配置示例
export PATH="$HOME/.local/bin:$PATH"       # 用户级安装
export PATH="$HOME/.pyenv/bin:$PATH"       # pyenv
export PATH="$HOME/.cargo/bin:$PATH"       # uv
export PATH="$HOME/miniconda3/bin:$PATH"   # conda
export PATH="/opt/python3.12/bin:$PATH"    # 自定义安装
```

### 5.2 工具选择建议
| 场景                     | 推荐工具       | 优势                          |
|--------------------------|----------------|-------------------------------|
| 多版本 Python 共存       | pyenv          | 纯净版本管理                  |
| 科学计算环境             | Miniconda      | 复杂依赖管理                  |
| 快速项目环境搭建         | uv             | 极速依赖解析                  |
| 生产环境定制安装         | 源码编译       | 完全控制编译参数              |

---

## 6. 常见问题解决

### 6.1 编译安装失败
- 现象：`configure: error: no acceptable C compiler found in $PATH`  
  解决：确保已安装开发工具包  
  ```bash
  sudo apt install build-essential  # Debian/Ubuntu
  sudo yum groupinstall "Development Tools"  # RHEL/CentOS
  ```

### 6.2 pyenv 安装缓慢
- 镜像加速：使用国内镜像源下载 Python 源码  
  ```bash
  export PYTHON_BUILD_MIRROR_URL="https://npm.taobao.org/mirrors/python"
  pyenv install 3.12.1
  ```

### 6.3 环境冲突
- 现象：`which python` 显示非预期路径  
  解决：检查激活顺序，使用绝对路径  
  ```bash
  /opt/python3.12/bin/python3.12 -V
  ```

---

> 通过合理组合这些工具，可以实现：  
> - 开发环境快速重建（uv + pyenv）  
> - 生产环境精准控制（源码编译）  
> - 科学计算环境隔离（conda）  
> - 多版本兼容测试（pyenv global/local）
