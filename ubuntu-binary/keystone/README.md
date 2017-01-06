# Keystone 安裝與設定
OpenStack 的  Keystone 提供了身份驗證服務，透過單點整合用於管理驗證、授權與服務目錄，來提升服務的使用安全。

- [Keystone 安裝前準備](#安裝前準備)
- [套件安裝與設定](#套件安裝與設定)
- [設定 Apache HTTP 伺服器](#設定-apache-http-伺服器)
- [建立 Service 與 API Endpoint](#建立-service-與-api-endpoint)
- [建立 Keystone demo user](#建立-keystone-demo-user)
- [驗證服務](#驗證服務)
- [使用腳本切換使用者](#使用腳本切換使用者)

> <font color=red> 提醒! </font>以下操作將一律在 Controller 進行。

### 安裝前準備
在開始安裝前，要預先建立一個資料庫給 Keystone 儲存相關資訊，使用以下指令建立資料庫：
```sh
$ mysql -u root -p
```
> `P.S.` 若無法登入的話，請進入 root 底下執行該指令修改：
```sh
$ mysql -u root
> use mysql;
> update user set password=PASSWORD("passwd") where User='root';
> GRANT ALL PRIVILEGES ON root.* TO 'root'@'localhost' IDENTIFIED BY 'passwd';
```

透過以下命令用來更新現有帳號資料或建立 Keystone 資料庫：
```sql
CREATE DATABASE keystone;
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' IDENTIFIED BY 'KEYSTONE_DBPASS';
GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' IDENTIFIED BY 'KEYSTONE_DBPASS';
```
> 這邊```KEYSTONE_DBPASS```可以隨需求修改。

### 套件安裝與設定
由於 Eventlet 將在後續版本被棄用，所以在安裝 Keystone 之前要手動設定服務在安裝完與開機時不自動啟動：
```sh
$ echo "manual" | sudo tee /etc/init/keystone.override
manual
```
> 在 Kilo 與 Liberty 版本中， Keystone 專案棄用了 Eventlet Networking，因此本教學將使用 Apache2 mod_wsgi 來提供身份證認服務，這邊預設下將佔用到 5000 與 35357 Port。因此在安裝 Keystone 時將停用原生服務。而在 Mitaka 版本將正式移除 Eventlet。

確認關閉自動啟動 Keystone 後，就可以直接安裝套件，透過以下指令安裝：
```sh
$ sudo apt-get install keystone apache2 libapache2-mod-wsgi
```

安裝完後，編輯`/etc/keystone/keystone.conf`設定檔，在`[DEFAULT]`部分修改使用以下方式：
```
[DEFAULT]
admin_token = 21d7fb48086e09f30d40be5a5e95a7196f2052b2cae6b491
```

在`[database]`部分修改使用以下方式：
```
[database]
# connection = sqlite:////var/lib/keystone/keystone.db
connection = mysql+pymysql://keystone:KEYSTONE_DBPASS@10.0.0.11/keystone
```

在`[memcache]`部分加入以下內容：
```
[memcache]
servers = 10.0.0.11:11211
```

在`[token]`部分加入以下內容：
```
[token]
provider = fernet
```

完成後，透過 Keystone 管理指令來同步資料庫建立資料表：
```sh
$ sudo keystone-manage db_sync
```

接著初始化 fernet keys：
```sh
$ sudo keystone-manage fernet_setup --keystone-user keystone --keystone-group keystone
2016-10-07 09:21:28.824 4953 INFO keystone.common.fernet_utils [-] key_repository does not appear to exist; attempting to create it
...

$ sudo keystone-manage credential_setup --keystone-user keystone --keystone-group keystone
2016-10-07 09:21:44.854 4963 INFO keystone.common.fernet_utils [-] key_repository does not appear to exist; attempting to create it
...
```

最後引導身份認證服務：
```sh
$ sudo keystone-manage bootstrap --bootstrap-password passwd \
--bootstrap-admin-url http://10.0.0.11:35357/v3/ \
--bootstrap-internal-url http://10.0.0.11:35357/v3/ \
--bootstrap-public-url http://10.0.0.11:5000/v3/ \
--bootstrap-region-id RegionOne
```
> 這邊`ADMIN_PASS`可以隨需求修改。

### 設定 Apache HTTP 伺服器
由於本教學採用 WSGI Mod 來提供 Keystone 服務，因此我們將使用到 Apache2 來建立 HTTP 服務，首先編`/etc/apache2/apache2.conf` 加入以下內容：
```
ServerName 10.0.0.11
```

接著建立一個設定檔`/etc/apache2/sites-available/keystone.conf`來提供 Keystone 服務，並加入以下內容（預設安裝即存在）：
```
Listen 5000
Listen 35357

<VirtualHost *:5000>
    WSGIScriptAlias / /usr/bin/keystone-wsgi-public
    WSGIDaemonProcess keystone-public processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-public
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    LimitRequestBody 114688

    <IfVersion >= 2.4>
      ErrorLogFormat "%{cu}t %M"
    </IfVersion>

    ErrorLog /var/log/apache2/keystone.log
    CustomLog /var/log/apache2/keystone_access.log combine

    <Directory /usr/bin>
        <IfVersion >= 2.4>
            Require all granted
        </IfVersion>
        <IfVersion < 2.4>
            Order allow,deny
            Allow from all
        </IfVersion>
    </Directory>
</VirtualHost>

<VirtualHost *:35357>
    WSGIScriptAlias / /usr/bin/keystone-wsgi-admin
    WSGIDaemonProcess keystone-admin processes=5 threads=1 user=keystone group=keystone display-name=%{GROUP}
    WSGIProcessGroup keystone-admin
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
    LimitRequestBody 114688

    <IfVersion >= 2.4>
      ErrorLogFormat "%{cu}t %M"
    </IfVersion>

    ErrorLog /var/log/apache2/keystone.log
    CustomLog /var/log/apache2/keystone_access.log combined

    <Directory /usr/bin>
        <IfVersion >= 2.4>
            Require all granted
        </IfVersion>
        <IfVersion < 2.4>
            Order allow,deny
            Allow from all
        </IfVersion>
    </Directory>
</VirtualHost>

Alias /identity /usr/bin/keystone-wsgi-public
<Location /identity>
    SetHandler wsgi-script
    Options +ExecCGI

    WSGIProcessGroup keystone-public
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
</Location>

Alias /identity_admin /usr/bin/keystone-wsgi-admin
<Location /identity_admin>
    SetHandler wsgi-script
    Options +ExecCGI

    WSGIProcessGroup keystone-admin
    WSGIApplicationGroup %{GLOBAL}
    WSGIPassAuthorization On
</Location>
```
> `P.S.` 在新版本中安裝 Apache2 時，會自動建立一個擁有以上內容的檔案，我們只需要透過 Apache2 enable 即可。

確認檔案無誤後，就可以讓 Apache2 使用這個設定檔，透過以下指令：
```sh
$ sudo ln -s /etc/apache2/sites-available/keystone.conf /etc/apache2/sites-enabled
```

完成後就可以重新啟動 Apache2 伺服器：
```sh
$ sudo service apache2 restart
```

最後由於第一次安裝時，預設會建立一個 SQLite 資料庫，因此這邊可以將之刪除：
```sh
$ sudo rm -f /var/lib/keystone/keystone.db
```

### 建立 Service 與 API Endpoint
在建立 Keystone service 與 Endpoint 之前，要先導入一些環境變數，由於 OpenStack client 會自動抓取系統某些環境變數來提供 API 的存取，首先透過以下指令導入：
```sh
export OS_USERNAME=admin
export OS_PASSWORD=passwd
export OS_PROJECT_NAME=admin
export OS_USER_DOMAIN_NAME=Default
export OS_PROJECT_DOMAIN_NAME=Default
export OS_AUTH_URL=http://10.0.0.11:35357/v3
export OS_IDENTITY_API_VERSION=3
```

完後上述後，就可以建立 Service 實體來提供身份認證：
```sh
$ openstack project create --domain default \
--description "Service Project" service
```

成功的話，會看到類似以下結果：
```
+-------------+----------------------------------+
| Field       | Value                            |
+-------------+----------------------------------+
| description | Service Project                  |
| domain_id   | default                          |
| enabled     | True                             |
| id          | abf889b4e323489687bf3f00afe9ab62 |
| is_domain   | False                            |
| name        | service                          |
| parent_id   | default                          |
+-------------+----------------------------------+
```

在後續安裝的 OpenStack 各套件服務都需要建立一個或多個 Service，以及 API Endpoint 目錄。
> `P.S.` 在 Newton 版本中，Keystone 一些操作被簡化了，但是其概念大致不變。

### 建立 Keystone demo user
身份認證服務會透過 domains、projects、roles 與 Users 組合來進行授權動作。由於 Newton 版本將 admin 在 bootstrap 的時候建立完成了，因此這邊說明建立一個 demo project：
```sh
# 建立 demo project
$ openstack project create --domain default \
--description "Demo Project" demo

# 建立 demo User
$ openstack user create --domain default \
--password-prompt demo

# 建立 admin Role
$ openstack role create user

# 加入 Project 與 User 到 admin Role
$ openstack role add --project demo --user demo user
```
> 以上指令可以重複的建立不同的 Project 與 User。

### 驗證服務
在進行其他服務安裝之前，一定要確認 Keystone 服務沒有任何錯誤，首先取消上面導入的環境變數：
```sh
unset OS_URL
```

首先透過以下指令來驗證 admin 可以登入取得當前 token：
```sh
$ openstack --os-auth-url http://10.0.0.11:35357/v3 \
--os-project-domain-name default \
--os-user-domain-name default \
--os-project-name admin \
--os-username admin \
token issue
```
> 其中 `default` 是當沒有指定 Domain 時的預設名稱。

成功的話，會看到類似以下結果：
```
+------------+----------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                              |
+------------+----------------------------------------------------------------------------------------------------+
| expires    | 2016-10-07 10:56:30+00:00                                                                          |
| id         | gAAAAABX93FOv_42CdkIDVr8GoKc3r0ChhA3Rtc37RI-                                                       |
|            | znsCRVucCiU6JxZApUGeEdMYXUVr_VbrvgEbepGH6D3lxHeFY8_Wct8Vc4pgIGqGSKjlAv2K4-L6H-                     |
|            | UraX2JKp6cVbe8jytiEkshzY7G5NmoTwaiCAYz-obBZ6tte7_HykpLVlfNbtE                                      |
| project_id | ba57e6880afe403dbf78f946766df5e0                                                                   |
| user_id    | 15d7a1ff307244c6836d16e2967ca5f9                                                                   |
+------------+----------------------------------------------------------------------------------------------------+
```

接著驗證是否可以使用 admin 的 API 來取得所有使用者列表：
```sh
$ openstack --os-auth-url http://10.0.0.11:35357/v3 \
--os-project-domain-name default \
--os-user-domain-name default \
--os-project-name admin \
--os-username admin \
project list
```

成功的話，會看到類似以下結果：
```
+----------------------------------+---------+
| ID                               | Name    |
+----------------------------------+---------+
| 6b3472aa28284861b46dbeddfc91ff60 | demo    |
| abf889b4e323489687bf3f00afe9ab62 | service |
| ba57e6880afe403dbf78f946766df5e0 | admin   |
+----------------------------------+---------+
```

然後再透 `demo` 使用者來驗證是否有存取權限，這邊利用 v3 來取得 Token：
```sh
$ openstack --os-auth-url http://10.0.0.11:5000/v3 \
--os-project-domain-name default \
--os-user-domain-name default \
--os-project-name demo \
--os-username demo \
token issue
```
> 本教學範例密碼為`demo`。

成功的話，會看到類似以下結果：
```
+------------+----------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                              |
+------------+----------------------------------------------------------------------------------------------------+
| expires    | 2016-10-07 15:56:30+00:00                                                                          |
| id         | gAAAAABX93FOv_42CdkIDVr8GoKc3r0ChhA3Rtc37RI-                                                       |
|            | znsCRVucCiU6JxZApUGeEdMYXUVr_VbrvgEbepGH6D3lxHeFY8_Wct8Vc4pgIGqGSKjlAv2K4-L6H-                     |
|            | UraX2JKp6cVbe8jytiEkshzY7G5NmoTwaiCAYz-obBZ6tte7_HykpLVlfNbtE                                      |
| project_id | ba57e6880afe403dbf78f946766df5e0                                                                   |
| user_id    | 15d7a1ff307244c6836d16e2967ca5f9                                                                   |
+------------+----------------------------------------------------------------------------------------------------+
```

> P.S 這邊會發現使用的 Port 從 35357 轉換成 5000，這邊只是為了區別 Admin URL 與 Public URL 中使用的 Port。

最後再透過 `demo` 來使用擁有管理者權限的 API：
```sh
$ openstack --os-auth-url http://10.0.0.11:5000/v3 \
--os-project-domain-name default \
--os-user-domain-name default \
--os-project-name demo \
--os-username demo \
user list
```

成功的話，會看到類似以下結果：
```
You are not authorized to perform the requested action: identity:list_users (HTTP 403)
```

若上述過程都沒有錯誤，表示 Keystone 目前很正常的被執行中。

### 使用腳本切換使用者
由於後續安裝可能會切換不同使用者來驗證一些服務，因此可以透過建立腳本來導入相關的環境變數，來達到不同使用者的切換，首先建立以下兩個檔案：
```sh
$ touch admin-openrc demo-openrc
```

編輯 `admin-openrc` 加入以下內容：
```sh
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=admin
export OS_USERNAME=admin
export OS_PASSWORD=passwd
export OS_AUTH_URL=http://10.0.0.11:35357/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

編輯 `demo-openrc` 加入以下內容：
```sh
export OS_PROJECT_DOMAIN_NAME=default
export OS_USER_DOMAIN_NAME=default
export OS_PROJECT_NAME=demo
export OS_USERNAME=demo
export OS_PASSWORD=demo
export OS_AUTH_URL=http://10.0.0.11:5000/v3
export OS_IDENTITY_API_VERSION=3
export OS_IMAGE_API_VERSION=2
```

完成後，可以透過 Linux 指令來執行檔案導入環境變數，如以下指令：
```sh
$ source admin-openrc
```
> 也可以使用以下方式來執行檔案：
```sh
$ . admin-openrc
```

完成後，在使用 OpenStack client 就可以省略一些基本參數了，如以下指令：
```sh
$ openstack token issue
+------------+----------------------------------------------------------------------------------------------------+
| Field      | Value                                                                                              |
+------------+----------------------------------------------------------------------------------------------------+
| expires    | 2016-10-07 11:10:08+00:00                                                                          |
| id         | gAAAAABX93SAMqmfwMFe2Ke0ASQV0_zVqrQ6VkvTh9tk75LHJeKJ79TTKGxOL6E7rdwrk8UpSaJCNNb-xzBxp2mrwUTOJ1CZbv |
|            | Hzz6MpijVizXrXZyNjhe1IfoKUvVjBNlIQOrjVE0HYRHOKA2DmpXXHnWaGzX_dipJ3oJb0bqf3HbNoscZRnkk              |
| project_id | ba57e6880afe403dbf78f946766df5e0                                                                   |
| user_id    | 15d7a1ff307244c6836d16e2967ca5f9                                                                   |
+------------+----------------------------------------------------------------------------------------------------+
```
