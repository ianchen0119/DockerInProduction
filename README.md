# DockerInProduction

本文件列舉了一些使用 Docker 於生產環境會碰到的問題，歡迎一同貢獻、分享！

## 前言

大家好，我是 Ian。之前發表在 StarBugs 的[技術文章（講述如何避免 Docker 佔用大量的磁碟空間）](https://medium.com/starbugs/%E6%89%BE%E5%9B%9E%E9%82%A3%E4%BA%9B%E8%A2%AB-docker-%E5%90%83%E6%8E%89%E7%9A%84%E7%A3%81%E7%A2%9F%E7%A9%BA%E9%96%93-6912cdb24dc0)收到了很不錯的迴響，所以我一直有在思考是不是應該開個課程或是繼續撰寫相關的專欄。

由於本人拖延症的毛病，這些文章草稿被封塵在我的 medium draft(s) 遲遲沒有進展。事情的轉機來自於前幾天看到地震如何影響了花蓮的受災戶，礙於個人僅能在救助捐款上盡一點綿薄之力，因此我借鑑 Backend TW 版大開講座救石虎的方式寫了篇文章，聊聊有關在 production 環境中使用 Docker 有可能碰到的坑，如果你有因為這篇文章受益且行有餘力的話，不妨考慮[小額捐款援助地震災區](https://tw.news.yahoo.com/%E8%8A%B1%E8%93%AE%E5%9C%B0%E9%9C%87%E6%80%A5%E5%BE%85%E6%8F%B4-%E6%8D%90%E6%AC%BE%E9%80%94%E5%BE%91%E4%B8%80%E6%AC%A1%E7%9C%8B-041419058.html)。


## 記憶體管理

Docker 為符合 [OCI](https://opencontainers.org/) 定義的實作，它讓我們可以撰寫簡易的配置檔案後再使用幾行指令啟動一個完全隔離的沙箱（Sandbox），社群上將這個沙箱稱之為「容器」。
儘管如此，這些被我們用 Docker 建立的 Container 在 Kernel Space 上還是依賴宿主機，這也是為什麼 Docker on MacOS/Windows 需要仰賴額外的基礎設施[[1]](https://www.docker.com/blog/introducing-linuxkit-container-os-toolkit/)[[2]](https://learn.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/containerd)。

然而，容器直接的仰賴宿主機的 Linux Kernel，這也意味著這些容器同樣受到 kernel 的管理，如果要將 Docker 這類的容器化技術使用在生產環境上，OOM 的問題值得我們特別注意！
- Out of Memory Killer 由 Linux Kernel 實作，是為了避免 Process 吃光記憶體資源設下的最後一道防線。
- Lorenzo Stoakes 在他的 [linux-vm-notes](https://github.com/lorenzo-stoakes/linux-vm-notes/blob/master/sections/oom.md) 專案上對 OOM Killer 進行了詳細的解說。
- 如何判斷自己是否遇到了 OOM？
  - 在應用程式已經有充分的錯誤處理的情況下，容器還是毫無預兆的發生狀態異常。
  - 使用 `docker inspect` 檢查 exit code 確認為 `exit code (137)`。
  - `docker inspect` 有時似乎不太靈光，經驗告訴我使用 `dmesg` 確認也十分有用。
  - 若經過評估後，宿主機的硬體對比其所執行的附載不至於造成應用程式崩潰，我們可以合理的懷疑應用程式可能有 memory leak 的狀況發生。

## 磁碟管理

如果在生產環境上大量的使用 Docker，在磁碟空間不充足的前提下其實有可能會遇到很多問題：
- 筆者先前在 Medium 撰寫的[文章](https://medium.com/starbugs/%E6%89%BE%E5%9B%9E%E9%82%A3%E4%BA%9B%E8%A2%AB-docker-%E5%90%83%E6%8E%89%E7%9A%84%E7%A3%81%E7%A2%9F%E7%A9%BA%E9%96%93-6912cdb24dc0)講述了如何定期清理無用的 docker 資源。
- 如果機器的數量不大，可以寫簡易的腳本配合 linux cronjob 定期回收資源。
- 若你需要管理大量的機器，可以考慮使用 Ansible 這類的工具一次性管理所有機器上的未使用資源。

## Docker 映像檔案最佳化

- 使用 [multiple stage build](https://docs.docker.com/build/building/multi-stage/)。
- 在 Dockerfile 中如果使用如 `apt`、`snap` 安裝了特定的工具，可在安裝完畢後清除無用的暫存檔案。
- 善用 `.dockerignore` 避免無用的檔案存放於 image。

## DevSecOps

### 不要使用 root 運作你的 container
- 如果非必要，不要使用 root 運作你的 container，使用 root 運作 container 可能會對系統帶來很多隱憂：
  - 預設情況下，Docker image 的預設使用者都是 root，建議在 image 建立一個獨立的 group 以及 user。
  - 盡可能使用 [linux capabilities](https://docs.docker.com/engine/reference/run/#runtime-privilege-and-linux-capabilities) 賦予 container 需要的權利。
  - 使用 root 可能造成的危害：
    1. 如果應用程式有修改檔案的行為，可能會無意間修改從外部掛載進來的檔案。
    2. 如果 container 使用 host network，惡意會無意的修改網路設定都有可能癱瘓宿主機的網路。

### 固定 image 的版本號碼

不要在 production 環境使用的 image 指定 `latest` tag，這會讓我們無法保證每個開發者、每一套 Server、每一次透過 CI/CD deliver 的 docker image 為一致的：
- 建議固定使用某個 stable version。
- 定期使用工具掃描 image 的漏洞，這篇 [HackMD 共筆](https://hackmd.io/@blueskyson/docker-security) 推薦了許多免費的實用工具。

### Rate limit

CI/CD 結合 Docker 非常的方便，但 Docker Hub 的 [rate limit 策略](https://docs.docker.com/docker-hub/download-rate-limit/)可能會讓你的 CI/CD 環境大停擺：
- 對於頻繁使用的 image(s)，可以考慮自架 container registry 或 image proxy 避免觸碰到上限值。
- 如果有創辦 Docker Hub 帳號，可以在 CI/CD Runner 上登入那些帳號，登入後的機器將會獲得更寬鬆的限制。
- 如果前兩者都無法滿足需求，建議考慮付費解鎖官方的 rate limit。

### 在 CI/CD 整合 image scanner

- 將靜態的掃描工具（如：Sonar Qube）整合到 CI/CD pipeline 上可以省去很多麻煩，若 Dockerfile 的撰寫方式會造成資安疑慮，這類工具也能即時的告知開發人員。
- 使用 [syft](https://github.com/anchore/syft) 對常用的 image 進行掃描，它能夠分析當前 image 以及其安裝的 packages 是否有資安漏洞或潛在風險。

### 使用精簡化的 container image

社群上有許多精簡化的 docker image 供大家使用，例如著名的 Alpine（其精簡的實作也不易受到 [CVE](https://cve.mitre.org/) 影響）。
- 需要注意的是，有些應用程式因為 dependency 的關係會沒辦法運作在 Alpine 之上[[1]](https://github.com/golang/go/issues/27264)。
  - 使用 [Distroless](https://github.com/GoogleContainerTools/distroless) 能夠避免這個問題（但空間還是較 Alpine 大了一些）。
  - GitHub 上也能找到一些 Alpine with glibc 的 image 範例，如果有人定期維護或是自己有餘力維護的話也是一個好選擇。


## 網路議題

Docker 的[官方文件](https://docs.docker.com/network/packet-filtering-firewalls/) 提到了一些與安全性相關的議題，對於使用 Docker 來運作對外的網路服務的案例，我們應該特別注意這些問題。

### Iptables

- Docker 通過操作 iptables 的規則提供網路層級的隔離（isolation）
- Docker 客製化了兩個 iptalbes 鏈（chain）存放客製化的規則，分別是 `DOCKER` 和 `DOCKER-USER`：
  - 不要手動修改 `DOCKER` 鏈上的規則，如果你希望為 container 設定特殊規則，請新增到 `DOCKER-USER` 鏈上。
  - 這兩個鏈屬於 `FORWARD` 鏈的一部分，確保所有封包會被兩條鏈上的規則檢查。
- 當封包進入到 `DOCKER-USER` 鏈上時，這些封包都已經被 DNAT 轉換 Dst IP 的資訊：
  - 因此，若要對指定的封包設定規則，應使用 internal IP。
  - 否則，應使用 conntrack（但這會很大程度的影響網路封包的處理效率）。

### Firewall

Debian 和 Ubuntu 使用 ufw 作為防火墻的前端，然而，ufw 與 docker 都使用了 iptables 且出於某些原因，你在 ufw 上設立的規則將無法套用到通往 container 的網路封包。

![圖片取自 wikipedia](https://upload.wikimedia.org/wikipedia/commons/3/37/Netfilter-packet-flow.svg)

- ufw 在 `INPUT` 以及 `OUTPUT` 鏈上對封包進行檢查。
- Ingress：經由 NAT 表（table）後通往 container 的網路封包會被發往 `FORWARD` 鏈，剛好巧妙的繞過 `INPUT` 鏈上的規則。
- Egress：參考上圖，經過 briging decision 且送往 `FORWARD` 鏈的封包不會送往 `OUTPUT` 鏈。
- 補充：Ric Hincapie 的[技術文章](https://richincapie.medium.com/docker-ufw-and-iptables-a-security-flaw-you-need-to-solve-now-40c85587b563) 使用圖解且搭配案例分析很好地解釋了這個問題。

### Dataplane

正常情況下，Container 具有獨立的 network namespace（Host Network Mode 除外），這也代表：
- 與 Host 的 Network Stack 是互相隔離的。
- Network device 是獨立的。
  - 就算在 container 中新增 tun/tap、veth、vlan 等 device，在 host 上也不可見。

但是涉及 kernel plugin 的部分，即使我們從 container 下手，對 host 仍是可見的：
- Container 中載入的 eBPF program（且 container 必須為 privileged mode 才有權限做這件事）。
- Kernel module。  
