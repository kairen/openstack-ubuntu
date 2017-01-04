# DevStack Murano 安裝
所有 OpenStack 的元件與服務都支援了 DevStack 的部署，因此本篇簡單介紹如何使用 DevStack 部署 Murano 服務，首先取得最新版的 devstack 專案：
```sh
$ git clone https://git.openstack.org/openstack-dev/devstack
$ cd devstack
```

建立並編輯`local.conf`，並加入以下內容：
```sh
[[local|localrc]]
HOST_IP="<YOUR_HOST_PUBLIC_IP>"
DATABASE_PASSWORD=password
RABBIT_PASSWORD=password
SERVICE_TOKEN=password
SERVICE_PASSWORD=password
ADMIN_PASSWORD=password

enable_plugin murano git://git.openstack.org/openstack/murano
enable_service murano-cfapi
enable_service g-glare
```

完成後執行`./stack.sh`開始進行安裝：
```sh
$ ./stack.sh
```
