# 构建appsmith镜像

## 1. 获取编译中使用到的Docker镜像

|组件|作用|版本|说明|
|:-:|:-:|:-:|:-:|
|caddy|运行|caddy:builder-alpine|web服务器|
|redis|运行|redis:8.0-M02-bookworm||
|mongodb|运行|mongodb/mongodb-community-server:5.0.3-ubuntu2004||
|postgresql|运行|postgres:14||
|ngrok|调试|ngrok/ngrok:3.19.0-debian|gateway|
|ubuntu|基座|ubuntu:20.04||
|Maven|编译|maven:3.9.9-eclipse-temurin-17-focal||
|NodeJS|编译|node:20.18.1-alpine3.21,node:20.11.1-bullseye||


## 2. 编译client
   
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

## 3. 编译RTS

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


## 4. 编译server

   进入编译容器
```bash
docker run -it --rm --name buile-backend --hostname build-backend \
  -v /home/wales/appsmith/code/appsmith:/home/wales/appsmith \
  -v /home/wales/appsmith/code/maven:/root/.m2 \
  -w /home/wales/appsmith \
  maven:3.9.9-eclipse-temurin-17-focal \
  bash
```

  执行命令
```bash
cd app/server
mvn -B clean compile && ./build.sh -DskipTests
```
## 5. 构建base镜像并配置数据

1. 构建base镜像
```bash
cd deploy/docker
docker buildx build -t appsmith.base -f base.dockerfile .
```

2. 进入base镜像，并生成info.json
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


3. 生成caddy配置文件
```bash
cd deploy/docker/fs/opt/appsmith
scripts/generate_caddyfile.sh
```
备份：/tmp/appsmith/Caddyfile

4. 生成infra.json
```bash
cd deploy/docker/fs/opt/appsmith
scripts/generate_infra_json.sh
```
备份：/tmp/appsmith/infra.json
使用：backend


## 6. 构建caddy镜像与运行

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

## 6. 构建caddy镜像与运行

```Dockerfile
FROM caddy:builder-alpine AS caddybuilder

# The env variables are needed for Appsmith server to correctly handle non-roman scripts like Arabic.
ENV LANG=C.UTF-8
ENV LC_ALL=C.UTF-8

RUN xcaddy build --with github.com/mholt/caddy-ratelimit

WORKDIR /opt/appsmith

# Add client UI - Application Layer
COPY ./app/client/build editor/
```

```bash
docker container run -d --name appsmith-caddy --hostname appsmith-caddy \
  -e "GO111MODULE=on" \
  -e "GOPROXY=https://goproxy.cn" \
  caddy:builder-alpine \
  "$_APPSMITH_CADDY" start --config "$TMP/Caddyfile"
```

## 7. 构建rts镜像与运行

```Dockerfile
FROM node:20.18.1-alpine3.21

# Add RTS - Application Layer
COPY ./app/client/packages/rts/dist rts/
```

## 8. 构建appsmith镜像与运行
```bash
docker container run -d --name appsmith-pg \
  -e POSTGRES_PASSWORD=password \
  -p 5432:5432 \
  postgres:14 postgres -N 1500
```

```bash
docker container run -d --name appsmith-redis \
  -p 6379:6379 \
  redis:8.0-M02-bookworm redis-server --save "" --appendonly no
```

```bash
docker container run -d --name appsmith-mongodb \
  -p 27017:27017 \
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
