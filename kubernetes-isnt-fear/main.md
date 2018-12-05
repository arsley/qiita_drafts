<!-- 怖くないKubernetes（とDocker） -->


# 目的

昨今の流行り（？）らしいKubernetesを試し、その復習と布教を兼ねて記事にします。
※間違いが散見される可能性がありますがお許しください※

実践というよりかは読み物だと思って「そうなんだ、今度やってみよー（棒）」くらいの気持ちで読むと良いかもしれません。

# 環境

- macOS 10.13.6 （Mojaveじゃないよ、Sierraだよ）
- Docker for Macインストール済み
  - Engine 18.06.1-ce
  - Kubernetes v1.10.3

基本的にはmacOSでの作業となるので、windowsやlinuxの人は適宜読み替えてください。

# 始める前に...

## 必要知識

Kubernetesに入門するに際して、知っておくべきことがあります。
全くのゼロ知識でも進められるとは思いますが、知っておくことでKubernetesによる恩恵をより感じられるものと思います。

- Dockerの基礎的な利用方法
- Container（DockerContainer 以下コンテナ）に対する簡単な理解
- Image（DockerImage 以下イメージ）に対する簡単な理解

それぞれについて私なりにですが簡単な説明をします。
知っている方は[次の章](#kubernetes)に進んでください。

## Dockerの基礎

[Docker](https://www.docker.com/)とは、オープンソースのコンテナ管理ソフトウェアです。
昨今では、仮想マシン（Virtual Machine）を用いた運用体制から、
コンテナによる環境の隔離や軽量性を重視する風潮となりつつ有ります。
コンテナの起動や作成、イメージのビルドなどをDockerを通じて簡単に行うことができます。
仮想マシンと比較した時の利点として、起動が早く扱いやすいという点が挙げられます。

```shell:terminal
# nginx イメージから nginxという名前のコンテナを作る（実行させる）
$ docker run --rm -d --name nginx -p 8080:80 nginx

# localhost:8080 で "Welcome to nginx!" が確認できたら止める
$ docker stop nginx
```

## コンテナの基礎

_ここでのコンテナはDockerにおけるコンテナを指しています。_

コンテナとは、基盤とするOS・実行するコマンドやアプリケーション・各種設定などを1つの環境としてまとめた単位です。
`Dockerfile`という構成ファイルを作成しておくことで、作成するコンテナの概要を決めることができます（後述します）。
後述するイメージとの違いとして、

- コンテナ : 実行されている環境
- イメージ : コンテナを実行するための基盤

というものがあります。
端的に言えば、「イメージ（設計図）からコンテナ（実体）を作成し、コンテナを実行する」というような関係にあります。

## イメージについて

イメージとは、コンテナという実体を作成するための基盤のようなものです。
OSインストールなどで用いられるディスク_イメージ_のようなもの言えばわかりやすいのではないでしょうか。

上述したように、`Dockerfile`なるものを作成しておくことで、
実体として（実行可能なものとして）作成されるコンテナの概要を決めることができます。

```Dockerfile:Dockerfile
FROM alpine:latest

RUN apk add --update nginx
RUN mkdir -p /run/nginx && touch /run/nginx/nginx.pid
RUN echo '<h1>Hello from MyNginx Container!</h1>' > /var/lib/nginx/html/index.html
RUN sed -i -e 's/return 404;/index index.html;/' -e 's/internal;/return 404;/' /etc/nginx/conf.d/default.conf

ENTRYPOINT ["nginx", "-g", "daemon off;"]
```

```shell:terminal
# イメージのビルド
$ docker build -t mynginx .

# mynginx イメージから mynginx-containerという名前のコンテナを作る（実行させる）
$ docker run --rm -d --name mynginx-container -p 8080:80 mynginx

# localhost:8080 で "Hello from MyNginx Container!" が、
# localhost:8080/404.html で "404 NotFound" が確認できたら止める
$ docker stop mynginx-container
```

`Dockerfile` にてコンテナ内で行ってほしい作業（`RUN ...`）および実行時の挙動（`ENTRYPOINT`）を記述し、
`docker build` にてイメージ化（コンパイルのようなもの）をします。
そして `docker run ...` にてイメージからコンテナを作成し、実行させるという流れになっています。

_Build once, Run anywhere_と言われているように、
イメージ化しておくことで、各人の環境に左右されることなく同じ結果（コンテナ）の再現が可能であるということもDockerの強みです。

<!-- ## ここまでのまとめ

- Dockerはコンテナ管理ソフトウェア
- コンテナはイメージを元に作られる実体（`ENTRYPOINT`の部分が実行されている）
- イメージはDockerfileから作成可能な、コンテナ基盤 -->

# Kubernetes

## 概要

[Kubernetes](https://kubernetes.io/)とは、
Dockerによってコンテナ化されたアプリケーションの自動デプロイ・スケーリング・管理を行うオープンソースのコンテナオーケストレーションシステムです。

DockerではDockerfileによるイメージの生成やコンテナの実行・停止などといった、単一のコンテナにおける環境の作成が主となっていますが、
Kubernetesでは1つ以上のコンテナを「Workloads」や「Discovery&LB」などという「リソース」として扱い、コンテナの運用基盤を提供するという点で異なっています。

Kubernetesではこの運用基盤をJSON|YAML形式で書き表されたマニフェストファイル（以下マニフェスト）で定義することができます。


```yml:kubernetes-isnt-fear.yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    web: nginx-pod-label
spec:
  containers:
  - name: nginx-container
    image: nginx:1.12

---

apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: redis-replicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      redis: redis-replicaset-label
  template:
    metadata:
      labels:
        redis: redis-replicaset-label
    spec:
      containers:
        - name: redis-container
          image: redis:5.0

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      httpd: httpd-deployment-label
  template:
    metadata:
      labels:
        httpd: httpd-deployment-label
    spec:
      containers:
        - name: httpd-container
          image: httpd:2.4

---

apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  type: LoadBalancer
  ports:
  - name: "http-port"
    protocol: "TCP"
    port: 8080
    targetPort: 80
  selector:
    web: nginx-pod-label
```

```shell:terminal
# 各々のリソースを起動
$ kubectl apply -f kubernetes-isnt-fear.yml

# 起動してるリソースの確認（一例）
$ kubectl get all -o wide
NAME                                   READY   STATUS    RESTARTS   AGE   IP          NODE
pod/httpd-replicaset-875598bb4-n8khn   1/1     Running   0          12s   10.1.0.32   docker-for-desktop
pod/httpd-replicaset-875598bb4-qvwzx   1/1     Running   0          12s   10.1.0.33   docker-for-desktop
pod/httpd-replicaset-875598bb4-tnlvt   1/1     Running   0          12s   10.1.0.30   docker-for-desktop
pod/nginx-pod                          1/1     Running   0          12s   10.1.0.28   docker-for-desktop
pod/redis-replicaset-g4m75             1/1     Running   0          12s   10.1.0.29   docker-for-desktop
pod/redis-replicaset-m8g7v             1/1     Running   0          12s   10.1.0.27   docker-for-desktop
pod/redis-replicaset-wq76w             1/1     Running   0          12s   10.1.0.31   docker-for-desktop

NAME                 TYPE           CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE   SELECTOR
service/kubernetes   ClusterIP      10.96.0.1     <none>        443/TCP          22m   <none>
service/nginx-lb     LoadBalancer   10.97.5.131   localhost     8080:31650/TCP   12s   web=nginx-pod-label

NAME                               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS        IMAGES      SELECTOR
deployment.apps/httpd-replicaset   3         3         3            3           12s   httpd-container   httpd:2.4   httpd=httpd-replicaset-label

NAME                                         DESIRED   CURRENT   READY   AGE   CONTAINERS        IMAGES      SELECTOR
replicaset.apps/httpd-replicaset-875598bb4   3         3         3       12s   httpd-container   httpd:2.4   httpd=httpd-replicaset-label,pod-template-hash=431154660
replicaset.apps/redis-replicaset             3         3         3       12s   redis-container   redis:5.0   redis=redis-replicaset-label


# (ローカル環境なら) localhost:8080にて "Welcome to nginx!" が確認できる
# リソースを終了（削除）
$ kubectl delete -f kubernetes-isnt-fear.yml

```

マニフェストを書き、コマンド一つでこれだけの環境がすぐに立ち上がります。
流石にこれらを `docker run...` を使ったりして立ち上げるのは根気が必要ですね。
一つ一つ見ていきます。

## リソースって?

リソースとは、Kubernetesの管理対象として「登録」することができるもののことを言います。
今回は（個人的によく使いそうだと思った）以下のものを紹介します。

- Workloadsリソース
  - Pod
  - ReplicaSet
  - Deployment
- Discovery&LBリソース
  - Serivce（LoadBalancer）

## Workloadsリソース

Workloadsリソースは、コンテナの起動や実行に関するリソースです。

### Pod

Podは、1つ以上のコンテナから構成される、Workloadsリソースにおける最小単位です。
概要にて記載した以下の部分

```yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    web: nginx-pod-label
spec:
  containers:
  - name: nginx-container
    image: nginx:1.12
```

に相当します。

1つ以上のコンテナから構成されるので、このようなことも可能です。

```yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    web: nginx-pod-label
spec:
  containers:
  - name: nginx-container
    image: nginx:1.12
  - name: redis-container
    image: redis:5.0
  - name: original-container
    image: yourAccount/original:latest
```

Podは基本的にコンテナを実行させるためだけのものであり、後述するReplicaSetやDeploymentとは異なり停止した時に再起動をしません。
障害耐性を持っていないため、基本的にPodの単体利用は試験的なものにのみすべきとのことです。

```shell
# 動作Podの確認
$ kubectl get pods
NAME        READY   STATUS    RESTARTS   AGE
nginx-pod   1/1     Running   0          17s

# Podの停止
$ kubectl delete pods nginx-pod

# 確認
$ kubectl get pods
No resources found.
```

### ReplicaSet

ReplicaSetは `replicas:` にて指定した個数だけPodを維持し続けるように管理を行うリソースです。
以下の部分

```yml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: redis-replicaset
spec:
  replicas: 3
  selector:
    matchLabels:
      redis: redis-replicaset-label
  template:
    metadata:
      labels:
        redis: redis-replicaset-label
    spec:
      containers:
        - name: redis-container
          image: redis:5.0
```

に相当します。

このマニフェストでは、 `redis:5.0` が実行されるPodを3つ（レプリカ数3）作成することを意味しています。
各Podには `redis-replicaset-label` というLabelが付加されており、この数が常に3つとなるようKubernetesが管理を行います。

```shell
# `-w` オプションでリアルタイムに変更を監視する
$ kubectl get pods -w
NAME                     READY   STATUS    RESTARTS   AGE
redis-replicaset-d7ms6   1/1     Running   0          62s
redis-replicaset-f8lvq   1/1     Running   0          62s
redis-replicaset-mqqqx   1/1     Running   0          62s

$ kubectl delete pods redis-replicaset-d7ms6

# こんなLogが流れる
redis-replicaset-d7ms6   1/1   Terminating   0     90s # 何かあって異常終了した
redis-replicaset-lhg8v   0/1   Pending       0     0s  # レプリカ数の変更（3→2）を検知して新しいPodを作り始める
redis-replicaset-lhg8v   0/1   Pending       0     0s
redis-replicaset-lhg8v   0/1   ContainerCreating   0     0s
redis-replicaset-lhg8v   1/1   Running       0     2s  # 新しいPodが動き始める
redis-replicaset-d7ms6   0/1   Terminating   0     92s
redis-replicaset-d7ms6   0/1   Terminating   0     93s
redis-replicaset-d7ms6   0/1   Terminating   0     93s
```

### Deployment

Deploymentは、複数のReplicaSetを管理するリソースです。
以下の部分

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: httpd-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      httpd: httpd-deployment-label
  template:
    metadata:
      labels:
        httpd: httpd-deployment-label
    spec:
      containers:
        - name: httpd-container
          image: httpd:2.4
```

に相当します。

ReplicaSetと比較してマニフェストの構成がほぼ変わりませんが、運用に適した機能が追加されています。
ReplicaSetと同様に障害耐性（異常終了した時などに新しくPodを立て直す）はもちろんのこと、
マニフェストの更新に応じたRollingUpdateやRollbackを行うことができるようになります。

RollingUpdateとは、ライブラリやコンテナなどのアップデート作業（いわゆるマイグレーション）時に、
稼働していないPodを作ることなくマイグレーション作業を遂行する仕組みです。
例えば、上述したものマニフェストにおける `image` を `httpd:2.4.37` に変更し、 `kubectl apply ...` してみましょう。

```shell
$ kubectl get pods -w
NAME                               READY   STATUS              RESTARTS   AGE
httpd-deployment-875598bb4-mzgjm   1/1     Running             0          3s
httpd-deployment-875598bb4-2l8vf   1/1     Running             0          4s
httpd-deployment-875598bb4-l9qs6   1/1     Running             0          4s

# マニフェストを書き換え、kubectl apply ...が実行されると

httpd-deployment-86db87c6c6-mdbbf   0/1   Pending   0     0s     # 1
httpd-deployment-86db87c6c6-mdbbf   0/1   Pending   0     0s
httpd-deployment-86db87c6c6-mdbbf   0/1   ContainerCreating   0     0s
httpd-deployment-86db87c6c6-mdbbf   1/1   Running   0     7s
httpd-deployment-875598bb4-2l8vf   1/1   Terminating   0     26s
httpd-deployment-86db87c6c6-lgk4g   0/1   Pending   0     0s     # 1
httpd-deployment-86db87c6c6-lgk4g   0/1   Pending   0     0s

httpd-deployment-86db87c6c6-lgk4g   0/1   ContainerCreating   0     0s
httpd-deployment-875598bb4-2l8vf   0/1   Terminating   0     27s
httpd-deployment-86db87c6c6-lgk4g   1/1   Running   0     2s
httpd-deployment-875598bb4-2l8vf   0/1   Terminating   0     28s
httpd-deployment-875598bb4-2l8vf   0/1   Terminating   0     29s # 2
httpd-deployment-86db87c6c6-t4fbq   0/1   Pending   0     0s     # 1
httpd-deployment-86db87c6c6-t4fbq   0/1   Pending   0     0s
httpd-deployment-86db87c6c6-t4fbq   0/1   ContainerCreating   0     0s
httpd-deployment-875598bb4-l9qs6   0/1   Terminating   0     30s
httpd-deployment-875598bb4-l9qs6   0/1   Terminating   0     31s
httpd-deployment-875598bb4-l9qs6   0/1   Terminating   0     31s
httpd-deployment-875598bb4-l9qs6   0/1   Terminating   0     31s # 2
httpd-deployment-86db87c6c6-t4fbq   1/1   Running   0     2s

httpd-deployment-875598bb4-mzgjm   1/1   Terminating   0     31s
httpd-deployment-875598bb4-mzgjm   0/1   Terminating   0     33s
httpd-deployment-875598bb4-mzgjm   0/1   Terminating   0     34s
httpd-deployment-875598bb4-mzgjm   0/1   Terminating   0     34s # 2
```

だいぶ見辛い（図を作れって話ですが）のですが、流れとしては `kubectl delete pods` した時に似ています。

1. 新しいマニフェストに基づいたPodを作り始める
2. Podが作成されたら、旧Podは終了される

また、マニフェストの更新によりPodだけが変更されるのではなく、Podを管理するReplicaSetも同様にして作成されます。
この時、上書きではなく新規作成をしている点に留意してください。

```shell
# 新しいReplicaSetも作成されている
$ kubectl get replicasets
NAME                          DESIRED   CURRENT   READY   AGE
httpd-deployment-86db87c6c6   3         3         3       12m
httpd-deployment-875598bb4    0         0         0       12m
```

旧バージョンのReplicaSetがあれば、その状態にRollbackすることが可能です。

```shell
# 旧バージョンへロールバック（出力は省略）
$ kubectl rollout undo deployment httpd-deployment-86db87c6c6
```

## Discovery&LBリソース

Discovery&LBリソースは、
ノード内（ローカルのKubernetesならMacOSなど）のコンテナを外部から参照可能な形にするリソースです。
IPアドレスの割り当て（localhost:8080へのバインドなど）といったエンドポイントを提供するものです。

### Service（LoadBalancerService）

Service（LoadBalancerService）は、Podへの通信をさせるために、
クラスタ（ノードの集合）の外に配置されるLoadBalancerへIPアドレスを割り振ることができるリソースです。
以下の部分

```yml
apiVersion: v1
kind: Service
metadata:
  name: nginx-lb
spec:
  type: LoadBalancer
  ports:
  - name: "http-port"
    protocol: "TCP"
    port: 8080
    targetPort: 80
  selector:
    web: nginx-pod-label
```

に相当します。

（書くのが疲れてきてしまいました）
Dockerにして例えれば、 `docker run ... -p 8080:80 ...` のようなものと思えば良いかもしれません。

# おさらい

今回の記事では

- Dockerの簡単な説明
- Kubernetesの概要・主な操作とリソースなどの説明

をしました。

本当は「Railsアプリを実際に乗っけてみよう」っていう章を用意しようと思っていたのですが、予想以上に文章量が増えてしまったので割愛します...

時間制限がありますが、Web上でKubernetesを試すことができる[Play with Kubernetes](https://labs.play-with-k8s.com/)なるものがありますので、少しでも興味を持ったらぜひチャレンジしてみてください:tada:

_追記_
家のWiFiがブツブツ切れて辛いです...（原因がわからない）
