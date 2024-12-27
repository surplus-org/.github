# 构建appsmith镜像

## 1. 获取编译中使用到的Docker镜像

|组件|作用|版本|说明|
|:-:|:-:|:-:|:-:|
|caddy|运行|caddy:builder-alpine|web服务器|
|redis|运行|redis:8.0-M02-bookworm||
|mongodb|运行|mongodb/mongodb-community-server:5.0.3-ubuntu2004||
|postgresql|运行|postgres:14||
|ngrok|调试|ngrok/ngrok:3.19.0-debian|gateway|
|java|基座|eclipse-temurin:17.0.13_11-jre-ubi9-minimal||
|ubuntu|基座|ubuntu:20.04||
|Maven|编译|maven:3.9.9-eclipse-temurin-17-focal||
|NodeJS|编译|node:20.18.1-alpine3.21,node:20.11.1-bullseye||

## 2. 构建base镜像并配置数据

```bash
cd deploy/docker
docker buildx build -t appsmith.base -f base.dockerfile .
```

## 3. 编译client
   
   进入编译容器
```bash
docker run -it --rm --name build-front --hostname build-front \
  --cpuset-cpus="1-3" --oom-kill-disable --oom-score-adj=-1000 --memory=4G --memory-swap=-1 \
  -v /home/wales/appsmith/code/appsmith:/home/wales/appsmith \
  -v /home/wales/appsmith/code/yarn/:/home/wales/appsmith/app/client/.yarn/cache \
  -v /home/wales/appsmith/code/node_modules/:/home/wales/appsmith/app/client/node_modules/ \
  -w /home/wales/appsmith \
  node:20.11.1-bullseye \
  bash
```

  执行命令
```bash
cd app/client
yarn install --immutable
yarn run check-types
REACT_APP_ENVIRONMENT=PRODUCTION \
REACT_APP_FUSIONCHARTS_LICENSE_KEY= \
SENTRY_AUTH_TOKEN= \
REACT_APP_VERSION_EDITION="Community" \
yarn build
```

## 4. 编译RTS

   进入编译容器
```bash
docker run -it --rm --name build-front --hostname build-front \
  --cpuset-cpus="1-3" --oom-kill-disable --oom-score-adj=-1000 --memory=4G --memory-swap=-1 \
  -v /home/wales/appsmith/code/appsmith:/home/wales/appsmith \
  -v /home/wales/appsmith/code/yarn/:/home/wales/appsmith/app/client/.yarn/cache \
  -v /home/wales/appsmith/code/node_modules/:/home/wales/appsmith/app/client/node_modules/ \
  -w /home/wales/appsmith \
  node:20.11.1-bullseye \
  bash
```

  执行命令
```bash
cd app/client/packages/rts
corepack enable
yarn install --immutable
yarn run test:unit
yarn build
```

  编译出的产出物为：app/client/packages/rts/dist/


## 5. 编译server

   进入编译容器
```bash
docker run -it --rm --name buile-backend --hostname buile-backend \
  --cpuset-cpus="1-3" --oom-kill-disable --oom-score-adj=-1000 --memory=4G --memory-swap=-1 \
  -v /home/wales/appsmith/code/appsmith:/home/wales/appsmith \
  -v /home/wales/appsmith/code/maven:/root/.m2 \
  -w /home/wales/appsmith \
  appsmith.base \
  bash
```

  执行命令
```bash
cd app/server
APPSMITH_DB_URL="mongodb://localhost:27017/appsmith" 
APPSMITH_REDIS_URL="redis://127.0.0.1:6379" 
APPSMITH_MAIL_ENABLED=false 
APPSMITH_ENCRYPTION_PASSWORD=abcd
APPSMITH_ENCRYPTION_SALT=abcd
mvn -B clean compile && ./build.sh -DskipTests
```

## 6. 初始化数据库

1. postgres模式(根据app/server/appsmith-server/src/main/resources/application.properties中的配置，不支持postgres模式）

```bash
docker container run -d --restart=always --name appsmith-pg --hostname appsmith-pg\
  -e POSTGRES_DB=appsmith \
  -e POSTGRES_PASSWORD=appsmith \
  -e POSTGRES_USER=appsmith \
  -v $(pwd)/pg/data:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:14
```

```SQL
SELECT * FROM pg_database WHERE datname='appsmith';
CREATE DATABASE appsmith;
CREATE USER \"$PG_DB_USER\" WITH PASSWORD '$PG_DB_PASSWORD';
CREATE SCHEMA IF NOT EXISTS appsmith;
CREATE EXTENSION IF NOT EXISTS pg_trgm;
GRANT ALL PRIVILEGES ON SCHEMA ${schema} TO ${user};
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA ${schema} TO ${user};
ALTER DEFAULT PRIVILEGES IN SCHEMA ${schema} GRANT ALL PRIVILEGES ON TABLES TO ${user};
GRANT CONNECT ON DATABASE ${db} TO ${user};
```

2. mongodb模式

```bash
docker run -d --restart always --name appsmith-mongo --hostname appsmith-mongo \
  -e MONGODB_INITDB_ROOT_USERNAME=appsmith \
  -e MONGODB_INITDB_ROOT_PASSWORD=appsmith \
  -v $(pwd)/mongodb/mongod.conf:/etc/mongod.conf \
  -v $(pwd)/mongodb/data:/var/lib/mongodb \
  -v $(pwd)/mongodb/log:/var/log/mongodb/ \
  mongodb/mongodb-community-server:5.0.3-ubuntu2004
```


docker run -it --restart always --name appsmith-mongo --hostname appsmith-mongo \
  -e MONGODB_INITDB_ROOT_USERNAME=appsmith \
  -e MONGODB_INITDB_ROOT_PASSWORD=appsmith \
  -v $(pwd)/mongodb/mongod.conf:/etc/mongod.conf \
  -v $(pwd)/mongodb/data:/var/lib/mongodb \
  -v $(pwd)/mongodb/log:/var/log/mongodb/ \
  mongodb/mongodb-community-server:5.0.3-ubuntu2004 /usr/bin/mongod --bind_ip 0.0.0.0 --port 27017 --logappend --tlsMode disabled  --fork

/usr/bin/mongod --bind_ip 0.0.0.0 --port 27017 --logappend --tlsMode disabled  --fork

```
parseFloat(db.adminCommand({getParameter: 1, featureCompatibilityVersion: 1}).featureCompatibilityVersion.version) < 5 &&
      db.adminCommand({setFeatureCompatibilityVersion: "5.0"})
```

3. 缓存

```bash
docker container run -d --restart=always --name appsmith-redis --hostname appsmith-redis \
  -e MASTER=0.0.0.0 \
  -e REDIS_USER=appsmith \
  -e REDIS_PASSWORD=appsmith \
  -v $(pwd)/redis/data:/data \
  -p 6379:6379 \
  redis:6.2
```

## 7. 构建配置数据

1. 进入base镜像，并生成info.json
```bash
export WWW_PATH=/tmp/appsmith/www
export _APPSMITH_CADDY=/opt/caddy/caddy
export TMP=/tmp/appsmith
apt-get update
apt-get install -y git jq
cd scripts
scripts/generate_info_json.sh
```
备份：deploy/docker/fs/opt/appsmith/info.json
使用：backend,editor,rts


2. 生成caddy配置文件
```bash
cd deploy/docker/fs/opt/appsmith
scripts/generate_caddyfile.sh
```
备份：/tmp/appsmith/Caddyfile

3. 生成infra.json
```bash
cd deploy/docker/fs/opt/appsmith
scripts/generate_infra_json.sh
```
备份：/tmp/appsmith/infra.json
使用：backend


## 8. 构建editor镜像与运行

```Dockerfile
FROM caddy:2.8.4-builder
ENV "GO111MODULE=on" \
ENV "GOPROXY=https://goproxy.cn" \
RUN xcaddy build --with github.com/mholt/caddy-ratelimit
```

```bash
docker container run -d --restart=always --name appsmith-caddy --hostname appsmith-caddy \
  -e "GO111MODULE=on" \
  -e "GOPROXY=https://goproxy.cn" \
  -v $(pwd)/caddy/Caddyfile:/opt/appsmith/Caddyfile \
  -v $(pwd)/code/appsmith/app/client/build:/opt/appsmith/editor/ \
  -v $(pwd)/code/appsmith/deploy/docker/fs/opt/appsmith/info.json:/opt/appsmith/info.json \
  -p 8080:80 -p 2019:2019 \
  -w /opt/appsmith \
  caddy:2.4.8-ok  \
  caddy run --config /opt/appsmith/Caddyfile
```


## 9. 构建rts镜像与运行

```Dockerfile
FROM node:20.18.1-alpine3.21

# Add RTS - Application Layer
COPY ./app/client/packages/rts/dist rts/
```

/opt/appsmith/run-with-env.sh node server.js

## 10. 构建backend镜像与运行

```Dockerfile
FROM eclipse-temurin:17.0.13_11-jre-ubi9-minimal

# Add client UI - Application Layer
COPY server.jar /opt/appsmith/server/mongo
```

```bash

docker.env
```



## 其他

### 问题1

因为使用的是性能较低的桌面级进行编译运行动作，所以会遇到时不时被OOM的问题。
```
                    The build failed because the process exited too early.
                    This probably means the system ran out of memory or someone called
                    `kill -9` on the process.
```
解决方式：
```
  --cpuset-cpus="1-3" --oom-kill-disable --oom-score-adj=-1000 --memory=4G --memory-swap=-1 
```

### 问题2

报错信息：
```
#
# Fatal error in , line 0
# Check failed: 0 == munmap(address, size).
#
#
#
#FailureMessage Object: 0x7d32107f8af0
----- Native stack trace -----
```
解决方法：
将app/client/build.sh中
```
craco --max-old-space-size=7168 build --config craco.build.config.js
```
的--max-old-space-size根据实际情况改小。

### 问题3

通过扩大swap区域解决oom问题。
