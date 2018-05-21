---
title: "Ansible (Tower) as your rolling deployment system"
date: 2015-09-25T00:23:42+03:00
draft: false
tags: ["ansible","python"]
categories: ["DevOps"]
---

Ansible is a great tool to manage your servers.
The ability to launch commands at the same time on all or some of your agent-less hosts makes it very powerful.
The open source part of Ansible allows you to run any command/task/playbook/role on your hosts by using commands.
If you want to create a pipeline there is no such option… that is where Ansible Tower comes in.

Ansible Tower allows you to easily use the same playbooks you created in a UI interface.
You can configure templates and Inventories and assign permissions for teams to use them within a UI interface.



![Tower Ui](/img/Screenshot-from-2015-09-25-212031.png)

With custom scripts you can sync your inventory (hosts list) before running a template so that if you have an autoscale group where instances are constantly changing
It won’t be an issue for you.

It only took a few days to set up a Production Deployment Tool for our RND, and it made build upgrade easier and most importantly faster.
One click on a template will deploy builds on multiple servers at the same time. how many is configurable for each group.

There are some flaws with tower but they are minor considering the advantage of not having tun run multiple actions over and over again in order to deploy build on
All of your hosts

Ansible Tower License costs money
Tower UI is not running so fast, and requires “refresh” most of the time.
If all host at the same iteration will fail the task will stop completely
Other technical limitations (some of them are already planned to get fixed on next releases)
Except from the above i personally find Tower to be an excellent tool, and mostly a time saver for managing your hosts.
It is not puppet or Chef, but it can work with them, and do things that would take much more effort to do with them.

In Our environment there are multiple systems that configure and monitor our servers.
1. Foreman
2. Puppet
3. Git
4. Zabbix

The environment is distributed between an on premise servers and AWS Sites (AutoScale)
I used tower and the above APIs to orchestrate a build deployment, and it was not so complicated.

> Each company has its own environment which could reside on different products.
However i’ll show you what i did at the above environment, in order to get Tower to deploy builds with a single click.


What i wanted to do is prepare a template which will run the same role on different groups of hosts by their location & service.
The inventory which the template runs on, will sync from a custom script that pulls the hosts list from zabbix-server each time a template runs.
This will make sure that tower will not try to run on terminated AWS Instances and will also run on all newly created instances as well.

In my case i use python to pull hosts in groups and return the Group Name + node list in json.
* note that ansible runs the script with “- -list” parameter so include it in your script.

Tower will provide you the zabbix sync script with installation. but it will return all of your monitored hosts

I chose to sync from zabbix since it always includes the updated list of instances and physical hosts from our data center.

I modified it as i needed.

Go to “Setup” and configure your custom script under “Inventory Scripts”


Now you can select it when you configure your “Host Group” under “Source”:


![Tower Ui](/img/Screenshot-from-2015-09-25-204354.png)

You can also use the already built in methods if they fir you.

The instances in our AWS AutoScale Groups are configured with a “meta” string in their zabbix.conf agent file in order to automatically register to their Zabbix-proxy and become a part of their Zabbix group.

This allows to create multiple “Inventories” easily and assign them to different “Templates”

Now create the playbook that you want to run in order to deploy a build.

> As mentioned above different companies got different ways for doing the same thing.

I’ll describe our flow:

* Builds run on jenkins
* Builds are uploaded to S3(AWS)
* The playbooks runs the following tasks in order to roll a build deployment:

1. Verify and install dependencies if required on the target host.
2. Get the current build that runs on the host.
3. Compare and fail if already running the deployment build (There is a limitation in Ansible Tower and you can’t skip the host… the task has to fail)
4. Put the host in maintenance in Zabbix.
5. Unregister the host from AWS ELB.
6. Wait for connection draining.
7. Stop tomcat.
8. Pull latest build from S3.
9. Start tomcat.
10. Test keepalive (retry every 2 seconds and wait 300 Secs until fail)
11. Test checkinit.
12. Test your service url and parse the response to verify it works as expected.
13. Cleanup old builds on the host (keep the last few build for backup)
14. Register to ELB.
15. Remove zabbix maintenance.

I recommend using “Role”, this way you can create different flows by including different tasks in it and use it as a template.

Now create your “Project” and “Template” and select your relevant playbook.yml file.

Create a “Team” in Tower and grant permission to the relevant templates. (for your R&D team)

Here is an example of a flow i use:

1. Verify httplib2 is installed and install if missing (required for http checks)
2. Check the current build on the server
3. Compare the desired build to the current build (optional)
4. Put the currently running host in maintenance at zabbix
5. If the host is an AWS instance remove the instance from it's Autoscale Group Load balancer
6. Change the keepalive check to "STOP" (so that it won't pass health check after restart)
7. Wait for connection draining
8. Deploy_latest_build (Pull from S3 and extracts it)
9. Restart tomcat service
10. Wait for open port
11. Wait for all checkinit statuses to be in "OK" status (retry for 600 seconds)
12. Check "gethtmlad" response and find a specific string in the response (meaning the response is valid)
13. Change the keepalive status to "OK" 
14. Register the host to ELB (again, case this is a AWS instance)
15. Cleanup old builds
16. Remove zabbix maintenance for the host.



##### As i see it there are two main advantages here (among many others):
1. You configure this once. after that its just “change that, update here” work.
2. Speed – you can run multiple templates at the same time and roll a deployment on all your regions at once.

Ansible and Ansible Tower got great documentation.You can find it here:

[Ansible]("https://www.ansible.com/")

[Ansible Tower]("https://www.ansible.com/products/tower")

