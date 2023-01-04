# Load Balancing RDS read traffic using Route53 Weighted Routing via AWSCLI

****This is a demostration of load Balancing MySQL RDS read traffic using Route53 Weighted Routing policy.****<br />
<br />
<br />

***Amazon RDS*** is a Relational Database Cloud Service, which minimizes relational database management by automation.<br />
It can create multiple instances for high availability and failovers.<br />
Amazon RDS supports PostgreSQL, MySQL, Maria DB, Oracle, SQL Server, and Amazon Aurora.<br />
RDS load balancing is a critical feature that enables organizations to authenticate and route traffic to available servers in order to maximize the efficiency of the network. When a server is down, the traffic is routed to other servers, ensuring there is no single point of failure.
<br />
<br />
<br />

This task basically requires following resources:

- ***IAM user having AmazonRDSFullAccess, AmazonRoute53FullAccess and AmazonEC2ReadOnlyAccess***
- ***Machine with AWSCLI installed.***
- ***One master RDS with Mysql***
- ***2 RDS read replicas***
- ***Route53 Hosted zone***
 <br />
  <br />
<br />

****KEY POINTS:****<br />
***
Load balacing will be implemented using readreplicas from an RDS instance and Route53 Hosted zone.<br />
We can reduce the load on your primary RDS instance by routing read queries from your applications to the read replica. <br />
Using read replicas, we can elastically scale out beyond the capacity constraints of a single RDS DB instance for read-heavy database workloads.<br />
We can use Amazon Route 53 weighted record sets to distribute requests across your read replicas. <br />
Within a Route 53 hosted zone, we can create individual record sets for each DNS endpoint associated with your read replicas. <br />
Then, we can assign give them the same weight, and direct requests to the endpoint of the record set.
<br />I'm demonstrating all these via AWSCLI method.
<br />

<br />
<br />
<br />
I have logged in to AWSCLI using the following command with an IAM user using it's Access Key ID and Secret Access Key as login credentials.
```sh 
[root@ip-172-31-47-196 ~]# aws configure
AWS Access Key ID [None]: AKIA54GMXHW7CXJLJVFK
AWS Secret Access Key [None]: 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
Default region name [None]: ap-south-1
Default output format [None]: json
[root@ip-172-31-47-196 ~]#
```


<br />
#### 1. Creating a Primary RDS instance.

Here I'm creating a MysQL RDS instance with following main options :<br />
a) RDS instance type: t2.micro 
b) DB engine with version: MySQL 5.7.40  
c) A security group to associate with this DB instance
