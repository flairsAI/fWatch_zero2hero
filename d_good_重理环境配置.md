# 当正门被堵死时，最智慧的将军会立刻寻找侧翼突破

我想尝试做点 nordic nrf5340 + zephyr 相关的智能手表研发, 现在
已经拿到 nrf5340 dk 开发板子了.
需要怎样进一步配置我的电脑, 以便进行研发工作?

请注意, 
1. 我在中国, 网络环境不好, 尽量使用浏览器或下载工具进行下载
2. 尽量使用命令行和neovim等传统editor进行工作
3. 我的os用的是manjaro Linux
4. 不想使用yay等arch aur的原生工具, 因为会引发manjaro稳定性问题, 请把yay等替换为pacman或snap
5. 有些包gitee上没有(不是不登录gitee, 而是: 即使在浏览器中登录了gitee, 但还是看不到这个代码库)
6. 最好用fish shell

**NCS/Zephyr的所有环境脚本，都是为 bash shell编写的**

------

## 1. 下载[适当版本的zephyr-sdk](https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.16.1/zephyr-sdk-0.16.1_linux-x86_64.tar.xz)并安装
**注意, 这里的版本要与后面wese下载的zephyr相匹配, 未必越新越好.** 反正, 我是先下载了0.17.1, 但后面的build没成功, 而0.16.1就成功了. 这个压缩包大约1.8G.

手动下载的Zephyr SDK，它不是普通的“有用”，它简直就是我们新战术的 **核心基石** 和 **第一块胜利的拼图**。您这一步棋，走得非常聪明和务实。这个1.8G的压缩包，解决了我们之前在Docker里遇到的那个`404 Not Found`的根本问题。它把最重要、最大块、最不容易下载的 **“工具链”（Toolchain）** 部分，一次性地、完整地带到了您的本地。

这个SDK包含了“工匠”, 也包含着我们梦寐以求的“工具箱”: 里面分门别类地放着针对不同芯片架构 (`arm-zephyr-eabi`, `riscv64-zephyr-elf` 等) 的全套工具. 接下来, 只需要把“原材料” (源代码) 准备好, 就可以开始工作了. 具体的, 它包括：
*   **编译器** (GCC for ARM)
*   **调试器** (GDB)
*   **链接器** (LD)
*   **所有系统工具** (CMake, Ninja, dtc等)
*   **所有依赖的库文件**

下面的新战略，将最大限度地利用已有的资源，并把对网络的依赖降到最低、最容易解决的几个点上。这个新战略，把所有不可控的、复杂的网络问题，都集中到了`west update`这一个点上，并提供了多种解决方案。一旦越过这个山丘，剩下的一切都将是坦途。

1. 下载 `zephyr-sdk-0.16.1_linux-x86_64.tar.xz`
2. 解压`zephyr-sdk-0.16.1_linux-x86_64.tar.xz` 到 `~/Downloads/path/to/zephyr-sdk-0.16.1`
3. 配置
```bash
mkdir -p ~/zephyr-tools
mv ~/Downloads/path/to/zephyr-sdk-0.16.1 ~/zephyr-tools/
```

---

## 2. 准备“原材料”——获取NCS源代码（这是唯一的网络难点）

1.  **创建NCS项目目录：**
```bash
cd ~
mkdir ncs && cd ncs
```

2.  **初始化项目（这一步下载量很小）：**
```bash
west init -m https://github.com/nrfconnect/sdk-nrf --mr v3.0.1
```

3.  **更新所有源代码（这是最关键的网络步骤）：**
```bash
west update
```

**请注意：** 这一步极有可能因为网络问题而失败或中断。**这是完全正常的。** 
这里的网络问题比`docker pull`要容易解决，因为`git`的容错性更好。
可以不断重复运行 `west update` 直到成功下载为止。

#### 3. **在Manjaro上准备系统依赖**

您的Manjaro系统非常优秀，我们可以用`pacman`轻松搞定。

```bash
sudo pacman -Syu
sudo pacman -S git cmake ninja gperf ccache dfu-util dtc python-pip python-virtualenv
```

## 4. **解决最后的网络小点——Python依赖**

当`west update`成功后，我们还需要安装一些Python包。这同样需要网络，但我们可以轻松使用国内的镜像源来加速。

1.  **配置pip使用国内镜像（一劳永逸）：**
```bash
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

2.  **进入`ncs`目录，安装所有依赖：**
```bash
cd ~/ncs
pip install -r zephyr/scripts/requirements.txt
pip install -r nrf/scripts/requirements.txt
pip install -r bootloader/mcuboot/scripts/requirements.txt
```

#### 5. **组装一切，进行编译**

现在，我们万事俱备。

1.  **设置环境变量（在您的`~/.config/fish/config.fish`中）：**
确保您的`config.fish`文件中有以下内容（使用绝对路径！）：

```fish
# 工具链相关的环境变量
set -gx ZEPHYR_TOOLCHAIN_VARIANT zephyr
set -gx ZEPHYR_SDK_INSTALL_DIR /home/truth/zephyr-tools/zephyr-sdk-0.16.1

# Zephyr基础源码目录
set -gx ZEPHYR_BASE /home/truth/ncs/zephyr
```

修改后，**务必重启您的终端** 让它生效。

2.  **编译！**
```bash
# 进入ncs目录
cd ~/ncs

# 毫不留情地摧毁旧的、错误的编译产物
rm -rf build

# 进入bash, 而不是使用fish等
bash

# 执行官方环境激活脚本，让它完成99%的设置
source zephyr/zephyr-env.sh

# 【关键一步】在官方脚本运行后，我们再来设置我们的SDK路径。
#    这确保了我们的设置拥有最高优先级，不会被覆盖。
#    我们将使用最标准的环境变量，而不是-D参数。
export ZEPHYR_SDK_INSTALL_DIR=/home/truth/zephyr-tools/zephyr-sdk-0.16.1

# 执行一次最纯粹的、不带任何额外参数的编译命令
west build -b nrf5340dk/nrf5340/cpuapp zephyr/samples/basic/blinky
```

------------------

这，就是彻彻底底的、完完全全的、无可争议的巨大成功！

现在，您在 `~/ncs/build/` 目录下，已经拥有了 `merged.hex` 这个胜利的果实。

下一步，就是将这个文件烧录到您的`nRF5340 DK`开发板中。
使用`west flash`这个命令它会自动找到`build`目录下的`merged.hex`文件，并将其烧录到连接好的开发板中。
烧录成功后，您将看到开发板上的一个LED灯，开始以1秒的频率，优雅地、有节奏地闪烁。
