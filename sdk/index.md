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

### Updating an Image with Update Engine

If you want to test that an image you built can successfully upgrade a running
VM you can use the `--image` argument to the devserver. Here is an example:

```
start_devserver --image ../build/images/amd64-generic/latest/chromiumos_image.bin
```

From the target virtual machine you run:

```
update_engine_client -update -omaha_url http://$WORKSTATION_HOSTNAME:8080/update
```

If the update fails you can check the logs of the update engine by running:

```
journalctl -u update-engine -o cat
```

If you want to download another update you may need to clear the reboot
pending status:

```
update_engine_client -reset_status
```

### Updating portage-stable ebuilds from Gentoo

There is a utility script called `update_ebuilds` that can pull from Gentoo's
CVS tree directly into your local portage-stable tree. Here is an example usage
bumping go to the latest version:

```
./update_ebuilds --commit dev-lang/go
```

To create a Pull Request after the bump run:

```
cd ~/trunk/src/third_party/portage-stable
git checkout -b 'bump-go'
git push <your remote> bump-go
```

## Production Workflows

### Building a Production Image

This will build an image that can be ran under KVM and uses near production
values.

Note: Add `COREOS_OFFICIAL=1` here if you are making a real release. That will
change the version and enable uploads by default.

```
./build_image prod
```

The generated production image is bootable as-is by qemu but for a
larger STATE partition or VMware images use `image_to_vm.sh` as
described in the final output of `build_image1`.

### Pushing updates to the dev-channel

#### Manual Builds

To push an update to the dev channel track on api.core-os.net build a
production images as described above and then use the following tool:

```
COREOS_OFFICIAL=1 ./core_upload_update <required flags> --track dev-channel --image ../build/images/amd64-generic/latest/coreos_production_image.bin
```

#### Automated builds

The automated build host does not have access to production signing keys
so the final signing and push to api.core-os.net must be done elsewhere.
The `au-generator.zip` archive provides the tools required to do this so
a full SDK setup is not required. This does require gsutil to be
installed and configured.

```
URL=gs://storage.core-os.net/coreos/amd64-generic/0000.0.0
cd $(mktemp -d)
gsutil cp $URL/au-generator.zip $URL/coreos_production_image.bin.bz2 ./
unzip au-generator.zip
bunzip2 coreos_production_image.bin.bz2
COREOS_OFFICIAL=1 ./core_upload_update <required flags> --track dev-channel --image coreos_production_image.bin
```

## Tips and Tricks

### Finding all open pull requests and issues

- [CoreOS Issues][issues]
- [CoreOS Pull Requests][pullrequests]

[issues]: https://github.com/organizations/coreos/dashboard/issues/
[pullrequests]: https://github.com/organizations/coreos/dashboard/pulls/

### Searching all repo code

Using `repo forall` you can search across all of the git repos at once:

```
repo forall -c  git grep 'CONFIG_EXTRA_FIRMWARE_DIR'
```

### Caching git https passwords

Note: You need git 1.7.10 or newer to use the credential helper

Turn on the credential helper and git will save your password in memory
for some time:

```
git config --global credential.helper cache
```

Why doesn't CoreOS use SSH in the git remotes? Because, we can't do
anonymous clones from github with a ssh URL. In the future we will fix
this.

### Base system dependency graph

Get a view into what the base system will contain and why it will contain those
things with the emerge tree view:

```
emerge-amd64-generic  --emptytree  -p -v --tree  coreos-base/coreos-dev
```

### SSH Config

You will be booting lots of VMs with on the fly ssh key generation. Add
this in your `$HOME/.ssh/config` to stop the annoying fingerprint warnings.

```
Host 127.0.0.1
  StrictHostKeyChecking no
  UserKnownHostsFile /dev/null
  User core
  LogLevel QUIET
```

### Hide loop devices from desktop environments

By default desktop environments will diligently display any mounted devices
including loop devices used to contruct CoreOS disk images. If the daemon
responsible for this happens to be ``udisks`` then you can disable this
behavior with the following udev rule:

```
echo 'SUBSYSTEM=="block", KERNEL=="ram*|loop*", ENV{UDISKS_PRESENTATION_HIDE}="1", ENV{UDISKS_PRESENTATION_NOPOLICY}="1"' > /etc/udev/rules.d/85-hide-loop.rules
udevadm control --reload
```

### Leaving developer mode

Some daemons act differently in "dev mode". For example update_engine refuses
to auto-update or connect to HTTPS URLs. If you need to test something out of
dev_mode on a vm you can do the following:

```
mv /root/.dev_mode{,.old}
```

If you want to permanently leave you can run the following:

```
crossystem disable_dev_request=1; reboot
```

## Known Issues

### build\_packages fails on coreos-base

Sometimes coreos-dev or coreos builds will fail in `build_packages` with a
backtrace pointing to `epoll`. This hasn't been tracked down but running
`build_packages` again should fix it. The error looks something like this:

```
Packages failed:
coreos-base/coreos-dev-0.1.0-r63
coreos-base/coreos-0.0.1-r187
```

## Constants and IDs

### CoreOS App ID

This UUID is used to identify CoreOS to the update service and elsewhere.

```
e96281a6-d1af-4bde-9a0a-97b76e56dc57
```

### GPT UUID Types

- CoreOS Root: 5dfbf5f4-2848-4bac-aa5e-0d9a20b745a6
- CoreOS Reserved: c95dc21a-df0e-4340-8d7b-26cbfa9a03e0
