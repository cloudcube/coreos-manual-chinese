---
layout: docs
slug: qemu
标题: 文档
---

# 在[QEMU][qemu-link]上运行CoreOS

[CoreOS](http://coreos.com) 正处于巨大的开发中，并且被积极的测试.

这些说明将指导你在QEMU下运行一个单独CoreOS实例,
[QEMU](http://www.qemu.org)有着小瑞士军刀美誉的虚拟机和CPU仿真器。
如果你需要做的更多，可参考[配置网络][qemunet]、[QEMU 文档][qemuwiki] 和 [用户手册][qemudoc].

你可以直接到 [IRC 通道][irc] 或者 [邮件列表][coreos-dev]咨询.

[qemunet]: http://wiki.qemu.org/Documentation/Networking
[qemuwiki]: http://wiki.qemu.org/Manual
[qemudoc]: http://qemu.weilnetz.de/qemu-doc.html
[irc]:irc://irc.freenode.org:6667/#coreos
[coreos-dev]:https://groups.google.com/forum/#!forum/coreos-dev


## 安装 [QEMU][qemu-link]

QEMU不但可以运行在Linux系统，还可运行在Windows和OSX,但是最好运行在Linux。
并且可运行在任何Linux发行版。

### [Debian][debian-link] 或者 [Ubuntu][ubuntu-link]

[Debian文档][qemudeb] 有更多的细节 ，但开始的话你只需要：

    sudo apt-get install qemu-system-x86 qemu-utils

[qemudeb]: https://wiki.debian.org/QEMU

### Fedora 或者 Red Hat

Fedora的维基上有一份 [快速指南][qemufed] 但是基础安装非常的简单:

    sudo yum install qemu-system-x86 qemu-img

[qemufed]: https://fedoraproject.org/wiki/How_to_use_qemu

### Arch

这是你开始所需要做的:

    sudo pacman -S qemu

[Arch's QEMU 维基页面][arch-qemu-wiki-link],你能够发现更多细节.
[arch-qemu-wiki-link]:https://wiki.archlinux.org/index.php/Qemu

### Gentoo

正如我们所预期的那样，Gentoo可能会复杂些，
但所有的所需的内核选项和USE标识都包含在 [Gentoo
维基][qemugen]. 一般来说，这样就足够了:

    echo app-emulation/qemu qemu_softmmu_targets_x86_64 virtfs xattr >> /etc/portage/package.use
    emerge -av app-emulation/qemu

[qemugen]: http://wiki.gentoo.org/wiki/QEMU


## 启动[CoreOS][coreos-link]

[QEMU][qemu-link]安装完成后，你可以下载并且启动最新的CoreOS镜像.
这里你需要两个文件:  

1. 磁盘镜像（以qcow2的格式提供）  

2. 启动[QEMU][qemu-link]的shell脚本 

    mkdir coreos; cd coreos     
    wget http://storage.core-os.net/coreos/amd64-generic/dev-channel/coreos_production_qemu.sh  
    wget http://storage.core-os.net/coreos/amd64-generic/dev-channel/coreos_production_qemu_image.img.bz2  
    chmod +x coreos_production_qemu.sh 
    bunzip2 coreos_production_qemu_image.img.bz2
	
启动很简单:    
  
    ./coreos_production_qemu.sh -nographic

### SSH Keys

In order to log in to the virtual machine you will need to use ssh keys.
If you don't already have a ssh key pair you can generate one simply by
running the command `ssh-keygen`. The wrapper script will automatically
look for public keys in ssh-agent if available and at the default
locations `~/.ssh/id_dsa.pub` or `~/.ssh/id_rsa.pub`. If you need to
provide an alternate location use the -a option:

    ./coreos_production_qemu.sh -a ~/.ssh/authoized_keys -- -nographic

Note: Options such as -a for the wrapper script must be specified before
any options for QEMU. To make the separation between the two explicit
you can use -- but that isn't required. See
`./coreos_production_qemu.sh -h` for details.

Once the virtual machine has started you can log in via SSH:

    ssh -l core -p 2222 localhost

### SSH Config

To simplify this and avoid potential host key errors in the future add
the following to `~/.ssh/config`:

    Host coreos
    HostName localhost
    Port 2222
    User core
    StrictHostKeyChecking no
    UserKnownHostsFile /dev/null

Now you can log in to the virtual machine with:

    ssh coreos


## Using CoreOS

Now that you have a machine booted it is time to play around. Check out
the [Using CoreOS][using-coreos] guide.
[debian-link]:http://www.debian.org
[ubuntu-link]:http://www.ubuntu.com
[qemu-link]:http://www.qemu.org
[coreos-link]:http://coreos.com