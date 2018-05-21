---
title: "Update DNS Records with Dyn API"
date: 2015-12-27T21:40:29+03:00
draft: false
tags: ["python","dyn","api"]
categories: ["Python", "Dyn", "Scripts"]
---

##### Here is a script that should run on an AutoScale instance when it launches and terminates in order to set and remove hostname by the instance-id.




This script will:
1. Update the instance hostname (locally) by instance id
2. Create Dyn DNS records for public and private IPs, then publish zone
3. Update the Instance Name in AWS by Instance-id + domain
4. removes records from Dyn Dns when instance terminates.

Your config.cfg file should look like this:


~~~ini
#API Login Information
[login]
cn: your-customer-name-here
un: api-username-here
pw: api-user-pass-here
~~~

> Notice the modules required for this script to run on AWS Linux instance.
Make sure your AMI has them pre-installed or [Pull them at launch]

~~~python

#!/usr/bin/python

from dynect.DynectDNS import DynectRest
from subprocess import call
import ConfigParser
import urllib2
import httplib2
import logging
import json
import boto
import sys

http = httplib2.Http()
http.force_exception_to_status_code = True

__author__ = "Haim Ari"
__license__ = "GPL"
__version__ = "0.0.1"

zone = 'yourdomain.com'
ttl = '360'
prefix = 'autoscale-prefix'
vpc = 'vpc'
pub = 'pub'

aws_access_key_id = 'XXXXXXXXXXXXXXXXXXXX'
aws_secret_access_key = 'YYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYYY'
ec2_region = ["us-east-1"]

logging.basicConfig(filename='/var/log/Update_Dyn.log', level=logging.INFO,
                    format='%(asctime)s %(message)s',
                    datefmt='%m/%d/%Y %I:%M:%S %p')
logging.getLogger("requests").setLevel(logging.WARNING)


def get_instanceid():
    instreq = urllib2.Request('http://169.254.169.254/latest/meta-data/instance-id')
    resp = urllib2.urlopen(instreq)
    instanceid = resp.read()
    return instanceid


def public_ip():
    pubipreq = urllib2.Request('http://169.254.169.254/latest/meta-data/public-ipv4')
    respip = urllib2.urlopen(pubipreq)
    pubip = respip.read()
    return pubip


def local_ip():
    localipreq = urllib2.Request('http://169.254.169.254/latest/meta-data/local-ipv4')
    resplocalip = urllib2.urlopen(localipreq)
    localip = resplocalip.read()
    return localip

instanceid = get_instanceid()
publicip = public_ip()
localip = local_ip()
fullpublicname = (prefix + '-' + pub + '-' + instanceid + '.' + zone)
publicname = (prefix + '-' + pub + '-' + instanceid)
fullprivatename = (prefix + '-' + vpc + '-' + instanceid + '.' + zone)
privatename = (prefix + '-' + vpc + '-' + instanceid)


def update_resolve():
    logging.info("OK: Updating resolv.conf file")
    with open("/etc/resolv.conf", "a") as myfile:
        myfile.write("search yourdomain.com")
        myfile.close()


def update_hostname():
    logging.info("OK: Updating hosts file")
    instanceid = get_instanceid()
    localip = local_ip()
    call(["hostname", fullprivatename])
    with open("/etc/hosts", "a") as hostfile:
        hostfile.write(localip + " " + privatename + " " + fullprivatename)
        hostfile.close()


def get_creds():
    """function to read API credentials from file"""
    try:
        # initialize config parser
        config = ConfigParser.ConfigParser()
        config.read('/your/path/config.cfg')
        # stash in dict
        creds = {
            'customer_name': config.get('login', 'cn'),
            'user_name': config.get('login', 'un'),
            'password': config.get('login', 'pw'),
        }
        return creds
    except:
        raise


def dynconnect():
    """Connect to Dyn REST API"""
    # read API credentials from file
    try:
        creds = get_creds()
    except:
        sys.exit('Unable to open configuration file: config.cfg')

    dyn_iface = DynectRest()

    # Log in

    response = dyn_iface.execute('/Session/', 'POST', creds,)
    if response['status'] != 'success':
        print ("Could not authenticate")
        # print response
        sys.exit("Unable to Login to DynECT DNS API.  Please check credentials in config.cfg")
    else:
        token = ((response)['data']['token'])
    return token


def add_ip_record(type):
    token = dynconnect()
    if type == "public":
        logging.info("OK: Adding A Record to Dyn with my public ip")
        data = {'rdata':{'address': publicip}, 'ttl': ttl}
        req = urllib2.Request('https://api2.dynect.net/REST/ARecord/' + zone + '/' + fullpublicname + '/')
    else:
        logging.info("OK: Adding A Record to Dyn with my private ip")
        data = {'rdata':{'address': localip}, 'ttl': ttl}
        req = urllib2.Request('https://api2.dynect.net/REST/ARecord/' + zone + '/' + fullprivatename + '/')
    data_json = json.dumps(data)
    req.add_header('Auth-Token', token)
    req.add_header('content-type', 'application/json')
    response = urllib2.urlopen(req, data_json, timeout=30)
    result = response.read()
    print (result)
    publish_zone(token)


def delete_record(type):
    token = dynconnect()
    if type == "public":
        logging.info("OK: Removing A Record from Dyn with my public ip")
        delreq = urllib2.Request('https://api.dynect.net/REST/ARecord/' + zone + '/' + fullpublicname)
    else:
        logging.info("OK: Removing A Record from Dyn with my private ip")
        delreq = urllib2.Request('https://api.dynect.net/REST/ARecord/' + zone + '/' + fullprivatename)
    delreq.add_header('Auth-Token', token)
    delreq.add_header('content-type', 'application/json')
    delreq.get_method = lambda: 'DELETE'
    delete = urllib2.urlopen(delreq, timeout=30)
    print delete.read()
    publish_zone(token)


def publish_zone(token):
    logging.info("OK: publishing Zone changes ")
    pubdata = {'publish':'true'}
    pubdata_json = json.dumps(pubdata)
    pubreq = urllib2.Request('https://api.dynect.net/REST/Zone/yourdomain.com/')
    pubreq.add_header('Auth-Token', token)
    pubreq.add_header('content-type', 'application/json')
    pubreq.get_method = lambda: 'PUT'
    publish = urllib2.urlopen(pubreq, pubdata_json, timeout=30)
    print publish.read()


def update_instance_name():
    c = boto.connect_ec2(aws_access_key_id, aws_secret_access_key)
    c.create_tags([instanceid], {"Name": fullprivatename})
    logging.info("OK: Updating my instance name by my instance id")


if sys.argv[1] == "start":
    update_hostname()
    update_instance_name()
    add_ip_record(type="public")
    add_ip_record(type="private")
elif sys.argv[1] == "stop":
    delete_record(type="public")
    delete_record(type="private")
else:
    print "Error"
~~~


