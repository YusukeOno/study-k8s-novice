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

 Traces（分散トレーシング）はユーザー内しアプリの通信を追跡するための概念。また、Tracesは複数のSpan（操作の集合）から成る。コンテナを複数稼働させている状況で環境障害が発生した時、ユーザーからのリクエストがどのコンテナを経由したかを正確に辿る必要が出てきた時に利用できる。

 また、Tracesを導入することで「どのリクエストにどれくらい時間がかかったか」を細かく可視化できるので、パフォーマンス改善・チューニングにも利用できる。

しかし、これまでのログやメトリクスと異なり、どこかのコンポーネント・インフラだけがトレーシングを実現しているのでは意味がない。正確にアクセスを追跡するためには、リクエストが通る経路上の全てでTraces/Spanが導入できている必要がある。そのため、ログやメトリクスに比べて導入のハードルは高いかもしれない。

Tracesを導入するためにはアプリケーションの実装に手を入れる必要がある。どこからどこまでのリクエストをTracesとして扱い、Spanとして扱うか。これらを考えた上でTraces/Spanを表現する実装を入れる必要がある

Tracesを導入するためにアプリケーションで実装を追加する必要があるが、実装を簡易化するためのOSSライブラリがある。

![OpenTelemetry](https://opentelemetry.io/)といい、Traces以外にもメトリクスやログなどのデータの標準仕様を定めたり、ツールを提供していたりするCNCFのプロジェクトである。

また、実装したTracesを収集する代表的なOSSとして、![Jaeger](https://www.jaegertracing.io/)や![Grafana Tempo](https://grafana.com/oss/tempo/)が挙げられる。

## モニタリングについて

### 情報を可視化する：ダッシュボード

オブザーバビリティはデータを収集しただけでは実現できない。観測できるようにするためには、可視化をする必要がある。とくにMetricsやTracesはある一点のデータだけでわかることは少なく、全体や、ある一定の時間軸の中で見ていくことが大事でさる。可視化をするために、ダッシュボードを作成する。障害調査に利用するだけでなく、障害を起こさないために、あるいは性能劣化や異常を見つけるために利用する。

### 異常を知らせる：アラート

観測可能な状態ができたら、異常が起きたことがわかるように、アラートの設定も行う。アラートにはMetricsを使うと良いだろう。アラートの設定の仕方、監視をするためのノウハウに関しては一つの本が出ているくらいさまざまなものがある。ダッシュボードで観測できた値を基にまずは設定してみて、継続的に改善していくと良い。

## モニタリング用システム構築

ここではPrometheus、そしてGrafanaを使ってモニタリング用のシステムを構築してみる。

### Prometheus/Grafanaをインストールする

`kubectl get pod --namespace monitoring`を実行し、すべてのPodがRunningになっていることを確認する。

```zsh
> kubectl get pod --namespace monitoring
NAME                                                        READY   STATUS    RESTARTS        AGE
alertmanager-kube-prometheus-stack-alertmanager-0           2/2     Running   8 (3h6m ago)    16d
kube-prometheus-stack-grafana-76b485bc4c-dqq92              3/3     Running   12 (3h6m ago)   16d
kube-prometheus-stack-kube-state-metrics-7db65fb76b-bhq6m   1/1     Running   8 (3h6m ago)    16d
kube-prometheus-stack-operator-7566b999fc-bxgml             1/1     Running   8 (3h6m ago)    16d
kube-prometheus-stack-prometheus-node-exporter-bsw9h        1/1     Running   4 (3h6m ago)    16d
prometheus-kube-prometheus-stack-prometheus-0               2/2     Running   8 (3h6m ago)    16d
```

### メトリクスを収集するアプリケーションを起動する

オブザーバビリティはObserveする対象が存在して初めて成り立つ。ということで、まずはアプリを立ち上げよう。これまでにも登場したhello-serverアプリを使う。これまでと一点だけ異なるのは、メトリクス用のエンドポイントを追加することだ。

Prometheusは`/metrics`というエンドポイントに対してアクセスし、メトリクスを収集する。`/metrics`でどのようなメトリクスを収集可能にするかはアプリケーション開発者の自由である。今回はPrometheusのGo用ライブラリを利用し、Goに関連するメトリクスを収集可能にする。

実装は次のとおりライブラリの追加と、metricsエンドポイントの追加のみ。

```golang
package main

import (
	"fmt"
	"github.com/prometheus/client_golang/prometheus/promhttp"
	"log"
	"net/http"
	"os"
)

func main() {
	port := os.Getenv("PORT")
	if port == "" {
		port = "8080"
	}

	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		if r.URL.Path != "/" {
			http.NotFound(w, r)
			return
		}
		fmt.Fprintf(w, "Hello, world! Let's learn Kubernetes!")
	})

	http.HandleFunc("/healthz", func(w http.ResponseWriter, r *http.Request) {
		if r.URL.Path != "/healthz" {
			http.NotFound(w, r)
			return
		}
		w.WriteHeader(http.StatusOK)
		fmt.Fprintf(w, "OK")
		log.Printf("Health Status OK")
	})

	http.Handle("/metrics", promhttp.Handler())

	log.Printf("Starting server on port %s\n", port)
	err := http.ListenAndServe(":"+port, nil)
	if err != nil {
		log.Fatal(err)
	}

}
```

Prometheusでメトリクスを収集しやすくするために、Serviceリソースも追加している。

```zsh
> kubectl apply --filename chapter-11/namespace.yaml
namespace/develop created

> kubectl apply --filename chapter-11/hello-server.yaml
deployment.apps/hello-server created
service/hello-server created
```

Podが三つ作成できていれば準備完了。

```zsh
> kubectl get pod --namespace develop
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-5ffcf97c8b-m7bft   1/1     Running   0          44s
hello-server-5ffcf97c8b-mcd68   1/1     Running   0          44s
hello-server-5ffcf97c8b-xj7hv   1/1     Running   0          44s
```

### メトリクスを収集するための設定を行う

PrometheusはPull型のアーキテクチャをとっているため、Prometheus側に「何を収集するか」の設定を書く必要がある。

今回は次のマニフェストを利用して収集を行う。

```yaml
> cat ./chapter-11/kube-prometheus-stack/values.yaml
prometheus:
  prometheusSpec:
    additionalScrapeConfigs:
      - job_name: hello-server
        scrape_interval: 10s
        static_configs:
        - targets:
          - hello-server.develop.svc.cluster.local:8080
```

設定ファイルを利用してhelm upgradeを行う。

```zsh
> helm upgrade kube-prometheus-stack -f chapter-11/kube-prometheus-stack/values.yaml prometheus-community/kube-prometheus-stack --namespace monitoring
Release "kube-prometheus-stack" has been upgraded. Happy Helming!
NAME: kube-prometheus-stack
LAST DEPLOYED: Fri Aug 30 22:27:37 2024
NAMESPACE: monitoring
STATUS: deployed
REVISION: 2
NOTES:
kube-prometheus-stack has been installed. Check its status by running:
  kubectl --namespace monitoring get pods -l "release=kube-prometheus-stack"

Visit https://github.com/prometheus-operator/kube-prometheus for instructions on how to create & configure Alertmanager and Prometheus instances using the Operator.
```

### Prometheusにアクセスする

PrometheusにGUIがついているので、アクセスしてみよう。まずはport-forwardを行う。

```zsh
> kubectl port-forward service/kube-prometheus-stack-prometheus --namespace monitoring 9090:9090
Forwarding from 127.0.0.1:9090 -> 9090
Forwarding from [::1]:9090 -> 9090
```

ブラウザで`http://localhost:9090`にアクセスする。

次は、実際のメトリクスを見てみよう。Prometheusでもメトリクスを参照できる。

1. メインページに戻る
2. 検索窓に`go_gc_duration_seconds{job="hello-server"}`と入力する
3. Executeボタンを押して検索する

hello-serverのgo_gc_duration_secondsメトリクスの値を取得できた。

![graph](./ScreenShot%202024-08-30%2022.34.40.png)

### Grafanaにアクセスする

続いてGrafanaにアクセスする。port-forwardを行う。

```zsh
kubectl port-forward service/kube-prometheus-stack-grafana --namespace monitoring 8080:80
Forwarding from 127.0.0.1:8080 -> 3000
Forwarding from [::1]:8080 -> 3000
```

ブラウザで `http://localhost:8080/` を開く。

ログイン画面が表示されたら、次の入力を行い、ログインする。

```plaintext
username: admin
password: prom-operator
```

ログインできたら、メトリクスを見てみる。左側のハンバーガーメニューからExploreを開く。

Metrics欄に`go_gc_duration_seconds`を、Select labelにjob、Select valueにhello-serverを入れると、クエリが出力される。

最後に右上の「Run query」ボタンをクリックしてグラフを表示する。

これだけでは、Prometheusでできたことと同じなので、ダッシュボードも作ってみる。

ダッシュボードを作ることで必要なクエ紙を保存してグラフ表示を固定できる。障害調査時に毎回クエリを実行するのではなく、関連するメトリクスをダッシュボードに保存しておき、パッと出せると良い。

また、メトリクスに詳しくない人に共有するという使い方もできる。Explorerの画面からダッシュボードを作成するのは簡単。上の方にある「Add to dashboard」をクリックし、「Open dashboard」クリックする。

まだグラフが一つしかないが、これがダッシュボードになる。また、アラートもGrafana上から設定できる。ハンバーガーメニューからAlertingを設定する。Alertingの設定方法が書かれた画面が出てくる。

左のメニューからAlert rulesをクリックすると、デフォルトで設定されているアラートルールが一覧表示されている。

以上
