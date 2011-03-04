---
layout: blog
title: RackSpace and iptables
summary: Battening down the hatches on RackSpace with iptables, plus an LVS-TUN caveat.
---

## {{ page.title }}

__4 Mar 2011 - Osaka, Japan__

## Introduction

This is a followup post to my last entry about [LVS - Nginx - NodeJS - MongoDB - Cluster Setup on RackSpace](http://boj.github.com/blog/2011/01/14/lvs-nginx-nodejs-mongodb-cluster-setup-on-rackspace/), and focuses on implementing iptables rules to lock down the cluster from both external and internal probing.  It's important to note that RackSpace's servers are wide open on all interfaces, and that other members in their internal network can sniff ports on your machine's internal interfaces.

I am by no means an iptables expert, and while I have used the tool for many reasons over the years it has never been with complete comprehension as to how it works beyond a superficial level.  Therefore, if anything in regards to my settings below seem to be off, strange, or makes no sense, please feel free to leave advice in the comments section below.  I am publishing this for myself as much as the next guy who needs a few hints about setting up iptables rules for things like LVS-TUN or RackSpace.

I will keep things simple and just breakdown my rule sets by their specific jobs.  I figure little to no explanation is needed.

### Note

It's very useful when working on a network server to remember to run the following rule first before anything else so you don't lock yourself out of your server.  RackSpace has a handy console which you can use via their browser interface should you kill your ssh connection, so this is a tip to cut down on hassle more than anything else.

    iptables -P INPUT ACCEPT
    
My rules also include the command _/sbin/service iptables save_ which is RHEL/CentOS specific and saves the configuration out for you.
    
### Admin Server Configuration

    iptables -P INPUT ACCEPT
    iptables -F
    iptables -A INPUT -p tcp --dport 22 -j ACCEPT
    iptables -A INPUT -p tcp --dport 80 -j ACCEPT
    iptables -A INPUT -p udp --dport 8646:8649 -j ACCEPT # gmond

    iptables -P INPUT DROP
    iptables -P FORWARD DROP
    iptables -P OUTPUT ACCEPT

    iptables -A INPUT -i lo -j ACCEPT

    iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

    /sbin/service iptables save

    iptables -L -v
    
The reason gmond runs on 4 ports will be explained in a later blog post about setting up Ganglia in unicast mode.

### Load Balancer Configuration

First you must edit /etc/services and add the following line.

    vrrp            112/raw
    
Then add your rules.

    iptables -P INPUT ACCEPT
    iptables -F
    iptables -A INPUT -p tcp --dport 22 -s x.x.x.x -j ACCEPT # Only allow access from the admin server
    iptables -A INPUT -p tcp --dport 80 -j ACCEPT
    iptables -A INPUT -p udp --dport 8649 -j ACCEPT # gmond
    iptables -A INPUT -p tcp --dport 8649 -j ACCEPT # gmond

    iptables -p vrrp -A INPUT -j ACCEPT
    iptables -p vrrp -A OUTPUT -j ACCEPT

    iptables -P INPUT DROP
    iptables -P FORWARD DROP
    iptables -P OUTPUT ACCEPT

    iptables -A INPUT -i lo -j ACCEPT

    iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

    /sbin/service iptables save

    iptables -L -v
    
The vrrp commands allow your LVS machines to broadcast to each other so they can take over in case the other fails.

### Application Server Configuration

    iptables -P INPUT ACCEPT
    iptables -F
    iptables -A INPUT -p tcp --dport 22 -s x.x.x.x -j ACCEPT # Only allow access from the admin server
    iptables -A INPUT -p tcp --dport 80 -j ACCEPT
    iptables -A INPUT -p udp --dport 8649 -j ACCEPT # gmond
    iptables -A INPUT -p tcp --dport 8649 -j ACCEPT # gmond

    iptables -P INPUT DROP
    iptables -P FORWARD DROP
    iptables -P OUTPUT ACCEPT

    iptables -A INPUT -i lo -j ACCEPT
    iptables -A INPUT -i eth1 -j ACCEPT

    iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

    /sbin/service iptables save

    iptables -L -v
    
The eth1 rule is to allow LVS to connect to the realserver.  Without it your connection will just hang at the director level.

### Database Cluster Configuration

    iptables -P INPUT ACCEPT
    iptables -F
    iptables -A INPUT -p tcp --dport 22 -s x.x.x.x -j ACCEPT # Only allow access from the admin server
    iptables -A INPUT -p tcp --dport 27017 -j ACCEPT # mongodb
    iptables -A INPUT -p tcp --dport 28017 -j ACCEPT # mongodb web stats
    iptables -A INPUT -p udp --dport 8649 -j ACCEPT # gmond
    iptables -A INPUT -p tcp --dport 8649 -j ACCEPT # gmond

    iptables -P INPUT DROP
    iptables -P FORWARD DROP
    iptables -P OUTPUT ACCEPT

    iptables -A INPUT -i lo -j ACCEPT

    iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

    /sbin/service iptables save

    iptables -L -v
    
## Conclusion

There's really not much different between the scripts except for a few token places specific to the server's services.  I will update this blog post if I discover better ways to tighten things down.

