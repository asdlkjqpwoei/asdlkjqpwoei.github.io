---
title: Arch Linux Installation
categories: Linux
---
大学时玩多个Linux发行版，是Arch Linux比较符合我的胃口，虽然说入门相比其他发行版有点困难，在单个物理机上装了Windows和Arch Linux双系统，但是其中学到了一些东西。工作以后玩得少了，由于没有两台物理机，我也没什么必要用双系统了，因为多数时候还是在Windows平台上进行开发，换了新硬盘，不用Linux了，做个纪念。

# Installation Medium

可以用一个无重要数据的U盘作为启动盘，需要一个制作启动盘的工具，我用的是Rufus，在Arch Linux下载安装镜像后，用制作工具将U盘制作成启动盘。

# Boot the live environment

根据自己计算机主板的供应商品牌，找到进入BIOS或UEFI的办法，通常是在启动后按键盘上的某一个键（Del或者F9，具体需要参考说明书或操作手册）。进入后修改启动顺序，使U盘启动盘为第一个启动的设备，具体需要参考主板的说明书或者操作手册。
Arch Linux安装镜像不支持安全启动(Secure Boot)，在进入UEFI以后需要安全启动，通常默认为开启。

# Verify the boot mode

目前最新的引导启动模式为UEFI，旧的一般为BIOS，可通过命令行检查引导启动模式：`cat /sys/firmware/efi/fw_platform_size`，或者查阅主板的说明书或操作手册

如果返回值为64，那么引导启动模式为UEFI，并且是64位 x64 UEFI。如果返回值为32，那么引导启动模式为UEFI，并且是32位 x32 UEFI。如果提示文件不存在，那么启动模式可能为BIOS。

# Set the console keyboard layout and font

默认的命令行键盘布局为标准的美国键盘布局，对于我们来说一般不用动。

# Connet to the Internet

可以通过命令`ip link`查看网络接口状况。

有线网络通常不太需要关注网络连接问题，因为一般在进入安装环境后就有网络了，可以通过命令`dhcp`保证网络连接或者查错。

但是无线网络可能需要通过命令`iwctl`自行连接。

可以通过命令`ping <URL>`来检查网络是否畅通。

# Update the system lock

可通过命令`timedatectl`进行网络时间同步，保证操作系统时间正确。


# Partition the disk

操作系统识别到存储硬件后，就会为其分配一个块设备，如/dev/sda，可以通过命令行`lsblk`或`fdisk`来进行查看硬盘状况。

Arch Linux的安装环境提供了不同的分区管理工具，可以通过这些分区管理工具给硬盘格式化和分区。

1. [cfdisk](https://linux.die.net/man/8/cfdisk)，[Arch Manul cfdisk](https://man.archlinux.org/man/cfdisk.8)
2. [fdisk](https://man7.org/linux/man-pages/man8/fdisk.8.html)，[Arch Linux介绍fdisk](https://wiki.archlinux.org/title/Fdisk)

一个装载着操作系统的主要设备必须要有根分区，如果使用UEFI引导模式，那还必须得有一个EFI系统分区。


## 分区样例

* UEFI引导模式，GPT分区表
|Mount Point(挂载点)|Partition(分区)|Partition Type(分区类型)|
|---|---|---|
|/mnt/boot|/dev/efi_system_partition|EFI System partition|
|[SWAP]|/dev/swap_partition|Linux swap|
|/mnt|/dev/root_partition|Linux x86-64 root(/)|

* BIOS引导模式，MBR分区表
|Mount Point(挂载点)|Partition(分区)|Partition Type(分区类型)|
|---|---|---|
|[SWAP]|/dev/swap_partition|Linux swap|
|/mnt|/dev/root_partition|Linux|

# Format the partitions

可通过命令`mkfs`来创建新的文件系统，关于该文件系统的选择和命令行的使用，可以从官方Wiki[文件系统](https://wiki.archlinux.org/title/File_systems#Create_a_file_system)了解更多。

可通过命令`mkswap`来创建交换分区，用法`mkswap /dev/swap_partition`

# Mount the file systems

完成分区和格式化以后，就需要挂载设备，挂载点一般是/mnt。可通过命令`mount <block device identity> <mount point>`，例如`mount /dev/partition /mnt`。

# Select the mirrors

通过Linux的文本编辑器打开文件/etc/pacman.d/mirrorlist，根据其中的格式，更换为国内镜像源以得到更好的网络传输速率，已知环境中有的文本编辑器有nano、vi、vim。

可从Arch Linux官方社区的[镜像列表](https://archlinux.org/mirrors/)寻找你想要更换的镜像源。

# Install essential packages

`pacstrap -K /mnt base linux linux-firmware networkmanager dhcpcd netctl`

# Configure the system

`genfstab -U /mnt >> /mnt/etc/fstab`

# Chroot

`arch-chroot /mnt`

在执行完该命令后我们就处在新的操作系统环境中了，可以趁这个时候安装自己所需要的软件。

# Time zone

`ln -sf /usr/share/zoneinfo/<region>/<city> /etc/localtime`

一般来说设置在上海，所以是`ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime`

`hwclock --systohc`

# Localization

使用文本编辑器打开/etc/locale/gen，去除自己所需语言的注释，一般来说应该有en_US.UTF-8 UTF8、zh_CN.UTF-8 UTF-8、zh_TW.UTF-8 UTF-8，然后执行命令`locale-gen`来生成本地化文件。

创建或打开locale.config，写入内容`LANG=en_US.UTF-8`。

如果需要改变默认的命令行键盘布局，编辑/etc/vconsole.conf。

# Network configuration

创建或打开/etc/hostname，在文件中写入一个自行设定的主机名。

使用文本编辑器打开/etc/hosts，在文件中添加内容：

```
127.0.0.1 localhost
::1 localhost
127.0.1.1 <hostname>.localdomain <hostname>
```

# Root password

通过命令`passwd`设置root用户的密码。

# Boot loader

可以选择使用grub作为启动引导，成功后将会显示在自己的BIOS或者UEFI的启动项中。不同启动方式需要的软件包可能有所不同，这里只记录UEFI的情况。

除了grub，还需要安装efibootmgr、os-prober和ntfs-3g，这两个软件会寻找其他设备中的启动项入口，比如你是双系统能选择启动哪个系统。

关于grub的使用，可参阅

主要使用grub-install和grub-mkconfig，先进行grub-install安装，再用grub-mkconfig生成配置文件，安装命令行样例：`grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=grub`，生成配置文件命令行样例：`grub-mkconfig -o /boot/grub/grub.cfg`。

完成后可以重启进入安装完毕的新操作系统Arch Linux了。

## Note

可以通过查看文件/boot/grub/grub.cfg来检查是否生成了所有你需要的操作系统入口，如果找不到其他操作系统入口比如Windows，可能需要手动编辑该配置文件来添加操作系统入口。

如果硬件经历了一些变动，导致Arch Linux启动项消失，可以通过重新配置系统和重新配置启动引导来恢复正常。

# Add new user

可通过命令`useradd`来添加新的用户，例如：`useradd -m -G wheel <username>`

可通过命令`passwd`来给新用户设置密码，例如：`passwrd <username>`

# Graphic user interface

可选择安装Xorg提供图形服务，有关Xorg和图形处理器驱动的安装和相关详情，需阅读[Xorg](https://wiki.archlinux.org/title/Xorg#Driver_installation)

# Desktop environment

可从多个Linux[桌面环境](https://wiki.archlinux.org/title/Desktop_environment)中选择一个安装。

# Display Manager

可从多个[显示管理器](https://wiki.archlinux.org/title/Display_manager)中选择一个安装。

# Arch User Repository

AUR是Arch Linux的灵魂之一，大多数不存在官方镜像源中的软件都可以从这里获取，由于AUR的存在使得Arch Linux的软件生态比其他发行版更为健壮。

# Note

Arch Linux的官方社区相当活跃，Wiki也很齐全，大部分问题或者疑惑都可以通过查阅Wiki解答，也可以去社区Newbie模块发帖询问。当Arch Linux用多了以后也能给予一点点开源的贡献了，开源的魅力会逐渐显现。

（在Arch Linux上玩Minecraft时有需求过一个地图编辑器Amulet，发现缺少依赖，反馈了这个问题，发布者给它修了）

# Reference

1. [Installation guide - ArchWiki](https://wiki.archlinux.org/title/Installation_guide)
2. [以官方Wiki的方式安装ArchLinux - Viseator's blog](https://www.viseator.com/2017/05/17/arch_install/)