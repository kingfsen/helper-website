---
title: "PostgreSQL命令行操作"
date: 2019-04-02T19:18:40+08:00
thumbnail: "img/postgresql.png"
description: "详细介绍用docker搭建一个PostgreSQL数据库实例以及PostgreSQL的基础命令操作"
Categories:
- docker
Tags:
- docker
- postgreSQL
---

PostgreSQL是一个功能强大的开源对象关系数据库管理系统(ORDBMS)，PostgreSQL(也称为Post-gress-Q-L)由PostgreSQL全球开发集团(全球志愿者团队)开发，
其源代码是免费提供的。但是在中国互联网的后端数据库中，PostgreSQL目前使用的比较少，大部分公司使用的应该是MySQL。PostgreSQL在国外项目中使用的比较广泛，
尤其是支持插件化的扩展功能，著名的时序数据库TimescaleDB就是在PostgreSQL基础的上进行插件开发的，可以随着PostgreSQL升级。

最近在工作中，需要在开源项目Harbor、Clair等项目的基础上做一些二次开发，这些项目都使用的是PostgreSQL。Harbor采用的开发框架beego虽然支持MySQL数据库，
但是Harbor从1.6版本开始弃用了MySQL，强制使用PostgreSQL，而coreos旗下的Clair等项目从始至终只用PostgreSQL，并且看其官方的意思是PostgreSQL某些方面的性能
优于MySQL，没有打算支持MySQL数据库，这里就不得不学习一下PostgreSQL了，从最基本的命令行开始操作。

这里以Docker容器启动一个PostgreSQL数据库实例服务。

```bash
docker run --name clair-db -p 9302:5432 -v /data/clair-db:/var/lib/postgresql/data -d -e POSTGRES_PASSWORD=password postgresql:latest
```

postgreSQL默认的端口是5432，容器启动之后将端口映射到宿主机的9302端口，数据持久化目录/data/clair-db，密码password。

```bash
CONTAINER ID        IMAGE                              COMMAND                  CREATED        STATUS        PORTS                
eb501bb45d21        postgresql:latest                 "/entrypoint.sh po..."   5 months ago   Up 2 months   0.0.0.0:9302->5432/tcp
```

Docker容器启动之后，检查PostgreSQL是否正常启动，正常启动访问其5432端口会返回empty reply。

```http
# curl http://localhost:9302/
curl: (52) Empty reply from server
```

进入PostgreSQL容器进行基本的命令行操作，先通过`su`切换到`postgres`用户，再输入`psql`。

```bash
# docker exec -it eb501bb45d21 sh
sh-4.3# su - postgres
No directory, logging in with HOME=/
postgres@eb501bb45d21 [ / ]$ psql
psql (9.6.10)
Type "help" for help.

postgres=# 
```

出现`postgres=#` ，则可以开始输入sql语句执行。

查询有哪些数据库

```mysql
postgres=# select datname from pg_database;
  datname  
-----------
 postgres
 template1
 template0
(3 rows)
```

切换到指定数据库使用 `\c`

```mysql
postgres=# \c postgres
You are now connected to database "postgres" as user "postgres".
```

查询数据库的public模式下有哪些表

```mysql
postgres=# select * from pg_tables where schemaname='public';
 schemaname |              tablename               | tableowner | tablespace | hasindexes | hasrules | hastriggers | rowsecurity 
------------+--------------------------------------+------------+------------+------------+----------+-------------+-------------
 public     | schema_migrations                    | postgres   |            | t          | f        | f           | f
 public     | layer                                | postgres   |            | t          | f        | t           | f
 public     | layer_diff_featureversion            | postgres   |            | t          | f        | t           | f
 public     | namespace                            | postgres   |            | t          | f        | t           | f
 public     | feature                              | postgres   |            | t          | f        | t           | f
 public     | featureversion                       | postgres   |            | t          | f        | t           | f
 public     | vulnerability_fixedin_feature        | postgres   |            | t          | f        | t           | f
 public     | vulnerability_affects_featureversion | postgres   |            | t          | f        | t           | f
 public     | keyvalue                             | postgres   |            | t          | f        | f           | f
 public     | lock                                 | postgres   |            | t          | f        | f           | f
 public     | vulnerability                        | postgres   |            | t          | f        | t           | f
 public     | vulnerability_notification           | postgres   |            | t          | f        | t           | f
(12 rows)
```

接下来的SQL则是标准的，与MySQL等并无区别。

退出psql命令行先输入`\q`，再输入`\exit`

```bash
postgres=# \q
could not save history to file "/home/postgres/.psql_history": No such file or directory
postgres@eb501bb45d21 [ / ]$ \exit
```
