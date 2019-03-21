# これはなに？
[Kubernetes tutorials](https://kubernetes.io/docs/tutorials/)を試すリポジトリです。

## 使い方
各章に合わせてディレクトリを切っています。ものによっては空のことがあります。

とりあえず、見よう見まねでやっています。
 [3_stateless_guestbook_redis ](./3_stateless_guestbook_redis)にはREADMEをつけていて、そのとおりやると、ローカルないしGoogle Kubernetes Engine にクラスタをデプロイできるかと思います。

## 目標
Railsの開発環境とProduction実行基盤をすぐに作れるようにしたい。
実行環境はこんなイメージ
  - minikube
    - 開発環境
  - GKE
    - 開発中デモ
    - テスト
    - 本番

### ここまでやってみてわかったこと
- "Railsの.."というのは不要。そこはcontainerに閉じてる。[ruby-bookshelfのチュートリアル](./5_Ruuning_Ruby_Bookshelf/README.md)を参照。
  - 具体的には、deployment.spec.template.spec.containers.image に開発したRubyのContainerを指定するだけ(のはず..)
- データの永続化が必要な場合は、
  - PersistentVolumeClaim(永続ボリューム請求)を使う。これはDynamic Provisioning機能を利用する場合は、PVCが適用された段階で、PersistentVolumeが動的に適用される。
  - GKEの場合は、CloudSQLも使える **!要調査!**
    - [./gke-tutorials/cloudsql](./gke-tutorials/cloudsql)が使えるかもしれない
  - GKEの場合は、Cloud Datastoreを使える
    - gcs_bucketへのつなぎ込みは設定値をどうにかする
- 公開したいとき、
  - serviceを使って、外部IPの割当を待つ。(type: LoadBalancer)
    - domainとはどうやって？ **!要調査!**
    - https化はどうやって？ **!要調査!**
- コードベースの変更はどうやって取り込む?
  - CI **!要調査!**
    - git.masterに変更があったら、コードを取り込んで新しいimageをビルドしてって感じなのかな？ **!要調査!** (dockerのほうかな？)



---
