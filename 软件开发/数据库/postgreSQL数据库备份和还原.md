## 使用pgbackrest备份和还原数据库
### 1. 安装pgbackrest
### 2. 配置pgbackrest
正确配置/etc/pgbackrest/pgbackrest.conf文件，示例如下：
```conf
[main]
pg1-path=/var/lib/postgresql/11/main
pg1-port=5432

[global]
repo1-block=y
repo1-bundle=y
repo1-cipher-pass=luF/FY/1A+1udXgQuQFk4XPfVWjZubz+RM5xcQMnFBhKDJv/7EumyaVJFGijCbWw
repo1-cipher-type=aes-256-cbc
repo1-path=/var/lib/pgbackrest
repo1-retention-diff=1
repo1-retention-full=2
start-fast=y

[global:archive-push]
compress-level=3
```

### 3. 创建stanza
stanza是pgbackrest的备份集，每个备份集可以配置不同的参数，例如备份路径、备份时间、备份保留策略等。
执行以下命令创建stanza：
```bash
sudo -u postgres pgbackrest --stanza=main stanza-create
sudo -u postgres pgbackrest --stanza=main check
```
### 4. 备份数据库
#### 4.1 全量备份
执行以下命令进行全量备份：
```bash
sudo -u postgres pgbackrest --stanza=main --type=full backup
```
--type=full 显式指定全量备份（默认行为也是全量，若首次备份无需指定）。
#### 4.2 增量备份
执行以下命令进行增量备份：
```bash
sudo -u postgres pgbackrest --stanza=main --delta backup
```
### 5. 还原数据库
#### 5.1 全量还原
执行以下命令进行全量还原：
```bash
# 停止 PostgreSQL
sudo systemctl stop postgresql

# 清空数据目录
sudo rm -rf /var/lib/postgresql/11/main/*

# 执行恢复
sudo -u postgres pgbackrest --stanza=main restore

# 启动 PostgreSQL
sudo systemctl start postgresql
```
#### 5.2 增量还原
执行以下命令进行增量还原：
```bash
sudo -u postgres pgbackrest --stanza=main --delta restore
```

### 6. 查看备份信息
执行以下命令查看备份信息：
```bash
sudo -u postgres pgbackrest --stanza=main info
```