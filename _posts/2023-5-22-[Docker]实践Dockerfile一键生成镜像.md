## Dockerfile实践



### 第一次实践（构建一个goland镜像并且运行一个http接口）以及问题总结

> Dockerfile      

```dockerfile
FROM golang:1.17 as builder

MAINTAINER XiaoTe33

# 这里注意WORKDIR不能设置成 .
#  . 是$GOPATH的目录，如果设置为工作目录go mod download会报错
WORKDIR /build

# 拷贝代码文件
COPY ./main.go .
COPY ./go.mod .
COPY ./go.sum .

# 设置环境变量
ENV GO111MODULE=on \
    CGO_ENABLED=0 \
    GOOS=linux \
    GOARCH=amd64 \
    GOPROXY=https://goproxy.cn,direct

# 下载依赖并且go build一下
RUN go mod download \
  && go build -o app .

# 新建一个busybox（一个小型linux）镜像
FROM busybox

# 拷贝刚刚go build出来的可执行文件到新的linux系统里
COPY --from=builder /build/app /

# 运行
ENTRYPOINT ["/app"]
```

> `docker build`一下      
>
> 运行时记得加端口映射`docker run -p src:dst`      

### 小结

熟悉Dockerfile文件的编写，踩了一些坑，最后成功运行
