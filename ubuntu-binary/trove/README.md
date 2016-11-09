# Trove 安裝與設定
OpenStack 的 Trove 提供了關聯性與非關聯性的資料庫即服務(DBaaS)，如 MySQL、Redis等，其利用 OpenStack 打造可擴展與可靠的服務。本章節將在`Controller`節點進行操作。

- [安裝前準備](#安裝前準備)
- [套件安裝與設定](#套件安裝與設定)
- [驗證服務](#驗證服務)

### 安裝前準備
在開始安裝前，要預先建立一個資料庫給 Trove 儲存相關資訊，使用以下指令建立資料庫：
```sh
$ mysql -u root -p
```

透過以下命令用來更新現有帳號資料或建立 Trove 資料庫：
```sql
CREATE DATABASE trove;
GRANT ALL PRIVILEGES ON trove.* TO 'trove'@'localhost' IDENTIFIED BY 'TROVE_DBPASS';
GRANT ALL PRIVILEGES ON trove.* TO 'trove'@'%' IDENTIFIED BY 'TROVE_DBPASS';
```
> 這邊`TROVE_DBPASS`可以隨需求修改。

完成後離開資料庫，接著要建立 Service 與 API Endpoint，首先導入`admin`環境變數：
```sh
$ . admin-openrc
```

接著透過以下流程來建立 Trove 的使用者、Service 以及 API Endpoint：
```sh
# 建立 Trove user
$ openstack user create --domain default --password TROVE_PASS --email trove@example.com trove

# 新增 Trove 到 Admin Role
$ openstack role add --project service --user trove admin

# 建立 Trove service
$ openstack service create --name trove \
--description "Database" database

# 建立 Trove public endpoints
$ openstack endpoint create --region RegionOne \
database public http://10.0.0.11:8779/v1.0/%\(tenant_id\)s

# 建立 Trove internal endpoints
$ openstack endpoint create --region RegionOne \
database internal http://10.0.0.11:8779/v1.0/%\(tenant_id\)s

# 建立 Trove admin endpoints
$ openstack endpoint create --region RegionOne \
database admin http://10.0.0.11:8779/v1.0/%\(tenant_id\)s
```
> 這邊若`TROVE_PASS`要更改的話，可以更改。

### 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install -y python-trove python-troveclient \
python-glanceclient trove-common trove-api trove-taskmanager \
trove-conductor
```

安裝完成後，編輯`/etc/trove/trove.conf`設定檔，在`[DEFAULT]`部分加入以下內容：
```
[DEFAULT]
...
log_dir = /var/log/trove
trove_auth_url = http://10.0.0.11:5000/v2.0
nova_compute_url = http://10.0.0.11:8774/v2
cinder_url = http://10.0.0.11:8776/v1
swift_url = http://10.0.0.11:8080/v1/AUTH_
notifier_queue_hostname = 10.0.0.11

auth_strategy = keystone
rpc_backend = rabbit
add_addresses = True
network_label_regex = .*
api_paste_config = /etc/trove/api-paste.ini
```
> 如果使用 radosgw 來提供 Swift 要改用`http://10.0.0.11:8080/swift/v1`。


在`[database]`部分修改使用以下方式：
```
[database]
connection = mysql+pymysql://trove:TROVE_DBPASS@10.0.0.11/trove
```
> 這邊```TROVE_DBPASS```可以隨需求修改。

在`[oslo_messaging_rabbit]`部分加入以下內容：
```
[oslo_messaging_rabbit]
rabbit_host = 10.0.0.11
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
> 這邊`RABBIT_PASS`可以隨需求修改。

在`[keystone_authtoken]`部分加入以下內容：
```
[keystone_authtoken]
memcached_servers = 10.0.0.11:11211
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = trove
password = TROVE_PASS
```
> 這邊`TROVE_PASS`可以隨需求修改。


編輯`/etc/trove/trove-taskmanager.conf`設定檔，在`[DEFAULT]`部分加入以下內容：
```
[DEFAULT]
...
log_dir = /var/log/trove
trove_auth_url = http://10.0.0.11:5000/v2.0
nova_compute_url = http://10.0.0.11:8774/v2
cinder_url = http://10.0.0.11:8776/v1
swift_url = http://10.0.0.11:8080/v1/AUTH_
notifier_queue_hostname = 10.0.0.11

rpc_backend = rabbit

nova_proxy_admin_user = admin
nova_proxy_admin_pass = ADMIN_PASS
nova_proxy_admin_tenant_name = service
taskmanager_manager = trove.taskmanager.manager.Manager
```
> 如果使用 radosgw 來提供 Swift 要改用`http://10.0.0.11:8080/swift/v1`。

> 這邊`ADMIN_PASS`可以隨需求修改。

在`[database]`部分修改使用以下方式：
```
[database]
connection = mysql+pymysql://trove:TROVE_DBPASS@10.0.0.11/trove
```
> 這邊`TROVE_DBPASS`可以隨需求修改。

在`[oslo_messaging_rabbit]`部分加入以下內容：
```
[oslo_messaging_rabbit]
rabbit_host = 10.0.0.11
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
> 這邊`RABBIT_PASS`可以隨需求修改。

編輯`/etc/trove/trove-conductor.conf`設定檔，，在`[DEFAULT]`部分加入以下內容：
```
[DEFAULT]
...
log_dir = /var/log/trove
trove_auth_url = http://10.0.0.11:5000/v2.0
nova_compute_url = http://10.0.0.11:8774/v2
cinder_url = http://10.0.0.11:8776/v1
swift_url = http://10.0.0.11:8080/v1/AUTH_
notifier_queue_hostname = 10.0.0.11

rpc_backend = rabbit
```
> 如果使用 radosgw 來提供 Swift 要改用`http://10.0.0.11:8080/swift/v1`。

> 這邊`ADMIN_PASS`可以隨需求修改。

在`[database]`部分修改使用以下方式：
```
[database]
connection = mysql+pymysql://trove:TROVE_DBPASS@10.0.0.11/trove
```
> 這邊`TROVE_DBPASS`可以隨需求修改。

在`[oslo_messaging_rabbit]`部分加入以下內容：
```
[oslo_messaging_rabbit]
rabbit_host = 10.0.0.11
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
> 這邊`RABBIT_PASS`可以隨需求修改。

接著編輯`/etc/trove/trove-guestagent.conf`設定檔，在`[DEFAULT]`部分加入以下內容：
```
[DEFAULT]
rabbit_host = 10.0.0.11
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
nova_proxy_admin_user = admin
nova_proxy_admin_pass = ADMIN_PASS
nova_proxy_admin_tenant_name = service
trove_auth_url = http://10.0.0.11:35357/v2.0
```

完成所有設定後，即可同步資料庫來建立 Trove 資料表：
```sh
$ sudo trove-manage db_sync
```

接著在重新啟動之前，需要修復 Ubuntu Package 的 Trove Bugs，修改`/etc/init/trove-taskmanager.conf`與`/etc/init/trove-conductor.conf`檔案，加入以下內容：
```sh
# Trove Task Manager 修改以下：
exec start-stop-daemon --start --chdir /var/lib/trove \
        --chuid trove:trove --make-pidfile --pidfile /var/run/trove/trove-taskmanager.pid \
        --exec /usr/bin/trove-taskmanager -- \
        --config-file=/etc/trove/trove-taskmanager.conf  ${DAEMON_ARGS}


# Trove Conductor 修改以下：
exec start-stop-daemon --start --chdir /var/lib/trove \
                --chuid trove:trove --make-pidfile --pidfile /var/run/trove/trove-conductor.pid \
                --exec /usr/bin/trove-conductor -- \
                --config-file=/etc/trove/trove-conductor.conf ${DAEMON_ARGS}
```

重新啟動所有 Trove 服務：
```sh
sudo service trove-api restart
sudo service trove-taskmanager restart
sudo service trove-conductor restart
```

# 驗證服務
首先回到`Controller`節點並接著導入`admin`帳號來驗證服務：
```sh
$ . admin-openrc
```

下載將使用的 Trove service 映像檔：
```sh
$ wget http://tarballs.openstack.org/trove/images/ubuntu/mysql.qcow2
```
> 其他映像檔可以參考 [Trove images](http://tarballs.openstack.org/trove/images/)。

> 也可以透過 [Diskimage Builder](https://github.com/openstack/diskimage-builder) 來建立 Image。

使用 Glance client 來上傳 Trove Service 映像檔：
```sh
$ glance image-create --name "mysqlTest" \
--file mysql.qcow2 \
--disk-format qcow2 --container-format bare \
--visibility public --progress
```

建立 datastore。這邊需要建立每種類型資料庫的 datastore，這邊使用 MySQL 為範例：
```sh
$ sudo trove-manage --config-file /etc/trove/trove.conf datastore_update mysql ''
Datastore 'mysql' updated.
```

然後更新 datastore 使用映像檔：
```sh
$ Glance_Image_ID=$(glance image-list | awk '/ mysqlTest / { print $2 }')
$ sudo trove-manage datastore_version_update mysql mysql-5.6 mysql ${Glance_Image_ID} '' 1
Datastore version 'mysql-5.6' updated.
```

完成後就可以建立 Instance，以下列表為建議的資料庫 flavor：

| Database  | RAM (MB) | Disk (GB) | VCPUs |
|-----------|----------|-----------|-------|
| MySQL     | 512      | 5         | 1     |
| Cassandra | 2048     | 5         | 1     |
| MongoDB   | 1024     | 5         | 1     |
| Redis     | 512      | 5         | 1     |

若已有建立的 flavor 符合要求，就可以透過以下指令建立一個 Trove Instance：
```sh
$ FLAVOR_ID=$(openstack flavor list | awk '/ m1.small / { print $2 }')
$ NET_ID=$(openstack network list | awk '/ admin-net / { print $2 }')
$ trove create mysql-instance ${FLAVOR_ID} \
--nic net-id=${NET_ID} \
--size 5 --databases myDB \
--users userA:password \
--datastore_version mysql-5.6 \
--datastore mysql
```

透過 Trove client 來查看服務列表，如以下方式：
```sh
$ trove list
+--------------------------------------+----------------+-----------+-------------------+--------+-----------+------+
| ID                                   | Name           | Datastore | Datastore Version | Status | Flavor ID | Size |
+--------------------------------------+----------------+-----------+-------------------+--------+-----------+------+
| 1cf29b00-f8e6-4836-bddc-0f24e73694ba | mysql-instance | mysql     | mysql-5.6         | BUILD  | 2         |    5 |
+--------------------------------------+----------------+-----------+-------------------+--------+-----------+------+
```

最後完成後，可以到有 MySQL Client 的電腦進行存取：
```sh
$ mysql -u userA -p password -h IP_ADDRESS myDB
```
> P.S. `IP_ADDRESS`為建立的虛擬機 IP 位址。
