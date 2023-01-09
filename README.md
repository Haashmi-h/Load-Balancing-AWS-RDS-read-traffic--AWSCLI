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

#### 1. Creating a Primary RDS instance.

Here I'm creating a MysQL RDS instance with following main options :<br />
a) RDS instance type: t2.micro 
b) DB engine with version: MySQL 5.7.40  
c) A security group to associate with this DB instance

A security group details can be obtained from following command:
```sh
[root@ip-172-31-47-196 ~]# aws ec2 describe-security-groups  --query "SecurityGroups[*].{Name:GroupName,ID:GroupId}"
```

```sh
[root@ip-172-31-47-196 ~]# aws rds create-db-instance \
>     --db-instance-identifier db-master \
>     --db-instance-class db.t2.micro \
>     --engine mysql \
>    --engine-version 5.7.40  \
>   --vpc-security-group-ids "sg-0bb5ede97080122b8" \
>     --master-username admin \
>     --master-user-password admin123 \
>     --allocated-storage 20
>    --publicly-accessible

```


[root@ip-172-31-47-196 ~]# aws rds describe-db-instances     --db-instance-identifier db-master

Last part:
```sh
            "Endpoint": {
                "HostedZoneId": "Z2VFMSZA74J7XZ",
                "Port": 3306,
                "Address": "db-master.clxsvqjvmbq6.ap-south-1.rds.amazonaws.com"
            },
            "DBInstanceStatus": "available",
            "IAMDatabaseAuthenticationEnabled": false,
            "EngineVersion": "5.7.40",
            "DeletionProtection": false,
            "AvailabilityZone": "ap-south-1b",
            "DomainMemberships": [],
            "StorageType": "gp2",
            "DbiResourceId": "db-3QHUZUP4K7ZDTY7YZ5LHVDP7UQ",
            "CACertificateIdentifier": "rds-ca-2019",
            "StorageEncrypted": false,
            "AssociatedRoles": [],
            "DBInstanceClass": "db.t2.micro",
            "DbInstancePort": 0,
            "DBInstanceIdentifier": "db-master"
        }
```

#### 2. Creating first read replica

```sh
[root@ip-172-31-47-196 ~]# aws rds create-db-instance-read-replica \
>     --db-instance-identifier db-readreplica1 \
>     --source-db-instance-identifier db-master

```

describe
```sh
[root@ip-172-31-47-196 ~]# aws rds describe-db-instances --query "DBInstances[*].{Name:DBInstanceIdentifier,ID:Endpoint}"
```


#### 3. Login
```sh
[root@ip-172-31-47-196 ~]# mysql -u admin -h db-master.clxsvqjvmbq6.ap-south-1.rds.amazonaws.com -p
```

Replica1
[root@ip-172-31-47-196 ~]# mysql -u admin -h db-readreplica1.clxsvqjvmbq6.ap-south-1.rds.amazonaws.com -p
[root@ip-172-31-47-196 ~]# mysql -u admin -h db-readreplica2.clxsvqjvmbq6.ap-south-1.rds.amazonaws.com -p
Enter password:





Replica1
[root@ip-172-31-47-196 ~]# mysql -u admin -h db-readreplica1.clxsvqjvmbq6.ap-south-1.rds.amazonaws.com -p
[root@ip-172-31-47-196 ~]# mysql -u admin -h db-readreplica2.clxsvqjvmbq6.ap-south-1.rds.amazonaws.com -p
Enter password:

[root@ip-172-31-47-196 ~]# aws route53 change-resource-record-sets --hosted-zone-id Z08712646EF6JSRNFSI8 --change-batch file://change-res ource-record-sets.json
{
    "ChangeInfo": {
        "Status": "PENDING",
        "Comment": "Weighted policy to loadbalance with RDS readreplicas",
        "SubmittedAt": "2022-12-20T17:18:12.269Z",
        "Id": "/change/C07206033KV2044455660"
    }
}
[root@ip-172-31-47-196 ~]#

https://docs.aws.amazon.com/cli/latest/reference/rds/delete-db-instance.html
aws rds delete-db-instance \
    --db-instance-identifier test-instance \
    --final-db-snapshot-identifier test-instance-final-snap

[root@ip-172-31-47-196 ~]# aws rds delete-db-instance --db-instance-identifier db-master --skip-final-snapshot

[root@ip-172-31-47-196 ~]# aws rds delete-db-instance --db-instance-identifier db-readreplica1 --skip-final-snapshot
[root@ip-172-31-47-196 ~]# aws rds delete-db-instance --db-instance-identifier db-readreplica2 --skip-final-snapshot




[root@ip-172-31-47-196 ~]# cat change-resource-record-sets.json
{
          "Comment": "Weighted policy to loadbalance with RDS readreplicas",
            "Changes": [
                        {
                                      "Action": "DELETE",
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

           "Action": "DELETE",

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
[root@ip-172-31-47-196 ~]#
[root@ip-172-31-47-196 ~]# aws route53 change-resource-record-sets --hosted-zone-id Z08712646EF6JSRNFSI8 --change-batch file://change-res ource-record-sets.json




