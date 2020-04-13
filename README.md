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

The Public DNS above can be obtained from the instance _Description_ (and it will change every time the instance is stopped/started)

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

