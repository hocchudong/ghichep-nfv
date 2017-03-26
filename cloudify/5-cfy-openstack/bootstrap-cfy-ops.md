# Bootstrap Cloudify Manager for integration with OpenStack

## OpenStack requirements
- OpenStack Mitaka installed on Ubuntu 14.04 with Keystone v2.0 (with key projects installed: Keystone, Glance, Nova, Neutron, Cinder).
- If you install OpenStack Mitaka using the guide on OpenStack document main page, default Keystone version should be v3.0. To using Keystone v2.0, follow these steps:
 - Get default domain id: `openstack domain list`
 - Open keystone config file: `vi /etc/keystone/keystone.conf` and set value of `default_domain_id` inside `[identity]` section to the default domain id you've gotten from previous step:

 ```sh
 [identity] 
 default_domain_id  = xxxxxxxx32ea5xxxxx2b6f93 
 ```

 Save changes and exit!

 - Don't forget to run Keystone DB sync:

 ```sh
 su -s /bin/sh -c "keystone-manage --config-file /etc/keystone/keystone.conf db_sync" keystone
 ```
 - Create additional keystone endpoint for keystone v2.0:

 ```sh
 source admin-openrc
 openstack endpoint create --region regionOne identity internal http://controller:5000/v2.0
 openstack endpoint create --region regionOne identity admin http://controller:35357/v2.0 
 openstack endpoint create --region regionOne identity public http://controller:5000/v2.0 
 ```

 - Create credential script for keystone v2.0 as below (don't set 2 environment variables `OS_PROJECT_DOMAIN_NAME` and `OS_USER_DOMAIN_NAME`). You can use openrc files for keystone v3 along side v2.0:

 ```sh
 cat << EOF > admin-openrc-v2
 export OS_PROJECT_NAME=admin
 export OS_USERNAME=admin
 export OS_PASSWORD=Welcome123
 export OS_AUTH_URL=http://controller:35357/v2.0
 export OS_IDENTITY_API_VERSION=2
 export OS_IMAGE_API_VERSION=2
 EOF
 ```

 - Downloads and create additional image and flavors for cloudify manager (cloudify manager as an instance on OpenStack cloud) and upload to glance store:

 ```sh
 wget -c http://cloud.centos.org/centos/7/images/CentOS-7-x86_64-GenericCloud-1511.qcow2
 source admin-openrc
 openstack image create "centos-7.2-1511"   --file CentOS-7-x86_64-GenericCloud-1511.qcow2   --disk-format qcow2 --container-format bare   --public
 openstack flavor create --public m1.cfymgr --id auto --ram 8192 --disk 40 --vcpus 4
 openstack flavor create --public m1.cfyagt --id auto --ram 2048 --disk 10 --vcpus 2
 export CFY_MGR_IMG_ID=`openstack image show centos-7.2-1511 -c id -f value`
 export CFY_MGR_FLAVOR_ID=`openstack flavor show m1.cfymgr -c id -f value`
 export CFY_AGT_FLAVOR_ID=`openstack flavor show m1.cfyagt -c id -f value`
 ```

## Cloudify setup

### Cloudify CLI
- On the OpenStack Controller node, install required python packages:

```sh
sudo add-apt-repository ppa:fkrull/deadsnakes-python2.7
sudo apt-get update
sudo apt-get install python2.7 python-dev python-pip build-essential -y --force-yes
sudo pip install --upgrade pip
sudo pip install virtualenv virtualenvwrapper
```

- Get cloudify CLI script and install cloudify CLI:

```sh
wget http://gigaspaces-repository-eu.s3.amazonaws.com/org/cloudify3/get-cloudify.py
python get-cloudify.py -e cfy-cli --installvirtualenv
# active virtualenv for cloudify cli 
source cfy-cli/bin/activate
```

Or you can install a specific version of Cloudify:

```sh
virtualenv cfy-cli
source cfy-cli/bin/activate
pip install cloudify==3.4.2 
```





