## Dockerfile语法梳理及最佳实践

### Docker image 原理：

1.Docker image的本质是什么？

- 是一个分层文件系统

2.Docker中一个Centos image为什么只有200MB，而一个centos操作系统的iso要几个GB？

- centos的iso image文件包含bootfs和rootfs，而docker的centos image复用操作系统的bootfs，只有rootfs和其他image layer

3.Docker中一个tomcat image  为什么有500MB，而一个tomcat安装包只有70多MB？

- 由于docker中image时分层的，tomcat虽然只有70多MB，但需要依赖于父镜像和基础镜像，所有整个对外暴露的tomcat image大小500多MB。

### FROM 指定初始image

- FROM scratch  #制作base image
- FROM centos   #使用base image
- FROM ubuntu:14.04

**FROM 尽量使用官方的image作为base image**

scratch镜像是一个完全空白的文件系统

### LABEL 定义image的metadata

- LABEL maintainer="username@gmail.com"  #例如：可以在里面包含image作者
- LABEL version="1.0"  #版本
- LABEL description="This is description" #描述信息

**LABEL Metadata不可少**

### RUN 指定的shell命令，是要在image里执行的

- ```
  RUN yum update && yum install -y vim \
  
  python-dev  #反斜杠换行
  ```

- ```
  RUN apt-get update && apt-get install -y perl \
  
  pwgen --no-install-recommends && rm -rf \
  
  /var/lib/apt/lists/*  #注意清理cache
  ```

- ```
RUN /bin/bash -c 'source' $HOME/.bashrc;echo $HOME'
  ```

**RUN 为了美观，复杂的RUN请用反斜线换行！避免无用分层，合并多条命令成一行！**

### WORKDIR 设定当前工作目录

- WORKDIR /test   #如果没有会自动创建test目录
- WORKDIR demo
- RUN pwd            #输出结果应该是/test/demo

**WORKDIR：用WORKDIR，不要用RUN cd！尽量使用绝对目录！**

### ADD and COPY  把本地文件添加到docker image 里面

**ADD能解压**

- ```
  ADD hello /
  ```

- ````
  ADD test.tar.gz / #添加到根目录并解压
  ```

- ```
  WORKDIR /root    #指定当前目录是根目录下的root 
  ADD hello test/   # 把hello添加到 test目录。结果应该是  /root/test/hello
  ```

- ```
  WORKDIR /rooy
  COPY hello test/   
  ```

**COPY**（推荐使用）

- COPY src dest 以及 COPY ["src", "dest"]
- 如果路径中有空格的话，那么必须使用 JSON 数组的格

**ADD or COPY ：大部分情况下，COPY优于ADD！ADD除了COPY还有额外的功能（解压）！添加远程文件/目录请使用RUN指令配合curl或者wget下载到docker image 里面!**



### ENV 设置镜像内的环境变量。这些变量可以被随后的指令引用

- ```
ENV MYSQL_VERSION 5.6 #设置常量MYSQL_VERSION
  
  RUN apt-get install -y mysql-server= "${MYSQL_VERSION}" && rm -rf /var/lib/apt/lists/* #引用常量5.6
  ```

比如版本号需要变更时，只需要改ENV即可

**ENV：尽量使用ENV，能增加可维护性！**

### EXPOSE 用来暴露端口

向 Docker 表示该容器将会有一个进程监听所指定的端口。提供这个信息的目的是用于**连接容器或在执行 docker run 命令时通过 -P 参数把端口发布开来**； EXPOSE 指令本身并不会对网络有实质性的改变。



### RUN vs CMD vs ENTRYPOINT

- RUN：执行命令并创建新的Image Layer(每执行一次都会创建新的layer)
- CMD：设置容器启动后默认执行的命令和参数
- ENTRYPOINT：设置容器启动时运行的命令

RUN、CMD和ENTRYPOINT能够接受shell和exec这两种格式。

关于image laer：Dockerﬁle 中的每个指令 执行后都会产生一个新的image layer，而这个image layer其实可以用来启动container。一个新的image layer的建立，是用上一层的image启动container，然后执行 Dockerﬁle 的指令后，把它保存为一个 新的image。当 Dockerﬁle 指令成功执行后，中间使用过的那个container会被删掉。

### Shell和Exec格式对比

**Shell**

- Shell 格式使用的是**自由形式的字符串**，字符串会传给 /bin/sh -c 执行

**Exec**

- 需要用到一个JSON 数组。（例如，["executable", "param1", "param2"]）
- 其中第一个元素是一个可执行文件，其他元素是它执行时所使用的参数

#### Shell 格式——赶紧学LinuxShell编程

把我们命令当成一个shell命令去执行。

```
RUN apt-get install -y vim

CMD echo "hello docker"

ENTRYPOINT echo "hello docker"
```

#### Exec格式

```
RUN ["apt-get","install","-y","vim"]

CMD ["/bin/echo","hello docker"]

ENTRYPOINT ["/bin/echo","hello docker"]
```

#### 例子：

**Dockerfile1——shell格式** 

```
FROM centos

ENV name Docker

ENTRYPOINT echo "hello $name"
```

**Dockerfile2——Exec格式**

```
FROM centos

ENV name Docker 

ENTRYPOINT ["/bin/echo","hello $name"]
```

#### 演习：

- **Shell格式：**

```
cd  #回到家目录
mkdir cmd_vs_entrypoint  #在当前目录下创建directory
touch Dockerfile  #生成Dockerfile
vim Dockerfile    #修改Dockerfile内容

#加入以下内容
FROM centos
ENV name Docker
ENTRYPOINT echo "hello $name" #打印ENV指定的名字
#

docker build -t username/centos-entrypoint-shell . #在当前目录下Dockerfile创建image
#               image仓库/image名称             Dokcerfile位置

docker run username/centos-entrypoint-shell

​```
hello docker
​```
```

- **下面将Dockerfile里面entrypoint改为Exec格式**

```
#在Dockerfile中输入
FROM centos
ENV name Docker
ENTRYPOINT ["/bin/echo","hello $name"]
```

```
#build一个新的image
docker build -t username/centos-entrypoint-exec . #在当前目录下根据Dockerfile创建image
docker run username/centos-entrypoint-exec
​```
hello $name
​```
```

结果显示name并没有被替换成ENV所指定的内容

原因：

- Shell格式执行时：会默认通过shell（比如说bash）去执行这个命令，
- Exec格式执行时：单纯执行echo这个命令，（并不是执行shell，没有在shell里面去执行命令）没法替换name

解决：**赶紧学Linux Shell编程**

增加"/bin/bash"变成在bash里面执行命令，后面所有的就都为参数

```
FROM centos
ENV name Docker
ENTRYPOINT["/bin/bash","-c","echo hello $name"] 
#-c指明后面是我们需要执行的参数，后面所有命令要合并成一个去执行
```



#### CMD 

- 容器启动时默认执行的命令
- **如果docker run 指定了其他命令，CMD命令被忽略**
- **如果定义了多个CMD，只有最后一个会执行**

```
FROM centos
ENV name Docker
CMD echo "hello $name"
```

##### 一个指定其他命令的例子：

```
docker bulid -t username/centos-cmd-shell .
#               image仓库名/image名称    根据当前目录Dockerfile创建

docker run -it username/centos-cmd-shell /bin/bash #执行"/bin/bash"命令进入到container里面，
```

#### ENTRYPOINT (用的最多)

- 让容器也应用程序或者服务的形式运行
- **不会被忽略，一定会执行**
- 最佳实践：写一个shell脚本作为entrypoint

例子：moongod官方Dockerfile的部分内容

```
COPY docker-entrypoint.sh /usr/local/bin/ #把shell脚本copy到本地
ENTRYPOINT ["docker-entrypoint.sh"]  #把该脚本作为ENTRYPOINT启动的脚本

EXPOSE 27017  
CMD ["mongod"]
```

创建一个Dockerfile，并创建image

```
FROM centos
ENV name Docker
ENTRYPOINT echo "hello $name"
```

```
build -t username/centos-entrypoint-shell .

docker run username/centos-entrypoint-shell
​```
hello docker
​```

docker run username/centos-entrypoint-shell /bin/bash #指定其他命令还是会运行ENTRYPOINT内容
​```
hello docker
​```
```



