# ProxySQL - 101 Tutorial

This tutorial will help you walk through various aspects of setting up and use a basic ProxySQL installation. Enjoy!

[TOC]

## Creating the tutorial environment

This tutorial uses Virtualbox and Vagrant. Follow this steps to get a setup:

### Install VirtualBox. 

Version 5.1.18 works. Download Virtualbox from [here](https://www.virtualbox.org/wiki/Downloads).

### Install Vagrant. 

Version 2.0.1 works. Download Vagrant from [here](http://vagrantup.com/).

### Install Vagrant plugin hostmanager

This plugin will take care of dealing with the /etc/hosts file so we can use hostnames instead of IPs. https://github.com/devopsgroup-io/vagrant-hostmanager

To install it, just run:

```bash
vagrant plugin install vagrant-hostmanager
```

Create the environment

For this tutorial, the env consist of an "App" node (where ProxySQL will run) and 3 MySQLs, which will become 1 Master and 2 Slaves.

Clone the repo

Run 

```bash
vagrant up; vagrant hostmanager;
```

The whole process takes a while the first time (around 20 minutes) so go grab some coffee and be back later.

Continue when the environment is done.

## Install ProxySQL from the Percona repo

```bash
sudo yum install -y http://www.percona.com/downloads/percona-release/redhat/0.1-4/percona-release-0.1-4.noarch.rpm
sudo yum install -y proxysql.x86_64 Percona-Server-client-56.x86_64
service proxysql start 
```

## Configure the Master/Slaves

On **mysql1**:

```mysql
grant replication slave, replication client on *.* to repl@'192.168.70.%' identified by 'repl';
```

On **mysql2** and **mysql3**:

NOTE: Please make sure the server_id is different on each server. The provision playbooks is not working well for the moment :) Fix coming soon.

```Mysql
change master to master_host = 'mysql1', master_user = 'repl', master_password='repl', master_log_file = 'mysql-bin.000001', master_log_pos = 330; start slave;
```

## Accessing the ProxySQL admin

The Admin can be accessed using the MySQL Cllient as if it was a regular MySQL installation, you just need to use the port 6032. The default credential values are admin/admin (user/pass):

```mysql
mysql -u admin -padmin -h 127.0.0.1 -P6032 --prompt='Admin> '
```

You should be able to see this databases:

```mysql
Admin> show databases;
+-----+---------+-------------------------------+
| seq | name    | file                          |
+-----+---------+-------------------------------+
| 0   | main    |                               |
| 2   | disk    | /var/lib/proxysql/proxysql.db |
| 3   | stats   |                               |
| 4   | monitor |                               |
+-----+---------+-------------------------------+
4 rows in set (0.00 sec)
```

Let's configure everything for the Master/Slave topology

## Configuring ProxySQL

We need to make sure that ProxySQL is aware of this things:

- The MySQL servers
- The user that can connect to the servers (both the monit one and the "queries" one)
- Who is the Master and who is/are the slave(s)

Let's start with the users

### Setting the Monit user

ProxySQL perform the monitoring checks using this user. This is needed for things like checking the value of the read_only variable.

This user requires the REPLICATION CLIENT grant for now. Just create it on the master and let the replication do the rest:

```mysql
GRANT REPLICATION CLIENT ON *.* TO repl@'app' IDENTIFIED BY 'repl';
```

Now, on the admin side, update the values of the variables mysql-monitor_username and mysql-monitor_password:

```mysql
SET mysql-monitor_username='repl';
SET mysql-monitor_password='repl';
```

Is this enough? 
NO!

Why?

Because the **Multi-Layer configuration** model of ProxySQL. Make sure you understand that: https://github.com/sysown/proxysql/wiki/Multi-layer-configuration-system

Okay, what is missing? Moving the configuration from memory to the Runtime layer and Persist the configuration on disk. To do that, execute:

```MySQL
LOAD MYSQL VARIABLES TO RUNTIME;
SAVE MYSQL VARIABLES TO DISK;
```

Now the change is really done.

### Setting the backend MySQL user

ProxySQL needs the username and the password of the user that can connect to the backend servers (the MySQL dbs). We need to provide that, but before we need to create the user in the dbs. 

In the Master, create the user:

```Mysql
GRANT ALL PRIVILEGES ON *.* TO proxysql@'%' IDENTIFIED BY 'proxysql';
```

And now let ProxySQL know the data:

```MySQL
INSERT INTO mysql_users (username,password, default_hostgroup) VALUES ('proxysql','proxysql',1);
```

Remember to load it to Runtime. To do that, execute:

```mysql
LOAD MYSQL USERS TO RUNTIME;
SAVE MYSQL USERS TO DISK;
```

Now the config is actually on runtime.

### Setting the servers

Server info is stored in the **mysql_servers** table. The most basi amount of info required to insert is the info that describes:

- The server hostname
- The MySQL port
- And the hostgroup where the server belongs. And this is where the concep of ***Replication Hostgroup*** becomes relevant, so let's define it

#### Replication hostgroup

You can add a server to all the hostgroups that you want. This will help on the query routing (eventually the hostgroup is the destination of the query rlues) and to have a controlled load distribution, among other things.

However, ProxySQL in an effort to simplify things have the special "replication hostgroup" type which is nothing that a way to say which hostgroup holds the master and which one holds the slaves. 

How is this different to a regular hostgroup? Simply: The task of moving servers between hostgroups becomes an automatic operation and depends on only one thing: the value of the **read_only** variable. 

If a server has "read_only = 1" it will be part of the reader_hostgroup. Otherwise, is the master and is part of the writer_hostgroup. This means that you need to be extra careful with this variable. A good practice will be to enforce read_only = 1 on the my.cnf file and just change it on the fly in the Master. 

To define the replication hostgroup, just do a insert to the mysql_replication_hostgroups table. For this tutorial, the insert is:

```mysql
INSERT INTO mysql_replication_hostgroups (writer_hostgroup, reader_hostgroup) VALUES(1,2);
```

### Adding the MySQL servers to ProxySQL

Let's add the server now:

```mysql
INSERT INTO mysql_servers (hostgroup_id, hostname, port, weight) VALUES (1,'mysql1',3306,1);
INSERT INTO mysql_servers (hostgroup_id, hostname, port, weight) VALUES (2,'mysql2',3306,1);
INSERT INTO mysql_servers (hostgroup_id, hostname, port, weight) VALUES (2,'mysql3',3306,1);
```

Remember to move the configuration to runtime:

```MySQL
LOAD MYSQL SERVERS TO RUNTIME;
SAVE MYSQL SERVERS TO DISK;
```

