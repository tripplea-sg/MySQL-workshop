# Clustering and Data Protection with MySQL InnoDB Cluster and MEB
This hands-on-Lab material covers the following topics:
1. Migrating MariaDB to MySQL Enterprise Edition 
2. Setup InnoDB Cluster 
3. Backup/Recovery with MySQL Enterprise Backup (MEB)

## About MySQL Enterprise Edition
MySQL Enterprise Edition includes the most comprehensive set of advanced features, management tools and technical support to achieve the highest levels of MySQL scalability, security, reliability, and uptime. It reduces the risk, cost, and complexity in developing, deploying, and managing business-critical MySQL applications to meet compliance, business and technical requirements. Oracle own, maintain, and develop all tech stacks in the MySQL Enterprise Edition, where no 3rd party is used for features such as hgh availability, backup/recovery, security, etc. 

## About InnoDB Cluster
InnoDB Cluster is a shared nothing clustering solution for MySQL Enterprise Edition. InnoDB Cluster uses Group Replication as the high availability engine to manage and automate clustering function such as auto-failover, autojoin, auto-healing, etc. as well as manage replication consistency between group members with RPO=0 based on quorum (majority win) as paxos variant. InnoDB Cluster has a requirement of minimum 3 members and maximum 9 members where usually odd number of total members are suggested. In a single primary mode, only 1 node is R/W and called as PRIMARY member, and the rest are SECONDARY members running with R/O. </br></br>
InnoDB Cluster uses MySQL Router to provide connection transparency from application to InnoDB Cluster, making database connection to an InnoDB Cluster the same as connecting to a standalone server. R/W port of MySQL Router will point to PRIMARY member, and R/O port will point to one of SECONDARY member. In case many R/O connections, MySQL Router will load balance R/O connections to all SECONDARY node in round-robin fashon</br></br>
MySQL Shell is used as a tool to build, monitor, and maintain InnoDB Cluster. MySQL Shell uses Admin API, a collection of single commands that can automate and orchestrate InnoDB Cluster operationalization, such as: deploying, switchover, adding nodes, start cluster, handling failure, etc. </br></br>
MySQL InnoDB Cluster has automatic failover. A failure of PRIMARY member will automatically trigger a promotion of one of SECONDARY member to be a new PRIMARY. When this failure member is back, it will automatically join to the group, run incremental recovery until all backlog transactions are applied, and finally ONLINE as SECONDARY member. As long as majority members are intact within the group, automatic failover can happen. Cluster will not work in situation in the event of lost majority. Application connection failover is transparent as it's managed by MySQL Router. </br></br>
InnoDB ClusterSet is a topology to club 2 or more InnoDB Cluster to a single set. One InnoDB Cluster will act as PRIMARY cluster and the rest of InnoDB Cluster within a clusterset will act as REPLICA cluster. Only PRIMARY member of PRIMARY cluster runs R/W, while the rest of members includng PRIMARY member of REPLICA clusters are running on R/O mode. DBA can switchover REPLICA cluster with PRIMARY cluster using MySQL Shell Admin API. MySQL Router for InnoDB Clusterset, by default, will always point to PRIMARY cluster. This solution is to provide disaster recovery tolerance where cross site InnoDB Cluster is not preferable due to network bandwidth and latency consideration. 

## About MySQL Enterprise Backup
MySQL Enterprise Backup (MEB) is an online backup feature to backup MySQL Enterprise Edition database. It is a multi-platform, high-performance tool, offering rich features like “hot” (online) backup, incremental and differential backup, selective backup and restore, support for direct cloud storage backup, backup encryption and compression, and many other valuable features. It's physical backup reading physical datafiles and move pages to a specified backup location as backup files. It doesn't do InnoDB tables locking apart from instance locking for backup in the beginning for less than a second and before completing the backup. For non-InnoDB tables, locking will happen when backing up, but can be skipped using --only-innodb or --no-locking options. We use MEB to restore and recover a MySQL Enterprise Edition database. We use MEB backup metadata information to restore mysql binlog for PITR and meeting business RPO.

## Lab Content
| Topics | Description |
|--------|--------------------------|
| [Lab 1](https://github.com/tripplea-sg/Migration_to_InnoDBCluster/tree/main/section-1) | Migration from MariaDB to MySQL |
| [Lab 2](https://github.com/tripplea-sg/Migration_to_InnoDBCluster/tree/main/section-2) | InnoDB Cluster |
| [Lab 3](https://github.com/tripplea-sg/Migration_to_InnoDBCluster/tree/main/section-3) | Backup using MEB |
| [Lab 4](https://github.com/tripplea-sg/Migration_to_InnoDBCluster/tree/main/section-4) | Restore using MEB |
