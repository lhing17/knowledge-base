# docker常用命令

## 容器相关操作
### 典型的docker启动命令
```docker
docker run -d \
    --name=xxx \
    -p 8080:8080 \
    -v /home/Software/myapp.war:/usr/local/myapp/myapp.war \
    tomcat:latest
``` 

### 查看docker容器列表
```docker
docker ps -a
```

### 进入docker容器
```docker
docker exec -it 容器ID /bin/bash
```

### 退出docker容器
```docker
exit
```

### 停止docker容器
```docker
docker stop 容器ID
```

### 删除docker容器
```docker
docker rm 容器ID
```

### 查看日志
```docker
docker logs -f 容器ID
```

## 镜像相关操作
### 查看镜像列表
```docker
docker images
```

### 拉取镜像
```docker
docker pull tomcat:latest
```

