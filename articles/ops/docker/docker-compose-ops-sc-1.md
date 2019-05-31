> 本文意在基于Docker-compose部署微服务，不会聊Docker和SpringCloud实现细节，所以读本文前你要对Docker和SpringCloud有一个基础的认识。
## 简单的单机服务架构部署
### 1. 简单的架构图
![](https://user-gold-cdn.xitu.io/2019/5/19/16acd9e5bf10b94e?w=1486&h=906&f=png&s=84937)
图中一共有四个小应用:
- ``nacos``阿里开源的服务发现
- ``fp-gateway``网关服务
- ``fp-api`` api服务
- ``fp-user`` 用户服务 
### 2. 构建服务镜像
>  通过简单的架构图可以了解到，``fp-gateway``、``fp-api``、``fp-user``是我们基于``SpringCloud``构建的应用,并将其注册给``nacos``。``nacos``我们直接使用官方的镜像即可，剩下这三个应用需编写镜像。
#### 2.1 fp服务镜像编写
##### Dockerfile：三个服务均使用这个
```
FROM java:8
# 将运行脚本加入到容器中
COPY startup.sh /shell/startup.sh
# 赋予运行权限
RUN chmod +x /shell/startup.sh
# 指定容器时区:东八区
ENV TZ=Asia/Shanghai 
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone
# 创建jar包目录
RUN mkdir jar
# 挂载 jar目录
VOLUME ["/jar"]
# 进行运行脚本
CMD [ "sh","/shell/startup.sh" ]
```
##### startup.sh：区别只是在于server_name
```
#!/bin/bash 
# 三个服务镜像时，只需改这个值gateway->api/user
server_name="fp-gateway"
kill -s 9 `ps -aux | grep $server_name | awk '{print $2}'`
# 切换到jar目录启动，目的配置外置
cd /jar
# 启动
java -jar -Duser.timezone=GMT+08 -XX:+HeapDumpOnOutOfMemoryError -Xms512m -Xmx512m $server_name.jar
```
#### 2.2 开始构建镜像
##### 先将三个服务需要的镜像和shell准备好
![](https://user-gold-cdn.xitu.io/2019/5/19/16acdc2c6d680c9c?w=664&h=318&f=png&s=29969)
##### 开始构建三个服务镜像
```
docker build -t fp-api:v1 ./fp-api/
docker build -t fp-gateway:v1 ./fp-gateway/
docker build -t fp-user:v1 ./fp-user/
```
```
# 执行docker images 出现如下图情况，说明镜像构建成功
docker images
```
![](https://user-gold-cdn.xitu.io/2019/5/19/16acdc84a5d5b074?w=1258&h=126&f=png&s=44298)
### 3 编写docker-compose
#### 3.1 安装docker-compose
```
curl -L "https://github.com/docker/compose/releases/download/1.24.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```
```
docker-compose -version
```
![](https://user-gold-cdn.xitu.io/2019/5/19/16ace0a783d58f97?w=614&h=28&f=png&s=9882)
#### 3.2 编写standalone.yml
```
version: "3.5"
services:
  fp-api:
    image: fp-api:v1
    container_name: fp-api
    volumes:
      - /docker/fp/services/fp-api:/jar
    ports:
      - "10001:10001"  
    depends_on:
      - fp-gateway  
  fp-user:
    image: fp-user:v1
    container_name: fp-user
    volumes:
      - /docker/fp/services/fp-user:/jar
    ports:
      - "10002:10002"
    depends_on:
      - fp-gateway  
  fp-gateway:
    image: fp-gateway:v1
    container_name: fp-gateway
    volumes:
      - /docker/fp/services/fp-gateway:/jar
    ports:
      - "10003:10003"
```
#### 3.3 打包jar和nacos初始化配置
>`nacos`这一块每个team选择的服务发现是不一样的，这一块很灵活，所以略过它的部署过程。
```
fp/
├── builds
│   ├── fp-api
│   │   ├── Dockerfile
│   │   └── startup.sh
│   ├── fp-gateway
│   │   ├── Dockerfile
│   │   └── startup.sh
│   └── fp-user
│       ├── Dockerfile
│       └── startup.sh
├── services
│   ├── fp-api
│   │   ├── config
│   │   │   └── application.yml
│   │   └── fp-api.jar
│   ├── fp-gateway
│   │   ├── config
│   │   │   └── application.yml
│   │   └── fp-gateway.jar
│   └── fp-user
│       ├── config
│       │   └── application.yml
│       └── fp-user.jar
└── yaml
    └── standalone.yml
```
### 4 启动服务
```
docker-compose -f ./yaml/standalone.yml up -d
```
#### 4.1 启动成功后，检查容器是否启动
```
docker ps
```
![](https://user-gold-cdn.xitu.io/2019/5/19/16ace5d59f743d3a?w=1936&h=124&f=png&s=65149)
#### 4.2 通过fp-gateway访问fp-api服务
> 在``fp-api``服务调用``fp-user``服务个接口。
```
@RestController
public class UserController {

    @GetMapping("user")
    public String user() {
        return "welcome to user app.";
    }

}
```
```
@FeignClient(value = "footprint-user")
public interface UserFeign {

    @GetMapping("user")
    public String user();

}
```
```
@RestController
public class ApiController {

    @Autowired
    private UserFeign userFeign;

    @GetMapping("test")
    public String test() {
        return userFeign.user();
    }

}

```
#### 访问api服务test接口
```
curl http://192.168.0.4:10003/fp/api/test
```
![](https://user-gold-cdn.xitu.io/2019/5/19/16ace6554c97b632?w=876&h=56&f=png&s=32934)