# DockerInProduction

本文件列舉了一些使用 Docker 於生產環境會碰到的問題，如果你覺得內容還不錯，請考慮[小額捐款援助地震災區](https://tw.news.yahoo.com/%E8%8A%B1%E8%93%AE%E5%9C%B0%E9%9C%87%E6%80%A5%E5%BE%85%E6%8F%B4-%E6%8D%90%E6%AC%BE%E9%80%94%E5%BE%91%E4%B8%80%E6%AC%A1%E7%9C%8B-041419058.html)。

## 記憶體管理

Docker 為符合 [OCI](https://opencontainers.org/) 定義的實作，它讓我們可以撰寫簡易的配置檔案後再使用幾行指令啟動一個完全隔離的沙箱（Sandbox），社群上將這個沙箱稱之為「容器」。
儘管如此，這些被我們用 Docker 建立的 Container 在 Kernel Space 還是依賴宿主機，這也是為什麼 Docker on MacOS/Windows 需要仰賴額外的工具[[1]](https://www.docker.com/blog/introducing-linuxkit-container-os-toolkit/)[[2]](https://learn.microsoft.com/en-us/virtualization/windowscontainers/deploy-containers/containerd)。

然而，容器直接的仰賴宿主機的 Linux Kernel，這也意味著這些容器也會受到 kernel 直接的管理，如果要將 Docker 這類的容器化技術使用在生產環境上，OOM 的問題值得我們特別注意！
- Out of Memory Killer 由 Linux Kernel 實作，是為了避免 Process 吃光記憶體資源設下的最後一道防線。
- Lorenzo Stoakes 在他的 [linux-vm-notes](https://github.com/lorenzo-stoakes/linux-vm-notes/blob/master/sections/oom.md) 專案上對 OOM Killer 進行了詳細的解說
- 如何判斷自己是否遇到了 OOM？
  - 在應用程式已經有充分的錯誤處理的情況下，容器還是毫無預兆的發生狀態異常。
  - 使用 `docker inspect` 檢查 exit code 確認為 `exit code (137)`。
  - `docker inspect` 有時似乎不太靈光，經驗告訴我使用 `dmesg` 確認也十分有用。
  - 若經過評估後，宿主機的硬體對比其所執行的附載不至於造成應用程式崩潰，我們可以合理的懷疑應用程式可能有 memory leak 的狀況發生。

## 磁碟管理

如果在生產環境上大量的使用 Docker，在磁碟空間不充足的前提下其實有可能會遇到很多問題：
- 筆者先前在 Medium 撰寫的[文章](https://medium.com/starbugs/%E6%89%BE%E5%9B%9E%E9%82%A3%E4%BA%9B%E8%A2%AB-docker-%E5%90%83%E6%8E%89%E7%9A%84%E7%A3%81%E7%A2%9F%E7%A9%BA%E9%96%93-6912cdb24dc0)講述了如何定期清理無用的 docker 資源。

## Docker 映像檔案最佳化

## 安全議題

## 網路議題

## CI/CD

CI/CD 結合 Docker 非常的方便，但 Docker Hub 的 [rate limit 策略](https://docs.docker.com/docker-hub/download-rate-limit/)可能會讓你的 CI/CD 環境大停擺：
- 對於頻繁使用的 image(s)，可以考慮自架 container registry 避免觸碰到上限值。
- 如果有創辦 dockerHub 帳號，可以在 CI/CD Runner 上登入那些帳號，登入後的機器將會獲得更寬鬆的限制。
