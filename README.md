# API Gateway

## Background

This is the result of discussions had with my colleagues
[Tomas](https://github.com/tvestelind) and
[Carl](https://github.com/chelmertz).

We wanted an API Gateway that did not lock us into a framework or language, so
that each endpoint may be implemented in the manner that is most appropriate
for its purpose. In addition it may help when a framework (possibly language)
fails, to not be completely invested in it, as a smaller component is easier to
translate to another framework/language.

We also wanted to reuse as much as possible, i.e. not implement anything
ourselves if there is something already existing that can be seen as
well-supported.

## Apache?

Apache was chosen simply due to my knowledge being greater in Apache than
Nginx, though after consideration it is likely this will be moved over to
Nginx. Mostly due to its architecture focusing more around being a proxy-server
and the performance benefits this gives. Though I would have to check this more
thoroughly before making the move.

## Getting started

Simply clone this repository and run:

```
make && ./api-gateway
```

## History

Just how I got around with what you are presented with

### Apache

I started with finding an apache container image, and located one that is
distributed by CentOS, i.e. it was the one provided on their github account.
This apache server rolls on a CentOS 7 and uses their run script in order to
start the apache server without using systemctl.

This apache cotainer works fine exposed on port 8080 of the host machine,
and I added a COPY statement in order to get some apache config into the
container for activating mod_proxy which was to be used for the endpoint
containers.

```
<IfModule !proxy_module>
  LoadModule proxy_module modules/mod_proxy.so
</IfModule>

ProxyRequests on
```

The directory structure now looks like:

* pgi
  * pgi
    * conf.d
      * api.conf
    * cccp.yml
    * Dockerfile
    * LICENSE
    * README.md
    * run-httpd.sh

### Node endpoint

I then created a node container running FROM the official node image and adding
a simple express catch-all app that binds to port 10000. I now needed to proxy
requests to the apache server on location /api/node.

So I added an apache config file to the endpoint's directory structure that
specifies how to proxy requests from the apache container to the node
container.

The node endpoints httpd conf
```
<Location /api/node>
        ProxyPass http://172.17.0.1:10000
        ProxyPassReverse http://172.17.0.1:10000
</Location>
```

But in order to get it into apache I decided that I would make the
folder containing all endpoints (repos) a shared volume for the apache
container.

Addition to the apache httpd conf
```
IncludeOptional /srv/repos/*/httpd/*.conf
```

With this configuration made I could now start the apache container and the
node container, send requests to the apache container on port 8080 and path
/api/node and it would proxy them to the node containers server process and
respond.

The directory structure now looks like:

* pgi
  * pgi
    * conf.d
      * api.conf
    * cccp.yml
    * Dockerfile
    * LICENSE
    * README.md
    * run-httpd.sh
  * repos
    * node
      * httpd
	* node.conf
      * Dockerfile
      * package.json
      * server.js

### Python endpoint

So this worked fine for a node endpoint, some hardcoded adresses and on-the-fly
shared volumes, but hey, if it works it works. So I wanted to try with another
language endpoint and see if there was any difference (in retrospect the
behaviours of node and python servers do not differ that much).

So I pulled the python3 image and took the catch-all-routes Flask example and
made it a container. Starting up this container on port 11000 with the others
works just fine.

The python endpoint httpd conf
```
<Location /api/node>
        ProxyPass http://172.17.0.1:11000
        ProxyPassReverse http://172.17.0.1:11000
</Location>
```

* pgi
  * pgi
    * conf.d
      * api.conf
    * cccp.yml
    * Dockerfile
    * LICENSE
    * README.md
    * run-httpd.sh
  * repos
    * node
      * httpd
	* node.conf
      * Dockerfile
      * package.json
      * server.js
    * python
      * httpd
	* python.conf
      * Dockerfile
      * requirements.txt
      * main.py

### Making it easier to manage

At this point I realized that starting up the different containers one-by-one
manually was more work than it was worth, remembering mentions of
docker-compose and docker-swarm I thought I'd have a look through them to see
if they would solve my problems.

Compose seemed to be the way forward by a configuration file defining container
relations and executing multiple setups. I set this file to spin up the apache,
node and python containers, sharing the endpoints directory as a volume into
the apache container, forwarding ports and setting a host machine address as an
environment variable for all containers (instead of hardcoding it in each
endpoint).

Can now start the entire thing using
```
docker-compose build
docker-compose up
```

Directory structure
* pgi
  * docker-compose
  * README.md
  * pgi
    * conf.d
      * api.conf
    * cccp.yml
    * Dockerfile
    * LICENSE
    * README.md
    * run-httpd.sh
  * repos
    * node
      * httpd
	* node.conf
      * Dockerfile
      * package.json
      * server.js
    * python
      * httpd
	* python.conf
      * Dockerfile
      * requirements.txt
      * main.py

### PHP Endpoint

PHP is a little more special in its setup due to PHP processes being setup and
torn down for each request, so to keep it alive in a docker container it forced
me into a PHP-FPM setup, which interfaces against FCGI. SO while the container
setup is basically the same the apache config is a little different.

```
<IfModule !proxy_fcgi_module>
  LoadModule proxy_fcgi_module modules/mod_proxy_fcgi.so
</IfModule>

<IfModule !rewrite_module>
  LoadModule rewrite_module modules/mod_rewrite.so
</IfModule>

<Location /api/php>
        RewriteEngine on
        RewriteBase /
        # NC = case-insensitive, L = Last rule
        # QSA = merge query string, P = Proxy
        RewriteRule ^(.*)$ fcgi://${DOCKER_HOST}:9000/usr/src/app/index.php [NC,L,QSA,P]
        DirectoryIndex /index.php index.php
</Location>
```
While the other endpoints get perfect paths to route for their endpoint. e.g. a
request to *localhost:8080/api/node/foo/bar* gives the node endpoint the path
*/foo/bar*. This FCGI approach does not, instead the entirety of the path
*/api/php/foo/bar* is passed to the PHP process. I do believe that can be
remedied but in any case it can be managed within the PHP application logic.

The directory structure is now:
* pgi
  * docker-compose
  * README.md
  * pgi
    * conf.d
      * api.conf
    * cccp.yml
    * Dockerfile
    * LICENSE
    * README.md
    * run-httpd.sh
  * repos
    * node
      * httpd
	* node.conf
      * Dockerfile
      * package.json
      * server.js
    * python
      * httpd
	* python.conf
      * Dockerfile
      * requirements.txt
      * main.py
    * php
      * httpd
	* php.conf
      * fpm
	* api.conf
      * Dockerfile
      * index.php
      * run-service.sh

### Reducing dead weight

At this point everything was running as I wanted, except that the node and
python services can be accessed directly on their respective ports over HTTP
but PHP is running via the FCGI protocol.

But going back to the first container that was setup, it was a bit to heavy,
basing on CentOS seemed unnecessary. So instead I sought out the pure apache
container and simply injected the required httpd conf into that one instead,
reducing some overhead.

However, learning a bit more you can minimize the footprint of every container
even further by using -alpine images, so I updated each and everyone to base on
-alpine versions, making adjustments where needed.

The document structure is now:
* pgi
  * docker-compose.yml
  * LICENSE
  * README.md
  * pgi
    * Dockerfile
    * httpd.conf
  * endpoints
    * node
      * Dockerfile
      * httpd
        * node.conf
      * package.json
      * server.js
    * php
      * Dockerfile
      * fpm
        * api.conf
      * httpd
        * php.conf
      * index.php
    * python
      * Dockerfile
      * httpd
        * python.conf
      * main.py
      * requirements.txt

### Added makefile

To handle the small steps to get everything up and running, in addition to
building a better volume to share with the gateway container, instead of
sharing everything.

And this is where the story end for now, up next is swagger and swagger-combine
from multiple containers.

### Moved to nginx

Because
