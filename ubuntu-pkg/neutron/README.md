# Neutron 安裝與設定
OpenStack Neutron 提供了管理 OpenStack 環境中的虛擬網路基礎設施(Virtual Networking Infrastructure, VNI)所有網路層面,以及實體網路基礎設施(Physical Networking Infrastructure, PNI)接入層面。

Neutron 提供了虛擬機器與運算之間的軟體定義網路功能。Neutron 引用了 plugin 來實現網路定義 API 的後端。這些 plugin 可能使用了基本的 Linux 網路功能，如 Linux Bridge、VLANs、IP Tables 等，更能夠使用 SDN 技術，如 OpenFlow、L2 - L3 隧道技術。目前 Neutron 有許多部署方式，如 Linix Bridge、Open vSwitch、廠商 Driver 等。本節將依據以下步驟說明如何部署 Neutron。

- [Controller Node](#controller-node)
    - [Controller 安裝前準備](#controller-安裝前準備)
    - [Controller 套件安裝與設定](#controller-套件安裝與設定)
    - [Controller 設定 ML2](#controller-設定-ml2)
    - [Controller 設定 Nova 使用 Network](#controller-設定-nova-使用-network)
    - [Controller 驗證服務](#controller-驗證服務)
- [Network Node](#network-node)
    - [Network 安裝前準備](#network-安裝前準備)
    - [Network 套件安裝與設定](#network-套件安裝與設定)
    - [Network 設定 ML2](#network-設定-ml2)
        - [Network Open vswitch agent](#network-open-vswitch-agent)
        - [Network Linux bridge agent](#network-linux-bridge-agent)
    - [設定 Layer 3 Proxy](#設定-layer-3-proxy)
    - [設定 DHCP Proxy](#設定-dhcp-proxy)
    - [設定 Metadata Agent](#設定-metadata-agent)
    - [設定 Open vSwitch 服務](#設定-open-vswitch-服務)
    - [Network 驗證服務](#network-驗證服務)
- [Compute Node](#compute-node)
    - [Compute 安裝前準備](#compute-安裝前準備)
    - [Compute 套件安裝與設定](#compute-套件安裝與設定)
    - [Compute 設定 ML2](#compute-設定-ml2)
        - [Compute Open vswitch agent](#compute-open-vswitch-agent)
        - [Compute Linux bridge agent](#compute-linux-bridge-agent)
    - [設定 Compute 使用 Network](#設定-compute-使用-network)
    - [Compute 驗證服務](#compute-驗證服務)

# Controller Node
在 Controller 節點上，只需要安裝 Neutron API Server 與 ML2 Plugins 即可。

### Controller 安裝前準備
在開始安裝前，要預先建立一個資料庫給 Neutron 儲存相關資訊，使用以下指令建立資料庫：
```sh
$ mysql -u root -p
```

透過以下命令用來更新現有帳號資料或建立 Neutron 資料庫：
```sql
CREATE DATABASE neutron;
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'localhost'  IDENTIFIED BY 'NEUTRON_DBPASS';
GRANT ALL PRIVILEGES ON neutron.* TO 'neutron'@'%'  IDENTIFIED BY 'NEUTRON_DBPASS';
```
> 這邊`NEUTRON_DBPASS`可以隨需求修改。

完成後離開資料庫，接著要建立 Service 與 API Endpoint，首先導入 `admin` 環境變數：
```sh
$ . admin-openrc
```

接著透過以下流程來建立 Glance 的使用者、Service 以及 API Endpoint：
```sh
# 建立 Neutron User
$ openstack user create --domain default --password NEUTRON_PASS --email neutron@example.com neutron

# 新增 Neutron 到 Admin Role
$ openstack role add --project service --user neutron admin

# 建立 Neutron Service
$ openstack service create --name neutron --description "OpenStack Networking" network

# 建立 Neutron public endpoints
$ openstack endpoint create --region RegionOne \
network public http://10.0.0.11:9696

# 建立 Neutron internal endpoints
$ openstack endpoint create --region RegionOne \
network internal http://10.0.0.11:9696

# 建立 Neutron admin endpoints
$ openstack endpoint create --region RegionOne \
network admin http://10.0.0.11:9696
```

### Controller 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt install -y neutron-server neutron-plugin-ml2
```

安裝完成後，編輯`/etc/neutron/neutron.conf`設定檔，在`[DEFAULT]`部分加入以下設定：
```
[DEFAULT]
...
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = true
auth_strategy = keystone
notify_nova_on_port_status_changes = True
notify_nova_on_port_data_changes = True

transport_url = rabbit://openstack:RABBIT_PASS@10.0.0.11
```
> 這邊`RABBIT_PASS`可以隨需求修改。

在`[database]`部分修改使用以下方式：
```
[database]
# connection = sqlite:////var/lib/neutron/neutron.sqlite
connection = mysql+pymysql://neutron:NEUTRON_DBPASS@10.0.0.11/neutron
```

在`[keystone_authtoken]`部分加入以下設定：
```
[keystone_authtoken]
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
memcached_servers = 10.0.0.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = NEUTRON_PASS
```
> 這邊`NEUTRON_PASS`可以隨需求修改。

在`[nova]`部分加入以下設定：
```
[nova]
auth_url = http://10.0.0.11:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = nova
password = NOVA_PASS
```
> 這邊`NOVA_PASS`可以隨需求修改。

### Controller 設定 ML2
在 Neutron 的 ML2 Plugins 中會使用到多種 Agent 來提供網路虛擬化功能，其中較為常見的為 Open vSwitch 與 Linux Bridge 兩種，然而之間各有其好處，這邊主要針對 Open vSwitch 進行說明，而 Linux Birdge 可以參考註解說明。

首先編輯`/etc/neutron/plugins/ml2/ml2_conf.ini`並在`[ml2]`部分加入以下設定：
```
[ml2]
type_drivers = flat,vlan,gre,vxlan
tenant_network_types = vxlan
mechanism_drivers = openvswitch,l2population
extension_drivers = port_security
```
> 一旦您設定好了 ML2 插件，修改 `type_drivers` 選項的值會導致資料庫的不一致。所以建議一開始就設定多個。

> `P.S.`若使用`Linux Bridge`則修改`mechanism_drivers`為`linuxbridge,l2population`。且注意 Linux Birdge <font color=red>只支援 VXLAN Overlay 網路</font>。

> `P.S.`若使用`GRE`則更改`tenant_network_types`為`gre`。

在`[ml2_type_vxlan]`部分加入以下設定：
```
[ml2_type_vxlan]
vni_ranges = 1:1000
```
> `P.S.`若使用`GRE`則修改以下內容：
```
[ml2_type_gre]
tunnel_id_ranges = 1:1000
```

在`[securitygroup]`部分加入以下設定：
```
[securitygroup]
enable_ipset = True
```

### Controller 設定 Nova 使用 Network
當完成 ML2 設定後，接著編輯`/etc/nova/nova.conf`設定檔，在`[neutron]`部分加入以下設定：
```
[neutron]
url = http://10.0.0.11:9696
auth_url = http://10.0.0.11:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS
```
> 這邊`NEUTRON_PASS`可以隨需求修改。

完成所有設定後，即可同步資料庫來建立 Neutron 資料表：
```sh
$ sudo neutron-db-manage --config-file /etc/neutron/neutron.conf \
--config-file /etc/neutron/plugins/ml2/ml2_conf.ini \
upgrade head
```

之後重新啟動 Nova API Server：
```sh
$ sudo service nova-api restart
```

再接著重新啟動 Neutron Server：
```sh
$ sudo service neutron-server restart
```

### Controller 驗證服務
導入 Keystone 的 `admin` 帳號，來透過 Neutron client 查看服務目錄：
```sh
$ . admin-openrc
```

這邊可以透過 Neutron client 來查看外部網路列表，如以下方式：
```sh
$ openstack extension list --network
+-------------------------------------------------------+---------------------------+--------------------------------------------------------+
| Name                                                  | Alias                     | Description                                            |
+-------------------------------------------------------+---------------------------+--------------------------------------------------------+
| Default Subnetpools                                   | default-subnetpools       | Provides ability to mark and use a subnetpool as the   |
|                                                       |                           | default                                                |
| Network IP Availability                               | network-ip-availability   | Provides IP availability data for each network and     |
|                                                       |                           | subnet.                                                |
| Network Availability Zone                             | network_availability_zone | Availability zone support for network.                 |
| Auto Allocated Topology Services                      | auto-allocated-topology   | Auto Allocated Topology Services.                      |
| Neutron L3 Configurable external gateway mode         | ext-gw-mode               | Extension of the router abstraction for specifying     |
| Neutron L3 Configurable external gateway mode         | ext-gw-mode               | Extension of the router abstraction for specifying     |
| ...                                                   | ...                       | ...                                                    |
+-------------------------------------------------------+---------------------------+--------------------------------------------------------+
```

# Network Node
Neutron 在 Network 節點會安裝提供虛擬機的 L2 與 L3 網路虛擬化，來處理內部與外部網路的路由，還有包含 DHCP 服務、Metadata 服務。

### Network 安裝前準備
在進行安裝 Neutron 相關套件之前，必須先讓 Network 節點主機設定一些 Kernel 網路參數，透過編輯`/etc/sysctl.conf`加入以下參數：
```
net.ipv4.ip_forward=1
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
```

完成後，可以透過 sysctl 指令來將參數載入：
```sh
$ sudo sysctl -p
```

### Network 套件安裝與設定
在開始設定之前，首先要安裝相關套件與 OpenStack 服務套件，可以透過以下指令進行安裝：
```sh
$ sudo apt install -y neutron-plugin-ml2 neutron-l3-agent \
neutron-dhcp-agent neutron-metadata-agent \
neutron-openvswitch-agent
```
> `P.S.`這邊若使用`Linux Bridge`則修改安裝套件`neutron-openvswitch-agent` 為 `neutron-linuxbridge-agent`。

安裝完成後，編輯`/etc/neutron/neutron.conf`設定檔，在`[DEFAULT]`部分加入以下設定：
```
[DEFAULT]
...
verbose = True
auth_strategy = keystone
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True
transport_url = rabbit://openstack:RABBIT_PASS@10.0.0.11
```
> 這邊`RABBIT_PASS`可以隨需求修改。

在`[database]`部分將所有 connection 與 sqlite 的參數註解掉：
```sh
[database]
# connection = sqlite:////var/lib/neutron/neutron.sqlite
```

在`[keystone_authtoken]`部分加入以下設定：
```
[keystone_authtoken]
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
memcached_servers = 10.0.0.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = NEUTRON_PASS
```
> 這邊`NEUTRON_PASS`可以隨需求修改。

### Network 設定 ML2
在 Neutron 的 ML2 Plugins 中會使用到多種 Agent 來提供網路虛擬化功能，其中較為常見的為 Open vSwitch 與 Linux Bridge 兩種，然而之間各有其好處，這邊主要針對 Open vSwitch 進行說明，而 Linux Birdge 可以參考註解說明。

首先編輯`/etc/neutron/plugins/ml2/ml2_conf.ini`並在`[ml2]`部分加入以下設定：
```
[ml2]
type_drivers = flat,vlan,gre,vxlan
tenant_network_types = vxlan
mechanism_drivers = openvswitch,l2population
extension_drivers = port_security
```
> `P.S.`若使用`Linux Bridge`則修改`mechanism_drivers`為`linuxbridge,l2population`。且注意 Linux Birdge <font color=red>只支援 VXLAN Overlay 網路</font>。

> `P.S.`若使用`GRE`則更改`tenant_network_types`為`gre`。

在`[ml2_type_flat]`部分加入以下設定：
```
[ml2_type_flat]
flat_networks = external
```

在`[ml2_type_vxlan]`部分加入以下設定：
```
[ml2_type_vxlan]
vni_ranges = 1:1000
```
> `P.S.`若使用`GRE`則修改以下內容：
```
[ml2_type_gre]
tunnel_id_ranges = 1:1000
```

在`[securitygroup]`部分加入以下設定：
```
[securitygroup]
enable_ipset = True
```

#### Network Open vswitch agent
接著要設定 Open vSwitch agent，編輯`/etc/neutron/plugins/ml2/openvswitch_agent.ini`在`[ovs]`部分加入以下設定：
```
[ovs]
local_ip = TUNNELS_IP
bridge_mappings = external:br-ex
```
> 將`TUNNELS_IP`取代成與 Compute 節點溝通的 IP，這邊為`10.0.1.21`。

在`[agent]`部分加入以下設定：
```
[agent]
tunnel_types = vxlan
l2_population = True
prevent_arp_spoofing = True
```
> `P.S.`若使用`GRE`則修改`tunnel_types`為`gre`。

在`[securitygroup]`部分加入以下設定：
```
[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
```

#### Network Linux bridge agent
若要設定 Linux bridge agent 則編輯 `/etc/neutron/plugins/ml2/linuxbridge_agent.ini`設定檔，在 `[linux_bridge]`部分加入以下設定：
```
[linux_bridge]
physical_interface_mappings = external:<physical_interface>
```
> 這邊`<physical_interface>`為`eth1`。

在`[vxlan]`部分加入以下設定：
```
[vxlan]
local_ip = OVERLAY_IP
enable_vxlan = true
l2_population = True
```
> 這邊`OVERLAY_IP`為`10.0.1.21`。

在`[agent]`部分加入以下設定：
```
[agent]
tunnel_types = vxlan
prevent_arp_spoofing = True
```

在`[securitygroup]`部分加入以下設定：
```
[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

### 設定 Layer 3 Proxy
由於本架構採用 Self-service，故需要設定 Layer 3 Proxy 來提供虛擬化路由器。編輯`/etc/neutron/l3_agent.ini`在`[DEFAULT]`部分加入以下設定:
```
[DEFAULT]
...
verbose = True
interface_driver = openvswitch
external_network_bridge =
```
> `P.S.`若使用`Linux bridge`則修改`interface_driver`為`linuxbridge`。

> external_network_bridge 參數預設保留空白，該參數是使用單一的 Agent 來提供多個外部網路，細節設定可以參考 [L3 Agent](http://docs.openstack.org/havana/config-reference/content/section_adv_cfg_l3_agent.html)。

### 設定 DHCP Proxy
由於本架構採用 Self-service，故需要設定 DHCP Proxy 來提供 DHCP 服務給虛擬機使用。編輯`/etc/neutron/dhcp_agent.ini`在`[DEFAULT]`部分加入以下設定：
```
[DEFAULT]
...
verbose = True
interface_driver = openvswitch
dhcp_driver = neutron.agent.linux.dhcp.Dnsmasq
enable_isolated_metadata = True
```
> `P.S.`若使用`Linux bridge`則修改`interface_driver`為`linuxbridge`。

完成上述後，下一步可以依需求設定，因為類似 VXLAN 的協定包含了額外的 Header 封包，這些封包增加了網路開銷，而減少了有效的封包可用空間。在不了解虛擬網路架構的情況下，Instance 會用預設的 ETH Maximum Transmission Unit（MTU）1500 bytes 來傳送封包。IP 網路利用 Path MTU discovery（PMTUD）機制來偵測與調整封包大小。但是有些作業系統、網路阻塞、缺乏對 PMTUD 支援等因素，會造成效能的損失與連接錯誤。

最好情況下，可以在 Network 節點上開啟 Jumbo frames 來避免該問題。因為 Jumbo frames 支援最大 9000 bytes 的 MTU，這大大解決了覆蓋網路（Overlay Network）的封包開銷影響。但是很多網路設備不一定支援，且 OpenStack 管理人員也可能忽略對網路架構的控制，因此考慮到複雜性，這邊選擇降低 MTU 大小來避免該問題，使用 VXLAN 大多環境採用 `1450` Bytes 來執行。

設定 MTU 可以透過編輯`/etc/neutron/dhcp_agent.ini`在`[DEFAULT]`部分加入以下設定：
```
[DEFAULT]
...
dnsmasq_config_file = /etc/neutron/dnsmasq-neutron.conf
```

然後建立並編輯`/etc/neutron/dnsmasq-neutron.conf`來設定 MTU 大小：
```sh
＄ echo 'dhcp-option-force=26,1450' | sudo tee /etc/neutron/dnsmasq-neutron.conf
```
> `P.S.`若是使用`GRE`則大多採用`1454`的 MTU。

### 設定 Metadata Agent
OpenStack Metadata 提供了一些主機客製化的設定訊息，諸如 Hostname、網路配置資訊等等。編輯`/etc/neutron/metadata_agent.ini`在`[DEFAULT]`部分加入以下設定：
```
[DEFAULT]
nova_metadata_ip = 10.0.0.11
metadata_proxy_shared_secret = METADATA_SECRET
```
> 將其中的`METADATA_SECRET`替換為一個合適的 metadata 代理的 secret。

完成上面設定後，先回到`Controller`節點，編輯`/etc/nova/nova.conf`，在`[neutron]`部分加入以下設定：
```
[neutron]
...
service_metadata_proxy = True
metadata_proxy_shared_secret = METADATA_SECRET
```
> 將其中的`METADATA_SECRET`替換為一個合適的 metadata 代理的 secret。

然後在`Controller`節點上重新啟動 Nova API Server：
```sh
$ sudo service nova-api restart
```

### 設定 Open vSwitch 服務
如果安裝採用 OVS 作為網路 Agent 的話，須回到`Network`節點上，由於外部網路透過 OVS 指令來設定網卡橋接，首先重新啟動 openvswitch-switch：
```sh
$ sudo service openvswitch-switch restart
```

然後新增一個外部網路橋接：
```sh
$ sudo ovs-vsctl add-br br-ex
```
> 這邊`br-ex`可自行定義。

新增外部網路橋接映射到實體網卡埠口：
```sh
$ sudo ovs-vsctl add-port br-ex INTERFACE_NAME
```
> `INTERFACE_NAME`為`Public`網路的網卡介面名稱，這邊範例為`eth1`。

根據網卡介面驅動差異，可能需要關閉 GRO（Generic Receive Offload）來實現虛擬機與外部網路的合適頻寬。在測試環境上，我們在外部網卡介面上暫時關閉 GRO：
```sh
$ sudo ethtool -K INTERFACE_NAME gro off
```
> `INTERFACE_NAME`為`Public`網路的網卡介面名稱，這邊範例為`eth1`。

### Network 完成安裝
當完成上述所有安裝與設定後，回到`Network`節點重新啟動 ovs 與 ovs agent：
```sh
sudo service openvswitch-switch restart
sudo service neutron-openvswitch-agent restart
```
> `P.S.`若是使用`Linux bridge`則重新啟動以下:
```sh
$ sudo service neutron-linuxbridge-agent restart
```

接著重新啟動 Neutron agents：
```sh
sudo service neutron-dhcp-agent restart
sudo service neutron-metadata-agent restart
sudo service neutron-l3-agent restart
```

### Network 驗證服務
接著回到`Controller`節點導入`admin` 帳號來驗證服務：
```sh
$ . admin-openrc
```

這邊可以透過 Neutron client 來查看 Agents 狀態，如以下方式：
```sh
$ openstack network agent list
+--------------------------------------+--------------------+----------+-------------------+-------+-------+---------------------------+
| ID                                   | Agent Type         | Host     | Availability Zone | Alive | State | Binary                    |
+--------------------------------------+--------------------+----------+-------------------+-------+-------+---------------------------+
| 5570d60e-737d-4970-b0cd-cac54460eb87 | DHCP agent         | network1 | nova              | True  | UP    | neutron-dhcp-agent        |
| 598780db-ad4c-4cac-b708-449b5dfcc455 | L3 agent           | network1 | nova              | True  | UP    | neutron-l3-agent          |
| c9489e94-53ab-4554-9901-2ae141ba41b5 | Metadata agent     | network1 | None              | True  | UP    | neutron-metadata-agent    |
| ca3840bc-5b99-4f82-bdec-a4ebe6f9baab | Open vSwitch agent | network1 | None              | True  | UP    | neutron-openvswitch-agent |
+--------------------------------------+--------------------+----------+-------------------+-------+-------+---------------------------+
```

# Compute Node
Neutron 在 Compute 節點主要安裝 Neutron L2 Agent 來讓虛擬機連接虛擬網路與安全群組。

### Compute 安裝前準備
在進行安裝 Neutron 相關套件之前，必須先讓 Compute 節點主機設定一些 Kernel 網路參數，透過編輯`/etc/sysctl.conf`加入以下參數：
```
net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
net.bridge.bridge-nf-call-iptables=1
net.bridge.bridge-nf-call-ip6tables=1
```

完成後，可以透過 sysctl 指令來將參數載入：
```sh
$ sudo sysctl -p
```

### Compute 套件安裝與設定
首先透過`apt`安裝套件：
```sh
$ sudo apt install -y neutron-openvswitch-agent
```
> `P.S.`若是使用`Linux bridge`則修改成`neutron-linuxbridge-agent`。

安裝完成後，編輯`/etc/neutron/neutron.conf`設定檔，在`[DEFAULT]`部分加入以下設定：
```
[DEFAULT]
...
verbose = True
auth_strategy = keystone
core_plugin = ml2
service_plugins = router
allow_overlapping_ips = True
transport_url = rabbit://openstack:RABBIT_PASS@10.0.0.11
```
> 這邊`RABBIT_PASS`可以隨需求修改。

在`[database]`部分將所有 connection 與 sqlite 的參數註解掉：
```sh
[database]
# connection = sqlite:////var/lib/neutron/neutron.sqlite
```

在`[keystone_authtoken]`部分加入以下設定：
```
[keystone_authtoken]
auth_uri = http://10.0.0.11:5000
auth_url = http://10.0.0.11:35357
memcached_servers = 10.0.0.11:11211
auth_type = password
project_domain_name = default
user_domain_name = default
project_name = service
username = neutron
password = NEUTRON_PASS
```
> 這邊`NEUTRON_PASS`可以隨需求修改。

### Compute 設定 ML2
在 Neutron 的 ML2 Plugins 中會使用到多種 Agent 來提供網路虛擬化功能，其中較為常見的為 Open vSwitch 與 Linux Bridge 兩種，然而之間各有其好處，這邊主要針對 Open vSwitch 進行說明，而 Linux Birdge 可以參考註解說明。

#### Compute Open vswitch agent
接著要設定 Open vSwitch agent，編輯`/etc/neutron/plugins/ml2/openvswitch_agent.ini`在`[ovs]`部分加入以下設定：
```
[ovs]
local_ip = TUNNELS_IP
```
> 將`TUNNELS_IP`取代成與 Network 節點溝通的 IP，這邊為`10.0.1.31`。

在`[agent]`部分加入以下設定：
```
[agent]
tunnel_types = vxlan
l2_population = True
prevent_arp_spoofing = True
```
> `P.S.`若使用`GRE`則修改`tunnel_types`為`gre`。

在`[securitygroup]`部分加入以下設定：
```
[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.OVSHybridIptablesFirewallDriver
```

#### Compute Linux bridge agent
接著要設定 Linux bridge agent，編輯`/etc/neutron/plugins/ml2/linuxbridge_agent.ini`，在`[vxlan]`部分加入以下設定：
```
[vxlan]
local_ip = OVERLAY_IP
enable_vxlan = true
l2_population = True
```
> 這邊`OVERLAY_IP`為`10.0.1.31`。

在`[agent]`部分加入以下設定：
```
[agent]
tunnel_types = vxlan
prevent_arp_spoofing = True
```

在`[securitygroup]`部分加入以下設定：
```
[securitygroup]
enable_security_group = True
firewall_driver = neutron.agent.linux.iptables_firewall.IptablesFirewallDriver
```

### 設定 Compute 使用 Network
當完成 ML2 設定後，接著編輯`/etc/nova/nova.conf`，在`[neutron]`部分加入以下設定：：
```
[neutron]
url = http://10.0.0.11:9696
auth_url = http://10.0.0.11:35357
auth_type = password
project_domain_name = default
user_domain_name = default
region_name = RegionOne
project_name = service
username = neutron
password = NEUTRON_PASS
```
> 這邊`NEUTRON_PASS`可以隨需求修改。

當完成上述安裝與設定後，在`Compute`節點重新啟動 Nova compute：
```sh
$ sudo service nova-compute restart
```

重新啟動 Open vSwitch Agent：
```sh
sudo service openvswitch-switch restart
sudo service neutron-openvswitch-agent restart
```
> `P.S.`若是使用`Linux Bridge`則重新啟動 Linux Bridge Agent：
```sh
$ sudo service neutron-linuxbridge-agent restart
```

### Compute 驗證服務
接著回到`Controller`節點導入`admin`帳號來驗證服務：
```sh
$ . admin-openrc
```

這邊可以透過 Neutron client 來查看 Agents 狀態，如以下方式：
```sh
$ openstack network agent list
```
> 若正確的話，會看到多一個`compute`節點執行的 agent

完成後，就可以建立外部網路與租戶網路（Self-service），請參閱 [建立外部網路與租戶網路](create-network.md)。
