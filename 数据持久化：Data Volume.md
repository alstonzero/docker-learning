## 数据持久化：Data Volume  

Docker container删除后，在container中产生的数据也会随之销毁。但绑定了data volume只后data会永久存在。

**数据卷Volume:是宿主机（host）中的一个目录或者文件**

data volume作用：

- container 的data的持久化
- host 和 container之间data交换
- container之间的data交换

### Dockerfile中，添加VOLUME参数

通过Dokcerfile里面的**VOLUME**关键字，去指定container里面的某一个directory里产生的data同时挂载到host上某个directory上面，并且创建一个叫Docker Volume的对象

方法一：在Dockerfile中，添加VOLUME参数

```
VOLUME["/var/lib/mysql"]
```

### 数据持久化之bind Mount

- docker run 添加`-v`参数。

- `-v`比起`-mount`更简洁，且不存在目录的会自动创建

- 通过`-v`指定本地目录和container目录的**一一对应**关系。因此本地目录下文件被修改，在contaienr中也会被同样修改。

```
docker run -v /home/aaa:/root/aaa  #把主机中/home/aaa挂载到container的/root/aaa中去
```

Docker 挂载数据卷的默认权限是读写，用户也可以通过 `:ro` 指定为只读。

```
docker run -v /home/aaa:/root/aaa:ro
```

若`：`前面没有指定宿主机的路径，则会随机分配



把host下面的mysql目录绑定到container下面的 /var/lib/mysql

```
docker run -v mysql:/var/lib/mysql  
```

当前目录$PWD

```
docker run -v $PWD/mysql:/var/lib/mysql
```



#### 实践：

首先创建一个container  c1

```
$ docker run -it --name=c1 -v $PWD/volumetest:/root/container_data centos /bin/bash

[root@67ae6d041c99 container_data]# ip addr >>1.txt

```

#### 实践2：两个container挂载同一个volume





### container volume 数据卷容器

当需要多个contianer都挂载到同一个volume时，避免重复设置。

原理：一个container挂载volume称为container volume，而其他container都挂载到这个container volume上。而使得其他container都挂载到宿主机的同一目录下。

1，创建启动c3数据卷容器，使用`-v`参数设置volume 

```
docker run -it --name=c3 -v /volume centos /bin/bash
```

由于没有设置`:`左侧宿主机对应的目录（Source），从而被随机分配路径。可以用`docker inspect c3`查看

2，创建启动c1 c2容器，使用`--volumes-from `参数设置data volume（挂载到c3上）即都挂载到同一个volume上

``` 
docker run -it --name=c1 --volumes-from c3 centos /bin/bash
docker run -it --name=c2 --volumes-from c3 centos /bin/bash
```

通过`docker inspect`可以发现，c1，c2，c3都挂载到了宿主机的同一个目录下

```
docker inspect c1



docker inspect c2
```

