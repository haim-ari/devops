---
title: "Remove a host from zabbix using Bash script and Zabbix API"
date: 2015-09-27T22:44:29+03:00
draft: false
tags: ["zabbix","bash"]
categories: ["DevOps", "Scripts"]
---

If you are using AWS Autoscaling and zabbix as your monitoring server,
Here is a simple way to remove a host before it is terminated.
simply configure it to run at shutdown (/etc/rcd)

~~~bash
#!/bin/bash
# Remove-Zabbix Init script should run when an AWS instance goes down and remove itself from Zabbix Server
# chkconfig: - 84 02
# description: Remove from zabbix
# Source function library.
. /etc/init.d/functions

start() {
/bin/touch /var/lock/subsys/Remove-Zabbix
}

stop() {
        /etc/init.d/zabbix-agent stop
        /bin/rm -f /var/lock/subsys/Remove-Zabbix
        HOST_NAME=`echo $(hostname)`

        USER='Your_Api_User' 
        PASS='Zabbix_Password' 
        ZABBIX_SERVER='zabbix.server.com'
        API='http://zabbix.server.com/zabbix/api_jsonrpc.php'

        # Authenticate with Zabbix API

        authenticate() {
                echo `curl -s -H  'Content-Type: application/json-rpc' -d "{\"jsonrpc\": \"2.0\",\"method\":\"user.login\",\"params\":{\"user\":\""${USER}"\",\"password\":\""${PASS}"\"},\"auth\": null,\"id\":0}" $API`
        }

        AUTH_TOKEN=`echo $(authenticate)|jq -r .result`

        # Get This Host HostId:

        gethostid() {
               echo `curl -s -H 'Content-Type: application/json-rpc' -d "{\"jsonrpc\": \"2.0\",\"method\":\"host.get\",\"params\":{\"output\":\"extend\",\"filter\":{\"host\":[\""$HOST_NAME"\"]}},\"auth\":\""${AUTH_TOKEN}"\",\"id\":0}" $API`
        }

        HOST_ID=`echo $(gethostid)|jq -r .result[0].hostid`

        # Remove Host

        remove_host() {
                echo `curl -s -H 'Content-Type: application/json-rpc' -d "{\"jsonrpc\": \"2.0\",\"method\":\"host.delete\",\"params\":[\""${HOST_ID}"\"],\"auth\":\""${AUTH_TOKEN}"\",\"id\":0}" $API`
        }
        RESPONSE=$(remove_host)
        echo ${RESPONSE}
}

case $1 in

        start)
          start
        ;;

        stop)  
          stop
        ;;

        restart)
          stop
          start
        ;;

esac    
exit 0
~~~

