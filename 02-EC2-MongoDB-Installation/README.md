#**MongoDB installation on Amazon Linux 2023**

1. Launch Amazon Linux 2023
2. Create a /etc/yum.repos.d/mongodb-org-7.0.repo file so that you can install MongoDB directly using dnf:
```SHELL
[mongodb-org-7.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/amazon/2023/mongodb-org/7.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://pgp.mongodb.com/server-7.0.asc
```
3. Install mongosh support for openssl3 and the latest stable version of MongoDB
```SHELL
sudo dnf install -qy mongodb-mongosh-shared-openssl3
sudo dnf install -y mongodb-org
```
4. By default, the MongoDB server (mongod) only allows loopback connections from IP address 127.0.0.1 (localhost). To allow connections from elsewhere in your Amazon VPC, do the following:
Edit the /etc/mongod.conf file and look for the following lines.
```SHELL
# network interfaces
net:
  port: 27017
  bindIp: **0.0.0.0**  # Enter 0.0.0.0,:: to bind to all IPv4 and IPv6 addresses or, alternatively, use the net.bindIpAll setting.


