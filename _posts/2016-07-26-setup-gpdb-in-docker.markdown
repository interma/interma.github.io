---
layout: post
title:  "A few tips on setup GPDB in Docker"
subtitle:  "一些在docker中配置GPDB数据库的小技巧"
author: 刘奎恩/Kuien, Peifeng Qiu
date:   2016-07-26 14:48 +0800
categories: docker 
published: true
---

### 0. compile gpdb source-code in docker 'gppkg'

```sh
git clone git@github.com:greenplum-db/gpdb4.git
git submodule update --init --recursive gpAux/extensions/pgbouncer/source
source /opt/gcc_env.sh
make sync_tools -C gpAux
make GPHOME=`pwd` BLD_TARGETS="gppkg" dist
```

### 1. setup docker hostname

__in host__:

```sh
docker stop gppkg
sudo service docker stop

vi /var/lib/docker/containers/CONTAINER_ID/config-xx.json
```

Replace ```"Hostname":"xxxx"``` with ```"Hostname":"new_hostname"```.

```sh
sudo service docker start
docker start gppkg
docker exec -u gpadmin -it gppkg /bin/bash
```

### 2. setup docker env

__in docker__:

```sh
sudo sed -ri 's/UsePAM yes/UsePAM no/g' /etc/ssh/sshd_config
sudo service sshd restart
    echo "service sshd start" >> /root/.bashrc
sudo chmod u+s /bin/ping
ssh-copy-id localhost
```

### 3. gpinitsystem

Now you can use the 'new_hostname' to setup gpdb.

Now, everything is READY!

```sql
psql -l
                  List of databases
   Name    |  Owner  | Encoding |  Access privileges
-----------+---------+----------+---------------------
 postgres  | gpadmin | UTF8     |
 template0 | gpadmin | UTF8     | =c/gpadmin
                                : gpadmin=CTc/gpadmin
 template1 | gpadmin | UTF8     | =c/gpadmin
                                : gpadmin=CTc/gpadmin
(3 rows)

[gpadmin@gppkg ~]$ psql postgres
psql (8.2.15)
Type "help" for help.

postgres=# select version();
                                                                                    version

-------------------------------------------------------------------------------------------------------------------------------
------------------------------------------------
 PostgreSQL 8.2.15 (Greenplum Database 4.3.9.0 build dev) on x86_64-unknown-linux-gnu, compiled by GCC gcc (GCC) 4.4.2 compiled
 on Jul 25 2016 02:31:25 (with assert checking)
(1 row)

postgres=# \q
```

### 4. gdb within docker

__in host__:

If you want to use gdb in Docker, you may run into a "ptrace: Operation not permitted" error. We found a workaround, which is to re-run the container with new flags:
```sh
docker run -t -v <absolute path to gpdb4 directory>:/home/gpadmin/gpdb4 --privileged --security-opt seccomp:unconfined -i pivotaldata/centos511-java7-gpdb-dev-image bash
```
See [this post](https://forums.docker.com/t/boot2docker-mac-os-x-1-10-failing-ptrace-gdb/6005). I have also updated the mounted docker guide. The simplest way to resolve is to destroy your gpdb4 container/image and redo the guide from the start, although you could commit your gpdb4 container to an image and use docker run on that...

    -- from Amil Khanzada <akhanzada@pivotal.io>