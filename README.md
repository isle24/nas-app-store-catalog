# NAS 应用商店目录与部署

当前商店公开 176 条应用记录，其中 13 款提供安装入口，163 条资料明确标注待适配。飞牛来源的 175 项均已收录，另保留 nasReader；原生 FPK 与复杂 Docker 定义不会被伪装成可安装的单容器。

- 公共资料：`catalog-apps.json`，以及 [服务器公开目录](https://zstore.2iwm.com/api/cloud/public-catalog)。
- 签名安装目录：`catalog-envelope.json`、`catalog-key.json`、`catalogs/` 与 `manifests/`。当前 7 个云交付使用原签名身份与不可变摘要，NAS 另有 6 个已审查的内置配方。
- `drafts/` 保留历史协议示例，它们不是已发布安装源。`schemas/` 保存协议结构。
- 用户账户、密码、令牌、签名私钥与 NAS 私有数据不在本仓库。

源码在私有 [zspace-apps](https://github.com/isle24/zspace-apps)，独立 NAS 商店使用 [zspace-shop 镜像](https://hub.docker.com/r/isle204/zspace-shop)；第三方桌面使用 [UI 镜像](https://hub.docker.com/r/isle204/zspace-ui)，中央管理后台使用 [后台镜像](https://hub.docker.com/r/isle204/zspace-store)。

<!-- combined-ui-shop-start -->
## UI 和商店合并部署

适用于全新部署的 AMD64 极空间，Docker 27.3 或更新版本，NAS 官方服务使用默认 5055 端口。把下面所有 `192.168.3.56` 替换为自己的 NAS 局域网 IP，直接粘贴到 Docker Compose 即可；不需要 `.env`、SSH 初始化或 PostgreSQL。

```yaml
# Replace every 192.168.3.56 with your NAS LAN IP. Fresh AMD64 ZSpace deployment.
name: zspace-suite
services:
  ui:
    image: isle204/zspace-ui:latest
    pull_policy: always
    platform: linux/amd64
    network_mode: bridge
    user: "10001:10001"
    init: true
    restart: unless-stopped
    read_only: true
    cap_drop: [ALL]
    security_opt: [no-new-privileges:true]
    labels:
      io.space-store.service: space-store
      io.space.role: ui
    ports: ["192.168.3.56:18436:18436"]
    environment:
      NAS_ORIGIN: http://192.168.3.56:5055
      UI_PUBLIC_ORIGIN: http://192.168.3.56:18436
      SPACE_STORE_ENDPOINT: http://192.168.3.56:18443
    volumes: [ui-data:/data]
    tmpfs: ["/tmp:rw,noexec,nosuid,size=32m"]
    depends_on:
      shop:
        condition: service_healthy

  shop:
    image: isle204/zspace-shop:latest
    pull_policy: always
    platform: linux/amd64
    network_mode: bridge
    restart: unless-stopped
    read_only: true
    cap_drop: [ALL]
    security_opt: [no-new-privileges:true]
    stop_grace_period: 45s
    labels:
      io.space-store.service: space-store
      io.space.role: standalone-shop-bootstrap
    environment:
      NAS_IP: 192.168.3.56
      SHOP_PORT: "18443"
      SHOP_UI_ORIGIN: http://192.168.3.56:18436
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - shop-control:/control
    tmpfs: ["/tmp:rw,noexec,nosuid,size=32m"]

volumes:
  ui-data:
  shop-control:
```

先打开 `http://你的NAS地址:18443/` 创建独立商店管理员，再打开 `http://你的NAS地址:18436/`，使用自己的极空间账号登录 UI；UI 内的空间商店使用前面创建的商店账号。两个账号各自独立。

`SHOP_UI_ORIGIN` 明确允许 UI 调用商店；首次创建账号仍在 18443 的直接页面完成。商店会自动生成自己的运行容器，无需另行安装执行器。镜像默认 `latest`；更新时保留原项目、目录和数据卷。

已有独立商店接入 UI 时，保留原项目名、`shop-control` 卷、NAS 地址、端口和数据目录，升级至 `2026.10.15-r2` 或更新镜像，并新增 `SHOP_UI_ORIGIN`。不要将上面的新项目名套用到旧商店。升级会核对并保留原账号与目录；已有 UI 来源的替换或删除不在自动迁移范围。
<!-- combined-ui-shop-end -->

<!-- standalone-shop-start -->
## 独立商店 Docker（无需第三方桌面）

只需要应用商店的极空间玩家，部署 `isle204/zspace-shop:latest` 即可。首次在网页创建本地管理员账号，之后使用自己的商店账号登录；无需第三方 UI、NAS 会话、SSH 初始化或 PostgreSQL。支持 AMD64、Docker 27.3 及以上。

把 `NAS_IP` 改为自己的 NAS 地址，粘贴到极空间 Docker Compose：

```yaml
# Paste into ZSpace Docker Compose. Change NAS_IP to your NAS LAN address.
name: zspace-shop
services:
  shop:
    image: isle204/zspace-shop:latest
    platform: linux/amd64
    network_mode: bridge
    restart: unless-stopped
    read_only: true
    cap_drop: [ALL]
    security_opt: [no-new-privileges:true]
    stop_grace_period: 45s
    labels:
      io.space-store.service: space-store
      io.space.role: standalone-shop-bootstrap
    environment:
      NAS_IP: 192.168.3.56
      SHOP_PORT: "18443"
      # Optional: an unused dedicated folder on a physical NAS data disk.
      # SHOP_DATA_ROOT: /data_n008/space-shop
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - shop-control:/control
    tmpfs: ["/tmp:rw,noexec,nosuid,size=32m"]
volumes:
  shop-control:
```

启动后打开 `http://你的NAS地址:18443/`。引导服务自动创建并管理独立商店运行容器；数据保存在 NAS 物理数据盘上的独立目录，`shop-control` 保存部署记录。保留控制卷、目录和原配置，停止后重启会复用同一商店。安装时默认应用目录自动核验，也可手动添加或创建专用目录。云端账号用于安装云端发布的应用、备份与同步，可在商店内单独注册登录。更新保留原项目与数据卷，拉取 `latest` 并重新创建即可；版本标签同时保留。

详见[独立商店部署说明](standalone-shop-docker.md)。

<!-- standalone-shop-end -->

## Docker Compose 直接复制部署

镜像使用 `latest`，本轮版本为 `2026.10.15-r2`，平台 `linux/amd64`。新用户使用上方组合配置；下方独立 UI 示例适用于已有受管控制器的部署，云后台示例用于自行运营目录。

## NAS 三方 UI

把 `192.168.3.56` 改为实际 NAS 地址，并确认官方 API 为 `5055`、已有受管安装控制器为 `18441`。控制器的 NAS 身份来源须与 UI_PUBLIC_ORIGIN 相同。当前 NAS 已部署 UI，不要在同一端口同时启动第二个实例；迁移保留原控制器和数据。

```yaml
# Save as compose.yaml; replace all example NAS addresses together.
name: zspace-ui
services:
  ui:
    image: isle204/zspace-ui:latest
    platform: linux/amd64
    user: 10001:10001
    restart: unless-stopped
    init: true
    read_only: true
    cap_drop: [ALL]
    security_opt: [no-new-privileges:true]
    labels:
      io.space-store.service: space-store
      io.space.role: ui
    ports: ["192.168.3.56:18436:18436"]
    environment:
      NAS_ORIGIN: http://192.168.3.56:5055
      UI_PUBLIC_ORIGIN: http://192.168.3.56:18436
      SPACE_STORE_ENDPOINT: http://192.168.3.56:18441
    volumes: [ui-data:/data]
    # Optional authorized host Docker management: uncomment both additions,
    # replace 989 with `stat -c '%g' /var/run/docker.sock`, then recreate ui.
    # group_add: ["989"]
    # Add "/var/run/docker.sock:/var/run/docker.sock:ro" to volumes above.
    tmpfs: ["/tmp:rw,noexec,nosuid,size=32m"]
    healthcheck:
      test: [CMD-SHELL, "wget -qO- http://127.0.0.1:18436/healthz >/dev/null"]
      interval: 30s
      timeout: 3s
      start_period: 15s
      retries: 3
volumes:
  ui-data:
```

```bash
docker compose up -d
```

打开 `http://NAS_IP:18436/`。需要管理宿主 Docker 时，按 YAML 注释添加 socket 和其 GID；默认通过现有 NAS 服务访问设备，安装功能由已配置的本地控制器执行。

## 云商店后台

将 `SPACE_PUBLIC_ORIGIN` 改为实际 HTTPS 来源。这里仅绑定本机 `127.0.0.1:18442`，现有 HTTPS 反向代理转发到该端口；本机测试可以把来源改为 `http://127.0.0.1:18442`。当前 `zstore.2iwm.com` 已有后台，已有实例迁移需恢复原数据库、私有元数据和密码卷。

```yaml
# Save as compose.yaml. No .env, host secret file or deployment script required.
name: zspace-store
services:
  secret-init:
    image: isle204/zspace-store:latest
    platform: linux/amd64
    restart: "no"
    read_only: true
    cap_drop: [ALL]
    cap_add: [SETUID, SETGID]
    security_opt: [no-new-privileges:true]
    labels: {io.space-store.service: space-store, io.space.role: secret-init}
    entrypoint: [/bin/sh, -ec]
    command:
      - |
        set +x
        umask 077
        task_secret=/secret-store/postgres-password
        if [ -e "$$task_secret" ] || [ -L "$$task_secret" ]; then
          [ -f "$$task_secret" ] && [ ! -L "$$task_secret" ] && [ -s "$$task_secret" ] &&
            [ "$$(stat -c '%a' "$$task_secret")" = 600 ] || {
              echo 'Existing database secret is invalid; restore the original secret.' >&2; exit 1;
            }
          exit 0
        fi
        # Both images create these data directories with private ownership.
        # Switch to their owners to inspect without DAC override permissions.
        task_pg_owner="$$(stat -c '%u:%g' /existing-postgres)"
        task_catalog_owner="$$(stat -c '%u:%g' /data)"
        task_pg_entries="$$(su-exec "$$task_pg_owner" ls -A /existing-postgres)"
        task_catalog_entries="$$(su-exec "$$task_catalog_owner" ls -A /data)"
        if [ -n "$$task_pg_entries" ] || [ -n "$$task_catalog_entries" ]; then
          echo 'Persistent database or catalog data exists but its secret is missing. Restore the original secret volume.' >&2
          exit 1
        fi
        chmod 700 /secret-store
        set -C
        od -An -N32 -tx1 /dev/urandom | tr -d ' \n' > "$$task_secret"
        [ "$$(wc -c < "$$task_secret")" -eq 64 ]
        chmod 600 "$$task_secret"
    volumes:
      - database-secret:/secret-store
      - postgres-data:/existing-postgres:ro
      - catalog-data:/data:ro
  postgres:
    image: postgres:17
    platform: linux/amd64
    restart: unless-stopped
    labels: {io.space-store.service: space-store, io.space.role: database}
    depends_on:
      secret-init: {condition: service_completed_successfully}
    environment:
      POSTGRES_USER: zstore
      POSTGRES_DB: zstore
      POSTGRES_PASSWORD_FILE: /secret-store/postgres-password
      POSTGRES_INITDB_ARGS: --auth-host=scram-sha-256
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - database-secret:/secret-store:ro
    networks: [database]
    healthcheck:
      test: [CMD-SHELL, "pg_isready -U zstore -d zstore"]
      interval: 10s
      timeout: 3s
      start_period: 15s
      retries: 10
  catalog:
    image: isle204/zspace-store:latest
    platform: linux/amd64
    restart: unless-stopped
    init: true
    read_only: true
    cap_drop: [ALL]
    cap_add: [SETUID, SETGID]
    security_opt: [no-new-privileges:true]
    labels: {io.space-store.service: space-store, io.space.role: catalog}
    depends_on:
      postgres: {condition: service_healthy}
    ports: ["127.0.0.1:18442:18442"]
    environment:
      SPACE_PUBLIC_ORIGIN: https://zstore.example.com
      SPACE_ADMIN_2FA_REQUIRED: "1"
      SPACE_IMAGE_CACHE_DEFAULT_BYTES: "5368709120"
      SPACE_DB_HOST: postgres
      SPACE_DB_USER: zstore
      SPACE_DB_NAME: zstore
      SPACE_DB_PASSWORD_FILE: /secret-store/postgres-password
    volumes:
      - catalog-data:/data
      - database-secret:/secret-store:ro
    tmpfs: ["/tmp:rw,noexec,nosuid,size=32m"]
    networks: [default, database]
    healthcheck:
      test: [CMD-SHELL, "wget -qO- http://127.0.0.1:18442/api/health >/dev/null"]
      interval: 30s
      timeout: 3s
      start_period: 20s
      retries: 3
  setup:
    profiles: [setup]
    image: isle204/zspace-store:latest
    platform: linux/amd64
    user: 10001:10001
    init: true
    read_only: true
    cap_drop: [ALL]
    security_opt: [no-new-privileges:true]
    labels: {io.space-store.service: space-store, io.space.role: setup}
    stdin_open: true
    tty: true
    entrypoint: [/usr/local/bin/setup-admin]
    volumes: [catalog-data:/data]
    tmpfs: ["/tmp:rw,noexec,nosuid,size=32m"]
networks:
  database:
    internal: true
volumes:
  database-secret:
  postgres-data:
  catalog-data:
```

```bash
docker compose up -d
```

数据库密码在首次运行时随机生成，保存在独立私有卷；重复运行沿用原密码。已有数据而密码丢失时初始化会停止，要求恢复原密码卷。

首次初始化管理员，在这个 Compose 目录执行一条命令：

```bash
docker compose stop catalog && docker compose run --rm -it setup && docker compose up -d
```

按提示输入用户名、密码和确认密码，密码不回显。然后打开 `https://你的域名/admin/`，扫码并输入正确动态验证码完成 2FA 绑定。此命令的元数据引导服务仅监听容器内回环地址。

四个持久卷保存 UI 数据、云私有元数据、PG 数据和 DB 密码。更新时只更换镜像版本并 `docker compose up -d`，保留数据卷。生产账户和目录状态不装入 Docker 镜像；新实例与已有服务的数据迁移见[备份与迁移](deploy/BACKUP.md)。

应用公开目录提供 176 条记录：13 款有安装入口（6 个已有 NAS 配方及 7 个签名云交付），163 条公开资料待适配。新增 6 款通过本轮镜像与部署清单核验；未把核验失败的镜像或飞牛原生 FPK 声明为可安装。


## 数据来源

飞牛应用资料来源于 [conversun/fnos-apps](https://github.com/conversun/fnos-apps/tree/d297f9ac155b15a26b92632ee045ff4d530e2571)，固定提交 `d297f9ac155b15a26b92632ee045ff4d530e2571`；来源元资料按 GPLv3 归档，完整许可见 `sources/fnos/LICENSE`。具体应用软件与镜像按各自许可证使用，安装及首次配置由应用自身处理。
