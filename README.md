# local-yum-mariadb
Local YUM Repository including MariaDB

In this example I will set up a local YUM repository in an AWS instance and use it to Install/Configure MariaDB.

## Launch the Instance

In the AWS console I will select the Red Hat Enterprise Linux 8 64-bit, t2.micro (Free tier eligible) instance. 

Additional Configuration:
* Under the _Add Tags_ tab with create (i.e., Add Tag) the _Name_, _MariaDB_ key/value pair.
* Under the _Configure Security Group_ tab we will add both the _HTTP_ and _HTTPS_ rules.

Once configured, we will _Launch_ the instance selecting an existing _key pair_ certificated I created earlier.

## Connect to and Set up the Instance

Once the instance is up an running (as verified in the AWS console) we are ready to now connect through an SSH-enabled terminal to your instance by running the command:

```
ssh -i /<path-to-pem>/<mycert>.pem ec2-user@ec2-xxx-xxx-xxx-xxx.compute-1.amazonaws.com
```

The Public DNS above can be obtained from the instance _Description_ (and it will change every time the instance is stopped/started). I will from here on generically reference it using 'xxx' for each 8-bit field.

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

## Set Up NGINX

Run the following commands as root user (last to verify that is is up an running):

```
yum install nginx
systemctl start nginx
systemctl enable nginx
systemctl status nginx
```

You can now verify that it is running by typing your instance public DNS in the browser (i.e., http://ec2-xxx-xxx-xxx-xxx.compute-1.amazonaws.com/) and verifying that you get the "Welcome to nginx on Red Hat Enterprise Linux!" page.

## Create the Local YUM Repository

Run the following commands as root user to create the YUM repository we will use in this example:

```
yum install createrepo  yum-utils
mkdir -p /var/www/html/repos/mariadb
touch /var/www/html/repos/mariadb/comps.xml
createrepo -g comps.xml /var/www/html/repos/mariadb
```

and edit a newly created _/etc/yum.repos.d/mariadb.repo_ file to include the following:

```
[mariadb]
 name=mariadb
 baseurl=http://ec2-54-198-51-112.compute-1.amazonaws.com/mariadb
 enabled=1
 gpgcheck=1
```

## Configure NGINX to Include the Repository

Edit the newly created _/etc/nginx/conf.d/repos.conf_ to include the following:

```
server {
        listen   80;
        server_name  ec2-xxx-xxx-xxx-xxx.compute-1.amazonaws.com; #change  test.lab to your real domain
        root   /var/www/html/repos;
        location / {
                index  index.php index.html index.htm;
                autoindex on;   #enable listing of directory index
        }
}
```

and run _service restart nginx_

Visiting our url http://ec2-xxx-xxx-xxx-xxx.compute-1.amazonaws.com/ will now show the _mariadb_ index.

## Download the RPMS

Run the following commands as _root_ to download the specific version 10.4.12 MariaDB rpms compatible with RHEL-8 (the second _wget_ will download any additional non-MariaDB rpms):

cd /var/www/html/repos/mariadb/
mkdir rpms
cd rpms
yum install wget
wget -r --no-parent --no-directories --accept-regex 'MariaDB.*10.4.12.*.rpm' http://mariadb.mirror.globo.tech//mariadb-10.4.12/yum/rhel8-amd64/rpms/
wget -r --no-parent --no-directories --reject-regex 'MariaDB.*.rpm' http://mariadb.mirror.globo.tech//mariadb-10.4.12/yum/rhel8-amd64/rpms/
rm index.html


## References:

* https://mariadb.com/kb/en/rpm/
