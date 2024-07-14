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

RollingUpdateStrategyフィールドではRolling Updateをどのように実現するかを書くことができる。

RollingUpdateStrategyで指定できるのはmaxUnavailableとmaxSurgeの２つ。maxUnavailableは「最大幾つのPodを同時にシャットダウンできるか」を指定する。デフォルトで指定されている25%とは「Pod全体の25%まで同時にシャットダウン可能」という意味である。

maxSurgeは「最大幾つのPodを新規作成できるか」を指定する。

Rolling Updateではアプリケーションのアップデートを行うために、古いPodをシャットダウンしながら更新先の新しいPodを作っていく。

一度に必要な数の新規Podを同時に作れば良いと思うかもしれないが、新旧のPodが同時に存在する時間はK8sクラスタ環境に2倍のPodが必要になる。Deploymentで指定しているreplicasが少ない時や、クラスタ全体のDeploymentの数が少ない時はどれでも良いかもしれない。

しかし、maxSurgeの数を多く指定するとそれだけ余分に作成するPodが増えるため、クラスタのキャパシティが必要になり、コストもかかる。

まずはStrageryTypeでRecreateを指定する。

```zsh
> kubectl apply --filename chapter-06/deployment-recreate.yaml --namespace default
deployment.apps/nginx-deployment configured
```

Podが正常に作成できていることを確認する。

```zsh
> kubectl get pod --namespace default
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-58556b4d6b-56545   1/1     Running   0          65m
nginx-deployment-58556b4d6b-8bvp4   1/1     Running   0          65m
nginx-deployment-58556b4d6b-997pr   1/1     Running   0          65m
nginx-deployment-58556b4d6b-crvp2   1/1     Running   0          65m
nginx-deployment-58556b4d6b-h4r4j   1/1     Running   0          65m
nginx-deployment-58556b4d6b-hn6bx   1/1     Running   0          65m
nginx-deployment-58556b4d6b-lg57d   1/1     Running   0          65m
nginx-deployment-58556b4d6b-mg5fb   1/1     Running   0          65m
nginx-deployment-58556b4d6b-p65g8   1/1     Running   0          65m
nginx-deployment-58556b4d6b-q6c6f   1/1     Running   0          65m
```

マニフェストに書いてあるimageタグを1.24.0から1.25.3に変更し、適用する。

マニフェストを適用した後にPodを参照し、コンテナが際作成されていることを確認する。

別のターミナルであらかじめ`kubectl get pod --watch`を実行しておき、遷移する様子を監査する。

```zsh
> kubectl apply --filename chapter-06/deployment-recreate.yaml --namespace default
deployment.apps/nginx-deployment configured

> kubectl get pod --watch
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-58556b4d6b-56545   1/1     Running   0          67m
nginx-deployment-58556b4d6b-8bvp4   1/1     Running   0          67m
nginx-deployment-58556b4d6b-997pr   1/1     Running   0          67m
nginx-deployment-58556b4d6b-crvp2   1/1     Running   0          67m
nginx-deployment-58556b4d6b-h4r4j   1/1     Running   0          67m
nginx-deployment-58556b4d6b-hn6bx   1/1     Running   0          67m
nginx-deployment-58556b4d6b-lg57d   1/1     Running   0          67m
nginx-deployment-58556b4d6b-mg5fb   1/1     Running   0          67m
nginx-deployment-58556b4d6b-p65g8   1/1     Running   0          67m
nginx-deployment-58556b4d6b-q6c6f   1/1     Running   0          67m
nginx-deployment-58556b4d6b-crvp2   1/1     Terminating   0          69m
nginx-deployment-58556b4d6b-56545   1/1     Terminating   0          69m
nginx-deployment-58556b4d6b-997pr   1/1     Terminating   0          69m
nginx-deployment-58556b4d6b-lg57d   1/1     Terminating   0          69m
nginx-deployment-58556b4d6b-h4r4j   1/1     Terminating   0          69m
nginx-deployment-58556b4d6b-p65g8   1/1     Terminating   0          69m
nginx-deployment-58556b4d6b-mg5fb   1/1     Terminating   0          69m
nginx-deployment-58556b4d6b-8bvp4   1/1     Terminating   0          69m
nginx-deployment-58556b4d6b-hn6bx   1/1     Terminating   0          69m
nginx-deployment-58556b4d6b-q6c6f   1/1     Terminating   0          69m
nginx-deployment-58556b4d6b-hn6bx   0/1     Terminating   0          69m
nginx-deployment-58556b4d6b-p65g8   0/1     Terminating   0          69m
nginx-deployment-58556b4d6b-crvp2   0/1     Terminating   0          69m
nginx-deployment-58556b4d6b-q6c6f   0/1     Terminating   0          69m
nginx-deployment-58556b4d6b-lg57d   0/1     Terminating   0          69m
nginx-deployment-58556b4d6b-8bvp4   0/1     Terminating   0          69m
nginx-deployment-58556b4d6b-mg5fb   0/1     Terminating   0          69m
nginx-deployment-58556b4d6b-h4r4j   0/1     Terminating   0          69m
nginx-deployment-58556b4d6b-997pr   0/1     Terminating   0          69m
nginx-deployment-58556b4d6b-56545   0/1     Terminating   0          69m
nginx-deployment-7947b6d4f6-bpm5h   0/1     Pending       0          0s
nginx-deployment-7947b6d4f6-fp9sw   0/1     Pending       0          0s
nginx-deployment-7947b6d4f6-bpm5h   0/1     Pending       0          0s
nginx-deployment-7947b6d4f6-wgnzt   0/1     Pending       0          0s
nginx-deployment-7947b6d4f6-fp9sw   0/1     Pending       0          0s
nginx-deployment-7947b6d4f6-wgnzt   0/1     Pending       0          0s
nginx-deployment-7947b6d4f6-jtmsn   0/1     Pending       0          0s
nginx-deployment-7947b6d4f6-pzfwm   0/1     Pending       0          0s
nginx-deployment-7947b6d4f6-mmqsw   0/1     Pending       0          0s
nginx-deployment-7947b6d4f6-5gvvk   0/1     Pending       0          0s
nginx-deployment-7947b6d4f6-bpm5h   0/1     ContainerCreating   0          0s
nginx-deployment-7947b6d4f6-ffvnb   0/1     Pending             0          0s
nginx-deployment-7947b6d4f6-2kqrz   0/1     Pending             0          0s
nginx-deployment-7947b6d4f6-x8jmd   0/1     Pending             0          0s
nginx-deployment-7947b6d4f6-mmqsw   0/1     Pending             0          0s
nginx-deployment-7947b6d4f6-jtmsn   0/1     Pending             0          0s
nginx-deployment-7947b6d4f6-pzfwm   0/1     Pending             0          0s
nginx-deployment-7947b6d4f6-5gvvk   0/1     Pending             0          0s
nginx-deployment-7947b6d4f6-ffvnb   0/1     Pending             0          0s
nginx-deployment-7947b6d4f6-x8jmd   0/1     Pending             0          0s
nginx-deployment-7947b6d4f6-2kqrz   0/1     Pending             0          0s
nginx-deployment-7947b6d4f6-fp9sw   0/1     ContainerCreating   0          0s
nginx-deployment-7947b6d4f6-wgnzt   0/1     ContainerCreating   0          0s
nginx-deployment-7947b6d4f6-5gvvk   0/1     ContainerCreating   0          0s
nginx-deployment-7947b6d4f6-jtmsn   0/1     ContainerCreating   0          0s
nginx-deployment-7947b6d4f6-x8jmd   0/1     ContainerCreating   0          0s
nginx-deployment-7947b6d4f6-2kqrz   0/1     ContainerCreating   0          0s
nginx-deployment-7947b6d4f6-ffvnb   0/1     ContainerCreating   0          0s
nginx-deployment-7947b6d4f6-mmqsw   0/1     ContainerCreating   0          0s
nginx-deployment-7947b6d4f6-pzfwm   0/1     ContainerCreating   0          0s
nginx-deployment-58556b4d6b-hn6bx   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-hn6bx   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-hn6bx   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-q6c6f   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-q6c6f   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-q6c6f   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-lg57d   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-lg57d   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-lg57d   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-997pr   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-997pr   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-997pr   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-crvp2   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-crvp2   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-crvp2   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-h4r4j   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-h4r4j   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-h4r4j   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-mg5fb   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-mg5fb   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-mg5fb   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-56545   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-56545   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-56545   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-p65g8   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-p65g8   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-p65g8   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-8bvp4   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-8bvp4   0/1     Terminating         0          69m
nginx-deployment-58556b4d6b-8bvp4   0/1     Terminating         0          69m
nginx-deployment-7947b6d4f6-5gvvk   1/1     Running             0          1s
nginx-deployment-7947b6d4f6-bpm5h   1/1     Running             0          1s
nginx-deployment-7947b6d4f6-x8jmd   1/1     Running             0          1s
nginx-deployment-7947b6d4f6-2kqrz   1/1     Running             0          1s
nginx-deployment-7947b6d4f6-ffvnb   1/1     Running             0          1s
nginx-deployment-7947b6d4f6-pzfwm   1/1     Running             0          1s
nginx-deployment-7947b6d4f6-fp9sw   1/1     Running             0          1s
nginx-deployment-7947b6d4f6-mmqsw   1/1     Running             0          1s
nginx-deployment-7947b6d4f6-wgnzt   1/1     Running             0          1s
nginx-deployment-7947b6d4f6-jtmsn   1/1     Running             0          1s
```

Podが一気にTerminating→ContainerCreating→Runningと遷移したことがわかる。Recreateは同時にすべてのPodを再作成するため全Podが更新完了するまでの速度は速いが、その分再作成時にアプリが一旦接続不能になってしまう。

最後に掃除をする。

```zsh
> kubectl delete --filename chapter-06/deployment-recreate.yaml --namespace default
deployment.apps "nginx-deployment" deleted
```

続いてRolling Updateを行う。

```zsh
> kubectl apply --filename chapter-06/deployment-rollingupdate.yaml --namespace default
deployment.apps/nginx-deployment created
```

Podが正常に作成できていることを確認する。

```zsh
> kubectl get pod --namespace default
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-58556b4d6b-4r5wx   1/1     Running   0          28s
nginx-deployment-58556b4d6b-6xjxf   1/1     Running   0          28s
nginx-deployment-58556b4d6b-dz6qs   1/1     Running   0          28s
nginx-deployment-58556b4d6b-f6krn   1/1     Running   0          28s
nginx-deployment-58556b4d6b-jspcm   1/1     Running   0          28s
nginx-deployment-58556b4d6b-mrsp4   1/1     Running   0          28s
nginx-deployment-58556b4d6b-pntgw   1/1     Running   0          28s
nginx-deployment-58556b4d6b-qpsfz   1/1     Running   0          28s
nginx-deployment-58556b4d6b-r6jtd   1/1     Running   0          28s
nginx-deployment-58556b4d6b-wj2qs   1/1     Running   0          28s
```

max surge 100%で試してみる。マニフェストがmax surge 100%になっていることを確認する。

```zsh
> kubectl get deployment nginx-deployment -o jsonpath='{.spec.strategy}'
{"rollingUpdate":{"maxSurge":"100%","maxUnavailable":"25%"},"type":"RollingUpdate"}
```

```zsh
> kubectl get pod --watch --namespace default
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-58556b4d6b-4r5wx   1/1     Running   0          3m29s
nginx-deployment-58556b4d6b-6xjxf   1/1     Running   0          3m29s
nginx-deployment-58556b4d6b-dz6qs   1/1     Running   0          3m29s
nginx-deployment-58556b4d6b-f6krn   1/1     Running   0          3m29s
nginx-deployment-58556b4d6b-jspcm   1/1     Running   0          3m29s
nginx-deployment-58556b4d6b-mrsp4   1/1     Running   0          3m29s
nginx-deployment-58556b4d6b-pntgw   1/1     Running   0          3m29s
nginx-deployment-58556b4d6b-qpsfz   1/1     Running   0          3m29s
nginx-deployment-58556b4d6b-r6jtd   1/1     Running   0          3m29s
nginx-deployment-58556b4d6b-wj2qs   1/1     Running   0          3m29s
```

```zsh
> kubectl apply --filename chapter-06/deployment-rollingupdate.yaml --namespace default
deployment.apps/nginx-deployment configured
```

```zsh
> kubectl get pod --watch --namespace default
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-58556b4d6b-4r5wx   1/1     Running   0          3m29s
nginx-deployment-58556b4d6b-6xjxf   1/1     Running   0          3m29s
nginx-deployment-58556b4d6b-dz6qs   1/1     Running   0          3m29s
nginx-deployment-58556b4d6b-f6krn   1/1     Running   0          3m29s
nginx-deployment-58556b4d6b-jspcm   1/1     Running   0          3m29s
nginx-deployment-58556b4d6b-mrsp4   1/1     Running   0          3m29s
nginx-deployment-58556b4d6b-pntgw   1/1     Running   0          3m29s
nginx-deployment-58556b4d6b-qpsfz   1/1     Running   0          3m29s
nginx-deployment-58556b4d6b-r6jtd   1/1     Running   0          3m29s
nginx-deployment-58556b4d6b-wj2qs   1/1     Running   0          3m29s
nginx-deployment-7947b6d4f6-c7cd5   0/1     Pending   0          0s
nginx-deployment-7947b6d4f6-c7cd5   0/1     Pending   0          0s
nginx-deployment-7947b6d4f6-nl4j4   0/1     Pending   0          0s
nginx-deployment-7947b6d4f6-ms4rn   0/1     Pending   0          0s
nginx-deployment-7947b6d4f6-nl4j4   0/1     Pending   0          0s
nginx-deployment-7947b6d4f6-c7cd5   0/1     ContainerCreating   0          0s
nginx-deployment-58556b4d6b-6xjxf   1/1     Terminating         0          4m17s
nginx-deployment-7947b6d4f6-ms4rn   0/1     Pending             0          0s
nginx-deployment-58556b4d6b-jspcm   1/1     Terminating         0          4m17s
nginx-deployment-7947b6d4f6-rg9k4   0/1     Pending             0          0s
nginx-deployment-7947b6d4f6-wb2ww   0/1     Pending             0          0s
nginx-deployment-7947b6d4f6-49lgl   0/1     Pending             0          0s
nginx-deployment-7947b6d4f6-bnbt4   0/1     Pending             0          0s
nginx-deployment-7947b6d4f6-rg9k4   0/1     Pending             0          0s
nginx-deployment-7947b6d4f6-nl4j4   0/1     ContainerCreating   0          0s
nginx-deployment-7947b6d4f6-wb2ww   0/1     Pending             0          0s
nginx-deployment-7947b6d4f6-49lgl   0/1     Pending             0          0s
nginx-deployment-7947b6d4f6-bnbt4   0/1     Pending             0          0s
nginx-deployment-7947b6d4f6-px6gv   0/1     Pending             0          0s
nginx-deployment-7947b6d4f6-gx4xf   0/1     Pending             0          0s
nginx-deployment-7947b6d4f6-v8szx   0/1     Pending             0          0s
nginx-deployment-7947b6d4f6-px6gv   0/1     Pending             0          0s
nginx-deployment-7947b6d4f6-gx4xf   0/1     Pending             0          0s
nginx-deployment-7947b6d4f6-v8szx   0/1     Pending             0          0s
nginx-deployment-7947b6d4f6-ms4rn   0/1     ContainerCreating   0          0s
nginx-deployment-7947b6d4f6-wb2ww   0/1     ContainerCreating   0          0s
nginx-deployment-7947b6d4f6-49lgl   0/1     ContainerCreating   0          0s
nginx-deployment-7947b6d4f6-rg9k4   0/1     ContainerCreating   0          0s
nginx-deployment-7947b6d4f6-bnbt4   0/1     ContainerCreating   0          0s
nginx-deployment-7947b6d4f6-gx4xf   0/1     ContainerCreating   0          0s
nginx-deployment-7947b6d4f6-v8szx   0/1     ContainerCreating   0          0s
nginx-deployment-7947b6d4f6-px6gv   0/1     ContainerCreating   0          0s
nginx-deployment-7947b6d4f6-c7cd5   1/1     Running             0          1s
nginx-deployment-7947b6d4f6-ms4rn   1/1     Running             0          1s
nginx-deployment-58556b4d6b-f6krn   1/1     Terminating         0          4m18s
nginx-deployment-7947b6d4f6-nl4j4   1/1     Running             0          1s
nginx-deployment-7947b6d4f6-wb2ww   1/1     Running             0          1s
nginx-deployment-58556b4d6b-wj2qs   1/1     Terminating         0          4m18s
nginx-deployment-58556b4d6b-pntgw   1/1     Terminating         0          4m18s
nginx-deployment-58556b4d6b-mrsp4   1/1     Terminating         0          4m18s
nginx-deployment-7947b6d4f6-gx4xf   1/1     Running             0          2s
nginx-deployment-7947b6d4f6-49lgl   1/1     Running             0          2s
nginx-deployment-7947b6d4f6-rg9k4   1/1     Running             0          2s
nginx-deployment-58556b4d6b-qpsfz   1/1     Terminating         0          4m19s
nginx-deployment-7947b6d4f6-bnbt4   1/1     Running             0          2s
nginx-deployment-7947b6d4f6-v8szx   1/1     Running             0          2s
nginx-deployment-58556b4d6b-dz6qs   1/1     Terminating         0          4m19s
nginx-deployment-58556b4d6b-4r5wx   1/1     Terminating         0          4m19s
nginx-deployment-7947b6d4f6-px6gv   1/1     Running             0          2s
nginx-deployment-58556b4d6b-r6jtd   1/1     Terminating         0          4m19s
nginx-deployment-58556b4d6b-jspcm   0/1     Terminating         0          4m27s
nginx-deployment-58556b4d6b-6xjxf   0/1     Terminating         0          4m28s
nginx-deployment-58556b4d6b-jspcm   0/1     Terminating         0          4m28s
nginx-deployment-58556b4d6b-jspcm   0/1     Terminating         0          4m28s
nginx-deployment-58556b4d6b-jspcm   0/1     Terminating         0          4m28s
nginx-deployment-58556b4d6b-6xjxf   0/1     Terminating         0          4m28s
nginx-deployment-58556b4d6b-6xjxf   0/1     Terminating         0          4m28s
nginx-deployment-58556b4d6b-6xjxf   0/1     Terminating         0          4m28s
nginx-deployment-58556b4d6b-f6krn   0/1     Terminating         0          4m28s
nginx-deployment-58556b4d6b-wj2qs   0/1     Terminating         0          4m28s
nginx-deployment-58556b4d6b-pntgw   0/1     Terminating         0          4m28s
nginx-deployment-58556b4d6b-mrsp4   0/1     Terminating         0          4m29s
nginx-deployment-58556b4d6b-mrsp4   0/1     Terminating         0          4m29s
nginx-deployment-58556b4d6b-f6krn   0/1     Terminating         0          4m29s
nginx-deployment-58556b4d6b-mrsp4   0/1     Terminating         0          4m29s
nginx-deployment-58556b4d6b-mrsp4   0/1     Terminating         0          4m29s
nginx-deployment-58556b4d6b-f6krn   0/1     Terminating         0          4m29s
nginx-deployment-58556b4d6b-f6krn   0/1     Terminating         0          4m29s
nginx-deployment-58556b4d6b-pntgw   0/1     Terminating         0          4m29s
nginx-deployment-58556b4d6b-pntgw   0/1     Terminating         0          4m29s
nginx-deployment-58556b4d6b-pntgw   0/1     Terminating         0          4m29s
nginx-deployment-58556b4d6b-wj2qs   0/1     Terminating         0          4m29s
nginx-deployment-58556b4d6b-wj2qs   0/1     Terminating         0          4m29s
nginx-deployment-58556b4d6b-wj2qs   0/1     Terminating         0          4m29s
nginx-deployment-58556b4d6b-r6jtd   0/1     Terminating         0          4m29s
nginx-deployment-58556b4d6b-4r5wx   0/1     Terminating         0          4m29s
nginx-deployment-58556b4d6b-qpsfz   0/1     Terminating         0          4m29s
nginx-deployment-58556b4d6b-dz6qs   0/1     Terminating         0          4m29s
nginx-deployment-58556b4d6b-4r5wx   0/1     Terminating         0          4m30s
nginx-deployment-58556b4d6b-4r5wx   0/1     Terminating         0          4m30s
nginx-deployment-58556b4d6b-4r5wx   0/1     Terminating         0          4m30s
nginx-deployment-58556b4d6b-r6jtd   0/1     Terminating         0          4m30s
nginx-deployment-58556b4d6b-dz6qs   0/1     Terminating         0          4m30s
nginx-deployment-58556b4d6b-dz6qs   0/1     Terminating         0          4m30s
nginx-deployment-58556b4d6b-dz6qs   0/1     Terminating         0          4m30s
nginx-deployment-58556b4d6b-qpsfz   0/1     Terminating         0          4m30s
nginx-deployment-58556b4d6b-qpsfz   0/1     Terminating         0          4m30s
nginx-deployment-58556b4d6b-qpsfz   0/1     Terminating         0          4m30s
nginx-deployment-58556b4d6b-r6jtd   0/1     Terminating         0          4m30s
nginx-deployment-58556b4d6b-r6jtd   0/1     Terminating         0          4m30s
```

途中、Podの数が倍になっている様子が観察できたと思う。max surgeが100%というのは、元あったPodの数と同じ数だけ新規Podを作成すること示す。max surge 100%はPodの更新が最も早く、かつ安全な方法ではある。

しかし、必要なリソースが倍になるため、使用する際はリソースキャパシティに注意すること。

最後に掃除をする。

```zsh
> kubectl delete --filename chapter-06/deployment-rollingupdate.yaml --namespace default
deployment.apps "nginx-deployment" deleted
```

### Deploymentを作って壊す

まず、Deployentを作る。

```zsh
> kubectl apply --filename chapter-06/deployment-hello-server.yaml --namespace default
deployment.apps/hello-server created
```

PodがRunningになっていることを確認する。

```zsh
> kubectl get pod --namespace default
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-6cc6b44795-5tcdz   1/1     Running   0          2s
hello-server-6cc6b44795-ktq5x   1/1     Running   0          2s
hello-server-6cc6b44795-sqzw7   1/1     Running   0          2s
```

まずはPodを消してみる。

```zsh
> kubectl delete pod hello-server-6cc6b44795-sqzw7 --namespace default
pod "hello-server-6cc6b44795-sqzw7" deleted

> kubectl get pod --namespace default
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-6cc6b44795-5tcdz   1/1     Running   0          63s
hello-server-6cc6b44795-ktq5x   1/1     Running   0          63s
hello-server-6cc6b44795-lnl8b   1/1     Running   0          11s
```

Podの一つだけAGEが若いことがわかる。つまり、消されたPodの代わりに、新しくPodが作られたことがわかる。このように、Deploymentを利用することで、Podが消されても必ずDesiredなPod数に一致するようK8sが自動で再作成してくれる。

次に、このDeploymentをRollingUpdateをしてみる。まずは現状のアプリが問題なく動いていることを試す。

```zsh
> kubectl port-forward deployments/hello-server 8080:8080
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080
```

```zsh
> curl localhost:8080
Hello, world!
```

続いて、マニフェストを適用してRollingUpdateする。

```zsh
> kubectl apply --filename chapter-06/deployment-hello-server-rollingupdate.yaml --namespace default
deployment.apps/hello-server configured
```

マニフェストの適用結果を確認する。

```zsh
> kubectl get pod --namespace default
NAME                            READY   STATUS         RESTARTS   AGE
hello-server-6cc6b44795-5tcdz   1/1     Running        0          101s
hello-server-6cc6b44795-ktq5x   1/1     Running        0          101s
hello-server-6cc6b44795-lnl8b   1/1     Running        0          49s
hello-server-6fb85ff748-msmmq   0/1     ErrImagePull   0          6s
```

Podが一つ増え、何かエラーが出ている。だが、動作確認しても、問題なく接続できる。

```zsh
> curl localhost:8080
Hello, world!
```

Deploymentの状況を確認する。

```zsh
> kubectl get deployment --namespace default
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
hello-server   3/3     1            3           3m37s
```

UP-TO-DATEが1になっている。これは、古いバージョンのPodをそのままに、新規バージョンのPodを1つ作成中にエラーになっていることを表している。

デフォルト設定「maxUnavaliable: 25%, maxSurge: 25%」ではPod数が3つの場合、25%は0.75になる。maxUnavailableは切り下げ、maxSurgeは切り上げなので、この場合はmaxUnavailable:0、maxSurge:1となる。

つまり、新規Podは同時に一つまで作成できるが、新規Podが正常に作成完了しない限り、次の古いPodを消すことができない。（新しいPodが一つ増えるとPod総数が4になり、Podを一つ消せるようになる）

次のコマンドを実行してReplicaSetを見る。

```zsh
> kubectl get replicaset --namespace default
NAME                      DESIRED   CURRENT   READY   AGE
hello-server-6cc6b44795   3         3         3       8m37s
hello-server-6fb85ff748   1         1         0       7m2s
```

古いバージョンのPodが残っているので、アプリの疎通ができている。

では、RollingUpdateを修正していく。まずは詳細を確認する。

```zsh
> kubectl describe pod hello-server-6fb85ff748-msmmq --namespace default
Name:             hello-server-6fb85ff748-msmmq
Namespace:        default
Priority:         0
Service Account:  default
Node:             kind-control-plane/172.18.0.2
Start Time:       Sat, 13 Jul 2024 09:54:43 +0900
Labels:           app=hello-server
                  pod-template-hash=6fb85ff748
Annotations:      <none>
Status:           Pending
IP:               10.244.0.67
IPs:
  IP:           10.244.0.67
Controlled By:  ReplicaSet/hello-server-6fb85ff748
Containers:
  hello-server:
    Container ID:   
    Image:          blux2/hello-server:1.3
    Image ID:       
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Waiting
      Reason:       ImagePullBackOff
    Ready:          False
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-npwhj (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       False 
  ContainersReady             False 
  PodScheduled                True 
Volumes:
  kube-api-access-npwhj:
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
  Normal   Scheduled  8m50s                   default-scheduler  Successfully assigned default/hello-server-6fb85ff748-msmmq to kind-control-plane
  Normal   Pulling    7m10s (x4 over 8m50s)   kubelet            Pulling image "blux2/hello-server:1.3"
  Warning  Failed     7m4s (x4 over 8m45s)    kubelet            Failed to pull image "blux2/hello-server:1.3": rpc error: code = NotFound desc = failed to pull and unpack image "docker.io/blux2/hello-server:1.3": failed to resolve reference "docker.io/blux2/hello-server:1.3": docker.io/blux2/hello-server:1.3: not found
  Warning  Failed     7m4s (x4 over 8m45s)    kubelet            Error: ErrImagePull
  Warning  Failed     6m49s (x6 over 8m44s)   kubelet            Error: ImagePullBackOff
  Normal   BackOff    3m45s (x19 over 8m44s)  kubelet            Back-off pulling image "blux2/hello-server:1.3"
  ```

`docker.io/blux2/hello-server:1.3: not found` とあるので、タグを1.3から1.2に修正する。

```zsh
> kubectl edit deployment hello-server --namespace default
deployment.apps/hello-server edited
```

PodとReplicaSetの状態を見てみる。

```zsh
> kubectl get pod,replicaset --namespace default
NAME                                READY   STATUS    RESTARTS   AGE
pod/hello-server-5d6fd6dbb9-bdvmn   1/1     Running   0          30s
pod/hello-server-5d6fd6dbb9-s94g7   1/1     Running   0          31s
pod/hello-server-5d6fd6dbb9-tr44h   1/1     Running   0          49s

NAME                                      DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-server-5d6fd6dbb9   3         3         3       49s
replicaset.apps/hello-server-6cc6b44795   0         0         0       13m
replicaset.apps/hello-server-6fb85ff748   0         0         0       11m
```

無事、RollingUpdateが完了している。

```zsh
> curl localhost:8080
Hello, world! Let's learn Kubernetes!
```

最後に掃除をしておく。

```zsh
> kubectl delete --filename chapter-06/deployment-hello-server-rollingupdate.yaml --namespace default
deployment.apps "hello-server" deleted
```

## Podへのアクセスを助けるService

DeploymentはIPアドレスを持たないため、Deploymentで作ったリソースにアクセスするためにはIPアドレスが割り振られているPod個々にアクセスする必要がある。それでは、Rolloing Updateの機能があってもアクセスしているPodが消えてしまえば接続が途切れてしまう。

Deploymentで作成した複数Podへのアクセスを適切にルーティングしてもらうために、Serviceというリソースを利用する。

Serviceだけ作成しても動かないので、Serviceに接続するDeploymentも作成して動作を確認する。

```zsh
> kubectl apply --filename chapter-06/deployment-hello-server.yaml --namespace default
deployment.apps/hello-server created
```

Podが作成できていることを確認する。

```zsh
> kubectl get pod --namespace default
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-6cc6b44795-4z6jb   1/1     Running   0          41s
hello-server-6cc6b44795-gtrx4   1/1     Running   0          41s
hello-server-6cc6b44795-wkp6n   1/1     Running   0          41s
```

では、Serviceリソースを作成する。

```zsh
> kubectl apply --filename chapter-06/service.yaml --namespace default
service/hello-server-service created
```

Serviceが作成できているか確認する。

```zsh
> kubectl get service hello-server-service --namespace default
NAME                   TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
hello-server-service   ClusterIP   10.96.189.2   <none>        8080/TCP   44s
```

Serviceが作成できた。port-forwardして動作を確認する。

```zsh
> kubectl port-forward svc/hello-server-service 8080:8080 --namespace default
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080

> curl localhost:8080
Hello, world!
```

hello-serverと通信ができた。

### ServiceのTypeについて

Typeを指定しない場合はデフォルトでClusterIPが指定される。

- ClusterIP:クラスタ内部のIPアドレスでServiceを公開する。このTypeで指定されたIPアドレスはクラスタ内部からしか疎通できない。Ingressというリソースを利用することで外部公開が可能になる。
- NodePort:すべてのNodeのIPアドレスで指定したポート番号（NodePort）を公開する。
- LoadBalancer:外部ロードバランサを用いて外部IPアドレスを公開する。ロードバランサは別で用意する必要がある。
- ExternalName:ServiceをexternalNameフィールドの内容にマッピングする。このマッピングにより、クラスのDNSサーバがその外部ホスト名の値を持つCNAMEレコードを返すように設定される。

ClusterIPでクラスタ内の通信ができることを確認する。まずはIPアドレスを参照する。

```zsh
> kubectl get service hello-server-service --namespace default
NAME                   TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)    AGE
hello-server-service   ClusterIP   10.96.189.2   <none>        8080/TCP   7m6s
```

続いて、新たにPodを作成し、curlを叩く。

```zsh
> kubectl run curl --image curlimages/curl --rm --stdin --tty --restart=Never --command -- curl 10.96.189.2:8080
Hello, world!pod "curl" deleted
```

ClusterIPを指定し、別のPodからhello-serverにアクセスできた。最後に掃除をする。

```zsh
> kubectl delete --filename chapter-06/deployment-hello-server.yaml --namespace default
deployment.apps "hello-server" deleted

> kubectl delete --filename chapter-06/service.yaml --namespace default
service "hello-server-service" deleted
```

次はNodePortでアクセスする。NodePortを利用するとクラスタ外からアクセスが可能になるため、port-forwardする必要がなくなる。

まずは、既存クラスタを一度掃除する。

```zsh
> kind delete cluster
Deleting cluster "kind" ...
Deleted nodes: ["kind-control-plane"]
```

続いて、次のファイルをクラスタ構築時に引数で参照する。

```zsh
> kind create cluster --name kind-nodeport --config kind/export-mapping.yaml --image=kindest/node:v1.29.0
Creating cluster "kind-nodeport" ...
 ✓ Ensuring node image (kindest/node:v1.29.0) 🖼
 ✓ Preparing nodes 📦  
 ✓ Writing configuration 📜 
 ✓ Starting control-plane 🕹️ 
 ✓ Installing CNI 🔌 
 ✓ Installing StorageClass 💾 
Set kubectl context to "kind-kind-nodeport"
You can now use your cluster with:

kubectl cluster-info --context kind-kind-nodeport

Have a question, bug, or feature request? Let us know! https://kind.sigs.k8s.io/#community 🙂
```

Deploymentを作成し直す。

```zsh
> kubectl apply --filename chapter-06/deployment-hello-server.yaml --namespace default
deployment.apps/hello-server created
```

続いて、Serviceを作成する。

```zsh
> kubectl apply --filename chapter-06/service-nodeport.yaml --namespace default
service/hello-server-external created
```

Serviceが作成されていることを確認する。

```zsh
> kubectl get service hello-server-external --namespace default
NAME                    TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
hello-server-external   NodePort   10.96.198.240   <none>        8080:30599/TCP   37s
```

アクセスする。まずはNodeのIPを取得する。

```zsh
> kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'
172.18.0.2
```

取得したInternalIPを利用してアクセスする。

```zsh
> curl localhost:30599
Hello, world!
```

NodePortは全Nodeに対してPortを紐づけるので、port-forwardをしなくてもhello-serverにアクセスできる。毎回port-forwardをする必要がなく便利だが、NodePortはNodeが故障などで利用できなくなると使えなくなってしまう。ローカルの開発環境で使うには便利だが、本番運用ではClusterIPやLoadBalancerを利用する方が良い。

掃除のため、リソースを削除する。

```zsh
> kubectl delete --filename chapter-06/deployment-hello-server.yaml --namespace default
deployment.apps "hello-server" deleted

> kubectl delete --filename chapter-06/service-nodeport.yaml --namespace default
service "hello-server-external" deleted
```

### Serviceを利用したDNS

K8sではService用のDNSレコードを自動で作成してくれるため、FQDNを覚えておくと良い。

まずは、DeploymentとServiceを作成する。

```zsh
> kubectl apply --filename chapter-06/service.yaml --namespace default
service/hello-server-service created

> kubectl apply --filename chapter-06/deployment-hello-server.yaml --namespace default
deployment.apps/hello-server created
```

続いて、kubectl runを利用してPodからcurlを時効し、hello-server-servicにアクセスできることを確認する。

```zsh
> kubectl --namespace default run curl --image curlimages/curl --rm --stdin --tty --restart=Never --command -- curl hello-server-service.default.svc.cluster.local:8080
Hello, world!pod "curl" deleted
```

最後に掃除をする。

```zsh
> kubectl delete --filename chapter-06/service.yaml --namespace default
service "hello-server-service" deleted

> kubectl delete --filename chapter-06/deployment-hello-server.yaml --namespace default
deployment.apps "hello-server" deleted
```

### Serviceを壊す

まずは正しく動く環境を作る。

```zsh
> kubectl apply --filename chapter-06/service-nodeport.yaml --namespace default
service/hello-server-external created

> kubectl apply --filename chapter-06/deployment-hello-server.yaml --namespace default
deployment.apps/hello-server created
```

アプリが動作していることを確認する。

```zsh
> curl localhost:30599
Hello, world!
```

続いて、次のようにマニフェストを適用する。

```zsh
> kubectl apply --filename chapter-06/service-destruction.yaml --namespace default
service/hello-server-external configured
```

動作確認してみる。

```zsh
> curl localhost:30599
curl: (52) Empty reply from server
```

動かないので、各種リソースを見ていく。

```zsh
> kubectl get pod --namespace default
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-6cc6b44795-4kklt   1/1     Running   0          2m51s
hello-server-6cc6b44795-647v2   1/1     Running   0          2m51s
hello-server-6cc6b44795-v96zp   1/1     Running   0          2m51s
```

Podは問題なく動作している。

```zsh
> kubectl get deployment --namespace default
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
hello-server   3/3     3            3           3m26s
```

Deploymentも問題なく動作している。

```zsh
> kubectl get service --namespace default
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
hello-server-external   NodePort    10.96.190.229   <none>        8080:30599/TCP   4m33s
kubernetes              ClusterIP   10.96.0.1       <none>        443/TCP          19h
```

Serviceも問題なく動作している。

原因を切り分けるには、なるべくアプリに近いとこから切り分けていくと良い。小さいところから切り分けていき、なるべく狭い範囲で原因を特定できるようにする。

1. Pod内からアプリの接続確認を行う
2. クラスタ内かつ別Podから接続確認を行う
3. クラスタ内かつ別PodからService経由で接続確認を行う

動作させているコンテナにはシェルが入っていないため、デバッグ用コンテナを起動して確認する。外部に公開しているポート番号とアプリが公開しているポート番号が異なるので注意する。

```zsh
> kubectl get pod --namespace default
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-6cc6b44795-4kklt   1/1     Running   0          7m5s
hello-server-6cc6b44795-647v2   1/1     Running   0          7m5s
hello-server-6cc6b44795-v96zp   1/1     Running   0          7m5s

> kubectl --namespace default debug --stdin --tty hello-server-6cc6b44795-v96zp --image curlimages/curl --target=hello-server -- sh
Targeting container "hello-server". If you don't see processes from this container it may be because the container runtime doesn't support this feature.
Defaulting debug container name to debugger-ctq8j.
If you don't see a command prompt, try pressing enter.

~ $ curl localhost:8080
Hello, world!~ $ 
~ $ exit
Session ended, the ephemeral container will not be restarted but may be reattached using 'kubectl attach hello-server-6cc6b44795-v96zp -c debugger-ctq8j -i -t' if it is still running
```

特に問題なさそうなので、Pod内の問題ではないことがわかる。

続いて、クラスタ内に新規に起動したPodから接続を確認する。まずはPod一覧を参照し、適当なPodのIPを取得しておく。

```zsh
> kubectl get pods -o custom-columns=NAME:.metadata.name,IP:.status.podIP
NAME                            IP
hello-server-6cc6b44795-4kklt   10.244.0.14
hello-server-6cc6b44795-647v2   10.244.0.12
hello-server-6cc6b44795-v96zp   10.244.0.13
```

続いて、新規作成Podから接続確認する。

```zsh
> kubectl --namespace default run curl --image curlimages/curl --rm --stdin --tty --restart=Never --command -- curl 10.244.0.14:8080
Hello, world!pod "curl" deleted
```

クラスタ内の別Podからのアクセスは問題ないことがわかる。

Serviceの情報を取得しておく。

```zsh
> kubectl get svc -o custom-columns=NAME:.metadata.name,IP:.spec.clusterIP
NAME                    IP
hello-server-external   10.96.190.229
kubernetes              10.96.0.1
```

ServiceのIPを利用してService経由でアプリにアクセスする。

```zsh
> kubectl --namespace default run curl --image curlimages/curl --rm --stdin --tty --restart=Never --command -- curl 10.96.190.229:8080
curl: (7) Failed to connect to 10.96.190.229 port 8080 after 0 ms: Couldn't connect to server
pod "curl" deleted
pod default/curl terminated (Error)
```

Serviceを通すとアクセスできなくなっている。

Serviceの設定を確認する。

```zsh
> kubectl describe service hello-server-external --namespace default
Name:                     hello-server-external
Namespace:                default
Labels:                   <none>
Annotations:              <none>
Selector:                 app=hello-serve
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.96.190.229
IPs:                      10.96.190.229
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  30599/TCP
Endpoints:                <none>
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

適用前のファイルとのdiffを取得する。

```diff
> kubectl diff --filename chapter-06/service-nodeport.yaml
diff -u -N /var/folders/c0/j8z9nmwj3r93swy5clqm0jb00000gn/T/LIVE-3816784694/v1.Service.default.hello-server-external /var/folders/c0/j8z9nmwj3r93swy5clqm0jb00000gn/T/MERGED-3691262470/v1.Service.default.hello-server-external
--- /var/folders/c0/j8z9nmwj3r93swy5clqm0jb00000gn/T/LIVE-3816784694/v1.Service.default.hello-server-external        2024-07-14 09:18:25
+++ /var/folders/c0/j8z9nmwj3r93swy5clqm0jb00000gn/T/MERGED-3691262470/v1.Service.default.hello-server-external      2024-07-14 09:18:25
@@ -24,7 +24,7 @@
     protocol: TCP
     targetPort: 8080
   selector:
-    app: hello-serve
+    app: hello-server
   sessionAffinity: None
   type: NodePort
 status:
```

selectorの値がtypoしていることがわかる。

今回は、マニフェストをapplyし直すことで修正が可能ではある。

```zsh
> kubectl apply --filename chapter-06/service-nodeport.yaml --namespace default
service/hello-server-external configured

> curl localhost:30599
Hello, world!

> kubectl --namespace default run curl --image curlimages/curl --rm --stdin --tty --restart=Never --command -- curl 10.96.190.229:8080
Hello, world!pod "curl" deleted
```

最後にクラスタごと削除し、掃除をする。

```zsh
> kind delete cluster -n kind-nodeport
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

## Podの外部から情報を読み込むConfigMap

ConfigMapを利用する方法は３つある。

1. コンテナ内のコマンドの引数として読み込む
2. コンテナの環境変数として読み込む
3. ボリュームを利用してアプリケーションのファイルとして読み込む

### コンテナの環境変数として読み込む

以下のマニフェストは、ポート番号を外部から指定できるようにしてある。

```yaml
> cat ./chapter-06/configmap/hello-server-env.yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-server
  labels:
    app: hello-server
spec:
  replicas: 1
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
        image: blux2/hello-server:1.4
        env:
        - name: PORT
          valueFrom:
            configMapKeyRef:
              name: hello-server-configmap
              key: PORT
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: hello-server-configmap
data:
  PORT: "8081"
```

```zsh
> kubectl apply --filename chapter-06/configmap/hello-server-env.yaml --namespace default
deployment.apps/hello-server created
configmap/hello-server-configmap created
```

リソースが作成できていることを確認する。

```zsh
> kubectl get deployment,configmap --namespace default
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello-server   1/1     1            1           41s

NAME                               DATA   AGE
configmap/hello-server-configmap   1      41s
configmap/kube-root-ca.crt         1      8m4s
```

port-forwardを利用して疎通確認してみる。

```zsh
> kubectl port-forward deployments/hello-server 8081:8081 --namespace default
Forwarding from 127.0.0.1:8081 -> 8081
Forwarding from [::1]:8081 -> 8081

> curl localhost:8081
Hello, world! Let's learn Kubernetes!
```

続いて、ConfigMapのPORTに書かれている値を変更してみる。まず、`./chapter-06/configmap/hello-server-env.yaml`を変更してみる。

```diff
> git diff -- chapter-06/configmap/hello-server-env.yaml
diff --git a/chapter-06/configmap/hello-server-env.yaml b/chapter-0>
index 04521f2..a2736eb 100644
--- a/chapter-06/configmap/hello-server-env.yaml
+++ b/chapter-06/configmap/hello-server-env.yaml
@@ -30,4 +30,4 @@ kind: ConfigMap
 metadata:
   name: hello-server-configmap
 data:
-  PORT: "8081"
+  PORT: "5555"
```

変更を適用する。

```zsh
> kubectl apply --filename chapter-06/configmap/hello-server-env.yaml --namespace default
deployment.apps/hello-server unchanged
configmap/hello-server-configmap configured
```

ConfigMap経由で設定した環境変数は、アプリの再起動をしないとアプリに反映されなイノで、次のコマンドでアプリを再起動する。

```zsh
> kubectl rollout restart deployment/hello-server --namespace default
deployment.apps/hello-server restarted
```

port-forwardで疎通確認する。以前のセッションが残っている場合は、一度切断する。

```zsh
> kubectl port-forward deployments/hello-server 5555:5555 --namespace default
Forwarding from 127.0.0.1:5555 -> 5555
Forwarding from [::1]:5555 -> 5555

> curl localhost:5555
Hello, world! Let's learn Kubernetes!
```

ConfigMapを利用して環境変数を変更できることがわかる。最後に掃除をする。

```zsh
> kubectl delete --filename chapter-06/configmap/hello-server-env.yaml --namespace default
deployment.apps "hello-server" deleted
configmap "hello-server-configmap" deleted
```

しかしながら、環境変数を変更するために毎回アプリの再作成が必要だとサービスの運用が現実的ではない。次の「ボリュームを利用してコンテナに設定ファイルを読み込ませる」を利用するとアプリの再作成なしでConfigMapの内容を再読み込みできる。

### ボリュームを利用してアプリのファイルとして読み込む

Podにはボリュームを設定することができ、消えてほしくないファイルを保存したり、Pod間でファイルを共有したりするファイルシステムを利用するために使用する。

以下のマニフェストではConfigMapからボリュームを作成し、コンテナにボリュームを読み込んでいる。

```yaml
> cat ./chapter-06/configmap/hello-server-volume.yaml
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
      containers:
      - name: hello-server
        image: blux2/hello-server:1.5
        volumeMounts:
        - name: hello-server-config
          mountPath: /etc/config
      volumes:
      - name: hello-server-config
        configMap:
          name: hello-server-configmap
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: hello-server-configmap
data:
  myconfig.txt: |-
    I am hungry.
```

```zsh
> kubectl apply --filename chapter-06/configmap/hello-server-volume.yaml --namespace default
deployment.apps/hello-server created
configmap/hello-server-configmap created
```

リソースが正常に作成できていることを確認する。

```zsh
> kubectl get pod --namespace default
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-594ccc7f64-9qvqg   1/1     Running   0          37s
hello-server-594ccc7f64-phv8x   1/1     Running   0          37s
hello-server-594ccc7f64-zfhvt   1/1     Running   0          37s

> kubectl describe configmap hello-server-configmap --namespace default
Name:         hello-server-configmap
Namespace:    default
Labels:       <none>
Annotations:  <none>

Data
====
myconfig.txt:
----
I am hungry.

BinaryData
====

Events:  <none>
```

ConfigMapの内容が確認できる。続いて、動作を確認する。

```zsh
> kubectl port-forward deployment/hello-server 8080:8080
Forwarding from 127.0.0.1:8080 -> 8080
Forwarding from [::1]:8080 -> 8080

> curl localhost:8080
I am hungry.
```

最後に掃除をする。

```zsh
> kubectl delete --filename chapter-06/configmap/hello-server-volume.yaml
deployment.apps "hello-server" deleted
configmap "hello-server-configmap" deleted
```

### ConfigMapを設定したら壊れた

ConfigMapを使ってアプリを壊してみる。まずは、正しく動く環境を作る。一度使用したhello-server-env.yamlを利用する。

```diff
> git diff -- chapter-06/configmap/hello-server-env.yaml
diff --git a/chapter-06/configmap/hello-server-env.yaml b/chapter-06/configmap/hello-server->
index a2736eb..04521f2 100644
--- a/chapter-06/configmap/hello-server-env.yaml
+++ b/chapter-06/configmap/hello-server-env.yaml
@@ -30,4 +30,4 @@ kind: ConfigMap
 metadata:
   name: hello-server-configmap
 data:
-  PORT: "5555"
+  PORT: "8081"
```

```zsh
> kubectl apply --filename chapter-06/configmap/hello-server-env.yaml --namespace default
deployment.apps/hello-server created
configmap/hello-server-configmap created
```

正常にPodが作成できることを確認する。

```zsh
> kubectl get pod --namespace default
NAME                            READY   STATUS    RESTARTS   AGE
hello-server-66b94d84cb-27drf   1/1     Running   0          32s
```

port-forwardで動作確認する。

```zsh
> kubectl port-forward deployment/hello-server 8081:8081
Forwarding from 127.0.0.1:8081 -> 8081
Forwarding from [::1]:8081 -> 8081

> curl localhost:8081
Hello, world! Let's learn Kubernetes!
```

では、新しいマニフェストを適用する。

```zsh
> kubectl apply --filename chapter-06/configmap/hello-server-destruction.yaml --namespace default
deployment.apps/hello-server configured
configmap/hello-server-configmap unchanged
```

疎通確認してみる。

```zsh
> curl localhost:8081
curl: (7) Failed to connect to localhost port 8081 after 0 ms: Couldn't connect to server
```

Podの状態を確認する。

```zsh
> kubectl get pod --namespace default
NAME                           READY   STATUS                       RESTARTS   AGE
hello-server-67588987f-cfxgs   0/1     CreateContainerConfigError   0          2m8s
```

STATUSがCreateContainerConfigErrorになっている。Podに問題がありそうだということがわかったので、詳細を確認する。

```zsh
> kubectl describe pod --namespace default
Name:             hello-server-67588987f-cfxgs
Namespace:        default
Priority:         0
Service Account:  default
Node:             kind-control-plane/172.18.0.2
Start Time:       Sun, 14 Jul 2024 13:22:53 +0900
Labels:           app=hello-server
                  pod-template-hash=67588987f
Annotations:      <none>
Status:           Pending
IP:               10.244.0.11
IPs:
  IP:           10.244.0.11
Controlled By:  ReplicaSet/hello-server-67588987f
Containers:
  hello-server:
    Container ID:   
    Image:          blux2/hello-server:1.4
    Image ID:       
    Port:           <none>
    Host Port:      <none>
    State:          Waiting
      Reason:       CreateContainerConfigError
    Ready:          False
    Restart Count:  0
    Environment:
      PORT:  <set to the key 'PORT' of config map 'hello-server-configmap'>  Optional: false
      HOST:  <set to the key 'HOST' of config map 'hello-server-configmap'>  Optional: false
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-hxzhc (ro)
Conditions:
  Type                        Status
  PodReadyToStartContainers   True 
  Initialized                 True 
  Ready                       False 
  ContainersReady             False 
  PodScheduled                True 
Volumes:
  kube-api-access-hxzhc:
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
  Type     Reason     Age                   From               Message
  ----     ------     ----                  ----               -------
  Normal   Scheduled  3m26s                 default-scheduler  Successfully assigned default/hello-server-67588987f-cfxgs to kind-control-plane
  Warning  Failed     80s (x12 over 3m25s)  kubelet            Error: couldn't find key HOST in ConfigMap default/hello-server-configmap
  Normal   Pulled     65s (x13 over 3m25s)  kubelet            Container image "blux2/hello-server:1.4" already present on machine
```

`Error: couldn't find key HOST in ConfigMap default/hello-server-configmap`なので、ConfigMapにHOSTというkeyが無いということがわかる。

次のコマンドを利用してDeploymentのマニフェストでHOSTのkeyを指定しているところを確認する。

```yaml
> kubectl get deployment hello-server --output yaml --namespace default
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "2"
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"labels":{"app":"hello-server"},"name":"hello-server","namespace":"default"},"spec":{"replicas":1,"selector":{"matchLabels":{"app":"hello-server"}},"strategy":{"rollingUpdate":{"maxSurge":0},"type":"RollingUpdate"},"template":{"metadata":{"labels":{"app":"hello-server"}},"spec":{"containers":[{"env":[{"name":"PORT","valueFrom":{"configMapKeyRef":{"key":"PORT","name":"hello-server-configmap"}}},{"name":"HOST","valueFrom":{"configMapKeyRef":{"key":"HOST","name":"hello-server-configmap"}}}],"image":"blux2/hello-server:1.4","name":"hello-server"}]}}}}
  creationTimestamp: "2024-07-14T04:19:36Z"
  generation: 2
  labels:
    app: hello-server
  name: hello-server
  namespace: default
  resourceVersion: "17567"
  uid: 98da961a-e343-4766-a531-4ea46ede4c4a
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: hello-server
  strategy:
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: hello-server
    spec:
      containers:
      - env:
        - name: PORT
          valueFrom:
            configMapKeyRef:
              key: PORT
              name: hello-server-configmap
        - name: HOST
          valueFrom:
            configMapKeyRef:
              key: HOST
              name: hello-server-configmap
        image: blux2/hello-server:1.4
        imagePullPolicy: IfNotPresent
        name: hello-server
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  conditions:
  - lastTransitionTime: "2024-07-14T04:19:38Z"
    lastUpdateTime: "2024-07-14T04:19:38Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: "2024-07-14T04:19:36Z"
    lastUpdateTime: "2024-07-14T04:22:53Z"
    message: ReplicaSet "hello-server-67588987f" is progressing.
    reason: ReplicaSetUpdated
    status: "True"
    type: Progressing
  observedGeneration: 2
  replicas: 1
  unavailableReplicas: 1
  updatedReplicas: 1
```

Deploymentでは確かにhello-server-configmapにHOSTがあることを想定している。

続いて、hello-server-configmapという名前のConfigMapの中身を見てみる。

```yaml
> kubectl get configmap hello-server-configmap --output yaml --namespace default
apiVersion: v1
data:
  PORT: "8081"
kind: ConfigMap
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","data":{"PORT":"8081"},"kind":"ConfigMap","metadata":{"annotations":{},"name":"hello-server-configmap","namespace":"default"}}
  creationTimestamp: "2024-07-14T04:19:36Z"
  name: hello-server-configmap
  namespace: default
  resourceVersion: "17276"
  uid: 35ff7db1-39d2-49aa-8d30-3c0d82e530b4
```

ConfigMapのdataにはHOSTが無いことがわかる。

こういった間違いでもDeploymentのRolling Upgradeを利用していれば、アプリが接続不要になることを防いでくれるはずだが、何が悪かったのだろうか。

改でDeploymentのマニフェストを確認してみる。

```yaml
> kubectl get deployment hello-server --output yaml --namespace default
apiVersion: apps/v1
kind: Deployment
metadata:
  annotations:
    deployment.kubernetes.io/revision: "2"
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"apps/v1","kind":"Deployment","metadata":{"annotations":{},"labels":{"app":"hello-server"},"name":"hello-server","namespace":"default"},"spec":{"replicas":1,"selector":{"matchLabels":{"app":"hello-server"}},"strategy":{"rollingUpdate":{"maxSurge":0},"type":"RollingUpdate"},"template":{"metadata":{"labels":{"app":"hello-server"}},"spec":{"containers":[{"env":[{"name":"PORT","valueFrom":{"configMapKeyRef":{"key":"PORT","name":"hello-server-configmap"}}},{"name":"HOST","valueFrom":{"configMapKeyRef":{"key":"HOST","name":"hello-server-configmap"}}}],"image":"blux2/hello-server:1.4","name":"hello-server"}]}}}}
  creationTimestamp: "2024-07-14T04:19:36Z"
  generation: 2
  labels:
    app: hello-server
  name: hello-server
  namespace: default
  resourceVersion: "17567"
  uid: 98da961a-e343-4766-a531-4ea46ede4c4a
spec:
  progressDeadlineSeconds: 600
  replicas: 1
  revisionHistoryLimit: 10
  selector:
    matchLabels:
      app: hello-server
  strategy:
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 25%
    type: RollingUpdate
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: hello-server
    spec:
      containers:
      - env:
        - name: PORT
          valueFrom:
            configMapKeyRef:
              key: PORT
              name: hello-server-configmap
        - name: HOST
          valueFrom:
            configMapKeyRef:
              key: HOST
              name: hello-server-configmap
        image: blux2/hello-server:1.4
        imagePullPolicy: IfNotPresent
        name: hello-server
        resources: {}
        terminationMessagePath: /dev/termination-log
        terminationMessagePolicy: File
      dnsPolicy: ClusterFirst
      restartPolicy: Always
      schedulerName: default-scheduler
      securityContext: {}
      terminationGracePeriodSeconds: 30
status:
  conditions:
  - lastTransitionTime: "2024-07-14T04:19:38Z"
    lastUpdateTime: "2024-07-14T04:19:38Z"
    message: Deployment has minimum availability.
    reason: MinimumReplicasAvailable
    status: "True"
    type: Available
  - lastTransitionTime: "2024-07-14T04:19:36Z"
    lastUpdateTime: "2024-07-14T04:22:53Z"
    message: ReplicaSet "hello-server-67588987f" is progressing.
    reason: ReplicaSetUpdated
    status: "True"
    type: Progressing
  observedGeneration: 2
  replicas: 1
  unavailableReplicas: 1
  updatedReplicas: 1
```

maxSurgeが0になっている。maxSurgeが1以上であれば、正常に動作するPodが残ったままなので、壊れることはなかった。

原因がわかったので。次のマニフェストを適用して修正する。

```diff
> git diff -- chapter-06/configmap/hello-server-destruction.yaml
diff --git a/chapter-06/configmap/hello-server-d>
index 0be6afe..42e6f99 100644
--- a/chapter-06/configmap/hello-server-destruct>
+++ b/chapter-06/configmap/hello-server-destruct>
@@ -40,3 +40,4 @@ metadata:
   name: hello-server-configmap
 data:
   PORT: "8081"
+  HOST: "localhost"
```

修正したら適用する。

```zsh
> kubectl apply --filename chapter-06/configmap/hello-server-destruction.yaml --namespace default
deployment.apps/hello-server unchanged
configmap/hello-server-configmap configured
```

Podを確認する。

```zsh
> kubectl get pod --namespace default
NAME                           READY   STATUS    RESTARTS   AGE
hello-server-67588987f-cfxgs   1/1     Running   0          12m
```

再度port-forwardしてアプリの接続確認を行う。

```zsh
> kubectl port-forward deployment/hello-server 8081:8081 --namespace default
Forwarding from 127.0.0.1:8081 -> 8081

> curl localhost:8081
Hello, world! Let's learn Kubernetes!
```

動作確認ができたので、最後に掃除をする。

```zsh
> kubectl delete --filename chapter-06/configmap/hello-server-destruction.yaml --namespace default
deployment.apps "hello-server" deleted
configmap "hello-server-configmap" deleted
```

## 機密データを扱うためのSecret

クレデンシャル情報をコーディングしたく無い場合や、環境ごとに情報が変わる場合など、アプリの外から値を設定したい場合にConfigMapが使えるが、ConfigMapを参照できる人が全員秘密情報にアクセスできるのは、セキュリティ上好ましくない。

そこでSecretというリソースを使用することで、アクセス権を分けられる。SecretのデータはBase64でエンコードして登録する必要がある。

SecretをPodに読み込む方法は2種類ある。

1. コンテナの環境変数として読み込む
2. ボリュームを利用してコンテナに設定ファイルを読み込む

### コンテナの環境変数として読み込む　Secret

Secretのデータを作成する。

