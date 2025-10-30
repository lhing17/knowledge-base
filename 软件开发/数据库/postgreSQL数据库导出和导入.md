## 常用导出数据库的命令
pg_dump -U XXX --format=p --create -d XXX > XXX.sql

其中：
- -U：指定数据库用户名
- --format：指定导出格式，p表示自定义格式
- --create：指定导出时包含创建数据库的命令
- -d：指定数据库名

## 常用导入数据库的命令
psql -d XXX -h localhost -p 5432 -U XXX -f path/to/file.sql

其中：
- -d：指定数据库名
- -h：指定主机名
- -p：指定端口号
- -U：指定用户名
- -f：指定要导入的SQL文件路径

## 从模板数据库复制数据库

如果数据库在同一个服务器上，我们可以使用模板数据库来复制数据库。

具体步骤如下：
1. 登录到PostgreSQL服务器上的psql命令行工具。
2. 执行以下命令来复制数据库：
```
CREATE DATABASE new_database WITH TEMPLATE old_database;
```
其中，new_database是新数据库的名称，old_database是旧数据库的名称。

如果old_database不是模板数据库，我们需要先使用以下命令将其设置为模板数据库：
```
ALTER DATABASE old_database WITH IS_TEMPLATE true;
```