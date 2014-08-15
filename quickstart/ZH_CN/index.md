# 使用 [CoreOS][coreos-link]

如果你还没有一个已经运行的CoreOS实例子，请参考文档[Vagrant运行CoreOS][vagrant-guide-link]、[Amazon运行CoreOS][ec2-guide-link]、[QEMU运行CoreOS][qemu-guide-link]. 这些手册能够帮助你在几分钟内新建并且启动一个虚拟机.

**注意**: ssh的用户名是 `core`. 例如 `ssh core@an.ip.compute-1.amazonaws.com`
[vagrant-guide-link]:../vagrant/index.md
[ec2-guide-link]:../ec2/index.md
[qemu-guide-link]:../qemu/index.md

[CoreOS][coreos-link] 提供了三个必要的工具: 服务发现, 容器管理和进程管理.

## 服务发现 etcd

etcd ([docs][etcd-docs-link]) 可用于节点间的服务发现. 这将是他能够想应用负载均衡的代理服务那样非常容易的做一些事情。etcd 的目的是在你构建的服务的地方增加更多的机器和服务自动扩展变得非常的容易。


API非常容易使用，通过etcd，你可以使用curl简单的设置和检索关键选项:

```
curl -L http://127.0.0.1:4001/v1/keys/message -d value="Hello world"
curl -L http://127.0.0.1:4001/v1/keys/message
```
如果你参考的是[ec2 guide][ec2-guide-link]，你可以通过SSH登录到另外一台你集群中的机器，检索相同的选项:

```
curl -L http://127.0.0.1:4001/v1/keys/message
```

etcd在集群中机器之间是持久话和可复制的。etcd不但可以在一个主机的不同容器之间共享配置，也可以单独使用.了解更多，在[github][github-link]查阅[完整API][etcd-docs-link]


[etcd-docs-link]:https://github.com/coreos/etcd/blob/master/README.md
[github-link]:http://github.com

## 容器管理 [docker][docker-link]

docker ([docs][docker-docs-link]) 包管理工具. 放置你的应用到容器中，使用etcd把不同机器的包绑定到一起.

你可以使用这些命令快速的尝试[ubuntu][ubuntu-link]容器:

```
docker run ubuntu /bin/echo hello world
docker run -i -t ubuntu /bin/bash
```  

docker为大规模的一致性应用部署提供了可能性。要知道更多，可以参考文档[docker.io][docker-docs-link].

[docker-docs-link]:http://docs.docker.io/en/latest/

## 进程管理 [systemd][systemd-link]

systemd 204 ([docs][systemd-link]). 我们认为套接字激活对于广泛的部署是非常有用的.

systemd的配置文件直接了当. 让我们创建一个在重启系统后启动的简单的服务耗尽ubuntu容器.:

第一，
由于需要修改系统状态，你需要用超级管理员`root`运行这些操作:

```
sudo -i
```

创建一个文件 `/media/state/units/hello.service`  

```  
[Unit]   
Description=My Service  
After=docker.service
  
[Service]   
Restart=always  
ExecStart=/usr/bin/docker  
run ubuntu /bin/sh -c "while true; do echo Hello World; sleep 1; done"

[Install]  
WantedBy=local.target  
```



然后，运行`systemctl restart local-enable.service`

他将启动你的守护进程并且记录systemd日志.你可以通过命令查看工作的有效输出:

```
journalctl -u hello.service -f
```

systemd提供了一个坚实的初始化系统和服务管理。了解更多，查阅[system主页][systemd-link].


[systemd-link]:http://www.freedesktop.org/wiki/Software/systemd/

#### Chaos Monkey

内建Chao Monkey(例如 随机重启).在alpha版本中,CoreOS应用更新后自动重新启动.



[coreos-link]:http://coreos.com
[docker-link]:http://docker.io
[ubuntu-link]:http://www.ubuntu.com
