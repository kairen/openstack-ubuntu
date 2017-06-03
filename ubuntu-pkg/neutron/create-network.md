# 建立外部網路與租戶網路
在開啟一台虛擬幾實例枝前，必須先建立要連接的`虛擬網路基礎設施`，如：`External Network`、`Tenant Network`等。採用 Self-service 的網路一定要先建立好才能提供正確的網路給虛擬機使用。下圖顯示一個基本網路架構預覽。

![架構圖](images/installguide-neutron-initialnetworks.png)
> 注意！這邊架構圖的`External Network`會隨著環境而不同。

## External network（外部網路）
外部網路提供給虛擬機分配網際網路的 IP，該網路通常指允許使用 NAT（Network Address Translation）來讓虛擬機存取網際網路。當然也可以透過建置 Floating IP Pool 與適當的 Security Group 規則來存取網際網路，一般外部網路是由 admin 建立的。

### 建立 External network 實例
回到`Controller`節點，導入 Keystone 的`admin`帳號來使用管理者權限：
```sh
. admin-openrc
```

這邊可以透過 Neutron client 來查看建立外部網路，如以下方式：
```sh
$ openstack network create --share --external \
--provider-physical-network external \
--provider-network-type flat ext-net
```
> 這邊採用`flat`。

成功的話會看到類似以下資訊：
```sh
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | UP                                   |
| availability_zone_hints   |                                      |
| availability_zones        |                                      |
| created_at                | 2017-06-03T11:01:08Z                 |
| description               |                                      |
| dns_domain                | None                                 |
| id                        | fcb5d5ef-f538-445f-b064-3e45d843f0ef |
| ipv4_address_scope        | None                                 |
| ipv6_address_scope        | None                                 |
| is_default                | False                                |
| mtu                       | 1500                                 |
| name                      | ext-net                              |
| port_security_enabled     | True                                 |
| project_id                | 54b11ae3f8d54b098dc057e7b4b1f0bd     |
| provider:network_type     | flat                                 |
| provider:physical_network | external                             |
| provider:segmentation_id  | None                                 |
| qos_policy_id             | None                                 |
| revision_number           | 4                                    |
| router:external           | External                             |
| segments                  | None                                 |
| shared                    | True                                 |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| updated_at                | 2017-06-03T11:01:08Z                 |
+---------------------------+--------------------------------------+
```

外部網路如同一個實體的網路，而一般一個虛擬網路都必須被分配子網路。外部網路透過`Netwrok`節點上的 Public 網卡介面來分享，以及連接實體網路的相關子網路、Gateway。然而我們需要替路由器以及 Floating IP 指定一個獨有的網段，來避免與外部網路的其他裝置衝突。

若要其他的外部網路類型可以參考以下列表：

| network_type |  physical_network | segmentation_id |
|:------------:|:-----------------:|:---------------:|
|     flat     | physical networks |       None      |
|     vlan     | physical networks |     vlan id     |
|   gre/vxlan  |        NULL       |    tunnel id    |
|     local    |        NULL       |       NULL      |

### 建立 External network 子網路
這邊可以透過 Neutron client 來查看建立外部網路子網路，如以下方式：
```sh
$ openstack subnet create --network ext-net \
--allocation-pool start=START_IP_ADDRESS,end=END_IP_ADDRESS \
--dns-nameserver DNS_RESOLVER --gateway PROVIDER_NETWORK_GATEWAY \
--subnet-range PROVIDER_NETWORK_CIDR ext-subnet
```
> * `PROVIDER_NETWORK_CIDR`為 Public net 的 CIDR 網路，如`192.168.0.0/24`。
* `START_IP_ADDRESS`與`END_IP_ADDRESS`為要分配的浮動 IP 範圍。
* `PROVIDER_NETWORK_GATEWAY`為 Public net 的閘道 IP 位址，如`192.168.0.1`。
* `DNS_RESOLVER` 為該網路的 DNS Server IP。

> <font color=red>最後要注意關閉此網路上的其他 DHCP 服務，因為虛擬機實例不直接連接到外部網路，而是透過 Neutron 來分配</font>

一個實際範例如下所示：
```sh
$ openstack subnet create --network ext-net \
--allocation-pool start=172.22.2.220,end=172.22.2.250 \
--dns-nameserver 8.8.4.4 --gateway 172.22.2.254 \
--subnet-range 172.22.2.0/24 \
--no-dhcp ext-subnet
```

成功的話會看到類似以下資訊：
```
+-------------------+--------------------------------------+
| Field             | Value                                |
+-------------------+--------------------------------------+
| allocation_pools  | 172.22.2.220-172.22.2.250            |
| cidr              | 172.22.2.0/24                        |
| created_at        | 2017-06-03T11:10:28Z                 |
| description       |                                      |
| dns_nameservers   | 8.8.4.4                              |
| enable_dhcp       | False                                |
| gateway_ip        | 172.22.2.254                         |
| host_routes       |                                      |
| id                | 6c0e3360-fe3a-40a3-9e9f-03db317e5f97 |
| ip_version        | 4                                    |
| ipv6_address_mode | None                                 |
| ipv6_ra_mode      | None                                 |
| name              | ext-subnet                           |
| network_id        | fcb5d5ef-f538-445f-b064-3e45d843f0ef |
| project_id        | 54b11ae3f8d54b098dc057e7b4b1f0bd     |
| revision_number   | 2                                    |
| segment_id        | None                                 |
| service_types     |                                      |
| subnetpool_id     | None                                 |
| updated_at        | 2017-06-03T11:10:28Z                 |
+-------------------+--------------------------------------+
```

## Tenant network (可略過，透過 Dashboard 操作即可)
租戶網路主要提供私有的內部網路給虛擬機使用。採用 Self-service 可以確保各租戶之間的網路隔離。這邊透過`demo`使用者來建立一個自己的私有網路，該網路只會提供內部的網路給虛擬機連接。

#### 建立 Tenant network 實例
回到`Controller`節點，導入 Keystone 的`demo`帳號來使用使用者權限：
```sh
. demo-openrc
```

這邊可以透過 Neutron client 來查看建立租戶網路，如以下方式：
```sh
$ neutron net-create demo-net
+---------------------------+--------------------------------------+
| Field                     | Value                                |
+---------------------------+--------------------------------------+
| admin_state_up            | True                                 |
| id                        | 174bee75-7a06-44f8-8d68-8722d8d6912d |
| mtu                       | 0                                    |
| name                      | demo-net                             |
| provider:network_type     | gre                                  |
| provider:physical_network |                                      |
| provider:segmentation_id  | 1                                    |
| router:external           | False                                |
| shared                    | False                                |
| status                    | ACTIVE                               |
| subnets                   |                                      |
| tenant_id                 | cf8f9b8b009b429488049bb2332dc311     |
+---------------------------+--------------------------------------+
```

這邊網路建立方式類似外部網路，故一樣需要建立一個子網路。我們可以指定任意的子網路給租戶網路使用，因為這架構的關係網路是不會與租戶直接共享的。預設情況下，這個子網路會啟動 DHPC 來自動分配 IP 給虛擬機。

### 建立 Tenant network 子網路
這邊可以透過 Neutron client 來查看建立租戶網路子網路，如以下方式：
```sh
$ neutron subnet-create demo-net TENANT_NETWORK_CIDR \
--name demo-subnet --gateway TENANT_NETWORK_GATEWAY
```
> * 將`TENANT_NETWORK_CIDR`替換為你想關聯到租戶網路的子網路。
* 將`TENANT_NETWORK_GATEWAY`替換為你想關聯的子網路 Gateway。

然後建立一個網路 CIDR 為 `192.168.1.0/24` 的子網路：
```sh
neutron subnet-create demo-net 192.168.1.0/24 \
--gateway 192.168.1.1 \
--dns-nameserver 8.8.8.8 \
--name demo-subnet
```

成功會看到類似以下資訊：
```
+-------------------+--------------------------------------------------+
| Field             | Value                                            |
+-------------------+--------------------------------------------------+
| allocation_pools  | {"start": "192.168.1.2", "end": "192.168.1.254"} |
| cidr              | 192.168.1.0/24                                   |
| dns_nameservers   |                                                  |
| enable_dhcp       | True                                             |
| gateway_ip        | 192.168.1.1                                      |
| host_routes       |                                                  |
| id                | 469dfc0d-80f8-4c1e-ab38-5be9ec937282             |
| ip_version        | 4                                                |
| ipv6_address_mode |                                                  |
| ipv6_ra_mode      |                                                  |
| name              | demo-subnet                                      |
| network_id        | 174bee75-7a06-44f8-8d68-8722d8d6912d             |
| subnetpool_id     |                                                  |
| tenant_id         | cf8f9b8b009b429488049bb2332dc311                 |
+-------------------+--------------------------------------------------+
```

當建立好子網路後，即可以使用該網路，但這個網路只是提供一個私有網路給虛擬機使用，若要讓外部網路能夠存取虛擬機的話，我們必須透過虛擬路由器來將 Tenant network 與 External network 連接起來ㄡ

### 建立一個虛擬路由器
這邊可以透過 Neutron client 來查看建立一個路由器，如以下方式：
```sh
$ neutron router-create demo-router
+-----------------------+--------------------------------------+
| Field                 | Value                                |
+-----------------------+--------------------------------------+
| admin_state_up        | True                                 |
| distributed           | False                                |
| external_gateway_info |                                      |
| ha                    | False                                |
| id                    | a65b2209-1473-4f4f-8a3b-a99ea9f7020b |
| name                  | demo-router                          |
| routes                |                                      |
| status                | ACTIVE                               |
| tenant_id             | cf8f9b8b009b429488049bb2332dc311     |
+-----------------------+--------------------------------------+
```

接著將剛建立的`demo`租戶網路附加到虛擬路由器上：
```sh
$ neutron router-interface-add demo-router demo-subnet
Added interface 276fc58e-bd54-4229-be4f-6e0abe5272f3 to router demo-router.
```

然後再將外部網路附加到虛擬路由器上：
```sh
$ neutron router-gateway-set demo-router ext-net
Set gateway for router demo-router
```

# 服務驗證
接著進行下一步之前，我們要先驗證網路是否已正常。當我們建立一個虛擬路由器，並附加一個外部網路時，會從上面建立的外部子網路的範圍中自動分配一個 IP 給路由器的介面使用，一般來說會循序分配（若該網路沒有其他硬體裝置的話），因此這邊會拿到`10.26.1.221`，這時候我們可以在該外部網路上的任一主機進行 Ping 的動作。

> 如果您在虛擬機上設定您的 OpenStack 節點，您必須設定管理程式以允許外部網路上的混雜模式。

這邊透過簡單的 ICMP 來檢查虛擬路由器有正常啟動：
```sh
ping -c 4 10.26.1.221
```

### 無法解析 domain name 問題
當建立的網路提供給虛擬機使用時，可以連接到網路網路，可能會發生無法解析網址導致無法下載與更新服務等問題。這可能是因為在建立租戶網路時，忘記設定 DNS，這邊可以透過以下指令來更新已存在的租戶子網路：
```sh
neutron subnet-update demo-subnet --dns-nameserver list=true 8.8.8.8 8.8.8.4 163.17.131.2
```

完成後，就可以正常連接了。
![Ubuntu instance](images/ubuntu_instance.png)
