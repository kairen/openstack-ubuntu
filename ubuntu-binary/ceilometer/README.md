# Ceilometer 安裝與設定
OpenStack 的 Ceilometer 提供了遙測資料收集服務，其提供了三大功能：

* 有效地輪詢 OpenStack 服務的計量資料。
* 透過監控通知從服務端發送收集事件與計量資料。
* 將收集資料發送到各種目標，其包含資料儲存與訊息佇列。

- [Controller Node](#controller-node)
    - [安裝前準備](#安裝前準備)
    - [Controller 套件安裝與設定](#controller-套件安裝與設定)
- [啟用 Image Service Meters](#啟用-image-service-meters)
- [啟用 Compute Service Meters](#啟用-compute-service-meters)
- [啟用 Block Storage Meters](#啟用-block-storage-meters)
- [啟用 Object Storage Meters](#啟用-object-storage-meters)
- [驗證服務](#驗證服務)

# Controller Node
在 Controller 節點我們需要安裝 Ceilometer 的 API Server 以及用於儲存資料的 MongoDB。

### 安裝前準備
由於 Ceilometer 的資料庫使採用`MongoDB`來做儲存，可以透過以下方式進行安裝：
```sh
$ sudo apt-get install -y mongodb-server mongodb-clients python-pymongo
```

安裝完成後，編輯`/etc/mongodb.conf`設定檔，並加入以下內容：
```
bind_ip = 10.0.0.11
smallfiles = true
```
> 預設下 MongoDB 會建立一個 /var/lib/mongodb/journal 目錄來儲存 journal 檔案（每個檔案 1GB 大小），若要縮小 journal 檔案大小可以設定`smallfiles`為`true`。

若有修改 journal 的設定，請務必停止服務並刪除 journal 初始化檔案，再重新啟動服務：
```sh
sudo service mongodb stop
sudo rm /var/lib/mongodb/journal/prealloc.*
sudo service mongodb start
```
> 你也可以關閉 journaling，詳細資訊可以看 [MongoDB manual](http://docs.mongodb.org/manual/).

確認修改後，透過以下命令用來更新現有帳號資料或建立 Ceilometer 資料庫：
```sh
$ mongo --host 10.0.0.11 --eval '
db = db.getSiblingDB("ceilometer");
db.addUser({user: "ceilometer",
pwd: "CEILOMETER_DBPASS",
roles: [ "readWrite", "dbAdmin" ]})'
```

建立成功會看到類似以下訊息：
```
MongoDB shell version: 2.6.10
connecting to: 10.0.0.11:27017/test
Successfully added user: { "user" : "ceilometer", "roles" : [ "readWrite", "dbAdmin" ] }
```
> 這邊`CEILOMETER_DBPASS`可以隨需求修改。

完成後離開資料庫，接著要建立 Service 與 API Endpoint，首先導入`admin`環境變數：
```sh
$ . admin-openrc
```

接著透過以下流程來建立 Ceilometer 的使用者、Service 以及 API Endpoint：
```sh
# 建立 Ceilometer User
$ openstack user create --domain default --password CEILOMETER_PASS --email ceilometer@example.com ceilometer

# 建立 Ceilometer Role
$ openstack role add --project service --user ceilometer admin

# 建立 Ceilometer Service
$ openstack service create --name ceilometer  --description "Telemetry" metering

# 建立 Ceilometer public endpoints
$ openstack endpoint create --region RegionOne \
metering public http://10.0.0.11:8777

# 建立 Ceilometer internal endpoints
$ openstack endpoint create --region RegionOne \
metering internal http://10.0.0.11:8777

# 建立 Ceilometer admin endpoints
$ openstack endpoint create --region RegionOne \
metering admin http://10.0.0.11:8777
```
> 這邊`CEILOMETER_PASS`可以隨需求修改。

### Controller 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，接著透過以下指令進行安裝：
```sh
$ sudo apt-get install -y ceilometer-api ceilometer-collector \
ceilometer-agent-central ceilometer-agent-notification \
python-ceilometerclient
```

安裝完成後，編輯`/etc/ceilometer/ceilometer.conf`設定檔，在`[DEFAULT]`部分加入以下內容：
```
[DEFAULT]
rpc_backend = rabbit
auth_strategy = keystone
```

在`[database]`部分修改使用以下方式：
```
[database]
connection = mongodb://ceilometer:CEILOMETER_DBPASS@10.0.0.11:27017/ceilometer
```
> 這邊`CEILOMETER_DBPASS`可以隨需求修改。

在`[oslo_messaging_rabbit]`部分加入以下內容：
```
[oslo_messaging_rabbit]
rabbit_host = 10.0.0.11
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```

在`[keystone_authtoken]`部分加入以下內容：
```
[keystone_authtoken]
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
memcached_servers = 10.0.0.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = ceilometer
password = CEILOMETER_PASS
```
> 這邊`CEILOMETER_PASS`可以隨需求修改。

在`[service_credentials]`部分加入以下內容：
```
[service_credentials]
auth_type = password
auth_url = http://10.0.0.11:5000/v3
project_domain_name = default
user_domain_name = default
project_name = service
interface = internalURL
region_name = RegionOne
username = ceilometer
password = CEILOMETER_PASS
```
> 這邊`CEILOMETER_PASS`可以隨需求修改。

完成後接著要設定 Apache2 來提供 Ceilometer API，新增檔案`/etc/apache2/sites-available/ceilometer.conf`，並加入以下內容：
```
Listen 8777

<VirtualHost *:8777>
    WSGIDaemonProcess ceilometer-api processes=2 threads=10 user=ceilometer group=ceilometer display-name=%{GROUP}
    WSGIProcessGroup ceilometer-api
    WSGIScriptAlias / "/var/www/ceilometer/app.wsgi"
    WSGIApplicationGroup %{GLOBAL}
    ErrorLog /var/log/apache2/ceilometer_error.log
    CustomLog /var/log/apache2/ceilometer_access.log combined
</VirtualHost>

WSGISocketPrefix /var/run/apache2
```

接著建立一個存放 WSGI 檔案的目錄：
```sh
$ sudo mkdir -p /var/www/ceilometer
```

複製 WSGI 檔案到 /var/www/ceilometer 目錄底下：
```sh
$ sudo cp /usr/lib/python2.7/dist-packages/ceilometer/api/app.wsgi /var/www/ceilometer/
```

設定完成後，接著啟用該設定來提供 API：
```sh
sudo service ceilometer-api stop
sudo a2ensite ceilometer
sudo service apache2 reload
sudo service apache2 restart
```

最後重新啟動 Ceilometer 其他 daemon：
```sh
sudo service ceilometer-agent-central restart
sudo service ceilometer-agent-notification restart
sudo service ceilometer-collector restart
```

# 啟用 Image Service Meters
Ceilometer 使用 Notifications 來收集映像檔服務的 Meters。到`Controller（Glance）` 節點上編輯`/etc/glance/glance-api.conf`、`/etc/glance/glance-registry.conf`設定檔，在`[DEFAULT]`部分加入以下內容：
```
[DEFAULT]
...
rpc_backend = rabbit
```

在`[oslo_messaging_notifications]`部分加入以下內容：
```
[oslo_messaging_notifications]
driver = messagingv2
```

在`[oslo_messaging_rabbit]`部分加入以下內容：
```
[oslo_messaging_rabbit]
rabbit_host = 10.0.0.11
rabbit_userid = openstack
rabbit_password = RABBIT_PASS
```
> 這邊`RABBIT_PASS`可以隨需求修改。

完成後，就可以重新啟動所有 Glance 服務：
```sh
sudo service glance-registry restart
sudo service glance-api restart
```

# 啟用 Compute Service Meters
Ceilometer 使用 Notifications 與 Agent 的組合來收集 Compute 的 meters。以下操作將會在`Compute`節點進行。


在開始設定之前，首先要在被監控的`Compute`節點安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install -y ceilometer-agent-compute
```

安裝完成後，編輯`/etc/ceilometer/ceilometer.conf`設定檔，在`[DEFAULT]`部分加入以下內容：
```
[DEFAULT]
rpc_backend = rabbit
auth_strategy = keystone
```

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
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
memcached_servers = 10.0.0.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = ceilometer
password = CEILOMETER_PASS
```
> 這邊`CEILOMETER_PASS`可以隨需求修改。

在`[service_credentials]`部分加入以下內容：
```
[service_credentials]
auth_url = http://10.0.0.11:5000
project_domain_id = default
user_domain_id = default
auth_type = password
username = ceilometer
interface = internalURL
region_name = RegionOne
project_name = service
password = CEILOMETER_PASS
```

接著編輯`/etc/nova/nova.conf`設定檔，在`[DEFAULT]`部分加入以下內容：
```
[DEFAULT]
...
instance_usage_audit = True
instance_usage_audit_period = hour
notify_on_state_change = vm_and_task_state
```

在`[oslo_messaging_notifications]`部分加入以下內容：
```
[oslo_messaging_notifications]
driver = messagingv2
```

完成後，就可以重新啟動所有相關服務：
```sh
sudo service ceilometer-agent-compute restart
sudo service nova-compute restart
```

# 啟用 Block Storage Meters
Ceilometer 使用 Notifications 來收集區塊儲存服務的 meters。我們須在 `Controller`與`Storage`節點上執行以下步驟。

首先編輯所有節點的`/etc/cinder/cinder.conf`設定檔，在`[oslo_messaging_notifications]`部分加入以下內容：
```
[oslo_messaging_notifications]
driver = messagingv2
```

在`Controller`節點，重新啟動以下服務：
```sh
sudo service cinder-api restart
sudo service cinder-scheduler restart
```

在`Storage`節點，重新啟動以下服務：
```sh
$ sudo service cinder-volume restart
```

# 啟用 Object Storage Meters
Ceilometer 使用 Pooling 與 Notifications 組合來收集物件儲存服務的 meters。由於 Ceilometer 服務需要使用到 ResellerAdmin 來取得物件儲存的 meters。首先在`Controller`執行以下步驟：
```sh
$ . admin-openrc
```

透過 Keystone client 建立 ResellerAdmin role：
```sh
$ openstack role create ResellerAdmin
```

然後將 ResellerAdmin role 加入到 Server project，並賦予給 ceilometer 使用者：
```sh
$ openstack role add --project service --user ceilometer ResellerAdmin
```

完成後，在進行設定前必須先安裝相關套件：
```sh
$ sudo apt-get install -y python-ceilometermiddleware
```

接著在`Controller`節點上編輯`/etc/swift/proxy-server.conf`設定檔，在`[filter:keystoneauth]`部分加入以下內容：
```
[filter:keystoneauth]
...
operator_roles = admin, user, ResellerAdmin
```

在`[pipeline:main]`部分加入以下內容：
```
[pipeline:main]
...
pipeline = ceilometer catch_errors gatekeeper healthcheck proxy-logging cache container_sync bulk ratelimit authtoken keystoneauth container-quotas account-quotas slo dlo versioned_writes proxy-logging proxy-server
```

在`[filter:ceilometer]`部分加入以下內容：
```
[filter:ceilometer]
paste.filter_factory = ceilometermiddleware.swift:filter_factory
control_exchange = swift
url = rabbit://openstack:RABBIT_PASS@10.0.0.11:5672/
driver = messagingv2
topic = notifications
log_level = WARN
```
> 這邊`RABBIT_PASS`可以隨需求修改。

完成後就可以重新啟動 Swift Proxy 服務：
```sh
$ sudo swift-init all restart
```

# 驗證服務
導入該環境參數，來透過 Ceilometer client 查看服務狀態：
```sh
$ . admin-openrc
```

這邊可以透過 Ceilometer client 來查看所有 meter，如以下方式：
```sh
$ ceilometer meter-list
```

透過 Ceilometer client 來查看哪個期間的資料，如以下方式：
```sh
$ ceilometer statistics -m cpu_util -p 60
+--------+----------------------------+----------------------------+---------------+---------------+---------------+---------------+-------+----------+----------------------------+----------------------------+
| Period | Period Start               | Period End                 | Max           | Min           | Avg           | Sum           | Count | Duration | Duration Start             | Duration End               |
+--------+----------------------------+----------------------------+---------------+---------------+---------------+---------------+-------+----------+----------------------------+----------------------------+
| 60     | 2016-10-14T07:34:22.212000 | 2016-10-14T07:35:22.212000 | 4.27248211297 | 4.27248211297 | 4.27248211297 | 4.27248211297 | 1     | 0.0      | 2016-10-14T07:34:26.288000 | 2016-10-14T07:34:26.288000 |
+--------+----------------------------+----------------------------+---------------+---------------+---------------+---------------+-------+----------+----------------------------+----------------------------+
```
