# External Ceph
這邊說明如何串接已部署就緒的 Ceph 叢集。雖然 Kolla 提供了自己部署 Ceph 的方法，但有些情況下 Ceph 可以已經部署完成，因此需要有其他方式進行整合。

### 需求
* 一組已經部署好的 Ceph 叢集
* 須預先建立 Ceph 儲存池，如:vms、images
* 須建立提供授權憑證給 OpenStack 服務連線 Ceph 使用。

### 啟用 External Ceph
若使用外部 Ceph 來作為儲存後端的話，這表示我們不會透過 Kolla 來部署 Ceph，因此需要在`/etc/kolla/global.yml`設定檔中，來關閉這個功能：
```
enable_ceph: "no"
```

而要啟動 OpenStack 服務使用 Ceph 的話，則需要編輯`/etc/kolla/global.yml`設定檔，修改以下內容來啟用：
```
glance_backend_ceph: "yes"
cinder_backend_ceph: "yes"
nova_backend_ceph: "yes"
```
(TBD..)
