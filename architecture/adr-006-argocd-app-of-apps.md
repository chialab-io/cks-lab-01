# ADR-006: 採用 ArgoCD App of Apps 模式

> **狀態**：Accepted  
> **日期**：2026-01-05  
> **決策者**：@wcwen

---

## 背景 (Context)

隨著 CKS 專案的應用數量增加，需要：
- 統一管理多個 ArgoCD Application
- 區分 Dev / Prod 環境的部署配置
- 管理基礎設施服務（Gitea、ArgoCD、Rancher）

目前的挑戰：
- 每個應用需要手動建立 ArgoCD Application
- 環境配置分散，難以追蹤
- 新增應用需要多個步驟

---

## 決策 (Decision)

我們選擇 **App of Apps** 模式來管理 ArgoCD Applications。

使用一個 Root App 來管理所有其他 Applications，形成層級結構。

---

## 架構

```
root-app (Root Application)
├── apps-dev.yaml        → dev namespace 下的所有應用
├── apps-prod.yaml       → prod namespace 下的所有應用
└── infrastructure.yaml  → 基礎設施服務
```

### GitOps Repo 結構

```
gitops-manifests/
├── argocd-apps/           # App of Apps 定義
│   ├── root-app.yaml
│   ├── apps-dev.yaml
│   ├── apps-prod.yaml
│   └── infrastructure.yaml
├── apps/
│   ├── dev/               # Dev 環境 manifests
│   └── prod/              # Prod 環境 manifests
└── infrastructure/        # 基礎設施 manifests
```

---

## 考量的替代方案

### 方案 A：手動管理每個 Application

| 優點 | 缺點 |
|------|------|
| 簡單直接 | 應用增多後管理困難 |
| 彈性高 | 無法統一追蹤變更 |

**不選擇的原因**：不符合 GitOps「Git 作為單一事實來源」原則

### 方案 B：ApplicationSet

| 優點 | 缺點 |
|------|------|
| 自動產生 Applications | 配置較複雜 |
| 支援多叢集 | 單叢集用 App of Apps 已足夠 |

**不選擇的原因**：單叢集環境，App of Apps 更簡單

---

## 相關服務

| 服務 | 影響 | 需要的變更 |
|------|------|-----------|
| ArgoCD | Root App 自動管理子 Apps | 建立 root-app.yaml |
| GitOps Repo | 目錄結構調整 | 重新組織 argocd-apps/ |

---

## 驗證方式

- [x] Root App 狀態為 Healthy
- [x] Sync Root App 自動建立/更新子 Apps
- [x] 新增應用只需在對應目錄加入 manifest

---

## 後果 (Consequences)

### 正面影響
- 所有 Application 定義集中在 Git
- 新增/刪除應用透過 PR 完成
- 環境分離清晰（dev/prod）

### 負面影響
- 多一層抽象，初學者需時間理解
- Root App sync 失敗會影響所有子 Apps

---

## 參考資料

- [ArgoCD App of Apps Pattern](https://argo-cd.readthedocs.io/en/stable/operator-manual/cluster-bootstrapping/)
