# OpenWrt路由器多核终端均衡脚本
## 目的
通过均衡分配用于处理中断的CPU核心，提升系统的吞吐性能
## 原理
OpenWrt默认、Lean OpenWrt默认、使用irqbalance及转发优化版本的核心分配情况如下(IPQ40xx)：
| 中断 | OpenWrt | Lean | irqbalance| 转发优化 | 
| :----: | :----: | :----: | :----: | :----: |
| 网络队列rx | CPU123 | CPU0123 | CPU123 | CPU12交替 |
| 网络队列tx | 交替 | 交替 | 交替 | CPU12交替 |
| 网络中断rx | CPU0123 | 4个一组交替 | 交替 | 4个一组CPU12反向交替 |
| 网络中断tx | CPU0123 | 交替 | 交替 | CPU12反向交替 |
| 无线ahb | CPU0123 | CPU2 | CPU2 | CPU3 |
| 无线pcie | CPU0123 | CPU3(设置无效) | CPU0 | CPU3 |
| 其他(usb,dma,gpio...) | CPU0123 | CPU0123 | CPU0/CPU2/CPU3 | CPU0 |

CPU123表示使用CPU1、CPU2、CPU3均可。为了提升局部性以提升缓存效率，中断往往被固定在所有指定CPU中最小的那个，在缺少硬件NAT与千兆网的情况下很容易占满1个CPU核心而其他核心空闲，出现性能瓶颈。因此需要调整中断与CPU的对应关系。
## 结构
脚本参照init.d的格式编写，可直接复制到路由器的/etc/init.d目录下，调整文件名为set_smp_affinity（如果有同名文件则删除），每次重启后即可自动设置irq与核心的绑定关系。
CPU核心与Mask遮罩值的对应关系如下所示：
| CPU0 | CPU1 | CPU2 | CPU3 |
| :----: | :----: | :----: | :----: |
| 1 | 2 | 4 | 8 |

如果需要同时使用多个核心，则直接将遮罩值累加取16进制，如使用CPU2与CPU3则值为C。
项目中脚本与平台的对应关系如下表所示：
| 脚本名称 | 对应平台 | 说明 |
| :----: | :----: | :----: |
| ipq40xx/openwrt.sh | 高通IPQ4018/4019/... | 针对有线与无线转发优化的配置 |
| ipq40xx/lean-default.sh | 高通IPQ4018/4019/... | Lean固件中的默认配置 |
| mt7621/openwrt.sh | 联发科mt7621 | 针对有线与无线转发优化的配置 |
| mt7621/lean-default.sh | 联发科mt7621 | Lean固件中的默认配置 |
| mt7621/pandorabox.sh | 联发科mt7621 | 适用于潘多拉的有线无线优化配置 |
| bcm53xx/openwrt.sh | 博通bcm4708/4709 | 针对有线与无线转发优化的配置 |
## 状态
目前只基于Asus RT-ACRH17测试了IPQ4019平台。
### 测试场景
使用两台间隔7米的ACRH17组建WDS无线桥接网络，主路由中运行iperf3服务器，主从路由通过QCA9984 5G进行连接，测试电脑连接于从路由的LAN1接口。
两台ACRH17均使用OpenWrt官方19.07.2固件与ath10k无线驱动。
测试电脑中运行代码`iperf3 -c 主路由IP -t 30 -P 4`。
使用从路由测试均衡脚本。
吞吐量的结果如下表所示：
| 中断绑定配置 | 速度/Mbps |
| :----: | :----: |
| OpenWrt默认 | 495 |
| Lean默认 | 386 |
| irqbalance | 476 |
| 转发优化 | 581 |
