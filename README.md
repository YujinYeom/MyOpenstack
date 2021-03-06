## Install Openstack ver.Queens

```
1. virtual box 
 - 2 cpu ,  4g ram , 10gb storage  * 2
 - network host - birdge (enable) / NAT
 //same controller node and compute node 

2. network interface
 
// controller node
 # vi /etc/interfaces
  auto enp0s3
  iface enp0s3 inet static
   address 10.0.0.11
   netmask 255.255.255.0
   gateway 10.0.0.1
   dns-nameserver 8.8.8.8

  auto enp0s8
  iface enp0s8 inet dhcp
 
// compute node
 # vi /etc/interfaces
  auto enp0s3
  iface enp0s3 inet static
   address 10.0.0.31
   netmask 255.255.255.0
   gateway 10.0.0.1
   dns-nameserver 8.8.8.8

  auto enp0s8
  iface enp0s8 inet dhcp

// controller node & compute node
 # vi /etc/hosts
  #127.0.1.1	controller
  10.0.0.11 	controller
  10.0.0.31	compute1

3. basic environment
 //controller node 
 # apt install chrony
 # service chrony restart
 # chronyc sources

 // compute node
 # apt install chrony
 # vi /etc/chrony/chrony.conf
 ...
 server controller iburst
 ...
 # service chrony restart
 # chronyc sources

 // controller node & compute node
 # apt install software-properties-common
 # add-apt-repository cloud-archive:queens 
 # apt update && apt dist-upgrade
 # apt install python-openstackclient
 
 // controller node
 # apt install mariadb-server python-pymysql
 # vi /etc/mysql/mariadb.conf.d/99-openstack.cnf

 [mysqld]
 bind-address = 10.0.0.11

 default-storage-engine = innodb
 innodb_file_per_table = on
 max_connections = 4096
 collation-server = utf8_general_ci
 character-set-server = utf8

 # service mysql restart
 # mysql_secure_installation
 # vi /root/mysql
 openstack
 
 # apt install rabbitmq-server -y
 # rabbitmqctl add_user openstack openstack
 # rabbitmqctl set_permissions openstack ".*" ".*" ".*"

 # apt install memcached python-memcache 
 # vi /etc/memcached.conf
 -l 10.0.0.11 //change 127.0.0.1
 # service memcached restart
 
 # apt install etcd -y
 # vi /etc/default/etcd
 ETCD_NAME="controller"
 ETCD_DATA_DIR="/var/lib/etcd"
 ETCD_INITIAL_CLUSTER_STATE="new"
 ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
 ETCD_INITIAL_CLUSTER="controller=http://10.0.0.11:2380"
 ETCD_INITIAL_ADVERTISE_PEER_URLS="http://10.0.0.11:2380"
 ETCD_ADVERTISE_CLIENT_URLS="http://10.0.0.11:2379"
 ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
 ETCD_LISTEN_CLIENT_URLS="http://10.0.0.11:2379"
  
 # systemctl enable etcd
 # systemctl start etcd

4. Install keystone

 // controller node

 # mysql
 MariaDB [(none)]> CREATE DATABASE keystone;
 MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY 'openstack';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
IDENTIFIED BY 'openstack';
 quit

 # apt install keystone  apache2 libapache2-mod-wsgi -y

# cp -a /etc/keystone/keystone.conf /etc/keystone/keystone.conf_org
# cat /etc/keystone/keystone.conf_org | grep -v ^$ | grep -v ^# > /etc/keystone/keystone.conf
 # vi /etc/keystone/keystone.conf
 [database]
 # ...
 connection = mysql+pymysql://keystone:openstack@controller/keystone
 [token]
 # ...
 provider = fernet
 
 # su -s /bin/sh -c "keystone-manage db_sync" keystone
 # keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
 # keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
 # keystone-manage bootstrap --bootstrap-password openstack \
  --bootstrap-admin-url http://controller:5000/v3/ \
  --bootstrap-internal-url http://controller:5000/v3/ \
  --bootstrap-public-url http://controller:5000/v3/ \
  --bootstrap-region-id RegionOne

 # vi /etc/apache2/apache2.conf
 ...
 ServerName controller
 ...

 # service apache2 restart
  export OS_USERNAME=admin 
  export OS_PASSWORD=openstack 
  export OS_PROJECT_NAME=admin 
  export OS_USER_DOMAIN_NAME=Default 
  export OS_PROJECT_DOMAIN_NAME=Default 
  export OS_AUTH_URL=http://controller:5000/v3 
  export OS_IDENTITY_API_VERSION=3 

  openstack domain create --description "An Example Domain" example

  openstack project create --domain default \
  --description "Service Project" service

  openstack project create --domain default \
  --description "Demo Project" demo

  openstack user create --domain default \
  --password-prompt demo

 openstack
 openstack // password 

  openstack role create user
  openstack role add --project demo --user demo user

  unset OS_AUTH_URL OS_PASSWORD
  openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name admin --os-username admin token issue

  openstack --os-auth-url http://controller:5000/v3 \
  --os-project-domain-name Default --os-user-domain-name Default \
  --os-project-name demo --os-username demo token issue

# vi /root/admin-openrc
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=openstack
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2 
 
# vi /root/demo-openrc
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=DEMO_PASS
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2 

$ . admin-openrc
$ openstack token issue

5. Install glance

// controller node

$ mysql -u root -p

MariaDB [(none)]> CREATE DATABASE glance;

MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
  IDENTIFIED BY 'openstack';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
  IDENTIFIED BY 'openstack';
quit

$ . admin-openrc

$ openstack user create --domain default --password-prompt glance
$ openstack role add --project service --user glance admin
$ openstack service create --name glance \
  --description "OpenStack Image" image

$ openstack endpoint create --region RegionOne \
  image public http://controller:9292

$ openstack endpoint create --region RegionOne \
  image internal http://controller:9292

$ openstack endpoint create --region RegionOne \
  image admin http://controller:9292

# apt install glance

# cp -a /etc/glance/glance-api.conf  /etc/glance/glance-api.conf_org
# cat /etc/glance/glance-api.conf_org | grep  -v ^$ | grep -v ^# > /etc/glance/glance-api.conf
# vi /etc/glance/glance-api.conf
[database]
# ...
connection = mysql+pymysql://glance:openstack@controller/glance
[keystone_authtoken]
# ...
auth_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = openstack

[paste_deploy]
# ...
flavor = keystone

[glance_store]
# ...
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/

# cp -a /etc/glance/glance-registry.conf /etc/glance/glance-registry.conf_org
# cat /etc/glance/glance-registry.conf_org | grep -v ^$ | grep -v ^# > /etc/glance/glance-registry.conf
# vi /etc/glance/glance-registry.conf
[database]
# ...
connection = mysql+pymysql://glance:openstack@controller/glance

[keystone_authtoken]
# ...
auth_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = openstack

[paste_deploy]
# ...
flavor = keystone

# su -s /bin/sh -c "glance-manage db_sync" glance

# service glance-registry restart
# service glance-api restart

$ . admin-openrc
$ wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
$ openstack image create "cirros" \
  --file cirros-0.4.0-x86_64-disk.img \
  --disk-format qcow2 --container-format bare \
  --public
$ openstack image list

6. Install nova

// controller node 

# mysql

MariaDB [(none)]> CREATE DATABASE nova_api;
MariaDB [(none)]> CREATE DATABASE nova;
MariaDB [(none)]> CREATE DATABASE nova_cell0;

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'openstack';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' \
  IDENTIFIED BY 'openstack';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'openstack';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
  IDENTIFIED BY 'openstack';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' \
  IDENTIFIED BY 'openstack';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' \
  IDENTIFIED BY 'openstack';
quit

$ . admin-openrc
$ openstack user create --domain default --password-prompt nova
password : openstack

$ openstack role add --project service --user nova admin

$ openstack service create --name nova \
  --description "OpenStack Compute" compute

$ openstack endpoint create --region RegionOne \
  compute public http://controller:8774/v2.1
$ openstack endpoint create --region RegionOne \
  compute internal http://controller:8774/v2.1
$ openstack endpoint create --region RegionOne \
  compute admin http://controller:8774/v2.1

$ openstack user create --domain default --password-prompt placement
password : openstack

$ openstack role add --project service --user placement admin
$ openstack service create --name placement --description "Placement API" placement

$ openstack endpoint create --region RegionOne placement public http://controller:8778
$ openstack endpoint create --region RegionOne placement internal http://controller:8778 
$ openstack endpoint create --region RegionOne placement admin http://controller:8778 

# apt install nova-api nova-conductor nova-consoleauth \
  nova-novncproxy nova-scheduler nova-placement-api

# cp -a /etc/nova/nova.conf /etc/nova/nova.conf_org
# cat /etc/nova/nova.conf_org | grep -v ^$ | grep -v ^# > /etc/nova/nova.conf
# vi /etc/nova/nova.conf
[api_database]
# ...
connection = mysql+pymysql://nova:openstack@controller/nova_api

[database]
# ...
connection = mysql+pymysql://nova:openstack@controller/nova

[DEFAULT]
//remove the log_dir
# ...
transport_url = rabbit://openstack:openstack@controller
my_ip = 10.0.0.11
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[vnc]

# ...
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip

[api]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = openstack

[glance]
# ...
api_servers = http://controller:9292

[oslo_concurrency]
# ...
lock_path = /var/lib/nova/tmp

[placement]
# ...
os_region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = openstack

# su -s /bin/sh -c "nova-manage api_db sync" nova
# su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova
# su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova
# su -s /bin/sh -c "nova-manage db sync" nova

# nova-manage cell_v2 list_cells

# service nova-api restart
# service nova-consoleauth restart
# service nova-scheduler restart
# service nova-conductor restart
# service nova-novncproxy restart

// compute node

# apt install nova-compute -y 

# cp -a /etc/nova/nova.conf /etc/nova/nova.conf_org
# cat /etc/nova/nova.conf_org | grep -v ^$ | grep -v ^# > /etc/nova/nova.conf
# vi /etc/nova/nova.conf
[DEFAULT]
# ...
transport_url = rabbit://openstack:openstack@controller
my_ip = 10.0.0.21
use_neutron = True
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[api]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = nova
password = openstack

[vnc]
# ...
enabled = True
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html

[glance]
# ...
api_servers = http://controller:9292

[oslo_concurrency]
# ...
lock_path = /var/lib/nova/tmp

[placement]
# ...
os_region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = openstack

# vi /etc/nova/nova-compute.conf
[libvirt]
# ...
virt_type = qemu

# service nova-compute restart

//controller node

$ . admin-openrc
$ openstack compute service list --service nova-compute
# su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova

$ openstack compute service list
$ openstack catalog list
$ openstack image list
# nova-status upgrade check


7. Install neutron

// controller node

$ mysql -u root -p
MariaDB [(none)] CREATE DATABASE neutron;
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost' \
  IDENTIFIED BY 'openstack';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' \
  IDENTIFIED BY 'openstack';
quit

$ . admin-openrc
$ openstack user create --domain default --password-prompt neutron
password : openstack

$ openstack role add --project service --user neutron admin
$ openstack service create --name neutron \
  --description "OpenStack Networking" network
$ openstack endpoint create --region RegionOne \
  network public http://controller:9696
$ openstack endpoint create --region RegionOne \
  network internal http://controller:9696
$ openstack endpoint create --region RegionOne \
  network admin http://controller:9696

# apt install neutron-server neutron-plugin-ml2 \
  neutron-linuxbridge-agent neutron-l3-agent neutron-dhcp-agent \
  neutron-metadata-agent

# cp -a /etc/neutron/neutron.conf /etc/neutron/neutron.conf_org
# cat /etc/neutron/neutron.conf_org | grep -v ^$ | grep -v ^# > /etc/neutron/neutron.conf

# vi /etc/neutron/neutron.conf

[database]
# ...
connection = mysql+pymysql://neutron:openstack@controller/neutron

[DEFAULT]
# ...
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true

transport_url = rabbit://openstack:openstack@controller
auth_strategy = keystone

notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[keystone_authtoken]
# ...
auth_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = openstack

[nova]
# ...
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = openstack

# cp -a /etc/neutron/plugins/ml2/ml2_conf.ini /etc/neutron/plugins/ml2/ml2_conf.ini_org
# cat /etc/neutron/plugins/ml2/ml2_conf.ini_org | grep -v ^$ | grep -v ^# > /etc/neutron/plugins/ml2/ml2_conf.ini
# vi /etc/neutron/plugins/ml2/ml2_conf.ini

[ml2]
# ...
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = linuxbridge,l2population
extension_drivers = port_security

[ml2_type_flat]
# ...
flat_networks = provider

[ml2_type_vxlan]
# ...
vni_ranges = 1:1000

[securitygroup]
# ...
enable_ipset = true

# cp -a /etc/neutron/plugins/ml2/linuxbridge_agent.ini /etc/neutron/plugins/ml2/linuxbridge_agent.ini_org
# cat /etc/neutron/plugins/ml2/linuxbridge_agent.ini_org | grep -v ^$ | grep -v ^# > /etc/neutron/plugins/ml2/linuxbridge_agent.ini
# vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini

[linux_bridge]
physical_interface_mappings = provider:enp0s3

[vxlan]
enable_vxlan = true
local_ip = 10.0.0.11
l2_population = true

[securitygroup]
# ...
enable_security_group = true
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

# cp -a /etc/neutron/l3_agent.ini /etc/neutron/l3_agent.ini_org
# cat /etc/neutron/l3_agent.ini_org | grep -v ^$ | grep -v ^# > /etc/neutron/l3_agent.ini
# vi /etc/neutron/l3_agent.ini

[DEFAULT]
# ...
interface_driver = linuxbridge

# cp -a /etc/neutron/dhcp_agent.ini /etc/neutron/dhcp_agent.ini_org
# cat /etc/neutron/dhcp_agent.ini_org | grep -v ^$ | grep -v ^# > /etc/neutron/dhcp_agent.ini
# vi /etc/neutron/dhcp_agent.ini

[DEFAULT]
# ...
interface_driver = linuxbridge
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true

# cp -a /etc/neutron/metadata_agent.ini /etc/neutron/metadata_agent.ini_org
# cat /etc/neutron/metadata_agent.ini_org | grep -v ^$ | grep -v ^# > /etc/neutron/metadata_agent.ini

# vi /etc/neutron/metadata_agent.ini

[DEFAULT]
# ...
nova_metadata_host = controller
metadata_proxy_shared_secret = openstack

# vi /etc/nova/nova.conf

//add 

[neutron]
# ...
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = openstack
service_metadata_proxy = true
metadata_proxy_shared_secret = openstack

# su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf \
  --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron
# service nova-api restart
# service neutron-server restart
# service neutron-linuxbridge-agent restart
# service neutron-dhcp-agent restart
# service neutron-metadata-agent restart
# service neutron-l3-agent restart

// compute node

# apt install neutron-linuxbridge-agent

# cp -a /etc/neutron/neutron.conf /etc/neutron/neutron.conf_org
# cat /etc/neutron/neutron.conf_org | grep -v ^$ | grep -v ^# > /etc/neutron/neutron.conf
# vi /etc/neutron/neutron.conf

[DEFAULT]
# ...
transport_url = rabbit://openstack:openstack@controller
auth_strategy = keystone

[keystone_authtoken]
# ...
auth_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = openstack

# cp -a  /etc/neutron/plugins/ml2/linuxbridge_agent.ini   /etc/neutron/plugins/ml2/linuxbridge_agent.ini_org
# cat  /etc/neutron/plugins/ml2/linuxbridge_agent.ini_org | grep -v ^$ | grep -v ^# >  /etc/neutron/plugins/ml2/linuxbridge_agent.ini
# vi /etc/neutron/plugins/ml2/linuxbridge_agent.ini

[linux_bridge]
physical_interface_mappings = provider:enp0s3

[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver

[vxlan]
enable_vxlan = true
local_ip = 10.0.0.31
l2_population = true

vi /etc/nova/nova.conf

[neutron]
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = Default
user_domain_name = Default
region_name = RegionOne
project_name = service
username = neutron
password = openstack

# service nova-compute restart
# service neutron-linuxbridge-agent restart

8. Install horizon(dashboard)

// controller node

# apt install openstack-dashboard

# vi /etc/openstack-dashboard/local_settings.py
:set number

OPENSTACK_HOST = "controller"

ALLOWED_HOSTS = ['*', ]

SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

CACHES = {
    'default': {
         'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
         'LOCATION': 'controller:11211',
    }
}

OPENSTACK_KEYSTONE_URL = "http://%s:5000/v3" % OPENSTACK_HOST

OPENSTACK_KEYSTONE_MULTIDOMAIN_SUPPORT = True

OPENSTACK_API_VERSIONS = {
    "identity": 3,
    "image": 2,
    "volume": 2,
}

OPENSTACK_KEYSTONE_DEFAULT_DOMAIN = "Default"

OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

TIME_ZONE = "Asia/Seoul"

# vi /etc/apache2/conf-available/openstack-dashboard.conf

WSGIApplicationGroup %{GLOBAL} //Add this line if not included 

# service apache2 reload

http://10.0.0.11/horizon

Domain: default
ID: admin
pw: openstack

```
