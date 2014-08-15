# CoreOS 开发者指南

这些都是构建CoreOS本身的说明. 最终，将指导你构建一个运行在KVM并且使你能够修改代码的开发者镜像.

CoreOS是一个开源项目.所有的源码都托管在[github][github-coreos].如果你发现这些文档和代码有问题，请发送一个pull请求.

你可以在 [IRC 通道][irc] 或者 [邮件列表][coreos-dev]直接提问.

[github-coreos]: https://github.com/coreos/
[irc]: irc://irc.freenode.org:6667/#coreos
[coreos-dev]: https://groups.google.com/forum/#!forum/coreos-dev

## 入门

让我们设置一个SDK的chroot并建立一个可启动的CoreOS镜像.
SDK chroot 有一个完整的工具链和从宽松的不同系统间构建进程得到隔离. SDK 必须运行在64位的Linux机器上,
任何Linux发行版都可([Ubuntu][ubuntu-link], [Fedora][fedora-link],等).
[ubuntu-link]:http://www.ubuntu.com
[fedora-link]:http://www.fedoraproject.org  

### 先决条件

系统先决条件入门:

- curl
- git

你还需要设置git:

```  

git config --global user.email "you@example.com"  
git config --global user.name "Your Name"  
  
```

**注意**: 配置git普通用户权限就够了，没有必要用超级管理员.

### 安装 depot_tools

`depot_tools`其中的一个是`repo`, 帮助我们收集制作CoreOS的git资源库. Pull down 代码并添加到你的路径:

```  
  
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git  
export PATH="$PATH":`pwd`/depot_tools  
  
```

你可以将此添加到`.bashrc`或者`/etc/profile.d/` 当你每次打开一个新的终端的时候，你就不用再重新手动设置`$PATH`.

### 引导 SDK chroot

创建一个工程目录. 这里将防止你的所有的git资源库和SDK chroot.拥有足够的空间是必须的.

```
mkdir coreos; cd coreos
```

使用描述开始所必须的git资源库的描述清单初始化.repo文件夹.

```
repo init -u https://github.com/coreos/manifest.git -g minilayout --repo-url  https://git.chromium.org/git/external/repo.git
```

根据清单同步所有需要的git资源库.

```
repo sync
```

### 构建一个镜像

下载并且进入SDK chroot,它包含了所有的编译器和工具. 


```
./chromite/bin/cros_sdk
```

**警告:** 如何你需要删除SDK chroot，请使用
`./chromite/bin/cros_sdk --delete`. 否则, 你将删除 `/dev`
被绑定挂载在chroot的条目.

设置 "core" 用户的密码.

```
./set_shared_user_password.sh
```

amd64-generic 通用镜像:

```
echo amd64-generic > .default_board
```

在/build/${BOARD}中，设置一个新的额根文件系统:

```
./setup_board
```

建立所有目标的二进制包:

```
./build_packages
```

构建一个内置的，覆盖开发之间的镜像:

```
./build_image --noenable_rootfs_verification dev
```

运行`image_to_vm.sh`命令,在这些命令完成后，这些将被转换为原型的可引导的虚拟机被打印.

### 启动

当你构建了一个镜像，你可以使用KVM运行他(运行`image_to_vm.sh`将打印出指令说明).

演示的大致方向是我们在现在的系统启动两个小的守护进程，使你能够通过HTTP的接口进行访问.首先，
systemd-rest,允许你通过HTTP停止或启动服务单元.另一种是可以随时使用和随时关闭的小型服务器.你可以试试这些守护进程:  

```
curl http://127.0.0.1:8000  
curl http://127.0.0.1:8080/units/motd-http.service/stop/replace  
curl http://127.0.0.1:8000  
curl http://127.0.0.1:8080/units/motd-http.service/start/replace  
```

## 改变

### git 和 repo

`repo`管理CoreOS. 它是专门为Android项目构建并且使管理大量的git仓库更加容易,以下是博客公告:

> repo 工具使用一种基于XML的清单文件描述上游的资料库, 以及如何合并他们到一个单独的工作台.
>. repo将递归所有的git子树并且处理上传，拉和其他需要的项目. repo 有内建的知识主题分支并且使他们成为工作流必不可少的一部分.
> -- 引用 [谷歌开源博客][repo-blog]

[repo-blog]: http://google-opensource.blogspot.com/2008/11/gerrit-and-repo-android-source.html

要知道repo完全使用手册，访问 [使用Repo and Git进行版本管理][vc-repo-git].

[vc-repo-git]: http://source.android.com/source/version-control.html

### repo 更新清单

CoreOS lives的repo清单在git仓库`.repo/manifests`中. 如果你需要更新这个清单，在这个目录中编辑文件`default.xml`.

`repo` 使用的一个名为 'default'分支 来跟踪你在`repo init`中指定的上游分支, 默认是'origin/master'. 在改变的时候一定要记住这一点, 开始的git版本库应该没有一个'default'分支.

## 开发流程

### 在一个镜像上更新软件包

构建一个新的虚拟机镜像是一个很耗时的一个过程. 在开发镜像上你可以使用`gmerge`在你的工作站来构建包并且在你的目标虚拟上传送他们.

1. 在你的工作站上SDK chroot内启动开发服务器:

```
start_devserver --port 8080
```

注意: 此端口将需要互联网访问

2. 从你的虚拟机运行/usr/local/bin/gmerge 并且确保在`/etc/lsb-release`中的设置指向你的工作站的IP/主机名称和端口

```
/usr/local/bin/gmerge coreos-base/update_engine
```

### 使用更新引擎更新镜像

如果你想测试一个你构建能够成功更新的运行的虚拟主机，你可以在开发服务器使用`--image`参数. 这里有个实例:

```
start_devserver --image ../build/images/amd64-generic/latest/chromiumos_image.bin
```

从目标虚拟机器你运行:

```
update_engine_client -update -omaha_url http://$WORKSTATION_HOSTNAME:8080/update
```

如果更新失败，你可以通过运行下面命令检查更新引擎的日志:

```
journalctl -u update-engine -o cat
```

如果你想下载另外一个更新，你可能需要清除重新启动挂起状态:

```
update_engine_client -reset_status
```

### 从Gentoo更新portage-stable安装程序

这是一种被称为`update_ebuilds`使用脚本 他能够从Gentoo's
CVS 树直接拉到你本地的portage-stable 树. 下面是一个例子的用法
替换到最新版本:

```
./update_ebuilds --commit dev-lang/go
```

创建一个Pull请求后，立即运行:

```
cd ~/trunk/src/third_party/portage-stable
git checkout -b 'bump-go'
git push <your remote> bump-go
```

## 产品工作流

### 构建一个产品镜像

这将建立一个能够在KVM下运行并且能够使用附近产品参数的镜像.

注意: 如果你创建一个正式的发行版，你在这里添加 `COREOS_OFFICIAL=1` . 这将更改版本，默认情况下启用上载.

```
./build_image prod
```

生成的产品镜像能够被qemu引导，但是对于一个大的正式的分区或者VMware镜像使用`image_to_vm.sh` 在`build_image1`终端作为描述.

### 推送更新到 dev-channel

#### 手动构建

推送一个更新到开发通道，通过在api.core-os.net按照上面描述的构建一个产品镜像，然后使用以下工具:

```
COREOS_OFFICIAL=1 ./core_upload_update <required flags> --track dev-channel --image ../build/images/amd64-generic/latest/coreos_production_image.bin
```

#### 自动构建

自动构建主机没有权限生产签名密匙，因此最终的签名和推送到api.core-os.net必须在别处进行.
`au-generator.zip` 存档提供了所需的工具，因此一个完成的SDK设置不是必须的. 这里需要安装和配置gsutil.

```
URL=gs://storage.core-os.net/coreos/amd64-generic/0000.0.0
cd $(mktemp -d)
gsutil cp $URL/au-generator.zip $URL/coreos_production_image.bin.bz2 ./
unzip au-generator.zip
bunzip2 coreos_production_image.bin.bz2
COREOS_OFFICIAL=1 ./core_upload_update <required flags> --track dev-channel --image coreos_production_image.bin
```

## 技巧和诀窍

### 查找所有打开的pull请求和问题

- [CoreOS 问题][issues]
- [CoreOS Pull 请求][pullrequests]

[issues]: https://github.com/organizations/coreos/dashboard/issues/
[pullrequests]: https://github.com/organizations/coreos/dashboard/pulls/

### 检索所有的 repo 代码

使用 `repo forall` 你可以同时检索所有的git repos:

```
repo forall -c  git grep 'CONFIG_EXTRA_FIRMWARE_DIR'
```

### 缓存git的https密码

注意: 你需要 git 1.7.10+ 使用凭证助手

开启credential helper 并且 git 将会一段时间内在内存中保存你的密码:

```
git config --global credential.helper cache
```

CoreOS为什么在git远程仓库中使用SSH? 因为, 我们不允许匿名者使用SSH地址从github上克隆项目. 在将来我们将修复这个问题.

### 基础系统依赖图

给一个试图深入到基础系统将包含那些和为什么包含这些emerge tree试图的东西:

```
emerge-amd64-generic  --emptytree  -p -v --tree  coreos-base/coreos-dev
```

### SSH 配置

你将启动大量的虚拟机并且生成ssh密匙. 添加这个到你的`$HOME/.ssh/config` 停止烦人的指纹警告.

```
Host 127.0.0.1
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
  User core
  LogLevel QUIET
```

### 从桌面环境隐藏loop devices

默认的桌面环境将显示任何挂在的设备，包括用于构建CoreOS磁盘镜像的loop devices. 如果守护进程作为``udisks``负责这些产生，那么你可以使用接下来的udev规则禁用这个行为:

```
echo 'SUBSYSTEM=="block", KERNEL=="ram*|loop*", ENV{UDISKS_PRESENTATION_HIDE}="1", ENV{UDISKS_PRESENTATION_NOPOLICY}="1"' > /etc/udev/rules.d/85-hide-loop.rules
udevadm control --reload
```

### 离开开发模式

有些守护进程在"dev mode"中实际不一样. 例如 update_engine 拒绝自动更新或者链接到https. 如果你需要在虚拟机上的dev_mod测试出一些东西，你可以参照下面的方法:

```
mv /root/.dev_mode{,.old}
```

如果你想永久的离开，你可以运行:

```
crossystem disable_dev_request=1; reboot
```

## 已知问题

### coreos-base构建包失败

一些时间 coreos-dev 或者 coreos 构建在`build_packages`中会失败 使用一个回溯指向`epoll`. 这没有被追踪但是再次运行`build_packages`将修复它. 错误看起来像这样:

```
Packages failed:
coreos-base/coreos-dev-0.1.0-r63
coreos-base/coreos-0.0.1-r187
```

## 常量 和 IDs

### CoreOS 应用 ID

此UUID用来识别在任何地方识别CoreOS的更新服务.

```
e96281a6-d1af-4bde-9a0a-97b76e56dc57
```

### GPT UUID 类型

- CoreOS Root: 5dfbf5f4-2848-4bac-aa5e-0d9a20b745a6
- CoreOS Reserved: c95dc21a-df0e-4340-8d7b-26cbfa9a03e0
