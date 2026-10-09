# 一键升级 / One-click upgrade

[中文](#中文) | [English](#english)

---

## 中文

有新版本时，**设置 → 更新** 会出现「立即升级」按钮。点击后，与 Bemby 同在一个 docker-compose.yml 中的 [Watchtower](https://github.com/nicholas-fedor/watchtower) 会拉取新镜像，并用相同的环境变量、数据卷和端口重建 Bemby 容器。页面显示「升级中…」，新版本启动后自动刷新。

- 拉取期间面板照常可用；重建时短暂离线。
- 正在运行的任务会被中断，新版本启动后自动重新运行（与普通重启相同）。
- 数据目录是挂载卷，升级后保留；数据库迁移在新版本启动时自动完成。

### 设置

仓库中的 `docker-compose.yml` 已包含 Watchtower 服务。只需在 `.env` 中设置令牌：

```bash
WATCHTOWER_TOKEN=$(openssl rand -hex 32)
```

然后：

```bash
docker compose up -d
```

未设置 `WATCHTOWER_TOKEN` 时，`docker compose up` 会提示并拒绝启动。不想使用一键升级？删除 `watchtower` 服务和 bemby 服务中的 `WATCHTOWER_TOKEN` 一行即可，设置页会恢复为显示手动升级命令。

旧的 docker-compose.yml：设置页的「设置一键升级」中有可复制的配置片段。

### 安全性

- **只有 Watchtower 拿到 Docker socket**。Bemby 会用浏览器访问外部网站、运行任务脚本、解析群消息，不应持有等同宿主机 root 的 socket。
- Bemby 只持有一个令牌，只能请求 Watchtower 更新 Bemby 自己的镜像（`liveinaus/bemby-pro`，沿用当前标签）；无法运行其他镜像或访问宿主机。
- Watchtower 只启用更新接口、不做定时更新、只管理带 `com.centurylinklabs.watchtower.enable=true` 标签的容器（即 bemby），其 API 不映射到宿主机端口，只在 compose 网络内可访问。
- 地址和令牌都来自环境变量，面板中无法修改。

### 注意事项

- **固定版本标签不会升级**：使用 `liveinaus/bemby-pro:1.5.0` 这类标签时，Watchtower 只会重新拉取同一版本。请使用 `:latest`、`:beta` 或 `:dev`。
- **Docker socket 路径**：rootless Docker 或部分 NAS 不在 `/var/run/docker.sock`，请相应修改，或删除 Watchtower 服务。
- **Portainer / NAS 管理的 stack**：Watchtower 直接重建容器，管理界面中显示的配置可能与实际运行的不同步。
- Watchtower 使用 nicholas-fedor 维护的分支（`nickfedor/watchtower`）；原 containrrr/watchtower 已于 2025 年 12 月停止维护。
- 15 分钟内没有看到新版本启动时，请查看 `docker logs watchtower`。

### 可选变量

| 变量 | 作用 |
|---|---|
| `WATCHTOWER_TOKEN` | 必填。Bemby 与 Watchtower 共用的令牌 |
| `WATCHTOWER_URL` | Watchtower 不在同一个 compose 中时填写其地址；默认 `http://watchtower:8080` |
| `BEMBY_UPGRADE_IMAGE` | 从镜像仓库镜像拉取 Bemby 时填写；默认 `liveinaus/bemby-pro` |

---

## English

When a newer build is out, **Settings → Updates** shows an **Upgrade now** button. Pressing it asks [Watchtower](https://github.com/nicholas-fedor/watchtower), running in the same docker-compose.yml, to pull the new image and recreate the Bemby container with the same environment, volumes and ports. The page shows "Upgrading…" and reloads once the new build is up.

- The panel stays usable while the image downloads, and goes away briefly while the container is recreated.
- Jobs running at that moment are interrupted and run again when the new build starts, as after any restart.
- The data directory is a volume and survives the upgrade; schema migrations run when the new build starts.

### Setup

The repository's `docker-compose.yml` already has the Watchtower service. Set the token in `.env`:

```bash
WATCHTOWER_TOKEN=$(openssl rand -hex 32)
```

then:

```bash
docker compose up -d
```

Without `WATCHTOWER_TOKEN`, `docker compose up` stops and says so. Don't want one-click upgrade? Remove the `watchtower` service and the `WATCHTOWER_TOKEN` line in the bemby service; Settings goes back to showing the command to upgrade by hand.

On an older docker-compose.yml, Settings → Updates → *Set up one-click upgrade* has a snippet to copy.

### Security

- **Only Watchtower gets the Docker socket.** Bemby drives a browser against outside sites, runs job scripts and parses group messages; it should not hold what amounts to host root.
- Bemby holds a token that can only ask Watchtower to update Bemby's own image (`liveinaus/bemby-pro`, on the tag it runs). It cannot run another image or reach the host.
- Watchtower has only its update endpoint on, makes no scheduled updates, and manages only containers labelled `com.centurylinklabs.watchtower.enable=true` (bemby). Its API is not published to the host; only the compose network reaches it.
- The address and the token come from the environment and cannot be changed from the panel.

### Things to know

- **A pinned version tag never moves.** On a tag such as `liveinaus/bemby-pro:1.5.0`, Watchtower only pulls the same version again. Use `:latest`, `:beta` or `:dev`.
- **Docker socket path.** Rootless Docker and some NAS systems keep it somewhere other than `/var/run/docker.sock`; change the mount, or remove the Watchtower service.
- **Stacks managed by Portainer or a NAS UI.** Watchtower recreates the container itself, so what the UI shows may drift from what runs.
- Watchtower is nicholas-fedor's maintained fork (`nickfedor/watchtower`); the original containrrr/watchtower was archived in December 2025.
- If no new build comes up within 15 minutes, check `docker logs watchtower`.

### Variables

| Variable | Purpose |
|---|---|
| `WATCHTOWER_TOKEN` | Required. The token Bemby and Watchtower share |
| `WATCHTOWER_URL` | Only for a Watchtower outside this compose file; default `http://watchtower:8080` |
| `BEMBY_UPGRADE_IMAGE` | Only when pulling Bemby from a mirror; default `liveinaus/bemby-pro` |
