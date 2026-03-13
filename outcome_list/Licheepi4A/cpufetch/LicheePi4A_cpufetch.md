# 使用 cpufetch 查看 RISC-V CPU 信息(基于Licheepi4A)

### 方法一：通过 apt 包管理器安装（推荐）

RevyOS 官方在 `revyos-addons` 软件源中提供了 riscv64 架构的 cpufetch 预编译包，可直接通过 apt 安装，步骤如下：

#### 步骤 1：确认系统架构

在安装前需验证当前系统为 riscv64 架构，确保与 cpufetch 包的架构匹配。

```bash
uname -m
dpkg --print-architecture
```

![image-20260311184334622](./images/image-20260311184334622.png)

#### 步骤 2：更新软件源

更新系统软件源索引，确保能获取到最新的 cpufetch 包信息：

```Bash
sudo apt update
```

![image-20260311184951416](./images/image-20260311184951416.png)

#### 步骤 3：安装 cpufetch 包

通过 apt 安装 cpufetch，若系统已启用 `revyos-addons` 源，将自动从 ISRC 镜像拉取 riscv64 版本的包：

```Bash
sudo apt install cpufetch
```

![image-20260311185001843](./images/image-20260311185001843.png)

#### 步骤 4：运行

```Bash
cpufetch
```

终端将输出当前 CPU 的架构、型号、核心数、频率等关键信息，以及 RISC-V 风格的 Logo

![image-20260311185058013](./images/image-20260311185058013.png)

###  方法二：从 GitHub 源码编译安装

若软件源中暂未提供 cpufetch 包，或需自定义代码测试，可从上游仓库编译源码，步骤如下：

#### 步骤 1：安装编译依赖工具

编译 cpufetch 需依赖 git（代码克隆）、gcc（编译器）、make（构建工具），需先安装这些工具：

```bash
sudo apt update
sudo apt install -y git gcc make
```

- 验证依赖是否安装成功：

  ```bash
  gcc --version 
  make --version 
  ```

![image-20260311185248260](./images/image-20260311185248260.png)

![image-20260311185300195](./images/image-20260311185300195.png)

#### 步骤 2：克隆 cpufetch 源码仓库

从 GitHub 拉取 cpufetch 最新源码：

1. 在终端执行克隆命令，将源码仓库下载到本地：

   ```Bash
   git clone https://github.com/Dr-Noob/cpufetch.git
   ```

2. 进入源码目录，为后续编译做准备：

   ```Bash
   cd cpufetch
   ```

![image-20260311185725595](./images/image-20260311185725595.png)

#### 步骤 3：直接编译

cpufetch 的 Makefile 会自动检测系统架构（通过 `uname -m`），无需手动指定 riscv64/riscv32：

```bash
make
```

![image-20260311185919007](./images/image-20260311185919007.png)

- 编译过程中若出现报错（如 “找不到 stdio.h”）：

  大概率是缺少 libc 开发库，补充安装：

  ```bash
  sudo apt install -y libc6-dev
  make clean && make # 清理后重新编译
  ```

编译完成后，源码目录会生成 `cpufetch` 可执行文件。

![image-20260311185633004](./images/image-20260311185633004.png)

#### 步骤 4：运行本地编译版本

```Bash
./cpufetch
```

终端将输出当前 CPU 的架构、型号、核心数、频率等关键信息，以及 RISC-V 风格的 Logo。

![image-20260311185936720](./images/image-20260311185936720.png)

## 进阶用法

cpufetch 支持多种自定义参数，可满足个性化的使用需求，以下为常用进阶用法：

### 1. 自定义配色方案

通过 `--color` 参数修改 Logo 和文字的配色，支持预设配色和自定义 RGB 配色：

1. 使用预设配色（如模拟 ARM 官方默认颜色）：

   - 终端执行命令：

     ```Bash
     cpufetch --color arm
     ```

   - 效果：Logo 和文字将切换为 ARM 风格的配色。

     ![image-20260311190134367](./images/image-20260311190134367.png)

2. 自定义 RGB 配色（格式：`R,G,B:R,G,B:...`，前三个为 Logo 颜色，后两个为文字颜色）：

   - 终端执行命令（示例配色）：

     ```Bash
     cpufetch --color 239,90,45:210,200,200:0,0,0:100,200,45:0,200,200
     ```

   - 效果：Logo 和文字将按照指定的 RGB 数值显示颜色。

     ![image-20260311190146209](./images/image-20260311190146209.png)

### 2. 切换 Logo 显示风格

通过 `-s`/`--style` 参数切换 Logo 的显示风格，支持 `fancy`（默认）、`retro`（复古）、`legacy`（兼容）三种风格：

1. 终端执行复古风格切换命令：

   ```Bash
   cpufetch --style retro
   ```

2. 效果：终端中 CPU Logo 将以复古像素风格展示。

![image-20260311190156464](./images/image-20260311190156464.png)

### 3. 显示完整 CPU 名称

默认情况下，cpufetch 会缩写 CPU 名称，通过 `-F`/`--full-cpu-name` 参数可显示全称：

1. 终端执行命令：

   ```Bash
   cpufetch -F
   ```

2. 效果：终端输出的 CPU 名称字段将展示完整的型号名称，无缩写。

![image-20260311190207175](./images/image-20260311190207175.png)

