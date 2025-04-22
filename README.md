# ConfigurePerconaXtraDbCluster
This repository explains how to configure Percona XtraDb Cluster for MySQL InnoDB Engine using Oracle Linux 8-U5.

- Percona XtraDB Cluster (PXC) is an open-source, high-availability and high-performance solution for MySQL and MariaDB database clustering.
- Cluster is a group of servers (nodes) working together to act as a single system for high availability, scalability, and fault tolerance. 

## Important to Know:
- It is recommended by **Percona** that you build the cluster with **odd number** of nodes to maintain **quorum**.
- A _quorum_ is the minimum number of nodes in a cluster that must agree and be active to make decisions and maintain consistency.
- With even number of nodes **Split Brain** problem can occur.
- _Split Brain_ is a situation in distributed systems where network partitions cause cluster nodes to operate independently, risking data inconsistency and conflicts.
- For even number of nodes, you should create another node (**arbitrator**) to avoid Split Brain. (_especially for Production level_)
- An _arbitrator machine_ is a lightweight node in a cluster that helps maintain quorum and resolve split-brain scenarios without storing actual data.
- Percona is based on Galera. And **Galera** only supports **row-level replication**.
- _Row-level replication_ is a replication method where individual row changes (INSERT, UPDATE, DELETE) are copied from one database node to another.
- _Galera_ is a **synchronous multi-master database replication** plugin that enables high-availability clustering for MySQL and MariaDB.
- **SSL (Secure Sockets Layer)** validation is very important during configuration.
- All nodes must have same SSL certificates, otherwise **connection timeout errors** will occur.
- _SSL certificates_ are digital credentials used at the _Presentation layer_ to encrypt data and ensure secure communication between clients and servers over the internet.
- For this lab, I have set the **pxc-encrypt-cluster-traffic = OFF** to bypass SSL security. By default, this attribute is ON. (_Not recommended for production level._)
- All machines must have **same OS version** and same **percona-xtradb-cluster** version.

## Multi-Master Architecture vs Master-Slave Architecture

## Multi-Master Architecture
> All nodes can handle both reads and writes, offering high availability and scalability but **requiring conflict resolution**.
>   
> #### Definition: 
> In a multi-master architecture, all nodes in the cluster can handle both read and write operations. Changes made to any node are replicated to all other nodes in the cluster, ensuring data consistency across all nodes.
> #### Write Operations:
> - Write operations can be performed on any node in the cluster.
> - The changes are synchronously or asynchronously replicated to all other nodes in the cluster, ensuring that all nodes eventually have the same data.
> #### Read Operations:
> - Read operations can also be performed on any node, and the data is guaranteed to be consistent across all nodes (though in some configurations, there might be small delays due to replication lag).

## Master-Slave Architecture
> Only the master handles writes while slaves handle reads, making it simpler but limiting write scalability and posing a **single point of failure**.
>  
> #### Definition: 
> In a master-slave architecture, one node (the master) is responsible for handling all write operations, while one or more slave nodes handle read operations. The master node replicates its data to the slave nodes.
> #### Write Operations:
> - All write operations (insert, update, delete) are performed on the master node.
> - The master node sends the changes to the slave nodes to ensure they have the same data.
> #### Read Operations:
> - Read operations can be distributed across the slave nodes to balance the load and reduce the pressure on the master node.

## Synchronous Replication vs Asynchronous Replication
> **In synchronous replication**, when a write operation (e.g., insert, update, delete) is performed on the primary node, the system ensures that the data is immediately written to all replica nodes before the operation is considered complete.

> **In asynchronous replication**, when a write operation is performed on the primary node, it is immediately confirmed without waiting for the replica nodes to receive the data. The changes are then replicated to the secondary nodes in the background, at a later time.

# In this Lab Manual 
- We will build **cluster with 2 nodes** without Arbitrator Machine.
- VMware Workstation Pro 17 is used to create 2 virtual machines in order to simulate real-world nodes. 

![image](https://github.com/user-attachments/assets/a0fd5e5e-6d8e-477c-9934-0b7c4b7c37b6)
1. I have created 2 virtual machines with above resources. 
2. If you already have mysql. Delete it or make sure it is compatible with latest **percona-xtradb-cluster**.
3. Before installation, stop MySQL service. 
4. Install Percona-xtradb-cluster using following commands:
```
   # Add Percona Repository
   sudo yum install -y https://repo.percona.com/yum/percona-release-latest.noarch.rpm
   # Enable the Percona Repository
   sudo persona-release setup pxc80
   # Install Percona XtraDb Cluster
   sudo yum install -y percona-xtradb-cluster
```
5. Verify installation by using following command:
` mysql --version `
You should see something like:
` mysql Ver 8.0.36-28.1 for Linux on x86_64 (Percona XtraDB Cluster (GPL), Release rel28, Revision bfb687f, WSREP version 26.1.4.3 `
6. Now, run following commands on both machines to check network connectivity between nodes.
```
# On machine with IP: 192.168.41.129
ping 192.169.41.131
# On machine with IP: 192.169.41.131
ping 192.169.41.129
```

     
 



