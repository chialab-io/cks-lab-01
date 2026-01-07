# ADR-005: 選擇 Gitea Actions 作為 CI 工具

> **狀態**：Accepted  
> **日期**：2025-12-25  
> **決策者**：@wcwen

---

## 背景 (Context)

CKS 專案需要 CI 工具來：
- 自動建置 Docker image
- 推送 image 到 Container Registry
- 更新 GitOps Repo 觸發部署

資源限制：
- 總共 32GB RAM，需預留給 K3s 和應用
- 單節點環境，無法分散 CI 負載

已安裝的服務：
- Gitea（Git server + Container Registry）

---

## 決策 (Decision)

我們選擇 **Gitea Actions** 作為 CI 工具。

Gitea Actions 是 Gitea 內建的 CI/CD 功能，語法與 GitHub Actions 相容。

---

## 考量的替代方案

### 方案 A：Jenkins

| 優點 | 缺點 |
|------|------|
| 功能強大，插件豐富 | 記憶體消耗大（建議 4Gi+） |
| 企業級支援 | 配置複雜（Groovy pipeline） |

**不選擇的原因**：資源消耗過大，家用環境無法負擔

### 方案 B：GitLab CI

| 優點 | 缺點 |
|------|------|
| 整合 Git + CI + Registry | 整體資源消耗大 |
| 成熟的 CI 功能 | 需要額外安裝 GitLab |

**不選擇的原因**：已有 Gitea，不需另外安裝 GitLab

### 方案 C：Drone CI

| 優點 | 缺點 |
|------|------|
| 輕量、雲原生 | 需要額外安裝和維護 |
| 與 Gitea 整合良好 | 需要學習新的 pipeline 語法 |

**不選擇的原因**：Gitea Actions 已內建，無需額外維護

---

## 資源影響

| 資源 | 預估用量 | 備註 |
|------|----------|------|
| CPU | 2-4 cores | 建置時 |
| Memory | 1-2Gi | Runner + Job container |
| Storage | ~5Gi | Runner 快取 |

---

## 相關服務

| 服務 | 影響 | 需要的變更 |
|------|------|-----------|
| Gitea | 啟用 Actions 功能 | Helm values 設定 |
| DinD | Runner 內部建置 Docker | 使用 Docker-in-Docker 架構 |
| Gitea Registry | 推送 image | 使用內部 DNS |

---

## 回滾計畫

若 Gitea Actions 不符需求：

```bash
# 可改用 Drone CI
helm install drone drone/drone -n drone
```

---

## 驗證方式

- [x] Runner 在 Gitea 管理介面顯示 Online
- [x] Push 到 Repo 觸發 workflow
- [x] Workflow 成功建置並推送 image

---

## 後果 (Consequences)

### 正面影響
- 無需額外安裝 CI 服務，減少資源消耗
- 與 GitHub Actions 語法相容，學習曲線低
- 與 Gitea Registry 原生整合

### 負面影響
- Actions 功能較 GitHub Actions 少（部分 action 需自行維護）
- DinD 架構增加配置複雜度

### 需要後續處理
- 安裝 act-runner（Docker-in-Docker 模式）
- 配置 Registration Token（必須用 API 取得，見 troubleshooting）

---

## 參考資料

- [Gitea Actions 文檔](https://docs.gitea.io/en-us/usage/actions/)
- [act-runner 部署指南](https://gitea.com/gitea/act_runner)
