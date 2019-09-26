Before using **`wsadmin`**, make sure the WebSphere Application Server instance is running.

In the below lab, a WAS Docker image is used to create a container as the running environment. In the image, WebSphere Application Server is installed in folder *`/opt/IBM/WebSphere/AppServer`*. And a deployment manager profile has been created.

> run container and start server
```sh
shu@PTDT232 MINGW64 ~
$ winpty docker run --rm -it wasnd855-dmgr:centos7 bash -il
[wasadmin@389b001a4c53 /]$
[wasadmin@389b001a4c53 /]$ cd /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/config/cells/DefaultCell01/nodes/DefaultNode01/servers/dmgr
[wasadmin@389b001a4c53 dmgr]$ cd ../../../../nodegroups/DefaultNodeGroup/
[wasadmin@389b001a4c53 DefaultNodeGroup]$ ls
nodegroup.xml
[wasadmin@389b001a4c53 DefaultNodeGroup]$ pwd
/opt/IBM/WebSphere/AppServer/profiles/Dmgr01/config/cells/DefaultCell01/nodegroups/DefaultNodeGroup
[wasadmin@389b001a4c53 DefaultNodeGroup]$ cd /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/bin/
[wasadmin@389b001a4c53 bin]$ ./startServer.sh dmgr
ADMU0116I: Tool information is being logged in file 
            /opt/IBM/WebSphere/AppServer/profiles/Dmgr01/logs/dmgr/startServer.log
ADMU0128I: Starting tool with the Dmgr01 profile
ADMU3100I: Reading configuration for server: dmgr
ADMU3200I: Server launched. Waiting for initialization status.
ADMU3000I: Server dmgr open for e-business; process id is 325
```

> Open `wsadmin` interactive mode specifying `jython` language
```sh
[wasadmin@389b001a4c53 bin]$ ./wsadmin.sh -lang jython
WASX7209I: Connected to process "dmgr" on node DefaultNode01 using SOAP connector;  The type of process is: DeploymentManager
WASX7031I: For help, enter: "print Help.help()"
```

> so far so good, issue commands:
```sh
wsadmin>AdminConfig.list('Cell')
```
>> the output is: 
`'DefaultCell01(cells/DefaultCell01|cell.xml#Cell_1)'`

```sh
wsadmin>import os
wsadmin>nl=os.linesep
wsadmin>AdminConfig.list('Node').split(nl)
```
>> the output is: `['DefaultNode01(cells/DefaultCell01/nodes/DefaultNode01|node.xml#Node_1)']`

```sh
wsadmin>cellID=AdminConfig.list('Cell')
wsadmin>print cellID
```
>> the output is: `DefaultCell01(cells/DefaultCell01|cell.xml#Cell_1)`

```sh
wsadmin>cellName=AdminConfig.showAttribute(cellID,'name')
wsadmin>print cellName
```
>> the output is: `DefaultCell01`
