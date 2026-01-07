# 問題排解記錄

> 更新日期：2025-01-06

---

## 目錄

1. [Tailscale 無法連線 Ingress 服務](#1-tailscale-無法連線-ingress-服務)
2. [ArgoCD Helm chart 9.x Ingress 設定問題](#2-argocd-helm-chart-9x-ingress-設定問題)
3. [ArgoCD 307 重定向循環](#3-argocd-307-重定向循環)
4. [ArgoCD 連接 Gitea Repository 失敗](#4-argocd-連接-gitea-repository-失敗)
5. [Gitea Registry Push 401 Unauthorized](#5-gitea-registry-push-401-unauthorized)
6. [Gitea Registry Push 413 Request Entity Too Large](#6-gitea-registry-push-413-request-entity-too-large)
7. [Gitea Actions Runner 註冊為「個人」類型](#7-gitea-actions-runner-註冊為個人類型)
8. [Gitea Actions Runner unregistered runner 錯誤](#8-gitea-actions-runner-unregistered-runner-錯誤)
9. [ArgoCD Repository 認證失效](#9-argocd-repository-認證失效)
10. [重開機後 K3s 資料遺失（LVM 未自動掛載）](#10-重開機後-k3s-資料遺失lvm-未自動掛載)
11. [Gitea Actions Runner TLS 憑證錯誤](#11-gitea-actions-runner-tls-憑證錯誤)
12. [CI Push Image 失敗 - context deadline exceeded](#12-ci-push-image-失敗---context-deadline-exceeded)
13. [主機無法解析 K8s 內部 DNS](#13-主機無法解析-k8s-內部-dns)
14. [Git Push 時 SSL 憑證錯誤](#14-git-push-時-ssl-憑證錯誤)
15. [主機缺少 unzip 指令](#15-主機缺少-unzip-指令)
16. [Git Push 時 main 分支不存在](#16-git-push-時-main-分支不存在)
17. [Image Pull 失敗 - K8s 內部 DNS 解析問題](#17-image-pull-失敗---k8s-內部-dns-解析問題)
18. [npm ci 需要 package-lock.json](#18-npm-ci-需要-package-lockjson)
19. [ArgoCD CLI Token 過期](#19-argocd-cli-token-過期)
20. [Git Clone 私有 Repo 認證失敗](#20-git-clone-私有-repo-認證失敗)
21. [Rancher 登入時 Invalid CSRF Token 錯誤](#21-rancher-登入時-invalid-csrf-token-錯誤)
22. [ArgoCD Sync 失敗 - Ingress Host 衝突](#22-argocd-sync-失敗---ingress-host-衝突)

---

## 1. Tailscale 無法連線 Ingress 服務

### 問題描述

透過 Tailscale IP（100.x.x.x）無法連線到 Ingress 服務，但 MetalLB IP（192.168.0.202）可以正常連線。

### 原因

- Tailscale MagicDNS 只解析主機名稱本身（`cks-lab-01.tailXXXXXX.ts.net`）
- 不支援子域名解析（`gitea.cks-lab-01.tailXXXXXX.ts.net`）
- Nginx Ingress 預設只監聽 MetalLB 分配的 IP

### 解決方案

**1. DNS 解析**：團隊成員需在本機 hosts 檔案加入對應：

```
# Windows: C:\Windows\System32\drivers\etc\hosts
# Mac/Linux: /etc/hosts
100.x.x.x rancher.cks-lab-01.tailXXXXXX.ts.net
100.x.x.x portainer.cks-lab-01.tailXXXXXX.ts.net
100.x.x.x gitea.cks-lab-01.tailXXXXXX.ts.net
100.x.x.x argocd.cks-lab-01.tailXXXXXX.ts.net
```

**2. Nginx Ingress 設定**：啟用 hostPort 讓 Ingress 監聽所有主機 IP：

```yaml
controller:
  hostPort:
    enabled: true
    ports:
      http: 80
      https: 443
  kind: DaemonSet
  service:
    externalIPs:
      - "100.x.x.x"
```

---

## 2. ArgoCD Helm chart Ingress 設定問題 (6.x+)

### 問題描述

使用 Helm 安裝 ArgoCD 時，即使在 values 中設定了正確的 `server.ingress.hosts`，Ingress 仍顯示預設的 `argocd.example.com`。

### 原因

ArgoCD Helm chart 在 **6.x 版本後**（目前為 9.x）Ingress 設定結構有重大變化，舊的 `hosts` 陣列設定不再生效，需使用 `hostname` 欄位。

### 解決方案

直接 patch Ingress 修正 host：

```bash
kubectl patch ingress argocd-server -n argocd --type='json' -p='[
  {"op": "replace", "path": "/spec/rules/0/host", "value": "argocd.cks-lab-01.tailXXXXXX.ts.net"},
  {"op": "replace", "path": "/spec/tls/0/hosts/0", "value": "argocd.cks-lab-01.tailXXXXXX.ts.net"}
]'
```

---

## 3. ArgoCD 307 重定向循環

### 問題描述

存取 ArgoCD Web UI 時瀏覽器顯示「重定向次數過多」，curl 測試回傳 HTTP 307。

### 原因

ArgoCD Helm chart 雖然設定了 `server.insecure: true`，但 ConfigMap `argocd-cmd-params-cm` 中缺少 `server.insecure` 設定，導致 ArgoCD server 仍嘗試 HTTPS 重定向。

### 解決方案

**1. 修正 ConfigMap**：

```bash
kubectl patch configmap argocd-cmd-params-cm -n argocd \
  --type='merge' -p='{"data":{"server.insecure":"true"}}'
```

**2. 重啟 ArgoCD Server**：

```bash
kubectl rollout restart deployment argocd-server -n argocd
```

**3. 修正 Ingress backend protocol**：

```bash
kubectl patch ingress argocd-server -n argocd --type='merge' -p='{
  "metadata": {
    "annotations": {
      "nginx.ingress.kubernetes.io/backend-protocol": "HTTP"
    }
  }
}'
```

---

## 4. ArgoCD 連接 Gitea Repository 失敗

### 問題描述

ArgoCD 無法連接 Gitea 的 `gitops-manifests` repository，顯示 `no such host` 或 `authentication required` 錯誤。

### 原因

ArgoCD repo-server 在 K3s 叢集內部運行，使用 CoreDNS 無法解析自定義的 Tailscale 域名。

### 解決方案

使用 **ArgoCD CLI** 加入 repository（使用內部 DNS）：

```bash
argocd repo add http://gitea-http.gitea.svc.cluster.local:3000/chialab/gitops-manifests.git \
  --username ci-bot \
  --password <ci-bot-token>
```

---

## 5. Gitea Registry Push 401 Unauthorized

### 問題描述

nerdctl push 到 Gitea Registry 時出現 `401 Unauthorized` 錯誤。

### 原因

ci-bot 帳號在 Gitea 組織中只有 Read 權限，無法 push image。

### 解決方案

在 Gitea UI 調整 ci-bot 權限：
1. 進入組織 `chialab` → **Settings** → **Teams**
2. 建立新 team `ci-writers`，權限設為 **Write**
3. 將 ci-bot 加入此 team

---

## 6. Gitea Registry Push 413 Request Entity Too Large

### 問題描述

push 較大的 image layer 時出現 `413 Request Entity Too Large` 錯誤。

### 原因

Nginx Ingress 預設有請求大小限制。

### 解決方案

```bash
kubectl patch ingress gitea-ingress -n gitea --type='merge' -p='{
  "metadata": {
    "annotations": {
      "nginx.ingress.kubernetes.io/proxy-body-size": "0"
    }
  }
}'
```

---

## 7. Gitea Actions Runner 註冊為「個人」類型

### 問題描述

Gitea Actions Runner 安裝後，在 Gitea 管理介面顯示為「個人」類型，導致 workflow 無法使用。

### 原因

> ⚠️ **Gitea UI Bug（確認於 1.24.6）**：從 UI 取得的 Token 會註冊為「個人」類型

### 解決方案

**必須使用 API 取得全域 Registration Token**：

```bash
kubectl port-forward svc/gitea-http -n gitea 3000:3000 &
GLOBAL_TOKEN=$(curl -s -u "admin:<admin密碼>" -X POST \
  http://127.0.0.1:3000/api/v1/admin/actions/runners/registration-token | jq -r .token)
kill %1
```

---

## 8. Gitea Actions Runner unregistered runner 錯誤

### 問題描述

Runner 重啟後顯示 `rpc error: code = Unauthenticated desc = unregistered runner` 錯誤。

### 原因

Runner Pod 中緩存了舊的註冊資訊，與新的 Token 不匹配。

### 解決方案

```bash
kubectl scale statefulset actions-runner-act-runner -n gitea-runner --replicas=0
kubectl delete pvc -n gitea-runner --all
# 更新 Token（見第 7 節）
kubectl scale statefulset actions-runner-act-runner -n gitea-runner --replicas=1
```

---

## 9. ArgoCD Repository 認證失效

### 問題描述

ArgoCD Application 顯示 `ComparisonError: failed to authenticate user` 錯誤。

### 原因

ci-bot 的 Access Token 已變更或失效。

### 解決方案

```bash
argocd repo rm http://gitea-http.gitea.svc.cluster.local:3000/chialab/gitops-manifests.git
argocd repo add http://gitea-http.gitea.svc.cluster.local:3000/chialab/gitops-manifests.git \
  --username ci-bot \
  --password <最新的ci-bot-token>
```

---

## 10. 重開機後 K3s 資料遺失（LVM 未自動掛載）

### 問題描述

主機重開機後，`kubectl get ns` 只看到預設的 4 個 namespace。

### 原因

LVM 邏輯卷沒有設定在 `/etc/fstab` 中自動掛載。

### 解決方案

```bash
# 加入 fstab
echo '/dev/ubuntu-vg/k3s-lv /var/lib/rancher ext4 defaults 0 2' | sudo tee -a /etc/fstab

# 掛載並重啟 K3s
sudo mount /dev/ubuntu-vg/k3s-lv /var/lib/rancher
sudo systemctl restart k3s
```

---

## 11. Gitea Actions Runner TLS 憑證錯誤

### 問題描述

Runner 啟動時顯示 `could not read CA certificate "/certs/client/ca.pem"` 錯誤。

### 原因

設定了 `DOCKER_TLS_CERTDIR=""`，導致 dind 不產生 TLS 憑證。

### 解決方案

**不要**設定 `DOCKER_TLS_CERTDIR=""`，讓 dind 自動產生憑證。

---

## 12. CI Push Image 失敗 - context deadline exceeded

> ✅ **已解決**（2025-12-30）

### 問題描述

CI workflow 在 Push image 步驟失敗：

```
Get "https://gitea.cks-lab-01.tailXXXXXX.ts.net/v2/token?account=ci-bot&...": 
context deadline exceeded
```

### 根本原因

Gitea 的 `ROOT_URL` 設為 HTTPS，導致 token endpoint 也是 HTTPS，DinD 容器無法連接。

### 解決方案

1. **修改 Gitea ROOT_URL 為 HTTP**
2. **配置 gitea-registry Service（多 port）**
3. **配置 DinD insecure-registries**
4. **配置 hosts 解析**

詳細步驟見 [runner-config-backup.md](runner-config-backup.md)

---

## 13. 主機無法解析 K8s 內部 DNS

### 問題描述

在主機上執行 `git push` 到 `gitea-http.gitea.svc.cluster.local` 時出現：

```
Could not resolve host: gitea-http.gitea.svc.cluster.local
```

### 原因

主機不在 K8s 網路內，無法解析 K8s Service 的內部 DNS。

### 解決方案

使用外部 URL（透過 Tailscale）：

```bash
# 改用外部 URL
git remote remove origin
git remote add origin https://gitea.cks-lab-01.tailXXXXXX.ts.net/chialab/<repo>.git
```

---

## 14. Git Push 時 SSL 憑證錯誤

### 問題描述

```
SSL certificate problem: self-signed certificate
```

### 原因

使用自簽憑證，Git 預設不信任。

### 解決方案

```bash
# 暫時跳過 SSL 驗證
git -c http.sslVerify=false push origin main

# 或永久設定此 domain
git config --global http.https://gitea.cks-lab-01.tailXXXXXX.ts.net/.sslVerify false
```

---

## 15. 主機缺少 unzip 指令

### 問題描述

```
Command 'unzip' not found
```

### 解決方案

```bash
# 安裝 unzip
sudo apt install unzip -y

# 或用 python3 解壓
python3 -m zipfile -e file.zip .
```

---

## 16. Git Push 時 main 分支不存在

### 問題描述

```
error: src refspec main does not match any
```

### 原因

Git 預設分支可能是 `master`，不是 `main`。

### 解決方案

```bash
# 檢查目前分支
git branch

# 如果是 master，重新命名為 main
git branch -M main

# 再推送
git push -u origin main
```

---

## 17. Image Pull 失敗 - K8s 內部 DNS 解析問題

> ✅ **已解決**（2025-12-30）

### 問題描述

ArgoCD 部署後，Pod 出現 ImagePullBackOff 錯誤：

```
Failed to pull image "gitea-http.gitea.svc.cluster.local:3000/chialab/test-frontend:dev-latest": 
failed to resolve reference: lookup gitea-http.gitea.svc.cluster.local: Try again
```

### 原因

Deployment 中的 image URL 使用了 K8s 內部 DNS（`gitea-http.gitea.svc.cluster.local`），但 kubelet 拉 image 時使用的是 K3s 的 `registries.yaml` 配置，該配置對應的是外部域名。

### 解決方案

**Deployment 中的 image URL 必須使用外部域名**：

```yaml
# ❌ 錯誤
image: gitea-http.gitea.svc.cluster.local:3000/chialab/test-frontend:dev-latest

# ✅ 正確
image: gitea.cks-lab-01.tailXXXXXX.ts.net/chialab/test-frontend:dev-latest
```

### 說明

| 用途 | URL |
|------|-----|
| CI Push（內部） | `gitea-http.gitea.svc.cluster.local:3000` |
| K8s Pull（外部） | `gitea.cks-lab-01.tailXXXXXX.ts.net` |

---

## 18. npm ci 需要 package-lock.json

### 問題描述

CI Build 失敗：

```
npm error The `npm ci` command can only install with an existing package-lock.json
```

### 原因

專案中沒有 `package-lock.json` 檔案，但 Dockerfile 使用了 `npm ci`。

### 解決方案

> ⚠️ **注意：本專案規範使用 pnpm**

根據專案規範 (`GEMINI.md`)，前端專案應強制使用 `pnpm`。如果遇到此錯誤，表示您的專案可能混用了 npm 與 pnpm，或者 CI 流程未正確設定。

**建議方案：轉換為 pnpm 環境（推薦）**

1. 確保本機使用 pnpm 安裝依賴：
   ```bash
   # 移除 npm lock (如果有的話)
   rm package-lock.json
   
   # 安裝 pnpm (如果沒有)
   corepack enable
   
   # 安裝依賴並產生 pnpm-lock.yaml
   pnpm install
   
   # 提交變更
   git add pnpm-lock.yaml
   git commit -m "chore: migrate to pnpm"
   git push
   ```

2. 修改 CI / Dockerfile 使用 pnpm：
   ```dockerfile
   # Dockerfile 範例
   RUN npm install -g pnpm
   COPY pnpm-lock.yaml package.json ./
   RUN pnpm install --frozen-lockfile
   ```

**替代方案：僅修復 npm 錯誤 (不推薦)**

如果不打算遷移到 pnpm，需產生 `package-lock.json`：

```bash
# 在本機執行
npm install
git add package-lock.json
git commit -m "fix: add package-lock.json"
git push
```

**方案 B：改用 npm install (僅限測試)**

修改 Dockerfile：

```dockerfile
# 改成
RUN npm install
```

---

## 19. ArgoCD CLI Token 過期

### 問題描述

執行 ArgoCD CLI 指令時出現：

```
rpc error: code = Unauthenticated desc = invalid session: token has invalid claims: token is expired
```

### 原因

ArgoCD CLI 的登入 session 已過期。

### 解決方案

重新登入：

```bash
argocd login argocd.cks-lab-01.tailXXXXXX.ts.net --username admin --insecure --grpc-web
```

---

## 20. Git Clone 私有 Repo 認證失敗

### 問題描述

```
remote: Failed to authenticate user
fatal: Authentication failed
```

### 原因

私有 Repo 需要認證。

### 解決方案

```bash
# 方法 1: URL 中帶入帳號（會提示輸入密碼或 Token）
git -c http.sslVerify=false clone https://wcwen@gitea.cks-lab-01.tailXXXXXX.ts.net/chialab/<repo>.git

# 方法 2: 使用 Access Token 當密碼
# 密碼處輸入 Gitea Access Token
```

---

## 21. Rancher 登入時 Invalid CSRF Token 錯誤

> ✅ **已解決**（2025-01-06）

### 問題描述

使用 HTTP 存取 `http://rancher.int.cks-lab.uk` 時，登入畫面顯示：

```
An error occurred logging in: Invalid CSRF token
```

但使用 HTTPS 存取 `https://rancher.cks-lab-01.tailXXXXXX.ts.net` 則可正常登入。

### 原因

Rancher 的 CSRF 保護機制會檢查：
1. **Cookie 的 Secure 屬性**：Rancher 預設在 HTTPS 下設定 Secure cookie
2. **Origin/Referer header**：必須與伺服器端配置匹配

當用 HTTP 存取時，瀏覽器不會發送 Secure cookie，導致 CSRF 驗證失敗。

### 解決方案

**為 Rancher 內網域名加上 HTTPS + 自簽憑證**：

```bash
kubectl apply -f - << 'EOF'
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
EOF
```

### 驗證

```bash
# 檢查憑證是否簽發成功
kubectl get certificate -n cattle-system

# 測試 HTTPS 連線
curl -k -s -o /dev/null -w "HTTP Status: %{http_code}\n" https://rancher.int.cks-lab.uk
```

### 存取方式

使用 `https://rancher.int.cks-lab.uk`（瀏覽器會警告自簽憑證，忽略即可）。

### 重要提醒

> ⚠️ **Rancher 必須使用 HTTPS**，這是 Rancher 的安全設計，無法繞過。其他服務（Gitea、ArgoCD、Portainer）如果沒有類似的 CSRF 限制，可以繼續使用 HTTP。

---

## 22. ArgoCD Sync 失敗 - Ingress Host 衝突

> ✅ **已解決**（2026-01-06）

### 問題描述

ArgoCD Sync 時出現錯誤：

```
admission webhook "validate.nginx.ingress.kubernetes.io" denied the request: 
host "demo.cks-lab.uk" and path "/" is already defined in ingress dev/demo-public
```

### 原因

同一個 host（`demo.cks-lab.uk`）已經在另一個 namespace 的 Ingress 中被定義。Nginx Ingress Controller 不允許同一個 host 被定義在多個 Ingress 中。

### 解決方案

刪除舊的 Ingress：

```bash
kubectl delete ingress demo-public -n dev
```

然後在 ArgoCD 重新 Sync。

### 預防措施

1. **域名規劃**：明確區分 Dev 和 Prod 的域名
   - Dev: `<app>-dev.int.cks-lab.uk`（內網）
   - Prod: `<app>-prod.int.cks-lab.uk`（內網）+ `<app>.cks-lab.uk`（對外）
2. **移除舊資源**：變更域名策略時，記得刪除舊的 Ingress 資源

---

*返回 [README](README.md) | 上一章 [常用指令](11-commands-reference.md) | 下一章 [設定檔備份](13-config-backup.md)*
