# Migrating MariaDB to MySQL Enterprise Edition
## MariaDB database Server
Following is the MariaDB database server to be migrated:
```
IP address: 10.0.0.110
MariaDB user: apps
MariaDB password: apps
```
## Prepare MySQL Server
If you have not done so, please follow these instruction </br>
Install using TAR is flexible and stright forward. However, we have to ensure all pre-requisites are installed on. new server
For new server, RPM or YUM based installation can be use to validate any missing OS dependencies.
```
cd /home/opc/software

unzip MySQL-Server-8.0.30.zip
tar -zxvf mysql-commercial-8.0.30-el7-x86_64.tar.gz

unzip MySQL-Shell-8.0.30.zip
sudo rpm -ivh mysql-shell-commercial-8.0.30-1.1.el7.x86_64.rpm

unzip MySQL-Router-8.0.30.zip
tar -zxvf mysql-router-commercial-8.0.30-el7-x86_64.tar.gz
```
Create environment file to source MySQL Enterprise. Copy the following line and create $HOME/.8030.env file </br>
command: 
```
vi $HOME/.8030.env
```
Copy and paste to vi
```
PATH=$PATH:/home/opc/software/mysql-commercial-8.0.30-el7-x86_64/bin:/home/opc/software/mysql-router-commercial-8.0.30-el7-x86_64/bin
```
Source environment: a process that we need to include MySQL into $PATH
```
. $HOME/.8030.env
```
Create MySQL Database Directory
```
mkdir -p /home/opc/db/3306/data /home/opc/db/3306/innodb_data_home_dir /home/opc/db/3306/innodb_undo_directory /home/opc/db/3306/innodb_temp_tablespace_dir /home/opc/db/3306/innodb_temp_data_file_path /home/opc/db/3306/innodb_log_group_home_dir /home/opc/db/3306/log_bin
```
Create option file / configuration file for the database with the following content </br>
command: 
```
vi /home/opc/db/3306/my.cnf
```
Copy and paste to vi
```
[mysqld]
datadir=/home/opc/db/3306/data
binlog-format=ROW
log-bin=/home/opc/db/3306/log_bin/bin
innodb_data_home_dir=/home/opc/db/3306/innodb_data_home_dir
innodb_undo_directory=/home/opc/db/3306/innodb_undo_directory
innodb_temp_tablespaces_dir=/home/opc/db/3306/innodb_temp_tablespace_dir 
innodb_temp_data_file_path=/home/opc/db/3306/innodb_temp_data_file_path/ibtmp1:12M:autoextend
innodb_log_group_home_dir=/home/opc/db/3306/innodb_log_group_home_dir
port=3306
server_id=10
socket=/home/opc/db/3306/data/mysqld.sock
log-error=/home/opc/db/3306/data/mysqld.log
enforce_gtid_consistency = ON
gtid_mode = ON
log_slave_updates = ON
innodb_buffer_pool_size=1G
innodb_buffer_pool_instances=1
innodb_log_file_size=1G
innodb_log_files_in_group=3
innodb_flush_log_at_trx_commit=1
```
Create database with random root password using the following:
```
mysqld --defaults-file=/home/opc/db/3306/my.cnf --initialize-insecure
```
Start MySQL database
```
mysqld_safe --defaults-file=/home/opc/db/3306/my.cnf &
```
## Execute out of place upgrade from MariaDB to MySQL
Backup MariaDB using MySQL Shell dumpInstance
```
mysqlsh -uapps -papps -h10.0.0.110 -- util dumpInstance '/home/opc/backup' --compatibility=strip_definers
```
Check backup directory
```
ls /home/opc/backup
```
Load metadata to MySQL from backup using MySQL Shell loadDump
```
mysqlsh -uroot -h127.0.0.1 --sql -e 'set global local_infile=on'
mysqlsh -uroot -h127.0.0.1 -- util loadDump '/home/opc/backup' --ignoreVersion=true --loadData=false --createInvisiblePKs=true 
```
Login to MySQL and modify character set definition
```
mysql -uroot -h::1

show databases;
use nation;
alter table continents default charset=utf8mb4;
alter table countries default charset=utf8mb4;
alter table country_languages default charset=utf8mb4;
alter table country_stats default charset=utf8mb4;
alter table guests default charset=utf8mb4;
alter table languages default charset=utf8mb4;
alter table region_areas default charset=utf8mb4;
alter table regions default charset=utf8mb4;
exit;
```
Now, load data from backup to MySQL using MySQL Shell loadDump
```
mysqlsh -uroot -h127.0.0.1 -- util loadDump '/home/opc/backup' --ignoreVersion=true --loadDdl=false --createInvisiblePKs=true
```
Login and query tables
```
mysql -uroot -h::1 -e "select * from nation.languages"
mysql -uroot -h::1 -e "select * from nation.countries"
```
## Smooth Migration with Async Replication
GTID is not compatible, thus replication has to use binlog position. Check binlog position captured by the backup
```
cat /home/opc/backup/@.json | grep binlogFile
cat /home/opc/backup/@.json | grep binlogPosition
```
Login to MySQL and create replication channel
```
mysql -uroot -h::1

<please change value for source_log_file with binlogFile output and source_log_pos with binlogPosition output>

set persist gtid_mode=on_permissive;

change replication source to source_host='10.0.0.110', source_port=3306, source_user='repl', source_password='repl', source_log_file='bin.000001', source_log_pos=327885;

start replica;
show replica status \G
```
During cut-over, remove replicaton channel and set GTID_MODE=on
```
stop replica;
reset replica all;
set persist gtid_mode=on;
``` 
