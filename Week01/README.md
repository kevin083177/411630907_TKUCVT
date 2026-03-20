# W01｜虛擬化概論、環境建置與 Snapshot 機制

## 環境資訊
- Host OS：Windows 11
- VM 名稱：vct-w01-411630907
- Ubuntu: 24.04
- Docker 版本：29.3.0 build 5927d80
- Docker Compose 版本：v5.1.0

## VM 資源配置驗證

| 項目 | VMware 設定值 | VM 內命令 | VM 內輸出 |
|---|---|---|---|
| CPU | 2 vCPU | `lscpu \| grep "^CPU(s)"` | 2 |
| 記憶體 | 4 GB | `free -h \| grep Mem` | 5.6Gi |
| 磁碟 | 40 GB | `df -h /` | 40G |
| Hypervisor | VMware | `lscpu \| grep Hypervisor` | VMware |

## 四層驗收證據
- [ ]() ① Repository：`cat /etc/apt/sources.list.d/docker.list` 輸出
- [ ] ② Engine：`dpkg -l | grep docker-ce` 輸出
- [ ] ③ Daemon：`sudo systemctl status docker` 顯示 active
- [ ] ④ 端到端：`sudo docker run hello-world` 成功輸出
- [ ] Compose：`docker compose version` 可執行

## 容器操作紀錄
- [ ] nginx：`sudo docker run -d -p 8080:80 nginx` + `curl localhost:8080` 輸出
- [ ] alpine：`sudo docker run -it --rm alpine /bin/sh` 內部命令與輸出
- [ ] 映像列表：`sudo docker images` 輸出

## Snapshot 清單

| 名稱 | 建立時機 | 用途說明 | 建立前驗證 |
|---|---|---|---|
| clean-baseline | Docker 安裝完成後、任何額外設定前 | 代表系統乾淨可用的起始狀態，作為最保底的回復點 | `hostnamectl`、`ip route`、`docker --version`、`docker compose version`、`systemctl status docker` 全部通過 |
| docker-ready | 拉取 nginx、alpine 映像確認可用後 | `alpine` 映像確認可用後代表 Docker 環境含映像的完整工作狀態，是本次實驗的主要回復點 | `systemctl status docker`、`hello-world`、`docker images` 確認 nginx 與 alpine 都在 |

## 故障演練三階段對照

| 項目 | 故障前（基線） | 故障中（注入後） | 回復後 |
|---|---|---|---|
| docker.list 存在 | 是 | 否 | 是 |
| apt-cache policy 有候選版本 | 是 | 否 | 是 |
| docker 重裝可行 | 是 | 否 | 是 |
| hello-world 成功 | 是 | N/A | 是 |
| nginx curl 成功 | 是 | N/A | 是 |

## 手動修復 vs Snapshot 回復

| 面向 | 手動修復 | Snapshot 回復 |
|---|---|---|
| 所需時間 | 約 30 秒 | 約 1–2 分鐘 |
| 適用情境 | 故障明確、改動單一、知道哪裡壞掉 | 改動範圍不確定、多個設定檔被動過、不知道從哪裡壞的 |
| 風險 | 修錯方向可能讓問題更複雜 | 回復期間 VM 短暫停機；回復後須重做 snapshot 之後的所有變更 |

## Snapshot 保留策略
- 新增條件：每次安裝新工具或大幅修改設定前，且當前狀態已完整驗證通過時。
- 保留上限：最多 3 個活躍 snapshot。
- 刪除條件：已有更新的穩定節點，且確認舊節點不再需要回復時，刪除最舊的那個。

## 最小可重現命令鏈
```bash
# 1. 確認健康基線
ls /etc/apt/sources.list.d/
apt-cache policy docker-ce | head -5
sudo systemctl status docker --no-pager
sudo docker run --rm hello-world

# 2. 注入故障
sudo mv /etc/apt/sources.list.d/docker.list /etc/apt/sources.list.d/docker.list.broken
sudo apt update
apt-cache policy docker-ce | head -5   # 應顯示無候選版本

# 3. 關機並回復 snapshot
sudo poweroff
# → VMware: Snapshot Manager → docker-ready → Revert → 開機

# 4. 回復後驗證
ls /etc/apt/sources.list.d/
apt-cache policy docker-ce | head -5
sudo systemctl status docker --no-pager
sudo docker run --rm hello-world
sudo docker images
```

## 排錯紀錄
- 症狀：`apt update` 出現 404 錯誤，`docker-ce` 無候選安裝版本
- 診斷：`ls /etc/apt/sources.list.d/` 發現 `docker.list` 不見，只剩 `docker.list.broken`
- 修正：`sudo mv /etc/apt/sources.list.d/docker.list.broken /etc/apt/sources.list.d/docker.list`
- 驗證：`sudo apt update` 成功，`apt-cache policy docker-ce` 重新出現候選版本

## 設計決策
選擇在 VM 裡跑 Docker，而不是直接裝在實體機上。主要原因是有 snapshot 可以反悔。實驗過程會故意把東西搞壞，如果直接裝在實體機，改爛了不知道怎麼救；在 VM 裡頂多 revert 回去重來，不用擔心搞壞整台電腦。