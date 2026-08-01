---
title: mariadb vs postgres
toc: true
header-includes:
    <link rel="stylesheet" href="/style.css">
---


A comparison of mariadb and postgresql based on chat data.

## Introduction

I wrote a chat bot in Golang that saves messages.  
These tests were run with 161 million chat messages saved.  
The sql table looks as follows (output from mariadb):

```sql
CREATE TABLE IF NOT EXISTS messages(
    id SERIAL,
    streamer VARCHAR(255),
    user VARCHAR(255),
    message LONGTEXT,
    intime DATETIME NOT NULL DEFAULT current_timestamp()
) ENGINE=InnoDB
```

The mariadb SERIAL type is a BIG UNSIGNED INT with auto increment and index.

The hardware used: 

 - Intel(R) Core(TM) i5-7500 CPU @ 3.40GHz
 - 16GB DDR4 RAM (2400 MT/s)
 - 256GB NVME SSD (Samsung PM961)

Database versions (from `SELECT VERSION();`):

 - PostgreSQL 15.8 (Debian 15.8-0+deb12u1) on x86_64-pc-linux-gnu, compiled by gcc (Debian 12.2.0-14) 12.2.0, 64-bit
 - 10.11.6-MariaDB-0+deb12u1

The data was initially transfered from mariadb to postgres with pgloader.


## Comparison

### COUNT(*)

Query:  
```sql
SELECT COUNT(*) FROM messages;
```

Output (same for both): `161572889`

Time taken:

| mariadb    | postgresql |
|------------|------------|
| 29.731 sec | 13.196 sec |

---

### GROUP BY DATE

Query:  
```sql
SELECT COUNT(*), DATE(intime) FROM messages GROUP BY DATE(intime);
```

Time taken:

| mariadb    | postgresql |
|------------|------------|
| 61.378 sec | 14.194 sec |


This benchmark showed mariadb only using one core at once with 100% usage. Postgres used 3 cores at once with 60-100% usage with each core.

---

### DATE Interval

Query:  
```sql
//mariadb:
SELECT COUNT(*) FROM messages WHERE intime >= DATE_SUB(NOW(), INTERVAL 1 DAY);
//postgres:
SELECT COUNT(*) FROM messages WHERE intime >= NOW() - INTERVAL '1 DAY';
```

The output was 0 for both systems, which was OK.

Time taken:

| mariadb    | postgresql |
|------------|------------|
| 33.871 sec | 13.458 sec |

---

### Backup & Restore

Backup and restore commands for each system:

| cmd/db  | mariadb   | postgres |
|---------|-----------|----------|
| backup  | mysqldump | pg_dump  |
| restore | mysql     | psql     |

Both database systems support the use of non-blocking backups. This means that you can create a complete and consistent snapshot of your database during ongoing operation without any downtime or waiting inserts.

Source:  

 - [https://www.postgresql.org/docs/current/backup-dump.html](https://www.postgresql.org/docs/current/backup-dump.html) , last paragraph before 26.1.1  
 - [https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html#option_mysqldump_single-transaction](https://dev.mysql.com/doc/refman/8.0/en/mysqldump.html#option_mysqldump_single-transaction)  

The times taken:

| type    | mariadb    | postgres  |
|---------|------------|-----------|
| dump    | 2m36.195s  | 2m8.843s  |
| restore | 18m57.465s | 7m21.162s |

Commands used:

```bash
mariadb-dump --databases chat > mariadb.sql
mysql chat < mariadb.sql
pg_dump chat > postgres.sql
psql -d chat -f postgres.sql
```

The mariadb.sql file was 14GB in size, the postgres.sql file was 12GB in size.

Old test with 45 million database rows years prior:

| type    | mariadb   | postgres |
|---------|-----------|----------|
| backup  | 1:11min   | 1:04min  |
| restore | 10:02min  | 2:10min  |


