## To see thread assignment in MySQL
```
ps -Lp `pgrep mysqld` -o tid,cmd,comm,psr.ni
```
## To see cpu core utilization for each MySQL thread
```
idstat -t -p `pgrep mysqld` 1:
```
## To see MySQL threads 
```
select * from performance_schema.threads;
```
## Implement Resource Group
```
sudo setcap cap_sys_nice+ep /usr/sbin/mysqld
```
## open file limit
```
vi /etc/security/limits.conf

* soft nofile 102400
* hard nofile 102400
* soft nproc unlimited
* hard nproc unlimited
* soft memlock unlimited
* hard memlock unlimited

vi /etc/security/limits.d/90-nproc.conf

* soft nofile 1024000
* hard nofile 1024000
* soft nproc 10240
* hard nproc 10240
* soft memlock unlimited
* hard memlock unlimited
root soft nproc unlimited

vi /etc/systemd/system/mysqld.service.d/perf-tuning.conf 
LimitNOFILE=1024000

sudo systemctl daemon-reload
sudo systemctl start mysqld

vi /etc/my.cnf
innodb_open_files = 1024000

vi /usr/bin/mkcnf
change
__file__ = "/etc/my.cnf.d/perf-tuning.cnf"
to
__file__ = "/tmp/perf-tuning.cnf"
```
## Setup MySQL thread priority for InnoDB Cluster stability
```
sudo setcap cap_sys_nice+ep /usr/sbin/mysqld

#!/bin/bash

cat /dev/null > /tmp/renice_mysql.lst

mysql -uroot -h127.0.0.1 -pQazse_123 -N -B -e "tee /tmp/renice_mysql.lst; select concat('set resource group mysql_internal for ',thread_id,';') from performance_schema.threads where resource_group='SYS_default' and name not like '%page%' and name<>'thread/sql/main'; notee;"

cat /tmp/renice_mysql.lst | sed 's/set /mysql -uroot -h127.0.0.1 -pQazse_123 -e \"set /g' | sed 's/;/;\"/g' > /tmp/renice_mysql.sh
chmod u+x /tmp/renice_mysql.sh
/tmp/renice_mysql.sh

 pidstat -t -p `pgrep mysqld`
 
 ps -Lp `pgrep mysqld` -o tid,cmd,comm,psr,ni 
```
## Setup Huge Pages
Check
```
cat /proc/meminfo | grep -i huge
```
Run
```
# Set the number of pages to be used.
# Each page is normally 2MB, so a value of 20 = 40MB.
# This command actually allocates memory, so this much
# memory must be available.
echo 20 > /proc/sys/vm/nr_hugepages

# Set the group number that is permitted to access this
# memory (102 in this case). The mysql user must be a
# member of this group.
echo 102 > /proc/sys/vm/hugetlb_shm_group

# Increase the amount of shmem permitted per segment
# (12G in this case).
echo 1560281088 > /proc/sys/kernel/shmmax

# Increase total amount of shared memory.  The value
# is the number of pages. At 4KB/page, 4194304 = 16GB.
echo 4194304 > /proc/sys/kernel/shmall

sudo id mysql

vi /etc/sysctl.conf
vm.hugetlb_shm_group= 27
kernel.shmmax = 1073741824
kernel.shmall = 2883584
vm.nr_hugepages = 5632

reboot
```
Check
```
cat /proc/meminfo | grep -i huge
```
Put this line to my.cnf
```
large-pages
```
Start mysql and check server log for this message (in case any): Warning: Using conventional memory pool
## Row-level locking
```
set global innodb_deadlock_detect=OFF;
set global innodb_lock_wait_timeout=50;
```
## InnoDB Read I/O Threads
```
set innodb_read_io_threads=4
```
