# LicheePi4A LED 闪烁功能测试文档

## 文档概述

1. **文档目的**：提供一份连贯、可落地、思路清晰的 LicheePi4A 开发板 LED 闪烁功能测试流程，覆盖环境准备、代码实现、问题排错、最终验证全环节，帮助使用者顺利实现 LED 闪烁效果，掌握嵌入式 Linux 下 LED 操作的核心思路。

2. **适用范围**：LicheePi4A 开发板（搭载 RevyOS 系统）、新手级嵌入式开发者，无需复杂底层驱动知识，仅需基础终端操作能力。

3. **核心目标**：通过两种方案（GPIO 导出方案 → 系统原生 LED 设备文件方案）实现板载 LED 周期性闪烁，解决过程中常见的「命令未找到」「权限不足」「LED 常亮无切换」等问题。

4. **文档约定**：所有终端命令均可直接复制执行，`debian` 为默认用户名，默认密码为 `debian`；代码修改均使用 `nano` 编辑器（新手友好）。

## 一、前置准备与测试环境

### 1.  硬件准备

| 硬件设备          | 规格要求                         | 备注                                                         |
| ----------------- | -------------------------------- | ------------------------------------------------------------ |
| LicheePi4A 开发板 | 正常刷写 RevyOS 系统，可正常开机 | 已插入系统启动盘（SD 卡/EMMC），避免无系统无法操作           |
| 供电设备          | 满足 5V/3A 输出能力              | 保证开发板稳定运行，避免供电不足导致终端断开                 |
| 连接方式          | 本地串口终端/SSH 远程连接        | SSH 连接需保证开发板与电脑在同一局域网，获取开发板 IP 后连接 |

### 2.  软件环境确认

1. 开发板开机，登录终端，确认系统正常运行，终端前缀显示正常（如 `«Ruyi venv-new» debian@revyos-lpi4a:~$`）。

2. 确认网络连通性（可选，用于安装依赖工具），执行 `ping www.baidu.com`，有数据包返回即网络正常。

3. 预安装核心工具（系统原生编译器、文本编辑器），执行以下命令：

   ```Bash
   # 更新系统软件源，安装 gcc（原生编译器）、make、nano（编辑器）
   sudo apt update && sudo apt install gcc make nano -y
   ```

4. 验证编译器是否可用，执行 `gcc --version`，若输出类似 `gcc (Debian 14.2.0-11revyos1) 14.2.0` 版本信息，说明环境准备完成。

## 二、测试流程

### 步骤 1：绕过系统自动控制，设置 LED 为「手动模式」

先通过终端命令手动控制 LED，验证是否能实现亮灭切换，命令可直接复制：

1. 进入 LED 设备目录（以 mmc0::为例，若无效换 mmc1::）：

   ```
   cd /sys/class/leds/mmc0::/
   ```

2. 查看当前 LED 触发方式（确认默认是自动模式）：

   ```
   cat trigger
   ```

   终端输出会包含 [mmc]（中括号标注当前生效的触发方式，说明是 SD 卡活动自动触发）。

3. 强制设置触发方式为「手动模式」（none），取消系统自动控制（需 sudo）：

   ```
   sudo echo none > trigger
   ```

4. 手动控制 LED 点亮（写入 0，适配低电平点亮，若无效换 1）：

   ```
   sudo echo 0 > brightness
   ```

5. 手动控制 LED 熄灭（写入 1，适配高电平熄灭，若无效换 0）：

   ```
   sudo echo 1 > brightness
   ```

### 步骤 2：进入任意可读写目录

进入任意可读写目录

```Bash
# 1. 新建目录（用于存放 RuyiSDK 示例代码，可选，直接在主目录操作可跳过此步）
mkdir -p ~/ruyi_led_example

# 2. 进入该目录（终端当前目录会切换到这个文件夹，后续所有操作都在此目录下进行）
cd ~/ruyi_led_example
```

创建并进入虚拟环境
参考链接：https://ruyisdk.org/docs/Package-Manager/installation

```bash
ruyi venv -t gnu-plct-xthead sipeed-lpi4a ./venv-new
. ~/licheepi/venv-new/bin/ruyi-activate
```

### 步骤 3：用 `nano` 编辑器保存代码文件

```Bash
nano led_blink_licheepi4a.c
```

1. 终端进入 nano 编辑界面后，**直接复制粘贴下述完整代码**（注意：SSH 连接时，右键粘贴即可；本地终端可按 `Ctrl+Shift+V`）。

   ```C
   #include <stdio.h>
   #include <stdlib.h>
   #include <unistd.h>
   #include <fcntl.h>
   #include <signal.h>
   #include <string.h>
   
   // 定义 LicheePi4A SD 卡指示灯路径（无需手动导出 GPIO）
   #define LED_BRIGHTNESS "/sys/class/leds/mmc1::/brightness"
   #define LED_TRIGGER "/sys/class/leds/mmc1::/trigger" // LED 触发方式路径
   // 定义 LED 亮灭值（已适配低电平点亮，若无效可互换）
   #define LED_ON "1"  // 低电平，点亮 LED
   #define LED_OFF "0" // 高电平，熄灭 LED
   #define LED_TRIGGER_NONE "none" // 手动模式，取消系统自动控制
   
   // 全局标志位，用于捕获 Ctrl+C 信号后正常退出循环
   volatile int keep_running = 1;
   
   // 信号处理函数：捕获 Ctrl+C（SIGINT），修改标志位终止循环
   void sigint_handler(int sig) {
       (void)sig; // 消除未使用参数警告
       keep_running = 0;
       printf("\n收到退出信号，准备退出程序...\n");
   }
   
   // 函数：向指定文件写入字符串（控制 LED 状态/触发方式）
   int write_to_file(const char *filename, const char *content) {
       int fd = open(filename, O_WRONLY); // 以只写模式打开文件
       if (fd < 0) {
           perror("打开文件失败");
           return -1;
       }
       ssize_t bytes_written = write(fd, content, strlen(content));
       if (bytes_written < 0) {
           perror("写入文件失败");
           close(fd);
           return -1;
       }
       close(fd);
       return 0;
   }
   
   // 主函数：LED 闪烁核心逻辑（先设置手动模式，再控制亮灭）
   int main(void) {
       // 1. 注册信号处理函数，捕获 Ctrl+C
       signal(SIGINT, sigint_handler);
   
       // 2. 第一步：设置 LED 为手动模式，取消系统自动控制（核心新增步骤）
       printf("正在设置 LED 为手动控制模式...\n");
       if (write_to_file(LED_TRIGGER, LED_TRIGGER_NONE) != 0) {
           printf("设置手动模式失败，请检查 LED 路径是否正确\n");
           return -1;
       }
   
       // 3. 验证 LED 设备文件是否可操作
       printf("正在验证 LED 设备文件...\n");
       if (write_to_file(LED_BRIGHTNESS, LED_OFF) != 0) {
           printf("LED 设备文件不可操作\n");
           return -1;
       }
   
       // 4. 循环控制 LED 闪烁（2 秒亮，2 秒灭，方便观察）
       printf("LED 开始闪烁，按 Ctrl+C 退出...\n");
       while (keep_running) {
           // 点亮 LED
           if (write_to_file(LED_BRIGHTNESS, LED_ON) != 0) {
               perror("点亮 LED 失败");
               break;
           }
           sleep(2); // 延时 2 秒，清晰观察点亮状态
   
           // 熄灭 LED
           if (write_to_file(LED_BRIGHTNESS, LED_OFF) != 0) {
               perror("熄灭 LED 失败");
               break;
           }
           sleep(2); // 延时 2 秒，清晰观察熄灭状态
       }
   
       // 5. 程序退出前，将 LED 熄灭并恢复默认触发模式（可选，优化体验）
       printf("正在熄灭 LED...\n");
       write_to_file(LED_BRIGHTNESS, LED_OFF);
       printf("正在恢复 LED 默认触发模式...\n");
       write_to_file(LED_TRIGGER, "mmc0");
   
       printf("程序正常退出\n");
           return 0;
   }
   ```

2. 粘贴完成后，**保存并退出 nano**：按 `Ctrl+O`（大写 O，代表保存），终端底部会提示文件名，直接按 `Enter` 确认；再按 `Ctrl+X` 退出编辑界面。

3. （可选）验证代码是否保存成功，用 `cat` 命令查看文件内容：

   ```bash
   cat led_blink_licheepi4a.c
   ```

   若终端输出完整的代码内容，说明保存成功。

### 步骤 4：编译（衔接之前的编译命令，当前目录已存在 .c 文件）

确保已激活 RuyiSDK 虚拟环境，终端执行编译命令：

```
gcc led_blink_licheepi4a.c -o led_blink_licheepi4a
ls
```

编译成功后，执行 `ls` 命令，会看到当前目录下有两个文件：`led_blink_licheepi4a.c`（源码文件）和 `led_blink_licheepi4a`（可执行文件）。

### 步骤 5：运行程序（当前目录下直接执行，无需额外路径）

```
sudo ./led_blink_licheepi4a
```

## 三、 预期结果与最终验证

1. **终端正常输出流程**：

   ```Plain Text
   正在设置 LED 为手动控制模式...
   正在初始化 LED 为熄灭状态...
   LED 开始闪烁（间隔 2 秒），按 Ctrl+C 退出...
   ```

2. **硬件效果**：LicheePi4A 板载对应 SD 卡指示灯（靠近 SD 卡槽）以 2 秒为周期，清晰交替点亮和熄灭。

3. **正常退出**：按 `Ctrl+C`，终端输出清理流程，LED 自动熄灭，恢复为系统默认的 SD 卡读写活动触发模式。

4. **若仍无闪烁（最后兜底）**：

   - 切换 LED 设备名（`mmc0::`→`mmc1::`→`mmc2::`），重新编译运行。

   - 互换 `LED_ON` 和 `LED_OFF` 的值（`#define LED_ON "1"`，`#define LED_OFF "0"`），重新编译运行。

## 四、常见问题汇总与排错指南（全流程覆盖）

| 问题现象                                          | 核心原因                                                     | 解决方案                                                     |
| ------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 终端提示「riscv64-xxx-gcc: command not found」    | RuyiSDK 工具链未安装/无对应二进制包                          | 放弃交叉编译，使用系统原生 `gcc` 编译器（已在前置准备中安装） |
| 终端提示「打开文件失败: Permission denied」       | 操作系统设备文件无 root 权限                                 | 运行程序时加 `sudo`，如 `sudo ./led_blink_leds`              |
| 终端提示「写入文件失败: Invalid argument」        | 1. GPIO 编号无效；2. LED 设备名错误；3. 恢复触发模式参数不匹配 | 1. 替换 GPIO 编号；2. 验证 LED 设备名（`ls /sys/class/leds/`）；3. 从 `cat trigger` 输出中复制有效参数 |
| 终端提示「写入文件失败: Device or resource busy」 | GPIO/LED 设备被系统其他进程占用                              | 1. 手动取消 GPIO 导出：`sudo sh -c "echo 37 > /sys/class/gpio/unexport"`；2. 重启开发板释放占用资源 |
| LED 常亮无切换，终端输出正常                      | 1. LED 被系统自动触发模式覆盖；2. 亮灭电平值写反             | 1. 代码中设置 `trigger` 为 `none`（手动模式）；2. 互换 `LED_ON` 和 `LED_OFF` 的值 |
| 手动 `sudo echo xxx > file` 提示权限拒绝          | bash 重定向 `>` 无 root 权限                                 | 使用 `sudo sh -c` 包裹整个命令，如 `sudo sh -c "echo none > /sys/class/leds/mmc1::/trigger"` |

## 五、测试总结与核心知识点回顾

### 1.  测试总结

1. 本次测试通过「GPIO 导出方案」和「系统原生 LED 设备文件方案」实现 LicheePi4A LED 闪烁，其中「系统原生 LED 设备文件方案」避开了 GPIO 编号无效、设备占用等问题，是更稳妥的终极方案。

2. 全流程核心步骤：「环境准备 → 代码编写与配置 → 编译 → 带权限运行 → 排错 → 正常退出与环境恢复」，适用于大多数嵌入式 Linux 硬件操作测试。

3. 测试成功的关键：① 获得 root 权限；② 取消系统自动控制，切换为手动模式；③ 匹配硬件的亮灭电平值；④ 选择有物理对应的设备文件。

### 2.  核心知识点回顾

1. 嵌入式 Linux 中，硬件设备通常以文件形式暴露在 `/sys/class/` 目录下，可通过文件读写实现硬件控制。

2. GPIO 操作核心流程：「导出 → 配置方向 → 控制电平 → 取消导出」，需注意 GPIO 编号的有效性。

3. 系统自动管理的设备（如 SD 卡指示灯），需先设置 `trigger` 为 `none` 进入手动模式，才能实现自定义控制。

4. 运行操作系统设备文件的程序时，需通过 `sudo` 获取 root 权限，避免权限不足导致操作失败。
