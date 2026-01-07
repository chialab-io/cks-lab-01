# Namespace 與資源配額

> 更新日期：2025-12-23

---

## 1. 環境用途

| 環境 | Namespace | 用途 | 部署方式 |
|------|-----------|------|----------|
| **dev** | `dev` | 前後端開發測試、功能驗證 | ArgoCD 自動同步 |
| **prod** | `prod` | 正式服務運行 | ArgoCD 手動同步/審核 |

---

## 2. 資源配額

| Namespace | CPU 上限 | Memory 上限 | Pod 上限 | GPU |
|-----------|----------|-------------|----------|-----|
| **dev** | 4 cores | 8Gi | 20 | ❌ 禁用 |
| **prod** | 4 cores | 8Gi | 15 | ❌ 禁用（管理員可調整）|
| **系統保留** | ~4 cores | ~12Gi | - | ✅ 管理員專用 |

---

## 3. Namespace 配置檔

```yaml
# namespaces.yaml
---
apiVersion: v1
kind: Namespace
metadata:
  name: dev
  labels:
    environment: development
    managed-by: argocd

---
apiVersion: v1
kind: Namespace
metadata:
  name: prod
  labels:
    environment: production
    managed-by: argocd

---
# ResourceQuota - Dev
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "3"
    requests.memory: 6Gi
    limits.cpu: "4"
    limits.memory: 8Gi
    pods: "20"
    persistentvolumeclaims: "10"
    requests.nvidia.com/gpu: "0"
    limits.nvidia.com/gpu: "0"

---
# ResourceQuota - Prod
apiVersion: v1
kind: ResourceQuota
metadata:
  name: prod-quota
  namespace: prod
spec:
  hard:
    requests.cpu: "3"
    requests.memory: 6Gi
    limits.cpu: "4"
    limits.memory: 8Gi
    pods: "15"
    persistentvolumeclaims: "8"
    requests.nvidia.com/gpu: "0"
    limits.nvidia.com/gpu: "0"

---
# LimitRange - Dev
apiVersion: v1
kind: LimitRange
metadata:
  name: dev-limits
  namespace: dev
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi

---
# LimitRange - Prod
apiVersion: v1
kind: LimitRange
metadata:
  name: prod-limits
  namespace: prod
spec:
  limits:
    - type: Container
      default:
        cpu: 500m
        memory: 512Mi
      defaultRequest:
        cpu: 100m
        memory: 128Mi
```

---

## 4. 典型應用 Stack

| 層級 | 建議方案 | 資源消耗 |
|------|----------|----------|
| **Frontend** | React / Vue / Next.js | 低 |
| **Backend** | Node.js / Go / Python FastAPI | 中 |
| **Database** | PostgreSQL（已安裝） | 中 |
| **Cache** | Redis（建議加入） | 低 |
| **Queue** | Redis / RabbitMQ（視需求） | 低-中 |

---

## 5. 檢查資源配額

```bash
# 檢查 dev namespace 配額使用情況
kubectl describe resourcequota -n dev

# 檢查 prod namespace 配額使用情況
kubectl describe resourcequota -n prod

# 檢查 LimitRange
kubectl describe limitrange -n dev
kubectl describe limitrange -n prod
```

---

## 6. imagePullSecret 狀態

| Namespace | Secret 名稱 | 狀態 |
|-----------|-------------|------|
| `dev` | `gitea-registry` | ✅ 已建立 |
| `prod` | `gitea-registry` | ⏳ 待建立 |

### 建立 imagePullSecret

```bash
kubectl create secret docker-registry gitea-registry \
  --namespace=<namespace> \
  --docker-server=gitea.cks-lab-01.tailXXXXXX.ts.net \
  --docker-username=ci-bot \
  --docker-password=<ci-bot-token>
```

---

*返回 [README](README.md) | 上一章 [K3s 叢集](03-k3s-cluster.md) | 下一章 [CI/CD](05-cicd.md)*
