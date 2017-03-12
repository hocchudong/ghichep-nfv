# Installing the Cloudify CLI

In this lab you will install the Cloudify CLI tools to the deploy VM. Before you go on, make sure you're back at deploy:

$ hostname
deploy
Start by installing the latest version of the Python installer, pip. You'll use wget to retrieve the pip bootstrap script, and then run it as root on deploy:

$ wget https://bootstrap.pypa.io/get-pip.py
$ sudo python get-pip.py
...
Successfully installed pip-8.1.2 wheel-0.29.0
Now, fetch the Cloudify CLI package from the cloudifysource repository, and install it:

$ wget http://repository.cloudifysource.org/org/cloudify3/3.3.1/sp-RELEASE/cloudify-centos-Core-cli-3.3.1-sp_b310.x86_64.rpm
$ sudo yum -y install cloudify-centos-Core-cli-3.3.1-sp_b310.x86_64.rpm
...
Installed:
  cloudify-centos-Core-cli.x86_64 0:3.3.1-sp_b310

## ACTIVATING THE VIRTUALENV

Next, activate the provided Cloudify CLI virtual environment. The virtual environment remains in effect until you either deactivate it (using the deactivate command), or log out:

$ source /opt/cfy/env/bin/activate
...
(env)[training@deploy ~]$
You can test that this works by running the following command:

$ cfy --version
...
Cloudify CLI 3.3.1
If the cfy virtualenv is not activated, the cfy command will not be found. Try it:

$ deactivate
$ cfy --version
...
-bash: cfy: command not found
To reactivate the cfy virtualenv, source the activate file as described above.

CHECK YOUR PROGRESS

Before moving on to the next section, please press the "Check progress" button below the lab terminal. This will check that you completed the instructions above correctly in your lab environment, and grade it accordingly.

If the automated check points out a mistake, please go back to your environment and double-check that all steps were successfully executed, fix problems if necessary, and press the "Check progress" button again