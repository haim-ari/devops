---
title: "Auto Scale your services with AWS Auto Scaling"
date: 2015-09-29T21:40:29+03:00
draft: false
tags: ["aws","autoscale"]
categories: ["DevOps", "Scripts"]
---

> Auto Scale

In this post i’ll describe how i use AWS Auto Scale in our environment.

If you have not read about AWS Auto Scale Group yet, i recommend reading this document 

Now that you know what is Auto-Scaling and what are the features and benefits, lets go over it.

In general what Auto-scale means is that instances will launch automatically by the policy you define, and will start serving by adding themselves to your
Auto Scale Group ELB.
These instances will serve until they are no longer needed (again your policy and “min” + “desired” will change the number of running instances inside the group)

Here are the steps i took go get an Auto Scale Group up and running:

1. Create your AutoScale Security Group.
2. Create your ELB.
3. Create your instance AMI.
4. Create your AutoScale Launch Configuration.
5. Configure your AutoScale Group.

![AutoScale-Group](/img/asgroup.png)


The above screenshost is the AWS side of Auto-Scaling, however this is not the full picture.
You also need to configure your instances launch sequence.
Your instances will have to:

1. Create a DNS fqdn record and assign to the hostname.
2. Register to your monitoring systems (in our Env – Zabbix)
3. Register to your configuration management system (Puppet/Chef…) and pull configuration as needed.
4. Pull your current production build.
5. Start Serving.

> There is also the termination process in which you want the instance to delete itself from all the above before termination.
This will keep your systems clean while containing only the currently running instances at all time.


##### Set it up

1. I recommend using Git / SVN / BitBucket to hold your Auto-Scale configuration. combined with “User Data” it will allow you to update your boot sequence easily:
The “User Data” runs only at first launch and will not run again if the instance will reboot.

> To my opinion AutoScaled instances should not be rebooted. if a reboot required, simply launch new instances replacing the old ones.
Any fix or change should be outside of your IMG and pulled at launch. if you must, update and create new IMG.


2. Create your AMI: set up one instance that has your application working on it with all dependencies.
3. I Pull one script from Git that runs other boot scripts (after pulling them too) and logs the boot process.
In our environment It also pulls the Puppet modules from Git. this allows to run puppet classes locally.
4. Once all dependencies are applied, and your scripts are pulled to the instance, run them to pull your latest build and start serving.
5. in your ELB you must configure your “Health check” and instance port, so the ELB could determine if the instance can serve requests and should be in status:
“In Service”

#### Scripts and examples

##### Set the hostname as the AWS instance id and update DNS (Bind)


~~~bash
#!/bin/bash
# Update-Hostname Init script to set the hostname
# chkconfig: - 84 02
# description: Init script to set the hostname as the AWS instance id and update DNS (Bind)
# Source function library.
. /etc/init.d/functions

/bin/touch /var/lock/subsys/Remove-Hostname
echo "search yourdomain.com" >>/etc/resolv.conf #

DNS_KEY=/usr/local/ec2/Kupdate_key.+*******.private
DOMAIN=yourdomain.com
PREFIX=virginia-web-prefix
VPC=vpc
PUB=public


#USER_DATA=`/usr/bin/curl -s http://169.254.169.254/latest/user-data`
# we are using the PUBIP instead if getting 'user-data' here in order to generate the hostname

HOSTNAME=`curl http://169.254.169.254//latest/meta-data/instance-id`
#set also the hostname to the running instance
hostname $PREFIX-$VPC-$HOSTNAME.$DOMAIN

PUBIP=`/usr/bin/curl -s http://169.254.169.254/latest/meta-data/public-ipv4`
cat<> /etc/hosts

/usr/bin/aws ec2 create-tags --resources $HOSTNAME  --tags Key=Name,Value=$PREFIX-$VPC-$HOSTNAME.$DOMAIN
~~~

* To summarize, auto scale is a great service.
Once the first time setup is done and you will start using it in your production environment
It will pay off, the bottom line is: using auto scale will for sure save you time once it’s up an running.


