---
title: MySQL-Docker部署
tags: 
  - MySQL
date: 2025-06-11 15:27:21
categories:
  - MySQL
---

## 使用Docker部署MySQL

```shell
# 拉取mysql镜像
docker pull mysql:8.0

# 创建mysql容器
docker run --name mysql8 -p 3307:3306 -e MYSQL_ROOT_PASSWORD=root -d mysql:8.0

# 进入mysql容器
docker exec -it mysql8 /bin/bash

# 进入mysql客户端
mysql -u root -p

# 实现远程连接
CREATE USER 'remote_user'@'%' IDENTIFIED BY 'user_password';
GRANT ALL PRIVILEGES ON *.* TO 'remote_user'@'%';
FLUSH PRIVILEGES;

```

