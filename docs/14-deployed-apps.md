# 已部署應用

> 更新日期：2026-01-06

---

## 1. 已部署應用列表

| 應用 | Namespace | 狀態 | ArgoCD App | 內網 URL | 對外 URL | 說明 |
|------|-----------|------|------------|----------|----------|------|
| **test-frontend** | dev | ✅ Running | ✅ Synced | http://test-frontend-dev.int.cks-lab.uk | - | Dev 環境測試 |
| **test-frontend-prod** | prod | ✅ Running | ✅ Synced | http://test-frontend-prod.int.cks-lab.uk | https://demo.cks-lab.uk | Prod 對外展示 |

---

## 2. test-frontend（端對端測試）

### 專案資訊

| 項目 | 值 |
|------|-----|
| **Repository** | `chialab/test-frontend` |
| **技術棧** | React 18 + Vite + Nginx |
| **Dev Image** | `gitea.cks-lab-01.tailXXXXXX.ts.net/chialab/test-frontend:dev-latest` |
| **Prod Image** | `gitea.cks-lab-01.tailXXXXXX.ts.net/chialab/test-frontend:prod-latest` |
| **Dev 內網 URL** | http://test-frontend-dev.int.cks-lab.uk |
| **Prod 內網 URL** | http://test-frontend-prod.int.cks-lab.uk |
| **Prod 對外 URL** | https://demo.cks-lab.uk |

### hosts 設定

團隊成員需在本機 hosts 檔案加入：

```
100.96.128.95 test-frontend.cks-lab-01.tailXXXXXX.ts.net
```

### ArgoCD Application

```bash
argocd app create test-frontend \
  --repo http://gitea-http.gitea.svc.cluster.local:3000/chialab/gitops-manifests.git \
  --path apps/dev/test-frontend \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace dev \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

### Manifest 內容

```yaml
# apps/dev/test-frontend/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test-frontend
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test-frontend
  template:
    metadata:
      labels:
        app: test-frontend
    spec:
      imagePullSecrets:
        - name: gitea-registry
      containers:
      - name: app
        image: gitea.cks-lab-01.tailXXXXXX.ts.net/chialab/test-frontend:dev-latest
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 10
---
apiVersion: v1
kind: Service
metadata:
  name: test-frontend
  namespace: dev
spec:
  selector:
    app: test-frontend
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test-frontend
  namespace: dev
spec:
  ingressClassName: nginx
  rules:
  - host: test-frontend.cks-lab-01.tailXXXXXX.ts.net
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

## 3. ArgoCD Application 建立指令

### nginx-test（已建立）

```bash
argocd app create nginx-test \
  --repo http://gitea-http.gitea.svc.cluster.local:3000/chialab/gitops-manifests.git \
  --path apps/dev/nginx-test \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace dev \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

---

## 4. 新增應用範本

### 建立新的 dev 應用

```bash
argocd app create <app-name> \
  --repo http://gitea-http.gitea.svc.cluster.local:3000/chialab/gitops-manifests.git \
  --path apps/dev/<app-name> \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace dev \
  --sync-policy automated \
  --auto-prune \
  --self-heal
```

### 建立新的 prod 應用

```bash
argocd app create <app-name>-prod \
  --repo http://gitea-http.gitea.svc.cluster.local:3000/chialab/gitops-manifests.git \
  --path apps/prod/<app-name> \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace prod \
  --sync-policy none  # prod 使用手動同步
```

---

## 5. 應用管理指令

```bash
# 列出所有應用
argocd app list

# 查看應用詳情
argocd app get <app-name>

# 同步應用
argocd app sync <app-name>

# 刪除應用
argocd app delete <app-name>

# 查看應用日誌
argocd app logs <app-name>
```

---

## 6. GitOps Repository 結構

```
gitops-manifests/
├── apps/
│   ├── dev/
│   │   ├── nginx-test/
│   │   │   ├── deployment.yaml
│   │   │   └── service.yaml
│   │   ├── test-frontend/        ✅ 已建立
│   │   │   └── deployment.yaml   # 含 Deployment + Service + Ingress
│   │   ├── app-frontend/         # 待建立
│   │   │   └── ...
│   │   └── app-backend/          # 待建立
│   │       └── ...
│   └── prod/
│       ├── app-frontend/         # 待建立
│       │   └── ...
│       └── app-backend/          # 待建立
│           └── ...
└── README.md
```

---

## 7. 應用 Manifest 範例

### Deployment + Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      imagePullSecrets:
        - name: gitea-registry
      containers:
      - name: app
        image: gitea.cks-lab-01.tailXXXXXX.ts.net/chialab/my-app:dev-latest
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 50m
            memory: 64Mi
          limits:
            cpu: 200m
            memory: 128Mi
---
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: dev
spec:
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 80
```

### Ingress（HTTP）

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  namespace: dev
spec:
  ingressClassName: nginx
  rules:
  - host: my-app.cks-lab-01.tailXXXXXX.ts.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app
            port:
              number: 80
```

### Ingress（HTTPS + 自簽憑證）

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  namespace: dev
  annotations:
    cert-manager.io/cluster-issuer: "selfsigned-issuer"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - my-app.cks-lab-01.tailXXXXXX.ts.net
    secretName: my-app-tls
  rules:
  - host: my-app.cks-lab-01.tailXXXXXX.ts.net
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app
            port:
              number: 80
```

---

## 8. Image URL 注意事項

> ⚠️ **重要**：Deployment 中的 image URL 必須使用**外部域名**

| 用途 | URL |
|------|-----|
| **CI Push（內部）** | `gitea-http.gitea.svc.cluster.local:3000/chialab/<app>:<tag>` |
| **K8s Pull（外部）** | `gitea.cks-lab-01.tailXXXXXX.ts.net/chialab/<app>:<tag>` |

原因：kubelet 拉 image 時會使用 K3s 的 `registries.yaml` 配置，該配置對應的是外部域名。

---

*返回 [README](README.md) | 上一章 [設定檔備份](13-config-backup.md) | 下一章 [快速參考](15-quick-reference.md)*
