# ArgoCD 快速上手：GitOps 部署 + Self-Heal 實測

目標：在本機建一個小叢集，用 GitOps 方式部署一個 App，實測「git commit 自動同步」與「self-heal 防手動漂移」這兩個 ArgoCD 核心能力。

前置需求：Mac + Docker（跑 kind 用）、`kubectl`、`brew`。

---

## Step 1：建立本機 Kubernetes 叢集（kind）

```bash
brew install kind
kind create cluster --name devops-demo
kubectl cluster-info --context kind-devops-demo
```

> `kind get nodes` 列出的是叢集節點（container 扮演的 VM），不是 App 或 Pod。要看 App 用 `kubectl get pods -n <namespace>`。

## Step 2：安裝 ArgoCD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl get pods -n argocd -w
```

> `argocd` namespace 放的是 ArgoCD 自己的控制平面元件（argocd-server、repo-server、application-controller），跟它管理的目標 App 要部署到哪個 namespace 是兩件互相獨立的事。

## Step 3：登入 ArgoCD（CLI + UI）

```bash
brew install argocd

kubectl port-forward svc/argocd-server -n argocd 8080:443 &

argocd admin initial-password -n argocd

argocd login localhost:8080 --username admin --password <上面拿到的密碼> --insecure
```

UI：瀏覽器開 `https://localhost:8080`，帳密同上。

## Step 4：準備 Git repo 當作 Source of Truth

Fork 官方範例 repo：`https://github.com/argoproj/argocd-example-apps`（整包 fork，`guestbook` 資料夾即可直接用）。

## Step 5：建立 ArgoCD Application

```bash
argocd app create guestbook \
  --repo https://github.com/<你的帳號>/argocd-example-apps.git \
  --path guestbook \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace default \
  --sync-policy automated \
  --self-heal
```

- `--dest-server`：目標叢集的 API Server 位址；`https://kubernetes.default.svc` 是每個叢集內建的服務位址，代表「ArgoCD 現在自己所在的這個叢集」，不需要手動建立。
- `--sync-policy automated --self-heal`：自動同步 Git 狀態，並在偵測到叢集實際狀態偏離 Git 宣告狀態時自動修正回去。

## Step 6：驗證同步

```bash
argocd app get guestbook
argocd app sync guestbook
kubectl get pods -n default
```

## Step 7：實測兩個核心行為

1. **GitOps 觸發**：改 fork repo 裡的 `replicas`，commit push，觀察叢集自動同步變化。
2. **Self-Heal**：`kubectl scale deployment guestbook --replicas=5` 手動繞過 Git 改動，觀察 ArgoCD 偵測到漂移後自動改回 Git 宣告的數字。

✅ 實測結果：commit push 後自動 sync 成功；手動 `kubectl scale` 後，ArgoCD 於同一秒內自動將 replica 數從 3 修正回 Git 宣告的 2，驗證 self-heal 生效。

## Step 8：收尾

```bash
argocd app delete guestbook --cascade --yes
kind delete cluster --name devops-demo
```

---

## 一句話總結

「使用 ArgoCD 建立 GitOps 部署流程，透過宣告式 Application 定義與 self-heal 機制，實現 Git commit 自動同步叢集狀態、防止手動配置漂移」
