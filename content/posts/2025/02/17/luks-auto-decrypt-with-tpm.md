---
title: "使用TPM自动解密LUKS分区"
date: 2025-02-17T18:00:50+08:00
categories: ["Linux"]
tags: ["LUKS", "TPM2", "Linux", "systemd"]
draft: false
---

我的系统是Fedora Silverblue 41，只需要执行以下命令并输入LUKS分区的密码即可：

```sh
sudo systemd-cryptenroll --tpm2-device=auto --tpm2-pcrs=7,11,14 /PATH/TO/LUKS_DEVICE
```

systemd-cryptenroll的文档建议PCR使用7、11、14，这样不会因固件或操作系统的更新而需要频繁注册TPM，足以应对大多数情况，详见`man systemd-cryptenroll`。

运行`sudo systemd-cryptenroll /PATH/TO/LUKS_DEVICE`可以查看LUKS设备中写入的所有密码，如下所示：

```text
SLOT TYPE
   0 password
   2 tpm2
```

可以看到0号槽位是密码，2号槽位是TPM2设备。不建议删除0号槽的密码，防止因主板更新固件或其他情况导致TPM2设备数据被擦除导致无法解密。
