# 硬體配置與作業系統

> 更新日期：2025-12-23

---

## 1. 硬體配置

| 項目 | 規格 |
|------|------|
| **CPU** | AMD Ryzen 5 5600 (6C/12T) |
| **RAM** | 32 GB |
| **儲存** | 2 TB NVMe SSD |
| **GPU** | NVIDIA GPU (CUDA 支援，低優先使用) |
| **可用空間** | 1.82 TiB (1,820 GB) |

---

## 2. 作業系統與基礎軟體

| 項目 | 版本/設定 |
|------|-----------|
| **OS** | Ubuntu Server 24.04 LTS |
| **容器編排** | K3s (輕量 Kubernetes) |
| **容器運行時** | containerd（K3s 原生） |
| **容器 CLI** | nerdctl v2.2.0（指向 K3s containerd） |
| **儲存管理** | LVM (`ubuntu-vg`) |
| **Swap** | 已停用（K8s 最佳實踐） |
| **套件管理** | Helm ✅ |
| **ArgoCD CLI** | ✅ 已安裝 |

---

## 3. 系統優化設定

### Swap 停用

K8s 建議停用 Swap 以確保資源管理的準確性：

```bash
# 檢查 Swap 狀態
free -h

# 永久停用（已設定）
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### LVM 管理

```bash
# 查看 LVM 狀態
sudo lvs
sudo vgs
sudo pvs

# 擴展邏輯卷（如需要）
sudo lvextend -L +50G /dev/ubuntu-vg/lv-data
sudo resize2fs /dev/ubuntu-vg/lv-data
```

---

## 4. GPU 配置

> ⚠️ **低優先級**：GPU 功能已就緒但目前非主要用途

### 現狀

- NVIDIA Device Plugin 已安裝並運作
- dev/prod namespace 已禁用 GPU（ResourceQuota 限制）
- 有需要時可由管理員調整

### 使用方式（未來需要時）

1. 由管理員調整 ResourceQuota 允許 GPU
2. 在 Pod spec 中加入：
   ```yaml
   resources:
     limits:
       nvidia.com/gpu: 1
   ```

### 驗證 GPU 可用性

```bash
kubectl describe node | grep -A5 "nvidia.com/gpu"
```

---

*返回 [README](README.md)*
