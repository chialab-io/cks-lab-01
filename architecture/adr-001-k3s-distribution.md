# ADR-001: 選擇 K3s 作為 Kubernetes 發行版

> **狀態**：Accepted  
> **日期**：2025-12-23  
> **決策者**：@wcwen

---

## 背景 (Context)

CKS 專案需要在家用伺服器上建置 Kubernetes 叢集，用於：
- 學習 K8s 生態系（ArgoCD、Helm、GitOps）
- 建立完整的 CI/CD 流程
- 部署實驗性前後端應用

硬體資源限制：
- CPU：AMD Ryzen 5 5600 (6C/12T)
- RAM：32GB
- Storage：2TB SSD
- 環境：單節點，無需 HA

---

## 決策 (Decision)

我們選擇 **K3s** 作為 Kubernetes 發行版。

K3s 是 Rancher Labs 開發的輕量 K8s 發行版，專為邊緣運算和資源受限環境設計。

---

## 考量的替代方案

### 方案 A：完整 K8s (kubeadm)

| 優點 | 缺點 |
|------|------|
| 完整功能，與雲端一致 | 安裝複雜，需要多個元件 |
| 社群資源豐富 | 資源消耗大（etcd 至少 2Gi RAM） |

**不選擇的原因**：資源開銷過大，單節點不需要 etcd HA

### 方案 B：MicroK8s (Canonical)

| 優點 | 缺點 |
|------|------|
| Snap 安裝簡單 | 與非 Ubuntu 系統整合較差 |
| 模組化 add-on | 社群生態較小 |

**不選擇的原因**：add-on 體系與 Helm chart 生態整合不如 K3s

### 方案 C：K0s (Mirantis)

| 優點 | 缺點 |
|------|------|
| 單一執行檔 | 社群較 K3s 小 |
| 無額外依賴 | 文檔較少 |

**不選擇的原因**：生態系成熟度不足

---

## 資源影響

| 資源 | 預估用量 | 備註 |
|------|----------|------|
| CPU | ~0.5 cores | 閒置時 |
| Memory | ~500Mi (server) + ~300Mi (agent) | 使用 SQLite 替代 etcd |
| Storage | 1TB (LVM 分配) | `/var/lib/rancher` |

---

## 相關服務

| 服務 | 影響 | 需要的變更 |
|------|------|-----------|
| etcd | 不需安裝 | K3s 使用內建 SQLite |
| Traefik | 預設安裝 | 見 ADR-002（已停用） |
| MetalLB | 需額外安裝 | Layer 2 模式，取代雲端 LoadBalancer |

---

## 驗證方式

- [x] `kubectl get nodes` 顯示 Ready
- [x] `kubectl get pods -A` 所有系統 Pod Running
- [x] 記憶體使用穩定在 ~800Mi（K3s 本身）

---

## 後果 (Consequences)

### 正面影響
- 安裝簡單，單指令完成
- 資源佔用低，留更多空間給應用
- 與標準 K8s API 完全相容

### 負面影響
- 部分 K8s 功能簡化（如 etcd 管理）
- 高可用需要額外配置（單節點無此需求）

---

## 參考資料

- [K3s 官方文檔](https://docs.k3s.io/)
- [K3s vs K8s vs MicroK8s 比較](https://www.cncf.io/blog/)
