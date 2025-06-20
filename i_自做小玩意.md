我已经配置好了zephyr sdk, ncs等; 现在想在nrf5340 dk上, 做点有趣的小玩意.
首先想到的一个是, 按右下面的四个按钮之一, 对应位置的右上的小灯闪烁.
如果想使用nvim+命令行进行开发, 应该怎样做?

---

# 关于项目核心(代码)的实现

这是一个非常棒的入门项目！是学习 GPIO（通用输入/输出）操作、中断处理和 nRF Connect SDK 基础知识的绝佳方式。

### 项目目标

按下 nRF5340 DK 上的四个按钮（BUTTON 1-4）之一，其正上方对应的 LED（LED 1-4）就会改变状态（亮或灭）。

### 理解硬件与设备树 (DeviceTree)

在 Zephyr 系统中，硬件资源（如 GPIO、UART 等）不是通过直接写死引脚号来访问的，
而是通过一个叫做“设备树 (DeviceTree)”的抽象层。 [[6]](https://blog.csdn.net/bjbz_cxy/article/details/119575487) 
这让代码有更好的可移植性。

nRF5340 DK 已经为我们定义好了板载资源别名：
*   **按钮 (Buttons)**: `button0`, `button1`, `button2`, `button3`
*   **LEDs**: `led0`, `led1`, `led2`, `led3`

控制关系大致如下：
*   `button0` (Button 1) -> `led0` (LED 1)
*   `button1` (Button 2) -> `led1` (LED 2)
*   `button2` (Button 3) -> `led2` (LED 3)
*   `button3` (Button 4) -> `led3` (LED 4)

我们将在代码中直接使用这些别名。

### 编写代码

现在，打开项目文件夹中的 `src/main.c` 文件，并用下面的代码替换其全部内容。

```c
/*
 * If you can't find your glasses, it's because you don't have them on.
 */

#include <zephyr/kernel.h>
#include <zephyr/device.h>
#include <zephyr/devicetree.h>
#include <zephyr/drivers/gpio.h>
#include <zephyr/sys/printk.h>

/*
 * 设备树 (DeviceTree) 宏定义
 * 使用 DT_ALIAS() 从设备树中获取节点的别名。
 * 我们需要4个LED和4个按钮。
 */
#define LED0_NODE DT_ALIAS(led0)
#define LED1_NODE DT_ALIAS(led1)
#define LED2_NODE DT_ALIAS(led2)
#define LED3_NODE DT_ALIAS(led3)

#define SW0_NODE DT_ALIAS(sw0) // sw0 是 button0 的另一个常用别名
#define SW1_NODE DT_ALIAS(sw1)
#define SW2_NODE DT_ALIAS(sw2)
#define SW3_NODE DT_ALIAS(sw3)

// 使用 GPIO_DT_SPEC_GET 宏从设备树节点中提取 GPIO 的完整规格（控制器、引脚号、标志位）
// 这是一个更现代、更安全的做法。
static const struct gpio_dt_spec leds[] = {
    GPIO_DT_SPEC_GET(LED0_NODE, gpios),
    GPIO_DT_SPEC_GET(LED1_NODE, gpios),
    GPIO_DT_SPEC_GET(LED2_NODE, gpios),
    GPIO_DT_SPEC_GET(LED3_NODE, gpios),
};

static const struct gpio_dt_spec buttons[] = {
    GPIO_DT_SPEC_GET(SW0_NODE, gpios),
    GPIO_DT_SPEC_GET(SW1_NODE, gpios),
    GPIO_DT_SPEC_GET(SW2_NODE, gpios),
    GPIO_DT_SPEC_GET(SW3_NODE, gpios),
};

// 定义一个静态的回调结构体，用于处理按钮中断
static struct gpio_callback button_cb_data;

// 按钮中断回调函数
// 当任何一个配置了中断的按钮被按下时，这个函数就会被调用
void button_pressed(const struct device *dev, struct gpio_callback *cb, uint32_t pins)
{
    // 遍历所有按钮，检查是哪一个触发了中断
    for (int i = 0; i < ARRAY_SIZE(buttons); i++) {
        // (1 << buttons[i].pin) 创建一个掩码，检查触发中断的引脚是否是当前按钮的引脚
        if (pins & (1 << buttons[i].pin)) {
            printk("Button %d pressed!\n", i + 1);
            // 切换对应位置LED的状态
            gpio_pin_toggle_dt(&leds[i]);
        }
    }
}

int main(void)
{
    int ret;
    printk("nRF5340 DK Button-LED demo started!\n");

    // 1. 配置所有LED引脚为输出模式
    for (int i = 0; i < ARRAY_SIZE(leds); i++) {
        if (!device_is_ready(leds[i].port)) {
            printk("Error: LED device %s is not ready\n", leds[i].port->name);
            return 0;
        }
        ret = gpio_pin_configure_dt(&leds[i], GPIO_OUTPUT_INACTIVE);
        if (ret != 0) {
            printk("Error %d: failed to configure LED %d pin %d\n", ret, i, leds[i].pin);
            return 0;
        }
    }

    // 2. 配置所有按钮引脚为输入模式
    for (int i = 0; i < ARRAY_SIZE(buttons); i++) {
        if (!device_is_ready(buttons[i].port)) {
            printk("Error: Button device %s is not ready\n", buttons[i].port->name);
            return 0;
        }
        // 配置为输入模式
        ret = gpio_pin_configure_dt(&buttons[i], GPIO_INPUT);
        if (ret != 0) {
            printk("Error %d: failed to configure Button %d pin %d\n", ret, i, buttons[i].pin);
            return 0;
        }
        // 配置中断，当引脚变为高电平（按钮按下）时触发
        ret = gpio_pin_interrupt_configure_dt(&buttons[i], GPIO_INT_EDGE_TO_ACTIVE);
        if (ret != 0) {
            printk("Error %d: failed to configure interrupt on pin %d\n", ret, buttons[i].pin);
            return 0;
        }
    }

    // 3. 设置中断回调函数
    // 初始化回调结构体，指定回调函数和要监控的引脚掩码
    // 注意：这里我们只使用第一个按钮的控制器(port)来添加回调，因为nRF5340 DK上所有按钮都在同一个GPIO控制器上。
    // 如果按钮在不同的控制器上，需要为每个控制器单独添加回调。
    uint32_t pin_mask = 0;
    for(int i = 0; i < ARRAY_SIZE(buttons); i++) {
        pin_mask |= (1 << buttons[i].pin);
    }
    gpio_init_callback(&button_cb_data, button_pressed, pin_mask);
    
    // 将回调函数添加到GPIO驱动中
    gpio_add_callback(buttons[0].port, &button_cb_data);

    printk("Configuration complete. Press a button...\n");

    // 主循环可以为空，因为所有工作都由中断驱动
    return 0;
}
```

### 项目配置

在 Zephyr 中，项目的功能开关通过 `prj.conf` 文件来管理。确保你的项目根目录下有这个文件，并包含以下内容：

**`prj.conf`**
```conf
# 启用 GPIO 驱动
CONFIG_GPIO=y

# 启用 printk 以便在控制台看到输出
CONFIG_PRINTK=y
```

### 代码解释

*   **设备树宏**: `DT_ALIAS` 和 `GPIO_DT_SPEC_GET` 是 Zephyr 提供的标准宏，用于从设备树中安全地获取硬件信息，避免了硬编码。 [[6]](https://blog.csdn.net/bjbz_cxy/article/details/119575487)
*   **数组管理**: 我们将 LED 和按钮的 `gpio_dt_spec` 结构体放入数组中，这样可以方便地使用循环进行初始化，代码更整洁。
*   **中断驱动**: 这个程序的核心是中断。我们没有在 `main` 函数的 `while(1)` 循环里不断地检查按钮状态（这被称为“轮询”，效率低下），而是将按钮配置为中断模式。 [[7]](https://docs2.listenai.com/x/yfxsSV7f-dS)[[8]](https://www.cnblogs.com/newbit/p/ti_rtos_int.html)
*   **回调函数 `button_pressed`**: 当硬件检测到按钮被按下（引脚电平变化）时，CPU 会暂停当前工作，立即执行我们注册的 `button_pressed` 函数。 [[6]](https://blog.csdn.net/bjbz_cxy/article/details/119575487) 函数执行完毕后，CPU 会返回原来的地方继续工作。这非常高效。
*   **`gpio_pin_toggle_dt`**: 这个函数的作用是“翻转”一个 GPIO 引脚的电平。如果原来是高电平，就变为低电平（灯灭）；如果原来是低电平，就变为高电平（灯亮）。

### 进阶玩法

这个小项目成功后，你可以尝试一些更有趣的扩展：

1.  **按键消抖 (Debounce)**: 物理按键在按下和释放的瞬间，会产生快速的、不稳定的电平抖动。你可能会发现快速按键时 LED 行为不符合预期。可以研究一下如何用 `k_timer` 定时器来实现按键消抖。
2.  **不同的闪烁模式**: 修改回调函数，让 LED 不是简单地翻转状态，而是实现特定的闪烁模式，比如“双击”按钮切换常亮/呼吸灯模式。
3.  **蓝牙联动**: 这是 nRF5340 的核心功能。尝试修改项目，当按下按钮时，通过蓝牙低功耗 (Bluetooth LE) 发送一条通知给你的手机。你可以从 `zephyr/samples/bluetooth/peripheral_lbs` (LED Button Service) 这个官方例程开始学习。 [[5]](https://blog.csdn.net/cczy_/article/details/112062466)

祝你在 nRF5340 的探索之路上玩得开心！

---
Learn more:
1. [nRF5340 DK - Get Started - nordicsemi.com](https://www.nordicsemi.com/Products/Development-hardware/nrf5340-dk/getstarted?lang=zh-CN)
2. [nrf5340开发入门原创 - CSDN博客](https://blog.csdn.net/m0_37597968/article/details/144005322)
3. [nRF52840开发环境搭建教程 - 亿佰特](https://www.ebyte.com/pdf-down/3253.html)
4. [nRF Connect SDK安装与入门- Nordic Semiconductor中文官网 - 诺迪克半导体](https://www.nordicsemi.cn/blog/nrf-connect-sdk-guide/)
5. [基于NRF5340-(P)DK 的第一个BLE例程原创 - CSDN博客](https://blog.csdn.net/cczy_/article/details/112062466)
6. [Zephry\_GPIO的中断使用详解以及中断原理原创 - CSDN博客](https://blog.csdn.net/bjbz_cxy/article/details/119575487)
7. [GPIO - 聆思文档中心](https://docs2.listenai.com/x/yfxsSV7f-dS)
8. [TI-RTOS 之GPIO中断（按键） - Newbit - 博客园](https://www.cnblogs.com/newbit/p/ti_rtos_int.html)

---

# 项目支撑(周边)的实现

使用 `Neovim + 命令行` 是很多经验丰富的嵌入式开发者的选择，因为它高效、灵活且高度可定制。
既然你已经配置好了 SDK 和工具链，那么剩下的流程会非常直接。

核心工具是 `west` (Zephyr's meta-tool)，它负责创建、构建、烧录和调试等所有工作。

下面是纯命令行环境下的完整开发流程。

### 第 1 步：激活 NCS 环境

在开始任何操作之前，最重要的一步是激活 nRF Connect SDK 的环境。这会将 `west`、`arm-none-eabi-gcc` 等所有必要的工具链路径添加到你当前的终端会话中。

进入你的 nRF Connect SDK 安装目录（例如 `~/ncs`），然后执行：

```bash
# 如果你使用 bash 或 zsh # 感觉以后还是老老实实使用zsh?
source zephyr/zephyr-env.sh

# # 如果你使用 fish shell
# source zephyr/zephyr-env.fish
```

**提示**: 你可以将这个 `source` 命令添加到你的 shell 配置文件中（如 `.bashrc` 或 `.zshrc`），这样每次打开新终端时环境就自动准备好了。

### 第 2 步：创建项目

最简单的方法是从一个官方的示例开始，然后进行修改。

1.  **寻找一个基础项目**: `zephyr/samples/basic/blinky` 是一个很好的起点，因为它包含了最基本的设备树和 GPIO 操作。
2.  **复制项目到你的工作区**:
    ```bash
    # 假设你的工作目录是 ~/projects
    cd ~/projects
    cp -r ~/ncs/zephyr/samples/basic/blinky ./nrf5340-button-led
    cd ./nrf5340-button-led
    ```
    现在，你就有了一个独立的、可以随意修改的项目了。

### 第 3 步：编写代码 (使用 nvim)

现在，使用你最喜欢的编辑器 `nvim` 来修改项目文件。

1.  **修改 `prj.conf`**: 这个文件用来配置项目需要启用的内核功能。
    ```bash
    nvim prj.conf
    ```
    替换其内容 为 我们之前讨论过的的代码

2.  **修改 `src/main.c`**: 这是我们的主程序逻辑。
    ```bash
    nvim src/main.c
    ```
    替换其全部内容 为 我们之前讨论过的、基于中断的代码

### 第 4 步：编译项目

在你的项目根目录 (`~/projects/nrf5340-button-led`) 下，使用 `west build` 命令进行编译。

```bash
# -b: 指定目标板。nrf5340dk_nrf5340_cpuapp 表示 nRF5340 DK 的应用核
# -p auto: (Pristine) 推荐使用，它会在每次构建前清理旧的构建文件，避免奇怪的缓存问题
west build -b nrf5340dk_nrf5340_cpuapp -p auto
```
编译过程的输出会显示在终端。如果一切顺利，你会在项目目录下看到一个 `build` 文件夹，里面包含了所有编译产物，如 `zephyr/zephyr.hex` 和 `zephyr/zephyr.elf`。

### 第 5 步：烧录程序

将 nRF5340 DK 连接到电脑，然后执行 `west flash`。

```bash
west flash
```
`west` 会自动从 `build` 目录中找到编译好的固件，并使用默认的烧录工具（对于 Nordic 板子，通常是 `nrfjprog`）将其烧录到开发板上。

### 第 6 步：查看日志输出

要查看 `printk` 的输出，你需要一个 RTT (Real-Time Transfer) 查看器。

#### 方法一：使用 SEGGER J-Link RTT Viewer (图形界面)

这是最简单可靠的方法。
1.  从 [SEGGER 官网](https://www.segger.com/downloads/jlink/#J-LinkSoftwareAndDocumentationPack) 下载并安装 J-Link Software and Documentation Pack。
2.  打开 `JLinkRTTViewer` 程序。
3.  在弹出的配置窗口中，通常直接点击 "OK" 即可自动连接。
4.  程序运行后，你就能在窗口中看到 `printk` 打印的 "demo started" 等信息。
    按下按钮，会看到 "Button X pressed!" 的输出。

#### 方法二：使用 J-Link 命令行工具 (纯命令行)

如果你想坚持使用命令行，可以这样做：
1.  打开一个 **新的终端**。
2.  启动 `JLinkExe`，这是 J-Link 的命令行工具。
    ```bash
    JLinkExe -device nRF5340_xxAA_APP -if SWD -speed 4000 -autoconnect 1
    ```
3.  在 J-Link 的交互式命令行中，输入 `rtt L` (或者 `rtt start`) 来启动 RTT 日志。
    ```
    J-Link> rtt L
    ```
    之后日志就会开始在这个终端里打印。

### 总结：你的日常开发流程

一旦项目建立起来，你每天的开发循环将非常简单：

1.  打开终端，`cd` 到项目目录 `~/projects/nrf5340-button-led`。
2.  (如果需要) `source ~/ncs/zephyr/zephyr-env.sh`。
3.  使用 `nvim src/main.c` 修改代码。
4.  运行 `west build -b nrf5340dk_nrf5340_cpuapp -p auto` 编译。
5.  运行 `west flash` 烧录。
6.  在另一个终端或 RTT Viewer 中观察结果。

这个流程完全摆脱了对 IDE 的依赖，对于习惯命令行的开发者来说，速度和效率都非常高。
