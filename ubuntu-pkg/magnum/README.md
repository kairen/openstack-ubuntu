# Magnum 安裝與設定
OpenStack Magnum 是容器基礎架構管理服務，是一個 OpenStack 容器開發團隊所開發的 Container Orchestration Engines (COE)  API 服務，可以透過 OpenStack 部署 Kubernetes、Docker Swarm 與 Mesos 的容器資源。

- [安裝需求](#安裝需求)
- [安裝前準備](#安裝前準備)
- [套件安裝與設定](#套件安裝與設定)
- [驗證服務](#驗證服務)
    - [佈署 Kubernetes](#佈署-kubernetes)
- [問題解決](#問題解決)

### 安裝需求
在進行安裝 Magnum 之前，需要滿足以下服務才能完整的部署完成服務：
* Keystone --- Identity service
* Glance ----- Image service
* Nova ------- Compute service
* Neutron ---- Network service
* Cinder ----- Block Storage service
* Heat ------- Orchestration service

### 安裝前準備
在開始安裝前，要預先建立一個資料庫給 Magnum 儲存相關資訊，使用以下指令建立資料庫：
```sh
$ mysql -u root -p
```

透過以下命令用來更新現有帳號資料或建立 Magnum 資料庫：
```sql
CREATE DATABASE magnum;
GRANT ALL PRIVILEGES ON magnum.* TO 'magnum'@'localhost'  IDENTIFIED BY 'MAGNUM_DBPASS';
GRANT ALL PRIVILEGES ON magnum.* TO 'magnum'@'%'  IDENTIFIED BY 'MAGNUM_DBPASS';
```
> 這邊`MAGNUM_DBPASS`可以隨需求修改。

完成後離開資料庫，接著要建立 Service 與 API Endpoint，首先導入`admin`環境變數：
```sh
$ . admin-openrc
```

接著透過以下流程來建立 Magnum 的使用者、Service 以及 API Endpoint：
```sh
# 建立 Magnum User
$ openstack user create --domain default --password MAGNUM_PASS --email magnum@example.com magnum

# 新增 Magnum 到 Admin Role
$ openstack role add --project service --user magnum admin

# 建立 Magnum service
$ openstack service create --name magnum \
--description "Container Infrastructure Management Service" container-infra

# 建立 Magnum public endpoints
$ openstack endpoint create --region RegionOne \
container-infra public http://10.0.0.11:9511/v1

# 建立 Magnum internal endpoints
$ openstack endpoint create --region RegionOne \
container-infra internal http://10.0.0.11:9511/v1

# 建立 Magnum admin endpoints
$ openstack endpoint create --region RegionOne \
container-infra admin http://10.0.0.11:9511/v1

# 建立 Magnum 擁有者權限 Domain
$ openstack domain create --description \
"Owns users and projects created by magnum" magnum

# 建立 Magnum Domain 管理者
$ openstack user create --domain magnum \
--password MAGNUM_DOMAIN_PASS magnum_domain_admin

# 將 admin role 加入到 magnum_domain_admin
$ openstack role add --domain magnum --user magnum_domain_admin admin
```
> 這邊`MAGNUM_PASS`與`MAGNUM_DOMAIN_PASS`可以隨需求修改。

### 套件安裝與設定
首先透過 apt-get 來取得 OpenStack Magnum 相關元件，透過以下指令進行安裝：
```sh
$ DEBIAN_FRONTEND=noninteractive sudo apt-get install -y magnum-api magnum-conductor
```

安裝完成後，編輯`/etc/magnum/magnum.conf`設定檔，在`[DEFAULT]`部分加入以下內容：
```
[DEFAULT]
debug = True
transport_url = rabbit://openstack:RABBIT_PASS@10.0.0.11
auth_strategy = keystone
```
> 這邊`RABBIT_PASS`可以隨需求修改。

接下來，在`[database]`部分修改使用以下方式：
```
[database]
connection = mysql+pymysql://magnum:MAGNUM_DBPASS@10.0.0.11/magnum
```
> 這邊`MAGNUM_DBPASS`可以隨需求修改。

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
username = magnum
password = MAGNUM_PASS
```
> 這邊`MAGNUM_PASS`可以隨需求修改。

在`[api]`部分加入以下內容：
```
[api]
port = 9511
host = 10.0.0.11
```

在`[certificates]`部分加入以下內容：
```
[certificates]
cert_manager_type = local
storage_path = /var/lib/magnum/certificates/
```
> 若有安裝 Barbican 則修改成以下：
```
[certificates]
cert_manager_type = barbican
```

在`[cinder_client]`部分加入以下內容：
```
[cinder_client]
region_name = RegionOne
```

在`[trust]`部分加入以下內容：
```
[trust]
trustee_domain_name = magnum
trustee_domain_admin_name = magnum_domain_admin
trustee_domain_admin_password = MAGNUM_DOMAIN_PASS
```
> 這邊`MAGNUM_DOMAIN_PASS`可以隨需求修改。

在`[oslo_messaging_notifications]`部分加入以下內容：
```
[oslo_messaging_notifications]
driver = messaging
```

在`[oslo_concurrency]`部分加入以下內容：
```
[oslo_concurrency]
lock_path = /var/lib/magnum/tmp
```

完成所有設定後，即可同步資料庫來建立 Magnum 資料表：
```sh
$ sudo magnum-db-manage upgrade
```

建立憑證放置目錄：
```sh
$ sudo mkdir -p /var/lib/magnum/certificates/
$ sudo chown magnum:magnum -R /var/lib/magnum/certificates/
```

重新啟動所有 Magnum 服務：
```sh
sudo service magnum-api restart
sudo service magnum-conductor restart
```

### 驗證服務
首先接著導入`admin`帳號來驗證服務：
```sh
$ . admin-openrc
```

透過 Magnum client 來查看服務列表，如以下方式：
```sh
$ magnum service-list
+----+------+------------------+-------+----------+-----------------+---------------------------+---------------------------+
| id | host | binary           | state | disabled | disabled_reason | created_at                | updated_at                |
+----+------+------------------+-------+----------+-----------------+---------------------------+---------------------------+
| 1  | -    | magnum-conductor | up    |          | -               | 2016-12-30T03:13:06+00:00 | 2016-12-30T03:13:06+00:00 |
+----+------+------------------+-------+----------+-----------------+---------------------------+---------------------------+
```

接著下載將使用的 Magnum service 映像檔，以下是提供 k8s 與 docker swarm 使用：
```sh
$ wget https://fedorapeople.org/groups/magnum/fedora-atomic-latest.qcow2
```
> 其他版本可以參考 [Magnum image elements](https://fedorapeople.org/groups/magnum/)。

透過 OpenStack client 來上傳映像檔：
```sh
$ openstack image create \
--disk-format=qcow2 \
--container-format=bare \
--file=fedora-atomic-latest.qcow2 \
--property os_distro='fedora-atomic' \
fedora-atomic-latest
```

若 Mesos 則下載該映像檔：
```sh
$ wget https://fedorapeople.org/groups/magnum/ubuntu-mesos-latest.qcow2
```

透過 OpenStack client 來上傳映像檔：
```sh
$ openstack image create \
--disk-format=qcow2 \
--container-format=bare \
--file=ubuntu-mesos-latest.qcow2 \
--property os_distro='ubuntu' \
ubuntu-mesos
```

### 佈署 Kubernetes
首先透過 Magnum client 建立 template，來提供叢集板模：
```sh
$ magnum cluster-template-create --name k8s-template-latest \
--image fedora-atomic-latest \
--keypair KEY_ID \
--external-network ext-net \
--dns-nameserver 8.8.8.8 \
--flavor m1.small \
--docker-volume-size 5 \
--network-driver flannel \
--coe kubernetes
```
> 這邊`KEY_ID`請取代成自己個 Key Pair Name。`EXTERNAL_NETWORK`取代成 Provider Network Name。

成功的話，會看到類似以下結果：
```
+-----------------------+--------------------------------------+
| Property              | Value                                |
+-----------------------+--------------------------------------+
| insecure_registry     | -                                    |
| labels                | {}                                   |
| updated_at            | -                                    |
| floating_ip_enabled   | True                                 |
| fixed_subnet          | -                                    |
| master_flavor_id      | -                                    |
| uuid                  | cbed5751-6d7a-40a6-b651-32e8dbd13f2c |
| no_proxy              | -                                    |
| https_proxy           | -                                    |
| tls_disabled          | False                                |
| keypair_id            | mykey                                |
| public                | False                                |
| http_proxy            | -                                    |
| docker_volume_size    | 5                                    |
| server_type           | vm                                   |
| external_network_id   | ext-net                              |
| cluster_distro        | fedora-atomic                        |
| image_id              | fedora-atomic-newton                 |
| volume_driver         | -                                    |
| registry_enabled      | False                                |
| docker_storage_driver | devicemapper                         |
| apiserver_port        | -                                    |
| name                  | k8s-cluster-template                 |
| created_at            | 2016-12-30T06:34:08+00:00            |
| network_driver        | flannel                              |
| fixed_network         | -                                    |
| coe                   | kubernetes                           |
| flavor_id             | m1.small                             |
| master_lb_enabled     | False                                |
| dns_nameserver        | 8.8.8.8                              |
+-----------------------+--------------------------------------+
```

完成後，就可以建立實例的 bay 來部署叢集：
```sh
$ magnum cluster-create --name k8s-cluster-latest \
--cluster-template k8s-template-latest \
--master-count 1 \
--node-count 2
```
> 若要更新節點數可以用該指令：
```sh
$ magnum cluster-update k8s-cluster-latest replace node_count=2
```

> 範例操作還是看這邊比較快 [Developer Quick-Start](http://docs.openstack.org/developer/magnum/dev/dev-quickstart.html)。

### 問題解決
若遇到`<urlopen error [Errno 2] No such file or directory: '/usr/lib/python2.7/dist-packages/magnum/drivers/k8s_fedora_atomic_v1/templates/kubecluster.yaml'>`問題，可能是目前 Ubuntu .deb 安裝的版本 templates 遺失問題，可以透過以下方式解決：
```sh
$ git clone https://github.com/openstack/magnum.git -b stable/newton
$ cd magnum && sudo pip install .
$ sudo service magnum-api restart && sudo service magnum-conductor restart
```
