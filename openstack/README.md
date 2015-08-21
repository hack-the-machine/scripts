#OpenStack deploy with DevStack

### Table of Contents
**[Create the Virtual Machine](#create-the-virtual-machine)**
**[Setting up the system](#setting-up-the-system)**

- **[Edit network Interfaces](#edit-network-interfaces)**
- **[Add OVS Bridges](#add-ovs-bridges)**

**[Setting up the OpenStack environment](#setting-up-the-openstack-environment)**

- **[Clone DevStack](#clone-devstack)**
- **[Change local.conf](#change-local-conf)**
- **[Edit ~/.bashrc](#edit-bashrc)**
- **[Run stack.sh](#run-stack-sh)**

**[Troubleshooting][#troubleshooting]**

##Create the Virtual Machine
- Processors:
	- Number of processors: 2
	- Number of cores per processor 1
- Memory: 6GB RAM (Recommended)
- HDD - SATA - Minimum 20 GB (Recommended *Preallocated*)
- Network:
	- Network Adapter 1:  **NAT**
	- Network Adapter 2:  **Host Only**
	- Network Adapter 3:  **Nat**
- Operating system - **Ubuntu Server 14.04** (Recommended) 

Note: The Hypervisor used for this example is **KVM**

##Setting up the system
```
# Update the apt-get
$> sudo apt-get update

# Update the system
$> sudo apt-get upgrade

# Install the required tools
$> sudo apt-get install -y git vim openssh-server openvswitch-switch ethtool

# Disable the firewall
$> sudo ufw disable

# Disable rx/tx vlan offloading
$> sudo ethtool -K eth1 txvlan off rxvlan off
```

###Edit network Interfaces
```
$> sudo vim /etc/network/interfaces
```

**IMPORTANT:** This is a template. Please use your own settings.
```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto eth0
iface eth0 inet static
	address 192.168.122.200
	netmask 255.255.255.0
	gateway 192.168.122.1
	broadcast 192.168.122.255
	dns-nameserver 8.8.8.8 8.8.4.4

# The public interface
auto eth2
iface eth2 inet manual
	up ip link set eth2 up
	down ip link set eth2 down

# The management interface
auto eth1
iface eth1 inet manual
	up ip link set eth1 up
	up ip link set eth1 promisc on
	down ip link set eth1 promisc off
	down ip link set eth1 down

```

**IMPORTANT:** After you edit ```/etc/network/interfaces``` the ```network service``` should be restarted.
```
$> sudo service network restart
```

### Add OVS Bridges
```
$> sudo ovs-vsctl add-br br-eth1
$> sudo ovs-vsctl add-port br-eth1 eth1

$> sudo ovs-vsctl add-br br-ex
$> sudo ovs-vsctl add-port br-ex eth2
```

##Setting up the OpenStack environment

###Clone DevStack
```
$> cd
$> git clone https://github.com/openstack-dev/devstack.git
$> cd devstack
```
###Change local.conf
```
$> sudo vim ~/devstack/local.conf
```

**IMPORTANT:** The following config file is a template. Please use your own settings.
```
[[local|localrc]]
HOST_IP=192.168.122.200
DEVSTACK_BRANCH=stable/kilo

# Change the following password
DATABASE_PASSWORD=CHANGE_ME
RABBIT_PASSWORD=CHANGE_ME
SERVICE_TOKEN=CHANGE_ME
SERVICE_PASSWORD=CHANGE_ME
ADMIN_PASSWORD=CHANGE_ME

#Services to be started
enable_service rabbit
enable_service mysql

enable_service key

enable_service n-api
enable_service n-crt
enable_service n-obj
enable_service n-cond
enable_service n-sch
enable_service n-cauth
enable_service n-novnc
# Disable Nova networking
disable_service n-net

enable_service neutron
enable_service q-svc
enable_service q-agt
enable_service q-dhcp
enable_service q-l3
enable_service q-meta
enable_service q-lbaas
enable_service q-fwaas
enable_service q-metering
enable_service q-vpn

enable_service horizon

enable_service g-api
enable_service g-reg

enable_service cinder
enable_service c-api
enable_service c-vol
enable_service c-sch
enable_service c-bak

disable_service s-proxy
disable_service s-object
disable_service s-container
disable_service s-account

enable_service heat
enable_service h-api
enable_service h-api-cfn
enable_service h-api-cw
enable_service h-eng

disable_service ceilometer-acentral
disable_service ceilometer-collector
disable_service ceilometer-api

disable_service tempest

# To add a local compute node, enable the following services
disable_service n-cpu
disable_service ceilometer-acompute

IMAGE_URLS+=",https://people.debian.org/~aurel32/qemu/amd64/debian_wheezy_amd64_standard.qcow2"

Q_PLUGIN=ml2
Q_ML2_PLUGIN_MECHANISM_DRIVERS=openvswitch
Q_ML2_TENANT_NETWORK_TYPE=vlan

PHYSICAL_NETWORK=physnet1
OVS_PHYSICAL_BRIDGE=br-eth1
OVS_BRIDGE_MAPPINGS=physnet1:br-eth1

OVS_ENABLE_TUNNELING=False
ENABLE_TENANT_VLANS=True
TENANT_VLAN_RANGE=500:2000

GUEST_INTERFACE_DEFAULT=eth1
PUBLIC_INTERFACE_DEFAULT=eth2

FLOATING_RANGE=192.168.122.0/24
PUBLIC_NETWORK_GATEWAY=192.168.122.1
Q_FLOATING_ALLOCATION_POOL=start=192.168.122.100,end=192.168.122.199

FIXED_RANGE=10.0.2.0/24
NETWORK_GATEWAY=10.0.2.1

CINDER_SECURE_DELETE=False
VOLUME_BACKING_FILE_SIZE=50000M

LIVE_MIGRATION_AVAILABLE=False
USE_BLOCK_MIGRATION_FOR_LIVE_MIGRATION=False

LIBVIRT_TYPE=kvm
API_RATE_LIMIT=False

SCREEN_LOGDIR=/opt/stack/logs/screen
VERBOSE=True
LOG_COLOR=False

KEYSTONE_BRANCH=$DEVSTACK_BRANCH
NOVA_BRANCH=$DEVSTACK_BRANCH
NEUTRON_BRANCH=$DEVSTACK_BRANCH
SWIFT_BRANCH=$DEVSTACK_BRANCH
GLANCE_BRANCH=$DEVSTACK_BRANCH
CINDER_BRANCH=$DEVSTACK_BRANCH
HEAT_BRANCH=$DEVSTACK_BRANCH
TROVE_BRANCH=$DEVSTACK_BRANCH
HORIZON_BRANCH=$DEVSTACK_BRANCH
TROVE_BRANCH=$DEVSTACK_BRANCH
REQUIREMENTS_BRANCH=$DEVSTACK_BRANCH

[[post-config|$NEUTRON_CONF]]
[database]
min_pool_size = 5
max_pool_size = 50
max_overflow = 50

```
More information regarding local.conf can be found on [Devstack configuration](http://docs.openstack.org/developer/devstack/configuration.html).

###Edit ~/.bashrc
```
$> vim ~/.bashrc
```

Add this lines at the end of file.
```
export OS_USERNAME=admin
export OS_PASSWORD=Passw0rd
export OS_TENANT_NAME=admin
export OS_AUTH_URL=http://127.0.0.1:5000/v2.0
```

###Run stack.sh
```
$> cd ~/devstack
$> ./stack.sh
```

**IMPORTANT:** If the scripts doesn't end properly or something else goes wrong, please unstack first using ```./unstack.sh``` script.


#Troubleshooting

##OpenStack role list raises unrecognized arguments: --group

```
::./stack.sh:780+openstack role list --group 3c65c1a8d12f40a2a9949d5b2922beae --project 18ab3a46314442b183db43bc13b175b4 --column ID --column Name
usage: openstack role list [-h] [-f {csv,html,json,table,yaml}] [-c COLUMN]
                           [--max-width <integer>]
                           [--quote {all,minimal,none,nonnumeric}]
                           [--project <project>] [--user <user>]
openstack role list: error: unrecognized arguments: --group 3c65c1a8d12f40a2a9949d5b2922beae
```

Code location at lib/keystone:418, invoked by functions-common:773.

The first reason is that the python-openstackclient version is too old (`openstack --version`), upgrade it:

```
sudo pip install --upgrade python-openstackclient
```

You need to add python-openstackclient to LIBS_FROM_GIT in local.conf, to make sure devstack uses the newest version of python-openstackclient. Note that, devstack will use master branch of python-openstackclient instead of stable/kilo.

```
# Add python-openstackclient to your LIBS_FROM_GIT
LIBS_FROM_GIT=python-openstackclient
```

The next step, since keystone v2.0 doesn't even have the concept "group", you need to force here to use keystone V3 api.

```
$ git diff
diff --git a/functions-common b/functions-common
index d3e93ed..bd55d7e 100644
--- a/functions-common
+++ b/functions-common
@@ -773,12 +773,15 @@ function get_or_add_user_project_role {
 # Gets or adds group role to project
 # Usage: get_or_add_group_project_role <role> <group> <project>
 function get_or_add_group_project_role {
+    local os_url="$KEYSTONE_SERVICE_URI_V3"
     # Gets group role id
     local group_role_id=$(openstack role list \
         --group $2 \
         --project $3 \
         --column "ID" \
         --column "Name" \
+        --os-identity-api-version=3 \
+        --os-url=$os_url \
         | grep " $1 " | get_field 1)
     if [[ -z "$group_role_id" ]]; then
         # Adds role to group
@@ -786,6 +789,8 @@ function get_or_add_group_project_role {
             $1 \
             --group $2 \
             --project $3 \
+            --os-identity-api-version=3 \
+            --os-url=$os_url \
             | grep " id " | get_field 2)
     fi
     echo $group_role_id
```

```
wget https://goo.gl/tkJdKW -o functions-common.diff
git apply functions-common.diff
rm functions-common.diff
```