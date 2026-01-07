# 團隊權限規劃

> 更新日期：2025-01-06

---

## 1. Gitea 帳號架構

```
┌─────────────────────────────────────────────────────────────────┐
│                        Gitea 帳號架構                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  【系統層級】                                                    │
│  ├── admin (Site Admin)         # Gitea 系統管理專用            │
│  │   └── 用途：系統設定、使用者管理、緊急處理                    │
│  │   └── 平常不用，只在需要時登入                               │
│  │                                                              │
│  └── ci-bot (普通帳號) ✅ 已建立  # CI/CD 自動化專用            │
│      └── 用途：ArgoCD 拉取、Actions push image                  │
│      └── Access Token 已產生                                    │
│      └── 權限：Write（可 push image）                           │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  【組織】chialab ✅ 已建立                                       │
│  │                                                              │
│  ├── Owners (管理者)                                            │
│  │   └── wcwen ✅（開發者帳號）                                  │
│  │                                                              │
│  ├── Members (開發者)                                           │
│  │   ├── 組員 A 的帳號（待加入）                                 │
│  │   └── 組員 B 的帳號（待加入）                                 │
│  │                                                              │
│  └── ci-writers (CI 寫入權限) ✅                                 │
│      └── ci-bot ✅                                               │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  【Repositories】                                               │
│  └── chialab/                                                   │
│      ├── app-frontend            # 前端專案（待建立）            │
│      ├── app-backend             # 後端專案（待建立）            │
│      └── gitops-manifests ✅     # K8s 部署檔（ArgoCD 用）       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Gitea 帳號清單

| 帳號 | 類型 | 用途 | 狀態 |
|------|------|------|------|
| `admin` | Site Admin | Gitea 系統管理 | ✅ 已建立 |
| `wcwen` | 一般帳號 | 日常開發、組織 Owner | ✅ 已建立 |
| `ci-bot` | 一般帳號 | CI/CD 自動化（ArgoCD、Gitea Actions） | ✅ 已建立 + Token + Write 權限 |
| `組員A` | 一般帳號 | 開發 | ⏳ 待建立 |
| `組員B` | 一般帳號 | 開發 | ⏳ 待建立 |

---

## 3. K8s RBAC 角色定義（待實作）

| 角色 | 人員 | 權限範圍 |
|------|------|----------|
| **管理員** | wcwen | cluster-admin，完整存取 |
| **開發者** | 組員 | View Plus 權限（見下方） |

---

## 4. View Plus 權限設計（待實作）

```yaml
View Plus 權限:
  ✅ 可查看:
    - Pod、Deployment、Service、Ingress、ConfigMap
    - Events、ReplicaSet、StatefulSet
  
  ✅ 可調試:
    - kubectl logs -f（查看日誌）
    - kubectl exec（進容器檢查）
    - kubectl port-forward（本機測試）
  
  ❌ 不可修改:
    - 無法 apply、patch、edit、delete
    - 無法存取系統 namespace
    - 無法請求 GPU 資源
```

---

## 5. 各平台存取方式

| 工具 | 管理員 (wcwen) | 組員 |
|------|----------------|------|
| **kubectl** | cluster-admin kubeconfig | 受限 kubeconfig（View Plus）|
| **Rancher** | Admin | Project Member (dev/prod) |
| **Portainer** | Admin | 限定環境存取 |
| **ArgoCD** | Admin | 唯讀（查看部署狀態）|
| **Gitea** | Organization Owner | Developer |

---

## 6. 團隊成員設定

### 新版設定（推薦）✅

**2025-01-04 更新：使用 Cloudflare DNS，無需設定 hosts！**

1. 安裝 [Tailscale](https://tailscale.com/)
2. 加入團隊網路
3. 直接存取以下 URL：

| 服務 | URL | 備註 |
|------|-----|------|
| Gitea | http://git.int.cks-lab.uk | |
| ArgoCD | http://argo.int.cks-lab.uk | |
| **Rancher** | **https://rancher.int.cks-lab.uk** | ⚠️ 必須 HTTPS，忽略憑證警告 |
| Portainer | http://portainer.int.cks-lab.uk | |

> ⚠️ **Rancher 必須使用 HTTPS**：Rancher 的 CSRF 保護機制要求 HTTPS，HTTP 存取會導致 "Invalid CSRF token" 錯誤。瀏覽器會警告自簽憑證，點擊「進階」→「繼續前往」即可。

### 舊版設定（仍可用）

如需使用舊版 `*.cks-lab-01.tailXXXXXX.ts.net` 域名，需設定 hosts：

**Windows:** `C:\Windows\System32\drivers\etc\hosts`  
**Mac/Linux:** `/etc/hosts`

```
# K3s 內網服務（透過 Tailscale）- 舊版
100.x.x.x rancher.cks-lab-01.tailXXXXXX.ts.net
100.x.x.x portainer.cks-lab-01.tailXXXXXX.ts.net
100.x.x.x gitea.cks-lab-01.tailXXXXXX.ts.net
100.x.x.x argocd.cks-lab-01.tailXXXXXX.ts.net
```

#### Mac 修改方式

```bash
sudo nano /etc/hosts
# 貼上上述內容後儲存
# Ctrl+O 儲存, Ctrl+X 離開
```

---

## 7. Git 操作設定

### 新版域名（推薦）

使用 HTTP，無需特殊設定：

```bash
git clone http://git.int.cks-lab.uk/chialab/<repo>.git
```

### 舊版域名（SSL 問題）

由於使用 self-signed 憑證，需跳過 SSL 驗證：

```bash
# 單次跳過 SSL 驗證
git -c http.sslVerify=false clone https://gitea.cks-lab-01.tailXXXXXX.ts.net/chialab/<repo>.git

# 進入 repo 後設定永久跳過
cd <repo>
git config http.sslVerify false

# 或針對此 domain 全域設定
git config --global http.https://gitea.cks-lab-01.tailXXXXXX.ts.net/.sslVerify false
```

---

*返回 [README](README.md) | 上一章 [管理介面](08-management-ui.md) | 下一章 [安裝進度](10-installation-progress.md)*
