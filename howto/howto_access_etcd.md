在内部的容器中，如何访问etcd
=======================

这是一个临时的解决方案，用etcd监听所有接口，使你可以在docker的容器中访问它

### 创建一个本地的etcd单元文件

切换到超级用户  
`sudo -i`  
创建文件`/media/state/units/etcd-local.service`,添加一下内容:  
```  
[Unit]
Description=etcd local

[Service]  
User=etcd  
PermissionsStartOnly=true  
ExecStartPre=/bin/systemctl kill etcd.service  
ExecStartPre=/usr/bin/etcd-pre-exec  
ExecStart=/usr/bin/etcd -d /var/lib/etcd -f -cl 0.0.0.0  

Restart=always  
\# Set a longish timeout in case this machine isn't behaving  
\# nicely and bothering the rest of the cluster  
RestartSec=10s

[Install]
WantedBy=local.target  
```  
### 启用本地etcd  
`sudo systemctl restart local-enable.service`  

### 从内部容器中curl etcd  
设置一个带curl的ubuntu容器  
```  
$ docker run -t -i ubuntu /bin/bash  
$ apt-get install curl
```  
现在你需要找到ip地址的桥  
```  
$ ip route  
default via 172.17.42.1 dev eth0  
172.17.0.0/16 dev eth0  proto kernel  scope link  src 172.17.0.2    
```  
现在通过curl默认路由ip  
`$ curl http://172.17.42.1:4001/v1/keys/`  
玩得开心







