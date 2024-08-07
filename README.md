# Cấu hình card mạng
![image](https://github.com/user-attachments/assets/743ac3a0-377b-4210-b59f-9e152a637eac)
# Cấu hình máy 
![image](https://github.com/user-attachments/assets/284054f0-8230-4b2c-8e17-f305d9b7cac5)
# Sửa file /etc/network/interfaces
    nano /etc/network/interfaces{
        auto ens33
        iface ens33 inet manual
        up ip link set dev $IFACE up
        down ip link set dev $IFACE down
    }

# Sửa file /etc/hosts
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
    reboot
# Cài đặt Network Time Protocol (NTP)
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
# Cài MariaDB
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
# Cài đặt Openstack Repository (Yoga)
    apt -y install software-properties-common
    add-apt-repository cloud-archive:yoga
    apt update
    apt -y upgrade
# Cài RabbitMQ
    apt install rabbitmq-server
    systemctl enable rabbitmq-server.service 
    systemctl start rabbitmq-server.service
    rabbitmqctl add_user openstack password
    rabbitmqctl set_permissions openstack ".*" ".*" ".*"
# Cài đặt Memcached
    apt install memcached
    nano /etc/memcached.conf{
        …….
        -l 10.0.0.18
    }
    service memcached restart
