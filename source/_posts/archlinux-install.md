---
title: archlinux-install
date: 2025-05-23 10:42:48
tags:
---

# 英伟达驱动安装
无需下载私有驱动，使用arch源的软件包
```shell
# 使用linux-lts
pacman -S linux-lts

# 安装nvidia
pacman -S nvidia cuda
```

# thinkpad t490女装后无法grub
进入bios里，把`Boot Order Lock`关闭，然后执行grub-install和grub-mkconfig.
据说这个设定会组织对nvme的更改？
