# Ceph 與 Keystone 整合
首先透過 Keystone 建立 Swift 的認證資訊：
```sh
# 建立 Swift User
$ openstack user create --domain default \
--password SWIFT_PASS --email swift@example.com swift

# 建立 Swift Role
$ openstack role add --project service --user swift admin

# 建立 Swift service
$ openstack service create --name swift  --description "OpenStack Object Storage" object-store

# 建立 Swift v1 public endpoints
$ openstack endpoint create --region RegionOne \
object-store public http://10.0.0.11:8080/swift/v1

# 建立 Swift v1 internal endpoints
$ openstack endpoint create --region RegionOne \
object-store internal http://10.0.0.11:8080/swift/v1

# 建立 Swift v1 admin endpoints
$ openstack endpoint create --region RegionOne \
object-store admin http://10.0.0.11:8080/swift/v1
```

接著進入到部署的`rgw`節點中，編輯`/etc/ceph/ceph.conf`設定檔加入以下內容：
```
[client.radosgw.controller1]
host = controller1
keyring = /etc/ceph/ceph.client.radosgw.controller1.keyring
rgw socket path = /tmp/radosgw.sock
log file = /var/log/ceph/radosgw.controller1.log
rgw dns name = controller1

rgw keystone url = http://10.0.0.11:5000
rgw keystone admin user = admin
rgw keystone admin password = admin
rgw keystone admin project = admin
rgw keystone admin domain = default
rgw keystone api version = 3
rgw keystone accepted roles = SwiftOperator,admin,_member_, project_admin, member2
rgw keystone token cache size = 500
rgw keystone revocation interval = 500
rgw s3 auth use keystone = true
rgw keystone verify ssl = false
```

完成後重新啟動 radosgw 服務：
```sh
$ sudo /etc/init.d/radosgw restart
```

設定 Boot 時啟動 radosgw 服務：
```sh
$ sudo update-rc.d radosgw defaults
```
