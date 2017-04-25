# Go Grid Router
[![Build Status](https://travis-ci.org/aerokube/ggr.svg?branch=master)](https://travis-ci.org/aerokube/ggr)
[![Coverage](https://codecov.io/github/aerokube/ggr/coverage.svg)](https://codecov.io/gh/aerokube/ggr)
[![GoReport](https://goreportcard.com/badge/github.com/aerokube/ggr)](https://goreportcard.com/report/github.com/aerokube/ggr)
[![Release](https://img.shields.io/github/release/aerokube/ggr.svg)](https://github.com/aerokube/ggr/releases/latest)
[![GoDoc](https://godoc.org/github.com/aerokube/ggr?status.svg)](https://godoc.org/github.com/aerokube/ggr)

Go Grid Router (aka Ggr) is a lightweight active load balancer used to create scalable and highly-available [Selenium](http://seleniumhq.org/) clusters.

## Quick Start Guide
To use Go Grid Router do the following:
1) Install [Docker](http://docker.com/) to host
2) Create configuration directory:
```
$ mkdir -p /etc/grid-router/quota
```
3) Create ```users.htpasswd``` file:
```
$ htpasswd -bc /etc/grid-router/users.htpasswd test test-password
```
4) Start Selenium standalone server on port 4445:
```
$ java -jar selenium-server-standalone.jar -port 4445
```
5) Create quota file (use correct browser name and version):
```
$ cat /etc/grid-router/quota/test.xml
<qa:browsers xmlns:qa="urn:config.gridrouter.qatools.ru">
<browser name="firefox" defaultVersion="45.0">
    <version number="45.0">
        <region name="1">
            <host name="localhost" port="4445" count="1"/>
        </region>
    </version>
</browser>
</qa:browsers>
```
***Note***: file name should correspond to user name you added to htpasswd file. For user ```test``` we added on previous steps you should create ```test.xml```.

6) Start Ggr container:
```
# docker run -d --name ggr -v /etc/grid-router/:/etc/grid-router:ro --net host aerokube/ggr:1.1.1
```
7) Access Ggr on port 4444 in the same way you do for Selenium Hub but using the following url:
```
http://test:test-password@localhost:4444/wd/hub
```

## Configuration
Ggr is using two types of configuration files: 
1) A single file to store user credentials - **users file**.
2) One **username.xml** file (**quota file**) for each user storing information about available browsers. 

### Creating Users File
Ggr is using [htpasswd](https://httpd.apache.org/docs/2.4/misc/password_encryptions.html) files to store authentication data. Passwords are stored in encrypted form. To create such file type:
1) Ensure you have ```htpasswd``` utility installed (e.g. from ```apache2-utils``` package on Ubuntu).
2) Create a new users file...
```
$ htpasswd -bc /path/to/new.htpasswd username password
```
... or update an existing one:
```
$ htpasswd -b /path/to/existing.htpasswd username password
```

### Creating Quota Files
1) Quota files define available browsers for each user. All quota files should be placed to the same directory. 
2) For user ```username``` quota file should be named ```username.xml```.
3) Each quota file contains the following XML:
```
<qa:browsers xmlns:qa="urn:config.gridrouter.qatools.ru">
<browser name="firefox" defaultVersion="45.0">
    <version number="45.0">
        <region name="1">
            <host name="host1.example.com" port="4444" count="1"/>
            <host name="host2.example.com" port="4444" count="1"/>
            ...
        </region>
        <region name="2">
            ...
        </region>
    </version>
    <version number="46.0">
        ...
    </version>    
</browser>
<browser name="firefox" defaultVersion="45.0">
    ...
</browser>
...
</qa:browsers>
```
Here we define a list of browser names, their versions and default version for each browser. Each version has one or more regions (in cloud term, i.e. data centers). Every region contains one or more hosts. Each host defined in XML should have Selenium listening on specified port. The XML namespace is needed to be fully compatible with [original](http://github.com/seleniumkit/gridrouter) Java GridRouter implementation.

### Configuration File Locations
1) Default users file locations are: ```.htpasswd``` for standalone binary and ```/etc/grid-router/users.htpasswd``` for Docker image.
2) Default quota directory location is ```quota``` for standalone binary and ```/etc/grid-router/quota``` for Docker image. This is why just attaching ```/etc/grid-router``` to container as read-only volume is enough.
3) To specify custom configuration file locations pass additional arguments to Ggr:
```
# ggr -quotaDir /path/to/quota/directory -users /path/to/.htpasswd # Standalone binary
# docker run -d --name ggr -v /etc/grid-router/:/etc/grid-router:ro --net host aerokube/ggr:1.1.1 -quotaDir /path/to/quota/directory -users /path/to/.htpasswd # Docker container
```

### Quota Reload and Graceful Restart
* To **reload quota files** just send **SIGHUP** to process or Docker container:
```
# kill -HUP <pid>
# docker kill -s HUP <container-id-or-name>
```
* To **gracefully restart** (without losing connections) send **SIGUSR2**:
```
# kill -USR2 <pid>
# docker kill -s USR2 <container-id-or-name>
```

## How it Works
See the full history in our article - **Selenium testing: a new hope**:
* [Part 1](https://hackernoon.com/selenium-testing-a-new-hope-7fa87a501ee9)
* [Part 2](https://hackernoon.com/selenium-testing-a-new-hope-a00649cdb100)

## Log Files
A typical log file looks like the following:
```
2017/04/18 03:52:36 [12413389] [SESSION_ATTEMPTED] [my_quota] [192.168.2.3] [firefox-42.0] [firefox42-1.example.com:4444] [1]
2017/04/18 03:52:40 [12413389] [SESSION_FAILED] [my_quota] [192.168.2.3] [firefox-42.0] [firefox42-1.example.com:4444] Error forwarding the new session Request timed out waiting for a node to become available.
2017/04/18 03:52:40 [12413390] [SESSION_ATTEMPTED] [my_quota] [192.168.2.3] [firefox-42.0] [firefox42-5.example.com:4444] [2]
2017/04/18 03:52:45 [12413390] [5.86s] [SESSION_CREATED] [my_quota] [192.168.2.3] [firefox-42.0] [firefox42-5.example.com:4444] [0c500a6f-98d2-4871-acb7-637d85e1416a] [2]
....
2017/04/18 03:53:05 [SESSION_DELETED] [192.168.2.3] [firefox42-5.example.com:4444] [0c500a6f-98d2-4871-acb7-637d85e1416a]
```
Every line contains:

| Field | Example | Notes |
| ----- | ------- | ----- |
| Time | 2017/04/18 03:52:36 | - |
| Request counter | [12413389] | Only present for new session requests. So far as session ID is unknown when doing attempts this counter is used to find all session attempts for each new session request. |
| Status | [SESSION_ATTEMPTED] | See table below for complete list of statuses. |
| Quota name | [my_quota] | Extracted from basic HTTP auth headers. |
| User IP | [192.168.2.3] | IPv4 or IPv6 address |
| Browser | [firefox-42.0] | Name and version. Only present for new session requests. |
| Hub host | [firefox42-1.example.com:4444] | Host from quota XML file |
| Attempt number | [1] | For SESSION_ATTEMPTED entries means current attempt number. For SESSION_CREATED entries means total number of attempts to create this session. |
| Session ID | [0c500a6f-98d2-4871-acb7-637d85e1416a] | As arrived from hub |
| Session start time | [5.86s] | - |
| Error | Error forwarding the new session Request timed out waiting for a node to become available. | Only present for SESSION_FAILED |

The following statuses are available:

| Status | Description |
| ------ | ----------- |
| BAD_JSON | User request does not contain valid Selenium data |
| BROWSER_NOT_SET | Browser name is not present or empty string |
| CLIENT_DISCONNECTED | User disconnected and doing session attempts was interrupted |
| INVALID_URL | Session ID does not contain information about host where it was created |
| ROUTE_NOT_FOUND | Trying to proxy session to unknown host. Usually means quota files inconsistency between multiple Ggr instances. |
| SESSION_ATTEMPTED | New user request for session arrived |
| SESSION_CREATED | A new session was created and returned to user |
| SESSION_DELETED | Existing session was deleted by user request |
| SESSION_FAILED | Session attempt on specified host failed |
| SESSION_NOT_CREATED | Attempts to create a new session on all hosts failed. An error was returned to user. |
| UNSUPPORTED_BROWSER | Requested browser name and version is not present in quota |

## Development
To build Ggr:

1) Install [Golang](https://golang.org/doc/install)
2) Setup `$GOPATH` [properly](https://github.com/golang/go/wiki/GOPATH)
3) Install [govendor](https://github.com/kardianos/govendor): 
```
$ go get -u github.com/kardianos/govendor
```
4) Get Ggr source:
```
$ go get -d github.com/aerokube/ggr
```
5) Go to project directory:
```
$ cd $GOPATH/src/github.com/aerokube/ggr
```
6) Checkout dependencies:
```
$ govendor sync
```
7) Build source:
```
$ go build
```
8) Run Ggr:
```
$ ./ggr --help
```
9) To build [Docker](http://docker.com/) container type:
```
$ GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build
$ docker build -t ggr:latest .
```
