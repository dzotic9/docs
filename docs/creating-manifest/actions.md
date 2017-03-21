# Actions

Actions represent a scripted logic for executing a set of commands to automate the tasks. The system provides a default list of actions and possibility to <a href="/creating-manifest/custom-scripts/" target="_blank">script custom actions</a> using <a href="https://docs.jelastic.com/api/" target="_blank">API calls</a>, Linux bash shell command, JS, and Java scripts. Any action, available to be performed by means of API (including custom scripts running), should be bound to some <a href="/creating-manifest/events" target="_blank">event</a> and executed as a result of this event occurrence.                                                      

With the help of actions you can achieve automation of the tasks related to:                

- increasing or decreasing CPU or RAM amount                    

- adjusting essential configs                                

- restarting a service or a container                             

- applying a database patch                                                   

The default workflow for any action execution is the following:                  

- replacing <a href="/creating-manifest/placeholders" target="_blank">placeholders</a>                                     

- getting a list of <a href="/creating-manifest/selecting-containers" target="_blank">target containers</a>                                                 

- checking permissions                                     

- executing the action itself                                                   

Thus, the following specific groups of actions are singled out:           

- [Container Operations](#container-operations)                   

- [Topology Nodes Management](#topology-nodes-management)             

- [Database Operations](#database-operations)                   

- [User-Defined Operations](#user-defined-operations)              

- [Custom Actions](#custom-actions)                                             

## Container Operations

There are actions that perform operations inside of a container. For a detailed guidance on how to set a target container, visit the <a href="/creating-manifest/selecting-containers" target="_blank"><em>Specifying Target Containers</em></a> page.                        

Any container operation can be performed using a [*cmd*](#cmd) action. Moreover, there are also some additional actions provided for your convenience. Thus, all the actions performed in confines of a container can be divided into three groups:       

- SSH commands ([*cmd*](#cmd))                            

- predefined modules ([*deploy*](#deploy), [*upload*](#upload), [*unpack*](#unpack))                                

- operations with files ([*createFile*](#createfile), [*createDirectory*](#createdirectory), [*writeFile*](#writefile), [*appendFile*](#appendfile), [*replaceInFile*](#replaceinfile))                                     

!!! note 
    To process any container operation (except for [*cmd*](#cmd)), the Cloud Scripting engine applies a default system user with restricted permissions.                       

### cmd

The *cmd* action executes <a href="https://docs.jelastic.com/ssh-overview" target="_blank">SSH</a> commands.             
<!--Available for all nodes.-->      

**Example**                  
``` json
{
  "cmd [nodeId,nodeType,nodeGroup]": [
    "cmd1",
    "cmd2"
  ],
  "sayYes" : true
}
```
where:       
     
- `nodeId`, `nodeGroup`, `nodeType` - parameters that determine target containers for the action execution (at least one of these parameters is required)                                                
- `cmd1` and `cmd2` - set of commands that are executed. Their values are wrapped by the Cloud Scripting engine via **echo cmd | base64 -d | su user** where:                    
    - **cmd** - is equal to the Base64 encoded string: **yes | (cmd1;cmd2)**. If your commands require the interactive input, by default, the Cloud Scripting engine always gives a positive answer, using **yes** utility.        
    - **user** - default system user with restricted permissions    
- `sayYes` *[optional]* - parameter that enables or disables the usage of **yes** utility. The default value is *'true'*.                  

The single SSH command can be passed in a string. For example, running a bash script from URL on all **Tomcat 6** nodes.                    
``` json 
{
  "cmd [tomcat6]": "curl -fsSL http://example.com/script.sh | /bin/bash -s arg1 arg2"
}
```

While accessing a container via *cmd*, you receive all the required permissions and additionally can manage the main services with **sudo** commands of the following types (and others).            
```no-highlight
sudo /etc/init.d/jetty start  
sudo /etc/init.d/mysql stop
sudo /etc/init.d/tomcat restart  
sudo /etc/init.d/memcached status  
sudo /etc/init.d/mongod reload  
sudo /etc/init.d/nginx upgrade  
sudo /etc/init.d/httpd help                             
```                                                        
**Examples**  

Setting SSH commands in an array.                    
``` json
{
  "cmd [tomcat6]": [
    "curl -fsSL http://example.com/script.sh | /bin/bash -s arg1 arg2"
  ]
}
```
                             
Downloading and unzipping the **WordPress** plugin on all the compute nodes. Here, the commands array is executed through a single SSH command. The same can be performed with the help of the [unpack](#unpack) method.                              
``` json
{
  "cmd [cp]": [
    "cd /var/www/webroot/ROOT/wp-content/plugins/",
    "curl -fsSL \"http://example.com/plugin.zip\" -o plugin.zip",
    "unzip plugin.zip"
  ]
}
```

Using **sudo** to reload Nginx balancer.       
``` json
{
  "cmd [nginx]": [
    "sudo /etc/init.d/nginx reload"
  ]
}
```
   
### api

Executing actions available by means of <a href="http://docs.jelastic.com/api" target="_blank">Jelastic Cloud API</a>.     

There are a number of parameters required by Jelastic API that are defined automatically:                            

- *envName* - environment domain name where the API method is executed             

- *appid* - unique environment identifier that can be passed to API instead of *envName*                         

- *session* - unique session of a current user                                  

Target containers, specified for the API methods execution can be passed by the nodes keywords. Therefore, API methods can be run on all nodes within a single <a href="/creating-manifest/selecting-containers/#all-containers-by-group" target="blank"><em>nodeGroup</em></a> (i.e. layer) or <a href="/creating-manifest/selecting-containers/#all-containers-by-type" target="_blank"><em>nodeType</em></a>. Also, API methods can be run on a <a href="/creating-manifest/selecting-containers/#particular-container" target="_blank">particular node</a>. In this case, the Node ID is required that is available either through the <a href="/creating-manifest/placeholders/#node-placeholders" target="_blank">node placeholders</a>, or a set of [custom action parameters](#action-placeholders) (*${this}*).                     

**Examples**

Restarting all compute nodes in the environment.                      
``` json
{
    "api [cp]" : "jelastic.environment.control.RestartNodesByGroup"
}
``` 
where:        
       
- `api [cp]` - target node group for the API method execution (*[cp]*)                                                         
- *jelastic.environment.control.RestartNodesByGroup* - Jelastic API method for restarting nodes by group              

This method (*jelastic.environment.control.RestartNodesByGroup*) can be simplified like shown in the next example.    
``` json
{
    "api [cp]" : "environment.control.RestartNodesByGroup"
}
```

Below, you can find one more approach to specify a target node group for the API method execution.                                  
``` json
{
    "api" : "jelastic.environment.control.RestartNodesByGroup",
    "nodeGroup" : "cp"
}
```

### deploy

Available for compute nodes (except for Docker containers)
``` json
{
  "deploy": [
    {
      "archive": "URL",
      "name": "string",
      "context": "string"
    }
  ]
}
```
where:

- `archive` - URL to the archive with a compressed application
- `name` - application's name that will be displayed at the dashboard
- `context`- desired context for the deployed app

### upload

Available for all nodes
<!--Available for all nodes (except for *Docker* containers and *Elastic VDS*)-->
``` json
{
  "upload": [
    {
      "nodeId": "number or string",
      "nodeGroup": "string",
      "nodeType": "string",
      
      "sourcePath" : "URL",
      "destPath" : "string"
    }
  ]
}
```
where:  

- `nodeId`, `nodeGroup`, `nodeType` - parameters that determine target containers for the action execution (at least one of these parameters is required)                                                                                                                                            
- `sourcePath` - URL to download an external file                    
- `destPath` - container path where the uploaded file is to be saved                         

### unpack

Available for all nodes
<!--Available for all nodes (except for *Docker* containers and *Elastic VDS*)--> 
``` json
{
  "unpack": [
    {
      "nodeId": "number or string",
      "nodeGroup": "string",
      "nodeType": "string",
      
      "sourcePath" : "URL",
      "destPath" : "string"
    }
  ]
}
```
where:   

- `nodeId`, `nodeGroup`, `nodeType` - parameters that determine target containers for the action execution (at least one of these parameters is required)                                             
- `sourcePath` - URL to download an external archive   
- `destPath` - container path where the uploaded archive is to be unpacked                               

### createFile

Available for all nodes
<!--Available for all nodes (except for *Docker* containers and *Elastic VDS*)--> 
``` json
{
  "createFile [nodeId, nodeGroup, nodeType]": "string"
}
```
where:   

- `nodeId`, `nodeGroup`, `nodeType` - parameters that determine target containers for the action execution (at least one of these parameters is required)                                                   
- `string` - container path where a file is to be created                              

### createDirectory

Available for all nodes
<!--Available for all nodes (except for *Docker* containers and *Elastic VDS*)--> 
``` json
{
  "createDirectory [nodeId, nodeGroup, nodeType]": "string"
}
```
where:  

- `nodeId`, `nodGroup`, `nodeType` - parameters that determine target containers for the action execution (at least one of these parameters is required)                                                                 
- `string` - container path where a directory is to be created                         

### writeFile

Available for all nodes
<!--Available for all nodes (except for *Docker* containers and *Elastic VDS*)--> 
``` json
{
  "writeFile": [
    {
      "nodeId": "number or string",
      "nodeGroup": "string",
      "nodeType": "string",
            
      "path" : "string",
      "body" : "string"
    }
  ]
}
```
where:  
  
- `nodeId`, `nodeGroup`, `nodeType` - parameters that determine target containers for the action execution (at least one of these parameters is required)                                                      
- `path` - container path where a file is to be written                
- `body` - content that is saved to a file                                         

### appendFile

Available for all nodes
<!--Available for all nodes (except for *Docker* containers and *Elastic VDS*)--> 
``` json
{
  "appendFile": [
    {
      "nodeId": "number or string",
      "nodeGroup": "string",
      "nodeType": "string",
            
      "path" : "string",
      "body" : "string"
    }
  ]
}
```
where:      

- `nodeId`, `nodeGroup`, `nodeType` - parameters that determine target containers for the action execution (at least one of these parameters is required)                               
- `path` - container path where a file is to be appended                                 
- `body` - content saved to a file                               

### replaceInFile

Available for all nodes
<!--Available for all nodes (except for *Docker* containers and *Elastic VDS*)--> 
``` json
{
  "replaceInFile": [
    {
      "nodeId": "number or string",
      "nodeGroup": "string",
      "nodeType": "string",
            
      "path" : "string",
      "replacements" : [{
        "pattern" : "string",
        "replacement" : "string"
      }]
    }
  ]
}
```
where:   

- `nodeId`, `nodeGroup`, `nodeType` - parameters that determine target containers for the action execution (at least one of these parameters is required)                                  
- `path` - path where a file is available               
- `replacements` - list of replacements within the node's configuration files                        
    - `pattern` - regular expressions to find a string (e.g. *app\\.host\\.url\\s*=\\s*.**)                   
    - `replacement` - you can use as a replacement any string value, including any combination of <a href="/creating-manifest/placeholders" target="_blank">placeholders</a>                                            

<!-- DeletePath -->
<!-- RenamePath --> 

## Topology Nodes Management

The present section introduces actions that are provided for managing the topology.                 

### addNodes
``` json
{
  "addNodes": [
    {
      "nodeType": "string",
      "extip": "boolean",      
      "fixedCloudlets": "number",
      "flexibleCloudlets": "number",
      "displayName": "string",
      "dockerName": "jelastic/wordpress-web:latest",
      "registryUrl": "string",
      "registryUser": "string",
      "registryPassword": "string",
      "dockerTag": "string",
      "dockerLinks": "sourceNodeGroup:alias",
      "dockerEnvVars": "object",
      "dockerVolumes": "array",
      "volumeMounts": "object",
      "dockerRunCmd": "array",
      "dockerEntryPoint": "object"
    }
  ]
}
```
where:

- `nodeType` *[required]* - parameter to specify <a href="/creating-manifest/selecting-containers/#predefined-nodetype-values" target="_blank">software stacks</a>. For Docker containers the *nodeType* value is **docker**.                                        
- `extip` *[optional]* - attaching the external IP address to a container. The default value is *'false'*.                     
- `fixedCloudlets` *[optional]* - number of reserved cloudlets. The default value is *'0'*.                             
- `flexibleCloudlets` *[optional]* - number of dynamic cloudlets. The default value is *'1'*.                           
- `displayName` *[optional]* - node's display name (i.e. <a href="https://docs.jelastic.com/environment-aliases" target="_blank">alias</a>)                                         
    The following parameters are required for <a href="https://docs.jelastic.com/dockers-overview" target="_blank">Docker</a> containers only:                                    
- `dockerName` *[optional]* - name and tag of Docker image
- `registryUrl` *[optional]* - custom Docker regitry
- `registryUser` *[optional]* - Docker registry username
- `registryPassword` *[optional]* - Docker registry password
- `dockerTag` - Docker tag for installation
- `dockerLinks` *[optional]* - Docker links                         
    - `sourceNodeGroup` - source node to be linked with another node                                
    - `alias` - prefix alias for linked variables                         
- `dockerEnvVars` *[optional]* - Docker environment variables                        
- `dockerVolumes` *[optional]* - Docker node volumes               
- `volumeMounts` *[optional]* - Docker external volumes mounts                             
- `dockerRunCmd` *[optional]* - Docker run configs                            
- `dockerEntryPoint` *[optional]* - Docker entry points                                          

<!-- SetCloudletsCount -->
### setNodeDisplayName

Available for all nodes
``` json
{
  "setNodeDisplayName [nodeId, nodeGroup, nodeType]": "string"
}
```
where:   

- `nodeId`, `nodeGroup`, `nodeType` - parameters that determine target containers for the action execution (at least one of these parameters is required)                   
- `string` - node’s display name (i.e. <a href="https://docs.jelastic.com/environment-aliases" target="_blank">alias</a>)                                                                        


### setNodeCount

Available for all nodes                  

The *setNodeCount* action allows to add or remove nodes that are grouped according to the same *nodeGroup* (layer). The node selector is available by *nodeId*, *nodeGroup*, or *nodeType*.             

``` json
{
  "setNodeCount [nodeId, nodeGroup, nodeType]": "number"
}
```
where:

- `nodeId`, `nodeGroup`, `nodeType` - parameters that determine target containers for the action execution (at least one of these parameters is required)                                                       
- `number` - total number of nodes after the action is finished                                          

### setExtIpEnabled

Available for all nodes                      

The *setExtIpEnabled* action allows to enable or disable the external IP address attachment to a particular node or *nodeGroup*.                                  

``` json
{
  "setExtIpEnabled [nodeId, nodeGroup, nodeType]": true or false
}
```
where:               

- `nodeId`, `nodeGroup`, `nodeType` - parameters that determine target containers for the action execution (at least one of these parameters is required)                                                                    
- `true` or `false` - parameter that allows to attach or detach the external IP address                              

### restartNodes

Available for all nodes (except for Elastic VPS)
``` json
{
  "restartNodes": [
    {
      "nodeId": "number or string",
      "nodeGroup": "string",
      "nodeType": "string"
    }
  ]
}
```
where:       

- `nodeId`, `nodeGroup`, `nodeType` - parameters that determine target containers for the action execution (at least one of these parameters is required)                                  

### restartContainers

Available for all nodes
``` json
{
  "restartContainers": [
    {
      "nodeId": "number or string",
      "nodeGroup": "string",
      "nodeType": "string"
    }
  ]
}
```
where:         

- `nodeId`, `nodeGroup`, `nodeType` - parameters that determine target containers for the action execution (at least one of these parameters is required)                                                            

### addContext

Available for compute nodes (except for Docker containers)
``` json
{
  "addContext": [
    {
      "name": "string",
      "fileName": "string",
      "type": "string"
    }
  ]
}
```
where:       

- `name` - context’s name    
- `fileName` - name of the file that is displayed at the dashboard                         
- `type` - context type with the following possible values:                             
    - `ARCHIVE`    
    - `GIT`    
    - `SVN`    

## Database Operations

Within this section, you can find actions that are intended for managing database containers.                 

### prepareSqlDatabase

Available for SQL databases (except for Docker containers)
``` json
{
  "prepareSqlDatabase": [
    {
      "nodeId": "number or string",
      "nodeGroup": "string",
      "nodeType": "string",
      "loginCredentials": {
        "user": "string",
        "password": "string"
      },
      "newDatabaseName": "string",
      "newDatabaseUser": {
        "name": "string",
        "password": "string"
      }
    }
  ]
}
```
where:          

- `nodeId`, `nodeGroup`, `nodeType` *[optional]* - parameters that determine target containers for the action execution. By default, the *nodeGroup* value is equal to *sqldb*.                            
- `loginCredentials` - root creadentials from a new node                    
    - `user` - username                    
    - `password` - password                 
- `newDatabaseName` - your custom database name              
- `newDatabaseUser` - new user with privileges granted for a new database instance                           
    - `name` - custom username that is set for a new database  
    - `password` - custom password that is generated for a new database 

!!! note
    The action is executed only for *mysql5*, *mariadb*, and *mariadb10* containers.                          

### restoreSqlDump

Available for SQL databases (except for Docker container)
``` json
{
  "restoreSqlDump": [
    {
      "nodeId": "number or string",
      "nodeGroup": "string",
      "nodeType": "string",
      "databaseName": "string",
      "user": "string",
      "password": "string",
      "dump": "URL"
    }
  ]
}
```
where:

- `nodeId`, `nodeGroup`, `nodeType` *[optional]* - parameters that determine target containers for the action execution. By default, the *nodeGroup* value is equal to *sqldb*.                                    
- `databaseName` - name of a database that is created                  
- `user` - username in a database, on behalf of which an application is used                
- `password` - password in a database, on behalf of which an application is used                         
- `dump` - URL to application's database dump                                

### applySqlPatch

Available for SQL databases (except for Docker containers)                                 
``` json
{
  "applySqlPatch": [
    {
      "nodeId": "number or string",
      "nodeGroup": "string",
      "nodeType": "string",
      "databaseName": "string",
      "user": "string",
      "password": "string",
      "patch": "string or URL"
    }
  ]
}
```
where:  

- `nodeId`, `nodeGroup`, `nodeType` *[optional]* - parameters that determine target containers for the action execution. By default, the *nodeGroup* value is equal to *sqldb*.                                   
- `databaseName` - name of a database for a patch to be applied                    
- `user` - username in a database, on behalf of which an application is used                                          
- `password` - password in a database, on behalf of which an application is used                              
- `patch` - SQL query or a link to it. It is used only for SQL databases. Here, the <a href="/creating-manifest/placeholders" target="_blank">placeholders</a> support is available.                    

!!! note
    The action is executed only for *mysql5*, *mariadb*, and *mariadb10* containers.                         

## User-Defined Operations

The current section provides data on the user-defined actions.                        

### script

A `script` is an ability to executing custom Java or Javascript codes. Therefore, this advantage helps to realize a custom logic.  
`executeScript` is deprecated alias.  
The simplest way to use Java or JavaScript object in your manifest in example below:

``` json
{
  "type": "update",
  "name": "Execute scripts",
  "onInstall": {
    "script": "return 'Hello World!';"
  }
}
```

A custom scripts can be set via external links instead of a **string**.  
The example execution result is a <a href="/creating-manifest/handling-custom-responses/" target="_blank">response type</a> `error` with message *"Hello World!"*.
The default action script type is `javascript`.

There is an ability to define language type or pass custom parameters. In this case the `script` action should be describe like in example below:

```json
{
  "type": "update",
  "name": "Execute scripts",
  "script": {
    "script": "return '${this.greeting}';",
    "params": {
      "greeting": "Hello World!"
    },
    "type": "js"
  }
}
```
where:   

- `script` - an object where are defined script code, optional parameters and languarge code type                                                
- `type` *[optional]* - script type with the following possible values (the default value is *'js'*):                                          
    - `js` `(javascript)` an alias    
    - `java`      
- `params` *[optional]* - script parameters. Can can be used in scripts like placeholder in example - *${this.greeting}*                               

A `script` action provides an ability to execute Jelastic API in custom scripts. Therefore, it is easy to manage Jelastic evnvironments by `scripts`.   
There are [ready-to-go solutions](/samples/#complex-ready-to-go-solutions) certified by Jelastic team.

!!! note
    Learn more about using <a href="http://docs.jelastic.com/api" target="_blank">Jelastic Cloud API</a>.    

### sleep

Setting a delay that is measured in milliseconds. The following example shows how to create a delay for one second.                                               
``` json
{
  "sleep": "1000"
}
```

### install

The *install* action allows to declare multiple installations within a single JPS manifest file. The action is available for the *install* and *update* installation types, therefore, it can initiate installation of both new environments and add-ons.                                 

**Examples**

Installing the add-on via the external link (with the *update* installation type).            
``` json
{
  "install" : {
    "jps" : "http://example.com/manifest.jps",
    "settings" : {
      "myparam" : "test"
    }
  }
}
```
where:

- `jps` - URL to your custom JPS manifest  
- `settings` - user <a href="/creating-manifest/visual-settings/" target="_blank">custom form</a>           

Installing the add-on from the local manifest file.                    
``` json
{
  "install" : {
    "type" : "update",
    "name" : "test",
    "onInstall" : {
      "log" : "install test"
    }
  }
}
```
where:

- `onInstall` - entry point for performed actions                                 

Installing the environment via the external link (with the *install* installation type).                 
``` json
{
  "install" : {
    "jps" : "http://example.com/manifest.jps",
    "envName" : "env-${fn.random}",
    "settings" : {
      "myparam" : "test"
    }
  } 
}
```
where: 

- `jps` - URL to your custom JPS manifest                    
- `envName` - short domain name of a new environment                                   
- `settings` - user <a href="/creating-manifest/visual-settings/" target="_blank">custom form</a>                                               

Installing the environment from the local manifest file.                      
``` json
{
  "install" : {
    "type" : "install",
    "region" : "dev",
    "envName" : "env-${fn.random}",
    "name" : "test",
    "nodes" : {
         "nodeType" : "apache2",
          "cloudlets" : 16
    },
    "onInstall" : {
      "log" : "install test"
    }
  }
}
```
where:

- `region` - hardware node's <a href="https://docs.jelastic.com/environment-regions" target="_blank">region</a>                                               
- `envName` - short domain name of a new environment                     
- `name` - JPS name  
- `nodes` - nodes description                                                           
- `onInstall` - entry point for performed actions                


### installAddon

You can install a <a href="/creating-manifest/addons/" target="_blank">custom add-on</a> within another - *parent* manifest. By default, custom add-ons have the *update* installation type.                                      

Thus, the custom add-on can be installed to the:                                         

- existing environment with the *update* installation type                         

- new environment with the *install* installation type. In this case, add-ons (if there are several ones) are installed sequentially one by one right after a new environment creation.                                                                   

The example below shows how to pass the add-on identifier to the *installAddon* action. This add-on's parameters are described in the *addons* section. As a result, the custom add-on with the *firstAddon* identifier initiates the creation of a new file in the *tmp* directory on the compute node layer.                                                 
``` json
{
	"type": "update",
	"name": "Install Add-on example",
	"onInstall": {
		"installAddon": {
			"id": "firstAddon"
		}
	},
	"addons": [{
		"id": "firstAddon",
		"name": "firstAddon",
		"onInstall": {
			"createFile [cp]": "/tmp/exampleFile.txt"
		}
	}]
}
```
where:  

- `id` - identifier of a custom add-on                           

You can locate the installed add-ons within the **Add-ons** tab at the Jelastic dashboard. 

<center>![new-addon](/img/new-addon.png)</center>

In the following example, the *nodeGroup* parameter is passed to the *installAddon* action, targeting the add-on at the balancer (*bl*) node group.                          
``` json
{
  "installAddon": {
    "id": "firstAddon",
    "nodeGroup": "bl"
  }
}
```

For more details about the <a href="/creating-manifest/addons/" target="_blank">add-ons</a> installation, visit the linked page.                                              

<!-- add example -->

### return

The action allows to return any string or object of values. As a result, the response is displayed via the pop-up window. By default, the *error* pop-up window is used.                       

**Example**      

```json
{
    "type": "update",
    "name": "Return Action",
    "onInstall": {
        "return": "Hello World!"
    }
}
```

Through the example above, the pop-up window with the following text is returned.                             
<center>![returnHelloWorld](/img/returnHelloWorld.jpg)</center>

The installation is not completed and the following installation window is displayed.                    

<center>![returnHelloWorld](/img/redCross.jpg)</center>

If the *return* action includes a string, then the response is displayed via the *error* pop-up window like in the screen-shot below.                            

```json
{
    "type": "update",
    "name": "Return Action",
    "onInstall": {
        "return": "{\"data\": \"${nodes.cp.id}\",\"type\": \"success\"}"
    }
}
```

The result window also returns the compute node's unique identifier at Jelastic Platform.                                                
<center>![returnNodeId](/img/returnNodeId.jpg)</center>

If the action returns an object, a response code can be redefined. So the *message* or *result* code parameters are required in the *return* object. Herewith, a zero (0) *result* code is not passed to the response code.        

Through the following example, a success message with a compute node identifier is displayed.                                                                          
```json
{
  "type": "update",
  "name": "Return Action",
  "onInstall": {
    "return": {
      "type": "success",
      "data": "${nodes.cp.id}",
      "message": "Compute node unique identifer - ${nodes.cp.id}"
    }
  }
}
```

For more details about [*Custom Response*](/creating-manifest/handling-custom-responses/), visit the linked page.                                    

All the other actions within the *onInstall* array are not executed after the *return* action.                
 
```json
{
    "type": "update",
    "name": "Return Action",
    "onInstall": [{
            "return": {
                "type": "success",
                "data": "${nodes.cp.id}",
                "message": "Compute node unique identifer - ${nodes.cp.id}"
            }
        },
        "restartNodes [cp]"
    ]
}
```

Therefore, the *restartNodes* action is not run to restart a compute node.                                                            

## Custom Actions

Particular actions can be run by means of calling actions with different parameters.             

The example below shows how to create a new file (e.g. the <b>*example.txt*</b> file in the <b>*tmp*</b> directory) by running a *createFile* action on the compute node.                 
``` json
{
  "type": "update",
  "name": "execution actions",
  "onInstall": {
    "createFile [cp]": "/tmp/example.txt"
  }
}
```
where: 

 - `createFile` - corresponding [*createFile*](#createfile) action                     

The next example illustrates how to create a new custom action (i.e. *customAction*) that can be called for several times.                                                        
``` json
{
	"type": "update",
    "name": "execution actions",
	"onInstall": "customAction",
	"actions": {
		"customAction": {
			"createFile [cp]": "/tmp/example.txt"
		}
	}
}
```
where:  

- `actions` - object where custom actions can be predefined                                    

### Action Placeholders

In order to access any required data or parameters of allocated resources inside a manifest, a special set of placeholders should be used. The parameters, sent to a call method, are transformed into a separate kit of placeholders, which can be further used within the appropriate actions by means of *${this}*  namespace. Access to a node inside environment can be gained according to its type, as well as according to its role in the environment.                             

The example below illustrates how to pass the dynamic parameters for running in the action. Here, the *name* parameter is sent to <b>*customAction*</b> where the *createFile* action is executed.                   

```json
{
    "type": "update",
    "name": "$this in Custom Actions",
    "onInstall": {
        "customAction": {
            "name": "simpleTxtFile"
        }
    },
    "actions": {
        "customAction": {
            "createFile [cp]": "/tmp/${this.name}.txt"
        }
    }
}
```

Therefore, the same custom actions can be reused for several times with different parameters. Moreover, any action can be targeted at a specific node by ID, at a particular layer (*nodeGroup*) or *nodeType*. For more details about <a href="/creating-manifest/selecting-containers/#types-of-selectors" target="_blank">*Node Selectors*</a>, visit the linked page.                             
 
### Code Reuse

You can use the already-existing code to perform a new action.                    

For example, outputting Hello World! twice in the <b>*greeting.txt*</b>.                                     
``` json
{
  "type": "update",
  "name": "Actions Example",
  "onInstall": [
    {
      "createFile [cp]": "${SERVER_WEBROOT}/greeting.txt"
    },
    "greeting",
    "greeting"
  ],
  "actions": {
    "greeting": {
      "appendFile [cp]": [
        {
          "path": "${SERVER_WEBROOT}/greeting.txt",
          "body": "Hello World!"
        }
      ]
    }
  }
}
```

### Call Action with Parameters

The following example shows how to pass additional parameters to the custom action. The parameters should be passed as an object to the custom action.                 

``` json
{
	"type": "update",
    "name": "execution actions",
	"onInstall": {
		"customAction": {
			"fileName": "example.txt"
		}
	},
	"actions": {
		"customAction": {
			"createFile [cp]": "/tmp/${this.fileName}.txt"
		}
	}
}
```
where:

- `fileName` - additional parameter

Writing Hello World! and outputting the first and the second compute nodes IP addresses.                                                             
``` json
{
  "type": "update",
  "name": "Action Example",
  "onInstall": [
    {
      "createFile [cp]": "${SERVER_WEBROOT}/greeting.txt"
    },
    "greeting",
    "greeting",
    {
      "log": {
        "message": "${nodes.cp[0].address}"
      }
    },
    {
      "log": {
        "message": "${nodes.cp[1].address}"
      }
    }
  ],
  "actions": {
    "greeting": {
      "appendFile [cp]": {
        "path": "${SERVER_WEBROOT}/greeting.txt",
        "body": "Hello World!"
      }
    },
    "log": {
      "appendFile [cp]": {
        "path": "${SERVER_WEBROOT}/greeting.txt",
        "body": "${this.message}"
      }
    }
  }
}
```
<br>       
<h2>What’s next?</h2>                   

- See the <a href="/creating-manifest/events/" target="_blank">Events</a> list the actions can be bound to            

- Find out the list of <a href="/creating-manifest/placeholders/" target="_blank">Placeholders</a> for automatic parameters fetching           

- See how to use <a href="/creating-manifest/conditions-and-iterations/">Conditions and Iterations</a>                                  

- Read how to integrate your <a href="/creating-manifest/custom-scripts/" target="_blank">Custom Scripts</a>                                               

- Learn how to customize <a href="/creating-manifest/visual-settings/" target="_blank">Visual Settings</a>                                    

- Examine a bunch of <a href="/samples/" target="_blank">Samples</a> with operation and package examples                                           