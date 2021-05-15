# Docker Getting started with Java
## 前提要求

* Java OpenJDK 版本在 15 以上。
* 安装 Docker。
* 安装 Git 客户端。


## 在本机运行项目
克隆项目源代码：

```sh
git clone https://github.com/spring-projects/spring-petclinic.git
cd spring-petclinic
```

在项目文件中包含了一个嵌入式的 Maven 版本，因此不需要在机器上单独安装 Maven。Maven 将管理所有的项目过程（编译，测试，打包等）。使用下面命令来启动项目：

```sh
./mvnw spring-boot:run
```

该命令将会下载依赖，构建项目，并且启动项目。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515105417.png)

命令执行完毕后，打开浏览器，输入 http://localhost:8080 访问项目。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515105315.png)

## 构建镜像
### 编写 Dockerfile
现在已经可以确认我们的应用程序可以在本机正常运行了，接下来我们编写一个 Dockerfile 文件用于构建镜像。

Dockerfile 是一个文本文档，包含了构建镜像需要调用的所有命令。Docker 读取 Dockerfile 中的命令并依次执行它们，每一条指令都会提交为一个镜像层，下一条指令都是基于上一条指令构建的。

在项目的根目录下创建名为 Dockerfile 的文件，文件内容如下：
```yaml
# syntax=docker/dockerfile:1

FROM openjdk:16-alpine3.13

WORKDIR /app

COPY .mvn/ .mvn
COPY mvnw pom.xml ./
RUN ./mvnw dependency:go-offline

COPY src ./src

CMD ["./mvnw", "spring-boot:run"]
```
现在解释一下每一行的作用：

* Dockerfile 的第一行是语法解析器指令，该指令指示 docker build 在解析 Dockerfile 时使用什么语法。**解释器指令可以不写，但是如果写了就必须出现在 Dockerfile 的第一行。**
解释器指令建议使用`docker/dockerfile:1`，因为它总是指向 version 1 语法的最新版本，BuildKit 会在构建镜像之前自动检测语法更新，确保使用的是最新版本。

* 指定构建的基础镜像，这里我们使用 openjdk 作为我们的基础镜像，上面已经安装的 maven 以及 java 应用程序所需要的依赖包：

```sh
FROM openjdk:16-alpine3.13
```
* 创建一个工作目录，Docker 后续的命令将使用此路径作为当前目录。相当于在容器中 `mkdir /app` 创建了一个目录，然后 `cd /app` 进入该目录。

```sh
WORKDIR /app
```

* 拷贝所需的文件到容器中：

```sh
COPY .mvn/ .mvn
COPY mvnw pom.xml ./
```

### 构建镜像
使用 `docker build` 命令构建镜像，指定镜像名为 java-docker，tag 为 v1.0.0：

```sh
docker build --tag java-docker:v1.0.0  .
```
![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515113137.png)

细心的同学可能会注意到 `docker build` 命令最后还有一个 `.`，`.`其实是指定了镜像构建过程中的上下文环境的目录。注意这个 `.` 并不是表示 Dockerfile 文件的路径，`-f` 参数才是用来指定 Dockerfile 的路径的（当 Dockerfile 名字为不为 Dockerfile/dockerfile 或者不在执行 docker build 命令的目录下，才需要使用 -f 参数指定 Dockerfile 文件）。

那么什么是上下文环境呢?
Docker 在运行时分为 Docker引擎（服务端守护进程） 以及 客户端工具，我们日常使用各种 docker 命令，其实就是在使用客户端工具与 Docker 引擎 进行交互。那么当我们使用 `docker build` 命令来构建镜像时，这个构建过程其实是在 Docker 引擎中完成的，而不是在本机环境。

那么如果在 Dockerfile 中使用了一些 COPY 等指令来操作文件，如何让 Docker引擎 获取到这些文件呢？
这里就有了一个镜像构建上下文的概念，当构建的时候，由用户指定构建镜像的上下文路径，而 `docker build` 会将这个路径下所有的文件都打包上传给 Docker 引擎，引擎内将这些内容展开后，就能获取到所有指定上下文中的文件了。比如说 Dockerfile 中的 COPY `./package.json /project`，其实拷贝的并不是本机目录下的 package.json 文件，而是 Docker 引擎中展开的构建上下文中的文件，所以如果拷贝的文件超出了构建上下文的范围，Docker引擎是找不到那些文件的。

查看构建好的镜像：

```sh
❯ docker images
REPOSITORY                                         TAG       IMAGE ID       CREATED          SIZE
java-docker                                        v1.0.0    f479e93b9881   22 minutes ago   574MB
```
## 启动容器

使用刚刚构建好的镜像来启动容器：
* -d：后台运行
* -p：将容器的端口映射到宿主机的端口，`-p [host port]:[container port]`
* --name：容器的名字
* 最后跟上镜像名

```sh
docker run -d -p 8080:8080 --name java-docker java-docker:v1.0.0
```

查看运行的容器：
```sh
❯ docker ps
CONTAINER ID   IMAGE                COMMAND                  CREATED         STATUS         PORTS                                       NAMES
21ed2e02c63f   java-docker:v1.0.0   "./mvnw spring-boot:…"   3 seconds ago   Up 2 seconds   0.0.0.0:8080->8080/tcp, :::8080->8080/tcp   java-docker
```

浏览器输入 http://localhost:8080 来访问应用程序：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515115118.png)

完成测试后，清理现场：

```sh
docker rm -f java-docker
```
## 构建本地开发环境
### 使用 Docker 部署服务
#### 在容器中运行数据库服务

首先创建两个 volume，用于持久化存储 MySQL 的数据和配置：

```sh
docker volume create mysql_data
docker volume create mysql_config
```
然后创建一个网络，应用程序和数据库的容器将使用该网络相互通信，该网络被称为用户自定义的桥接网络，在自定义的桥接网络中，容器之间可以使用 DNS 名称互相通信（Docker 默认自带的桥接网络不能使用 DNS 名称通信）。

```h
docker network create mysqlnet
```

启动数据库容器：
* -v：挂载 volume
* --network：指定使用的网络
* --name：容器名
* -e：设置环境变量，`MYSQL_ROOT_PASSWORD` 必须设置
* -p：将容器的端口映射到宿主机的端口，`-p [host port]:[container port]`
* 最后跟上镜像名


```sh
docker run -it --rm -d -v mysql_data:/var/lib/mysql \
-v mysql_config:/etc/mysql/conf.d \
--network mysqlnet \
--name mysqlserver \
-e MYSQL_USER=petclinic -e MYSQL_PASSWORD=petclinic \
-e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=petclinic \
-p 3306:3306 mysql:8.0.23
```
#### 在容器中运行应用服务
修改应用程序的 Dockerfile 文件，修改最后 CMD 的指令即可，修改后的文件如下：

```yaml
# syntax=docker/dockerfile:1

FROM openjdk:16-alpine3.13

WORKDIR /app

COPY .mvn/ .mvn
COPY mvnw pom.xml ./
RUN ./mvnw dependency:go-offline

COPY src ./src

CMD ["./mvnw", "spring-boot:run", "-Dspring-boot.run.profiles=mysql"]
```

构建镜像：

```sh
docker build --tag java-docker:v1.0.1 .
```

启动容器：

```sh
docker run --rm -d \
--name springboot-server \
--network mysqlnet \
-e MYSQL_URL=jdbc:mysql://mysqlserver/petclinic \
-p 8080:8080 java-docker:v1.0.1 
```

浏览器输入 http://localhost:8080 来访问应用程序：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515122332.png)

完成测试后，清理现场：

```sh
docker rm -f mysqlserver
docker rm -f springboot-server
docker volume rm mysql_data
docker volume rm mysql_config
docker network rm mysqlnet
```
### 使用 Docker Compose 部署服务
#### 安装 Docker Compose

```sh
curl -L https://get.daocloud.io/docker/compose/releases/download/1.24.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```
#### 编写 dockcer-compose.yaml 文件
```yaml
version: '3.8'
services:
  petclinic:
    build:
      context: .
    ports:
      - 8000:8000
      - 8080:8080
    networks:
      - mysqlnet
    environment:
      - SERVER_PORT=8080
      - MYSQL_URL=jdbc:mysql://mysqlserver/petclinic
    volumes:
      - ./:/app
    command: ./mvnw spring-boot:run -Dspring-boot.run.profiles=mysql -Dspring-boot.run.jvmArguments="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:8000"

  mysqlserver:
    image: mysql:8.0.23
    ports:
      - 3306:3306
    networks:
      - mysqlnet
    environment:
      - MYSQL_ROOT_PASSWORD=
      - MYSQL_ALLOW_EMPTY_PASSWORD=true
      - MYSQL_USER=petclinic
      - MYSQL_PASSWORD=petclinic
      - MYSQL_DATABASE=petclinic
    volumes:
      - mysql_data:/var/lib/mysql
      - mysql_config:/etc/mysql/conf.d
volumes:
  mysql_data:
  mysql_config:

networks:
  mysqlnet:
```
#### 通过 Docker Compose 启动服务
```sh
docker-compose -f docker-compose.dev.yml up --build
```
![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515123643.png)

```sh
❯ docker-compose -f docker-compose.dev.yml ps
             Name                           Command               State                                         Ports
------------------------------------------------------------------------------------------------------------------------------------------------------------
spring-petclinic_mysqlserver_1   docker-entrypoint.sh mysqld      Up      0.0.0.0:3306->3306/tcp,:::3306->3306/tcp, 33060/tcp
spring-petclinic_petclinic_1     ./mvnw spring-boot:run -Ds ...   Up      0.0.0.0:8000->8000/tcp,:::8000->8000/tcp, 0.0.0.0:8080->8080/tcp,:::8080->8080/tcp
```

使用以下命令来测试 API 接口：

```sh
curl  --request GET \
  --url http://localhost:8080/vets \
  --header 'content-type: application/json'
```

你应该能看到以下返回结果：

```sh
{"vetList":[{"id":1,"firstName":"James","lastName":"Carter","specialties":[],"nrOfSpecialties":0,"new":false},{"id":2,"firstName":"Helen","lastName":"Leary","specialties":[{"id":1,"name":"radiology","new":false}],"nrOfSpecialties":1,"new":false},{"id":3,"firstName":"Linda","lastName":"Douglas","specialties":[{"id":3,"name":"dentistry","new":false},{"id":2,"name":"surgery","new":false}],"nrOfSpecialties":2,"new":false},{"id":4,"firstName":"Rafael","lastName":"Ortega","specialties":[{"id":2,"name":"surgery","new":false}],"nrOfSpecialties":1,"new":false},{"id":5,"firstName":"Henry","lastName":"Stevens","specialties":[{"id":1,"name":"radiology","new":false}],"nrOfSpecialties":1,"new":false},{"id":6,"firstName":"Sharon","lastName":"Jenkins","specialties":[],"nrOfSpecialties":0,"new":false}]}%
```
### 连接调试器
![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515123937.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515124003.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515124231.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515124349.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515124435.png)


```sh
curl --request GET --url http://localhost:8080/vets
```
![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515124528.png)

测试完成后，清理现场：

```sh
docker-compose -f docker-compose.dev.yml down
```
## 单元测试

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515125540.png)

### 运行单元测试
```sh
docker run -it --rm --name springboot-test java-docker:v1.0.1 ./mvnw test
```

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515125954.png)

### 多阶段构建

```yaml
# syntax=docker/dockerfile:1

FROM openjdk:16-alpine3.13 as base

WORKDIR /app

COPY .mvn/ .mvn
COPY mvnw pom.xml ./
RUN ./mvnw dependency:go-offline
COPY src ./src

FROM base as test #test构建阶段，名字自定义
CMD ["./mvnw", "test"]

FROM base as development
CMD ["./mvnw", "spring-boot:run", "-Dspring-boot.run.profiles=mysql", "-Dspring-boot.run.jvmArguments='-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:8000'"]

FROM base as build
RUN ./mvnw package

FROM openjdk:11-jre-slim as production
EXPOSE 8080

COPY --from=build /app/target/spring-petclinic-*.jar /spring-petclinic.jar

CMD ["java", "-Djava.security.egd=file:/dev/./urandom", "-jar", "/spring-petclinic.jar"]
```

这次我们在构建镜像的时候添加了 --target 参数，表示只运行 test 这个构建阶段。

```sh
docker build -t java-docker:v1.0.2 --target test .
```
根据输出结果可以看到，在 test 阶段后面的部分并没有执行：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515130348.png)

启动容器来运行单元测试：
```sh
docker run -it --rm --name springboot-test java-docker:v1.0.2
```

完成单元测试后，输出结果如下：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515130759.png)

上面的步骤稍微有一些麻烦，我们先构建了镜像，然后再通过镜像启动容器来完成单元测试，我们可以通过修改 Dockerfile 文件来做一些优化：

```yaml
# syntax=docker/dockerfile:1

FROM openjdk:16-alpine3.13 as base

WORKDIR /app

COPY .mvn/ .mvn
COPY mvnw pom.xml ./
RUN ./mvnw dependency:go-offline
COPY src ./src

FROM base as test #test构建阶段，名字自定义
RUN ["./mvnw", "test"] #将CMD改为RUN

FROM base as development
CMD ["./mvnw", "spring-boot:run", "-Dspring-boot.run.profiles=mysql", "-Dspring-boot.run.jvmArguments='-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:8000'"]

FROM base as build
RUN ./mvnw package

FROM openjdk:11-jre-slim as production
EXPOSE 8080

COPY --from=build /app/target/spring-petclinic-*.jar /spring-petclinic.jar

CMD ["java", "-Djava.security.egd=file:/dev/./urandom", "-jar", "/spring-petclinic.jar"]
```
CMD 指令是在启动容器时执行的，在构建镜像期间不会执行，我们可以将单元测试的指令改成 RUN，RUN指令在构建镜像的时候运行，并且在失败的时候会停止构建。这样我们只需要使用 `docker build` 命令在构建镜像的时候就完成的了单元测试：


```sh
docker build -t java-docker:v1.0.3 --target test .
```

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515131811.png)



![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515131856.png)

```sh
docker build -t java-docker:v1.0.4 --target test .
```

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515132122.png)


```sh
docker build -t java-docker:v1.0.5 --target production  .
```
## CI/CD
![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515132759.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515132834.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515133359.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515133442.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515133552.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515133754.png)
## 参考链接
* https://www.cnblogs.com/panpanwelcome/p/12603701.html
* https://docs.docker.com/language/java/

