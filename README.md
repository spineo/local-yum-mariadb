# local-yum-mariadb
Local YUM Repository including MariaDB

In this example I will set up a local YUM repository in an AWS instance and use it to Install/Configure MariaDB.

## Launch the Instance

In the AWS console I will select the Red Hat Enterprise Linux 8 64-bit, t2.micro (Free tier eligible) instance. 

Additional Configuration:
* Under the _Add Tags_ tab with create (i.e., Add Tag) the _Name_, _MariaDB_ key/value pair.
* Under the _Configure Security Group_ tab we will add both the _HTTP_ and _HTTPS_ rules.

Once configured, we will _Launch_ the instance selecting an existing _key pair_ certificated I created earlier.

## Connect to the Instance

Once the instance is up an running (as verified in the AWS console) we are ready to now connect through an SSH-enabled terminal to your instance by running the command:

```
ssh -i /<path-to-pem>/<mycert>.pem ec2-user@ec2-xxx-xxx-xxx-xxx.compute-1.amazonaws.com
```

The Public DNS above can be obtained from the instance _Description_ (and it will change every time the instance is stopped/started)
