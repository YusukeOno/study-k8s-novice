# トラブルシューティングガイドとkubectlコマンドの使い方

K8sはリソースを作成する特徴上、「リソースの作成に失敗している」「依存する別のリソースが動いていない」など、障害ポイントは多岐にわたる。

そんなときに、kubectlを使って調査ができると、さまざまな状況でトラブルシューティングを可能にしてくれる。

## トラブルシューティングガイド

![over view](./Troubleshooting.drawio.svg)

### PodのSTATUSカラム

`kubectl get pod`で得られるSTATUSカラムには役立つ情報が出力される。

| Staus         | Note |
|---------------|------|
| Pending       | KubernetesクラスタからはPodの作成は許可されたものの、１つ以上のコンテナが準備中であることを意味している。Pod起動直後にこのSTAUSが表示されることがあるが、長時間このSTATUSである場合は異常を疑う。PodのEventsを参照し、ヒントが書かれていないか確認する。     |
| Runing        | Podがノードにスケジュールされ、すべてのコンテナが作成された状態。少なくとも１つのコンテナがまだ実行中、起動または再起動のプロセス中である。常時起動が想定されるPodであれば正常なSTATUSである。     |
| Completed     | Pod内のすべてのコンテナが完了した状態。再起動はされない。     |
| Unknown       | 何らかの理由でPodの状態が取得できなかったことを表している。このSTATUSは通常、Podが↑行されるべきノードとの通信エラーが原因で発生する。     |
| ErrImagePull  | Imageの取得に失敗したことを表している。PodのEventを参照し、ヒントが書かれていないか確認する。     |
| Error         | コンテナが異常終了したことを表している。Podのログを参照し、ヒントが書かれていないか確認する。     |
| OOMKilled     | コンテナOut Of Memory(OOM)で終了したことを表している。Podの使用リソースを増やす。     |
| Terminateting | Podが削除中の状態を表している。Terminatingを繰り返す場合は異常と考える。PodのEventsを参照し、ヒントが書かれていないか確認する。     |

## 現状を把握するためにkubectlコマンドを使う

### リソースを取得する : `kubectl get`

リソースの情報を取得する。`--namespace`オプションは省略可能。

`kubectl get pod --namespace default`

リソース名を指定して、特定のリソース情報のみを取得することも可能。

`kubectl get pod <Pod名> --namespace default`

IPアドレスやNode情報が取得できる`--output wide`

`kubectl get pod --output wide --namespace default`

YAMLファイル形式でリソースの情報を取得する`--output yaml`

`kubectl get pod myapp --output yaml --namespace default`

--outputでjsonpathでフィールドを指定してgetする  
マニフェストをJSON形式で出力し、欲しい情報を取得する。

`kubectl get pod <Pod名> --output jsonpath='{.spec.containers[].image}`

--vでkubectlの出力結果のログレベルを変更する  
`--v=7`で詳細なログレベルを指定できる。

`kubectl get pod <Pod名> --v=<ログレベル>`

