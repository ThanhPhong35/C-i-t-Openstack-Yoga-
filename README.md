# C-i-t-Openstack-Yoga-


    nano /etc/network/interfaces{
        auto ens33
        iface ens33 inet manual
        up ip link set dev $IFACE up
        down ip link set dev $IFACE down
    }
    nano /etc/hosts{
        # controller
        10.0.0.18 controller
        # compute1
        10.0.0.23 compute1
        # block1
        10.0.0.41 block1
        # object1
        10.0.0.51 object1
        # object2
        10.0.0.52 object2
    }

Network Time Protocol (NTP)
apt -y install chrony
nano /etc/chrony/chrony.conf{
#pool ntp.ubuntu.com        iburst maxsources 4
#pool 0.ubuntu.pool.ntp.org iburst maxsources 1
#pool 1.ubuntu.pool.ntp.org iburst maxsources 1
#pool 2.ubuntu.pool.ntp.org iburst maxsources 2
pool ntp.nict.jp iburst 
# add to the end : add network range you allow to receive time syncing requests from clients
allow 10.0.0.0/24
}
systemctl restart chrony
chronyc sources

Cài MariaDB
apt install mariadb-server python3-pymysql
nano /etc/mysql/mariadb.conf.d/99-openstack.cnf{
[mysqld]
bind-address = 10.0.0.18
default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
}
service mysql restart
mysql_secure_installation

Cài đặt Openstack Repository (Yoga)
apt -y install software-properties-common
add-apt-repository cloud-archive:yoga
apt update
apt -y upgrade

Cài RabbitMQ
apt install rabbitmq-server
systemctl enable rabbitmq-server.service 
systemctl start rabbitmq-server.service
rabbitmqctl add_user openstack password
rabbitmqctl set_permissions openstack ".*" ".*" ".*"

Cài đặt Memcached
apt install memcached
nano /etc/memcached.conf{
…….
-l 10.0.0.18
}
service memcached restart

Cài đặt keystone
mysql
create database keystone; 
grant all privileges on keystone.* to keystone@'localhost' identified by 'password';
grant all privileges on keystone.* to keystone@'%' identified by 'password'; 
exit

apt install keystone python3-openstackclient apache2 libapache2-mod-wsgi-py3 python3-oauth2client -y

vi /etc/keystone/keystone.conf{
[database]
# ...
connection = mysql+pymysql://keystone:password@controller/keystone
[token]
# ...
provider = fernet
}
su -s /bin/sh -c "keystone-manage db_sync" keystone
keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
keystone-manage bootstrap --bootstrap-password adminpassword --bootstrap-admin-url http://controller:5000/v3/ --bootstrap-internal-url http://controller:5000/v3/ --bootstrap-public-url http://controller:5000/v3/ --bootstrap-region-id RegionOne

vi /etc/apache2/apache2.conf{
ServerName controller
}
service apache2 restart
export OS_USERNAME=admin
export OS_PASSWORD=adminpassword
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3

nano admin-openrc{
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=adminpassword
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
}
. admin-openrc
openstack domain create --description "An Example Domain" example
openstack project create --domain default --description "Service Project" service
openstack project create --domain default --description "Demo Project" myproject
openstack user create --domain default --password-prompt myuser //123456
openstack role create myrole
openstack role add --project myproject --user myuser myrole

unset OS_AUTH_URL OS_PASSWORD
openstack --os-auth-url http://controller:5000/v3 --os-project-domain-name Default --os-user-domain-name Default --os-project-name admin --os-username admin token issue
pss adminpassword

nano demo-openrc{
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=myproject
export OS_USERNAME=myuser
export OS_PASSWORD=123456
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
}
. demo-openrc

mysql
CREATE DATABASE glance;
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'password';
exit
. admin-openrc
openstack user create --domain default --password-prompt glance //123456
openstack role add --project service --user glance admin
openstack service create --name glance --description "OpenStack Image" image
openstack endpoint create --region RegionOne  image public http://controller:9292
openstack endpoint create --region RegionOne image internal http://controller:9292
openstack endpoint create --region RegionOne  image admin http://controller:9292

apt install glance
vi /etc/glance/glance-api.conf
{
[database]
...
connection=mysql+pymysql://glance:password@controller/glance
[keystone_authtoken]
...
www_authenticate_uri =http://controller:5000
auth_url=http://controller:5000
memcached_servers=controller:11211
auth_type=password
project_domain_name=Default
user_domain_name=Default
project_name=service
username=glance
password=123456
[paste_deploy]
...
flavor=keystone
[glance_store]
...
stores=file,http
default_store=file
filesystem_store_datadir=/var/lib/glance/images/
[DEFAULT]
use_keystone_quotas = True
}

su -s /bin/sh -c "glance-manage db_sync" glance
service glance-api restart
. admin-openrc
glance image-create --name "cirros" --file cirros-0.4.0-x86_64-disk.img --disk-format qcow2 --container-format bare --visibility=public
glance image-list

Cài đặt Placement
mysql
CREATE DATABASE placement;
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'password';
exit
openstack user create --domain default --password-prompt placement //123456
openstack role add --project service --user placement admin
openstack service create --name placement --description "Placement API" placement
openstack endpoint create --region RegionOne placement public http://controller:8778
openstack endpoint create --region RegionOne placement internal http://controller:8778
openstack endpoint create --region RegionOne placement admin http://controller:8778
apt install placement-api
vi  /etc/placement/placement.conf{
[placement_database]
# ...
connection = mysql+pymysql://placement:password@controller/placement
[api]
# ...
auth_strategy = keystone
[keystone_authtoken]
# ...
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = placement
password = 123456
}
. admin-openrc
placement-status upgrade check
pip3 install osc-placement
openstack --os-placement-api-version 1.2 resource class list --sort-column name



Cài đặt Nova
mysql
CREATE DATABASE nova_api;
CREATE DATABASE nova;
CREATE DATABASE nova_cell0;

GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'password';

openstack user create --domain default --password-prompt nova
openstack role add --project service --user nova admin
openstack service create --name nova --description "OpenStack Compute" compute
openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1
openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1
apt install nova-api nova-conductor nova-novncproxy nova-scheduler
vi  /etc/nova/nova.conf{
[api_database]
# ...
connection = mysql+pymysql://nova:password@controller/nova_api

[database]
# ...
connection = mysql+pymysql://nova:password@controller/nova
[DEFAULT]
# ...
transport_url = rabbit://openstack:password@controller:5672/

[api]
# ...
auth_strategy = keystone

[keystone_authtoken]
# ...
www_authenticate_uri = http://controller:5000/
auth_url = http://controller:5000/
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = 123456
[service_user]
send_service_user_token = true
auth_url = https://controller/identity
auth_strategy = keystone
auth_type = password
project_domain_name = Default
project_name = service
user_domain_name = Default
username = nova
password = NOVA_PASS
[DEFAULT]
# ...
my_ip = 10.0.0.11
[vnc]
enabled = true
# ...
server_listen = $my_ip
server_proxyclient_address = $my_ip
[glance]
# ...
api_servers = http://controller:9292
[oslo_concurrency]
# ...
lock_path = /var/lib/nova/tmp
}


