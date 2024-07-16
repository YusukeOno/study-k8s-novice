# 安全なステートレス・アプリケーションを作るには

## アプリのヘルスチェックを行う

K8sはアプリのヘルスチェックを行い、異常の時は自動でServiceやPodを制御する仕組みがある。

3つのProbeはそれぞれ用途に合った使い方をすると、とても強力に働いてくれる。

- Readiness probe
- Liveness probe
- Startup probe

### Readiness probe

コンテナが起動することと、とら義っくが受けられる状態になることは必ずしもイコールではない。

コンテナがReadyになるまでの時間やエンドポイントを制御するのがReadiness probeである。

```yaml
> cat chapter-07/pod-readiness.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: httpserver
  name: httpserver-readiness
spec:
  containers:
  - name: httpserver
    image: blux2/delayfailserver:1.1
    readinessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

このマニフェストでは`/healthz`エンドポイントの8080ポートに対して、5秒に一回ヘルスチェック用リクエストを送る設定になっている。

また、initialDelaySecondsは最初のProbeが実施されるまでに5秒待つという設定になっている。リクエストのHTTPレスポンスが200以上400未満は成功とみなされ、それ以外は失敗とみなされる。

このマニフェストではHTTPリクエストを利用しているが、コマンドを実行したりTCPソケットを使用するProbeを設定することも可能である。

Readinessと名前についているが、コンテナ起動時のみならずPodのライフサイクル全てにおいて、このProbeは有効である。

Readiness probeが失敗すると、Serviceリソースの接続対象から外され、トラフィックを受けなくなる。そのため、適切にモニタリングをしないとPodの数が減っていることに気づかないかもしれない。

まずは、マニフェストを適用する。

```zsh
> kubectl apply --filename chapter-07/pod-readiness.yaml --namespace default
pod/httpserver-readiness created
```

今回利用したコンテナイメージは起動10秒後にヘルスチェックでエラーを返すようになっている。

```zsh
> kubectl get pod --watch --namespace default
NAME                   READY   STATUS    RESTARTS   AGE
httpserver-readiness   0/1     Running   0          1s
httpserver-readiness   1/1     Running   0          5s
httpserver-readiness   0/1     Running   0          25s
```

最初はREADY 1/1だったが、READY 0/1になった。logを参照すると、ヘルスチェックが途中でエラーになったことがわかる。

```zsh
> kubectl logs httpserver-readiness --namespace default
2024/07/16 11:57:51 Starting server...
2024/07/16 11:57:56 Health Check: OK
2024/07/16 11:58:01 Health Check: OK
2024/07/16 11:58:06 Error: Service Unhealthy
2024/07/16 11:58:11 Error: Service Unhealthy
2024/07/16 11:58:16 Error: Service Unhealthy
```

ヘルスチェックがエラーになったことでコンテナがREADYカウントに含まれなくなったことがわかる。また、設定通り、5秒ごとにヘルスチェックが行われていることがわかる。

最後に掃除をする。

```zsh
> kubectl delete --filename chapter-07/pod-readiness.yaml --namespace default
pod "httpserver-readiness" deleted
```
