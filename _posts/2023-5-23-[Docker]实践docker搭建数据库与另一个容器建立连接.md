

## 用Docker运行带有数据库的go服务

### :heart:构建MySQL镜像

> Dockerfile      

```dockerfile
FROM mysql:latest

# /docker-entrypoint-initdb.d 文件夹内的sql文件会用来初始化mysql
WORKDIR /docker-entrypoint-initdb.d

# 拷贝初始化文件到初始化文件夹 /docker-entrypoint-initdb.d
COPY ./init.sql .

# 暴露3306端口
EXPOSE 3306

# UTF-8编码
ENV LANG=C.UTF-8
```

> `docker build -f <mysql_Dockerfile> -t <mysql_image:tag> .`

### :blue_heart:运行mysql容器

> `docker run -it -d --name <mysql_container_name> -p 3306:3306 <mysql_image:tag> `

### :green_heart:构建http登陆注册服务镜像

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

### :broken_heart:运行go服务（感觉是新手大坑）:exclamation::exclamation::exclamation:

> docker run的时候加上 --link 参数        
>
> 同时go代码中的mysql dsn 的IP host也要改成容器名（--link 后面的参数）    
>
> 
>
> :rocket:`docker run -it -d --name <go-server_container_name> --link <mysql_container_name>:3306 -p 3306:3306 <go-server_image:tag>`       
>
> :rocket:dsn : `root:root@tcp(<mysql_container_name>:3306)/dockerhttp?charset=utf8mb4&parseTime=True&loc=Local`        

### :yellow_heart:服务端代码

```go
package main

import (
	"fmt"
	"github.com/gin-gonic/gin"
	"gorm.io/driver/mysql"
	"gorm.io/gorm"
)

var gormDB *gorm.DB

type User struct {
	Username string `gorm:"column:username;unique;not null" json:"username"`
	Password string `gorm:"column:password;not null" json:"password"`
}

func main() {
	initDB()
	initGin()

}

func initGin() {
	r := gin.Default()
	r.GET("/register", register)
	r.GET("/login", login)
	r.Run()
}

func initDB() {
	dsn := "root:root@tcp(127.0.0.1:3306)/dockerhttp?charset=utf8mb4&parseTime=True&loc=Local"
	db, err := gorm.Open(mysql.Open(dsn), &gorm.Config{})
	if err != nil {
		panic("mysql init failed, err: " + err.Error())
	}
	fmt.Println("mysql init successfully!")
	err = db.AutoMigrate(&User{})
	if err != nil {
		panic("AutoMigrate err! err: " + err.Error())
	}
	gormDB = db
}

func register(c *gin.Context) {
	u := c.Query("username")
	p := c.Query("password")
	if u == "" {
		jsonMsg(c, "用户名不能为空")
	}
	if p == "" {
		jsonMsg(c, "密码不能为空")
	}
	if handleError(c, gormDB.Create(&User{u, p}).Error) {
		return
	}
	jsonSuccess(c)
}

func login(c *gin.Context) {
	u := c.Query("username")
	p := c.Query("password")
	if u == "" {
		jsonMsg(c, "用户名不能为空")
	}
	if p == "" {
		jsonMsg(c, "密码不能为空")
	}
	user := User{}
	gormDB.Where("username = ? and password = ?", u, p).Find(&user)
	if user == (User{}) {
		jsonMsg(c, "用户名或者密码错误")
		return
	}
	jsonSuccess(c)
}

func jsonMsg(c *gin.Context, msg string) {
	c.JSON(200, gin.H{
		"status": 400,
		"msg":    msg,
	})
}

func jsonSuccess(c *gin.Context) {
	c.JSON(200, gin.H{
		"status": 200,
		"msg":    "操作成功",
	})
}

func jsonError(c *gin.Context, err error) {
	jsonMsg(c, err.Error())
}

func handleError(c *gin.Context, err error) bool {
	if err == nil {
		return false
	}
	jsonError(c, err)
	return true
}
```


