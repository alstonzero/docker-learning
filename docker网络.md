# docker网络

## Default Networks

Docker をインストールした時にbridge、noneとhostという3 つのDocker Network がインストールされる。

```
$ sudo docker network ls 
NETWORK ID          NAME                DRIVER
10aa51e2d993        bridge              bridge
202b87d7b8a4        none                null
92316fc5522f        host                host
```

コンテナを起動するときに`--net` フラグできる。bridge networkはdefault networkであり、指定する必要がない。

# Linux网络命名空间



Linux 3.8内核中包括了6种命名空间：

| 命名空间                          | 描述                         |
| --------------------------------- | ---------------------------- |
| Mount（mnt）                      | 隔离挂载点                   |
| Process ID(process)               | 隔离进程ID                   |
| Network（net）                    | 隔离网络设备、协议栈、端口等 |
| InterProcess Communication（ipc） | 隔离进程间通信               |
| UTS                               | 隔离Hostname和NIS域名        |
| User ID(user)                     | 隔离用户和group ID           |

还有一个CGroup Name Space，和上述这些命名空间是container技术的一个基础。

这篇文章我们只讨论Network Namespaces

## 1. 创建一个新的network namespace

```csharp
$ ip netns add blue
```

查看新创建network namespace

```cpp
$ ip netns list
```

## 2. 分配一个网络接口给Network Namespaces

首先我们考虑一下虚拟网卡Veth， veth很有意思；它都是成对出现的，就像一个管道的两端，从这个管道的一端的veth进去的数据会从另一端的veth再出来。也就是说，你可以使用veth接口把一个网络命名空间连接到外部的默认命名空间或者global命名空间，而物理网卡就存在这些命名空间里。
首先我们来创建一个veth对：

```rust
$ ip link add veth0 type veth peer name veth1
```

这一条命令就会同时创建veth0和veth1两个虚拟网卡，运行如下命令来验证一下

```cpp
$ ip link list
```

这个时候你就可以看到这一对veth接口了

```csharp
4: veth1@veth0: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether be:68:5b:da:60:1d brd ff:ff:ff:ff:ff:ff
5: veth0@veth1: <BROADCAST,MULTICAST,M-DOWN> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether d2:70:e9:c1:31:2f brd ff:ff:ff:ff:ff:ff
```

这个时候这两个网卡还都属于“default”或“global”命名空间，和物理网卡一样。
现在我们要把这个global namesapce连接到blue namespace，做法就是把其中的一个veth转移到blue命名空间中去，

```bash
$ ip link set veth1 netns blue 
```

这个时候再用命令ip link list来查看就会发现veth1不见了，这个时候veth1以及设置到了blue命名空间中了，运行如下命令查看

```bash
$ ip netns exec blue ip link list
```

这个时候你可以看到里面有两个网络接口

- 一个是回环网络lo
- 另一个是veth1

```csharp
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
4: veth1@if5: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether be:68:5b:da:60:1d brd ff:ff:ff:ff:ff:ff link-netnsid 0
```

## 3. 配置Network Namespaces里面的网络接口

可以通过运行如下命令来配置blue命名空间中的veth1接口

```bash
$ ip netns exec blue ip addr add 10.1.1.1/24 dev veth1
$ ip netns exec blue ip link set veth1 up
$ ip netns exec blue ip link set lo up
```

这个命令中，通过ip addr add命令给veth1分配了一个IP 地址，并且把这个网卡启动了；同时也启动了回环网卡；
然后查看一下blue命名空间里面的两个网卡的状态

```dart
$ ip netns exec blue ip addr show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
11: veth1@if12: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state LOWERLAYERDOWN group default qlen 1000
    link/ether a6:41:bf:fc:8c:5e brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet 10.1.1.1/32 scope global veth1
       valid_lft forever preferred_lft forever
```

同时还需要配置一下host上面对端网卡的ip

```csharp
$ ip addr add 10.1.1.2/24 dev veth0
$ ip link set veth0 up
```

这个时候在blue命名空间里面ping 10.1.1.1（veth1）已经ok了

```ruby
root@ubuntu:~# ip netns exec blue ping 10.1.1.1
PING 10.1.1.1 (10.1.1.1) 56(84) bytes of data.
64 bytes from 10.1.1.1: icmp_seq=1 ttl=64 time=0.058 ms
```

但是依旧ping不通10.1.1.2（veth0）
这是因为blue命名空间中的路由还没有设置。

## 4. 设置命名空间内的路由

通过如下命令来设置一个默认路由

```csharp
$ ip netns exec blue ip route add default via 10.1.1.1
$ ip netns exec blue ip route show
default via 10.1.1.1 dev veth1
```

所有找不到目的地址的数据包都通过设备veth1转发出去。
这个时候就可以成功ping通host上的veth0网卡了

```bash
$ip netns exec blue ping 10.1.1.2
PING 10.1.1.2 (10.1.1.2) 56(84) bytes of data.
64 bytes from 10.1.1.2: icmp_seq=1 ttl=64 time=0.086 ms
```

但是到了这里依旧还不够，blue命名空间中已经可以联通主机上的网络，但是依旧连不同主机以外的外部网络。这个时候必须在host主机上启动转发，并且在iptables中设置伪装

## 5. 使能IP转发：IP_FORWARD

Linux系统的IP转发的意思是，当Linux主机存在多个网卡的时候，允许一个网卡的数据包转发到另外一张网卡；在linux系统中默认禁止IP转发功能，可以打开如下文件查看，如果值为0说明禁止进行IP转发

```undefined
cat /proc/sys/net/ipv4/ip_forward
```

运行如下的命令，其中ens160是host的一个对外网卡，这样的就允许ens160和veth0之间的转发；也就是说blue命名空间可以和外网联通了。

```ruby
#使能ip转发

echo 1 > /proc/sys/net/ipv4/ip_forward

#刷新forward规则
iptables -F FORWARD
iptables -t nat -F

#刷新nat规则
iptables -t nat -L -n

#使能IP伪装
iptables -t nat -A POSTROUTING -s 10.1.1.0/255.255.255.0 -o ens160 -j MASQUERADE

#允许veth0和ens160之间的转发
iptables -A FORWARD -i ens160 -o veth0 -j ACCEPT
iptables -A FORWARD -o ens160 -i veth0 -j ACCEPT
```

- 使能IP伪装这条语句，添加了一条规则到NAT表的POSTROUTING链中，对于源IP地址为10.1.1.0网段的数据包，用ens160网口的IP地址替换并发送。
- iptables -A FORWARD这两条语句使能物理网口ens160和veth0之间的数据转发

## 6. 演示一下整个流程



![img](https://upload-images.jianshu.io/upload_images/7298148-0297172323688f20.png?imageMogr2/auto-orient/strip|imageView2/2/w/799/format/webp)

## 7. 完整的示例代码

```bash
#!/bin/bash
#Create new namesapce
ip netns delete ns1
ip netns add ns1

ip link add veth0 type veth peer name veth1
ip link set veth0 netns ns1

echo "====Create New Network namespace and spcify a eth===="
ip netns list
ip netns ns exec if config -a

#Assign the IP and bring up
ip addr add 10.100.1.1/24 dev veth1
ip link set veth1 up

ip netns exec ns1 ip addr add 10.100.1.2/24 dev veth0
ip netns exec ns1 ip link set veth0 up
ip netns exec ns1 ip link set lo up

echo "====Bring up the veth0 and lo inside Namespace===="
ip netns exec ns1 ip addr show

#add route inside namespace
ip netns exec ns1 ip route add default via 10.100.1.1

echo "====Add new default rout inside the namespace===="
ip netns exec ns1 ip route show

echo "====Tryting to ping the veth1 on host===="
ip netns exec ns1 ping 10.100.1.1 -c 4

#Config the host to enable forwarding
echo 1 > /proc/sys/net/ipv4/ip_forward

iptables -P FORWARD DROP
iptables -F FORWARD

iptables -t nat -F

#enalbe masquerading of 10.100.1.0
iptables -t nat -A POSTROUTING -s 10.100.1.0/255.255.255.0 -o ens160 -j MASQUERADE

#Allow forwarding
iptables -A FORWARD -i ens160 -o veth1 -j ACCEPT
iptables -A FORWARD -o ens160 -i veth1 -j ACCEPT

echo "====Enable the forwarding of veth1 to ens160(NIC)on host ===="
echo "Show the iptables of filter"
iptables -L -n

echo "Show the iptables of nat"
iptables -t nat -L -n
```



### 慕课视频课例子

首先使用**busybox** image创建一个后台运行的container test1，在里面运行一个/bin/sh（shell命令），这个命令是个循环，每次sleep 3600秒，然后再循环执行

```
docker run -d --name test1 busybox /bin/sh -c "while true; do sleep 3600; done"
```

 用docker ps 查出contianer的id，再用exec -it 以交互式命令格式操作该container并进入命令行格式

```
docker exec -it test1_id /bin/sh
ip a  #显示该container所有的网络接口
​```
得到 test1 的ip地址为 172.17.0.2/16
​```
```

container的network namespace和host networknamespace是隔离开的



接下来再同样创建一个后台运行的container test2

```
docker run -d --name test1 busybox /bin/sh -c "while true; do sleep 3600; done"
```

用 docker ps 查看test2的container_id，exec直接查看test2所有网络接口

```
docker exec test2_id ip a
​```
得到test2的ip地址为172.17.0.3/16
​```
```



exec -it 交互式进入contaienr test1 中，去ping test2的id address

```
docker exec -it test1_id /bin/sh
ping 172.17.0.3 #test2的ip地址
​```
可以ping通
​```
```

结果证明了这两个container是可以相互之间通信的

### 创建两个network namespace 并连接

首先创建network namespace  test1 ,test2

```
sudo ip netns add test1
```

若删除用delete

```
sudo ip netns delete test1
```

查看当前的network namespace

```
sudo ip netns list
```

通过exec去查看network namespace test1的ip address

```
sudo ip netns exec test1 ip a  #在test1里面执行ip a这个命令
```

通过exec去查看network namespace test1的ip link

```
sudo ip netns exec test1 ip link
​```
结果显示只有lo 且 DOWN
​```
```

让lo端口从DOWN变成UP

```
sudo ip netns exec test1 ip link set dev lo up
```



#### 让test1和test2连接起来

创建一对veth，分别放到test1和test2这两个name space里面

首先在host宿主机中创建一对link

```
sudo ip link add veth-test1 type veth peer name veth-test2
```



veth-test1，veth-test2 分别添加到namespace test1和test2里面

```
sudo ip link set veth-test1 netns test1
sudo ip link set veth-test2 netns test2
​```
结果显示只有mac address 没有 ip address
​```
```

再查看test1和test2的namespace中是否增加了

但是目前veth-test1veth-test2还没有ip address，手动添加ip address

```
sudo ip netns exec test1 ip addr add 192.168.1.1/24 dev veth-test1
sudo ip netns exec test2 ip addr add 192.168.1.2/24 dev veth-test2
```

接下来用exec去查看test1hetest2ip link

````
sudo ip netns exec test1 ip link 

​```
结果刚刚ip address并没有添加给veth-test1veth-test2
​```
````

接下来需要启动veth-test1,veth-test2

```
sudo ip netns exec test1 ip link set dev veth-test1 up
sudo ip netns exec test2 ip link set dev veth-test2 up
```

最后，用exec操作test1去ping veth-test2的ip address

```
sudo ip netns exec test1 ping 192.168.1.2
​```
结果联通
​```
```



## Docker Bridge0详解

首先停止container test2，只保留test1，通过docker ps 查出test1 的container id

```shell
docker stop test2  #停止test2运行

docker network ls  
​```
NETWORK ID          NAME                DRIVER              SCOPE
dc7c0ff74a53        bridge              bridge              local
ecb1ee98a948        host                host                local
1fd0bf197d78        none                null                local
​```
```

结果显示有bridge

接下来查看连接到bridge上面的contianer

```shell
docker network inspect dc7c0ff74a53 #bridge的NETWORK ID  
​```
.......
"Containers": {
            "5a53225956c23923390d2e725f1a284f784ce66b3d7b0d1df1063944c2fc7a90": {
                "Name": "test1",
                "EndpointID": "0646b2f2bbfccb4c884d3abd8dcb09b1f8a7ca9c325e4305dd0cb3ebe60105ed",
                "MacAddress": "02:42:ac:11:00:02",
                "IPv4Address": "172.17.0.2/16",
                "IPv6Address": ""
            }
        },
......
​```
```

显示container test1连接到了bridge网络上面



如何证明veth是连接到了docker0上，需要安装brctl（表示bridge的情况）

```shell
sudo yum install bridge-utils
```

先查看当前host 的ip a

```
ip a
​```
......
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:c5:02:c0:5b brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:c5ff:fe02:c05b/64 scope link
       valid_lft forever preferred_lft forever
5: veth7ab15e5@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
    link/ether f6:67:56:0f:60:dd brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::f467:56ff:fe0f:60dd/64 scope link
       valid_lft forever preferred_lft forever
......
​```
```

再查看当前的bridge上面的连接

```
brctl show
​```
bridge name     bridge id               STP enabled     interfaces
docker0         8000.0242c502c05b       no              veth7ab15e5  
​```
```

结果显示，当前这台机器上只有一个Linux bridge叫做docker0。而bridge docker0的接口veth7ab15e5就是test1的veth

这同时证明了container需要一对veth，一个veth在container里，另一个veth在host里，来连接docker0，从而进行外部通信。

这时启动container test2

```
docker start test2
```

检查当前host的 ip a

```
ip a
​```
......
3: docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default
    link/ether 02:42:c5:02:c0:5b brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:c5ff:fe02:c05b/64 scope link
       valid_lft forever preferred_lft forever
5: veth7ab15e5@if4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
    link/ether f6:67:56:0f:60:dd brd ff:ff:ff:ff:ff:ff link-netnsid 0
    inet6 fe80::f467:56ff:fe0f:60dd/64 scope link
       valid_lft forever preferred_lft forever
9: veth0d78781@if8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue master docker0 state UP group default
    link/ether 8a:d9:de:0b:b8:96 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet6 fe80::88d9:deff:fe0b:b896/64 scope link
       valid_lft forever preferred_lft forever
......
​``````
```

这时多出来了veth0d78781。证明启动test2的同时，生成了一对veth用来连接 bridge docker0

再查看当前bridge docker0上面的连接

```
brctl show
​```
bridge name     bridge id               STP enabled     interfaces
docker0         8000.0242c502c05b       no              veth0d78781
                                                        veth7ab15e5
​```
```

bridge docker0上面连了两个接口，分别是test1的veth0d78781和test2的veth7ab15e5。

test1和test2之间也可以通过bridge docker0来通信



## contaienr之间的link

首先stop 并且删除rm test2

```
docker stop test2
docker rm test2
```

接下来，使用`--link`创建以一个与test1相连接的新test2

```
docker run -d --name test2 --link test1 busybox /bin/sh -c "while true; do sleep 3600; done"
```

接下来用exec操作test2看看能否ping通test1这个名字

```
docker exec test2 ping test1
​```
PING test1 (172.17.0.2): 56 data bytes
64 bytes from 172.17.0.2: seq=0 ttl=64 time=0.099 ms
64 bytes from 172.17.0.2: seq=1 ttl=64 time=0.216 ms
^C #Ctrl+C 强制停止
​```
```

结果显示可以ping通。

**实际上**，给test2添加link等于是给test2添加了DNS的记录

反过来试着用test1去ping test2的名字和ip address

```
docker exec test1 ping test2
​```
ping: bad address 'test2'
​```

docker exec test1 ping 172.17.0.3
​```
PING 172.17.0.3 (172.17.0.3): 56 data bytes
64 bytes from 172.17.0.3: seq=0 ttl=64 time=0.145 ms
64 bytes from 172.17.0.3: seq=1 ttl=64 time=0.087 ms
​```
```

test1能ping通test2的ip address，但是test1不能直接ping 通test2的名字。

**证明link是单向的**

### 如何自己新建一个docker bridge并让contrainer连接

创建自己的brige

```
docker network create -d bridge my-bridge #用-d指定device 
```



```
docker network ls
​```
NETWORK ID          NAME                DRIVER              SCOPE
dc7c0ff74a53        bridge              bridge              local
ecb1ee98a948        host                host                local
8cfacc1bda63        my-bridge           bridge              local
1fd0bf197d78        none                null                local
​```
```

用`brtcl`去查看当前存在的brige，进行确认

```
brctl show
​```
bridge name     bridge id               STP enabled     interfaces
br-8cfacc1bda63         8000.024270cb7cc4       no
docker0         8000.0242c502c05b       no              veth7ab15e5
                                                        vethfedf9a3
​```
```

结果显示my-bridge 已经成功添加

新建test3，`--network`指定test3连接到my-bridge上面

```
docker run -d --name test3 --network my-bridge busybox /bin/sh -c "while true; do sleep 3600; done"
```

inspect my-bridge里面的container是否有test3

```
docker network inspect 8cfacc1bda63 #my-bridge的id
​```
......
 "Containers": {
            "7e1197d0de5fc3e9a5581ef1f51b8ba6a2b8989b1a0ca0431c8a2912c985374c": {
                "Name": "test3",
                "EndpointID": "c4a095ace6682bb816a63d88cd503ab8953ad3962398318e496a5825f4fe4117",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            }
        },
......
​```
```

证明test3已经成功的link到了my-bridge上面了。

此时，我们将连接到bridge docker0上面的test2，也连接到my-bridge上面，并inspect my-bridge

```
docker network connect my-bridge test2
docker network inspect 8cfacc1bda63
​```
"Containers": {
            "7e1197d0de5fc3e9a5581ef1f51b8ba6a2b8989b1a0ca0431c8a2912c985374c": {
                "Name": "test3",
                "EndpointID": "c4a095ace6682bb816a63d88cd503ab8953ad3962398318e496a5825f4fe4117",
                "MacAddress": "02:42:ac:12:00:02",
                "IPv4Address": "172.18.0.2/16",
                "IPv6Address": ""
            },
            "b17fc854ea1e63592e17ba4e64a39115fb4efb0002f6db452f72969b19601168": {
                "Name": "test2",
                "EndpointID": "fbf1dd0a9b185bf59d0d749f03bb4cd3f57e38ca9c4c7df1c414aa0aa7461514",
                "MacAddress": "02:42:ac:12:00:03",
                "IPv4Address": "172.18.0.3/16",
                "IPv6Address": ""
            }
        },

​```
```

结果显示test2也连接到了my-bridge上面。

test2即连接到了





**神奇的事情：**test2与test3之间能够用name ping通

```
docker exec test3 ping  test2
​```
PING test2 (172.18.0.3): 56 data bytes
64 bytes from 172.18.0.3: seq=0 ttl=64 time=0.122 ms
64 bytes from 172.18.0.3: seq=1 ttl=64 time=0.117 ms
^C
​```
```

因为test2和test3连接的是我们创建的bridge，而不是系统默认的bridge docker0。默认test2和test3已经link好了。





## host & none

