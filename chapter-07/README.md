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

アプリに適切なリソースを指定することは、安全にアプリを運用する上で大切である。特にK8sではリソースの指定方法によってスケジュールが変わるため、必ず指定するようにする。デフォルトで指定できるリソースはCPU・メモリ・Ephemeral Storage。ここではよく使うメモリとCPUについて説明する。

マニフェストは以下。

```yaml
> cat chapter-07/pod-resource-handson.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    app: hello-server
  name: hello-server
spec:
  containers:
  - name: hello-server
    image: blux2/hello-server:1.6
    resources:
      requests:
        memory: "64Mi"
        cpu: "10m"
      limits:
        memory: "64Mi"
        cpu: "10m"
```

### コンテナのリソース使用量を要求する：Resource requests

確保したいリソースの最低使用量を指定することができる。K8sのスケジューラはこの値を見てスケジュールするNodeを決めている。Requestsの値が確保できるNodeを調べ、該当するNodeにスケジュールする。どのNodeもRequestsに書かれている量が確保できなければ、Podがスケジュールされることはない。

コンテナごとにCPUとメモリのRequestsを指定することができる。

### コンテナのリソース使用量を制限する：Resource limits

コンテナが使用できるリソース使用量の上限を指定する。コンテナはこのLimitsを超えてリソースを使用することはできない。メモリが上限値を超える場合、Out Of Memory(OOM)でPodはkillされる。CPUが上限値を超えた場合、即座にPodがkillされることはないが、その代わりスロットリングが発生し、アプリの動作が遅くなる。

### リソースの単位

リソースの指定ができることはわかったが、単位は一体を何を意味しているのかを詳しく見ていく。

メモリ：単位を指定しない場合はバイトを意味する。他にもK、KiやM、Miなどをつけることができる、

CPU：単位を指定しない場合はCPUのコアを意味する。1mならば、0.001コアとなる。

### PodのQuality of Service(QoS) Classes

リソース設定に関連してQoSは覚えておいた方が良い。Nodeのメモリが完全に枯渇してしまうと、そのNodeに載っているすべてのコンテナが動作できなくなってしまうのを防ぐため、OOM Killerという役割のプログラムがいる。これは、QoSに応じてOOMKillするPodの優先順位を決定し、必要に応じて優先度の低いPodからOOMKillする。

QoSクラスにはGuaranteed、Burstable、BestEffortの3種類がある。BestEffort、Burstable、Guaranteedの順にOOM Killが発生する。

- Guaranteed: Pod内のコンテナ全てにリソースのRequestsとLimitsが指定されている。さらに、メモリのRequests=Limits、CPUもRequests＝Limitsとなる値が指定されている。
- Burstable: Pod内のコンテナのうち少なくとも1つはメモリまたはCPUのRequests/Limitsが指定されている。
- BestEffort: GuaranteedでもBurstableでもないもの。リソースに何も指定していない。

次のコマンドでPodを作成する。

```zsh
> kubectl apply --filename chapter-07/pod-resource-handson.yaml --namespace default
pod/hello-server created
```

次のコマンドでPodのQoSクラスを知ることができる。QoSクラスのどれに当たるかわからなくなった場合、直接確認すると良い。

```zsh
> kubectl get pod hello-server --output jsonpath='{.status.qosClass}' --namespace default
Guaranteed
```

掃除をする。

```zsh
> kubectl delete --filename chapter-07/pod-resource-handson.yaml --namespace default
pod "hello-server" deleted
```

### またしてもPodが壊れた

リソース周りの指定内容が理由でPodが動かなくなることはよくある。

```zsh
> kubectl apply --filename chapter-07/deployment-resource-handson.yaml --namespace default
deployment.apps/hello-server created
```

しばらく待ってもPodが全て起動しない。Podの状況を確認する。

```zsh
> kubectl get pod --namespace default
NAME                           READY   STATUS    RESTARTS      AGE
hello-server-9cbfdfd5c-2t4pt   0/1     Pending   0             10h
hello-server-9cbfdfd5c-56tfr   0/1     Pending   0             10h
hello-server-9cbfdfd5c-wck98   1/1     Running   1 (37s ago)   10h
```

何が起こっているのか、Podの詳細を見ていく。先ほど参照したPodのうちPendingになっているものを確認する。

```yaml
> kubectl describe pod hello-server-9cbfdfd5c-2t4pt --namespace default
Name:             hello-server-9cbfdfd5c-2t4pt
Namespace:        default
Priority:         0
Service Account:  default
Node:             <none>
Labels:           app=hello-server
                  pod-template-hash=9cbfdfd5c
Annotations:      <none>
Status:           Pending
IP:               
IPs:              <none>
Controlled By:    ReplicaSet/hello-server-9cbfdfd5c
Containers:
  hello-server:
    Image:      blux2/hello-server:1.6
    Port:       8080/TCP
    Host Port:  0/TCP
    Limits:
      cpu:     10m
      memory:  5Gi
    Requests:
      cpu:        10m
      memory:     5Gi
    Liveness:     http-get http://:8080/health delay=10s timeout=1s period=5s #success=1 #failure=3
    Readiness:    http-get http://:8080/health delay=5s timeout=1s period=5s #success=1 #failure=3
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-jldbg (ro)
Conditions:
  Type           Status
  PodScheduled   False 
Volumes:
  kube-api-access-jldbg:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Guaranteed
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  9h (x10 over 10h)  default-scheduler  0/1 nodes are available: 1 Insufficient memory. preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod.
  Warning  FailedScheduling  4m33s              default-scheduler  0/1 nodes are available: 1 Insufficient memory. preemption: 0/1 nodes are available: 1 No preemption victims found for incoming pod.
```

`FailedScheduling`となっていることがわかる。どうやら要求した量のメモリを割り当てられるNodeがなかったようだ。

では、Nodeはどのような設定になっているのか確認する。

```yaml
> kubectl describe node --namespace default
Name:               kind-control-plane
Roles:              control-plane
Labels:             beta.kubernetes.io/arch=arm64
                    beta.kubernetes.io/os=linux
                    kubernetes.io/arch=arm64
                    kubernetes.io/hostname=kind-control-plane
                    kubernetes.io/os=linux
                    node-role.kubernetes.io/control-plane=
Annotations:        kubeadm.alpha.kubernetes.io/cri-socket: unix:///run/containerd/containerd.sock
                    node.alpha.kubernetes.io/ttl: 0
                    volumes.kubernetes.io/controller-managed-attach-detach: true
CreationTimestamp:  Sun, 14 Jul 2024 09:24:24 +0900
Taints:             <none>
Unschedulable:      false
Lease:
  HolderIdentity:  kind-control-plane
  AcquireTime:     <unset>
  RenewTime:       Fri, 19 Jul 2024 07:13:24 +0900
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Fri, 19 Jul 2024 07:12:25 +0900   Sun, 14 Jul 2024 09:24:22 +0900   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Fri, 19 Jul 2024 07:12:25 +0900   Sun, 14 Jul 2024 09:24:22 +0900   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Fri, 19 Jul 2024 07:12:25 +0900   Sun, 14 Jul 2024 09:24:22 +0900   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            True    Fri, 19 Jul 2024 07:12:25 +0900   Sun, 14 Jul 2024 09:24:42 +0900   KubeletReady                 kubelet is posting ready status
Addresses:
  InternalIP:  172.18.0.2
  Hostname:    kind-control-plane
Capacity:
  cpu:                2
  ephemeral-storage:  102625208Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  hugepages-32Mi:     0
  hugepages-64Ki:     0
  memory:             6065108Ki
  pods:               110
Allocatable:
  cpu:                2
  ephemeral-storage:  102625208Ki
  hugepages-1Gi:      0
  hugepages-2Mi:      0
  hugepages-32Mi:     0
  hugepages-64Ki:     0
  memory:             6065108Ki
  pods:               110
System Info:
  Machine ID:                 991e7e917bbc491f80867172735e1983
  System UUID:                991e7e917bbc491f80867172735e1983
  Boot ID:                    d1c95a6e-6f57-4558-bf23-3d6f5b657dca
  Kernel Version:             6.6.30-0-virt
  OS Image:                   Debian GNU/Linux 11 (bullseye)
  Operating System:           linux
  Architecture:               arm64
  Container Runtime Version:  containerd://1.7.1
  Kubelet Version:            v1.29.0
  Kube-Proxy Version:         v1.29.0
PodCIDR:                      10.244.0.0/24
PodCIDRs:                     10.244.0.0/24
ProviderID:                   kind://docker/kind/kind-control-plane
Non-terminated Pods:          (10 in total)
  Namespace                   Name                                          CPU Requests  CPU Limits  Memory Requests  Memory Limits  Age
  ---------                   ----                                          ------------  ----------  ---------------  -------------  ---
  default                     hello-server-9cbfdfd5c-wck98                  10m (0%)      10m (0%)    5Gi (86%)        5Gi (86%)      10h
  kube-system                 coredns-76f75df574-62kgn                      100m (5%)     0 (0%)      70Mi (1%)        170Mi (2%)     4d21h
  kube-system                 coredns-76f75df574-ptcxn                      100m (5%)     0 (0%)      70Mi (1%)        170Mi (2%)     4d21h
  kube-system                 etcd-kind-control-plane                       100m (5%)     0 (0%)      100Mi (1%)       0 (0%)         4d21h
  kube-system                 kindnet-rbjnd                                 100m (5%)     100m (5%)   50Mi (0%)        50Mi (0%)      4d21h
  kube-system                 kube-apiserver-kind-control-plane             250m (12%)    0 (0%)      0 (0%)           0 (0%)         4d21h
  kube-system                 kube-controller-manager-kind-control-plane    200m (10%)    0 (0%)      0 (0%)           0 (0%)         4d21h
  kube-system                 kube-proxy-8r5n5                              0 (0%)        0 (0%)      0 (0%)           0 (0%)         4d21h
  kube-system                 kube-scheduler-kind-control-plane             100m (5%)     0 (0%)      0 (0%)           0 (0%)         4d21h
  local-path-storage          local-path-provisioner-6f8956fb48-bl257       0 (0%)        0 (0%)      0 (0%)           0 (0%)         4d21h
Allocated resources:
  (Total limits may be over 100 percent, i.e., overcommitted.)
  Resource           Requests      Limits
  --------           --------      ------
  cpu                960m (48%)    110m (5%)
  memory             5410Mi (91%)  5510Mi (93%)
  ephemeral-storage  0 (0%)        0 (0%)
  hugepages-1Gi      0 (0%)        0 (0%)
  hugepages-2Mi      0 (0%)        0 (0%)
  hugepages-32Mi     0 (0%)        0 (0%)
  hugepages-64Ki     0 (0%)        0 (0%)
Events:
  Type    Reason                   Age                    From             Message
  ----    ------                   ----                   ----             -------
  Normal  Starting                 6m5s                   kube-proxy       
  Normal  Starting                 6m11s                  kubelet          Starting kubelet.
  Normal  NodeHasSufficientMemory  6m11s (x8 over 6m11s)  kubelet          Node kind-control-plane status is now: NodeHasSufficientMemory
  Normal  NodeHasNoDiskPressure    6m11s (x8 over 6m11s)  kubelet          Node kind-control-plane status is now: NodeHasNoDiskPressure
  Normal  NodeHasSufficientPID     6m11s (x7 over 6m11s)  kubelet          Node kind-control-plane status is now: NodeHasSufficientPID
  Normal  NodeAllocatableEnforced  6m11s                  kubelet          Updated Node Allocatable limit across pods
  Normal  RegisteredNode           5m38s                  node-controller  Node kind-control-plane event: Registered Node kind-control-plane in Controller
```

今回のkindのクラスタでは1つのNodeにControl Planeも乗せているため、kube-apisserverなども同じNodeに乗っていることがわかる。

```plane text
  default                     hello-server-9cbfdfd5c-wck98                  10m (0%)      10m (0%)    5Gi (86%)        5Gi (86%)      10h
```

今回、applyしたマニフェストのコンテナのメモリが全体の86%使用していることがわかる。これでは、あと二つのPodは起動できない。Capacityに記載されている通り、このNodeは`memory: 6065108Ki`だと書かれている。

次のコマンドを実行してDeploymentで指定されているメモリを稼働しているリソースから取得する。

```zsh
> kubectl get deployment hello-server -o=jsonpath='{.spec.template.spec.containers[0].resources.requests}' --namespace default
{"cpu":"10m","memory":"5Gi"}
```

リソース指定量が十分かどうかは実環境の使用量や性能試験の結果などをもとにチューニングすべ気である。

では、次のようにメモリのrequestsとlimitsを64Miに変更しよう。

```diff
> kubectl diff --filename chapter-07/deployment-resource-handson.yaml --namespace default
diff -u -N /var/folders/c0/j8z9nmwj3r93swy5clqm0jb00000gn/T/LIVE-1987337710/apps.v1.Deployment.default.hello-server /var/folders/c0/j8z9nmwj3r93swy5clqm0jb00000gn/T/MERGED-1149805053/apps.v1.Deployment.default.hello-server
--- /var/folders/c0/j8z9nmwj3r93swy5clqm0jb00000gn/T/LIVE-1987337710/apps.v1.Deployment.default.hello-server       2024-07-19 07:22:36
+++ /var/folders/c0/j8z9nmwj3r93swy5clqm0jb00000gn/T/MERGED-1149805053/apps.v1.Deployment.default.hello-server     2024-07-19 07:22:36
@@ -6,7 +6,7 @@
     kubectl.kubernetes.io/last-applied-configuration: |
       {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"labels":{"app":"hello-server"},"name":"hello-server","namespace":"default"},"spec":{"replicas":3,"selector":{"matchLabels":{"app":"hello-server"}},"template":{"metadata":{"labels":{"app":"hello-server"}},"spec":{"containers":[{"image":"blux2/hello-server:1.6","livenessProbe":{"httpGet":{"path":"/health","port":8080},"initialDelaySeconds":10,"periodSeconds":5},"name":"hello-server","ports":[{"containerPort":8080}],"readinessProbe":{"httpGet":{"path":"/health","port":8080},"initialDelaySeconds":5,"periodSeconds":5},"resources":{"limits":{"cpu":"10m","memory":"5Gi"},"requests":{"cpu":"10m","memory":"5Gi"}}}]}}}}
   creationTimestamp: "2024-07-18T12:01:38Z"
-  generation: 2
+  generation: 3
   labels:
     app: hello-server
   name: hello-server
@@ -61,10 +61,10 @@
         resources:
           limits:
             cpu: 10m
-            memory: 64Gi
+            memory: 5Gi
           requests:
             cpu: 10m
-            memory: 64Gi
+            memory: 5Gi
         terminationMessagePath: /dev/termination-log
         terminationMessagePolicy: File
       dnsPolicy: ClusterFirst
```

修正内容を確認する。

```zsh
> kubectl get deployment hello-server -o=jsonpath='{.spec.template.spec.containers[0].resources.requests}' --namespace default
{"cpu":"10m","memory":"64Gi"}
```

64Miが反映されていることが確認できる。しばらくするとPodが全て正常に稼働していることがわかる。

最後に掃除をする。

```zsh
> kubectl delete --filename chapter-07/deployment-resource-handson.yaml --namespace default

deployment.apps "hello-server" deleted
```

続いて、メモリリークを起こしてOOMを発生されてみる。他のPodに影響を起こさないためのOOMなので、危険性はないが、もちろん開発環境でのみ実行しよう。

このハンズオンでは、Goのプログラム内であらかじめメモリリークを発生させるようにしてある。

```zsh
> kubectl apply --filename chapter-07/deployment-memory-leak.yaml --namespace default
deployment.apps/hello-server created
```

しばらくするとPodが全てRunningになる。

```zsh
> kubectl get pod --namespace default
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-856d5c6c7b-28pxz   1/1     Running   0          13m
hello-server-856d5c6c7b-cglsz   1/1     Running   0          13m
hello-server-856d5c6c7b-vlkz2   1/1     Running   0          13m
```

続いて、port-forwardを行う。アプリにアクセスすることでメモリリークが発生するようにできている。15秒ほど経つと、OOMKilledと出ているのがわかる。

```zsh
> kubectl port-forward deployments/hello-server 8080:8080 --namespace default
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080

> curl localhost:8080
curl: (52) Empty reply from server

> kubectl get pod --watch --namespace default
NAME                            READY   STATUS    RESTARTS      AGE
hello-server-856d5c6c7b-28pxz   1/1     Running   0             2d10h
hello-server-856d5c6c7b-cglsz   1/1     Running   0             2d10h
hello-server-856d5c6c7b-vlkz2   1/1     Running   1 (75s ago)   2d10h
hello-server-856d5c6c7b-vlkz2   0/1     OOMKilled   1 (106s ago)   2d10h
hello-server-856d5c6c7b-vlkz2   0/1     CrashLoopBackOff   1 (13s ago)    2d10h
hello-server-856d5c6c7b-vlkz2   1/1     Running            2 (16s ago)    2d10h
```

またしばらく観察するとRESTARTSが1、2と増えていく。PodのRESTARTSが1以上であること自体はよくあるので、通常、RESTARTSが1になっているだけでは気づかないかもしれない。継続的にRESTARTSが増えているなど、モニタリングを工夫すると良い。

OOMKilledが発生したPodの詳細を確認する。

```yaml
> kubectl describe pod hello-server-856d5c6c7b-vlkz2 --namespace default
Name:             hello-server-856d5c6c7b-vlkz2
Namespace:        default
Priority:         0
Service Account:  default
Node:             kind-control-plane/172.18.0.2
Start Time:       Fri, 19 Jul 2024 21:16:08 +0900
Labels:           app=hello-server
                  pod-template-hash=856d5c6c7b
Annotations:      <none>
Status:           Running
IP:               10.244.0.6
IPs:
  IP:           10.244.0.6
Controlled By:  ReplicaSet/hello-server-856d5c6c7b
Containers:
  hello-server:
    Container ID:   containerd://31a83376d5427aa3c904375b29b6a2410c3f5bc57169ea3a5fd90035d5a2fb67
    Image:          blux2/hello-server:1.7
    Image ID:       docker.io/blux2/hello-server@sha256:e34bb060e65c7f5cc58001c7e373e781e481b8875426227c3e1e4ac7709059af
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Mon, 22 Jul 2024 07:19:38 +0900
    Last State:     Terminated
      Reason:       OOMKilled
      Exit Code:    137
      Started:      Mon, 22 Jul 2024 07:17:41 +0900
      Finished:     Mon, 22 Jul 2024 07:19:23 +0900
    Ready:          True
    Restart Count:  2
    Limits:
      cpu:     10m
      memory:  100Mi
    Requests:
      cpu:        10m
      memory:     100Mi
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-g4vvj (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       True 
  ContainersReady             True 
  PodScheduled                True 
Volumes:
  kube-api-access-g4vvj:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   Guaranteed
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason   Age                    From     Message
  ----     ------   ----                   ----     -------
  Warning  BackOff  3m2s                   kubelet  Back-off restarting failed container hello-server in pod hello-server-856d5c6c7b-vlkz2_default(617a0d2e-3982-45ec-bfcf-35d801f79fc0)
  Normal   Created  2m50s (x3 over 2d10h)  kubelet  Created container hello-server
  Normal   Pulled   2m50s (x2 over 4m47s)  kubelet  Container image "blux2/hello-server:1.7" already present on machine
  Normal   Started  2m48s (x3 over 2d10h)  kubelet  Started container hello-server
```

Eventsを確認しても、OOMKilledされたかどうかわからない。Readiness/Liveness probeが設定されていても「タイムアウト」と出ているだけで正確な理由がわからない。

そこで参照するのがLast StateとReasonである。次のコマンドでlastStateを取得する。

```json
> kubectl get pod hello-server-856d5c6c7b-vlkz2 --output=jsonpath='{.status.containerStatuses[0].lastState}' --namespace default| jq .
{
  "terminated": {
    "containerID": "containerd://8d32824087deb8a977a1eedadf04c0b798720d4fb201b88a8e601bd260e0bf4e",
    "exitCode": 137,
    "finishedAt": "2024-07-21T22:19:23Z",
    "reason": "OOMKilled",
    "startedAt": "2024-07-21T22:17:41Z"
  }
}
```

lastStateでterminatedでreasonにOOMKilledと書かれている。イメージタグを1.8に更新することでOOMKilledが発生しなくなるようになっているので、kubectl editで修正する。

```zsh
> kubectl edit deployment/hello-server --namespace default
deployment.apps/hello-server edited
```

Podが全て再作成されていることを確認する。

```zsh
> kubectl get pod --namespace default
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-58654c5c9f-7ztsp   1/1     Running   0          48s
hello-server-58654c5c9f-ftf9q   1/1     Running   0          28s
hello-server-58654c5c9f-px27q   1/1     Running   0          24s
```

port-forwardで動作確認する。

```zsh
> kubectl port-forward deployments/hello-server 8080:8080 --namespace default
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080

> curl localhost:8080
Hello, world! Let's learn Kubernetes!
```

最後に掃除する。

```zsh
> kubectl delete --filename chapter-07/deployment-memory-leak.yaml --namespace default
deployment.apps "hello-server" deleted
```

今回のようにメモリリークが発生しても無尽蔵にリソースをしようしないために、リソースにRequestsとLimitsを指定しておくことでNode全体を安全に運用できる。

## Podのスケジュールに便利な機能を理解する

Podのスケジューリングを制御することは本番でサービスを安全に運用するにあたって知っておきたい。例えば、同じNodeにPodを乗せないことでNodeの障害に備えたり、特定の用途に使うPod専用にNodeを立ち上げたりすることができる。ここではNodeとPodの関係性を制御できる機能を紹介する。

### Nodeを指定する:Node selector

Node selectorは特定のNodeにのみスケジュールするという制御を行う機能である。例えば、SSDを使っているNodeにのみ`disktype: ssd`というラベルが付与されている場合、次のようなマニフェストでSSDを使っているNodeにのみPodをスケジュールできる。

```yaml
> cat chapter-07/pod-nodeselector.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.25.3
  nodeSelector:
    disktype: ssd
```

### Podのスケジュールを柔軟に指定する:AffinityとAnti-Affinity

後述するTaint/Tolerationと混同されるが、役割は異なる。Affinityは日本語では「類似性」や「密接な関係」と訳される。NodeとPod、あるいはPod同士が「近くなるように」または「近づかないように」スケジューリングを制約する。

Affinity/Anti-affinityは3種類ある。

- Node affinity
- Pod affinity
- Pod anti-affinity

#### Node Affinity

Node selectorとかなり近いが、Node selectorと異なり「可能ならスケジュールする」という選択が可能。Node selectorは対応するNodeが存在しないとPodをスケジュールできなくなるため、Node障害に対して弱くなってしまう。Node affinityを使うとスケジュールを制御しつつNode障害にも対応可能になっているので、「絶対に特定のNodeにスケジュールする必要がある」というケース以外はこちらを選択する方が良い。

Node selectorよりも柔軟にNodeを指定できるため、書き方が多少複雑になる。

agginity.nodeAffinityの下にはrequiredDuringSchedulingIgnoredFuringExecutionとpreferredDuringSchedulingIgnoredDuringExecutionの二つを指定可能。どちらを指定するかでマニフェストの書き方が少し変わる。

- requiredDuringSchedulingIgnoredDuringExecution：対応するNodeが見つからない場合、Podをスケジュールしない。Node selectorと考え方が同じだが、Nodeの指定方法がより柔軟にできる。
- preferredDuringSchedulingIgnoredDuringExecution：対応するNodeが見つからない場合、適応なNodeにスケジュールする

matchExpressionsを利用してNodeを指定する。Nodeを指定する方法は複数あるが、ここでは詳しく説明しない。

```yaml
> cat chapter-07/pod-nodeaffinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: node-affinity-pod
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 1
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: node-affinity-pod
    image: nginx:1.25.3
```

このマニフェストでは「Nodeについているラベルのkeyがdisktypeで、valueにssdが含まれているときにはそのNodeにスケジュールするが、対応するNodeがない場合でもPodをスケジュールする」ということを言っている。

operatorにはIn以外にもNotInやExistsなどを指定できるため、ラベルお指定が柔軟にできる。また、preferredDuringSchedulingIgnoredDuringExecutionを指定した場合、weightの指定が必要。複数preferredDuringSchedulingIgnoredDuringExecutionを指定したときに、各条件に重み付けをすることで一番weightの合計値が高いNodeにスケジュールされる。

このマニフェストではNodeラベルが一致していなくてもPodがスケジュールされる。

#### Pod AffinityとPod Anti-affinity

spec.affinity以下にpodAffinity/podAntiAffiityを指定するが、「Pod間のAffinity」と理解した方が良い。Node affinityがNodeのラベルに基づいてスケジューリングしていた一方、Pod affinity/anti-affinityではすでにNodeにスケジュールされているPodのラベルに基づいてスケジューリングする。

よく使うユースケースとしては、Node障害に備えて「同じアプリケーションを動かしているPodは同じNodeに乗せない」とするルールを追加することである。せっかくDeploymentでPodを冗長化しても、同じNodeに載っているとそのNodeが故障しただけでPodは全滅（サービス停止）してしまう。このルールを追加することで、Nodeに乗るPodを分散させることができる。ただし、このユースケースは最近登場したPod Topology Spread Constraintsで代替できることもある。

Node affinityと同様に、requiredDuringSheduringIgnoredDuringExecutionとpreferredDuringSchedulingIgnoredDuringExecutionの二つのルールを指定可能。

```yaml
> cat chapter-07/pod-antiaffinity.yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-anti-affinity
  labels:
    app: nginx
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - nginx
          topologyKey: kubernetes.io/hostname
  containers:
  - name: nginx
    image: nginx:1.25.3
```

このマニフェストでは「app:nginxのラベルがついているPodが割り当てられているNodeには、同じラベルを持つPodをなるべくスケジュールしない」というルールを指定している。

今回、topologyKeyにkubernetes.io/hostnameを指定することで「同じホスト（node）に乗せない」という指定をしているが、ここをkubernetes.io/zoneと指定することで「同じデータセンター（zone）にPodを配置しない」というケースに使える。

### Podを分散するための設定:Pod Topology Spread Constraints

Pod Topology Spread ConstraintsはPodを分散するための設定です。topologyKeyを使うことでどのようにPodを分散させるかを表現できる。例えば、topologyKeyにNodeのkubernetes.io/hostnameラベルを指定すると、ホスト間でPodを分散してスケジュールできうr。Pod anti-affinityでも似たような設定ができるが、この機能はPod anti-affinityより後に入っただけあって柔軟に設定できる。

pod anti-anffinityではpreferredDuringSchedulingIgnoredDuringExecutionを指定すると、Pod数がNode数を超えると、それ以上は制御できなかった（単一のNodeにPodが偏ってしまうこともあり得る）。逆に、完全に分散させようとrequiredDuringScjedulingIgnoredDuringExecutionを利用すると、今度はPod数がNode数を超えられなくなってしまう。

Pod Topology Spread ConstraintsではPod数がNode数を超えてもなるべく分散させるような設定を行うことができる。

```yaml
> cat chapter-07/pod-topology.yaml
kind: Pod
apiVersion: v1
metadata:
  name: mypod
  labels:
    app: nginx
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: nginx
  containers:
  - name: nginx
    image: nginx:1.25.3
```

ポイントとなる設定値はmaxSkew。NodeにスケジュールされているPod数の差分を見て、maxSke:1だと差分が1より大きくならないようにスケジュールする。

ただし、スケールダウン時に再分散されないなどPod Topology Spread Constraintsも万能ではない。

### TaintとToleration

TaintとTolerationはそれぞれ対になる概念である。TaintはNodeに付与する設定で、TolerationはPodに付与する設定である。Taintは「汚れ」、Tolerationは「寛容」と訳される。NodeについているTaint（汚れ）をPodが許容できるかどうか、と言った設定の考え方になる。

Node affinityでは「あるPodをどういうNodeにスケジュールしたいか」という指定方法だが、Taint/Tolerationは「あるNodeが特定のPodしかスケジュールしたくない（特に指定のないPodはスケジュールを拒否したい）」といった指定方法になる。

NodeにTaintをつける場合は、次の方法でつけられる（マニフェストで指定する方法はクラウドプロバイダによって方法が変わる）

`kubectl taint noded <対象ノード名> <label名>=<labelの値>:<Taintの効果>`

例えば、次のようにTaintをつけたとする。

`kubectl taint nodes node1 disktype=ssd:NoSchedule`

このTaintに対応するtolerationは次のように指定する。

```yaml
> cat chapter-07/pod-tolerations.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.25.3
    imagePullPolicy: IfNotPresent
  tolerations:
  - key: "disktype"
    value: "ssd"
    operator: "Equal"
    effect: "NoSchedule"
```

Node affinityのサンプルマニフェストとにた設定にしている。Node affinityはNode側でスケジュールを制御できないため、affinityがついていないPodもスケジュールされている。Taintをつけることで「SSDを使用したいPod以外はスケジュールしない」と言った制御が可能になる。

普段はNodeの管理をしていない場合でも、TaintによってPodがスケジュールできないこともあるため、TaintとTolerationは覚えていくと良い。

### Tips:Pod PriorityとPreemption

PodにはPriority（優先度）を設定できるという便利機能がある。しかし、思わぬスケジューリングが発生する可能性があるため、注意して使う。PodのPriorityはPod一つ一つに付与するのではなく、PriorityClassというリソースを使う。

次の手順でPriorityを設定できる。

1. PriorityClassを作成する
2. 1で設定したPriorityClassをPodのマニフェストに指定する

Priorityの高いPodがどのようにスケジューリングに作用するか説明する。例えば、priorityClassNameを指定した先ほどのnginx PodがどのNodeにもスケジュールできないときに、preemptionが発生する。あるNode上にスケジュールされている、nginx Podよりもpriorityが低いPodをEvict（強制退去）させることで、nginx Podをスケジュール可能にする。

ちなみに、Kubernetesではsystem-cluster-criticalとsystem-node-criticalというPriorityClassをデフォルトで作成する。

```zsh
> kubectl describe priorityclasses --namespace default
Name:              system-cluster-critical
Value:             2000000000
GlobalDefault:     false
PreemptionPolicy:  PreemptLowerPriority
Description:       Used for system critical pods that must run in the cluster, but can be moved to another node if necessary.
Annotations:       <none>
Events:            <none>


Name:              system-node-critical
Value:             2000001000
GlobalDefault:     false
PreemptionPolicy:  PreemptLowerPriority
Description:       Used for system critical pods that must not be moved from their current node.
Annotations:       <none>
Events:            <none>
```

一般的なアプリのPodよりもK8sクラスタ用のPodを優先的にスケジュールするため、高いPriorityが付けられている。試しにsystem-node-criticalのPriorityClassを利用しているPodを確認する。

```zsh
> kubectl get pod --all-namespaces -o jsonpath='{range .items[?(@.spec.priorityClassName=="system-node-critical")]}{.metadata.name}{"\t"}{.metadata.namespace}{"\n"}{end}'
etcd-kind-control-plane kube-system
kube-apiserver-kind-control-plane       kube-system
kube-controller-manager-kind-control-plane      kube-system
kube-proxy-8r5n5        kube-system
kube-scheduler-kind-control-plane       kube-system
```

K8sクラスタのcontrol-plane用Podがリストアップされた。

### Podのスケジューリングがうまくいかない

このハンズオンではマルチノード環境が求められる。kindを利用するか、killercodaを使う。

まずは現在使っているクラスタを削除する。

```zsh
> kind delete cluster
Deleting cluster "kind" ...
Deleted nodes: ["kind-control-plane"]
```

続いて、次のconfigは以下。

```yaml
> cat kind/multinode-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
- role: worker
- role: worker
```

このconfigを元にクラスタを作成する。

```zsh
> kind create cluster -n kind-multinode --config kind/multinode-config.yaml --image=kindest/node:v1.29.0
Creating cluster "kind-multinode" ...
 ✓ Ensuring node image (kindest/node:v1.29.0) 🖼
 ✓ Preparing nodes 📦 📦 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
 ✓ Joining worker nodes 🚜 
Set kubectl context to "kind-kind-multinode"
You can now use your cluster with:

kubectl cluster-info --context kind-kind-multinode

Thanks for using kind! 😊
```

ワーカーノード2台のクラスタができたことを確認する。

```zsh
> kubectl get node
NAME                           STATUS   ROLES           AGE   VERSION
kind-multinode-control-plane   Ready    control-plane   88s   v1.29.0
kind-multinode-worker          Ready    <none>          65s   v1.29.0
kind-multinode-worker2         Ready    <none>          65s   v1.29.0
```

kind-multinode-wokerとkind-multinode-worker2ができていることが確認できる。

では、マニフェストをapplyする。

```zsh
> kubectl apply --filename chapter-07/deployment-schedule-handson.yaml --namespace default
deployment.apps/hello-server created
```

続いて、Podの状態を確認する。

```zsh
> kubectl get pod --namespace default
NAME                           READY   STATUS    RESTARTS   AGE
hello-server-9c5ff67bd-9rtgj   1/1     Running   0          28s
hello-server-9c5ff67bd-ch4wz   0/1     Pending   0          28s
hello-server-9c5ff67bd-q9ms9   1/1     Running   0          28s
```

Podが一つPendingになっている。詳細を見てみる。

```yaml
> kubectl describe pod hello-server-9c5ff67bd-ch4wz --namespace default
Name:             hello-server-9c5ff67bd-ch4wz
Namespace:        default
Priority:         0
Service Account:  default
Node:             <none>
Labels:           app=hello-server
                  pod-template-hash=9c5ff67bd
Annotations:      <none>
Status:           Pending
IP:               
IPs:              <none>
Controlled By:    ReplicaSet/hello-server-9c5ff67bd
Containers:
  hello-server:
    Image:        blux2/hello-server:1.8
    Port:         8080/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-wcpcj (ro)
Conditions:
  Type           Status
  PodScheduled   False 
Volumes:
  kube-api-access-wcpcj:
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
  Type     Reason            Age   From               Message
  ----     ------            ----  ----               -------
  Warning  FailedScheduling  72s   default-scheduler  0/3 nodes are available: 1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }, 2 node(s) didn't match pod anti-affinity rules. preemption: 0/3 nodes are available: 1 Preemption is not helpful for scheduling, 2 No preemption victims found for incoming pod.
```

Eventsを確認する。

- `0/3 nodes are available:`　3つあるNodeのどれにもスケジュールできていない。
- `1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }`　1つのNodeにTaintがついているが、それに対応するtolerationがPodについていないので、スケジュールできていない。
- `2 node(s) didn't match pod anti-affinity rules`　2つのNodeはPod anti-affinity ruleにマッチしないためスケジュールができない
- `preemption: 0/3 nodes are available:`　PreemptionによってスケジュールできるNodeがない

スケジュールできない事情はわかったが、どう直すのが正しいのだろうか。まずは、TolerationとAffinityについて、マニフェストを確認する。

```zsh
> kubectl get deployment hello-server --output=jsonpath='{.spec.template.spec.tolerations}' --namespace default | jq
# 何も表示されない
> kubectl get deployment hello-server --output=jsonpath='{.spec.template.spec.affinity}' --namespace default | jq
{
  "podAntiAffinity": {
    "requiredDuringSchedulingIgnoredDuringExecution": [
      {
        "labelSelector": {
          "matchExpressions": [
            {
              "key": "app",
              "operator": "In",
              "values": [
                "hello-server"
              ]
            }
          ]
        },
        "topologyKey": "kubernetes.io/hostname"
      }
    ]
  }
}
```

Tolerationは何もついておらず、Pod anti-affinityがついている。Podnoanti-affinityの内容としては「同じKubernetes Nodeにapp: hello-serverというラベルがついたPodはスケジュールしない」ということを書いている。Nodeが3つあるのに、なぜPodが3つスケジュールできないか。ここでヒントとなるのは先ほど見たPodのEventsの`1 node(s) had untolerated taint {node-role.kubernetes.io/control-plane: }`となる。

次のコマンドでNodeのTolerationを見てみる。

```zsh
> kubectl get nodes -o custom-columns='NAME:.metadata.name,TAINTS-KEY:.spec.taints[*].key'
NAME                           TAINTS-KEY
kind-multinode-control-plane   node-role.kubernetes.io/control-plane
kind-multinode-worker          <none>
kind-multinode-worker2         <none>
```

`kind-multinode-control-plane`という名前のNodeにnode-role.kubernetes.io/control-planeというTaintがついている。直し方はいくつかある。

1. Tolerationを付けてTaint: {mode-role.kubernetes.io/control-plain:}がついているNodeにスケジュール可能とする
2. Nodeを増やし、Pod anti-affinityが守れるようにする
3. Deploymentのreplicasを減らし、Pod anti-affinityが守れるようにする
4. requiredDuringSchedulingIgnoredDuringExecutionをpreferredDuringSchedulingIgnoredDuringExecutionに変更し、Pod anti-affinityが守れなくても問題ないようにする。

他にもNodeのTaintを外したり、Pod anti-affinityを外したりする方法もある。2は環境によってはムズカシかもしれない。

本番運用環境はどうだろうか？アプリサーバはControl Plane用のNodeに乗せるのは適切ではないため、1は推奨されない。2はコストがかかるため、どうしても今の設定をいじれない場合以外はおすすめではない。3と4のどちらかでTaintやPod anti-affinityを修正することが多い。

今回は最も簡単な3を選択する。次のコマンドでreplicasを2に変更する。今回はDeploymentのreplica数を変更する`kubectl scale`コマンドを利用する。

```zsh
> kubectl scale deployment hello-server --replicas=2 --namespace default
deployment.apps/hello-server scaled
```

Podの状態を確認する。

```zsh
> kubectl get pod --namespace default
NAME                           READY   STATUS    RESTARTS   AGE
hello-server-9c5ff67bd-9rtgj   1/1     Running   0          22m
hello-server-9c5ff67bd-q9ms9   1/1     Running   0          22m
```

Podが全てRunningとなった。最後に掃除をする。

```zsh
> kubectl delete --filename chapter-07/deployment-schedule-handson.yaml --namespace default
deployment.apps "hello-server" deleted
```

Node数が多くなり、複数のTaint/Pod affiniry/Pod anti-affinityを使っているとFailed SchedulingのEvent欄にたくさんの理由が書かれる。読み解くのが大変だが、必ず何かヒントがあるので、自分が想定した設定になっているかを確認しよう。

## アプリをスケールさせる

アプリのアクセスが増えると、一つのPodでは負荷に耐えられなくなってくる。アプリをスケールさせることで安定性を上げる。一般に水平スケールと垂直スケールの二つの方法がある。

K8sでは自動で水平スケール、垂直スケールを行うことができる。

### 水平スケール

Horizontal Pod Autoscaler(HPA)を利用することで自動的にPod数を増やしたり、減らしたりすることができる。HPAは通常CPUやメモリの値に応じてPod数が増減するが、任意のメトリクスを利用して増減させることも可能。

HPAを利用するためにはmetrics-serverをインストールする必要がある。

```zsh
> kubectl apply --filename https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.6.4/components.yaml
serviceaccount/metrics-server created
clusterrole.rbac.authorization.k8s.io/system:aggregated-metrics-reader created
clusterrole.rbac.authorization.k8s.io/system:metrics-server created
rolebinding.rbac.authorization.k8s.io/metrics-server-auth-reader created
clusterrolebinding.rbac.authorization.k8s.io/metrics-server:system:auth-delegator created
clusterrolebinding.rbac.authorization.k8s.io/system:metrics-server created
service/metrics-server created
deployment.apps/metrics-server created
apiservice.apiregistration.k8s.io/v1beta1.metrics.k8s.io created
```

```zsh
> kubectl patch --namespace kube-system deployment metrics-server --type=json --patch '[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'
deployment.apps/metrics-server patched
```

metrics-serverが正常に起動していることを確認する。

```zsh
> kubectl get deployment metrics-server --namespace kube-system
NAME             READY   UP-TO-DATE   AVAILABLE   AGE
metrics-server   1/1     1            1           10m
```

READYが1/1、AVAILABLEが1になっていれば良い。次のようにマニフェストを書くことで水平スケールを実現する。

```yaml
> cat chapter-07/hpa-hello-server.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hpa-handson
  labels:
    app: hello-server
spec:
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
        image: blux2/hello-server:1.8
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "10Mi"
            cpu: "5m"
          limits:
            memory: "10Mi"
            cpu: "5m"
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: hello-server-hpa
spec:
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - resource:
      name: cpu
      target:
        averageUtilization: 50
        type: Utilization
    type: Resource
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: hpa-handson
---
apiVersion: v1
kind: Service
metadata:
  name: hello-server-service
spec:
  selector:
    app: hello-server
  ports:
    - protocol: TCP
      port: 8080
      targetPort: 8080
```

HPAのマニフェストでは`minReplicas`と`maxReplicas`を指定し、どれくらいPodを増減するかを決める。増減を決めるためのメトリクスは`metrics`以下に書く。

`target.averageUtilication`にはアプリの望ましいCPU使用率を書く。ここでは50とあるため、CPU使用率が常に50%を下回るようにPod数を増減させる。

では、このマニフェストを使って実際にスケールすることを確認する。

```zsh
> kubectl apply --filename chapter-07/hpa-hello-server.yaml --namespace default
deployment.apps/hpa-handson created
horizontalpodautoscaler.autoscaling/hello-server-hpa created
service/hello-server-service created
```

しばらくHPAの様子を観察する。

```zsh
> kubectl get hpa --watch --namespace default
NAME               REFERENCE                TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
hello-server-hpa   Deployment/hpa-handson   <unknown>/50%   1         10        1          40s
hello-server-hpa   Deployment/hpa-handson   0%/50%          1         10        1          45
```

TARGETSが0%から動かず、REPLICASも1から増えない。負荷をかけてPod数が増える様子を見てみる。

```zsh
> kubectl --namespace default run --stdin --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://hello-server-service.default.svc.cluster.local:8080; done"
```

上記のコマンドで負荷をかける。

しばらくすると、REPLICASが増えていると思う。TARGETSに書かれているパーセンテージの左側の値が増えているが、これが実際に使用している平均CPUの値である。

```zsh
> kubectl get hpa --watch --namespace default
NAME               REFERENCE                TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
hello-server-hpa   Deployment/hpa-handson   0%/50%    1         10        1          15h
hello-server-hpa   Deployment/hpa-handson   20%/50%   1         10        1          15h
hello-server-hpa   Deployment/hpa-handson   200%/50%   1         10        1          15h
hello-server-hpa   Deployment/hpa-handson   200%/50%   1         10        4          15h
hello-server-hpa   Deployment/hpa-handson   180%/50%   1         10        4          15h
hello-server-hpa   Deployment/hpa-handson   85%/50%    1         10        4          15h
hello-server-hpa   Deployment/hpa-handson   80%/50%    1         10        7          15h
hello-server-hpa   Deployment/hpa-handson   70%/50%    1         10        7          15h
hello-server-hpa   Deployment/hpa-handson   50%/50%    1         10        7          15h
hello-server-hpa   Deployment/hpa-handson   42%/50%    1         10        7          15h
hello-server-hpa   Deployment/hpa-handson   45%/50%    1         10        7          15h
hello-server-hpa   Deployment/hpa-handson   42%/50%    1         10        7          15h
```

負荷をかけ続けると、maxReplicasで指定した10個までPodが増える。このように、負荷に応じてPodがスケールするため、急な負荷に対応できるようになる。ただし、試してわかるがスケールするには少し時間がかかるため、本当に急なスパイクには対応できない。

最後に掃除をする。

```zsh
> kubectl delete --filename chapter-07/hpa-hello-server.yaml --namespace default
deployment.apps "hpa-handson" deleted
horizontalpodautoscaler.autoscaling "hello-server-hpa" deleted
service "hello-server-service" deleted
```

### 垂直スケール

