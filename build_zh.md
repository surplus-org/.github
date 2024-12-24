# 构建appsmith镜像

## 1. 获取编译中使用到的Docker镜像

|组件|作用|版本|说明|
|:-:|:-:|:-:|:-:|
|caddy|运行|caddy:builder-alpine|web服务器|
|redis|运行|redis:8.0-M02-bookworm||
|mongodb|运行|mongodb/mongodb-community-server:5.0.3-ubuntu2004||
|postgresql|运行|postgres:14||
|ngrok|调试|node:20.18.1-alpine3.21|gateway|
|ubuntu|基座|ubuntu:20.04||
|JDK|编译|openjdk:17.0.2-oraclelinux8||
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

cwd=app/server
Cache maven dependencies=~/.m2
mvn -B clean compile && ./build.sh -DskipTests

## 5. 构建镜像

"deploy/docker/base.dockerfile"
platforms: linux/arm64,linux/amd64

Generate info.json
run: |
    if [[ -f scripts/generate_info_json.sh ]]; then
    scripts/generate_info_json.sh
    fi

Place server artifacts-es
        env:
          EDITION: ${{ vars.EDITION }}
        run: |
          scripts/prepare_server_artifacts.sh

  base_tag=release
else
  base_tag=nightly
base_tag=${{ steps.set_base_tag.outputs.base_tag }}
if [[ base_tag != 'nightly' ]]; then
args+=(--build-arg "APPSMITH_CLOUD_SERVICES_BASE_URL=https://release-cs.appsmith.com")
fi
args+=(--build-arg "BASE=${{ vars.DOCKER_HUB_ORGANIZATION }}/base-${{ vars.EDITION }}:$base_tag")
docker build -t cicontainer "${args[@]}" .

## 6. 运行

```bash
docker run --name appsmith-pg -p 5432:5432 -d -e POSTGRES_PASSWORD=password postgres:alpine postgres -N 1500
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
