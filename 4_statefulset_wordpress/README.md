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













---
