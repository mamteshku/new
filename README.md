# Deploy Your Node.JS Server Using MySQL Database on AWS 
# I. Set up Security Groups

In the EC2 Console, “Security Groups”:

1. For Load Balancer — Load Balancer

    Type: Custom TCP, Protocol: TCP, Port Range: 3000, Source: Anywhere

2. For App Server Instances — Web Tier

    Type: Custom TCP, Protocol: TCP, Port Range: 3000, Source: Load Balancer Security Group ID
    Type: Custom TCP, Protocol: TCP, Port Range: 3000, Source: My IP
    Type: SSH, Protocol: TCP, Port Range: 22, Source: My IP

3. For Database — DB Tier

    Type: MYSQL/Aurora, Protocol: TCP, Port Range: 3306, Source: My IP
    Type: MYSQL/Aurora, Protocol: TCP, Port Range: 3306, Source: Web Tier Security Group ID

# II. Set up an app server in the web tier

    Create an EC2 instance and bootstrap it

use web tier security group, specify the region

Follow this:

Enter the following lines of bash script in the User Data to

    Install Node.js and MySQL server on the EC2 instance
    clone the code for my server
    install all the node dependencies
    start the server app.js

“sudo” in first 2 lines may not be needed

#!/bin/bash
yum update -y
curl --silent --location https://rpm.nodesource.com/setup_6.x | sudo bash -
sudo yum -y install nodejs mysql-server git -y

Also, add tags to help yourself manage your instances.

    name: AppServer
    project: p3
    env: test/prod

# 2. Connect to it via SSH

Follow this and after typing the following line, hit “yes”, then the instance is connected successful

cd Documents/S3/EDISS/PROJ
ssh -i "./Tina-EDISS.pem" ec2-user@[public DNS/public IP]

You will see something like this but it’s correct. Just ignore it.

The authenticity of host 'ec2-198-51-100-1.compute-1.amazonaws.com (10.254.142.33)' can't be established.

# 3. Deploy the code and Launch the server

git clone https://github.com/TinaHongBu/PROJ.git
cd PROJ
npm install
node app

# 4. When you have to use a bigger instance, you have to copy file from local repo to EC2

scp -i /Users/Tina/Documents/S3/EDISS/PROJ_local/Tina-EDISS.pem /Users/Tina/Documents/S3/EDISS/PROJ/test/ ec2-user@34.230.41.147:~/inputData

# III. Create more servers in the web tier using an AMI
1. Create an AMI of the current instance

Select the current running instance, create AMI.
2. Create multiple instances with the AMI

Select the AMI at the left panel, create a new instance. Specify the availability zone to be different from the first instance, but within the AZ of your load balancer.
3. Launch the Server in all the instances

cd PROJ
npm install
node app

# IV. Set up an Classic Load Balancer
1. Create a Load Balancer

In the EC2 console, “load balancer”:

Create Load Balancer →

    Classic Load Balancer →
    Listeners: HTTP port 3000, Availability Zones: at least chose 2 →
    Security Groups: Group for Load Balancer →
    Ping Port: 3000, Ping Path: /health, Response Timeout:4, Interval: 5 →
    Add EC2 Instances later
    Routing: Create a new target group →
    Register instances to the target group
    
    Don’t set it to be sticky, because

    it sends requests from the same client to the same server the whole time so when the instance got killed, all following requests from its clients will still be routed to it and fail. And,
    it will cause unbalance load among instances as the load balancer balances requests based on the traffic but not server load. So it could happen that one server gets a lot of big requests.

Connection draining

# 2. Set up the ping target response in your code

The health ping target will be: http:3000/health

app.get('/health', function(req, res){
    res.send({"message": "Instance is alive"});
}); // just return anythin with 200

# 3. Check if Load Balancer is attached to all instances

Everything should be in service if set up correctly.

Debug using this.

curl -s -k -o /dev/null -v http://private-IP-address-of-the-instance:3000/health

# V. Set up an Auto Scaling Group (ASG)
1. Create a Launch Configuration

In the EC2 console, choose Launch Configuration

    Create from existing AMI
    User Data

#!/bin/sh

yum update -y

git clone https://github.com/TinaHongBu/PROJ.git

    Use app server security group
    
# 2. Create a Scaling Group

In the EC2 console, choose Auto Scaling Group

    Add subnets →
    
# 3. Using Application Load Balancer with Auto Scaling Group

Create a new target group, in the “target” tag choose all instances launched by the auto scaling group. Add to register, then save.

Go to load balancer, add rule.
V(1) Setup MySQL Database using the RDS service

# 1. Setup a MySQL Database Instance

In the RDS dashboard,

Step 1: Select Engine

    MySQL →

Step 2: Production?

    Dev/Test (Production version has 2 database instances in 2 different availability zones with one being primary and one being the backup, and synchronization is handled by Amazon) →

Step 3: Specify DB Details

    DB Instance Class chose db.t2.micro — 1 vCPU, 1 GiB RAM,
    Multi-AZ Deployment: No,
    DB Instance Identifier: EDISS-P3,
    Master Username: admin,
    Master Password: adminadmin →

Step 4: Configure Advanced Settings

    Availability Zone: Make sure same as your primary instances to decrease latency
    VPC Security Groups: DB tier

# 2. Connect App Servers to the DB

Copy your DB endpoint: ediss-p3.c6xzjopfczba.us-east-1.rds.amazonaws.com:3306

Delete the port number, do

mysql -h ediss-p3.c6xzjopfczba.us-east-1.rds.amazonaws.com -P 3306 -u admin -p

adminadmin (the password)

mysql -h ediss-p4.c6xzjopfczba.us-east-1.rds.amazonaws.com -P 3306 -u admin -p

to enter the database.

mysql> show databases;

+--------------------+

| Database |

+--------------------+

| information_schema |

| EDISS |

| innodb |

| mysql |

| performance_schema |

| sys |

+--------------------+

6 rows in set (0.00 sec)

mysql> use EDISS;

Database changed

show tables;

select * from p2_prdct;

select * from p2_user;

select * from p2_usertype;

shows that the EDISS database is successfully created already.
V(2) Or, Hold MySQL Server on Ec2

The steps are taken from here.
1. Install MySQL

Install the MySQL database through the CentOS package manager (yum):

sudo yum install mysql-server
2. Start mysql

sudo /sbin/service mysqld start
3. Enable chkconfig on MySQL

sudo chkconfig mysqld on
4. Launch the mysql shell and enter it as the root user:

/usr/bin/mysql -u root -p

Hit enter because no password is set for root user so far.

Now set root password

SET PASSWORD FOR 'root'@'localhost' = PASSWORD('root');
5. Launch MySQL

brew services restart mysql

sudo mysql -u root -p

// enter password for root user
6. Create database, table and insert default users in the table

Hopefully last line will generate duplicate entry error
7. Exit mysql from the terminal

exit
8. Or stop mysql when you need to

sudo /sbin/service mysqld stop
VI. System Functionality Tests

If you use Postman to test requests, type in http://“your DNS address:”“Port number your system is listening to/” “end points”

For example: http://ec2-34-204-68-2.compute-1.amazonaws.com:3000/login

and hit send to see if you get your results back.

Or to use Artillery, in the test file config session modify to “target”: “http://ec2-34-204-68-2.compute-1.amazonaws.com:3000", and run this in terminal: artillery run test.json.
VI. System Availability Tests

Kill instance and see if the traffic gets routed to other nodes.

Add elastic load balancing.
VII. System Performance Tests

Hold Elasticsearch locally

    Install Elasticsearch

brew install elasticsearch

    Start the service

brew services start elasticsearch

    Set Elasticsearch to be accessible from the network

Move to Directory:

elasticsearch-5.5.1/config

vim elasticsearch.yml

## put

network.host: 0.0.0.0

Restart elasticsearch

brew services restart elasticsearch

Hold Elasticsearch on AWS EC2 instance

https://www.elastic.co/blog/running-elasticsearch-on-aws

sudo rpm -i https://download.elastic.co/elasticsearch/release/org/elasticsearch/distribution/rpm/elasticsearch/2.3.3/elasticsearch-2.3.3.rpm

sudo chkconfig --add elasticsearch

cd /usr/share/elasticsearch/

sudo bin/plugin install cloud-aws -y

sudo service elasticsearch start

curl localhost:9200/_cluster/health?pretty

http://pavelpolyakov.com/2014/08/14/elasticsearch-cluster-on-aws-part-2-configuring-the-elasticsearch/
