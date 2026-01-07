# 網路架構

> 更新日期：2026-01-06

---

## 1. 網路架構總覽

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           cks-lab.uk 網路架構                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  【對外展示】Cloudflare Tunnel                                               │
│  ├── https://demo.cks-lab.uk      → test-frontend (展示)                   │
│  └── https://*.cks-lab.uk         → (未來更多對外服務)                       │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  【團隊內網】Tailscale + Cloudflare DNS                                      │
│  ├── http://git.int.cks-lab.uk       → Gitea                               │
│  ├── http://argo.int.cks-lab.uk      → ArgoCD                              │
│  ├── https://rancher.int.cks-lab.uk  → Rancher ⚠️ 需 HTTPS                 │
│  └── http://portainer.int.cks-lab.uk → Portainer                           │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  【CI/CD 內部】K8s DNS（不變）                                                │
│  └── gitea-http.gitea.svc.cluster.local:3000                               │
│                                                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  【舊版內網】Tailscale + hosts（仍可用，但建議遷移到新域名）                    │
│  ├── https://gitea.cks-lab-01.tailXXXXXX.ts.net                            │
│  ├── https://argocd.cks-lab-01.tailXXXXXX.ts.net                           │
│  └── ...                                                                    │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. 對外存取（Cloudflare Tunnel）

**使用者：** 外部人員（客戶、訪客）  
**用途：** 專案展示、Demo

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   外部人員  │─────>│ Cloudflare  │─────>│ cloudflared │─────>│ Nginx       │
│  (Internet) │      │   Edge      │      │   (K8s Pod) │      │ Ingress     │
└─────────────┘      └─────────────┘      └─────────────┘      └──────┬──────┘
                                                                      │
                                                               ┌──────┴──────┐
                                                               │  展示應用    │
                                                               └─────────────┘
```

### Cloudflare Tunnel 配置

| 項目 | 設定值 |
|------|--------|
| **Tunnel 名稱** | `cks-lab` |
| **Tunnel ID** | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| **部署方式** | K8s Deployment（2 replicas） |
| **Namespace** | `cloudflare-system` |
| **狀態** | ✅ Healthy |

### 對外服務路由

| 域名 | 目標服務 | 說明 |
|------|----------|------|
| `demo.cks-lab.uk` | `ingress-nginx-controller.ingress-nginx.svc.cluster.local:80` | Prod 環境展示 |

### 新增對外服務步驟

1. 到 Cloudflare Zero Trust → Networks → Tunnels → `cks-lab`
2. 點擊 "Public Hostnames" → "Add a public hostname"
3. 填入：
   - Subdomain: `<your-app>`
   - Domain: `cks-lab.uk`
   - Type: `HTTP`
   - URL: `ingress-nginx-controller.ingress-nginx.svc.cluster.local:80`
4. 在 K8s 建立對應的 Ingress rule

---

## 3. 對內存取（團隊開發）

**使用者：** 管理員 + 團隊成員  
**用途：** 開發、測試、K8s 部署操作

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│  管理員/    │─────>│ Cloudflare  │─────>│  Tailscale  │
│  團隊成員   │ DNS  │   DNS       │      │   IP        │
└─────────────┘      │ (*.int)     │      │ 100.96.x.x  │
                     └─────────────┘      └──────┬──────┘
                                                 │
                                          ┌──────┴──────┐
                                          │ Nginx       │
                                          │ Ingress     │
                                          └──────┬──────┘
                                                 │
                     ┌───────────────────────────┼───────────────────────────┐
                     │                           │                           │
              ┌──────┴──────┐            ┌───────┴───────┐           ┌───────┴───────┐
              │   Gitea     │            │   ArgoCD      │           │  dev/prod     │
              │   Rancher   │            │   Portainer   │           │   Apps        │
              └─────────────┘            └───────────────┘           └───────────────┘
```

### 內網配置

| 項目 | 設定 |
|------|------|
| **VPN** | Tailscale（主機層級安裝） |
| **Tailscale IP** | 100.96.128.95 |
| **DNS 解析** | Cloudflare DNS（`*.int.cks-lab.uk` → Tailscale IP） |
| **負載均衡** | MetalLB (192.168.0.200-220) |
| **Ingress** | Nginx Ingress Controller |

### Cloudflare DNS 記錄

| Type | Name | Content | Proxy |
|------|------|---------|-------|
| A | `*.int` | `100.96.128.95` | ❌ DNS only（灰色雲朵） |
| A | `@` | `100.96.128.95` | ❌ DNS only（灰色雲朵） |

> ⚠️ **重要**：內網服務的 DNS 記錄必須關閉 Proxy（灰色雲朵），否則流量會走 Cloudflare 而非直連 Tailscale。

---

## 4. 域名分流策略

```
cks-lab.uk
│
├── *.cks-lab.uk（公網）
│   └── 走 Cloudflare Tunnel
│   └── 用於：對外展示、客戶 Demo
│   └── 例：demo.cks-lab.uk、app.cks-lab.uk
│
└── *.int.cks-lab.uk（內網）
    └── 走 Tailscale 直連（Cloudflare DNS-only）
    └── 用於：團隊開發、管理介面
    └── 例：git.int.cks-lab.uk、argo.int.cks-lab.uk
```

---

## 5. 服務 URL 對照表

### 新版域名（推薦）

| 服務 | 內網域名（團隊開發） | 協定 | 對外域名（展示） |
|------|----------------------|------|------------------|
| **Gitea** | git.int.cks-lab.uk | HTTP | ❌ 不對外 |
| **ArgoCD** | argo.int.cks-lab.uk | HTTP | ❌ 不對外 |
| **Rancher** | rancher.int.cks-lab.uk | **HTTPS** ⚠️ | ❌ 不對外 |
| **Portainer** | portainer.int.cks-lab.uk | HTTP | ❌ 不對外 |
| **Demo App** | test-frontend-prod.int.cks-lab.uk | HTTP | https://demo.cks-lab.uk |

> ⚠️ **Rancher 必須使用 HTTPS**：Rancher 的 CSRF 保護機制要求 HTTPS，HTTP 存取會導致 "Invalid CSRF token" 錯誤。

### 舊版域名（仍可用）

| 服務 | 內網域名（需 hosts 設定） | 備用位址 |
|------|----------------------|----------|
| **Rancher** | `https://rancher.cks-lab-01.tailXXXXXX.ts.net` | `https://192.168.0.200` |
| **Portainer** | `https://portainer.cks-lab-01.tailXXXXXX.ts.net` | `http://192.168.0.201:9000` |
| **Gitea** | `https://gitea.cks-lab-01.tailXXXXXX.ts.net` | - |
| **ArgoCD** | `https://argocd.cks-lab-01.tailXXXXXX.ts.net` | - |

---

## 6. 團隊成員設定

### 新版設定（推薦）✅

**只需要安裝 Tailscale，零額外設定！**

1. 安裝 [Tailscale](https://tailscale.com/)
2. 加入團隊網路
3. 直接存取 `*.int.cks-lab.uk`

> ⚠️ **注意**：Rancher 需使用 `https://rancher.int.cks-lab.uk`（瀏覽器會警告自簽憑證，忽略即可）

### 舊版設定（仍可用）

如需使用舊版 `*.cks-lab-01.tailXXXXXX.ts.net` 域名，需設定 hosts：

**Windows:** `C:\Windows\System32\drivers\etc\hosts`  
**Mac/Linux:** `/etc/hosts`

```
# K3s 內網服務（透過 Tailscale）- 舊版
100.96.128.95 rancher.cks-lab-01.tailXXXXXX.ts.net
100.96.128.95 portainer.cks-lab-01.tailXXXXXX.ts.net
100.96.128.95 gitea.cks-lab-01.tailXXXXXX.ts.net
100.96.128.95 argocd.cks-lab-01.tailXXXXXX.ts.net
```

---

## 7. Cloudflare Tunnel 部署配置

### cloudflared Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cloudflared
  namespace: cloudflare-system
  labels:
    app: cloudflared
spec:
  replicas: 2
  selector:
    matchLabels:
      app: cloudflared
  template:
    metadata:
      labels:
        app: cloudflared
    spec:
      containers:
      - name: cloudflared
        image: cloudflare/cloudflared:2024.12.2
        args:
        - tunnel
        - --no-autoupdate
        - run
        env:
        - name: TUNNEL_TOKEN
          valueFrom:
            secretKeyRef:
              name: cloudflared-token
              key: TUNNEL_TOKEN
        resources:
          requests:
            cpu: 10m
            memory: 64Mi
          limits:
            cpu: 100m
            memory: 128Mi
      restartPolicy: Always
```

### 檢查 Tunnel 狀態

```bash
# 檢查 Pod 狀態
kubectl get pods -n cloudflare-system

# 檢查 logs
kubectl logs -n cloudflare-system -l app=cloudflared --tail=20
```

---

## 8. 網路檢查指令

```bash
# Tailscale 狀態
tailscale status

# 檢查所有 Ingress
kubectl get ingress -A

# 檢查 MetalLB 分配的 IP
kubectl get svc -A | grep LoadBalancer

# 測試內網連線（新版域名）
curl http://git.int.cks-lab.uk
curl -k https://rancher.int.cks-lab.uk  # Rancher 需 HTTPS + 跳過憑證驗證

# 測試內網連線（舊版域名）
curl -k https://gitea.cks-lab-01.tailXXXXXX.ts.net

# 測試對外連線
curl -I https://demo.cks-lab.uk

# 檢查 cloudflared Pod
kubectl get pods -n cloudflare-system
kubectl logs -n cloudflare-system -l app=cloudflared --tail=20
```

---

## 9. 流量路徑總覽

### 對外請求（訪客看展示頁）

```
訪客瀏覽器
  → https://demo.cks-lab.uk
  → Cloudflare Edge（終止 SSL）
  → Cloudflare Tunnel
  → cloudflared Pod（K8s 內）
  → ingress-nginx-controller:80
  → Ingress rule（host: demo.cks-lab.uk）
  → test-frontend Service
  → test-frontend Pod
```

### 內網請求（開發者存取 Gitea）

```
開發者電腦（Tailscale 已連線）
  → http://git.int.cks-lab.uk
  → Cloudflare DNS（A 記錄，DNS-only）
  → 100.96.128.95（Tailscale IP）
  → Nginx Ingress
  → Gitea Service
  → Gitea Pod
```

### 內網請求（開發者存取 Rancher）

```
開發者電腦（Tailscale 已連線）
  → https://rancher.int.cks-lab.uk
  → Cloudflare DNS（A 記錄，DNS-only）
  → 100.96.128.95（Tailscale IP）
  → Nginx Ingress（TLS 終止，自簽憑證）
  → Rancher Service（HTTPS backend）
  → Rancher Pod
```

### CI/CD 內部請求

```
Gitea Actions Runner Pod
  → gitea-http.gitea.svc.cluster.local:3000
  → CoreDNS 解析
  → Gitea Service ClusterIP
  → Gitea Pod
```

> 三種流量路徑完全獨立，互不影響。

---

*返回 [README](README.md) | 上一章 [Registry](06-registry.md) | 下一章 [管理介面](08-management-ui.md)*
