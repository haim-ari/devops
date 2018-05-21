---
title: "MasterLess Puppet for AWS autoscaled instances"
date: 2015-09-23T21:40:29+03:00
draft: false
tags: ["aws","puppet"]
categories: ["Puppet"]
---

#####Why i moved to MasterLess Puppet for our AWS autoscaled Groups ?
In general this is because having multiple puppet masters is not so fun, especially if you are using Foreman.
I don’t want to have another Puppet master in each site and having to sync all the facts between them.

MasterLess puppet has one disadvantage: any host/instance needs to have all modules available on itself.
Once you get use to that (it’s a one time configuration) the rest is easy and you are no longer depending on you Puppet master to apply modules.
Instead of pulling the Catalog from your Puppet Master to all of you geo-distributed hosts, you simply push your modules from your Puppet Master to Git / SVN / BitBucket, Pull them to the hosts and deploy them locally.

This method is also great when your Puppet Master is not available, cause in some cases this can break you AutoScaling dependencies.
If you are already using Puppet, you can go Masterless with a few easy steps:

1. Push your puppet files and modules to Git every X minutes (use Git / BitBucket / SVN)
2. Configure your instances to pull the puppet files from your repo and run “puppet apply” locally
3. the above is enough for the “First Launch” of an autoscale instance.

For continues update of the instances, create crons to run “pull” and “puppet apply” every x minutes.

##### Create repo, commit and push the puppet files:


~~~bash
cd /etc/puppet
git init
git add .
git commit -m "Initial commit of Puppet files"
git remote add origin git@your_server_ip:username/puppet.git
git push -u origin master
~~~
##### Configure the instance puppet.conf:
~~~ini
[main]
logdir=/var/log/puppet
vardir=/var/lib/puppet
ssldir=/var/lib/puppet/ssl
rundir=/var/run/puppet
factpath=$confdir/facter
templatedir=/your/templates/path
modulepath=/your/modules/path
~~~

##### Add a puppet module:

~~~puppet
class puppet-masterless_cron {

        service { "puppet":
                ensure => stopped,
                enable => false,
        } ->

        file { '/etc/cron.d/puppet-pull':
                ensure  => present,
                content => "*/4 * * * * root cd /etc/puppet ; /usr/bin/git fetch git@git.yourgit.com:DevOps.puppet.git ; /usr/bin/git checkout FETCH_HEAD -- environments/production/modules ; /usr/bin/git checkout FETCH_HEAD -- manifests/templates ; /usr/bin/git checkout FETCH_HEAD -- files\n",
                mode    => '0644',
                owner   => 'root',
                group   => 'root',
        } ->

        file { '/etc/cron.d/puppet-cron':
                ensure  => 'present',
                content => "*/5 * * * * root /usr/bin/puppet apply /etc/puppet/site.pp\n",
                mode    => '0644',
                owner   => 'root',
                group   => 'root',
        }
}
~~~


* Also add another module to pull your site.pp file. this way you can always add / remove new modules for all instances that uses this file.
* You can have a different file for each of your instances groups.

Now commit a module and push it from your Puppet Master.
See if it runs on your MasterLess instances.
