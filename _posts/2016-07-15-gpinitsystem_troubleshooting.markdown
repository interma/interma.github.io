---
layout: post
title:  "gpinitsystem troubleshooting"
author: 姚延栋
date:   2016-07-15 09:49
categories: gpdb gpinitsystem
published: true
---

有时候 gpinitsystem 会失败，但是不清楚失败原因是什么。 下面提供一些思路来 RCA：

* 查看 ~/gpAdmin/gpinitsystem_*** 日志文件
* 进入 master 的日志目录 （例如 /data/master/gpseg-1/pg_log/) 查看日志。 这里面有2种类型的日志：
  * startup.log
  * gpdb-<date>.csv


## 初始化 master 数据库失败

手动执行查看错误信息：

    $ initdb -E UNICODE -D /data/master/gpseg-1 --locale=en_US.utf8 --max_connections=250 \
     --shared_buffers=128000kB --is_filerep_mirrored=no --backend_output=/data/master/gpseg-1.initdb


## 如果master 起不来

手动启动master观看日志是否有问题：


Utility mode 启动 master，仅仅允许utility 模式连接。

     $ postgres -D /data/master/gpseg-1 -i -p 5432 -c gp_role=utility -M master -b 1 -C -1 -z 0 -m

## 创建segment

     /home/gpadmin/build/gpdb.master/bin/lib/gpcreateseg.sh 40584 1 sdw13~40006~/data2/primary/gpseg18~20~18~0 IS_PRIMARY no 13 /home/gpadmin/gpAdminLogs/gpinitsystem_20160717.log ::1~10.153.101.115~192.168.101.115~fe80::92e2:baff:feb1:7904~fe80::ce46:d6ff:fe58:e10c


     cmd='export LD_LIBRARY_PATH=/home/gpadmin/build/gpdb.master/lib:/lib:;/home/gpadmin/build/gpdb.master/bin/initdb  -E UNICODE -D /data2/primary/gpseg18 --locale=en_US.utf8        --max_connections=750 --shared_buffers=128000kB --is_filerep_mirrored=no --backend_output=/data2/primary/gpseg18.initdb'

     /bin/ssh sdw13 export 'LD_LIBRARY_PATH=/home/gpadmin/build/gpdb.master/lib:/lib:;/home/gpadmin/build/gpdb.master/bin/initdb' -E UNICODE -D /data2/primary/gpseg18 --locale=en_US.utf8 --max_connections=750 --shared_buffers=128000kB --is_filerep_mirrored=no --backend_output=/data2/primary/gpseg18.initdb


## 启动 segment

     export LD_LIBRARY_PATH=/home/gpadmin/build/gpdb.master/lib:/lib:;export PGPORT=40006; /home/gpadmin/build/gpdb.master/bin/pg_ctl -w -l /data2/primary/gpseg18/pg_log/startup.log -D /data2/primary/gpseg18 -o "-i -p 40006 -M mirrorless -b 20 -C 18 -z 0" start

## gp_bash_functions.sh 函数出错

     /home/gpadmin/build/gpdb.master/bin/lib/gp_bash_functions.sh: line 493: [: -gt: unary operator expected

## 问题

一旦出错，不知道原因是什么。非常难于 trouble shooting。 很多错误直接重定向到 /dev/null 了。

## tricks

## set -x in bin/lib/gp_bash_functions.sh

