# Kubernetes のネットワーク

## pod レベルのネットワーク

基本設定での振る舞い

- pod ごとにIPアドレスがふられる
  - Kubernetes が内部IPのプールを持っていて、pod作成時にその中から自動でIPが割り振られ、podが消えるとIPも解放される
- デフォルトでは、pod同士はお互いに通信できるようになっている
  - 各ノードがIPテーブルを持っている（自動作成）
- pod　内のコンテナはローカルホストを通じてお互いに通信できる

ネットワークの設定に関する詳細ドキュメントはこちら
https://github.com/kubernetes/design-proposals-archive/blob/main/network/networking.md

## コンテナレベルのネットワーク

同じpodの中のコンテナには、それぞれ異なるポートを振る必要がある。

## CNI プラグイン

CNIはContainer Network Interfaceの略で、ネットワーク接続方法をプラグインとして提供する規格。

CNIプラグインには様々な選択肢がある。たとえば以下。

- Cilium
- Calico
- Flannel

https://www.netstars.co.jp/kubestarblog/k8s-3/

AKS の場合は Azure CNI, Kubenet, Bring my own CNI などが選べる。

### どのCNIを使っているかの確認

pod ではなく 管理ツール側でシェルをたたく。

たとえば Rancher Desktop の場合なら以下。

```
rdctl shell bash
```

この状態で /etc/cni 以下のネットワーク関係のディレクトリ（net.d など）を見ることで、何を使っているかがだいたいわかる

```
tree /etc/cni/
```

以下のような表示が出てきたら、flannelなんだなとわかる

```
.
└── net.d
    └── 10-flannel.conflist

```

conflist の中身を cat すると、以下のような定義が記述されている。

```
{
  "name":"cbr0",
  "cniVersion":"0.3.1",
  "plugins":[
    {
      "type":"flannel",
      "delegate":{
        "hairpinMode":true,
        "forceAddress":true,
        "isDefaultGateway":true
      }
    },
    {
      "type":"portmap",
      "capabilities":{
        "portMappings":true
      }
    }
  ]
}
```

## Service

[公式ドキュメント](https://kubernetes.io/ja/docs/concepts/services-networking/service/)の定義では「KubernetesにおけるServiceとは、 クラスター内で1つ以上のPodとして実行されているネットワークアプリケーションを公開する方法です」。

以下の説明がわかりやすいかも。

>Kubernetesにおいて、ServiceはPodの論理的なセットや、そのPodのセットにアクセスするためのポリシーを定義します(このパターンはよくマイクロサービスと呼ばることがあります)。 ServiceによってターゲットとされたPodのセットは、たいてい セレクターによって定義されます。 その他の方法について知りたい場合はセレクターなしのServiceを参照してください。

Service を必要とするモチベーションとして、Pod は短命であり、永遠に存在するわけではないという前提がある。

→ IPアドレスが変化し続けるのを、どう keep track するか？


### 例で理解する

たとえば Kubernetes 上で Web アプリケーションをホストする場合で、Frontend という Service と Database という Service を作るとする。

Frontend に属する pod　のIPは Frontend Service が、 Database に属する pod の IP は Database Service が管理する。

LB的な役割も Service が担うようだ。


## expose コマンド

以下のコマンドを使うと、Service が frontend という名前の Deployment に 8080 ポートからアクセスできるようにしてくれる

```shell
k expose deployment frontend --port 8080
```

この状態で `k get service` をたたくと、以下のような出力が返ってくる

```
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
frontend     ClusterIP   10.43.216.139   <none>        8080/TCP   114s
kubernetes   ClusterIP   10.43.0.1       <none>        443/TCP    12
```

`-o wide` をつけるとこのようになる


```
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE     SELECTOR
frontend     ClusterIP   10.43.216.139   <none>        8080/TCP   2m56s   app=frontend
kubernetes   ClusterIP   10.43.0.1       <none>        443/TCP    12d     <none>
```

ClusterIP - Service を作って expose すると、クラスタにIPがつく
-> このIPがLBのIPのように機能する（実際には外部IPがないのでLBとは違う）

NodePort という Service もある
-> これを使うとNodeにIPをつけ、特定のPortをあけて特定のNodeにアクセスすることができる
-> あまり使わない

AKS の場合は AzureLBを作ってそいつが Nodes へのルーティングを行う
-> K3s や Rancher Desktop でもLBは使える

以下のコマンドで、podではなくサービス単位でのポートフォワードができる

```shell
k port-forward service/frontend 8080
```

-> localhost:8080 で frontend サービスにアクセスができる状態

## ClusterIP から LB への切り替え

以下のコマンドで Service の設定を編集する（svcは Service の略）

```
k edit svc <Service名>
```

`spec` 配下の `type` を `ClusterIP` から `LoadBalancer` に変える


この状態で `k get svc` をたたくと以下のような出力になる

```
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)          AGE
frontend     LoadBalancer   10.43.216.139   192.168.205.2   8080:31073/TCP   15m
kubernetes   ClusterIP      10.43.0.1       <none>          443/TCP          12d
```

-> `TYPE` が `LoadBalancer` になって外部IPがふられている！


## ポートフォワードの限界

ポートフォワードコマンドを終了した瞬間にポートフォワードがされなくなってしまう

## Service の定義ファイルを yaml に書き出す

```
k get svc <サービス名> -o yaml
```

## LoadBalancer を使う定義ファイルの例

```
apiVersion: v1
kind: Service
metadata:
  labels:
    app: mealie
  name: mealie
  namespace: mealie
spec:
  ports:
  - port: 9000
    protocol: TCP
    targetPort: 9000
  selector:
    app: mealie
  type: LoadBalancer
```


## yaml ファイルから service を作る

Service を定義したファイル名が service.yaml の場合

```
k apply -f service.yaml
```