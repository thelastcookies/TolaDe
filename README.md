# NODE-API-BASE

A general WEB API using NodeJS.

## 关键字

- Node.js;
- Koa;
- Prisma;

## 项目目录及说明

```
|-- .env                    # 环境变量
|-- |-- types
|-- |-- |-- env.d.ts        # 环境变量类型声明
|-- |-- .env                # 项目环境变量
|-- |-- .env.development    # 开发环境变量
|-- |-- .env.production     # 发布环境变量
|-- .github                 # GitHub 配置
|-- |-- workflows           # GitHub Workflows 配置
|-- |-- |-- release-please.yml # release-please 配置文件
|-- .husky                  # Git hooks 脚本
|-- |-- commit-msg
|-- |-- pre-commit
|-- logs                    # 日志文件目录
|-- prisma                  # Prisma 目录
|-- |-- schema.prisma       # Prisma 数据库模型
|-- |-- seed.ts             # 数据库初始数据
|-- src                     # 源代码目录，包含项目的主要代码文件
|-- |-- controller          # 控制器层，处理请求并返回响应
|-- |-- |-- index.ts        # 各控制器的总输出文件
|-- |-- |-- user.ts         # 示例用户控制器文件
|-- |-- middleware          # 中间件层，用于处理请求的中间件函数
|-- |-- |-- logger.ts       # 日志中间件
|-- |-- router              # 路由层，定义应用程序的路由
|-- |-- |-- index.ts        # 路由的总输出文件
|-- |-- |-- user.ts         # 示例用户路由
|-- |-- service             # 服务层，包含业务逻辑和数据处理
|-- |-- |-- index.ts        # 各服务的总输出文件
|-- |-- |-- user.ts         # 示例用户服务
|-- |-- utils               # 工具库，包含辅助函数和工具类
|-- |-- |-- log.ts          # 日志工具库 log4js 配置
|-- |-- index.ts            # 应用入口文件
|-- |-- prisma.ts           # Prisma 客户端实例
|-- .gitignore              # Git 忽略文件
|-- LICENSE                 # License 文件
|-- package.json            # 项目的配置文件，包含依赖、脚本等
|-- README.md               # 项目说明文档
|-- tsconfig.json           # TypeScript 配置文件
```

## 安装与环境配置

### DotEnv

DotEnv 用于加载环境变量文件（通常是 .env 文件）中的变量到 process.env 中。

这有助于将敏感数据（如数据库连接字符串、API 密钥等）与代码分离，同时使项目的配置更加灵活和安全。

```shell
pnpm install dotenv --save
```

#### 配置 DATABASE_URL

.env

```
DATABASE_URL=mysql://USER:PASSWORD@HOST:PORT/DATABASE
```

之后使用的 prisma 工具会默认读取环境变量中的 `DATABASE_URL` 作为数据库连接字符串。

### Prisma

Prisma 是一个现代化的 TypeScript 和 Node.js ORM（对象关系映射工具），旨在简化与数据库的交互。
它适合开发人员快速、安全地管理数据库结构和数据操作。

#### 安装与初始化

```shell
pnpm install prisma -D
```

```shell
pnpm init
```

初始化后会生成 prisma/schema.prisma 文件，用于定义数据库模型。

如果在初始化 Prisma 时，数据库中并不存在相关的业务表，则继续之后的章节来从零开始创建数据库表。

如果数据库中已经存在相关业务表，则直接跳转到[自省](#introspection自省)章节将数据库表内容同步到 Prisma。

#### 基本配置

Prisma 会针对不同架构的发布平台生成不同的 Query Engine，因此在 `schema.prisma` 文件中，根据实际的目标平台配置
`binaryTargets`。

如下：

```prisma
generator client {
  provider      = "prisma-client-js"
  binaryTargets = ["linux-musl-arm64-openssl-3.0.x"]
}
```

#### 定义数据模型

在 schema.prisma 中描述数据库的表结构和关系。如：

```prisma
model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  name      String?  
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
}
```

#### 模型迁移

应用模型更改到数据库：

```shell
pnpm migrate
```

该指令会在 `prisma/migrations` 目录下生成迁移内容，并将数据模型的迁移记录同步到数据库 `_prisma_migrations` 表中。

首次迁移会重置数据库，可能造成数据的丢失，在实际开发中要谨慎处理。

#### Introspection（自省）

对于已经存在的数据库，可以通过指令来获取已有的数据库表，并将数据模型写入 `schema.prisma` 中。

```shell
pnpm pull
```

在实际生产中，数据库表和字段名的命名风格（如 snake_case）可能与 Prisma 的风格（camelCase）不同。

假设通过自省获得了以下基于 snake_case 表示法的模型：

```prisma
model my_user {
  user_id    Int     @id @default(autoincrement())
  first_name String? 
  last_name  String  @unique
}
```

如果使用这个模型来生成 Prisma Client，那么会获得一个 snake_case 风格的 API，可能并不符合传统 JS/TS 的命名风格。

```ts
const user = await prisma.my_user.create({
  data: {
    first_name: 'Alice',
    last_name: 'Smith',
  },
});
```

想要生成符合预期的 API， 可以通过使用 `@map` 和 `@@map` 来进行转换与关联。

`@map` 用来重命名字段名，`@@map` 用来重命名表名。

```prisma
model MyUser {
  userId    Int     @id @default(autoincrement()) @map("user_id")
  firstName String? @map("first_name")
  lastName  String  @unique @map("last_name")

  @@map("my_user")
}
```

```ts
const user = await prisma.myUser.create({
  data: {
    firstName: 'Alice',
    lastName: 'Smith',
  },
})
```

#### 校准（Baseline）

对于已经存在的数据库，在迁移之前需要先进行校准（Baseline），为已包含数据且无法重置的数据库创建初始化迁移记录。

首先，需要创建 migrations 目录并在其中添加一个空文件夹来容纳初始化迁移的内容，如：

```shell
mkdir -p prisma/migrations/0_init
```

然后使用指令来生成迁移文件。

```shell
pnpm baseline
```

运行指令后，会在 `prisma/migrations/0_init` 目录下生成 `migration.sql`。检查生成的 sql 是否准确无误。

之后，使用指令来将此次迁移标记为已完成。

```shell
pnpm resolve
```

这个指令会将 `0_init` 的迁移记录同步到数据库 `_prisma_migrations` 表中。

#### 生成 Prisma Client

在项目根目录下运行命令生成客户端：

```shell
pnpm generate
```

使用 `generate` 指令会默认添加 `@prisma/client` 工具，也可以通过 `pnpm install @prisma/client` 指令手动安装。

该指令会根据 `schema.prisma` 文件在 `node_modules` 下生成 `@prisma/client`。

#### 使用 Prisma Client

在代码中通过 PrismaClient 进行数据库操作。

#### seed.ts

seed.ts 文件用于向数据库填充初始数据。

##### package.json 配置

```
"prisma": {
  "seed": "dotenv -e .env/.env.development -- tsx prisma/seed.ts"
},
```

### Nodemon

nodemon 是一个开发工具，会自动监视文件的变化，并在检测到变化时重启应用程序。可以使开发过程更加高效，减少手动重启服务器的频率。

```shell
pnpm install nodemon -D
```

### 启动服务

```shell
pnpm dev
```

## 构建思路

使用 Esbuild 将接口工程打包压缩，再使用 Docker 制作镜像并发布。

## 构建步骤：Esbuild

Esbuild 是一个轻量，高性能，简单易上手的 JavaScript 打包工具和编译器。

核心功能包括代码打包、Tree Shaking、Minify、TypeScript 转译等。

打包指令：

```shell
esbuild src/index.ts --bundle --outdir=dist --platform=node
```

## 构建步骤：Docker

Docker 是一种开源的容器化平台，可以协助开发者打包、分发和运行应用程序及其依赖环境。

它通过轻量级的容器技术，使应用程序在不同环境中保持一致性，简化开发、测试和部署流程。

### Login

首先在命令行或者桌面应用中登录 Docker 账户。

### 拉取基本镜像

根据项目的实际情况进行基本镜像的拉取，如：

```shell
# 适用于当前主机架构的镜像
sudo docker pull node:22-alpine

# 指定 arm64 架构的镜像
sudo docker pull --platform=arm64 node:22-alpine

# 指定 amd64 架构的镜像
sudo docker pull --platform=amd64 node:22-alpine
```

### 默认构建（当前主机架构）

```shell
docker build -t thelastcookies/node-api .
```

### 指定架构构建

```shell
docker build --platform linux/amd64 -t thelastcookies/node-api .
```

### 使用 BuildX（多架构构建）

#### 检查是否启用了 BuildX

```shell
docker buildx version
```

#### 创建构建实例

如果没有启用 BuildX 构建实例，运行以下命令：

```shell
docker buildx create --use
```

这会创建并使用一个新的 BuildX 构建上下文。

验证当前的构建实例是否已启用：

```shell
docker buildx ls
```

#### 指定目标架构

使用 BuildX 的 --platform 参数来指定需要支持的架构。如：

```shell
docker buildx build --platform linux/amd64,linux/arm64 -t thelastcookies/node-api .
```

#### 保存

将多架构镜像保存到本地，供调试使用。

```shell
docker buildx build --platform linux/amd64,linux/arm64 -t your_image_name:tag --output type=docker .
```

#### 推送

将镜像推送到注册表。

默认情况下，多架构镜像需要推送到镜像注册表（如 Docker Hub、Harbor 等），因为本地 Docker Daemon 无法直接存储多架构镜像。

添加 --push 参数将镜像推送到远程注册表：

```shell
docker buildx build --platform linux/amd64,linux/arm64 -t your_image_name:tag --push .
```

#### 验证

构建完成后，验证镜像是否支持多架构：

```shell
docker buildx imagetools inspect your_image_name:tag
```

### Troubleshoot

#### 最佳实践

先将基本镜像，如：`node:21-alpine` 使用 `docker pull` 拉取到本地。

这样可以跳过 Docker build 时可能存在的验证问题，如 `failed to resolve source metadata`。

```shell
sudo docker pull node:21-alpine 
```

#### Docker 访问超时问题

如果在拉取镜像时存在超时问题，则需要设置镜像源。

Docker 守护进程配置文件路径：

```
~/.docker/daemon.json
```

在该文件中添加下列配置，并重启 Docker。

```json
{
  "registry-mirrors": [
    "https://hub.geekery.cn/",
    "https://docker.unsee.tech",
    "https://dockerpull.org"
  ]
}
```

#### Prisma `Could not locate the Query Engine`

目前看来是 Prisma 存在的一个 BUG，prisma 在生成后需要将 Query Engine 一起加入打包。

因此需要在 Dockerfile 中将生成的 `libquery_engine-[xxx].so.node` 复制到同样的目录下。

```Dockerfile
COPY 'node_modules/.pnpm/@prisma+client@6.1.0_prisma@6.1.0/node_modules/.prisma/client/<相应的包>' '/app/dist/node_modules/.pnpm/@prisma+client@6.1.0_prisma@6.1.0/node_modules/.prisma/client/'
```

## Docker 部署

### 线下部署

#### 打包

使用 `docker save` 指令将镜像打包为 tar 文件。

```shell
docker save -o node-api.tar thelastcookies/node-api
```

#### 加载

将 tar 文件移动到目标主机后使用指令 `docker load` 加载到 Docker 中。

```shell
sudo docker load -i node-api.tar
```

#### 验证与运行

检验是否加载成功。

```shell
sudo docker images
```

如果镜像文件存在于列表中，则表示加载成功。

之后可以通过 `docker run` 指令运行容器。

```shell
sudo docker run -d \
  --name node-api \
  -p 8192:8192 \
  --restart=unless-stopped \
  thelastcookies/node-api
```

### 线上部署

使用 Docker 指令将本地镜像上传到远程镜像仓库，如 Docker Hub、私有仓库。

```shell
docker push [OPTIONS] <镜像名>:<标签>
```

#### 登录

在推送镜像之前，需要登录到目标镜像仓库：

```shell
docker login -u <用户名>
```

如果使用私有仓库，需要提供仓库地址：

```shell
docker login <仓库地址> -u <用户名>
```

#### 标签（如果在构建时指定了标签，则跳过）

使用 `docker tag` 指令将构建后的镜像标记为仓库格式的名称，类似 `<仓库地址>/<用户名>/<镜像名>:<标签>`的格式。

示例：

```shell
docker tag node-api thelastcookies/node-api:latest
```

#### 推送

使用 `docker push` 指令将镜像推送到远程仓库。

如果推送到 Docker Hub，名称格式为：<用户名>/<镜像名>:<标签>。

```shell
docker push <用户名>/<镜像名>:<标签>
```

如果推送到私有仓库，格式为：<仓库地址>/<镜像名>:<标签>。

```shell
docker push <仓库地址>/<镜像名>:<标签>
```

示例：

```shell
docker push thelastcookies/node-api:latest
```

如果未指定 `:<标签>`，默认会推送 `latest` 标签。

#### 验证

成功推送后，可以在 Docker Hub 或私有仓库中查看该镜像。

#### 拉取

在目标主机上使用指令 `docker pull` 拉取镜像。

```shell
sudo docker pull thelastcookies/node-api:latest
```

拉取成功后即可开始新建容器运行服务。
