---
title: RDS MySQL 物理备份文件恢复到自建数据库
date: 2020-04-27 15:40:24
tags:
- Aliyun
- Mysql
- Docker
categories: Linux
---

![20200428182017](https://img.yjll.art/img/20200428182017.png)

阿里云提供了定期备份数据库的功能，可以在阿里云的控制台将数据库恢复到备份点，那我们想将这部分数据导入到我们自建的数据库怎么操作呢？官方推荐使用开源软件Percona Xtrabackup对数据库进行恢复，Percona Xtrabackup目前没有Windows版本，但是我们可以使用Docker进行恢复。

<!-- more -->

### 准备工作

在阿里云的后台下载备份文件

![bak](https://img.yjll.art/img/bak.png)

备份前肯定要先安装mysql的，生产环境的数据库使用的Mysql 8.0，我直接通过docker安装。

    docker pull mysql:8.0

`/etc/my.cnf` 是mysql的配置文件，可以手动创建一个

    docker run -p 3308:3306 --name mysql-bak -v $PWD/var/lib/mysql:/var/lib/mysql -v $PWD/etc/my.cnf:/etc/my.cnf -v $PWD/var/log:/var/log  -e MYSQL_ROOT_PASSWORD=123456 -d mysql:8.0


将之前下载的备份文件拷贝到容器中

    docker cp /bak/hins12384993_data_20200427051200_qp.xb mysql-bak:/root/

启动成功后我们进入容器进行恢复

    docker exec -it mysql-bak /bin/bash

安装我们的主角percona-xtrabackup用户恢复文件，直接下载二进制包安装的话需要手动解决依赖问题，我添加官方的源进行安装

```

apt -y install lsb-release

wget https://repo.percona.com/apt/percona-release_latest.$(lsb_release -sc)_all.deb

dpkg -i percona-release_latest.$(lsb_release -sc)_all.deb

apt-get update

apt-get install percona-xtrabackup-80

```


创建解压目录，官方文档中使用的这个路径，我懒得改了，就创建了一个

    mkdir -p /home/mysql/data

开始解压

```
    先安装qpress解压软件
    aria2c http://www.quicklz.com/qpress-11-linux-x64.tar
    tar xvf qpress-11-linux-x64.tar
    chmod 775 qpress
    cp qpress /usr/bin

    cat hins12384993_data_20200427051200_qp.xb | xbstream -x -v -C /home/mysql/data

    xtrabackup --decompress --remove-original --target-dir=/home/mysql/data

```


解压后进行数据恢复
```
xtrabackup --prepare --target-dir=/home/mysql/data
xtrabackup --datadir=/var/lib/mysql --copy-back --target-dir=/home/mysql/data

```

自建数据库不支持如下参数，需要注释掉。

vi /home/mysql/data/backup-my.cnf

```
#innodb_log_checksum_algorithm
#innodb_fast_checksum
#innodb_log_block_size
#innodb_doublewrite_file
#rds_encrypt_data
#innodb_encrypt_algorithm
#redo_log_version
#master_key_id
#server_uuid

```

重启

```
chown -R mysql:mysql /home/mysql/data

mysqld_safe --defaults-file=/home/mysql/data/backup-my.cnf --user=mysql --datadir=/home/mysql/data &

控制台打印，如果mysql服务没起来，可以查看/home/mysql/data/14864923d3cb.err这个日志文件
2020-04-28T09:43:49.658819Z mysqld_safe Logging to '/home/mysql/data/14864923d3cb.err'.
2020-04-28T09:43:49.703474Z mysqld_safe Starting mysqld daemon with databases from /home/mysql/data

```

### 参考资料
https://help.aliyun.com/knowledge_detail/41817.html