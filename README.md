# Load Balancing RDS read traffic using Route53 Weighted Routing via AWSCLI

*This is a demostration of load Balancing MySQL RDS read traffic using Route53 Weighted Routing policy.*<br />
<br />
<br />

****Amazon RDS**** is a Relational Database Cloud Service, which minimizes relational database management by automation.<br />
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
Load balacing on RDS instances are implemented using readreplicas from an RDS instance and Route53 Hosted zone routing policy.<br />
We can reduce the load on your primary RDS instance by routing read queries from applications to the read replica. <br />
Using read replicas, we can elastically scale out beyond the capacity constraints of a single RDS DB instance for read-heavy database workloads.<br />
We can use Amazon Route 53 weighted record sets to distribute requests across your read replicas. <br />
Within a Route 53 hosted zone, we can create individual record sets for each DNS endpoint associated with the read replicas. <br />
Then, we can assign them the same weight, and direct requests to the endpoint of the record set.
<br />
![Untitled Diagram drawio](https://user-images.githubusercontent.com/117455666/212580676-5725deb1-695d-4cbe-9826-c94db8453a3c.png)
<br />I'm demonstrating all these via AWSCLI method.
***
<br />
<br />
<br />
I have logged in to AWSCLI using the following command with an IAM user using it's Access Key ID and Secret Access Key as login credentials.<br />

```sh 
[root@ip-172-31-47-196 ~]# aws configure
AWS Access Key ID [None]: xxxxxxxxxxxxxxxxxxxxxxxx
AWS Secret Access Key [None]:xxxxxxxxxxxxxxxxxxxxx
Default region name [None]: ap-south-1
Default output format [None]: json
[root@ip-172-31-47-196 ~]#
```


<br />

### 1. Creating a Master  RDS instance.

Here I'm creating a MysQL RDS instance with following main options :<br />
***a) RDS instance type:*** <br />
*We choose free tier, t2.micro*
<br/>
***b) DB engine with version:*** <br />
*Any database server can be used. Here, MySQL 5.7.40 is used.*
<br/>
***c) A security group:*** <br />
*We can use any existing security group allowing access to your IP address in such a way that we can access the Amazon RDS instances.*
*A security group ID can be obtained from following command:*
```sh
[root@ip-172-31-47-196 ~]# aws ec2 describe-security-groups  --query "SecurityGroups[*].{Name:GroupName,ID:GroupId}"
```
***d) RDS login credentials:*** <br/>
*Username for the master user and password to access the RDS instance from any AWS resources.*
<br/>
***e) allocated-storage:*** <br />
*The amount of storage in GB to allocate for the DB instance. 20GB is used here.*
<br />
***f) Access mode:*** <br />
*It should be assigned to provide an RDS instance the public access or not. Here, we have chosen "public access" to give the DB instance a public IP address so that it is publicly accessible.*
<br />
Creating an RDS instance with these details: <br />

```sh
[root@ip-172-31-47-196 ~]# aws rds create-db-instance \
>     --db-instance-identifier db-master \
>     --db-instance-class db.t2.micro \
>     --engine mysql \
>     --engine-version 5.7.40  \
>     --vpc-security-group-ids "sg-0bb5ede97080122b8" \
>     --master-username admin \
>     --master-user-password admin123 \
>     --allocated-storage 20
>     --publicly-accessible
```

To know the details of the master RDS instance created, "describe" option can be used as follows with the identifier name, here it is "db-master".
```sh
[root@ip-172-31-47-196 ~]# aws rds describe-db-instances     --db-instance-identifier db-master
```
If this command results with  ***"DBInstanceStatus": "available"***, it means the RDS instance is created successfully.
Once completed, we need an entry named "Address" from the commandline result that is the endpoint address to connect the RDS instance from outside.
<br />
<br />
### 2. Accessing Master RDS instance

We can use the previously created username and password with the MySQL port 3306 and endpoint as database host to connect the MySQL master RDS instance from our local server.  
```sh
[root@ip-172-31-47-196 ~]# mysql -u admin -h db-master.clxsvqjvmbq6.ap-south-1.rds.amazonaws.com -p
```
To demonstrate these whole setup of load Balancing MySQL RDS read traffic, created database with few contents as shown in the image shown below:<br />
<img width="769" alt="queris" src="https://user-images.githubusercontent.com/117455666/212570103-909c0930-dc87-4290-9dc4-459bb29a619f.png">
<br />
<br />
### 3. Creating first read replicas

****a) Replica1****
<br />
From the above master RDS instance we have created, a read replica with identifier name "db-readreplica1" is created using "source-db-instance-identifier" option.<br/>
Here is the command.

```sh
[root@ip-172-31-47-196 ~]# aws rds create-db-instance-read-replica \
>     --db-instance-identifier db-readreplica1 \
>     --source-db-instance-identifier db-master

```
To get the RDS read replica instance details especially endpoint, the command with "describe" option can be used.
<br />
```sh
[root@ip-172-31-47-196 ~]# aws rds describe-db-instances --query "DBInstances[*].{Name:DBInstanceIdentifier,ID:Endpoint}"
```
Execute this until result shows ***"DBInstanceStatus": "available"***.
<br />
<br />
Logged in to the readreplica "db-readreplica1" with the endpoint obtained from the "describe" command.
```sh
[root@ip-172-31-47-196 ~]# mysql -u admin -h db-readreplica1.clxsvqjvmbq6.ap-south-1.rds.amazonaws.com -p
```

<br />

Contents from the "db-readreplica1":
<img width="789" alt="replica1-contents" src="https://user-images.githubusercontent.com/117455666/212570425-8fe52380-0509-4af5-b8aa-ec90883d3a97.png">

<br />

****b) Replica2****
<br />
Similarly, another readreplica from the same master RDS instance is created with name "db-readreplica2".
```sh
[root@ip-172-31-47-196 ~]# aws rds create-db-instance-read-replica \
>     --db-instance-identifier db-readreplica2 \
>     --source-db-instance-identifier db-master

```
The command with "describe" option can be used to get the RDS read replica instance details especially endpoint.
```sh
[root@ip-172-31-47-196 ~]# aws rds describe-db-instances --query "DBInstances[*].{Name:DBInstanceIdentifier,ID:Endpoint}"
```
Checked this until result shows ***"DBInstanceStatus": "available"***.
<br />
<br />
Logged in to the readreplica "db-readreplica2" with the endpoint obtained from the above "describe" command.
```sh
[root@ip-172-31-47-196 ~]# mysql -u admin -h db-readreplica2.clxsvqjvmbq6.ap-south-1.rds.amazonaws.com -p
```
<br />
Contents from the "db-readreplica2":
<img width="789" alt="replica2-contents" src="https://user-images.githubusercontent.com/117455666/212570706-8d0acafe-bd5d-4d55-848f-db720e2a913a.png">

<br />
<br />

### 4. Creating records with "Weighted" policy in Route53

This is the important phase of configuring the RDS readreplicas as loadbalancers using "Weighted routing policy" in Route53.<br />
For that we need following details:<br />
***a) hosted zone id.***
<br />
Following command can be used to find existing hostedzones with IDs in the AWS account:
```sh
[root@ip-172-31-47-196 ~]# aws route53 list-hosted-zones
```
***b) Endpoints of each Readreplicas***
We can get this from the "describe" command in previous steps. That will show following result. <br />
<img width="789" alt="list-replicas" src="https://user-images.githubusercontent.com/117455666/212573792-fb9634a5-2662-4391-98e5-590465fb2ae2.png">
<br />
*"Address"* shown at the highlighted entries denotes the endpoints associated with the RDS readreplicas.<br />
***c)Routing policy setup***
<br />
Here, we create individual record sets in a Route 53 hosted zone for each RDS endpoint associated with your read replicas just created. <br />
Then, we direct requests to the endpoint of the Route53 records by assigning them the same weight.<br />
By default, when we create a traffic policy by using the Amazon Route 53 API, one of the AWS SDKs, the AWS CLI, we should specify the definition of the traffic policy in a Document element in JSON format.<br />
So, we have created JSON file to create records with both Readreplica end points ***db-readreplica1.clxsvqjvmbq6.ap-south-1.rds.amazonaws.com*** and ***db-readreplica2.clxsvqjvmbq6.ap-south-1.rds.amazonaws.com*** as CNAME to the domain name, to which load balances using RDS readreplicas.<br />
Contents of that JSON file:<br />
```sh
[root@ip-172-31-47-196 ~]# cat change-resource-record-sets.json
 {
  "Comment": "Weighted policy to loadbalance with RDS readreplicas",
  "Changes": [
        {
         "Action": "CREATE",
         "ResourceRecordSet": {
         "Name": "rdsdb-read.haashdev.tech",
         "Type": "CNAME",
         "SetIdentifier": "rdsread endpoint1",
         "Weight": 50,
         "TTL": 5,
         "ResourceRecords": [
			  
           {
              "Value": "db-readreplica1.clxsvqjvmbq6.ap-south-1.rds.amazonaws.com"
           }


          ]
        }
    },

     {
         "Action": "CREATE",
         "ResourceRecordSet": {
         "Name": "rdsdb-read.haashdev.tech",
         "Type": "CNAME",
         "SetIdentifier": "rdsread endpoint2",
         "Weight": 50,
         "TTL": 5,
         "ResourceRecords": [

           {
               "Value": "db-readreplica2.clxsvqjvmbq6.ap-south-1.rds.amazonaws.com"
            }
		 
         ]
       }
  ]
}
```
To add this record to the associated Route53 hosted zone with ID, we can use the following format:<br />

```sh
[root@ip-172-31-47-196 ~]# aws route53 change-resource-record-sets --hosted-zone-id Z08712646EF6JSRNFSI8 --change-batch file://change-resource-record-sets.json
{
    "ChangeInfo": {
        "Status": "PENDING",
        "Comment": "Weighted policy to loadbalance with RDS readreplicas",
        "SubmittedAt": "2022-12-20T17:18:12.269Z",
        "Id": "/change/C07206033KV2044455660"
    }
}
[root@ip-172-31-47-196 ~]#
```

<br />
<br />
To confirm this loadbalancing via Route53, check the DNS lookup on the domain we have implemented load Balancing MySQL RDS read traffic using Route53 Weighted Routing policy.<br />
Refer to the image given below: <br />
<img width="673" alt="dig" src="https://user-images.githubusercontent.com/117455666/212574128-76ea75a0-a639-47d2-a9f3-53f59beaf8fc.png">
<br />
<br />
<br />
<br />

****ADDITIONAL INFORMATION**** 
<br />
<br />
To delete the RDS instance after keeping a snapshot, use the following command:
```sh
aws rds delete-db-instance \
    --db-instance-identifier test-instance \
    --final-db-snapshot-identifier test-instance-final-snap
```
To delete the RDS instances without snapshot, use the option "--skip-final-snapshot"

`[root@ip-172-31-47-196 ~]# aws rds delete-db-instance --db-instance-identifier db-master --skip-final-snapshot` <br />
`[root@ip-172-31-47-196 ~]# aws rds delete-db-instance --db-instance-identifier db-readreplica1 --skip-final-snapshot` <br />
`[root@ip-172-31-47-196 ~]# aws rds delete-db-instance --db-instance-identifier db-readreplica2 --skip-final-snapshot` <br />


To remove the Hosted zone records in Route53, edit the same JSON file with ***"Action": "DELETE"*** instead of ***"Action": "CREATE"*** and use the following command
```sh
[root@ip-172-31-47-196 ~]# aws route53 change-resource-record-sets --hosted-zone-id Z08712646EF6JSRNFSI8 --change-batch file://change-res ource-record-sets.json
```

<br />
<br />

***Thank you***




