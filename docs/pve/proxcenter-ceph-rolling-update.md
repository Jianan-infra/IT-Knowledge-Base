---
layout: default
title: "ProxCenter 執行 PVE／Ceph Rolling Update：疑難排解紀錄"
date: 2026-08-26
categories: [PVE, ProxCenter, Ceph]
---

# ProxCenter 執行 PVE／Ceph Rolling Update：疑難排解紀錄

## 摘要

三節點 Proxmox VE 叢集使用 ProxCenter 執行 Rolling Update 時，先後遇到 Repository 驗證誤判、SSH 自動選錯網段，以及非 root SSH 使用者未通過通用免密碼 sudo 檢查。逐項排除後，流程成功進入套件升級與節點健康驗證。

本文不包含密碼、Private Key、Token Secret 或完整憑證。

## 環境概觀

- 三節點：`node10`、`node11`、`node12`
- 管理／SSH 網段範例：`192.168.10.0/24`
- Corosync 專用網段範例：`172.16.10.0/24`
- Ceph：3 MON、3 OSD
- dedicated SSH user：`proxcenter`
- 節點間採人工批准

## 1. APT 正常，但 Verify 回報 Repository Issues

手動 `apt update` 正常，ProxCenter 卻固定回報每台兩個錯誤。檢查發現 `/etc/apt/sources.list.d/` 留有 `*.BAK`，內容包含已停用的 Enterprise URL。APT 不使用這些備份檔，但 ProxCenter 仍掃描到它們。

先只在一台節點做可回復測試：

```bash
mkdir -p /root/apt-source-backup
mv /etc/apt/sources.list.d/*.BAK /root/apt-source-backup/
grep -Rni 'enterprise.proxmox.com' /etc/apt/ 2>/dev/null
apt update
```

重新 Verify 後問題數量由 6 降為 4，證實誤判來源；其餘節點做相同處理後歸零。

**經驗：** 管理工具可能自行掃描設定目錄，備份檔不應留在正式設定目錄中。

## 2. Rolling Update 選錯 SSH 網段

Test SSH Connection 曾成功，Rolling Update 卻嘗試連線 Corosync 網段並 timeout。原因是 Rolling Update Configuration 使用 **Auto-detect**，選到叢集專用網卡。

修正為明確指定管理位址：

```text
node10 → 192.168.10.10
node11 → 192.168.10.11
node12 → 192.168.10.12
```

不需修改 Corosync，也不應只為管理工具開放叢集專用網段的 SSH 路由。

## 3. Maintenance mode 顯示 lacks passwordless sudo

`proxcenter` 原本只有特定命令的 `NOPASSWD` allowlist。特定命令可成功，ProxCenter 仍判定缺少 passwordless sudo。關鍵測試：

```bash
sudo -u proxcenter sudo -n true
echo $?
sudo -u proxcenter sudo -n id
```

`sudo -n true` 失敗代表未通過該版本的通用前置檢查。依實際錯誤要求，最終使用：

```text
proxcenter ALL=(ALL) NOPASSWD: ALL
```

並驗證：

```bash
visudo -c -f /etc/sudoers.d/proxcenter
sudo -u proxcenter sudo -n true
sudo -u proxcenter sudo -n id
```

> **安全風險：** 此帳號成為 root-equivalent。SSH Key 必須獨立保管、限制來源、定期輪替並監控登入；若新版支援精確 allowlist，應回復最小權限。

## 4. 本機儲存 VM 無法自動遷移

部分 VM 位於 local storage。因首次操作關閉 **Shutdown VMs with local storage**，它們被安全跳過，而非直接關機。若節點需要 reboot，仍須安排停機窗口或先遷移磁碟。

## 成功流程證據

```text
Rolling update started
Set Ceph noout flag
[node12] Starting node update
[node12] Enabling maintenance mode
[node12] Waiting for HA migrations
[node12] Migrating non-HA VMs
[node12] Running apt update && apt full-upgrade
[node12] Verifying node health
```

## 更新前檢查

- [ ] `pvecm status` 顯示 `Quorate: Yes`
- [ ] 三台節點 online
- [ ] `ceph -s` 為 `HEALTH_OK`，OSD 全部 up/in
- [ ] `ha-manager status` 顯示 quorum OK、LRM active
- [ ] SSH Address 固定為管理網段
- [ ] `visudo` 與 `sudo -n` 測試通過
- [ ] 已確認 local storage VM 的策略
- [ ] Abort on failure、Manual approval、Wait for Ceph HEALTH_OK 已開啟
- [ ] Minimum healthy nodes 設為 2

## 每台節點完成後

先不要立即批准下一台，確認：

```bash
pvecm status
ceph -s
ha-manager status
pveversion -v
```

並確認節點已重新加入、Quorum 正常、Ceph 回到 `HEALTH_OK`、HA/LRM active，且 `noout` 沒有意外殘留。

## 結論

本次是三個獨立條件疊加：

1. Repository Verify 掃描到停用的 `*.BAK`
2. Auto-detect 選到 Corosync 網段
3. dedicated user 的最小 sudo allowlist 不符合當時版本的通用檢查

有效的排錯策略是每次只改一個變因，並用問題數量、目標 IP、`sudo -n` exit code，以及 Cluster／Ceph／HA 健康狀態驗證。

