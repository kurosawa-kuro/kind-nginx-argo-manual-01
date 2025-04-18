# Kind + Nginx + ArgoCD 手動セットアップガイド

このガイドでは、Kindを使用してローカルKubernetesクラスターを構築し、Nginxアプリケーションをデプロイし、ArgoCDでGitOpsを実践する手順を説明します。

## 前提条件

- Docker
- kubectl
- Helm
- Kind

## 1. Kindクラスターの構築

### 1.1 クラスター設定ファイルの作成

```bash
cat <<EOF > kind-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  kubeadmConfigPatches:
  - |
    kind: InitConfiguration
    nodeRegistration:
      kubeletExtraArgs:
        system-reserved: memory=2Gi
        eviction-hard: memory.available<500Mi
  extraPortMappings:
  - containerPort: 80
    hostPort: 80
    protocol: TCP
  - containerPort: 443
    hostPort: 443
    protocol: TCP
  - containerPort: 30000
    hostPort: 30000
    protocol: TCP
EOF
```

### 1.2 クラスターの作成

```bash
kind create cluster --config kind-cluster.yaml
```

### 1.3 クラスターの確認

```bash
kubectl cluster-info
```

## 2. Nginxアプリケーションのデプロイ

### 2.1 Helmチャートの作成

```bash
# チャートの作成
helm create nginx-chart

# チャートディレクトリに移動
cd nginx-chart
```

### 2.2 values.yamlの編集

```bash
# values.yamlを編集
cat <<EOF > values.yaml
replicaCount: 1

image:
  repository: nginx
  tag: "1.25"
  pullPolicy: IfNotPresent

nameOverride: ""
fullnameOverride: ""

service:
  type: NodePort
  port: 80
  nodePort: ""

resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

serviceAccount:
  create: true
  name: nginx-app

# Ingress設定を追加
ingress:
  enabled: false
  className: ""
  annotations: {}
  hosts:
    - host: chart-example.local
      paths:
        - path: /
          pathType: ImplementationSpecific
  tls: []

# Autoscaling設定を追加
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
  targetMemoryUtilizationPercentage: 80
  scaleDownDelaySeconds: 300
EOF
```

### 2.3 チャートのインストール

```bash
# チャートディレクトリにいることを確認
pwd  # /home/ubuntu/dev/kind-nginx-argo-manual-01/nginx-chart であることを確認

# チャートのインストール
helm install nginx-demo ./ -f values.yaml --namespace default
```

### 2.4 デプロイの確認

```bash
# ポッドとサービスの確認
kubectl get pods,svc -l app.kubernetes.io/name=nginx-chart

# アプリケーションへのアクセス
# ノードIPとポートを取得
export NODE_PORT=$(kubectl get --namespace default -o jsonpath="{.spec.ports[0].nodePort}" services nginx-demo-nginx-chart)
export NODE_IP=$(kubectl get nodes --namespace default -o jsonpath="{.items[0].status.addresses[0].address}")
echo http://$NODE_IP:$NODE_PORT

# ブラウザまたはcurlでアクセス
curl http://$NODE_IP:$NODE_PORT
```

## 3. ArgoCDのセットアップ

### 3.1 ArgoCDのインストール

```bash
# ArgoCD名前空間の作成
kubectl create namespace argocd

# ArgoCDのインストール
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# ArgoCDサーバーの準備完了を待つ
kubectl wait --for=condition=ready pod -l app.kubernetes.io/name=argocd-server -n argocd --timeout=300s
```

### 3.2 ArgoCDへのアクセス

```bash
# ArgoCDサーバーのポートフォワード
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

ブラウザで https://localhost:8080 にアクセスし、初期パスワードを取得：

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

#### 3.2.1 ポートフォワードのトラブルシューティング

ポートフォワードに問題がある場合は、以下の方法を試してください：

```bash
# 別のポートを使用
kubectl port-forward svc/argocd-server -n argocd 8081:443
```

その後、ブラウザで https://localhost:8081 にアクセスします。

セキュリティ警告が表示された場合は、「詳細情報」→「安全ではないサイトにアクセスする」を選択します。

curlコマンドで確認する場合：

```bash
# 証明書の検証をスキップしてアクセス
curl -k https://localhost:8080
```

### 3.3 Application YAMLの作成

```bash
# Application YAMLファイルの作成
cat <<EOF > argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-demo
  namespace: argocd
spec:
  project: default

  source:
    repoURL: 'https://github.com/kurosawa-kuro/kind-nginx-argo-manual-01.git'
    targetRevision: development
    path: nginx-chart
    helm:
      releaseName: nginx-demo
      valueFiles:
        - values.yaml

  destination:
    server: 'https://kubernetes.default.svc'
    namespace: default

  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

### 3.4 アプリケーションの登録

```bash
# Applicationリソースの適用
kubectl apply -f argocd-application.yaml
```

## 4. トラブルシューティング

### 4.1 イメージプルエラー

```bash
# イメージプルエラーの確認
kubectl describe pod <pod-name>
```

### 4.2 ArgoCDエラー

```bash
# ArgoCDアプリケーションの状態確認
kubectl get application -n argocd

# ArgoCDアプリケーションの詳細確認
kubectl describe application nginx-demo -n argocd
```

#### 4.2.1 同期エラーの解決

「app path does not exist」エラーが発生した場合：

1. リポジトリの構造を確認する：
   ```bash
   ls -la
   ```

2. Application YAMLのパス設定を確認する：
   - `path` フィールドが正しいディレクトリを指しているか確認
   - 古いバージョンのArgoCDでは `helm.chart` フィールドがサポートされていない場合があります

3. リポジトリのブランチ名が正しいか確認する：
   - `targetRevision` フィールドが正しいブランチ名を指しているか確認

4. リポジトリへのアクセス権限を確認する：
   ```bash
   # GitHubリポジトリをArgoCDに追加
   argocd repo add https://github.com/kurosawa-kuro/kind-nginx-argo-manual-01.git
   ```

5. ArgoCDのバージョンを確認する：
   ```bash
   kubectl get deployment argocd-server -n argocd -o jsonpath='{.spec.template.spec.containers[0].image}'
   ```

### 4.3 ポートフォワードの問題

ポートフォワードに問題がある場合は、以下の手順を試してください：

1. 別のポートを使用する
2. ポートフォワードを再起動する
3. ArgoCDサーバーを再起動する：
   ```bash
   kubectl rollout restart deployment argocd-server -n argocd
   ```

## 5. クリーンアップ

### 5.1 リソースの削除

```bash
# Helmリリースの削除
helm uninstall nginx-demo

# ArgoCDアプリケーションの削除
kubectl delete -f argocd-application.yaml

# ArgoCDの削除
kubectl delete -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl delete namespace argocd

# Kindクラスターの削除
kind delete cluster
```

## 6. 参考リソース

- [Kind公式ドキュメント](https://kind.sigs.k8s.io/docs/)
- [Helm公式ドキュメント](https://helm.sh/docs/)
- [ArgoCD公式ドキュメント](https://argo-cd.readthedocs.io/)
- [Nginx公式ドキュメント](https://nginx.org/en/docs/)

