# 管理介面與 Ingress

> 更新日期：2025-01-06

---

## 1. 存取位址

### 新版域名（推薦）✅

無需設定 hosts，只需 Tailscale 連線即可存取。

| 服務 | 內網域名 | 協定 | 用途 |
|------|----------|------|------|
| **Gitea** | git.int.cks-lab.uk | HTTP | 代碼倉庫 + Container Registry |
| **ArgoCD** | argo.int.cks-lab.uk | HTTP | GitOps 部署管理 |
| **Rancher** | rancher.int.cks-lab.uk | **HTTPS** ⚠️ | K8s 叢集管理 |
| **Portainer** | portainer.int.cks-lab.uk | HTTP | 容器視覺化管理 |

> ⚠️ **Rancher 必須使用 HTTPS**：Rancher 的 CSRF 保護機制要求 HTTPS，HTTP 存取會導致 "Invalid CSRF token" 錯誤。瀏覽器會警告自簽憑證，忽略即可。

### 舊版域名（需 hosts 設定）

| 服務 | 內網域名（Tailscale） | 備用位址 | 用途 |
|------|----------------------|----------|------|
| **Rancher** | `https://rancher.cks-lab-01.tailXXXXXX.ts.net` | `https://192.168.0.200` | K8s 叢集管理 |
| **Portainer** | `https://portainer.cks-lab-01.tailXXXXXX.ts.net` | `http://192.168.0.201:9000` | 容器視覺化管理 |
| **Gitea** | `https://gitea.cks-lab-01.tailXXXXXX.ts.net` | - | 代碼倉庫 + Container Registry |
| **ArgoCD** | `https://argocd.cks-lab-01.tailXXXXXX.ts.net` | - | GitOps 部署管理 |

---

## 2. 新版 Ingress 配置（*.int.cks-lab.uk）

### Gitea Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitea-int
  namespace: gitea
  annotations:
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
spec:
  ingressClassName: nginx
  rules:
  - host: git.int.cks-lab.uk
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gitea-http
            port:
              number: 3000
```

### ArgoCD Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-int
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
spec:
  ingressClassName: nginx
  rules:
  - host: argo.int.cks-lab.uk
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 80
```

### Rancher Ingress（HTTPS + 自簽憑證）⚠️

> ⚠️ **重要**：Rancher 必須使用 HTTPS，否則會出現 "Invalid CSRF token" 錯誤

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rancher-int
  namespace: cattle-system
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "off"
    cert-manager.io/cluster-issuer: "selfsigned-issuer"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - rancher.int.cks-lab.uk
    secretName: rancher-int-tls
  rules:
  - host: rancher.int.cks-lab.uk
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: rancher
            port:
              number: 443
```

### Portainer Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: portainer-int
  namespace: portainer
spec:
  ingressClassName: nginx
  rules:
  - host: portainer.int.cks-lab.uk
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: portainer
            port:
              number: 9000
```

---

## 3. 舊版 Ingress 配置（*.cks-lab-01.tailXXXXXX.ts.net）

### Rancher Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rancher-ingress
  namespace: cattle-system
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "off"
    cert-manager.io/cluster-issuer: "selfsigned-issuer"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - rancher.cks-lab-01.tailXXXXXX.ts.net
    secretName: rancher-tls
  rules:
  - host: rancher.cks-lab-01.tailXXXXXX.ts.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: rancher
            port:
              number: 443
```

### Portainer Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: portainer-ingress
  namespace: portainer
  annotations:
    cert-manager.io/cluster-issuer: "selfsigned-issuer"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - portainer.cks-lab-01.tailXXXXXX.ts.net
    secretName: portainer-tls
  rules:
  - host: portainer.cks-lab-01.tailXXXXXX.ts.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: portainer
            port:
              number: 9000
```

### Gitea Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitea-ingress
  namespace: gitea
  annotations:
    cert-manager.io/cluster-issuer: "selfsigned-issuer"
    nginx.ingress.kubernetes.io/proxy-body-size: "0"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - gitea.cks-lab-01.tailXXXXXX.ts.net
    secretName: gitea-tls
  rules:
  - host: gitea.cks-lab-01.tailXXXXXX.ts.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gitea-http
            port:
              number: 3000
```

### ArgoCD Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server
  namespace: argocd
  annotations:
    cert-manager.io/cluster-issuer: "selfsigned-issuer"
    nginx.ingress.kubernetes.io/backend-protocol: "HTTP"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - argocd.cks-lab-01.tailXXXXXX.ts.net
    secretName: argocd-server-tls
  rules:
  - host: argocd.cks-lab-01.tailXXXXXX.ts.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: argocd-server
            port:
              number: 80
```

---

## 4. 對外展示 Ingress（Cloudflare Tunnel）

### Demo 應用 Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-public
  namespace: dev
spec:
  ingressClassName: nginx
  rules:
  - host: demo.cks-lab.uk
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: test-frontend
            port:
              number: 80
```

---

## 5. TLS 憑證

### 內網服務（新版 *.int.cks-lab.uk）

| 服務 | 協定 | TLS 設定 |
|------|------|----------|
| Gitea | HTTP | 無（Tailscale 已加密） |
| ArgoCD | HTTP | 無（Tailscale 已加密） |
| **Rancher** | **HTTPS** | 自簽憑證（cert-manager） |
| Portainer | HTTP | 無（Tailscale 已加密） |

> ⚠️ Rancher 因 CSRF 保護機制必須使用 HTTPS

### 內網服務（舊版 *.cks-lab-01.tailXXXXXX.ts.net）

使用 self-signed 憑證：

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
```

### 對外服務（*.cks-lab.uk）

- Cloudflare 自動提供 SSL 憑證
- 內部走 HTTP（Tunnel 已加密）

---

## 6. 檢查 Ingress 狀態

```bash
# 列出所有 Ingress
kubectl get ingress -A

# 檢查特定 Ingress 詳情
kubectl describe ingress gitea-int -n gitea
kubectl describe ingress argocd-int -n argocd
kubectl describe ingress rancher-int -n cattle-system
kubectl describe ingress demo-public -n dev

# 檢查憑證狀態（Rancher 內網 + 舊版 Ingress 用）
kubectl get certificates -A
kubectl get clusterissuer
```

---

*返回 [README](README.md) | 上一章 [網路架構](07-networking.md) | 下一章 [團隊權限](09-team-permissions.md)*
