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