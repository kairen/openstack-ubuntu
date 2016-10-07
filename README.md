# OpenStack Ubuntu 相關整理
主要整理 Ubuntu 作業系統上，建置 OpenStack 的相關教學整理。目標為任何 Ubuntu 建立 OpenStack 叢集的教學整理。

# 參與貢獻
如果您想一起參與貢獻的話，您可以協助以下項目：
* 幫忙校正、挑錯別字、語病等等
* 提供一些修改建議
* 提出對某些術語翻譯的建議
* 提供 OpenStack 新套件教學、簡介或者是程式碼分析。

# 透過 Github 進行協作
1. 在 ```Github``` 上 ```fork``` 到自己的 Repository，例如：```<User>/openstack-ubuntu.git```，然後 ```clone```到 Local 端，並設定 Git 使用者資訊。
```sh
$ git clone https://github.com/<User>/openstack-ubuntu.git
$ cd openstack-ubuntu
$ git config user.name "User"
$ git config user.email user@email.com
```

2. 修改程式碼或內容後，透過 ```commit``` 來提交到自己的 Repository：
```sh
$ git commit -am "Fix issue #1: change helo to hello"
$ git push
```
> 若新增採用一般文字訊息，如 ```Add neutron vlan tag intro```。

3. 在 GitHub 上提教一個 Pull Request。
4. 持續針對專案的 Repository 進行更新：
```sh
$ git remote add upstream  https://github.com/kairen/openstack-ubuntu.git
$ git fetch upstream
$ git checkout master
$ git rebase upstream/master
$ git push -f origin master
```
