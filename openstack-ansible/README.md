#  Openstack Ansible AIO 安裝
這邊說明如何使用 openstack-ansible 進行單節點的環境部署，部署環境作業系統採用 Ubuntu 14.04 LTS，Ansible 版本為 2.2.0，首先在節點所有將 Package 更新：
```sh
$ sudo apt-get dist-upgrade
$ reboot
```

接著透過 git clone 最新版本的 openstack-ansible：
```sh
$ sudo su -
$ git clone https://github.com/openstack/openstack-ansible /opt/openstack-ansible
$ cd /opt/openstack-ansible
```

這邊可以依據需求切換自己想要部署的版本：
```sh
# List all existing tags.
$ git tag -l

# Checkout the stable branch and find just the latest tag
$ git checkout stable/newton

# Checkout the latest tag from either method of retrieving the tag.
$ git checkout 13.0.1
```

設定一個選擇性參數，這邊主要設定 Bootstrap 的 Disk 與 ubuntu pacage repos:
```sh
$ export BOOTSTRAP_OPTS="bootstrap_host_data_disk_device=sdb"
$ export BOOTSTRAP_OPTS="${BOOTSTRAP_OPTS} bootstrap_host_ubuntu_security_repo=http://tw.archive.ubuntu.com//ubuntu"
$ export BOOTSTRAP_OPTS="${BOOTSTRAP_OPTS} bootstrap_host_ubuntu_repo=http://tw.archive.ubuntu.com//ubuntu"
```
> AIO 預設是使用 `apt-get` 安裝，也可以使用 source 方式，輸入以下參數來改變：
```sh
$ export ANSIBLE_ROLE_FETCH_MODE=git-clone
```

上面完成後，執行以下指令來啟動 ansible 相關設定：
```sh
$ scripts/bootstrap-ansible.sh
```

接著執行以下指令來進行 AIO 環境的準備:
```sh
$ scripts/bootstrap-aio.sh
```

最後，執行以下指令開始進行叢集部署:
```sh
$ scripts/run-playbooks.sh
```
