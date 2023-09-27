在 [[LS104x] 使用ostree更新rootfs](https://github.com/carloscn/blog/issues/194#top) 中我们在ramdisk中使用ostree的checkout功能还原出一个完整版本rootfs，但是这个方法并没有完全彻底的使用ostree的部署功能。本节将要总结和整理一个ostree的部署功能。类似于该demo：[YOUTUBE - Designing OSTree based embedded Linux systems with the Yocto Project](https://www.youtube.com/watch?v=3i48NbAS2jUube)

从ubuntu更新为busybox，更新原理如图所示：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309270931825.png)

Note，这些ostree的部署的逻辑被包含在了meta-updater里面， https://github.com/advancedtelematic/meta-updater/tree/master 。但metaupdater集成度很高，而且该仓库已经很久没有更新了，就导致和新版的yocto集成有很多问题，需要不少的工作量。因此，本文手动部署ostree，并且参考meta-updater里面的逻辑手动补充一些脚本来完成ostree的部署和ostree部署的rootfs的使用。


# 1. 服务器端配置

本文使用HTTP而不是HTTPS，并且服务器使用本机HOST局域网更新。

## 1.1 仓库配置

- 需要进行ostree仓库的初始化
- 需要在ostree仓库中增加busybox的rootfs文件
- **对rootfs文件进行部署标准化改造**
- 需要commit仓库变化
- 需要生成summary文件
- 需要启动服务器监听程序

创建初始化仓库：

`mkdir -p repo && ostree --repo=repo init --mode=archive-z2 && mkdir -p rootfs`

- rootfs文件为存放busybox rootfs文件的路径；
- repo文件夹为仓库的配置文件；

### 1.1.1 rootfs改造

需要把rootfs下面根目录的etc转移到/usr/下面：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309271112368.png)

> The deployment should not have a traditional UNIX `/etc`; instead, it should include `/usr/etc`. This is the “default configuration”. When OSTree creates a deployment, it performs a 3-way merge using the _old_ default configuration, the active system’s `/etc`, and the new default configuration. In the final filesystem tree for a deployment then, `/etc` is a regular writable directory.
> https://ostreedev.github.io/ostree/deployment/

### 1.1.2 kernel文件

除此之外，还需要创建kernel、devicetree和initramfs的文件到下面的文件路径：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309271117761.png)

命名规则如下所示：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309271117320.png)

Note，不要使用软链接符号。

**没有以上两步，ostree无法在板子上部署**。

## 1.2 仓库上传

可能有的需要sudo命令：

`ostree --repo=repo commit --branch=master --subject="image v1 (ls104x) rootfs)" rootfs`


不需要sudo：

`ostree log master --repo=repo`

`ostree summary --repo=repo ./repo/branch.summary -u`

使用服务器监听repo：

`python3 -m http.server 8000 --bind 10.10.192.121 --directory repo`

# 2. 客户端配置（板级）

## 2.1 初始化版本的rootfs

我们需要准备一个能够正常启动，包括ostree和网络功能的的rootfs，作为基本的rootfs。然后我们依赖于这个rootfs，进入Linux runtime对ostree进行部署。

在我的demo中使用的是ubuntu：（当然ubuntu会很大，可以采用一些轻量的rootfs）

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309271121771.png)

使用这个版本进入正常的Linux Runtime，接下来我们要在板子上部署ostree。

## 2.2 部署ostree

部署之前请检查你的板子：
* 联网功能（包括ip是否正确）
* 时间正确
* 足够的空间

### 2.2.1 创建标准的sysroot

进入板子的根目录`cd /` 并且创建一个`sysroot`文件夹 `mkdir sysroot`。

初始化一个标准的sysroot文件夹：

`ostree admin init-fs sysroot`

你可以得到一个：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309271126980.png)

### 2.2.2 增加远程repo

给仓库增加

`ostree remote add --repo=/sysroot/ostree/repo --no-gpg-verify origin http://10.10.192.121:8000/`

从远程仓库拉去数据：

`ostree pull --repo=/sysroot/ostree/repo origin:master`

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309271129072.png)

`ostree log --repo=/sysroot/ostree/repo origin:master`

### 2.2.3 部署仓库

`ostree admin --sysroot=sysroot --os=tcu deploy origin:master`

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309271131277.png)

查看部署情况： `ostree admin --sysroot=sysroot status`

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309271131638.png)

部署之后的文件：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309271132330.png)

在`/sysroot/ostree/deploy/tcu/deploy`目录就可以看到：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309271133880.png)

upgrade：

`ostree admin --sysroot=sysroot --os=tcu upgrade`

通过这个命令，可以看到，ostree把最新的部署rootfs中的etc从/usr/etc拿了出来：

![](https://raw.githubusercontent.com/carloscn/images/main/typora202309271343539.png)

之后可以通过一些脚本，在部署完成之后，拿到rootfs的地址，然后在ramdisk里面配合一些脚本解析进行切换。





