---
title: "基于Docker快速搭建wordpress博客"
date: 2019-03-13T10:28:01+08:00
thumbnail: "img/wordpress.jpg"
description: "用docker快速搭建wordpress博客服务"
Tags: 
- Docker
- Mysql
- Wordpress
Categories:
- docker
---

wordpress数据存储依赖mysql数据库，以docker容器方式部署完整的wordpress博客服务，则需要从镜像仓库拉取mysql、wordpress镜像，这里选择从开源的docker hub 获取mysql 5.7版本，wordpress latest版本，同时需要准备一台具备外网环境的机器，硬件配置最好1C2G以上，当然1C1G也是没有问题的。

```Bash
docker pull mysql:5.7
docker pull wordpress:latest
```

### 数据库服务

启动mysql数据库服务

```Bash
docker run --name mysql-db -p 3306:3306 -v /data/mysql:/var/lib/mysql -d -e MYSQL_ROOT_PASSWORD=root mysql:5.7
```
mysql容器的名字为mysql-db，数据库文件则保存在当前机器的/data/mysql目录下，-e MYSQL_ROOT_PASSWORD以环境变量的方式设置mysql的数据库密码为root，用户名默认root，可以通过注入环境变量修改，更多参数设置请参考docker hub。命令执行成功之后，利用docker ps查询容器是否成功启动。

```Bash
CONTAINER ID        IMAGE                           COMMAND                  CREATED             STATUS              PORTS                                  NAMES
95808c37fea6        mysql:5.7                       "docker-entrypoint.s…"   2 minutes ago       Up 2 minutes        0.0.0.0:3306->3306/tcp, 33060/tcp      mysql-db
```
当然除了以数据库文件作为备份之外，还可以通过执行msyql命令备份sql脚本数据。

```Docker
docker exec mysql-db sh -c 'exec mysqldump --databases wordpress -uroot -p"$MYSQL_ROOT_PASSWORD"' > wordpress.sql
```

### Wordpress服务
数据库成功启动之后，再启动wordpress容器服务。

```Bash
docker run -v /data/wordpress/wp-content:/var/www/html/wp-content --name my-wordpress --link mysql-db:db -p 80:80 -d wordpress
```

在/data/wordpress/wp-content目录下修改文件内容，可以马上在服务中体现，非常方便，--link 表示关联了mysql-db容器，那么在my-wordpress容器中可以通过db直接访问mysql-db，可以进入my-wordpress容器查看--link增加了hosts记录。

```Bash
[root@izj6c data]# docker exec -it 46cc10552a12 sh
# cat /etc/hosts
127.0.0.1       localhost
::1     localhost ip6-localhost ip6-loopback
fe00::0 ip6-localnet
ff00::0 ip6-mcastprefix
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters
172.17.0.3      db 2ea59b2d6344 mysql-db
172.17.0.4      46cc10552a12
```
-p指定了容器的端口映射，当前wordpress的服务通过当前机器的80端口即可访问。

```Docker
CONTAINER ID        IMAGE                           COMMAND                  CREATED             STATUS              PORTS                               NAMES
46cc10552a12        wordpress                       "docker-entrypoint.s…"   4 seconds ago       Up 3 seconds        0.0.0.0:80->80/tcp 
```
### 配置Wordpress
现在通过浏览器直接访问 http://ip:80/ ,比如当前服务器的ip为192.168.3.8，那么直接访问 http://192.168.3.8:80/ 即可访问wordpress，然后跳转到管理员配置页面。

![language_select](/blog/docker_deploy_wordpress/002.png)

选择简体中文，然后下一步，填写数据库配置。

![database_select](/blog/docker_deploy_wordpress/001.png)

数据库主机直接写db，不再用写ip了，因为在启动wordpress容器的时候，我们使用了--link 参数，这就是docker --link方便。填写配置之后，提交，这时出现下面这个错误。

![error_tips](/blog/docker_deploy_wordpress/003.png)

这是因为我们还没有在mysql中创建数据库，进入容器中手动创建wordpress数据库。

```MySQL
[root@izj6c2 ~]# docker exec -it 2ea59b2d6344 sh
# mysql -uroot -p
Enter password:
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 7
Server version: 5.7.23 MySQL Community Server (GPL)
 
Copyright (c) 2000, 2018, Oracle and/or its affiliates. All rights reserved.
 
Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.
 
Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.
 
mysql> create database wordpress default character set utf8;
Query OK, 1 row affected (0.00 sec)
```

数据库创建成功之后，则在wordpress页面重新提交数据库配置，跟着下一步执行，即可创建完成博客系统了。

### 问题

> 外部无法连接mysql数据库

利用docker exec命令进入mysql容器内部，执行

```MySQL
USE mysql;
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '你的密码';
FLUSH PRIVILEGES;
```

> 目录无权限问题

因为这个wp-content目录是mount到容器内部的，即使在宿主机上`chmod 777 content/*`,可能还是无法上传图片或者文件，执行以下操作即可。
```Bash
docker exec -it containerId sh
chown -R www-data:www-data wp-content/*
```









