# Installing the Cloudify CLI

In this lab you will install the Cloudify CLI tools to the deploy VM. Before you go on, make sure you're back at deploy:

```sh
hostname
# sample output: deploy
```

Start by installing the latest version of the Python installer, pip. You'll use wget to retrieve the pip bootstrap script, and then run it as root on deploy:

```sh
wget https://bootstrap.pypa.io/get-pip.py
sudo python get-pip.py
```

Now, fetch the Cloudify CLI package from the cloudifysource repository, and install it:

```sh
wget http://repository.cloudifysource.org/org/cloudify3/3.3.1/sp-RELEASE/cloudify-centos-Core-cli-3.3.1-sp_b310.x86_64.rpm
sudo yum -y install cloudify-centos-Core-cli-3.3.1-sp_b310.x86_64.rpm
```

## ACTIVATING THE VIRTUALENV

Next, activate the provided Cloudify CLI virtual environment. The virtual environment remains in effect until you either deactivate it (using the deactivate command), or log out:

```sh
source /opt/cfy/env/bin/activate
```

You can test that this works by running the following command:

```sh
$ cfy --version
...
Cloudify CLI 3.3.1
```
If the cfy virtualenv is not activated, the cfy command will not be found. Try it:

```sh
$ deactivate
$ cfy --version
...
-bash: cfy: command not found
```

To reactivate the cfy virtualenv, source the activate file as described above.

