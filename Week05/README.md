# W05｜把容器拆開來看：Namespace / Cgroups / Union FS / OCI

## Docker 環境

- Storage Driver: overlay2
- Cgroup Driver: systemd
- Cgroup Version: 2
- Default Runtime: runc

## Namespace 觀察

### 六種 namespace 用途（用自己的話）
- PID：讓容器有一套編號，容器內的第一個進程是 PID 1，跟主機完全隔開，互不干擾。
- NET：容器獨立的網路環境，有自己的網卡、IP、路由表、port。
- MNT：容器只看到自己被掛載的檔案系統視圖，主機上其他目錄對容器不可見。
- UTS：讓容器可以設定自己的 hostname，不影響主機，主要用於辨識節點名稱。
- IPC：隔離 System V shared memory、semaphore、message queue 等跨進程通訊資源。
- USER：`UID` 和 `GID` 在容器內外重新對應，讓容器裡看起來是 root ，但在主機上只是一般使用者。

### Host vs 容器 inode 對照
| Namespace | Host PID 1 inode | 容器 sleep inode | 一樣嗎？ |
|---|---|---|---|
| pid | 4026532597 | 4026531836 | ❌ |
| net | 4026532599 | 4026531833 | ❌ |
| mnt | 4026532594 | 4026531832 | ❌ |
| uts | 4026532595 | 4026531838 | ❌ |
| ipc | 4026532596 | 4026531839 | ❌ |
| user | 4026531837 | 4026531837 | ✅ |

### 容器內 `ps aux` 輸出
```bash
green@app:~$ docker exec -it ns-demo sh
/ # ps aux
PID   USER   TIME   COMMAND
  1   root   0:00   sleep 3600
  7   root   0:00   sh
 14   root   0:00   ps aux
```
> 容器啟動時，Docker 建立一個新的 PID namespace。在這個 namespace 裡，sleep 3600 是第一個運行的 process (PID 1)。整個 namespace 裡只有容器自己的 process，host 上的幾百個 process 根本不在這個命名空間裡，所以 `ps aux` 看不到。

## Cgroups 實驗

### 容器內讀到的限制
- memory.max：268435456
- cpu.max：50000 100000

### Host 端對照（用 `docker inspect -f '{{.HostConfig.CgroupParent}}'` 動態取得路徑）
- memory.max：268435456
- cpu.max：50000 100000
- memory.current（執行時某一刻）：約 794624 bytes

| 項目 | 故障前 | 故障中（memory=32m + dd 200m）| 回復後（memory=256m）|
|---|---|---|---|
| 容器 exit code | - | 137 | 0 |
| OOMKilled | - | true | false |
| dmesg 關鍵字 | 無 OOM | `Memory cgroup out of memory: Killed process ... dd` | 無 OOM |

## Image 分層

### `docker image inspect nginx:1.27-alpine` layer 數量
**8 層**

### 兩個同源 image 共享 layer 的證據
```
sha256:08000c18d16dadf9553d747a58cf44023423a9ab010aab96cf263d2216b8b350  ← 共用
sha256:d71eae0084c1aa823dd8fb2ecf8604d5c0f4911226c042bb1f8297e819f4b192  ← 共用
sha256:c56f134d380585340a68d0db2f2c170641a1c0ff72ccf2438cf2f693df756a85  ← 共用
sha256:e244aa659f612a80c40dd8645812301e3def6b15ec67b9e486ed2201172b51d1  ← 共用
sha256:b8d7d1d2263425d6044e059b2810017d062d659b9b755241f3747eda77726250  ← 共用
sha256:811a4dbbf4a5309e4390cf655c12db92e1a4304fb9d9731f83e7b02e95a617c6  ← 共用
sha256:947e805a4ac71f68e6703550c0b36c2aa2e554c4fa670ca2da6a25c6d7dccb66  ← 共用
sha256:0d853d50b128aa460b47e7121849463a14b18d4fd976caf5014744aae24d28aa  ← 僅 1.27-alpine 有
```

### `docker diff` 輸出範例與解讀
```bash
C /etc
C /etc/nginx
C /etc/nginx/conf.d
A /etc/nginx/conf.d/custom.conf
D /etc/nginx/conf.d/default.conf
C /tmp
A /tmp/hello.txt
```

| 符號 | 意義 | 範例 |
|---|---|---|
| `A` | Added，容器可寫層新增的檔案 | `/tmp/hello.txt`、`/etc/nginx/conf.d/custom.conf` |
| `C` | Changed，目錄內容有異動 | `/etc/nginx/conf.d` |
| `D` | Deleted，從唯讀層繼承的檔案被標記刪除 | `/etc/nginx/conf.d/default.conf` |
 
所有異動都只存在容器的 **upperdir**，唯讀的 lowerdir 完全沒動。其他使用同一 image 的容器看到的 `/etc/nginx/conf.d/default.conf` 還在。

## OCI 呼叫鏈
從 `docker run` 到容器 process 跑起來共經過四層：
 
**dockerd**：使用者面向的 API server。負責解析 `docker run` 的各種 flag（`--memory`、`-p`、`-v`）、管理網路與 volume、處理 build。它本身不直接建容器，而是把請求丟給 containerd。
 
**containerd**：容器生命週期管理器。負責拉 image、管理 snapshot、追蹤容器狀態。收到 dockerd 的請求後，它準備好 OCI bundle（rootfs + config.json），再叫 runc 去執行。
 
**containerd-shim**：每個容器一支的 process。containerd 建完容器後就退出，但 shim 會繼續待著，持有容器的 stdio、等待 exit code。這樣 containerd 重啟也不會把容器一起帶走。
 
**runc**：OCI Runtime Spec 的參考實作。真正呼叫 Linux kernel 的 `clone()` 建 namespace、寫 cgroup 限制、exec 進容器 process 的就是它。跑完後 runc 本身退出，容器 process 由 shim 接管。

## 排錯紀錄
- 症狀：執行 `CPID=$(docker inspect -f '{{.State.Pid}}' ns-demo)` 後 bash 出現 `>` 提示，指令沒有執行。
- 診斷：從網頁複製貼上時，指令內的單引號 `'` 被轉成彎引號，bash 把它當成字串開頭一直等結尾，所以卡住等待繼續輸入。
- 修正：按 `Ctrl + C` 中斷，改用鍵盤手動輸入整行指令，確保引號是直的 `'`。
- 驗證：`echo $CPID` 印出一個數字，確認變數有正確賦值。


## 想一想（回答 3 題）
1. 容器裡的 PID 1 跟 host PID 1 是同一支 process 嗎？`kill -9 1`（在容器內）會發生什麼？
    - 不是同一支。host 的 PID 1 是 `systemd`，容器的 PID 1 是容器啟動時執行的第一個指令。它們活在不同 PID namespace，只是同一支 process 在兩個視角下有不同的編號。
 
    - 在容器內跑 `kill -9 1`，容器的主 process 會被殺掉，容器直接退出。但 host 的 `systemd` 完全不受影響——因為 signal 只在同一個 PID namespace 內有效，容器內的 `kill` 根本碰不到 namespace 外的 process。

2. 兩個容器都基於 `ubuntu:24.04`，磁碟空間是吃兩份還是共用？怎麼驗證？
    - 共用。兩個容器的 lowerdir 指向同一組 overlay2 layer 目錄，磁碟上只有一份。各自的 upperdir 是分開的，但那只佔容器實際寫入的差異量。
    
    - 驗證方式：
    ```bash
    # 拉 image 前後對照磁碟用量
    sudo du -sh /var/lib/docker/overlay2/
    
    # 跑兩個容器
    docker run -d --name u1 ubuntu:24.04 sleep 3600
    docker run -d --name u2 ubuntu:24.04 sleep 3600
    
    # 再看一次，增加量遠小於一份 ubuntu:24.04 的大小
    sudo du -sh /var/lib/docker/overlay2/
    
    # 確認兩個容器的 lowerdir 指向相同路徑
    docker inspect u1 --format '{{.GraphDriver.Data.LowerDir}}'
    docker inspect u2 --format '{{.GraphDriver.Data.LowerDir}}'
    # 兩行輸出應該完全一樣
    ```

3. 如果 host 的 kernel 爆漏洞，容器還能稱為「隔離」嗎？這個限制跟 VM 差在哪？
    - 不能。容器共用 host kernel，kernel 爆漏洞等於所有容器的隔離邊界同時失效。攻擊者只要在任一容器內利用漏洞提權，就能拿到 host 的 root，進而控制同一台機器上的所有容器。
 
    - VM 的差別在於 guest OS 有自己獨立的 kernel，Hypervisor 才是隔離邊界。host kernel 爆洞不會直接影響 VM 裡的 guest kernel——攻擊者還需要額外穿透 Hypervisor 層。這層多出來的隔離就是 Kata Containers 和 Firecracker 存在的原因：它們把容器跑在輕量 VM 裡，讓容器擁有獨立 kernel，同時保留接近容器的啟動速度，兼顧效能與安全隔離。