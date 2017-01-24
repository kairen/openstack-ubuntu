# OpenStack Kolla 部署
Kolla 提供 OpenStack 生產環境就緒的容器與部署工具，其具備快速、擴展、可靠等特性，並提供社群版本的最佳本版升級實現。

- [節點配置](#節點配置)
- [事前準備](#安裝-openStack-套件)
    - [部署 Docker Registry](#部署docker-registry)
    - [部署節點準備](#部署節點準備)
- [建立 OpenStack Kolla Docker 映像檔](#建立openstack-kolla-docker-映像檔)
- [部署 OpenStack 節點](#部署-openstack-節點)
- [驗證服務](#驗證服務)

## 節點配置
本安裝將使用三台實體主機與一台虛擬機器來進行簡單叢集部署，主機規格如以下所示：

|Role           |RAM   |CPUs |Disk  |IP Address |
|---------------|------|-----|------|-----------|
|controller1    | 8 GB |4vCPU|250 GB|10.0.0.11  |
|network1       | 2 GB |1vCPU|250 GB|10.0.0.21  |
|compute1       | 8 GB |8vCPU|500 GB|10.0.0.31  |
|docker-registry| 2 GB |1vCPU|50 GB |10.0.0.90 |

> 作業系統皆為`Ubuntu Server 14.04`。

另外每台實體主機的網卡網路分別為以下：

|Role       | Management | Tunnel | Public |
|-----------|------------|--------|--------|
|controller1|    eth0    |  eth1  |  eth2  |
|network1   |    eth0    |  eth1  |  eth2  |
|compute1   |    eth0    |  eth1  |  eth2  |

網卡若是實體主機，請設定為固定 IP，如以下：
```
auto eth0
iface eth0 inet static
       	address 10.0.0.11
       	netmask	255.255.255.0
       	gateway	10.0.0.1
       	dns-nameservers 8.8.8.8
```
> 若想修改主機的網卡名稱，可以編輯`/etc/udev/rules.d/70-persistent-net.rules`。

其中`network`節點的 Public 需設定網卡為以下：
```
auto <ethx>
iface <ethx> inet manual
        up ip link set dev $IFACE up
        down ip link set dev $IFACE down
```

## 事前準備
在開始部署 OpenStack 之前，我們需要先將一些相依軟體與函式庫安裝完成，並且需要部署一用於存取映像檔的倉庫(Docker registry)。

### 部署 Docker Registry
首先進入到 `docker-registry` 節點，本教學採用一台虛擬機作為部署使用，並安裝 Docker engine：
```sh
$ curl -fsSL https://get.docker.com/ | sh
```

安裝完成 Docker engine 後，透過以下指令建置 Docker registry：
```sh
$ docker run -d -p 5000:5000 --restart=always --name registry \
-v $(pwd)/data:/var/lib/registry \
registry:2
```

接著為了方便檢視 Docker image，這邊另外部署 Docker registry UI：
```sh
$ docker run -d -p 8080:80 \
-e ENV_DOCKER_REGISTRY_HOST=10.26.1.49 \
-e ENV_DOCKER_REGISTRY_PORT=5000 \
konradkleine/docker-registry-frontend:v2
```

完成以上即可透過瀏覽器查看該主機 `8080` Port。也可以透過以下指令檢查是否部署成功：
```sh
$ docker pull ubuntu:14.04
$ docker tag ubuntu:14.04 localhost:5000/ubuntu:14.04
$ docker push localhost:5000/ubuntu:14.04

The push refers to a repository [localhost:5000/ubuntu]
447f88c8358f: Pushed
df9a135a6949: Pushed
...
```
> 其他 Docker registry 列表：
> * [Portus](https://github.com/SUSE/Portus)
> * [Atomic Registry](http://www.projectatomic.io/registry/)
> * [Private Registries in RancherOS](https://docs.rancher.com/os/configuration/private-registries/)

### 部署節點準備
當完成上述後，即可進行部署節點的相依軟體安裝，首先在`每台節點`透過以下指令更新與安裝一些套件：
```sh
$ sudo apt-get update -y && sudo apt-get upgrade -y
$ sudo apt-get install -y python-pip python-dev
$ curl -fsSL https://get.docker.com/ | sh
$ sudo timedatectl set-timezone Asia/Taipei
```
> 由於使用`Ubuntu server 14.04`，故需要更新 Kernel:
```sh
$ sudo apt-get install -y linux-image-generic-lts-wily
```

接著更新 pip，並安裝 docker-python 函式庫：
```sh
$ sudo pip install -U pip docker-py
```

編輯每台節點的`/etc/default/docker`檔案，加入以下內容：
```
DOCKER_OPTS="--insecure-registry <registry-ip>:5000"
```
> > 這邊`{registry-ip}`為 Docker registry 的 IP 位址。若使用 Ubuntu Server 16.04 版本的話，需要編輯`/lib/systemd/system/docker.service`檔案，修改以下：
```
EnvironmentFile=-/etc/default/%p
ExecStart=/usr/bin/dockerd -H fd:// \
                           $DOCKER_OPTS
```
> 完成上述後，需要 reload service：
```sh
$ sudo systemctl daemon-reload
```

接著編輯每台節點的`/etc/rc.local`檔案，加入以下內容：
```
mount --make-shared /run

# 在 compute 節點額外加入
mount --make-shared /var/lib/nova/mnt
```

接著在`compute1`節點執行以下指令：
```sh
sudo mkdir -p /var/lib/nova/mnt /var/lib/nova/mnt1
sudo mount --bind /var/lib/nova/mnt1 /var/lib/nova/mnt
sudo mount --make-shared /var/lib/nova/mnt
```

完成後`重新啟動每台節點`。

當上述步驟都完成後，進入到`controller1`的`root`使用者執行以下指令安裝與設定額外套件：
```sh
$ sudo apt-get install -y software-properties-common
$ sudo apt-add-repository -y ppa:ansible/ansible && sudo apt-get update
$ sudo apt-get install -y ansible libffi-dev libssl-dev gcc git
```
> NTP 部分可自行設定，這邊不再說明。

接著繼續在`controller1`節點的`root`使用者執行以下指令：
```sh
$ ssh-keygen -t rsa
$ cat .ssh/id_rsa.pub >> .ssh/authorized_keys
```

複製`controller1`節點的`root`底下的`.ssh/id_rsa.pub`公有金鑰到其他節點的`root`使用者底下的`.ssh/authorized_keys`。

## 建立 OpenStack Kolla Docker 映像檔
在使用 Kolla 部署 OpenStack 叢集之前，我們需要預先建立用於部署的映像檔，才能進行節點的部署。首先進入到`controller1`節點，並下載 Kolla 專案：
```sh
$ git clone https://github.com/openstack/kolla.git -b stable/newton
$ cd kolla && pip install .
$ pip install tox python-openstackclient && tox -e genconfig
$ cp -r etc/kolla /etc/

## 安裝 kolla-ansible
$ git clone https://github.com/openstack/kolla-ansible.git
$ cd kolla-ansible/ && pip install .
```

接著執行指令進行建立 Docker 映像檔，若不指定名稱預設下將建立全部映像檔，如以下：
```sh
$ kolla-build --base ubuntu --type source --registry {registry-ip}:5000 --push
```
> 這邊`{registry-ip}`為 Docker registry 的 IP 位址。

> 若要改變 OS 與版本，可以編輯`/etc/kolla/kolla-build.conf`檔案修改以下內容：
```
base = centos
base_tag = 3.0.3
push = true
install_type = rdo
registry = {registry-ip}:5000
```
> 可參考官方文件 [Building Container Images](http://docs.openstack.org/developer/kolla/image-building.html)。

當映像檔建置完成後，即可查看`docker-regisrtry`節點的 UI 介面，來確認是否上傳成功。

## 部署 OpenStack 節點
當完成映像檔建立後，即可開始進行部署 OpenStack 節點。

首先到`controller1`進入剛下載的`kolla-ansible`專案目錄，並編輯`ansible/inventory/multinode`檔案，修改以下內容：
```sh
[control]
controller1

[network]
network1

[compute]
compute1

[storage]
controller1
```
> 這邊由於機器不夠，所以將 storage 也放到 controller。

接著編輯`/etc/kolla/globals.yml`檔案，修改以下內容：
```yml
config_strategy: "COPY_ALWAYS"
kolla_base_distro: "ubuntu"
kolla_install_type: "source"
openstack_release: "3.0.3"
kolla_internal_vip_address: "10.0.0.10"
docker_registry: "10.0.0.90:5000"
network_interface: "eth0"
tunnel_interface: "eth1"
neutron_external_interface: "eth2"
neutron_plugin_agent: "openvswitch"

# OpenStack service
enable_cinder: "yes"
enable_neutron_lbaas: "yes"
```
> 若要將 API 開放給 Public network 存取的話，需要設定以下內容：
```
kolla_external_vip_address: "10.26.1.251"
kolla_external_vip_interface: "eth3"
```

(Option)建立 Cinder LVM volume group:
```sh
$ sudo apt-get install lvm2 -y
$ sudo pvcreate /dev/sdb
$ sudo vgcreate cinder-volumes /dev/sdb
$ sudo  pvdisplay

--- Physical volume ---
PV Name               /dev/sdb
VG Name               cinder-volumes
PV Size               465.76 GiB / not usable 4.02 MiB
Allocatable           yes
PE Size               4.00 MiB
Total PE              119234
Free PE               119234
Allocated PE          0
PV UUID               5g01hz-Ebds-EVSF-yGm4-evNm-5Kfp-2h8kUR
```

然後透過以下指令產生亂數密碼：
```sh
$ kolla-genpwd
```

接著要執行預先檢查確認節點是否可以進行部署，透過以下指令：
```sh
$ kolla-ansible prechecks -i ansible/inventory/multinode

PLAY RECAP *********************************************************************
compute1                   : ok=8    changed=0    unreachable=0    failed=0
controller1                : ok=35   changed=0    unreachable=0    failed=0
network1                   : ok=29   changed=0    unreachable=0    failed=0
```
> 若中途發生錯誤，請檢查錯誤訊息。

確認沒問題後即可部署 OpenStack，透過以下指令進行：
```sh
$ kolla-ansible deploy -i ansible/inventory/multinode
PLAY RECAP *********************************************************************
compute1                   : ok=44   changed=31   unreachable=0    failed=0
controller1                : ok=154  changed=96   unreachable=0    failed=0
network1                   : ok=37   changed=28   unreachable=0    failed=0
```
> P.S. 若發生映像檔無法下載，請自行透過以下指令傳到該節點：
```sh
$ docker save {image} > image.tar
$ scp image.tar network1:~/
$ ssh network1 "docker load < image.tar"
```

## 驗證服務
完成後，就可以產生 Credential 檔案來進行系統驗證：
```sh
$ kolla-ansible post-deploy
$ source /etc/kolla/admin-openrc.sh
```

透過 OpenStack Client 來檢查 Keystone，並查看所有使用者：
```sh
$ openstack user list

+----------------------------------+-------------------+
| ID                               | Name              |
+----------------------------------+-------------------+
| 19af62000d334c4d9de3aa29b9fd06df | heat_domain_admin |
| 83fa8ba1deff46e58e716470dbdaeb03 | heat              |
| 9661e2a4b1f648a6a3ed5c52f22c52bb | nova              |
| 9fa2a71716d046a893dcbaef3d7375ce | admin             |
| acdfdc4fae2248b0af4c8788a9858cf3 | glance            |
| b683b3c611dd4bdba2c25321a0f03cf0 | cinder            |
| dc57ebf4e6c548e990767f00ffda19d8 | neutron           |
+----------------------------------+-------------------+
```

透過 OpenStack Client 檢查 Nova 的服務列表：
```sh
$ openstack compute service list

+----+------------------+-------------+----------+---------+-------+----------------------------+-----------------+
| Id | Binary           | Host        | Zone     | Status  | State | Updated_at                 | Disabled Reason |
+----+------------------+-------------+----------+---------+-------+----------------------------+-----------------+
| 3  | nova-consoleauth | controller1 | internal | enabled | up    | 2016-08-16T05:40:16.000000 | -               |
| 5  | nova-scheduler   | controller1 | internal | enabled | up    | 2016-08-16T05:40:20.000000 | -               |
| 6  | nova-conductor   | controller1 | internal | enabled | up    | 2016-08-16T05:40:12.000000 | -               |
| 10 | nova-compute     | compute1    | nova     | enabled | up    | 2016-08-16T05:40:16.000000 | -               |
+----+------------------+-------------+----------+---------+-------+----------------------------+-----------------+
```

透過 OpenStack Client 檢查 Neutron 的服務列表：
```sh
$ openstack network agent list
+------------------+------------------+----------+-------------------+-------+----------------+------------------+
| id               | agent_type       | host     | availability_zone | alive | admin_state_up | binary           |
+------------------+------------------+----------+-------------------+-------+----------------+------------------+
| 2254dcb0-9d5b-   | Open vSwitch     | network1 |                   | :-)   | True           | neutron-         |
| 46da-b1f2-b03719 | agent            |          |                   |       |                | openvswitch-     |
| 5913b1           |                  |          |                   |       |                | agent            |
| acb81021-2efd-4b | DHCP agent       | network1 | nova              | :-)   | True           | neutron-dhcp-    |
| b6-9af7-a3df95ce |                  |          |                   |       |                | agent            |
| 479b             |                  |          |                   |       |                |                  |
| b564c005-779d-   | Metadata agent   | network1 |                   | :-)   | True           | neutron-         |
| 45f4-b10f-       |                  |          |                   |       |                | metadata-agent   |
| 788bb788491b     |                  |          |                   |       |                |                  |
| d95d75e5-6429-44 | Open vSwitch     | compute1 |                   | :-)   | True           | neutron-         |
| 65-aef1-a6a9f4e7 | agent            |          |                   |       |                | openvswitch-     |
| 995f             |                  |          |                   |       |                | agent            |
| e0dbbff4-d7a7-4f | L3 agent         | network1 | nova              | :-)   | True           | neutron-l3-agent |
| a2-8fa2-b3d3de42 |                  |          |                   |       |                |                  |
| d899             |                  |          |                   |       |                |                  |
+------------------+------------------+----------+-------------------+-------+----------------+------------------+
```

從網路上下載一個測試用映像檔`cirros-0.3.4-x86_64`，並上傳至 Glance 服務：
```sh
$ wget http://download.cirros-cloud.net/0.3.4/cirros-0.3.4-x86_64-disk.img
$ openstack image create "cirros-0.3.4-x86_64" \
--file cirros-0.3.4-x86_64-disk.img \
--disk-format qcow2 --container-format bare \
--public
```

這邊可以透過 Neutron client 來查看建立外部網路：
```sh
# Create external network
$ openstack network create --external \
--provider-physical-network physnet1 \
--provider-network-type flat public1

# Create external subnet
$ openstack subnet create --no-dhcp --network public1 \
--allocation-pool start=10.26.1.230,end=10.26.1.240 \
--subnet-range 10.26.1.0/24 --gateway 10.26.1.254 public1-subnet

# Create tenant network
$ openstack network create --provider-network-type vxlan admin-net

# Create tenant subnet
$ openstack subnet create --subnet-range 192.168.10.0/24 \
--network admin-net --gateway 192.168.10.1 \
--dns-nameserver 8.8.8.8 admin-subnet

# Create and add router
$ openstack router create admin-router
$ openstack router add subnet admin-router admin-subnet
$ openstack router set --external-gateway public1 admin-router
```

建立 flavor 來提供虛擬機樣板：
```sh
openstack flavor create --id 1 --ram 512 --disk 1 --vcpus 1 m1.tiny
openstack flavor create --id 2 --ram 2048 --disk 20 --vcpus 1 m1.small
openstack flavor create --id 3 --ram 4096 --disk 40 --vcpus 2 m1.medium
openstack flavor create --id 4 --ram 8192 --disk 80 --vcpus 4 m1.large
openstack flavor create --id 5 --ram 16384 --disk 160 --vcpus 8 m1.xlarge
```

上傳 Keypair 來提供虛擬機 SSH 登入認證用：
```sh
$ openstack keypair create --public-key ~/.ssh/id_rsa.pub mykey
```

開啟一台虛擬機進行測試：
```sh
$ openstack server create --image cirros-0.3.4-x86_64 \
--flavor m1.tiny \
--nic net-id=admin-net \
--key-name mykey \
admin-instance
```

查看 instance 列表：
```sh
$ openstack server list

+--------------------------------------+----------------+--------+------------------------+---------------------+
| ID                                   | Name           | Status | Networks               | Image Name          |
+--------------------------------------+----------------+--------+------------------------+---------------------+
| bb3b31b6-56a4-4c68-bd53-b7efb451f0fc | admin-instance | ACTIVE | admin-net=192.168.10.6 | cirros-0.3.4-x86_64 |
+--------------------------------------+----------------+--------+------------------------+---------------------+
```
