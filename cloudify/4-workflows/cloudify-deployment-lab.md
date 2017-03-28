# Deploying with the Manager

In this lab you'll once again use a Cloudify blueprint to install the NodeCellar sample application. This time around, however, you'll do so via the Manager instance you bootstrapped in the previous lab. You'll upload the blueprint, create a deployment for it, and finally install it in a distributed manner to the bob and charlie VMs.

## COMPARE BLUEPRINTS

The cloudify-nodecellar-example repository you cloned previously also contains a blueprint suitable for installing NodeCellar to two pre-created machines. You'll find it in the following file on deploy:

```sh
$ vi ~/cloudify-nodecellar-example/simple-blueprint.yaml
```

In contrast to local-blueprint.yaml, which installs everything to the box where the Cloudify CLI itself resides, simple-blueprint.yaml installs the NodeCellar application to a separate VM, its database to another, and configures both so they can communicate correctly.

When configuring the simple blueprint inputs, you'll define nodejs_host_ip as bob's IP address, and mongod_host_ipas charlie's.

## SSH KEY TRANSFER

Because Cloudify Manager will be deploying NodeCellar from alice, as opposed to the CLI on deploy, you'll need to make the cluster's SSH private key available to it as well. To do so, simply copy it from the deploy node onto alice, as follows:

```sh
$ scp ~/.ssh/id_rsa alice:.ssh/
```

## INPUTS FILE

The NodeCellar example repository also contains a template for the simple blueprint's inputs file. This template should be edited to reflect your environment. You'll continue using ~/work, which you created previously to bootstrap the Manager instance, so start by changing to that directory:

```sh
$ cd ~/work

$ cp ../cloudify-nodecellar-example/inputs/simple.yaml.template nc-simple.yaml

$ vi nc-simple.yaml
```

Fill it in with the following values:

```sh
nodejs_host_ip: '192.168.122.112'
mongod_host_ip: '192.168.122.113'
agent_user: 'training'
agent_private_key_path: '/home/training/.ssh/id_rsa'
```

As discussed above, nodejs_host_ip is the IP address of the node that will receive the NodeCellar application code: you'll use bob's address for it, 192.168.122.112. mongod_host_ip configures the IP address of the database node: in your case, that must be charlie's IP, 192.168.122.113.

As before, you'll configure Cloudify to use the "training" user to connect to the other VMs. Again, you'll tell it to use the SSH private key that was pre-generated for you. The path to it, however, refers to the copy you made on alice - the one available to the Cloudify Manager instance.

## UPLOAD THE BLUEPRINT

At this point, you're ready to upload the static blueprint to the Manager. Provided you have already activated your CLI's virtualenv, this is how you should go about it:

```sh
$ cfy blueprints upload \
  -p ../cloudify-nodecellar-example/simple-blueprint.yaml \
  -b nodecellar
```

-p selects the blueprint file, and -b gives it an identifier; in this case, "nodecellar".

You should output similar to the following:

```sh
Validating cloudify-nodecellar-example/simple-blueprint.yaml
Blueprint validated successfully
Uploading blueprint cloudify-nodecellar-example/simple-blueprint.yaml to management server 192.168.122.111
Uploaded blueprint, blueprint's id is: nodecellar
```

You can further verify that the upload was successful by going to the Manager's web interface, and checking that the nodecellar blueprint is listed.

Reminder: to access your Manager UI, you need alice's public IP address, which you can obtain as follows:

```sh
$ ssh alice

$ curl http://icanhazip.com
...
188.212.108.187
```

You can then enter this IP address directly into your browser's address bar.

## CREATE A DEPLOYMENT

Once NodeCellar simple blueprint is uploaded, you'll create a deployment for it, using the inputs file customized above. Back in the ~/work directory on deploy, execute the following:

```sh
$ cd ~/work

$ cfy deployments create -b nodecellar -i nc-simple.yaml -d nc-dep-1
```

In the above, -b selects the nodecellar blueprint you uploaded previously. -i uses the customized input file, and -d sets the deployment's name.

You should see the output similar to the following:

```sh
Creating new deployment from blueprint nodecellar at management server 192.168.122.111

Deployment created, deployment's id is: nc-dep-1
```

## EXECUTE THE INSTALL WORKFLOW

Once the nc-dep-1 deployment has been created, you can actually install NodeCellar. Trigger the install workflow by running, still from deploy:

```sh
$ cfy executions start -d nc-dep-1 -w install
```

-d selects the deployment you created above, and -w specifies the workflow.

You should see several events printed to the screen as the application is installed:

```sh
2016-05-25T12:21:31 CFY <nc-dep-1> [nodecellar_529d1.configure] Task succeeded 'script_runner.tasks.run'
Finished executing workflow 'install' on deployment 'nc-dep-1'
* Run 'cfy events list --include-logs --execution-id 1fb740e0-151e-4edd-85f5-f87ac77d3ffc' to retrieve the execution's events/logs
```

If you require access to the logs after installation, run the command line described in the last line of the output. There's no need to remember the execution ID; it is easy to find it for a particular deployment with the following command:

```sh
$ cfy executions list -d nc-dep-1
```

Alternatively, these and other logs can be read in the Logs & Events tab in the Manager UI.

## CHECK SERVICE INSTALLATION

Now, check that the nodejs and mongod services were installed and enabled on the expected VMs: nodejs on bob, and mongod on charlie.

SSH into bob and look for nodejs in the process list. It should be at the very bottom. Use PageUp, PageDown, or '/' to search:

```sh
$ ssh bob

$ ps -AHfww | less

...
training 17515     1  0 12:20 ?        00:00:00   /tmp/1fb740e0-151e-4edd-85f5-f87ac77d3ffc/nodejs/nodejs-binaries/bin/node /tmp/1fb740e0-151e-4edd-85f5-f87ac77d3ffc/nodecellar/nodecellar-source/server.js
```

Next, go back to deploy and SSH into charlie, looking for the mongod process:

```sh
$ ssh charlie

$ ps -AHfww | less

...

training 17271     1  0 12:19 ?        00:01:35   /tmp/1fb740e0-151e-4edd-85f5-f87ac77d3ffc/mongodb/mongodb-binaries/bin/mongod --port 27017 --dbpath /tmp/1fb740e0-151e-4edd-85f5-f87ac77d3ffc/mongodb/data --rest --journal --shardsvr --smallfiles
```

## ACCESS NODECELLAR

To see this installation of NodeCellar in action, you must find out where it is deployed. To do so, start by going back to deploy and getting the outputs from the nc-dep-1 deployment (the one you just installed):

```sh
$ cfy deployments outputs -d nc-dep-1
...
 - "endpoint":
     Description: Web application endpoint
     Value: {u'ip_address': u'192.168.122.112', u'port': 8080}
```

As expected, you can see the application itself has been deployed to bob's IP address, at port 8080. However, this IP is not accessible from your browser: you have to find bob's public address. You go about it like you did for alice, previously:

```sh
$ ssh bob

$ curl http://icanhazip.com
...
188.212.108.185
```

In a new browser tab, navigate to http://[BOB_PUBLIC_IP]:8080, where BOB_PUBLIC_IP is the address you obtained, and you should see NodeCellar's front page.