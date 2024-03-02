---
title: "Linux开启IP转发"
date: 2024-03-02T17:38:00+08:00
draft: false
---

# 临时启用

系统重启后就会失效：

```bash
# IPv6
sudo sysctl -w net.ipv6.conf.all.forwarding=1

# IPv4
sudo sysctl -w net.ipv4.ip_forward=1
```

# 永久启用

修改`/etc/sysctl.conf`或`/etc/sysctl.d/`目录中的配置文件，更改已有的配置或在文件中添加如下内容：

```conf
# IPv6
net.ipv6.conf.all.forwarding=1

# IPv4
net.ipv4.ip_forward=1
```

加载配置：

```bash
sudo sysctl -p /etc/sysctl.conf
```