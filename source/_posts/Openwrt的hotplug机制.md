---
title: Openwrt的hotplug机制
date: 2020-07-08 17:03:25
tags: Openwrt,hotplug
---

## 热插拔
当某些事件发生时，Procd（init系统和进程管理守护进程）执行位于/etc/hotplug.d/中的脚本，例如当接口启动或关闭时，检测到新的存储驱动器时，或者按下按钮时.

当使用PPPoE连接或者在不稳定的网络中，或使用硬件按钮时非常有用。

该功能模块模拟/扩展了长期未用的Hotplug2程序包所执行的操作。

## 工作原理
在 /etc/hotplug.d 文件夹你可以发现 block iface, net 和 ntp 等文件夹.

触发事件触发后，Procd将按字母顺序执行该触发器子文件夹中的所有脚本。 这就是为什么那里的大多数脚本都使用数字前缀。

- block 文件夹用于块设备事件（块设备已连接/已断开连接）
- iface 文件夹用于接口事件（当LAN或WAN等接口连接/断开时）
- net 文件夹用于：？:(可能与网络相关）
- ntp 文件夹用于时间同步事件（Time step，时间服务器层变化）
- button 文件夹用于按钮事件 (缺省不创建, 由 /etc/rc.button 代替)
- usb 文件夹用于类似3g-modem和tty*的USB设备

对于其他类型的触发器，可能（应该）是其他的。通过查看源码 in git 他们可以是按钮, 声音设备, 串口和USB串口加密狗。

## 用法
只需将脚本放入正确的hotplug.d子目录中，如果没有，只需创建正确的脚本。

### 提供给脚本的信息 / 故障排除
在hotplug.d中执行脚本时，Procd会公开大量信息，通常作为环境变量。

如果要查看它提供的环境变量，请创建一个包含以下行的脚本：
```
env > /tmp/envs_log.log
```
并将其放在您要使用的文件夹中，然后触发连接到该文件夹的事件，然后通过查看/tmp/envs_log.log这个文本文件，您可以看到传递了什么环境变量．

#### block 文件夹
对于 block文件夹中的脚本，他们是这些（相关的）环境变量

变量名|描述
-|-
ACTION|“add” 或 “remove”
DEVICENAME|与下面的DEVNAME相同
DEVNAME|设备或分区名称（如果连接驱动器，你会得到一个使用“sda”的热插拔调用，另一个使用“sda1”进行热插拔调用）
DEVPATH|完整的设备路径（如 “/devices/pci0000:00/0000:00:0b.0/usb1/1-1/1-1:1.0/host7/target7:0:0/7:0:0:0/block/sdc/sdc1 “）
DEVTYPE|DEVNAME和DEVICENAME的名称是什么类型，当插入具有可读分区的设备时，该值为“partition”，当移除该设备时，该值为“disk”。
MAJOR	|主设备号
MINOR	|次要设备号
SEQNUM	|序号(一个数字)
SUBSYSTEM|	固定值 “block”

#### iface 文件夹
传递给每个 iface 热插拔脚本有三个主要的环境量：

变量名	|描述
-|-
ACTION	|“ifup” 或 “ifdown”
INTERFACE	|上线或下线的接口名称（例如“wan”或“ppp0”）
DEVICE	|接口上线或下线的物理设备名称（例如“eth0.1”或“br-lan”）
#### ntp 文件夹
变量名	|描述
-|-
ACTION	|step, stratum, unsync 或 periodic
freq_drift_ppm|	ntp 变量
offset	|ntp 变量
stratum	|ntp 变量
poll_interval	|ntp 变量

即使没有NTP同步，您也会收到一个定期的热插拔事件，其中stratum=16，开机后大约每11分钟一次。

#### usb 文件夹
变量名	|描述
-|-
ACTION	|add, remove 如上
DEVNAME	|如 “bus/usb/001/002”
DEVPATH	|如 ”/devices/platform/ehci-platform/usb1/1-1”
DEVICENAME	|如 “1-1”
DEVNUM	|如 002
DRIVER	|“usb”
TYPE	|如 9/0/1
PRODUCT	|供应商/产品代码/版本, 如用lsusb看到的 “424/2640/0”
SEQNUM	|? 如 335
BUSNUM	|如 001
MAJOR	|如 189
MINOR	|如 1

## 示例
```
cat << "EOF" > /etc/hotplug.d/iface/99-my-action
[ "$ACTION" = ifup ] && {
  logger -t button-hotplug Device: $DEVICE / Action: $ACTION
} 
EOF
```
每次接口上线时，都会执行if / fi内声明的语句。

#### 当USB WiFi热插拔时重启wlan
Niii发布了这个简单的USB WiFi设备热插拔事件示例，以触发init.d 网络重启wlan0脚本。

检测 RTL8188SU_PRODID 的一些参数数据:
```
# lsusb -v
  idVendor           0x0bda Realtek Semiconductor Corp.
  idProduct          0x8171 RTL8188SU 802.11n WLAN Adapter
  bcdDevice            2.00
```
```
cat << "EOF" > /etc/hotplug.d/usb/20-rtl8188su
BINARY="/sbin/wifi up"
RTL8188SU_PRODID="bda/8171/200"
 
if [ "${PRODUCT}" = "${RTL8188SU_PRODID}" ]; then
    if [ "${ACTION}" = "add" ]; then
        ${BINARY}
    fi
fi
EOF
```
#### 符号链接代替设备重命名
另一个用于创建符号链接而不是重命名设备的脚本。
因为当我插入一个usb设备时会收到2个add事件，而为了确保在创建符号链接前设备已经创建，我在这里增加了一个判断DEVICE_NAME是否为空．
```
cat << "EOF" > /etc/hotplug.d/usb/20-cp210x
CP210_PRODID="10c4/ea60/100"
SYMLINK="my_link"
 
if [ "${PRODUCT}" = "${CP210_PRODID}" ];
   then if [ "${ACTION}" = "add" ];
      then
         DEVICE_NAME=$(ls /sys/$DEVPATH | grep tty)
         if [ -z ${DEVICE_NAME} ];
            then logger -t Hotplug Warning DEVICE_NAME is empty
            exit
         fi
         logger -t Hotplug Device name of cp210 is $DEVICE_NAME
         ln -s /dev/$DEVICE_NAME /dev/${SYMLINK}
         logger -t Hotplug Symlink from /dev/$DEVICE_NAME to /dev/${SYMLINK} created
   fi
fi
 
if [ "${PRODUCT}" = "${CP210_PRODID}" ];
   then if [ "${ACTION}" = "remove" ];
         then 
         rm /dev/${SYMLINK}
         logger -t Hotplug Symlink /dev/${SYMLINK} removed
   fi
fi
EOF
```
#### 检测插入的usb设备是否蓝牙的脚本
```
cat << "EOF" > /etc/hotplug.d/usb/20-bt_test
BT_PRODID="a12/1/"
BT_PRODID_HOT=`echo $PRODUCT | cut -c 1-6`
 
#logger -t HOTPLUG "PRODUCT ID is" $BT_PRODID_HOT
 
if [ "$BT_PRODID_HOT" = "$BT_PRODID" ]; then
    if [ "$ACTION" = "add" ]; then
        logger -t HOTPLUG "bluetooth device has been plugged in!"
        if [ "$BSBTID_NEW" = "$BSBTID_OLD" ]; then
            logger -t HOTPLUG "bluetooth device hasn't changed"
        else
            logger -t HOTPLUG "bluetooth device has changed"
        fi
    fi
    if [ "$ACTION" = "remove" ]; then
        logger -t HOTPLUG "bluetooth device has been removed!"
    fi
else
    logger -t HOTPLUG "USB device is not bluetooth"
fi
EOF
```
####当usb摄像头插入时自动启动mjpg-streamer
```
cat << "EOF" > /etc/hotplug.d/usb/20-mjpg_start
case "$ACTION" in
    add)
            # start process
        /etc/init.d/mjpg-streamer start
            ;;
    remove)
            # stop process
        /etc/init.d/mjpg-streamer stop
            ;;
esac
EOF
```
#### xfs的自定义自动挂载脚本
```
cat << "EOF" > /etc/hotplug.d/block/xfs_automount
# 如果新的block设备已连接
if [ "$ACTION" = "add" ] ; then
    # 获取设备 UUID
    detected_uuid=$( xfs_admin -u /dev/$DEVICENAME | awk '{print $3}' )
    # 确定已知UUID的挂载点
    mountpoint=""
    case "$detected_uuid" in
    6a5d7c5c-c9d0-41cc-8f19-78d97f839c05)
            mountpoint="/path/to/first/mountpoint"
            ;;
    02880b1f-0c67-46b6-9b05-5535680ccc89)
            mountpoint="/path/to/second/mountpoint"
            ;;
   esac
 
   # 如果有一个挂载点，则挂载它
   if [ "$mountpoint" != "" ] ; then 
       mount /dev/$DEVICENAME $mountpoint
   fi
fi
# 无论如何，卸载在设备断开时自动发生，因此没有逻辑操作
EOF
```

#### USB 4G模块固定AT指令端口
```
cat << "EOF" > /etc/hotplug.d/usb/10-usb

[ "$SUBSYSTEM" == tty -a "$ACTION" == add ] && {
        echo "$DEVICENAME" | grep 'ttyUSB' || exit 0
        USBDEVICENAME=$(echo $DEVPATH | cut -d '/' -f 8)
        if [ ${USBDEVICENAME} = '1-1.1:1.0' ] ; then
                ln -s /dev/$DEVICENAME /dev/ttyMODUL485
                logger -t Hotplug Symlink from /dev/$DEVICENAME to /dev/ttyMODUL232 created
        fi
        if [ ${USBDEVICENAME} = '1-1.2:1.0' ] ; then
                ln -s /dev/$DEVICENAME /dev/ttyMODUL232
                logger -t Hotplug Symlink from /dev/$DEVICENAME to /dev/ttyMODUL485 created
        fi
        if [ ${USBDEVICENAME} = '1-1.3:1.2' ] ; then
                ln -s /dev/$DEVICENAME /dev/ttyMODULAT
                logger -t Hotplug Symlink from /dev/$DEVICENAME to /dev/ttyMODULAT created
        fi
}

[ "$SUBSYSTEM" == tty -a "$ACTION" == remove ] && {
        echo "$DEVICENAME" | grep 'ttyUSB' || exit 0
        USBDEVICENAME=$(echo $DEVPATH | cut -d '/' -f 8)
        if [ ${USBDEVICENAME} = '1-1.1:1.0' ] ; then
                 rm  /dev/ttyMODUL485
        fi
        if [ ${USBDEVICENAME} = '1-1.2:1.0' ] ; then
                rm  /dev/ttyMODUL232
        fi
        if [ ${USBDEVICENAME} = '1-1.3:1.2' ] ; then
                rm  /dev/ttyMODULAT
        fi
}
EOF
```

## 冷插拔
如果您注意到在openwrt 18.0.*版本中删除了udev和eudev，请不要担心，因为您仍然可以使这些东西有效。

**使用hotplug脚本作为coldplug**

你只需要注意ACTION环境变量，在启动时执行'bind'动作。
所以，只需将此选项添加到hotplug运行。 在这个例子里我用如下脚本:
```
cat << "EOF" > /etc/hotplug.d/usb/22-symlinks
# Description: Action executed on boot (bind) and with the system on the fly
if [ "$ACTION" = 'bind' ] ; then
  case "${PRODUCT}" in
    1bc7*) # Telit HE910 3g modules product id prefix
      DEVICE_NAME=$(ls /sys/$DEVPATH | grep tty)
      DEVICE_TTY=$(ls /sys/$DEVPATH/tty/)
      # Module Telit HE910-* connected to minipciexpress slot MAIN
      if [ ${DEVICENAME} = '1-1.3:1.0' ] ; then
        ln -s /dev/$DEVICE_TTY /dev/ttyMODULO1_DIAL
        logger -t Hotplug Symlink from /dev/$DEVICE_TTY to /dev/ttyMODULO1_DIAL created
      elif [ ${DEVICENAME} = '1-1.3:1.6' ] ; then
        ln -s /dev/$DEVICE_TTY /dev/ttyMODULO1_DATA
        logger -t Hotplug Symlink from /dev/$DEVICE_TTY to /dev/ttyMODULO1_DATA created
      # Module Telit HE910-* connected to minipciexpress slot SECONDARY
      elif [ ${DEVICENAME} = '1-1.2:1.0' ] ; then
        ln -s /dev/$DEVICE_TTY /dev/ttyMODULO2_DIAL
        logger -t Hotplug Symlink from /dev/$DEVICE_TTY to /dev/ttyMODULO2_DIAL created
      elif [ ${DEVICENAME} = '1-1.2:1.6' ] ; then
        ln -s /dev/$DEVICE_TTY /dev/ttyMODULO2_DATA
        logger -t Hotplug Symlink from /dev/$DEVICE_TTY to /dev/ttyMODULO2_DATA created
      fi
    ;;
  esac
fi
# Action to remove the symlinks
if [ "$ACTION" = 'remove' ]  ; then
  case "${PRODUCT}" in
    1bc7*)  # Telit HE910 3g modules product id prefix
     # Module Telit HE910-* connected to minipciexpress slot MAIN
      if [ ${DEVICENAME} = '1-1.3:1.0' ] ; then
        rm /dev/ttyMODULO1_DIAL
        logger -t Hotplug Symlink /dev/ttyMODULO1_DIAL removed
      elif [ ${DEVICENAME} = '1-1.3:1.6' ] ; then
        rm /dev/ttyMODULO1_DATA
        logger -t Hotplug Symlink /dev/ttyMODULO1_DATA removed
      # Module Telit HE910-* connected to minipciexpress slot SECONDARY
      elif [ ${DEVICENAME} = '1-1.2:1.0' ] ; then
        rm /dev/ttyMODULO2_DIAL
        logger -t Hotplug Symlink /dev/ttyMODULO2_DIAL removed
      elif [ ${DEVICENAME} = '1-1.2:1.6' ] ; then
        rm /dev/ttyMODULO2_DATA
        logger -t Hotplug Symlink /dev/ttyMODULO2_DATA removed
      fi
    ;;
  esac
fi
EOF
```