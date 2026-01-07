# K3s 叢集設定

> 更新日期：2025-01-04

---

## 1. 安裝配置

| 項目 | 設定值 |
|------|--------|
| **叢集類型** | 單節點 (Single Node) |
| **Traefik** | ❌ 已停用 (`--disable=traefik`) |
| **ServiceLB** | ❌ 已停用（使用 MetalLB 替代） |
| **PVC 儲存路徑** | `/data/k3s-storage` |

---

## 2. 已安裝元件

| 元件 | 狀態 | 說明 |
|------|------|------|
| **MetalLB** | ✅ 已安裝 | Layer 2 模式，IP 池 192.168.0.200-220 |
| **cert-manager** | ✅ 已安裝 | TLS 憑證自動化 |
| **NVIDIA Device Plugin** | ✅ 已安裝 | GPU 支援（低優先，有需要時使用） |
| **Rancher** | ✅ 已安裝 | 叢集管理 UI |
| **Portainer** | ✅ 已安裝 | 容器管理 UI |
| **Helm** | ✅ 已安裝 | 套件管理 |
| **Nginx Ingress** | ✅ 已安裝 | Ingress Controller（192.168.0.202）|
| **PostgreSQL** | ✅ 已安裝 | 共用資料庫（database namespace）|
| **Gitea** | ✅ 已安裝 | Git 倉庫 + Container Registry + Actions |
| **ArgoCD** | ✅ 已安裝 | GitOps 持續部署（v3.2.2，Helm chart 9.1.9）|
| **Gitea Actions Runner** | ✅ 已安裝 | CI 自動化（gitea-runner namespace）|
| **Cloudflare Tunnel** | ✅ 已安裝 | 對外服務暴露（cloudflare-system namespace）|

---

## 3. MetalLB 配置

```yaml
IP 池範圍: 192.168.0.200 - 192.168.0.220
模式: Layer 2 Advertisement
可用 IP 數: 21 個
```

### 已分配 IP

| IP | 服務 | Port |
|----|------|------|
| 192.168.0.200 | Rancher | 443 |
| 192.168.0.201 | Portainer | 9000 |
| 192.168.0.202 | Nginx Ingress | 80, 443 |
| 192.168.0.203+ | (可用) | - |

---

## 4. TLS 憑證管理

### ClusterIssuer 配置

```yaml
# self-signed-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
```

| ClusterIssuer | 用途 | 適用範圍 |
|---------------|------|----------|
| `selfsigned-issuer` | 內網自簽憑證 | 管理介面（Rancher、Portainer、Gitea、ArgoCD） |

---

## 5. K3s 服務管理

```bash
# 檢查 K3s 狀態
sudo systemctl status k3s

# 重啟 K3s
sudo systemctl restart k3s

# 查看 K3s 日誌
sudo journalctl -u k3s -f

# 使用 K3s 內建 kubectl
sudo k3s kubectl get nodes
```

---

## 6. Namespace 架構

```
┌─────────────────────────────────────────────────────────────┐
│                    K3s Cluster                              │
├─────────────────────────────────────────────────────────────┤
│  系統層                                                      │
│  ├── kube-system          # K3s 核心元件                    │
│  ├── metallb-system       # MetalLB                         │
│  ├── cert-manager         # 憑證管理                        │
│  ├── ingress-nginx        # Ingress Controller              │
│  ├── cattle-system        # Rancher                         │
│  ├── portainer            # Portainer                       │
│  └── cloudflare-system    # Cloudflare Tunnel ✅ 新增       │
├─────────────────────────────────────────────────────────────┤
│  DevOps 層                                                   │
│  ├── gitea                # Git 倉庫 + Registry             │
│  ├── gitea-runner         # Gitea Actions Runner            │
│  └── argocd               # GitOps CD                       │
├─────────────────────────────────────────────────────────────┤
│  應用層                                                      │
│  ├── dev                  # 開發環境 ✅ 已建立               │
│  └── prod                 # 正式環境 ✅ 已建立               │
└─────────────────────────────────────────────────────────────┘
```

---

*返回 [README](README.md) | 下一章 [Namespace 配置](04-namespaces.md)*
