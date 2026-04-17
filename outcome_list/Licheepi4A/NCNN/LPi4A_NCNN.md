# 使用 Ruyi 工具链在 LicheePi 4A 上编译并运行 NCNN (NanoDet) 示例

## 环境说明

- 硬件环境：LicheePi 4A 开发板（TH1520）
- 软件环境：RevyOS for RISC-V（Debian 衍生）
- 源码仓库：https://github.com/Tencent/ncnn.git
- 官方文档：https://wiki.sipeed.com/hardware/en/lichee/th1520/lpi4a/8_application.html#Use-of-NCNN

## 一、Ruyi环境搭建

#### 更新系统并安装依赖

```bash
sudo apt update && sudo apt install -y git build-essential cmake gcc g++ pkg-config \
                    libprotobuf-dev protobuf-compiler libvulkan-dev vulkan-tools
ruyi install gnu-plct-xthead
```

#### 创建并激活 Ruyi 虚拟环境

```bash
# 创建虚拟环境：指定工具链+目标开发板+环境名称
ruyi venv -t gnu-plct-xthead sipeed-lpi4a ncnn-env
# 进入虚拟环境目录
cd ncnn-env
# 激活虚拟环境
source ./bin/ruyi-activate
```

#### 验证工具链版本

```bash
riscv64-plctxthead-linux-gnu-gcc --version
cmake --version
```

## 二、获取 NCNN 源码并配置编译

#### 获取 NCNN 官方源码

```bash
cd ~
git clone https://github.com/Tencent/ncnn.git
cd ncnn
git submodule update --init
```

#### CMake配置与编译

```bash
# 创建独立的编译目录
mkdir build-lpi4a && cd build-lpi4a
# CMake配置
# 说明：此处结合了 RevyOS 多架构路径兼容方案与 TH1520(C920) 特性优化：
# --sysroot=/ -B/-I/-L ：禁用 Ruyi 自带 sysroot，解决与 Debian 多架构目录的冲突。
# NCNN_RVV=ON         ：启用 RISC-V Vector 扩展指令集，提升 AI 推理性能。
# NCNN_SIMPLEOCV=ON   ：使用内置简易图像读写，避免在嵌入式端强依赖庞大完整的 OpenCV。
# NCNN_BUILD_EXAMPLES=ON：编译 examples 目录下的 NanoDet 示例程序。
cmake .. \
  -DCMAKE_C_COMPILER=riscv64-plctxthead-linux-gnu-gcc \
  -DCMAKE_CXX_COMPILER=riscv64-plctxthead-linux-gnu-g++ \
  -DCMAKE_C_FLAGS="--sysroot=/ -B/usr/lib/riscv64-linux-gnu -I/usr/include/riscv64-linux-gnu -L/usr/lib/riscv64-linux-gnu -Wl,-rpath-link=/usr/lib/riscv64-linux-gnu" \
  -DCMAKE_CXX_FLAGS="--sysroot=/ -B/usr/lib/riscv64-linux-gnu -I/usr/include/riscv64-linux-gnu -L/usr/lib/riscv64-linux-gnu -Wl,-rpath-link=/usr/lib/riscv64-linux-gnu" \
  -DCMAKE_BUILD_TYPE=Release \
  -DNCNN_OPENMP=OFF \
  -DNCNN_THREADS=ON \
  -DNCNN_RUNTIME_CPU=OFF \
  -DNCNN_RVV=ON \
  -DNCNN_SIMPLEOCV=ON \
  -DNCNN_BUILD_EXAMPLES=ON \
  -DNCNN_BUILD_BENCHMARK=ON
# 编译（根据核心数开启多线程加速）
cmake --build . -j$(nproc)
```

## 三、运行测试

#### 准备模型与测试图片

```bash
# 进入示例生成目录
cd examples
# 下载 NanoDet 官方 ncnn 模型文件
wget -O nanodet_m_ncnn_model.zip https://github.com/RangiLyu/nanodet/releases/download/v0.3.0/nanodet_m_ncnn_model.zip
unzip nanodet_m_ncnn_model.zip
# 准备一张测试图片（需用户提前通过 U盘/SCP 等方式传入板子，此处假设命名为 a.jpg）
# cp /path/to/your/test.jpg a.jpg
```

#### 执行 NanoDet 目标检测

```bash
./nanodet a.jpg 2>&1 | tee run.log
```

## 四、返回上级目录并退出工具链虚拟环境

```bash
# 返回虚拟环境根目录
cd ~/ncnn-env
# 退出 Ruyi 虚拟环境
ruyi-deactivate
```
