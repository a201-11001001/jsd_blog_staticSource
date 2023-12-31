---
title: mysqldump 在linux中的定时备份
date: 2023-08-25 13:52:21
tags: 
    - MySql
description: 使用mysqldump 实现在linux中的定时备份 ## 页面描述
keywords: mysqldump  ## 关键字
comments: true  ## 评论模块
cover: /top_img/mysqldump_top_img.jpeg  ## 文章缩略图
# top_img:  ## 顶部图片
aside: false  ## 显示侧边栏（默认 true）
---

#### Mysql 在Linux 中的备份
* 准备环境
```
mysql -V
mysql  Ver 14.14 Distrib 5.7.43, for Linux (x86_64) using  EditLine wrapper
```

* 配置MySql vim/etc/my.cof mysql 5.7之后需要配置文件中加入以下代码
```
[client]
host=127.0.0.1
user=root
password='jzkj@2023#$'
```

* 测试备份命令 生成sql 文件
> mysql dump-u用户名-p数据库密码--default-character-set=utf8要备份的数据库>/back/备份路径及文件名.sq1
```
出现警告不用管 
mysqldump: [Warning] Using a password on the command line interface can be insecure.
```

* 备份后的文件内容
```
-- MySQL dump 10.13  Distrib 5.7.43, for Linux (x86_64)
--
-- Host: 127.0.0.1    Database: jz_lxny
-- ------------------------------------------------------
-- Server version       5.7.43

/*!40101 SET @OLD_CHARACTER_SET_CLIENT=@@CHARACTER_SET_CLIENT */;
/*!40101 SET @OLD_CHARACTER_SET_RESULTS=@@CHARACTER_SET_RESULTS */;
/*!40101 SET @OLD_COLLATION_CONNECTION=@@COLLATION_CONNECTION */;
/*!40101 SET NAMES utf8 */;
/*!40103 SET @OLD_TIME_ZONE=@@TIME_ZONE */;
/*!40103 SET TIME_ZONE='+00:00' */;
/*!40014 SET @OLD_UNIQUE_CHECKS=@@UNIQUE_CHECKS, UNIQUE_CHECKS=0 */;
/*!40014 SET @OLD_FOREIGN_KEY_CHECKS=@@FOREIGN_KEY_CHECKS, FOREIGN_KEY_CHECKS=0 */;
/*!40101 SET @OLD_SQL_MODE=@@SQL_MODE, SQL_MODE='NO_AUTO_VALUE_ON_ZERO' */;
/*!40111 SET @OLD_SQL_NOTES=@@SQL_NOTES, SQL_NOTES=0 */;

--
-- Table structure for table `ces_order_customer`
-- 
```

* sql 备份恢复
```
## 1.登录 MySQL 

[root@VM-0-16-centos /]# mysql -u root -p
Enter password: 
Welcome to the MySQL monitor.  Commands end with ; or \g.
Your MySQL connection id is 24
Server version: 5.7.43 MySQL Community Server (GPL)

Copyright (c) 2000, 2023, Oracle and/or its affiliates.

Oracle is a registered trademark of Oracle Corporation and/or its
affiliates. Other names may be trademarks of their respective
owners.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

mysql> 
```

```
## 2.切换要备份的数据库
## 数据库已经存在情况下先切换 数据库 (第一次登录没有自定数据库默认是在 mysql 下)
## 使用 use 命令切换数据库

mysql> use back_test
Reading table information for completion of table and column names
You can turn off this feature to get a quicker startup with -A

Database changed
```

```
## 3.使用 source 命令 回复备份
source /back/要恢复的文件备份.sql
```

```
## 4.出现效果表示成功
mysql> source /back_lxny_test_001.sql
Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)

Query OK, 0 rows affected (0.00 sec)
```

* 在linx中使通过定时任务(crontab)白动备份my sq数据库
> 创建mysqldump 脚本

```
#!/bin/bash
  
#备份位置
backup_dir='/usr/data/backup/mysql/'
#获取当前时间
current_time=$(date +'%Y-%m-%d_%H%M%S')
#将backup_dir和时间组合起来，再加个后缀
filepath=$backup_dir$current_time'.sql.gz'

#数据库信息，配置数据库账号密码
s_ip='127.0.0.1'
username='root'
password='xxx'
source_database='xxx'

echo "-----------------------"$(date +%F%r)"操作开始--------------------------------"
echo $(date +%F%r)"开始进行数据备份..."
# 2>/dev/null 可以抑制警告信息
mysqldump -h${s_ip} -P3306 -u ${username} -p"${password}"  ${source_database} table1 table2 table3 | gzip > $filepath

echo $(date +%F%r)"完成同步..."
```

```
## 测试脚本 创建脚本可执行命令

chmod u+x  脚本名称.sh

## 运行脚本 并解压 (gunzip)
2924 -rw-r--r-- 1 root root 2990363 Aug 22 17:49 back_lxny_2023-08-22_174957.sql
 552 -rw-r--r-- 1 root root  563125 Aug 22 17:51 back_lxny_2023-08-22_175116.sql.gz
```

> 使用 crontab + mysqldump 创建定时备份任务
```
## 编辑当前用户定时任务 (没有crontab 可自行安装)

crontab -e

## 运行上述命令后，会打开一个文本编辑器，可以在其中输入要执行的命令和时间规则。格式如下：

* * * * * command

## 其中，五个星号分别表示分钟、小时、日、月、周几，command 表示要执行的命令或脚本文件路径。

crontab -e
*    *    *    *    *
-    -    -    -    -
|    |    |    |    |
|    |    |    |    +----------星期中星期几 (0 - 6) (星期天 为0)
|    |    |    +---------------月份 (1 - 12) 
|    |    +--------------------一个月中的第几天 (1 - 31)
|    +-------------------------小时 (0 - 23)
+------------------------------分钟 (0 - 59)添加定时任务(每天12:50以及23:50执行备份操作)
```

```
## 例如 

crontab -e
05 18 * * *  /usr/local/mysql/mysql_lxny_back.sh

[root@VM-0-16-centos back_lxny]# ll -s
total 552
552 -rw-r--r-- 1 root root 563126 Aug 22 18:05 back_lxny_2023-08-22_180501.sql.gz
```