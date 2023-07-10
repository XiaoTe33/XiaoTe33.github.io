## 编写docker-compose.yaml文件

```yaml
# 版本号
version: "3"
services:
# 以下为两个sevice的名称其中server镜像的构建未命名将会默认以<dir>-server命名
# server容器未命名将会默认以<dir>-server-1命名
    server:
        build: .
        networks:
            - mynetwork1
        ports:
            - 1010:1010
# 确定依赖关系，让db先启动
        depends_on:
            - db
        volumes:
            - ./output:/output
 # db容器未命名将会默认以<dir>-db-1命名
 # 由于mysql连接使用容器名字（服务名或许也可以？）所以server连接mysql时用的IP应该与容器名保持一致<dir>-db-1
    db:
        image: my-mysql:1.0
        environment:
            - MYSQL_ROOT_PASSWORD=root
        networks:
            - mynetwork1
        ports:
            - 3306:3306
# 网络未命名，将会以默认<dir>-mynetwork1命名
networks:
    mynetwork1:
```

## 踩坑

```yaml
# 这一操作虽然首先启动了mysql但是由于mysql初始化时间过长server启动时mysql依然未正常运行，导致连接失败
depends_on:
        - db
```

### 解决办法

- 在连接mysql的init函数中实现重试与超时机制

```go
var GDB *gorm.DB

func ConnectGorm() {
	var db *gorm.DB
	var err error
	success := make(chan struct{}, 1)
    //设置超时时间30秒
	timeout, cancelFunc := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancelFunc()
	go func() {
        //连接失败则每两秒重试一次
		for {
			db, err = gorm.Open(mysql.Open(getMysqlDSN()), &gorm.Config{})
			if err != nil {
				mylog.Log.Error("gorm: mysql initialized failed,try again...")
				time.Sleep(2 * time.Second)
				continue
			} else {
				success <- struct{}{}
				mylog.Log.Println("gorm: mysql initializes successfully")
				return
			}
		}
	}()
	select {
    //超时则报panic
	case <-timeout.Done():
		panic("gorm: mysql initialized timeout!")
    //成功则继续完成初始化
	case <-success:
	}
    GDB=db
}
```

