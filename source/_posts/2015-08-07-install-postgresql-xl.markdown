---
layout: post
title: "install postgresql xl"
date: 2015-08-07 16:01:10 +0800
comments: true
categories:  ['pgxl']
tags: pgxl
description: pgxl 的部署
toc: true
---

pgxl 是一个基于 pg 可扩展的 架构，安装部署在几台机器上

<!--more-->

postgresql 有兴趣的可以了解一下，也是一种关系型数据库。postgresql 配合 pg-xl 组件，可以达到类似 hadoop 一样的分布式效果。而且还有分布式事务的特性哦

---

## 环境准备

PS：本文的研究者为同事 立超 同学，本人仅在测试环境下，根据指导实践了一遍

centos6 机器多台，可选下面的依赖库

``` bash
yum -y install lrzsz sysstat e4fsprogs ntp readline-devel zlib zlib-devel openssl openssl-devel pam-devel libxml2-devel libxslt-devel python-devel tcl-devel gcc make smartmontools flex bison perl perl-devel perl-ExtUtils* OpenIPMI-tools openldap openldap-devel
```

---

## 安装步骤

### 下载pgxl，并编译安装

``` bash
wget http://jaist.dl.sourceforge.net/project/postgres-xl/Releases/Version_9.2rc/postgres-xl-v9.2-src.tar.gz
tar -zxvf postgres-xl-v9.2-src.tar.gz

#解决html.index Error 1的问题
cd postgres-xl/doc-xc/src/sgml/
rm -rf bookindex.sgmlin && cd ../../..

#编译安装
./configure --prefix=/home/pgxl/pgxl9.2 --with-pgport=11921 --with-perl --with-tcl --with-python --with-openssl --with-pam --with-ldap --with-libxml --with-libxslt --enable-thread-safety --with-blocksize=32

make && make install
```

### 添加环境，用户

``` bash
useradd pgxl
su - pgxl
vi .bashrc
export PGHOME=/home/pgxl/pgxl9.2
export LD_LIBRARY_PATH=$PGHOME/lib:/lib64:/usr/lib64:/usr/local/lib64:/lib:/usr/lib:/usr/local/lib:$LD_LIBRARY_PATH
export PATH=$PGHOME/bin:$PATH:.

```

### 角色介绍 coordinator 和 datanode

* coordinator 作为协调者 ，类似 hdfs Namenode
* datanode 作为存储者，类似 hdfs datanode
* GTM（global Transcation Manager)

在配置时 需要让 端口访问 不仅 datanode 需要互通 ，还要都能访问到 coordinator 互通

#### 配置coordinator

``` bash
initdb -D /data/pgxl/c11921/pg_root --nodename=c11921 -E UTF8 --locale=C -U postgres -W
cd /data/pgxl/c11921/pg_root

vi pg_hba.conf
host all all 0.0.0.0/0 md5

vi postgresql.conf
listen_addresses = '*'          # what IP address(es) to listen on;
port = 11921                            # (change requires restart)
max_connections = 500                   # (change requires restart)
superuser_reserved_connections = 13     # (change requires restart)
unix_socket_directory = '.'             # (change requires restart)
unix_socket_permissions = 0700          # begin with 0 to use octal notation
tcp_keepalives_idle = 60                # TCP_KEEPIDLE, in seconds;
tcp_keepalives_interval = 10            # TCP_KEEPINTVL, in seconds;
tcp_keepalives_count = 10               # TCP_KEEPCNT;
shared_buffers = 2048MB                 # min 128kB
max_prepared_transactions = 500         # zero disables the feature
vacuum_cost_delay = 10ms                # 0-100 milliseconds
vacuum_cost_limit = 10000               # 1-10000 credits
bgwriter_delay = 10ms                   # 10-10000ms between rounds
shared_queues = 64                      # min 16
shared_queue_size = 262144               # min 16KB
wal_level = hot_standby                 # minimal, archive, or hot_standby
synchronous_commit = off                # synchronization level;
wal_sync_method = fdatasync             # the default is the first option
wal_buffers = 16384kB                   # min 32kB, -1 sets based on shared_buffers
wal_writer_delay = 10ms         # 1-10000 milliseconds
checkpoint_segments = 128               # in logfile segments, min 1, 16MB each
archive_mode = on               # allows archiving to be done
archive_command = '/bin/date'           # command to use to archive a logfile segment
max_wal_senders = 32            # max number of walsender processes
hot_standby = on                        # "on" allows queries during recovery
max_standby_archive_delay = 300s        # max delay before canceling queries
max_standby_streaming_delay = 300s      # max delay before canceling queries
wal_receiver_status_interval = 1s       # send replies at least this often
hot_standby_feedback = on               # send info from standby to prevent
remote_query_cost = 100.0               # same scale as above
effective_cache_size = 96000MB
log_destination = 'csvlog'              # Valid values are combinations of
logging_collector = on          # Enable capturing of stderr and csvlog
log_directory = 'pg_log'                # directory where log files are written,
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log' # log file name pattern,
log_file_mode = 0600                    # creation mode for log files,
log_truncate_on_rotation = on           # If on, an existing log file with the
log_min_duration_statement = 1s # -1 is disabled, 0 logs all statements
log_checkpoints = on
log_connections = on
log_disconnections = on
log_error_verbosity = verbose           # terse, default, or verbose messages
log_lock_waits = on                     # log lock waits >= deadlock_timeout
log_statement = 'ddl'                   # none, ddl, mod, all
log_timezone = 'PRC'
autovacuum = on                 # Enable autovacuum subprocess?  'on'
log_autovacuum_min_duration = 0 # -1 disables, 0 logs all actions and
autovacuum_vacuum_cost_delay = 10ms     # default vacuum cost delay for
datestyle = 'iso, mdy'
timezone = 'PRC'
lc_messages = 'C'                       # locale for system error message
lc_monetary = 'C'                       # locale for monetary formatting
lc_numeric = 'C'                        # locale for number formatting
lc_time = 'C'                           # locale for time formatting
default_text_search_config = 'pg_catalog.english'
pooler_port = 21921                     # Pool Manager TCP port
max_pool_size = 100                     # Maximum pool size
pool_conn_keepalive = 60                # Close connections if they are idle
pool_maintenance_timeout = 30           # Launch maintenance routine if pooler
max_coordinators = 16                   # Maximum number of Coordinators
max_datanodes = 16                      # Maximum number of Datanodes
gtm_host = '127.0.0.1'                  # Host name or address of GTM
gtm_port = 11926                        # Port of GTM
pgxc_node_name = 'c11921'                       # Coordinator or Datanode name
sequence_range = 10000
```

#### 配置datanode

``` bash
initdb -D /data/pgxl/d11922/pg_root --nodename=d11922 -E UTF8 --locale=C -U postgres -W
cd /data/pgxl/d11922/pg_root

vi pg_hba.conf
local   all             all                                     trust
host    all             all             127.0.0.1/32            trust
host    all             all             ::1/128                 trust

vi postgresql.conf
listen_addresses = '*'          # what IP address(es) to listen on;
port = 11922                            # (change requires restart)
max_connections = 500                   # (change requires restart)
superuser_reserved_connections = 13     # (change requires restart)
unix_socket_directory = '.'             # (change requires restart)
unix_socket_permissions = 0700          # begin with 0 to use octal notation
tcp_keepalives_idle = 60                # TCP_KEEPIDLE, in seconds;
tcp_keepalives_interval = 10            # TCP_KEEPINTVL, in seconds;
tcp_keepalives_count = 10               # TCP_KEEPCNT;
shared_buffers = 2048MB                 # min 128kB
max_prepared_transactions = 500         # zero disables the feature
vacuum_cost_delay = 10ms                # 0-100 milliseconds
vacuum_cost_limit = 10000               # 1-10000 credits
bgwriter_delay = 10ms                   # 10-10000ms between rounds
shared_queues = 64                      # min 16
shared_queue_size = 262144               # min 16KB
wal_level = hot_standby                 # minimal, archive, or hot_standby
synchronous_commit = off                # synchronization level;
wal_sync_method = fdatasync             # the default is the first option
wal_buffers = 16384kB                   # min 32kB, -1 sets based on shared_buffers
wal_writer_delay = 10ms         # 1-10000 milliseconds
checkpoint_segments = 128               # in logfile segments, min 1, 16MB each
archive_mode = on               # allows archiving to be done
archive_command = 'cp %p /home/postgres/archive/%f'           # command to use to archive a logfile segment
max_wal_senders = 32            # max number of walsender processes
hot_standby = on                        # "on" allows queries during recovery
max_standby_archive_delay = 300s        # max delay before canceling queries
max_standby_streaming_delay = 300s      # max delay before canceling queries
wal_receiver_status_interval = 1s       # send replies at least this often
hot_standby_feedback = on               # send info from standby to prevent
remote_query_cost = 100.0               # same scale as above
effective_cache_size = 96000MB
log_destination = 'csvlog'              # Valid values are combinations of
logging_collector = on          # Enable capturing of stderr and csvlog
log_directory = 'pg_log'                # directory where log files are written,
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log' # log file name pattern,
log_file_mode = 0600                    # creation mode for log files,
log_truncate_on_rotation = on           # If on, an existing log file with the
log_min_duration_statement = 1s # -1 is disabled, 0 logs all statements
log_checkpoints = on
log_connections = on
log_disconnections = on
log_error_verbosity = verbose           # terse, default, or verbose messages
log_lock_waits = on                     # log lock waits >= deadlock_timeout
log_statement = 'ddl'                   # none, ddl, mod, all
log_timezone = 'PRC'
autovacuum = on                 # Enable autovacuum subprocess?  'on'
log_autovacuum_min_duration = 0 # -1 disables, 0 logs all actions and
autovacuum_vacuum_cost_delay = 10ms     # default vacuum cost delay for
datestyle = 'iso, mdy'
timezone = 'PRC'
lc_messages = 'C'                       # locale for system error message
lc_monetary = 'C'                       # locale for monetary formatting
lc_numeric = 'C'                        # locale for number formatting
lc_time = 'C'                           # locale for time formatting
default_text_search_config = 'pg_catalog.english'
pooler_port = 21925                     # Pool Manager TCP port
max_pool_size = 100                     # Maximum pool size
pool_conn_keepalive = 60                # Close connections if they are idle
pool_maintenance_timeout = 30           # Launch maintenance routine if pooler
max_coordinators = 16                   # Maximum number of Coordinators
max_datanodes = 16                      # Maximum number of Datanodes
gtm_host = '10.100.5.3'                  # Host name or address of GTM
gtm_port = 11926                        # Port of GTM
pgxc_node_name = 'd_41_11922'                       # Coordinator or Datanode name
```

### 启动datanode

``` bash
pg_ctl start -Z datanode -D /data/pgxl/d11922/pg_root
```

### 配置GTM

``` bash
vi /data/pgxl/g11920/gtm.conf
listen_addresses = '0.0.0.0'
log_file = 'gtm.log'
log_min_messages = WARNING
nodename = 'g11920'
port = 11920
startup = act
```

### 启动GTM

``` bash
gtm_ctl -Z gtm start -D /data/pgxl/g11920/
```

### 连接datanode

``` bash
psql -h 127.0.0.1 -p 11922 postgres postgres
```

### 添加节点

``` bash
create node d_41_11922 with (type=datanode, host='10.100.5.41', port=11922);
select pgxc_pool_reload();
```

测试环境类似

``` bash
create node c11921 with (type=coordinator, host='10.200.8.47', port=11921);
create node d_47_11922 with (type=datanode, host='10.200.8.47', port=11922);
create node d_48_11922 with (type=datanode, host='10.200.8.48', port=11922);
```

``` postgresql
postgres=# select * from pgxc_node;
 node_name  | node_type | node_port |  node_host  | nodeis_primary | nodeis_preferred |  node_id
------------+-----------+-----------+-------------+----------------+------------------+------------
 c11921     | C         |      5432 | localhost   | f              | f                | 1123528383
 c_47_11921 | C         |     11921 | 10.200.8.47 | f              | f                | -389791404
 d_47_11922 | D         |     11922 | 10.200.8.47 | f              | f                |  851696902
 d_48_11922 | D         |     11922 | 10.200.8.48 | f              | f                |  715971365

drop node c11921;
```


同理连接 coord
psql -h 127.0.0.1 -p 11921 postgres postgres
有时删不掉 默认的localhost coord，需要删除自己的，然后alter local 成自己的

pg_hba.conf中，要保证coordinator到各datanode、各datanode之间可以无密码登陆

## 后记: 
如果机器物理内存小于16G，则启动报错，可以增加swap空间[http://blog.slogra.com/post-246.html](http://blog.slogra.com/post-246.html)
