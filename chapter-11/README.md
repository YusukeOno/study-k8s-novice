# オブザーバビリティとモニタリングに触れてみる

## オブザーバビリティについて

システムが「観測可能である」状態にするにはどうすれば良いか。

CNCFのホワイトペーパーではシステムの出力シグナルと呼び、好みのシグナルを一つ使うところから始めると良い、と書かれている。シグナルは3つのプライマリシグナルと二つの新規シグナルが紹介されており、プライマリシグナルから使うと良いとも書かれている。

3つのプライマリシグナル

1. Logs
2. Metrics
3. Traces

2つの新規シグナル

1. Profiles
2. Dumps

### 情報を収集する：Logs

Kubernetesではデフォルトでログを収集する仕組みがある。コンテナの標準出力/エラー（Stdout/Stderr）の内容をコンテナのログとして収集する。これまで`kuberctl logs`を使ってきたと思うが、これはKubernetesのデフォルトにあるログ収集の仕組みを使っている。

しかし、このデフォルトの仕組みを利用したログはPodがNodeから削除されると消えてしまう。そのため、本番運用を行うにあたってログを永続化する仕組みを入れる必要がある。KubernetesのPersisstent Volumesを利用して保存しておいても良いが、外部に転送して保存しておくこともできる。

ログを外部に転送するためによく利用されるのは、FluentdやFluentbitがある。また、ログを収集・検索可能にするOSSとしてGrafana Lokiが挙げられる。

### 測定値を処理する:Metrics

メトリクスは測定値の集合である。測定を利用した投擲的な処理をしたり、傾向を見たい場合に活用できたりする。ログは多くの情報を取得できるが、その分データ容量が必要になる。また、「一日に何件アクセスがあったか」を調べるためにログファイルを開いて件数を数えるのはあまり現実的ではない。そこで必要になるのがメトリクスである。

メトリクスもログ同様にクラウドベンダー標準機能を利用することもできるが、さらなる高機能を求めて外部クラウドサービスを利用するケースも多く見られる。Kubernetesは標準でメトリクスを収集する機能はない。そのため、メトリクスを収集するためには外部サービスを利用する必要がある。

メトリクスを収集するためによく利用されるOSSにPrometheusが挙げられる。独自のPromQLというクエリ言語を利用して収取したメトリクスを参照することが可能。

### 通信を追跡する:Traces