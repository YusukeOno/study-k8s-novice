# アプリを直す

## 準備環境を作る

## アプリ環境を構築する

```zsh
> kubectl apply --filename chapter-08/hello-server.yaml --namespace default
deployment.apps/hello-server created
configmap/hello-server-configmap created
service/hello-server-external created
poddisruptionbudget.policy/hello-server-pdb created
```

Podが作成されていることを確認する。

```zsh
> kubectl get pod --namespace default
NAME                            READY   STATUS              RESTARTS   AGE
hello-server-7765f564df-j2cnq   0/1     ContainerCreating   0          17s
hello-server-7765f564df-ldfk2   0/1     ContainerCreating   0          17s
hello-server-7765f564df-tc4qt   0/1     ContainerCreating   0          17s
```

このマニフェストを利用して立ち上げたhello-serverはポート30599でアクセスできる。アプリに接続できることを確認する。次のコマンドでNodeのIPを取得する。

```zsh
> kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'
172.18.0.5
```

取得したInternalIPを利用してアクセスする。

```zsh
> curl localhost:30599
Hello, world! Let's learn Kubernetes!
```

## アプリを更新する

マニフェストを適用してアプリを更新する。

```zsh
> kubectl apply --filename chapter-08/hello-server-update.yaml --namespace default
deployment.apps/hello-server configured
configmap/hello-server-configmap configured
service/hello-server-external unchanged
poddisruptionbudget.policy/hello-server-pdb configured
```

## 正常性確認を行なってみる

問題なく更新できているか確認する。

```zsh
> curl localhost:30599
Hello, world! Let's learn Kubernetes!
```

アプリと疎通できているようだ。問題なくアプリの更新を行うことができたか、Deploymentのステータスを確認する。

```zsh
> kubectl get deployment --namespace default
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
hello-server   3/3     1            3           7h41m
```

UP-TO-DATEが1になっている。これはPodが1つ更新中ということを示している。続いて、ReplicaSetを確認する。

```zsh
> kubectl get replicaset --namespace default
NAME                      DESIRED   CURRENT   READY   AGE
hello-server-69fdcc8fb9   1         1         0       2m26s
hello-server-7765f564df   3         3         3       7h42m
```

AGEの若いReplicaSetのREADYが0で、古いReplicaSetのREADYが3になっている。どうやら古いバージョンのアプリが動き続けているだけのようだ。

ここからは次のことをゴールに原因調査と解消を行う。

- アプリにアクセスして「Hello, world! Let's build, break and fix」が返ってくる
- AGEの若いReplicaSetのREADYが3つになる
