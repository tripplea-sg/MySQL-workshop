# InnoDB ClusterSet with Read Replica
This is small tutorial for InnoDB ClusterSet with Read Replica deployment
## Check host files on all MySQL servers
Ensure /etc/hosts file includes all servers
```
10.0.0.166 	clustera-01
10.0.0.57	clustera-02
10.0.0.5	clustera-03
10.0.0.68	clusterb-01
10.0.0.161	clusterb-02
10.0.0.219	clusterb-03
10.0.0.48	readreplica-01
```
## Set my.cnf
Set different server_id on each server 
```
[mysqld]
datadir=/var/lib/mysql
binlog-format=ROW
log-bin=/var/lib/mysql/bin
port=3306
server_id=1
socket=/var/lib/mysql/mysqld.sock
log-error=/var/log/mysqld.log
enforce_gtid_consistency = ON
gtid_mode = ON
log_slave_updates = ON
```
Start mysql using
```
sudo systemctl start mysqld
```
Check /var/log/mysqld.log for temporary root password </br>
Login to MySQL and change root password to Root#123. 
## Configure Instance
Run configure-instance against all instances
```
mysqlsh -- dba configure-instance { --host=127.0.0.1 --port=3306 --user=root --instance-password=Root#123 } --clusterAdmin=gradmin --clusterAdminPassword='Grpass#123' --interactive=false --restart=true
```
## Create InnoDB ClusterSet
Login into 1st server and create InnoDB Cluster
```
mysqlsh -ugradmin -p"Grpass#123" -h::1 -- dba createCluster mycluster --consistency=BEFORE_ON_PRIMARY_FAILOVER
mysqlsh gradmin@localhost:3306 -- cluster add-instance gradmin@clustera-02:3306 --recoveryMethod=clone
mysqlsh gradmin@localhost:3306 -- cluster add-instance gradmin@clustera-03:3306 --recoveryMethod=clone

mysqlsh gradmin@localhost:3306 -- cluster create-cluster-set clusterset
```
## Add 2nd cluster
```
mysqlsh gradmin@localhost:3306 -- clusterset createReplicaCluster gradmin@clusterb-01:3306 cluster2 --recoveryMethod=clone
```
Connect to PRIMARY node of 2nd cluster and add 2nd & 3rd nodes to Replica Cluster
```
mysqlsh gradmin@localhost:3306 -- cluster add-instance gradmin@clusterb-02:3306 --recoveryMethod=clone
mysqlsh gradmin@localhost:3306 -- cluster add-instance gradmin@clusterb-03:3306 --recoveryMethod=clone

mysqlsh gradmin@localhost:3306 -- cluster add-instance gradmin@readreplica-01:3306 --recoveryMethod=clone
mysqlsh gradmin@localhost:3306 -- cluster remove-instance gradmin@readreplica-01:3306 
```
## Add Read Replica
Connect to Read Replica
```
change replication source to source_host='clusterb-01', source_port=3306, source_user='gradmin', source_password='Grpass#123', source_auto_position=1, get_master_public_key=1, source_ssl=1, source_connection_auto_failover=1, source_connect_retry=3, source_retry_count=3, source_delay=3600 for channel 'channel1';

SELECT asynchronous_connection_failover_add_source('channel1', 'clusterb-01', 3306, '',90);
SELECT asynchronous_connection_failover_add_source('channel1', 'clusterb-02', 3306, '',80);
SELECT asynchronous_connection_failover_add_source('channel1', 'clusterb-03', 3306, '',60);

select * from mysql.replication_asynchronous_connection_failover;
SELECT asynchronous_connection_failover_delete_managed('clusterset_replication','68b461cc-7430-11ed-92bb-0200170183e6');
select * from mysql.replication_asynchronous_connection_failover;

start replica for channel 'channel1';
show replica status for channel 'channel1' \G

set global super_read_only=on;
```
