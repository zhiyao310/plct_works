# Ruyi SDK GLES/SDL 中 Allegro 迁移测试及核心问题总结

## 文档概述

本文档完整梳理基于 Ruyi SDK 环境，将 Allegro 5.2.9 图形库迁移至 LicheePi 4A（RevyOS/riscv64 架构）的全流程，明确**本次迁移最终失败**的核心结论，并从版本设计、交叉编译环境、操作逻辑、架构适配四个维度分析“问题始终无法解决”的底层原因，为后续基于 Ruyi SDK 的 GLES/SDL 图形渲染方案选型提供关键参考。

## 一、测试背景

### 1. 核心环境信息

| 环境维度     | 具体信息                                                     |
| ------------ | ------------------------------------------------------------ |
| 硬件平台     | LicheePi 4A                                                  |
| 系统版本     | RevyOS                                                       |
| 开发工具链   | Ruyi SDK                                                     |
| 目标图形库   | Allegro 5.2.9.0                                              |
| 核心测试目标 | 验证 Allegro 适配 riscv64+纯 FrameBuffer 场景的可行性，生成可运行的库文件 |

### 2. 前置背景补充

Allegro 5.2.9 为 2022 年发布版本，其设计核心面向「桌面 Linux（X11）+ x86/arm 架构」，官方测试矩阵未覆盖 riscv64 架构，且未针对“纯 FrameBuffer 嵌入式场景”做适配，这是本次迁移从底层就存在的核心风险。

## 二、测试进度梳理（按问题递进拆解）

### 阶段1：RuyiSDK 工具链文件

#### `toolchain/ruyi-toolchain.cmake`

安装 Allegro 编译依赖，执行命令：

```cmake
set(CMAKE_SYSTEM_NAME Linux)
set(CMAKE_SYSTEM_PROCESSOR riscv64)

set(CMAKE_C_COMPILER riscv64-plctxthead-linux-gnu-gcc)
set(CMAKE_CXX_COMPILER riscv64-plctxthead-linux-gnu-g++)

set(CMAKE_SYSROOT $ENV{RUYI_SYSROOT})

set(CMAKE_FIND_ROOT_PATH
    ${CMAKE_SYSROOT}
)

set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
```

### 阶段2：Allegro 本体编译（RuyiSDK 环境中）

#### 1. 初始依赖安装

安装 Allegro 编译依赖，执行命令：

```Bash
sudo apt install -y libgles2-mesa-dev libegl1-mesa-dev mesa-common-dev
```

#### 2. 激活 RuyiSDK

```bash
source ~/ruyi/venv/bin/activate
```

确认：

```bash
which riscv64-plctxthead-linux-gnu-gcc
```

#### 3. 获取 Allegro

```bash
git clone https://github.com/liballeg/allegro5.git
cd allegro5
```

#### 4. CMake 配置

```bash
cmake .. \
    -C ../cmake_override.cmake \
    -DCMAKE_SYSROOT=${RUYI_SYSROOT} \
    -DCMAKE_INSTALL_PREFIX=${RUYI_VENV} \
    -DWITH_AUDIO=OFF \
    -DWITH_VIDEO=OFF \
    -DWITH_PHYSFS=OFF \
    -DWITH_OPENAL=OFF \
    -DWITH_DISPLAY=ON \
    -DWITH_FONT=ON \
    -DWITH_TTF=ON \
    -DWITH_GL=ON \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_C_STANDARD=99 \
    -DALLEGRO_NO_OPTIMIZATIONS=OFF
```

##### 测试结果

- CMake 配置报错：`X11 not found. You may need to install X11 development libraries`。

![allegro1_cmake_error.png](https://github.com/zhiyao310/plct_works/blob/main/outcome_list/Licheepi4A/Allegro/images/allegro1_cmake_error.png)

##### 根因分析（版本设计硬伤）

Allegro 5.2.9 的 Display 模块 CMake 脚本存在设计缺陷：即使显式禁用 X11、启用 FrameBuffer，代码仍会强制执行 X11 检测，且未设置“检测失败则降级到 FrameBuffer”的逻辑——该缺陷无配置项可绕过，仅能通过安装 X11 开发库“妥协”，违背嵌入式纯 FrameBuffer 场景的核心需求。

### 阶段3：安装 X11 开发库，突破 CMake 配置卡点

#### 操作内容

1. 安装宿主系统 X11 开发库（冗余依赖，降低检测报错概率）：

   ```Bash
   sudo apt install -y libx11-dev libxext-dev libxrandr-dev libxcursor-dev libxi-dev libxinerama-dev x11proto-core-dev
   ```

2. 创建 `override.cmake` 文件，**强制覆盖CMake缓存变量**，绕过默认X11检测逻辑：

   ```CMake
   cat > ../cmake_override.cmake << 'EOF'
   # Force disable X11
   set(WANT_X11 OFF CACHE BOOL "Disable X11" FORCE)
   set(HAVE_X11 OFF CACHE BOOL "No X11" FORCE)
   set(X11_FOUND FALSE CACHE BOOL "X11 not found" FORCE)
   
   # Force enable FB
   set(WANT_FB ON CACHE BOOL "Enable FB" FORCE)
   set(HAVE_LINUX_FB_H ON CACHE BOOL "Have FB header" FORCE)
   
   # Force enable DRM
   set(WANT_DRM ON CACHE BOOL "Enable DRM" FORCE)
   
   # Set OpenGL ES
   set(WANT_OPENGLES ON CACHE BOOL "Enable OpenGL ES" FORCE)
   set(OPENGLES2_FOUND TRUE CACHE BOOL "OpenGL ES2 found" FORCE)
   set(OPENGLES2_LIBRARY "${CMAKE_SYSROOT}/usr/lib/libGLESv2.so" CACHE FILEPATH "OpenGL ES2 library")
   
   # Disable other X11 components
   set(WANT_X11_XF86VIDMODE OFF CACHE BOOL "" FORCE)
   set(WANT_X11_XINERAMA OFF CACHE BOOL "" FORCE)
   set(WANT_X11_XRANDR OFF CACHE BOOL "" FORCE)
   set(WANT_X11_XCURSOR OFF CACHE BOOL "" FORCE)
   set(WANT_X11_XINPUT OFF CACHE BOOL "" FORCE)
   EOF
   ```

3. 加载 override 文件执行 CMake 配置：

   ```Bash
   rm -rf *
   cmake .. \
       -C ../cmake_override.cmake \
       -DCMAKE_SYSROOT=${RUYI_SYSROOT} \
       -DCMAKE_INSTALL_PREFIX=${RUYI_VENV} \
       -DWITH_AUDIO=OFF \
       -DWITH_VIDEO=OFF \
       -DWITH_PHYSFS=OFF \
       -DWITH_OPENAL=OFF \
       -DWITH_DISPLAY=ON \
       -DWITH_FONT=ON \
       -DWITH_TTF=ON \
       -DWITH_GL=ON \
       -DCMAKE_BUILD_TYPE=Release \
       -DCMAKE_C_STANDARD=99 \
       -DALLEGRO_NO_OPTIMIZATIONS=OFF
   ```

##### 测试结果

- CMake 配置顺利完成，输出“Configuration summary”关键信息；

- 次要警告“Manually-specified variables were not used”为参数重复提示，不影响核心配置；

- 核心突破：X11检测报错彻底解决，编译流程可启动。

![allegro2_cmake_right.png](https://github.com/zhiyao310/plct_works/blob/main/outcome_list/Licheepi4A/Allegro/images/allegro2_cmake_right.png)
![allegro3_camke_finish.png](https://github.com/zhiyao310/plct_works/blob/main/outcome_list/Licheepi4A/Allegro/images/allegro3_camke_finish.png)

##### 根因分析

通过`override.cmake`的`FORCE`关键字强制覆盖CMake缓存变量，直接跳过Allegro默认的X11检测逻辑，是本次唯一能突破X11卡点的特殊手段，但仅解决“配置阶段”问题，未触及编译阶段的依赖适配核心。

### 阶段4：编译 & 安装

```bash
make -j$(nproc)
make install
```

##### 测试结果

1. 编译初期进展正常：

   - 先生成示例资源文件（`fire_2.ogg`、`health.png`等）；

   - 核心源文件（`config.c`、`convert.c`等）编译成功，进度推进至30%左右；

2. 编译突然终止：

   - 编译`shader.c`/`system.c`时触发致命错误：`fatal error: GLES2/gl2.h: No such file or directory`；

   - 连锁错误：因`gl2.h`缺失，`aintern_opengl.h`中依赖的`GLint`/`GLenum`/`GLubyte`等GLES2核心类型未定义，编译返回Error 1/Error 2；

   - 即使通过`override.cmake`指定了GLES2库路径，仍无法解决头文件查找问题。

![allegro4_make_finish1.png](https://github.com/zhiyao310/plct_works/blob/main/outcome_list/Licheepi4A/Allegro/images/allegro4_make_finish1.png)

##### 根因分析

1. 头文件路径适配缺失：

   Allegro 5.2.9的编译脚本为“宿主编译”设计，仅默认查找`/usr/include`等宿主路径，未自动添加交叉编译环境的`${CMAKE_SYSROOT}/usr/include`路径，导致编译器找不到Ruyi SDK中已存在的`GLES2/gl2.h`；

2. 版本架构适配不足：

   即使手动添加`-I${CMAKE_SYSROOT}/usr/include`编译参数，Allegro 5.2.9无riscv64架构支持的底层问题仍会导致后续隐性错误（如指令集不兼容、函数调用约定不匹配）

## 三、核心问题总结

### 1. 版本设计硬伤：Allegro 5.2.9 对嵌入式场景的天然不兼容

| 核心缺陷             | 具体表现                                                     |
| -------------------- | ------------------------------------------------------------ |
| X11 检测逻辑未解耦   | 常规参数无法禁用X11检测，仅能通过`override.cmake`强制覆盖    |
| GLES2 依赖路径硬编码 | 仅查找宿主`/usr/include`，不识别`${CMAKE_SYSROOT}`，导致`gl2.h`缺失 |
| 无 riscv64 架构支持  | 官方未适配该架构的指令集/调用约定，即使解决依赖问题，后续仍有隐性兼容性错误 |

### 2. 交叉编译环境瓶颈：Ruyi SDK 配套能力不足

- 依赖维度混淆：`override.cmake`仅指定GLES2**库路径**，未补充交叉编译环境的头文件包含路径，编译器仍从宿主路径查找`gl2.h`；

- 开发库配套缺失：Ruyi SDK的riscv64版本GLES2/EGL开发库要么缺失、要么版本不匹配，手动补充易出现“路径/架构不兼容”；

- 工具链适配缺失：Allegro无法自动识别Ruyi SDK的交叉编译路径规则，需手动添加`-I`/`-L`参数，但无法覆盖所有源文件的编译逻辑。

### 3. 操作层局限：特殊手段仅解决表面问题

- 突破X11卡点的`override.cmake`属于“非常规手段”，仅绕过配置检测，未修复Allegro对交叉编译的适配缺陷；

- 编译阶段的头文件问题无类似“强制覆盖”的解决方案，因涉及编译器的基础路径查找逻辑，无法通过CMake变量绕过。

## 四、后续优化建议

### 1. 放弃Allegro，转向适配性更强的方案

- 优先测试SDL2：Ruyi SDK原生支持SDL2，且SDL2对riscv64+FrameBuffer+GLES2场景的适配更完善，是嵌入式图形渲染的主流选择；

- 极简方案：直接使用EGL/GLES2原生接口开发，减少第三方库依赖，手动控制交叉编译路径参数。

### 2. 交叉编译依赖管理优化

- 编译前强制指定头文件/库路径：在CMake命令中添加`-DCMAKE_C_FLAGS="-I${CMAKE_SYSROOT}/usr/include"`和`-DCMAKE_LINKER_FLAGS="-L${CMAKE_SYSROOT}/usr/lib"`；

- 优先使用`ruyi install`安装目标平台依赖，避免手动拷贝文件导致的架构/路径不兼容。

### 3. 前置验证流程补充

在启动任何第三方库迁移前，先编译极简测试程序验证工具链能力：

1. FrameBuffer基础绘图程序：验证`/dev/fb0`操作与权限；

2. GLES2+EGL极简渲染程序：验证GLES2/EGL依赖和riscv64架构兼容性；

确认工具链基础能力完整后，再推进第三方库迁移。

## 五、关键参考价值

1. 嵌入式riscv64场景需优先选择“官方支持该架构”的图形库，避免使用桌面导向的旧版本库（如Allegro 5.2.9）；

2. CMake强制覆盖变量（`override.cmake`）可作为嵌入式迁移的临时手段，但无法解决底层的路径/架构适配问题；

3. 交叉编译中“头文件路径”比“库文件路径”更核心，需优先确保编译器能找到所有依赖头文件。
