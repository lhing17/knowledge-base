# PostgreSQL数据库备份和还原

> 本文档介绍了使用pgbackrest进行PostgreSQL数据库备份和还原的两种方式：Docker环境和传统安装方式。推荐优先使用Docker环境，它提供了更好的隔离性和可移植性。

## 目录

- [一、Docker环境下的pgbackrest使用](#一docker环境下的pgbackrest使用)
- [二、传统安装方式](#二传统安装方式)

---

## 一、Docker环境下的pgbackrest使用

> 💡 **说明**：本章节介绍如何在Docker环境中使用pgbackrest进行PostgreSQL数据库的备份和还原。适合容器化部署的场景。
> 
> 📋 **章节导航**：
> - [1. 安装方式](#1-安装方式) - 两种Docker安装方法
> - [2. 部署配置](#2-部署配置) - Docker Compose和docker run两种部署方式
> - [3. 配置文件设置](#3-配置文件设置) - 详细的配置文件创建和挂载说明
> - [4. 备份和还原操作](#4-备份和还原操作) - 完整的备份还原流程
> - [5. 高级功能](#5-高级功能) - 数据卷备份、自动化脚本等
> - [6. 故障排除](#6-故障排除) - 常见问题解决方案

### 1. 安装方式

**方法一：使用包含pgbackrest的PostgreSQL镜像**

创建自定义Dockerfile：
```dockerfile
FROM postgres:15

# 安装pgbackrest
RUN apt-get update && \
    apt-get install -y wget gnupg2 && \
    wget --quiet -O - https://apt.postgresql.org/pub/repos/apt/ACCC4CF8.asc | apt-key add - && \
    echo "deb http://apt.postgresql.org/pub/repos/apt/ $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list && \
    apt-get update && \
    apt-get install -y pgbackrest && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# 创建pgbackrest配置目录
RUN mkdir -p /etc/pgbackrest && \
    mkdir -p /var/lib/pgbackrest && \
    chown postgres:postgres /var/lib/pgbackrest && \
    chmod 750 /var/lib/pgbackrest
```

构建镜像：
```bash
docker build -t postgres-pgbackrest:15 .
```

**方法二：在运行的容器中安装**

进入PostgreSQL容器：
```bash
docker exec -it postgres_container bash
```

在容器内安装pgbackrest：
```bash
apt-get update
apt-get install -y pgbackrest
mkdir -p /etc/pgbackrest /var/lib/pgbackrest
chown postgres:postgres /var/lib/pgbackrest
chmod 750 /var/lib/pgbackrest
```

#### 1.2 配置文件准备和位置说明

在开始部署之前，需要准备以下配置文件：

**目录结构建议：**
```text
/your/project/directory/
├── docker-compose.yml          # Docker Compose配置文件（可选）
├── pgbackrest.conf            # pgbackrest配置文件
├── postgresql.conf            # PostgreSQL配置文件（可选）
└── backup_scripts/            # 备份脚本目录
    └── backup.sh
```

**配置文件位置说明：**
- `pgbackrest.conf`：在宿主机上创建，通过卷挂载到容器的 `/etc/pgbackrest/pgbackrest.conf`
- `docker-compose.yml`：在项目根目录创建（如果使用Docker Compose）
- 备份脚本：可以放在单独的目录中便于管理

**部署方式选择：**

> 💡 **提示**：Docker Compose不是必须的，你可以根据需要选择以下任一方式：
> 
> - **Docker Compose**（推荐）：适合复杂配置和多容器管理
> - **docker run命令**：适合简单部署和快速测试
> - **混合方式**：根据需要灵活选择

### 2. 部署配置

> 💡 **提示**：完成安装准备后，需要选择合适的部署方式。根据你的需求选择以下两种部署方式之一：
> - **方式一**：使用Docker Compose（推荐，便于管理）
> - **方式二**：使用docker run命令（适合简单场景）
> 
> ⚠️ **前置条件**：确保已完成[1. 环境准备](#1-环境准备)中的环境准备工作。

#### 2.1 方式一：Docker Compose配置

创建docker-compose.yml文件：
```yaml
version: '3.8'

services:
  postgres:
    image: postgres:15-alpine
    container_name: postgres_db
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./pgbackrest.conf:/etc/pgbackrest/pgbackrest.conf
      - backup_data:/var/lib/pgbackrest
    ports:
      - "5432:5432"
    networks:
      - postgres_network

  pgbackrest:
    image: pgbackrest/pgbackrest:latest
    container_name: pgbackrest_service
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./pgbackrest.conf:/etc/pgbackrest/pgbackrest.conf
      - backup_data:/var/lib/pgbackrest
    depends_on:
      - postgres
    networks:
      - postgres_network

volumes:
  postgres_data:
  backup_data:

networks:
  postgres_network:
    driver: bridge
```

#### 2.2 方式二：使用docker run命令

如果你不想使用Docker Compose，可以直接使用docker run命令：

**步骤1：创建数据卷**

```bash
# 创建数据卷
docker volume create postgres_data
docker volume create pgbackrest_repo
```

**步骤2：启动PostgreSQL容器**

```bash
# 启动PostgreSQL容器
docker run -d \
  --name postgres_pgbackrest \
  -e POSTGRES_DB=mydb \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=password \
  -p 5432:5432 \
  -v postgres_data:/var/lib/postgresql/data \
  -v $(pwd)/pgbackrest.conf:/etc/pgbackrest/pgbackrest.conf:ro \
  -v pgbackrest_repo:/var/lib/pgbackrest \
  postgres:15-alpine
```

**步骤3：使用官方PostgreSQL镜像的简化版本**

如果你想使用官方PostgreSQL镜像并在运行时安装pgbackrest：

```bash
# 启动PostgreSQL容器
docker run -d \
  --name postgres_container \
  -e POSTGRES_DB=mydb \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=password \
  -p 5432:5432 \
  -v postgres_data:/var/lib/postgresql/data \
  -v $(pwd)/pgbackrest.conf:/etc/pgbackrest/pgbackrest.conf:ro \
  -v pgbackrest_repo:/var/lib/pgbackrest \
  postgres:15-alpine

# 进入容器安装pgbackrest
docker exec -it postgres_container bash
apt-get update && apt-get install -y pgbackrest
mkdir -p /var/lib/pgbackrest /var/log/pgbackrest
chown postgres:postgres /var/lib/pgbackrest /var/log/pgbackrest
chmod 750 /var/lib/pgbackrest
```

**配置文件挂载说明：**

> ⚠️ **重要提示**：
> - `$(pwd)/pgbackrest.conf` 表示当前目录下的pgbackrest.conf文件
> - 确保pgbackrest.conf文件在运行docker命令的目录中
> - 如果配置文件在其他位置，请使用绝对路径，例如：`/path/to/your/pgbackrest.conf`

**权限设置：**
```bash
# 确保配置文件权限正确
chmod 644 pgbackrest.conf
```

### 3. 配置文件设置

> 🔗 **接续说明**：完成Docker容器部署后，需要创建和配置pgbackrest配置文件。本节将详细介绍配置文件的创建、内容设置和挂载方式。

#### 3.1 配置文件创建步骤

**步骤1：创建项目目录**
```bash
# 创建项目目录
mkdir -p ~/postgres-pgbackrest
cd ~/postgres-pgbackrest

# 创建备份脚本目录
mkdir -p backup_scripts
```

**步骤2：创建pgbackrest.conf配置文件**

在项目目录中创建pgbackrest.conf文件：
```bash
# 创建配置文件
touch pgbackrest.conf
```

**步骤3：编辑配置文件内容**

将以下内容写入pgbackrest.conf：
```ini
[global]
# 日志配置
log-level-console=info
log-level-file=debug
log-path=/var/log/pgbackrest

# 仓库配置
repo1-path=/var/lib/pgbackrest
repo1-retention-full=2
repo1-retention-diff=7
repo1-retention-incr=7

# 加密配置（可选）
repo1-cipher-type=aes-256-cbc
repo1-cipher-pass=your-encryption-password

# 压缩配置
repo1-bundle=y
repo1-block=y
compress-type=gz
compress-level=6

# 性能优化
process-max=4
start-fast=y

[main]
# PostgreSQL配置
pg1-path=/var/lib/postgresql/data
pg1-port=5432
pg1-socket-path=/var/run/postgresql

[global:archive-push]
compress-level=3
```

#### 3.2 配置文件挂载方式详解

**挂载方式对比：**

| 挂载方式 | 语法示例 | 适用场景 | 优缺点 |
|----------|----------|----------|--------|
| 相对路径 | `./pgbackrest.conf:/etc/pgbackrest/pgbackrest.conf:ro` | 配置文件在当前目录 | 简单，但依赖工作目录 |
| 绝对路径 | `/home/user/config/pgbackrest.conf:/etc/pgbackrest/pgbackrest.conf:ro` | 配置文件在固定位置 | 明确，不依赖工作目录 |
| 环境变量 | `${CONFIG_DIR}/pgbackrest.conf:/etc/pgbackrest/pgbackrest.conf:ro` | 动态配置路径 | 灵活，适合脚本化部署 |

**权限说明：**
- `:ro` 表示只读挂载，容器内无法修改配置文件
- `:rw` 表示读写挂载（默认），容器内可以修改配置文件
- 建议配置文件使用只读挂载以防止意外修改

**实际示例：**

```bash
# 方式1：使用相对路径（推荐用于开发环境）
cd ~/postgres-pgbackrest
docker run -d \
  --name postgres_pgbackrest \
  -v ./pgbackrest.conf:/etc/pgbackrest/pgbackrest.conf:ro \
  postgres-pgbackrest:15

# 方式2：使用绝对路径（推荐用于生产环境）
docker run -d \
  --name postgres_pgbackrest \
  -v /home/user/postgres-pgbackrest/pgbackrest.conf:/etc/pgbackrest/pgbackrest.conf:ro \
  postgres-pgbackrest:15

# 方式3：使用环境变量
export CONFIG_DIR=/home/user/postgres-pgbackrest
docker run -d \
  --name postgres_pgbackrest \
  -v ${CONFIG_DIR}/pgbackrest.conf:/etc/pgbackrest/pgbackrest.conf:ro \
  postgres-pgbackrest:15
```

#### 3.3 配置文件验证

**验证配置文件是否正确挂载：**
```bash
# 检查配置文件是否存在
docker exec postgres_pgbackrest ls -la /etc/pgbackrest/

# 查看配置文件内容
docker exec postgres_pgbackrest cat /etc/pgbackrest/pgbackrest.conf

# 验证配置文件语法
docker exec -u postgres postgres_pgbackrest pgbackrest --stanza=main check
```

**常见挂载问题排查：**
```bash
# 问题1：配置文件不存在
# 解决：检查宿主机文件路径是否正确
ls -la ~/postgres-pgbackrest/pgbackrest.conf

# 问题2：权限问题
# 解决：修改文件权限
chmod 644 ~/postgres-pgbackrest/pgbackrest.conf

# 问题3：路径错误
# 解决：使用绝对路径或确认当前工作目录
pwd
realpath ./pgbackrest.conf
```

### 4. 备份和还原操作

> 🔗 **接续说明**：配置文件设置完成后，就可以开始执行备份和还原操作了。本节将介绍完整的备份还原流程，包括初始化、各种备份类型和数据还原。

#### 4.1 初始化stanza

```bash
# 进入容器
docker exec -it postgres_pgbackrest bash

# 切换到postgres用户并创建stanza
su - postgres
pgbackrest --stanza=main stanza-create
pgbackrest --stanza=main check
```

#### 4.2 执行备份

**全量备份：**
```bash
docker exec -u postgres postgres_pgbackrest pgbackrest --stanza=main --type=full backup
```

**增量备份：**
```bash
docker exec -u postgres postgres_pgbackrest pgbackrest --stanza=main --type=incr backup
```

**差异备份：**
```bash
docker exec -u postgres postgres_pgbackrest pgbackrest --stanza=main --type=diff backup
```

> 📝 **备份类型说明**：
> - **全量备份**：备份所有数据，首次备份必须是全量备份
> - **增量备份**：只备份自上次备份以来的变化
> - **差异备份**：备份自上次全量备份以来的所有变化

#### 4.3 查看备份信息

```bash
docker exec -u postgres postgres_pgbackrest pgbackrest --stanza=main info
```

#### 4.4 数据还原

> ⚠️ **注意**：数据还原会覆盖现有数据，请确保已做好备份！

**停止容器并还原：**
```bash
# 停止容器
docker stop postgres_pgbackrest

# 启动容器进行还原
docker run --rm -it \
  -v postgres_data:/var/lib/postgresql/data \
  -v pgbackrest_repo:/var/lib/pgbackrest \
  -v ./pgbackrest.conf:/etc/pgbackrest/pgbackrest.conf:ro \
  postgres-pgbackrest:15 bash

# 在容器内执行还原
su - postgres
pgbackrest --stanza=main restore

# 退出并重新启动正常容器
exit
docker start postgres_pgbackrest
```

### 5. 高级功能

#### 5.1 数据卷备份策略

**备份Docker数据卷：**

```bash
# 备份PostgreSQL数据卷
docker run --rm -v postgres_data:/data -v $(pwd):/backup alpine \
  tar czf /backup/postgres_data_backup.tar.gz -C /data .

# 备份pgbackrest仓库卷
docker run --rm -v pgbackrest_repo:/data -v $(pwd):/backup alpine \
  tar czf /backup/pgbackrest_repo_backup.tar.gz -C /data .
```

**还原Docker数据卷：**

```bash
# 还原PostgreSQL数据卷
docker run --rm -v postgres_data:/data -v $(pwd):/backup alpine \
  tar xzf /backup/postgres_data_backup.tar.gz -C /data

# 还原pgbackrest仓库卷
docker run --rm -v pgbackrest_repo:/data -v $(pwd):/backup alpine \
  tar xzf /backup/pgbackrest_repo_backup.tar.gz -C /data
```

#### 5.2 自动化备份脚本

创建自动备份脚本backup.sh：
```bash
#!/bin/bash

CONTAINER_NAME="postgres_pgbackrest"
BACKUP_TYPE=${1:-incr}  # 默认增量备份
LOG_FILE="/var/log/pgbackrest_docker.log"

echo "$(date): Starting $BACKUP_TYPE backup" >> $LOG_FILE

# 检查容器是否运行
if ! docker ps | grep -q $CONTAINER_NAME; then
    echo "$(date): Container $CONTAINER_NAME is not running" >> $LOG_FILE
    exit 1
fi

# 执行备份
if docker exec -u postgres $CONTAINER_NAME pgbackrest --stanza=main --type=$BACKUP_TYPE backup; then
    echo "$(date): $BACKUP_TYPE backup completed successfully" >> $LOG_FILE
    
    # 显示备份信息
    docker exec -u postgres $CONTAINER_NAME pgbackrest --stanza=main info >> $LOG_FILE
else
    echo "$(date): $BACKUP_TYPE backup failed" >> $LOG_FILE
    exit 1
fi
```

设置定时任务：
```bash
# 编辑crontab
crontab -e

# 添加定时备份任务
# 每天凌晨2点执行增量备份
0 2 * * * /path/to/backup.sh incr

# 每周日凌晨1点执行全量备份
0 1 * * 0 /path/to/backup.sh full
```

#### 5.3 容器重启后的配置持久化

**确保配置文件持久化：**

在docker-compose.yml中正确映射配置文件：
```yaml
volumes:
  - ./pgbackrest.conf:/etc/pgbackrest/pgbackrest.conf:ro
  - ./postgresql.conf:/var/lib/postgresql/data/postgresql.conf
```

**PostgreSQL配置优化：**

在postgresql.conf中添加归档配置：
```conf
# 启用归档
archive_mode = on
archive_command = 'pgbackrest --stanza=main archive-push %p'

# WAL配置
wal_level = replica
max_wal_senders = 3
checkpoint_completion_target = 0.9
```

**健康检查脚本：**

在docker-compose.yml中添加健康检查：
```yaml
services:
  postgres:
    # ... 其他配置
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s
```

### 6. 故障排除

#### 6.1 常见问题

**权限问题：**
```bash
# 修复权限
docker exec -it postgres_pgbackrest bash
chown -R postgres:postgres /var/lib/pgbackrest
chmod 750 /var/lib/pgbackrest
```

**配置文件问题：**
```bash
# 检查配置文件语法
docker exec -u postgres postgres_pgbackrest pgbackrest --stanza=main check
```

**日志查看：**
```bash
# 查看pgbackrest日志
docker exec postgres_pgbackrest cat /var/log/pgbackrest/pgbackrest.log

# 查看PostgreSQL日志
docker logs postgres_pgbackrest
```

---

## 二、传统安装方式

> 💡 **说明**：传统安装方式适用于直接在服务器上安装PostgreSQL和pgbackrest的场景。
> 
> 📋 **章节导航**：
> - [1. 安装pgbackrest](#1-安装pgbackrest) - 软件安装
> - [2. 配置pgbackrest](#2-配置pgbackrest) - 配置文件设置
> - [3. 初始化仓库](#3-初始化仓库) - 备份仓库初始化
> - [4. 执行备份](#4-执行备份) - 各种备份类型
> - [5. 数据还原](#5-数据还原) - 完整和增量还原
> - [6. 查看备份信息](#6-查看备份信息) - 备份状态查询
> - [7. 数据还原优化配置](#7-数据还原优化配置) - 性能优化设置

### 1. 安装pgbackrest

请参考pgbackrest官方文档进行安装。

### 2. 配置pgbackrest

> 🔗 **接续说明**：安装完成后，需要配置pgbackrest。正确配置/etc/pgbackrest/pgbackrest.conf文件，示例如下：

```ini
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

### 3. 初始化仓库

> 🔗 **接续说明**：配置文件设置完成后，需要创建stanza（备份集）。stanza是pgbackrest的备份集，每个备份集可以配置不同的参数，例如备份路径、备份时间、备份保留策略等。

执行以下命令创建stanza：
```bash
sudo -u postgres pgbackrest --stanza=main stanza-create
sudo -u postgres pgbackrest --stanza=main check
```
### 4. 执行备份

> 🔗 **接续说明**：stanza创建完成后，就可以开始执行各种类型的备份操作了。

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

### 7. 数据还原优化配置

#### 7.1 session_replication_role 设置

在进行大量数据还原或迁移时，可以使用 `session_replication_role` 来优化性能和避免触发器干扰。

##### 作用说明
`SET session_replication_role = 'replica'` 主要用于控制触发器和约束的执行行为：

- **禁用用户定义的触发器**：避免业务逻辑触发器在数据还原时执行
- **禁用外键约束检查**：提高数据导入性能
- **禁用CHECK约束**：跳过数据验证约束
- **禁用NOT NULL约束**（部分情况）

##### 三种模式对比

| 模式 | 触发器执行 | 约束检查 | 使用场景 |
|------|------------|----------|----------|
| `'origin'` (默认) | 执行所有触发器 | 执行所有约束 | 正常业务操作 |
| `'replica'` | 只执行REPLICA触发器 | 跳过大部分约束 | 数据导入、备份还原 |
| `'local'` | 只执行LOCAL触发器 | 执行所有约束 | 特殊复制场景 |

##### 实际应用场景

**1. 配合pgbackrest还原使用**
```sql
-- 还原数据前设置
SET session_replication_role = 'replica';

-- 执行数据修复或补充操作
INSERT INTO table1 SELECT * FROM backup_table;
UPDATE table2 SET column1 = new_value WHERE condition;

-- 完成后恢复正常模式
SET session_replication_role = 'origin';
```

**2. 大批量数据迁移**
```sql
-- 提高数据迁移性能
SET session_replication_role = 'replica';
COPY large_table FROM '/path/to/data.csv' CSV;
SET session_replication_role = 'origin';
```

**3. 数据库还原后的数据修复**
```sql
-- 在事务中使用（推荐）
BEGIN;
SET session_replication_role = 'replica';
-- 执行数据修复操作
-- ...
SET session_replication_role = 'origin';
COMMIT;
```

##### 注意事项

⚠️ **重要警告**
1. **数据一致性风险**：禁用约束检查可能导致数据不一致
2. **业务逻辑跳过**：触发器中的业务逻辑不会执行
3. **会话级别**：只影响当前会话，不影响其他连接
4. **需要手动恢复**：操作完成后必须手动设置回 `'origin'`

##### 最佳实践
```sql
-- 推荐的使用模式
BEGIN;
SET session_replication_role = 'replica';
-- 执行数据操作
-- ...
SET session_replication_role = 'origin';
COMMIT;
```

##### 与pgbackrest的配合使用
在使用pgbackrest进行数据还原后，如果需要进行数据修复、清理或验证，可以：

1. **还原完成后的数据修复**
```bash
# 完成pgbackrest还原
sudo -u postgres pgbackrest --stanza=main restore
sudo systemctl start postgresql

# 连接数据库进行数据修复
psql -U postgres -d your_database -c "
BEGIN;
SET session_replication_role = 'replica';
-- 执行数据修复SQL
SET session_replication_role = 'origin';
COMMIT;
"
```

2. **数据验证和清理**
```sql
-- 在不触发业务逻辑的情况下进行数据操作
SET session_replication_role = 'replica';
DELETE FROM temp_table WHERE created_date < '2023-01-01';
SET session_replication_role = 'origin';
```

这个设置是PostgreSQL数据库管理中的强大工具，特别适用于备份还原场景，但需要谨慎使用并确保在合适的时机恢复到正常模式。

---

## 总结

本文档详细介绍了使用pgbackrest进行PostgreSQL数据库备份和还原的完整流程，包括：

### 🐳 Docker环境方式（推荐）
- **优势**：环境隔离、部署简单、配置统一
- **适用场景**：容器化部署、开发测试环境、云原生应用
- **核心步骤**：安装配置 → 部署容器 → 配置文件 → 备份还原

### 🖥️ 传统安装方式
- **优势**：性能更优、资源占用少、配置灵活
- **适用场景**：生产环境、物理服务器、高性能要求
- **核心步骤**：软件安装 → 配置文件 → 初始化 → 备份还原

### 📝 最佳实践建议
1. **备份策略**：定期全量备份 + 增量备份
2. **配置管理**：使用版本控制管理配置文件
3. **监控告警**：设置备份状态监控和失败告警
4. **恢复测试**：定期验证备份文件的可恢复性
5. **安全考虑**：启用加密、控制访问权限

### 🔧 故障排除
- 权限问题：确保postgres用户有足够权限
- 配置错误：检查配置文件语法和路径
- 网络问题：验证容器间网络连通性
- 存储空间：监控备份存储空间使用情况

选择合适的部署方式，遵循最佳实践，可以确保PostgreSQL数据库的安全可靠备份。