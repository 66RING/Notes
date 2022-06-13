---
title: 双显卡下GPU直通
author: 66RING
date: 2021-05-08
tags: 
- passthrough
mathjax: true
---

# Abstract


# Preface


# Overview

double graphic card only, since host have not card when isolate

- reboot and enable iommu in bios
	* like `intel-vd`, `amd-vi` search google
	* find and enable `IOMMU`
- enable iommu in grub
	* like `amd_iommu=on`
- `lspci -nn` search for device id that connect to your system
	* like `lspci -nn | grep 'NVIDIA'` => `[xxxx:xxxx]`
- pass the id over to grub
	* like `vfio-pci.ids=xxxx:xxxx, xxx2:xxx2`, graph card or sonud card what ever
	* TODO learn about vfio
	* **WARN** the `quiet` option may hide the message from passthrough
- rebuild grub `grub-mkconfig -o /boot/grub/grub.cfg` and reboot



- preventing the linux kernel from loading the driver for that card
	* TODO so you need to have **more than one GPU** in your system for host to running
- add ids of pass-through card into `/etc/modprobe.d/vfio.conf` file, if no exist just add it
	* `options vfio-pci ids=xxxx:xxxx, xxx1:xxx2`
	* TODO `softdep nvidia pre: vfio-pci` set proprietary driver if needed
- rebuild `initramfs`: `mkinitcpio -p linux`
- reboot and `lspci -k` to check device that passthrough is `Kernel driver in use: vfio-pci`



- get graphic card passthrough to VM 
	* make sure the firmware of your vm is uefi
- add hardware: `pic host device`
	* to find out which one is which using iommu group from `lspci`
* install virtio-win on your window VM
* install graphic card driver on your VM

