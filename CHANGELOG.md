# 變更紀錄

> 此文件記錄文檔的重要變更

---

## 2026-01-07（文檔結構整理）

### 目錄結構調整

將非技術手冊類文件移至 `lab-01/` 根目錄：

| 檔案 | 原位置 | 新位置 |
|------|--------|--------|
| `README.md` | `docs/` | `lab-01/` |
| `CHANGELOG.md` | `docs/` | `lab-01/` |
| `GEMINI.md` | `docs/` | `lab-01/` |

### 文件合併

- 將 `runner-config-backup.md` 的完整內容合併到 `13-config-backup.md` 第 6 節
- 刪除原 `runner-config-backup.md`

### 最終結構

```
cks/lab/lab-01/
├── README.md              # 專案入口
├── CHANGELOG.md           # 變更紀錄
├── GEMINI.md              # AI 協助指南
├── architecture/          # WHY - 設計決策 (ADR)
│   └── adr-*.md
└── docs/                  # WHAT - 技術手冊
    ├── 01-15 技術文件
    └── 13-config-backup.md  # 含 Runner 完整配置
```

### README.md 更新

- 更新所有相對路徑連結（加上 `docs/` 前綴）
- 移除 `runner-config-backup.md` 連結
- 文檔版本升級至 v21.0

---

## 2026-01-06（Prod 部署流程驗證）

### 🎉 里程碑：Prod 環境部署驗證完成！

**完成 Dev/Prod 雙環境部署，端對端 CI/CD 流程驗證成功**

### 完成項目

#### 域名策略更新
- Dev 環境：`<app>-dev.int.cks-lab.uk`（內網）
- Prod 環境：`<app>-prod.int.cks-lab.uk`（內網）+ `<app>.cks-lab.uk`（對外）
- test-frontend 對外域名：`demo.cks-lab.uk` → Prod

#### Prod 部署配置
- 建立 `apps/prod/test-frontend/deployment.yaml`
- Prod 使用 2 replicas、較高資源配置
- Prod 使用手動同步策略（安全性考量）

#### CI/CD 流程驗證
- 驗證 `ci-prod.yaml` workflow（push to main 觸發）
- 驗證 `prod-latest` / `prod-<date>-<hash>` image tag 產生
- 驗證 ArgoCD ApplicationSet 自動發現 Prod 應用

### 問題修復

#### Ingress Host 衝突
- **問題**：ArgoCD Sync 失敗，錯誤 `host "demo.cks-lab.uk" is already defined in ingress dev/demo-public`
- **原因**：同一 host 被定義在多個 Ingress 中
- **解決**：刪除舊的 `dev/demo-public` Ingress

### 文檔更新
- 10-installation-progress.md：新增 Prod 部署驗證完成、更新進度總覽
- 12-troubleshooting.md：新增第 22 節「Ingress Host 衝突」

### 當前 ArgoCD Applications

| Application | Path | Namespace | Sync Policy |
|-------------|------|-----------|-------------|
| root-app | argocd-apps | argocd | Automated |
| test-frontend | apps/dev/test-frontend | dev | Automated |
| test-frontend-prod | apps/prod/test-frontend | prod | Manual |
| infra-cloudflared | infrastructure/cloudflared | cloudflare-system | Automated |

---

## 2025-01-06（Rancher 內網 HTTPS 修正）

### 問題修復

#### Rancher Invalid CSRF Token 錯誤
- **問題**：使用 HTTP 存取 `http://rancher.int.cks-lab.uk` 時出現 "Invalid CSRF token" 錯誤
- **原因**：Rancher 的 CSRF 保護機制要求 HTTPS，HTTP 存取會導致 Secure cookie 無法正確傳遞
- **解決方案**：為 Rancher 內網域名加上 HTTPS + 自簽憑證

#### Ingress 配置更新
- `rancher-int` Ingress 改為 HTTPS（加入 TLS 配置）
- 使用 `selfsigned-issuer` 簽發 `rancher-int-tls` 憑證

### 文檔更新

#### 07-networking.md
- 更新網路架構圖：Rancher 標示需 HTTPS
- 更新服務 URL 對照表：Rancher 協定改為 HTTPS
- 新增 Rancher 必須使用 HTTPS 的說明
- 新增內網請求（存取 Rancher）的流量路徑

#### 08-management-ui.md
- 更新存取位址表格：Rancher 標示需 HTTPS
- 更新 Rancher Ingress 配置（加入 TLS）
- 新增 TLS 憑證章節說明各服務協定差異

#### 12-troubleshooting.md
- 新增第 21 節「Rancher 登入時 Invalid CSRF Token 錯誤」
- 詳細記錄問題原因與解決步驟

#### 13-config-backup.md
- 更新 Rancher 內網 Ingress 配置（HTTPS 版本）
- 新增 2025-01-06 更新說明

#### 15-quick-reference.md
- 更新服務 URL 總覽：Rancher 改為 HTTPS
- 新增備註說明需忽略憑證警告

#### README.md
- 更新最新進度
- 更新管理介面 URL（Rancher 改為 HTTPS）
- 文檔版本升級至 v20.0

### 內網服務協定總覽

| 服務 | 協定 | 原因 |
|------|------|------|
| Gitea | HTTP | 無 CSRF 限制 |
| ArgoCD | HTTP | 已設定 insecure mode |
| **Rancher** | **HTTPS** | CSRF 保護機制要求 |
| Portainer | HTTP | 無 CSRF 限制 |

---

## 2025-01-05（cloudflared GitOps + Gitea ROOT_URL 更新）

### 完成項目

#### cloudflared 加入 GitOps
- 建立 `infrastructure/cloudflared/` 資料夾結構
- 建立 ArgoCD Application（自動同步）
- Secret 維持手動管理（不進 Git）
- 修正 health probe 配置（啟用 metrics server）

#### Gitea ROOT_URL 更新
- 更新為 `http://git.int.cks-lab.uk/`
- 透過 Helm values 設定（直接改 app.ini 會被覆蓋）
- 停用 Helm 管理的 Ingress（使用手動建立的 Ingress）
- 解決登入時的 ROOT_URL 不匹配警告

#### gitops-manifests 結構更新
```
gitops-manifests/
├── apps/
│   ├── dev/
│   └── prod/
└── infrastructure/     ✅ 新增
    └── cloudflared/
        ├── namespace.yaml
        ├── deployment.yaml
        └── README.md
```

### 文檔更新
- 10-installation-progress.md：新增完成項目、ArgoCD Applications 列表
- 13-config-backup.md：新增 Gitea Helm Values 設定、更新 cloudflared 配置
- README.md：版本升級至 v19.0

### 問題修復
- cloudflared health probe 失敗：需啟用 `--metrics 0.0.0.0:2000` 參數
- Gitea ROOT_URL 不匹配警告：透過 Helm upgrade 更新設定

---

## 2025-01-04（Cloudflare Tunnel + 域名設定）

### 🎉 里程碑：網路架構全面升級！

**團隊成員不再需要設定 hosts 檔案！**

### 完成項目

#### 域名註冊
- 註冊 `cks-lab.uk` 域名（Cloudflare Registrar，$5/年）

#### Cloudflare Tunnel
- 建立 Tunnel（名稱：`cks-lab`）
- 部署方式：K8s Deployment（2 replicas，高可用）
- Namespace：`cloudflare-system`
- 狀態：Healthy ✅

#### DNS 分流設定
- `*.int.cks-lab.uk` → A 記錄 → 100.96.128.95（Tailscale IP，DNS-only）
- `*.cks-lab.uk` → Cloudflare Tunnel（對外展示）

#### 內網 Ingress 新增
- `git.int.cks-lab.uk` → Gitea
- `argo.int.cks-lab.uk` → ArgoCD
- `rancher.int.cks-lab.uk` → Rancher
- `portainer.int.cks-lab.uk` → Portainer

#### 對外服務路由
- `demo.cks-lab.uk` → test-frontend（展示用）

### 文檔更新

#### 07-networking.md（大幅重寫）
- 新增完整網路架構圖
- 新增域名分流策略說明
- 新增 Cloudflare Tunnel 配置說明
- 新增流量路徑總覽
- 更新團隊成員設定（簡化版）

#### 03-k3s-cluster.md
- Namespace 架構新增 `cloudflare-system`
- 已安裝元件新增 Cloudflare Tunnel

#### 08-management-ui.md（大幅重寫）
- 新增新版域名（*.int.cks-lab.uk）存取方式
- 新增新版 Ingress 配置
- 保留舊版域名配置（向後相容）
- 新增對外展示 Ingress 配置

#### 09-team-permissions.md
- 更新團隊成員設定為簡化版
- 新增新版域名使用說明
- 新增 Git 操作設定（新版 HTTP vs 舊版 HTTPS）

#### 10-installation-progress.md
- 新增已完成項目：Cloudflare 相關設定
- 更新進度總覽：新增 Step 11
- 新增網路架構章節
- 更新待完成項目

#### 13-config-backup.md
- 新增 Cloudflare Tunnel 配置
- 新增內網 Ingress 配置（*.int.cks-lab.uk）
- 新增對外 Ingress 配置
- 新增 Cloudflare DNS 記錄

#### README.md
- 更新最新進度
- 更新快速開始（簡化版團隊成員設定）
- 更新管理介面 URL（新版域名）
- 新增對外展示 URL
- 新增網路架構圖
- 文檔版本升級至 v18.0

### 新舊域名對照

| 服務 | 舊版域名（需 hosts） | 新版域名（免設定） |
|------|----------------------|-------------------|
| Gitea | gitea.cks-lab-01.tailXXXXXX.ts.net | git.int.cks-lab.uk |
| ArgoCD | argocd.cks-lab-01.tailXXXXXX.ts.net | argo.int.cks-lab.uk |
| Rancher | rancher.cks-lab-01.tailXXXXXX.ts.net | rancher.int.cks-lab.uk |
| Portainer | portainer.cks-lab-01.tailXXXXXX.ts.net | portainer.int.cks-lab.uk |

---

## 2025-12-31（CI/CD 流程整理與文檔更新）

### 完成項目

#### Prod 環境配置
- prod namespace imagePullSecret 建立完成
- main 分支保護設定完成（template-app、test-frontend）

#### gitops-manifests 整理
- 移除 nginx-test 測試應用
- 刪除空的 `app/` 資料夾（統一使用 `apps/`）
- 刪除測試用 `.gitea/workflows/test-runner.yaml`
- 建立 `apps/prod/.gitkeep` 預留資料夾
- 更新 README 加入完整使用說明

### 文檔更新

#### 05-cicd.md（大幅重寫）
- 新增「整體架構概覽」圖示
- 新增「Repository 分類與用途」章節
- 新增「建立新專案流程」完整步驟
- 新增「Gitea Repository 描述建議」

#### 10-installation-progress.md
- 新增已完成項目
- 更新待完成項目優先級
- 新增「Repository 架構」章節

---

## 2025-12-30（晚上 - 端對端 GitOps 流程驗證）

### 🎉 里程碑：端對端 GitOps 流程驗證完成！

**從模板建立專案 → CI Build → Push Image → ArgoCD 部署 → Ingress 存取，全流程驗證成功**

### 完成項目

#### test-frontend 端對端測試
- 從 template-app 模板建立 test-frontend 專案
- 建立最小可運行 React App（Vite + React 18）
- CI Pipeline 成功 Build 並 Push Image 到 Registry
- ArgoCD Application 建立並自動同步
- 建立 Ingress 透過域名存取

---

## 2025-12-30（下午 - CI/CD 分支策略與模板）

### 🎉 里程碑：CI/CD 完整架構建立

**完成 develop/main 分支策略、模板儲存庫建立**

### 新增功能

#### CI/CD 分支策略
- **develop 分支**：開發測試，所有人可 push，自動 build `dev-<hash>` image
- **main 分支**：正式環境，需透過 PR 審核，自動 build `prod-<date>-<hash>` image

#### 模板儲存庫 (template-app)
- CI workflow 模板（ci-dev.yaml / ci-prod.yaml）
- Dockerfile 多選項模板
- nginx.conf（SPA 路由支援）

---

## 2025-12-30（上午 - CI Pipeline 完整驗證）

### 🎉 里程碑：CI Pipeline 完整驗證通過

**CI Build → Push Image 流程已完整運作！**

---

## 2025-12-25

### 修改檔案

#### 12-troubleshooting.md
- 新增第 10 節「重開機後 K3s 資料遺失（LVM 未自動掛載）」

#### 02-storage.md
- 新增第 2 節「LVM 自動掛載設定（fstab）」

---

## 2025-12-23

### 初始版本
- 建立完整文檔結構（01-15 共 15 個檔案）
- 文檔版本：v12.0

---

*返回 [README](README.md)*
