# ADR-007: Gitea Registry 使用 HTTP 協議

> **狀態**：Accepted  
> **日期**：2025-12-30  
> **決策者**：@wcwen

---

## 背景 (Context)

在配置 CI/CD pipeline 時，需要讓 Gitea Actions Runner（DinD 架構）能夠：
- 建置 Docker image
- 登入 Gitea Container Registry
- 推送 image 到 Registry

遇到的問題：
- Gitea 的 `ROOT_URL` 設為 HTTPS
- Container Registry 的 token endpoint URL 是根據 `ROOT_URL` 產生
- DinD 容器無法驗證 Gitea 的自簽憑證，導致 `context deadline exceeded` 錯誤

---

## 決策 (Decision)

我們選擇將 **Gitea ROOT_URL 設為 HTTP**，並使用 **insecure registry** 配置。

---

## 技術細節

### 問題根本原因

Gitea Container Registry 的認證流程：
1. `docker login` 時，Docker 請求 `/v2/token` endpoint
2. Gitea 根據 `ROOT_URL` 產生 token endpoint URL
3. 若 `ROOT_URL` 是 HTTPS，token endpoint 也會是 HTTPS
4. DinD 容器不信任自簽憑證，無法連接 HTTPS endpoint

### 解決方案

1. **Gitea ROOT_URL 改為 HTTP**：透過 Helm values 設定
2. **DinD 配置 insecure-registries**：允許 HTTP registry 存取
3. **建立 gitea-registry Service**：提供內部 DNS 解析

---

## 考量的替代方案

### 方案 A：配置 DinD 信任自簽憑證

| 優點 | 缺點 |
|------|------|
| 維持 HTTPS | DinD 憑證配置複雜 |
| 更安全 | 需要額外的 Secret 管理 |

**不選擇的原因**：Gitea Helm chart 的 TLS 設定複雜，且內網環境 HTTP 已足夠安全

### 方案 B：使用公開 Registry（如 Docker Hub）

| 優點 | 缺點 |
|------|------|
| 無需自建 | 私有 image 需付費 |
| 無 TLS 問題 | 對外流量增加 |

**不選擇的原因**：希望完整體驗私有 Registry 架構

---

## 資源影響

無額外資源消耗。

---

## 相關服務

| 服務 | 影響 | 需要的變更 |
|------|------|-----------|
| Gitea | ROOT_URL 設為 HTTP | Helm values 更新 |
| DinD | 信任 HTTP registry | 啟動參數加入 `--insecure-registry` |
| K3s | 拉取 image | `registries.yaml` 設定 `insecure_skip_verify` |

---

## 回滾計畫

若需改回 HTTPS：

```bash
helm upgrade gitea gitea/gitea -n gitea \
  --set gitea.config.server.ROOT_URL=https://gitea.example.com/
```

並配置 DinD 信任憑證（較複雜）。

---

## 驗證方式

- [x] `docker login` 到 Gitea Registry 成功
- [x] `docker push` image 成功
- [x] K8s Pod 可拉取 Gitea Registry 的 image

---

## 後果 (Consequences)

### 正面影響
- CI pipeline 可以正常建置和推送 image
- 配置簡化，無需處理 TLS 憑證

### 負面影響
- 內網通訊使用 HTTP（未加密）
- 安全性依賴內網隔離（Tailscale）

### 安全考量

此配置適用於 **內網環境**，因為：
- 所有存取都透過 Tailscale VPN
- 外部存取使用 Cloudflare Tunnel（自帶 HTTPS）
- 無敏感資料在 HTTP 傳輸（Registry 認證 token 有時效性）

---

## 參考資料

- [Gitea Container Registry 文檔](https://docs.gitea.io/en-us/packages/container/)
- [Docker insecure registries](https://docs.docker.com/registry/insecure/)
