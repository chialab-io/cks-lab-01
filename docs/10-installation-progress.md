# 安裝進度追蹤

> 更新日期：2026-01-06

---

## 1. 已完成項目

- [x] Ubuntu Server 24.04 LTS 安裝
- [x] LVM 儲存空間規劃
- [x] K3s 單節點安裝（停用 Traefik）
- [x] MetalLB 安裝與配置（192.168.0.200-220）
- [x] cert-manager 安裝
- [x] NVIDIA Device Plugin 安裝（Helm）
- [x] GPU 可用性驗證
- [x] Rancher 安裝（192.168.0.200）
- [x] Portainer 安裝（192.168.0.201:9000）
- [x] Helm 安裝
- [x] Tailscale 安裝（主機層級）
- [x] PVC ConfigMap 配置（`/data/k3s-storage`）
- [x] Nginx Ingress Controller 安裝（192.168.0.202 + hostPort）
- [x] Docker 移除、nerdctl 安裝、LVM 空間調整
- [x] PostgreSQL 安裝（共用資料庫）
- [x] Gitea 安裝（含 Container Registry）
- [x] Self-Signed ClusterIssuer 配置
- [x] 管理介面 Ingress 設定（Rancher、Portainer、Gitea）
- [x] ArgoCD 安裝（v3.2.2，Helm chart 9.1.9）
- [x] Gitea 帳號架構建立（wcwen、ci-bot、chialab 組織）
- [x] ci-bot Access Token 產生
- [x] dev/prod Namespace 建立（含 ResourceQuota、LimitRange）
- [x] gitops-manifests Repository 建立
- [x] ArgoCD 連接 Gitea
- [x] ArgoCD CLI 安裝
- [x] 主機 /etc/hosts 設定（argocd 域名解析）
- [x] 測試應用部署（nginx-test）
- [x] GitOps 流程驗證
- [x] Gitea Container Registry 啟用與測試
- [x] K3s Registry 配置（registries.yaml）
- [x] nerdctl insecure registry 配置
- [x] ci-bot Write 權限設定
- [x] Nginx Ingress proxy-body-size 設定
- [x] Image Push/Pull 測試成功
- [x] dev namespace imagePullSecret 建立
- [x] Gitea Actions Runner 安裝
- [x] Gitea Actions Runner 全域註冊 ✅（必須用 API 取得 Token）
- [x] CI Pipeline 完整驗證 ✅（Build → Push Image 成功）
- [x] Gitea ROOT_URL 修正為 HTTP ✅
- [x] DinD insecure-registries 配置 ✅
- [x] gitea-registry Service 多 port 配置 ✅
- [x] CI/CD 分支策略設計 ✅（develop/main 分離）
- [x] CI Workflow 模板化 ✅（ci-dev.yaml / ci-prod.yaml）
- [x] Gitea Organization Variables 設定 ✅（REGISTRY_URL）
- [x] Gitea Organization Secrets 設定 ✅（REGISTRY_PASSWORD）
- [x] template-app 模板儲存庫建立 ✅
- [x] test-frontend 端對端測試 ✅（模板 → CI → Registry → ArgoCD → 部署）
- [x] test-frontend Ingress 設定 ✅（內網 HTTP 存取）
- [x] prod namespace imagePullSecret 建立 ✅【2025-12-31】
- [x] main 分支保護設定 ✅【2025-12-31】（template-app、test-frontend）
- [x] gitops-manifests 結構整理 ✅【2025-12-31】
- [x] **Cloudflare 域名註冊 ✅**【2025-01-04】（cks-lab.uk）
- [x] **Cloudflare Tunnel 建立 ✅**【2025-01-04】（K8s Deployment 模式）
- [x] **Cloudflare DNS 設定 ✅**【2025-01-04】（內網分流：*.int.cks-lab.uk）
- [x] **對外服務路由設定 ✅**【2025-01-04】（demo.cks-lab.uk）
- [x] **內網 Ingress 設定 ✅**【2025-01-04】（git/argo/rancher/portainer.int.cks-lab.uk）
- [x] **cloudflared 加入 GitOps 管理 ✅**【2025-01-05】
  - infrastructure/ 資料夾結構建立
  - ArgoCD Application 建立（自動同步）
  - Health probe 配置（metrics server）
- [x] **Gitea ROOT_URL 更新 ✅**【2025-01-05】
  - 更新為 `http://git.int.cks-lab.uk/`
  - 透過 Helm values 設定
  - 停用 Helm 管理的 Ingress
- [x] **App of Apps 架構 ✅**【2025-01-06】
  - argocd-apps/ 資料夾建立
  - ApplicationSet（apps-dev、apps-prod、infrastructure）
  - Root Application 自動管理所有 ApplicationSet
- [x] **Rancher 內網 HTTPS 修正 ✅**【2025-01-06】
  - 修復 HTTP 存取時的 "Invalid CSRF token" 錯誤
  - rancher-int Ingress 改為 HTTPS + 自簽憑證
  - 簽發 rancher-int-tls 憑證

---

## 2. 待完成項目

| 優先級 | 項目 | 說明 |
|--------|------|------|
| **1** | RBAC 權限配置 | View Plus 角色給組員 |
| **2** | develop 分支保護設定 | 防止誤刪，允許直推 |
| **3** | Redis 安裝（可選） | 前後端常用 cache/session |

---

## 3. 當前進度總覽

```
Step 1: Gitea 帳號設定 ✅
           ↓
Step 2: 套用 Namespace 配置 ✅
           ↓
Step 3: ArgoCD 連接 Gitea ✅
           ↓
Step 4: 部署測試應用 ✅
           ↓
Step 5: Container Registry 配置 ✅
           ↓
Step 6: Gitea Actions Runner ✅
           ↓
Step 7: CI Pipeline 完整驗證 ✅【2025-12-30】
           ↓
Step 8: CI/CD 分支策略與模板 ✅【2025-12-30】
           ↓
Step 9: 端對端測試 ✅【2025-12-30】
           ↓
Step 10: Prod 環境配置 ✅【2025-12-31】
           ↓
Step 11: Cloudflare Tunnel + 域名 ✅【2025-01-04】
           ↓
Step 12: cloudflared GitOps 管理 ✅【2025-01-05】
        - infrastructure/ 資料夾結構
        - ArgoCD Application（自動同步）
        - Secret 手動管理（不進 Git）
           ↓
Step 13: App of Apps ✅【2025-01-06】
        - argocd-apps/ 資料夾建立
        - ApplicationSet 自動發現應用
        - root-app 管理所有 ApplicationSet
           ↓
Step 14: Rancher 內網 HTTPS ✅【2025-01-06】
        - 修復 CSRF token 錯誤
        - rancher-int Ingress 改為 HTTPS
           ↓
Step 15: Prod 部署流程驗證 ✅【2026-01-06】
        - Dev: test-frontend-dev.int.cks-lab.uk
        - Prod: test-frontend-prod.int.cks-lab.uk + demo.cks-lab.uk
        - Prod CI/CD 端對端驗證完成
           ↓
Step 16: RBAC 權限配置（待完成）
        - View Plus 角色給組員
```

---

## 4. 網路架構

### 域名分流策略

```
cks-lab.uk
│
├── *.cks-lab.uk（公網）
│   └── 走 Cloudflare Tunnel
│   └── 例：demo.cks-lab.uk
│
└── *.int.cks-lab.uk（內網）
    └── 走 Tailscale 直連
    └── 例：git.int.cks-lab.uk、argo.int.cks-lab.uk
```

### Cloudflare DNS 記錄

| Type | Name | Content | Proxy |
|------|------|---------|-------|
| A | `*.int` | `100.x.x.x` | ❌ DNS only |
| A | `@` | `100.x.x.x` | ❌ DNS only |

### Cloudflare Tunnel

| 項目 | 設定值 |
|------|--------|
| **Tunnel 名稱** | `cks-lab` |
| **Tunnel ID** | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| **部署方式** | K8s Deployment（2 replicas） |
| **Namespace** | `cloudflare-system` |
| **管理方式** | ArgoCD GitOps ✅ |
| **狀態** | ✅ Healthy |

### 服務 URL 總覽

| 服務 | 內網域名（新）| 協定 | 對外域名 |
|------|---------------|------|----------|
| Gitea | git.int.cks-lab.uk | HTTP | ❌ |
| ArgoCD | argo.int.cks-lab.uk | HTTP | ❌ |
| **Rancher** | **rancher.int.cks-lab.uk** | **HTTPS** ⚠️ | ❌ |
| Portainer | portainer.int.cks-lab.uk | HTTP | ❌ |
| Demo App | - | - | https://demo.cks-lab.uk |

> ⚠️ **Rancher 必須使用 HTTPS**：CSRF 保護機制要求，瀏覽器會警告自簽憑證，忽略即可。

---

## 5. Repository 架構

### 目前 Repository 清單

| Repository | 類型 | 分支保護 | 描述 |
|------------|------|----------|------|
| `gitops-manifests` | GitOps | ✅ main | K8s 部署清單，ArgoCD 自動同步 |
| `template-app` | 模板 | ✅ main | 專案模板，含 CI/CD workflow |
| `test-frontend` | 應用程式 | ✅ main | 端對端測試用 React App |

### gitops-manifests 結構

```
gitops-manifests/
├── argocd-apps/             ✅ 新增【2025-01-06】
│   ├── root-app.yaml         # Root Application
│   ├── apps-dev.yaml         # Dev ApplicationSet
│   ├── apps-prod.yaml        # Prod ApplicationSet
│   ├── infrastructure.yaml   # Infra ApplicationSet
│   └── README.md
├── apps/
│   ├── dev/
│   │   └── test-frontend/
│   │       └── deployment.yaml
│   └── prod/
│       └── test-frontend/
│           └── deployment.yaml  ✅ 新增【2026-01-06】
├── infrastructure/          ✅ 新增【2025-01-05】
│   └── cloudflared/
│       ├── namespace.yaml
│       ├── deployment.yaml
│       └── README.md
└── README.md
```

---

## 6. ArgoCD Applications

| Application | Path | Namespace | Sync Policy | 狀態 |
|-------------|------|-----------|-------------|------|
| root-app | argocd-apps | argocd | Automated | ✅ Synced |
| test-frontend | apps/dev/test-frontend | dev | Automated | ✅ Synced |
| infra-cloudflared | infrastructure/cloudflared | cloudflare-system | Automated | ✅ Synced |

---

## 7. 版本號格式

| 環境 | 格式 | 範例 | 說明 |
|------|------|------|------|
| Dev | `dev-<hash>` | `dev-a1b2c3d` | commit hash 前 7 碼 |
| Prod | `prod-<date>-<hash>` | `prod-20251230-a1b2c3d` | 日期 + hash，方便排序 |

---

*返回 [README](README.md) | 上一章 [團隊權限](09-team-permissions.md) | 下一章 [常用指令](11-commands-reference.md)*
