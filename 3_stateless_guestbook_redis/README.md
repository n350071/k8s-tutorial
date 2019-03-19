## Minikube

1. VM上にk8sクラスタを作成
```
$ minikube start
```

2. Deploymentを作成
```
$ kubectl apply -f application/guestbook/redis-master-deployment.yaml
$ kubectl apply -f application/guestbook/redis-slave-deployment.yaml
$ kubectl apply -f application/guestbook/frontend-deployment.yaml
```

3. Serviceを作成
```
$ kubectl apply -f application/guestbook/redis-master-service.yaml
$ kubectl apply -f application/guestbook/redis-slave-service.yaml
$ kubectl apply -f application/guestbook/frontend-service.yaml
```

4. 外部IPを与える
得られたIPに接続して動作確認する
```
$ minikube service frontend --url
```

5. クリーンアップ
```
$ kubectl delete deployment -l app=redis
$ kubectl delete service -l app=redis
$ kubectl delete deployment -l app=guestbook
$ kubectl delete service -l app=guestbook
```

## GKE

0. 準備
```
gcloud projects create <PROJECT-ID>
gcloud projects list
gcloud config list
gcloud config set project <PROJECT-ID>
gcloud config get-value project
gcloud services enable container.googleapis.com
gcloud services list --available
```
支払い情報も入力(Billing設定)



1. GKE上にk8sクラスターを作成
```
$ gcloud container clusters create guestbook
$ kubectl config current-context
```
目的のプロジェクト、目的のクラスタになっているか確認する


2. Deploymentを作成
```
$ kubectl apply -f application/guestbook/redis-master-deployment.yaml
$ kubectl apply -f application/guestbook/redis-slave-deployment.yaml
$ kubectl apply -f application/guestbook/frontend-deployment.yaml
```

3. Serviceを作成
```
$ kubectl apply -f application/guestbook/redis-master-service.yaml
$ kubectl apply -f application/guestbook/redis-slave-service.yaml
$ kubectl apply -f application/guestbook/frontend-service-gke.yaml
```

4. 外部IPを与える
GCPは自動で与えられる..
EXTERNAL-IPの発行を待ち、`http://{EXTERNAL-IP}`を確認する
```
$ kubectl get services
NAME           TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
frontend       LoadBalancer   10.55.252.139   35.221.127.83   80:31136/TCP   1m
kubernetes     ClusterIP      10.55.240.1     <none>          443/TCP        19m
redis-master   ClusterIP      10.55.255.226   <none>          6379/TCP       1m
redis-slave    ClusterIP      10.55.240.236   <none>          6379/TCP       1m
```

5. スケール（ダウン）
```
$ kubectl scale deployment frontend --replicas=1
$ kubectl scale deployment redis-slave --replicas=1
```

6. クリーンアップ
```
$ kubectl delete deployment -l app=redis
$ kubectl delete service -l app=redis
$ kubectl delete deployment -l app=guestbook
$ kubectl delete service -l app=guestbook
```
