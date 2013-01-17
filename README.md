nginx.filedist
==============

A simple HTTP based solution to manage files across machines. Mainly it's a generator for nginx.conf files. Based on [HttpEchoModule](http://wiki.nginx.org/HttpEchoModule) and [ngx_lua](https://github.com/chaoslawful/lua-nginx-module). We're just using a build of [OpenResty](http://openresty.org/). Uses Ruby to generate the files.

When running nginx with the generated configuration files you basically get a HTTP PUT/DELETE/GET interface to the local filesystem of the participating machines. PUT and DELETE requests are broadcasted to all machines. GET requests are distributed (via round robin) to the respective child nodes, if the GET request can't be satisfied locally.

We where looking for a solution to get uploaded MP3s onto our streaming backends. Files are accessed on the streaming backends via the normal filesystem. Top requirement was:

* Local access throughput must not be compromised by remote management:
  * Local access throughput over local consitency.
  * Local access throughput over remote availability.
  * Local access throughput over remote concurrency.
  * Local access throughput over remote throughput.

Requirements
------------

* [nginx](http://wiki.nginx.org/Main)
  * [HttpEchoModule](http://wiki.nginx.org/HttpEchoModule)
  * [ngx_lua](https://github.com/chaoslawful/lua-nginx-module)
* [Ruby](http://www.ruby-lang.org/)
* [wget](http://www.gnu.org/software/wget/)

Features
--------

* Fileservers are organised in a tree with one master endpoint, the root and as many children as you like.
* Children may have children theirself.
* Software on parent node and child nodes is identical.
* No action is required on the parent node after a child node reboot.
* When reading it suffices if one of the nodes actually has the file.
* Write and delete errors are logged in a missing log. This log itself is accessable via HTTP.
* Catch up for missing files can happen via HTTP crawling of the missing logs.
* Provisioning of new boxes can happen via HTTP crawling of index files.

Techical Details
----------------

* PUT requests body gets written to the file corresponding to the URL and get broadcasted to all child nodes.
* DELETE requests delete the file corresponding to the URL and get broadcasted to all child nodes.
* GET requests are satisfied by local filesystem or - when this fails - per round robin to the child nodes. If a child node returns a 404, the next node is tried.
* PUTs and DELETEs which fail to get broadcasted to a child nodes are written as HTML log to a file at `/__missing__` which then can be crawled (e.g. by wget) to catch up.
* For provisioning of a new machine you can generate one (or many) html file with links to all the files and put it into `/__index__`. How the file is created depends on your needs. This file can then be crawled (e.g. by wget). Consider throttling wget when doing this.
* Example commands for PUT and DELETE, catch up and provisioning are available at `/__readme.txt`

The bad and the ugly
--------------------

* Needs a client_body_buffer_size as large as your largest files. The whole file is kept in memory. That's 25MB in our case which is not an issue. Really large files will be a problem. This is a problem induced by my the implementation, not by the concept. I considered using the upload module and a systemcall for subrequests to avoid this. As we didn't really need this I left it as (for now).

Synopsis
--------

    nginx.filedist -u niko -i /tmp/nginx.[SERVERNAME].pid -l /tmp -r /tmp/foo2 -p /export/webroot/lautfm/var/ -s localhost:8082 conf/nginx.conf.erb > conf/nginx-child2.conf
    nginx.filedist -u niko -i /tmp/nginx.[SERVERNAME].pid -l /tmp -r /tmp/foo1 -p /export/webroot/lautfm/var/ -s localhost:8081 conf/nginx.conf.erb > conf/nginx-child1.conf
    nginx.filedist -u niko -i /tmp/nginx.[SERVERNAME].pid -l /tmp -r /tmp/foo -p /export/webroot/lautfm/var/ -s localhost:8080 -c localhost:8081,localhost:8082 conf/nginx.conf.erb > conf/nginx-master.conf

Then look at std error output or visit http://localhost:8080/__readme.txt.

