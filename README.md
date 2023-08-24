Setting up Galera Replication + HA cluster using Corosync and Pacemaker  on ubuntu 18 by Pranish Ghimire

**If you Dont Want Galera and would like to Use Mysql 8.0 Master/Slave Replication , Skip Galera and proceed , and Follow the tutorial in the End**


setup hostname using the below command on NODE01 and NODE02 respectively 

    hostnamectl set-hostname NODE01
    hostnamectl set-hostname NODE02

 ****In both VM (nodes) make the following entries in /etc/hosts****
 
    192.168.1.102 NODE01 
    192.168.1.103 NODE02
    

### Installation of Galera Cluster 


    sudo apt-key adv --keyserver keyserver.ubuntu.com --recv BC19DDBA
nano /etc/apt/sources.list.d/galera.list

paste below line 

    deb https://releases.galeracluster.com/galera-4/ubuntu bionic main
    deb https://releases.galeracluster.com/mysql-wsrep-8.0/ubuntu bionic main

sudo nano /etc/apt/preferences.d/galera.pref

paste the below line 

    #Prefer Codership repository
    Package: *
    Pin: origin releases.galeracluster.com
    Pin-Priority: 1001

Update the server and install mysql galera 


    sudo apt update
    apt install galera-4 mysql-wsrep-8.0

In Node01 

    sudo nano /etc/mysql/conf.d/galera.cnf

### paste the below and change needed things 


    [mysqld]
    binlog_format=ROW
    default-storage-engine=innodb
    innodb_autoinc_lock_mode=2
    bind-address=0.0.0.0

    #Galera Provider Configuration
    wsrep_on=ON
    wsrep_provider=/usr/lib/galera/libgalera_smm.so

    #Galera Cluster Configuration
    wsrep_cluster_name="Galera_Cluster"
    wsrep_cluster_address="gcomm://192.168.1.102,192.168.1.103"

    #Galera Synchronization Configuration
    wsrep_sst_method=rsync

    #Galera Node Configuration
    wsrep_node_address="192.168.1.102"
    wsrep_node_name=“NODE01”

### One second node paste the same and change like below 

    sudo nano /etc/mysql/conf.d/galera.cnf
. . .
#Galera Node Configuration
    wsrep_node_address="192.168.1.103"
    wsrep_node_name="NODE02"
    . . .

#### Allow Different ports Through Firewall 

    sudo ufw allow 3306,4567,4568,4444/tcp
    sudo ufw allow 4567/udp

### Disable AppArmor by executing the following on each server:

    systemctl stop apparmor
    systemctl disable apparmor

### Enable MySQL to Start on Boot on All Servers

    sudo systemctl enable mysql

#### Run the following on your first server:

    sudo mysqld_bootstrap

#### Bring up the Second Node 

    sudo systemctl start mysql

##### You can see the Cluster Status by executing below on any Node 

    mysql -u root -p -e "SHOW STATUS LIKE 'wsrep_cluster_size'"

##### you should see something like below 

+--------------------+-------+
| Variable_name      | Value |
+--------------------+-------+
| wsrep_cluster_size | 2     |
+--------------------+-------+

----------------------------------------------------



HA Using Corosync and Pacemaker 

My Network is as follows 

    NODE01 192.168.1.102
    NODE02 192.168.1.103

    IP FLOAT  192.168.1.104


***Suggested to Disable Firewall , But if you want , allow All Traffic flow between two nodes using the below command***

##### ON NODE01
####
    ufw allow from 192.168.100.103 to any 

##### ON Node02
####
    ufw allow from 192.168.100.102 to any
 
 
 #### Install Pacemaker and crms execute bellow commands on both servers 
 
     apt-get install pacemaker crmsh
     update-rc.d corosync defaults
    update-rc.d pacemaker defaults


*****Do the steps below on the primary node only:*****
   
    apt-get install haveged
    corosync-keygen
    apt-get remove --purge haveged
    scp /etc/corosync/authkey root@NODE02:/etc/corosync/.

*****Do the steps below on the secondary node only:*****

    chown root: /etc/corosync/authkey
    chmod 400 /etc/corosync/authkey

*****On both nodes do:*****


EDIT CONFIG FILES 

nano  /etc/corosync/corosync.conf

FOR NODE01

    totem {
      version: 2
      cluster_name: lbcluster
      transport: udpu
      interface {
        ringnumber: 0
        bindnetaddr: 192.168.1.102
        broadcast: yes
        mcastport: 5405
      }
    }

    quorum {
      provider: corosync_votequorum
      two_node: 1
    }

    nodelist {
      node {
        ring0_addr: 192.168.1.102
        name: NODE01
        nodeid: 1
      }
      node {
        ring0_addr: 192.168.1.103
        name: NODE02
        nodeid: 2
      }
    }

    logging {
      to_logfile: yes
      logfile: /var/log/corosync/corosync.log
      to_syslog: yes
      timestamp: on
    }



FOR NODE02

    totem {
      version: 2
      cluster_name: lbcluster
      transport: udpu
      interface {
        ringnumber: 0
        bindnetaddr: 192.168.1.103
        broadcast: yes
        mcastport: 5405
      }
    }

    quorum {
      provider: corosync_votequorum
      two_node: 1
    }

    nodelist {
      node {
        ring0_addr: 192.168.1.102
        name: NODE01
        nodeid: 1
      }
      node {
        ring0_addr: 192.168.1.103
        name: NODE02
        nodeid: 2
      }
    }

    logging {
      to_logfile: yes
      logfile: /var/log/corosync/corosync.log
      to_syslog: yes
      timestamp: on
    }


On both servers, create the pcmk file in the Corosync service directory

    mkdir -p /etc/corosync/service.d/

    sudo nano /corosync/service.d/pcmk

    service {
     name: pacemaker
     ver: 1
    }

nano /etc/default/corosync

add the below in the begining 

    START=yes



*****Start corosync and pacemaker*****

    service corosync start
    service pacemaker start


*****On NODE01 run:*****

    crm configure property stonith-enabled=false
    crm configure property no-quorum-policy=ignore

***To add the floating IP resource where MySQL will be listening on run on either one of the nodes the following command after replacing the IP address and netmask with the appropriate values:***

    crm configure primitive floating-ip ocf:heartbeat:IPaddr2 params ip=192.168.1.104 cidr_netmask="24" op monitor interval="30s"

The above will Create HA , and the floating IP will forward to the Active Server 

***TO Create MYSQL specific Resource Execute the Below Command in any node*** (Have Some Bugs in the RA , mysql crashes after enabling monitor)

    crm configure
    
    primitive mysql-resource ocf:heartbeat:mysql params binary="/usr/sbin/mysqld" config="/etc/mysql/my.cnf" op start timeout=60s interval=0 op stop timeout=60s interval=0 op monitor interval=10s
    
    colocation mysql-float-with-ip inf: mysql-resource floating-ip
    
    commit
   
    exit
    
TO Check Status 
    crm status
    
    
    
Installing MYSQL8 Server 

    sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 467B942D3A79BD29
    wget https://dev.mysql.com/get/mysql-apt-config_0.8.12-1_all.deb
    dpkg -i mysql-apt-config_0.8.12-1_all.deb
        select mysql8 and okay 
        
    sudo apt install -f mysql-client=8.0* mysql-community-server=8.0* mysql-server=8.0*
    
****Configure MySQL on Master Node****
    
    nano /etc/mysql/mysql.conf.d/mysqld.cnf

Add the below 

    bind-address = 192.168.1.102
    server-id = 1
    log-bin=master-bin
    
*****Create a Replication User on Master Node*****

    mysql -u root -p
    CREATE USER 'replication_user'@'%' IDENTIFIED BY 'Pranish@1234';
    GRANT REPLICATION SLAVE ON *.* TO 'replication_user'@'%';
    FLUSH PRIVILEGES;
    
Next, verify the Master status with the following command:

    SHOW MASTER STATUS\G
-------------------------------------------------------------    
    mysql> SHOW MASTER STATUS\G
*************************** 1. row ***************************
             File: master-bin.000001
         Position: 851
     Binlog_Do_DB:
 Binlog_Ignore_DB:
Executed_Gtid_Set:
1 row in set (0.00 sec)
-----------------------------------------------

From the above output, note down the mysql-bin.000001 value and the Position ID 851. You will need both to set up a slave server.


*****On Slave Server  edit the Config file as Master and only change the bind IP to the original IP and server-id = 2*****


Login to Mysql in slave server
    
    mysql -u root -p 
    STOP SLAVE;
    CHANGE MASTER TO MASTER_HOST ='192.168.1.102', MASTER_USER ='replication_user', MASTER_PASSWORD ='Pranish@1234', MASTER_LOG_FILE = 'master-bin.000001', MASTER_LOG_POS = 851;
    START SLAVE;
    
    
Test the Master-Slave Replication

At this point, Master-Slave replication is configured. Now it's time to test whether the replication works or not.

To do so, we will create a database on the Master Node and verify whether it will be replicated on the Slave Node.

First, log in to the MySQL on the Master Node:

    mysql -u root -p
    CREATE DATABASE newdb;
    EXIT;
    
Next, log in to the MySQL on the Slave Node:

    mysql -u root -p
    SHOW DATABASES;

If the DB is replicated , we are done here 











