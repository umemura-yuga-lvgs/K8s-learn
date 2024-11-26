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

1. Preferences -> KubernetesでKubernetesの設定ページを開いてくださ。
2. "Enable Kubernetes"の横のチェックボックスにチェックを入れてください。

# 2. Pod
## 2.1 Podの作成
以下のコマンドを実行してください
マニフェストを元にPodを作成します。
```sh
kubectl apply -f src/sample-pod.yaml
```

起動したPodを確認してみましょう。

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

立ち上げたPodはもう使わないので削除しておきましょう。
```sh
kubectl delete -f src/
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

ReplicaSetはもう使わないので削除しておきましょう。
```sh
kubectl delete -f src/sample-rs.yaml
```

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

NGINXのバージョンを`1.26`->`1.27`にアップデートしてみましょう。

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


# 5. Service
## 5.1 Pod宛トラフィックのロードバランシング
Podのラベルを確認しましょう。
```sh
❯ kubectl get po sample-deployment-d68df7d7-5tz4h -o jsonpath='{.metadata.labels}'

{"app":"sample-app","pod-template-hash":"d68df7d7"}
```

ClusterIP Serviceを作成してください。
```sh
kubectl apply -f src/sample-clusterip.yaml
```

作成されたロードバランサのトラフィックの転送先を確認します。
（＝「app: sample-app」のラベルを持つPodのIPアドレスを確認します。）
```sh
❯ kubectl get po -l app=sample-app \
 -o custom-columns="NAME:{metadata.name}, IP:{status.podIP}"

NAME                                IP
sample-deployment-d68df7d7-5tz4h   10.42.0.117
sample-deployment-d68df7d7-tw5x2   10.42.0.118
sample-deployment-d68df7d7-x79fz   10.42.0.119
```
※「-l」はlabel。K8sオブジェクトを指定されたラベルに基づいてフィルタリングします。

Serviceの情報を確認しましょう。
```sh
❯ kubectl get svc sample-clusterip

NAME               TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
sample-clusterip   ClusterIP   10.43.90.25   <none>        8080/TCP   18m
```

Serviceの詳細情報を確認
```sh
❯ kubectl describe svc sample-clusterip

Name:                     sample-clusterip
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=sample-app
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.43.90.25
IPs:                      10.43.90.25
Port:                     http-port  8080/TCP
TargetPort:               80/TCP
Endpoints:                10.42.0.119:80,10.42.0.118:80,10.42.0.117:80
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>
```
Endpoint欄にトラフィックの転送先のIPアドレスとポートを確認することができます。
また、ロードバランシングするためのエンドポイントの仮想IPはCLUSTER-IP欄やIP欄より確認することができます。
先ほど確認した「app: sample-app」のラベルを持つPodとIPアドレスが同一であることから、トラフィックの転送がSelectorの条件に従って選択されていることが確認できます。

ServiceにHTTPリクエストを送って、作成したロードバランサが各Podにトラフィックをロードバランシングして転送しているか確認しましょう。
```sh
# ServiceのIPアドレスは各自で設定
# 1回目
❯ kubectl run --image=amsy810/tools:v2.0 --restart=Never --rm -i testpod --command -- curl -s http://10.43.90.25:8080

Host=10.43.90.25  Path=/  From=sample-deployment-d68df7d7-x79fz  ClientIP=10.42.0.125  XFF=
pod "testpod" deleted

# 2回目
❯ kubectl run --image=amsy810/tools:v2.0 --restart=Never --rm -i testpod --command -- curl -s http://10.43.90.25:8080

Host=10.43.90.25  Path=/  From=sample-deployment-d68df7d7-tw5x2  ClientIP=10.42.0.127  XFF=
pod "testpod" deleted

# 3回目
❯ kubectl run --image=amsy810/tools:v2.0 --restart=Never --rm -i testpod --command -- curl -s http://10.43.90.25:8080

Host=10.43.90.25  Path=/  From=sample-deployment-d68df7d7-5tz4h  ClientIP=10.42.0.130  XFF=
pod "testpod" deleted
```

FromにPod名が記載されている。Pod名は[リソース名]-[ReplicaSetのHash値]-[Podの識別子]の構造をとっており、識別子の部分を見ると、異なるPodにリクエストが分散されていることがわかります。

名前付けされたポートを持つPodの例

Podの作成してください。
```sh
kubectl apply -f src/sample-named-port-pods.yaml
```
次に、Serviceの作成してください。
```sh
kubectl apply -f src/sample-named-port-service.yaml 
```

Serviceの宛先エンドポイントを確認しましょう。
```sh
❯ kubectl describe svc sample-named-port-service  

Name:                     sample-named-port-service
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=sample-app
Type:                     ClusterIP
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.43.219.113
IPs:                      10.43.219.113
Port:                     http-port  8080/TCP
TargetPort:               http/TCP
Endpoints:                10.42.0.132:80,10.42.0.131:81,10.42.0.119 + 2 more...
Session Affinity:         None
Internal Traffic Policy:  Cluster
Events:                   <none>
```
PodのIPアドレスを確認してください。
```sh
kubectl get po -o wide
```

## 5.2 クラスタ内DNSとサービスディスカバリ
### 環境変数を利用したサービスディスカバリ
Pod内からは、環境変数で同じNamespace内のサービスが確認できるようになっています。
```sh
❯ kubectl exec -it sample-deployment-d68df7d7-6vfwf -- env | grep -i sample_clusterip

SAMPLE_CLUSTERIP_SERVICE_PORT=8080
SAMPLE_CLUSTERIP_SERVICE_HOST=10.43.90.25
SAMPLE_CLUSTERIP_PORT_8080_TCP=tcp://10.43.90.25:8080
SAMPLE_CLUSTERIP_PORT_8080_TCP_PORT=8080
SAMPLE_CLUSTERIP_PORT_8080_TCP_ADDR=10.43.90.25
SAMPLE_CLUSTERIP_SERVICE_PORT_HTTP_PORT=8080
SAMPLE_CLUSTERIP_PORT=tcp://10.43.90.25:8080
SAMPLE_CLUSTERIP_PORT_8080_TCP_PROTO=tcp
```

### DNS Aレコードを利用したサービスディスカバリ
一時的にPodを立ち上げて、コンテナ内からsample-clusterip:8080宛にHTTPリクエストを送ってみましょう。
```sh
kubectl run --image=amsy810/tools:v2.0 --restart=Never --rm -i testpod \
    --command -- curl -s http://sample-clusterip:8080
```
複数回実行すると、ほぼ均等に3つのPodが表示されるはずです。
これは、sample-clusteripの名前解決が行われ、10.43.90.25宛にリクエストが送られるようになっているからです。
実際に登録されているFQDNは[Service名].[Namespace名].svc.cluster.localとなっています。

# 6. クリーンアップ
以下のコマンドを実行すると、作成したK8sリソースが全て削除されます。

```sh
kubectl delete -f src/
```

