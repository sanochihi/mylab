# 作成

## yamlの定義から pod を作る

```
k apply -f [yamlのファイルパス]
```

# 管理

## podの状態を見る(簡略)

```
k get pods
```

```
NAME            READY   STATUS    RESTARTS   AGE
httpd-chihiro   1/1     Running   0          6d20h
nginx-chihiro   1/1     Running   0          6d20h
```

## podの状態を見る(詳細)

```
k get pods -o wide
```

```
NAME            READY   STATUS    RESTARTS   AGE     IP           NODE                   NOMINATED NODE   READINESS GATES
httpd-chihiro   1/1     Running   0          6d20h   10.42.0.22   lima-rancher-desktop   <none>           <none>
nginx-chihiro   1/1     Running   0          6d20h   10.42.0.21   lima-rancher-desktop   <none>           <none>
```

-> ここで表示されたIPアドレスに対して、pod 同士で ping を通したり curl をたたいたりすることができる

# コマンドの実行

## podに入って bash でコマンドを実行する

```
k exec -it [pod名] -- /bin/bash
```

抜けるには `exit`



# 削除

## pod の削除

```
k delete pod [pod名]
```

