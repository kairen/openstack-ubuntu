# Murano UI 安裝
在 OpenStack Kilo 版本中，Murano 的 Application Catalog 已經正式釋出正式版，且 Murano 也整合於 Dashboard 上，本章節就是安裝該 UI 套件。

### Install Murano Dashboard
由於 Murano Dashboard 目前還沒有相關 .deb 安裝來源，因此需要使用原始碼來安裝，透過 Git 來取得：
```sh
$ sudo git clone https://github.com/openstack/murano-dashboard /opt/murano-dashboard -b stable/newton
$ sudo chown -R ${USER}:${USER} /opt/murano-dashboard

$ cd /opt/ && sudo pip install -e murano-dashboard/
```
> 安裝 Horizon 參考本書 Git 安裝章節

> 如果沒有使用 keystone, 加入`MURANO_API_URL = 'http://10.0.0.11:8082'`到 Horizon `local_settings.py`檔案。

將 Murano dashboard 相關程式檔案複製到 Horizon:
```sh
$ cd /usr/share/openstack-dashboard
$ sudo cp /opt/murano-dashboard/muranodashboard/local/enabled/_50_murano.py openstack_dashboard/local/enabled/
$ sudo cp /opt/murano-dashboard/muranodashboard/local/local_settings.d/_50_murano.py openstack_dashboard/local/local_settings.d/
$ sudo cp /opt/murano-dashboard/muranodashboard/conf/murano_policy.json openstack_dashboard/conf/
```

完成後讓 Django 進行 collectstatic 與 compress：
```sh
$ sudo ./manage.py collectstatic
$ sudo ./manage.py compress
```

完成後可以重新啟動 Apache2 :
```sh
$ sudo service apache2 restart
```

![](horizon-magnum.png)
