---
title: mariadb
toc: true
header-includes:
    <link rel="stylesheet" href="/style.css">
---


## About

This page is a summary of my learnings with mariadb.

All performance results were tested on an Debian11 server with a 4core 2GHz CPU (Burst 3.35GHz, AMD EPYC 7702P) and 8GB RAM.  
The mariadb was not changed from its default settings, the innodb cache is (according to grafana) 128MiB.

It is not supposed to be a complete overview or a detailed guide on how to use it, but instead just a collection of experiences and accidental benchmarks.


## Installing and Configuration

To install mariadb simply use the default command:

    sudo apt-get install mariadb-server

The only config you may have to change is the listening IP.  
By default the mariadb server only listens on localhost port 3306, which means no one can connect from the outside.  
To enable this you have to edit this in the config that is located in Debian 11 at:

    /etc/mysql/mariadb.conf.d/50-server.cnf

There around line 30 you can change the bind-address to 0.0.0.0 to allow external access.


## RAM usage and performance

By default mariadb only uses 128MB of RAM as cache.  
This can be increased to increase performance. Citation from the mariadb wiki:

> The InnoDB buffer pool is a key component for optimizing MariaDB. It stores data and indexes, and you usually want it as large as possible so as to keep as much of the data and indexes in memory, reducing disk IO, as main bottleneck.

Source: [https://mariadb.com/kb/en/innodb-buffer-pool/](https://mariadb.com/kb/en/innodb-buffer-pool/)

To check how large your current buffer is you can use one of those 2 commandlines as root (the first one uses SQL):

```bash
mysql -e "SHOW VARIABLES LIKE 'innodb_buffer_pool_size'"
mysqld --help --verbose | grep innodb-buffer-pool-size
```

To set it temporary to 1GB you can use the following SQL:

```sql
SET GLOBAL innodb_buffer_pool_size=1073741824;
```

To make this setting persist between restarts you have to edit your config file.  
This config file is most likely at `/etc/mysql/my.cnf`  
In there add the following beneath the socket line:

```
[server]
innodb_buffer_pool_size=1073741824
```

After adding this, my configuration file looks like this:

```
[client-server]
port = 3306
socket = /run/mysqld/mysqld.sock
[server]
innodb_buffer_pool_size=1073741824

!includedir /etc/mysql/conf.d/
!includedir /etc/mysql/mariadb.conf.d/
```

You can now restart mariadb (`systemctl restart mariadb`) and verify with the above commands that the 1GB setting is still set.

It is recommended to set the buffer size to more than 1GB if your server has enough RAM, please consult the wiki or other sources for recommended settings beyond this.


## Storage

I have over 50.000.000 (50million) chat messages saved in one table and around 10.000.000 log lines in another table.  
If i export both with mariadb-dump I get around 6GB of .sql text files.  
The lib folder which contains the database files directly are 7GB in size.


## Backups

Taken from my mariadb-vs-postgres blog:  
Backing up 45.000.000 chat messages with mariadb-dump (or the old mysqldump command) takes around 1.5minutes.

Important: During a backup of a MariaDB InnoDB Database the tables and data are NOT blocked!  
If you create a backup with the following command:

    mariadb-dump --single-transaction --databases mydatabase > backup.sql

Then any other connection can still INSERT or SELECT data from the tables of this database.  
This is because of the `--single-transaction` flag, which requires the storage engine InnoDB and allows you to do non-blocking backups.  

*Warning: If you insert data during a backup it will not be included in it!*

If you do not use this flag then a backup blocks INSERT statements but you can still use SELECT during it.

Source:

 - Tested myself
 - [https://mariadb.com/docs/server/ref/mdb/cli/mariadb-dump/single-transaction/](https://mariadb.com/docs/server/ref/mdb/cli/mariadb-dump/single-transaction/)
 - [https://mariadb.com/kb/en/mariadb-dump/#examples](https://mariadb.com/kb/en/mariadb-dump/#examples)


## Performance for inserts

My installation of mariadb was never "tuned for performance".  

An example of how to raise performance: Set the InnoDB buffer to ~80% of your available RAM.

I have done nothing the like (buffer is at 128MB) and I still get an immense performance out of it.

**A short story about an accidental database benchmark:**

I am currently storing logs for gameservers.  
The server for storing this has 4 CPU cores and 8GB of RAM with 128MB of that being InnoDB buffer.  
They send me logs in 100 lines per request.  
The request flow is: gameserver -> nginx -> golang-webserver -> mariadb  
I save all those 100 lines in a single transaction via my golang webserver (which uses gin and sqlx)

One time someone found out how to create an exponential damage-creator on a gameserver.  
This created 330.000 log lines in ~15 seconds.  
These were sent via 3.300 requests to me and were successfully saved.

Grafana showed me in a single 15s scrape-tick:

 - CPU usage of ~54%
 - 3300 requests to my golang webserver
 - 3260 of these requests were done within 10ms
 - 24Mb/s network traffic
 - 330.000 new log lines in mariadb
 - 7.720 QPS (queries per second) on the mariadb
 - 10MB/s disk R/W
 - 254 io/s IOps on the disk
 - only 2 database connections open from my go webserver
 - no crashes, lags or other

According to the timestamps with which the log lines have been inserted into the mariadb it was around 20.000 log lines per second for around 15 seconds.  
The SQL used to see this is:

    SELECT COUNT(datetime),datetime FROM logs WHERE datetime BETWEEN '1970-01-02 13:01:00' AND '1970-01-02 13:02:15' GROUP BY datetime;

Also, the above SQL statement took, in a table with 5 million loglines, only 3.8s to complete.

To summarize this, you do not need to tune for performance as long as you do not have more than ~30.000 queries/second consistently.


## Performance for SELECT WHERE LIKE

I have a table with chat messages, one row is one chat message with sender,receiver/channel,datetime,id.  
This table has 65630443 messages, counting this takes 15.07s.

If you now want to count how many chat messages have the word sleep in it that will take only 85seconds on average.

```
MariaDB [chat]> SELECT COUNT(*) FROM messages WHERE message LIKE "%sleep%";
+----------+
| COUNT(*) |
+----------+
|   112946 |
+----------+
1 row in set (1 min 25.829 sec)
```

Reruns of this query, after restart or after different queries, always returns a time-taken of about 85seconds.


## Performance by using index

Using an index for an often filtered column can help your performance a lot.

I have a table with logs for different servers. There are around 20 different server IDs and a total of 20.503.200 rows in the table.  

If you want, for example, all log message that contains the word "cheat" on server X you can use the following query:

```
MariaDB [llog]> SELECT ts,msg FROM llog WHERE serverid="X" AND msg LIKE "%cheat%";
Empty set (14.329 sec)
```

As you can see above, it takes 15seconds to search for this, subsequent queries take the same amount of time.

If you now add a simple index like this:

```
MariaDB [llog]> CREATE INDEX llogserverid ON llog(serverid);
Query OK, 0 rows affected (1 min 52.698 sec)
```

The time will be taken down to 0.043s (-99.6%) if you execute the same query again:

```
MariaDB [llog]> SELECT ts,msg FROM llog WHERE serverid="X" AND msg LIKE "%cheat%";
Empty set (0.043 sec)
```


Another example:  
A table with 1.000.000 rows and a column that gets filtered often (column is named serverid with around 30 unique values). The application executes 5 different queries on this table that all use the `WHERE serverid="X"` filter.  

Before creating an index the request took 0.6s (one of the 5 queries took 0.25s alone).  
After creating an index the request took 0.05s (the query above took only 0.01s instead).

This index took 3seconds to create (1mil rows) and dropping the index took 3seconds too.
