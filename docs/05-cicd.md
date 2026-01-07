# CI/CD 流程

> 更新日期：2026-01-06

---

## 1. 整體架構概覽

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              CI/CD 完整流程                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  【App Repository】              【GitOps Repository】                       │
│  ├── template-app (模板)         └── gitops-manifests                       │
│  ├── test-frontend                   ├── apps/dev/<app>/                    │
│  └── (未來專案...)                    └── apps/prod/<app>/                   │
│         │                                    │                              │
│         │ git push                           │ git push                     │
│         ▼                                    ▼                              │
│  ┌─────────────┐                     ┌─────────────┐                       │
│  │   Gitea     │                     │   ArgoCD    │                       │
│  │   Actions   │ ── push image ──>   │   監聽變更   │                       │
│  │   (CI)      │                     │   (CD)      │                       │
│  └──────┬──────┘                     └──────┬──────┘                       │
│         │                                    │                              │
│         ▼                                    ▼                              │
│  ┌─────────────┐                     ┌─────────────┐                       │
│  │  Container  │                     │  K8s        │                       │
│  │  Registry   │ ◄── pull image ──── │  Cluster    │                       │
│  └─────────────┘                     └─────────────┘                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Repository 分類與用途

### 2.1 Repository 類型

| 類型 | Repository | 用途 | 分支策略 |
|------|------------|------|----------|
| **模板** | `template-app` | CI workflow 模板、Dockerfile 模板 | develop + main |
| **應用程式** | `test-frontend` 等 | 實際應用代碼 | develop + main |
| **GitOps** | `gitops-manifests` | K8s 部署清單 | 僅 main |

### 2.2 各 Repository 說明

#### template-app（專案模板）

```
template-app/
├── .gitea/
│   └── workflows/
│       ├── ci-dev.yaml      # develop 分支 CI
│       └── ci-prod.yaml     # main 分支 CI
├── Dockerfile               # 多選項模板
├── nginx.conf               # SPA 路由支援
├── .dockerignore
├── .gitignore
└── README.md
```

**用途：** 建立新專案時的起點，包含標準化的 CI/CD 配置
**描述：** `專案模板，含 CI/CD workflow`

#### 應用程式 Repository（如 test-frontend）

```
test-frontend/
├── .gitea/workflows/    # 從模板繼承
├── src/                 # 應用程式代碼
├── Dockerfile
└── ...
```

**用途：** 實際應用程式代碼，觸發 CI 建構 image
**描述：** `端對端測試用 React App`（依專案調整）

#### gitops-manifests（GitOps 部署清單）

```
gitops-manifests/
├── apps/
│   ├── dev/
│   │   └── <app-name>/
│   │       └── deployment.yaml
│   └── prod/
│       └── <app-name>/
│           └── deployment.yaml
└── README.md
```

**用途：** 集中管理所有 K8s 部署清單，由 ArgoCD 監聽並自動部署
**描述：** `K8s 部署清單，ArgoCD 自動同步`

---

## 3. CI/CD 工具分工

| 工具 | 職責 | 狀態 |
|------|------|------|
| **Gitea** | 代碼託管、CI（build/test/push image） | ✅ 已配置 |
| **Gitea Actions Runner** | 執行 CI workflow | ✅ 運作中 |
| **Gitea Container Registry** | Image 儲存 | ✅ 已驗證 |
| **ArgoCD** | CD（GitOps 部署、環境同步） | ✅ 已配置 |

---

## 4. 分支策略

### 4.1 App Repository 分支規劃

| 分支 | 用途 | 誰可以 push | 觸發 CI | 分支保護 |
|------|------|-------------|---------|----------|
| `develop` | 開發測試 | 所有人 | ✅ 自動 | 無 |
| `main` | 正式環境 | 只有管理員（透過 PR） | ✅ PR merge 後 | ✅ 需設定 |

### 4.2 GitOps Repository 分支規劃

| 分支 | 用途 | 誰可以 push | 分支保護 |
|------|------|-------------|----------|
| `main` | 所有環境部署清單 | 所有人（或限管理員） | 可選 |

> **注意：** gitops-manifests 不需要 develop 分支，因為不需要 CI 建構

### 4.3 開發流程

```
┌──────────────────────────────────────────────────────────────────┐
│                     完整開發部署流程                              │
├──────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. 開發者 push to develop (App Repo)                            │
│         ↓                                                        │
│  2. Gitea Actions 自動觸發 CI                                    │
│         ↓                                                        │
│  3. Build + Push dev image (dev-<hash>)                          │
│         ↓                                                        │
│  4. 更新 gitops-manifests (apps/dev/<app>/)                      │
│         ↓                                                        │
│  5. ArgoCD 自動同步到 dev namespace                              │
│         ↓                                                        │
│  6. Dev 環境測試                                                 │
│         ↓                                                        │
│  7. 建立 PR: develop → main                                      │
│         ↓                                                        │
│  8. 管理員 Review + Approve + Merge                              │
│         ↓                                                        │
│  9. Gitea Actions 自動觸發 prod CI                               │
│         ↓                                                        │
│ 10. Build + Push prod image (prod-<date>-<hash>)                 │
│         ↓                                                        │
│ 11. 更新 gitops-manifests (apps/prod/<app>/)                     │
│         ↓                                                        │
│ 12. ArgoCD 同步到 prod namespace（手動或自動）                    │
│                                                                  │
└──────────────────────────────────────────────────────────────────┘
```

### 4.4 權限控制

| 動作 | 組員 | 管理員 |
|------|------|--------|
| Push to `develop` | ✅ | ✅ |
| 建立 PR (develop → main) | ✅ | ✅ |
| Approve PR | ❌ | ✅ |
| Merge PR to `main` | ❌ | ✅ |
| 直接 Push to `main` | ❌ | ❌ |
| 修改 gitops-manifests | ✅ | ✅ |

---

## 5. Image Tag 策略

### 5.1 時間戳記版本號

採用自動化時間戳記版本，不需手動打 tag：

| 環境 | Tag 格式 | 範例 |
|------|----------|------|
| **Dev** | `dev-<hash>` | `dev-a1b2c3d` |
| **Dev Latest** | `dev-latest` | `dev-latest` |
| **Prod** | `prod-<date>-<hash>` | `prod-20251230-a1b2c3d` |
| **Prod Latest** | `prod-latest`, `latest` | |

### 5.2 版本號說明

- `dev-a1b2c3d`：commit hash 前 7 碼
- `prod-20251230-a1b2c3d`：日期 (YYYYMMDD) + commit hash
- 日期在前方便排序，hash 方便回追代碼

---

## 6. CI Workflow 檔案

### 6.1 ci-dev.yaml（develop 分支）

```yaml
name: Dev Build and Push

on:
  push:
    branches: [develop]

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      DOCKER_HOST: tcp://127.0.0.1:2376
      DOCKER_TLS_VERIFY: 1
      DOCKER_CERT_PATH: /certs/client
      REGISTRY: ${{ vars.REGISTRY_URL }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set build info
        run: |
          echo "IMAGE_NAME=${{ github.repository }}" >> $GITHUB_ENV
          echo "SHORT_SHA=$(echo ${GITHUB_SHA} | cut -c1-7)" >> $GITHUB_ENV
          echo "BUILD_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ)" >> $GITHUB_ENV

      - name: Login to Registry
        run: |
          echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login ${{ env.REGISTRY }} -u ci-bot --password-stdin

      - name: Build and Push
        run: |
          docker build \
            --build-arg VERSION=dev-${{ env.SHORT_SHA }} \
            --build-arg BUILD_TIME=${{ env.BUILD_TIME }} \
            -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:dev-${{ env.SHORT_SHA }} \
            -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:dev-latest \
            .
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:dev-${{ env.SHORT_SHA }}
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:dev-latest

      - name: Summary
        run: |
          echo "## ✅ Dev Build Success" >> $GITHUB_STEP_SUMMARY
          echo "| Item | Value |" >> $GITHUB_STEP_SUMMARY
          echo "|------|-------|" >> $GITHUB_STEP_SUMMARY
          echo "| **Image** | \`${{ env.IMAGE_NAME }}:dev-${{ env.SHORT_SHA }}\` |" >> $GITHUB_STEP_SUMMARY
          echo "| **Commit** | \`${{ env.SHORT_SHA }}\` |" >> $GITHUB_STEP_SUMMARY
```

### 6.2 ci-prod.yaml（main 分支）

```yaml
name: Prod Build and Push

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    env:
      DOCKER_HOST: tcp://127.0.0.1:2376
      DOCKER_TLS_VERIFY: 1
      DOCKER_CERT_PATH: /certs/client
      REGISTRY: ${{ vars.REGISTRY_URL }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Set build info
        run: |
          echo "IMAGE_NAME=${{ github.repository }}" >> $GITHUB_ENV
          echo "SHORT_SHA=$(echo ${GITHUB_SHA} | cut -c1-7)" >> $GITHUB_ENV
          echo "DATE=$(date -u +%Y%m%d)" >> $GITHUB_ENV
          echo "BUILD_TIME=$(date -u +%Y-%m-%dT%H:%M:%SZ)" >> $GITHUB_ENV

      - name: Login to Registry
        run: |
          echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login ${{ env.REGISTRY }} -u ci-bot --password-stdin

      - name: Build and Push
        run: |
          VERSION="prod-${{ env.DATE }}-${{ env.SHORT_SHA }}"
          
          docker build \
            --build-arg VERSION=$VERSION \
            --build-arg BUILD_TIME=${{ env.BUILD_TIME }} \
            -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:$VERSION \
            -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:prod-latest \
            -t ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest \
            .
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:$VERSION
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:prod-latest
          docker push ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest

      - name: Summary
        run: |
          VERSION="prod-${{ env.DATE }}-${{ env.SHORT_SHA }}"
          echo "## 🚀 Production Build Success" >> $GITHUB_STEP_SUMMARY
          echo "| Item | Value |" >> $GITHUB_STEP_SUMMARY
          echo "|------|-------|" >> $GITHUB_STEP_SUMMARY
          echo "| **Version** | \`$VERSION\` |" >> $GITHUB_STEP_SUMMARY
          echo "| **Image** | \`${{ env.IMAGE_NAME }}:$VERSION\` |" >> $GITHUB_STEP_SUMMARY
```

---

## 7. Gitea Actions Runner 配置

| 項目 | 設定值 |
|------|--------|
| **Namespace** | `gitea-runner` |
| **Runner 名稱** | `actions-runner-act-runner-0` |
| **版本** | v0.2.13 |
| **Labels** | `ubuntu-latest`, `ubuntu-22.04` |
| **註冊層級** | 全域 (Instance) |
| **狀態** | ✅ 運作中 |

### Runner Pod 架構

```
┌─────────────────────────────────────────────────────────┐
│              actions-runner-act-runner-0                │
│  ┌─────────────────┐      ┌─────────────────┐          │
│  │   act-runner    │◄────►│      dind       │          │
│  │   (Container)   │ TLS  │   (Container)   │          │
│  │                 │:2376 │                 │          │
│  └────────┬────────┘      └─────────────────┘          │
│           │                      │                      │
│           │                      │ insecure-registries  │
│           │                      │ /etc/hosts 解析      │
│           │                      │                      │
│           │ 執行 Job             │                      │
│           ▼                      │                      │
│  ┌─────────────────┐            │                      │
│  │   Job Container │────────────┘                      │
│  │  (ubuntu-latest)│  docker push                      │
│  └─────────────────┘                                   │
└─────────────────────────────────────────────────────────┘
```

---

## 8. Organization 設定

### 8.1 Variables（組織層級）

```
Gitea → chialab → Settings → Actions → Variables
```

| Name | Value | 說明 |
|------|-------|------|
| `REGISTRY_URL` | `gitea-http.gitea.svc.cluster.local:3000` | 內部 Registry URL |

### 8.2 Secrets（組織層級）

```
Gitea → chialab → Settings → Actions → Secrets
```

| Name | Value | 說明 |
|------|-------|------|
| `REGISTRY_PASSWORD` | ci-bot Access Token | Registry 認證 |

> 設定在 Organization 層級，所有 repo 自動繼承，不需每個專案設定。

---

## 9. 建立新專案流程

### 9.1 從模板建立 App Repository

1. 到 `chialab/template-app`
2. 點擊 **"Use this template"**
3. 輸入新專案名稱
4. 修改 `Dockerfile`（依專案類型）
5. 設定 `main` 分支保護（見 9.3）
6. 建立 `develop` 分支
7. 開始開發

### 9.2 在 GitOps Repository 新增部署清單

1. 在 `gitops-manifests/apps/dev/<app-name>/` 建立資料夾
2. 加入 `deployment.yaml`（含 Deployment + Service + Ingress）
3. 提交並推送
4. 在 ArgoCD 建立 Application

### 9.3 分支保護設定

每個 App Repository 需設定分支保護：

```
Settings → Branches → Branch Protection
```

| 設定 | 值 |
|------|-----|
| Branch name | `main` |
| Enable Branch Protection | ✅ |
| Disable Push | ✅ |
| Enable Push Whitelist | ✅ |
| Whitelist users | `wcwen` |
| Required Approvals | 1 |

---

## 10. ArgoCD 應用管理

### 10.1 手動建立 Application

```bash
# Dev 環境（自動同步）
argocd app create <app-name> \
  --repo http://gitea-http.gitea.svc.cluster.local:3000/chialab/gitops-manifests.git \
  --path apps/dev/<app-name> \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace dev \
  --sync-policy automated \
  --auto-prune \
  --self-heal

# Prod 環境（手動同步）
argocd app create <app-name>-prod \
  --repo http://gitea-http.gitea.svc.cluster.local:3000/chialab/gitops-manifests.git \
  --path apps/prod/<app-name> \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace prod \
  --sync-policy none
```

### 10.2 App of Apps 模式 ✅

透過 ApplicationSet 自動管理所有 Application，新增應用只需：
1. 在 `apps/dev/<app>/` 加入 manifest
2. git push
3. ArgoCD 自動建立 Application

#### argocd-apps 資料夾結構

```
argocd-apps/
├── root-app.yaml           # Root Application，管理所有 ApplicationSet
├── apps-dev.yaml           # Dev 環境 ApplicationSet（自動同步）
├── apps-prod.yaml          # Prod 環境 ApplicationSet（手動同步）
├── infrastructure.yaml     # 基礎設施 ApplicationSet（自動同步）
└── README.md
```

#### ApplicationSet 說明

| ApplicationSet | 監聽路徑 | 目標 Namespace | 同步策略 |
|----------------|----------|----------------|----------|
| apps-dev | `apps/dev/*` | dev | 自動同步 |
| apps-prod | `apps/prod/*` | prod | 手動同步 |
| infrastructure | `infrastructure/*` | 由各資料夾定義 | 自動同步 |

#### Root Application

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: http://gitea-http.gitea.svc.cluster.local:3000/chialab/gitops-manifests.git
    targetRevision: HEAD
    path: argocd-apps
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

---

## 11. 故障排除檢查清單

如果 CI 失敗，依序檢查：

1. **Runner Pod 狀態**
   ```bash
   kubectl get pods -n gitea-runner
   kubectl logs -n gitea-runner actions-runner-act-runner-0 -c act-runner --tail=50
   ```

2. **Organization Variables/Secrets**
   - 確認 `REGISTRY_URL` Variable 存在
   - 確認 `REGISTRY_PASSWORD` Secret 存在

3. **DinD insecure-registries**
   ```bash
   kubectl exec -it actions-runner-act-runner-0 -n gitea-runner -c dind -- docker info | grep -A10 Insecure
   ```

4. **Gitea ROOT_URL**
   ```bash
   kubectl exec -n gitea deploy/gitea -- cat /data/gitea/conf/app.ini | grep ROOT_URL
   # 應該是 http://（不是 https://）
   ```

---

## 12. Gitea Repository 描述建議

| Repository | 建議描述 |
|------------|----------|
| `gitops-manifests` | K8s 部署清單，ArgoCD 自動同步 |
| `template-app` | 專案模板，含 CI/CD workflow |
| `test-frontend` | 端對端測試用 React App |

---

*返回 [README](README.md) | 上一章 [Namespace](04-namespaces.md) | 下一章 [Registry](06-registry.md)*
