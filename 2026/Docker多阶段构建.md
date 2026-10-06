# Docker 多阶段构建

## 概述

一个常见问题：为了编译要装整套 JDK/Maven，但运行时根本不需要这些，结果镜像动辄
几百 MB、甚至把源码和构建缓存都打了进去。**多阶段构建（Multi-stage Build）**用一个
Dockerfile 定义多个 `FROM` 阶段，最终镜像只保留运行时需要的东西。

## 传统做法的问题

```dockerfile
FROM maven:3.9-eclipse-temurin-21
COPY . /app
WORKDIR /app
RUN mvn -DskipTests package
EXPOSE 8080
CMD ["java","-jar","target/app.jar"]
```

- 基础镜像 `maven` 本身就很大
- 最终镜像里残留了源码、依赖仓库、Maven 本体
- 镜像体积大、攻击面大、拉取慢

## 多阶段写法

```dockerfile
# ---------- 阶段一：构建 ----------
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline     # 先拉依赖，利于缓存
COPY src ./src
RUN mvn -DskipTests package

# ---------- 阶段二：运行 ----------
FROM eclipse-temurin:21-jre
WORKDIR /app
COPY --from=build /app/target/app.jar app.jar
EXPOSE 8080
CMD ["java","-jar","app.jar"]
```

最终镜像只包含一个 JRE 和一个 jar，几百 MB 缩到一两百 MB。

## 语法要点

- 每个 `FROM` 开启一个新阶段，`AS name` 给阶段命名
- `COPY --from=build ...` 从指定阶段拷贝文件；`--from=0` 可用下标
- 只有**最后一个阶段**进入最终镜像；中间阶段只存在于构建缓存

## 按需只构建某个阶段

```bash
# 只想拿到构建产物，不产最终镜像
docker build --target build -o type=local,dest=./out .
```

`--target` 指定目标阶段，`-o` 直接导出文件而非镜像。

## 常见技巧

1. **先 COPY 依赖清单，再 COPY 源码**：依赖变化少，能命中缓存层。

```dockerfile
COPY pom.xml .
RUN mvn dependency:go-offline
COPY src ./src
```

2. **用 `alpine` / `distroless` 缩小运行时**：`distroless` 无 shell、无包管理器，攻击面更小。

```dockerfile
FROM gcr.io/distroless/java17-debian12
COPY --from=build /app/target/app.jar /app.jar
CMD ["/app.jar"]
```

3. **前端构建同理**：`node:20` 里 `npm run build`，产物 `COPY --from` 到 `nginx:alpine`。

```dockerfile
FROM node:20-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine
COPY --from=build /app/dist /usr/share/nginx/html
```

4. **多架构**：配合 `docker buildx build --platform linux/amd64,linux/arm64` 一次出多平台镜像。

## 注意事项

- `COPY --from` 只能拿该阶段 `WORKDIR` 里的文件，路径要写对
- 别把 `node_modules`、`.git`、构建缓存误 `COPY` 进运行阶段
- 用 `.dockerignore` 排除 `target/`、`node_modules/`、`.git/` 等，加速构建上下文传输
- 运行时尽量用**非 root 用户**，进一步收敛权限

## 小结

多阶段构建本质是「**构建用一套环境，运行用另一套更小的环境**」。
先稳定依赖层、再拷源码，能显著提升缓存命中率；最后用 `alpine`/`distroless`
把体积和攻击面一并压下来。
