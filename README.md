## これは何
kubernetesの基礎を試した記録です。

[your-project] : project name
[your-account] : google account id
[user-identifer] : 任意の名前

でドキュメント全体を置換してください。

## 準備
```
$ brew cask install docker
$ brew cask install google-cloud-sdk
$ brew install kubectl

$ gcloud config set compute/zone asia-northeast1-a
$ gcloud config set project [your-project]
$ gcloud config set account [your-account]
$ gcloud auth login
```

## kubernetes on gcp
### 1.準備 image を gcr に push
``` 
$ docker tag nginx:latest gcr.io/[your-project]/[user-identifer]/nginx:latest
$ docker tag nginx:1.14.0-alpine gcr.io/[your-project]/[user-identifer]/nginx:1.14.0-alpine

$ gcloud docker -- push gcr.io/[your-project]/[user-identifer]/nginx:latest
$ gcloud docker -- push gcr.io/[your-project]/[user-identifer]/nginx:1.14.0-alpine
```

### 2. nodeの作成
podsを配置するための箱のクラスタ. VM.

```sh
$ gcloud container clusters create sample-cluster \
                                     --machine-type g1-small \
                                     --disk-size=10 \
                                     --num-nodes=3

$ kubectl get nodes
NAME                                             STATUS    ROLES     AGE       VERSION
gke-suyamai-cluster-default-pool-93519135-d9f4   Ready     <none>    50m       v1.8.10-gke.0
gke-suyamai-cluster-default-pool-93519135-fnt6   Ready     <none>    50m       v1.8.10-gke.0
gke-suyamai-cluster-default-pool-93519135-wmjc   Ready     <none>    50m       v1.8.10-gke.0
```

### 3. Deploymentの作成
```sh
$ kubectl create -f deployment.yaml --save-config --record=true
deployment.apps "suyamai-deployment" created

$ kubectl get pods
NAME                                  READY     STATUS    RESTARTS   AGE
suyamai-deployment-7f8c7bc444-sc9j7   1/1       Running   0          12s
suyamai-deployment-7f8c7bc444-vn842   1/1       Running   0          12s
suyamai-deployment-7f8c7bc444-zj9w4   1/1       Running   0          12s

$ kubectl get deployments
NAME                 DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
suyamai-deployment   3         3         3            3           3m
```

### 4. Serviceの作成
作成した時点で外部公開される。（type=LoadBaranserのおかげ？）

```sh
$ kubectl create -f service.yaml --save-config --record=true

$ kubectl get services
NAME              TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
kubernetes        ClusterIP      10.11.240.1     <none>          443/TCP        1h
suyamai-service   LoadBalancer   10.11.253.161   35.200.86.154   80:31988/TCP   55s

⟩ curl 35.200.86.154
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
:
```

### 5. Deploymentの更新（rolling deploy)
replicaを満たすよう、順次新しいコンテナが起動,古いコンテナが停止する

```sh
⟩ kubectl apply -f deployment-rolling.yaml
deployment.apps "suyamai-deployment" configured

~/src/k8s-handson
⟩ kubectl get pods
NAME                                  READY     STATUS              RESTARTS   AGE
suyamai-deployment-7f8c7bc444-sc9j7   1/1       Running             0          7m
suyamai-deployment-7f8c7bc444-vn842   1/1       Running             0          7m
suyamai-deployment-7f8c7bc444-zj9w4   1/1       Running             0          7m
suyamai-deployment-9ff69b5d7-s4txk    0/1       ContainerCreating   0          5s
```

### 6. Blue/Greenデプロイ
serviceのselectorを切り替える。
（他にもやりようはありそう）

deploymentをblue/greenで作成

```sh
$ kubectl create -f deployment-blue.yaml --save-config --record=true
deployment.apps "suyamai-deployment-blue" created

$ kubectl create -f deployment-green.yaml --save-config --record=true
deployment.apps "suyamai-deployment-green" created

$ kubectl get dep
error: the server doesn't have a resource type "dep"

$ kubectl get deployments
NAME                       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
suyamai-deployment-blue    3         3         3            3           15s
suyamai-deployment-green   3         3         3            3           8s

⟩ kubectl get pods
NAME                                        READY     STATUS    RESTARTS   AGE
suyamai-deployment-blue-7d6446c555-2kggx    1/1       Running   0          47s
suyamai-deployment-blue-7d6446c555-rrfxl    1/1       Running   0          47s
suyamai-deployment-blue-7d6446c555-wlrcc    1/1       Running   0          47s
suyamai-deployment-green-57fcc4b545-6f4vq   1/1       Running   0          40s
suyamai-deployment-green-57fcc4b545-6klnb   1/1       Running   0          40s
suyamai-deployment-green-57fcc4b545-tkxds   1/1       Running   0          40s
```

serviceを作成してblueを公開

```sh
$ kubectl create -f service-blue.yaml --save-config --record=true
service "suyamai-service" created

$ kubectl get services
NAME              TYPE           CLUSTER-IP      EXTERNAL-IP     PORT(S)        AGE
kubernetes        ClusterIP      10.11.240.1     <none>          443/TCP        1h
suyamai-service   LoadBalancer   10.11.247.143   35.200.108.92   80:30643/TCP   1m

$ kubectl describe services
:
Selector:                 app=suyamai-app-blue
:
```

serviceを更新してgreenを公開
```sh
⟩ kubectl apply -f service-green.yml
service "suyamai-service" configured

~/src/k8s-handson · (master±)
⟩ kubectl describe service
:
Selector:                 app=suyamai-app-green
:
```

### 7. 削除
```
$ kubectl delete service suyamai-service
service "suyamai-service" deleted

$ kubectl delete deployment suyamai-deployment
deployment.extensions "suyamai-deployment" deleted

$ gcloud container clusters delete sample-cluster
```