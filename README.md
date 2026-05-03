# Raspberry Pi 4 (Ubuntu) 系統優化與瘦身指南

這份文件紀錄了如何將 RPi 4 上的預設 Ubuntu 系統進行「去雲端化」與「極致輕量化」的過程，特別適用於僅運行 Docker 容器的環境。

---

## 1. 優化目標清單
針對實體 RPi 4 環境，我們移除以下在雲端環境才需要的肥大套件：

| 套件名稱 | 功能描述 | 移除原因 |
| :--- | :--- | :--- |
| **cloud-init** | 雲端主機初始設定工具 | RPi 是實體機，不需要透過雲端 metadata 初始化。 |
| **snapd** | Canonical 套件格式 | 產生大量 Loop 裝置，對 Docker 使用者來說純屬冗餘。 |
| **needrestart** | 程序重啟掃描器 | 導致 `apt` 結尾出現惱人的 `No VM guests...` 訊息。 |
| **apport** | 自動錯誤報告系統 | 背景掃描 Log 浪費資源。 |
| **apparmor** | 安全防護核心模組 | 簡化內核負載，減少對 Docker 的權限限制。 |
| **avahi-daemon** | mDNS 自動發現服務 | 減少不必要的網路廣播封包。 |
| **unattended-upgrades** | 安全性自動更新 | 避免背景佔用 `apt lock`，改由人工手動控管。 |

---

## 2. 自動化優化腳本 (`cleanup.sh`)

將以下內容存成 `cleanup.sh`，執行 `sudo bash cleanup.sh` 即可一鍵優化。
```bash
#!/bin/bash
# =================================================================
# 腳本名稱: cleanup.sh
# 適用環境: Ubuntu on Raspberry Pi 4
# 目標: 移除所有非必要服務 (cloud-init, snapd, apparmor, etc.)
# =================================================================

# 確保以 root 權限執行
if [[ $EUID -ne 0 ]]; then
   echo "❌ 此腳本必須以 root 權限執行，請使用 sudo bash $0"
   exit 1
fi

echo "--- 🚀 開始系統極致瘦身程序 ---"

# 1. 移除雲端與自動化相關
echo "📦 正在移除 cloud-init..."
apt purge -y cloud-init
rm -rf /etc/cloud/ /var/lib/cloud/

# 2. 移除 Snapd 系統
echo "📦 正在移除 Snapd..."
snap list 2>/dev/null | awk 'NR>1 {print $1}' | xargs -I{} snap remove {} 2>/dev/null
apt purge -y snapd
rm -rf ~/snap /var/cache/snapd/ /var/snap /var/lib/snapd

# 3. 移除 AppArmor (核心安全模組)
echo "📦 正在移除 AppArmor..."
systemctl stop apparmor
systemctl disable apparmor
apt purge -y apparmor
rm -rf /etc/apparmor.d/ /etc/apparmor/

# 4. 移除開發與診斷擾人套件
echo "📦 正在移除 needrestart & apport..."
apt purge -y needrestart apport

# 5. 移除網路探索與自動更新
echo "📦 正在移除 avahi-daemon & unattended-upgrades..."
apt purge -y avahi-daemon unattended-upgrades

# 6. 深度清理系統殘留
echo "🧹 執行最後的自動清理與依賴移除..."
apt autoremove --purge -y
apt clean

echo "--- ✅ 系統優化完成！ ---"
echo "💡 提示：請重啟系統以完全釋放資源：sudo reboot"
```

## 3. 維護與檢查工具
優化後，可以使用以下指令確保系統維持簡潔：

檢查開機耗時： systemd-analyze blame

查看套件空間佔用： dpigs -n 10 (需安裝 debian-goodies)

確認 Docker 狀態： docker info (移除 AppArmor 後會出現警告，但不影響運作)


## 4. 常見問答 (FAQ)
移除 AppArmor 會影響 Docker 嗎？

Docker 預設會利用 AppArmor 強化隔離。移除後安全性會略微下降，但對個人實體主機而言，效能提升與權限管理較為方便。

移除後可以恢復嗎？

可以，隨時透過 sudo apt install <套件名> 重新裝回。

