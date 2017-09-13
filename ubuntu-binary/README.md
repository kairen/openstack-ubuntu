# OpenStack Ubuntu Manual 安裝
當您已完成單節點或是 DevStack 後，想當然會進一步嘗試架設多台的 OpenStack 叢集，這邊安裝提供了基於 OpenStack Neutron 的網路虛擬化部署模式，該架構需利用到三個節點進行最小架構安裝，當然這邊可以依需求增加節點來安裝諸如 Cinder 區塊儲存、Swift 物件儲存、Manila 共享式檔案系統等其他服務。以下為使用到節點角色介紹：
* **Controller Node**：OpenStack 叢集中的主要控制者，會透過 Message Queue 系統來與其他節點進行溝通，該節點也會安裝 MySQL 來儲存套件資訊與 NTP 來提供網路時間同步。在 OpenStack 套件部分會包含 Glance、Keystone、各套件的排程器與 API Server等。

* **Network Node**：該節點主要是 OpenStack 中的網路虛擬化提供者，包含 L2、L3 與 DHCP agent 等。該節點有可以提供一些 Layer 7 的網路服務，諸如：LBaaS、FWaaS 等。

* **Compute node**：該節點主要是 OpenStack 中運算虛擬化技術 Hypervisor 的提供者，可以提供如：KVM、QEMU、Xen 等虛擬化技術。該節點也會安裝網路的 Agent 來存取 Network 節點提供的虛擬化網路給虛擬機實例使用。

以下節點屬於非核心套件的服務，可以依需求加入：
* **Block Storage Node**：該節點主要提供區塊儲存服務給虛擬機使用，讓區塊裝置可以被持久性的保存與遷移。

* **Object Storage Node**：該節點主要提供物件儲存，並且可基於 Amazon S3 RESTful API 來提供物件儲存服務。

## OpenStack services
本安裝教學將一步一步的部署 OpenStack 的核心服務，後面也會針對其他非核新服務進行教學。

首先介紹 OpenStack 中的幾個核心服務，由於 OpenStack 在 Liberty 開始使用 Big Tent 機制，該機制將 OpenStack 拆分核心服務與第三方服務，因此只要更新核心服務版本也能跟舊版的第三方服務進行串接，以下介紹核心服務名稱與功能：

| 服務功能         	| 專案名稱 	| 描述                                                                                                                                                                                                 	|
|------------------	|----------	|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------	|
| Identity service 	| Keystone 	| 提供其他 OpenStack 服務身份證認與授權功能，因為 OpenStack 各服務 API Server 若沒有使用 Keystone 做認證機制的話，就能夠被隨意的存取資源。                                                             	|
| Image service    	| Glance   	| 提供虛擬機映像檔服務，讓使用者可以檢視與上下載映像檔。該映像檔最主要是提供作業系統給 OpenStack 的虛擬機使用。                                                                                        	|
| Compute          	| Nova     	| 管理 OpenStack 中 Compute 節點的虛擬機實例生命週期。包含建置虛擬機、排程尋找最佳 Compute 節點等。                                                                                                    	|
| Networking       	| Neutron  	| 提供 OpenStack 其他服務擁有網路連接功能，好比提供給 OpenStack 虛擬機實例虛擬化網路、建立虛擬路由器、管理容器（LXC、Docker）網路等，也支援了多家廠商的硬體外掛（Plugins）與驅動（Driver）來進行串接。 	|
| Dashboard        	| Horizon  	| 提供一個基於 Web 的自助服務，讓 OpenStack 使用者可以與各套件服務進行互動操作，比如建立虛擬機、虛擬路由與網路、上傳檔案到物件儲存等。                                                                 	|
| Block Storage    	| Cinder   	| 提供持久性儲存服務（Persistent Block Storage）給 OpenStack 虛擬機實例，該服務的隨插即用驅動架構簡化了區塊儲存裝置的建立、管理與快照等。                                                              	|


隨著 IT 人員的需求，有些特有的功能需要不同的 OpenStack 套件來提供，好比容器服務就可以使用 Magnum，由於安裝會進行到一些第三方服務，這邊說明以下幾個第三方服務名稱與功能：

| 服務功能      | 專案名稱   | 描述                                                                                                                                                                                             |
|---------------|------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Object Storage|Swift| 提供基於 RESTful API 的物件儲存服務，透過 HTTP 就能任意查看非結構化的資料物件，且該服務擁有高度容錯機制、資料副本以及橫向擴展架構。該服務不需要安裝一個集中式的檔案目錄伺服器。|
| Shared File Systems|Manila| 共享式檔案系統服務提供了共享與分散式檔案系統來協調與存取使用。|
| Orchestration |Heat| 該服務提供 OpenStack 編配（Orchestrates）服務，可以基於 HOT 板模格式或是 AWS CloudFormation 板模格式，且可以透過 OpenStack 原生 REST API 與 CloudFormation-compatible Query API 來進行功能存取。|
| Telemetry Data Collection|Ceilometer| OpenStack 雲端系統資源的監控與評測，該服務基於上面提供計費（Billing）、基準測試（Benchmarking）、可擴展性（Scalability）以及統計（Statistical）。|
| Telemetry Alarming|aodh|Telemetry Alarming 服務提供當收集的 metering 或事件資料脫離定義的規則時觸發警報通知管理者。|
| Database as a Service|Trove|Trove 提供了資料庫即服務(DBaaS)，並可以透過雲端配置功能來部署各種資料庫引擎。|
| Container Infrastructure Management|Magnum|Magnum 由 OpenStack Container Team 開發的 container orchestration engines，像是 Docker 和 Kubernetes 可作為 OpenStack 上第一類資源開發的 OpenStack API 服務。Magnum 使用 Heat 來編配(orchestrate) OS 映像檔(image)，其中包含 Docker 和 Kubernetes，並依照叢集配置執行在任何的虛擬機或裸機(bare metal)中。 |
| Messaging service|Zaqar|Zaqar 是對 Web 開發人員提供了多租戶（Multi tenant）的雲端訊息服務。它結合開創了 Amazon 的 SQS 產品與附加的語義來支援事件的廣播想法。|
| Key Manager service|Barbican|Barbican 是一個以 REST API 設計來進行安全儲存、配置以及機密的管理，如密碼、加密金鑰以及 X.509 憑證。其目的是為了適用於所有環境，包含大型短暫性雲端。|
| Bare Metal service|Ironic|Ironic 裸機部署功能，使 IT 人員可以在實體伺服器自動化部署 OpenStack，等於能用管理虛擬機器的方式，來管理實體伺服器，有助於一次部署大量 OpenStack 主機來滿足大型 IaaS 環境的需要。|

> `P.S.` 上述清單是基於 OpenStack 基金會已提供擁有正式 Install Guide 的項目撰寫。但本安裝教學也會針對一些額外服務進行安裝說明。每當有新服務釋出也會盡力更新。
