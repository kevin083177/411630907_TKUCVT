# W03｜多 VM 架構：分層管理與最小暴露設計

## 網路配置

| VM | 角色 | 網卡 | 模式 | IP | 開放埠與來源 |
|---|---|---|---|---|---|
| bastion | 跳板機 | NIC 1 | NAT | 192.168.192.144 | SSH from any |
| bastion | 跳板機 | NIC 2 | Host-only | 192.168.70.130 | — |
| app | 應用層 | NIC 1 | Host-only | 192.168.70.132 | SSH from 192.168.70.0/24 |
| db | 資料層 | NIC 1 | Host-only | 192.168.70.133 | SSH from app + bastion |

## SSH 金鑰認證

- 金鑰類型：ed25519
- 公鑰部署到：`app` 和 `db` 的 `~/.ssh/authorized_keys`
- 免密碼登入驗證：
  - bastion → app：`ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICvaBnK6cVUNHAdBofPTuVSp+PnLshKIvRthoRPW3ZX1 bastion-key`
  - bastion → db：`ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAICvaBnK6cVUNHAdBofPTuVSp+PnLshKIvRthoRPW3ZX1 bastion-key`

## 防火牆規則

### app 的 ufw status
```bash
狀態: 啓用
日誌: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
新建設定檔案: skip

至                          動作          來自
-                          --          --
22/tcp                     ALLOW IN    192.168.70.0/24   
```

### db 的 ufw status
```bash
狀態: 啓用
日誌: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
新建設定檔案: skip

至                          動作          來自
-                          --          --
22/tcp                     ALLOW IN    192.168.70.132            
22/tcp                     ALLOW IN    192.168.70.130 
```

### 防火牆確實在擋的證據
```bash
curl: (28) Connection timed out after 5003 milliseconds
```

## ProxyJump 跳板連線
- 指令：

![](https://github.com/kevin083177/411630907_TKUCVT/blob/main/Week03/assets/1.png?raw=true)
- 驗證輸出：

![](https://github.com/kevin083177/411630907_TKUCVT/blob/main/Week03/assets/2.png?raw=true)
- SCP 傳檔驗證：

![](https://github.com/kevin083177/411630907_TKUCVT/blob/main/Week03/assets/3.png?raw=true)
![](https://github.com/kevin083177/411630907_TKUCVT/blob/main/Week03/assets/4.png?raw=true)

## 故障場景一：防火牆全封鎖
| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| app ufw status | active + rules | deny all | active + rules |
| bastion ping app | 成功 | 失敗（timeout） | 成功 |
| bastion SSH app | 成功 | **timed out** | 成功 |

## 故障場景二：SSH 服務停止
| 項目 | 故障前 | 故障中 | 回復後 |
|---|---|---|---|
| ss -tlnp grep :22 | 有監聽 | 無監聽 | 有監聽 |
| bastion ping app | 成功 | 成功 | 成功 |
| bastion SSH app | 成功 | **refused** | 成功 |

## timeout vs refused 差異
**Connection timed out** 
- 代表封包送出去之後沒有收到任何回應。通常是防火牆把封包靜默丟棄造成的，排錯方向要往 L3.5 查，先  確認防火牆規則有沒有放行 `sudo ufw status`。
 
**Connection refused** 
- 代表目標機器有收到封包，但馬上回了一個拒絕。這代表網路是通的，只是 port 上沒有服務在監聽。排錯方向要往 L4 查，使用 `ss -tlnp | grep :22` 確認 SSH 服務有沒有在跑。
 
## 網路拓樸圖
![](https://github.com/kevin083177/411630907_TKUCVT/blob/main/Week03/assets/Network%20Topology.png?raw=true)

## 排錯紀錄

- 症狀：從 bastion SSH 到 app 時出現 `Connection timed out`，但 ping 是通的
- 診斷：ping 通代表 L3 網路沒問題，timeout 而非 refused 指向防火牆問題，
  先到 app 上跑 `sudo ufw status verbose` 確認規則
- 修正：發現 ufw 啟用後缺少允許 SSH 的規則，補上
  `sudo ufw allow from 192.168.70.0/24 to any port 22 proto tcp`
- 驗證：回到 bastion 重跑 `ssh green@192.168.70.132 "hostname"`，
  成功回傳 `app` 確認修正有效

## 設計決策
**Q: `db` 的防火牆要不要允許 `bastion` 直連？還是只開放從 `app` 過來的連線就好**
  1. 只允許 `app` 連 `db` 在架構上層級最清楚，但有一個管理上的盲點：如果 `app` 故障導致 SSH 無法進入，`db` 就完全失去管理路徑。因此決定同時開放 `bastion` 直連 `db`，保留一條獨立的緊急維護入口。
  2. 代價是 `db` 多了一個允許來源，攻擊面略微增加。但考量到 `bastion` 本身已是唯一對外入口且採用金鑰認證，這條額外路徑的風險在可接受範圍內。