# Kubernetesとは

K8sとはOpen-source system for automating deployment, scaling, and managmement of containerized applications.

多くのコンテナを運用しようとすると、次の問題に直面する。

* 障害発生時に、各コンテナの設定・復旧をするのが大変
* コンテナの仕様を個々に管理するのが大変
* サーバが複数台あるときに、どのサーバでコンテナを起動させるべきかを決めるのが大変

ただし、コンテナを使っているからといって必ずしもK8sが最適な手段とは限らない。

また、アプリサーバをコンテナ化しただけで、コンテナの使い方仮想マシン時代と変わらないケースも見受けられる。

## Features of K8s

K8sの特徴を挙げる。

* Reconciliation Lopp（調整ループ）によって障害から自動復旧するように試みる
* YAMLファイルをし利用して各設定を管理できる
* K8sのAPIでインフラレイヤが中酥油化されており、サーバ固有の設定を知る必要がない

## Architecture of K8s

![over view](./overview.drawio.svg)

Control PlaneはWorker Nodeに直接指示しない。Woker NodeがControl Planeに問い合わせる方式を取ることで、Control Planeが壊れても、即座にWorker Node上に起動するコンテナが破壊されるわけではない。

## Install kubectl

インストール

```zsh
brew install kind
```

クラスタを作成する

```zsh
kind create cluster --image=kindest/node:v1.29.0
```

クラスタと接続できることを確認する

```zsh
kubectl cluster-info --context kind-kind
```

`--context`オプションは、利用するクラスタのコンテキストを指定する。1つのクラスタしか利用しない場合は、省略可能。

## Configuration kubectl

kubectlのconfig情報は、`~/.kube/config`に記載されている。

`--kubeconfig`フラグや`KUBECONFIG`環境変数を利用して設定することも可能。

設定される順番は以下。

1. `--kubeconfig`フラグ
2. `KUBECONFIG`環境変数
3. `~/.kube/config`ファイル

## Kubernetesクラスタを消す

```zsh
kind delete cluster
```

