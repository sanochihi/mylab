# 前提

シェルの設定で、alias k='kubectl' を指定済み

# 作成

## 単体の pod を作る

```shell
k run <pod名> --image=<image名> [-n <名前空間名>]
```

## yamlの定義から pod/deployment を作る

```shell
k apply -f <yamlのファイルパス>
```

-> yaml の定義（kind:）がPod なら pod, Deployment ならデプロイメントができる
-> 同じ名前のデプロイメントがすでに上がっている状態で、別の spec を定義した yaml を指定して apply するとローリングアップデートが行われる

## Deployment の作成

```shell
k create deploy <pod名のプレフィックス> --image=<image名> --replicas=<レプリカ数>
```

## ドライラン

pod をやデプロイメントを実際には作成せずに、定義ファイルだけを出力する

### podの場合

```shell
k run <pod名> --image=<image名> --dry-run=client -o yaml
```

### デプロイメントの場合

```shell
k create deploy <デプロイ名> --image=<image名> --replicas=<replica数> --dry-run=client -o yaml
```

-> これらを yaml ファイルに吐き出し、必要な部分を編集することで同じ pod や deployment を繰り返し作ることができる

# 管理

## context

現在のクラスタを確認する

```
k config current-context
```

名前空間を切り替える

```
k config set-context --current --namespace=<名前空間名>
```


## pod

### podの状態を見る(簡略)

```shell
k get pods [-n <名前空間名>]
```

```
NAME            READY   STATUS    RESTARTS   AGE
httpd-chihiro   1/1     Running   0          6d20h
nginx-chihiro   1/1     Running   0          6d20h
```

### podの状態を見る(詳細)

```shell
k get pods -o wide
```

```
NAME            READY   STATUS    RESTARTS   AGE     IP           NODE                   NOMINATED NODE   READINESS GATES
httpd-chihiro   1/1     Running   0          6d20h   10.42.0.22   lima-rancher-desktop   <none>           <none>
nginx-chihiro   1/1     Running   0          6d20h   10.42.0.21   lima-rancher-desktop   <none>           <none>
```

-> ここで表示されたIPアドレスに対して、pod 同士で ping を通したり curl をたたいたりすることができる

### pod のリソース詳細表示

```shell
k describe pod <pod名>
```

## Deployment

### Deployment の状態を見る

```shell
k get deployments.apps
```

以下のような出力が出てくる

```shell
NAME   READY   UP-TO-DATE   AVAILABLE   AGE
test   3/3     3            3           4m24s
```

### Deployment のリソース詳細表示

```shell
k describe deployment.apps <デプロイ名（プレフィックス部分）>
```

# コマンドの実行

## podに入って bash でコマンドを実行する

```shell
k exec -it <pod名> -- /bin/bash
```

抜けるには `exit`


# 編集

## pod の編集

```shell
k edit pod <pod名>
```

-> デフォルトのテキストエディタ（viなど）でpodの設定を編集できる

## Deployment の編集

```shell
k edit deployments.apps <デプロイ名（プレフィックス部分）>
```

-> Deployment の設定がデフォルトのテキストエディタ（viなど）で出てくる

# 削除

## pod の削除

```shell
k delete pod <pod名>
```

## Deployment の削除


```shell
k deployments.apps <デプロイ名（プレフィックス部分）>
```

-> 成功すると、そのデプロイの配下に紐づく pod も消える

# レプリカセット

Deployment が直接 pod を作るわけではなく、Deployment は レプリカセットを作りそれを管理する。
手動で直接レプリカセットを作ることは推奨されない（Deployment に管理を任せることを推奨）


レプリカセットの状態は以下のコマンドで参照できる

```shell
k get replicasets.apps
```

```
NAME              DESIRED   CURRENT   READY   AGE
test-6546ccdcf9   10        10        10      102s
```

さらに、以下のコマンドで該当のレプリカセットの詳細を確認できる

```shell
k describe replicasets.apps test-6546ccdcf9(レプリカセット名)
```

# ローリングアップデート

yamlの例

```yaml
spec:
...
  strategy:
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
    type: RollingUpdate

```

- maxSurge - replica の数を超えて立てていい pod の数または割合
- maxUnavailable - replica の中で unavailable になっていい podの数または割合


-> type を Recreate にするとすべての pod が一気につぶれて作成される

