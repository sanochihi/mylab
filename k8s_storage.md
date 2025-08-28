# Kubernetes のストレージ

pod は短命だが、データを永続的に保存しておくレイヤーが必要
→ ローカルなどの永続的なストレージをマウントする

## Volume の確認

`k describe pod <pod名>` を実行すると、出力情報の中に以下の情報が入っている


```
  Volumes:
  kube-api-access-mvv9f:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    Optional:                false
    DownwardAPI:             true
```

## 短命な volume 

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-storage
  labels:
    app: nginx-storage
spec:
  containers:
    - name: nginx
      image: nginx
      volumeMounts:
        - name: scratch-volume
          mountPath: /scratch  # Default content directory
  volumes:
    - name: scratch-volume
      emptyDir:
        sizeLimit: 500Mi  # This is temporary storage; it is deleted when the Pod stops

```

この定義の中にある `emptyDir` は、podが消えたら消える volume。

`containers` 配下の `volumeMounts` で、どのコンテナがどこのボリュームにデータをマウントするべきかを定義している。

ここの `name` が `volumes` 配下の `name` と対応している

この yamlを `k apply -f` すると、出力結果の `Volumes` 配下はこんな感じになる。

```
Volumes:
  scratch-volume:
    Type:       EmptyDir (a temporary directory that shares a pod's lifetime)
    Medium:
    SizeLimit:  500Mi
```

`k exec -it nginx-storage --bash` で pod に入って ls すると、以下のような結果が出る。

```
bin  boot  dev  docker-entrypoint.d  docker-entrypoint.sh  etc  home  lib  media  mnt  opt  proc  root  run  sbin  scratch  srv  sys  tmp  usr  var
```

scratch ディレクトリがあるのは、上記の yaml で `/scratch` を定義していたから。
（デフォルトの nginx のテンプレートなどを使った場合は、このディレクトリは存在しない）

### コンテナ間で volume を共有する