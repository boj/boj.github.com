---
layout: blog
title: LVS - Nginx - NodeJS - MongoDB - Cluster Setup on RackSpace
summary: LVS - Nginx - NodeJS - MongoDB - Configuring a high availability scalable server cluster on RackSpace.
---

## {{ page.title }}

__20 Jan 2011 - Osaka, Japan__

## Introduction

This article will lay out the steps I used to build a high availability scalable social game server cluster on the RackSpace Cloud for the latest project I am working on at [Istpika](http://www.istpika.com/en).  The basic requirements were a simple HTTP based API which exposes a RESTful-like interface to the client, pioneer our first use of MongoDB as the backend database, a fault tolerant setup, and a service in North America.  The actual client is built on smartphones with the game logic written in JavaScript using a currently undisclosed SDK.  With that in mind, it seemed like building a full stack JavaScript system would make development a lot smoother for all parties involved, so I also decided to write the server logic in NodeJS, all of which ties in neatly with MongoDB's JavaScript system.

Ultimately the service selection boiled down to either Amazon EC2, or the RackSpace Cloud.  While I am admittedly biased towards RackSpace due to the fact that my favorite service [GitHub](https://github.com/blog/493-github-is-moving-to-rackspace) runs on them and I run my personal servers on their cloud service, after a bit of research they were still clearly the winners in regards to flexibility and compute power.  I have no solid opinion in regards to Amazon EC2 since I have never directly used them, but large social game companies like Playfish seem to be [using them without any problems](http://highscalability.com/blog/2010/9/21/playfishs-social-gaming-architecture-50-million-monthly-user.html).  I did find an interesting [article](http://www.thebitsource.com/featured-posts/rackspace-cloud-servers-versus-amazon-ec2-performance-analysis/) showing speed comparisons between the two services, with RackSpace coming out ahead on most benchmarks.  My only major concern at this point is the 16GB memory limit of the largest Cloud Servers for the databases, but am sure that if it becomes a problem MongoDB's sharding will come into play and save the day.

I will leave out most of the small configuration details in this blog and will be left as an exercise to the reader.

## Initial Setup

There's a handful of things I like to do to make managing the servers and securing them easier.  The first step is to make an administration server which all SSH access to the servers will be routed through.  This allows us to shut down the public facing interfaces on most of the internal servers, disable external SSH access on servers with external interfaces, and create a barrier between them and the internet.  We also typically use this server for secondary services, such as remote logging, system monitoring, data mining tools, etc.

After creating a new server instance via the RackSpace control panel and logging in for the first time, it's recommended to create an 'access user' and append your public SSH key to their .ssh/authorized_keys file, add them to the admin group and sudoers file, and turn off root login and password access to SSH.  After that create a SSH key for the root user, this will be used to allow direct login access to the other servers.

For each additional server that is brought up we copy the root SSH key over, and in the case of an internal only server disable the external interface that RackSpace automatically assigns a public IP to.  To make things easy I also edit the _/etc/hosts_ file and add a simple name entry based on the server's function along with the internal IP of the new server (e.g. 10.0.01 admin).

## Load Balancers - LVS and Keepalived

In order to get all of this going we need to set up some frontend load balancers which direct all traffic to the application servers behind them, and will failover in case one of the load balancers go down.  RackSpace has announced that they will eventually offer this as a service and [that it is currently in private beta for those interested](http://www.rackspacecloud.com/blog/2010/11/09/announcing-cloud-load-balancing-private-beta/), however due to time constraints I decided to implement my own solution.  I found a helpful RackSpace blog about [setting up LVS-TUN on their service](http://www.rackspacecloud.com/blog/2010/09/22/installing-and-configuring-lvs-tun/) using pulse, but opted to use LVS-TUN + keepalived instead due to the fact that I've had more experience with it up to this point.

This kind of setup is ideal because it allows you to maximize the throughput of all your servers (the connection comes in through the load balancers, but all traffic is handled by the application servers), and is very scalable in regards to geographical locations.

### Terminology

These are some terms used in regards to LVS.

* Directors - The load balancers.
* Realservers - The app servers which are being load balanced.
* VIP - The virtual IP which floats between the directors.
* RIP - Realserver IP.

### Initial Preparation

First setup two servers from the RackSpace control panel to act as your directors.  A server size of 256mb memory will be just fine since it is just routing traffic to your application servers behind it.

In order for load balancing to work we will need to request a shared IP.  You will need to open a support ticket and request an IP which they will assign to the two directors for you, as well as tell them which realservers will be replying back to the clients.  This is very simple and painless, just make sure to explain to them that the additional IP will be used for failover and there shouldn't be any problems.

It's also ideal to disable your iptables rules until everything has been properly debugged and is working.

### Directors

The first step is to build two LVS-TUN configured directors.  The two fundamental packages we will need are _ipvsadm_ and _keepalived_.  The keepalived configuration should be similar to the following, which assumes an HTTP service running on port 80.

    #
    # /etc/keepalived/keepalived.conf
    #

    # global defs
    global_defs {
        notification_email {
          someone@example.com
        }
        notification_email_from keepalived@director
        smtp_server 127.0.0.1
        smtp_connect_timeout 30
        router_id my_router
    }

    # vips app-master
    vrrp_instance app_master {
        state MASTER # Set this to BACKUP on the secondary server
        interface eth0
        virtual_router_id 36 # I usually use the last octect of the internal IP here, but anything is OK
        priority 100
        nopreempt
        advert_int 1
        garp_master_delay 5
        smtp_alert
        virtual_ipaddress {
            w.x.y.z # The VIP address, requested via RackSpace ticket
        }
    }

    # master app
    virtual_server w.x.y.z 80 { # VIP address
      delay_loop  1
      lvs_sched   wlc
      lvs_method  TUN
      protocol    TCP
      real_server a.b.c.d 80 { # Internal IP of our realserver
        weight 10
        TCP_CHECK {
          connect_port 80
          connect_timeout 15
        }
      }
      # Continue to add entries for each realserver
    }
    
You can erase the entry for the eth0:1 device which the RackSpace techs setup, since this will be automatically configured for you via keepalived.
    
You will also want to edit _/etc/sysctl.conf_ on the directors and add the following.  This setting allows the VIP to be shared between two machines.

    net.ipv4.ip_nonlocal_bind = 1
    
Then run the _sysctl -p_ command so the settings take effect.
    
### Realservers
    
Next you will need to configure the realservers and setup a tunnel device.  This allows the traffic to be forwarded from the director to the realserver, and the realserver to reply directly back to the client.  On CentOS 5.5 you would edit _/etc/sysconfig/network-scripts/ifcfg-tunl0_ and add the following.

    DEVICE=tunl0
    TYPE=ipip
    IPADDR=w.x.y.z # The VIP assigned to you by RackSpace
    NETMASK=255.255.255.255
    BROADCAST=w.x.y.z # Broadcast address is the same as the VIP
    ONBOOT=yes
    
Then bring it up with the _ifup tunl0_ command.

Next, in order to get around what is known as the [ARP Problem](http://www.austintek.com/LVS/LVS-HOWTO/HOWTO/LVS-HOWTO.arp_problem.html) we also need to edit the realserver's _/etc/sysctl.conf_.

Borrowed from the RackSpace LVS-TUN blog, it should look similar to this.

    net.ipv4.ip_forward = 0
    net.ipv4.conf.default.rp_filter = 1
    net.ipv4.conf.default.accept_source_route = 0
    kernel.sysrq = 0
    kernel.core_uses_pid = 1
    net.ipv4.tcp_syncookies = 1
    kernel.msgmnb = 65536
    kernel.msgmax = 65536
    kernel.shmmax = 68719476736
    kernel.shmall = 4294967296
    net.ipv4.conf.lo.arp_ignore = 1
    net.ipv4.conf.lo.arp_announce = 2
    net.ipv4.conf.tunl0.arp_ignore = 1
    net.ipv4.conf.tunl0.arp_announce = 2
    net.ipv4.conf.all.arp_ignore = 1
    net.ipv4.conf.all.arp_announce = 2
    net.ipv4.tcp_tw_reuse = 1
    net.ipv4.tcp_tw_recycle = 1
    net.ipv4.tcp_fin_timeout = 15
    net.ipv4.tcp_synack_retries = 2
    net.ipv4.tcp_max_syn_backlog = 2048
    net.ipv4.conf.all.rp_filter = 0
    net.ipv4.conf.default.rp_filter = 0
    net.ipv4.conf.lo.rp_filter = 0
    net.ipv4.conf.eth0.rp_filter = 0
    net.ipv4.conf.eth1.rp_filter = 0
    net.ipv4.conf.tunl0.rp_filter = 0
    
Then set with _sysctl -p_.

## Application Servers - Node, Nginx, and Monit

*UPDATE - 2011-03-4:* The Mongoose module had just reached it's 1.0 release, but still had a lot of incomplete features and bugs and had gone through 10 minor revisions within a week.  I'm still a huge fan of NodeJS and Mongoose, but had to set it aside this time around.  For posterity sake I will keep the following section intact, but want to point out that in it's place I setup Ruby Enterprise Edition + Passenger on Nginx and wrote the server logic using the super lightweight Ramaze framework plus MongoMapper.  I have to admit the Ruby fan in me had no regrets about going this direction.

The main goal here is to get multiple instances of our NodeJS server running on a single server instance, and proxying them behind an Nginx server.  The main reason for this is because RackSpace servers have 4 CPUs, and we would like to make use of all of them.  I am also not 100% confident in running NodeJS as a stable standalone process, so we want another layer in front of it in the case of failure.

### NodeJS

First of all we setup our NodeJS server.

    var http = require('http');
    http.createServer(function (req, res) {
      res.writeHead(200, {'Content-Type': 'text/plain'});
      res.end('Hello World\n');
    }).listen(8001);
    console.log('Server running on port 8001');
    
I also have some database code written with the [Mongoose](https://github.com/LearnBoost/mongoose) package to test connectivity to the database.  This becomes relevant after the database servers are brought up, but will be left as an exercise to the reader.
    
### Monit

I then use Monit to make sure that the NodeJS server processes continue to run even after a critical failure.  There are a few methods with which to set this up, and I am still experimenting with my settings to get it working correctly.  Currently my config looks like the following.

    # /etc/monit.d/my_project
    
    check host my_project01 with address 127.0.0.1 
        start program = "/usr/local/bin/node /path/to/project/server1.js"
            as uid nobody and gid nobody
        stop program = "/usr/bin/pkill -f '/usr/local/bin/node /path/to/project/server1.js'"
        if failed port 8001 protocol HTTP
            request /
            with timeout 10 seconds
            then restart

    check host my_project02 with address 127.0.0.1
        start program = "/usr/local/bin/node /path/to/project/server2.js"
            as uid nobody and gid nobody
        stop program = "/usr/bin/pkill -f '/usr/local/bin/node /path/to/project/server2.js'"
        if failed port 8002 protocol HTTP
            request /
            with timeout 10 seconds
            then restart
            
    # ... etc.
    
My eventual goal here is to write the server code with an optional port parameter as a command line option.

I looked into using [forever](http://blog.nodejitsu.com/keep-a-nodejs-server-up-with-forever) to keep the NodeJS processes running, but that is just a CLI tool and not a long running server process which comes back up after a server reset.

### Nginx

Finally we setup Nginx as our proxy balancer.  Apache is also a viable option, but in the end boils down to preference.

    # /etc/nginx/nginx.conf
    
    worker_processes  1;

    events {
        worker_connections  1024;
    }


    http {
        include       mime.types;
        default_type  application/octet-stream;

        sendfile        on;

        keepalive_timeout  65;

        upstream my_project {
            server 127.0.0.1:8001;
            server 127.0.0.1:8002;
        }

        server {
            listen       80;
            server_name  my_project;

            location / {
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                proxy_set_header Host $http_host;
                proxy_set_header X-NginX-Proxy true;
                proxy_pass http://my_project/;
                proxy_redirect off;
            }

        }
    }
    
This is a very basic config, and should probably be expanded upon.

### ipvsadm

We should now be be able to verify our connection externally via our VIP, and see traffic coming in through the director.

    [root@director01 ~]# watch ipvsadm -Ln
    
    Every 2.0s: ipvsadm -Ln                                 Thu Jan 20 09:06:40 2011

    IP Virtual Server version 1.2.1 (size=4096)
    Prot LocalAddress:Port Scheduler Flags
      -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
    TCP  VIP:80 wlc
      -> RIP:80                       Tunnel  10     1          0

If this step doesn't work, verify that you can access each realserver from it's public IP.  If that fails and your iptables rules are off, double check that the RackSpace techs have setup your realserver so I can be accessed using the assigned VIP.

## Database Servers - MongoDB Replica Sets (v1.6)

As of MongoDB v1.6 [Replica Sets](http://www.mongodb.org/display/DOCS/Replica+Sets) are now stable and production ready.  The basic idea is to allow for master/slave replication, failover and recovery, and in the case of master failure automatic election of a new master node.  Up to a total of 7 servers can automatically be added in case the database load needs to be scaled, and is as simple as typing a single command.

Ideally you want to start with three servers running, one master, one slave, and one backup which we turn off and backup periodically.

For those of you that use CentOS or Fedora, [packages are available](http://www.mongodb.org/display/DOCS/CentOS+and+Fedora+Packages) to install.

The only setting you need to add to the default config is your replication set name.

    # /etc/mongod.conf
    
    replSet = my_project
    
After that bring up all three database instances, and then log into any node with the _mongo_ command line tool.  We will now tell the cluster which servers to bring into the replication set.  You may want to watch the log output on all three servers as well with _tail -f /var/log/mongo/mongod.log_, I admit it was pretty awesome watching them negotiate who was the master and slaves for the first time.

    config = {
      _id: 'my_project', 
      members: [ 
        {_id: 0, host: 'db-01'},
        {_id: 1, host: 'db-02'},
        {_id: 2, host: 'db-03'}
      ] 
    }
    rs.initiate(config)
    
If you ever need to attach additional nodes, you can simply do the following.

    rs.add('db-04')
    
The mongo client drivers automatically learn new server addresses from the seed addresses they were set with, so no client updates are required when expanding your database cluster.

## Final Notes

### Considerations

Assuming all has gone well up to this point you will then want to harden your servers with _iptables_, disable any services that you don't need running, and batten down the hatches.

You will want to setup a database backup system and schedule, as well as logging so you can track down what your cluster is doing.

I also like to setup Nagios and Ganglia to monitor my services and see what kind of usage they are getting, although there are other tools and services which work just as well.

### Last Words

Feel free to contact me directly at _brian.jones at istpika dot com_ or throw a message [@mojobojo](http://twitter.com/mojobojo) on twitter if you have any questions you'd like to ask me directly, although please refrain from asking any general "you could have googled it" questions as they will be promptly ignored.

