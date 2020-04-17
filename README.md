# local-yum-mariadb
Local YUM Repository and a MariaDB Cluster

In this example I will set up a local YUM repository in an AWS instance and use it to install/configure a 2-node MariaDB cluster.

# Prepare the AWS Instance

## Launch the Instance

In the AWS console I will select the Red Hat Enterprise Linux 8 64-bit, t2.micro (Free tier eligible) instance. 

Additional Configuration:
* Under the _Add Tags_ tab with create (i.e., Add Tag) the _Name_, _MariaDB_ key/value pair.
* Under the _Configure Security Group_ tab we will add the _HTTP_ (or _HTTPS_ if configuring SSL) rule.

Once configured, we will _Launch_ the instance selecting an existing _key pair_ certificate I created earlier.

## Connect to and Set up the Instance

Once the instance is up an running (as verified in the AWS console) we are ready to now connect through an SSH-enabled terminal to your instance by running the command:

```
ssh -i /<path-to-pem>/<mycert>.pem ec2-user@ec2-xxx-xxx-xxx-xxx.compute-1.amazonaws.com
```

The Public DNS above can be obtained from the instance _Description_ (and it will change every time the instance is re-started). I will from here on generically reference it using 'xxx' for each 8-bit field.

Create and configure your user id by running the following commands (_passwd_ set optional) and log in using your user:

```
sudo su
useradd <your_user>
passwd <your_user>
mkdir /home/<your_user>/.ssh
cp /home/ec2-user/.ssh/authorized_keys /home/<your_user>/.ssh/
chown -R <your_user>.<your_user> /home/<your_user>/.ssh
```

Grant _root_ access to your user by running as root the _visudo_ command and adding the line below the _ec2-user_ line:
```
<your_user>        ALL=(ALL)       NOPASSWD: ALL
```

# Set Up NGINX/YUM

## Install/Configure NGINX

Run the following commands as root user (last to verify that is is up an running):

```
yum install nginx
systemctl start nginx
systemctl enable nginx
systemctl status nginx
```

You can now verify that it is running by typing your instance public DNS in the browser (i.e., http://ec2-xxx-xxx-xxx-xxx.compute-1.amazonaws.com/) and verifying that you get the "Welcome to nginx on Red Hat Enterprise Linux!" page (if it fails to start, the _journalctl -xe_ command, as indicated in the output can help troubleshoot the cause)

## Create the Local YUM/DNF Repository

While I have used _yum_ for this installation, as of RHEL 8 it is being replaced by the _dnf_ installer (either one can be used). Run the following commands as _root_ user to support the YUM repository creation:

```
yum install createrepo  yum-utils
mkdir -p /var/www/html/repos/mariadb
```

and edit a newly created _/etc/yum.repos.d/mariadb.repo_ file to include the following:

```
[mariadb]
name=mariadb
baseurl=file:///var/www/html/repos/mariadb
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-MariaDB
gpgcheck=1
```

The _gpgcheck_ can be disabled initially but, if you want to set this up as I did, you can run the below commands as _root_:

```
cd /etc/pki/rpm-gpg
wget https://yum.mariadb.org/RPM-GPG-KEY-MariaDB
```


## Configure NGINX to Include the Repository

Edit the newly created _/etc/nginx/conf.d/repos.conf_ to include the following (this configuration should automatically be imported by the _/etc/nginx/nginx.conf_ main configuration file):

```
server {
        listen   80;
        server_name  ec2-xxx-xxx-xxx-xxx.compute-1.amazonaws.com; #change to your real domain
        root   /var/www/html/repos;
        location / {
                autoindex on;   #enable listing of directory index
        }
}
```


## Download the RPMS and Create the Repo

Run the following commands as _root_ to download the specific version 10.4.12 MariaDB (and non-MariaDB) rpms compatible with RHEL-8 (last _rm_ includes any non-rpm files). For this exercise we will only install the Server/Client RPMS and the _galera_ dependency:

```
cd /var/www/html/repos/mariadb/
yum install wget
wget -r --no-parent --no-directories --accept-regex 'MariaDB.*10.4.12.*.rpm' http://mariadb.mirror.globo.tech//mariadb-10.4.12/yum/rhel8-amd64/rpms/
wget -r --no-parent --no-directories --reject-regex 'MariaDB.*.rpm' http://mariadb.mirror.globo.tech//mariadb-10.4.12/yum/rhel8-amd64/rpms/
rm index.html
createrepo .
```

Note: If you add RPMS to the repository you can run _yum clean all_ followed _createrepo --update /var/www/html/repos/mariadb_


## Setup File Listing and Restart NGINX

Run the below commands (first one will enable the directory files listing/access):
```
restorecon -R /var/www/html/repos
service restart nginx
```

Visiting our url http://ec2-xxx-xxx-xxx-xxx.compute-1.amazonaws.com/ will now show the _mariadb_ index. Clicking on the _/mariadb_ folder will display the list of rpms as shown below.

![mariadb index](images/mariadb_index.png)

## Test Accessing the Repo Locally

As _root_ user (or sudo) run _yum list | grep MariaDB_ to get a listing like below:

```
MariaDB-backup.x86_64                                10.4.12-1.el8                                     mariadb                         
MariaDB-backup-debuginfo.x86_64                      10.4.12-1.el8                                     mariadb                         
MariaDB-client.x86_64                                10.4.12-1.el8                                     mariadb                         
MariaDB-client-debuginfo.x86_64                      10.4.12-1.el8                                     mariadb                         
MariaDB-common.x86_64                                10.4.12-1.el8                                     mariadb                         
MariaDB-common-debuginfo.x86_64                      10.4.12-1.el8                                     mariadb                         
MariaDB-connect-engine.x86_64                        10.4.12-1.el8                                     mariadb                         
MariaDB-connect-engine-debuginfo.x86_64              10.4.12-1.el8                                     mariadb                         
MariaDB-cracklib-password-check.x86_64               10.4.12-1.el8                                     mariadb                         
MariaDB-cracklib-password-check-debuginfo.x86_64     10.4.12-1.el8                                     mariadb                         
MariaDB-devel-debuginfo.x86_64                       10.4.12-1.el8                                     mariadb                         
MariaDB-gssapi-server.x86_64                         10.4.12-1.el8                                     mariadb                         
MariaDB-gssapi-server-debuginfo.x86_64               10.4.12-1.el8                                     mariadb                         
MariaDB-rocksdb-engine.x86_64                        10.4.12-1.el8                                     mariadb                         
MariaDB-rocksdb-engine-debuginfo.x86_64              10.4.12-1.el8                                     mariadb                         
MariaDB-server-debuginfo.x86_64                      10.4.12-1.el8                                     mariadb                         
MariaDB-shared.x86_64                                10.4.12-1.el8                                     mariadb                         
MariaDB-shared-debuginfo.x86_64                      10.4.12-1.el8                                     mariadb                         
MariaDB-test-debuginfo.x86_64                        10.4.12-1.el8                                     mariadb                         
MariaDB-tokudb-engine.x86_64                         10.4.12-1.el8                                     mariadb                         
MariaDB-tokudb-engine-debuginfo.x86_64               10.4.12-1.el8                                     mariadb
```

Note that all RPMS shown in the file listing screenshot are not visible (for instance, MariaDB-server is missing). This is because RHEL 8 already comes with its own conflicting RPM distribution named using a lowercase naming convention:

```
[root@ip-xxx-xxx-xxx-xxx ~]# yum list | grep -i mariadb-server
MariaDB-server-debuginfo.x86_64                      10.4.12-1.el8                                     mariadb                         
mariadb-server.x86_64                                3:10.3.17-1.module+el8.1.0+3974+90eded84          rhel-8-appstream-rhui-rpms      
mariadb-server-galera.x86_64                         3:10.3.17-1.module+el8.1.0+3974+90eded84          rhel-8-appstream-rhui-rpms      
mariadb-server-utils.x86_64                          3:10.3.17-1.module+el8.1.0+3974+90eded84          rhel-8-appstream-rhui-rpms
```

If I temporarily disable that repo when performing the listing, the version I want install is now visible:
```
[root@ip-172-31-39-159 ~]# yum list --disablerepo=rhel-8-appstream-rhui-rpms | grep -i mariadb-server
MariaDB-server.x86_64                            10.4.12-1.el8                               mariadb                        
MariaDB-server-debuginfo.x86_64                  10.4.12-1.el8                               mariadb
```

## Test Accessing the Repo Remotely

As _root_ user, create the _/etc/yum.repos.d/mariadb.repo_ with contents below (for now, disabling _gpgcheck_ though it can easily be enabled as shown earlier) in a remote VM:
```
[mariadb]
name=mariadb
baseurl=http://ec2-xxx-xxx-xxx-xxx.compute-1.amazonaws.com/mariadb
enabled=1
gpgcheck=0
```

Assuming no access issues, the yum listing command above should produce the same output as the local run.

# MariaDB Setup

## Install the MariaDB Server/Client on the Master VM

As _root_ user (or using _sudo_) run the following commands:
```
yum clean metadata
yum install galera-4
yum install --disablerepo=rhel-8-appstream-rhui-rpms MariaDB-server MariaDB-client
```

Once installation is complete, start the server by running _service mariadb start_ (and _service mariadb status_ to verify that it is running)

Access the database by running _mariadb -u root_ and once in, run the test command shown below:
```
[root@ip-xxx-xxx-xxx-xxx ~]# mariadb -u root
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 9
Server version: 10.4.12-MariaDB MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| test               |
+--------------------+
4 rows in set (0.000 sec)

MariaDB [(none)]> 
```
### Secure Installation

Set Up the secure installation by running the command _mysql_secure_installation_ and entering information as shown below:
```
[root@ip-xxx-xxx-xxx-xxx ~]# mysql_secure_installation

NOTE: RUNNING ALL PARTS OF THIS SCRIPT IS RECOMMENDED FOR ALL MariaDB
      SERVERS IN PRODUCTION USE!  PLEASE READ EACH STEP CAREFULLY!

In order to log into MariaDB to secure it, we'll need the current
password for the root user. If you've just installed MariaDB, and
haven't set the root password yet, you should just press enter here.

Enter current password for root (enter for none): 
OK, successfully used password, moving on...

Setting the root password or using the unix_socket ensures that nobody
can log into the MariaDB root user without the proper authorisation.

You already have your root account protected, so you can safely answer 'n'.

Switch to unix_socket authentication [Y/n] n
 ... skipping.

You already have your root account protected, so you can safely answer 'n'.

Change the root password? [Y/n] n
 ... skipping.

By default, a MariaDB installation has an anonymous user, allowing anyone
to log into MariaDB without having to have a user account created for
them.  This is intended only for testing, and to make the installation
go a bit smoother.  You should remove them before moving into a
production environment.

Remove anonymous users? [Y/n] Y
 ... Success!

Normally, root should only be allowed to connect from 'localhost'.  This
ensures that someone cannot guess at the root password from the network.

Disallow root login remotely? [Y/n] Y
 ... Success!

By default, MariaDB comes with a database named 'test' that anyone can
access.  This is also intended only for testing, and should be removed
before moving into a production environment.

Remove test database and access to it? [Y/n] Y
 - Dropping test database...
 ... Success!
 - Removing privileges on test database...
 ... Success!

Reloading the privilege tables will ensure that all changes made so far
will take effect immediately.

Reload privilege tables now? [Y/n] Y
 ... Success!

Cleaning up...

All done!  If you've completed all of the above steps, your MariaDB
installation should now be secure.

Thanks for using MariaDB!
```

## Configure the Database

As _root_ user (or _sudo_) run below commands:
```
mkdir /var/log/mariadb
chown -R mysql. /var/log/mariadb
cp /etc/my.cnf.d/server.cnf  /etc/my.cnf.d/server.cnf.orig
```

Edit the _/etc/my.cnf.d/server.cnf_ file to include the below configuration and run _service mariadb restart_ after:
```
[mysqld]
server-id=1
log_bin=/var/log/mariadb/mariadb-bin.log
```

## Configure the Database

Run _mysql -u root -p_ (enter the password set during the secure installation) and run commands as shown below:
```
[root@ip-***-***-***-*** yum.repos.d]# mysql -u root -p
Enter password: 
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 9
Server version: 10.4.12-MariaDB-log MariaDB Server

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]> GRANT REPLICATION SLAVE ON *.* TO replication_user IDENTIFIED BY '********';
Query OK, 0 rows affected (0.001 sec)

MariaDB [(none)]> FLUSH PRIVILEGES;
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]> FLUSH TABLES WITH READ LOCK;
Query OK, 0 rows affected (0.002 sec)

MariaDB [(none)]> SHOW MASTER STATUS;
+--------------------+----------+--------------+------------------+
| File               | Position | Binlog_Do_DB | Binlog_Ignore_DB |
+--------------------+----------+--------------+------------------+
| mariadb-bin.000001 |      647 |              |                  |
+--------------------+----------+--------------+------------------+
1 row in set (0.000 sec)

MariaDB [(none)]> UNLOCK TABLES;
Query OK, 0 rows affected (0.000 sec)

MariaDB [(none)]> 
```

and then run _service mariadb restart_

## Install the MariaDB Server/Client on a Slave VM

Before starting the installation we will create another RHEL 8 AWS instance (same flavor) and give it the tag _Name MariaDB-Slave_. No need to assign any security groups (other than leave the default SSH access).

Once up, we will connect to and set up the instance like we did earlier with the YUM/Master VM.


## References:

* https://mariadb.com/kb/en/rpm/
* https://mariadb.com/kb/en/yum/
