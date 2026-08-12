# PostgreSQL Docker + pgBackRest 配置问题日志

## 一、环境概况

* Docker 容器运行 PostgreSQL 15
* 数据库：`gsein`
* PostgreSQL 容器名：`postgres`
* 数据目录映射：`/var/lib/postgresql/data`
* pgBackRest 配置文件路径：`/var/lib/pgbackrest/pgbackrest.conf`
* pgBackRest 存储仓库：`/var/lib/pgbackrest`
* 数据库用户：`postgres`

---

## 二、配置过程与遇到的问题

### 1️⃣ 初始配置 pgBackRest

**操作：**

* 创建配置文件 `/var/lib/pgbackrest/pgbackrest.conf`，内容如下：

```ini
[global]
log-level-console=info
log-level-file=debug
log-path=/var/log/pgbackrest
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2
repo1-retention-diff=7
repo1-retention-incr=7
repo1-cipher-type=aes-256-cbc
repo1-cipher-pass=axelor
repo1-bundle=y
repo1-block=y
compress-type=gz
compress-level=6
process-max=4
start-fast=y

[main]
pg1-path=/var/lib/postgresql/data
pg1-port=5432
pg1-socket-path=/var/run/postgresql

[global:archive-push]
compress-level=3
```

**问题：**

```bash
docker exec postgres pgbackrest --stanza=main check
P00  ERROR: [037]: check command requires option: pg1-path
HINT: does this stanza exist?
```

**分析：**

* pgBackRest 尚未创建 stanza 元信息。

---

### 2️⃣ 创建 Stanza

**操作：**

```bash
docker exec -u postgres postgres \
pgbackrest --stanza=main --config=/var/lib/pgbackrest/pgbackrest.conf stanza-create
```

**结果：**

* 成功创建 stanza
* 允许执行 `check` 命令验证数据库和仓库配置。

---

### 3️⃣ 验证配置 check

**操作：**

```bash
docker exec -u postgres postgres \
pgbackrest --stanza=main --config=/var/lib/pgbackrest/pgbackrest.conf check
```

**遇到问题 1：**

```
ERROR: [087]: archive_mode must be enabled
```

**解决：**

* 修改 `postgresql.conf`，确保启用：

```conf
archive_mode = on
archive_command = 'pgbackrest --stanza=main archive-push %p'
```

* 重启容器：

```bash
docker restart postgres
```

**遇到问题 2：**

```
ERROR: [055]: unable to load info file '/var/lib/pgbackrest/archive/main/archive.info'
```

**原因：**

* Stanza 创建后还未推送 WAL
* 归档目录为空

**解决：**

* 确认仓库目录存在：

```bash
docker exec -u postgres postgres mkdir -p /var/lib/pgbackrest/archive/main
```

* 强制切换 WAL 测试：

```bash
docker exec -it postgres psql -U axelor -d gsein -c "SELECT pg_switch_wal();"
```

---

### 4️⃣ WAL 归档报错

**报错：**

```
ERROR: [037]: option 'pg1-path' must be specified when relative wal paths are used
```

**分析：**

* 即便 `archive_command` 使用 `%p`，Docker 容器环境可能传相对路径
* pgBackRest 需要显式指定 `pg1-path`

**解决：**

```conf
archive_command = 'pgbackrest --stanza=main --config=/var/lib/pgbackrest/pgbackrest.conf --pg1-path=/var/lib/postgresql/data archive-push %p'
```

* 重启容器
* 再次 `check` 成功

---

### 5️⃣ 全量备份执行

**命令：**

```bash
docker exec -u postgres postgres \
pgbackrest --stanza=main --type=full --config=/var/lib/pgbackrest/pgbackrest.conf backup
```

**验证备份：**

```bash
docker exec -u postgres postgres \
pgbackrest --stanza=main --config=/var/lib/pgbackrest/pgbackrest.conf info
```

* 输出显示全量备份时间、大小、WAL 范围，状态 OK
* 备份文件存放在：

```
/var/lib/pgbackrest/backup/main/
```

* 归档 WAL 文件存放在：

```
/var/lib/pgbackrest/archive/main/
```

---

### 6️⃣ 自动定时备份配置

**备份策略：**

* 全量备份：每天 0-6 点（示例选 2 点）
* 增量备份：每天中午 12 点和晚上 20 点

**Cron 配置示例：**

```cron
# 全量备份
0 2 * * * docker exec -u postgres postgres pgbackrest --stanza=main --config=/var/lib/pgbackrest/pgbackrest.conf --type=full backup >> /var/log/pgbackrest/full.log 2>&1

# 增量备份
0 12 * * * docker exec -u postgres postgres pgbackrest --stanza=main --config=/var/lib/pgbackrest/pgbackrest.conf --type=incr backup >> /var/log/pgbackrest/incr.log 2>&1
0 20 * * * docker exec -u postgres postgres pgbackrest --stanza=main --config=/var/lib/pgbackrest/pgbackrest.conf --type=incr backup >> /var/log/pgbackrest/incr.log 2>&1
```

**注意事项：**

* 日志目录 `/var/log/pgbackrest` 必须存在且可写
* 容器内使用 `postgres` 用户执行
* pgBackRest 配置文件必须正确映射

---

### 7️⃣ 查看备份

* 查看备份信息：

```bash
docker exec -u postgres postgres pgbackrest --stanza=main --config=/var/lib/pgbackrest/pgbackrest.conf info
```

* 查看备份文件：

```bash
docker exec -it postgres ls -lh /var/lib/pgbackrest/backup/main/
```

* 查看 WAL 归档：

```bash
docker exec -it postgres ls -lh /var/lib/pgbackrest/archive/main/
```

* 测试恢复（非生产环境）：

```bash
docker exec -u postgres postgres mkdir -p /var/lib/postgresql/restore_test
docker exec -u postgres postgres \
pgbackrest --stanza=main --delta --type=full --db-path=/var/lib/postgresql/restore_test restore
```

---

## 三、总结经验

1. Docker 容器下 `%p` 可能被解析为相对路径，需要 `--pg1-path` 明确指定
2. 每次执行 `backup` 前，stanza 必须创建成功
3. archive_mode + archive_command 配置必须正确，且 PostgreSQL 能执行 pgBackRest
4. 容器内日志路径可能为空，需要通过 `docker logs` 或启用 logging_collector 查看
5. 定时备份建议使用宿主机 cron 结合 `docker exec`，避免容器内复杂 cron 配置



