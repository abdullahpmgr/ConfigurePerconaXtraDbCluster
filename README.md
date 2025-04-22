# ConfigurePerconaXtraDbCluster
This repository explains how to configure Percona XtraDb Cluster for MySQL InnoDB Engine using Oracle Linux 8-U5.

- Percona XtraDB Cluster (PXC) is an open-source, high-availability and high-performance solution for MySQL and MariaDB database clustering.
- Cluster is a group of servers (nodes) working together to act as a single system for high availability, scalability, and fault tolerance. 
- It is based on Galera Cluster and is designed to provide _**synchronous multi-master replication**_ for MySQL databases.

## Important to Know:
- It is recommended by Percona that you build the cluster with **odd number** of nodes to maintain **quorum**.
- Quorum is the minimum number of nodes in a cluster that must agree and be active to make decisions and maintain consistency.
- With even number of nodes **Split Brain** problem can occur.
- Split Brain is a situation in distributed systems where network partitions cause cluster nodes to operate independently, risking data inconsistency and conflicts.
- For even number of nodes, you should create another node (**arbitrator**) to avoid Split Brain. (_especially for Production level_)
- An arbitrator machine is a lightweight node in a cluster that helps maintain quorum and resolve split-brain scenarios without storing actual data.
- Percona is based on Galera. And **Galera** only supports **row-level replication**.
- Galera is a synchronous multi-master database replication plugin that enables high-availability clustering for MySQL and MariaDB.
- Row-level replication is a replication method where individual row changes (INSERT, UPDATE, DELETE) are copied from one database node to another.
- **SSL (Secure Sockets Layer)** certificates validation is very important during configuration.
- All nodes must have same SSL certificates, otherwise **connection timeout errors** will occur.
- SSL certificates are digital credentials used at the _Presentation layer_ to encrypt data and ensure secure communication between clients and servers over the internet.
- For this lab, I have to set the pxc-encrypt-cluster-traffic = OFF to bypass SSL security. By default, this attribute is ON. (_Not recommended for production level._)
- All machines must have **same OS version** and same **percona-xtradb-cluster** version.

