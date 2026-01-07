# ADR-008: 使用 Cloudflare Tunnel 對外暴露服務

> **狀態**：Accepted  
> **日期**：2026-01-05  
> **決策者**：@wcwen

---

## 背景 (Context)

CKS 專案需要對外展示應用（如 demo、portfolio），但：
- 家用 ISP 無固定 IP
- 不想直接暴露伺服器 IP
- 需要 HTTPS（瀏覽器安全要求）

需求：
- 外部使用者可透過公開域名存取應用
- 管理員控制哪些服務對外開放
- 自動 HTTPS 憑證管理

---

## 決策 (Decision)

我們選擇 **Cloudflare Tunnel** 作為對外暴露服務的方案。

Cloudflare Tunnel 在本地運行 `cloudflared` daemon，建立到 Cloudflare edge 的加密連線。

---

## 考量的替代方案

### 方案 A：Ngrok

| 優點 | 缺點 |
|------|------|
| 設定簡單 | 免費版限制多（隨機域名） |
| 即時可用 | 有流量限制 |

**不選擇的原因**：免費版無法使用自訂域名

### 方案 B：直接 Port Forwarding + DDNS

| 優點 | 缺點 |
|------|------|
| 無第三方依賴 | 暴露家用 IP |
| 完全自控 | 需自行處理 HTTPS 憑證 |

**不選擇的原因**：安全風險高，暴露家用網路

### 方案 C：VPS 反向代理

| 優點 | 缺點 |
|------|------|
| 固定 IP | 需額外租用 VPS |
| 完全自控 | 增加維護成本 |

**不選擇的原因**：增加月費支出，維護負擔

---

## 架構

```
外部使用者 → Cloudflare Edge (HTTPS)
                    ↓ Tunnel（加密）
              cloudflared Pod
                    ↓
              Nginx Ingress → Service → Pod
```

### 域名策略

| 域名模式 | 用途 | 存取方式 |
|----------|------|----------|
| `*.cks-lab.uk` | 對外展示 | Cloudflare Tunnel |
| `*.int.cks-lab.uk` | 內網管理 | Tailscale |

---

## 資源影響

| 資源 | 預估用量 | 備註 |
|------|----------|------|
| CPU | ~0.05 cores | cloudflared daemon |
| Memory | ~50Mi | |

---

## 相關服務

| 服務 | 影響 | 需要的變更 |
|------|------|-----------|
| Cloudflare DNS | 域名解析 | 託管 `cks-lab.uk` |
| Nginx Ingress | 路由對外流量 | 建立對應 Ingress |
| cloudflared | 運行 Tunnel | 部署為 K8s Deployment |

---

## 驗證方式

- [x] `cloudflared tunnel list` 顯示 tunnel healthy
- [x] 外部瀏覽器可存取 `https://demo.cks-lab.uk`
- [x] HTTPS 憑證有效（Cloudflare 管理）

---

## 後果 (Consequences)

### 正面影響
- 家用 IP 完全隱藏
- 自動 HTTPS 憑證（Cloudflare Universal SSL）
- 無需 port forwarding，安全性高
- 免費方案足夠個人使用

### 負面影響
- 依賴 Cloudflare 服務
- 多一層 latency（經過 Cloudflare edge）
- 需要將域名 DNS 託管到 Cloudflare

---

## 參考資料

- [Cloudflare Tunnel 文檔](https://developers.cloudflare.com/cloudflare-one/connections/connect-apps/)
- [cloudflared K8s 部署](https://developers.cloudflare.com/cloudflare-one/tutorials/many-cfd-one-tunnel/)
