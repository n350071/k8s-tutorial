## minikube

1. Secretを作成
```
$ kubectl create secret generic mysql-pass --from-literal=password=mysqlpass
```

2. mysqlとwordpressをデプロイ
```
$ kubectl apply -f mysql-deployment.yaml
$ kubectl create -f wordpress-deployment.yaml
```

3. pvcが立ち上がっているのを確認
```
$ kubectl get pvc
```

4. 外部IPを与える
得られたIPに接続して動作確認する
```
$ kubectl get services
$ minikube service wordpress --url
```

5. クリーンアップ
```
$ kubectl delete secret mysql-pass
$ kubectl delete deployment -l app=wordpress
$ kubectl delete service -l app=wordpress
$ kubectl delete pvc -l app=wordpress
```

## GKE
