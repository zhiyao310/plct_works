# 使用 Ruyi 工具链在 LicheePi 4A 上编译并运行 llama.cpp

## 环境说明

- 硬件环境：Licheepi 4A 开发板（th1520）
- 软件环境：Debian/openEuler for RISC-V
- 源码仓库：https://github.com/ggml-org/llama.cpp

## 一、Ruyi环境搭建

#### 更新 Ruyi 索引并安装工具链

```bash
sudo apt update && sudo apt install -y cmake
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
![1虚拟环境](https://github.com/zhiyao310/plct_works/blob/main/outcome_list/Licheepi4A/llama.cpp/images/1%E8%99%9A%E6%8B%9F%E7%8E%AF%E5%A2%83.png)

#### 验证GCC版本

```bash
riscv64-plctxthead-linux-gnu-gcc --version
make --version
```
![2版本测试](https://github.com/zhiyao310/plct_works/blob/main/outcome_list/Licheepi4A/llama.cpp/images/2%E7%89%88%E6%9C%AC%E6%B5%8B%E8%AF%95.png)

## 二、获取 llama.cpp 源码并编译

#### 获取官方 llama.cpp 源码

```bash
# 克隆 llama.cpp 仓库
git clone https://github.com/ggml-org/llama.cpp
cd llama.cpp
```
![3克隆](https://github.com/zhiyao310/plct_works/blob/main/outcome_list/Licheepi4A/llama.cpp/images/3%E5%85%8B%E9%9A%86.png)

#### CMake编译

```bash
# 创建编译目录
mkdir build && cd build

# CMake配置
cmake .. \
  -DCMAKE_C_COMPILER=riscv64-plctxthead-linux-gnu-gcc \
  -DCMAKE_CXX_COMPILER=riscv64-plctxthead-linux-gnu-g++ \
  -DCMAKE_SYSTEM_NAME=Linux \
  -DCMAKE_SYSTEM_PROCESSOR=riscv64 \
  -DCMAKE_BUILD_TYPE=Release \
  -DBUILD_SHARED_LIBS=OFF \
  -DGGML_RVV=OFF \
  -DGGML_RV_ZFH=OFF \
  -DGGML_RV_ZVFH=OFF \
  -DGGML_XTHEADVECTOR=OFF

# 编译
cmake --build . --config Release -j$(nproc)
```
![4编译](https://github.com/zhiyao310/plct_works/blob/main/outcome_list/Licheepi4A/llama.cpp/images/4%E7%BC%96%E8%AF%91.png)


## **三、模型下载与运行测试**

#### 下载适配 LicheePi 4A 的轻量中文模型

```bash
# 回到llama.cpp根目录
cd ..

# 创建模型目录
mkdir -p models

# 下载 Qwen2.5-0.5B 模型（约 400MB）
wget -O models/qwen-0.5b-q5.gguf \
  "https://hf-mirror.com/Qwen/Qwen2.5-0.5B-Instruct-GGUF/resolve/main/qwen2.5-0.5b-instruct-q5_k_m.gguf"
```

#### 运行推理测试

```bash
# 运行推理测试
# -m: 指定模型路径
# -p: 提示词
# -n: 生成的最大 token 数
# -t: 使用线程数 (建议根据 TH1520 核心数调整，如 4)

./build/bin/llama-cli \
    -m models/qwen-0.5b-q5.gguf \
    -p "Explain RISC-V briefly." \
    -n 32 \
    -t 2
```
![5运行测试](https://github.com/zhiyao310/plct_works/blob/main/outcome_list/Licheepi4A/llama.cpp/images/5%E8%BF%90%E8%A1%8C%E6%B5%8B%E8%AF%95.png)

#### 输出结果

```bash
Â«Ruyi llama-envÂ» debian@revyos-lpi4a:~/llama-env/llama.cpp./build/bin/llama-cli \ \
    -m models/qwen-0.5b-q5.gguf \
    -p "Explain RISC-V briefly." \
    -n 32 \
    -t 2

Loading model...  


â
 â
   â
    â

ââ ââ
ââ ââ  ââââ
            ââââ
                ââââ
                      ââââ
                              â
                               ââââ âââââ
                                          âââââ

ââ ââ â
       ââââ ââ ââ ââ â
                      ââââ    ââ    ââ ââ ââ ââ
ââ ââ âââ
         ââ ââ ââ ââ âââ
                        ââ ââ âââââ âââââ âââââ
                                    ââ    ââ
                                    ââ    ââ

build      : b8508-9f102a140
model      : qwen-0.5b-q5.gguf
modalities : text

available commands:
  /exit or Ctrl+C     stop or exit
  /regen              regenerate the last response
  /clear              clear the chat history
  /read               add a text file


> Explain RISC-V briefly.

RISC-V is a microarchitecture designed by the ARM Corporation to make embedded devices more energy-efficient. Unlike traditional x86 architecture, which uses a wide range

[ Prompt: 1.7 t/s | Generation: 1.3 t/s ]

> 
```

## 四、 返回上级目录并退出工具链虚拟环境

```bash
# 返回上级目录
cd ..

# 退出 Ruyi 虚拟环境
ruyi-deactivate
```
![6退出](https://github.com/zhiyao310/plct_works/blob/main/outcome_list/Licheepi4A/llama.cpp/images/6%E9%80%80%E5%87%BA.png)
