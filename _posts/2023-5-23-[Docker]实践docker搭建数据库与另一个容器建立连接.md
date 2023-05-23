

## 用Docker运行带有数据库的go服务

### 构建MySQL镜像

> Dockerfile      

```dockerfile
FROM mysql:latest

# docker-entrypoint-initdb.d 文件夹内的sql文件会用来初始化mysql
WORKDIR /docker-entrypoint-initdb.d

# 拷贝初始化文件到初始化文件夹 /docker-entrypoint-initdb.d
COPY ./init.sql .

# 暴露3306端口
EXPOSE 3306

# UTF-8编码
ENV LANG=C.UTF-8
```

> `docker build -f <mysql_Dockerfile> -t <mysql_image:tag> .`

### 运行mysql容器

> `docker run -it -d --name <mysql_container_name> -p 3306:3306 <mysql_image:tag> `

### 构建http登陆注册服务镜像

> Dockerfile

```dockerfile
FROM golang:1.17 as builder

WORKDIR /dockerhttp

COPY . .

ENV GO111MODULE=on \
    CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64 \
    GOPROXY=https://goproxy.cn,direct

RUN go mod download \
 && go build -o dockerhttp


FROM busybox

# 疑似不用，只需要暴露数据库的端口应该就行
EXPOSE 3306

COPY --from=builder /dockerhttp/dockerhttp /

ENTRYPOINT ["/dockerhttp"]
```

> `docker build -f <go-server_Dockerfile> -t <go-server_image:tag> .`      

### 运行go服务（新手大坑）

> docker run的时候记得加上 --link 参数        
>
> 同时go代码中的mysql dsn 的IP host也要改成容器名（--link 后面的参数）    
>
> 
>
> `docker run -it -d --name <go-server_container_name> --link <mysql_container_name>:3306 -p 3306:3306 <go-server_image:tag>`       
>
> dsn : `root:root@tcp(<mysql_container_name>:3306)/dockerhttp?charset=utf8mb4&parseTime=True&loc=Local`        