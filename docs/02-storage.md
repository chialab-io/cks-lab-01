# 儲存空間規劃

> 更新日期：2025-12-25

---

## 1. 分區配置

```
┌─────────────────────┬────────┬──────────────────────────────┐
│ 分區                │ 大小   │ 用途                          │
├─────────────────────┼────────┼──────────────────────────────┤
│ /                   │ 200G   │ 系統核心、基礎軟體            │
├─────────────────────┼────────┼──────────────────────────────┤
│ /var/lib/rancher    │ 350G   │ K3s 核心資料                  │
│                     │        │ containerd images            │
├─────────────────────┼────────┼──────────────────────────────┤
│ /data               │ 1.2TB  │ K3s PVC 儲存                  │
│                     │        │ 本地 Registry                 │
│                     │        │ 資料庫資料                    │
│                     │        │ 備份                          │
├─────────────────────┼────────┼──────────────────────────────┤
│ 保留未分配           │ 70G   │ 彈性擴展空間                  │
└─────────────────────┴────────┴──────────────────────────────┘

總計：1,820 GB
```

---

## 2. LVM 自動掛載設定（fstab）

> ⚠️ **重要**：務必設定 fstab，否則重開機後 LVM 不會自動掛載，導致 K3s 資料遺失！

### /etc/fstab 設定

```bash
# 查看目前設定
cat /etc/fstab
```

**必須包含以下條目**：

```
/dev/ubuntu-vg/k3s-lv   /var/lib/rancher  ext4  defaults  0 2
/dev/ubuntu-vg/data-lv  /data             ext4  defaults  0 2
```

### 新增 fstab 條目

```bash
# 新增 K3s 資料目錄掛載
echo '/dev/ubuntu-vg/k3s-lv /var/lib/rancher ext4 defaults 0 2' | sudo tee -a /etc/fstab

# 新增 /data 掛載（如尚未設定）
echo '/dev/ubuntu-vg/data-lv /data ext4 defaults 0 2' | sudo tee -a /etc/fstab

# 驗證設定
cat /etc/fstab | grep -E "k3s|data"
```

### 驗證掛載狀態

```bash
# 檢查 LVM 是否已掛載（Attr 應為 -wi-ao----，o 表示 open/mounted）
sudo lvs

# 測試 fstab 設定是否正確（不會實際重掛載已掛載的分區）
sudo mount -a
```

---

## 3. /data 子目錄結構

```
/data
├── k3s-storage/        # K3s PVC 持久化儲存
├── registry/           # (規劃) 本地 Container Registry
├── gitea/              # Gitea 資料
├── postgresql/         # PostgreSQL 資料
└── backups/            # (規劃) 備份資料
```

---

## 4. 重要路徑

| 路徑 | 用途 |
|------|------|
| `/etc/rancher/k3s/k3s.yaml` | kubeconfig 檔案 |
| `/etc/rancher/k3s/registries.yaml` | K3s Registry 配置 |
| `/var/lib/rancher/k3s/` | K3s 資料目錄 |
| `/run/k3s/containerd/containerd.sock` | containerd socket（nerdctl 使用） |
| `/etc/nerdctl/nerdctl.toml` | nerdctl 配置檔 |
| `/data/k3s-storage/` | PVC 持久化儲存 |
| `/data/postgresql/` | PostgreSQL 資料 |
| `/data/gitea/` | Gitea 資料 |
| `/data/backups/` | (規劃) 備份資料 |
| `/etc/hosts` | 主機 DNS 解析（含 argocd 域名） |
| `/root/.docker/config.json` | nerdctl Registry 認證 |
| `/etc/fstab` | LVM 自動掛載設定 |

---

## 5. 儲存監控指令

```bash
# 檢查各分區使用量
df -h / /var/lib/rancher /data

# 檢查 LVM 狀態（Attr 欄位 o 表示已掛載）
sudo lvs

# 檢查 fstab 設定
cat /etc/fstab | grep -E "k3s|data"

# 檢查 PVC 使用情況
kubectl get pvc -A

# 檢查 PV 狀態
kubectl get pv
```

---

## 6. 擴展空間

如需擴展儲存空間：

```bash
# 1. 檢查可用空間
sudo vgs

# 2. 擴展邏輯卷（例如擴展 /data）
sudo lvextend -L +50G /dev/ubuntu-vg/lv-data
sudo resize2fs /dev/ubuntu-vg/lv-data

# 3. 驗證
df -h /data
```

---

*返回 [README](README.md)*
