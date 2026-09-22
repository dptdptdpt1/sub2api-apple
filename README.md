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
