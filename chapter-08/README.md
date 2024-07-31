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

