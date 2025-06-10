## 将tomcat或jar包创建为系统服务
### 1. 创建systemd服务文件
```bash
sudo vim /etc/systemd/system/xxx.service
```

### 2. 写入以下内容
```ini
[Unit]
Description=xxx
After=network.target

[Service]
# 使用启动命令的用户
User=root
Group=root

# 工作目录
WorkingDirectory=/usr/local/xxx

# 启动命令
ExecStart=/usr/bin/java -jar xxx.jar
#ExecStart=/usr/local/tomcat/bin/startup.sh
# 关闭命令
#ExecStop=/usr/local/tomcat/bin/shutdown.sh

# 服务类型
Type=simple

# 重启策略
Restart=always
RestartSec=10

# 日志输出
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=xxx

[Install]
WantedBy=multi-user.target
```

### 3. 启动服务/设置开机自启
```bash
# 1. 重载配置
sudo systemctl daemon-reload

# 2. 开机自启
sudo systemctl enable xxx

# 3. 启动服务
sudo systemctl start xxx

# 4. 检查状态
sudo systemctl status xxx
journalctl -u xxx -b --no-pager
```

### 4. 查看日志
```bash
journalctl -t xxx -f
```
其中xxx为SyslogIdentifier