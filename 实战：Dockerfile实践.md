# Dockerfile实践

Python程序打包成docker image，并运行这个image

### 首先创建一个python app小程序

```python
from flask import Flask
app = Flask(__name__)
@app.route('/')
def hello():
    return "hello docker"

```



### 定义Dockerfile

选择安装好python2.7的base image

```
FROM python:2.7     #base image
LABEL "maintainer=username <user@mail.com>"
RUN pip install flask   #安装flask（执行命令并创建新的Image Layer）
COPY app.py  /app/   #在根目录下创建app目录，把app.py复制到里面
WORKDIR /app      #设置工作目录为之前创建的/app(因为app.py在里面)
EXPOSE  5000      #暴露端口
CMD ["python","app.py"]  #这次使用Exec格式
```

注: `COPY app.py /app`会报错，原因：该命令意思是把app.py copy到根目录下面，并改名为app。所以app不是一个目录

### 创建image

```
docker build -t username/flask-hello-world .

```

### 通过image创建container

```
docker run username/flask-hello-world
​```
Running on http://127.0.0.1:5000/(Press CTRL+C to quit)
​```
```

证明container已经在运行了

#### 让container在后台执行  -d

```
docker run -d username/flask-hello-world
```



## 对运行中的container能进行的操作

### docker exec ：对运行中的container执行命令

例如：让运行中的contianer进入到shell里面

```
docker exec -it container_id /bin/bash  #container_id是例如11a767sd3a558
```

例如：打印出运行中的container的ip address

```
docker exec -it container_id ip a
```

### docker stop ：停止运行contianer

```
docker ps  #查看当前运行的contianer 的id
docker stop container_id
```

### docker rm:清理进程

清理所有退出状态的容器

```
docker rm $(docker ps -aq)
```

### 给container设置名字(和id一样具有唯一性)

通过添加 `--name` 参数

```
docker run -d --name=demo username/flask-hello-world  #把该container名字设置为demo
```

#### 指定名字之后可以之间通过输入名字来stop或start

```
docker stop demo
```

```
docker start demo
```

### docker inspect：显示container详细信息

```
docker inspect container_id 
```

### docker logs：查看logs（日志）

```
docker logs container_id
```



### 更多container指令看官网



## 通过Dockerfile去build一个命令行工具

运行这个工具可以通过指定参数来得到不同的结果

首先进入ubantu的container

```
docker run -it ubuntu
```

安装stress（stress是用来对Linux系统进行压力测试的工具）

```
apt-get update && apt-get install -y stress
```

 首先查看stress

```
which stress
```



```
stress --help
```



```
stress --vm 1 
```



### 通过创建Dokcerfile创建stress的image

#### ENTRYPOINT和CMD的组合使用

同时有ENTRYPOINT和CMD时，CMD只为ENTRYPOINT提供参数

```
FROM ubuntu
RUN apt-get update && apt-get install -y stress
ENTRYPOINT ["/usr/bin/stress"]
CMD [] #用来接收通过docker run 给它传入的参数
```



```
docker bulid -t username/ubuntu-stress
docker run -it username/ubuntu-stress
​```
打印出
···

docker run -it username/ubuntu-stress --vm 1  #加上--vm 1参数创建一个task



```

