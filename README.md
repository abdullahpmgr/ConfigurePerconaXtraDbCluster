![image](https://github.com/user-attachments/assets/490ded0c-c9fa-40ee-a699-e594c2806885)


# ConfigurePerconaXtraDbCluster
This repository explains how to configure Percona XtraDb Cluster for MySQL InnoDB Engine using Oracle Linux 8-U5.

> - Percona XtraDB Cluster (PXC) is an open-source, high-availability and high-performance solution for MySQL and MariaDB database clustering.
> - Cluster is a group of servers (nodes) working together to act as a single system for high availability, scalability, and fault tolerance. 

## Important to Know:
> - It is recommended by **Percona** that you build the cluster with **odd number** of nodes to maintain **quorum**.
> - A _quorum_ is the minimum number of nodes in a cluster that must agree and be active to make decisions and maintain consistency.
> - With even number of nodes **Split Brain** problem can occur.
> - _Split Brain_ is a situation in distributed systems where network partitions cause cluster nodes to operate independently, risking data inconsistency and conflicts.
> - For even number of nodes, you should create another node (**arbitrator**) to avoid Split Brain. (_especially for Production level_)
> - An _arbitrator machine_ is a lightweight node in a cluster that helps maintain quorum and resolve split-brain scenarios without storing actual data.
> - Percona is based on Galera. And **Galera** only supports **row-level replication**.
> - _Row-level replication_ is a replication method where individual row changes (INSERT, UPDATE, DELETE) are copied from one database node to another.
> - _Galera_ is a **synchronous multi-master database replication** plugin that enables high-availability clustering for MySQL and MariaDB.
> - **SSL (Secure Sockets Layer)** validation is very important during configuration.
> - All nodes must have same SSL certificates, otherwise **connection timeout errors** will occur.
> - _SSL certificates_ are digital credentials used at the _Presentation layer_ to encrypt data and ensure secure communication between clients and servers over the internet.
> - For this lab, I have set the **pxc-encrypt-cluster-traffic = OFF** to bypass SSL security. By default, this attribute is ON. (_Not recommended for production level._)
> - All machines must have **same OS version** and same **percona-xtradb-cluster** version.

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
   sudo percona-release setup pxc80
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
7. Deactivate the firewall on both machines using following commands:
   ```
   sudo su - root
   # To check firewall's status
   systemctl status firewalld
   # if it is active, stop it using following command
   systemctl stop firewalld
   # Also disable firewalld process
   systemctl disable firewalld
   # Make sure firewalld is inactive
   systemctl status firewalld
   ```
8. Disable SELinux on both machines:
   ```
   # Open the security Linux file in vim editor
   vi /etc/sysconfig/selinux
   # Set SELinux to disabled
   SELINUX = disabled
   # Reboot the system
   init 6
   # Check the SELinux status
   sestatus
   ```
   (If the firewall is inactive, selinux is disabled, and both machines are connected with eachother, we are good to go)

9. Open configuration file in Virtual Machine 01 (node1)
   ```
   # open my.cnf file in vim editor
   vi /etc/my.cnf
   --------------------------------------------- my.cnf -------------------------------------------------------------
   
   # my.cnf for Percona XtraDB Cluster (PXC)
   # Customize based on your environment
   
   [client]
   socket = /var/lib/mysql/mysql.sock
   
   [mysqld]
   server-id = 1
   datadir = /var/lib/mysql
   socket = /var/lib/mysql/mysql.sock
   log-error = /var/log/mysqld.log
   pid-file = /var/run/mysqld/mysqld.pid
   
   # Binary log expiration period (7 days)
   binlog_expire_logs_seconds = 604800
   
   ######## Galera / wsrep Settings ########
   
   # Path to Galera library
   wsrep_provider = /usr/lib64/galera4/libgalera_smm.so
   
   # Cluster node addresses (comma-separated, no space after comma)
   # Use gcomm:// with IPs of all nodes in the cluster
   wsrep_cluster_address = gcomm://192.168.41.131,192.168.41.129
   
   # Required for Galera
   binlog_format = ROW
   
   # Number of replication threads
   wsrep_slave_threads = 8
   
   # Log conflicts (corrected key name)
   wsrep_log_conflicts = ON
   
   # Required setting for Galera
   innodb_autoinc_lock_mode = 2
   
   # Optional: define node IP address
   # wsrep_node_address = 192.168.41.131
   
   # Cluster name
   wsrep_cluster_name = pxc_cluster
   
   # Optional: define node name (otherwise hostname is used)
   wsrep_node_name = node1
   
   # PXC strict mode values: DISABLED, PERMISSIVE, ENFORCING, MASTER
   pxc_strict_mode = DISABLED
   
   # Cluster traffic encryption (set OFF for non-SSL cluster)
   pxc-encrypt-cluster-traffic = OFF
   
   # SST (State Snapshot Transfer) method
   wsrep_sst_method = xtrabackup-v2
   
   [sst]
   # Disable SST encryption
   encrypt = 0


   # important to set encrypt = 0, to bypass encrypt
   -------------------------------------------------------------------------------------
   ```
   When done editing, save changes to file:
   Press ESC.
   :wq!  

10. Open configuration file in Virtual Machine 02 (node2)
   ```
   # open my.cnf file in vim editor
   vi /etc/my.cnf
   --------------------------------------------- my.cnf -------------------------------------------------------------
   # my.cnf for Percona XtraDB Cluster - Node 2
   # Adjust settings as needed for your environment
   
   [client]
   socket = /var/lib/mysql/mysql.sock
   
   [mysqld]
   server-id = 2
   datadir = /var/lib/mysql
   socket = /var/lib/mysql/mysql.sock
   log-error = /var/log/mysqld.log
   pid-file = /var/run/mysqld/mysqld.pid
   
   # Binary log expiration period (7 days)
   binlog_expire_logs_seconds = 604800
   
   ######## Galera / wsrep Settings ########
   
   # Path to Galera library
   wsrep_provider = /usr/lib64/galera4/libgalera_smm.so
   
   # Cluster node addresses (comma-separated, no space after comma)
   wsrep_cluster_address = gcomm://192.168.41.131,192.168.41.129
   
   # Required for Galera replication
   binlog_format = ROW
   
   # Number of replication threads
   wsrep_slave_threads = 8
   
   # Log conflicts (corrected)
   wsrep_log_conflicts = ON
   
   # Required InnoDB setting for Galera
   innodb_autoinc_lock_mode = 2
   
   # Optional: node IP address
   # wsrep_node_address = 192.168.41.129
   
   # Cluster name
   wsrep_cluster_name = pxc_cluster
   
   # Optional: define node name (otherwise hostname is used)
   wsrep_node_name = node2
   
   # PXC strict mode: DISABLED, PERMISSIVE, ENFORCING, MASTER
   pxc_strict_mode = DISABLED
   
   # Disable encryption of cluster traffic (only use in trusted environments)
   pxc-encrypt-cluster-traffic = OFF
   
   # SST (State Snapshot Transfer) method
   wsrep_sst_method = xtrabackup-v2
   
   [sst]
   # Disable SST encryption
   encrypt = 

   # important to set encrypt = 0, to bypass encrypt
   -------------------------------------------------------------------------------------
   ```
   When done editing, save changes to file:
   Press ESC.
   :wq!  
   
11. Now, Start mysql@bootstrap in Virtual Machine 01 (node1):
   ```
   sudo su - root
   systemctl start mysql@bootstrap
   # check the status of mysql@bootstrap
   systemctl status mysql@bootstrap
   ```
It should be ACTIVE now.
   ```
   # OPEN MySQL
   mysql -u root -p
   # Enter password for root user
   # if you forget the roow user password:
      # run the following command to get temporary password for root:
      sudo grep 'temporary password' /var/log/mysqld.log
   
   # Enter the last password to enter into mysql.
   mysql -u root -p
   # Now, change the root user's password before any SQL
   ALTER USER 'root'@'localhost' IDENTIFIED BY 'YourNewPasswordForRoot';
   FLUSH PRIVILEGES;
   
   # Run the following command to make sure that the cluster has been initialized.
   show status LIKE 'wsrep%'
   # You should see the following output:
   +------------------------------+--------------------------------------+
   | Variable_name                | Value                                |
   +------------------------------+--------------------------------------+
   | wsrep_cluster_status         | Primary                              |
   | wsrep_connected              | ON                                   |
   | wsrep_ready                  | ON                                   |
   | wsrep_local_state_comment    | Synced                               |
   | wsrep_cluster_size           | 1                                    |
   | wsrep_local_state            | 4                                    |
   | wsrep_local_index            | 1                                    |
   | wsrep_incoming_addresses     | node1:3306,node2:3306,node3:3306     |
   +------------------------------+--------------------------------------+
   ```
12. Now, on Virtual Machine 02 (node2), start mysql as normal service.
   ` systemctl start mysql`
If it is running, you should see that your cluster's size has now increased from 1 to 2.

# Restarting the Cluster
Galera Clusters are generally meant to run non-stop, so shutting down the entire cluster is not required during normal operation. Yet, if there is a need to perform such procedure, it is likely that it will be happening under pressure, so it is important for it to complete safely and as quickly as possible in order to avoid extended downtime and potential data loss.

## Whole Cluster Restart
First, a few words on cluster restart in general. Regardless of whether it was an orderly shutdown or a sudden crash of all nodes, restarting the entire cluster is governed by the following principles:
- Since the old cluster no longer logically exists, a new logical cluster is being created.
- The first node being started must be bootstrapped.
- _It is important to select the node that has the last transactions committed as the first node in the new cluster._
- Yesterday, when I halted the machines, my VirtualMachine02 (node2) was the last to shutdown.
- Now, I have to bootstrap VirtualMachine02 (node2) first.
- And then we need to start mysql service normally on VirtualMachine01.
- Follow the following document to understand _SAFE TO BOOTSTRAP FEATURE_. This feature facilitates to restart cluster again quickly in case of failure.
- [Safe to Bootstrap Feature in Galera Cluster]([https://example.com](https://galeracluster.com/2016/11/introducing-the-safe-to-bootstrap-feature-in-galera-cluster/))

# Adding Galera Arbitrator machine
   ## Galera Arbitrator:
      Galera Arbitrator is a member of Percona XtraDb Cluster that is used for voting in case you have small number of servers (usually two) and don't want to add any more resources.
      Galera Arbitrator does not need a dedicated server. It can be installed on a machine running some other application. Just make sure it has good network connectivity.
      When deploying a Galera Cluster, it's recommended that you use a minimum of three instances: Three nodes, three data centers and so on.
      If the cost of adding resources (e.g., a third data center) is too much, you can use Galera Arbitrator. Galera Arbitrator is a member of a cluster that participates in voting,
      but not in actual replication.
 


 
 



