---
title: "Syntax highlighting test page"
date: 2018-05-18T23:25:18+03:00
draft: true
tags: ["hidden"]
categories: ["notes"]

---


This is a test page i made in order to see if all 
syntax highlighting works ok:

* HTML
* Bash
* Json
* YAML
* Python

> For YAML to work, i added the following to the configured template, in my case:

~~~bash
hyde-hyde/layouts/partials/commenting.html
~~~


(HTML)
~~~html
<script src="//cdnjs.cloudflare.com/ajax/libs/highlight.js/9.12.0/languages/yaml.min.js"></script>
<script type="text/javascript">
    hljs.configure({languages: []});
    hljs.initHighlightingOnLoad();
</script>
~~~

(BASH)
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
(JSON)
~~~json
[
  {
    "title": "haim",
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
(YAML)
~~~yaml
string_1: "Bar"
string_2: 'bar'
string_3: bar
~~~
(PYTHON)
~~~python
from something import something
test = "1"
def something():
    print "OK"
~~~
