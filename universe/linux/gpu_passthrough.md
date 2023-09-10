---
title: 显卡下GPU直通
author: 66RING
date: 2021-05-08
tags: 
- passthrough
mathjax: true
---

# 显卡直通

总结只要能用vfio把显卡隔离出来就算成功了, 剩下都是调试问题

## 双显卡直通

> [教程](https://ivonblog.com/posts/ubuntu-gpu-passthrough/)
>
> 需要注意有的CPU没有集成显卡, 如i513600kf, k代表超频, f代表没有核显

双显卡: CPU核显 + GPU

启用IOMMU, 如果双amd cpu则`amd_iommu=on`, 此时需要重启来获取设备组id以配置vfio

```
$ vim /etc/default/grub

GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet intel_iommu=on iommu=pt"
```

通过这个脚本可以获取设备所在组id:

```bash
#!/bin/bash
shopt -s nullglob
for g in /sys/kernel/iommu_groups/*; do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

启动vfio内核模块, `/etc/modprobe.d/vfio.conf`

```bash
# vfio.conf
options vfio-pci.ids=10de:2482,10de:228b
```

启动vfio内核模块, `/etc/modprobe.d/vfio-pci.conf`

```bash
# vfio-pci.conf
vfio-pci
```

initramfs或者mkinitcpio

```
update-initramft -u
# 或
mkinitcpio -p linux
```

屏蔽GPU驱动模块

```bash
$ sudo vim /etc/modprobe.d/blacklist.conf

blacklist nvidia
blacklist nouveau
```

进入BIOS启用核显(一般情况下有独显时主板会屏蔽掉核显)。这里按Del键可以进入BIOS。

检查VFIO状态, `sudo lspci -nnk`可以看到显卡用了vfio驱动


## 单显卡直通

1. 启用iommu, `intel_iommu=on iommu=pt`, amd的CPU就用`amd_iommu=on`

```
$ vim /etc/default/grub

GRUB_CMDLINE_LINUX_DEFAULT="loglevel=3 quiet intel_iommu=on iommu=pt"

$ grub-mkconfig -o /boot/grub/grub.cfg
```

2. 获取GPU所在组

```bash
#!/bin/bash
shopt -s nullglob
for g in /sys/kernel/iommu_groups/*; do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```

3. 启用vfio内核模块, archlinux中默认vfio已经嵌入linux了, 所以可以跳过

4. 配置启动脚本和恢复脚本, 主要就算启动虚拟机时关掉nvidia驱动, 换上vfio驱动做显卡隔离。而要关掉nvidia驱动则需要关闭所有正在使用的程序, 如窗口管理器。恢复则是换上nvidia驱动。

libvirt在启动和关闭虚拟机时会出发的钩子脚本`/etc/libvirt/hooks/qemu`

```bash
# qemu

OBJECT="$1"
OPERATION="$2"

if [[ $OBJECT == "win10" ]]; then
	case "$OPERATION" in
        	"prepare")
                systemctl start libvirt-nosleep@"$OBJECT"  2>&1 | tee -a /var/log/libvirt/custom_hooks.log
                /bin/vfio-startup.sh 2>&1 | tee -a /var/log/libvirt/custom_hooks.log
                ;;

            "release")
                systemctl stop libvirt-nosleep@"$OBJECT"  2>&1 | tee -a /var/log/libvirt/custom_hooks.log  
                /bin/vfio-teardown.sh 2>&1 | tee -a /var/log/libvirt/custom_hooks.log
                ;;
	esac
fi
```

启动脚本`vfio-startup.sh`

```bash
set -x

# 关闭窗口管理器, 如
# systemctl stop sdm.service
killall bspwm

# 放置数据竞争
sleep 4

# 关闭nvidia驱动, 以便后续能启动vfio
modprobe -r nvidia_drm
modprobe -r nvidia_uvm
modprobe -r nvidia_modeset
modprobe -r nvidia

# 关闭console连接, 之后在console中就看不到光标闪烁了, echo 1能恢复
echo 0 > /sys/class/vtconsole/vtcon0/bind
echo 0 > /sys/class/vtconsole/vtcon1/bind

# 有的GPU没有就注释掉
# echo efi-framebuffer.0 > /sys/bus/platform/drivers/efi-framebuffer/unbind

sleep 2

# detach GPU, 01_00_0是GPU对应的id, 如
# $ lspci -nnk
# 01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GA104 [GeForce RTX 3070 Ti] [10de:2482] (rev a1)
# 01:00.1 Audio device [0403]: NVIDIA Corporation GA104 High Definition Audio Controller [10de:228b] (rev a1)
virsh nodedev-detach pci_0000_01_00_0
virsh nodedev-detach pci_0000_01_00_1

# 启动vfio
modprobe vfio-pci
```

恢复脚本`vfio-startup.sh`, 如果不卸载vfio就可以挂在nvidia驱动的话就可以不卸载。

```bash
set -x

virsh nodedev-reattach pci_0000_01_00_0
virsh nodedev-reattach pci_0000_01_00_1

modprobe nvidia
modprobe nvidia_modeset
modprobe nvidia_uvm
modprobe nvidia_drm

echo 1 > /sys/class/vtconsole/vtcon0/bind
# 我这里echo 1到vtcon1会死机所以注释掉了, 问题不大
# echo 1 > /sys/class/vtconsole/vtcon1/bind

# echo efi-framebuffer.0 > /sys/bus/platform/drivers/efi-framebuffer/bind

# 重新启动窗口管理器
# systemctl start sdm.service
```

调试虚拟机, 虚拟机的宿主机开启ssh然后使用另一台机器连接调试, 因为在隔离掉显卡后会黑屏。先分别单独测试启动脚本`vfio-startup.sh`和恢复脚本`vfio-teardown.sh`的执行没有bug。

然后就可以在virtmanager中配置PCI直通了。开启虚拟机的vnc, 因为刚开始时没有显卡驱动虚拟机启动会黑屏(稍等久一点windows会自动安装驱动, 不过我们通过vnc连接手动安装)。不需要配置vbios, 因为我们可以通过vnc手动安装显卡驱动。


### trouble shooting

1. 不使用窗口管理器的话(即使用`startx`), 虚拟机关闭触发完恢复脚本可能仍然黑屏, 但其实键盘已经可以操作了, 盲打`startx`即可恢复
2. 进入虚拟机鼠标用不了, 配置鼠标usb的直通即可

