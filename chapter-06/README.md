# Kubernetesリソースを作って壊す

## Podのライフサイクル

Podはマニフェストが登録されてからNodeにスケジュールされ、kubeletがコンテナを起動し、異常があったり完了条件を満たしたりする場合、終了する。

## Podを冗長化するためのReplicaSetとDeployment

Pod単体ではコンテナの冗長化ができないので、本番環境での運用には向かない。そこで利用するのがDeployment。DeploymentはReplicaSetというリソースを作り、ReplicaSetがPodを作る。

![ReplicaSet](./ReplicaSet.drawio.svg)

### ReplicaSet

ReplicaSetは指定した数のPodを複製するリソースである。Podリソースと異なるのは、Podを複製できるところである。複製するPodの数をreplicasで指定できる。

```yaml
> cat ./chapter-06/replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: httpserver
  labels:
    app: httpserver
spec:
  replicas: 3
  selector:
    matchLabels:
      app: httpserver
  template:
    metadata:
      labels:
        app: httpserver
    spec:
      containers:
      - name: nginx
        image: nginx:1.25.3
```

このマニフェストを適用するとPodが3つ作られる。

```zsh
> kubectl apply --filename chapter-06/replicaset.yaml --namespace default
replicaset.apps/httpserver created
```

ReplicaSetは同じPodを複製する関係上、自動でPodにsuffixが付与される。

```zsh
> kubectl get pod --namespace default
NAME               READY   STATUS    RESTARTS        AGE
httpserver-9qscc   1/1     Running   1 (4m32s ago)   3d23h
httpserver-cc25m   1/1     Running   1 (4m32s ago)   3d23h
httpserver-rq4dc   1/1     Running   1 (4m32s ago)   3d23h
```

次のコマンドでReplicaSetのリソースも直接参照できる。DESIREDカラムから、幾つPodが作成されるべきかがわかる。

```zsh
> kubectl get replicaset --namespace default
NAME         DESIRED   CURRENT   READY   AGE
httpserver   3         3         3       3d23h
```

最後にdeleteコマンドで掃除をする。

```zsh
> kubectl delete replicaset httpserver --namespace default
replicaset.apps "httpserver" deleted
```

### Deployment

DeploymentとReplicaSetの違いとしては、Pod更新時に無停止で更新する場合に、v1を稼働しながらv2を立ち上げることがDeploymentでは可能になる。DeploymentはReplicaSetをの上位概念とも考えられる。

```zsh
> kubectl apply --filename chapter-06/deployment.yaml --namespace default
deployment.apps/nginx-deployment created
```

次のコマンドでDeploymentが作成できていることを確認する。

```zsh
> kubectl get deployment --namespace default
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
nginx-deployment   3/3     3            3           45s
```

次のコマンドでPodが作成できていることを確認する。

```zsh
> kubectl get pod --namespace default
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-595dff4799-8j4np   1/1     Running   0          58m
nginx-deployment-595dff4799-pj4jx   1/1     Running   0          58m
nginx-deployment-595dff4799-vxw55   1/1     Running   0          58m
```

ReplicaSetが作成されていることも確認できる。

```zsh
> kubectl get replicaset --namespace default
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-595dff4799   3         3         3       59m
```

続いて、Podの更新を行う。

```diff
> git diff HEAD origin
diff --git a/chapter-06/deployment.yaml b/chapter-06/deployment.yaml
index cce76d3..415b940 100644
--- a/chapter-06/deployment.yaml
+++ b/chapter-06/deployment.yaml
@@ -16,6 +16,6 @@ spec:
     spec:
       containers:
       - name: nginx
-        image: nginx:1.25.3
+        image: nginx:1.24.0
         ports:
         - containerPort: 80
```

```zsh
> kubectl apply --filename chapter-06/deployment.yaml --namespace default
deployment.apps/nginx-deployment configured
```

Pod名が新しくなっていることを確認する。

```zsh
> kubectl get pod --namespace default
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-789bf7b8fc-6p4lv   1/1     Running   0          43s
nginx-deployment-789bf7b8fc-8zsjx   1/1     Running   0          44s
nginx-deployment-789bf7b8fc-vspxl   1/1     Running   0          42s
```

次のコマンドでReplicaSetが新しくなっていることを確認する。

```zsh
> kubectl get replicaset --namespace default
NAME                          DESIRED   CURRENT   READY   AGE
nginx-deployment-595dff4799   0         0         0       24h
nginx-deployment-789bf7b8fc   3         3         3       91s
```

次のコマンドを実行することで、imageを参照できる。

```zsh
> kubectl get deployment nginx-deployment -o=jsonpath='{.spec.template.spec.containers[0].image}'
nginx:1.25.3
```

Deploymentでは新規バージョン追加時の挙動を制御することもできる。次のコマンドで作成したマニフェストを参照すると、StrategyTypeやRollingUpdateStrategyという項目が追加されていると思う。

```zsh
> kubectl describe deployment nginx-deployment
Name:                   nginx-deployment
Namespace:              default
CreationTimestamp:      Mon, 08 Jul 2024 21:26:49 +0900
Labels:                 app=nginx
Annotations:            deployment.kubernetes.io/revision: 2
Selector:               app=nginx
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=nginx
  Containers:
   nginx:
    Image:        nginx:1.25.3
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  nginx-deployment-595dff4799 (0/0 replicas created)
NewReplicaSet:   nginx-deployment-789bf7b8fc (3/3 replicas created)
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  25h    deployment-controller  Scaled up replica set nginx-deployment-595dff4799 to 3
  Normal  ScalingReplicaSet  5m12s  deployment-controller  Scaled up replica set nginx-deployment-789bf7b8fc to 1
  Normal  ScalingReplicaSet  5m11s  deployment-controller  Scaled down replica set nginx-deployment-595dff4799 to 2 from 3
  Normal  ScalingReplicaSet  5m11s  deployment-controller  Scaled up replica set nginx-deployment-789bf7b8fc to 2 from 1
  Normal  ScalingReplicaSet  5m10s  deployment-controller  Scaled down replica set nginx-deployment-595dff4799 to 1 from 2
  Normal  ScalingReplicaSet  5m10s  deployment-controller  Scaled up replica set nginx-deployment-789bf7b8fc to 3 from 2
  Normal  ScalingReplicaSet  5m9s   deployment-controller  Scaled down replica set nginx-deployment-595dff4799 to 0 from 1
```

RollingUpdateStrategyはPodにもReplicaSetにもない、Deploymentのみ存在するフィールドである。マニフェストの中ではStrategyTypeとRollingUpdateStrategyを指定しなかったので、デフォルト値が指定されている。

StrategyTypeとは、Deploymentを利用してPodを更新するときに、どのような戦略で更新するかを指定する。RecreateとRollingUpdateの２つが選択可能である。Recreateは全部のPodを同時に更新し、逆にRollingUpdateはPodを順番に更新する方法である。

RollingUpdateを選択した場合、RollingUpdateStrategyを記載することができる。デフォルトではRollingUpdate方式で更新し、25% max unavailable, 25% max surgeが指定される。


