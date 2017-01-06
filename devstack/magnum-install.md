# DevStack Magnum 安裝
所有 OpenStack 的元件與服務都支援了 DevStack 的部署，因此本篇簡單介紹如何使用 DevStack 部署 Magnum 服務，首先取得最新版的 devstack 專案：
```sh
$ git clone https://git.openstack.org/openstack-dev/devstack
```

建立並編輯`local.conf`，並加入以下內容：
```sh
HOST_IP="<YOUR_HOST_PUBLIC_IP>"
DATABASE_PASSWORD=password
RABBIT_PASSWORD=password
SERVICE_TOKEN=password
SERVICE_PASSWORD=password
ADMIN_PASSWORD=password

PUBLIC_INTERFACE="YOUR_PUBLIC_INTERFACE"

enable_plugin barbican https://git.openstack.org/openstack/barbican
enable_plugin heat https://git.openstack.org/openstack/heat
enable_plugin magnum https://git.openstack.org/openstack/magnum
enable_plugin magnum-ui https://github.com/openstack/magnum-ui

# Ensure we are using neutron networking rather than nova networking
disable_service n-net
enable_service q-svc
enable_service q-agt
enable_service q-dhcp
enable_service q-l3
enable_service q-meta
enable_service neutron

# Enable LBaaS(v2) services
enable_service q-lbaasv2
enable_service octavia
enable_service o-cw
enable_service o-hk
enable_service o-hm
enable_service o-api

VOLUME_BACKING_FILE_SIZE=20G
```

若要加入 ceilometer 進行監測的話，可以加入以下到`local.conf`：
```sh
enable_plugin ceilometer https://git.openstack.org/openstack/ceilometer
```

完成後執行`./stack.sh`開始進行安裝：
```sh
$ ./stack.sh
```
