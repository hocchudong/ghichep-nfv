# Installing Nodecellar locally


In this lab you will install a sample application, "NodeCellar", using the Cloudify CLI, and through it gain an understanding of how Cloudify uses blueprints.

(NodeCellar was created by Christophe Coenraets to demonstrate the usage of various technologies such as Backbone.js, Node.js, and MongoDB).

## GETTING THE BLUEPRINTS

To begin, check out the NodeCellar blueprints into the home directory on the deploy node:

$ cd ~
$ git clone -b 3.3.1-maint-hastexo \
  https://github.com/Cloudify-PS/cloudify-nodecellar-example.git
This will create a cloudify-nodecellar-example directory.

## INSPECTING THE BLUEPRINT

You will deploy local-blueprint.yml shortly, but first, take a quick look at it:

$ vi cloudify-nodecellar-example/local-blueprint.yaml
The description at the top gives you a hint as to what will happen: you will deploy NodeCellar locally, to the deploy node itself.

This blueprint takes a single input, host_ip, and specifies four nodes: a host, the nodejs and mongod services, and the nodecellar application itself.

It also sets the relationships between the nodes. You've seen the contained_in built-in type before; but the relationships between nodecellar and the two services are new. They're defined in types/nodecellar.yaml, and imported in the imports section of the blueprint. Feel free to take a look at that file as well, but a detailed explanation falls outside the scope of this lab. It is enough to know that each of node_connected_to_mongo and node_contained_in_nodejs do what you'd expect: they configure NodeCellar to connect to each service.

The blueprint also provides an output, endpoint, which lists the IP address and port of the deployed application.

## INITIALIZING A WORKING DIRECTORY

You will need an empty directory to initialize the blueprint with the cfy client. Create one now:

$ mkdir cfylocal
Next, make sure your Cloudify virtual environment is activated. If it is, your prompt should be prefixed with (env):

(env)[training@deploy ~]
If it isn't, please activate it before proceeding:

$ source /opt/cfy/env/bin/activate
...
(env)[training@deploy ~]
Next, change into the empty directory and initialize it with the blueprint you were looking at previously:

$ cd cfylocal
$ cfy local init -p ../cloudify-nodecellar-example/local-blueprint.yaml
...
Initiated ../cloudify-nodecellar-example/local-blueprint.yaml

## DEPLOYING

You're now ready to deploy NodeCellar. From within the cfylocal directory, run the following command:

$ cfy local execute -w install
...
2016-05-24 15:27:12 CFY <local> 'install' workflow execution succeeded
You should now see the CLI in action: iterating through the blueprint's nodes, creating them, starting them, and instantiating relationships.

## CHECKING SERVICES

Before continuing, inspect the deploy VM to check if the required services were installed. First, check that the nodejs and mongod services are running. You can do so by listing all running processes, and looking for each; they should be at the very bottom of the listing (use the PageUp/PageDown keys on your keyboard to navigate it, and press 'q' to exit):

$ ps -AHfww | less
You can also check that nodejs is listening on the configured port by running:

$ netstat -lntp | grep 8080
...
tcp 0 0 0.0.0.0:8080 0.0.0.0:* LISTEN 23079/node
Test the application

The application is now installed. To test it, you will point your browser to it. First, though, you'll need the IP address and port. The latter can be obtained from Cloudify itself:

$ cfy local outputs
...
{
  "endpoint": {
    "ip_address": "localhost", 
    "port": 8080
  }
}
The port is 8080, but you'll need the deploy VM's public IP address. An easy way to get it is to run the following command:

$ curl http://icanhazip.com
...
91.106.193.188
You will get a different IP address from the above, but using it as an example, you would type the following into the address bar of a new tab in your browser:

http://91.106.193.188:8080
If everything worked correctly, you should see the NodeCellar front page.

