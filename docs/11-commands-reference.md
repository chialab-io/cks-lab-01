# 常用指令參考

> 更新日期：2025-12-23

---

## 1. 叢集狀態

```bash
# 節點狀態
kubectl get nodes -o wide

# 所有 Pod 狀態
kubectl get pods -A

# LoadBalancer 服務
kubectl get svc -A | grep LoadBalancer
```

---

## 2. K3s 服務

```bash
# K3s 狀態
sudo systemctl status k3s

# 使用 K3s 內建 kubectl
sudo k3s kubectl get nodes

# 重啟 K3s
sudo systemctl restart k3s

# K3s 日誌
sudo journalctl -u k3s -f
```

---

## 3. Helm

```bash
# 列出所有安裝的 chart
helm list -A

# 更新 repo
helm repo update

# 安裝 chart
helm install <name> <chart> -n <namespace> -f values.yaml

# 升級 chart
helm upgrade <name> <chart> -n <namespace> -f values.yaml
```

---

## 4. nerdctl（容器操作）

```bash
# 列出容器
sudo nerdctl ps

# 列出 images
sudo nerdctl images

# 登入 Registry
sudo nerdctl login gitea.cks-lab-01.tailXXXXXX.ts.net --username ci-bot --insecure-registry

# Tag image
sudo nerdctl tag <source> gitea.cks-lab-01.tailXXXXXX.ts.net/chialab/<image>:<tag>

# Push image
sudo nerdctl push --insecure-registry gitea.cks-lab-01.tailXXXXXX.ts.net/chialab/<image>:<tag>
```

---

## 5. crictl（K3s 原生）

```bash
# 列出容器
sudo k3s crictl ps

# 列出 images
sudo k3s crictl images

# 清理未使用的 images
sudo k3s crictl rmi --prune
```

---

## 6. 網路

```bash
# Tailscale 狀態
tailscale status

# 列出所有 Ingress
kubectl get ingress -A

# 測試連線
curl -k https://gitea.cks-lab-01.tailXXXXXX.ts.net
```

---

## 7. 憑證

```bash
# 列出憑證
kubectl get certificates -A

# 列出 ClusterIssuer
kubectl get clusterissuer

# 檢查憑證詳情
kubectl describe certificate <name> -n <namespace>
```

---

## 8. PVC / PV

```bash
# 列出所有 PVC
kubectl get pvc -A

# 列出所有 PV
kubectl get pv

# 檢查 PVC 詳情
kubectl describe pvc <name> -n <namespace>
```

---

## 9. LVM

```bash
# 檢查邏輯卷
sudo lvs

# 檢查分區使用量
df -h /data /var/lib/rancher

# 擴展邏輯卷
sudo lvextend -L +50G /dev/ubuntu-vg/lv-data
sudo resize2fs /dev/ubuntu-vg/lv-data
```

---

## 10. ArgoCD

```bash
# Pod 狀態
kubectl get pods -n argocd

# CLI 登入
argocd login argocd.cks-lab-01.tailXXXXXX.ts.net --insecure --grpc-web

# 列出 Repo
argocd repo list

# 列出 Application
argocd app list

# 查看 Application 詳情
argocd app get <app-name>

# 同步 Application
argocd app sync <app-name>

# 刪除 Application
argocd app delete <app-name>
```

---

## 11. Gitea Actions Runner

```bash
# Pod 狀態
kubectl get pods -n gitea-runner

# 查看 Runner 日誌
kubectl logs -n gitea-runner actions-runner-act-runner-0 -c act-runner --tail=30

# 即時追蹤日誌
kubectl logs -n gitea-runner actions-runner-act-runner-0 -c act-runner -f

# 重啟 Runner
kubectl rollout restart statefulset actions-runner-act-runner -n gitea-runner
```

---

## 12. Namespace 資源配額

```bash
# dev 配額
kubectl describe resourcequota -n dev

# prod 配額
kubectl describe resourcequota -n prod

# dev LimitRange
kubectl describe limitrange -n dev

# prod LimitRange
kubectl describe limitrange -n prod
```

---

## 13. 測試應用

```bash
# dev namespace Pod
kubectl get pods -n dev

# dev namespace Service
kubectl get svc -n dev

# 查看 ArgoCD Application
argocd app get nginx-test
```

---

## 14. GPU 驗證（低優先）

```bash
# 檢查 GPU 資源
kubectl describe node | grep -A5 "nvidia.com/gpu"

# 檢查 NVIDIA Device Plugin
kubectl get pods -n kube-system | grep nvidia
```

---

## 15. 日誌與除錯

```bash
# Pod 日誌
kubectl logs <pod-name> -n <namespace>

# 即時追蹤
kubectl logs <pod-name> -n <namespace> -f

# 進入容器
kubectl exec -it <pod-name> -n <namespace> -- /bin/sh

# 端口轉發
kubectl port-forward svc/<service> <local-port>:<service-port> -n <namespace>

# 描述資源（除錯用）
kubectl describe pod <pod-name> -n <namespace>
kubectl describe deployment <deployment-name> -n <namespace>
```

---

*返回 [README](README.md) | 上一章 [安裝進度](10-installation-progress.md) | 下一章 [問題排解](12-troubleshooting.md)*
