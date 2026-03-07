# LicheePi4A LED Blink测试文档

## 文档概述

1. **文档目的**：提供一份连贯、可落地、思路清晰的 LicheePi4A 开发板 LED  Blink测试流程，覆盖环境准备、代码实现、问题排错、最终验证全环节，帮助使用者顺利实现 LED 闪烁效果，掌握嵌入式 Linux 下 LED 操作的核心思路。

2. **适用范围**：LicheePi4A 开发板（搭载 RevyOS 系统）、新手级嵌入式开发者，无需复杂底层驱动知识，仅需基础终端操作能力。

3. **核心目标**：验证通过C语言调用libgpiod库控制GPIO引脚实现LED闪烁的功能完整性、流程可行性。

4. **文档约定**：所有终端命令均可直接复制执行，`debian` 为默认用户名，默认密码为 `debian`；代码修改均使用 `nano` 编辑器（新手友好）。

## 一、测试环境

### 1.1 硬件环境

|硬件组件|规格/型号|备注|
|---|---|---|
|开发板|LicheePi4A|已刷Debian/revyos系统|
|LED灯|3V-3.2V 直插LED|长脚=正极，短脚=负极|
|杜邦线|公对公杜邦线 2-3根|用于GPIO/LED/GND连接|
|电阻（可选）|1kΩ/220Ω 贴片/直插电阻|限流用，无电阻也可测试|
### 1.2 软件环境

|软件组件|版本/要求|备注|
|---|---|---|
|操作系统|Debian 11/12 或 revyos|64位（riscv64架构）|
|依赖库|libgpiod-dev ≥ 1.6.3|GPIO硬件控制核心库|
|编译器|gcc ≥ 8.0|编译C语言代码|
|权限|拥有root/sudo权限|GPIO操作需管理员权限|
## 二、测试准备

### 2.1 硬件接线

按以下方式完成硬件连接，确保回路闭合：

```Plain Text
LicheePi4A GPIO1_3引脚 → LED长脚（正极） → LED短脚（负极） → LicheePi4A GND引脚
```

- 可选：在GPIO1_3与LED正极之间串联1kΩ电阻（限流保护，无电阻不影响测试）；
- 确认：杜邦线两端完全插入开发板排针和LED引脚，无松动。

![LED_blink_1](./images/LED_blink_1.jpg)

### 2.2 依赖环境确认

执行以下命令确认libgpiod-dev已安装（若未安装，先执行安装命令）：

```Bash
# 检查libgpiod-dev是否安装
dpkg -l | grep libgpiod-dev

# 若未安装，执行以下命令安装
sudo apt update && sudo apt install -y libgpiod-dev
```

![image-20260306220351607](./images/image-20260306220351607.png)

## 三、详细测试步骤

### 步骤1：创建并进入测试目录

进入任意可读写目录

```Bash
# 1. 新建目录（用于存放 RuyiSDK 示例代码，可选，直接在主目录操作可跳过此步）
mkdir -p ~/ruyi_led_example

# 2. 进入该目录（终端当前目录会切换到这个文件夹，后续所有操作都在此目录下进行）
cd ~/ruyi_led_example
```

### 步骤2：编写LED blink代码

```Bash
nano gpio_blink.c
```

1. 终端进入 nano 编辑界面后，**直接复制粘贴下述完整代码**（注意：SSH 连接时，右键粘贴即可；本地终端可按 `Ctrl+Shift+V`）。

   ```C
   #include <stdio.h>
   #include <stdlib.h>
   #include <unistd.h>
   #include <gpiod.h>
   
   int main()
   {
       int i;
       int ret;
   
       struct gpiod_chip * chip;
       struct gpiod_line * line;
   
       chip = gpiod_chip_open("/dev/gpiochip1");
       if(chip == NULL)
       {
           printf("gpiod_chip_open error\n");
           return -1;
       }
   
       line = gpiod_chip_get_line(chip, 3);
       if(line == NULL)
       {
           printf("gpiod_chip_get_line error\n");
           gpiod_line_release(line);
       }
   
       ret = gpiod_line_request_output(line, "gpio", 0);
       if(ret < 0)
       {
           printf("gpiod_line_request_output error\n");
           gpiod_chip_close(chip);
       }
   
       printf("Starting LED blink on GPIO1_3 (num 427), press Ctrl+C to stop.\n");
       for(i = 0; i < 10; i++) 
       {
           gpiod_line_set_value(line, 1);
           printf("ON\n");
           sleep(1);
           gpiod_line_set_value(line, 0);
           printf("OFF\n");
           sleep(1);
       }
   
       gpiod_line_release(line);
       gpiod_chip_close(chip);
   
       return 0;
   }
   ```

2. 粘贴完成后，**保存并退出 nano**：按 `Ctrl+O`（大写 O，代表保存），终端底部会提示文件名，直接按 `Enter` 确认；再按 `Ctrl+X` 退出编辑界面。

3. （可选）验证代码是否保存成功，用 `cat` 命令查看文件内容：

   ```bash
   cat gpio_blink.c
   ```

   若终端输出完整的代码内容，说明保存成功。

![image-20260306220500680](./images/image-20260306220500680.png)

### 步骤3：编译

执行编译命令，链接libgpiod库生成可执行文件：

```Bash
gcc gpio_blink.c -I /usr/include/ -L /usr/lib/riscv64-linux-gnu/ -lgpiod -o gpio_blink
ls
```

编译成功后，执行 `ls` 命令，会看到当前目录下有两个文件：`gpio_blink`（源码文件）和 `gpio_blink`（可执行文件）。

![image-20260306220515661](./images/image-20260306220515661.png)

### 步骤4：运行程序

以root权限执行编译后的程序（GPIO操作需管理员权限）：

```Bash
sudo ./gpio_blink
```

![image-20260306220537095](./images/image-20260306220537095.png)

## 四、 预期结果与最终验证

1. **终端正常输出流程**：

   ```Plain Text
   Starting LED blink on GPIO1_3 (num 427), press Ctrl+C to stop.
   ON
   OFF
   ON
   ...
   ```

2. **硬件效果**：连接GPIO1_3的LED灯与终端输出同步，亮1秒、灭1秒，共闪烁10次后熄灭。

3. **正常退出**：按 `Ctrl+C`，终端输出清理流程，LED 自动熄灭，恢复为系统默认的 SD 卡读写活动触发模式。

4. **异常终止后的资源清理（可选）**：若测试中按`Ctrl+C`强制终止程序，执行以下命令释放GPIO引脚资源：

```Bash
sudo gpioset gpiochip1 3 in
```

![image-20260306220549383](./images/image-20260306220549383.png)

## 五、异常场景及处理方案

|异常现象|可能原因|处理方案|
|---|---|---|
|编译报错：`fatal error: gpiod.h: No such file or directory`|未安装libgpiod-dev或头文件路径错误|重新执行`sudo apt install libgpiod-dev`，确认`/usr/include/gpiod.h`存在|
|运行报错：`Error: 打开gpiochip1失败`|gpiochip1权限不足/不存在|执行`sudo ls /dev/gpiochip1`确认文件存在，确保以sudo运行程序|
|运行报错：`Error: 获取GPIO1_3引脚失败`|引脚号错误/引脚被占用|执行`sudo gpioinfo gpiochip1`查看有效引脚，修改代码中`gpiod_chip_get_line`的引脚号|
|程序运行无报错，但LED不亮|LED接线错误/电平逻辑反|1. 调换LED正负极；2. 将代码中`gpiod_line_set_value(line, 1)`改为`0`，`0`改为`1`|
|LED闪烁但亮度微弱|串联电阻阻值过大（如10kΩ）|更换为220Ω/470Ω电阻，或直接移除电阻|
