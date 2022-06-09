# 通过 Wi-Fi 连接到设备（Android 10 及更低版本）

> 注意：以下说明不适用于搭载 Android 10 或更低版本的 Wear 设备。如需了解详情，请参阅调试 Wear OS 应用指南。

一般情况下，adb 通过 USB 与设备进行通信，但您也可以通过 Wi-Fi 使用 adb。如要连接到搭载 Android 10 或更低版本的设备，您必须通过 USB 执行一些初始步骤，如下所述：

将 Android 设备和 adb 主机连接到这两者都可以访问的同一 Wi-Fi 网络。请注意，并非所有接入点都适用；您可能需要使用防火墙已正确配置为支持 adb 的接入点。 如果您要连接到 Wear
OS 设备，请关闭手机上与该设备配对的蓝牙。

## 1使用 USB 线将设备连接到主机。

```
设置目标设备以监听端口 5555 上的 TCP/IP 连接。

adb tcpip 5555
```

## 2 拔掉连接目标设备的 USB 线。

找到 Android 设备的 IP 地址。例如，对于 Nexus 设备，您可以在设置 > 关于平板电脑（或关于手机）> 状态 > IP 地址下找到 IP 地址。或者，对于 Wear OS
设备，您可以在设置 > WLAN 设置 > 高级 > IP 地址下找到 IP 地址。 通过 IP 地址连接到设备。

```

adb connect device_ip_address:5555
```

## 3 确认主机已连接到目标设备：

```
$ adb devices
List of devices attached
device_ip_address:5555 device
```

现在，您可以开始操作了！

## 4 如果 adb 连接断开：

确保主机仍与 Android 设备连接到同一个 WLAN 网络。 通过再次执行 adb connect 步骤重新连接。 如果上述操作未解决问题，重置 adb 主机：

```
adb kill-server
```

然后，从头开始操作。

# 查询设备

在发出 adb 命令之前，了解哪些设备实例已连接到 adb 服务器会很有帮助。您可以使用 devices 命令生成已连接设备的列表。

```
adb devices -l
```

作为回应，adb 会针对每个设备输出以下状态信息：

序列号：由 adb 创建的字符串，用于通过端口号唯一标识设备。 下面是一个序列号示例：emulator-5554 状态：设备的连接状态可以是以下几项之一： offline：设备未连接到 adb
或没有响应。 device：设备现已连接到 adb 服务器。请注意，此状态并不表示 Android 系统已完全启动并可正常运行，因为在设备连接到 adb
时系统仍在启动。不过，在启动后，这将是设备的正常运行状态。 no device：未连接任何设备。 说明：如果您包含 -l 选项，devices
命令会告知您设备是什么。当您连接了多个设备时，此信息很有用，可帮助您将它们区分开来。 以下示例展示了 devices
命令及其输出。有三个设备正在运行。列表中的前两行表示模拟器，第三行表示连接到计算机的硬件设备。

(官网教程adb原理篇)[https://developer.android.com/studio/command-line/adb?hl=zh-cn#wireless]
(知乎adb实践篇 )[https://zhuanlan.zhihu.com/p/89060003]
