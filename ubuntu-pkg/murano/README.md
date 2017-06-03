# Murano 安裝與設定
Murano 是 OpenStack 的專案之一，它將許多應用程式與多功能工具進行封裝，形成應用程式目錄，透過目錄方式來快速提交應用程式，讓IT人員只需要點擊設定參數就可以快速部署應用程式。

- [安裝前準備](#安裝前準備)
- [套件安裝與設定](#套件安裝與設定)
- [驗證服務](#驗證服務)

### 安裝需求
在進行安裝 Murano 之前，需要滿足以下服務才能完整的部署完成服務：
* Keystone --- Identity service
* Glance ----- Image service
* Nova ------- Compute service
* Neutron ---- Network service
* Cinder ----- Block Storage service
* Heat ------- Orchestration service

### 安裝前準備
在開始安裝前，要預先建立一個資料庫給 Murano 儲存相關資訊，使用以下指令建立資料庫：
```sh
$ mysql -u root -p
```

透過以下命令用來更新現有帳號資料或建立 Murano 資料庫：
```sql
CREATE DATABASE murano;
GRANT ALL PRIVILEGES ON murano.* TO 'murano'@'localhost'  IDENTIFIED BY ' MURANO_DBPASS';
GRANT ALL PRIVILEGES ON murano.* TO 'murano'@'%'  IDENTIFIED BY 'MURANO_DBPASS';
```
> 這邊`MURANO_DBPASS`可以隨需求修改。

完成後離開資料庫，接著要建立 Service 與 API Endpoint，首先導入`admin`環境變數：
```sh
$ . admin-openrc
```

接著透過以下流程來建立 Murano 的使用者、Service 以及 API Endpoint：
```sh
# 建立 Murano user
$ openstack user create --domain default --password MURANO_PASS --email murano@example.com murano

# 新增 Murano user 到 Admin Role
$ openstack role add --project service --user murano admin

# 建立 Murano service
$ openstack service create --name murano \
--description "Application Catalog" application-catalog

# 建立 Murano public endpoints
$ openstack endpoint create --region RegionOne \
application-catalog public http://10.0.0.11:8082

# 建立 Murano internal endpoints
$ openstack endpoint create --region RegionOne \
application-catalog internal http://10.0.0.11:8082

# 建立 Murano admin endpoints
$ openstack endpoint create --region RegionOne \
application-catalog admin http://10.0.0.11:8082
```
> 這邊`MURANO_PASS`可以隨需求修改。

### 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ DEBIAN_FRONTEND=noninteractive sudo apt-get install murano-api murano-engine python-muranoclient
```

安裝完成後，編輯`/etc/murano/murano.conf`設定檔，在`[DEFAULT]`部分加入以下內容：
```
[DEFAULT]
home_region = RegionOne
use_syslog = False
debug = True

transport_url = rabbit://openstack:RABBIT_PASS@10.0.0.11
auth_strategy = keystone
```
> 這邊`RABBIT_PASS`可以隨需求修改。

接下來，在`[database]`部分修改使用以下方式：
```
[database]
connection = mysql+pymysql://murano:MURANO_DBPASS@10.0.0.11/murano
```
> 這邊`MURANO_DBPASS`可以隨需求修改。

在`[keystone_authtoken]`部分加入以下內容：
```
[keystone_authtoken]
memcached_servers = 10.0.0.11:11211
auth_version = v3
auth_url = http://10.0.0.11:35357
auth_uri = http://10.0.0.11:5000/v3
project_domain_id = default
project_name = service
user_domain_id = default
auth_type = password
username = murano
password = MURANO_PASS
```
> 這邊`MURANO_PASS`可以隨需求修改。

在`[engine]`部分加入以下內容：
```
[engine]
enable_model_policy_enforcer = False
```

在`[murano]`部分加入以下內容：
```
[murano]
url = http://127.0.0.1:8082
```

在`[networking]`部分加入以下內容：
```
[networking]
create_router = true
external_network = EXTERNAL_NETWORK
```
> 這邊`EXTERNAL_NETWORK`請取代成環境的 Provider 網路。

在`[oslo_messaging_notifications]`部分加入以下內容：
```
[oslo_messaging_notifications]
driver = messaging
```

在`[oslo_policy]`部分加入以下內容：
```
[oslo_policy]
policy_file = /etc/murano/policy.json
```

在`[oslo_concurrency]`部分加入以下內容：
```
[oslo_concurrency]
lock_path = /var/lib/murano/tmp
```

完成所有設定後，即可同步資料庫來建立 Murano 資料表：
```
$ sudo murano-db-manage upgrade
```

重新啟動所有 Murano 服務：
```sh
sudo service murano-api restart
sudo service murano-engine restart
```

# 驗證服務
首先回到`Controller`節點並接著導入`admin`帳號來驗證服務：
```sh
$ . admin-openrc
```

下載將使用的 Murano meta core 套件，並上傳核心套件：
```sh
$ git clone https://github.com/openstack/murano.git -b stable/newton
$ cd murano
$ pushd ./meta/io.murano
$ zip -r ../../io.murano.zip *
$ popd && murano package-import --is-public io.murano.zip
```

透過 Murano client 來查看服務列表，如以下方式：
```sh
$ murano package-list
```
