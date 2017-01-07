---
title:  External MySQL slaves with RDS reloaded
date: 2014-06-15
---
In [an earlier first post](/2013/07/07/replicating-aws-rds-mysql-databases-to-external-slaves/ "Replicating AWS RDS MySQL databases to external slaves") I demonstrated a way to connect an external slave to a running RDS instance. Later then AWS added the native possibility to import and export via replication.

In my case, several problems popped up:  
* My initial blog post did not show how to start from an existing data set, e. g. do a mysqldump, and import
* RDS does not allow --master-data mysqldumps as "FLUSH TABLES WITH READ LOCK" is forbidden in RDS
* So we do not have any chance to get the exact starting point for reading the binlog on the slave.
  
The [RDS documentation](http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/MySQL.Procedural.Exporting.NonRDSRepl.html "http://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/MySQL.Procedural.Exporting.NonRDSRepl.html") documents an export of data, but not a longer lasting softmigration. For me it's critical to have a working replication to on-premise MySQL over several months, not only a few hours to export my data. Actually we are migrating into AWS and have to connect our old replication chain to the AWS RDS master.

Another point: The RDS documentation is even unclear and buggy. For example it states

> Run the MySQL SHOW SLAVE STATUS statement against the MySQL instance running external to Amazon RDS, and note the master\_host, master\_port, master\_log\_file, and read\_master\_log\_pos values.

But then

> Specify the master\_host, master\_port, master\_log\_file, and read\_master\_log\_pos values you got from the Mysql SHOW SLAVE STATUS statement you ran on the RDS read replica.

Ok, to which master shall I connect? The MySQL instance outside of RDS should not have any master host data set yet, because it's a fresh instance? The master host on the read replica is a private RDS network address, so we could never connect to that from our VPC.

Next point: RDS lets us set a binlog retention time, which is NULL by default. That means binlogs are purged as fast as possible. We had the following case with an external connected slave: The slave disconnected because of some network problem and could not reconnect for some hours. In the meantime the RDS master already purged the binary logs and thus the slave could not replicate anymore:

    
    Got fatal error 1236 from master when reading data from binary log: 'Could not find first log file name in binary log index file'
    

So I was forced to find a solution to setup a fresh external slave from an existing RDS master. And without any downtime of the master because it's mission critical!

First I started to contemplate a plan with master downtime in order to exercise the simple case first.

Here is the plan:  

1. Set binlog rentition on the RDS master to a high value so you are armed against potential network failures:

        > call mysql.rds_set_configuration('binlog retention hours', 24*14);  
        Query OK, 0 rows affected (0.10 sec)  
        
        > call mysql.rds_show_configuration;  
        +------------------------+-------+------------------------------------------------------------------------------------------------------+  
        | name                   | value | description                                                                                          |  
        +------------------------+-------+------------------------------------------------------------------------------------------------------+  
        | binlog retention hours | 336   | binlog retention hours specifies the duration in hours before binary logs are automatically deleted. |  
        +------------------------+-------+------------------------------------------------------------------------------------------------------+  
        1 row in set (0.14 sec)  
      
2. Deny all application access to the RDS database so no new writes can happen and the binlog position stays the same. Do that by removing inbound port 3306 access rules (except your admin connection) from the security groups attached to your RDS instance. Write them down because you have to re-add them later. At this time your master is "offline".
3. Get the current binlog file and position from the master, do it at least 2 times and wait some seconds inbetween in order to validate it does not change anymore. Also check `SHOW PROCESSLIST` whether you and rdsadmin are the only connected users against the RDS master.
4. Get a mysqldump (without locking which is forbidden by RDS, as stated above):

        $ mysqldump -h <read replica endpoint> -u <user> -p<password> --single-transaction --routines --triggers --databases <list of databases> | gzip > mydump.sql.gz

5. rsync/scp to slave
6. call `STOP SLAVE` on your broken or new external slave
7. Import dump
8. Set binlog position on the external slave (I assume the remaining slave settings, e. g. credentials, are already set up).

        CHANGE MASTER TO MASTER_LOG_FILE='mysql-bin-changelog.021761', MASTER_LOG_POS=120

9. Re-add RDS master ingress security rules (or at least add the inbound security rule which allows the external slave to connect to the RDS master).
10. Start external slave. The slave should now catch up with the RDS master.
11. Re-add remaining RDS master security group ingress rules if any.

Ok, now we know how to do it with downtime. This might be OK for testing and staging environments, but not for production databases.

### How can we do it without downtime of the RDS master?

The AWS manual says we should create a RDS read replica and mysqldump the read replica instead of the master, but it is unclear and buggy about how to obtain the master binlog position.  
  
But using a read replica is actually the first correct step.  
  
So here is my alternative plan:  
  
Spin up a read replica, stop the replication manually.

    
    > CALL mysql.rds_stop_replication;  
    +---------------------------+  
    | Message                   |  
    +---------------------------+  
    | Slave is down or disabled |  
    +---------------------------+  
    1 row in set (1.10 sec)
    

Now we can see which master binlog position the slave currently is at via the Exec\_Master\_Log\_Pos variable. This is the pointer to the logfile of the RDS master and thus we now know the the exact position from where to start after setting up our new external slave. The second value we need to know is the binlog file name, this is Relay\_Master\_Log\_File - for example:

    
            Relay_Master_Log_File: mysql-bin-changelog.022019  
              Exec_Master_Log_Pos: 422
    

As the [mysql documentation states](http://dev.mysql.com/doc/refman/5.6/en/show-slave-status.html "http://dev.mysql.com/doc/refman/5.6/en/show-slave-status.html"):

> The position in the current master binary log file to which the SQL thread has read and executed, marking the start of the next transaction or event to be processed. You can use this value with
> the CHANGE MASTER TO statement's MASTER\_LOG\_POS option when starting a new slave from an existing slave, so that the new slave reads from this point. The coordinates given by
> (Relay\_Master\_Log\_File, Exec\_Master\_Log\_Pos) in the master's binary log correspond to the coordinates given by (Relay\_Log\_File, Relay\_Log\_Pos) in the relay log.

Now we got the 2 values we need and we have consistent state to create a dump because the read replica stopped replication.
    
    $ mysqldump -h <read replica endpoint> -u <user> -p<password> --single-transaction --routines --triggers --databases <list of databases> | gzip > mydump.sql.gz
    

Now follow the steps 5-8 and 10 from above.  
  
You should have a running external read slave which is connected to the RDS master by now. You may delete the RDS read replica again as well.  
  
**Happy replicating!**
