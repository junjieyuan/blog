---
title: "把英伟达显卡直通给Linux虚拟机"
date: 2024-12-22T21:49:15+08:00
draft: false
---

我的电脑的CPU是AMD锐龙9 7950X，带有核显，独显是英伟达RTX 4080，宿主机系统是Fedora Silverblue 41。

由于Fedora不会在给Linux源码树外的英伟达显卡驱动签名，所以自己安装驱动的话需要关闭UEFI安全启动或在UEFI中加入自己的密钥，不然会无法启动系统。但我不打算改宿主机的UEFI，因为会降低安全性。我选择把显卡直通到虚拟机，关闭虚拟机的UEFI安全启动，由于我的宿主机的硬盘是用LUKS2加密的，也不会有额外的安全风险。

# 宿主机配置

注意，配置完成后需要重启系统。

## 修改内核参数

需要开启IOMMU、VFIO并屏蔽宿主机的英伟达显卡和其开源驱动Nouveau。

注意事项：

* AMD和英特尔的IOMMU参数不一样；
* 需要把`vfio-pci.ids`参数的显卡PCI ID替换成自己的，可以用`lspci`查看，中间用半角逗号分隔。

```bash
rpm-ostree kargs --append-if-missing="amd_iommu=on" --append-if-missing="iommu=pt" --append-if-missing="rd.driver.pre=vfio_pci" --append-if-missing=rd.driver.blacklist=nouveau --append-if-missing=modprobe.blacklist=nouveau --append-if-missing="vfio-pci.ids=XXXX,YYYY"
```

## 修改initramfs参数

添加VFIO模块：

```bash
rpm-ostree initramfs --enable --arg="--add-drivers" --arg="vfio vfio_iommu_type1 vfio_pci"
```

# 虚拟机配置


我用的虚拟机管理工具是[virt-manager](https://virt-manager.org/)，虚拟机系统是Fedora Linux Server 41。

在创建虚拟机时需要添加显卡对应的PCI设备，在我的系统上的名字是“NVIDIA Corporation AD103 [GeForce RTX 4080]“和“NVIDIA Corporation”：

![给虚拟机添加显卡。为保护隐私，遮住了所有PCI设备信息。](https://blog-files.junjie.pro/images/a046ef14-227f-4adc-9431-29d89db0a4aa)

为了方便，我直接关闭了虚拟机的UEFI安全启动，安装系统后可以参考[RPM Fusion的英伟达驱动文档](https://rpmfusion.org/Howto/NVIDIA)安装驱动。

# 参考文献

1. [Configuring GPU Pass-Through for NVIDIA cards](https://doc.opensuse.org/documentation/leap/virtualization/html/book-virtualization/app-gpu-passthru.html)
