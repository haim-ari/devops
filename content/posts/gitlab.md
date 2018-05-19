---
title: "Gitlab"
date: 2018-05-18T23:25:18+03:00
draft: true
---

This is My Test Page
~~~bash
#!/bin/bash

###### CONFIG
ACCEPTED_HOSTS="/root/.hag_accepted.conf"
BE_VERBOSE=false

if [ "$UID" -ne 0 ]
then
 echo "Superuser rights required"
 exit 2
fi

genApacheConf(){
 echo -e "# Host ${HOME_DIR}$1/$2 :"
}

~~~

~~~json
[
  {
    "title": "apples",
    "count": [12000, 20000],
    "description": {"text": "...", "sensitive": false}
  },
  {
    "title": "oranges",
    "count": [17500, null],
    "description": {"text": "...", "sensitive": false}
  }
]
~~~

~~~yaml
string_1: "Bar"
string_2: 'bar'
string_3: bar
~~~

~~~python
from something import something
test = "1"
def something():
    print "OK"
~~~
