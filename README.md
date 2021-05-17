# Docker Getting started with Java

Docker 官网提供了 python，nodejs，java 3种不同编程语言的 [Language-specific guides](https://docs.docker.com/language/) 学习指南。该指南详细说明了如何编写 Dockerfile 文件，部署 Docker 容器以及构建 CI/CD pipline。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210516192251.png)

本文介绍该指南了Java 部分的内容。

## 前提要求

* Java OpenJDK 版本在 15 以上。
* 安装 Docker。
* 安装 Git 客户端。
* IntelliJ IDEA 客户端.

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

mvnw 全名是 Maven Wrapper,它的原理是在 maven-wrapper.properties 文件中记录你要使用的 Maven 版本，当用户执行 mvnw 命令时，如果发现当前用户的 Maven 版本和期望的版本不一致，那么就下载期望的版本，然后用期望的版本来执行 mvn 命令。

该命令将会下载依赖，构建项目，并且启动项目。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515105417.png)

命令执行完毕后，打开浏览器，输入 http://localhost:8080 访问项目。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515105315.png)

## 快速开始
现在已经可以确认我们的应用程序可以在本机正常运行了，接下来我们通过一个简单的示例将该项目构建为 Docker 镜像，然后用镜像运行容器。
### 构建镜像
#### 编写 Dockerfile

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

* 指定构建的基础镜像，这里我们使用 openjdk 作为我们的基础镜像，上面已经安装的 maven 以及 Java 应用程序所需要的依赖包：

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
* 在构建镜像时运行命令，拷贝 pom.xml 和 mvnw 文件到容器中，就可以运行下面的命令下载所需要 maven 依赖：

```sh
RUN ./mvnw dependency:go-offline
```
*  拷贝项目源代码到容器中：

```sh
COPY src ./src
```

* 容器启动时执行的命令，该命令在构建镜像时不会执行：

```sh
CMD ["./mvnw", "spring-boot:run"]
```
#### 构建镜像
使用 `docker build` 命令构建镜像，指定镜像名为 java-docker，tag 为 v1.0.0：

```sh
docker build --tag java-docker:v1.0.0  .
```
![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515113137.png)

细心的同学可能会注意到 `docker build` 命令最后还有一个 `.`，`.`其实是指定了镜像构建过程中的上下文环境的目录。注意这个 `.` 并不是表示 Dockerfile 文件的路径，`-f` 参数才是用来指定 Dockerfile 的路径的（当 Dockerfile 名字为不为 Dockerfile/dockerfile 或者不在执行 docker build 命令的目录下，才需要使用 -f 参数指定 Dockerfile 文件）。

**那么什么是上下文环境呢?**

Docker 在运行时分为 Docker引擎（服务端守护进程） 以及客户端工具，我们日常使用各种 docker 命令，其实就是在使用客户端工具与 Docker 引擎 进行交互。那么当我们使用 `docker build` 命令来构建镜像时，这个构建过程其实是在 Docker 引擎中完成的，而不是在本机环境。

**那么如果在 Dockerfile 中使用了一些 COPY 等指令来操作文件，如何让 Docker引擎 获取到这些文件呢？**

这里就有了一个镜像构建上下文的概念，当构建的时候，由用户指定构建镜像的上下文路径，而 `docker build` 会将这个路径下所有的文件都打包上传给 Docker 引擎，引擎内将这些内容展开后，就能获取到所有指定上下文中的文件了。比如说 Dockerfile 中的 COPY `./package.json /project`，其实拷贝的并不是本机目录下的 package.json 文件，而是 Docker 引擎中展开的构建上下文中的文件，所以如果拷贝的文件超出了构建上下文的范围，Docker引擎是找不到那些文件的。

查看构建好的镜像：

```sh
❯ docker images
REPOSITORY                                         TAG       IMAGE ID       CREATED          SIZE
java-docker                                        v1.0.0    f479e93b9881   22 minutes ago   574MB
```
### 启动容器

使用刚刚构建好的镜像来启动容器：
* -d：后台运行。
* -p：将容器的端口映射到宿主机的端口，`-p [host port]:[container port]`。
* --name：容器的名字。
* 最后跟上镜像名。

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

前面的示例中，我们已经可以通过容器的方式部署我们的服务了。但是刚刚的服务只是一个单体服务，接下来我们会分别通过手动部署和 Docker Compose 两种方式部署Java 应用服务 和 MySQL 数据库服务两个有关联的服务。

### 手动部署服务
#### 在容器中运行 MySQL 数据库服务

首先创建两个 volume，用于持久化存储 MySQL 的数据和配置：

```sh
docker volume create mysql_data
docker volume create mysql_config
```
然后创建一个网络，Java 应用程序和数据库的容器将使用该网络相互通信，该网络被称为用户自定义的桥接网络，在自定义的桥接网络中，容器之间可以使用 DNS 名称互相通信（Docker 默认自带的桥接网络不能使用 DNS 名称通信）。

```h
docker network create mysqlnet
```

启动数据库容器：
* -v：挂载 volume。
* --network：指定使用的网络。
* --name：容器名。
* -e：设置环境变量，`MYSQL_ROOT_PASSWORD` 必须设置。
* -p：将容器的端口映射到宿主机的端口，`-p [host port]:[container port]`。
* 最后跟上镜像名。


```sh
docker run -it --rm -d -v mysql_data:/var/lib/mysql \
-v mysql_config:/etc/mysql/conf.d \
--network mysqlnet \
--name mysqlserver \
-e MYSQL_USER=petclinic -e MYSQL_PASSWORD=petclinic \
-e MYSQL_ROOT_PASSWORD=root -e MYSQL_DATABASE=petclinic \
-p 3306:3306 mysql:8.0.23
```
#### 在容器中运行 Java 应用服务
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

使用以下命令来测试 API 接口，Java 应用服务会去查询 MySQL 数据库并返回结果：

```sh
curl  --request GET \
  --url http://localhost:8080/vets \
  --header 'content-type: application/json'
```

你应该能看到以下返回结果：

```sh
{"vetList":[{"id":1,"firstName":"James","lastName":"Carter","specialties":[],"nrOfSpecialties":0,"new":false},{"id":2,"firstName":"Helen","lastName":"Leary","specialties":[{"id":1,"name":"radiology","new":false}],"nrOfSpecialties":1,"new":false},{"id":3,"firstName":"Linda","lastName":"Douglas","specialties":[{"id":3,"name":"dentistry","new":false},{"id":2,"name":"surgery","new":false}],"nrOfSpecialties":2,"new":false},{"id":4,"firstName":"Rafael","lastName":"Ortega","specialties":[{"id":2,"name":"surgery","new":false}],"nrOfSpecialties":1,"new":false},{"id":5,"firstName":"Henry","lastName":"Stevens","specialties":[{"id":1,"name":"radiology","new":false}],"nrOfSpecialties":1,"new":false},{"id":6,"firstName":"Sharon","lastName":"Jenkins","specialties":[],"nrOfSpecialties":0,"new":false}]}%
```

完成测试后，清理现场：

```sh
docker rm -f mysqlserver
docker rm -f springboot-server
docker volume rm mysql_data
docker volume rm mysql_config
docker network rm mysqlnet
```
### 使用 Docker Compose 部署服务
刚刚手动部署的方式我们需要事先创建 volume，network 等资源，我们可以使用 Docker Compose 来部署多个容器服务，将多个服务以及所需的资源定义在一个 docker-compose.yml 文件，只需要一条命令就可以快速部署服务。

#### 安装 Docker Compose

```sh
curl -L https://get.daocloud.io/docker/compose/releases/download/1.24.0/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```
#### 编写 dockcer-compose.yaml 文件
```yaml
version: '3.8'
services:
  petclinic:  #自定义服务名，Java服务
    build:  #上下文环境
      context: .
    ports: #暴露到主机的端口号
      - 8000:8000
      - 8080:8080
    networks: #容器使用的网络
      - mysqlnet
    environment:  #环境变量
      - SERVER_PORT=8080
      - MYSQL_URL=jdbc:mysql://mysqlserver/petclinic
    volumes: #挂载主机当前目录到容器/app目录
      - ./:/app
    #启动容器后执行的命令
    command: ./mvnw spring-boot:run -Dspring-boot.run.profiles=mysql -Dspring-boot.run.jvmArguments="-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:8000"  

  mysqlserver: #自定义服务名，MySQL服务
    image: mysql:8.0.23 #使用的镜像名
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
    volumes: #使用volmue
      - mysql_data:/var/lib/mysql
      - mysql_config:/etc/mysql/conf.d
volumes: #创建volume
  mysql_data:
  mysql_config:

networks: #创建network
  mysqlnet:
```
#### 通过 Docker Compose 启动服务
使用 docker-compose 命令启动服务，Docker 会自动帮我们创建好需要的 volume 和 network 资源并启动容器：
* **-f**：Docker Compose 默认的文件名为 docker-compose.yml, docker-compose.yaml, compose.yml, compose.yaml，如果文件名不是这几个，需要使用 `-f` 参数指定文件名。
* **up**：启动服务。
* **-d**：在后台运行。
* **--build**：启动的时候重新构建镜像。

```sh
docker-compose -f docker-compose.dev.yml up -d --build
```
![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515123643.png)


查看 Docker Compose 运行的容器：

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
### 远程调试

保留前面 Docker Compose 的运行环境，接下来使用 Intellij IDEA 远程调试程序。 Run menu > Edit Configuration，添加 Remote JVM Debug。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515123937.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515124003.png)

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515124231.png)

打开 `src/main/java/org/springframework/samples/petclinic/vet/VetController.java` 文件，在第 54 行添加一个断点。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515124349.png)

点击 Debug，开始调试。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515124435.png)

客户端使用以下命令发起请求：

```sh
curl --request GET --url http://localhost:8080/vets
```

程序会在断点处暂停，你可以检查和观察变量，设置条件断点，查看堆栈跟踪等。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515124528.png)

测试完成后，清理现场，Docker 会删除容器以及 volume，network 等资源：

```sh
docker-compose -f docker-compose.dev.yml down
```
## 单元测试

测试是现代软件开发的重要组成部分。测试对于不同的开发团队来说意味着很多事情。测试包含单元测试、集成测试和端到端测试。在本指南中，我们将看看如何在 Docker 中运行单元测试。以下红色部分是单元测试的代码位置：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515125540.png)

使用前面已经构建的 java-docker:v1.0.1 镜像来运行容器，启动容器时使用 `./mvnw test` 运行单元测试：

```sh
docker run -it --rm --name springboot-test java-docker:v1.0.1 ./mvnw test
```

完成单元测试后，输出结果如下：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515125954.png)

## 多阶段构建

Docker 允许我们在 Dockerfile 中使用多个 FROM 语句，而每个 FROM 语句都可以使用不同基础镜像，每一个 FROM 代表一个构建阶段。下面这个 Dockerfile 中定义了 base，test，development 和 production 4 个构建阶段，我们可以自由选择构建镜像阶段，比如我只想做单元测试，那么我就只选择 test 阶段，如果我想要构建生产环境使用的镜像，那么就选择 production阶段。

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
### 多阶段构建单元测试

我们在构建镜像的时候可以使用 --target，表示只运行 test 这个构建阶段。

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

接下来修改 `src/test/java/org/springframework/samples/petclinic/model/ValidatorTests.java`  文件第 57 行代码如下：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515131856.png)

使用 docker build 命令构建镜像：
```sh
docker build -t java-docker:v1.0.4 --target test .
```

由于前面我们故意修改了代码，会导致单元测试失败，因此在构建镜像的时候就会失败退出：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515132122.png)

### 多阶段构建生产镜像

使用 `--target` 参数指定构建生产环境使用的容器镜像：

```sh
docker build -t java-docker:v1.0.5 --target production  .
```

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515155531.png)

### 多阶段构建 Docker Compose 

更新 docker-compose.dev.yml 文件，指定 Java 服务使用 production 阶段的镜像。

```yaml
version: '3.8'
services:
  petclinic:
    build:
      context: .
      target: production #指定production阶段
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

启动服务：

```sh
 docker-compose -f docker-compose.dev.yml up -d --build
```
![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210516010853.png)

完成测试后，清理现场：

```sh
 docker-compose -f docker-compose.dev.yml down
```

## Github Action CI/CD
GitHub Actions 是 GitHub 的持续集成服务，于2018年10月推出。GitHub Actions 的基本概念如下：
* **workflow（工作流程）**：持续集成一次运行的过程，就是一个 workflow。
* **job（任务）**：一个 workflow 由一个或多个 jobs 构成，含义是一次持续集成的运行，可以完成多个任务。
* **step（步骤）**：每个 job 由多个 step 构成，一步步完成。
* **action（动作）**：每个 step 可以依次执行一个或多个命令（action）。

GitHub Actions 的配置文件叫做 workflow 文件，存放在代码仓库的.github/workflows目录。workflow 文件采用 YAML 格式，文件名可以任意取，但是后缀名统一为.yml，比如 foo.yml。一个库可以有多个 workflow 文件。GitHub 只要发现.github/workflows目录里面有.yml文件，就会自动运行该文件。

GitHub 做了一个官方市场，可以搜索到他人提交的 actions，每个 action 就是一个独立的脚本。如果你需要某个 action，不必自己写复杂的脚本，直接引用他人写好的 action 即可。例如我们可以找到关于 docker 操作的一系列 action：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210516001501.png)


接下来将实现 GitHub Action CI / CD 流程。

### 创建镜像仓库 Access Token

登录 Docker Hub，依次点击 Account Setting > Security > New Access Token，创建一个 Access Token，用于 Github Action 推送镜像。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515132759.png)

创建完成后，复制 Access Token。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515132834.png)

### 创建 Github 仓库

创建一个新的 Github Repository。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515133359.png)

### 设置环境变量

在仓库中点击 Setting > Secret > New repository secret 创建加密的环境变量。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515133442.png)

创建两个变量分别为 Docker Hub 的 Access Token 和 Docker Hub 的用户名。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515234254.png)

### 创建 Github Action Workflow

在仓库中点击 Actions > Set up this workflow 为该仓库，创建一个 Github Action Workflow。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515133754.png)

文件内容如下：

```yaml
# workflow的名称，如果省略该字段，默认为当前workflow的文件名。
name: Java docker CI to Docker Hub
# 定触发workflow的条件，当推送tag时，才会触发workflow
on:
  push:
    tags:
      - "v*.*.*"
# 表示要执行的任务
jobs:
  build:
    # 使用GitHub托管的ubuntu实例来构建镜像
    runs-on: ubuntu-latest
    steps:
     # 检验仓库并且下载到runner（ubuntu实例）
      - name: Check Out Repo
        uses: actions/checkout@v2 #uses表示使用别人定义好的action
     # 登录Docker Hub
      - name: Login to Docker Hub
        uses: docker/login-action@v1
        with:
          # 使用之前在Github中定义的环境变量
          username: ${{ secrets.DOCKER_HUB_USERNAME }}
          password: ${{ secrets.DOCKER_HUB_ACCESS_TOKEN }}
     # 构建缓存，减少构建时间，为它不必重新下载所有镜像
      - name: Cache Docker layers
        uses: actions/cache@v2
        with:
          path: /tmp/.buildx-cache
          key: ${{ runner.os }}-buildx-${{ github.sha }}
          restore-keys: |
            ${{ runner.os }}-buildx-
     # 创建builder实例，BuildKit 是下一代的镜像构建组件，是 docker build 的扩展
      - name: Set up Docker Buildx
        id: buildx
        uses: docker/setup-buildx-action@v1
      # 获取push的tag值作为镜像的tag 
      - name: Get Commit Tag
        run: echo "GIT_TAG=`echo $(git describe --tags --abbrev=0)`" >> $GITHUB_ENV
      # 构建并推送镜像
      - name: Build and push
        id: docker_build
        uses: docker/build-push-action@v2
        with:
          context: ./
          file: ./Dockerfile
          # 使用前面创建的builder实例构建docker镜像
          builder: ${{ steps.buildx.outputs.name }}
          push: true
          tags: ${{ secrets.DOCKER_HUB_USERNAME }}/java-docker:${{ env.GIT_TAG }}
          # 使用缓存
          cache-from: type=local,src=/tmp/.buildx-cache
          cache-to: type=local,dest=/tmp/.buildx-cache
     # 打印镜像摘要
      - name: Image digest
        run: echo ${{ steps.docker_build.outputs.digest }}
```

### 推送代码
初始化本地仓库，并且提交代码到 Github 上。
```sh
#初始化本地仓库
git init 
#建立远程仓库
git remote add origin https://github.com/cr7258/spring-petclinic.git
#添加要push到远程仓库的文件或文件夹  
git add .  
#提交到本地仓库
git commit -m "java docker" 
#拉取远程仓库代码
git pull origin master --allow-unrelated-histories
#将本地仓库push到远程仓库mater分支
git push -u origin master  
```

由于我们在 Github Action 的 workflow 配置文件中设置了只有推送 tag 时才会触发 workflow，因此刚才的推送代码并不会触发 workflow。接下来推送 tag 来触发 workflow：

```sh
git tag -a v1.0.11
git push origin v1.0.11
```

查看推送的 tag：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515233431.png)

Github Action 会自动触发 workflow：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515233338.png)

查看 workflow 详情：

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515233325.png)

完成 workflow 后，在 Docker Hub 上可以看到构建的镜像，镜像 tag 为推送代码时的 tag。

![](https://chengzw258.oss-cn-beijing.aliyuncs.com/Article/20210515233503.png)


## 参考链接
* https://www.cnblogs.com/panpanwelcome/p/12603701.html
* https://docs.docker.com/language/java/
* https://docs.github.com/cn/actions/guides/caching-dependencies-to-speed-up-workflows
* https://www.ruanyifeng.com/blog/2019/09/getting-started-with-github-actions.html
