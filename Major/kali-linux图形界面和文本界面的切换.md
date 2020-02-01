---
Title: kali linux图形界面和文本界面的切换
date: 2019-9-2
Tags: kali
---

kalilinux的图形界面和文本界面的切换
文件修改开机是否图形配置：

配置图行界面的文件是 `vi /etc/default/grub`
找到：GRUB_CMDLINE_LINUX_DEFAULT="quiet"
复制本行然后把`quiet`替换成`text`。
把本行注释掉（以免以后想改回来时不知道怎么改回来）。
保存后 执行`sudo update-grub`命令后 重启即可

如果想kali每次启动是文本模式可以修改如下文件：
`vi /etc/X11/default-display-manager`
把里面内容`/usr/sbin/gdm3`改为`false`之后重启会以文本模式登录，想改回图形就把false还原回`/usr/sbin/gdm3`

快捷键切换（推荐）：`ctrl+alt+F1`文本模式`ctrl+alt+F7`图形界面



Systemd是一种新的linux系统服务管理器。
它替换了init系统，能够管理系统的启动过程和一些系统服务，一旦启动起来，就将监管整个系统。
切换至字符界面：
	sudo systemctl set-default multi-user.target
切换至图形界面：
	sudo systemctlset-default graphical.target
打开图形界面：
sudo init 5