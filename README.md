# K3s Homelab GitOps 實作文檔

> 在資源受限的單節點環境下，實現企業級的 GitOps 開發流程

[![K3s](https://img.shields.io/badge/K3s-Single_Node-blue)](https://k3s.io/)
[![ArgoCD](https://img.shields.io/badge/CD-ArgoCD-orange)](https://argoproj.github.io/cd/)
[![Gitea](https://img.shields.io/badge/CI-Gitea_Actions-green)](https://gitea.io/)

---

## 這是什麼？

這是我個人 Homelab 的**完整建置紀錄**，目標是在一台實體主機上搭建：

- **GitOps 工作流** — ArgoCD 自動同步 K8s 部署
- **CI/CD Pipeline** — Gitea Actions 建構 + 推送 Image
- **內外網分流** — Tailscale 內網 + Cloudflare Tunnel 對外展示

### 硬體規格

| 項目 | 規格 |
|------|------|
| CPU | AMD Ryzen 5 5600 (6C/12T) |
| RAM | 32 GB |
| Storage | 2 TB NVMe SSD |
| OS | Ubuntu Server 24.04 LTS |

---

## 架構總覽

```
┌─────────────────────────────────────────────────────────────┐
│                      網路架構                                │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  【對外】Cloudflare Tunnel                                   │
│  └── https://demo.example.com → 展示應用                    │
│                                                             │
│  【內網】Tailscale                                           │
│  ├── Gitea (Git + Container Registry)                       │
│  ├── ArgoCD (GitOps CD)                                     │
│  ├── Rancher (K8s 管理)                                     │
│  └── Portainer (容器管理)                                    │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────────────────────────┐
│                     CI/CD 流程                               │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  App Repo (Gitea)          GitOps Repo (Gitea)              │
│       │                           │                         │
│       │ git push                  │                         │
│       ▼                           ▼                         │
│  ┌─────────┐               ┌─────────────┐                  │
│  │ Gitea   │──push image──▶│   ArgoCD    │                  │
│  │ Actions │               │  (自動同步)  │                  │
│  └─────────┘               └──────┬──────┘                  │
│       │                           │                         │
│       ▼                           ▼                         │
│  Container Registry         K8s Cluster                     │
│                            (dev / prod)                     │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 技術棧

| 類別 | 技術選型 |
|------|----------|
| **容器編排** | K3s (輕量 Kubernetes) |
| **GitOps CD** | ArgoCD (App of Apps 模式) |
| **CI** | Gitea Actions (DinD 架構) |
| **Git + Registry** | Gitea (自建) |
| **Ingress** | Nginx Ingress Controller |
| **負載均衡** | MetalLB (Layer 2) |
| **憑證管理** | cert-manager (Self-signed) |
| **內網存取** | Tailscale |
| **對外存取** | Cloudflare Tunnel |

---

## 文檔目錄

| 文檔 | 說明 |
|------|------|
| [01-hardware-and-os.md](docs/01-hardware-and-os.md) | 硬體配置與作業系統 |
| [02-storage.md](docs/02-storage.md) | 儲存空間規劃 (LVM) |
| [03-k3s-cluster.md](docs/03-k3s-cluster.md) | K3s 叢集設定與元件 |
| [04-namespaces.md](docs/04-namespaces.md) | Namespace 與資源配額 |
| [05-cicd.md](docs/05-cicd.md) | CI/CD 流程與分支策略 |
| [06-registry.md](docs/06-registry.md) | Container Registry 設定 |
| [07-networking.md](docs/07-networking.md) | 網路架構 |
| [08-management-ui.md](docs/08-management-ui.md) | 管理介面 |
| [09-team-permissions.md](docs/09-team-permissions.md) | 團隊權限規劃 |
| [10-installation-progress.md](docs/10-installation-progress.md) | 安裝進度追蹤 |
| [11-commands-reference.md](docs/11-commands-reference.md) | 常用指令參考 |
| [12-troubleshooting.md](docs/12-troubleshooting.md) | 問題排解紀錄 ⭐ |

> ⭐ 推薦閱讀：`12-troubleshooting.md` 記錄了實際遇到的問題與解法

---

## 為什麼這樣設計？

### K3s 而非 K8s
- 單節點環境，不需要多節點 HA
- K3s 原生整合輕量元件，減少資源消耗

### Gitea 而非 GitHub Actions
- 完整掌控 CI/CD 環境
- Container Registry 與 Git 整合
- 學習自建基礎設施的經驗

### ArgoCD App of Apps
- 透過 ApplicationSet 自動發現新應用
- 新增服務只需 `git push`，無需手動建立 ArgoCD Application

---

## 授權

本文檔以 [MIT License](LICENSE) 授權，歡迎參考使用。

---

## 聯絡

如有問題或建議，歡迎開 Issue 討論。
