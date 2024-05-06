# Kubernetesハンズオン

## 1. 事前準備

まずは、CLIツールが正常に動作しているか確認します。
以下のコマンドを入力してください。

```Bash
kubectl get nodes
```

Nodeの一覧が出力されるはずです。

```Log
NAME                 STATUS   ROLES           AGE   VERSION
kind-control-plane   Ready    control-plane   30d   v1.27.3
kind-worker          Ready    <none>          30d   v1.27.3
kind-worker2         Ready    <none>          30d   v1.27.3
```

Nodeが表示されない場合は、kubeconfigが設定されていない可能性があります。

以下のコマンドでkubeconfigの設定を確認します。

```Bash
kubectl config get-contexts
```

```
CURRENT   NAME        CLUSTER     AUTHINFO    NAMESPACE
*         kind-kind   kind-kind   kind-kind   
```


Kubernetesは、kubectlというCLIツールを提供しています。
kubectlは、ネットワークリーチャビリティのあるController NodeにAPIのリクエストを送ることで
リモートでの操作を可能にするものです。
正しくセットアップされていないとAPIリクエストを送ることができずに
Kubernetesの操作ができなくなってしまいます。

以下のコマンドでkubectlコマンドのバージョンの確認ができます。
```Bash
kubectl version --client
```

```
Client Version: v1.28.1-eks-43840fb
Kustomize Version: v5.0.4-0.20230601165947-6ce0bf390ce3
```

続いて、kubectlのコマンド補完の設定を行います。

> 現在のbashシェルにコマンド補完を設定するには、最初にbash-completionパッケージをインストールする必要があります。

```Bash
source <(kubectl completion bash)
```

> bashシェルでのコマンド補完を永続化するために.bashrcに追記します。

```Bash
echo "source <(kubectl completion bash)" >> ~/.bashrc
```

> 下記までの設定を行うと、コマンド補完が行え"k"のみでkubectlとみなされます。

```
alias k=kubectl
complete -F __start_kubectl k
```

## 2. アプリケーションデプロイ



### 2.2 Deployment manifestファイル作成

続いて、manifestファイルを作成します。
manifestファイルはyaml形式もしくはjson形式がサポートされています。
今回はyaml形式のmanifestを用意していますので、そのManifestを使ってPodをデプロイします。
以下のコマンドを入力してください。

```Bash
cd manifest
kubectl apply -f test-deployment.yaml
```

以下のコマンドでPodの確認ができます。

```Bash
kubectl get pods
```

### 2.4. ポートフォワードと通信確認

続いて、作成したPodにアクセスします。
今回はポートフォワードを使いインターネット上のpodにアクセスしていきます。

```Bash
kubectl port-forward <Pod名>  8888:80 -n <namespace名>
```

以下のように出力されたら操作が受け付けられなくなりますが、ctrl＋Cを押さずにそのままでいてください。

```Log
Forwarding from 127.0.0.1:8888 -> 80
Forwarding from [::1]:8888 -> 80
```

この時点でアクセスが可能になっているはずなので、新しくターミナルを開き、以下のコマンドでアクセスしてみましょう。

```Bash
curl http://localhost:8888
```


成功すると、nginxのテストページが表示されるはずです。

### 2.5 Pod削除

続いて、Podを削除してみます。
以下のコマンドを入力してください。

```Bash
kubectl delete pod <pod名>
```

Pod名、及び削除されたかどうかは以下で調べることができます。

```Bash
kubectl get pod
```

上記の対応では、対象PodのRESTARTSのみがリセットされPodが削除できていないことがわかります。
Kubernetesはあるべき状態をManifestとして定義します。
このケースではPodを削除したことをトリガーにあるべき状態、つまり対象のPodが1つ存在する状態に戻そうと、Podの上位リソースであるDeploymentが働きかけたことが原因です。
このようなケースでPodを完全に削除したい場合はDeploymentごと削除する必要があります。

まず、以下のコマンドでDeploymentの状態を確認します。

```Bash
kubectl get deployments
```

続いて、以下のコマンドで対象PodのDeploymentを削除します。

```Bash
kubectl delete deployment test-deployment
```

以下のコマンドでDeployment及びPodが削除されたことを確認します。

```Bash
kubectl get deployments
kubectl get pod
```

### 2.6 Tips

先ほどまではDeployment Manifestを作成しPodを作成しましたが、簡単なテストを実行したい場合などに手軽にPodを起動したい場合などがあると思います。
以下のようなコマンドを実行すると、ワンライナーでPodの起動までが行えます。

```Bash

kubectl run <Pod名> --image=<image名> 

```

また、Manifestを1から書くことが難しい場合は、以下のようにdry-runとyaml出力を組み合わせてファイルに書き込むことでサンプルファイルを作成することができます。


```Bash

kubectl run <pod名> --image=<image名> --dry-run=client -o yaml > <ファイル名>
```

## 3. オリジナルコンテナデプロイ (Secretの活用)

### 3.1. Deployment Manifestの修正

続いて、前回DockerHubにPushしたオリジナルのImageを使い
Podを作成していきます。

まずはmanifestを編集します。

```Bash
vi hello-world.yaml
```


```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  labels:
    app: hello-world
  name: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    spec:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - image: <DockerHubのユーザ名>/<リポジトリ名>:<タグ>
        name: hello-world
        ports:
        - containerPort: 80
```

以下を、ご自身のDockerHubのユーザ名、リポジトリ名、タグに変更してください。

```
      - image: <DockerHubのユーザ名>/<リポジトリ名>:<タグ名>
```


### 3.2. Deploymentの適用

デプロイを試みます。

```Bash
kubectl apply -f hello-world.yaml
```

以下のコマンドで確認すると、Podの作成が失敗していることがわかります。

```Bash
kubectl get pod
```

```Log
NAME                                  READY   STATUS         RESTARTS   AGE
hands-on-nginx-8f5b8f48c-xb9kx     0/1     ErrImagePull       0                 14s
```

このようなエラーが起こった場合は、原因の解析にPodの詳細出力が役立つ場合があります。
以下のコマンドを入力します。

```Bash
kubectl describe pod <pod名> -n <namespace名>
```

```Log
Name:             hands-on-nginx-8f5b8f48c-xb9kx
Namespace:        test1
Priority:         0
Service Account:  default
Node:             ip-192-168-34-191.ap-northeast-1.compute.internal/192.168.34.191
Start Time:       Thu, 07 Dec 2023 05:08:14 +0000
Labels:           app=hands-on-nginx
                  pod-template-hash=8f5b8f48c
Annotations:      <none>
Status:           Pending
IP:               192.168.49.0
IPs:
  IP:           192.168.49.0
Controlled By:  ReplicaSet/hands-on-nginx-8f5b8f48c
Containers:
  hands-on-nginx:
    Container ID:   
    Image:          ryuichitakei/hands-on:hands-on-nginx
    Image ID:       
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-4wdf2 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             False 
  ContainersReady   False 
  PodScheduled      True 
Volumes:
  kube-api-access-4wdf2:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
  Normal   Scheduled  2m23s                default-scheduler  Successfully assigned test1/hands-on-nginx-8f5b8f48c-xb9kx to ip-192-168-34-191.ap-northeast-1.compute.internal
  Normal   Pulling    52s (x4 over 2m22s)  kubelet            Pulling image "ryuichitakei/hands-on:hands-on-nginx"
  Warning  Failed     51s (x4 over 2m21s)  kubelet            Failed to pull image "ryuichitakei/hands-on:hands-on-nginx": rpc error: code = Unknown desc = failed to pull and unpack image "docker.io/ryuichitakei/hands-on:hands-on-nginx": failed to resolve reference "docker.io/ryuichitakei/hands-on:hands-on-nginx": pull access denied, repository does not exist or may require authorization: server message: insufficient_scope: authorization failed
  Warning  Failed     51s (x4 over 2m21s)  kubelet            Error: ErrImagePull
  Warning  Failed     37s (x6 over 2m20s)  kubelet            Error: ImagePullBackOff
  Normal   BackOff    25s (x7 over 2m20s)  kubelet            Back-off pulling image "ryuichitakei/hands-on:hands-on-nginx"
```
### 3.3. Secretの追加

上記のログから、最終的にImageのPullに失敗しErrorになっているのがわかります。
この原因は、格納されているリポジトリがプライベート設定であることです。
外部公開されていないイメージをPullしたい場合は、Secretと呼ばれる認証情報を格納するためのリソース指定が必要です。

現状、Default NameSpaceにはSecretリソースが存在しないことを確認します。

```Bash
kubectl get secret
```

```
No resources found in  default namespace.
```

今回はそれぞれのnamespaceにPodをデプロイする想定なので、namespace毎に認証情報が必要です。namespaceから外のリソースは互いに干渉しないため、それぞれのnamespace内でのみ認証情報の共有が有効となります。
今回のケースでは、ImageをPullする際にこのSecretを使うようManifestに指示を書くことでプライベートリポジトリからもImageのPullが可能になります。


以下のコマンドでSecretを作成します。

```Bash
kubectl create secret docker-registry dockerhub-secret --docker-username=<DockerHubのユーザ名> --docker-password=<Dockerhubのパスワード>
```

### 3.4. SecretをDeploymentで利用

先ほどのManifestに、Secretに関する指示を追記します。

```Bash
vi hello-world.yaml
``` 

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  labels:
    app: hands-on-nginx
  name: hands-on-nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hands-on-nginx
  template:
    spec:
    metadata:
      labels:
        app: hands-on-nginx
    spec:
      containers:
      - image: ryuichitakei/hands-on:hands-on-nginx # タグ名を自身のImageのものに変更
        name: hands-on-nginx
        ports:
        - containerPort: 80
      imagePullSecrets: # 追記
      - name: dockerhub-secret # 追記
```

先ほど作成したPodの設定を更新します。

```Bash
kubectl apply -f hands-on-nginx.yaml -n <namespace名>
```

ImageのPullが成功し、Podが起動しているはずです。

```Bash
kubectl get pod -n <namespace名>
```

## 4. ReplicaSetの仕組み

ReplicaSetは稼働しているPod数を明示的に指定し、それを維持するためのリソースです。
2.アプリケーションデプロイの章でも体感していただきましたが、指定したReplica数を維持するために
自動的にPodの作成、削除が行われます。
現在、みなさんのManifestにはReplica数1が設定されています。
そのため、起動しているPodも1つになっているはずです。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  labels:
    app: hands-on-nginx
  name: hands-on-nginx
spec:
  replicas: 1 # ここが1に設定されている
  selector:
    matchLabels:
      app: hands-on-nginx
  template:
    spec:
    metadata:
      labels:
        app: hands-on-nginx
    spec:
      containers:
      - image: ryuichitakei/hands-on:hands-on-nginx 
        name: hands-on-nginx
        ports:
        - containerPort: 80
      imagePullSecrets: # 追記
      - name: <secret名> # 追記
```

では以下のようにManifestを修正し、再度Manifestを登録しなおしてみます。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "1"
  labels:
    app: hands-on-nginx
  name: hands-on-nginx
spec:
  replicas: 2 # 修正
  selector:
    matchLabels:
      app: hands-on-nginx
  template:
    spec:
    metadata:
      labels:
        app: hands-on-nginx
    spec:
      containers:
      - image: ryuichitakei/hands-on:hands-on-nginx 
        name: hands-on-nginx
        ports:
        - containerPort: 80
      imagePullSecrets:
      - name: <secret名>
```

```Bash
kubectl apply -f hands-on-nginx.yaml -n <namespace名>
```

Podが2つに増えているか確認します。

```Bash
kubectl get pod -n <namespace名>
```

> 出力例

```Log
NAME                              READY   STATUS    RESTARTS      AGE
hands-on-nginx-65f87b65fb-mx7n8   1/1     Running   0             9s
hands-on-nginx-65f87b65fb-wlvvw   1/1     Running   0             8s
```

## 5. Podの外部公開

続いて、Podの外部公開の方法を紹介します。
前回のセッションではPortForwardを使ってPodのアクセスを行いましたが


### 5.1. Service Manifestの作成

では、ManifestファイルからServiceを作成していきます。

```Bash
vi hello-world-service.yaml
```

以下のサンプルコードを参考にyamlファイルを作成します。

```yaml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: hands-on-nginx-service
  name: hands-on-nginx-service
  namespace: <namespace名> 
spec:
  ports:
  - nodePort: 32001 # Port番号を設定(重複しないように注意)
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: hands-on-nginx # 公開したいPodのラベル名
  sessionAffinity: None
  type: NodePort # ServiceTypeをNodePortにする
```

設定したラベルについては以下で確認が可能です。

```Bash
kubectl get pod -n <namespace名> --show-labels
```

### 5.2. Service Manifestの適用

作成したManifestを使ってServiceを作成します。

```Bash
kubectl apply -f hands-on-nginx-service.yaml -n <namespace名>
```

作成したServiceは以下で確認が可能です。

```Bash
kubectl get service -n <namespace名>
```

> 出力例

```Log
NAME                     TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
hands-on-nginx-service   NodePort   10.100.108.144   <none>        80:32001/TCP   44m
```

### 5.3. Service 通信確認

続いて、NodeのグローバルIP＋設定したNodePortのポート番号でアクセス確認を行います。

現在、２つのNodeに振り分けられてるIPをお伝えします。

どちらのIPで接続しても、ポート番号が正しければ適切なPodに通信を割り振ってくれるので
自身が作成したHTMLコンテンツが表示されるはずです。

```Bash
curl http://＊＊＊＊:32001
<!DOCTYPE html>
<html lang="ja">
  <style>
    body {
      margin: 0;
    }

    .center-me {
      display: flex;
      justify-content: center;
      align-items: center;
      /*font-family: 'Saira Condensed', sans-serif;*/
  font-family: 'Lobster', cursive;
      font-size: 100px;
      height: 100vh;
    }
  </style>  
<head>
    <meta charset="utf-8">
      <title>Test</title>
  </head>
  <body>
    <div class="center-me" >
    <p>
      <h1>Hello World!!🙂</h1>
    </p>
  </div>
  </body>
</html>
```

## 6. Ingressとアプリケーションの更新

このセクションでは、Podを外部公開するためのリソースであるIngressと、Kubernetesが持つPodの更新方法について紹介します。

### 6.1. Rolling Update

Kubernetesには、Podを別のイメージに変更したりバージョンを更新する際に、サービスに影響が出ないよう段階的に更新の動作を行うRolling Updateという機能があります。

それでは、実際に更新動作を確認していきましょう。
更新するときの処理はstrategy で指定します。デフォルトはRollingUpdateです。
ローリングアップデートの処理をコントロールするためにmaxUnavailableとmaxSurgeを指定することができます。

- minReadySeconds
  新しく作成されたPodが利用可能となるために、最低どれくらいの秒数コンテナーがクラッシュすることなく稼働し続ければよいか
- maxSurge
  理想状態のPod数を超えて作成できる最大のPod数(割合でも設定可)
- maxUnavailable
  更新処理において利用不可となる最大のPod数(割合でも設定可)


今回は4つのReplica数に対して25%、つまり1つずつ更新がかかるような設定をしています。
また、Podは作成後直ぐに利用可能になるので、動作イメージをつかみやすくするためにminReadySecondsは10秒に設定しています。



動作確認用のManifestを適用しましょう。

```
kubectl apply -f rollout.yaml
```

続いて、ブラウザで以下にアクセスを行います。

```
http://rollout.example.com
```

Pod更新前の状態では、`This app is Blue`の画面が表示がされていると思います。


続いて、先ほどデプロイしたDeplpymentに対して、イメージの更新を行います。


その際、Rolling Updateの機能が働き、25%のPod数(1個)ずつ追加されていく様子が確認できます。

```Bash
# 適用
kubectl set image deployment/rolling rolling-app=ryuichitakei/green-app:1.0

# 確認
kubectl rollout status deployment 
kubectl rollout history deployment 
kubectl get pod
kubectl get deployment

```

更新後、ブラウザで再度以下にアクセスを行うと`This app is Green`の表示に更新されていることが確認できます。

```
http://rollout.example.com
```

尚、ロールバックを行う場合は以下のコマンドで実行可能です。

```Bash
kubectl rollout undo deployment rolling
```

動作確認実施後、リソースの削除を行います。

```Bash
kubectl delete deployment rolling
kubectl delete service rolling
lubectl delete ingress rolling
```

### 6.2 Blue-Green Deployment


古い環境と新しい環境を混在させ、ルーティングなどによってトラフィックを制御し、ダウンタイム無しで環境を切り替えます。
今回はIngressのHost名によって、新旧どちらのアプリケーションにもアクセスできるような環境を用意しています。


まずは、対象のManifestを適用します。

```
kubectl apply -f blue-green.yaml
```

続いて、Pod,Service,Ingressがそれぞれデプロイされているか確認を行います。


```
kubectl get pod,service,ingress
```

それぞれのリソースが正常に動作していることが確認できたら、ブラウザから以下のようにアクセスができるはずです。


```
http://blue.example.com → Blue App
http://green.example.com → Green App
```

動作確認実施後、リソースの削除を行います。

```Bash
kubectl delete pod blue
kubectl delete pod green
kubectl delete service blue-service
kubectl delete service green-service
lubectl delete ingress blue-green
```

## 7. データの永続化 (PVとPVC)

ここまでのハンズオンで、コンテナの特性がある程度見えてきたかと思います。
おさらいすると、以下のような特性があります。

- カーネルを持たず、プロセスのような振る舞いをする
- 起動・停止がVMに比べて高速で行える
- データを自身で持たずエフェメラルな存在として扱う。

上記の特性から、コンテナのデータをどう扱う(システムとしてどう設計する)かは非常に重要な観点です。
このセクションでは、Nodeが持つストレージにPodをマウントさせ、データの永続化が確認できるまでのテストを行います。


PV(Persistent Volume)は外部ストレージとの接続を司るリソースです。
以下がPVを作成するためのサンプルコードです。


```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: handson-pv 
spec:
  capacity:
    storage: 1Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:
    path: /tmp
```

### 7.2. PVCの作成

PVC(Persistent Volume Claim)は、PodのVolumeに関する要求事項を定義するためのリソースです。

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: handson-pvc
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: standard
  resources:
    requests:
      storage: 1Gi
```

### 7.3. Podの作成

データの永続化を行うPodは、volumes属性に使いたいPVCの名前を書くことで作成できます。

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: volume-pod
spec:
  containers:
  - name: volume-app
    image: busybox
    command: ["/bin/sh"]
    args: ["-c","while true; do echo $(date -u) >> /data/out1.txt; sleep 5; done"]
    volumeMounts:
    - name: persistent-storage
      mountPath: /data
  volumes:
  - name: persistent-storage
    persistentVolumeClaim:
      claimName: handson-pvc
```

### 7.4. リソースの作成と動作確認

以下のコマンドで各リソースの作成を行います。


```Bash
kubectl apply -f handson-volume.yaml
```

#### 7.4.4 動作確認

以下のコマンドで各リソースの確認を行います。

```
kubectl get pv,pvc,pod
kubectl describe pv handson-pv
kubectl describe pvc handson-pvc
```

今回のシナリオでは、5秒ごとにdateコマンドで日付をマウント先のファイル`/data/out1.txt`に書き込むPodを作成しています。
以下のコマンドで動作確認が行えます。

```
kubectl exec -ti volume-pod -- tail /data/out1.txt
```

動作確認後、リソースの削除を行います。


```
kubectl delete pod volume-pod
kubectl delete pvc handson-pvc
kubectl delete pv handson-pv
```

### 8. Init Container


PodはKubernetesにおける最小の単位ですが、その実態は複数(単独の場合もある)のコンテナで実行するリソースです。
例えば、Serice Meshを実現するためにネットワークプロキシとなるコンテナAとサービスアプリケーションとなるコンテナBを1つのPodとして稼働させることで、ネットワーク周りの処理をコンテナBに任せてコンテナはサービスの提供に全てのリソースを割くといったことができます。
また、init containerと呼ばれる一時的な用途のコンテナを作成することも可能です。

今回はinit containerの動作を確認してみましょう。
このシナリオでは、起動時に作成されるコンテナ(Init Container)が'CNDT2024!!'というメッセージを出力するコンテンツを作成しマウント先のボリュームに保存します。
その後、nginxが起動しInit Containerが作成したコンテンツを参照することで、nginxにアクセスした際に上記メッセージが返却されます。

まずは以下のManifestをapplyします。

```
kubectl apply -f handson-init.yaml
```

続いて、動作確認のためPodのIPを確認します。

```
kubectl get pod -o wide | grep init
```

最後に一時的な確認Podを使ってcurlでのアクセス確認をしてみましょう。


```
kubectl run tmp --restart=Never --rm -i --image=nginx:alpine -- curl <PodのIP>
```

以下のように`CNDS2024!!`のメッセージが確認できます。


```
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
CNDS2024!!
100    11  100    11    0     0   8094      0 --:--:-- --:--:-- --:--:-- 11000
pod "tmp" deleted
```

動作確認後、リソースを削除します。

```
kubectl delete deployment handson-init-container
```


### 9. ServiceAccountとUser Account

KubernetesにはPodにマッピングされるServiceAccountと、管理者もしくは開発者のkubectlの適用範囲を司るUser Accountの概念が存在します。
まずは、`handson-sa`という名前のServiceAccountを作成してPodが実行することができるコマンドの範囲が制御できることを確認してみましょう。


#### Service Accountの作成と動作確認

> ServiceAccount作成

```Bash
kubectl get serviceaccounts
kubectl create serviceaccount handson-sa
kubectl get serviceaccounts
```

> Role作成

```Bash
kubectl get role
kubectl create role handson-role --resource=pods --verb=get,watch,list
kubectl get role
```

> RoleBinding作成

```Bash
kubectl get rolebinding
kubectl create rolebinding handson-rolebinding --role=handson-role --serviceaccount=default:handson-sa
kubectl get rolebinding
```

> Podデプロイ

```Bash

kubectl apply -f kubectl-pod.yaml

```

> ログからコマンドを実行していることを確認

```Bash
kubectl logs kubectl-pod
```

> 一度Podを削除し、コマンドを変更して再デプロイ

```Bash

kubectl delete pod kubectl-pod

```

vimなどのエディタを使って、Pod名で実行するコマンドを変更します。

```Yaml
apiVersion: v1
kind: Pod
metadata:
  name: kubectl-pod
spec:
  containers:
  - image: bitnami/kubectl
    name: kubectl
    command:
    - sh
    - -c
    - |
      while true
      do
        kubectl get deployment # 変更箇所
        sleep 30
      done
  serviceAccountName: <sa名>
```

```Bash

kubectl apply -f kubectl-pod.yaml

```

> ログからコマンドが弾かれていることを確認

```Bash
kubectl logs kubectl-pod
```

>確認できたらPodを削除
>
>
```Bash

kubectl delete pod kubectl-pod

```

#### User Accountの作成と動作確認

続いてUserの作成を行います。
User Accountは厳密にはK8sのリソースとして定義されておらず、getでも確認ができません。
しかしながら、APIとマッピングしてkubectlの適用範囲を明示的に制御することができます。
まずはUser Accountを作成するために秘密鍵とCSRを作成し、それを元にUser Accountを作成します。
続いて、User Accountに紐づくRoleとRole Bindingを作成し、動作確認を行います。

> 秘密鍵とCSRの作成

```
openssl genrsa -out handson.pem 2048
openssl req -new -key handson.pem -out handson.csr -subj "/CN=<任意のCN>"
```

> csrをbase64にエンコード

```
cat handson.csr | base64 | tr -d '\n'
```

> UserAccount作成

```Yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: <UserAccount名>
spec:
  signerName: kubernetes.io/kube-apiserver-client
  request: <base64でエンコードしたテキストを貼り付ける>
  usages:
  - client auth
```

```Bash
kubectl apply -f handson-csr
```

> CSRをApprove


```Bash
kubectl get csr
kubectl certificate approve handson-user
kubectl get csr
```

> Role作成

```Bash
kubectl get role
kubectl create role handson-user-role --resource=pods --verb=create,list,get,update,delete
kubectl get role
```

> RoleBinding作成

```Bash
kubectl get rolebinding
kubectl create rolebinding handson-user-rolebinding --role=handson-user-role --user=handson-user
kubectl get rolebinding
```
> 動作確認

リソース名などを変えてみて、yes or noの出力を確かめます。

まずはPodの更新が可能かどうかを確かめます。
Roleを作成した際に、PodのUpdateを許可しているので、`yes`を返すはずです。

```Bash
kubectl auth can-i update pods --as=handson-user
```

続いて、deploymentの作成が可能かを確かめます。
deploymentに関する許可は行なっていないため`no`を返すはずです。

```Bash
kubectl auth can-i create deployment --as=handson-user
```

このように、対象のユーザがどのリソースの操作を許可するかを細かく設定することが可能です。

動作確認後、リソースを削除します。

```
kubectl delete rolebinding handson-user-rolebinding 
kubectl delete role handson-user-role
kubectl delete csr handson-user
```

### 11. おまけ(jsonpath)

jsonpathは、ワンライナーで欲しい情報のみを引き抜く際に便利な機能です。
jsonpathでNodeの内部IPのみをファイルに書き出してみましょう。

```Bash
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}' > <ファイルのPathとファイル名>
```

## 12. Readiness/Liveness Probe

KubernetesにはPodが正常に起動したか、または正常に動作を続けているかを監視する機能が存在します。
このセクションで取り扱うReadiness/Liveness Probeは、コマンドの実行結果やTCP・HTTPリクエストなどのリターンコードによって
そのPodの準備が出来ているかどうか、または正常に動作しているかどうかを判断します。


以下はヘルスチェックのオプションです。


- initialDelaySeconds
  初回ヘルスチェックまでの遅延時間（秒）
- periodSeconds
  Probeが実行される間隔（秒）
- timeoutSeconds
  タイムアウトまでの時間（秒）
- successThreshold
  成功と判断する最小連続成功数（回数）
- failureThreshold
  失敗と判断する試行回数（回数）

### 12.1 Readiness Probe

今回は`/tmp/ready`ファイルの有無によって、Podの準備が出来ているかを判断するシナリオです。
まずは対象のファイルを作成しない状態でPodをデプロイしてみます。


```Bash
kubectl apply -f readiness-pod.yaml
```

対象のファイルが作成されていない状態ではPodがReadyのステータスにならないことがわかります。

```
kubectl get pod
```

```
NAME                    READY   STATUS    RESTARTS      AGE
readiness-pod           0/1     Running   0             7s
```


続いてreadiness-pod.yamlを以下のように編集して、コンテナ内に対象のファイルを作成するようにします。


```Yaml
  - command:
    - sh
    - -c
    - touch /tmp/ready && sleep 1d
```

vimなどでファイルを編集し、以下のコマンドでPodを入れ替えてみましょう。

```
kubectl replace -f readiness-pod.yaml --force
```

再度Podの状態を確認すると、状態がReadyになっていることが確認できます。


```
NAME                    READY   STATUS    RESTARTS      AGE
readiness-pod           1/1     Running   0             7s
```

動作確認後、リソースを削除します。

```
kubecttl delete pod readiness-pod
```

### 12.2 Liveness Probe


続いて、Liveness Probeの動作確認を行います。
Readiness Probe同様、ファイルの有無によってPodの正常性を確認します。
このシナリオでは`/tmp/healthy`ファイルが自動的に削除されます。
そのため、ヘルスチェックに失敗したPodは自動的にリスタートを行います。

以下のコマンドでPodの挙動が確認できます。
Pod作成からしばらく経つと、`RESTARTS`のカウンタが上昇していくのが確認できます。

```
watch -n 1 kubectl get pod
```

動作確認後、リソースを削除します。

```
kubectl delete pod liveness-pod
```

## 13. Network Policy

Network PolicyはPod同士の通信を制御し、特定のPodやプロトコルを許可/拒否させることができるリソースです。

前提として、以下のCNIを使ってクラスタを構築している必要があります。

- Calico
- Cilium
- Kube-router
- Romana
- Weave Net

尚、今回はCiliumを使用しています。

設定方法として、以下を意識する必要があります。

- 通信の方向
  - Ingress：　あるPodからの通信（インバウンド）
  - Egress：　あるPodへの通信（アウトバウンド）
- Policy
  - podSelector: あるPodから、もしくはPodへの通信可否
  - namespaceSelector: あるNamespaceから、もしくはNamespaceへの通信可否
  - ipBlock: あるIPアドレスから、もしくはIPアドレスへの通信可否

今回は3つのテスト用のPodをデプロイし、curlを使って通信確認を行なっていきます。

```
kubectl apply -f netpol-pod.yaml
```

通信確認を行うためにPodに付与されているIPアドレスを確認します。

```
kubectl get pod -o wide -L app | grep app
```

以下のように、curlを使ってPod同士の通信確認をそれぞれ行なっていきます。

```
kubectl exec -it nginx-app1 -- curl -I <PodのIP>
kubectl exec -it nginx-app2 -- curl -I <PodのIP>
kubectl exec -it nginx-app3 -- curl -I <PodのIP>
```

続いて、すべての通信を拒否するNetwork Policyを適用します。


```
kubectl apply -f default-deny-all.yaml
```

先ほどと同じようにcurlを投げても、タイムアウトになることが確認できます。

```
kubectl exec -it nginx-app1 -- curl -I <PodのIP>
kubectl exec -it nginx-app2 -- curl -I <PodのIP>
kubectl exec -it nginx-app3 -- curl -I <PodのIP>
```

続いて、`app1`と`app3`同士の通信のみが許可されるポリシーを適用していきます。
今回は`podSelector`を使用してポリシーを設定しています。

```
kubectl apply -f handson-policy.yaml
```

設定後、`app1`と`app3`同士の通信のみが可能であることが確認できます。

```
kubectl exec -it nginx-app1 -- curl -I <PodのIP>
kubectl exec -it nginx-app2 -- curl -I <PodのIP>
kubectl exec -it nginx-app3 -- curl -I <PodのIP>
```

動作確認後、リソースを削除します。

```
kubectl delete networkpolicy default-deny-all
kubectl delete networkpolicy app1-app3
kubectl delete networkpolicy app3-app1
kubectl delete pod nginx-app1
kubectl delete pod nginx-app2
kubectl delete pod nginx-app3
```



## JobとCronJob

### Job

Jobは、ReplicaSetと同様、Podを管理するためのPodの上位リソースに該当します。
Podを使って一時的な処理を行う際に利用するリソースで、処理を実行後にPodは自動的に削除されます。
今回はechoでメッセージを出力する簡単なJobを実行します。

Jobリソースは、並列動作や繰り返し動作に関するオプションのパラメータを設定することが可能です。


- completion
  指定した回数Podが正常終了したら、Jobが終了する
- parallelism
  指定した数値分Podを並列で起動する
- backoffLimit
  指定した回数分Podをリトライする。リトライ回数が上限に達するとエラー判定となる


今回は以下のように設定しているため、計6回Jobが実行され
2つのPodが並列で動作します。

```
completion: 6
parallelism: 2
```

以下のManifestを適用します。

```
kubectl apply -f handson-job.yaml
```

動作確認は以下のコマンドで行います。

```
kubectl get job
```

以下のように、完了したJobは`COMPLETIONS`としてカウントされていきます。

```
NAME          COMPLETIONS   DURATION   AGE
handson-job   6/6           15s        58s
```

また、Podの挙動を観察することで動作確認をすることも可能です。
以下のコマンドで2つずつ並列でJobが実行されていくのが確認できます。

```
watch -n 1 kubectl get pod
```

実際のJobの実行結果はLogを確認します。

```
kubectl logs <Pod名>
```

確認が完了したらリソースを削除します。


```
kubectl delete job handson-job
```

### CronJob

CronJobは、リソース内のCronに従って、スケジュールされた時間にJobを実行します。
CronJobは、先ほど実行したJobの上位リソースに当たります。

今回は1分ごとにJobを動作させるシナリオです。
それでは、前回のJobのシナリオ同様にManifestをapplyして動作を確認していきましょう。

```
kubectl apply -f handson-cronjob.yaml
```

1分ごとにJobが増えていくのが確認できます。

```
watch -n 1 kubectl get pod
```

今回はdateコマンドを実行するJobなので、日付が出力されているはずです。

```
kubectl logs <Pod名>
```

以下のコマンドはCron Jobのステータスや詳細が確認できます。

```
kubectl get cronjob

kubectl describe cronjob　handson-cronjob

```

Cron Jobを一時停止したい場合は、kubectl patchコマンドを使用します。
リソース内の`spec.suspend`パラメータを`true`にすることで停止が可能です。

`kubectl get cronjob`を実行すると、現在は`SUSPEND`が`False`になっていることが確認できます。

```
NAME              SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
handson-cronjob   */1 * * * *   False     0        24s             7m24s
```

以下のコマンドを実行します。

```
kubectl patch cronjob handson-cronjob -p '{"spec":{"suspend":true}}'
```

実行後、以下のようにステータスが変更されていることが確認できます。

```
NAME              SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
handson-cronjob   */1 * * * *   True      0        8s              13m
```

動作確認後、リソースを削除します。


```
kubectl delete cronjob handson-cronjob
```

## ConfigMap


```
kubectl apply -f handson-configmap.yaml
```


```
kubectl apply -f configmap-pod.yaml
```


```
kubectl get pod -o wide
```


```
kubectl run tmp --restart=Never --rm -i --image=nginx:alpine -- curl <PodのIPアドレス>
```


```
kubectl delete pod configmap-pod
kubectl delete pod handson-configmap
```