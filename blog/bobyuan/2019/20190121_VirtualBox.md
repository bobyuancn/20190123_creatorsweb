---
title: VirtualBox
date: 2019-01-21
description: 使用 Oracle VirtualBox 的一些心得体会。包括：共享文件夹、网络连接方式、Portable-VirtualBox、增加磁盘空间、VirtualBox v6.0 等。
---

# VirtualBox

对于个人或非商业使用，免费的桌面端虚拟机软件有以下几个选择：

1. Windows Virtual PC，免费，而且可以安装免费的 Windows XP 虚拟机。
   仅适用于 Windows 7 操作系统，不支持 Windows 10。
   官方下载链接: [中文](https://www.microsoft.com/zh-CN/download/details.aspx?id=3702)  | [英文](https://www.microsoft.com/en-us/download/details.aspx?id=3702)
   
2. Hyper-V on Windows 10，免费。
     仅适用于 Windows 10 专业版、企业版、教育版。不支持家庭版（Home）。
     官方介绍链接：[Install Hyper-V on Windows 10](https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/quick-start/enable-hyper-v) 
     
3. Oracle VirtualBox，开源免费，支持 Windows、MacOS X、Linux 等多个操作系统。
     官方下载链接: https://www.virtualbox.org
     
4. VMware Workstation Player，免费，
     官方下载链接：<https://www.vmware.com/products/workstation-player.html>


关于 VirtualBox 和 VMware 的比较，可以参考以下的文章：

* [VMware vs. VirtualBox: Which is Better for Desktop Virtualization?](https://technologyadvice.com/blog/information-technology/vmware-vs-virtualbox/)
* [VirtualBox or VMWare: Which is best for you?](http://techgenix.com/virtualbox-vmware-compared/)


相对而言，VirtualBox 因开源的缘故，功能性、可玩性更胜一筹，也是我喜欢它的原因。例如，它比VMWare 多具备了2个很实用，且是 VMWare 收费版才具备的功能：

1. 建立快照（Snapshots）功能。可以将虚拟机的当前状态建立一个快照，在虚拟机上面进行各种操作后，可以很方便快速地回滚到之前存储的快照，好像什么都没有发生一样。对于在虚拟机上安装测试第三方软件很有用。

![Snapshots](img/20190121/VirtualBox_Snapshots.png)

2. 共享文件夹（Shared Folders）功能。可以在虚拟机上 Mount 一个网络文件夹，指向宿主机的某个文件夹，方便宿主机和虚拟机之间文件交换。注意，此功能必须要在虚拟机内安装“Guest Additions”才能使用。

![Shared Folders](img/20190121/VirtualBox_SharedFolders_vmshare.png)


本文将描述一下我使用 VirtualBox 的几个功能，供有同样兴趣的读者参考。


[[toc]]

----



## 共享文件夹

如前所述，共享文件夹（Shared Folders）需要依赖于“Guest Additions”，因此必须在虚拟机上先安装它。安装步骤如下：

启动虚拟机，点选菜单“Devices | Insert Guest Additions CD image...”来将此 CD 镜像放到虚拟机的光驱里。

![Insert GuestAdditions CD](img/20190121/VirtualBox_Insert_GuestAddition_CD.png)

如果虚拟机是 Windows 操作系统，在资源浏览器里找到此光驱，打开并安装即可。安装完成后，对光驱点右键，选择弹出此 CD 镜像。虚拟机重启生效。

如果虚拟机是 Linux 操作系统，需要以 root 用户先挂载 CD-ROM 再安装。

```shell
# mount the CD image at first.

cd /mnt
mkdir cdrom 
mount /dev/cdrom /mnt/cdrom

# run the installer.
cd /mnt/cdrom
./VBoxLinuxAdditions.run

# if everything is OK, reboot to complete the installation.
reboot
```

在本例中，虚拟机的共享文件夹是这样设置的。共享文件夹名称为“vmshare”，它是宿主机上的文件夹“D:\VirtualBox_VMs\vmshare”，勾择复选框“Auto-mount”和“Make Permanent”。

![VirtualBox Shared Folders Setting](img/20190121/VirtualBox_SharedFolders_Settings.png)


重启生效后，此共享文件夹将被挂载到“/media/sf_vmshare”，只有 root 用户才有权限访问。我们观察到此路径属于“vboxsf”组：

```
bobyuan@ubuntuvm1:~$ ls -l /media
total 8
drwxrwx--- 1 root vboxsf 8192 Jan 15 08:21 sf_vmshare
```

为了让当前用户“bobyuan”能够访问此文件夹，需要将“bobyuan”也加入到“vboxsf”组内。

```
# add user to group.
sudo usermod -aG vboxsf bobyuan
```

重新登录后，通过“groups”命令检查一下，确保自己（即“bobyuan”用户）已经是“vboxsf”组的成员。至此，即可以顺利读写此共享文件夹了。



## 网络连接方式

网络连接方式常用的有2种，即桥接模式（Bridged Adapter）和网络地址转换模式（NAT）。其中，桥接模式最简单，它让虚拟机更像是一台独立的机器，虚拟机的网卡直接连接物理网络。这种情况下，我们在虚拟机里面查得它的 IP 地址，就可以在当前网络上直接访问。

另一种是 NAT 网络连接模式，相对复杂些，虚拟机的网卡会分配一个内部网址，而这个内部地址是在当前网络上无法直接访问的。为了访问 NAT 网络连接模式的虚拟机，必须通过端口映射（Port Forwarding）。

首先确保 NAT 网络连接的设置如下图。请注意选择 Adapter Type 为 “Paravirtualized Network (virtio-net)”：

![NAT PortForwarding](img/20190121/VirtualBox_PortForwarding.png)

按“Port Forwarding”按钮，在弹出的对话框里面输入规则。例如，下面增加了一条（也可以多条），将宿主机的“10022”端口，映射到虚拟机的“22”端口，我们用“SSH”来命名此规则：

![NAT PortForwarding Rules](img/20190121/VirtualBox_PortForwarding_Rules.png)

若用 Putty 来连接此服务器，可以这样输入：

![Putty Connection](img/20190121/Putty_Connection.png)

虚拟机上每个需要暴露给外界的端口，都需要在宿主机上设一个端口用于跳转。因此，需要保证所选的端口号在宿主机上未被占用，以免端口冲突。

等等，这就完了吗？没遗漏什么吧？事实上就是这些，任务已经完成了。细心的读者可能已经发现，这里并未输入虚拟机的 IPv4 地址（在例子中是 10.0.2.15），它不需要。



## Portable VirtualBox

[Portable-VirtualBox](http://www.vbox.me/) 是一个非官方的个人作品，能将 VirtualBox 作为移动模式，安装在 USB 移动硬盘上。

它好处显而易见。虚拟机通常都很占磁盘空间，而且通常情况下会有多个虚拟机，因此占用几十或上百 GB 的空间是很轻松平常的。将虚拟机转移到移动硬盘上，可以很大程度缓解电脑有限的磁盘空间的占用。更别说我们可以用多个移动硬盘了。

我经常是这样使用它的：

1. Windows 宿主机事先已经安装了 VirtualBox，但不要启动安装的 VirtualBox 图形界面。将安装了 Portable-VirtualBox 的移动硬盘接上，在宿主机上“磁盘管理”中设置指定此移动硬盘使用“V”盘符（即VirtualBox的首字符，您也可以选择其它盘符），目的是锁定移动硬盘的盘符。这样上设置后，每次此移动硬盘接驳上后，将固定作为 V 盘访问。
2. 在移动硬盘上启动 Portable-VirtualBox，稍等片刻等图形界面出现，在设置中配置虚拟机的存放位置，共享文件夹的位置等，保存这些设置。
3. 在移动硬盘上启动 Portable-VirtualBox 的图形界面里使用虚拟机。

要退出使用时，需先关闭全部虚拟机，再关闭 Portable-VirtualBox 图形界面。最后弹出此移动硬盘。

这种使用模式也有需要注意的地方：

1. 保持宿主机上安装的 VirtualBox 和移动硬盘上安装的版本一致。如果宿主机升级了，也要保证所有移动硬盘上的安装升级到同样的版本。升级 VirtualBox 也同时会升级虚拟机里安装的 “Guest Additions”。
2. 移动硬盘最好是高速接驳，例如 USB 3.0 或更高速的连接方式。还可以考虑使用固态的移动硬盘。
3. 注意保持连接的稳定，不能在使用过程中中断，否则有可能导致整个虚拟机的存储文件损坏，造成数据丢失。

鉴于上述第3点的风险，使用这种方式请谨慎。



## 增加硬盘空间

新建虚拟机的时候，有一项设置是预设虚拟磁盘空间。若在后期使用中发现之前设定的磁盘空间不够，想扩容，可以使用以下方法。

1. 添加一块新的虚拟硬盘，挂载到虚拟机上成为第二块硬盘。
2. 先备份虚拟机，可以选择文件复制的方式来备份。在“File | Virtual Disk Manager...”里扩大虚拟磁盘“.vdi”的存储空间，然后启动虚拟机，在虚拟机内用磁盘管理工具将未分配的磁盘空间利用上。

上述第1种方式最简单，适用于之前的虚拟硬盘尚可以安装操作系统，需要扩充数据存储空间的情况。即在不改变之前服役的虚拟硬盘前提下，增加额外的（可以多块）虚拟硬盘来扩大数据存储空间。

第2种方式较复杂，适用于之前的虚拟硬盘已经无法安装操作系统的情况。这在有些情况下是做不到的（例如之前硬盘的分区方式无法扩容），且有一些技术上的挑战。您可以参考：

* [How do I increase the hard disk size of the virtual machine?](https://askubuntu.com/questions/88647/how-do-i-increase-the-hard-disk-size-of-the-virtual-machine)



## VritualBox 6.0

在 2018-12-18 日，新的大版本 Oracle VirtualBox 6.0 发布了。新版本带来了多个重要的功能增强：

*  Graphics: major update of 3D graphics support for Windows guests,  and VMSVGA 3D graphics device emulation on Linux and Solaris guests 

* Added support for using Hyper-V as the fallback execution core  on Windows host, to avoid inability to run VMs at the price of reduced  performance In addition.

不仅仅是界面上，性能上也将较前一版本 5.x 有较大改进。不到一个月后，v6.0.2 在 2019-1-15 日发布了，官方网页也将 v6.x 作为主推的版本。

一般情况下，新的大版本可能有较多不稳定因素，从更新日志也可以看出，新版本的更新总是有较多的缺陷修复记录。

从我目前的使用情况来看，新版的运行情况良好，界面有一些变化，但也很快适应了。



## 结语

以上是我使用 Oracle VirtualBox 的一些心得体会，您是怎样使用 VirtualBox 呢？如果您对本文有什么好的建议或意见，欢迎给我写电子邮件。

