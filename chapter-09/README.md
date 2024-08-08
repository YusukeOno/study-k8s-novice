# K8sの仕組み、アーキテクチャを理解する

K8sの仕組みやアーキテクチャを知っておくと、複雑な問題に当たった時に理解が進みやすくなる。

## K8sのアーキテクチャについて

Kubernetesは、実は「アプリサーバとデータベースというWebサービスでよくみるインフラ構成と似ている」。

## アーキテクチャ概要

![OverView](../chapter-02/overview.drawio.svg)

## K8sクラスタの要となるControl Plane

kubectlやkube-apiserverについては以前説明したが、改めて見ていく。

これらのコンポーネントはkube-systemのNamespace内のPodを参照することで実際に動いていることを確認できる。

```zsh
> kubectl get pod --namespace kube-system
NAME                                         READY   STATUS    RESTARTS   AGE
coredns-76f75df574-5shx6                     1/1     Running   0          24m
coredns-76f75df574-x5cff                     1/1     Running   0          24m
etcd-kind-control-plane                      1/1     Running   0          25m
kindnet-4cxtw                                1/1     Running   0          24m
kube-apiserver-kind-control-plane            1/1     Running   0          25m
kube-controller-manager-kind-control-plane   1/1     Running   0          25m
kube-proxy-wmtl4                             1/1     Running   0          24m
kube-scheduler-kind-control-plane            1/1     Running   0          25m
```

kube-apiserverはRESTで通信可能なAPIサーバである。etcdは分散型キーバリューストアである。Control PlaneはAPIサーバとデータベースでできている。

実際、kube-apiserverはユーザ(kubectl)からのリクエストを受けてetcdにデータを保存している。また、kubectl getではetcdに保存指定あるデータをkube-apiserverを通じて受け取っている。これらの操作は、kubectlで確認することもできる。

```zsh
> kubectl get pod --v 7 --namespace kube-system
I0802 11:10:56.893185   43889 loader.go:395] Config loaded from file:  /Users/yusuke.ono/.kube/config
I0802 11:10:56.902934   43889 round_trippers.go:463] GET https://127.0.0.1:60732/api/v1/namespaces/kube-system/pods?limit=500
I0802 11:10:56.902947   43889 round_trippers.go:469] Request Headers:
I0802 11:10:56.902955   43889 round_trippers.go:473]     Accept: application/json;as=Table;v=v1;g=meta.k8s.io,application/json;as=Table;v=v1beta1;g=meta.k8s.io,application/json
I0802 11:10:56.902962   43889 round_trippers.go:473]     User-Agent: kubectl1.29.0/v1.29.0 (darwin/amd64) kubernetes/3f7a50f
I0802 11:10:56.914116   43889 round_trippers.go:574] Response Status: 200 OK in 11 milliseconds
NAME                                         READY   STATUS    RESTARTS   AGE
coredns-76f75df574-5shx6                     1/1     Running   0          27m
coredns-76f75df574-x5cff                     1/1     Running   0          27m
etcd-kind-control-plane                      1/1     Running   0          27m
kindnet-4cxtw                                1/1     Running   0          27m
kube-apiserver-kind-control-plane            1/1     Running   0          27m
kube-controller-manager-kind-control-plane   1/1     Running   0          27m
kube-proxy-wmtl4                             1/1     Running   0          27m
kube-scheduler-kind-control-plane            1/1     Running   0          27m
```

kubectl get podに対して、どのようなリクエストが行われているかをみることができる。

`https://127.0.0.1:60732/api/v1/namespaces/kube-system/pods?limit=500`に対しGET通信が行われ、200 OKのレスポンスが返ってきている。

それでは、他のコンポーネントは何をしているのだろうか。kube-schedulerはPodをNodeにスケジュールする役割を担っている。これまでAffinityの説明などで「Podをスケジュールする」などと書いたが、これはkube-schedulerが決めている。

kube-contoller-managerはKubernetesを最低限動かすために必要な複数のコントローラを動かしている。コントローラは、「マニフェストに書かれている内容に応じて動作する」プログラム全般をコントローラという。

## アプリの実行を担うWorker Node

Worker Nodeは、実際にアプリコンテナの起動を行うNodeである。Control Planeは冗長化を考慮してNodeが3台程度で動くのに対して、Worker Nodeは規模によっては100台動かすこともある。では、各コンポーネントの詳細を見ていく。

- kubelet: クラスタ内の各Nodeで動いている。Podに紐づくコンテナを管理している。kubeletが起動しているNodeにPodがスケジュールされると、コンテナランタイムに指示して今点を起動する。
- kube-proxy: Kubernetes Serviceリソースなどに応じてネットワーク設定を行うコンポーネントである。クラスタ内の各ノード上で動作する。kube-proxyによってクラス内外のネットワークよってクラス内外のネットワークセッションからPodへのネットワーク通信が可能となる。
- コンテナランタイム: コンテナを実行する役割のソフトウェアである。K8s特有の技術ではない。コンテナランタイムはソフトウェアの総称であり、具体的にはcontainerdやCRI-Oが挙げられる。

## KubernetesクラスタにアクセスするためのCLI: kubectl

kube-apiserverに関連して　kubectlについて説明したが、改めて説明する。

kubectlとはkube-apiserverと通信するためのCLIツールである。kube-apiserverはRESTfulなAPIサーバですので、kubectlなしでもcurlなどを利用して通信することは可能。

しかし実際に利用しようとすると、非常に面倒である。そこで利用するのがラッパーツールであるkubectlである。kube-apiserverと通信するためには全てJSON形式である必要があるが、これまでJSONを意識したことはないはずである。出力もYAML形式であった。これらはすべてkubectlがkube-apiserverとの通信をユーザーによって見やすいYAML形式に変換してくれていたからだ。

## kubectl applyしてからコンテナが起動するまでの流れ

`kubectl apply --filename pod.yaml` を実施すると何が起きるだろうか？

kubectlからkube-apiserverにPodの作成が指示され、etcdにマニフェストで指定した情報が保存される。マニフェストに指定された内容をもとにスケジューラがどのNodeにコンテナを起動すべきか決定する。kubectlは自分のNodeにコンテナを起動すべきことを検知し、コンテナランタイムに指示してコンテナを起動する。

## 作って、壊す　K8sは壊せない？

これまで説明したように、K8sでは各コンポーネントがそれぞれ役割を持って自立し動いている。そのおかけでK8sは障害に強いと言われており、壊そうと思っても壊れない。

ここではK8sのControl Planeを破壊して、K8sの壊れにくさを体感する。今回はControl PlaneとWoker Nodeが分かれている必要があり、かつNodePortが使用可能である必要がある。

### 準備クラスタを構築する

既存クラスタがあれば、一旦削除する。

```zsh
> kubectl delete cluster
Deleting cluster "kind" ...
Deleted nodes: ["kind-control-plane"]
```

マニフェスト`kind/multinode-nodeport.yaml`を利用して、次のコマンドでクラスタを構築する。

```zsh
> kind create cluster -n multinode-nodeport --config ./kind/multinode-nodeport.yaml --image=kindest/node:v1.29.0
Creating cluster "multinode-nodeport" ...
 ✓ Ensuring node image (kindest/node:v1.29.0) 🖼
 ✓ Preparing nodes 📦 📦 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
 ✓ Joining worker nodes 🚜 
Set kubectl context to "kind-multinode-nodeport"
You can now use your cluster with:

kubectl cluster-info --context kind-multinode-nodeport

Have a nice day! 👋
```

node一覧を確認する。

```zsh
> kubectl get node
NAME                               STATUS   ROLES           AGE   VERSION
multinode-nodeport-control-plane   Ready    control-plane   44s   v1.29.0
multinode-nodeport-worker          Ready    <none>          24s   v1.29.0
multinode-nodeport-worker2         Ready    <none>          24s   v1.29.0
```

想定通りマルチノードになっている。

### hello-serverを起動する

hello-serverを起動する。マニフェストは`chapter-09/hello-server.yaml`を使用する。

マニフェストをapplyする。

```zsh
> kubectl apply --filename chapter-09/hello-server.yaml --namespace default
deployment.apps/hello-server created
poddisruptionbudget.policy/hello-server-pdb created
service/hello-server-external created
```

Podが起動していることを確認する。

```zsh
> kubectl get pod --namespace default
NAME                          READY   STATUS    RESTARTS   AGE
hello-server-965f5b86-5vn25   0/1     Running   0          27s
hello-server-965f5b86-d27kq   0/1     Running   0          27s
hello-server-965f5b86-wm2br   0/1     Running   0          27s
```

hello-serverへの疎通を確認する。まずはNodeのIPを取得する。

```zsh
> kubectl get node multinode-nodeport-worker -o jsonpath='{.status.addresses[?(@.type=="InternalIP")].address}'
172.18.0.3%
```

取得したInternalIPを利用してアクセスする。

```zsh
> curl localhost:30599
Hello, world! Let's learn Kubernetes!
```

アプリが問題なく動いているようだ。

### Control Planeを停止する

いきなりControl Planeを停する。kindを利用しているのであれば、Control Plane用のDockerコンテナを止める。まずは、Control PlaneのdockerコンテナIDを確認する。

```zsh
> docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED              STATUS              PORTS                        NAMES
6a9d676c8a0a   kindest/node:v1.29.0   "/usr/local/bin/entr…"   About a minute ago   Up About a minute   127.0.0.1:30599->30599/tcp   multinode-nodeport-worker
fa75ea597b3b   kindest/node:v1.29.0   "/usr/local/bin/entr…"   About a minute ago   Up About a minute                                multinode-nodeport-worker2
a41f895611f6   kindest/node:v1.29.0   "/usr/local/bin/entr…"   About a minute ago   Up About a minute   127.0.0.1:57172->6443/tcp    multinode-nodeport-control-plane
```

multinode-nodeport-control-planeと書かれているコンテナのCONTAINER IDをコピーし、コンテナを止める。

```zsh
> docker stop a41f895611f6
a41f895611f6
```

では、hello-serverはどうなっただろうか？接続確認してみる。

```zsh
> curl localhost:30599
Hello, world! Let's learn Kubernetes!
```

問題なく動いていそうだ。では、podのSTATUSも見てみる。

```zsh
> kubectl get pod --namespace default
The connection to the server 127.0.0.1:57172 was refused - did you specify the right host or port?
```

接続ができない、これはControl Planeを停止したため、kube-apiserverに接続できなくなっていることが理由である。しかし、Control Planeが停止したとしてもコンテナは起動し続ける。kube-apiserverと接続できないのでコンテナの更新やPod数の増減は行えないが、少なくともサービスの稼働が即ざいに損なわれるということはない。Kubernetesが障害に強いと言われる理由がわかっただろうか。

Control Planeを起動し直すことが、今まで通りkubectlが使えるようになる。

```zsh
> docker start a41f895611f6
a41f895611f6
```

Podを参照できることを確認する。

```zsh
> kubectl get pod --namespace default
NAME                          READY   STATUS    RESTARTS   AGE
hello-server-965f5b86-djs88   1/1     Running   0          4m53s
hello-server-965f5b86-fd6xq   1/1     Running   0          4m53s
hello-server-965f5b86-vzppj   1/1     Running   0          4m53s
```

最後にクラスタごと掃除する。

```zsh
> kind delete cluster -n multinode-nodeport
Deleting cluster "multinode-nodeport" ...
Deleted nodes: ["multinode-nodeport-worker" "multinode-nodeport-worker2" "multinode-nodeport-control-plane"]
```

デフォルトクラスタを立ち上げ直す。

```zsh
> kind create cluster --image=kindest/node:v1.29.0
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.29.0) 🖼
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Thanks for using kind! 😊
```

## Kubernetesを拡張する

K8sの特徴の一つとして「Kubernetesユーザーが自分でKubernetesを拡張できる」ということが挙げられる。どんなに素晴らしいシステムだとしても、全てのユーザーのニーズを満たすことは難しい。

その分、拡張性を持ったシステムであれば、自分たちの要件・用途に合わせて改善を繰り返すことができ、多くのユーザーにとって便利なものとなる。

これまでPodやDeploymentなどの「リソース」を作成してきたが、Kubernetesが標準で用意しているリソースでは物足りないことがある。Argo CDというOSSは「リポジトリ名、パス名を指定すると指定場所に書かれているマニフェストを参照して自動でデプロイを行う」ことが可能になる。

しかし、既存のリソースを使ってもこの機能を実現することはできない。「リポジトリ名」「パス名」を指定するリソースと、このリソースの内容をもとに「自動デプロイを行う」プログラムが必要になってきている。ここでいう独自リソースが「Custom Resource(CR)」となり、CRを作るために必要が定義が「Custom Desource Definition(CRD)」である。

また、CRを参照して動くプログラムを「カスタムコントローラ（コントローラ）」という。ここでいうコントローラというのはkube-controller-managerで説明したコントローラと同じ概念である。K8s標準で搭載されているDeployment Controllerはkube-ontroller-managerに内包されているが、各自がKubernetesを「カスタム」するために使うコントローラーがカスタムコントローラーである。

ConfigMapに必要な情報を書いておき、その情報を読むようなプログラムを書けば良いと思うかもしれがい。kubernetes公式ドキュメントには、「Should I use a ConfigMap or a custom resource?」という項目で判断記述となる例を挙げている。公式ドキュメントでは次のうち一つでも当てはまるものがあればConfigMapを使うと良いと書かれている。

- 既存の、よく文書化された設定ファイル形式がある。例えば、mysql.cnfやpom.xml
- 設定全体をConfigMapの一つのキーに入れたい
- Podで動作するプログラムが、自信を設定するためにそのファイルを利用する
- Kubernetes APIではなく、PodのファイルやPodの環境変数を通じて利用したい
- ファイルが更新された時、Deploymentなどを使ってローリングアップデートを実施したい

例えば、`kubectl get <CR名>`で情報を取得したり、宣言的なマニフェストを利用してReconcile処理を走らせたりしたいときなど、Kubernetesの流儀に沿ってKubernetesを拡張したいケースにおいてCRを利用すると相性が良い。Argo CDはまさにKubernetes上で動くKubernetes用のデプロイツールなので、相性が良いケースである。

以上
