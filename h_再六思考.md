乱打乱撞一糟后, 发现: **还是 Zephyr / ZSWatch 最为路子最正, 产品最高, 质量最严!**
最好还是从它出发.

它最严重的 (可能也是唯一的) 问题是, 没有心率传感器!
尝试自己加上? 然后怎么做片子?...

zswatch官网blog说, [他们也正在搞心率检测的事情(如下)](https://zswatch.dev/blog/progress-hr-fota), 那更好了 -- 
follow他们, or 自己搞?

```
# 博客解读：ZSWatch 健康追踪 PCB 与固件更新进展

    发布时间：2025 年 5 月 15 日
    作者：Jakob Krantz（高级软件工程师、ZSWatch 创建者及维护者）、Daniel Kampert
    核心主题：ZSWatch 在健康监测硬件和固件升级功能上的最新进展

### 主要里程碑
1. 健康追踪原型 PCB 完成

    目标：验证 ZSWatch 未来健康追踪附加 PCB 的设计，便于测试组件、编写驱动及故障排查。
    技术亮点：
        集成 nRF54L15 BLE MCU，使 PCB 可作为独立设备运行。
        支持通过蓝牙向 ZSWatch 实时流式传输心率数据，并能实时绘制数据图表（当前为原型阶段，功能已初步实现）。
    相关资源：驱动和应用程序代码可在 GitHub 仓库查看：Zephyr-MAX32664C ( https://github.com/ZSWatch/Zephyr-MAX32664C ) 。

2. 固件更新支持 BLE 和 USB 双模式功能：添加引导加载程序（Bootloader），支持通过蓝牙（BLE）无线更新或 USB 有线更新（固件故障时的备用）。
3. 技术突破：XIP（Execute-in-Place）功能：代码可直接从外部闪存运行，无需加载到内存。

### 下一步计划
-  评估健康追踪 PCB 的性能表现。
-  设计适配的外壳，便于佩戴健康追踪 PCB 进行实际测试。
```
