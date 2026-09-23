# sub2api-apple

用 **Apple `container`** CLI（非 Docker）在 Apple Silicon macOS 上部署 [Sub2API](https://github.com/wei-shaw/sub2api) 的一套运维脚本。

`apple-container.sh` 是一个约 1000 行的单文件管理器，负责编排三个容器：

| 容器 | 镜像 | 说明 |
|---|---|---|
| `sub2api-apple` | `ghcr.io/wei-shaw/sub2api` | 应用本体，对宿主机暴露端口 |
| `sub2api-apple-postgres` | `postgres:18-alpine` | 仅网络内可达 |
| `sub2api-apple-redis` | `redis:8-alpine` | 仅网络内可达 |

三者跑在专用网络 `sub2api-apple` 上，数据落在 `sub2api-apple-data` / `-postgres-data` / `-redis-data` 三个卷里。

## 前置条件

- Apple Silicon Mac
- Apple `container` **1.1.0 及以上**（脚本会校验版本）
- `openssl`、`plutil`（系统自带）

## 快速开始

```bash
./apple-container.sh init   # 由 .env.example 生成 .env，并随机生成密钥
./apple-container.sh up     # 拉起整套服务
./apple-container.sh status # 查看健康状态
```

`init` 会用 `openssl rand -hex 32` 自动生成 `POSTGRES_PASSWORD`、`JWT_SECRET`、`TOTP_ENCRYPTION_KEY`，并把 `.env` 权限设为 `600`。`ADMIN_EMAIL` / `ADMIN_PASSWORD` / `BIND_HOST` / `SERVER_PORT` 需要自己按需改。

## 命令

| 命令 | 作用 |
|---|---|
| `init` | 生成 `.env` 及所需密钥 |
| `up [--recreate]` | 创建并启动整套服务 |
| `down` | 停止服务，保留数据 |
| `restart` | 按依赖顺序重启 |
| `status` | 查看容器与健康状态 |
| `logs <app\|postgres\|redis> [-f]` | 查看日志 |
| `pull` | 拉取 linux/arm64 镜像 |
| `destroy [--volumes] [--yes]` | 删除容器与网络，`--volumes` 连数据卷一起删 |

环境变量 `SUB2API_ENV_FILE` 可指定 env 文件路径，默认取脚本同级目录的 `.env`。

脚本内置了若干防护：启动前加文件锁避免并发操作、校验 `.env` 不可被 group/other 读取、对既有资源做 `org.sub2api.stack` 标签归属校验，避免误删非本栈的容器。

## 使用自定义镜像（前端改动）

前端是 Vue3 + Vite，用 `go:embed` 编译进 Go 二进制，**改 UI 必须重新出镜像**，
无法只替换静态文件。UI 源码维护在 fork 里：
[dptdptdpt1/sub2api](https://github.com/dptdptdpt1/sub2api) 分支 `ui-custom`。

本仓库只需改 `.env` 里的一行镜像地址，脚本本身不用动。

### 1. 构建（在开发机）

```bash
cd <fork 路径>
docker buildx build --platform linux/arm64 \
  --build-arg VERSION=0.2.7-ui1 \
  --build-arg COMMIT=$(git rev-parse --short HEAD) \
  -t local/sub2api:0.2.7-ui1 --load .
```

Go 与 pnpm 都在构建容器内，开发机不用安装。根目录 Dockerfile 已带
`-tags embed`；直接跑 `make build-backend` **不带**这个 tag，编出来的二进制
不含前端，页面会全部 404。

### 2. 传输到部署机

因为 GitHub token 通常没有 `write:packages` 权限，推不了 ghcr.io，改用离线传输：

```bash
docker save local/sub2api:0.2.7-ui1 -o /tmp/sub2api-ui1.tar
scp /tmp/sub2api-ui1.tar seven.local:/tmp/
ssh seven.local 'container image load -i /tmp/sub2api-ui1.tar && rm -f /tmp/sub2api-ui1.tar'
```

buildx 默认附带的 attestation manifest（使镜像成为 manifest list）Apple
`container image load` 可正常识别，无需 `--provenance=false`。

### 3. 切换并部署

```bash
cd ~/sub2api-apple
cp -p .env ".env.before-ui-image-$(date +%Y%m%d-%H%M%S)"
sed -i '' 's|^APPLE_CONTAINER_SUB2API_IMAGE=.*|APPLE_CONTAINER_SUB2API_IMAGE=local/sub2api:0.2.7-ui1|' .env
./apple-container.sh up
```

**用 `up`，不要加 `--recreate`。** `cmd_up` 本身就无条件删除并重建应用容器；
加 `--recreate` 会连 postgres 和 redis 一起拆掉重建，徒增风险。

切换镜像后 wrapper 会比对 `/app/storage/runtime/base-image-id` 与新镜像 digest，
自动把新二进制复制进卷，无需手工清理。

### 数据安全

`up` 全程不会删除任何卷：`ensure_volume` 遇到已存在的卷会直接返回，
`delete_container_if_present` 只处理容器。唯一删卷的 `delete_volume_if_present`
只在 `cmd_destroy` 中、且被 `--volumes` 判断包裹。所以换镜像不影响
PostgreSQL、Redis 与应用数据。

### 回滚

上游镜像仍缓存在部署机上，回滚只需改回地址重新 `up`，无需重新下载：

```bash
cd ~/sub2api-apple
sed -i '' 's|^APPLE_CONTAINER_SUB2API_IMAGE=.*|APPLE_CONTAINER_SUB2API_IMAGE=ghcr.io/wei-shaw/sub2api:0.2.7|' .env
./apple-container.sh up
```

### 站点名称与 Logo

`site_name`、`site_subtitle`、`site_logo` 是**数据库设置**，在管理后台 →
系统设置里改，改完立即生效，不需要重新构建镜像。只有 `site_logo` 留空时
才会回落到镜像内置的 `/logo.svg`。

## 不在本仓库中的文件

以下文件只存在于部署机上，**不会**提交（见 `.gitignore`）：

- `.env` 及其历史备份 —— 数据库/Redis/管理员密码、JWT 与 TOTP 密钥
- `tls/*.key`、`tls/*.crt` —— 自签 CA 与服务器证书，按机器生成
- `*.dump`、`*.rdb`、`*.tsv` —— 数据库导出与运行期快照

`tls/openssl-san.cnf` 保留在仓库里作为签发证书的模板，里面的 SAN 需要按自己的主机名和网段改。

## TLS

```bash
openssl req -x509 -newkey rsa:4096 -nodes -days 3650 \
  -keyout tls/ca.key -out tls/ca.crt -subj "/CN=sub2api-local-ca"

openssl req -new -newkey rsa:2048 -nodes \
  -keyout tls/server.key -out tls/server.csr -config tls/openssl-san.cnf

openssl x509 -req -in tls/server.csr -CA tls/ca.crt -CAkey tls/ca.key \
  -CAcreateserial -out tls/server.crt -days 825 \
  -extfile tls/openssl-san.cnf -extensions req_ext
```
