# Kubernetesクラスタ上にアプリケーションを動作させる

## リソースの使用を表すマニフェスト

マニフェストファイルには動作させたいリソースの仕様を記述する。

このマニフェストを使ってアプリを起動するためには、`kubectl`を使う必要がある。

`kubectl`を利用してKubernetesクラスタと通信することで、Kubernetes上にアプリケーションのコンテナを起動することができる。

## コンテナを起動するための最小構成リソース、Pod

最小構成単位としてPodというリソースがある。

例えばAというサービスと、ログを転送するサービスがあったときに、これらは1つのPodとして起動することが多い。

## リソースを作成するための場所、Namespace

Namespaceは単一クラス内のリソース群を分離するメカニズムを提供する。例えば、リソースの名前はNamespace内で一位である必要があるが、Namespace間では一位である必要はない。また、Namespaceごとに権限を分けることもできる。

まとまった単位でリソースをまとめたい要件があるときにNamespaceを使う。すべてのリソースがNamespaceを利用できるわけではなく、例えばクラスタワイドに作成するリソース（例：Node）はNamespaceの適用範囲外である。

## Podを作成する前にK8sクラスタの起動を確認する

Podを作成する前にクラスタが起動できているか、kubectlを利用できるか、まずは確認する。

```zsh
kubectl get noeds
```

kindを利用していれば次のようにクラスタの存在を確認できる。

```zsh
kind get clusters
```

## マニフェストを利用する

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp
  labels:
    app: myapp
spec:
  containers:
  - name: hello-server
    image: blux2/hello-server:1.0
    ports:
      - containerPort: 8080
```

## マニフェストをK8sクラスタに適用する

`kubectl apply --filename <filename>` でK8sクラスタ上にリソースを作成できる。まずはPodが存在しないことを確認する。

```zsh
kubectl get pod --namespace default
```

続いて、マニフェストを適用する。

```zsh
kubectl apply --filename chapter-04/myapp.yaml --namespace default
```

Podが作成できていることを確認する。

```zsh
kubectl get pod --namespace default
```

STATUSがRunningになっていることが確認できればPodの作成完了である。STATUSがContainerCreatingなど、Running以外が表示されたとしても、しばらく待っていればRunningになるはず。
