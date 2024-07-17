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

Startup probeは、Podの初回起動時のみに利用するProbeである。起動が遅いアプリなどに使用することが想定される。Startup probeはK8s version 1.18から導入された機能なので、それいぜんはReadiness probeやLiveness probeのinitialDelaySecondsで代替していた。

マニフェストは、Readiness/Liveness probeとほぼ同じ。

```yaml
startupProbe:
  httpGet:
    path: /healthz
    port: 8080
  failureThreshold: 30
  periodSeconds: 10
```

この設定では最大30秒 x 10回 = 300秒コンテナの起動を待つ設定となる。

### STATEはRunningだが、、

Readiness probeとLiveness probeを使ったハンズオンをやってみる。

```zsh
> kubectl apply --filename chapter-07/deployment-destruction.yaml --namespace default
deployment.apps/hello-server created
```

Podが作成できているか確認する。

```zsh
> kubectl get pod --namespace default
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-5bc7ccb8dd-7wlh4   1/2     Running   0          5s
hello-server-5bc7ccb8dd-qxrp4   1/2     Running   0          5s
hello-server-5bc7ccb8dd-xwnls   1/2     Running   0          5s
```

コンテナのSTATUSはRunningになっているようだ。問題無いように見えるが、READYが1/2になっている。これはPod内にコンテナが2つあるうちの1つがREADYで、もう1つはREADYでは無いということを示している。Pod内にコンテナが複数あることを知らず、1つ以上READYになっていれば良いと勘違いすることがあるので、気をつけてみること。

再度、時間をおいて確認する。

```zsh
> kubectl get pod --namespace default
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-5bc7ccb8dd-7wlh4   1/2     Running   0          7m38s
hello-server-5bc7ccb8dd-qxrp4   1/2     Running   0          7m38s
hello-server-5bc7ccb8dd-xwnls   1/2     Running   0          7m38s
```

さらに詳しく見ていく。表示されたPodのうち適当にPod名を1つ確認する。

```yaml
> kubectl describe pod hello-server-5bc7ccb8dd-xwnls --namespace default
Name:             hello-server-5bc7ccb8dd-xwnls
Namespace:        default
Priority:         0
Service Account:  default
Node:             kind-control-plane/172.18.0.2
Start Time:       Wed, 17 Jul 2024 20:43:05 +0900
Labels:           app=hello-server
                  pod-template-hash=5bc7ccb8dd
Annotations:      <none>
Status:           Running
IP:               10.244.0.24
IPs:
  IP:           10.244.0.24
Controlled By:  ReplicaSet/hello-server-5bc7ccb8dd
Containers:
  hello-server:
    Container ID:   containerd://f5be51ec53894f425caefb070a24e5173511a71f8289207515ca12245a953368
    Image:          blux2/hello-server:1.6
    Image ID:       docker.io/blux2/hello-server@sha256:035c114efa5478a148e5aedd4e2209bcc46a6d9eff3ef24e9dba9fa147a6568d
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Wed, 17 Jul 2024 20:43:05 +0900
    Ready:          False
    Restart Count:  0
    Liveness:       http-get http://:8080/health delay=10s timeout=1s period=5s #success=1 #failure=3
    Readiness:      http-get http://:8081/health delay=5s timeout=1s period=5s #success=1 #failure=3
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-8gpsh (ro)
  busybox:
    Container ID:  containerd://7896c4ec0936284be8f057f821ecc823a90640f2ec469206c27e9facc3244261
    Image:         busybox:1.36.1
    Image ID:      docker.io/library/busybox@sha256:9ae97d36d26566ff84e8893c64a6dc4fe8ca6d1144bf5b87b2b85a32def253c7
    Port:          <none>
    Host Port:     <none>
    Command:
      sleep
      9999
    State:          Running
      Started:      Wed, 17 Jul 2024 20:43:06 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-8gpsh (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       False 
  ContainersReady             False 
  PodScheduled                True 
Volumes:
  kube-api-access-8gpsh:
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
  Type     Reason     Age                     From               Message
  ----     ------     ----                    ----               -------
  Normal   Scheduled  9m35s                   default-scheduler  Successfully assigned default/hello-server-5bc7ccb8dd-xwnls to kind-control-plane
  Normal   Pulled     9m35s                   kubelet            Container image "blux2/hello-server:1.6" already present on machine
  Normal   Created    9m35s                   kubelet            Created container hello-server
  Normal   Started    9m35s                   kubelet            Started container hello-server
  Normal   Pulled     9m35s                   kubelet            Container image "busybox:1.36.1" already present on machine
  Normal   Created    9m35s                   kubelet            Created container busybox
  Normal   Started    9m34s                   kubelet            Started container busybox
  Warning  Unhealthy  4m30s (x65 over 9m30s)  kubelet            Readiness probe failed: Get "http://10.244.0.24:8081/health": dial tcp 10.244.0.24:8081: connect: connection refused
```

Readiness probeがFailしていることが確認できる。また、hello-serverとbusyboxのコンテナでそれぞれ異なるステータツそなっている。hello-serverはState: Running, Ready: FalseでbusyboxはState: Running, Ready: Trueになっている。これで失敗しているコンテナがhello-serverだということがわかる。

では、Readiness probeの設定内容を確認する。

```yaml
> cat chapter-07/deployment-destruction.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-server
  labels:
    app: hello-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello-server
  template:
    metadata:
      labels:
        app: hello-server
    spec:
      containers:
      - name: hello-server
        image: blux2/hello-server:1.6
        ports:
        - containerPort: 8080
        readinessProbe:
          httpGet:
            path: /health
            port: 8081
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
      - name: busybox
        image: busybox:1.36.1
        command:
        - sleep
        - "9999"
```

コンテナのポート(8080)とReadiness probe(8081)のポートが異なることは問題ないが、Liveness probe(8080)のポートはコンテナのポートと同じである。Readiness probeだけポートが異なるのが気になることがわかる。

Podのログを確認する。

```zsh
> kubectl logs hello-server-5bc7ccb8dd-xwnls --namespace default
Defaulted container "hello-server" out of: hello-server, busybox
2024/07/17 11:43:05 Starting server on port 8080
2024/07/17 11:43:15 Health Status OK
2024/07/17 11:43:20 Health Status OK
2024/07/17 11:43:25 Health Status OK
2024/07/17 11:43:30 Health Status OK
2024/07/17 11:43:35 Health Status OK
2024/07/17 11:43:40 Health Status OK
2024/07/17 11:43:45 Health Status OK
2024/07/17 11:43:50 Health Status OK
```

ログを参照するとヘルスチェックが定期的に実行されていることがわかる。これはLiveness probeの設定によるものである。Readiness probeのポート番号が間違っている可能性を疑う。

実装を確認する。

```go
> cat ~/Develop/GitHub/bbf-kubernetes/hello-server/main.go
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

環境変数PORTが設定されていなければ、8080ポートを使用するようだ。

マニフェストを見た結果では、環境変数でPORTは指定されていない。そのため、8080ポートなっている。これはLiveness probeが使用している8080ポートは問題ないことからもわかる。

よって、8081ポートは誤りとわかるので、8080ポートに修正を行う。

```zsh
> kubectl edit deployment --namespace default
deployment.apps/hello-server edited
```

しばらくすると、全てRunning, READY 2/2となっていることを確認できる。

```zsh
> kubectl get pod --namespace default
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-64bf989855-5bxzn   2/2     Running   0          43s
hello-server-64bf989855-7nx64   2/2     Running   0          54s
hello-server-64bf989855-sf2t2   2/2     Running   0          48s
```

最後に掃除をする。

```zsh
> kubectl delete --filename chapter-07/deployment-destruction.yaml --namespace default
deployment.apps "hello-server" deleted
```

## アプリに適切なリソースを指定する
