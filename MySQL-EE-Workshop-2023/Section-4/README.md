# MySQL High Availability
## InnoDB Cluster
Create instance 3308
```
mysqlsh -e "dba.deploySandboxInstance(3308)"

mysql -uroot -h::1 -P3308

set persist_only innodb_redo_log_capacity=2147483648;
set persist_only innodb_flush_neighbors=2;
set persist_only innodb_io_capacity=3000;
set persist_only innodb_io_capacity_max=3000;
set persist_only innodb_buffer_pool_size=25769803776;
set persist_only innodb_buffer_pool_instances=12;
set persist_only innodb_lru_scan_depth=250;
set persist_only innodb_page_cleaners=12;
set persist_only innodb_checksum_algorithm=strict_crc32;
set persist_only binlog_row_image=MINIMAL;

set persist_only innodb_thread_sleep_delay=500; 
set persist_only innodb_spin_wait_delay=2;
set persist_only innodb_spin_wait_pause_multiplier=10;

set persist_only sql_generate_invisible_primary_key=on;

restart;

exit;
```
Configure instance
```
mysqlsh -- dba configure-instance { --host=127.0.0.1 --port=3306 --user=root } --clusterAdmin=gradmin --clusterAdminPassword='grpass' --interactive=false --restart=true

mysqlsh -- dba configure-instance { --host=127.0.0.1 --port=3307 --user=root } --clusterAdmin=gradmin --clusterAdminPassword='grpass' --interactive=false --restart=true

mysqlsh -- dba configure-instance { --host=127.0.0.1 --port=3308 --user=root } --clusterAdmin=gradmin --clusterAdminPassword='grpass' --interactive=false --restart=true

```
Create InnoDB Cluster
```
mysqlsh gradmin:grpass@localhost:3306 -- dba create-cluster mycluster
```
Tuning InnoDB Cluster
```
mysql -uroot -h::1

set persist group_replication_paxos_single_leader=on;
set persist_only group_replication_compression_threshold=100;
set persist group_replication_poll_spin_loops=100;

restart;
exit
```
Start InnoDB Cluster
```
mysqlsh gradmin:grpass@localhost:3306 -- dba rebootClusterFromCompleteOutage

mysqlsh gradmin:grpass@localhost:3306 -- cluster status
```
Add instances
```
mysqlsh gradmin:grpass@localhost:3306 -- cluster add-instance gradmin:grpass@localhost:3307 --recoveryMethod=incremental

mysqlsh gradmin:grpass@localhost:3306 -- cluster add-instance gradmin:grpass@localhost:3307 --recoveryMethod=clone

mysqlsh gradmin:grpass@localhost:3306 -- cluster status
```
Fine Tune instance 3307
```
mysql -uroot -h::1 -P3307

set persist group_replication_poll_spin_loops=100;
set persist_only group_replication_compression_threshold=100;

restart.
exit;
```
Fine Tune instance 3308
```
mysql -uroot -h::1 -P3308

set persist group_replication_poll_spin_loops=100;
set persist_only group_replication_compression_threshold=100;

restart.
exit;
```
Check InnoDB Cluster Status
```
mysqlsh gradmin:grpass@localhost:3306 -- cluster status
```
## InnoDB ClusterSet
Create instance 5506
```
mysqlsh -e "dba.deploySandboxInstance(5506)"

mysql -uroot -h::1 -P5506

set persist_only innodb_redo_log_capacity=2147483648;
set persist_only innodb_flush_neighbors=2;
set persist_only innodb_io_capacity=3000;
set persist_only innodb_io_capacity_max=3000;
set persist_only innodb_buffer_pool_size=25769803776;
set persist_only innodb_buffer_pool_instances=12;
set persist_only innodb_lru_scan_depth=250;
set persist_only innodb_page_cleaners=12;
set persist_only innodb_checksum_algorithm=strict_crc32;
set persist_only binlog_row_image=MINIMAL;

set persist_only innodb_thread_sleep_delay=500; 
set persist_only innodb_spin_wait_delay=2;
set persist_only innodb_spin_wait_pause_multiplier=10;

set persist_only sql_generate_invisible_primary_key=on;

restart;

exit;
```
Configure instance
```
mysqlsh -- dba configure-instance { --host=127.0.0.1 --port=5506 --user=root } --clusterAdmin=gradmin --clusterAdminPassword='grpass' --interactive=false --restart=true
```
Create clusterset and add a replica cluster
```
mysqlsh gradmin:grpass@localhost:3306 -- cluster createClusterSet mycs

mysqlsh gradmin:grpass@localhost:3306 -- clusterset status

mysqlsh gradmin:grpass@localhost:3306 -- clusterset createReplicaCluster gradmin:grpass@localhost:5506 myrep --recoveryMethod=clone

mysqlsh gradmin:grpass@localhost:3306 -- clusterset status

mysqlsh gradmin:grpass@localhost:5506 -- cluster status
```
Fine Tune Replica Cluster
```
mysql -uroot -h::1 -P5506

set persist group_replication_paxos_single_leader=on;
set persist group_replication_poll_spin_loops=100;
set persist_only group_replication_compression_threshold=100;

restart;
exit;
```
Start REPLICA cluster 
```
mysqlsh gradmin:grpass@localhost:5506 -- dba rebootClusterFromCompleteOutage

mysqlsh gradmin:grpass@localhost:5506 -- cluster status
```
## Adding Read Replica to PRIMARY Cluster
Create instance 2206
```
mysqlsh -e "dba.deploySandboxInstance(2206)"

mysql -uroot -h::1 -P2206

set persist_only innodb_redo_log_capacity=2147483648;
set persist_only innodb_flush_neighbors=2;
set persist_only innodb_io_capacity=3000;
set persist_only innodb_io_capacity_max=3000;
set persist_only innodb_buffer_pool_size=25769803776;
set persist_only innodb_buffer_pool_instances=12;
set persist_only innodb_lru_scan_depth=250;
set persist_only innodb_page_cleaners=12;
set persist_only innodb_checksum_algorithm=strict_crc32;
set persist_only binlog_row_image=MINIMAL;

set persist_only innodb_thread_sleep_delay=500; 
set persist_only innodb_spin_wait_delay=2;
set persist_only innodb_spin_wait_pause_multiplier=10;

set persist_only sql_generate_invisible_primary_key=on;

restart;

exit;
```
Configure instance
```
mysqlsh -- dba configure-instance { --host=127.0.0.1 --port=2206 --user=root } --clusterAdmin=gradmin --clusterAdminPassword='grpass' --interactive=false --restart=true
```
Adding node as read replica
```
mysqlsh gradmin:grpass@localhost:3306 -- cluster addReplicaInstance gradmin:grpass@localhost:2206 --recoveryMethod=clone
```
Check cluster status
```
mysqlsh gradmin:grpass@localhost:3306 -- cluster status
```
Change replication sources from PRIMARY to SECONDARY
```
mysqlsh gradmin:grpass@localhost:3306 --cluster -e "cluster.setInstanceOption('localhost:2206','replicationSources','secondary')"

mysqlsh gradmin:grpass@localhost:3306 -- cluster rejoinInstance localhost:2206
```
Check cluster status
```
mysqlsh gradmin:grpass@localhost:3306 -- cluster status
```
## Install and Configure MySQL Router
Install MySQL Router
```
unzip MySQL-router-8.1.zip

sudo yum localinstall -y mysql-router-commercial-8.1.0-1.1.el8.x86_64.rpm
```
Configure MySQL Router
```
mysqlrouter --bootstrap gradmin:grpass@localhost:3306 --directory router
```
Start MySQL Router
```
router/start.sh
```
Connecting to PRIMARY through MySQL Router
```
mysql -ugradmin -pgrpass -h127.0.0.1 -P6446 -e "select @@port"
```
Connecting to SECONDARY through MySQL Router
```
mysql -ugradmin -pgrpass -h127.0.0.1 -P6447 -e "select @@port"

mysql -ugradmin -pgrpass -h127.0.0.1 -P6447 -e "select @@port"
```
Change MySQL Router default R/O to Read Replica
```
mysqlsh gradmin:grpass@localhost:3306 --cluster -e "cluster.getClusterSet().setRoutingOption('read_only_targets','read_replicas')"
```
Now, test connection to MySQL Router port 6447
```
mysql -ugradmin -pgrpass -h127.0.0.1 -P6447 -e "select @@port"
```


