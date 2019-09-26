# [Creating cluster using Jython wsadmin](https://jython4wsadmin.blogspot.com/2014/09/creating-cluster-using-jython.html)

## Pre requisites 
As per todays experiment what I understood is we need the following are the pre-requisites for the Cluster configuration in WebSphere Environment:
1. Cell created with a DMGR Profile
2. Nodeagents created and added to the cell.
3. Dmgr, Nodeagent must be running.

## keynotes
```sh
AdminTask.createCluster('[-clusterConfig [-clusterName samplecluster] -replicationDomain [-createDomain true]]')

AdminTask.createClusterMember('[-clusterName samplecluster -memberConfig [-memberNode dmgrNode01 -memberName SampleMember1 -memberWeight 2 -replicatorEntry true] -firstMember [-templateName "Sample Template" -nodeGroup DefaultNodeGroup -coreGroup DefaultCoreGroup]]')

AdminTask.createClusterMember('[-clusterName samplecluster -memberConfig [-memberNode sunpatilNode02 -memberName SampleMember2 -memberWeight 2 -replicatorEntry true]]')
```

This script creates `samplecluster` by calling *AdminTask.createCluster* command and then creates `SampleMember1` as cluster member using *AdminTask.createClusterMember* call. The *AdminTask.createClusterMember* is made for adding additional cluster members.

## one Jython Script sample for Cluster
```py
import sys
import os
import socket

#Get the operating system line separator from the linesep method of the os object
nl = os.linesep

# Store the Cell identifier for the specified Cell
cellID = AdminConfig.list('Cell')
cellName = AdminConfig.showAttribute(cellID, 'name')

#derives the environment type from the cellName
sName = cellName.replace("Cell","")
sName = sName.replace("Demo","")

# Store the Node identifier for the specified Node
nodeIDs = AdminConfig.list('Node').split(nl)
nodeNames = AdminTask.listNodes().split(nl)
nodeNames.sort()

# Identify each node and it's purpose so we can start creating servers and clusters on the
# appropriate nodes
for nid in nodeIDs:
 nname=AdminConfig.showAttribute(nid, 'name')
 nname=nname.replace(sName, "")
 if nname == "AppSrvNode":
  NID01=nid
 elif nname == "AppSrvNode02":
  NID02=nid
 continue
# Store the node names
Node1Name = AdminConfig.showAttribute(NID01, 'name')
Node2Name = AdminConfig.showAttribute(NID02, 'name') 
 
print Node1Name,Node2Name

# Create the cluster and store the cluster name
print "\nCreating the Demo cluster"
clusterID = AdminTask.createCluster('[-clusterConfig [-clusterName Demo' + sName + 'Cluster -preferLocal true]]')
clusterName = AdminConfig.showAttribute(clusterID, 'name')


# Creates all of the cluster members on the specified nodes
print "\nCreating the cluster members for the Demo cluster"
def createClusterMembers(nn01, nn02, clustName):
 i = 1
 while i <= 2:
  if i == 1:
   AdminTask.createClusterMember('[-clusterName ' + clustName + ' -memberConfig [-memberNode ' + nn01 + ' -memberName Demo' + sName + 'CM' + str(i) + ' -memberWeight 2 -genUniquePorts true -replicatorEntry false] -firstMember [-templateName default -nodeGroup DefaultNodeGroup -coreGroup DefaultCoreGroup]]')
  elif i > 1:
   AdminTask.createClusterMember('[-clusterName ' + clustName + ' -memberConfig [-memberNode ' + nn01 + ' -memberName Demo' + sName + 'CM' + str(i) + ' -memberWeight 2 -genUniquePorts true -replicatorEntry false]]')
  i+=1
  continue
 
 while i <=4:
  AdminTask.createClusterMember('[-clusterName ' + clustName + ' -memberConfig [-memberNode ' + nn02 + ' -memberName Demo' + sName + 'CM' + str(i) + ' -memberWeight 2 -genUniquePorts true -replicatorEntry false]]')
  i+=1
  continue

createClusterMembers(Node1Name, Node2Name, clusterName)
# Gets a list of all the newly created cluster members
cmIDs = AdminConfig.list('ClusterMember', clusterID).split(nl)

#Lists all of the application servers
serverName = []
serverID = AdminTask.listServers('-serverType APPLICATION_SERVER').split(nl)
for sid in serverID:
 serverName.append(AdminConfig.showAttribute(sid, 'name'))

serverName.sort()


# This function saves any configuration changes to the master respository
def saveConfig():
 print ""
 print "Saving changes to the master repository"
 AdminConfig.save()
 print ""
 print "DONE!"

# The following function Syncs all of the nodes with an active/started nodeagent task
def syncNodes():
 nodelist = AdminConfig.list('Node').split(nl)
 print "\n\n########################################"
 print "### Synchronizing the Nodes"
 print "########################################\n"
 for n in nodelist:
  nodename = AdminConfig.showAttribute(n, 'name')
  objname = "type=NodeSync,node=" + nodename + ",*"
  Syncl = AdminControl.completeObjectName(objname)
  if Syncl != "":
   if AdminControl.invoke(Syncl, 'isNodeSynchronized') == "true":
    print "\n!! " + nodename + " is already synchronized"
   else:
    AdminControl.invoke(Syncl, 'sync')
    print "\n)) " + nodename + " synchronized"
  continue
  
print "Save the configuration changes?\n\n"

saveConfig()

syncNodes()
```

Run the script:
```bash
wsadmin>execfile('/home/krish/jython/cluster.py')
```
the output: 
```sh
AppSrvNode AppSrvNode02
Creating the Demo cluster
Creating the cluster members for the Demo cluster
Save the configuration changes?

Saving changes to the master repository

DONE!

########################################
### Synchronizing the nodes
########################################

)) AppSrvNode synchronized

)) AppSrvNode02 synchronized
```

## Another Jython Script to Create Cluster, Server and Configuring them Fully

> This Script does the following Tasks:
  1. Create Cluster and Server and make server Member of cluster.
  2. If cluster exits then it creates Server and make it member of cluster.
  3. If cluster and server exists and if server is not member of cluster, it makes that server member of cluster.
  4. Debug Argument.
  5. Http Transports.
  6. Ports --> SOAP, DRS, and Bootstrap ports.
  7. JVM heap Size.
  8. Virtual host.
  9. AppServer Classpath
  10. AppServer BootClasspath.
  11. Generic JVM argument.
  12. Cluster Level and Server Level Variables.
  13. WebContainer Thread Pool Setting.


Written by Charanjeet Singh

```py
import sys,java
from java.util import Properties
from org.python.modules import time
from java.io import FileInputStream

lineSep = java.lang.System.getProperty('line.separator')


def clusterServer(node,cluster,server,http,http1,host,bootstrap,soap,orb,csiv2_multi,csiv2_server,adminhost,adminhost1,dcs,sib,sib1,sib_mq,sib_mq1,sip,sip1,sas,maxHS,minHS,varname1,varvalue1,varname2,varvalue2,varname3,varvalue3,JvmArguments,maxtp,mintp,debug):

global AdminApp
global AdminConfig
global AdminControl

print " ----------------------------------------------------------------------------------------- "

######################################## Getting config ID of cell ################################################

cell = AdminControl.getCell()

cellid = AdminConfig.getid('/Cell:'+ cell)

##################################### checking for existence of node #########################################################


nodeid= AdminConfig.getid('/Cell:'+ cell +'/Node:'+ node)

print " Checking for existence of node --> " + node

if len(nodeid) == 0:

print "Error -- node not found for name --> " + node

else:

print " ----------------------------------------------------------------------------------------- "

print " Node exists --> " + node


print " ----------------------------------------------------------------------------------------- "


###################### Checking for the existence of cluster and creating cluster if it doesnot exist ###########################


Clusterid = AdminConfig.getid('/ServerCluster:'+ cluster +'/')

print " ----------------------------------------------------------------------------------------- "

print " Checking for existence of Cluster --> " + cluster

if len(Clusterid) == 0:

print " ----------------------------------------------------------------------------------------- "

print "Cluster doesnot exists "

print " ----------------------------------------------------------------------------------------- "

print " Creating cluster --> " + cluster

name_attr = ["name", cluster]
desc_attr = ["description", cluster+" cluster"]
pref_attr = ["preferLocal", "true"]
statem_attr = ["stateManagement", [["initialState", "STOP"]]]
attrs = [name_attr, desc_attr, pref_attr, statem_attr]


AdminConfig.create('ServerCluster', cellid, attrs)

AdminConfig.save()

print " ----------------------------------------------------------------------------------------- "

print " Verify that cluster created successfully "

print " ----------------------------------------------------------------------------------------- "

newClusterid = AdminConfig.getid('/ServerCluster:'+ cluster +'/')

if len(newClusterid) > 0:

print " ----------------------------------------------------------------------------------------- "

print " cluster --> " + cluster + " created successfully "

print " ----------------------------------------------------------------------------------------- "

###################### Checking for the existence of Server and creating Server if it doesnot exist ###########################

Serverid = AdminConfig.getid('/Cell:'+ cell +'/Node:'+ node +'/Server:'+ server)

print " Checking for existence of Server :" + server

if len(Serverid) == 0:

print " ----------------------------------------------------------------------------------------- "

print "Server doesnot exists "

print " ----------------------------------------------------------------------------------------- "

print " Creating --> " + server + " and making this server as a member of cluster --> " + cluster

AdminConfig.createClusterMember(newClusterid, nodeid, [['memberName', server ]])

AdminConfig.save()

Serverid = AdminConfig.getid('/Cell:'+ cell +'/Node:'+ node +'/Server:'+ server)

if len(Serverid) > 0:

print " ----------------------------------------------------------------------------------------- "

print " Server --> " + server +" created and added as a member of --> " + cluster


else:

print " ----------------------------------------------------------------------------------------- "

print "Server already exist with this name --> " + server + " , please provide different name in property file "


else:

print " ----------------------------------------------------------------------------------------- "

print " Cluster exist --> "+ cluster + " and making this server as a member of cluster --> " + cluster +""

Serverid = AdminConfig.getid('/Cell:'+ cell +'/Node:'+ node +'/Server:'+ server)

if len(Serverid) == 0:

print " ----------------------------------------------------------------------------------------- "

print "Server doesnot exists "

print " ----------------------------------------------------------------------------------------- "

print " Creating --> " + server + " and making this server as a member of cluster --> " + cluster

AdminConfig.createClusterMember(Clusterid, nodeid, [['memberName', server ]])

AdminConfig.save()

Serverid = AdminConfig.getid('/Cell:'+ cell +'/Node:'+ node +'/Server:'+ server)

if len(Serverid) > 0:

print " ----------------------------------------------------------------------------------------- "

print " Server --> " + server +" created and added as a member of --> " + cluster


else:

print " ----------------------------------------------------------------------------------------- "

print "Server already exist with this name --> " + server + " , please provide different name in property file "

print " ----------------------------------------------------------------------------------------- "


################################# configuring properties for Server , variables and Virtual host ###################################

Clusterid = AdminConfig.getid('/ServerCluster:'+ cluster +'/')

if len(Clusterid) > 0:

Serverid = AdminConfig.getid('/Cell:'+ cell +'/Node:'+ node +'/Server:'+ server)

if len(Serverid) > 0:

print " Following properties will be configured for --> " + server

print " 1. Debug Argument "

print " 2. Http Transports "

print " 3. Ports --> SOAP , DRS , BootStrap ports "

print " 4. JVM heap Size "

print " 5. Virtual host "

print " 6. Classpath "

print " 7. Boot Classpath "

print " 8. Generic JVM argument "

print " 9. Create Server Level Variables "

print " 10. WebContainer Thread Pool Setting"

print " ----------------------------------------------------------------------------------------- "

########################################## configuring JVM debug mode #####################################################

jvm = AdminConfig.list('JavaVirtualMachine', Serverid)

AdminConfig.modify(jvm, [['debugMode', 'false'], ['debugArgs', debug ]])

AdminConfig.save()

print " JVM Debug argument Configured ............."

print " ----------------------------------------------------------------------------------------- "

######################################### Configuring HTTP Transports ######################################################

portsDict = {}
portsDict["nodeName"] = node
portsDict["endPointName"] = "WC_defaulthost"
portsDict["host"] = host
portsDict["port"] = http
AdminTask.modifyServerPort(server,
["-%s %s" % (key, value) for key, value in portsDict.items()])

portsDict = {}
portsDict["nodeName"] = node
portsDict["endPointName"] = "WC_defaulthost_secure"
portsDict["host"] = host
portsDict["port"] = http1
AdminTask.modifyServerPort(server,
["-%s %s" % (key, value) for key, value in portsDict.items()])


print " HTTP Transports Configured .............. "

print " ----------------------------------------------------------------------------------------- "


################################################# Configuring Ports ###################################################


#### WAS 6.1 ####

## Bootstrap port

portsDict = {}
portsDict["nodeName"] = node
portsDict["endPointName"] = "BOOTSTRAP_ADDRESS"
portsDict["host"] = host
portsDict["port"] = bootstrap
AdminTask.modifyServerPort(server,
["-%s %s" % (key, value) for key, value in portsDict.items()])

print " BOOTSTRAP_ADDRESS Configured ........... "

print " ----------------------------------------------------------------------------------------- "

## SOAP Port

portsDict = {}
portsDict["nodeName"] = node
portsDict["endPointName"] = "SOAP_CONNECTOR_ADDRESS"
portsDict["host"] = host
portsDict["port"] = soap
AdminTask.modifyServerPort(server,
["-%s %s" % (key, value) for key, value in portsDict.items()])

print " Soap Port Configured ...................."

print " ----------------------------------------------------------------------------------------- "


## CSIV2_SSL_MUTUALAUTH_LISTENER_ADDRESS Port

portsDict = {}
portsDict["nodeName"] = node
portsDict["endPointName"] = "CSIV2_SSL_MUTUALAUTH_LISTENER_ADDRESS"
portsDict["host"] = host
portsDict["port"] = csiv2_multi
AdminTask.modifyServerPort(server,
["-%s %s" % (key, value) for key, value in portsDict.items()])

print " CSIV2_SSL_MUTUALAUTH_LISTENER_ADDRESS Port Configured ......... "

print " ------------------------------------------------------------------------------------------------------------------------ "

## CSIV2_SSL_SERVERAUTH_LISTENER_ADDRESS Port

portsDict = {}
portsDict["nodeName"] = node
portsDict["endPointName"] = "CSIV2_SSL_SERVERAUTH_LISTENER_ADDRESS"
portsDict["host"] = host
portsDict["port"] = csiv2_server
AdminTask.modifyServerPort(server,
["-%s %s" % (key, value) for key, value in portsDict.items()])

print " CSIV2_SSL_SERVERAUTH_LISTENER_ADDRESS Port Configured .......... "

print " ------------------------------------------------------------------------------------------------------------------------ "

## ORB_LISTENER_ADDRESS Port

portsDict = {}
portsDict["nodeName"] = node
portsDict["endPointName"] = "ORB_LISTENER_ADDRESS"
portsDict["host"] = host
portsDict["port"] = orb
AdminTask.modifyServerPort(server,
["-%s %s" % (key, value) for key, value in portsDict.items()])

print " ORB_LISTENER_ADDRESS Port Configured .............."

print " ------------------------------------------------------------------------------------------------------------------------ "

## SAS_SSL_SERVERAUTH_LISTENER_ADDRESS Port

portsDict = {}
portsDict["nodeName"] = node
portsDict["endPointName"] = "SAS_SSL_SERVERAUTH_LISTENER_ADDRESS"
portsDict["host"] = host
portsDict["port"] = sas
AdminTask.modifyServerPort(server,
["-%s %s" % (key, value) for key, value in portsDict.items()])

print " SAS_SSL_SERVERAUTH_LISTENER_ADDRESS Port configured ........ "


print " ------------------------------------------------------------------------------------------------------------------------ "

## WC_adminhost Port

portsDict = {}
portsDict["nodeName"] = node
portsDict["endPointName"] = "WC_adminhost"
portsDict["host"] = host
portsDict["port"] = adminhost
AdminTask.modifyServerPort(server,
["-%s %s" % (key, value) for key, value in portsDict.items()])


print " WC_adminhost Port Configured ...... "

print " ------------------------------------------------------------------------------------------------------------------------ "

## WC_adminhost_secure Port

portsDict = {}
portsDict["nodeName"] = node
portsDict["endPointName"] = "WC_adminhost_secure"
portsDict["host"] = host
portsDict["port"] = adminhost1
AdminTask.modifyServerPort(server,
["-%s %s" % (key, value) for key, value in portsDict.items()])


print " WC_adminhost_secure Port Configured ...... "

print " ------------------------------------------------------------------------------------------------------------------------ "

## DCS_UNICAST_ADDRESS Port

portsDict = {}
portsDict["nodeName"] = node
portsDict["endPointName"] = "DCS_UNICAST_ADDRESS"
portsDict["host"] = host
portsDict["port"] = dcs
portsDict["modifyShared"] = "true"
AdminTask.modifyServerPort(server,
["-%s %s" % (key, value) for key, value in portsDict.items()])


print " DCS_UNICAST_ADDRESS Configured ...... "

print " ------------------------------------------------------------------------------------------------------------------------ "

## SIB_ENDPOINT_ADDRESS Port

portsDict = {}
portsDict["nodeName"] = node
portsDict["endPointName"] = "SIB_ENDPOINT_ADDRESS"
portsDict["host"] = host
portsDict["port"] = sib
AdminTask.modifyServerPort(server,
["-%s %s" % (key, value) for key, value in portsDict.items()])


print " SIB_ENDPOINT_ADDRESS Port Configured ......... "

print " ------------------------------------------------------------------------------------------------------------------------ "

## SIB_ENDPOINT_SECURE_ADDRESS Port

portsDict = {}
portsDict["nodeName"] = node
portsDict["endPointName"] = "SIB_ENDPOINT_SECURE_ADDRESS"
portsDict["host"] = host
portsDict["port"] = sib1
AdminTask.modifyServerPort(server,
["-%s %s" % (key, value) for key, value in portsDict.items()])


print " SIB_ENDPOINT_SECURE_ADDRESS Port Configured ......... "

print " ------------------------------------------------------------------------------------------------------------------------ "

## SIB_MQ_ENDPOINT_ADDRESS Port

portsDict = {}
portsDict["nodeName"] = node
portsDict["endPointName"] = "SIB_MQ_ENDPOINT_ADDRESS"
portsDict["host"] = host
portsDict["port"] = sib_mq
AdminTask.modifyServerPort(server,
["-%s %s" % (key, value) for key, value in portsDict.items()])


print " SIB_MQ_ENDPOINT_ADDRESS Port Configured ......... "

print " ------------------------------------------------------------------------------------------------------------------------ "

## SIB_MQ_ENDPOINT_SECURE_ADDRESS Port

portsDict = {}
portsDict["nodeName"] = node
portsDict["endPointName"] = "SIB_MQ_ENDPOINT_SECURE_ADDRESS"
portsDict["host"] = host
portsDict["port"] = sib_mq1
AdminTask.modifyServerPort(server,
["-%s %s" % (key, value) for key, value in portsDict.items()])


print " SIB_MQ_ENDPOINT_SECURE_ADDRESS Port Configured ......... "

print " ------------------------------------------------------------------------------------------------------------------------ "

## SIP_DEFAULTHOST Port

portsDict = {}
portsDict["nodeName"] = node
portsDict["endPointName"] = "SIP_DEFAULTHOST"
portsDict["host"] = host
portsDict["port"] = sip
portsDict["modifyShared"] = "true"
AdminTask.modifyServerPort(server,
["-%s %s" % (key, value) for key, value in portsDict.items()])


print " SIP_DEFAULTHOST Port Configured ......... "

print " ------------------------------------------------------------------------------------------------------------------------ "

## SIP_DEFAULTHOST_SECURE Port

portsDict = {}
portsDict["nodeName"] = node
portsDict["endPointName"] = "SIP_DEFAULTHOST_SECURE"
portsDict["host"] = host
portsDict["port"] = sip1
portsDict["modifyShared"] = "true"
AdminTask.modifyServerPort(server,
["-%s %s" % (key, value) for key, value in portsDict.items()])


print " SIP_DEFAULTHOST_SECURE Port Configured ......... "

print " ------------------------------------------------------------------------------------------------------------------------ "

print " All Port's Configured ............... "

print " ------------------------------------------------------------------------------------------------------------------------ "


#################################################### configuring JVM heap Size #############################################

AdminConfig.modify(jvm, [['initialHeapSize', minHS ], ['maximumHeapSize', maxHS ]])

AdminConfig.save()

print " JVM heap Size configured ........... "

print " ------------------------------------------------------------------------------------------------------------------------ "

################################################### Configuring WebContainer thread pool Size ################################


tpList=AdminConfig.list('ThreadPool', Serverid).split(lineSep)

for tp in tpList:

tpWebContainer=tp

AdminConfig.show(tpWebContainer)

AdminConfig.modify(tpWebContainer, [['maximumSize', maxtp ]] )

AdminConfig.save()

AdminConfig.modify(tpWebContainer, [['minimumSize', mintp ]] )

AdminConfig.save()


print " WebContainer thread pool Size configured ........... "

print " ----------------------------------------------------------------------------------------------------------------------"

################################################# Creating Virtual Hosts ############################################################

hostname = ["hostname", "*"]

port1 = ["port", http]

port2 = ["port" ,http1 ]

hostAlias1 = [hostname, port1]

hostAlias2 = [ hostname , port2 ]

vh = AdminConfig.getid("/VirtualHost:"+cluster+"/")

if len(vh) == 0 :

vtempl= "default_host(templates/default|virtualhosts.xml#VirtualHost_1)"

parentVhost = AdminConfig.createUsingTemplate('VirtualHost', cellid, [['name', cluster]], vtempl)

AdminConfig.create('HostAlias', parentVhost, hostAlias1 )

AdminConfig.create('HostAlias', parentVhost, hostAlias2 )

AdminConfig.save()

else :

AdminConfig.create('HostAlias', vh, hostAlias1 )

AdminConfig.create('HostAlias', vh, hostAlias2 )

AdminConfig.save()

print " Virtual Host Created .............. "


print " ------------------------------------------------------------------------------------------------------------------------ "


################################################# Creating Variables ###############################################################

############################################ SERVER LEVEL ################################################


varMapserver = AdminConfig.getid('/Cell:'+ cell +'/Node:'+ node +'/Server:'+ server +'/VariableMap:/')

if len(varMapserver) > 0:

AdminConfig.remove(varMapserver)

AdminConfig.save()

varMapserver1 = AdminConfig.create('VariableMap', Serverid, [])

varMapserver2 = AdminConfig.getid('/Cell:'+ cell +'/Node:'+ node +'/Server:'+ server +'/VariableMap:/')

nameattr1 = ['symbolicName', varname1]
valattr1 = ['value', varvalue1]
desc1 = ["description", varname1]
nameattr2 = ['symbolicName', varname2]
valattr2 = ['value', varvalue2]
desc2 = ["description", varname2]
nameattr3 = ['symbolicName', varname3]
valattr3 = ['value', varvalue3 +'/'+ server ]
desc3 = ["description", varname3]
attr1 = [nameattr1, valattr1, desc1]
attr2 = [nameattr2, valattr2, desc2]
attr3 = [nameattr3, valattr3, desc3]
attrs1 = [attr1, attr2, attr3]

entries6 = ['entries', attrs1 ]

AdminConfig.modify(varMapserver2, [entries6] )

AdminConfig.save()

print " Server Level Variables Configured ................ "

print " ------------------------------------------------------------------------------------------------------------------------ "


################################################# Configuring Classpath for AppServer ##############################################


classpath1 = ['classpath', '${'+ varname2 +'}/properties.jar']
classpath2 = ['classpath', '${'+ varname1 +'}/ojdbc14.jar']
classpath3 = ['classpath', '${'+ varname1 +'}/log4j-1.2.4.jar']
classpath4 = ['classpath', '${'+ varname1 +'}/poi-2.5.1-final-20040804.jar']
classpath5 = ['classpath', '${'+ varname1 +'}/jdom.jar']
classpath6 = ['classpath', '${'+ varname1 +'}/idssecl.jar']
classpath7 = ['classpath', '${'+ varname1 +'}/com.ibm.mq.jar']
classpath8 = ['classpath', '${'+ varname1 +'}/com.ibm.mqbind.jar']
classpath9 = ['classpath', '${'+ varname1 +'}/com.ibm.mqjms.jar']
classpath10 = ['classpath', '${'+ varname1 +'}/connector.jar']
classpath11 = ['classpath', '${'+ varname1 +'}/jms.jar']
classpath12 = ['classpath', '${'+ varname1 +'}/jta.jar']
classpath13 = ['classpath', '${'+ varname1 +'}/mqji.jar']
classpath14 = ['classpath', '${'+ varname1 +'}/idsclie.jar']
classpath15 = ['classpath', '${'+ varname2 +'}/'+ server]

AdminConfig.modify(jvm, [classpath1])
AdminConfig.modify(jvm, [classpath2])
AdminConfig.modify(jvm, [classpath3])
AdminConfig.modify(jvm, [classpath4])
AdminConfig.modify(jvm, [classpath5])
AdminConfig.modify(jvm, [classpath6])
AdminConfig.modify(jvm, [classpath7])
AdminConfig.modify(jvm, [classpath8])
AdminConfig.modify(jvm, [classpath9])
AdminConfig.modify(jvm, [classpath10])
AdminConfig.modify(jvm, [classpath11])
AdminConfig.modify(jvm, [classpath12])
AdminConfig.modify(jvm, [classpath13])
AdminConfig.modify(jvm, [classpath14])
AdminConfig.modify(jvm, [classpath15])

AdminConfig.save()

print " Classpath for AppServer Configured ......... "

print " ------------------------------------------------------------------------------------------------------------------------ "

############################################### Configuring Boot Classpath for AppServer ############################################

bootclasspath = ['bootClasspath', '${'+ varname1 +'}/cal141.jar']

AdminConfig.modify(jvm, [bootclasspath])

AdminConfig.save()

print " Boot Classpath for AppServer Configured ............... "

print " ------------------------------------------------------------------------------------------------------------------------ "


############################################# Configuring Generic JVM argument for AppServer ########################################

genericJVMargument = ['genericJvmArguments', JvmArguments]

AdminConfig.modify(jvm, [genericJVMargument])

AdminConfig.save()

print " Generic JVM argument for AppServer Configured ............. "

print " ------------------------------------------------------------------------------------------------------------------------ "


######################################################### Full Resyncronization #########################################################

print " Fully Resyncronizing nodes ........... "

nodelist = AdminTask.listManagedNodes().split(lineSep)

for nodename in nodelist :

####################Identifying the ConfigRepository MBean and assign it to variable######################

repo = AdminControl.completeObjectName('type=ConfigRepository,process=nodeagent,node='+ nodename +',*')

print AdminControl.invoke(repo, 'refreshRepositoryEpoch')

sync = AdminControl.completeObjectName('cell='+ cell +',node='+ nodename +',type=NodeSync,*')

print AdminControl.invoke(sync , 'sync')

print " ----------------------------------------------------------------------------------------- "

print " Full Resyncronization completed "

print " ----------------------------------------------------------------------------------------- "


########################################################## Main Program #############################################################

arglen=len(sys.argv)

num_exp_args=11

if (arglen != num_exp_args):

print "eleven arguments are required. This argument should be a properties file and Server level Variables."

print " ----------------------------------------------------------------------------------------- "

sys.exit(-1)

propFile=sys.argv[0]

properties=Properties();

try:

properties.load(FileInputStream(propFile))

print " ----------------------------------------------------------------------------------------- "

print "Succesfully read property file "+propFile

print " ----------------------------------------------------------------------------------------- "

except:

print "Cannot read property file "+propFile
sys.exit(-1)

print " ----------------------------------------------------------------------------------------- "

node = sys.argv[7]

cluster = sys.argv[8]

server = sys.argv[9]

http = int(properties.getProperty("WC_defaulthost"))

http1 = int(properties.getProperty("WC_defaulthost_secure"))

host = sys.argv[10]

bootstrap = int(properties.getProperty("BOOTSTRAP_ADDRESS"))

soap = int(properties.getProperty("SOAP_CONNECTOR_ADDRESS"))

orb = int(properties.getProperty("ORB_LISTENER_ADDRESS"))

csiv2_multi = int(properties.getProperty("CSIV2_SSL_MUTUALAUTH_LISTENER_ADDRESS"))

csiv2_server = int(properties.getProperty("CSIV2_SSL_SERVERAUTH_LISTENER_ADDRESS"))

adminhost = int(properties.getProperty("WC_adminhost"))

adminhost1 = int(properties.getProperty("WC_adminhost_secure"))

dcs = int(properties.getProperty("DCS_UNICAST_ADDRESS"))

sib = int(properties.getProperty("SIB_ENDPOINT_ADDRESS"))

sib1 = int(properties.getProperty("SIB_ENDPOINT_SECURE_ADDRESS"))

sib_mq = int(properties.getProperty("SIB_MQ_ENDPOINT_ADDRESS"))

sib_mq1 = int(properties.getProperty("SIB_MQ_ENDPOINT_SECURE_ADDRESS"))

sip = int(properties.getProperty("SIP_DEFAULTHOST"))

sip1 = int(properties.getProperty("SIP_DEFAULTHOST_SECURE"))

sas = int(properties.getProperty("SAS_SSL_SERVERAUTH_LISTENER_ADDRESS"))

maxHS = int(properties.getProperty("MAXIMUM_HEAP_SIZE"))

minHS = int(properties.getProperty("MINIMUM_HEAP_SIZE"))

varname1 = sys.argv[1]

varvalue1 = sys.argv[2]

varname2 = sys.argv[3]

varvalue2 = sys.argv[4]

varname3 = sys.argv[5]

varvalue3 = sys.argv[6]

JvmArguments = str(properties.getProperty("GenericJvmArguments"))

maxtp = str(properties.getProperty("MAXIMUM_THREAD_POOL"))

mintp = str(properties.getProperty("MINIMUM_THREAD_POOL"))

debug = str(properties.getProperty("DebugArguments"))

clusterServer(node,cluster,server,http,http1,host,bootstrap,soap,orb,csiv2_multi,csiv2_server,adminhost,adminhost1,dcs,sib,sib1,sib_mq,sib_mq1,sip,sip1,sas,maxHS,minHS,varname1,varvalue1,varname2,varvalue2,varname3,varvalue3,JvmArguments,maxtp,mintp,debug)

############################################################################################################################
```