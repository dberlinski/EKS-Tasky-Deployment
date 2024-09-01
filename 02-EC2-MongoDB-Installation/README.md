# **MongoDB installation on Amazon Linux 2023**

1. Launch Amazon Linux 2023
2. Create a /etc/yum.repos.d/mongodb-org-7.0.repo file so that you can install MongoDB directly using dnf:
```sh
[mongodb-org-7.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/amazon/2023/mongodb-org/7.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://pgp.mongodb.com/server-7.0.asc
```
3. Install mongosh support for openssl3 and the latest stable version of MongoDB
```sh
sudo dnf install -qy mongodb-mongosh-shared-openssl3
sudo dnf install -y mongodb-org
```
4. By default, the MongoDB server (mongod) only allows loopback connections from IP address 127.0.0.1 (localhost). To allow connections from elsewhere in your Amazon VPC, do the following:
Edit the /etc/mongod.conf file and look for the following lines.
```sh
# network interfaces
net:
  port: 27017
  bindIp: 0.0.0.0  # Enter 0.0.0.0,:: to bind to all IPv4 and IPv6 addresses or, alternatively, use the net.bindIpAll setting.
```
MongoDB accepts connections on TCP port 27017. The instance Security group was edited to accept traffic bound for TCP port 27017. 

## MongoDB EC2 Instance Profile configuration

The MongoDB database server will need permissions to copy its backup data to S3. The following configuration steps use the AWS CLI to create the required IAM role, policy, and instance profile that will be attached to the instance.

1. Trust policy definition. The ec2-trustpolicy-mongodb.json file is created.
```sh
cat >> ec2-trustpolicy-mongodb.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "sts:AssumeRole"
            ],
            "Principal": {
                "Service": [
                    "ec2.amazonaws.com"
                ]
            }
        }
    ]
}
EOF

2. IAM role creation.
```
aws iam create-role --role-name db-ec2-role-mongodb-ekslab --assume-role-policy-document file://ec2-trustpolicy-mongodb.json
```

3. IAM inline policy document creation.  The following policy document grants access to S3 and EC2.
```sh
cat >> ec2-policy-mongodb.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "s3:ListAccessPointsForObjectLambda",
                "s3:GetAccessPoint",
                "s3:PutAccountPublicAccessBlock",
                "s3:ListAccessPoints",
                "s3:ListJobs",
                "s3:CreateStorageLensGroup",
                "s3:PutStorageLensConfiguration",
                "s3:ListMultiRegionAccessPoints",
                "s3:ListStorageLensGroups",
                "s3:ListStorageLensConfigurations",
                "s3:GetAccountPublicAccessBlock",
                "s3:ListAllMyBuckets",
                "s3:ListAccessGrantsInstances",
                "s3:PutAccessPointPublicAccessBlock",
                "ec2:*",
                "s3:CreateJob"
            ],
            "Resource": "*"
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": "s3:*",
            "Resource": "arn:aws:s3:::*"
        }
    ]
}
EOF
```

4. Add the inline policy to the IAM role.
```
aws iam put-role-policy --role-name db-ec2-role-mongodb-ekslab --policy-name db-ec2-policy-mongodb-ekslab --policy-document file://ec2-policy-mongodb.json
```

5. Create the IAM instance profile.
```
aws iam create-instance-profile --instance-profile-name ec2-mongodb-ekslab
```

6. Add the IAM role to the instance profile.
```
aws iam add-role-to-instance-profile --instance-profile-name ec2-mongodb-ekslab --role-name db-ec2-role-mongodb-ekslab
```

7. Finally, attach the instance profile to the ec2 instance running MongoDB.
```
aws ec2 associate-iam-instance-profile --iam-instance-profile Name=ec2-mongodb-ekslab --instance-id <instance-id>
```

## MongoDB Authentication and Authorization notes

1- In the MongoDB instance, issue:
```sh
mongosh
```

2- Use the "use admin" command to change into the admin database.  We will create an admin user in that database.  Note: It is possible to create administration users local to a given database.
```sh
use admin

db.createUser({ user: "admin", pwd: "retracted", roles: ["root"] })
```

3- Edit the mongodb configuration file and allow authorization:
```sh
sudo vim /etc/mongod.conf
Security:
  authorization: enabled
```

4- Restart mongod
```sh
sudo systemctl restart mongod
```

## Shell script to backup MongoDB to S3

The following method based on the mongodump utility produces a bash script that backs up the mongodb database to a S3 bucket. It leverages s3cmd to ship the backup data to S3 and cron to automate script execution.
The following script is saved to /home/ec2-user/scripts/mongo_backup.sh:

```sh
#!/bin/bash

MONGODUMP_PATH="/usr/bin/mongodump"
MONGO_DATABASE="go-mongodb" #replace with your database name

TIMESTAMP=`date +%F-%H%M`
S3_BUCKET_NAME="somebucket" #replace with your bucket name on Amazon S3
S3_BUCKET_PATH="mongodb-backups"

# Create backup
mongodump --host=192.168.0.100 --db=go-mongodb --port=27017 --authenticationDatabase="admin" -u="admin" -p="dogcat"

# Add timestamp to backup
mv dump mongodb-$HOSTNAME-$TIMESTAMP
tar cf mongodb-$HOSTNAME-$TIMESTAMP.tar mongodb-$HOSTNAME-$TIMESTAMP

# Upload to S3
/usr/local/bin/s3cmd put mongodb-$HOSTNAME-$TIMESTAMP.tar s3://$S3_BUCKET_NAME/$S3_BUCKET_PATH/mongodb-$HOSTNAME-$TIMESTAMP.tar
```

## Cron installation on Amazon Linux 2023

On Amazon Linux 2023, crontab is not available by default.  Therefore, we need to install it with the following commands:

```sh
# Install the ‘cronie’ package.
sudo dnf install cronie -y

# Enable the 'cronie' service.
sudo systemctl enable crond.service

# Start the 'cronie' service. 
sudo systemctl start crond.service

# Check status.
sudo systemctl status crond.service | grep Active
```

The /etc/crontab file needs to be edited with the desired backup schedule. In my example it backs up daily at 1:00hrs.

```sh
SHELL=/bin/bash
PATH=/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root

# For details see man 4 crontabs

# Example of job definition:
# .---------------- minute (0 - 59)
# |  .------------- hour (0 - 23)
# |  |  .---------- day of month (1 - 31)
# |  |  |  .------- month (1 - 12) OR jan,feb,mar,apr ...
# |  |  |  |  .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# |  |  |  |  |
# *  *  *  *  * user-name  command to be executed
0 1 * * * root sh /home/ec2-user/scripts/mongo_backup.sh >> /tmp/$(date +\%d-\%m-\%Y_\%H:\%M:\%S)-errors 2>&1
```

## s3cmd installation

The s3cmd utility is used to ship mongodb backup data to S3.  Its installation is done via the python pip package manager.  If need be, install pip and then install s3cmd.

```sh
sudo dnf install python-pip

sudo pip install s3cmd
```

Following these we will need to run s3cmd –configure and provide our credentials for it to access our account’s buckets.

The s3cmd utility installation information can be found at https://github.com/s3tools/s3cmd/blob/master/INSTALL.md 












