# Getting Node id, Cell ID, Server ID of Mbean in wsadmin
https://jython4wsadmin.blogspot.com/2014/08/getting-id-of-mbean.html

## Basic understanding of wsadmin objects
Node – an individual system, either physical or virtual
Node Manager – the process controlling the individual node and all servers in all cells, it executes the commands of the Deployment Manager
Profile – a WebSphere entity similar to a node concept wise
Application Server – a Java Virtual Machine (JVM) process
Application – a Java enterprise application it could be .ear file with some modules
Cluster – a group of Servers, all running the same applications
Cell – an administrative domain of one or more servers
Deployment Manager (DM) – the administration application for a cell
How do all these pieces work together? “Multiple Nodes in a Cell run Servers that contain Applications. All pieces are controlled via the Deployment Manager”

## Retrieving  Node ID

This task is very much required for most of the create method executions. When you create servers, when you construct Cluster Member servers your script need Node id. Ofcourse if you wish to Sync the differe!!nt members of a node also requird the Node id.

wsadmin>AdminConfig.getid('/Node:DmgrNode05')
'DmgrNode05(cells/DmgrCell05/nodes/DmgrNode05|node.xml#Node_1)'

## Retrieving Cell Name 
wsadmin>CellName=AdminControl.getCell()

## Retrieving Cell ID
wsadmin>AdminConfig.getid('/Cell:'+CellName)
'DmgrCell05(cells/DmgrCell05|cell.xml#Cell_1)'

## Retrieving Servers list
wsadmin>AdminTask.listServers('[-serverType APPLICATION_SERVER]')
'server1(cells/DmgrCell05/nodes/AppSrvNode/servers/server1|server.xml)'

## Retrieving Server ID

wsadmin>AdminConfig.getid('/Server:server1')
'server1(cells/DmgrCell05/nodes/AppSrvNode/servers/server1|server.xml#Server_1407376951191)'
wsadmin>

Diffrent automation scripts requires the wsadmin provided MBean values. so that we can store in the Jython variables like list and use them.

Keep writing your comments and thoughts on this blog 