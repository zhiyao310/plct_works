# 使用 Ruyi 工具链在 LicheePi 4A 上编译并运行GStreamer

## 环境说明

- 硬件环境：Licheepi 4A 开发板（th1520）
- 软件环境：Debian/openEuler for RISC-V
- 官方文档（参考）：
  https://wiki.sipeed.com/hardware/zh/lichee/th1520/lpi4a/8_application.html

## 一、Ruyi环境搭建

#### 更新系统并安装编译工具与 GStreamer 开发包

```bash
# 1. 更新软件列表并安装基础编译工具
sudo apt update
sudo apt install -y cmake git build-essential pkg-config

# 2. 安装 GStreamer 1.0 核心库与开发文件（使用 apt，不需要用 Ruyi 编译它们）
sudo apt install -y libgstreamer1.0-dev \
                    gstreamer1.0-tools \
                    gstreamer1.0-plugins-base-apps \
                    gstreamer1.0-plugins-good \
                    gstreamer1.0-plugins-bad \
                    gstreamer1.0-libav
```

### 安装 Ruyi 工具链

```bash
# 安装 Ruyi（如已安装可跳过）
ruyi install gnu-plct-xthead
```

### 创建并激活 Ruyi 虚拟环境

```bash
# 创建虚拟环境：指定工具链 + 目标开发板 + 环境名称
ruyi venv -t gnu-plct-xthead sipeed-lpi4a gst-tutorial-env

# 进入虚拟环境目录
cd gst-tutorial-env

# 激活虚拟环境
source ./bin/ruyi-activate
```

#### 验证GCC版本

```bash
riscv64-plctxthead-linux-gnu-gcc --version
pkg-config --modversion gstreamer-1.0
```

## 二、获取 GStreamer  源码并编译

#### 获取官方 GStreamer  源码

```bash
# 克隆 GStreamer 文档仓库（里面包含 tutorials C 示例）
git clone https://gitlab.freedesktop.org/gstreamer/gst-docs.git
cd gst-docs/examples/tutorials
```

#### 使用 Ruyi GCC 编译 basic-tutorial-1

```bash
# 使用 Ruyi GCC 代替系统 gcc
riscv64-plctxthead-linux-gnu-gcc \
  basic-tutorial-1.c \
  -o basic-tutorial-1 \
  $(pkg-config --cflags --libs gstreamer-1.0)
```

## **三、运行 GStreamer示例 **

#### 运行 basic-tutorial-1

```bash
cd ~/gst-tutorial-env/gst-docs/examples/tutorials

# 运行并保存日志（不截图）
./basic-tutorial-1 2>&1 | tee run.log
```

#### 输出结果

```bash
«Ruyi gst-tutorial-env» debian@revyos-lpi4a:~/gst-tutorial-env/gst-docs/examples/tutorials$ ./basic-tutorial-1 2>&1 | tee run.log
GStreamer version: 1.22.0
...
(中间为 GStreamer 管线启动与运行日志)
...
«Ruyi gst-tutorial-env» debian@revyos-lpi4a:~/gst-tutorial-env/gst-docs/examples/tutorials$ 
```

## 四、 返回上级目录并退出工具链虚拟环境

```bash
# 返回上级目录
cd ..

# 退出 Ruyi 虚拟环境
ruyi-deactivate
```
