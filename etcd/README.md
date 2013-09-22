# etcd
README 版本 0.1.0

[![构建状态](https://travis-ci.org/coreos/etcd.png)](https://travis-ci.org/coreos/etcd)

一个高度可用的共享配置和服务发现的键值存储. etcd 的灵感来源与动物管理员和表演者, 它将专注与:

* 简单: curl可访问的用户的API (HTTP+JSON)
* 安全: 可选的SSL客户端证书认证
* 快速: 单实例基准1000的写入速度
* 可靠: 大量的应用分布式

Etcd 使用Go语言编写并且使用 [raft][raft] 一致性算法来管理一个高靠用复制日志.

查看 [etcdctl][etcdctl] 一个简单的命令行客户端. 或者随意使用curl，Or feel free to just use curl, 在下面的例子中.

[raft]: https://github.com/coreos/go-raft
[etcdctl]: http://coreos.com/docs/etcdctl/

## 入门

### 获取 etcd

在github[github-release]可获取最新的可用的二进制包.  

[github-release]: https://github.com/coreos/etcd/releases/

### 构建

你可以从源码构建etcd:

```sh
git clone https://github.com/coreos/etcd
cd etcd
./build
```

这将在基本目录生成一个名为 `./etcd`二进制包.

_注意_: 你需要 go 1.1+. 请使用一下命令检查你的go的安装

```
go version
```

### 单节点运行

这些例子将使用单节点集群，向你展示基础的etcd REST API.让我们启动etcd:

```sh
./etcd -d node0 -n node0
```

etcd将为客户端通信监听4001端口和为服务器之间通信监听7001端口.
参数 `-d node0` 告诉我们etcd将向`./node0/`目录写入节点配置，日志和快照.
`-n node0` 告诉其余的集群节点名`node0`.

## 用法

### 设置该值为主键

让我们设置第一个节点键值对. 在这个例子中键是 `/message` 和值是 `Hello world`.

```sh
curl -L http://127.0.0.1:4001/v1/keys/message -d value="Hello world"
```

```json
{"action":"SET","key":"/message","value":"Hello world","newKey":true,"index":3}
```

这个响应包含5个字段，我们将介绍三个字段，我们也可以尝试更多的命令.

1. 请求动作;我们通过POST请求设置值，因此这个动作是`SET`.

2. 请求关键字;我们设置 `/message` 到 `Hello world!`, 因此关键字段是 `/message`.
注意 我们使用一个像结构的文件系统来表示键值对. 因此每个键使用`/`开始.

3. 当前的键值;我们设置值为`Hello world`.

4. 如果我们设置一个新的键; 在之前`/message`不存在, 因此这是一个新的键.

5. 索引是单独的内部日志索引的请求集合. 改变日志索引的请求包括 `SET`, `DELETE` 和 `TESTANDSET`.  `GET`, `LIST` 和 `WATCH` 命令 不能够改变存储中的状态，所以他们不改变索引. 你需要注意的是在这个例子中索引是3, 尽管这是你发给服务器的第一个请求. 这是因为这丽有想添加和同步服务器的内部命令改变命令.

### 获取键值

我们通过发送 GET在 `/message`中获取值:

```sh
curl -L http://127.0.0.1:4001/v1/keys/message
```

```json
{"action":"GET","key":"/message","value":"Hello world","index":3}
```
### 修改键值

使用另外一个POST修改`/message`的键值从 from `Hello world` to到 `Hello etcd`:

```sh
curl -L http://127.0.0.1:4001/v1/keys/message -d value="Hello etcd"
```

```json
{"action":"SET","key":"/message","prevValue":"Hello world","value":"Hello etcd","index":4}
```

注意`prevValue` is 被设置为 `Hello world`.

### 删除一个键

使用DELETE移除键 `/message`:

```sh
curl -L http://127.0.0.1:4001/v1/keys/message -X DELETE
```

```json
{"action":"DELETE","key":"/message","prevValue":"Hello etcd","index":5}
```

### 使用键 TTL

在etcd中可以设置键在一个制定的秒数后过期. 当你POST,这是一个在键上设置TTL(生存时间):

```sh
curl -L http://127.0.0.1:4001/v1/keys/foo -d value=bar -d ttl=5
```

```json
{"action":"SET","key":"/foo","value":"bar","newKey":true,"expiration":"2013-07-11T20:31:12.156146039-07:00","ttl":4,"index":6}
```

注意 最后两个字段的响应:

1. 到期时间键将过期并且被删除.

2. ttl是键的生存时间.

现在你可以尝试通过发送一下信息获得键:

```sh
curl -L http://127.0.0.1:4001/v1/keys/foo
```

如果TTL已经过期, 键已经被删除,并且你会被返回一个100.

```json
{"errorCode":100,"message":"Key Not Found","cause":"/foo"}
```

### 查看前缀

如果任何键在前缀发生改变，我们可以查看前缀路径，并且获取通知.

在一个终端，我们发送查看请求:

```sh
curl -L http://127.0.0.1:4001/v1/watch/foo
```

现在，我们可以查看在前缀路径 `/foo`并且在这个路径下等待改变.

在另外一个终端, 我们设置一个键从 `/foo/foo` 到 `barbar` 将会发生什么呢:

```sh
curl -L http://127.0.0.1:4001/v1/keys/foo/foo -d value=barbar
```

第一个终端可能获取到通知并且返回使用相同的响应作为请求集合.

```json
{"action":"SET","key":"/foo/foo","value":"barbar","newKey":true,"index":7}
```

然而，查看命令能够做的不仅于此. 使用索引我们可以查看过去发生的命令. 这是非常有用的确保你不会错过两个命令之间的事件.

让我们再次尝试查看索引为6的命令集合:

```sh
curl -L http://127.0.0.1:4001/v1/watch/foo -d index=7
```

查看命令立即返回以前相同的响应.

### 原子测试与设置

在集群中Etcd被作为集中协调服务和`TestAndSet` 是构建分布式锁服务的最基本的操作. 只有当客户端提供的`prevValue`等于当前键值，这个命令将设置值.

这里有一个简单的例子. 首先我们创建一个键值对: `testAndSet=one`.

```sh
curl -L http://127.0.0.1:4001/v1/keys/testAndSet -d value=one
```

然我们尝试一个无效的 `TestAndSet` 命令.
我们可以给出一个prevValue参数来设置TestAndSet命令.

```sh
curl -L http://127.0.0.1:4001/v1/keys/testAndSet -d prevValue=two -d value=three
```

这将尝试测试如果以前的键是2，那么他就变成了3.

```json
{"errorCode":101,"message":"The given PrevValue is not equal to the value of the key","cause":"TestAndSet: one!=two"}
```

这意味着`testAndSet` 失败.

让我们尝试一个有效的.

```sh
curl -L http://127.0.0.1:4001/v1/keys/testAndSet -d prevValue=one -d value=two
```

响应应该这样:

```json
{"action":"SET","key":"/testAndSet","prevValue":"one","value":"two","index":10}
```

我们成功的修改值从 “one” 到 “two”, 因为我们给了正确的前一个值.

### 监听目录

最后我们提供一个简单的List命令列出在一个前缀路径下的所有的键.

首先让我们创建一些键.

我们已经有 `/foo/foo=barbar`

我们创建了另外一个 `/foo/foo_dir/foo=barbarbar`

```sh
curl -L http://127.0.0.1:4001/v1/keys/foo/foo_dir/bar -d value=barbarbar
```

现在列出在`/foo`下的键  

```sh
curl -L http://127.0.0.1:4001/v1/keys/foo/
```

我们可以看到作为一个数据条目的响应

```json
[{"action":"GET","key":"/foo/foo","value":"barbar","index":10},{"action":"GET","key":"/foo/foo_dir","dir":true,"index":10}]
```

这意味着 `foo=barbar` 是一个键值对，在目录 `/foo` 和 `foo_dir`.

## 高级用法

### 使用HTTPS安全传输

Etcd 支持 客户端到服务端的SSL/TLS 和 客户端证书认证,以及服务器到服务器的通信

首先, 你需要有一个CA证书 `clientCA.crt` 和签名的密匙对`client.crt`, `client.key`. 这个站点有一个很好的生成自签名的密匙对:
http://www.g-loaded.eu/2005/11/10/be-your-own-ca/

作为测试你可以在`fixtures/ca` 目录使用证书.

接下来,让我们配置etcd使用这个密匙对:

```sh
./etcd -n node0 -d node0 -clientCert=./fixtures/ca/server.crt -clientKey=./fixtures/ca/server.key.insecure -f
```

`-f` 如果配置被发现，将覆盖新的节点配置(注意: 数据丢失!)
`-clientCert` 和 `-clientKey` 是在客户端和服务器安全传输层的密匙和证书

现在你可以使用https测试这个链接:

```sh
curl --cacert fixtures/ca/ca.crt https://127.0.0.1:4001/v1/keys/foo -d value=bar -v
```

你能够看到握手成功的信息.

```
...
SSLv3, TLS handshake, Finished (20):
...
```

并且从etcd服务有一样的响应.

```json
{"action":"SET","key":"/foo","value":"bar","newKey":true,"index":3}
```

### 使用HTTPS客户端证书认证

我们也可以使用CA证书进行认证. 客户端将提供他们的证书到服务器并且服务器将检查整个证书是否有CA签发并且决定是否送到请求.

```sh
./etcd -n node0 -d node0 -clientCAFile=./fixtures/ca/ca.crt -clientCert=./fixtures/ca/server.crt -clientKey=./fixtures/ca/server.key.insecure -f
```

```-clientCAFile``` 是指想CA证书的路径.

向这个服务器发送相同的请求:

```sh
curl --cacert fixtures/ca/ca.crt https://127.0.0.1:4001/v1/keys/foo -d value=bar -v
```

这个请求被服务器拒绝.

```
...
routines:SSL3_READ_BYTES:sslv3 alert bad certificate
...
```

我们需要向服务器给出签名的CA证书.

```sh
curl -L https://127.0.0.1:4001/v1/keys/foo -d value=bar -v --key myclient.key --cert myclient.crt -cacert clientCA.crt
```

你将看见
```
...
SSLv3, TLS handshake, CERT verify (15):
...
TLS handshake, Finished (20)
```

并且从服务器一样的响应:

```json
{"action":"SET","key":"/foo","value":"bar","newKey":true,"index":3}
```

## 集群

### 三台机器的集群实例

让我探测使用etcd的集群. 我们使用go-raft作为底层的分布式协议，该协议在所有的etcd实例之间提供数据一直性和持久性.

让我们创建3个新的etcd实例.

我们使用`-s`指定服务端口 并且` -c` 制定客户端端口和`-d` 制定目录存储日志和在集群中的节点信息

```sh
./etcd -s 127.0.0.1:7001 -c 127.0.0.1:4001 -d nodes/node1 -n node1
```

**注意:** 如果你想在外部IP地址运行etcd并且在本地能够访问，你需要添加 `-cl 0.0.0.0` 以至于将同时监听外部和本地地址.
一个简单的参数 `-sl` 被用于为服务器端口设置监听地址.

让我们使用`-C`参数在这个集群中添加两个或多个节点:

```sh
./etcd -c 127.0.0.1:4002 -s 127.0.0.1:7002 -C 127.0.0.1:7001 -d nodes/node2 -n node2
./etcd -c 127.0.0.1:4003 -s 127.0.0.1:7003 -C 127.0.0.1:7001 -d nodes/node3 -n node3
```

在集群中获取机器:

```sh
curl -L http://127.0.0.1:4001/v1/machines
```

我们将会看见集群中的三个节点

```
http://127.0.0.1:4001, http://127.0.0.1:4002, http://127.0.0.1:4003
```

通过这个API这个机器列表也是可用:

```sh
curl -L http://127.0.0.1:4001/v1/keys/_etcd/machines
```

```json
[{"action":"GET","key":"/_etcd/machines/node1","value":"raft=http://127.0.0.1:7001&etcd=http://127.0.0.1:4001","index":4},{"action":"GET","key":"/_etcd/machines/node2","value":"raft=http://127.0.0.1:7002&etcd=http://127.0.0.1:4002","index":4},{"action":"GET","key":"/_etcd/machines/node3","value":"raft=http://127.0.0.1:7003&etcd=http://127.0.0.1:4003","index":4}]
```

当被添加，机器的键基于 ```commit index``` .机器的值是 ```hostname```, ```raft port``` 和 ```client port```.

同时尝试获取集群中的当前主节点

```
curl -L http://127.0.0.1:4001/v1/leader
```
第一个服务器我们设置为主节点, 如果在这些命令期间没有失效.

```
http://127.0.0.1:7001
```

现在我们可以在这些键上执行一般的 SET 和 GET 操作作为我们早期的遍历.

```sh
curl -L http://127.0.0.1:4001/v1/keys/foo -d value=bar
```

```json
{"action":"SET","key":"/foo","value":"bar","newKey":true,"index":5}
```

### 在集群中停止节点

让我们停止集群中的主节点并且从其他的机器获取值:

```sh
curl -L http://127.0.0.1:4002/v1/keys/foo
```
一个新的主节点将被选择出来.

```
curl -L http://127.0.0.1:4001/v1/leader
```

```
http://127.0.0.1:7002
```

或者

```
http://127.0.0.1:7003
```

你可能会看见这些:

```json
{"action":"GET","key":"/foo","value":"bar","index":5}
```

证明成功了!

### 测试持久化

好的，接下来让我们停止掉所有的节点来测试持久化. 并且使用一下命令来重新启动所有的节点.

你对键`foo`的请求将会返回正确的值:

```sh
curl -L http://127.0.0.1:4002/v1/keys/foo
```

```json
{"action":"GET","key":"/foo","value":"bar","index":5}
```

### 在服务器之间使用HTTPS

在前面的例子我们展示了如何为客户端到服务端使用SSL客户端证书进行通信. Etcd 也能够在外部服务器与服务器使用SSL客户端证书进行通信.需要做的仅仅改变```-client*``` 标识到 ```-server*```.

如果你在服务器与服务器之间使用SS，你在所有的etcd节点也必须使用.

## 库和工具

**工具**

- [etcdctl](https://github.com/coreos/etcdctl) - 一个为etcd的命令行的客户端工具

**Go 库**

- [go-etcd](https://github.com/coreos/go-etcd)

**Java 库**

- [justinsb/jetcd](https://github.com/justinsb/jetcd)
- [diwakergupta/jetcd](https://github.com/diwakergupta/jetcd)


**Python 库**

- [transitorykris/etcd-py](https://github.com/transitorykris/etcd-py)

**Node 库**

- [stianeikeland/node-etcd](https://github.com/stianeikeland/node-etcd)

**Ruby 库**

- [iconara/etcd-rb](https://github.com/iconara/etcd-rb)
- [jpfuentes2/etcd-ruby](https://github.com/jpfuentes2/etcd-ruby)
- [ranjib/etcd-ruby](https://github.com/ranjib/etcd-ruby)

**Chef 手册**

- [spheromak/etcd-cookbook](https://github.com/spheromak/etcd-cookbook)

**使用 etcd的项目**

- [calavera/active-proxy](https://github.com/calavera/active-proxy) - 使用etcd的http代理配置
- [gleicon/goreman](https://github.com/gleicon/goreman/tree/etcd) - 使用etcd支持的Go Foreman clone分支
- [garethr/hiera-etcd](https://github.com/garethr/hiera-etcd) - Puppet hiera 后端使用 etcd
- [mattn/etcd-vim](https://github.com/mattn/etcd-vim) - 从VIM内部SET 和 GET 键
- [mattn/etcdenv](https://github.com/mattn/etcdenv) - "env" shebang 使用etcd 集成

## 问答

### 我们应该使用多大的集群?

每一个命令客户端发送一个主广播给所有的追随者.
但是,命令不会被提交直到 大部分集群集群接收了这些命令.

因为大部分表的属性保持理想的簇很小以保持基数台机器的速度.

基数数量非常好因为如何你有8台机器大部分应该是5并且如果你有9太机器大部分还是5.
这个结果是8节点机器集群能够容忍3节点故障，9节点机器集群能够4节点故障.
在最好的情况下，9节点机器集群的响应速度最快的5节点机器将被执行.

## 项目详情

### 版本

etcd 使用 [语义版本][semver].
当我们发布v1.0.0的etcd我们将承诺不会破坏"v1" REST API.
新的次要版本的可能会添加额外功能的API.

通过根路径的请求，你可以获得etcd的版本:

```sh
curl -L http://127.0.0.1:4001
```

在v0系列发布期间，我们会破坏API作为我们修复缺陷和获得反馈 .

[semver]: http://semver.org/

### 许可

etcd 基于Apache 2.0 许可. 查看 [LICENSE][license] 文件了解详情.

[license]: https://github.com/coreos/etcd/blob/master/LICENSE
