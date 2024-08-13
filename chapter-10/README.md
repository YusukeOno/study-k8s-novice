# K8sの開発ワークフローを利用する

## Kubernetesにデプロイする

これまでのハンズオンでK8sにデプロイするために`kubectl apply --file name`を実施してきたが、継続的にデプロイするためには次の課題がある。

- いつ誰がコマンドを実施したかわからない
- コマンドの実施により、マニフェストの衝突が起きてしまう
- 毎回手動で実施するのでは、手間がかかる。さらにヒューマンエラーも起きやすい

この課題を解決するためのデプロイ手法として、K8sを利用するケースでは大きく分けてCIOpsとGitOpsがある。いずれもGitHubなど共有リポジトリを利用することでマニフェストの衝突・差分の管理を行なっている。

### Push型のデプロイ方法: CIOps

`kubectl apply --filename <fileName>`を自動化しようとした時に、例えば「mainブランチにfeatureブランチがマージされたら本番環境にkubectl applyを実行する」ようなケースが最初に思いつく。この自動化はCIツールで実行すると思うが、これをCIOpsと呼ぶ。

CIOpsはわかりやすく、かつ実現しやすい手法である。デメリットとしてCI/CD用のツールに強い権限が必要であることや、デプロイ用のスクリプトが長く複雑になりがちということが挙げられる。

### Pull型のデプロイ方法： GitOps

Gitを使っていれば、GitOpsなのでは、と誤解されることも多いが、Gitは直接関係ない。具体的には次の4つの定義に基づいている。

1. 宣言的：システム全体は宣言的に記述される必要がある。
2. バージョン管理と不変：正規望ましいシステムの状態はバージョン管理されている。
3. 自動的に取得：承認された変更はシステムに自動的に適用される
4. 継続的な調整：ソフトウェアエージェントが正確さを保証し、乖離があった場合にアラートを出す。

ここではKubernetesのワークフローを実装するにあたって、大事なポイントを説明する。

多くの開発者はCIOpsから入ると思うが、GitOpsを選択する理由はなんだろうか。4つの定義のうち「3.自動的に取得（原文：Pulled automatically）」がCIOpsとGitOpsの大きな違いの一つである。よく「Push型」「Pull型」と言われるが、CIOpsがPush型で、GitOpsがPull型である。その名の通り、Push操作が行われたタイミングで動作するのがPush型、一定間隔で対象をPullし、動作する必要がある内容をPullしたタイミングで動作するのがPull型である。Pull型であることで、次のようなメリットを得られる。

#### メリット１：セキュリティリスクを低減できる

K8sクラスタにマニフェストを適用するという性質上、書き込み権限が必要になる。このため、書き込み権限を持つ認証情報が盗まれたり、CIが乗っ取られたりする場合は読み書きが可能になってしまうということになる。GitOpsはPull型という性質上、読み取り権限さえあれば実現可能である。認証情報が盗まれたとしても、書き込み権限が無い分、CIOpsよりは被害が抑えられるだろう。

#### メリット２：CIとCDが分離できる

CIOpsではCIもCDもCIツールを用いるため、実行タイミング・実行スクリプトなどが一緒になってしまうこともある。サービスやデプロイの単位・範囲が地裁時は特に問題に感じないかもしれないが、規模が大きくなってくると、「CIが重くてデプロイが遅い」「CI＆CDスクリプトが長大になり、メンテナンスができない」という状態になってしまうことがある。GitOpsでは専用のデプロイツールを利用し、さらにデプロイの情報を宣言的に管理することでCIと分離することが可能。

また、これはメリット１にも通じる話だが、CIとCDが分離できるので、CI用の権限を持つ人（ツール）とCD用の権限を持つ人（ツール）を分けて管理できるようになる。

これらのメリットはどんな環境でも享受できるわけではない。特に小さい規模でK8sを利用している場合、メリットよりもデメリットが大きくなるだろう。とはいえ、いつか規模が大きくなることを見越してGitOpsについて知っておくと良いだろう。

GitOpsは概念だが、ではどのようにしてワークフローに組み込むのだろうか？GitOpsを実現するためのソフトウェアがいくるかOSSとして公開されている。

#### Argo CD

`https://argoproj.github.io/cd/`

Argo CDではApplicationという名前のCustom Resourceを利用し、「どのリポジトリの」「どのマニフェストの」「どのバージョン（例：ブランチ）の」マニフェストを「どの環境に」適用するかを指定します。

#### Spinnaker

`https://spinnaker.io/`

SpinnakerはKubernetes以外にも、主要なクラウドプロバイダに対応していることを売りにしている。そのため、Argo CDはKubernetesクラスタへのデプロイのみを扱うが、SpinnakerではDocker ImageのビルドなどCIパイプラインの構築もできる。

#### FluxCD

`https://fluxcd.io/`

Argo CDとかなり近いが、v2になってマルチテナンシーが違い。

## Kubernetesのマニフェスト管理

これまでのハンズオンで様々なマニフェストを扱ってきた。例えば、Staging環境とProduction環境でSecretの値だけが異なるような場合、ほぼにたマニフェストを書くことになる。

マニフェストの運用を続けていくと、マニフェストの管理・運用にコストがかかるようになってくる。マニフェストをよりわかりやすく管理するために、いろいろな方法・ツールがある。

### Helm

Helmはパッケージマネージャとなっており、マニフェストを作成する以上のことが可能。Chartというテンプレートをもとに、helm installすることでKubernetesクラスタにマニフェストをデプロイする仕組みなっている。

テンプレートに書かれていること以上のことをやりたくなった場合、別の方法を検討する必要がある。

Helmは他の開発者が開発したカスタムコントローラ用のマニフェストを利用したいケースで特に有効である。

Chapter 11で出てくるオブザーバビリティに関連するGrafanaというダッシュボードを表示するOSSをインストールする。

#### Helmをインストールする

`https://helm.sh/docs/intro/install/`

`brew install helm`

#### Helm Chart Repositoryを追加する

Helm Chartをインストールする前にHelm Chart Repositoryを追加する必要がある。次のコマンドでPrometheus用のリポジトリを追加する。

```zsh
> helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
"prometheus-community" has been added to your repositories

> helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "prometheus-community" chart repository
Update Complete. ⎈Happy Helming!⎈
```

#### インストール先のnamespaceを作成する

namespaceを作成しておく。

```zsh
> kubectl create namespace monitoring
namespace/monitoring created
```

#### helm installを実行する

```zsh
> helm install kube-prometheus-stack --namespace monitoring prometheus-community/kube-prometheus-stack
NAME: kube-prometheus-stack
LAST DEPLOYED: Wed Aug 14 07:10:59 2024
NAMESPACE: monitoring
STATUS: deployed
REVISION: 1
NOTES:
kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace monitoring get pods -l "release=kube-prometheus-stack"

Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator.
```

しばらくすると、幾つものPodが立ち上がっていることが確認できる。

```zsh
> kubectl get pod --namespace monitoring
NAME                                                        READY   STATUS    RESTARTS   AGE
alertmanager-kube-prometheus-stack-alertmanager-0           2/2     Running   0          2m54s
kube-prometheus-stack-grafana-76b485bc4c-dqq92              3/3     Running   0          3m41s
kube-prometheus-stack-kube-state-metrics-7db65fb76b-bhq6m   1/1     Running   0          3m41s
kube-prometheus-stack-operator-7566b999fc-bxgml             1/1     Running   0          3m41s
kube-prometheus-stack-prometheus-node-exporter-bsw9h        1/1     Running   0          3m41s
prometheus-kube-prometheus-stack-prometheus-0               2/2     Running   0          2m54s
```

ダッシュボードのログイン画面を表示してみる。自動生成されたServiceを使ってport-forwardを行う。

```zsh
> kubectl port-forward service/kube-prometheus-stack-grafana --namespace monitoring 8080:80
Forwarding from 127.0.0.1:8080 -> 3000
Forwarding from [::1]:8080 -> 3000
```

ブラウザで`http://localhost:8080`を実行すると、Grafanaのログイン画面が表示される。

Helm Chartはテンプレートだが、次のコマンドを打つとどのような値を設定してカスタマイズ可能かがわかる。

```yaml
> helm show values prometheus-community/kube-prometheus-stack
# Default values for kube-prometheus-stack.
# This is a YAML-formatted file.
# Declare variables to be passed into your templates.

## Provide a name in place of kube-prometheus-stack for `app:` labels
##
nameOverride: ""

## Override the deployment namespace
##
namespaceOverride: ""

以下略
```

ここで得られた値がデフォルトの設定となっている。このデフォルト設定を変更するためには変更する設定値を書いたvalues.yamlをローカルに保存し、helm intallの引数として設定する。こうすることで独自カスタマイズされた状態でCustom Controllerを環境にデプロイできる。

しかし、helm installコマンドを直接実行する方法はGitOpsと相性が悪い。そのため、Argo CDなど各GitOpsエージェントの仕様に従ってHelmインストールを行ったり、CIを利用して生成したマニフェストをGitOpsで管理したりするといった方法がある。

後者はhelm templateというローカルでテンプレートをレンダリングする、という方法が利用可能。

