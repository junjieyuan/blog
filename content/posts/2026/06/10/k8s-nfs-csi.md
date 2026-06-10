---
title: "Kubernetes NFS CSI 踩坑：FCOS + NFSv4.2 的那些坑"
date: 2026-06-10T16:46:27+08:00
categories: ["Kubernetes"]
tags: ["NFS", "CSI", "FCOS"]
draft: false
---

家里的 K8s 集群跑在三台 Fedora CoreOS 虚拟机上，一直没搞持久化存储。这周末抽空搭了个 NFS 存储服务器，整个过程以为半小时搞定，结果断断续续折腾了一下午。NFSv4 的伪文件系统平时没怎么关注，这次真是被教做人了。

## 环境

- **存储服务器**：FCOS 虚拟机，IP `192.168.122.84`，hostname `storage-001.k8s.junjie.pro`
- **CSI driver**：[csi-driver-nfs](https://github.com/kubernetes-csi/csi-driver-nfs) v4.13.2，Helm 部署
- **目标**：动态 Provisioning，PVC 声明后自动在 `/var/nfs-data/` 下创建 `namespace/pvc-name` 子目录

## 架构

```
┌─────────────────────────────┐
│  k8s-storage-001 (FCOS VM)  │
│  /var/nfs-data/             │  ← fsid=0 → NFSv4 pseudoroot
│  ├─ default/                │
│  │  └─ nfs-pvc-test/        │  ← subDir 自动创建
│  └─ ...                     │
│  nfs-server (NFSv4.2)      │
└──────────────┬──────────────┘
               │ NFSv4.2, vers=4.2
┌──────────────┴──────────────┐
│  csi-driver-nfs             │
│  ├─ controller plugin       │  ← provisioning
│  ├─ node plugin (DaemonSet) │  ← mount/unmount
│  └─ StorageClass: nfs-csi   │
└─────────────────────────────┘
```

## 存储服务器配置

FCOS 是 immutable 系统，所有配置走 Butane / Ignition。这里有两个 systemd 单元比较关键。

第一个在首次启动时安装 `nfs-utils` 包。FCOS 用 `rpm-ostree` 分层安装系统级包：

```yaml
- name: rpm-ostree-install-packages.service
  enabled: true
  contents: |
    [Unit]
    Description=Install required packages via rpm-ostree
    After=network-online.target
    Wants=network-online.target
    [Service]
    Type=oneshot
    RemainAfterExit=yes
    ExecStart=/usr/bin/rpm-ostree install --reboot --idempotent --assumeyes nfs-utils
    [Install]
    WantedBy=multi-user.target
```

注意 `--reboot`——rpm-ostree 安装完需要重启才能生效，所以首次启动是：Ignition → 安装包 → 自动重启 → 系统起来后包才可用。

第二个单元在重启后启用 NFS 服务：

```yaml
- name: nfs-server-enable.service
  enabled: true
  contents: |
    [Unit]
    Description=Enable NFS server after packages are installed
    After=network-online.target rpm-ostree-install-packages.service
    Wants=network-online.target
    ConditionPathExists=/usr/sbin/rpc.nfsd
    [Service]
    Type=oneshot
    RemainAfterExit=yes
    ExecStart=/usr/bin/systemctl enable --now nfs-server
    [Install]
    WantedBy=multi-user.target
```

`ConditionPathExists=/usr/sbin/rpc.nfsd` 是精髓：第一次启动时 nfs-utils 还没分层，condition 不满足，单元跳过；重启后二进制存在，单元执行，enable 并 start nfs-server。之后每次启动都是空操作。

导出配置写 Ignition 阶段的 `storage.files`，扔到 `/etc/exports.d/k8s.exports`：

```
/var/nfs-data  192.168.122.0/24(rw,sync,no_root_squash,no_subtree_check,fsid=0)
```

`/var/nfs-data` 目录用 Butane 的 `storage.directories` 创建，不用自己写 `mkdir` 脚本。

## 部署 CSI Driver

Helm 一把梭，不赘述：

```bash
helm repo add csi-driver-nfs \
  https://raw.githubusercontent.com/kubernetes-csi/csi-driver-nfs/master/charts
helm upgrade --install csi-driver-nfs csi-driver-nfs/csi-driver-nfs \
  --namespace kube-system --create-namespace --version 4.13.2
```

StorageClass 初始版本长这样（后面会改很多次）：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
provisioner: nfs.csi.k8s.io
parameters:
  server: storage-001.k8s.junjie.pro
  share: /var/nfs-data
  subPathPattern: "${pvc.metadata.namespace}/${pvc.metadata.name}"
reclaimPolicy: Retain
mountOptions:
  - nfsvers=4.2
  - hard
  - rsize=524288
  - wsize=524288
```

然后建个测试 PVC：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc-test
spec:
  accessModes: [ReadWriteMany]
  storageClassName: nfs-csi
  resources:
    requests:
      storage: 1Gi
```

## 第一坑：subPathPattern 不存在

```bash
$ kubectl get pvc nfs-pvc-test
NAME           STATUS    VOLUME   CAPACITY   STORAGECLASS
nfs-pvc-test   Pending                      nfs-csi

$ kubectl describe pvc nfs-pvc-test
Events:
  Warning  ProvisioningFailed  ...
  rpc error: code = InvalidArgument desc = invalid parameter "subPathPattern" in storage class
```

`subPathPattern`？这参数名是我凭印象编的。翻 [csi-driver-nfs 的 driver-parameters 文档](https://github.com/kubernetes-csi/csi-driver-nfs/blob/master/docs/driver-parameters.md)，正确的参数叫 `subDir`，而且支持 `${pvc.metadata.namespace}` 和 `${pvc.metadata.name}` 这种变量替换，对的就是这两种，没有别的。

修正：

```yaml
parameters:
  subDir: "${pvc.metadata.namespace}/${pvc.metadata.name}"
```

StorageClass 的 `parameters` 字段不允许原地修改，得删了重建：

```bash
kubectl delete storageclass nfs-csi
kubectl apply -f storage-class.yaml
```

PVC 也要删了重建才能触发新的 provisioning：

```bash
kubectl delete pvc nfs-pvc-test --force --grace-period=0
# wait for finalizer...
kubectl apply -f nfs-pvc-test.yaml
```

## 第二坑：NFSv4.2 返回 "No such file or directory"

参数修好了，provisioner 不再报 `invalid parameter`，开始真正尝试挂载。然后又挂了：

```bash
$ kubectl describe pvc nfs-pvc-test
Events:
  Warning  ProvisioningFailed  ...
  Mounting command: mount
  Mounting arguments: -t nfs -o nfsvers=4.2,hard,rsize=524288,wsize=524288,... \
    storage-001.k8s.junjie.pro:/var/nfs-data /tmp/pvc-xxx
  Output: mount.nfs: mounting storage-001.k8s.junjie.pro:/var/nfs-data failed,
          reason given by server: No such file or directory
```

"reason given by server" 说明 NFS 服务器收到了请求，也给出了回复，但它说目录不存在。

但目录明明在啊：

```bash
$ ssh core@storage-001.k8s.junjie.pro ls /var/nfs-data/
a                                        # 之前手动建的测试目录
```

`showmount` 也正常：

```bash
$ showmount -e storage-001.k8s.junjie.pro
Export list for storage-001.k8s.junjie.pro:
/var/nfs-data 192.168.122.0/24
```

RPC 服务全在：

```bash
$ rpcinfo -p storage-001.k8s.junjie.pro | grep nfs
100003  3  tcp  2049  nfs
100003  4  tcp  2049  nfs             # NFSv3 和 v4 都注册了
```

## 第三坑：NFSv3 能通但不能用

换个思路，会不会是 NFSv4.2 的版本问题？降级到 NFSv3 试试。把 StorageClass 里的 `nfsvers` 改成 3：

```yaml
mountOptions:
  - nfsvers=3
```

重建 SC 和 PVC，这次不再是 "No such file"，而是：

```
Output: mount.nfs: rpc.statd is not running but is required for remote locking.
Mount.nfs: Either use '-o nolock' to keep locks local, or start statd.
```

NFSv3 需要 `rpc.statd` 做文件锁，但 CSI driver 跑在容器里，没有这个服务。

加上 `nolock` 能临时跑通，但我不想要 NFSv3——都什么年代了，NFSv4.2 的文件锁是协议内置的，不走 statd。所以这只能是 fallback，不能是最终方案。

不过通过这个测试确认了一件事：网络和 mountd 都是正常的，问题出在 NFSv4 协议层面的路径解析。

## 第四坑：手动 mount 的障眼法

我在 K8s 节点上手动 mount 了一把：

```bash
$ sudo mount -t nfs storage-001.k8s.junjie.pro:/var/nfs-data /mnt
# 没有报错，挂载成功？
```

这让我困惑了好一阵。CSI provisioner 在 pod 里挂着同样的参数却报错，手动 mount 却成功了？

查了下实际协商的版本：

```bash
$ findmnt -n -o FSTYPE,OPTIONS /mnt
nfs    ...,vers=3,...
```

果然，`mount -t nfs` 不带版本号时，客户端会自动协商：先试 4.2 → 4.1 → 4.0 → 回退到 3。NFSv4 失败后它默默地降级到 v3 了。CSI driver 在 mountOptions 里显式指定了 `nfsvers=4.2`，没有这个降级逻辑。

从这里可以确定：NFS 服务器配置有问题，**NFSv4 特定地挂了**，NFSv3 是通的。

## 根因：NFSv4 伪文件系统与 FCOS 的 /var

NFSv4 和 v3 最大的架构区别是，v4 有一个伪文件系统（pseudofilesystem）的概念。所有 NFSv4 导出都在一个虚拟的文件树上，客户端挂载时提供路径，服务端在伪文件树里查找。

默认配置下（不加 `fsid` 参数），NFS 服务器的伪文件树根是 `/`。当客户端请求挂载 `/var/nfs-data` 时，服务端在伪文件树里找 `/var/nfs-data`：

- `/` → 能找到，根当然在
- `/var` → **找不到**

为什么找不到 `/var`？在标准 Linux 发行版上，`/var` 就是文件系统上的一个普通目录，NFSv4 可以正常穿越。但在 Fedora CoreOS 上，`/var` 是一个独立的 bind mount（`/sysroot/var` → `/var`），NFSv4 的伪文件系统默认不穿越 mount 边界。路径在 `/var` 这里断了，后面 `/var/nfs-data` 自然也找不到。

这就是 "No such file or directory" 的来源——不是目录不存在，是 NFSv4 在伪文件树的路径解析里迷路了。

相关文档：[NFSv4 伪文件系统](https://www.man7.org/linux/man-pages/man5/exports.5.html)（`fsid` 参数部分）、[Linux NFS Wiki](https://wiki.linux-nfs.org/wiki/index.php/Main_Page)。

## 解决：fsid=0

`fsid=0` 把指定的导出路径设为 NFSv4 伪文件树的根节点。思路就是不让 NFSv4 去穿越 `/var`——直接让 `/var/nfs-data` 成为根：

```
/var/nfs-data  192.168.122.0/24(rw,sync,no_root_squash,no_subtree_check,fsid=0)
```

`exportfs -ra` 或重启 nfs-server 生效。手动验证：

```bash
# 注意：挂载路径变成了 / 而不是 /var/nfs-data
$ sudo mount -t nfs -o nfsvers=4.2 storage-001.k8s.junjie.pro:/ /mnt
$ findmnt -n -o FSTYPE,OPTIONS /mnt
nfs4   ...,vers=4.2,...
```

确认是 NFSv4.2。但是注意：**挂载路径从 `server:/var/nfs-data` 变成了 `server:/`**。这是因为伪文件树的根就是 `/var/nfs-data`，挂载 `/` 就自然对应到它。

## StorageClass 也要改

CSI driver 的 mount 路径是由 `share` 参数决定的。之前是：

```yaml
parameters:
  share: /var/nfs-data    # → mount server:/var/nfs-data → NFSv4 失败
```

现在 NFSv4 根变了，share 得跟着改：

```yaml
parameters:
  share: /                 # → mount server:/ → NFSv4 根 = /var/nfs-data
```

`subDir` 的逻辑不受影响——它是在 `share` 路径下创建 `namespace/pvc-name` 子目录，物理上还是在 `/var/nfs-data/` 下面：

```
服务端:  /var/nfs-data/default/nfs-pvc-test/
CSI挂载: server:/default/nfs-pvc-test           ← 相对于 share: /
```

删了 SC 和 PVC 重建：

```bash
kubectl delete storageclass nfs-csi
kubectl apply -f storage-class.yaml
kubectl delete pvc nfs-pvc-test --force --grace-period=0
kubectl apply -f nfs-pvc-test.yaml
```

```bash
$ kubectl get pvc nfs-pvc-test
NAME           STATUS   VOLUME   CAPACITY   STORAGECLASS
nfs-pvc-test   Bound    pvc-...  1Gi        nfs-csi
```

终于 Bound 了。确认挂载版本：

```bash
$ kubectl exec -n kube-system csi-nfs-controller-xxx -c nfs -- mount | grep nfs
storage-001.k8s.junjie.pro:/default/nfs-pvc-test on /var/lib/kubelet/...
type nfs4 (...,vers=4.2,rsize=524288,wsize=524288,hard,proto=tcp,timeo=600,retrans=2,...)
```

`type nfs4, vers=4.2`，就是它。

## 最终 StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-csi
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: nfs.csi.k8s.io
allowVolumeExpansion: true
parameters:
  server: storage-001.k8s.junjie.pro
  share: /
  subDir: "${pvc.metadata.namespace}/${pvc.metadata.name}"
reclaimPolicy: Retain
volumeBindingMode: Immediate
mountOptions:
  - nfsvers=4.2
  - hard
  - rsize=524288
  - wsize=524288
  - timeo=600
  - retrans=2
```

## 总结

| 现象 | 排查方向 | 根因 | 解法 |
|------|---------|------|------|
| `invalid parameter "subPathPattern"` | `kubectl describe pvc` | 参数名不存在，csi-driver-nfs 用 `subDir` | 改参数名 |
| `No such file or directory` from NFS server | `showmount -e` 正常，`rpcinfo` 正常 | FCOS 上 `/var` 是 bind mount，NFSv4 伪文件系统无法穿越 | `fsid=0` 将导出设为伪文件系统根 |
| 手动 mount 成功但 CSI 失败 | `findmnt` 看实际协商版本 | `mount -t nfs` 自动降级到 v3，CSI 显式指定 v4 | 修好 NFSv4 服务端配置 |
| NFSv3 `rpc.statd` missing | Pod 内无 statd 服务 | v3 文件锁依赖外部的 statd | 回 NFSv4（v4 锁内置于协议） |
| `fsid=0` 后路径变了 | 伪文件系统根变了，挂载路径也要变 | NFSv4 根 = `/var/nfs-data`，挂载路径是 `/` 而非 `/var/nfs-data` | SC 里 `share: /` |

整个过程最核心的一步是弄懂了 NFSv4 的伪文件系统——平时用 NFS 基本是 mount 一把梭，不出问题就不会去想底层。这次 FCOS 的 `/var` bind mount 刚好触发了这个边界条件。NFSv3 能通也是因为 v3 不走伪文件系统，直接用 MOUNT 协议请求实际文件路径。

参考链接：

- [csi-driver-nfs Driver Parameters](https://github.com/kubernetes-csi/csi-driver-nfs/blob/master/docs/driver-parameters.md)
- [Linux exports(5) — NFS 导出配置详解](https://www.man7.org/linux/man-pages/man5/exports.5.html)
- [Linux NFS Wiki](https://wiki.linux-nfs.org/wiki/index.php/Main_Page)
- [Fedora CoreOS 存储配置](https://docs.fedoraproject.org/en-US/fedora-coreos/storage/)
