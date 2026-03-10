---
title: Ubuntu安装RocketMQ v5.3.2和docker安装rocketmq-dashboard
author: setKing
date: 2026-03-07 11:33:00 +0800
categories: [教程, 文档]
tags: [学习, 文档]
pin: 教程
math: true
mermaid: true
---

本教程系统是Ubuntu 24.04，RocketMQ版本：5.3.2。RocketMQ安装在宿主机上rocketmq-dashboard安装在docker里

RocketMQ 需要 Java：

```
sudo apt update
sudo apt install -y openjdk-17-jdk

# 验证
java -version
```

下载和移动rocketMQ二进制文件：

```
cd ~
wget https://archive.apache.org/dist/rocketmq/5.3.2/rocketmq-all-5.3.2-bin-release.zip
unzip rocketmq-all-5.3.2-bin-release.zip
sudo rm -rf /usr/local/rocketmq
sudo mv rocketmq-all-5.3.2-bin-release /usr/local/rocketmq
ls /usr/local/rocketmq/bin/
```

设置自启动和后台启动

配置 systemd 自启动

> 我的rocket安装目录：/usr/rocketmq/

1. namesrv 服务

   ```
   sudo nano /etc/systemd/system/rocketmq-namesrv.service
   ```

   添加

   ```
   [Unit]
   Description=Apache RocketMQ NameServer
   After=network.target

   [Service]
   Type=simple
   Environment=JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
   ExecStart=/usr/rocketmq/bin/mqnamesrv
   ExecStop=/usr/rocketmq/bin/mqshutdown namesrv
   Restart=on-failure
   RestartSec=5s
   LimitNOFILE=65536

   [Install]
   WantedBy=multi-user.target
   ```

2. 配置 Broker

   ```
   sudo nano /usr/rocketmq/conf/broker.conf
   ```

   添加或修改以下内容

   ```

   brokerClusterName = DefaultCluster
   brokerName = broker-a
   brokerId = 0
   namesrvAddr = 127.0.0.1:9876
   defaultTopicQueueNums = 4
   autoCreateTopicEnable = true
   autoCreateSubscriptionGroup = true
   listenPort = 10911
   deleteWhen = 04
   fileReservedTime = 48
   brokerRole = ASYNC_MASTER
   flushDiskType = ASYNC_FLUSH
   # 存储路径（可自定义）
   storePathRootDir = /opt/rocketmq/store
   storePathCommitLog = /opt/rocketmq/store/commitlog
   ```

3. broker 服务

   ```
   sudo nano /etc/systemd/system/rocketmq-broker.service
   ```

   添加

   ```
   [Unit]
   Description=Apache RocketMQ Broker
   After=network.target rocketmq-namesrv.service
   Requires=rocketmq-namesrv.service

   [Service]
   Type=simple
   Environment=JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
   ExecStart=/opt/rocketmq/bin/mqbroker -c /opt/rocketmq/conf/broker.conf
   ExecStop=/opt/rocketmq/bin/mqshutdown broker
   Restart=on-failure
   RestartSec=5s
   LimitNOFILE=65536

   [Install]
   WantedBy=multi-user.target
   ```

4. 自启动

   ```
   # 重载 systemd
   sudo systemctl daemon-reload

   # 启动 NameServer
   sudo systemctl enable --now rocketmq-namesrv
   sudo systemctl status rocketmq-namesrv

   # 启动 Broker
   sudo systemctl enable --now rocketmq-broker
   sudo systemctl status rocketmq-broker
   ```

5. 确认状态

   ```
   sudo systemctl status rocketmq-namesrv
   sudo systemctl status rocketmq-broker
   ```

6. 调整 JVM 内存（重要！）

   ```
   // broker
   sed -i 's/-Xms8g -Xmx8g/-Xms1g -Xmx1g/g' /usr/local/rocketmq/bin/runbroker.sh
   sed -i 's/-XX:MaxDirectMemorySize=15g/-XX:MaxDirectMemorySize=1g/g' /usr/local/rocketmq/bin/runbroker.sh
   // namesrv
   sed -i 's/-Xms4g -Xmx4g -Xmn2g/-Xms512m -Xmx512m -Xmn256m/g' /usr/local/rocketmq/bin/runserver.sh
   ```

   验证：

   ```
   grep "Xms" /usr/local/rocketmq/bin/runbroker.sh
   grep "Xms" /usr/local/rocketmq/bin/runserver.sh
   ```

   重启：

   ```
   sudo systemctl restart rocketmq-namesrv
   sudo systemctl restart rocketmq-broker
   free -h
   ```

使用docker安装dashboard

```
docker run -d \
  --name rocketmq-dashboard \
  -e "JAVA_OPTS=-Drocketmq.namesrv.addr=host.docker.internal:9876" \
  -p 8080:8082 \
  --add-host=host.docker.internal:host-gateway \
  apacherocketmq/rocketmq-dashboard:latest
```

1.  确认容器正常运行：

```
docker ps
docker logs rocketmq-dashboard
```

2.  确认端口监听：

```
ss -tlnp | grep 8080
```

3.  确认防火墙：

```
# 查看防火墙状态
ufw status

# 如果防火墙开着，放行8080
ufw allow 8080
```
