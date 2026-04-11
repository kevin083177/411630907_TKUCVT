# W04｜Linux 系統基礎：檔案系統、權限、程序與服務管理

## FHS 路徑表

| FHS 路徑 | FHS 定義 | Docker 用途 |
|---|---|---|
| /etc/docker/ | 系統級設定檔目錄存 | 存放 `daemon.json`，用於自訂 Docker daemon 行為；若無自訂設定則不存在此檔案，daemon 使用預設值運行|
| /var/lib/docker/ | 程式的持久性狀態資料目錄 | 存放所有映像、容器、volumes 及 overlay2 層資料；拉取映像後此目錄大小會增加 |
| /usr/bin/docker | 使用者可執行檔目錄 | Docker CLI 工具本體；which docker 指向此路徑；所有人可執行，但執行後是否能連到 daemon 取決於 socket 權限 |
| /run/docker.sock | 執行期暫存目錄 | （PID/socket）Docker daemon 的 Unix socket；daemon 啟動時建立，關機後消失；CLI 透過此 socket 發送 API 請求給 daemon |

## Docker 系統資訊

- Storage Driver： ```Storage Driver: overlayfs
  driver-type: io.containerd.snapshotter.v1```
- Docker Root Dir： ```/var/lib/docker```
- 拉取映像前 /var/lib/docker/ 大小： `360K`
- 拉取映像後 /var/lib/docker/ 大小： `364K`

## 權限結構

### Docker Socket 權限解讀
```srw-rw---- 1 root docker 0  4月 12 01:27 /var/run/docker.sock```
1. 檔案類型 + 權限位元
    - `s → socket` 不是一般檔案，而是 Unix Socket
2. Owner(rw-)
    - r（read）→ 可讀
    - w（write）→ 可寫
    - -（execute）→ 沒有執行權
3. Group(rw-)
    - r（read）→ 可讀
    - w（write）→ 可寫
    - -（execute）→ 沒有執行權
4. Others(---)
    - 沒有任何權限
### 使用者群組
```bash
uid=1000(green) gid=1000(green) groups=1000(green),4(adm),24(cdrom),27(sudo),30(dip),46(plugdev),100(users),114(lpadmin)
```

### 安全意涵
Docker daemon 本身以 root 身份運行，掌控整台 Host 的容器生命週期。CLI 透過 `/var/run/docker.sock` 這個 Unix socket 和 daemon 溝通——凡是能讀寫這個 socket 的使用者，就等於能對 daemon 下任何指令。
Docker group 的成員對 socket 有完整的讀寫權限（rw-），因此能做到的事和 root 幾乎等價。實際示範如下：
```bash
docker run --rm -v /etc/shadow:/host-shadow:ro alpine cat /host-shadow
```
一個普通的 `docker run` 指令，就能讀取 Host 的 `/etc/shadow`（密碼雜湊檔）——這是只有 root 才該有的能力。能掛載 Host 任意目錄、讀取任意檔案，等同取得了 Host 的 root 存取權。

## 程序與服務管理

### systemctl status docker
```bash
● docker.service - Docker Application Container Engine
     Loaded: loaded (/usr/lib/systemd/system/docker.service; enabled; preset: e>
     Active: active (running) since Sun 2026-04-12 01:28:03 CST; 11min ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 1664 (dockerd)
      Tasks: 10
     Memory: 136.5M (peak: 150.6M)
        CPU: 3.548s
     CGroup: /system.slice/docker.service
             └─1664 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/cont>

 4月 12 01:28:02 bastion dockerd[1664]: time="2026-04-12T01:28:02.731592774+08:>
 4月 12 01:28:02 bastion dockerd[1664]: time="2026-04-12T01:28:02.736572747+08:>
 4月 12 01:28:03 bastion dockerd[1664]: time="2026-04-12T01:28:03.842181591+08:>
 4月 12 01:28:03 bastion dockerd[1664]: time="2026-04-12T01:28:03.866079105+08:>
 4月 12 01:28:03 bastion dockerd[1664]: time="2026-04-12T01:28:03.866700876+08:>
 4月 12 01:28:03 bastion dockerd[1664]: time="2026-04-12T01:28:03.909845277+08:>
 4月 12 01:28:03 bastion dockerd[1664]: time="2026-04-12T01:28:03.935806313+08:>
 4月 12 01:28:03 bastion dockerd[1664]: time="2026-04-12T01:28:03.936293729+08:>
 4月 12 01:28:03 bastion systemd[1]: Started docker.service - Docker Applicatio>
 4月 12 01:34:11 bastion dockerd[1664]: time="2026-04-12T01:34:11.190415476+08:>
```

### journalctl 日誌分析
```bash
4月 12 01:28:01  Starting docker.service - Docker Application Container Engine...
4月 12 01:28:02  level=info msg="Starting up"
4月 12 01:28:02  level=info msg="CDI directory does not exist, skipping" dir=/etc/cdi
4月 12 01:28:02  level=info msg="detected 127.0.0.53 nameserver, assuming systemd-resolved..."
4月 12 01:28:02  level=info msg="Creating a containerd client" address=/run/containerd/containerd.sock
4月 12 01:28:02  level=info msg="Loading containers: start."
4月 12 01:28:02  level=info msg="NRI is disabled"
4月 12 01:28:03  level=info msg="Loading containers: done."
4月 12 01:28:03  level=info msg="Docker daemon" version=29.3.0 storage-driver=overlayfs
4月 12 01:28:03  level=info msg="Daemon has completed initialization"
4月 12 01:28:03  level=info msg="API listen on /run/docker.sock"
4月 12 01:28:03  Started docker.service - Docker Application Container Engine.
4月 12 01:34:11  level=info msg="image pulled" remote="docker.io/library/nginx:latest"
```
- `01:28:01`：systemd 開始啟動 docker.service。
- `01:28:02`：daemon 自身開始初始化，依序建立 containerd client、載入既有容器清單。其中出現兩則 nftables 刪除規則失敗的 info 訊息（No such file or directory），這是 daemon 啟動時清理舊網路規則的正常嘗試，在全新環境中沒有舊規則可刪屬正常現象，不影響運作。
- `01:28:03`：daemon 初始化完成，開始監聽 `/run/docker.sock`，服務正式上線。storage-driver 確認為 `overlayfs`，版本 29.3.0。
- `01:34:11`：記錄到一次 `nginx:latest` 映像拉取事件。

### CLI vs Daemon 差異
- Docker CLI（`/usr/bin/docker`）是一個普通的命令列工具，安裝後就靜靜放在 `/usr/bin/`。你打指令時它才被呼叫，做的事是把你的輸入包成 API 請求，透過 socket 送給 daemon。它本身不管理任何容器，也不知道有沒有容器在跑。
- Docker Daemon（`dockerd`）才是真正做事的角色，由 systemd 管理，開機自動啟動，持續在背景監聽 `/run/docker.sock`，負責實際建立、啟動、停止容器。
## 環境變數
- $PATH：
    ```
    /usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin:/snap/bin
    which docker: /usr/bin/docker
    ```
- which docker：`/usr/bin/docker`
- 容器內外環境變數差異觀察：
    1. Host 的 $PATH 包含 /snap/bin、/usr/games 等發行版特有路徑，
    2. 容器內（alpine）的 $PATH 只有六個基本系統路徑。
    3. $HOME 在 Host 是 /home/green，容器內是 /root（預設 root 身份執行）。
    4. 容器沒有 $USER、$SHELL 等變數，HOSTNAME 是容器 ID 而非機器名稱。
    5. 容器是獨立的執行環境，不繼承 Host 的任何 shell 設定。
## 故障場景一：停止 Docker Daemon

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| systemctl status docker | active | inactive | active |
| docker --version | 正常 | 正常 | 正常 |
| docker ps | 正常 | Cannot connect | 正常 |
| ps aux grep dockerd | 有 process | 無 process | 有 process |

## 故障場景二：破壞 Socket 權限

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ls -la docker.sock 權限 | srw-rw---- | srw------- | srw-rw---- |
| docker ps（不加 sudo） | 正常 | permission denied | 正常 |
| sudo docker ps | 正常 | 正常 | 正常 |
| systemctl status docker | active | active | active |

## 錯誤訊息比較

| 錯誤訊息 | 根因 | 診斷方向 |
|---|---|---|
| Cannot connect to the Docker daemon | Daemon 沒有在跑，socket 根本不存在或無人監聽 | `systemctl status docker` 確認狀態 → `systemctl start docker` 啟動；若反覆失敗則看 `journalctl -u docker` 找原因 |
| permission denied…docker.sock | Daemon 正常運行，但當前使用者對 socket 無存取權限 | `ls -la /var/run/docker.sock` 確認權限 → 檢查 `id` 是否在 docker group → 修正權限或加入群組 |

- Cannot connect 是 socket 那端沒人，daemon 死了
- Permission denied 是 daemon 活著只是沒資格。sudo docker 能不能用，就是最快的分辨線索——能用代表 daemon 還活著，問題在權限。

## 排錯紀錄
- 症狀： 執行 `docker ps` 回傳 `permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock`，但 `sudo docker ps` 正常。
- 診斷：確認 daemon 狀態 → 確認 socket 權限 → 確認使用者群組
- 修正：`sudo usermod -aG docker $USER`
- 驗證：`id`、`docker ps`、`docker run --rm hello-world`

## 設計決策
**Q: 為什麼教學環境用 `usermod -aG docker $USER` 而不是每次 `sudo`**

- 每次執行 docker 指令都加 `sudo`，會讓操作變得繁瑣。在個人開發機或教學環境中，使用者對這台機器有完整控制權，加入 docker group 省下的摩擦成本是合理的。

- 如前面安全示範所見，docker group 成員等同 root——能透過容器掛載 Host 任意路徑、讀取 `/etc/shadow` 等敏感檔案。這在單人開發機上風險可控，但在以下情境中絕對不應這樣做：

    1. 多人共用的機器
    2. 面向外部網路的伺服器
    3. CI/CD runner 等自動化環境

- 正確的生產環境做法是使用 rootless Docker 或透過明確的 `sudo` 規則控制誰能執行哪些 docker 指令，而不是把使用者直接加進 docker group。