https://cloud.google.com/ruby/tutorials/bookshelf-on-kubernetes-engine

---

## 始める前に..
1. プロジェクトをつくり、支払い設定をすませ、
2. [APIの有効化](https://console.cloud.google.com/flows/enableapi?apiid=datastore.googleapis.com,container.googleapis.com,pubsub,storage_api,logging,plus.googleapis.com&_ga=2.141868978.-1854617211.1547984722)を行う

## はじめる
1. クラスタをつくる
```
gcloud container clusters create bookshelf \
    --scopes "cloud-platform,userinfo-email" \
    --num-nodes 2 \
    --enable-basic-auth \
    --issue-client-certificate \
    --enable-ip-alias \
    --zone asia-northeast1-b
```
zone情報は自分のものに置き換える
[GCP ZONE](https://cloud.google.com/compute/docs/regions-zones/#available)


クラスタにアクセスできることを確認する
```
kubectl get nodes
```

2. リポジトリをコピーする

https://github.com/GoogleCloudPlatform/getting-started-ruby
つかうのは `optional-kubernetes-engine` のディレクトリ

3. Cloud Datastore の初期化
https://console.cloud.google.com/datastore/entities/query/kind?project=ruby-bookshelf-shirofune
https://cloud.google.com/datastore/

View Console → Create Data Store → Reagion = tokyo
"Create Entry"が表示されるまで進む

4. Cloud Storage Bucket の作成
→ https://console.cloud.google.com/storage/browser?project=ruby-bookshelf-shirofune&folder&organizationId
(Cloud Strage は AWS S3のようなものっぽい)
そこへに写真などのデータを保存するときに使うのが、
Bucketコンテナで、これを通じてデータ保存できるようだ。

Cloud Storage Bucket の名前は好きに決められるけれど、
プロジェクトIDと一致させておくのがいいようだ。

クラウドストレージを作成し、
```
$gsutil mb gs://ruby-bookshelf-shirofune
Creating gs://ruby-bookshelf-shirofune/...
```

defacl(access control定義)で、public-readを設定
```
$gsutil defacl set public-read gs://ruby-bookshelf-shirofune
Setting default object ACL on gs://ruby-bookshelf-shirofune/...
```

5. a Cloud Pub/Sub topic and subscription の作成
- topicはパブリッシャー(送信元?)からみたPub/Subの送信先
- subscriptionはtopicからメッセージを受け取りサブスクライバーへ送信する元の窓口

```
$gcloud pubsub topics create topic-ruby-bookshelf-shirofune
Created topic [projects/ruby-bookshelf-shirofune/topics/topic-ruby-bookshelf-shirofune].

$gcloud pubsub subscriptions create --topic topic-ruby-bookshelf-shirofune subscription-ruby-bookshelf-shirofune
Created subscription [projects/ruby-bookshelf-shirofune/subscriptions/subscription-ruby-bookshelf-shirofune].
```

6. アプリの設定をする
ファイルをコピーして、中身を書き換える。
```
cp config/database.example.yml config/database.yml
cp config/settings.example.yml config/settings.yml
```

project_idとか、↑で作ったバケットのIDなどを入力する。

oauth2.client_id, oauth2.client_secretがわからなかった。

解決策
https://console.developers.google.com/apis/credentials
ここから行き、 `Create OAuth client ID` を押す。
同意ページ(consent)を作成しろと言われたら作る。
基本的には、[このページ](https://developers.google.com/adwords/api/docs/guides/authentication?hl=ja#webapp)に従えばできるはず

(なお、redirect_uriを設定しなければならないが、domainを持っていないので、loginできずに失敗する)

7. アプリをコンテナ化する

```
docker build . -t gcr.io/ruby-bookshelf-shirofune/bookshelf
```
-t は、ビルド作成後に、リポジトリ名のタグをつける

```
gcloud docker -- push gcr.io/ruby-bookshelf-shirofune/bookshelf
```

8. フロントエンド, バックエンド, サービスをデプロイする
`bookshelf-frontend.yaml`のproject_idを書き換える
```
kubectl create -f bookshelf-frontend.yaml
kubectl get deployments
kubectl delete deployments bookshelf-frontend
kubectl get pods

kubectl create -f bookshelf-worker.yaml
kubectl get pods

kubectl create -f bookshelf-service.yaml
kubectl describe service bookshelf
```
