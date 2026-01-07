# ADR-003: 採用 Tailscale 作為遠端存取方案

> **狀態**：Accepted  
> **日期**：2025-12-23  
> **決策者**：@wcwen

---

## 背景 (Context)

CKS 伺服器位於家用網路，需要允許：
- 管理員遠端管理 K8s 叢集
- 團隊成員存取內部服務（Gitea、ArgoCD、Rancher）
- 行動裝置上進行緊急維護

家用 ISP 限制：
- 無固定 IP
- NAT 環境，無法直接對外開放 port
- 頻寬有限，不適合大量對外流量

---

## 決策 (Decision)

我們選擇 **Tailscale** 作為遠端存取方案。

Tailscale 是基於 WireGuard 的零配置 VPN 網格，提供 NAT 穿透和 MagicDNS 功能。

---

## 考量的替代方案

### 方案 A：自建 WireGuard VPN

| 優點 | 缺點 |
|------|------|
| 完全自控 | 需自行處理 NAT 穿透 |
| 無訂閱費用 | 需維護 endpoint 配置 |

**不選擇的原因**：NAT 穿透配置複雜，需要 STUN server

### 方案 B：OpenVPN

| 優點 | 缺點 |
|------|------|
| 成熟穩定 | 效能較差（TCP 封裝） |
| 廣泛支援 | 配置繁瑣 |

**不選擇的原因**：效能不如 WireGuard，配置管理困難

### 方案 C：Zerotier

| 優點 | 缺點 |
|------|------|
| 類似 Tailscale | 免費版功能較少 |
| 跨平台支援 | 穿透成功率略低 |

**不選擇的原因**：Tailscale 的 MagicDNS 更易用

---

## 資源影響

| 資源 | 預估用量 | 備註 |
|------|----------|------|
| CPU | ~0.01 cores | 加密開銷極低 |
| Memory | ~20Mi | Tailscale daemon |
| Network | 視使用量 | P2P 連線 |

---

## 相關服務

| 服務 | 影響 | 需要的變更 |
|------|------|-----------|
| Nginx Ingress | 監聽 Tailscale IP | 啟用 hostPort |
| DNS | 使用 MagicDNS 主機名 | 子域名需額外 hosts 設定 |

---

## 驗證方式

- [x] `tailscale status` 顯示 online
- [x] 從遠端設備可 ping 到伺服器 Tailscale IP
- [x] 透過 Tailscale IP 可存取 Ingress 服務

---

## 後果 (Consequences)

### 正面影響
- 零配置 NAT 穿透，家用環境友善
- MagicDNS 提供主機名解析
- 團隊成員輕鬆加入（手機、筆電）
- 免費版足夠小型團隊使用

### 負面影響
- MagicDNS 不支援子域名（需手動維護 /etc/hosts）
- 依賴 Tailscale 協調伺服器（P2P 連線建立後不依賴）

### 需要後續處理
- 團隊成員需在各自設備安裝 Tailscale
- 需在本機 hosts 檔案加入服務子域名對應

---

## 參考資料

- [Tailscale 官方文檔](https://tailscale.com/kb/)
- [Tailscale vs WireGuard 比較](https://tailscale.com/compare/wireguard/)
