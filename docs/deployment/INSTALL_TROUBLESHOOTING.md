# Installation Troubleshooting / 安装排错指南

This guide collects common first-time Docker installation issues for QuantDinger, especially on Windows with Docker Desktop and a local proxy.

本文整理 QuantDinger 首次 Docker 安装时常见的问题，尤其适用于 Windows、Docker Desktop、本地代理环境。

## Quick Checks / 快速检查

Run these commands from the QuantDinger repository root, next to `docker-compose.yml`.

请在 QuantDinger 仓库根目录执行，也就是 `docker-compose.yml` 所在目录。

```bash
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail=100 postgres
docker compose logs --tail=100 backend
```

## Docker Hub Pull Fails / Docker Hub 镜像拉取失败

### Symptom / 现象

You may see errors like:

可能会看到类似错误：

```text
failed to resolve reference "docker.io/library/redis:8-alpine"
failed to do request: Head "https://registry-1.docker.io/v2/library/redis/manifests/7-alpine"
connecting to registry-1.docker.io:443: connectex: A connection attempt failed
```

or:

或者：

```text
container via direct connection because Docker Desktop has no HTTPS proxy
```

### Cause / 原因

Docker Desktop cannot reach Docker Hub from its own internal network. A browser or terminal may work, but Docker Desktop may still be using a different proxy path.

Docker Desktop 的内部网络无法访问 Docker Hub。浏览器或普通终端能访问，并不代表 Docker Desktop 拉镜像时也能走同一个代理。

### Fix / 解决方法

First find the working local proxy port. Common ports are `7890`, `7897`, `10808`, and `10809`.

先确认本机真正可用的代理端口。常见端口有 `7890`、`7897`、`10808`、`10809`。

```powershell
curl.exe -x http://127.0.0.1:10808 https://registry-1.docker.io/v2/
```

If the proxy works, Docker Hub should return `UNAUTHORIZED`. That is expected because this Registry API endpoint requires Docker authentication.

如果代理可用，Docker Hub 通常会返回 `UNAUTHORIZED`。这是正常的，因为这个 Registry API 地址需要 Docker 自己的认证 token。

```json
{"errors":[{"code":"UNAUTHORIZED","message":"authentication required"}]}
```

Then configure Docker Desktop:

然后配置 Docker Desktop：

1. Open Docker Desktop.
2. Go to `Settings` -> `Resources` -> `Proxies`.
3. Select `Manual configuration`.
4. Set both proxy fields to the working port, for example:

1. 打开 Docker Desktop。
2. 进入 `Settings` -> `Resources` -> `Proxies`。
3. 选择 `Manual configuration`。
4. 两个代理地址都填可用端口，例如：

```text
Web Server (HTTP):         http://127.0.0.1:10808
Secure Web Server (HTTPS): http://127.0.0.1:10808
```

If Docker still pulls directly, set `Containers proxy` to `Same as host proxy`, apply the change, then fully restart Docker Desktop from the system tray.

如果 Docker 仍然直连，把下面的 `Containers proxy` 改成 `Same as host proxy`，保存后从系统托盘完全退出并重启 Docker Desktop。

Verify Docker Desktop has picked up the proxy:

验证 Docker Desktop 是否已经读取代理配置：

```powershell
docker info | findstr /i proxy
```

Typical output:

典型输出：

```text
HTTP Proxy: http.docker.internal:3128
HTTPS Proxy: http.docker.internal:3128
No Proxy: hubproxy.docker.internal
```

`http.docker.internal:3128` is normal. Docker Desktop maps your host proxy to an internal proxy address.

看到 `http.docker.internal:3128` 是正常的。Docker Desktop 会把宿主机代理映射成内部代理地址。

Retry:

重新执行：

```powershell
docker compose pull
docker compose up -d
```

## Browser Shows `UNAUTHORIZED` / 浏览器显示 `UNAUTHORIZED`

Opening this URL in a browser may show `UNAUTHORIZED`:

浏览器打开下面地址时可能显示 `UNAUTHORIZED`：

```text
https://registry-1.docker.io/v2/library/redis/manifests/8-alpine
```

This is not an image problem. It means the Docker Registry endpoint is reachable, but the browser did not send Docker's authentication token.

这不是镜像有问题。它说明 Docker Registry 地址可以访问，只是浏览器没有携带 Docker 拉镜像所需的认证 token。

If the network were blocked, the browser would usually time out or fail to connect instead.

如果网络不通，浏览器通常会超时或连接失败，而不是返回 `UNAUTHORIZED`。

## Proxy Port Does Not Work / 代理端口不可用

### Symptom / 现象

`curl` through the proxy fails:

通过代理执行 `curl` 失败：

```powershell
curl.exe -x http://127.0.0.1:10809 https://registry-1.docker.io/v2/
```

```text
Failed to connect to registry-1.docker.io port 443 via 127.0.0.1
Could not connect to server
```

### Cause / 原因

The configured proxy port is not listening, or it is not an HTTP proxy port.

当前填写的代理端口没有服务监听，或者它不是 HTTP 代理端口。

### Fix / 解决方法

Check common local proxy ports:

检查常见本地代理端口：

```powershell
netstat -ano | findstr 7890
netstat -ano | findstr 7897
netstat -ano | findstr 10808
netstat -ano | findstr 10809
```

Then test the port that is actually listening:

然后测试实际正在监听的端口：

```powershell
curl.exe -x http://127.0.0.1:10808 https://registry-1.docker.io/v2/
```

If your proxy client supports it, enable `Allow LAN` or `Allow connections from LAN`. Docker Desktop sometimes needs access from its internal VM/network rather than only the Windows loopback path.

如果代理软件支持，请开启 `Allow LAN` / `允许来自局域网的连接`。Docker Desktop 有时需要从内部虚拟机网络访问代理，而不只是 Windows 自己的 `127.0.0.1`。

## Postgres Container Is Unhealthy / Postgres 容器不健康

### Symptom / 现象

`docker compose up -d` fails with:

执行 `docker compose up -d` 失败：

```text
dependency postgres failed to start
container quantdinger-db is unhealthy
```

Postgres logs show:

Postgres 日志显示：

```text
FATAL: database files are incompatible with server
DETAIL: The data directory was initialized by PostgreSQL version 16, which is not compatible with this version 18.3.
```

### Cause / 原因

The existing Docker volume or mounted data directory was initialized by PostgreSQL 16, but the current Compose stack is starting PostgreSQL 18. PostgreSQL data directories cannot be reused across major versions without a migration.

已有 Docker volume 或挂载的数据目录是 PostgreSQL 16 初始化的，但当前 Compose 启动的是 PostgreSQL 18。PostgreSQL 的数据目录不能跨大版本直接复用，必须迁移。

### Fix if Local Data Is Disposable / 本地数据可以清空时

For a fresh development setup where local database data is not important:

如果只是开发环境，本地数据库数据不重要，可以直接清空 volume：

```powershell
docker compose down -v
docker compose up -d
```

Warning: `-v` deletes Compose volumes, including the local Postgres data volume.

注意：`-v` 会删除 Compose volumes，包括本地 Postgres 数据。

### Fix if Data Must Be Kept / 需要保留数据时

Do not delete the volume. Either:

不要删除 volume。可以选择：

1. Start with the same PostgreSQL major version that created the data directory, for example `postgres:16-alpine`.
2. Export and import data with `pg_dump` / `pg_restore`.
3. Use a proper PostgreSQL major-version migration process such as `pg_upgrade`.

1. 改回创建该数据目录时使用的 PostgreSQL 大版本，例如 `postgres:16-alpine`。
2. 使用 `pg_dump` / `pg_restore` 导出并恢复数据。
3. 使用正式的 PostgreSQL 大版本迁移流程，例如 `pg_upgrade`。

## Administrator Credentials and Settings Save / 管理员凭据与设置保存

Choose a language / 请选择语言：

- [English](ADMIN_AND_SETTINGS_TROUBLESHOOTING_EN.md)
- [中文](ADMIN_AND_SETTINGS_TROUBLESHOOTING_CN.md)

## Useful Recovery Commands / 常用恢复命令

Check service status:

查看服务状态：

```powershell
docker compose ps
```

Read service logs:

查看服务日志：

```powershell
docker compose logs --tail=100 postgres
docker compose logs --tail=100 redis
docker compose logs --tail=100 backend
```

Pull images again:

重新拉取镜像：

```powershell
docker compose pull
```

Recreate containers without deleting data:

重建容器但保留数据：

```powershell
docker compose up -d --force-recreate
```

Stop and remove containers but keep volumes:

停止并删除容器，但保留 volume：

```powershell
docker compose down
```

Stop and remove containers and volumes:

停止并删除容器和 volume：

```powershell
docker compose down -v
```

Use `down -v` only when you are sure the local database data can be deleted.

只有确认本地数据库数据可以删除时，才使用 `down -v`。
