# 云备份（无持久化存储时使用） / Cloud backup (for hosts without a persistent disk)

[中文](#中文) | [English](#english)

---

## 中文

Bemby 把所有数据保存在容器内的 SQLite 数据库中。如果部署平台没有**持久化存储卷**，每次重启或重新部署，数据都会清空。

开启云备份后，Bemby 会把数据同步到一个 S3 兼容的存储桶（推荐 **Cloudflare R2**，默认设置在其免费额度内）：

- **启动时**：先从存储桶恢复数据库和需要保留的文件，然后再启动面板。
- **运行中**：数据库的每次改动每 10 秒上传一次；文件夹有变化时每 5 分钟上传一次。
- **正常停止 / 重新部署**：不丢任何数据。
- **意外崩溃**：最多丢失最后约 10 秒的数据库写入。

已有持久化存储的用户也可以开启，作为异地备份。

### 会备份哪些内容

| 内容 | 是否备份 |
|---|---|
| 数据库：账号、会话、任务、模板、设置、日志 | ✅ 持续同步 |
| 上传的任务图标 `job-icons` | ✅ |
| 头像图片池 `avatars` | ✅ |
| 浏览器配置 `cf-profiles`（不含缓存） | ✅ |
| 浏览器、字体、xray、x11vnc | ❌ 需要时会自动重新下载，重启后首次使用会稍慢 |
| 任务截图 `run-shots` | ❌ 不重要，不备份 |

### 第 1 步：创建 Cloudflare R2 存储桶

1. 登录 [Cloudflare 控制台](https://dash.cloudflare.com)，左侧菜单进入 **R2 对象存储**（R2 Object Storage）。
   首次使用需要开通 R2，并**绑定一张银行卡**（在免费额度内不会扣费）。
2. 点击 **创建存储桶**（Create bucket），名称例如 `bemby-backup`，其他保持默认。
3. 进入存储桶的 **设置**（Settings）页，复制 **S3 API** 地址，形如
   `https://<账户ID>.r2.cloudflarestorage.com/bemby-backup`（末尾带不带存储桶名称都可以）。

### 第 2 步：创建访问密钥

1. 回到 R2 首页，点击 **管理 API 令牌**（Manage API tokens）→ **创建 API 令牌**（Create API token）。
2. 权限选择 **对象读和写**（Object Read & Write），并限定为刚才创建的存储桶。
3. 创建后会显示 **访问密钥 ID**（Access Key ID）和 **机密访问密钥**（Secret Access Key）。**机密访问密钥只显示一次**，请立即保存。

### 第 3 步：在部署平台设置环境变量

| 变量 | 值 |
|---|---|
| `BACKUP_S3_BUCKET` | 存储桶名称，例如 `bemby-backup` |
| `BACKUP_S3_ENDPOINT` | 第 1 步复制的 S3 API 地址 |
| `BACKUP_S3_ACCESS_KEY_ID` | 访问密钥 ID |
| `BACKUP_S3_SECRET_ACCESS_KEY` | 机密访问密钥 |

保存后重新部署即可。

### 第 4 步：确认已生效

在部署日志中应能看到：

```
[backup] holding the backup lock at bemby-backup/bemby/lock
[backup] no backup yet: starting a new database
```

之后重启一次，日志应显示 `[backup] restored the database`，面板中的数据也都还在。

### 重要：同一时间只能运行一个实例

两个容器同时写入同一个备份会导致备份损坏。Bemby 会在存储桶中加锁来防止这种情况：新容器发现旧容器仍在运行时，会**等待旧容器停止**后再启动。

- 如果平台采用"零停机 / 重叠部署"（新容器先启动，旧容器后停止），部署时新容器最多会等待约 4 分钟。平台如提供关闭重叠部署的选项，建议关闭。
- 不要让两个 Bemby 实例使用同一个存储桶路径。多个实例共用一个存储桶时，为每个实例设置不同的 `BACKUP_S3_PREFIX`。
- 旧容器被强制终止（没有正常停止）时，锁会在约 3 分钟后自动失效，无需手动处理。

### 费用

按 Cloudflare R2 免费额度（每月 10 GB 存储、100 万次 A 类操作、1000 万次 B 类操作，下载流量免费）：

- 即使数据库一刻不停地写入，默认设置每月也只用约 **41 万次 A 类操作**。
- 存储通常只有几 MB 到几十 MB。

**请不要把 `BACKUP_SYNC_INTERVAL` 调低到 1 秒**，那样繁忙时会超出免费额度。

### 高级选项

| 变量 | 默认值 | 说明 |
|---|---|---|
| `BACKUP_S3_REGION` | `auto` | R2 使用 `auto`；其他服务填写存储桶所在区域 |
| `BACKUP_S3_PREFIX` | `bemby` | 存储桶中的文件夹，每个 Bemby 实例一个 |
| `BACKUP_SYNC_INTERVAL` | `10s` | 数据库上传间隔。越短，崩溃时丢失越少，但操作次数越多 |
| `BACKUP_FILES_INTERVAL` | `300` | 检查文件夹变化的间隔（秒） |
| `BACKUP_DIRS` | `job-icons avatars cf-profiles` | 需要备份的数据目录下的文件夹 |
| `BACKUP_LOCK_WAIT` | `600` | 等待其他实例释放锁的最长时间（秒） |

也可使用 AWS S3、Backblaze B2 等其他 S3 兼容服务：填写对应的 `BACKUP_S3_ENDPOINT` 和 `BACKUP_S3_REGION`（AWS S3 只需填写区域）。

### 常见问题

| 日志 | 原因与处理 |
|---|---|
| `BACKUP_S3_BUCKET is set but BACKUP_S3_ACCESS_KEY_ID / BACKUP_S3_SECRET_ACCESS_KEY are not` | 缺少密钥变量，请检查第 3 步 |
| `set BACKUP_S3_ENDPOINT` | 使用 R2 时必须填写 `BACKUP_S3_ENDPOINT` |
| `cannot write to ... (HTTP 403)` | 密钥错误，或令牌没有该存储桶的读写权限 |
| `cannot write to ... (HTTP 404)` | 存储桶名称错误或不存在 |
| `another container ... still holds the backup lock` | 另一个实例仍在运行。停止它，或关闭平台的重叠部署 |
| `the backup lock was taken by another container` | 有另一个实例接管了备份，本实例已自动停止以保护数据 |

**从头开始**：在 R2 中删除存储桶里的 `bemby/` 文件夹（或你设置的 `BACKUP_S3_PREFIX`），然后重新部署。

**恢复到之前的某个时间点**（高级）：存储桶中保留了近期的历史，可用 [Litestream](https://litestream.io) 的 `litestream restore -timestamp` 把数据库恢复到这段历史内的某个时间点。

---

## English

Bemby keeps all its data in a SQLite database inside the container. On a platform without a **persistent volume**, that data is wiped on every restart or redeploy.

With cloud backup turned on, Bemby keeps a copy in an S3-compatible bucket (**Cloudflare R2** recommended; the defaults stay inside its free tier):

- **On start**: the database and the folders worth keeping are restored from the bucket before the panel starts.
- **While running**: database changes are uploaded every 10 seconds; changed folders every 5 minutes.
- **Normal stop / redeploy**: nothing is lost.
- **Crash**: at most the last ~10 seconds of database writes are lost.

Installs that do have a persistent disk can turn it on as an off-site backup too.

### What is backed up

| Data | Backed up |
|---|---|
| Database: accounts, sessions, jobs, templates, settings, logs | ✅ continuously |
| Uploaded job icons (`job-icons`) | ✅ |
| Avatar pool (`avatars`) | ✅ |
| Browser profiles (`cf-profiles`, without caches) | ✅ |
| Browser, fonts, xray, x11vnc | ❌ downloaded again when needed; the first use after a restart is slower |
| Run screenshots (`run-shots`) | ❌ disposable |

### Step 1: Create a Cloudflare R2 bucket

1. Sign in to the [Cloudflare dashboard](https://dash.cloudflare.com) and open **R2 Object Storage** from the left menu.
   The first time, R2 has to be enabled and **a card added** (nothing is charged inside the free tier).
2. Click **Create bucket**, name it e.g. `bemby-backup`, leave the rest as default.
3. Open the bucket's **Settings** tab and copy the **S3 API** URL, e.g.
   `https://<account id>.r2.cloudflarestorage.com/bemby-backup` (with or without the bucket name at the end).

### Step 2: Create an access key

1. Back on the R2 overview, click **Manage API tokens** → **Create API token**.
2. Choose **Object Read & Write** and limit it to the bucket you just created.
3. You get an **Access Key ID** and a **Secret Access Key**. **The secret is shown only once**; save it now.

### Step 3: Set the environment variables on your platform

| Variable | Value |
|---|---|
| `BACKUP_S3_BUCKET` | Bucket name, e.g. `bemby-backup` |
| `BACKUP_S3_ENDPOINT` | The S3 API URL from step 1 |
| `BACKUP_S3_ACCESS_KEY_ID` | Access Key ID |
| `BACKUP_S3_SECRET_ACCESS_KEY` | Secret Access Key |

Save and redeploy.

### Step 4: Check it works

The deploy log should show:

```
[backup] holding the backup lock at bemby-backup/bemby/lock
[backup] no backup yet: starting a new database
```

Restart once: the log should say `[backup] restored the database`, and everything in the panel should still be there.

### Important: only one instance at a time

Two containers writing to the same backup would corrupt it. Bemby prevents this with a lock in the bucket: a new container that finds an old one still running **waits for it to stop** before starting.

- On platforms with zero-downtime / overlapping deploys (the new container starts before the old one stops), a deploy can wait up to about 4 minutes. Turn overlapping deploys off if your platform lets you.
- Never point two Bemby installs at the same bucket path. To share a bucket, give each install its own `BACKUP_S3_PREFIX`.
- If the old container was killed without a clean stop, its lock expires by itself after about 3 minutes. No manual unlock is needed.

### Cost

Against Cloudflare R2's free tier (10 GB-month of storage, 1 million Class A and 10 million Class B operations a month, free egress):

- Even with a database written to non-stop, the defaults use about **410,000 Class A operations** a month.
- Storage is usually a few MB to a few tens of MB.

**Don't lower `BACKUP_SYNC_INTERVAL` to 1s**: a busy install then goes past the free tier.

### Advanced options

| Variable | Default | Meaning |
|---|---|---|
| `BACKUP_S3_REGION` | `auto` | `auto` for R2; the bucket's region for other services |
| `BACKUP_S3_PREFIX` | `bemby` | Folder inside the bucket, one per Bemby install |
| `BACKUP_SYNC_INTERVAL` | `10s` | How often the database is uploaded. Shorter loses less on a crash but costs more operations |
| `BACKUP_FILES_INTERVAL` | `300` | Seconds between checks of the folders |
| `BACKUP_DIRS` | `job-icons avatars cf-profiles` | Folders in the data directory to back up |
| `BACKUP_LOCK_WAIT` | `600` | Longest wait, in seconds, for another instance to release the lock |

Other S3-compatible services (AWS S3, Backblaze B2, ...) work too: set their `BACKUP_S3_ENDPOINT` and `BACKUP_S3_REGION` (AWS S3 needs only the region).

### Troubleshooting

| Log line | Cause and fix |
|---|---|
| `BACKUP_S3_BUCKET is set but BACKUP_S3_ACCESS_KEY_ID / BACKUP_S3_SECRET_ACCESS_KEY are not` | A key variable is missing; see step 3 |
| `set BACKUP_S3_ENDPOINT` | R2 needs `BACKUP_S3_ENDPOINT` |
| `cannot write to ... (HTTP 403)` | Wrong key, or the token has no read & write access to this bucket |
| `cannot write to ... (HTTP 404)` | Wrong bucket name, or the bucket doesn't exist |
| `another container ... still holds the backup lock` | Another instance is still running. Stop it, or turn off overlapping deploys |
| `the backup lock was taken by another container` | Another instance took over the backup; this one stopped itself to protect the data |

**Start from scratch**: delete the `bemby/` folder (or your `BACKUP_S3_PREFIX`) from the bucket in R2, then redeploy.

**Go back to an earlier point in time** (advanced): the bucket holds recent history, and [Litestream](https://litestream.io)'s `litestream restore -timestamp` restores the database as it was at a moment within it.
