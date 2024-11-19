# 1. 環境構築
## 1.1 kubectlのインストール
```sh
brew install kubectl
```

バージョンの確認（オプション）
```sh
kubectl version --client
```

## 1.2 Rancher DesktopでKubernetesを有効化

1. Preferences -> KubernetesでKubernetesの設定ページを開く。
2. "Enable Kubernetes"の横のチェックボックスにチェックを入れる。

## 1.3 サンプルコードのダウンロード
```sh
git clone 
```

# 2. Pod
## 2.1 Podの作成
以下のコマンドを実行してください
マニフェストを元にPodを作成します。
```sh
kubectl apply -f src/sample-pod.yaml
```

起動したPodを確認してみましょう

```sh
kubectl get po
```

詳細情報を表示したい場合は「-o wide（--output wide）」オプションを使用して出力します。

```sh
kubectl get po -o wide
```
## 2.2 2つのコンテナを内包するPodの作成

```sh
kubectl apply -f src/sample-2pod.yaml
```

Podを確認すると、READYのコンテナ数が「2/2」と表示されます。

```sh
kubectl get po
```

# 3. ReplicaSet
## 3.1 ReplicaSetの作成

```sh
kubectl apply -f src/sample-rs.yaml
```

ReplicaSetを見てみると、Podが3つ起動していることを確認できます。
```sh
kubectl get rs -o wide
```

実際にラベルを指定してPodを確認してみても、3つ起動していることが確認できます。

```sh
kubectl get po -l app=sample-app -o wide
```
## 3.2 Podの停止とセルフヒーリング
ReplicaSetでは、ノードやPodに障害が生じた場合でも、Pod数が指定された数を満たすように別のノードでPodを起動してくれるため、障害時の影響を低減できます。

セルフヒーリングの挙動を確認するために、Podを1台停止してみます。

```sh
# Pod名は実際に起動しているものを指定
kubectl delete po sample-rs-ht4tr
```

Podを確認すると、すぐにReplicaSetによって別名のPodが新規に作成されていることがわかります。
```sh
kubectl get po -o wide
```

ReplicaSetのPodの増減履歴は以下のコマンドで確認できます。

```sh
kubectl describe rs sample-rs
```

ReplicaSetはK8sがPodの監視を行うことでPodの数の調整をしていますが、監視を行う際は特定のラベルがつけられたPodの数をカウントする形で実現しています。

# 4. Deployment
# 4.1 Deploymentの作成

Deploymentを起動
```sh
kubectl apply -f src/sample-deployment.yaml
```

Deploymentの確認
```sh
kubectl get deploy
```

NGINXのバージョンを`1.26`->`1.27`にアップデートしてみましょう

```sh
kubectl set image deploy sample-deployment sample-deployment nginx-container=nginx:1.27
```

アップデートの状況は以下のコマンドで確認できます。

```sh
kubectl rollout status deploy sample-deployment
```

Deploymentを確認
```sh
kubectl get deploy
```

更新後は新たにReplicaSetが作成されており、そこに紐づく形でPodが再作成されます。
内部的にはローリングアップデートが行われているので、基本的にサービスへの影響はありません。

```sh
kubectl get rs
```

## 4.2 ロールバック
変更履歴は以下のコマンドで確認できます。
```sh
kubectl rollout history deploy sample-deployment
```

各リビジョンの詳細は以下のコマンドで確認できます。
```sh
kubectl rollout history deploy sample-deployment --revision 1
```

ロールバックする際は以下のコマンドで行います。（リビジョンを指定しない場合は一個前のリビジョンに戻ります。）

```sh
kubectl rollout undo deploy sample-deployment --to-revision 1
```

ロールバックした際には、指定したリビジョンのReplicaSetのPodが起動されます。

```sh
kubectl get rs
```

実際、この「kubectl rollout」を利用するケースは多くありません。「kubectl apply -f」のほうが、宣言的な状態管理の考え方に適合しており、相性がいいからです。


# 5. クリーンアップ
以下のコマンドを実行すると、作成したK8sリソースが全て削除されます。

```sh
kubectl delete -f src/
```