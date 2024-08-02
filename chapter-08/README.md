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

マニフェストを適用してアプリを更新する。

```zsh
> kubectl apply --filename chapter-08/hello-server-update.yaml --namespace default
deployment.apps/hello-server configured
configmap/hello-server-configmap configured
service/hello-server-external unchanged
poddisruptionbudget.policy/hello-server-pdb configured
```

## 正常性確認を行なってみる

問題なく更新できているか確認する。

```zsh
> curl localhost:30599
Hello, world! Let's learn Kubernetes!
```

アプリと疎通できているようだ。問題なくアプリの更新を行うことができたか、Deploymentのステータスを確認する。

```zsh
> kubectl get deployment --namespace default
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
hello-server   3/3     1            3           7h41m
```

UP-TO-DATEが1になっている。これはPodが1つ更新中ということを示している。続いて、ReplicaSetを確認する。

```zsh
> kubectl get replicaset --namespace default
NAME                      DESIRED   CURRENT   READY   AGE
hello-server-69fdcc8fb9   1         1         0       2m26s
hello-server-7765f564df   3         3         3       7h42m
```

AGEの若いReplicaSetのREADYが0で、古いReplicaSetのREADYが3になっている。どうやら古いバージョンのアプリが動き続けているだけのようだ。

ここからは次のことをゴールに原因調査と解消を行う。

- アプリにアクセスして「Hello, world! Let's build, break and fix」が返ってくる
- AGEの若いReplicaSetのREADYが3つになる

## 原因調査を行う

唯一の正解というわけではないが、正常性確認で「新規ReplicaSetがどうやらうまく動いていない」ということがわかった。まずPodを確認する。

```zsh
> kubectl get pod --namespace default
NAME                            READY   STATUS             RESTARTS        AGE
hello-server-69fdcc8fb9-dngs8   0/1     CrashLoopBackOff   419 (85s ago)   26h
hello-server-7765f564df-j2cnq   1/1     Running            0               34h
hello-server-7765f564df-ldfk2   1/1     Running            0               34h
hello-server-7765f564df-tc4qt   1/1     Running            0               34h
```

ReadyになっていないPodがいる。おそらく、このPodが新規ReplicaSetのものと想像がつく。さらに次のコマンドで詳細を見ていく。

```yaml
> kubectl describe pod hello-server-69fdcc8fb9-dngs8 --namespace default
Name:             hello-server-69fdcc8fb9-dngs8
Namespace:        default
Priority:         0
Service Account:  default
Node:             kind-nodeport-control-plane/172.18.0.5
Start Time:       Thu, 01 Aug 2024 07:22:33 +0900
Labels:           app=hello-server
                  pod-template-hash=69fdcc8fb9
Annotations:      <none>
Status:           Running
IP:               10.244.0.8
IPs:
  IP:           10.244.0.8
Controlled By:  ReplicaSet/hello-server-69fdcc8fb9
Containers:
  hello-server:
    Container ID:   containerd://2eebfa207bee225c46e0229a072105f1c4928dbaec96d9e47e4337ae13813603
    Image:          blux2/hello-server:2.0.1
    Image ID:       docker.io/blux2/hello-server@sha256:7cf006b172cb3d5a2d54f7a035dfb556405e3df90677ff3383b5c8001035d1f6
    Port:           <none>
    Host Port:      <none>
    State:          Waiting
      Reason:       CrashLoopBackOff
    Last State:     Terminated
      Reason:       Error
      Exit Code:    2
      Started:      Fri, 02 Aug 2024 10:00:05 +0900
      Finished:     Fri, 02 Aug 2024 10:00:27 +0900
    Ready:          False
    Restart Count:  419
    Limits:
      cpu:     10m
      memory:  256Mi
    Requests:
      cpu:      10m
      memory:   256Mi
    Liveness:   http-get http://:8081/health delay=10s timeout=1s period=5s #success=1 #failure=3
    Readiness:  http-get http://:8081/health delay=5s timeout=1s period=5s #success=1 #failure=3
    Environment:
      PORT:  <set to the key 'PORT' of config map 'hello-server-configmap'>  Optional: false
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-jwtjk (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       False 
  ContainersReady             False 
  PodScheduled                True 
Volumes:
  kube-api-access-jwtjk:
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
  Type     Reason     Age                    From     Message
  ----     ------     ----                   ----     -------
  Warning  Unhealthy  105m (x1195 over 26h)  kubelet  Liveness probe failed: Get "http://10.244.0.8:8081/health": dial tcp 10.244.0.8:8081: connect: connection refused
  Warning  Unhealthy  70m (x2068 over 26h)   kubelet  Readiness probe failed: Get "http://10.244.0.8:8081/health": dial tcp 10.244.0.8:8081: connect: connection refused
  Warning  BackOff    50m (x4341 over 26h)   kubelet  Back-off restarting failed container hello-server in pod hello-server-69fdcc8fb9-dngs8_default(795e506f-4010-46d9-a7ee-d51abf21751a)
```

Readiness probeもLiveness proveも両方失敗しているため、ヘルスチェック方のエンドポイントにアクセスできていない。いくつか原因が考えられる。

- アプリの内部が壊れてしまい、ヘルスチェックが通らなくなってしまった
- ヘルスチェック用のエンドポイント（ポート番号、パス）が間違っている。

さらにログも見てみる。

```zsh
> kubectl logs hello-server-69fdcc8fb9-dngs8 --namespace default
2024/08/02 01:05:32 Starting server on port 8082
```

8082ポートで受け付けているといっている。マニフェストのヘルスチェック設定を確認してみる。

```yaml
> kubectl get deployment hello-server --output yaml --namespace default
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "2"
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"labels":{"app":"hello-server"},"name":"hello-server","namespace":"default"},"spec":{"replicas":3,"selector":{"matchLabels":{"app":"hello-server"}},"template":{"metadata":{"labels":{"app":"hello-server"}},"spec":{"affinity":{"podAntiAffinity":{"preferredDuringSchedulingIgnoredDuringExecution":[{"podAffinityTerm":{"labelSelector":{"matchExpressions":[{"key":"app","operator":"In","values":["hello-server"]}]},"topologyKey":"kubernetes.io/hostname"},"weight":1}]}},"containers":[{"env":[{"name":"PORT","valueFrom":{"configMapKeyRef":{"key":"PORT","name":"hello-server-configmap"}}}],"image":"blux2/hello-server:2.0.1","livenessProbe":{"httpGet":{"path":"/health","port":8081},"initialDelaySeconds":10,"periodSeconds":5},"name":"hello-server","readinessProbe":{"httpGet":{"path":"/health","port":8081},"initialDelaySeconds":5,"periodSeconds":5},"resources":{"limits":{"cpu":"10m","memory":"256Mi"},"requests":{"cpu":"10m","memory":"256Mi"}}}]}}}}
  creationTimestamp: "2024-07-31T14:42:29Z"
  generation: 2
  labels:
    app: hello-server
  name: hello-server
  namespace: default
  resourceVersion: "37255"
  uid: 951012fb-eed0-4c41-8b45-75332100eb89
spec:
  progressDeadlineSeconds: 600
  replicas: 3
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: hello-server
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: hello-server
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - hello-server
              topologyKey: kubernetes.io/hostname
            weight: 1
      containers:
      - env:
        - name: PORT
          valueFrom:
            configMapKeyRef:
              key: PORT
              name: hello-server-configmap
        image: blux2/hello-server:2.0.1
        imagePullPolicy: IfNotPresent
        livenessProbe:
          failureThreshold: 3
          httpGet:
            path: /health
            port: 8081
            scheme: HTTP
          initialDelaySeconds: 10
          periodSeconds: 5
          successThreshold: 1
          timeoutSeconds: 1
        name: hello-server
        readinessProbe:
          failureThreshold: 3
          httpGet:
            path: /health
            port: 8081
            scheme: HTTP
          initialDelaySeconds: 5
          periodSeconds: 5
          successThreshold: 1
          timeoutSeconds: 1
        resources:
          limits:
            cpu: 10m
            memory: 256Mi
          requests:
            cpu: 10m
            memory: 256Mi
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  availableReplicas: 3
  conditions:
  - lastTransitionTime: "2024-07-31T14:43:09Z"
    lastUpdateTime: "2024-07-31T14:43:09Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: "2024-07-31T22:32:34Z"
    lastUpdateTime: "2024-07-31T22:32:34Z"
    message: ReplicaSet "hello-server-69fdcc8fb9" has timed out progressing.
    reason: ProgressDeadlineExceeded
    status: "False"
    type: Progressing
  observedGeneration: 2
  readyReplicas: 3
  replicas: 4
  unavailableReplicas: 1
  updatedReplicas: 1
```

ヘルスチェックのポート番号8081番号になっている。これを修正すれば良さそう。アプリの更新とともにヘルスチェック用のポート番号を変更したのだろう。ローカルにダウンロードしてあるマニフェストをコピーし、修正用に新規マニフェストを作成する。

```zsh
> cp chapter-08/hello-server-update.yaml chapter-08/hello-server-update-fix.yaml
```

chapter-08/hello-server-update-fix.yamlのヘルsyチェックのポート番号を8082に変更する。

```diff
> diff chapter-08/hello-server-update.yaml chapter-08/hello-server-update-fix.yaml
49c49
<             port: 8081
---
>             port: 8082
55c55
<             port: 8081
---
>             port: 8082
```

修正したマニフェストの変更を適用する。

```zsh
> kubectl apply --filename chapter-08/hello-server-update-fix.yaml --namespace default
deployment.apps/hello-server configured
configmap/hello-server-configmap unchanged
service/hello-server-external unchanged
poddisruptionbudget.policy/hello-server-pdb configured
```

Podの様子をwatchしてみる。

```zsh
> kubectl get pod --watch --namespace default
NAME                            READY   STATUS    RESTARTS     AGE
hello-server-758cd6dbd7-fbkdn   0/1     Running   1 (6s ago)   32s
hello-server-7765f564df-j2cnq   1/1     Running   0            34h
hello-server-7765f564df-ldfk2   1/1     Running   0            34h
hello-server-7765f564df-tc4qt   1/1     Running   0            34h
hello-server-758cd6dbd7-fbkdn   0/1     Running   2 (2s ago)   53s
```

Readyにならない。どうやらこれだけでは修正が足りない。再度Podの詳細を見る。

```yaml
> kubectl describe pod hello-server-758cd6dbd7-fbkdn --namespace default
Name:             hello-server-758cd6dbd7-fbkdn
Namespace:        default
Priority:         0
Service Account:  default
Node:             kind-nodeport-control-plane/172.18.0.5
Start Time:       Fri, 02 Aug 2024 10:11:54 +0900
Labels:           app=hello-server
                  pod-template-hash=758cd6dbd7
Annotations:      <none>
Status:           Running
IP:               10.244.0.9
IPs:
  IP:           10.244.0.9
Controlled By:  ReplicaSet/hello-server-758cd6dbd7
Containers:
  hello-server:
    Container ID:   containerd://6b91ae383aa8ca4b28bced2724482ceaedfb2ae647af89063401b3e607800a71
    Image:          blux2/hello-server:2.0.1
    Image ID:       docker.io/blux2/hello-server@sha256:7cf006b172cb3d5a2d54f7a035dfb556405e3df90677ff3383b5c8001035d1f6
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Fri, 02 Aug 2024 10:13:37 +0900
    Last State:     Terminated
      Reason:       Error
      Exit Code:    2
      Started:      Fri, 02 Aug 2024 10:13:12 +0900
      Finished:     Fri, 02 Aug 2024 10:13:35 +0900
    Ready:          False
    Restart Count:  4
    Limits:
      cpu:     10m
      memory:  256Mi
    Requests:
      cpu:      10m
      memory:   256Mi
    Liveness:   http-get http://:8082/health delay=10s timeout=1s period=5s #success=1 #failure=3
    Readiness:  http-get http://:8082/health delay=5s timeout=1s period=5s #success=1 #failure=3
    Environment:
      PORT:  <set to the key 'PORT' of config map 'hello-server-configmap'>  Optional: false
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-54lpk (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       False 
  ContainersReady             False 
  PodScheduled                True 
Volumes:
  kube-api-access-54lpk:
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
  Type     Reason     Age                 From               Message
  ----     ------     ----                ----               -------
  Normal   Scheduled  109s                default-scheduler  Successfully assigned default/hello-server-758cd6dbd7-fbkdn to kind-nodeport-control-plane
  Normal   Created    83s (x2 over 108s)  kubelet            Created container hello-server
  Normal   Started    81s (x2 over 105s)  kubelet            Started container hello-server
  Normal   Pulled     58s (x3 over 108s)  kubelet            Container image "blux2/hello-server:2.0.1" already present on machine
  Warning  Unhealthy  58s (x10 over 98s)  kubelet            Readiness probe failed: HTTP probe failed with statuscode: 404
  Warning  Unhealthy  58s (x6 over 93s)   kubelet            Liveness probe failed: HTTP probe failed with statuscode: 404
  Normal   Killing    58s (x2 over 83s)   kubelet            Container hello-server failed liveness probe, will be restarted
```

`Readiness probe failed`と`Liveness probe failed`が404と言っている。更新前と更新後のマニフェストではコンテナイメージのタグが1.8から2.0に上がっている。タグの差分から変更を確認する。

`https://github.com/aoi1/bbf-kubernetes/compare/1.8...2.0`

ソース差分を確認すると、ヘルスチェック用のパスが `/healthz` に変わっている。環境に適用したマニフェストを参照してみる。

```yaml
> cat chapter-08/hello-server-update-fix.yaml
---
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
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  values:
                  - hello-server
                  operator: In
              topologyKey: kubernetes.io/hostname
      containers:
      - name: hello-server
        image: blux2/hello-server:2.0.1
        env:
        - name: PORT
          valueFrom:
            configMapKeyRef:
              name: hello-server-configmap
              key: PORT
        resources:
          requests:
            memory: "256Mi"
            cpu: "10m"
          limits:
            memory: "256Mi"
            cpu: "10m"
        readinessProbe:
          httpGet:
            path: /health
            port: 8082
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health
            port: 8082
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: hello-server-configmap
data:
  PORT: "8082"
  HOST: "localhost"
---
apiVersion: v1
kind: Service
metadata:
  name: hello-server-external
spec:
  type: NodePort
  selector:
    app: hello-server
  ports:
    - port: 8081
      targetPort: 8081
      nodePort: 30599
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: hello-server-pdb
spec:
  maxUnavailable: 10%
  selector:
    matchLabels:
      app: hello-server
```

ヘルスチェック用のパスが `/health` になっている。再度修正する。

環境に適用する前にdiffで修正内容を見てみる。

```diff
> diff chapter-08/hello-server-update.yaml chapter-08/hello-server-update-fix.yaml
48,49c48,49
<             path: /health
<             port: 8081
---
>             path: /healthz
>             port: 8082
54,55c54,55
<             path: /health
<             port: 8081
---
>             path: /healthz
>             port: 8082
```

修正を適用する。

```zsh
> kubectl apply --filename chapter-08/hello-server-update-fix.yaml --namespace default
deployment.apps/hello-server configured
configmap/hello-server-configmap unchanged
service/hello-server-external unchanged
poddisruptionbudget.policy/hello-server-pdb configured
```

Podを確認する。

```zsh
> kubectl get pod --namespace default
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-5ff799f65f-hmf2x   1/1     Running   0          30s
hello-server-5ff799f65f-ws2ds   1/1     Running   0          40s
hello-server-5ff799f65f-xlp6j   1/1     Running   0          19s
```

しばらく待ってDeploymentの様子を見てみる。

```zsh
> kubectl get deployment hello-server --namespace default
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
hello-server   3/3     3            3           34h
```

無事、UP-TO-DATEが3になっている。ReplicaSetとPodを確認する。

```zsh
> kubectl get replicaset,pod --namespace default
NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-server-5ff799f65f   3         3         3       2m29s
replicaset.apps/hello-server-69fdcc8fb9   0         0         0       26h
replicaset.apps/hello-server-758cd6dbd7   0         0         0       10m
replicaset.apps/hello-server-7765f564df   0         0         0       34h

NAME                                READY   STATUS    RESTARTS   AGE
pod/hello-server-5ff799f65f-hmf2x   1/1     Running   0          2m19s
pod/hello-server-5ff799f65f-ws2ds   1/1     Running   0          2m29s
pod/hello-server-5ff799f65f-xlp6j   1/1     Running   0          2m8s
```

一番AGEが若いReplicaSetのREADYが3になっている。Podも全てRunningになっている。無事アプリの更新が完了した。

最後にアプリにアクセスする。

```zsh
> curl localhost:30599
curl: (52) Empty reply from server
```

今度はアプリにアクセスできなくなってしまった。何が問題なのだろうか。PodがRunningになっているので、デバッグ用のコンテナを立ち上げてコンテナ内部から接続確認してみる。

```zsh
> kubectl debug --stdin --tty hello-server-5ff799f65f-hmf2x --image=curlimages/curl --container=debug-container -- sh
~ $ curl localhost:8082
Hello, world! Let's build, break and fix!
```

localhost宛であれば問題なく動く。別ターミナルでPodのIPアドレスを確認してPod間通信を確かめてみる。まずはPodのIPアドレスを確認する。

```zsh
> kubectl get pod --output wide --namespace default
NAME                            READY   STATUS    RESTARTS   AGE     IP            NODE                          NOMINATED NODE   READINESS GATES
hello-server-5ff799f65f-hmf2x   1/1     Running   0          9m24s   10.244.0.11   kind-nodeport-control-plane   <none>           <none>
hello-server-5ff799f65f-ws2ds   1/1     Running   0          9m34s   10.244.0.10   kind-nodeport-control-plane   <none>           <none>
hello-server-5ff799f65f-xlp6j   1/1     Running   0          9m13s   10.244.0.12   kind-nodeport-control-plane   <none>           <none>
```

適当なPodを1つ選び、IPアドレスをコピーする。そして先ほどデバッグ用コンテナを立ち上げたターミルに戻り、接続確認する。

```zsh
~ $ curl 10.244.0.11:8082
Hello, world! Let's build, break and fix!~
```

Pod間通信も問題なさそう。こうなるとServiceが疑わしい。

```zsh
> kubectl get service --namespace default
NAME                    TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
hello-server-external   NodePort    10.96.98.46   <none>        8081:30599/TCP   34h
kubernetes              ClusterIP   10.96.0.1     <none>        443/TCP          34h
```

Readiness/Liveness probeのポートを8082に変更した時点で気づいたかもしれないが、ヘルスチェック用のポート番号が変わっただけではなかった。サーバーが待ち受けるポート番号が8081から8082に変更されている。しかし。Serviceはどうやらポート番号の変更に追従できていないようだ。

今回のマニフェストではポート番号をConfigMapから読み込んでおり、イメージの更新とともにConfigMapでポート番号が書き換えられている。適用したマニフェストを確認してみる。

```yaml
> cat chapter-08/hello-server-update-fix.yaml
---
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
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 1
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  values:
                  - hello-server
                  operator: In
              topologyKey: kubernetes.io/hostname
      containers:
      - name: hello-server
        image: blux2/hello-server:2.0.1
        env:
        - name: PORT
          valueFrom:
            configMapKeyRef:
              name: hello-server-configmap
              key: PORT
        resources:
          requests:
            memory: "256Mi"
            cpu: "10m"
          limits:
            memory: "256Mi"
            cpu: "10m"
        readinessProbe:
          httpGet:
            path: /healthz
            port: 8082
          initialDelaySeconds: 5
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /healthz
            port: 8082
          initialDelaySeconds: 10
          periodSeconds: 5
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: hello-server-configmap
data:
  PORT: "8082"
  HOST: "localhost"
---
apiVersion: v1
kind: Service
metadata:
  name: hello-server-external
spec:
  type: NodePort
  selector:
    app: hello-server
  ports:
    - port: 8081
      targetPort: 8081
      nodePort: 30599
---
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: hello-server-pdb
spec:
  maxUnavailable: 10%
  selector:
    matchLabels:
      app: hello-server
```

Serviceの更新が漏れていたということで、Serviceが利用するポート番号を変更する。

```diff
> diff chapter-08/hello-server-update.yaml chapter-08/hello-server-update-fix.yaml
48,49c48,49
<             path: /health
<             port: 8081
---
>             path: /healthz
>             port: 8082
54,55c54,55
<             path: /health
<             port: 8081
---
>             path: /healthz
>             port: 8082
76,77c76,77
<     - port: 8081
<       targetPort: 8081
---
>     - port: 8082
>       targetPort: 8082
```

次のコマンドで適用する。

```zsh
> kubectl apply --filename chapter-08/hello-server-update-fix.yaml --namespace default
deployment.apps/hello-server unchanged
configmap/hello-server-configmap unchanged
service/hello-server-external configured
poddisruptionbudget.policy/hello-server-pdb configured
```

Podに変更がないことを確認する。

```> kubectl get pod --namespace default
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-5ff799f65f-hmf2x   1/1     Running   0          20m
hello-server-5ff799f65f-ws2ds   1/1     Running   0          20m
hello-server-5ff799f65f-xlp6j   1/1     Running   0          19m
```

Serviceで8082ポートが指定されていることを確認する。

```zsh
> kubectl get service --namespace default
NAME                    TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)          AGE
hello-server-external   NodePort    10.96.98.46   <none>        8082:30599/TCP   34h
kubernetes              ClusterIP   10.96.0.1     <none>        443/TCP          34h
```

Serviceの修正が反映されている。接続確認してみる。

```zsh
> curl localhost:30599
Hello, world! Let's build, break and fix!
```

今度ことアプリが正常に稼働していることを確認できた。

最後にクラスタごと削除し、掃除をする。

```zsh
>  kind delete cluster -n kind-nodeport
Deleting cluster "kind-nodeport" ...
Deleted nodes: ["kind-nodeport-control-plane"]
```

デフォルトクラスタを立ち上げ直す。

```zsh
> kind create cluster --image=kindest/node:v1.29.0
Creating cluster "kind" ...
 ✓ Ensuring node image (kindest/node:v1.29.0) 🖼
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-kind"
You can now use your cluster with:

kubectl cluster-info --context kind-kind

Have a nice day! 👋
```

以上
