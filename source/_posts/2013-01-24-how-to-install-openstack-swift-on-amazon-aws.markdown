---
layout: post
title: "How to install Openstack Swift on Amazon AWS"
date: 2013-01-24 12:19
comments: true
categories:
---

At [DropMyEmail](http://dropmyemail), we are evaluating open source private cloud storage solutions. We have found [OpenStack Swift](http://swift.openstack.org) to be the best choice in the market right now. We are documenting the steps to build a prototype swift cluster. Much of the work is from this amazing [blog post by Amar](http://www.buildcloudstorage.com/2011/10/installing-openstack-swift-cluster-on.html) and the [multiple server swift installation](http://docs.openstack.org/developer/swift/howto_installmultinode.html) document. Our setup is largely similar except that we are on a more recent stack.

## Overview

{% img https://s3.amazonaws.com/static.engineering.dropmyemail.com/DropMyEmail+Prototype+OpenStack+Swift+Cluster.png DropMyEmail Prototype OpenStack Swift Cluster %}

It is a simple architecture. We have a proxy server which acts as the interface for the outside world. The proxy server includes the authentication server. The proxy server is connected to 5 storage nodes. Each storage node includes account, container and object servers. Each node is a single [EC2](http://aws.amazon.com/ec2) instance.

## Setting up the instances

We assume you know how to set up an EC2 instance. Our instance are all __Ubuntu Linux 12.04__ images from [Alestic](http://alestic.com).

For the security groups, here are the settings we used for each type of node.

### Security group

- Port 22 from source 0.0.0.0/0
- Port 43 from source 0.0.0.0/0
- Port 6000-6100 from source 0.0.0.0/0

For the rest of the article, we'll be using the public and private IP addresses of the EC2 instance at different scenarios. Do note [the differences](http://aws.amazon.com/articles/1145) betwen them. We'll be assuming you are logging into the instance via ssh and running commands.

## Preparing the base Swift image

We are going to prepare an [AMI](https://aws.amazon.com/amis) image with the core Swift software installed.

First we add the swift repository and install the base swift packages.

```
sudo add-apt-repository ppa:swift-core/release
sudo apt-get update
sudo apt-get install swift
```

Next we set up the swift configuration files. Create the Swift configuration directory.

```
sudo mkdir -p /etc/swift
```

We need to create a random string to act as the hash for Swift. Run the following command, you should see a random string as the result.

```
sudo od -t x8 -N 8 -A n </dev/random
```

Create __/etc/swift/swift.conf__. Add the following lines to it.

```
[swift-hash]
# random unique string that can never change (DO NOT LOSE)
swift_hash_path_suffix = <RESULTS FROM ABOVE od COMMAND>
```

Change the owner to swift.

```
sudo chown -R swift:swift /etc/swift
```

Now save the image as an AMI. This base image can be used for the other instances.

## Setting up the Swift Proxy server

Now choose one instance for the proxy server. The proxy server contains both the proxy and authentication server. Let us install the proxy server first.

Run these commands to install the dependencies.

```
sudo apt-get install swift-proxy memcached swauth
```

You could use [OpenStack Keystone](http://keystone.openstack.org) for authentication. Cons is you need to set up and maintain another system for authentication. Swauth is a self-contained authentication server on the proxy server. If you just want to run Swift on its own, we'd recommend going with [Swauth](https://github.com/gholt/swauth).

Next we create the SSL cert to encrypt our requests.

```
cd /etc/swift
sudo openssl req -new -x509 -nodes -out cert.crt -keyout cert.key
```

We need to replace the IP address in the memcached configuration file and then restart memcached. Check the private IP address for the instance and replace the IP with your own. We are _assuming_ it is __10.0.0.0__.

```
sudo export PROXY_LOCAL_NET_IP=10.0.0.0
sudo perl -pi -e "s/-l 127.0.0.1/-l $PROXY_LOCAL_NET_IP/" /etc/memcached.conf
sudo service memcached restart
```

Next we need to create the configuration file for the proxy server. Create __/etc/swift/proxy-server.conf__ with the following contents. Remember to replace the URL and IP address with those from your instance.

```
[DEFAULT]
bind_port = 443
cert_file = /etc/swift/cert.crt
key_file = /etc/swift/cert.key
workers = 8
user = swift

[pipeline:main]
pipeline = healthcheck cache swauth proxy-server

[app:proxy-server]
use = egg:swift#proxy
allow_account_management = true

[filter:swauth]
use = egg:swauth#swauth
set log_name = swauth
super_admin_key = swauthkey
default_swift_cluster = cluster_name#https://ec2-1-2-3-4.compute-1.amazonaws.com:443/v1#https://127.0.0.1:443/v1

[filter:healthcheck]
use = egg:swift#healthcheck

[filter:cache]
use = egg:swift#memcache
memcache_servers = 10.0.0.0:11211
```

Now we create the rings.

```
cd /etc/swift
sudo swift-ring-builder account.builder create 18 3 1
sudo swift-ring-builder container.builder create 18 3 1
sudo swift-ring-builder object.builder create 18 3 1
```

And set the owner back to swift.

```
sudo chown -R swift:swift /etc/swift
```

This is it for now. We'll set up the storage nodes first before coming back to the proxy server.

## Setting up the Swift Storage nodes

Unlike what Amar proposed, there is no need to keep the instances under the same availability zone. Ideally the storage nodes should be in different regions to prevent multiple failures. For our case, we are setting up everything in the same region for convenience.

We are going to add EBS volumes to mimic hard drives. Create a new EBS volume for each storage server and attach them. You may use the default value of __/dev/sdf__ for __Device__. They will be renamed to __/dev/xvdf__ in your instance.

The storage node contain the account, container and object servers. Install the dependencies.

```
sudo apt-get install swift-account swift-container swift-object xfsprogs
```

Next we want to set up the partition.

```
sudo fdisk /dev/xvdf
```

FDisk will be prompt you for the values. You may follow the responses in order

```
n
p
1
Enter
Enter
w
```

Next we build the [XFS](http://en.wikipedia.org/wiki/XFS) file system within the partition.

```
sudo mkfs.xfs -i size=1024 /dev/xvdf1
```

Append the following line to __/etc/fstab__

```
/dev/xvdf1 /srv/node/xvdf1 xfs noatime,nodiratime,nobarrier,logbufs=8 0 0
```

We mount it.

```
sudo mkdir -p /srv/node/xvdf1
sudo mount /srv/node/xvdf1
```

And change the owner to swift.

```
sudo chown -R swift:swift /srv/node
```

### Rsync

We need to configure [rsync](http://en.wikipedia.org/wiki/Rsync) as well. Create __/etc/rsyncd.conf__. Add the following to the file. __STORAGE_LOCAL_NET_IP__ is the IP address of the instance. Change it accordingly.

```
uid = swift
gid = swift
log file = /var/log/rsyncd.log
pid file = /var/run/rsyncd.pid
address = <STORAGE_LOCAL_NET_IP>

[account]
max connections = 2
path = /srv/node/
read only = false
lock file = /var/lock/account.lock

[container]
max connections = 2
path = /srv/node/
read only = false
lock file = /var/lock/container.lock

[object]
max connections = 2
path = /srv/node/
read only = false
lock file = /var/lock/object.lock
```

Enable rsync in the rsync configuration at __/etc/default/rsync__.

```
RSYNC_ENABLE = true
```

And start rsync

```
service rsync start
```

### Account/Container/Object server configurations

Next we need to set the IP address for the account, container and object server configuration files. Use the instance's IP address for __STORAGE_LOCAL_NET_IP__.

The account server's configuration file is at __/etc/swift/account-server.conf__.

```
[DEFAULT]
bind_ip = <STORAGE_LOCAL_NET_IP>
workers = 2

[pipeline:main]
pipeline = account-server

[app:account-server]
use = egg:swift#account

[account-replicator]

[account-auditor]

[account-reaper]
```

The container server's configuration file is at __/etc/swift/container-server.conf__.

```
[DEFAULT]
bind_ip = <STORAGE_LOCAL_NET_IP>
workers = 2

[pipeline:main]
pipeline = container-server

[app:container-server]
use = egg:swift#container

[container-replicator]

[container-updater]

[container-auditor]
```

The object server's configuration file is at __/etc/swift/object-server.conf__.

```
[DEFAULT]
bind_ip = <STORAGE_LOCAL_NET_IP>
workers = 2

[pipeline:main]
pipeline = object-server

[app:object-server]
use = egg:swift#object

[object-replicator]

[object-updater]

[object-auditor]
```

Repeat these steps for the rest of the storage nodes.

## Starting the servers

Let's go back to the proxy server. We are going to create a script to build the rings. Create a file at __/etc/swift/build_rings.sh__. Replace the IP address accordingly.

```
#!/bin/sh
export ZONE=1                  # set the zone number for that storage device
export STORAGE_LOCAL_NET_IP=10.0.0.1   # and the IP address
export WEIGHT=100               # relative weight (higher for bigger/faster disks)
export DEVICE=sdf1
sudo swift-ring-builder account.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6002/$DEVICE $WEIGHT
sudo swift-ring-builder container.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6001/$DEVICE $WEIGHT
sudo swift-ring-builder object.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6000/$DEVICE $WEIGHT

export ZONE=2                  # set the zone number for that storage device
export STORAGE_LOCAL_NET_IP=10.0.0.2   # and the IP address
export WEIGHT=100               # relative weight (higher for bigger/faster disks
export DEVICE=sdf1
sudo swift-ring-builder account.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6002/$DEVICE $WEIGHT
sudo swift-ring-builder container.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6001/$DEVICE $WEIGHT
sudo swift-ring-builder object.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6000/$DEVICE $WEIGHT

export ZONE=3                  # set the zone number for that storage device
export STORAGE_LOCAL_NET_IP=10.0.0.3   # and the IP address
export WEIGHT=100               # relative weight (higher for bigger/faster disks)
export DEVICE=sdf1
sudo swift-ring-builder account.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6002/$DEVICE $WEIGHT
sudo swift-ring-builder container.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6001/$DEVICE $WEIGHT
sudo swift-ring-builder object.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6000/$DEVICE $WEIGHT

export ZONE=4                  # set the zone number for that storage device
export STORAGE_LOCAL_NET_IP=10.0.0.4   # and the IP address
export WEIGHT=100               # relative weight (higher for bigger/faster disks
export DEVICE=sdf1
sudo swift-ring-builder account.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6002/$DEVICE $WEIGHT
sudo swift-ring-builder container.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6001/$DEVICE $WEIGHT
sudo swift-ring-builder object.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6000/$DEVICE $WEIGHT

export ZONE=5                  # set the zone number for that storage device
export STORAGE_LOCAL_NET_IP=10.0.0.5   # and the IP address
export WEIGHT=100               # relative weight (higher for bigger/faster disks
export DEVICE=sdf1
sudo swift-ring-builder account.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6002/$DEVICE $WEIGHT
sudo swift-ring-builder container.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6001/$DEVICE $WEIGHT
sudo swift-ring-builder object.builder add z$ZONE-$STORAGE_LOCAL_NET_IP:6000/$DEVICE $WEIGHT
```

Make the script executable and run it.

```
sudo chmod +x /etc/swift/build_rings.sh
sudo /etc/swift/build_rings.sh
```

We continue the build the rings.

```
sudo swift-ring-builder account.builder
sudo swift-ring-builder container.builder
sudo swift-ring-builder object.builder
```

And rebalance them.

```
sudo swift-ring-builder account.builder rebalance
sudo swift-ring-builder container.builder rebalance
sudo swift-ring-builder object.builder rebalance
```

Copy your instance's private key(the key file which you use for sshing in to your instance) to the proxy server. I'm assuming it is __private_key.pem__. Change the permission of __private_key.pem__ to __0600__.

Next we need to propagate the ring files to all the storage nodes

```
scp -i private_key.pem /etc/swift/*.gz ubuntu@10.0.0.1:/etc/swift
scp -i private_key.pem /etc/swift/*.gz ubuntu@10.0.0.2:/etc/swift
scp -i private_key.pem /etc/swift/*.gz ubuntu@10.0.0.3:/etc/swift
scp -i private_key.pem /etc/swift/*.gz ubuntu@10.0.0.4:/etc/swift
scp -i private_key.pem /etc/swift/*.gz ubuntu@10.0.0.5:/etc/swift
```

Log in to each of the storage node and run these commands.

```
cd /etc/swift
sudo chown -R swift:swift /etc/swift
sudo swift-init all start
```

Your storage node should be running!

Now we need to start the proxy server. Log into the proxy server and run these commands.

```
sudo chown -R swift:swift /etc/swift
sudo swift-init proxy start
```

### Setting the authentication

We need to create the user for authentication.

```
sudo swauth-prep -K swauthkey -A https://127.0.0.1/auth/
sudo swauth-add-user -K swauthkey -A https://127.0.0.1/auth/ -a test tester testing
```

## Using the Swift cluster

We have chosen to use another EC2 instance to act as a Swift client. As usual, create an instance and run the following commands.

```
sudo add-apt-repository ppa:swift-core/release
sudo apt-get update
sudo apt-get install swift
```

We need to obtain the authentication token first. As usual, replace the url with your own.

```
curl -k -v -H 'X-Storage-User: test:tester' -H 'X-Storage-Pass: testing' https://ec2-1-2-3-4.compute-1.amazonaws.com/auth/v1.0
```

You should get this output

```
* About to connect() to ec2-1-2-3-4.compute-1.amazonaws.com port 443 (#0)
*   Trying 10.0.0.1... connected
* successfully set certificate verify locations:
*   CAfile: none
  CApath: /etc/ssl/certs
* SSLv3, TLS handshake, Client hello (1):
* SSLv3, TLS handshake, Server hello (2):
* SSLv3, TLS handshake, CERT (11):
* SSLv3, TLS handshake, Server finished (14):
* SSLv3, TLS handshake, Client key exchange (16):
* SSLv3, TLS change cipher, Client hello (1):
* SSLv3, TLS handshake, Finished (20):
* SSLv3, TLS change cipher, Client hello (1):
* SSLv3, TLS handshake, Finished (20):
* SSL connection using AES256-SHA
* Server certificate:
*        subject: C=AU; ST=Some-State; O=Internet Widgits Pty Ltd
*        start date: 2013-01-23 06:24:57 GMT
*        expire date: 2013-02-22 06:24:57 GMT
* SSL: unable to obtain common name from peer certificate
> GET /auth/v1.0 HTTP/1.1
> User-Agent: curl/7.22.0 (x86_64-pc-linux-gnu) libcurl/7.22.0 OpenSSL/1.0.1 zlib/1.2.3.4 libidn/1.23 librtmp/2.3
> Host: ec2-1-2-3-4.compute-1.amazonaws.com
> Accept: */*
> X-Storage-User: test:tester
> X-Storage-Pass: testing
>
< HTTP/1.1 200 OK
< X-Storage-Url: https://ec2-1-2-3-4.compute-1.amazonaws.com:443/v1/AUTH_f1e9d209-4543-4907-8ae3-98f24e0b051f
< X-Storage-Token: AUTH_tkd26a954e86534b9498f1b196fdb60ac2
< X-Auth-Token: AUTH_tkd26a954e86534b9498f1b196fdb60ac2
< Content-Length: 155
< Date: Thu, 24 Jan 2013 03:59:09 GMT
<
* Connection #0 to host ec2-1-2-3-4.compute-1.amazonaws.com left intact
* Closing connection #0
* SSLv3, TLS alert, Client hello (1):
{"storage": {"cluster_name": "https://ec2-1-2-3-4.compute-1.amazonaws.com:443/v1/AUTH_f1e9d209-4543-4907-8ae3-98f24e0b051f", "default": "cluster_name"}}
```

Next we make a request with the authentication token.

```
curl -k -v -H 'X-Auth-Token: AUTH_tkd26a954e86534b9498f1b196fdb60ac2' https://ec2-184-73-4-50.compute-1.amazonaws.com:443/v1/AUTH_f1e9d209-4543-4907-8ae3-98f24e0b051f
```

You should get this output.

```
* About to connect() to ec2-1-2-3-4.compute-1.amazonaws.com port 443 (#0)
*   Trying 10.0.0.1... connected
* successfully set certificate verify locations:
*   CAfile: none
  CApath: /etc/ssl/certs
* SSLv3, TLS handshake, Client hello (1):
* SSLv3, TLS handshake, Server hello (2):
* SSLv3, TLS handshake, CERT (11):
* SSLv3, TLS handshake, Server finished (14):
* SSLv3, TLS handshake, Client key exchange (16):
* SSLv3, TLS change cipher, Client hello (1):
* SSLv3, TLS handshake, Finished (20):
* SSLv3, TLS change cipher, Client hello (1):
* SSLv3, TLS handshake, Finished (20):
* SSL connection using AES256-SHA
* Server certificate:
*        subject: C=AU; ST=Some-State; O=Internet Widgits Pty Ltd
*        start date: 2013-01-23 06:24:57 GMT
*        expire date: 2013-02-22 06:24:57 GMT
* SSL: unable to obtain common name from peer certificate
> GET /v1/AUTH_f1e9d209-4543-4907-8ae3-98f24e0b051f HTTP/1.1
> User-Agent: curl/7.22.0 (x86_64-pc-linux-gnu) libcurl/7.22.0 OpenSSL/1.0.1 zlib/1.2.3.4 libidn/1.23 librtmp/2.3
> Host: ec2-1-2-3-4.compute-1.amazonaws.com
> Accept: */*
> X-Auth-Token: AUTH_tkd26a954e86534b9498f1b196fdb60ac2
>
< HTTP/1.1 204 No Content
< X-Account-Object-Count: 0
< X-Account-Bytes-Used: 0
< X-Account-Container-Count: 0
< Accept-Ranges: bytes
< Content-Length: 0
< Date: Thu, 24 Jan 2013 04:00:47 GMT
<
* Connection #0 to host ec2-1-2-3-4.compute-1.amazonaws.com left intact
* Closing connection #0
* SSLv3, TLS alert, Client hello (1):
```

Now we are able to use our Swift store. The __stat__ command shows you what is on the server.

```
swift -A https://ec2-1-2-3-4.compute-1.amazonaws.com/auth/v1.0 -U test:tester -K testing stat
```

Have a look at the options available with the help option.

```
swift -h
```

Try uploading a file with the __upload__ command.

```
swift -A https://ec2-1-2-3-4.compute-1.amazonaws.com/auth/v1.0 -U test1:tester1 -K testing upload test foo.gz
```

And there you go, a working swift cluster. Please let us know if there are any mistakes. Thanks!
