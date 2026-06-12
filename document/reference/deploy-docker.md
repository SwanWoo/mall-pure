# Docker 部署手册

> 面向有 Docker 基础的读者，基于 Spring Boot 3.5 + Java 17 + Ubuntu 22.04 环境。重点在项目特有的踩坑和解决方案，通用 Docker 命令简略带过。

## 目录

- [架构总览](#架构总览)
- [快速部署（一键脚本）](#快速部署一键脚本)
- [分步部署](#分步部署)
  - [1. 安装 Docker 及基础工具](#1-安装-docker-及基础工具)
  - [2. 配置镜像加速](#2-配置镜像加速)
  - [3. 启动中间件](#3-启动中间件)
  - [4. 初始化数据库及中间件配置](#4-初始化数据库及中间件配置)
  - [5. 构建项目 JAR 包](#5-构建项目-jar-包)
  - [6. 构建 Docker 镜像](#6-构建-docker-镜像)
  - [7. 启动应用服务](#7-启动应用服务)
  - [8. 验证服务](#8-验证服务)
- [踩坑记录（13 条）](#踩坑记录13-条)
- [运维常用命令](#运维常用命令)

---

## 架构总览

```
                    ┌─────────────┐
                    │   Nginx     │ :80
                    └──────┬──────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
  ┌─────┴──────┐   ┌──────┴──────┐   ┌──────┴──────┐
  │ mall-admin │   │ mall-search │   │ mall-portal │
  │   :8080    │   │   :8081     │   │   :8085     │
  └─────┬──────┘   └──────┬──────┘   └──────┬──────┘
        │                 │                  │
   ┌────┴─────────────────┴──────────────────┴────┐
   │              mall-network (Docker)            │
   └──┬──────┬──────┬──────┬──────┬──────┬───────┘
      │      │      │      │      │      │
   MySQL  Redis  RabbitMQ  ES   MongoDB MinIO
   :3306  :6379  :5672   :9200  :27017  :9090
```

**服务端口汇总**：

| 服务 | 端口 | 说明 |
|------|------|------|
| mall-admin | 8080 | 后台管理 API |
| mall-search | 8081 | 搜索服务 API |
| mall-portal | 8085 | 前台门户 API |
| MySQL | 3306 | 数据库 |
| Redis | 6379 | 缓存 |
| RabbitMQ | 5672 / 15672 | 消息队列 / 管理界面 |
| Elasticsearch | 9200 / 9300 | 搜索引擎 |
| Logstash | 4560-4563 | 日志收集 |
| Kibana | 5601 | 日志可视化 |
| MongoDB | 27017 | 文档数据库 |
| MinIO | 9090 / 9001 | 对象存储 API / 控制台 |
| Nginx | 80 | 反向代理 |

---

## 快速部署（一键脚本）

适合全新环境，将以下内容保存为 `deploy.sh` 并执行：

```bash
#!/bin/bash
set -e

echo "===== 1. 安装 Docker ====="
sudo apt-get update -qq
sudo apt-get install -y -qq ca-certificates curl gnupg lsb-release openjdk-17-jdk maven
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update -qq
sudo apt-get install -y -qq docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl start docker && sudo systemctl enable docker

echo "===== 2. 配置镜像加速 ====="
sudo tee /etc/docker/daemon.json <<EOF
{
  "registry-mirrors": [
    "https://docker.1ms.run",
    "https://docker.xuanyuan.me",
    "https://docker.m.daocloud.io"
  ]
}
EOF
sudo systemctl daemon-reload && sudo systemctl restart docker

echo "===== 3. 创建目录和配置文件 ====="
sudo mkdir -p /mydata/mysql/{data,log,conf/conf.d,conf/mysql.conf.d}
sudo mkdir -p /mydata/nginx/{html,logs,conf}
sudo docker run --rm nginx:1.22 tar cf - /etc/nginx | sudo tar xf - -C /mydata/nginx/conf --strip-components=1
if [ -d /mydata/nginx/conf/nginx ]; then
  sudo cp -r /mydata/nginx/conf/nginx/* /mydata/nginx/conf/
  sudo rm -rf /mydata/nginx/conf/nginx
fi
sudo mkdir -p /mydata/logstash
sudo cp document/elk/logstash.conf /mydata/logstash/logstash.conf
sudo sed -i 's/localhost:9200/es:9200/' /mydata/logstash/logstash.conf
sudo mkdir -p /mydata/elasticsearch/{plugins,data}
sudo chmod 777 /mydata/elasticsearch/data
sudo mkdir -p /mydata/redis/data /mydata/rabbitmq/data /mydata/mongo/db /mydata/minio/data

echo "===== 4. 启动中间件 ====="
sudo docker network create mall-network 2>/dev/null || true
cd document/docker
sudo docker compose -f docker-compose-env.yml up -d
sleep 30

echo "===== 5. 初始化数据库及中间件 ====="
sudo docker exec -i mysql mysql -uroot -proot -e "CREATE DATABASE IF NOT EXISTS mall DEFAULT CHARACTER SET utf8mb4;"
sudo docker exec -i mysql mysql -uroot -proot mall < document/sql/mall.sql
sudo docker exec -i mysql mysql -uroot -proot -e "
CREATE USER IF NOT EXISTS 'reader'@'%' IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON mall.* TO 'reader'@'%';
FLUSH PRIVILEGES;
"
sudo docker exec rabbitmq rabbitmqctl add_user mall mall 2>/dev/null || true
sudo docker exec rabbitmq rabbitmqctl add_vhost /mall 2>/dev/null || true
sudo docker exec rabbitmq rabbitmqctl set_permissions -p /mall mall ".*" ".*" ".*"
sudo docker exec rabbitmq rabbitmqctl set_user_tags mall administrator

echo "===== 6. 接入共享网络 ====="
sudo docker network connect --alias db mall-network mysql 2>/dev/null || true
sudo docker network connect --alias es mall-network elasticsearch 2>/dev/null || true
sudo docker network connect --alias rabbit mall-network rabbitmq 2>/dev/null || true
for c in redis mongo logstash kibana minio nginx; do
  sudo docker network connect mall-network $c 2>/dev/null || true
done

echo "===== 7. 构建项目 ====="
cd ../..
sudo JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64 mvn clean package -DskipTests

echo "===== 8. 构建 Docker 镜像 ====="
cd mall-admin && sudo docker build -t mall/mall-admin:1.0-SNAPSHOT .
cd ../mall-search && sudo docker build -t mall/mall-search:1.0-SNAPSHOT .
cd ../mall-portal && sudo docker build -t mall/mall-portal:1.0-SNAPSHOT .

echo "===== 9. 启动应用 ====="
cd ../document/docker
sudo docker compose -f docker-compose-app-v2.yml up -d

echo "===== 部署完成 ====="
echo "mall-admin:   http://localhost:8080"
echo "mall-search:  http://localhost:8081"
echo "mall-portal:  http://localhost:8085"
sudo docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"
```

---

## 分步部署

### 1. 安装 Docker 及基础工具

```bash
# 添加 Docker 官方仓库（Ubuntu）
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg lsb-release
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 安装 Docker + JDK 17 + Maven
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin openjdk-17-jdk maven
sudo systemctl start docker && sudo systemctl enable docker
```

> ⚠️ 项目要求 Java 17，Ubuntu 默认可能装 Java 11，确认 `java -version` 输出为 17。

### 2. 配置镜像加速

国内拉 Docker Hub 镜像经常超时，必须配加速源：

```bash
sudo tee /etc/docker/daemon.json <<EOF
{
  "registry-mirrors": [
    "https://docker.1ms.run",
    "https://docker.xuanyuan.me",
    "https://docker.m.daocloud.io"
  ]
}
EOF
sudo systemctl daemon-reload && sudo systemctl restart docker
```

### 3. 启动中间件

#### 3.1 创建宿主机目录和配置文件

**这一步必须在 `docker compose up` 之前完成**，否则容器会因挂载问题启动失败（详见[坑 4/5/6](#坑-4宿主机挂载目录文件缺失导致容器启动失败)）：

```bash
# MySQL：需要 conf.d 和 mysql.conf.d 子目录
sudo mkdir -p /mydata/mysql/{data,log,conf/conf.d,conf/mysql.conf.d}

# Nginx：从镜像提取完整默认配置（挂载会覆盖容器内 /etc/nginx）
sudo mkdir -p /mydata/nginx/{html,logs,conf}
sudo docker run --rm nginx:1.22 tar cf - /etc/nginx \
  | sudo tar xf - -C /mydata/nginx/conf --strip-components=1
# 解压可能产生嵌套目录，修正一下
[ -d /mydata/nginx/conf/nginx ] && sudo cp -r /mydata/nginx/conf/nginx/* /mydata/nginx/conf/ && sudo rm -rf /mydata/nginx/conf/nginx
# 用项目配置覆盖
sudo cp document/docker/nginx.conf /mydata/nginx/conf/nginx.conf

# Logstash：配置文件必须提前存在（否则 Docker 会创建同名目录）
sudo mkdir -p /mydata/logstash
sudo cp document/elk/logstash.conf /mydata/logstash/logstash.conf
sudo sed -i 's/localhost:9200/es:9200/' /mydata/logstash/logstash.conf

# Elasticsearch：数据目录需要写权限
sudo mkdir -p /mydata/elasticsearch/{plugins,data}
sudo chmod 777 /mydata/elasticsearch/data

# 其他中间件
sudo mkdir -p /mydata/redis/data /mydata/rabbitmq/data /mydata/mongo/db /mydata/minio/data
```

#### 3.2 创建共享网络

```bash
sudo docker network create mall-network
```

#### 3.3 启动中间件

```bash
cd document/docker
sudo docker compose -f docker-compose-env.yml up -d
```

#### 3.4 接入共享网络并添加别名

prod 配置中使用了简写主机名，需要给容器加网络别名：

```bash
sudo docker network connect --alias db mall-network mysql
sudo docker network connect --alias es mall-network elasticsearch
sudo docker network connect --alias rabbit mall-network rabbitmq
# 以下名称一致，无需别名
for c in redis mongo logstash kibana minio nginx; do
  sudo docker network connect mall-network $c
done
```

别名映射：`db` → mysql、`es` → elasticsearch、`rabbit` → rabbitmq

### 4. 初始化数据库及中间件配置

#### 4.1 MySQL

```bash
# 创建数据库并导入数据
sudo docker exec -i mysql mysql -uroot -proot \
  -e "CREATE DATABASE IF NOT EXISTS mall DEFAULT CHARACTER SET utf8mb4;"
sudo docker exec -i mysql mysql -uroot -proot mall < document/sql/mall.sql

# 创建 prod 配置使用的 reader 用户
sudo docker exec -i mysql mysql -uroot -proot -e "
CREATE USER 'reader'@'%' IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON mall.* TO 'reader'@'%';
FLUSH PRIVILEGES;
"
```

#### 4.2 RabbitMQ

```bash
sudo docker exec rabbitmq rabbitmqctl add_user mall mall
sudo docker exec rabbitmq rabbitmqctl add_vhost /mall
sudo docker exec rabbitmq rabbitmqctl set_permissions -p /mall mall ".*" ".*" ".*"
sudo docker exec rabbitmq rabbitmqctl set_user_tags mall administrator
```

> 注意：先 `add_vhost` 再 `set_permissions`，顺序不能反。

### 5. 构建项目 JAR 包

```bash
cd /home/ubuntu/mall
sudo JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64 mvn clean package -DskipTests
```

> Maven Docker 插件会尝试连接 `192.168.3.101:2375`（示例 IP）导致超时，但 JAR 包本身已经构建成功，忽略该错误即可。

### 6. 构建 Docker 镜像

#### 6.1 创建 Dockerfile

在各模块目录下创建 Dockerfile：

**mall-admin/Dockerfile**：
```dockerfile
FROM eclipse-temurin:17-jdk
ADD target/mall-admin-1.0-SNAPSHOT.jar /mall-admin-1.0-SNAPSHOT.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "-Dspring.profiles.active=prod", "/mall-admin-1.0-SNAPSHOT.jar"]
```

**mall-search/Dockerfile**：
```dockerfile
FROM eclipse-temurin:17-jdk
ADD target/mall-search-1.0-SNAPSHOT.jar /mall-search-1.0-SNAPSHOT.jar
EXPOSE 8081
ENTRYPOINT ["java", "-jar", "-Dspring.profiles.active=prod", "/mall-search-1.0-SNAPSHOT.jar"]
```

**mall-portal/Dockerfile**：
```dockerfile
FROM eclipse-temurin:17-jdk
ADD target/mall-portal-1.0-SNAPSHOT.jar /mall-portal-1.0-SNAPSHOT.jar
EXPOSE 8085
ENTRYPOINT ["java", "-jar", "-Dspring.profiles.active=prod", "/mall-portal-1.0-SNAPSHOT.jar"]
```

> ⚠️ 使用 `eclipse-temurin:17-jdk` 替代已停维的 `openjdk:17`，详见[坑 12](#坑-12openjdk17-镜像不可用)。

#### 6.2 构建镜像

```bash
cd mall-admin && sudo docker build -t mall/mall-admin:1.0-SNAPSHOT .
cd ../mall-search && sudo docker build -t mall/mall-search:1.0-SNAPSHOT .
cd ../mall-portal && sudo docker build -t mall/mall-portal:1.0-SNAPSHOT .
```

#### 6.3 关于 ES IK 中文分词插件

> ⚠️ IK 插件 v7.17.x 已从 GitHub Releases 和 Maven Central 全部下架，详见[坑 13](#坑-13elasticsearch-ik-中文分词插件无法安装)。

**临时方案**：将 `mall-search/src/main/java/com/macro/mall/search/domain/EsProduct.java` 中的 `ik_max_word` 改为 `standard`，然后重新打包构建镜像。

```java
// 改前：@Field(analyzer = "ik_max_word", type = FieldType.Text)
// 改后：
@Field(analyzer = "standard", type = FieldType.Text)
```

**推荐方案**：提前缓存 IK 插件 zip 或升级 ES 到 8.x 版本。

### 7. 启动应用服务

项目自带的 `docker-compose-app.yml` 使用已废弃的 `external_links`，推荐使用以下改进版 `document/docker/docker-compose-app-v2.yml`：

```yaml
services:
  mall-admin:
    image: mall/mall-admin:1.0-SNAPSHOT
    container_name: mall-admin
    ports:
      - 8080:8080
    volumes:
      - /mydata/app/mall-admin/logs:/var/logs
      - /etc/localtime:/etc/localtime
    environment:
      - 'TZ=Asia/Shanghai'
      - 'spring.profiles.active=prod'
    networks:
      - mall-network

  mall-search:
    image: mall/mall-search:1.0-SNAPSHOT
    container_name: mall-search
    ports:
      - 8081:8081
    volumes:
      - /mydata/app/mall-search/logs:/var/logs
      - /etc/localtime:/etc/localtime
    environment:
      - 'TZ=Asia/Shanghai'
      - 'spring.profiles.active=prod'
    networks:
      - mall-network

  mall-portal:
    image: mall/mall-portal:1.0-SNAPSHOT
    container_name: mall-portal
    ports:
      - 8085:8085
    volumes:
      - /mydata/app/mall-portal/logs:/var/logs
      - /etc/localtime:/etc/localtime
    environment:
      - 'TZ=Asia/Shanghai'
      - 'spring.profiles.active=prod'
    networks:
      - mall-network

networks:
  mall-network:
    external: true
```

启动：

```bash
cd document/docker
sudo docker compose -f docker-compose-app-v2.yml up -d
```

### 8. 验证服务

```bash
# 查看容器状态（应全部为 Up）
sudo docker ps

# 测试接口
curl -s -o /dev/null -w "%{http_code}" http://localhost:8080   # 期望 200
curl -s -o /dev/null -w "%{http_code}" http://localhost:8081   # 期望 404（根路径无映射，正常）
curl -s -o /dev/null -w "%{http_code}" http://localhost:8085   # 期望 200

# Swagger 文档
# http://localhost:8080/swagger-ui/index.html
# http://localhost:8085/swagger-ui/index.html
```

---

## 踩坑记录（13 条）

### 坑 1：Docker 用户组权限未即时生效

**现象**：`usermod -aG docker $USER` 后仍需 sudo。

**原因**：加入用户组需重新登录才生效。

**解决**：当前终端用 `sudo`，或 `newgrp docker` 临时生效。

---

### 坑 2：Java 版本不匹配

**现象**：Maven 打包失败。

**原因**：Ubuntu 默认 OpenJDK 11，项目要求 Java 17。

**解决**：`sudo apt install openjdk-17-jdk`，`sudo update-alternatives --set java /usr/lib/jvm/java-17-openjdk-amd64/bin/java`。

---

### 坑 3：Docker Hub 镜像拉取超时 / 限流

**现象**：`docker compose up` 报 `i/o timeout` 或 `429 Too Many Requests`。

**解决**：配置 `/etc/docker/daemon.json` 中的 `registry-mirrors`，多配几个备选源。

---

### 坑 4：宿主机挂载目录/文件缺失导致容器启动失败

**现象**：MySQL 报 `Can't read dir of '/etc/mysql/conf.d/'`，Logstash 报 `not a directory`。

**原因**：Docker Bind Mount 时，宿主机路径不存在会**自动创建目录**（而非文件）。比如需要挂载 `logstash.conf` 文件，但宿主上没有，Docker 就创建了同名目录。

**解决**：`docker compose up` 之前手动创建所有目录和配置文件。如果已被误创建为目录，先 `rm -rf` 再 `cp` 文件过去。

---

### 坑 5：Nginx 挂载覆盖导致缺少 mime.types

**现象**：Nginx 报 `open() "/etc/nginx/mime.types" failed (2: No such file or directory)`。

**原因**：挂载 `/mydata/nginx/conf` 到 `/etc/nginx` 会完全覆盖容器内默认配置。只放 `nginx.conf` 不够，`mime.types` 等文件丢失。

**解决**：从镜像提取完整配置后用项目配置覆盖：
```bash
sudo docker run --rm nginx:1.22 tar cf - /etc/nginx \
  | sudo tar xf - -C /mydata/nginx/conf --strip-components=1
sudo cp document/docker/nginx.conf /mydata/nginx/conf/nginx.conf
```

---

### 坑 6：Logstash 配置文件被 Docker 创建为目录

**现象**：Logstash 报 `not a directory: Are you trying to mount a directory onto a file`。

**原因**：同坑 4，`/mydata/logstash/logstash.conf` 不存在时 Docker 自动创建为目录。

**解决**：
```bash
sudo rm -rf /mydata/logstash/logstash.conf
sudo cp document/elk/logstash.conf /mydata/logstash/logstash.conf
sudo sed -i 's/localhost:9200/es:9200/' /mydata/logstash/logstash.conf
```

---

### 坑 7：Elasticsearch 数据目录权限不足

**现象**：ES 启动后无法写入数据。

**原因**：ES 容器内以非 root 用户运行，宿主机目录权限不匹配。

**解决**：`sudo chmod 777 /mydata/elasticsearch/data`

---

### 坑 8：容器间 DNS 解析失败

**现象**：应用报 `UnknownHostException: db` 或 `es`。

**原因**：prod 配置用 `db`、`es`、`rabbit` 等别名，但容器名是 `mysql`、`elasticsearch`、`rabbitmq`。原 `docker-compose-app.yml` 的 `external_links` 已废弃。

**解决**：创建 `mall-network` 自定义网络，用 `--alias` 添加别名：
```bash
sudo docker network connect --alias db mall-network mysql
sudo docker network connect --alias es mall-network elasticsearch
sudo docker network connect --alias rabbit mall-network rabbitmq
```

---

### 坑 9：MySQL reader 用户不存在

**现象**：`Access denied for user 'reader'@'xxx' (using password: YES)`

**原因**：`application-prod.yml` 使用 `reader/123456`，但 MySQL 容器只创建了 root。

**解决**：
```sql
CREATE USER 'reader'@'%' IDENTIFIED BY '123456';
GRANT ALL PRIVILEGES ON mall.* TO 'reader'@'%';
FLUSH PRIVILEGES;
```

---

### 坑 10：RabbitMQ 用户和 vhost 未配置

**现象**：mall-portal 报 `ACCESS_REFUSED - Login was refused`

**原因**：prod 配置用 `mall/mall` 连接 `/mall` vhost，RabbitMQ 默认只有 `guest`（仅 localhost）。

**解决**：
```bash
sudo docker exec rabbitmq rabbitmqctl add_user mall mall
sudo docker exec rabbitmq rabbitmqctl add_vhost /mall
sudo docker exec rabbitmq rabbitmqctl set_permissions -p /mall mall ".*" ".*" ".*"
sudo docker exec rabbitmq rabbitmqctl set_user_tags mall administrator
```

---

### 坑 11：Maven Docker 插件连接超时

**现象**：`mvn package` 报 `Connect to 192.168.3.101:2375 failed: Connection timed out`

**原因**：pom.xml 配置了 `docker-maven-plugin`，在 package 阶段尝试连接示例 IP 构建 Docker 镜像。

**解决**：JAR 已成功构建，Docker 镜像改用 `docker build` 手动构建。或注释 pom.xml 中 `docker-maven-plugin` 的 `execution` 配置。

---

### 坑 12：openjdk:17 镜像不可用

**现象**：`docker build` 拉取 `openjdk:17` 报 `not found` 或 `429`。

**原因**：OpenJDK 官方 Docker 镜像已停止维护。

**解决**：Dockerfile 改用 `eclipse-temurin:17-jdk`（Adoptium 维护，完全兼容 OpenJDK）。

---

### 坑 13：Elasticsearch IK 中文分词插件无法安装

**现象**：mall-search 报 `analyzer [ik_max_word] has not been configured in mappings`。

**原因**：IK 分词插件 v7.17.x 已被作者从 GitHub Releases、Maven Central 全部下架，源码仓库也不再保留 7.x 分支。所有版本（7.17.0 ~ 7.17.7）均 404。

**临时解决**：将 `EsProduct.java` 中 `ik_max_word` 改为 `standard`，重新打包构建镜像。中文分词效果会降低。

**推荐方案**：
1. 提前缓存 IK 插件 zip 文件，通过 `docker cp` 或挂载安装
2. 升级 ES 到 8.x 并使用对应版本的 IK 插件
3. 自行从 IK 源码历史编译

---

## 运维常用命令

```bash
# 查看所有容器状态
sudo docker ps -a

# 查看容器日志
sudo docker logs -f <容器名>                  # 实时跟踪
sudo docker logs --tail 100 <容器名>           # 最后 100 行

# 重启单个服务
sudo docker compose -f docker-compose-env.yml restart mysql

# 重启应用服务
sudo docker compose -f docker-compose-app-v2.yml restart

# 停止所有服务
sudo docker compose -f docker-compose-env.yml down
sudo docker compose -f docker-compose-app-v2.yml down

# 进入容器内部
sudo docker exec -it <容器名> /bin/bash

# 查看容器资源占用
sudo docker stats
```
