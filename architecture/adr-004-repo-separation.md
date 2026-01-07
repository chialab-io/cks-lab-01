# ADR-004: 分離 App Repo 與 GitOps Repo

> **狀態**：Accepted  
> **日期**：2025-12-25  
> **決策者**：@wcwen

---

## 背景 (Context)

在建置 CI/CD 流程時，需要決定如何管理：
- **應用程式原始碼**（React、FastAPI 等）
- **K8s 部署清單**（Deployment、Service、Ingress）

主要考量：
- 讓開發者專注於功能開發，不需理解 K8s 複雜度
- 讓部署變更有獨立的變更歷史
- 符合 GitOps 原則（Git 作為單一事實來源）

---

## 決策 (Decision)

我們選擇 **分離 App Repo 和 GitOps Repo**。

| Repo | 內容 | 監聽者 |
|------|------|--------|
| **App Repo** (如 `test-frontend`) | Source + Dockerfile + CI workflow | Gitea Actions |
| **GitOps Repo** (`gitops-manifests`) | K8s manifests + ArgoCD Apps | ArgoCD |

---

## 考量的替代方案

### 方案 A：Monorepo（應用 + K8s 清單放一起）

| 優點 | 缺點 |
|------|------|
| 單一 Repo，管理簡單 | 變更歷史混雜 |
| 版本一致性 | 開發者需理解 K8s |

**不選擇的原因**：變更追蹤困難，開發者與 K8s 配置耦合

### 方案 B：App Repo 內建 overlay（Kustomize）

| 優點 | 缺點 |
|------|------|
| 配置靠近原始碼 | 每個 App 都要維護 K8s 配置 |
| 開發者可自行調整 | 跨 App 配置難以統一 |

**不選擇的原因**：單一團隊維護多 App 時，集中管理更有效率

---

## 相關服務

| 服務 | 影響 | 需要的變更 |
|------|------|-----------|
| Gitea Actions | 觸發 CI | 監聽 App Repo 的 push |
| ArgoCD | 觸發 CD | 監聽 GitOps Repo 的變更 |
| CI Workflow | 更新 manifests | CI 完成後更新 GitOps Repo 的 image tag |

---

## 驗證方式

- [x] App Repo push → CI 觸發 → image build/push 成功
- [x] CI 更新 GitOps Repo 的 image tag
- [x] ArgoCD 偵測變更 → 自動/手動 Sync

---

## 後果 (Consequences)

### 正面影響
- 關注點分離：開發者寫代碼，維運者管 manifests
- 變更歷史清晰：App 變更和部署變更分開追蹤
- 符合 GitOps 最佳實踐

### 負面影響
- CI 需要額外步驟更新 GitOps Repo
- 需維護兩個 Repo 的存取權限

### 需要後續處理
- CI workflow 需要有權限 push 到 GitOps Repo
- 建立 ci-bot 帳號，設定適當的 Token scope

---

## 參考資料

- [GitOps 最佳實踐 - Weaveworks](https://www.weave.works/technologies/gitops/)
- [ArgoCD 官方範例](https://github.com/argoproj/argocd-example-apps)
