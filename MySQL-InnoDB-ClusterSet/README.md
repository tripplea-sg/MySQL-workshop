# InnoDB Clusterset

## Create InnoDB Cluster:
```
mysqlsh -e "dba.deploySandboxInstance(3311)"
mysqlsh -e "dba.deploySandboxInstance(3312)"
mysqlsh -e "dba.deploySandboxInstance(3313)"

mysqlsh -- dba configure-instance { --host=127.0.0.1 --port=3311 --user=root } --clusterAdmin=gradmin --clusterAdminPassword='grpass' --interactive=false --restart=true

mysqlsh -- dba configure-instance { --host=127.0.0.1 --port=3312 --user=root } --clusterAdmin=gradmin --clusterAdminPassword='grpass' --interactive=false --restart=true

mysqlsh -- dba configure-instance { --host=127.0.0.1 --port=3313 --user=root } --clusterAdmin=gradmin --clusterAdminPassword='grpass' --interactive=false --restart=true

mysqlsh gradmin:grpass@localhost:3311 -- dba createCluster mycluster --consistency=BEFORE_ON_PRIMARY_FAILOVER

mysqlsh gradmin:grpass@localhost:3311 -- cluster add-instance gradmin:grpass@localhost:3312 --recoveryMethod=incremental

mysqlsh gradmin:grpass@localhost:3311 -- cluster add-instance gradmin:grpass@localhost:3313 --recoveryMethod=incremental

mysqlsh gradmin:grpass@localhost:3311 -- cluster status
```

## Create a Clusterset
```
mysqlsh gradmin:grpass@localhost:3311 --cluster --interactive -e "cluster.createClusterSet('clusterset')"
```

### Create a Replica Cluster
```
mysqlsh -e "dba.deploySandboxInstance(5311)"
mysqlsh -- dba configure-instance { --host=127.0.0.1 --port=5311 --user=root } --clusterAdmin=gradmin --clusterAdminPassword='grpass' --interactive=false --restart=true
mysqlsh gradmin:grpass@localhost:3311 --cluster --interactive -e "cluster.getClusterSet().createReplicaCluster('gradmin:grpass@localhost:5311','cluster2')"

mysqlsh gradmin:grpass@localhost:5311 -- cluster status

mysqlsh gradmin:grpass@localhost:3311 --cluster --interactive -e "cluster.getClusterSet().status()"
```

### Create a Router
```
mysqlrouter --bootstrap gradmin:grpass@localhost:3311 --directory router --account myrouter --account-create always --force

router/start.sh
```

### Adding Node to Replica Cluster
```
mysqlsh -e "dba.deploySandboxInstance(5312)"
mysqlsh -e "dba.deploySandboxInstance(5313)"

mysqlsh -- dba configure-instance { --host=127.0.0.1 --port=5312 --user=root } --clusterAdmin=gradmin --clusterAdminPassword='grpass' --interactive=false --restart=true

mysqlsh -- dba configure-instance { --host=127.0.0.1 --port=5313 --user=root } --clusterAdmin=gradmin --clusterAdminPassword='grpass' --interactive=false --restart=true

mysqlsh gradmin:grpass@localhost:5311 -- cluster add-instance gradmin:grpass@localhost:5312 --recoveryMethod=clone

mysqlsh gradmin:grpass@localhost:5311 -- cluster add-instance gradmin:grpass@localhost:5313 --recoveryMethod=clone

mysqlsh gradmin:grpass@localhost:5311 -- cluster status
```

## Clusterset Testing

### TEST:
```
mysql -uroot -h127.0.0.1 -P3311
mysql> create database test;
mysql> create table test.test (i int primary key);
mysql> insert into test.test values (1), (2), (3);
mysql> exit;

mysql -uroot -h127.0.0.1 -P3311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3312 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3313 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5312 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5313 -e "select * from test.test"
```


### DISASTER RECOVERY SCENARIO TEST:

#### A.	Switch PRIMARY Cluster
```
mysqlsh gradmin:grpass@localhost:3311 --cluster --interactive -e "cluster.getClusterSet().setPrimaryCluster('cluster2')"

mysqlsh gradmin:grpass@localhost:3311 --cluster --interactive -e "cluster.getClusterSet().status()"

mysql -uroot -h127.0.0.1 -P5311 -e "insert into test.test values (4), (5), (6)"

mysql -uroot -h127.0.0.1 -P3311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3312 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3313 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5312 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5313 -e "select * from test.test"
```

#### B.	Switch back PRIMARY Cluster
```
mysqlsh gradmin:grpass@localhost:3311 --cluster --interactive -e "cluster.getClusterSet().setPrimaryCluster('mycluster')"

mysqlsh gradmin:grpass@localhost:3311 --cluster --interactive -e "cluster.getClusterSet().status()"

mysql -uroot -h127.0.0.1 -P6446 -e "insert into test.test values (7), (8), (9)"

mysql -uroot -h127.0.0.1 -P3311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3312 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3313 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5312 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5313 -e "select * from test.test"
```
#### C.	Restart Secondary Node of PRIMARY Cluster
```
mysql -uroot -h127.0.0.1 -P3312 -e "restart"
mysql -uroot -h127.0.0.1 -P6446 -e "insert into test.test values (10), (11), (12)"

mysql -uroot -h127.0.0.1 -P3311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3312 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3313 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5312 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5313 -e "select * from test.test"
```

#### D.	Restart Primary Node of PRIMARY Cluster
```
mysql -uroot -h127.0.0.1 -P3311 -e "restart"
mysql -uroot -h127.0.0.1 -P6446 -e "insert into test.test values (13), (14), (15)"

mysql -uroot -h127.0.0.1 -P3311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3312 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3313 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5312 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5313 -e "select * from test.test"
```

#### E.	PRIMARY node switchover on PRIMARY Cluster
```
mysqlsh gradmin:grpass@localhost:3311 -- cluster setPrimaryInstance localhost:3313

mysql -uroot -h127.0.0.1 -P6446 -e "insert into test.test values (16), (17), (18)"

mysql -uroot -h127.0.0.1 -P3311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3312 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3313 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5312 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5313 -e "select * from test.test"
```
#### F.	Restart Primary Node of SECONDARY Cluster
```
mysql -uroot -h127.0.0.1 -P5311 -e "restart"
mysql -uroot -h127.0.0.1 -P6446 -e "insert into test.test values (19), (20), (21)"

mysql -uroot -h127.0.0.1 -P3311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3312 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3313 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5312 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5313 -e "select * from test.test"

mysqlsh gradmin:grpass@localhost:5312 -- cluster status
```

#### G.	PRIMARY node switchover on SECONDARY Cluster
```
mysqlsh gradmin:grpass@localhost:5311 -- cluster setPrimaryInstance localhost:5313
mysql -uroot -h127.0.0.1 -P6446 -e "insert into test.test values (22), (23), (24)"

mysql -uroot -h127.0.0.1 -P3311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3312 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3313 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5312 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5313 -e "select * from test.test"
```

#### H.	Loss of PRIMARY Cluster
```
mysqladmin -uroot -h127.0.0.1 -P3311 shutdown
mysqladmin -uroot -h127.0.0.1 -P3312 shutdown
mysqladmin -uroot -h127.0.0.1 -P3313 shutdown

mysqlsh gradmin:grpass@localhost:5311 --cluster --interactive -e "cluster.getClusterSet().status()"

mysqlsh gradmin:grpass@localhost:5311 --cluster --interactive -e "cluster.getClusterSet().forcePrimaryCluster('cluster2')"

mysqlsh gradmin:grpass@localhost:5311 --cluster --interactive -e "cluster.getClusterSet().status()"

mysql -uroot -h127.0.0.1 -P6446 -e "insert into test.test values (25), (26), (27)"

mysql -uroot -h127.0.0.1 -P5311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5312 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5313 -e "select * from test.test"
```

#### I.	Resume PRIMARY Cluster back as SECONDARY Cluster
```
mysqlsh -e "dba.startSandboxInstance(3311)"
mysqlsh -e "dba.startSandboxInstance(3312)"
mysqlsh -e "dba.startSandboxInstance(3313)"

mysqlsh gradmin:grpass@localhost:3312 -- dba rebootClusterFromCompleteOutage

mysqlsh gradmin:grpass@localhost:3312 -- cluster status
```
Result:
```
[opc@clusterset ~]$ mysqlsh gradmin:grpass@localhost:3312 -- cluster status
WARNING: Using a password on the command line interface can be insecure.
{
    "clusterName": "mycluster", 
    "defaultReplicaSet": {
        "name": "default", 
        "primary": "127.0.0.1:3312", 
        "ssl": "REQUIRED", 
        "status": "OK", 
        "statusText": "Cluster is ONLINE and can tolerate up to ONE failure.", 
        "topology": {
            "127.0.0.1:3311": {
                "address": "127.0.0.1:3311", 
                "memberRole": "SECONDARY", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "replicationLagFromImmediateSource": "", 
                "replicationLagFromOriginalSource": "", 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.27"
            }, 
            "127.0.0.1:3312": {
                "address": "127.0.0.1:3312", 
                "memberRole": "PRIMARY", 
                "mode": "R/W", 
                "readReplicas": {}, 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.27"
            }, 
            "127.0.0.1:3313": {
                "address": "127.0.0.1:3313", 
                "memberRole": "SECONDARY", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "replicationLagFromImmediateSource": "", 
                "replicationLagFromOriginalSource": "", 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.27"
            }
        }, 
        "topologyMode": "Single-Primary"
    }, 
    "groupInformationSourceMember": "127.0.0.1:3312"
}
```
Continue testing:
```
mysql -uroot -h127.0.0.1 -P6446 -e "insert into test.test values (28), (29), (30)"

mysql -uroot -h127.0.0.1 -P5311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5312 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5313 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3312 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3313 -e "select * from test.test"
```
Rejoin Cluster
```
mysqlsh gradmin:grpass@localhost:5311 --cluster --interactive -e "cluster.getClusterSet().rejoinCluster('mycluster')"

mysql -uroot -h127.0.0.1 -P5311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5312 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5313 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3312 -e "select * from test.test"
```
Result:
```
[opc@clusterset ~]$ mysqlsh gradmin:grpass@localhost:3312 -- cluster status
WARNING: Using a password on the command line interface can be insecure.
{
    "clusterName": "mycluster", 
    "defaultReplicaSet": {
        "name": "default", 
        "primary": "127.0.0.1:3312", 
        "ssl": "REQUIRED", 
        "status": "OK", 
        "statusText": "Cluster is ONLINE and can tolerate up to ONE failure.", 
        "topology": {
            "127.0.0.1:3311": {
                "address": "127.0.0.1:3311", 
                "memberRole": "SECONDARY", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "replicationLagFromImmediateSource": "", 
                "replicationLagFromOriginalSource": "", 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.27"
            }, 
            "127.0.0.1:3312": {
                "address": "127.0.0.1:3312", 
                "memberRole": "PRIMARY", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "replicationLagFromImmediateSource": "", 
                "replicationLagFromOriginalSource": "", 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.27"
            }, 
            "127.0.0.1:3313": {
                "address": "127.0.0.1:3313", 
                "memberRole": "SECONDARY", 
                "mode": "R/O", 
                "readReplicas": {}, 
                "replicationLagFromImmediateSource": "", 
                "replicationLagFromOriginalSource": "", 
                "role": "HA", 
                "status": "ONLINE", 
                "version": "8.0.27"
            }
        }, 
        "topologyMode": "Single-Primary"
    }, 
    "groupInformationSourceMember": "127.0.0.1:3312", 
    "metadataServer": "127.0.0.1:5313"
}
```
Continue testing:
```
mysql -uroot -h127.0.0.1 -P6446 -e "insert into test.test values (31), (32), (33)"

mysql -uroot -h127.0.0.1 -P5311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5312 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5313 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3312 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3313 -e "select * from test.test"
```

#### J.	Switch Back PRIMARY Cluster
```
mysqlsh gradmin:grpass@localhost:3312 --cluster --interactive -e "cluster.getClusterSet().setPrimaryCluster('mycluster')"

mysql -uroot -h127.0.0.1 -P6446 -e "insert into test.test values (34), (35), (36)"

mysql -uroot -h127.0.0.1 -P5311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5312 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P5313 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3311 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3312 -e "select * from test.test"
mysql -uroot -h127.0.0.1 -P3313 -e "select * from test.test"
```
