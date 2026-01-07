# ADR-002: 使用 Nginx Ingress 取代 Traefik

> **狀態**：Accepted  
> **日期**：2025-12-23  
> **決策者**：@wcwen

---

## 背景 (Context)

K3s 預設安裝 Traefik 作為 Ingress Controller。然而，在實際使用中發現：
- 團隊對 Nginx 的配置語法更熟悉
- 部分管理工具（Rancher、ArgoCD）在 Nginx Ingress 下的設定文件較多
- Traefik 的 middleware 設定與 Nginx annotation 不相容

---

## 決策 (Decision)

我們選擇 **停用 K3s 預設的 Traefik**，改用 **Nginx Ingress Controller**。

---

## 考量的替代方案

### 方案 A：繼續使用 Traefik

| 優點 | 缺點 |
|------|------|
| K3s 預設，無需額外安裝 | 團隊不熟悉 middleware 語法 |
| 自動 Let's Encrypt | 路徑重寫設定較複雜 |

**不選擇的原因**：學習成本高，網路上 Nginx annotation 範例更多

### 方案 B：使用 HAProxy Ingress

| 優點 | 缺點 |
|------|------|
| 高效能 | 社群生態較小 |
| 細緻的負載均衡 | 配置複雜度高 |

**不選擇的原因**：對小型專案過度設計

---

## 資源影響

| 資源 | 預估用量 | 備註 |
|------|----------|------|
| CPU | ~0.1 cores | DaemonSet 模式 |
| Memory | ~150Mi | 單 Pod |

---

## 相關服務

| 服務 | 影響 | 需要的變更 |
|------|------|-----------|
| K3s 安裝 | 停用 Traefik | `--disable=traefik` 參數 |
| MetalLB | 提供 LoadBalancer IP | 需先安裝 |
| cert-manager | 管理 TLS 憑證 | 發行自簽憑證 |

---

## 回滾計畫

若需要回退到 Traefik：

```bash
# 刪除 Nginx Ingress
helm uninstall ingress-nginx -n ingress-nginx

# 重新啟用 Traefik（需重裝 K3s）
curl -sfL https://get.k3s.io | sh -
```

---

## 驗證方式

- [x] `kubectl get pods -n ingress-nginx` 顯示 Running
- [x] `kubectl get svc -n ingress-nginx` 有 LoadBalancer IP
- [x] Ingress 資源可正常路由到後端 Service

---

## 後果 (Consequences)

### 正面影響
- 團隊熟悉的 annotation 語法（如 `nginx.ingress.kubernetes.io/rewrite-target`）
- 網路上大量 Nginx Ingress 設定範例可參考
- 與 Rancher、ArgoCD 官方文件匹配

### 負面影響
- 失去 Traefik 的自動 Let's Encrypt 整合（改用 cert-manager）
- 需額外維護 Nginx Ingress 的 Helm release

---

## 參考資料

- [Nginx Ingress Controller 文檔](https://kubernetes.github.io/ingress-nginx/)
- [K3s 停用服務](https://docs.k3s.io/installation/packaged-components)
