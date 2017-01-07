---
title:  Replicating AWS RDS MySQL databases to external slaves
date: 2013-07-07
---
**Update:** Using [**an external slave with an RDS master** is now possible as well as **RDS as a slave with an external master**](http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/MySQL.Procedural.Importing.html "http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/MySQL.Procedural.Importing.html")  

Connecting external MySQL slaves to AWS RDS mysql instances is one of the most wanted features, for example to have migration strategies into and out of RDS or to support strange replication chains for legacy apps. Listening to binlog updates is also a [**great way to update search indexes or to invalidate caches**](https://github.com/noplay/python-mysql-replication).  
  
As of now it is possible to access binary logs from outside RDS with [**the release of MySQL 5.6 in RDS**](http://aws.typepad.com/aws/2013/07/mysql-56-support-for-amazon-rds.html). What amazon does not mention is the possibility to connect external slaves to RDS.  
  
Here is the proof of concept (details on how to set up a master/slave setup is not the focus here :-) )  
  
First, we create a new database in RDS somehow like this:

    
    soenke♥kellerautomat:~$ rds-create-db-instance soenketest --backup-retention-period 1 --db-name testing --db-security-groups soenketesting --db-instance-class db.m1.small --engine mysql --engine-version 5.6.12 --master-user-password testing123 --master-username root --allocated-storage 5 --region us-east-1 
    DBINSTANCE  soenketest  db.m1.small  mysql  5  root  creating  1  ****  n  5.6.12  general-public-license
          SECGROUP  soenketesting  active
          PARAMGRP  default.mysql5.6  in-sync
          OPTIONGROUP  default:mysql-5-6  in-sync  
    

So first lets check if binlogs are enabled on the newly created RDS database:

    
    master-mysql> show variables like 'log_bin';
    +---------------+-------+
    | Variable_name | Value |
    +---------------+-------+
    | log_bin       | ON    |
    +---------------+-------+
    1 row in set (0.12 sec)
    
    master-mysql> show master status;
    +----------------------------+----------+--------------+------------------+-------------------+
    | File                       | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
    +----------------------------+----------+--------------+------------------+-------------------+
    | mysql-bin-changelog.000060 |      120 |              |                  |                   |
    +----------------------------+----------+--------------+------------------+-------------------+
    1 row in set (0.12 sec)
    

Great! Lets have another check with the mysqlbinlog tool as stated in the [RDS docs.](http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_LogAccess.Concepts.MySQL.html)

But first we have to create a user on the RDS instance which will be used by the connecting slave.

    
    master-mysql> CREATE USER 'repl'@'%' IDENTIFIED BY 'slavepass';
    Query OK, 0 rows affected (0.13 sec)
    
    master-mysql> GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
    Query OK, 0 rows affected (0.12 sec)
    

Now lets have a look at the binlog:

    
    soenke♥kellerautomat:~$ mysqlbinlog -h soenketest.something.us-east-1.rds.amazonaws.com -u repl -pslavepass --read-from-remote-server -t mysql-bin-changelog.000060
    ...
    SET @@session.character_set_client=33,@@session.collation_connection=33,@@session.collation_server=8/*!*/;
    CREATE USER 'repl'@'%' IDENTIFIED BY PASSWORD '*809534247D21AC735802078139D8A854F45C31F3'
    /*!*/;
    # at 582
    #130706 20:12:02 server id 933302652  end_log_pos 705 CRC32 0xc2729566  Query   thread_id=66    exec_time=0     error_code=0
    SET TIMESTAMP=1373134322/*!*/;
    GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%'
    /*!*/;
    DELIMITER ;
    # End of log file
    ROLLBACK /* added by mysqlbinlog */;
    /*!50003 SET COMPLETION_TYPE=@OLD_COMPLETION_TYPE*/;
    /*!50530 SET @@SESSION.PSEUDO_SLAVE_MODE=0*/;
    

As we can see, even the grants have been written to the RDS binlog. Great! Now lets try to connect a real slave! Just set up a vanilla mysql server somewhere (local, vagrant, whatever) and assign a server-id to the slave. RDS uses some (apparently) random server-ids like 1517654908 or 933302652 so I currently don't know how to be sure there are no conflicts with external slaves. Might be one of the reasons AWS doesn't publish the fact that slave connects actually got possible.  
  
After setting the server-id and optionally a database to replicate:

    
    server-id       =  12345678
    replicate-do-db=soenketesting
    

lets restart the slave DB and try to connect it to the master:

    
    slave-mysql> change master to master_host='soenketest.something.us-east-1.rds.amazonaws.com', master_password='slavepass', master_user='repl', master_log_file='mysql-bin-changelog.000067', master_log_pos=0;
    Query OK, 0 rows affected, 2 warnings (0.07 sec)
    
    slave-mysql> start slave;
    Query OK, 0 rows affected (0.01 sec)
    

And BAM, it's replicating:

    
    slave-mysql> show slave status\G
    *************************** 1. row ***************************
                   Slave_IO_State: Waiting for master to send event
                      Master_Host: soenketest.something.us-east-1.rds.amazonaws.com
                      Master_User: repl
                      Master_Port: 3306
                    Connect_Retry: 60
                  Master_Log_File: mysql-bin-changelog.000068
              Read_Master_Log_Pos: 422
                   Relay_Log_File: mysqld-relay-bin.000004
                    Relay_Log_Pos: 595
            Relay_Master_Log_File: mysql-bin-changelog.000068
                 Slave_IO_Running: Yes
                Slave_SQL_Running: Yes
                  Replicate_Do_DB: soenketesting
              Replicate_Ignore_DB: 
               Replicate_Do_Table: 
           Replicate_Ignore_Table: 
          Replicate_Wild_Do_Table: 
      Replicate_Wild_Ignore_Table: 
                       Last_Errno: 0
                       Last_Error: 
                     Skip_Counter: 0
              Exec_Master_Log_Pos: 422
                  Relay_Log_Space: 826
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
                 Master_Server_Id: 933302652
                      Master_UUID: ec0eef96-a6e9-11e2-bdf0-0015174ecc8e
                 Master_Info_File: /var/lib/mysql/master.info
                        SQL_Delay: 0
              SQL_Remaining_Delay: NULL
          Slave_SQL_Running_State: Slave has read all relay log; waiting for the slave I/O thread to update it
               Master_Retry_Count: 86400
                      Master_Bind: 
          Last_IO_Error_Timestamp: 
         Last_SQL_Error_Timestamp: 
                   Master_SSL_Crl: 
               Master_SSL_Crlpath: 
               Retrieved_Gtid_Set: 
                Executed_Gtid_Set: 
                    Auto_Position: 0
    1 row in set (0.00 sec)
    

So lets issue some statements on the master:

    
    master-mysql> create database soenketesting;
    Query OK, 1 row affected (0.12 sec)
    master-mysql> use soenketesting
    Database changed
    master-mysql> create table example (id int, data varchar(100));
    Query OK, 0 rows affected (0.19 sec)
    

And it's getting replicated:

    
    slave-mysql> use soenketesting;
    Database changed
    slave-mysql> show create table example\G
    *************************** 1. row ***************************
           Table: example
    Create Table: CREATE TABLE `example` (
      `id` int(11) DEFAULT NULL,
      `data` varchar(100) DEFAULT NULL
    ) ENGINE=InnoDB DEFAULT CHARSET=latin1
    1 row in set (0.00 sec)
   