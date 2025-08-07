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