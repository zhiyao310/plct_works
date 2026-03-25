# 使用 Ruyi 工具链在 LicheePi 4A 上编译并运行 llama.cpp

## 环境说明

- 硬件环境：Licheepi 4A 开发板（th1520）
- 软件环境：Debian/openEuler for RISC-V
- 源码仓库：https://github.com/ggml-org/llama.cpp

## 一、Ruyi环境搭建

#### 更新 Ruyi 索引并安装工具链

```bash
ruyi update
ruyi install gnu-plct-xthead
```

#### 创建并激活 Ruyi 虚拟环境

```bash
# 创建虚拟环境：指定工具链+目标开发板+环境名称
ruyi venv -t gnu-plct-xthead sipeed-lpi4a llama-env

# 进入虚拟环境目录
cd llama-env

# 激活虚拟环境
source ./bin/ruyi-activate
```

#### 验证GCC版本

```bash
riscv64-plctxthead-linux-gnu-gcc --version
make --version
```

## 二、获取 llama.cpp 源码并编译

```bash
# 克隆 llama.cpp 仓库
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp

# 配置环境变量，使用 Ruyi 提供的工具链
export CC=riscv64-plctxthead-linux-gnu-gcc
export CXX=riscv64-plctxthead-linux-gnu-g++

# 开始编译
# -j$(nproc) 表示使用所有 CPU 核心加速编译
make -j$(nproc)
```

```bash
# 编译命令：关闭RVV1.0，兼容TH1520 RVV0p7
make \
  CC=riscv64-plctxthead-linux-gnu-gcc \
  CXX=riscv64-plctxthead-linux-gnu-g++ \
  CFLAGS="-march=rv64gcxthead -mabi=lp64d -O3 -ffast-math" \
  CXXFLAGS="-march=rv64gcxthead -mabi=lp64d -O3 -ffast-math" \
  LLAMA_RVV=0 \
  -j$(nproc)
```

## **三、模型下载与运行测试**

#### 下载模型文件

```bash
# 创建模型目录
mkdir -p models

# 下载 Qwen2.5-1.5B-Instruct 的 Q4 量化模型 (约 1GB)
wget -O models/qwen-1.5b-q4.gguf "https://huggingface.co/Qwen/Qwen2.5-1.5B-Instruct-GGUF/resolve/main/qwen2.5-1b-instruct-q4_k_m.gguf"
```

#### 运行推理测试

```bash
# 运行推理测试
# -m: 指定模型路径
# -p: 提示词
# -n: 生成的最大 token 数
# -t: 使用线程数 (建议根据 TH1520 核心数调整，如 4)

./llama-cli -m models/qwen-1.5b-q4.gguf \
    -p "你好，请用一句话介绍 RISC-V 架构。" \
    -n 64 \
    -t 4
```

## 四、 返回上级目录并退出工具链虚拟环境

```bash
# 返回上级目录
cd ..

# 退出 Ruyi 虚拟环境
ruyi-deactivate
```
