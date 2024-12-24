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
  --oom-kill-disable --memory=4G --memory-swap=-1 \
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

cwd=app/client/packages/rts
yarn cache=app/client/.yarn/cache, app/client/node_modules/.cache/webpack/
corepack enable
yarn install --immutable
yarn run test:unit
yarn build

app/client/packages/rts/dist/


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

docker run --name appsmith-pg -p 5432:5432 -d -e POSTGRES_PASSWORD=password postgres:alpine postgres -N 1500
