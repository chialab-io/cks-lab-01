# 快速參考

> 更新日期：2026-01-06

---

## 1. 服務 URL 總覽

### 內網服務（新版域名，推薦）✅

無需設定 hosts，只需 Tailscale 連線。

| 服務 | URL | 備註 |
|------|-----|------|
| Gitea | http://git.int.cks-lab.uk | |
| ArgoCD | http://argo.int.cks-lab.uk | |
| Rancher | **https://rancher.int.cks-lab.uk** | ⚠️ 必須 HTTPS，忽略憑證警告 |
| Portainer | http://portainer.int.cks-lab.uk | |

### 對外服務

| 服務 | URL | 說明 |
|------|-----|------|
| Demo App (Prod) | https://demo.cks-lab.uk | 指向 Prod 環境 |

### 內網應用服務

| 應用 | 內網 URL | 環境 |
|------|----------|------|
| test-frontend | http://test-frontend-dev.int.cks-lab.uk | Dev |
| test-frontend-prod | http://test-frontend-prod.int.cks-lab.uk | Prod |

### 內網服務（舊版域名，需 hosts）

| 服務 | URL |
|------|-----|
| Rancher | https://rancher.cks-lab-01.tailXXXXXX.ts.net |
| Portainer | https://portainer.cks-lab-01.tailXXXXXX.ts.net |
| Gitea | https://gitea.cks-lab-01.tailXXXXXX.ts.net |
| ArgoCD | https://argocd.cks-lab-01.tailXXXXXX.ts.net |

---

## 2. ArgoCD 快速參考

### 登入資訊

- **URL（新）**: http://argo.int.cks-lab.uk
- **URL（舊）**: https://argocd.cks-lab-01.tailXXXXXX.ts.net
- **帳號**: `admin`
- **密碼**: 已在 UI 修改（非初始密碼）

### CLI 登入

```bash
# 新版域名
argocd login argo.int.cks-lab.uk --username admin --insecure --grpc-web

# 舊版域名
argocd login argocd.cks-lab-01.tailXXXXXX.ts.net --username admin --insecure --grpc-web
```

### 常用操作

```bash
# 查看 ArgoCD Pod 狀態
kubectl get pods -n argocd

# Repository 管理
argocd repo list
argocd repo add <url> --username <user> --password <token>

# Application 管理
argocd app list
argocd app get <app-name>
argocd app sync <app-name>
argocd app delete <app-name>
```

---

## 3. Cloudflare Tunnel 快速參考

### 狀態檢查

```bash
# 檢查 Pod 狀態
kubectl get pods -n cloudflare-system

# 檢查 logs
kubectl logs -n cloudflare-system -l app=cloudflared --tail=20

# 重啟 Tunnel（如遇問題）
kubectl rollout restart deployment cloudflared -n cloudflare-system
```

### Tunnel 資訊

| 項目 | 值 |
|------|-----|
| **Tunnel 名稱** | `cks-lab` |
| **Tunnel ID** | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| **Namespace** | `cloudflare-system` |
| **Replicas** | 2（高可用） |

### 新增對外服務步驟

1. Cloudflare Zero Trust → Networks → Tunnels → `cks-lab`
2. Public Hostnames → Add a public hostname
3. 填入 Subdomain、Domain、Type（HTTP）、URL
4. 在 K8s 建立對應的 Ingress rule

---

## 4. Gitea Actions Runner 快速參考

### Runner 狀態

```bash
# 查看 Runner Pod
kubectl get pods -n gitea-runner

# 查看 Runner logs
kubectl logs -n gitea-runner actions-runner-act-runner-0 -c act-runner --tail=50

# 重啟 Runner
kubectl rollout restart statefulset actions-runner-act-runner -n gitea-runner
```

### Runner 重新註冊流程

> ⚠️ **重要**：必須使用 API 取得 Token，不要用 UI！

```bash
# 1. 用 API 取得全域 Token
kubectl port-forward svc/gitea-http -n gitea 3000:3000 &
GLOBAL_TOKEN=$(curl -s -u "admin:<admin密碼>" -X POST \
  http://127.0.0.1:3000/api/v1/admin/actions/runners/registration-token | jq -r .token)
kill %1

# 2. 更新 Secret
kubectl delete secret runner-token -n gitea-runner
kubectl create secret generic runner-token \
  --namespace=gitea-runner \
  --from-literal=token=$GLOBAL_TOKEN

# 3. 清除 PVC 並重啟
kubectl scale statefulset actions-runner-act-runner -n gitea-runner --replicas=0
kubectl delete pvc -n gitea-runner --all
kubectl scale statefulset actions-runner-act-runner -n gitea-runner --replicas=1
```

---

## 5. 主機連線資訊

| 項目 | 值 |
|------|-----|
| **Tailscale IP** | 100.96.128.95 |
| **MetalLB IP 池** | 192.168.0.200 - 192.168.0.220 |
| **Ingress IP** | 192.168.0.202 |
| **域名** | cks-lab.uk |

---

## 6. 網路架構速查

### 域名分流

| 域名模式 | 走向 | 用途 |
|----------|------|------|
| `*.cks-lab.uk` | Cloudflare Tunnel | 對外展示 |
| `*.int.cks-lab.uk` | Tailscale 直連 | 團隊內網 |

### DNS 記錄

| Type | Name | Content | Proxy |
|------|------|---------|-------|
| A | `*.int` | 100.96.128.95 | ❌ DNS only |
| A | `@` | 100.96.128.95 | ❌ DNS only |

---

## 7. 常用 kubectl 指令

```bash
# 叢集狀態
kubectl get nodes -o wide
kubectl get pods -A

# Namespace 資源
kubectl get pods -n dev
kubectl get pods -n prod

# Ingress 檢查
kubectl get ingress -A

# 日誌與除錯
kubectl logs <pod-name> -n <namespace> -f
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh
kubectl port-forward svc/<service> <local-port>:<service-port> -n <namespace>
```

---

## 8. GPU 使用備註

> ⚠️ **低優先級**：GPU 功能已就緒但目前非主要用途

### 現狀

- NVIDIA Device Plugin 已安裝並運作
- dev/prod namespace 已禁用 GPU（ResourceQuota 限制）
- 有需要時可由管理員調整

### 驗證 GPU 可用性

```bash
kubectl describe node | grep -A5 "nvidia.com/gpu"
```

---

*返回 [README](README.md) | 上一章 [已部署應用](14-deployed-apps.md)*
