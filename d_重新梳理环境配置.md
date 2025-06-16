我想尝试做点 nordic nrf5340 + zephyr 相关的智能手表研发, 现在
已经拿到 nrf5340 dk 开发板子了.
需要怎样进一步配置我的电脑, 以便进行研发工作?

请注意, 
1. 我在中国, 网络环境不好, 尽量使用浏览器或下载工具进行下载
2. 尽量使用命令行和neovim等传统editor进行工作
3. 我的os用的是manjaro Linux
4. 不想使用yay等arch aur的原生工具, 因为会引发manjaro稳定性问题, 请把yay等替换为pacman或snap
5. 有些包gitee上没有(不是不登录gitee, 而是: 即使在浏览器中登录了gitee, 但还是看不到这个代码库)

------

## 解决方案：混合镜像源配置

### 一、包管理（不使用 AUR）

```bash
# 使用 pacman 安装基础包
sudo pacman -S --needed \
    git cmake ninja gperf ccache \
    dtc wget python python-pip \
    python-setuptools python-wheel \
    xz file make gcc \
    sdl2 libusb hidapi \
    neovim tmux ripgrep fd \
    bear clang picocom

# ARM 工具链从官方下载
cd ~/Downloads
wget https://developer.arm.com/-/media/Files/downloads/gnu/13.2.rel1/binrel/arm-gnu-toolchain-13.2.rel1-x86_64-arm-none-eabi.tar.xz

# 解压到 /opt
sudo tar -xf arm-gnu-toolchain-13.2.rel1-x86_64-arm-none-eabi.tar.xz -C /opt/
sudo ln -s /opt/arm-gnu-toolchain-13.2.Rel1-x86_64-arm-none-eabi/bin/* /usr/local/bin/

# dfu-util 使用 snap
sudo snap install dfu-util
```

### 二、重新配置 Zephyr（混合镜像源）

#### 1. 清理之前的尝试
```bash
cd ~/
rm -rf zephyrproject
mkdir -p zephyrproject
```

#### 2. 使用官方 GitHub 获取主仓库
```bash
cd ~/zephyrproject

# 如果有代理，设置代理
export https_proxy=http://127.0.0.1:7890
export http_proxy=http://127.0.0.1:7890

# 克隆主仓库
git clone https://github.com/zephyrproject-rtos/zephyr.git
cd zephyr
git checkout v3.5.0
```

#### 3. 创建智能的 west.yml 配置

创建脚本 `~/fix-west-mirrors.py`：
```python
#!/usr/bin/env python3
import yaml
import sys

# 已知在 Gitee 上存在的仓库
gitee_available = [
    'canopennode', 'cmsis', 'edtt', 'fatfs', 
    'hal_altera', 'zcbor', 'mbedtls', 'mcuboot',
    'net-tools', 'openthread', 'segger',
    'tinycrypt', 'littlefs', 'mipi-sys-t',
    'nrf_hw_models', 'hal_nordic'
]

# 读取 west.yml
with open('west.yml', 'r') as f:
    data = yaml.safe_load(f)

# 修改 URL
for project in data['manifest']['projects']:
    name = project['name']
    if 'url' in project:
        continue
    
    # 默认使用 GitHub
    base_url = 'https://github.com/zephyrproject-rtos'
    
    # 如果在 Gitee 列表中，使用 Gitee
    if name in gitee_available:
        base_url = 'https://gitee.com/zephyr-rtos'
    
    # 设置完整 URL
    project['url'] = f'{base_url}/{name}'

# 保存修改后的文件
with open('west.yml', 'w') as f:
    yaml.dump(data, f, default_flow_style=False)

print("west.yml has been updated with mixed mirror sources")
```

运行脚本：
```bash
cd ~/zephyrproject/zephyr
pip install pyyaml --user
python3 ~/fix-west-mirrors.py
```

#### 4. 使用国内 CDN 加速 GitHub（如果没有代理）

创建 `~/.gitconfig.zephyr`：
```ini
[url "https://github.moeyy.xyz/https://github.com/"]
    insteadOf = https://github.com/
[url "https://mirror.ghproxy.com/https://github.com/"]
    insteadOf = https://github.com/
```

使用时：
```bash
# 临时使用加速配置
export GIT_CONFIG_GLOBAL=~/.gitconfig.zephyr
```

### 三、完成 Zephyr 安装

```bash
# 初始化 west
cd ~/zephyrproject/zephyr
west init -l .

# 更新模块（使用混合源）
cd ~/zephyrproject
west update

# 安装 Python 依赖
pip install --user -r zephyr/scripts/requirements.txt \
    -i https://pypi.tuna.tsinghua.edu.cn/simple
```

### 四、Fish Shell 配置

创建 `~/.config/fish/functions/zephyr-env.fish`：
```fish
function zephyr-env
    # Zephyr 环境变量
    set -gx ZEPHYR_BASE ~/zephyrproject/zephyr
    set -gx PATH ~/.local/bin $PATH
    set -gx BOARD nrf5340dk_nrf5340_cpuapp
    
    # 工具链设置
    set -gx ZEPHYR_TOOLCHAIN_VARIANT gnuarmemb
    set -gx GNUARMEMB_TOOLCHAIN_PATH /opt/arm-gnu-toolchain-13.2.Rel1-x86_64-arm-none-eabi
    
    echo "Zephyr environment loaded!"
    echo "Board: $BOARD"
    echo "Zephyr: $ZEPHYR_BASE"
end

# 别名定义
alias zb='west build -b $BOARD'
alias zf='west flash'
alias zc='rm -rf build && west build -b $BOARD'
alias zd='west debug'
alias zm='picocom -b 115200 /dev/ttyACM0'
alias zp='west build -b $BOARD --pristine'

# 自动生成编译数据库
function zbb
    bear -- west build -b $BOARD $argv
end
```

加载配置：
```fish
# 加载环境
source ~/.config/fish/functions/zephyr-env.fish
zephyr-env
```

### 五、nRF Command Line Tools 安装（不使用 AUR）

```bash
# 下载官方包
cd ~/Downloads
# 访问 https://www.nordicsemi.com/Products/Development-tools/nrf-command-line-tools/download
# 下载 Linux tar.gz 版本

# 解压安装
tar -xzf nRF-Command-Line-Tools_*_Linux-amd64.tar.gz
sudo mkdir -p /opt/nordic
sudo cp -r nrf-command-line-tools /opt/nordic/
sudo ln -sf /opt/nordic/nrf-command-line-tools/bin/* /usr/local/bin/

# udev 规则
sudo cp /opt/nordic/nrf-command-line-tools/share/99-jlink.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
```

### 六、备用方案：完全离线安装

如果网络问题持续，可以：

#### 1. 下载完整的 Zephyr SDK
```bash
# 访问 https://github.com/zephyrproject-rtos/sdk-ng/releases
# 下载 zephyr-sdk-0.16.4_linux-x86_64.tar.xz

cd ~/Downloads
tar -xf zephyr-sdk-0.16.4_linux-x86_64.tar.xz
sudo mv zephyr-sdk-0.16.4 /opt/
/opt/zephyr-sdk-0.16.4/setup.sh
```

#### 2. 使用预下载的模块包
```bash
# 找朋友或在网络好的地方下载完整的 west update
# 然后打包 modules 目录
tar -czf zephyr-modules.tar.gz modules/
# 传输到你的电脑解压
```

### 七、测试安装

```bash
# 创建测试项目
cd ~/
mkdir test-project && cd test-project

# 简单的 blinky 示例
cat > main.c << 'EOF'
#include <zephyr/kernel.h>
#include <zephyr/drivers/gpio.h>

void main(void)
{
    printk("Hello from nRF5340!\n");
    while (1) {
        k_sleep(K_SECONDS(1));
    }
}
EOF

cat > CMakeLists.txt << 'EOF'
cmake_minimum_required(VERSION 3.20.0)
find_package(Zephyr REQUIRED HINTS $ENV{ZEPHYR_BASE})
project(test)
target_sources(app PRIVATE main.c)
EOF

cat > prj.conf << 'EOF'
CONFIG_GPIO=y
EOF

# 构建
west build -b nrf5340dk_nrf5340_cpuapp
```

这样配置应该能解决你的问题。需要其他帮助吗？
