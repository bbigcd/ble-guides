# 理解&优化 BLE 的吞吐量

这篇文章着重说明在使用 GATT 过程中控制 BLE 吞吐量的因素,  你会需要把它调到最大程度.

当我试图去了解 BLE 的吞吐量时, 我因为缺少关于这个主题的文章而感到沮丧... 这就是我写这篇文章的目的!

如果你正在试图去提高你的 BLE 应用的吞吐量, 或者仅仅为了了解更多关于 BLE 通用协议栈的内容, 请继续阅读!

# 术语

在我们开始之前, 我们需要去了解一些基本的术语.

* **连接事件** - 对于 BLE, 两个设备在一次连接中彼此进行通讯. 即使没有数据交互, 彼此必须定期 ping 对方, 以确保*连接*仍然活着. 每一次设备之间的彼此 ping 对方的过程被称为*连接事件*
* **连接间隔** - 每个*连接事件*之间的间隔时间(有效区间为7.5ms到4秒). 当两个设备连接好之后, *连接间隔*可以进行协商.为了优化功耗, 动态的改变*连接间隔*通常是有效的.

# 理解 LE 包层

为了优化吞吐量, 第一件事就是要查看蓝牙的 LE 包的组成.

## 蓝牙 基带 / 无线电

蓝牙无线电每微秒能传输一个符号(比特率为1兆/秒), 然而这并不是真正的吞吐量, 原因如下:

1. 在每个发送的数据包之间必须有一个强制性的*150μs* *延迟(称为*帧空间(T_IFS))
2. BLE 是一种*可靠性传输*, 意味着从一方发送的每个数据包都必须被另一方确认(ACK).一个 ACK 包的大小是80 bits, 因此需要 *80μs* 来传输.
3. 最大的数据包为 41bytes(328 bits), 因此需要 328μs 来传输.

### 数据交换示例

![Data exchange](ble_throughput_pics/connection_event.png)


## 链路层包(LL)

这是通过空中发送数据使用的格式.所有更高级别的消息被打包在*数据有效负载* 的 *LL 包* 中 . 下面是 LL 包的格式:

![Data exchange](ble_throughput_pics/ll_packet.png)

### 最大的LL数据有效负载吞吐量

交换一个数据包涉及:

* A 端发送一个最大的LL数据包(27 bytes大小的数据)
* B 端接受这个LL数据包并等待 T_IFS (*150μs*)
* B 端发送一个 ACK LL 包(0 bytes 大小)
* A 端等待 T_IFS 然后可以停止发送或传输更多的数据

根据上面的讨论，可以计算出该序列所花费的时间:

`328μs data packet + 150μs T_IFS + 80μs ACK + 150μs T_IFS = 708μs`

在此期间，实际数据可传输27字节，用时216μs。

这产生一个数据吞吐量:

`(216μs / 708μs) * 1Mbit/sec = 305,084 bits/second = 38.1kBytes/s`

## L2CAP 通道&有效载荷

L2CAP 是更高级的协议，封装在LL包的**数据有效负载**中。L2CAP 允许:

* 在不同的虚拟通道上复用不相关的数据流
* 跨多个LL包分段和反分段数据(因为一个L2CAP有效负载可以跨多个LL包)

只有几个不同的 L2CAP 频道用于 LE:

1. **安全管理器协议(SMP)** -用于BLE安全设置
2. **属性协议(ATT)** - Android和iOS都提供api，允许第三方应用程序通过该通道控制传输。
3. **LE L2CAP面向连接的通道** - 自定义通道，对流媒体应用程序非常有利。然而，目前在移动平台上还没有第三方api，所以它在实践中并不是很有用.

### L2CAP 包

![Data exchange](ble_throughput_pics/l2cap_packet.png)

## 属性协议(ATT)包

在 *L2CAP 信息负载* 中我们有 ATT 包. 这个包的结构是*GATT*协议的封装，BLE设备使用它来彼此交换数据. 它看起阿里是这样的:

![Data exchange](ble_throughput_pics/att_packet.png)

### GATT 协议最高吞吐量

The Attribute Protocol *Handle Value Notification* is the best way for a server to stream data to a client. The *Opcode Overhead* for this operation is 2 bytes. That means there is 3 bytes of ATT packet overhead and 4 bytes of L2CAP overhead for each ATT payload. We can determine the max ATT throughput by taking the max LL throughput and multiplying it by the efficiency of the ATT packet:

`ATT Throughput = LL throughput * ((MTU Size - ATT Overhead) / (L2CAP overhead + MTU Size))`

#### 按 ATT MTU 大小可实现的最大吞吐量
MTU size (bytes)         | Throughput (kBytes/sec) |
-------------            | -------------           |
23 (default)             |  28.2                   |
185 (iOS 10 max)         |  36.7                   |
512 (bluetooth max)      |  37.6                   |

### GATT 的实际吞吐量

In practice, the achievable throughput rates are further restricted by the devices being used. Some common limiting factors are:

* The number of packets which can be transmitted in one *connection event*. An entire *connection event* could be filled with data but many devices will only transmit several packets per event.
* For devices which will only send a few packets per *connection event*, it then becomes important to try and make the *connection interval* very short so data can be sent often. However, devices will often only support a portion of the 7.5ms - 4s range.
* Max MTU size supported by the device (many BT chips in Android devices drop data when the size is greater than the default of 23)
* BT LE, BT Classic & Wifi share the same antenna / chip / scheduler and so bursts of wifi/classic traffic can reduce how many packets of LE data will be sent per *connection event*

Below I'll walk through two practical phone examples and their peak *GATT* throughput. Do note that often a developer will add another protocol on top of *GATT* to stream data. The scheme chosen can also have a significant impact on the throughput.

#### Example 1 - iOS 10 / iPhoneSE

##### Background
For iOS < 10, ~4-6 packets are transmitted per connection interval and the max MTU size supported is 158 bytes. For iOS 10, many iOS devices now support 185 byte MTU and 7 packets per connection interval.

When you connect to a BLE device with iOS, it will start with a *connection interval* of 30ms (or 11.25ms if the device supports HID over GATT). However, it is possible to negotiate the interval down to 15ms once connected. Do note that depending on how many packets a device is capable of transmitting, a lower connection interval may not impact performance because the device could fill all the space in a larger *connection interval* with packets.

##### Achievable Throughput
The following assumes a 15ms conn interval, 185 byte ATT MTU size and that 7 Data packets and be transmitted and acknowledged per *connection interval*

First let's make sure we can transmit 7 packets in 15ms:
` 7 * (328μs Data + 150μs + 80μs ACK + 150μs) = 4.956ms`

Now that we know the packets fit, we can take the duty cycle we are achieving and multiply it by the ATT throughput maximums tabulated above.

`Data Rate = (4.956ms / 15ms) * 36.7 kBytes/Sec = 12.2 kBytes/sec`

#### Example 2 - Nexus 5x

The Nexus 5x has one of the better performing BLE chips I've seen to date. It is willing to fill an entire *Connection Event* with packets and supports an MTU size of 512 bytes. With a connection interval of 15ms, I've seen as many as 21 packets transmitted. This takes:

`21 * (328μs Data + 150μs + 80μs ACK + 150μs) = 14.868ms`

Unfortunately, this seems a bit too overzealous and the chip often skips transmitting any data for the following connection event effectively halving the duty cycle. This still yields a pretty impressive throughput:

`Data Rate = (14.868ms / 30ms) * 37.6 kBytes/sec = 18.6 kBytes/sec`

*Full disclosure:* The Android BLE ecosystem is *very* diverse and while some devices perform quite well, others devices perform very poorly. I've even seen some Android devices only capable of transmitting 1 packet per connection interval!


# Additional Topics ... coming soon

* Bluetooth 4.2 Packet Length Extension (As of BT 4.2, LL packet data payload can be negotiated up to 255 bytes!)
* Auditing what device is limiting throughput
* Dealing with (numerous) BLE bugs in the mobile phone & vendor stacks