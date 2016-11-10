# Barbican 安裝與設定
本節將介紹如何在`Controller`節點上安裝與設定金鑰管理服務。

- [安裝前準備](#安裝前準備)
- [套件安裝與設定](#套件安裝與設定)
- [驗證服務](#驗證服務)

### 安裝前準備
在開始安裝前，要預先建立一個資料庫給 Barbican 儲存相關資訊，使用以下指令建立資料庫：
```sh
$ mysql -u root -p
```

透過以下命令用來更新現有帳號資料或建立 Barbican 資料庫：
```sql
CREATE DATABASE barbican;
GRANT ALL PRIVILEGES ON barbican.* TO 'barbican'@'localhost' IDENTIFIED BY 'BARBICAN_DBPASS';
GRANT ALL PRIVILEGES ON barbican.* TO 'barbican'@'%' IDENTIFIED BY 'BARBICAN_DBPASS';
```
> 這邊`BARBICAN_DBPASS`可以隨需求修改。

完成後離開資料庫，接著要建立 Service 與 API Endpoint，首先導入`admin`環境變數：
```sh
$ . admin-openrc
```

接著透過以下流程來建立 Barbican 的使用者、Service 以及 API Endpoint：
```sh
# 建立 Barbican user
$ openstack user create --domain default --password BARBICAN_PASS --email barbican@example.com barbican

# 新增 Barbican 到 Admin Role
$ openstack role add --project service --user barbican admin

# 建立名為 Creator 的 Role
$ openstack role create creator

# 新增 Barbican 到 Creator Role
$ openstack role add --project service --user barbican creator

# 建立 Barbican service
$ openstack service create --name barbican \
--description "Key Manager" key-manager

# 建立 Barbican public endpoints
$ openstack endpoint create --region RegionOne \
key-manager public http://10.0.0.11:9311/v1/%\(tenant_id\)s

# 建立 Barbican internal endpoints
$ openstack endpoint create --region RegionOne \
key-manager internal http://10.0.0.11:9311/v1/%\(tenant_id\)s

# 建立 Barbican admin endpoints
$ openstack endpoint create --region RegionOne \
key-manager admin http://10.0.0.11:9311/v1/%\(tenant_id\)s
```
> 這邊若`BARBICAN_PASS`要更改的話，可以更改。

### 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt-get install -y barbican-api barbican-common python-barbican
```

安裝完成後，編輯`/etc/barbican/barbican.conf`設定檔，在`[DEFAULT]`部分加入以下內容：
```
[DEFAULT]
...
rpc_backend = rabbit
db_auto_create = False

sql_connection = mysql+pymysql://barbican:BARBICAN_DBPASS@10.0.0.11/barbican
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
memcached_servers = 10.0.0.11:11211
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = barbican
password = BARBICAN_PASS
```
> 這邊`BARBICAN_PASS`可以隨需求修改。

接著編輯`/etc/barbican/barbican-api-paste.ini`，修改`[pipeline:barbican_api]`部分如下：
```
[pipeline:barbican_api]
pipeline = cors authtoken context apiapp
```

完成所有設定後，即可同步資料庫來建立 Barbican 資料表：
```sh
$ sudo barbican-manage db upgrade
```

最後重新啟動服務，由於 Barbican API 是使用 Apache2 來提供 HTTP 服務，故重新啟動 Apache2：
```sh
$ sudo service apache2 restart
```

Barbican 擁有一個插件架構，允許部署者能儲存秘鑰於多個不同的後端 secret stores。在預設下，Barbican 設定儲存秘鑰於一個基本基於檔案的金鑰庫中。然而這種設定對於生產環境使用是不安全的，因此需要選擇其他方式進行。

一些支援的插件可以參考 [Secret Store Back-ends](http://docs.openstack.org/project-install-guide/key-manager/newton/barbican-backend.html#barbican-backend)。

# 驗證服務
首先回到`Controller`節點並接著導入`admin`帳號來驗證服務：
```sh
$ . admin-openrc
```

使用 OpenStack CLI 建立儲存一個 secret：
```sh
$ openstack secret store --name mysecret --payload j4=]d21
+---------------+-----------------------------------------------------------------------+
| Field         | Value                                                                 |
+---------------+-----------------------------------------------------------------------+
| Secret href   | http://10.0.0.11:9311/v1/secrets/651d7f30-c11a-49d9-a0f1-34cd313a36fa |
| Name          | mysecret                                                              |
| Created       | None                                                                  |
| Status        | None                                                                  |
| Content types | None                                                                  |
| Algorithm     | aes                                                                   |
| Bit length    | 256                                                                   |
| Secret type   | opaque                                                                |
| Mode          | cbc                                                                   |
| Expiration    | None                                                                  |
+---------------+-----------------------------------------------------------------------+
```

檢索 secret 中儲存的 Payload：
```sh
$ openstack secret get http://10.0.0.11:9311/v1/secrets/651d7f30-c11a-49d9-a0f1-34cd313a36fa --payload
+---------+---------+
| Field   | Value   |
+---------+---------+
| Payload | j4=]d21 |
+---------+---------+
```
