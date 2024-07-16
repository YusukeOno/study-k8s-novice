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

### Liveness probe

Libeness probeはReadiness probeと似ているが、Probeが失敗した時の挙動が変わる。Readiness probeはServiceから接続を外すのに対して、Liveness probeはPodを再起動する。これは「Podがハングしてしまい、再起動で直る」といったケースが想定される場合に有効である。逆にLiveness probeは再起動を無限に繰り返してしまうリスクがある。

マニフェストを見ると、Readiness probeとほとんど同じになる。

```yaml
> cat chapter-07/pod-liveness.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: httpserver
  name: httpserver-liveness
spec:
  containers:
  - name: httpserver
    image: blux2/delayfailserver:1.1
    livenessProbe:
      httpGet:
        path: /healthz
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
```

Liveness probeとReadiness probeを同時に設定することは可能である。ただし、Liveness probeはReadiness probeを待つといった挙動はしないため、Readinessを先に実行したい場合はinitialDelaySecondsを調整するか、後述するStartup probeを使用する必要がある。

一般にはReadiness probeが先に実行されることが推奨される。

では、Liveness probeを確認する。

```zsh
> kubectl apply --filename chapter-07/pod-liveness.yaml --namespace default
pod/httpserver-liveness created
```

Podをwatchする。Readiness Probeで使用したイメージと同様、10秒後にヘルスチェックがエラーを返す。

```zsh
> kubectl get pod --watch --namespace default
NAME                  READY   STATUS    RESTARTS     AGE
httpserver-liveness   1/1     Running   1 (2s ago)   27s
httpserver-liveness   1/1     Running   2 (1s ago)   51s
httpserver-liveness   1/1     Running   3 (1s ago)   76s
httpserver-liveness   1/1     Running   4 (1s ago)   101s
httpserver-liveness   0/1     CrashLoopBackOff   4 (0s ago)   2m5s
httpserver-liveness   1/1     Running            5 (51s ago)   2m56s
httpserver-liveness   0/1     CrashLoopBackOff   5 (0s ago)    3m20s
httpserver-liveness   1/1     Running            6 (84s ago)   4m44s
httpserver-liveness   0/1     CrashLoopBackOff   6 (0s ago)    5m5s
```

CrashLoopBackOffが発生し、RESTARTSの回数が次第に増えていることがわかる。このようにLiveness probeに失敗すると、原因が解消されるまで再起動を繰り返す。再起動で治るようなケースであれば良いのだが、アプリを修正しないと治らないバグの場合、修正がデプロイされるまで再起動し続ける。また、Readiness probeを設定しないケースや、Readiness probeとLiveness probeで設定内容が異なるケースでは要注意である。

このようなケースでは、今回行ったようにSTATUSはRunningなので、kubectl get lodしただけではLiveness Probeが失敗していることに気づけないこともある。Liveness probeは慎重に設定を行う必要がある。

PodをdescribeするとprobeがFailしていることが確認できる。RESTARTの値が以上だと感じた時には見てみることをお勧めする。

```yaml
> kubectl describe pod httpserver-liveness --namespace default
Name:             httpserver-liveness
Namespace:        default
Priority:         0
Service Account:  default
Node:             kind-control-plane/172.18.0.2
Start Time:       Tue, 16 Jul 2024 21:07:09 +0900
Labels:           app=httpserver
Annotations:      <none>
Status:           Running
IP:               10.244.0.19
IPs:
  IP:  10.244.0.19
Containers:
  httpserver:
    Container ID:   containerd://8f66fc0f5fa58a56f679419fd2072194da7bf5488420df4e56a51ba9e85c5b65
    Image:          blux2/delayfailserver:1.1
    Image ID:       docker.io/blux2/delayfailserver@sha256:84c46dd90117eda4f2545504e8ce9b2e595eef9fedb02aa2e0dcaa0c13cfeba0
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Tue, 16 Jul 2024 21:14:58 +0900
    Last State:     Terminated
      Reason:       Error
      Exit Code:    2
      Started:      Tue, 16 Jul 2024 21:11:53 +0900
      Finished:     Tue, 16 Jul 2024 21:12:14 +0900
    Ready:          True
    Restart Count:  7
    Liveness:       http-get http://:8080/healthz delay=5s timeout=1s period=5s #success=1 #failure=3
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-7964n (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       True 
  ContainersReady             True 
  PodScheduled                True 
Volumes:
  kube-api-access-7964n:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age                    From               Message
  ----     ------     ----                   ----               -------
  Normal   Scheduled  8m6s                   default-scheduler  Successfully assigned default/httpserver-liveness to kind-control-plane
  Normal   Pulled     6m51s (x4 over 8m6s)   kubelet            Container image "blux2/delayfailserver:1.1" already present on machine
  Normal   Created    6m51s (x4 over 8m6s)   kubelet            Created container httpserver
  Normal   Started    6m51s (x4 over 8m6s)   kubelet            Started container httpserver
  Normal   Killing    6m51s (x3 over 7m41s)  kubelet            Container httpserver failed liveness probe, will be restarted
  Warning  Unhealthy  3m1s (x21 over 7m51s)  kubelet            Liveness probe failed: HTTP probe failed with statuscode: 503
```

### Startup probe

Startup probeは、Podの初回起動時のみに利用す流Probeである。

