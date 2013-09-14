# etcd
README 版本 0.1.0

[![构建状态](https://travis-ci.org/coreos/etcd.png)](https://travis-ci.org/coreos/etcd)

一个高度可用的共享配置和服务发现的键值存储. etcd 的灵感来源与动物管理员和表演者, 它将专注与:

* 简单: curl可访问的用户的API (HTTP+JSON)
* 安全: 可选的SSL客户端证书认证
* 快树: 单实例基准1000的写入速度
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

Keys in etcd can be set to expire after a specified number of seconds. That is done by setting a TTL (time to live) on the key when you POST:

```sh
curl -L http://127.0.0.1:4001/v1/keys/foo -d value=bar -d ttl=5
```

```json
{"action":"SET","key":"/foo","value":"bar","newKey":true,"expiration":"2013-07-11T20:31:12.156146039-07:00","ttl":4,"index":6}
```

Note the last two new fields in response:

1. The expiration is the time that this key will expire and be deleted.

2. The ttl is the time to live of the key.

Now you can try to get the key by sending:

```sh
curl -L http://127.0.0.1:4001/v1/keys/foo
```

If the TTL has expired, the key will be deleted, and you will be returned a 100.

```json
{"errorCode":100,"message":"Key Not Found","cause":"/foo"}
```

### Watching a prefix

We can watch a path prefix and get notifications if any key change under that prefix.

In one terminal, we send a watch request:

```sh
curl -L http://127.0.0.1:4001/v1/watch/foo
```

Now, we are watching at the path prefix `/foo` and wait for any changes under this path.

In another terminal, we set a key `/foo/foo` to `barbar` to see what will happen:

```sh
curl -L http://127.0.0.1:4001/v1/keys/foo/foo -d value=barbar
```

The first terminal should get the notification and return with the same response as the set request.

```json
{"action":"SET","key":"/foo/foo","value":"barbar","newKey":true,"index":7}
```

However, the watch command can do more than this. Using the the index we can watch for commands that has happened in the past. This is useful for ensuring you don't miss events between watch commands.

Let's try to watch for the set command of index 6 again:

```sh
curl -L http://127.0.0.1:4001/v1/watch/foo -d index=7
```

The watch command returns immediately with the same response as previous.

### Atomic Test and Set

Etcd can be used as a centralized coordination service in a cluster and `TestAndSet` is the most basic operation to build distributed lock service. This command will set the value only if the client provided `prevValue` is equal the current key value.

Here is a simple example. Let's create a key-value pair first: `testAndSet=one`.

```sh
curl -L http://127.0.0.1:4001/v1/keys/testAndSet -d value=one
```

Let's try an invaild `TestAndSet` command.
We can give another parameter prevValue to set command to make it a TestAndSet command.

```sh
curl -L http://127.0.0.1:4001/v1/keys/testAndSet -d prevValue=two -d value=three
```

This will try to test if the previous of the key is two, it is change it to three.

```json
{"errorCode":101,"message":"The given PrevValue is not equal to the value of the key","cause":"TestAndSet: one!=two"}
```

which means `testAndSet` failed.

Let us try a vaild one.

```sh
curl -L http://127.0.0.1:4001/v1/keys/testAndSet -d prevValue=one -d value=two
```

The response should be

```json
{"action":"SET","key":"/testAndSet","prevValue":"one","value":"two","index":10}
```

We successfully changed the value from “one” to “two”, since we give the correct previous value.

### Listing a directory

Last we provide a simple List command to list all the keys under a prefix path.

Let us create some keys first.

We already have `/foo/foo=barbar`

We create another one `/foo/foo_dir/foo=barbarbar`

```sh
curl -L http://127.0.0.1:4001/v1/keys/foo/foo_dir/bar -d value=barbarbar
```

Now list the keys under `/foo`

```sh
curl -L http://127.0.0.1:4001/v1/keys/foo/
```

We should see the response as an array of items

```json
[{"action":"GET","key":"/foo/foo","value":"barbar","index":10},{"action":"GET","key":"/foo/foo_dir","dir":true,"index":10}]
```

which meas `foo=barbar` is a key-value pair under `/foo` and `foo_dir` is a directory.

## Advanced Usage

### Transport security with HTTPS

Etcd supports SSL/TLS and client cert authentication for clients to server, as well as server to server communication

First, you need to have a CA cert `clientCA.crt` and signed key pair `client.crt`, `client.key`. This site has a good reference for how to generate self-signed key pairs:
http://www.g-loaded.eu/2005/11/10/be-your-own-ca/

For testing you can use the certificates in the `fixtures/ca` directory.

Next, lets configure etcd to use this keypair:

```sh
./etcd -n node0 -d node0 -clientCert=./fixtures/ca/server.crt -clientKey=./fixtures/ca/server.key.insecure -f
```

`-f` forces new node configuration if existing configuration is found (WARNING: data loss!)
`-clientCert` and `-clientKey` are the key and cert for transport layer security between client and server

You can now test the configuration using https:

```sh
curl --cacert fixtures/ca/ca.crt https://127.0.0.1:4001/v1/keys/foo -d value=bar -v
```

You should be able to see the handshake succeed.

```
...
SSLv3, TLS handshake, Finished (20):
...
```

And also the response from the etcd server.

```json
{"action":"SET","key":"/foo","value":"bar","newKey":true,"index":3}
```

### Authentication with HTTPS client certificates

We can also do authentication using CA certs. The clients will provide their cert to the server and the server will check whether the cert is signed by the CA and decide whether to serve the request.

```sh
./etcd -n node0 -d node0 -clientCAFile=./fixtures/ca/ca.crt -clientCert=./fixtures/ca/server.crt -clientKey=./fixtures/ca/server.key.insecure -f
```

```-clientCAFile``` is the path to the CA cert.

Try the same request to this server:

```sh
curl --cacert fixtures/ca/ca.crt https://127.0.0.1:4001/v1/keys/foo -d value=bar -v
```

The request should be rejected by the server.

```
...
routines:SSL3_READ_BYTES:sslv3 alert bad certificate
...
```

We need to give the CA signed cert to the server.

```sh
curl -L https://127.0.0.1:4001/v1/keys/foo -d value=bar -v --key myclient.key --cert myclient.crt -cacert clientCA.crt
```

You should able to see
```
...
SSLv3, TLS handshake, CERT verify (15):
...
TLS handshake, Finished (20)
```

And also the response from the server:

```json
{"action":"SET","key":"/foo","value":"bar","newKey":true,"index":3}
```

## Clustering

### Example cluster of three machines

Let's explore the use of etcd clustering. We use go-raft as the underlying distributed protocol which provides consistency and persistence of the data across all of the etcd instances.

Let start by creating 3 new etcd instances.

We use -s to specify server port and -c to specify client port and -d to specify the directory to store the log and info of the node in the cluster

```sh
./etcd -s 127.0.0.1:7001 -c 127.0.0.1:4001 -d nodes/node1 -n node1
```

**Note:** If you want to run etcd on external IP address and still have access locally you need to add `-cl 0.0.0.0` so that it will listen on both external and localhost addresses.
A similar argument `-sl` is used to setup the listening address for the server port.

Let the join two more nodes to this cluster using the -C argument:

```sh
./etcd -c 127.0.0.1:4002 -s 127.0.0.1:7002 -C 127.0.0.1:7001 -d nodes/node2 -n node2
./etcd -c 127.0.0.1:4003 -s 127.0.0.1:7003 -C 127.0.0.1:7001 -d nodes/node3 -n node3
```

Get the machines in the cluster:

```sh
curl -L http://127.0.0.1:4001/v1/machines
```

We should see there are three nodes in the cluster

```
http://127.0.0.1:4001, http://127.0.0.1:4002, http://127.0.0.1:4003
```

The machine list is also available via this API:

```sh
curl -L http://127.0.0.1:4001/v1/keys/_etcd/machines
```

```json
[{"action":"GET","key":"/_etcd/machines/node1","value":"raft=http://127.0.0.1:7001&etcd=http://127.0.0.1:4001","index":4},{"action":"GET","key":"/_etcd/machines/node2","value":"raft=http://127.0.0.1:7002&etcd=http://127.0.0.1:4002","index":4},{"action":"GET","key":"/_etcd/machines/node3","value":"raft=http://127.0.0.1:7003&etcd=http://127.0.0.1:4003","index":4}]
```

The key of the machine is based on the ```commit index``` when it was added. The value of the machine is ```hostname```, ```raft port``` and ```client port```.

Also try to get the current leader in the cluster

```
curl -L http://127.0.0.1:4001/v1/leader
```
The first server we set up should be the leader, if it has not dead during these commands.

```
http://127.0.0.1:7001
```

Now we can do normal SET and GET operations on keys as we explored earlier.

```sh
curl -L http://127.0.0.1:4001/v1/keys/foo -d value=bar
```

```json
{"action":"SET","key":"/foo","value":"bar","newKey":true,"index":5}
```

### Killing Nodes in the Cluster

Let's kill the leader of the cluster and get the value from the other machine:

```sh
curl -L http://127.0.0.1:4002/v1/keys/foo
```

A new leader should have been elected.

```
curl -L http://127.0.0.1:4001/v1/leader
```

```
http://127.0.0.1:7002
```

or

```
http://127.0.0.1:7003
```

You should be able to see this:

```json
{"action":"GET","key":"/foo","value":"bar","index":5}
```

It succeeded!

### Testing Persistence

OK. Next let us kill all the nodes to test persistence. And restart all the nodes use the same command as before.

Your request for the `foo` key will return the correct value:

```sh
curl -L http://127.0.0.1:4002/v1/keys/foo
```

```json
{"action":"GET","key":"/foo","value":"bar","index":5}
```

### Using HTTPS between servers

In the previous example we showed how to use SSL client certs for client to server communication. Etcd can also do internal server to server communication using SSL client certs. To do this just change the ```-client*``` flags to ```-server*```.

If you are using SSL for server to server communication, you must use it on all instances of etcd.

## Libraries and Tools

**Tools**

- [etcdctl](https://github.com/coreos/etcdctl) - A command line client for etcd

**Go libraries**

- [go-etcd](https://github.com/coreos/go-etcd)

**Java libraries**

- [justinsb/jetcd](https://github.com/justinsb/jetcd)
- [diwakergupta/jetcd](https://github.com/diwakergupta/jetcd)


**Python libraries**

- [transitorykris/etcd-py](https://github.com/transitorykris/etcd-py)

**Node libraries**

- [stianeikeland/node-etcd](https://github.com/stianeikeland/node-etcd)

**Ruby libraries**

- [iconara/etcd-rb](https://github.com/iconara/etcd-rb)
- [jpfuentes2/etcd-ruby](https://github.com/jpfuentes2/etcd-ruby)
- [ranjib/etcd-ruby](https://github.com/ranjib/etcd-ruby)

**Chef Cookbook**

- [spheromak/etcd-cookbook](https://github.com/spheromak/etcd-cookbook)

**Projects using etcd**

- [calavera/active-proxy](https://github.com/calavera/active-proxy) - HTTP Proxy configured with etcd
- [gleicon/goreman](https://github.com/gleicon/goreman/tree/etcd) - Branch of the Go Foreman clone with etcd support
- [garethr/hiera-etcd](https://github.com/garethr/hiera-etcd) - Puppet hiera backend using etcd
- [mattn/etcd-vim](https://github.com/mattn/etcd-vim) - SET and GET keys from inside vim
- [mattn/etcdenv](https://github.com/mattn/etcdenv) - "env" shebang with etcd integration

## FAQ

### What size cluster should I use?

Every command the client sends to the master is broadcast it to all of the followers.
But, the command is not be committed until the majority of the cluster machines receive that command.

Because of this majority voting property the ideal cluster should be kept small to keep speed up and be made up of an odd number of machines.

Odd numbers are good because if you have 8 machines the majority will be 5 and if you have 9 machines the majority with be 5.
The result is that an 8 machine cluster can tolerate 3 machine failures and a 9 machine cluster can tolerate 4 nodes failures.
And in the best case when all 9 machines are responding the cluster will perform at the speed of the fastest 5 nodes.

## Project Details

### Versioning

etcd uses [semantic versioning][semver].
When we release v1.0.0 of etcd we will promise not to break the "v1" REST API.
New minor versions may add additional features to the API however.

You can get the version of etcd by requesting the root path of etcd:

```sh
curl -L http://127.0.0.1:4001
```

During the v0 series of releases we may break the API as we fix bugs and get feedback.

[semver]: http://semver.org/

### License

etcd is under the Apache 2.0 license. See the [LICENSE][license] file for details.

[license]: https://github.com/coreos/etcd/blob/master/LICENSE
