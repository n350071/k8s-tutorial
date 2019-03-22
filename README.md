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
  - GKEの場合は、CloudSQLも使える
    - その場合は、minikubeではどうやって動くのだろう?
    - コンソールで[SQL](https://console.cloud.google.com/sql/instances)を選択すると、インスタンスを作れるっぽい。おそらく、Cloud Datastoreなどと同じように、設定値でつなぎこむのだろう
  - GKEの場合は、Cloud Datastoreを使える
    - gcs_bucketへのつなぎ込みは設定値をどうにかする
- 公開したいとき、
  - serviceを使って、外部IPの割当を待つ。(type: LoadBalancer)
    - domainとはどうやって？ [Configure static IP and a domain name for your application.](https://cloud.google.com/kubernetes-engine/docs/tutorials/configuring-domain-name-static-ip)
    - https化はどうやって？
      - 参考になりそう→[GKEクラスタに小さなWebアプリケーションを配備して、ついでにhttps化してみる](https://blog.a-know.me/entry/2018/06/24/224424)
- コードベースの変更はどうやって取り込む?
  - CI
    - [GitOps-style continuous delivery with Cloud Build](https://cloud.google.com/kubernetes-engine/docs/tutorials/gitops-cloud-build)
      - アプリとインフラのリポジトリは分けて管理
      - "environments-as-code"に基づいている
      - CI/CD pipeline that automatically builds a container image from committed code, stores the image in Container Registry, updates a Kubernetes manifest in a Git repository, and deploys the application to Google Kubernetes Engine using that manifest.
        - Cloud Build, Container Registry が必要っぽい
    - [Continuous Delivery Pipelines with Spinnaker and Google Kubernetes Engine](https://cloud.google.com/solutions/continuous-delivery-spinnaker-kubernetes-engine)


### 実際にどうやるか？
- 第一段階
  - Railsアプリをminikubeで動かせる(hello yay)
    - [nao0515ki/docker-for-rails](https://github.com/nao0515ki/docker-for-rails)を応用して、ralis newした状態のものを用意しよう(そもそも docker-composeじゃないしね)
      - k8s-for-rails リポジトリを作ってもいいかもしれない
    - deploymentのimageはどうしよう？
      - コードをdocker buildして、それを使う感じかな？
        - ローカルのimageをpullして使ってくれるかな？
        - 開発環境では、skaffoldを使うといいとか..
    - postgressのところは、mysqlに置き換える
    - mysqlへの接続情報のところは、pvcなどを使っていい感じにする
- 第二段階
  - Railsアプリのコードベースを変更したら、それがminikube上で反映されていることが確認できる
- 第三段階
  - RailsアプリをGKEにデプロイして公開できる
    - 基本的には、プロジェクトを作って、クラスタを作ったら..
    - あとは同じkubectlコマンドで済むはず
- 第四段階
  - RailsアプリのIPを固定化し、domainを与え、https化する
  - 手順をメモしておいて、すぐに再現できるようにする
- 第五段階
  - GitOps式のCICD pipeline を実現する
  - この段階まで、リポジトリ内にマニフェストが混在していてもいい






---
