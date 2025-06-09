## 常用导出数据库的命令
pg_dump -U XXX --format=p --clean --create -d XXX > XXX.sql

其中：
- -U：指定数据库用户名
- --format：指定导出格式，p表示自定义格式
- --clean：指定导出时清理数据库
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