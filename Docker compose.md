# Docker compose

## Docker Compose是什么

- Dokcer Compose 是一个工具
- 这个工具可以通过一个yaml文件定义多容器的docker应用
- 通过一条命令就可以根据yaml文件的定义去创建或者管理这多个容器

## docker-compose.yml文件三大概念

- Services：

  - 一个servieces代表一个container，这个container可以从dockerhub的iamge来创建，或者从本地的Dockerfile build出来的image来创建

  - service的启动类似docker run，我们可以给其指定network和volume，所以可以给service指定network和volume的引用

  例1：

    ```yaml
  services:
    db:                          #contianer名字为db
      image: postgres:9.4        #从dockerhub上面pull来的
      volumes:
        - "db-data:/var/lib/postgresql/data"
      networks:
        - back-tier
          
          
    ```

  等价于

    ```shell
  docker run -d --network back-tier -v db-data：/var/lib/postgresql/data postgres:9.4
    ```

   例2

    ```yaml
  services:
    worker:              #contianer的名称为worker
      build: ./worker    #在本地build image (./worker是Dokcerfile所在的location)
      links:             #worker会和哪几个容器做link
        - db
        - redis
      networks:          
        - back-tier
    ```

    

- Networks：

  在services同级别定义

  ```yaml
  networks:
    front-tier:
      driver: bridge
    back-tier:
      driver: bridge
  ```

  

- Volumes

  在servicess同级别定义

  ```
  volumes:
    db-data:
  ```




## docker-compose使用

- `docker-compose up`:启动，默认会显示所有log
- `docker-compose up -d`:后台执行，不会显示log
- `docker-compose ps`



例子：

Dockerfile

```


```

docker-compose.yml 文件

```yaml
version: "3"

services:
  #定义两个container
  redis:                
    image: redis        #从dockerhub上面pull
    
  web:
    build:   #通过传入Dockerfile去build image
      context: .  #Dockerfile的位置
      dockerfile: Dockerfile  #Dockerfile的名字
    ports:   
      - 8080:5000
    environment:
      REDIS_HOST: redis
```

