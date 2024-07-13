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

Podが一気にTerminating→ContainerCreating→Runningと遷移したことがわかる。Recreateは同時にすべてのPodを再作成するため全Podが更新完了するまでの速度は速いが、その分再作成時にアプリが一旦接獄不能になってしまう。

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
