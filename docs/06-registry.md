# Container Registry

> 更新日期：2025-12-23

---

## 1. Gitea Container Registry

| 項目 | 設定值 |
|------|--------|
| **Registry URL** | `gitea.cks-lab-01.tailXXXXXX.ts.net` |
| **內部 URL** | `gitea-http.gitea.svc.cluster.local:3000` |
| **認證方式** | Gitea 帳號 + Token |
| **TLS** | Self-signed（需 insecure 設定） |

---

## 2. K3s Registry 配置

```yaml
# /etc/rancher/k3s/registries.yaml
mirrors:
  "gitea.cks-lab-01.tailXXXXXX.ts.net":
    endpoint:
      - "https://gitea.cks-lab-01.tailXXXXXX.ts.net"

configs:
  "gitea.cks-lab-01.tailXXXXXX.ts.net":
    tls:
      insecure_skip_verify: true
```

修改後需重啟 K3s：

```bash
sudo systemctl restart k3s
```

---

## 3. nerdctl 配置

```toml
# /etc/nerdctl/nerdctl.toml
namespace = "k8s.io"
address = "unix:///run/k3s/containerd/containerd.sock"
insecure_registry = true
```

---

## 4. Image Push/Pull 操作

### 登入 Registry

```bash
sudo nerdctl login gitea.cks-lab-01.tailXXXXXX.ts.net --username ci-bot --insecure-registry
```

### Tag Image

```bash
sudo nerdctl tag <source-image> gitea.cks-lab-01.tailXXXXXX.ts.net/chialab/<image-name>:<tag>
```

### Push Image

```bash
sudo nerdctl push --insecure-registry gitea.cks-lab-01.tailXXXXXX.ts.net/chialab/<image-name>:<tag>
```

### Pull Image（在 K8s 中）

Pod 會自動使用 `registries.yaml` 配置拉取 image。

---

## 5. K8s imagePullSecret

### 建立 Secret

```bash
kubectl create secret docker-registry gitea-registry \
  --namespace=<namespace> \
  --docker-server=gitea.cks-lab-01.tailXXXXXX.ts.net \
  --docker-username=ci-bot \
  --docker-password=<ci-bot-token>
```

### 已建立的 imagePullSecret

| Namespace | Secret 名稱 | 狀態 |
|-----------|-------------|------|
| `dev` | `gitea-registry` | ✅ 已建立 |
| `prod` | `gitea-registry` | ⏳ 待建立 |

### 在 Deployment 中使用

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      imagePullSecrets:
        - name: gitea-registry
      containers:
        - name: app
          image: gitea.cks-lab-01.tailXXXXXX.ts.net/chialab/my-app:latest
```

---

## 6. 常用指令

```bash
# 列出本地 images
sudo nerdctl images

# 檢查 Registry 認證
cat /root/.docker/config.json

# 清理未使用的 images
sudo k3s crictl rmi --prune

# 列出 K3s containerd images
sudo k3s crictl images
```

---

*返回 [README](README.md) | 上一章 [CI/CD](05-cicd.md) | 下一章 [網路架構](07-networking.md)*
