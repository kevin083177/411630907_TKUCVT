# W02｜VMware 網路模式與雙 VM 排錯

## 網路配置

| VM | 網卡 | 模式 | IP | 用途 |
|---|---|---|---|---|
| dev-a | NIC 1 | NAT | 192.168.192.144 | 上網 |
| dev-a | NIC 2 | Host-only | 192.168.70.130 | 內網互連 |
| server-b | NIC 1 | Host-only | 192.168.70.132 | 內網互連 |

## 連線驗證紀錄

- dev-a NAT 可上網：`ping google.com` 輸出
![](https://github.com/kevin083177/411630907_TKUCVT/blob/main/Week02/resources/1.png?raw=true)
- 雙向互 ping 成功：貼上雙方 `ping` 輸出
![](https://github.com/kevin083177/411630907_TKUCVT/blob/main/Week02/resources/2-1.png?raw=true)
![](https://github.com/kevin083177/411630907_TKUCVT/blob/main/Week02/resources/2-2.png?raw=true)
- SSH 連線成功：`ssh <user>@<ip> "hostname"` 輸出
![](https://github.com/kevin083177/411630907_TKUCVT/blob/main/Week02/resources/4.png?raw=true)
- SCP 傳檔成功：`cat /tmp/test-from-dev.txt` 在 server-b 上的輸出
![](https://github.com/kevin083177/411630907_TKUCVT/blob/main/Week02/resources/3.png?raw=true)
- server-b 不能上網：`ping 8.8.8.8` 失敗輸出
![](https://github.com/kevin083177/411630907_TKUCVT/blob/main/Week02/resources/5.png?raw=true)

## 故障演練一：介面停用

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| server-b 介面狀態 | UP | DOWN | UP |
| dev-a ping server-b | 成功 | 失敗 | 成功 |
| dev-a SSH server-b | 成功 | 失敗 | 成功 |

## 故障演練二：SSH 服務停止

| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ss -tlnp grep :22 | 有監聽 | 無監聽 | 有監聽 |
| dev-a ping server-b | 成功 | 成功 | 成功 |
| dev-a SSH server-b | 成功 | Connection refused | 成功 |

## 排錯順序
遇到連線問題時，由下往上逐層排查：

- **L2（介面層）** — 先確認網卡有沒有起來、有沒有 IP
    ```bash
    ip address show
    ```
    介面顯示 `DOWN` 或沒有 IP，就在這層解決，不往上查。

- **L3（網路層）** — 確認路由正確、封包能到對端
    ```bash
    ip route show
    ping -c 4 192.168.70.132
    ```
    ping 不通就檢查路由表，看有沒有對應的路徑。

- **L4（服務層）** — 確認服務有在監聽、連線沒被擋
    ```bash
    ss -tlnp | grep :22
    ssh @192.168.70.132 "hostname"
    ```
    ping 通但 SSH 不通，才往這層查。

## 網路拓樸圖
![](https://github.com/kevin083177/411630907_TKUCVT/blob/main/Week02/resources/Network%20Topology.png?raw=true)

## 排錯紀錄
- 症狀：dev-a 無法 SSH 到 server-b，出現 `Connection refused`
- 診斷：先跑 `ping -c 4 192.168.70.132`，ping 成功，確認 L3 可達，問題不在網路層。再跑 `ss -tlnp | grep :22`，發現 port 22 無監聽
- 修正：`sudo systemctl start ssh`
- 驗證：`ss -tlnp | grep :22` 重新出現監聽，`ssh <user>@192.168.70.132 "hostname"` 成功回傳 `server-b`


## 設計決策

`server-b` 只設 `Host-only` 不給 NAT，是因為它只需要被 `dev-a` 管理，不需要自己上網。保持單網卡也讓架構更清楚，職責不混淆：`dev-a` 負責對外、`server-b` 只在內網。如果需要裝套件，臨時從 `dev-a` 用 SCP 傳過去或暫時加一張 NAT 卡即可。