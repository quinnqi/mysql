
### 主从同步是分布式mysql数据库相当重要的配置，现在我在虚拟机上完成主从配置，mysql版本是5.7.13

> 主服务器的ip是192.168.19.139 副服务器的ip是192.168.19.142

#### 1.主服务器配置
(1)修改my.cnf(注意使用root)
```
vim /etc/my.cnf
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLES
log-bin=mysqlbin-log
server_id=1
basedir=/home/mysql
user=mysql
datadir=/home/mysql/data
port=3306
innodb_flush_log_at_trx_commit=1
sync_binlog=1
log-slave-updates
```
(2)进入mysql安装目录下的bin，启动mysql
```
cd /home/mysql/bin 2 ./mysqld
```
(3)重开启mysql客户端，进入mysql安装目录下的bin，授权副服务器主从复制，用mysqld的好处是可以查看错误。
```
./mysql -uroot -p 2 mysql> grant replication slave on *.* to test@192.168.19.142 identified by '123';
```
(4)显示主服务器状态
```
mysql> show master status;
+---------------------+----------+--------------+------------------+-------------------+
| File                | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+---------------------+----------+--------------+------------------+-------------------+
| mysqlbin-log.000019 |    1290 |              |                  |                  |
+---------------------+----------+--------------+------------------+-------------------+
```
#### 2.副服务器配置
(1)修改my.cnf(注意使用root)
```
vim /etc/my.cnf
sql_mode=NO_ENGINE_SUBSTITUTION,STRICT_TRANS_TABLE
user=mysql
server_id = 2
replicate-do-db = sbfxd
master-info-file = master.info
relay-log = relay-relay-bin
relay-log-index = relay-relay-bin.index
relay-log-info-file=relay-relay-log.info
```
(2)进入mysql安装目录下的bin，启动mysql
```
cd /home/mysql/bin 2 ./mysqld
```
(3)重开启mysql客户端，进入mysql安装目录下的bin，设置slave属性并开始服务
```
./mysql -uroot -p
mysql> change master to master_host='192.168.19.139',master_port= 3306, master_log_file='mysqlbin-log.000019', master_log_pos= 1290, master_bind='', master_user='test',master_password='123';
mysql> start slave;
```
(4)显示副服务器的状态
```
show slave status \G;

            Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.19.139
                  Master_User: test
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysqlbin-log.000019
          Read_Master_Log_Pos: 1290
              Relay_Log_File: relay-relay-bin.000002
                Relay_Log_Pos: 323
        Relay_Master_Log_File: mysqlbin-log.000019
            Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
              Replicate_Do_DB: sbfxd
          Replicate_Ignore_DB:
          Replicate_Do_Table:
      Replicate_Ignore_Table:
      Replicate_Wild_Do_Table:
  Replicate_Wild_Ignore_Table:
                  Last_Errno: 0
                  Last_Error:
                Skip_Counter: 0
          Exec_Master_Log_Pos: 1290
              Relay_Log_Space: 530
              Until_Condition: None
              Until_Log_File:
                Until_Log_Pos: 0
          Master_SSL_Allowed: No
          Master_SSL_CA_File:
          Master_SSL_CA_Path:
              Master_SSL_Cert:
            Master_SSL_Cipher:
              Master_SSL_Key:
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error:
              Last_SQL_Errno: 0
              Last_SQL_Error:
  Replicate_Ignore_Server_Ids:
            Master_Server_Id: 1
                  Master_UUID: 05408021-5ab2-11e6-869b-000c293990c1
            Master_Info_File: /home/mysql/data/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
          Master_Retry_Count: 86400
                  Master_Bind:
      Last_IO_Error_Timestamp:
    Last_SQL_Error_Timestamp:
              Master_SSL_Crl:
          Master_SSL_Crlpath:
          Retrieved_Gtid_Set:
            Executed_Gtid_Set:
                Auto_Position: 0
        Replicate_Rewrite_DB:
                Channel_Name:
          Master_TLS_Version:
```
出现两个yes表示slave可以跑了

#### 3.注意事项：
(1)楼主的虚拟机是通过克隆产生的，所以不需要对mysql进行备份，然后在副服务器上恢复，这步对于数据库极其重要，否则就会出现如下错误:
2016-08-17T03:11:27.579193Z 5 [ERROR] Slave SQL for channel '': Error executing row event: 'Table 'sbfxd.sbtable' doesn't exist', Error_code: 1146
2 2016-08-17T03:11:27.584268Z 5 [Warning] Slave: Table 'sbfxd.sbtable' doesn't exist Error_code: 1146

(2)如果将主服务器的属性配错了需要stop slave，然后设置好之后start slave

#### dba要求配置如下：
```
max_connections=<具体数值由系统需求而定>
Port=3309
sql_mode=STRICT_TRANS_TABLES,NO_AUTO_CREATE_USER
Local_infile=0
open_files_limit=65535
long_query_time = 0.5
key_buffer_size=32M
innodb_flush_log_at_trx_commit=1
query_cache_type=0
performance_schema=ON
innodb_buffer_pool_size = 20G
innodb_log_file_size = 2000M

innodb_log_files_in_group=4
rpl_semi_sync_master_timeout = 60000
socket          = /alauda/data/mysql/mysql1.sock
bind_address = hostname
datadir  = /alauda/data/mysql
log-error=/alauda/data/mysql/log/mysqld1.log
pid-file=/alauda/data/mysql/bin/mysqld1.pid
innodb_data_home_dir =/alauda/data/mysql/data
innodb_log_group_home_dir =  /alauda/data/mysql/data/redo   
innodb_undo_directory=/alauda/data/mysql/data/undo
innodb_undo_tablespaces=4
server-id = 1
tmpdir=/alauda/data/mysql/data/tmp
port=3316
innodb_data_file_path = ibdata1:500M:autoextend
log-slave-updates=1
slave-skip-errors=1007,1008,1050,1060,1061,1062,1068
rpl_semi_sync_master_enabled = ON
rpl_semi_sync_master_timeout = 10000
rpl_semi_sync_slave_enabled = ON

```
