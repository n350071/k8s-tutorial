## minikube

1. Secretを作成
```
kubectl create secret generic mysql-pass --from-literal=password=mysqlpass
```

2. mysqlとwordpressをデプロイ
```
kubectl apply -f mysql-deployment.yaml
kubectl create -f wordpress-deployment.yaml
```

3. pvcが立ち上がっているのを確認
```
kubectl get pvc
```

4. 外部IPを与える
得られたIPに接続して動作確認する
```
kubectl get services
minikube service wordpress --url
```

5. クリーンアップ
```
kubectl delete secret mysql-pass
kubectl delete deployment -l app=wordpress
kubectl delete service -l app=wordpress
kubectl delete pvc -l app=wordpress
```

## GKE

- ここで使うファイルの位置
  - [../gke-tutorials/wordpress-persistent-disk](../gke-tutorials/wordpress-persistent-disk)
  - clone元 : https://github.com/GoogleCloudPlatform/kubernetes-engine-samples
  - `cd ../gke-tutorials/wordpress-persistent-disk` を推奨
- その他
  - 公式では `create` と書いているところを `apply` にしています。


1. クラスタの作成
```
gcloud container clusters create persistent-disk-tutorial --num-nodes=3
```

2. ファイルのコピー
```
git submodule add https://github.com/GoogleCloudPlatform/kubernetes-engine-samples gke-tutorials
```

3. PVCの作成
- persistent Volume がなければ、作成される
- StorageClassが指定されていなければ、default StorageClass が適用される(今回は指定ないので、デフォルトが適用される)
- 実行すると、 kubernetes engine のstrageのところに出来上がる(`kubectl get pvc` でも確認できる)
```
kubectl config current-context
kubectl apply -f mysql-volumeclaim.yaml
kubectl apply -f wordpress-volumeclaim.yaml
kubectl get pvc
```

4. MySQLで使うSecretと、MySQLのDeployment作成
- Secretは`YOUR_PASSWORD`のところを書き換えて実行
- mysqlは実行
```
kubectl create secret generic mysql --from-literal=password=YOUR_PASSWORD
kubectl create -f mysql.yaml
```

5. MySQLのサービスを作成
```
kubectl apply -f mysql-service.yaml
kubectl get service mysql
```

6. Wordpressのdeployを作成
`WORDPRESS_DB_HOST`の`mysql:3306`の`mysql`は↑で作ったサービスと、kubernetes DNSによって名前解決される。
```
kubectl apply -f wordpress.yaml
kubectl get pod -l app=wordpress
```

7. Wordpressサービスを外部に晒す
```
kubectl apply -f wordpress-service.yaml
kubectl get services
```
LoadBalancerのExternalIPを待って、そこにアクセスする。
なお、セットアップ中の状態で放置しないこと。誰かが乗っ取り悪意のある情報を載せるかもしれない。

8. (スルー可)永続化データのテスト
ためしに、mysqlのpodを破壊してみる。そうすると、k8sがmysqlのdeploymentを再スケジュールし、永続化データとの接続を再開する。

```
kubectl get pods -o=wide
kubectl delete pod -l app=mysql
kubectl get pods -o=wide
```

9. アプリのimageを更新する
アプリを開発してバージョンアップしていくことは大事なことだ。
そこで、wordpressの最新のバージョンを [Docker Hub](https://hub.docker.com/_/wordpress)から探して、wordpress.yamlのimageの値を書き換えて、適用(apply)してみよう。
```
kubectl apply -f wordpress.yaml
```

ここで問題が..,
`wordpress.yaml`が利用している`wordpress-volumeclaim`は`accessModes: ReadWriteOnce`なのである。

`kubectl describe pod wordpress`をやってみるとわかるが、
```
Events:
  Type     Reason              Age                 From                                                          Message
  ----     ------              ----                ----                                                          -------
  Normal   Scheduled           9m9s                default-scheduler                                             Successfully assigned default/wordpress-8648c887dc-2hcxf to gke-persistent-disk-tuto-default-pool-f0cd3ac1-htwh
  Warning  FailedAttachVolume  9m9s                attachdetach-controller                                       Multi-Attach error for volume "pvc-740f57ab-496c-11e9-aa81-42010a92023f" Volume is already used by pod(s) wordpress-78c9b8d684-vr9zd
  Warning  FailedMount         21s (x4 over 7m6s)  kubelet, gke-persistent-disk-tuto-default-pool-f0cd3ac1-htwh  Unable to mount volumes for pod "wordpress-8648c887dc-2hcxf_default(2a78750e-4986-11e9-aa81-42010a92023f)": timeout expired waiting for volumes to attach or mount for pod "default"/"wordpress-8648c887dc-2hcxf". list of unmounted volumes=[wordpress-persistent-storage]. list of unattached volumes=[wordpress-persistent-storage default-token-9swck]
  ```

  というように、  `Multi-Attach error` となる。
  そこで、`wordpress.yaml`に↓のように`strategy:`の部分を追加する。こうすると、まず生きているコンテナを壊して、瞬断があったのちにワードプレスが復活するのである。
  ```
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: wordpress
    strategy:
      type: RollingUpdate
      rollingUpdate:
        maxUnavailable: 1
        maxSurge: 0
    template:
      metadata:
        labels:
          app: wordpress
```

### GKEをpreemptibleインスタンスを使って安くする
[安価なGKE（k8s）クラスタを作って趣味開発に活用する](https://blog.a-know.me/entry/2018/06/17/220222) を参考にさせていただきました。

せっかく作ったので、 `gcloud container clusters update --preemptible` できないか？と思いましたが、 [ダメの様子](https://cloud.google.com/sdk/gcloud/reference/container/clusters/update)です。

1. preemptibleクラスタの作成
```
gcloud container clusters create preemptible-persistent-disk-tutorial --preemptible --machine-type=f1-micro --num-nodes=3 --disk-size=10 --enable-autorepair
```

- `--preemptible` : 価格が安い代わりに、最大起動時間が24時間だったり、保証がなかったりする。詳細は[こちら](https://cloud.google.com/kubernetes-engine/docs/how-to/preemptible-vms)
- `--machine-type=f1-micro` : 0.2 基の vCPU と 0.60 GB のメモリを備え、共有物理コアを基盤としたマイクロ マシンタイプ (バースト機能あり) 詳細は[こちら](https://cloud.google.com/compute/docs/machine-types?hl=ja)

作ろうとしたが、すでに２個も作っていて、`Compute Engine API In-use IP addresses` が足りなくなったようで、
```
ERROR: (gcloud.container.clusters.create) ResponseError: code=403, message=Insufficient regional quota to satisfy request: resource "IN_USE_ADDRESSES": request requires '3.0' and is short '1.0'. project has a quota of '8.0' with '2.0' available. View and manage quotas at https://console.cloud.google.com/iam-admin/quotas?usage=USED&project=labo-shirohune.
```
というエラーが出る。

先にクラスタを削除する
```
gcloud container clusters delete persistent-disk-tutorial --async
```
`--async` で待たずに削除。
GUI上では、まだ残っているが、作成できた。

2. 同クラスタに、preemptibleでないノードインスタンスのサブセットを作る
- クラスタ
  - デフォルトクラスタ ←さっき作った(いつ壊れるともわからない)
  - これから作るクラスタ ←(保証ある)

```
gcloud container node-pools create not-preemptible-pool --cluster preemptible-persistent-disk-tutorial --machine-type=f1-micro --num-nodes=1 --disk-size=10
```

確認
```
gcloud container node-pools list --cluster preemptible-persistent-disk-tutorial

NAME                  MACHINE_TYPE  DISK_SIZE_GB  NODE_VERSION
default-pool          f1-micro      10            1.11.7-gke.4
not-preemptible-pool  f1-micro      10            1.11.7-gke.4
```

3. 以降
上記の"PVCの作成"に戻る

4. 問題
MySQLが ` Does not have minimum availability` で起動できない..
おそらくは、マシンタイプを指定したから、デフォルトのものより小さくなったことが原因なのだろう。

同じプロジェクト内にある、もう１つのクラスタを削除してみる。（クラスタの能力が足りないのだから意味がないかもしれないが）

ノード数を変えてみる
```
gcloud container clusters resize preemptible-persistent-disk-tutorial --size=4 --node-pool=default-pool
```

プロジェクトを作り直して、作業してみたが変わらなかった。
そこで、マシンタイプをデフォルトのプールを加えてみようと思う。
```
gcloud container node-pools create preemptible-n1-standard-1-pool --cluster preemptible-persistent-disk-tutorial --machine-type=n1-standard-1 --num-nodes=3 --disk-size=10 --preemptible
```

---
