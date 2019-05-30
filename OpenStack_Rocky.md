

[TOC]

# 部署环境

* 控制节点：172.20.10.120	controller
* 计算节点：172.20.10.121	compute01
* 计算节点：172.20.10.122	compute02


* 系统：ubuntu-18.04.2
* CPU：4核
* 内存：32G
* 硬盘：100G

# 准备工作【所有节点】

## 安装ubuntu-18.04.2

## 配置root用户密码
> sudo passwd root

## 配置网卡
> vim /etc/netplan/50-cloud-init.yaml
    
```
network:
    ethernets:
        ens160:
            addresses:
              - 172.20.10.120/16
            gateway4: 172.20.0.1
            nameservers:
                addresses: [114.114.114.114, 8.8.8.8]
        ens192:
            addresses:
              - 172.16.10.120/24
            nameservers: {}
        ens224:
            dhcp4: false
    version: 2
```

## 配置ssh的root登陆权限
> vim /etc/ssh/sshd_config

```
    # PermitRootLogin prohibit-password
    PermitRootLogin yes
```

## 重启ssh服务
> service ssh restart


## 修改时区
> tzselect

> sudo ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime


## 替换阿里安装源
> mv /etc/apt/sources.list /etc/apt/sources.list.bak
vim /etc/apt/sources.list

```
deb http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-backports main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ bionic-proposed main restricted universe multiverse
```

# 客户端安装【所有节点】

## 添加rocky安装源
> add-apt-repository cloud-archive:rocky


## 更新安装源列表及更新软件包
> apt update && apt dist-upgrade


## 安装openstack客户端
> apt install python-openstackclient
 
# 基础服务安装【控制节点】：

## 安装mysql
> apt install mariadb-server python-pymysql


## 配置mysql监听地址
> vim /etc/mysql/mariadb.conf.d/99-openstack.cnf

```
[mysqld]
bind-address = 172.20.10.120

default-storage-engine = innodb
innodb_file_per_table = on
max_connections = 4096
collation-server = utf8_general_ci
character-set-server = utf8
```

## 重启mysql服务
> service mysql restart


## 配置root密码
> mysql_secure_installation


## 安装Rabbitmq：
> apt install rabbitmq-server


## 添加openstack用户和密码
> rabbitmqctl add_user openstack openstack


## 设置openstack权限
> rabbitmqctl set_permissions openstack ".*" ".*" ".*"


## 安装Memcache：
> apt install memcached python-memcache


## 配置监听地址
> vim /etc/memcached.conf

```
# -l 127.0.0.1
-l 172.20.10.120
```

## 重启Memcache服务
> service memcached restart


## 安装etcd
> apt install etcd


## 配置etcd
> vim /etc/default/etcd

```
ETCD_NAME="controller"
ETCD_DATA_DIR="/var/lib/etcd"
ETCD_INITIAL_CLUSTER_STATE="new"
ETCD_INITIAL_CLUSTER_TOKEN="etcd-cluster-01"
ETCD_INITIAL_CLUSTER="controller=http://172.20.10.120:2380"
ETCD_INITIAL_ADVERTISE_PEER_URLS="http://172.20.10.120:2380"
ETCD_ADVERTISE_CLIENT_URLS="http://172.20.10.120:2379"
ETCD_LISTEN_PEER_URLS="http://0.0.0.0:2380"
ETCD_LISTEN_CLIENT_URLS="http://172.20.10.120:2379"
```

## 使能并启动etcd
> systemctl enable etcd

> systemctl start etcd


--------------------------------------------------------------------------------
   
# Keystone安装【控制节点】

## 添加Keystone数据库
> mysql -u root -p

```
Enter password: root
```

> MariaDB [(none)]> CREATE DATABASE keystone;

> MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'controller' IDENTIFIED BY 'keystone';

> MariaDB [(none)]> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'keystone';


## 安装keystone和apache2
> apt install keystone apache2 libapache2-mod-wsgi


## 配置keystone
> crudini --set /etc/keystone/keystone.conf database connection mysql+pymysql://keystone:keystone@controller/keystone

> crudini --set /etc/keystone/keystone.conf token provider fernet


> cat /etc/keystone/keystone.conf | grep ^\[\\[a-z]

```
[DEFAULT]
log_dir = /var/log/keystone
[application_credential]
[assignment]
[auth]
[cache]
[catalog]
[cors]
[credential]
[database]
connection = mysql+pymysql://keystone:keystone@controller/keystone
[domain_config]
[endpoint_filter]
[endpoint_policy]
[eventlet_server]
[extra_headers]
[federation]
[fernet_tokens]
[healthcheck]
[identity]
[identity_mapping]
[ldap]
[matchmaker_redis]
[memcache]
[oauth1]
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[policy]
[profiler]
[resource]
[revoke]
[role]
[saml]
[security_compliance]
[shadow_users]
[signing]
[token]
provider = fernet
[tokenless_auth]
[trust]
[unified_limit]
[wsgi]
```

## 同步keystone数据库
> su -s /bin/sh -c "keystone-manage db_sync" keystone

> keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone

> keystone-manage credential_setup --keystone-user keystone --keystone-group keystone

> keystone-manage bootstrap --bootstrap-password admin --bootstrap-admin-url http://controller:5000/v3/ --bootstrap-internal-url http://controller:5000/v3/ --bootstrap-public-url http://controller:5000/v3/ --bootstrap-region-id RegionOne

## 配置apache
> vim /etc/apache2/apache2.conf

```
ServerName controller
```

## 重启apache2服务
> service apache2 restart


## 配置临时管理员账户
> export OS_USERNAME=admin

> export OS_PASSWORD=admin

> export OS_PROJECT_NAME=admin

> export OS_USER_DOMAIN_NAME=Default

> export OS_PROJECT_DOMAIN_NAME=Default

> export OS_AUTH_URL=http://controller:5000/v3

> export OS_IDENTITY_API_VERSION=3


## 创建service项目
> openstack project create --domain default --description "Service Project" service

```
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | 4ccc2ff33aa64602a1ed22bf38cfae3b |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+
```

## 创建demo项目
> openstack project create --domain default --description "Demo Project" demo

```
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Demo Project                     |
| domain_id   | default                          |
| enabled     | True                             |
| id          | aef88d500d1c456a96f5185e31467cad |
| is_domain   | False                            |
| name        | demo                             |
| parent_id   | default                          |
| tags        | []                               |
+-------------+----------------------------------+
```

## 创建demo用户
> openstack user create --domain default --password-prompt demo

```
User Password: demo
Repeat User Password: demo
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 3426ef835ffa494cba573f6c5e404379 |
| name                | demo                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

## 创建user角色
> openstack role create user

```
+-----------+----------------------------------+
| Field     | Value                            |
+-----------+----------------------------------+
| domain_id | None                             |
| id        | 105b85fd2d2a48618987d86ae921df1f |
| name      | user                             |
+-----------+----------------------------------+
```

## 将demo用户添加到user角色
> openstack role add --project demo --user demo user


## 删除OS_AUTH_URL OS_PASSWORD变量
> unset OS_AUTH_URL OS_PASSWORD


## 确认操作，请求admin认证令牌
> openstack --os-auth-url http://controller:5000/v3 --os-project-domain-name Default --os-user-domain-name Default --os-project-name admin --os-username admin token issue

```
Password: admin

+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2019-05-19T08:46:38+0000                                                                                                                                                                |
| id         | gAAAAABc4QneH4P5pjtrZcej_vEzHKo1J1h9WZYz2Zx0skfd70EGwSKhrnmVm9h0LY-rlJau6Br11nv1P1G4lxpavY_5ear5hQRuvFKDveN7o_xr6vQ1mw8FNfqxc0g9fR69b1shd5YIEJWg-IerhFh1y4OanBmtESkOv3B_mT-5D-g-eNRp1kU |
| project_id | fe13643127904142b74c0bfa2ea34794                                                                                                                                                        |
| user_id    | 28022c0955b04ffb884a90ef97142419                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

## 确认操作，请求demo认证令牌
> openstack --os-auth-url http://controller:5000/v3 --os-project-domain-name Default --os-user-domain-name Default --os-project-name demo --os-username demo token issue

```
Password: demo

+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2019-05-19T08:46:51+0000                                                                                                                                                                |
| id         | gAAAAABc4Qnr3GQ5c4adkB8Wd0USlbNz0qpN7FxmN5ijbpV1PB7cRMdWyWREFsvJjqR53pYjJRTtMCSdJUNRQlXKr-zUXyeW9idHudE9sE6-GTipvlz-g73ALSHyKbX9c_VDNpV63HDj6sqNIPWTaMwuCn3Hh7Ze-3wM3Tw4Zuu8yUSJaMEe-yo |
| project_id | aef88d500d1c456a96f5185e31467cad                                                                                                                                                        |
| user_id    | 3426ef835ffa494cba573f6c5e404379                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

## 创建admin客户端环境脚本
> vim admin-openrc

```
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=admin
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

## 创建demo客户端环境脚本
> vim demo-openrc

```
export OS_PROJECT_DOMAIN_NAME=Default
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=demo
export OS_AUTH_URL=http://controller:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

## 使用脚本导入admin环境变量
> . admin-openrc


## 请求admin认证令牌
> openstack token issue

```
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                                                                                                                   |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| expires    | 2019-05-19T09:22:17+0000                                                                                                                                                                |
| id         | gAAAAABc4RI59XqFoe_1N2vbRTjwclC_hXI2jLlgLcRS4UILhInGYzj_TtHQInb_5_BsDsXrvWIbrtKnpNOxdc0X1wWeLukynh2AE4hvqrh4Uucaib9GIrDyQb1x9ObFEUEnkEuovlE9Z3HglCBEloCQNOutkjtG3gZomY2ITRR3KN1SEJxqj5w |
| project_id | fe13643127904142b74c0bfa2ea34794                                                                                                                                                        |
| user_id    | 28022c0955b04ffb884a90ef97142419                                                                                                                                                        |
+------------+-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+


--------------------------------------------------------------------------------
```

# Glance安装【控制节点】

> mysql -u root -p

```
Enter password: root
```

> MariaDB [(none)]> CREATE DATABASE glance;

> MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'controller' IDENTIFIED BY 'glance';

> MariaDB [(none)]> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' IDENTIFIED BY 'glance';


> . admin-openrc


> openstack user create --domain default --password-prompt glance

```
User Password: glance
Repeat User Password: glance 
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 6bf6fc21e0bf471f9b14781e1ad50dae |
| name                | glance                           |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

> openstack role add --project service --user glance admin


> openstack service create --name glance --description "OpenStack Image" image

```
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Image                  |
| enabled     | True                             |
| id          | f176166b33fc48a490005d2ad8822ca9 |
| name        | glance                           |
| type        | image                            |
+-------------+----------------------------------+
```

> openstack endpoint create --region RegionOne image public http://controller:9292

```
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | e13b038cb1154dd5aef3755b9b8d85ca |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | f176166b33fc48a490005d2ad8822ca9 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+
```

> openstack endpoint create --region RegionOne image internal http://controller:9292

```
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 33d8d7d0a30b4d128da1210912079ad2 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | f176166b33fc48a490005d2ad8822ca9 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+
```

> openstack endpoint create --region RegionOne image admin http://controller:9292

```
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | c7ca167426bf40899c59202c86e46007 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | f176166b33fc48a490005d2ad8822ca9 |
| service_name | glance                           |
| service_type | image                            |
| url          | http://controller:9292           |
+--------------+----------------------------------+
```

> apt install glance


> crudini --set /etc/glance/glance-api.conf database connection mysql+pymysql://glance:glance@controller/glance

> crudini --set /etc/glance/glance-api.conf keystone_authtoken www_authenticate_uri http://controller:5000

> crudini --set /etc/glance/glance-api.conf keystone_authtoken auth_url http://controller:5000

> crudini --set /etc/glance/glance-api.conf keystone_authtoken memcached_servers controller:11211

> crudini --set /etc/glance/glance-api.conf keystone_authtoken auth_type password

> crudini --set /etc/glance/glance-api.conf keystone_authtoken project_domain_name Default

> crudini --set /etc/glance/glance-api.conf keystone_authtoken user_domain_name Default

> crudini --set /etc/glance/glance-api.conf keystone_authtoken project_name service

> crudini --set /etc/glance/glance-api.conf keystone_authtoken username glance

> crudini --set /etc/glance/glance-api.conf keystone_authtoken password glance

> crudini --set /etc/glance/glance-api.conf paste_deploy flavor keystone

> crudini --set /etc/glance/glance-api.conf glance_store stores file,http

> crudini --set /etc/glance/glance-api.conf glance_store default_store file

> crudini --set /etc/glance/glance-api.conf glance_store filesystem_store_datadir /var/lib/glance/images/


> cat /etc/glance/glance-api.conf | grep ^\[\\[a-z]

```
[DEFAULT]
[cors]
[database]
connection = mysql+pymysql://glance:glance@controller/glance
backend = sqlalchemy
[glance_store]
stores = file,http
default_store = file
filesystem_store_datadir = /var/lib/glance/images/
[image_format]
disk_formats = ami,ari,aki,vhd,vhdx,vmdk,raw,qcow2,vdi,iso,ploop.root-tar
[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = glance
[matchmaker_redis]
[oslo_concurrency]
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[paste_deploy]
flavor = keystone
[profiler]
[store_type_location_strategy]
[task]
[taskflow_executor]
```

> vim /etc/glance/glance-registry.conf

```
[database]
# connection = sqlite:////var/lib/glance/glance.sqlite
connection = mysql+pymysql://glance:glance@controller/glance

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = glance

[paste_deploy]
flavor = keystone
```

> cat /etc/glance/glance-registry.conf | grep ^\[\\[a-z]

```
[DEFAULT]
[database]
connection = mysql+pymysql://glance:glance@controller/glance
backend = sqlalchemy
[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = glance
password = glance
[matchmaker_redis]
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_policy]
[paste_deploy]
flavor = keystone
[profiler]
```

> su -s /bin/sh -c "glance-manage db_sync" glance

```
2019-05-19 17:04:36.657 30932 INFO alembic.runtime.migration [-] Context impl MySQLImpl.
2019-05-19 17:04:36.658 30932 INFO alembic.runtime.migration [-] Will assume non-transactional DDL.
2019-05-19 17:04:36.668 30932 INFO alembic.runtime.migration [-] Context impl MySQLImpl.
2019-05-19 17:04:36.668 30932 INFO alembic.runtime.migration [-] Will assume non-transactional DDL.
......
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
Database is synced successfully.
```

> service glance-registry restart

> service glance-api restart


> . admin-openrc


> wget http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img

> openstack image create "cirros" --file cirros-0.4.0-x86_64-disk.img --disk-format qcow2 --container-format bare --public

```
+------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| Field            | Value                                                                                                                                                                                      |
+------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
| checksum         | 443b7623e27ecf03dc9e01ee93f67afe                                                                                                                                                           |
| container_format | bare                                                                                                                                                                                       |
| created_at       | 2019-05-19T09:22:09Z                                                                                                                                                                       |
| disk_format      | qcow2                                                                                                                                                                                      |
| file             | /v2/images/072bf6a3-4d71-40c4-bf38-a0719c9e00ae/file                                                                                                                                       |
| id               | 072bf6a3-4d71-40c4-bf38-a0719c9e00ae                                                                                                                                                       |
| min_disk         | 0                                                                                                                                                                                          |
| min_ram          | 0                                                                                                                                                                                          |
| name             | cirros                                                                                                                                                                                     |
| owner            | fe13643127904142b74c0bfa2ea34794                                                                                                                                                           |
| properties       | os_hash_algo='sha512', os_hash_value='6513f21e44aa3da349f248188a44bc304a3653a04122d8fb4535423c8e1d14cd6a153f735bb0982e2161b5b5186106570c17a9e58b64dd39390617cd5a350f78', os_hidden='False' |
| protected        | False                                                                                                                                                                                      |
| schema           | /v2/schemas/image                                                                                                                                                                          |
| size             | 12716032                                                                                                                                                                                   |
| status           | active                                                                                                                                                                                     |
| tags             |                                                                                                                                                                                            |
| updated_at       | 2019-05-19T09:22:09Z                                                                                                                                                                       |
| virtual_size     | None                                                                                                                                                                                       |
| visibility       | public                                                                                                                                                                                     |
+------------------+--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------+
```

> openstack image list

```
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 072bf6a3-4d71-40c4-bf38-a0719c9e00ae | cirros | active |
+--------------------------------------+--------+--------+
```

# Nova安装【控制节点】

> mysql -u root -p

```
Enter password: root

MariaDB [(none)]> CREATE DATABASE nova_api;
MariaDB [(none)]> CREATE DATABASE nova;
MariaDB [(none)]> CREATE DATABASE nova_cell0;
MariaDB [(none)]> CREATE DATABASE placement;

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'controller' IDENTIFIED BY 'nova';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_api.* TO 'nova'@'%' IDENTIFIED BY 'nova';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'controller' IDENTIFIED BY 'nova';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' IDENTIFIED BY 'nova';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'controller' IDENTIFIED BY 'nova';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON nova_cell0.* TO 'nova'@'%' IDENTIFIED BY 'nova';

MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'controller' IDENTIFIED BY 'placement';
MariaDB [(none)]> GRANT ALL PRIVILEGES ON placement.* TO 'placement'@'%' IDENTIFIED BY 'placement';
```

> . admin-openrc


> openstack user create --domain default --password-prompt nova

```
User Password: nova
Repeat User Password: nova
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 3ab383c838f04f84a40664333ac19782 |
| name                | nova                             |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

> openstack role add --project service --user nova admin


> openstack service create --name nova --description "OpenStack Compute" compute

```
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Compute                |
| enabled     | True                             |
| id          | 73a8bf5a6a3247a6b50e2322367ed2a7 |
| name        | nova                             |
| type        | compute                          |
+-------------+----------------------------------+
```

> openstack endpoint create --region RegionOne compute public http://controller:8774/v2.1

```
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 5dd2eaa26d284c1ea913f87b346ae416 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 73a8bf5a6a3247a6b50e2322367ed2a7 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://controller:8774/v2.1      |
+--------------+----------------------------------+
```

> openstack endpoint create --region RegionOne compute internal http://controller:8774/v2.1

```
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 6b0355b7b4844901b2463b993e6e63e6 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 73a8bf5a6a3247a6b50e2322367ed2a7 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://controller:8774/v2.1      |
+--------------+----------------------------------+
```

> openstack endpoint create --region RegionOne compute admin http://controller:8774/v2.1

```
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | fc8fd13d0346451593291e14e2b11990 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | 73a8bf5a6a3247a6b50e2322367ed2a7 |
| service_name | nova                             |
| service_type | compute                          |
| url          | http://controller:8774/v2.1      |
+--------------+----------------------------------+
```

> openstack user create --domain default --password-prompt placement

```
User Password: placement
Repeat User Password: placement
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 0d518897a05a4c05842b3047ab414433 |
| name                | placement                        |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

> openstack role add --project service --user placement admin


> openstack service create --name placement --description "Placement API" placement

```
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Placement API                    |
| enabled     | True                             |
| id          | edc445060e9943d594361dcfa0326fce |
| name        | placement                        |
| type        | placement                        |
+-------------+----------------------------------+
```

> openstack endpoint create --region RegionOne placement public http://controller:8778

```
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | b1d5d0e228154cc5a0277683146eb0d3 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | edc445060e9943d594361dcfa0326fce |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+
```

> openstack endpoint create --region RegionOne placement internal http://controller:8778

```
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | fbc0688d5181444ab37eaf95cbb4ea45 |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | edc445060e9943d594361dcfa0326fce |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+
```

> openstack endpoint create --region RegionOne placement admin http://controller:8778

```
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 066b57a89f3f4308954a0c7e2e291af8 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | edc445060e9943d594361dcfa0326fce |
| service_name | placement                        |
| service_type | placement                        |
| url          | http://controller:8778           |
+--------------+----------------------------------+
```

> apt install nova-api nova-conductor nova-consoleauth nova-novncproxy nova-scheduler nova-placement-api

> vim /etc/nova/nova.conf

```
[api_database]
# connection = sqlite:////var/lib/nova/nova_api.sqlite
connection = mysql+pymysql://nova:nova@controller/nova_api

[database]
# connection = sqlite:////var/lib/nova/nova.sqlite
connection = mysql+pymysql://nova:nova@controller/nova

[placement_database]
connection = mysql+pymysql://placement:placement@controller/placement

[DEFAULT]
# log_dir = /var/log/nova
transport_url = rabbit://openstack:openstack@controller

[api]
auth_strategy = keystone

[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova

[DEFAULT]
my_ip = 172.20.10.120

[DEFAULT]
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[vnc]
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip

[glance]
api_servers = http://controller:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[placement]
# os_region_name = openstack
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = placement
```

> cat /etc/nova/nova.conf | grep ^\[\\[a-z]

```
[DEFAULT]
lock_path = /var/lock/nova
state_path = /var/lib/nova
transport_url = rabbit://openstack:openstack@controller
my_ip = 172.20.10.120
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver
[api]
auth_strategy = keystone
[api_database]
connection = mysql+pymysql://nova:nova@controller/nova_api
[barbican]
[cache]
[cells]
enable = False
[cinder]
[compute]
[conductor]
[console]
[consoleauth]
[cors]
[database]
connection = mysql+pymysql://nova:nova@controller/nova
[devices]
[ephemeral_storage_encryption]
[filter_scheduler]
[glance]
api_servers = http://controller:9292
[guestfs]
[healthcheck]
[hyperv]
[ironic]
[key_manager]
[keystone]
[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova
[libvirt]
[matchmaker_redis]
[metrics]
[mks]
[neutron]
[notifications]
[osapi_v21]
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[pci]
[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = placement
[placement_database]
connection = mysql+pymysql://placement:placement@controller/placement
[powervm]
[profiler]
[quota]
[rdp]
[remote_debug]
[scheduler]
[serial_console]
[service_user]
[spice]
[upgrade_levels]
[vault]
[vendordata_dynamic_auth]
[vmware]
[vnc]
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip
[workarounds]
[wsgi]
[xenserver]
[xvp]
[zvm]
```

> su -s /bin/sh -c "nova-manage api_db sync" nova

> su -s /bin/sh -c "nova-manage cell_v2 map_cell0" nova

> su -s /bin/sh -c "nova-manage cell_v2 create_cell --name=cell1 --verbose" nova

```
2019-05-19 18:23:30.341 24929 INFO migrate.versioning.api [-] 0 -> 1... 
2019-05-19 18:23:30.488 24929 INFO migrate.versioning.api [-] done
2019-05-19 18:23:30.488 24929 INFO migrate.versioning.api [-] 1 -> 2... 
2019-05-19 18:23:30.688 24929 INFO migrate.versioning.api [-] done
......
2019-05-19 18:23:48.718 24929 INFO migrate.versioning.api [-] 59 -> 60... 
2019-05-19 18:23:48.862 24929 INFO migrate.versioning.api [-] done
2019-05-19 18:23:48.862 24929 INFO migrate.versioning.api [-] 60 -> 61... 
2019-05-19 18:23:49.034 24929 INFO migrate.versioning.api [-] done
```

> su -s /bin/sh -c "nova-manage db sync" nova

```
2019-05-19 18:24:39.924 25436 INFO migrate.versioning.api [req-56e8c9c0-8e62-404b-bb97-88f2fb502f08 - - - - -] 215 -> 216... 
2019-05-19 18:24:58.258 25436 INFO migrate.versioning.api [req-56e8c9c0-8e62-404b-bb97-88f2fb502f08 - - - - -] done
2019-05-19 18:24:58.259 25436 INFO migrate.versioning.api [req-56e8c9c0-8e62-404b-bb97-88f2fb502f08 - - - - -] 216 -> 217... 
2019-05-19 18:24:58.269 25436 INFO migrate.versioning.api [req-56e8c9c0-8e62-404b-bb97-88f2fb502f08 - - - - -] done
......
2019-05-19 18:26:03.444 25436 INFO migrate.versioning.api [req-56e8c9c0-8e62-404b-bb97-88f2fb502f08 - - - - -] 388 -> 389... 
2019-05-19 18:26:03.513 25436 INFO migrate.versioning.api [req-56e8c9c0-8e62-404b-bb97-88f2fb502f08 - - - - -] done
2019-05-19 18:26:03.514 25436 INFO migrate.versioning.api [req-56e8c9c0-8e62-404b-bb97-88f2fb502f08 - - - - -] 389 -> 390... 
2019-05-19 18:26:03.770 25436 INFO migrate.versioning.api [req-56e8c9c0-8e62-404b-bb97-88f2fb502f08 - - - - -] done
```

> su -s /bin/sh -c "nova-manage cell_v2 list_cells" nova

```
+-------+--------------------------------------+------------------------------------+-------------------------------------------------+----------+
|  Name |                 UUID                 |           Transport URL            |               Database Connection               | Disabled |
+-------+--------------------------------------+------------------------------------+-------------------------------------------------+----------+
| cell0 | 00000000-0000-0000-0000-000000000000 |               none:/               | mysql+pymysql://nova:****@controller/nova_cell0 |  False   |
| cell1 | 460f9ad2-6b89-4467-b1a0-fcf44d1553fe | rabbit://openstack:****@controller |    mysql+pymysql://nova:****@controller/nova    |  False   |
+-------+--------------------------------------+------------------------------------+-------------------------------------------------+----------+
```

> service nova-api restart

> service nova-consoleauth restart

> service nova-scheduler restart

> service nova-conductor restart

> service nova-novncproxy restart

# Nova安装【计算节点】

> apt install nova-compute

> vim /etc/nova/nova.conf

```
[DEFAULT]
# log_dir = /var/log/nova
transport_url = rabbit://openstack:openstack@controller

[api]
auth_strategy = keystone

[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova

[DEFAULT]
my_ip = 172.20.10.121

[DEFAULT]
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver

[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://172.20.10.120:6080/vnc_auto.html

[glance]
api_servers = http://controller:9292

[oslo_concurrency]
lock_path = /var/lib/nova/tmp

[placement]
# os_region_name = openstack
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = placement
```

> cat /etc/nova/nova.conf | grep ^\[\\[a-z]

```
[DEFAULT]
lock_path = /var/lock/nova
state_path = /var/lib/nova
transport_url = rabbit://openstack:openstack@controller
my_ip = 172.20.10.121
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver
[api]
auth_strategy = keystone
[api_database]
connection = sqlite:////var/lib/nova/nova_api.sqlite
[barbican]
[cache]
[cells]
enable = False
[cinder]
[compute]
[conductor]
[console]
[consoleauth]
[cors]
[database]
connection = sqlite:////var/lib/nova/nova.sqlite
[devices]
[ephemeral_storage_encryption]
[filter_scheduler]
[glance]
api_servers = http://controller:9292
[guestfs]
[healthcheck]
[hyperv]
[ironic]
[key_manager]
[keystone]
[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova
[libvirt]
[matchmaker_redis]
[metrics]
[mks]
[neutron]
[notifications]
[osapi_v21]
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[pci]
[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = placement
[placement_database]
[powervm]
[profiler]
[quota]
[rdp]
[remote_debug]
[scheduler]
[serial_console]
[service_user]
[spice]
[upgrade_levels]
[vault]
[vendordata_dynamic_auth]
[vmware]
[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html
[workarounds]
[wsgi]
[xenserver]
[xvp]
[zvm]
```

> egrep -c '(vmx|svm)' /proc/cpuinfo

```
0
```

> vim /etc/nova/nova-compute.conf

```
[libvirt]
# virt_type=kvm
virt_type = qemu
```

> cat /etc/nova/nova-compute.conf | grep ^\[\\[a-z]

```
[DEFAULT]
compute_driver=libvirt.LibvirtDriver
[libvirt]
virt_type = qemu
```

> service nova-compute restart

# 添加计算节点【控制节点】

> . admin-openrc


> openstack compute service list --service nova-compute

```
+----+--------------+-----------+------+---------+-------+----------------------------+
| ID | Binary       | Host      | Zone | Status  | State | Updated At                 |
+----+--------------+-----------+------+---------+-------+----------------------------+
|  8 | nova-compute | compute01 | nova | enabled | up    | 2019-05-19T11:53:09.000000 |
|  9 | nova-compute | compute02 | nova | enabled | up    | 2019-05-19T11:53:10.000000 |
+----+--------------+-----------+------+---------+-------+----------------------------+
```

> su -s /bin/sh -c "nova-manage cell_v2 discover_hosts --verbose" nova

```
+----+--------------+-----------+------+---------+-------+----------------------------+
| ID | Binary       | Host      | Zone | Status  | State | Updated At                 |
+----+--------------+-----------+------+---------+-------+----------------------------+
|  8 | nova-compute | compute01 | nova | enabled | up    | 2019-05-19T11:47:19.000000 |
|  9 | nova-compute | compute02 | nova | enabled | up    | 2019-05-19T11:47:20.000000 |
+----+--------------+-----------+------+---------+-------+----------------------------+
```

> openstack compute service list

```
+----+------------------+------------+----------+---------+-------+----------------------------+
| ID | Binary           | Host       | Zone     | Status  | State | Updated At                 |
+----+------------------+------------+----------+---------+-------+----------------------------+
|  1 | nova-scheduler   | controller | internal | enabled | up    | 2019-05-19T11:49:14.000000 |
|  5 | nova-consoleauth | controller | internal | enabled | up    | 2019-05-19T11:49:09.000000 |
|  6 | nova-conductor   | controller | internal | enabled | up    | 2019-05-19T11:49:10.000000 |
|  8 | nova-compute     | compute01  | nova     | enabled | up    | 2019-05-19T11:49:09.000000 |
|  9 | nova-compute     | compute02  | nova     | enabled | up    | 2019-05-19T11:49:10.000000 |
+----+------------------+------------+----------+---------+-------+----------------------------+
```

> openstack catalog list

```
+-----------+-----------+-----------------------------------------+
| Name      | Type      | Endpoints                               |
+-----------+-----------+-----------------------------------------+
| keystone  | identity  | RegionOne                               |
|           |           |   admin: http://controller:5000/v3/     |
|           |           | RegionOne                               |
|           |           |   internal: http://controller:5000/v3/  |
|           |           | RegionOne                               |
|           |           |   public: http://controller:5000/v3/    |
|           |           |                                         |
| nova      | compute   | RegionOne                               |
|           |           |   public: http://controller:8774/v2.1   |
|           |           | RegionOne                               |
|           |           |   internal: http://controller:8774/v2.1 |
|           |           | RegionOne                               |
|           |           |   admin: http://controller:8774/v2.1    |
|           |           |                                         |
| placement | placement | RegionOne                               |
|           |           |   admin: http://controller:8778         |
|           |           | RegionOne                               |
|           |           |   public: http://controller:8778        |
|           |           | RegionOne                               |
|           |           |   internal: http://controller:8778      |
|           |           |                                         |
| glance    | image     | RegionOne                               |
|           |           |   internal: http://controller:9292      |
|           |           | RegionOne                               |
|           |           |   admin: http://controller:9292         |
|           |           | RegionOne                               |
|           |           |   public: http://controller:9292        |
|           |           |                                         |
+-----------+-----------+-----------------------------------------+
```

> openstack image list

```
+--------------------------------------+--------+--------+
| ID                                   | Name   | Status |
+--------------------------------------+--------+--------+
| 072bf6a3-4d71-40c4-bf38-a0719c9e00ae | cirros | active |
+--------------------------------------+--------+--------+
```

> nova-status upgrade check

```
+--------------------------------+
| Upgrade Check Results          |
+--------------------------------+
| Check: Cells v2                |
| Result: Success                |
| Details: None                  |
+--------------------------------+
| Check: Placement API           |
| Result: Success                |
| Details: None                  |
+--------------------------------+
| Check: Resource Providers      |
| Result: Success                |
| Details: None                  |
+--------------------------------+
| Check: Ironic Flavor Migration |
| Result: Success                |
| Details: None                  |
+--------------------------------+
| Check: API Service Version     |
| Result: Success                |
| Details: None                  |
+--------------------------------+
| Check: Request Spec Migration  |
| Result: Success                |
| Details: None                  |
+--------------------------------+
| Check: Console Auths           |
| Result: Success                |
| Details: None                  |
+--------------------------------+
```

# Neutron安装【控制节点】

> mysql -u root -p

```
Enter password: root
```

> MariaDB [(none)]> CREATE DATABASE neutron;

> MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'controller' IDENTIFIED BY 'neutron';

> MariaDB [(none)]> GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%' IDENTIFIED BY 'neutron';


> . admin-openrc


> openstack user create --domain default --password-prompt neutron

```
User Password: neutron
Repeat User Password: neutron
+---------------------+----------------------------------+
| Field               | Value                            |
+---------------------+----------------------------------+
| domain_id           | default                          |
| enabled             | True                             |
| id                  | 30796cda895244e6951ca79b7952ef1c |
| name                | neutron                          |
| options             | {}                               |
| password_expires_at | None                             |
+---------------------+----------------------------------+
```

> openstack role add --project service --user neutron admin


> openstack service create --name neutron --description "OpenStack Networking" network

```
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | OpenStack Networking             |
| enabled     | True                             |
| id          | a5028e0afc2e466aa4240c5f09ddf427 |
| name        | neutron                          |
| type        | network                          |
+-------------+----------------------------------+
```

> openstack endpoint create --region RegionOne network public http://controller:9696

```
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | aa3390e76c4a4647b7caea11129d40e5 |
| interface    | public                           |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | a5028e0afc2e466aa4240c5f09ddf427 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+
```

> openstack endpoint create --region RegionOne network internal http://controller:9696

```
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | 8aab3933a5b64be99c549b5adb64a4ea |
| interface    | internal                         |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | a5028e0afc2e466aa4240c5f09ddf427 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+
```

> openstack endpoint create --region RegionOne network admin http://controller:9696

```
+--------------+----------------------------------+
| Field        | Value                            |
+--------------+----------------------------------+
| enabled      | True                             |
| id           | b110916348634d1fbf912bcd2cf67408 |
| interface    | admin                            |
| region       | RegionOne                        |
| region_id    | RegionOne                        |
| service_id   | a5028e0afc2e466aa4240c5f09ddf427 |
| service_name | neutron                          |
| service_type | network                          |
| url          | http://controller:9696           |
+--------------+----------------------------------+
```

> apt install neutron-server neutron-plugin-ml2 neutron-openvswitch-agent neutron-l3-agent neutron-dhcp-agent neutron-metadata-agent

> ovs-vsctl add-br br-ext


> vim /etc/neutron/neutron.conf

```
[database]
# connection = sqlite:////var/lib/neutron/neutron.sqlite
connection = mysql+pymysql://neutron:neutron@controller/neutron

[DEFAULT]
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true

[DEFAULT]
transport_url = rabbit://openstack:openstack@controller

[DEFAULT]
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron

[DEFAULT]
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true

[nova]
# ...
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = nova

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
```

> cat /etc/neutron/neutron.conf | grep ^\[\\[a-z]

```
[DEFAULT]
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
transport_url = rabbit://openstack:openstack@controller
auth_strategy = keystone
notify_nova_on_port_status_changes = true
notify_nova_on_port_data_changes = true
[agent]
root_helper = "sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf"
[cors]
[database]
connection = mysql+pymysql://neutron:neutron@controller/neutron
[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron
[matchmaker_redis]
[nova]
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = nova
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[quotas]
[ssl]
```

> vim /etc/neutron/plugins/ml2/ml2_conf.ini

```
[ml2]
type_drivers = flat,vlan,vxlan

[ml2]
tenant_network_types = vxlan

[ml2]
mechanism_drivers = openvswitch,l2population

[ml2]
extension_drivers = port_security

[ml2_type_flat]
flat_networks = provider

[ml2_type_vxlan]
vni_ranges = 10001:20000

[securitygroup]
enable_ipset = true
```

> cat /etc/neutron/plugins/ml2/ml2_conf.ini | grep ^\[\\[a-z]

```
[DEFAULT]
[l2pop]
[ml2]
type_drivers = flat,vlan,vxlan
tenant_network_types = vxlan
mechanism_drivers = openvswitch,l2population
extension_drivers = port_security
[ml2_type_flat]
flat_networks = provider
[ml2_type_geneve]
[ml2_type_gre]
[ml2_type_vlan]
[ml2_type_vxlan]
vni_ranges = 10001:20000
[securitygroup]
enable_ipset = true
```

> vim /etc/neutron/plugins/ml2/openvswitch_agent.ini

```
[ovs]
bridge_mappings = provider:br-ext
local_ip = 172.16.10.120

[agent]
tunnel_types = vxlan
l2_population = True

[securitygroup]
firewall_driver = iptables_hybrid
```

> cat /etc/neutron/plugins/ml2/openvswitch_agent.ini | grep ^\[\\[a-z]

```
[DEFAULT]
[agent]
tunnel_types = vxlan
l2_population = True
[network_log]
[ovs]
bridge_mappings = provider:br-ext
local_ip = 172.16.10.120
[securitygroup]
firewall_driver = iptables_hybrid
[xenapi]
```

> vim /etc/neutron/l3_agent.ini

```
[DEFAULT]
interface_driver = openvswitch
external_network_bridge =
```

> cat /etc/neutron/l3_agent.ini | grep ^\[\\[a-z]

```
[DEFAULT]
interface_driver = openvswitch
external_network_bridge =
[agent]
[ovs]
```

> vim /etc/neutron/dhcp_agent.ini 

```
[DEFAULT]
interface_driver = openvswitch
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
```

> cat /etc/neutron/dhcp_agent.ini | grep ^\[\\[a-z]

```
[DEFAULT]
interface_driver = openvswitch
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = true
[agent]
[ovs]
```

> vim /etc/neutron/metadata_agent.ini

```
[DEFAULT]
# ...
nova_metadata_host = controller
metadata_proxy_shared_secret = metadata
```

> cat /etc/neutron/metadata_agent.ini | grep ^\[\\[a-z]

```
[DEFAULT]
nova_metadata_host = controller
metadata_proxy_shared_secret = metadata
[agent]
[cache]
```

> vim /etc/nova/nova.conf

```
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
password = neutron
service_metadata_proxy = true
metadata_proxy_shared_secret = metadata
```

> cat /etc/nova/nova.conf | grep ^\[\\[a-z]

```
[DEFAULT]
lock_path = /var/lock/nova
state_path = /var/lib/nova
transport_url = rabbit://openstack:openstack@controller
my_ip = 172.20.10.120
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver
[api]
auth_strategy = keystone
[api_database]
connection = mysql+pymysql://nova:nova@controller/nova_api
[barbican]
[cache]
[cells]
enable = False
[cinder]
[compute]
[conductor]
[console]
[consoleauth]
[cors]
[database]
connection = mysql+pymysql://nova:nova@controller/nova
[devices]
[ephemeral_storage_encryption]
[filter_scheduler]
[glance]
api_servers = http://controller:9292
[guestfs]
[healthcheck]
[hyperv]
[ironic]
[key_manager]
[keystone]
[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova
[libvirt]
[matchmaker_redis]
[metrics]
[mks]
[neutron]
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron
service_metadata_proxy = true
metadata_proxy_shared_secret = metadata
[notifications]
[osapi_v21]
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[pci]
[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = placement
[placement_database]
connection = mysql+pymysql://placement:placement@controller/placement
[powervm]
[profiler]
[quota]
[rdp]
[remote_debug]
[scheduler]
[serial_console]
[service_user]
[spice]
[upgrade_levels]
[vault]
[vendordata_dynamic_auth]
[vmware]
[vnc]
enabled = true
server_listen = $my_ip
server_proxyclient_address = $my_ip
[workarounds]
[wsgi]
[xenserver]
[xvp]
[zvm]
```

> su -s /bin/sh -c "neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head" neutron

```
INFO  [alembic.runtime.migration] Context impl MySQLImpl.
INFO  [alembic.runtime.migration] Will assume non-transactional DDL.
  Running upgrade for neutron ...
......
INFO  [alembic.runtime.migration] Running upgrade 458aa42b14b -> f83a0b2964d0, rename tenant to project
INFO  [alembic.runtime.migration] Running upgrade f83a0b2964d0 -> fd38cd995cc0, change shared attribute for firewall resource
  OK
```

> service nova-api restart

> service neutron-server restart

> service neutron-openvswitch-agent restart

> service neutron-dhcp-agent restart

> service neutron-metadata-agent restart

> service neutron-l3-agent restart


# Neutron安装【计算节点】

> apt install neutron-openvswitch-agent


> vim /etc/neutron/neutron.conf

```
[database]
# connection = sqlite:////var/lib/neutron/neutron.sqlite

[DEFAULT]
transport_url = rabbit://openstack:openstack@controller

[DEFAULT]
auth_strategy = keystone

[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron

[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
```

> cat /etc/neutron/neutron.conf | grep ^\[\\[a-z]

```
[DEFAULT]
core_plugin = ml2
transport_url = rabbit://openstack:openstack@controller
auth_strategy = keystone
[agent]
root_helper = "sudo /usr/bin/neutron-rootwrap /etc/neutron/rootwrap.conf"
[cors]
[database]
[keystone_authtoken]
www_authenticate_uri = http://controller:5000
auth_url = http://controller:5000
memcached_servers = controller:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = neutron
[matchmaker_redis]
[nova]
[oslo_concurrency]
lock_path = /var/lib/neutron/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[quotas]
[ssl]
```

> vim /etc/neutron/plugins/ml2/openvswitch_agent.ini

```
[ovs]
bridge_mappings = provider:br-ext
local_ip = 172.16.10.121

[agent]
tunnel_types = vxlan
l2_population = True
```

> cat /etc/neutron/plugins/ml2/openvswitch_agent.ini | grep ^\[\\[a-z]

```
[DEFAULT]
[agent]
tunnel_types = vxlan
l2_population = True
[network_log]
[ovs]
local_ip = 172.16.10.121
[securitygroup]
[xenapi]
```

> vim /etc/nova/nova.conf

```
[neutron]
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron
```

> cat /etc/nova/nova.conf | grep ^\[\\[a-z]

```
[DEFAULT]
lock_path = /var/lock/nova
state_path = /var/lib/nova
transport_url = rabbit://openstack:openstack@controller
my_ip = 172.20.10.121
use_neutron = true
firewall_driver = nova.virt.firewall.NoopFirewallDriver
[api]
auth_strategy = keystone
[api_database]
connection = sqlite:////var/lib/nova/nova_api.sqlite
[barbican]
[cache]
[cells]
enable = False
[cinder]
[compute]
[conductor]
[console]
[consoleauth]
[cors]
[database]
connection = sqlite:////var/lib/nova/nova.sqlite
[devices]
[ephemeral_storage_encryption]
[filter_scheduler]
[glance]
api_servers = http://controller:9292
[guestfs]
[healthcheck]
[hyperv]
[ironic]
[key_manager]
[keystone]
[keystone_authtoken]
auth_url = http://controller:5000/v3
memcached_servers = controller:11211
auth_type = password
project_domain_name = Default
user_domain_name = Default
project_name = service
username = nova
password = nova
[libvirt]
[matchmaker_redis]
[metrics]
[mks]
[neutron]
url = http://controller:9696
auth_url = http://controller:5000
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = neutron
[notifications]
[osapi_v21]
[oslo_concurrency]
lock_path = /var/lib/nova/tmp
[oslo_messaging_amqp]
[oslo_messaging_kafka]
[oslo_messaging_notifications]
[oslo_messaging_rabbit]
[oslo_messaging_zmq]
[oslo_middleware]
[oslo_policy]
[pci]
[placement]
region_name = RegionOne
project_domain_name = Default
project_name = service
auth_type = password
user_domain_name = Default
auth_url = http://controller:5000/v3
username = placement
password = placement
[placement_database]
[powervm]
[profiler]
[quota]
[rdp]
[remote_debug]
[scheduler]
[serial_console]
[service_user]
[spice]
[upgrade_levels]
[vault]
[vendordata_dynamic_auth]
[vmware]
[vnc]
enabled = true
server_listen = 0.0.0.0
server_proxyclient_address = $my_ip
novncproxy_base_url = http://controller:6080/vnc_auto.html
[workarounds]
[wsgi]
[xenserver]
[xvp]
[zvm]
```

> service nova-compute restart

> service neutron-openvswitch-agent restart


# 确认操作【控制节点】

> openstack network agent list

```
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host       | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
| 40f103b1-115b-40cd-a960-fa2111d68c40 | Open vSwitch agent | controller | None              | :-)   | UP    | neutron-openvswitch-agent |
| 49671c5f-307d-4bb2-b5db-e4040e4b05dc | L3 agent           | controller | nova              | :-)   | UP    | neutron-l3-agent          |
| 741916c3-c2ec-4a56-a24c-5f557ff42f9a | Metadata agent     | controller | None              | :-)   | UP    | neutron-metadata-agent    |
| 8389c912-36c2-410d-b31e-2d7c56045c61 | Open vSwitch agent | compute01  | None              | :-)   | UP    | neutron-openvswitch-agent |
| a4457089-9cc4-4ae4-9421-1f4af667b965 | Open vSwitch agent | compute02  | None              | :-)   | UP    | neutron-openvswitch-agent |
| ad0f2e55-ae59-48b8-a83f-92dbb31d62e7 | DHCP agent         | controller | nova              | :-)   | UP    | neutron-dhcp-agent        |
+--------------------------------------+--------------------+------------+-------------------+-------+-------+---------------------------+
```

# Horizon安装【控制节点】

> apt install openstack-dashboard


> vim /etc/openstack-dashboard/local_settings.py

```
# OPENSTACK_HOST = "127.0.0.1"
OPENSTACK_HOST = "controller"
ALLOWED_HOSTS = ['*']

SESSION_ENGINE = 'django.contrib.sessions.backends.cache'

#CACHES = {
#    'default': {
#        'BACKEND': 'django.core.cache.backends.memcached.MemcachedCache',
#        'LOCATION': '127.0.0.1:11211',
#    },
#}

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

# OPENSTACK_KEYSTONE_DEFAULT_ROLE = "_member_"
OPENSTACK_KEYSTONE_DEFAULT_ROLE = "user"

# TIME_ZONE = "UTC"
TIME_ZONE = "Asia/Shanghai"
```

> vim /etc/apache2/conf-available/openstack-dashboard.conf

```
WSGIApplicationGroup %{GLOBAL}
```

> service apache2 reload

# LBaas安装【控制节点】

> crudini --set /etc/neutron/neutron.conf DEFAULT service_plugins router,neutron_lbaas.services.loadbalancer.plugin.LoadBalancerPluginv2

> crudini --set /etc/neutron/neutron_lbaas.conf service_providers service_provider LOADBALANCERV2:Haproxy:neutron_lbaas.drivers.haproxy.plugin_driver.HaproxyOnHostPluginDriver:default

> crudini --set /etc/neutron/lbaas_agent.ini DEFAULT interface_driver neutron.agent.linux.interface.OVSInterfaceDriver


> neutron-db-manage --subproject neutron-lbaas upgrade head

> service neutron-lbaasv2-agent restart

> service neutron-server restart

> git clone https://git.openstack.org/openstack/neutron-lbaas-dashboard

> cd neutron-lbaas-dashboard

> git checkout stable/rocky

> python setup.py install


# 集成ODL

## 检查内核版本>=4.3
> uname -r

```
4.15.0-45-generic
```

## 检查conntrack内核模块
> lsmod | grep conntrack

```
nf_conntrack_ipv6      20480  1
nf_conntrack_ipv4      16384  1
nf_defrag_ipv4         16384  1 nf_conntrack_ipv4
nf_defrag_ipv6         36864  2 nf_conntrack_ipv6,openvswitch
nf_conntrack          131072  6 nf_conntrack_ipv6,nf_conntrack_ipv4,nf_nat,nf_nat_ipv6,nf_nat_ipv4,openvswitch
libcrc32c              16384  4 nf_conntrack,nf_nat,openvswitch,raid456
```

## 控制节点、计算节点
> git clone https://github.com/openstack/networking-odl.git 

> cd networking-odl/

> git checkout stable/rocky

> python ./setup.py install


## 控制节点
* Note：neutron-server服务报错，按照以下步骤安装websocket

> apt install python-pip

> pip install websocket

> pip install websocket-client==0.47.0


## 控制节点
> nova list

> nova delete <instance names>

> neutron subnet-list

> neutron router-list

> neutron router-port-list <router name>

> neutron router-interface-delete <router name> <subnet ID or name>

> neutron subnet-delete <subnet name>
> neutron net-list

> neutron net-delete <net name>

> neutron router-delete <router name>

> neutron port-list


## 控制节点
> systemctl stop neutron-server

> systemctl stop neutron-l3-agent

> systemctl disable neutron-l3-agent

> systemctl stop neutron-openvswitch-agent

> systemctl disable neutron-openvswitch-agent


## 计算节点
> systemctl stop neutron-openvswitch-agent
> systemctl disable neutron-openvswitch-agent


## 控制节点
> crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 mechanism_drivers opendaylight_v2

> crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 tenant_network_types vxlan

> crudini --set /etc/neutron/plugins/ml2/ml2_conf.ini ml2 extension_drivers port_security

> crudini --set /etc/neutron/neutron.conf DEFAULT service_plugins odl-router_v2

> crudini --set /etc/neutron/dhcp_agent.ini DEFAULT force_metadata True

> crudini --set /etc/neutron/dhcp_agent.ini ovs ovsdb_interface vsctl

> crudini --set /etc/neutron/dhcp_agent.ini DEFAULT ovs_integration_bridge br-int

> cat <<EOT>> /etc/neutron/plugins/ml2/ml2_conf.ini \
[ml2_odl] \
url = http://172.20.10.131:8181/controller/nb/v2/neutron \
password = admin \
username = admin \
EOT

> mysql -e "DROP DATABASE IF EXISTS neutron;"

> mysql -e "CREATE DATABASE neutron CHARACTER SET utf8;"
/usr/bin/neutron-db-manage --config-file /etc/neutron/neutron.conf --config-file /etc/neutron/plugins/ml2/ml2_conf.ini upgrade head

> systemctl start neutron-server


## 控制节点
> systemctl stop openvswitch-switch

> rm -rf /var/log/openvswitch/*

> rm -rf /etc/openvswitch/conf.db

> systemctl start openvswitch-switch

> ovs-vsctl set-manager tcp:172.20.10.131:6640

> ovs-vsctl set Open_vSwitch . other_config:local_ip=172.20.10.120

> ovs-vsctl set Open_vSwitch . other_config:provider_mappings=provier:ens224
neutron-odl-ovs-hostconfig --datapath_type=system

## 计算节点1
> systemctl stop openvswitch-switch

> rm -rf /var/log/openvswitch/*

> rm -rf /etc/openvswitch/conf.db

> systemctl start openvswitch-switch

> ovs-vsctl set-manager tcp:172.20.10.131:6640

> ovs-vsctl set Open_vSwitch . other_config:local_ip=172.20.10.121

> ovs-vsctl set Open_vSwitch . other_config:provider_mappings=provier:ens224
neutron-odl-ovs-hostconfig --datapath_type=system


## 计算节点2
> systemctl stop openvswitch-switch

> rm -rf /var/log/openvswitch/*

> rm -rf /etc/openvswitch/conf.db

> systemctl start openvswitch-switch

> ovs-vsctl set-manager tcp:172.20.10.131:6640

> ovs-vsctl set Open_vSwitch . other_config:local_ip=172.20.10.122

> ovs-vsctl set Open_vSwitch . other_config:provider_mappings=provier:ens224
neutron-odl-ovs-hostconfig --datapath_type=system


## ODL节点
> tar zxf jdk-8u212-linux-x64.tar.gz -C /root

> tar zxf karaf-0.8.4.tar.gz -C /root

> cd /root/karaf-0.8.4

> ./bin/start

> ./bin/client

> opendaylight-user@root> feature:install odl-netvirt-openstack odl-dlux-core odl-mdsal-apidocs


## 所有节点
> ovs-vsctl show

```
356aeefd-1fc7-4f2d-b5b0-69150ae00b94
    Manager "tcp:172.20.10.131:6640"
        is_connected: true
    Bridge br-int
        Controller "tcp:172.20.10.131:6653"
            is_connected: true
        fail_mode: secure
        Port "tun97119295314"
            Interface "tun97119295314"
                type: vxlan
                options: {key=flow, local_ip="172.20.10.120", remote_ip="172.20.10.122"}
                bfd_status: {diagnostic="No Diagnostic", flap_count="1", forwarding="true", remote_diagnostic="No Diagnostic", remote_state=up, state=up}
        Port br-int
            Interface br-int
                type: internal
        Port "ens224"
            Interface "ens224"
        Port "tun249b7250379"
            Interface "tun249b7250379"
                type: vxlan
                options: {key=flow, local_ip="172.20.10.120", remote_ip="172.20.10.121"}
                bfd_status: {diagnostic="No Diagnostic", flap_count="1", forwarding="true", remote_diagnostic="No Diagnostic", remote_state=up, state=up}
    ovs_version: "2.10.0"
```

## 控制节点
> openstack network agent list

```
+--------------------------------------+----------------+------------+-------------------+-------+-------+------------------------------+
| ID                                   | Agent Type     | Host       | Availability Zone | Alive | State | Binary                       |
+--------------------------------------+----------------+------------+-------------------+-------+-------+------------------------------+
| 24319b30-3928-48a4-8f0c-a448732dae05 | ODL L2         | compute02  | None              | :-)   | UP    | neutron-odlagent-portbinding |
| 36a5de48-98c1-4ccf-8aa1-717000924f06 | Metadata agent | controller | None              | :-)   | UP    | neutron-metadata-agent       |
| 58d89f7d-df6c-4b80-8e58-3bdb4b2d73fe | DHCP agent     | controller | nova              | :-)   | UP    | neutron-dhcp-agent           |
| d3fafd3d-9564-48af-958b-2f05cb21718e | ODL L2         | compute01  | None              | :-)   | UP    | neutron-odlagent-portbinding |
| e671e52e-f7f5-48d0-a75c-0598f67f50c8 | ODL L2         | controller | None              | :-)   | UP    | neutron-odlagent-portbinding |
+--------------------------------------+----------------+------------+-------------------+-------+-------+------------------------------+
```
