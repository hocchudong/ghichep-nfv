# Cloudify Manager lab

In this lab you will bootstrap a Cloudify Manager instance on the provided alice VM using the simple blueprint.

alice not only meets the minimum system requirements for the Manager (4GB of RAM and 2 vCPUs), but has already been configured with suitable credentials. You can check by SSHing there and verifying the virtual hardware available:

$ ssh alice
$ free -m
$ cat /proc/cpuinfo
## FETCH THE MANAGER BLUEPRINTS

Go back to the deploy node: you'll need the CLI environment you installed earlier.

You'll also need the official simple blueprint. To get it, clone the cloudify-manager-blueprints repository:

$ git clone -b 3.3.1 \
  https://github.com/cloudify-cosmo/cloudify-manager-blueprints.git
This will create a new cloudify-manager-blueprints directory under the training home directory.

## A NEW WORKING DIRECTORY

For clarity and convenience, you'll create a new working directory for the bootstrapping process. This lab assumes that the chosen directory is ~/work:

$ mkdir ~/work
$ cd ~/work
## CONFIGURE THE INPUTS FILE

The provided manager blueprints ship with templates for manager inputs. These templates have to be edited to reflect the environment in which the manager is to be installed.

Start by copying the appropriate input template into ~/work, and editing it:

$ cp ~/cloudify-manager-blueprints/simple-manager-blueprint-inputs.yaml manager-inputs.yaml

$ vi manager-inputs.yaml
Since you'll deploy the manager to alice, change the inputs as follows:

```sh
public_ip: '192.168.122.111'
private_ip: '192.168.122.111'
ssh_user: 'training'
ssh_key_filename: '~/.ssh/id_rsa'
agents_user: 'training'
resources_prefix: ''
Make sure to save the file before exiting.
```

## TRIGGER THE BOOTSTRAP PROCESS

Before continuing, make sure your virtualenv is activated by running the following, if it isn't:

$ source /opt/cfy/env/bin/activate
Next, initialize the working directory:

$ cfy init -r
Finally, start the bootstrap procedure with the following. It should take around 15 minutes to complete, during which you will see the output of the bootstrapping process. At the end you should see the IP address of the Manager as an output.

$ cfy bootstrap --install-plugins \
  -p ../cloudify-manager-blueprints/simple-manager-blueprint.yaml \
  -i manager-inputs.yaml
...
bootstrapping complete
management server is up at 192.168.122.111
## VERIFY THAT THE MANAGER STARTED SUCCESSFULLY

Type the following command to verify that all manager components are up and running:

$ cfy status
You should see output similar to the following. Make sure all components are "running":

Getting management services status... [ip=192.168.122.111]

Services:
```sh
+--------------------------------+---------+
|            service             |  status |
+--------------------------------+---------+
| InfluxDB                       | running |
| Celery Management              | running |
| Logstash                       | running |
| RabbitMQ                       | running |
| AMQP InfluxDB                  | running |
| Manager Rest-Service           | running |
| Cloudify UI                    | running |
| Webserver                      | running |
| Riemann                        | running |
| Elasticsearch                  | running |
+--------------------------------+---------+
```

## ACCESS THE WEB UI

To access the Manager's web interface directly, you'll need to find out the alice VM's public IP address. To do so, SSH into it and run:

$ ssh alice
$ curl http://icanhazip.com
...
91.106.193.189
Using your browser, navigate to your Cloudify Manager's public IP address. For example:

http://91.106.193.189