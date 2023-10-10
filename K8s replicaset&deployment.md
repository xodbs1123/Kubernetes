## 레플리카셋(replicaset) ##
https://kubernetes.io/ko/docs/concepts/workloads/controllers/replicaset/

- 일정 개수의 파드를 유지하는 컨트롤러
  - 정해진 수의 동일한 파드가 항상 실행되도록 관리
  - 노드 장애 등의 이유로 파드 사용 제한 시 다른 노드에 파드를 다시 생성

### 시크릿 생성 ###
- 도커 허브에서 이미지를 가져올 때 사용할 자격증명 정보를 저장할 시크릿 생성

```
vagrant@master-node:~$ kubectl create secret docker-registry regcred --docker-server=https://index.docker.io/v1/ --docker-username=tyunme --docker-password="qntuqjfu3!" --docker-email=xosbs1123@gmail.com

secret/regcred created

vagrant@master-node:~$ kubectl get secret regcred --output=yaml
apiVersion: v1
data:
  .dockerconfigjson: eyJhdXRocyI6eyJodHRwczovL2luZGV4LmRvY2tlci5pby92MS8iOnsidXNlcm5hbWUiOiJ0eXVubWUiLCJwYXNzd29yZCI6InFudHVxamZ1MyEiLCJlbWFpbCI6Inhvc2JzMTEyM0BnbWFpbC5jb20iLCJhdXRoIjoiZEhsMWJtMWxPbkZ1ZEhWeGFtWjFNeUU9In19fQ==
kind: Secret
metadata:
  creationTimestamp: "2023-10-06T02:40:04Z"
  name: regcred
  namespace: default
  resourceVersion: "14029"
  uid: 9f35b94b-1ec0-471b-b988-bdb23cf1a094
type: kubernetes.io/dockerconfigjson
```
### 파드를 삭제하면 파드 내 컨테이너도 함께 삭제됨 ###
- /home/vagrant/nginx-pod-with-ubuntu.yaml 작성
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx-pod
spec:
  containers:
  - name: my-nginx-container
    image: docker.io/nginx:latest              <<< 레지스트리를 명시
    ports:
    - containerPort: 80
      protocol: TCP
  - name: ubuntu-sidecar-container
    image: docker.io/alicek106/rr-test:curl    <<< 레지스트리를 명시
    command: ["tail"]
    args: ["-f", "/dev/null"]
  imagePullSecrets:                            <<< 이미지를 가져올 때 사용할 크리덴셜을 지정
  - name: regcred 
```
- yaml 파일 apply
```
vagrant@master-node:~$ kubectl apply -f nginx-pod-with-ubuntu.yaml
pod/my-nginx-pod created

vagrant@master-node:~$ kubectl get pod
NAME           READY   STATUS    RESTARTS   AGE
my-nginx-pod   2/2     Running   0          20m

vagrant@master-node:~$ kubectl get pod -o wide
NAME           READY   STATUS    RESTARTS   AGE   IP              NODE            NOMINATED NODE   READINESS GATES
my-nginx-pod   2/2     Running   0          20m   172.16.158.17   worker-node02   <none>           <none>

vagrant@master-node:~$ kubectl delete -f nginx-pod-with-ubuntu.yaml	⇐ 파드를 삭제하면 컨테이너도 함께 종료(삭제)
pod "my-nginx-pod" deleted								

```

### 동일한 컨테이너 여러개를 실행하는 경우 ⇒ 동일한 파드를 여러개 정의해서 실행 ###
- **파드가 삭제되거나, 파드가 위치한 노드에 장애가 발생해서 파드에 접근하지 못하는 경우, 관리자가 직접 파드를 삭제하고 다시 생성해야 하기 때문**
- yaml 파일 수정
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx-pod-a				<<< 
spec:
  containers:
  - name: my-nginx-container
    image: docker.io/nginx:latest
    ports:
    - containerPort: 80
      protocol: TCP
  imagePullSecrets:
  - name: regcred
---
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx-pod-b				<<<
spec:
  containers:
  - name: my-nginx-container
    image: docker.io/nginx:latest
    ports:
    - containerPort: 80
      protocol: TCP
  imagePullSecrets:
  - name: regcred  
```

- yaml 파일 apply
```
vagrant@master-node:~$ kubectl apply -f nginx-pod-with-ubuntu.yaml
pod/my-nginx-pod-a created
pod/my-nginx-pod-b created

vagrant@master-node:~$ kubectl get pod -o wide
NAME             READY   STATUS    RESTARTS   AGE   IP              NODE            NOMINATED NODE   READINESS GATES
my-nginx-pod-a   1/1     Running   0          24s   172.16.158.19   worker-node02   <none>           <none>
my-nginx-pod-b   1/1     Running   0          24s   172.16.158.18   worker-node02   <none>           <none>
```

### 레플리카셋 정의 ###
/home/vagrant/replicaset-nginx.yaml
```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replicaset-nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-nginx-pods-label
  template:
    metadata:
      name: my-nginx-pod
      labels:
        app: my-nginx-pods-label
    spec:
      containers:
      - name: my-nginx-container
        image: docker.io/nginx:latest
        ports:
        - containerPort: 80
          protocol: TCP
      imagePullSecrets:
      - name: regcred
```
### 레플리카셋 생성 및 확인 ###
```
vagrant@master-node:~$ kubectl apply -f replicaset-nginx.yaml

vagrant@master-node:~$ kubectl get pod -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP              NODE            NOMINATED NODE   READINESS GATES
replicaset-nginx-47mlh   1/1     Running   0          29s   172.16.158.3    worker-node02   <none>           <none>
replicaset-nginx-lsnh5   1/1     Running   0          29s   172.16.87.194   worker-node01   <none>           <none>
replicaset-nginx-x72p8   1/1     Running   0          29s   172.16.87.193   worker-node01   <none>           <none>
```
- 레플리카셋 확인
```
vagrant@master-node:~$ kubectl get rs -o wide
NAME               DESIRED   CURRENT   READY   AGE    CONTAINERS           IMAGES                   SELECTOR
replicaset-nginx   3         3         3       102s   my-nginx-container   docker.io/nginx:latest   app=my-nginx-pods-label
```

### 파드의 개수 늘려서 실행 ###
- 매니패스트 파일을 수정
```
vagrant@master-node:~$ cp replicaset-nginx.yaml replicaset-nginx-4pods.yaml
```
``` yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replicaset-nginx
spec:
  replicas: 4
  selector:
    matchLabels:
      app: my-nginx-pods-label
  template:
    metadata:
      name: my-nginx-pod
      labels:
        app: my-nginx-pods-label
    spec:
      containers:
      - name: my-nginx-container
        image: docker.io/nginx:latest
        ports:
        - containerPort: 80
          protocol: TCP
      imagePullSecrets:
      - name: regcred
```
- yaml 파일 실행 및 생성 확인
```
vagrant@master-node:~$ kubectl apply -f replicaset-nginx-4pods.yaml
replicaset.apps/replicaset-nginx configured

vagrant@master-node:~$ kubectl get pods
NAME                     READY   STATUS    RESTARTS   AGE
replicaset-nginx-2k64c   1/1     Running   0          17s      <= 하나의 파드가 추가됨
replicaset-nginx-47mlh   1/1     Running   0          5m57s
replicaset-nginx-lsnh5   1/1     Running   0          5m57s
replicaset-nginx-x72p8   1/1     Running   0          5m57s
```

### scale 명령으로 replicas 수정 ###
```
vagrant@master-node:~$ kubectl scale replicaset replicaset-nginx --replicas=5
replicaset.apps/replicaset-nginx scaled

vagrant@master-node:~$ kubectl get pod
NAME                     READY   STATUS    RESTARTS   AGE
replicaset-nginx-2k64c   1/1     Running   0          113s
replicaset-nginx-47mlh   1/1     Running   0          7m33s
replicaset-nginx-4q6hq   1/1     Running   0          5s
replicaset-nginx-lsnh5   1/1     Running   0          7m33s
replicaset-nginx-x72p8   1/1     Running   0          7m33s
vagrant@master-node:~$
```

### edit 명령으로 속성 수정 ###
```
vagrant@master-node:~$ kubectl edit replicaset replicaset-nginx
```
```
spec:
  replicas: 6      <= 4에서 6으로 수정
  selector:
    matchLabels:
      app: my-nginx-pods-label
```
```
vagrant@master-node:~$ kubectl get pods

NAME                     READY   STATUS    RESTARTS   AGE
replicaset-nginx-6mndc   1/1     Running   0          6m55s
replicaset-nginx-cst8t   1/1     Running   0          2m3s	⇐ 하나의 파드가 추가
replicaset-nginx-hdqlk   1/1     Running   0          12m
replicaset-nginx-nxzk9   1/1     Running   0          12m
replicaset-nginx-t9pvx   1/1     Running   0          12m
replicaset-nginx-wr6vk   1/1     Running   0          5m1s
```

### 레플리카셋 삭제 ###
- 레플리카셋 이름을 이용해서 삭제하는 방법
  - 레플리카셋에 의해 생성된 파드로 함께 삭제됨
```
vagrant@master-node:~$ kubectl delete rs replicaset-nginx
replicaset.apps "replicaset-nginx" deleted

vagrant@master-node:~$ kubectl get rs,pod
No resources found in default namespace.
```
- yaml 파일을 이용해서 삭제하는 방법
```
vagrant@master-node:~$ kubectl delete -f replicaset-nginx-4pods.yaml
replicaset.apps "replicaset-nginx" deleted

vagrant@master-node:~$ kubectl get pod,rs
No resources found in default namespace.
```
- 파드는 유지하고 레플리카셋만 삭제하는 방법
```
vagrant@master-node:~$ kubectl delete -f replicaset-nginx-4pods.yaml --cascade=orphan
replicaset.apps "replicaset-nginx" deleted

vagrant@master-node:~$ kubectl get pod,rs
NAME                         READY   STATUS    RESTARTS   AGE
pod/replicaset-nginx-lx6bs   1/1     Running   0          80s
pod/replicaset-nginx-nh5w5   1/1     Running   0          80s
pod/replicaset-nginx-sxnjn   1/1     Running   0          80s
pod/replicaset-nginx-w7s5g   1/1     Running   0          80s
```

## 레플리카셋의 동작 원리 ##
- 라벨 셀렉터(Label Selector)를 이용해서 유지할 파드를 정의
- 레플리카셋은 spec.selector.matchLabels에 정의된 라벨을 통해 생성해야 하는 파드를 찾음 -> app: my-nginx-pods-label 라벨을 가지는 파드의 개수가 replicas 항목에 정의된 숫자보다 적으면 파드를 정의하는 파드 템플릿(template) 항목의 내용으로 파드를 생성


### 레플리카셋 생성전 app: my-nginx-pods-label 라벨을 가지는 파드 생성 ###
- /home/vagrant/nginx-pod-without-rs.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-nginx-pod
  labels:
    app: my-nginx-pods-label
spec:
  containers:
  - name: my-nginx-container
    image: docker.io/nginx:latest
    ports:
    - containerPort: 80
      protocol: TCP
  imagePullSecrets:
  - name: regcred
```

### 파드 생성 및 확인 ###
```
vagrant@master-node:~$ kubectl apply -f nginx-pod-without-rs.yaml
pod/my-nginx-pod created

vagrant@master-node:~$ kubectl get pod --show-labels
NAME           READY   STATUS    RESTARTS   AGE   LABELS
my-nginx-pod   1/1     Running   0          10s   app=my-nginx-pods-label
```
### app: my-nginx-pods-label 라벨을 가지는 파드 3개를 생성하는 레플리카셋 실행 ###
- /home/vagrant/replicaset-nginx.yaml
- 두 개의 파드만 생성되는 것을 확인할 수 있음
```yaml
vagrant@master-node:~$ kubectl apply -f replicaset-nginx.yaml
replicaset.apps/replicaset-nginx created

vagrant@master-node:~$ kubectl get pod --show-labels
NAME                     READY   STATUS    RESTARTS   AGE     LABELS
my-nginx-pod             1/1     Running   0          2m25s   app=my-nginx-pods-label
replicaset-nginx-kbbwr   1/1     Running   0          11s     app=my-nginx-pods-label
replicaset-nginx-vpf8f   1/1     Running   0          11s     app=my-nginx-pods-label
```
### 수동으로 생성한 파드를 삭제 ###
- 레플리카셋이 새로운 파드를 생성해주는 모습
```
vagrant@master-node:~$ kubectl delete pod my-nginx-pod
pod "my-nginx-pod" deleted

vagrant@master-node:~$ kubectl get pod --show-labels
NAME                     READY   STATUS    RESTARTS   AGE     LABELS
replicaset-nginx-kbbwr   1/1     Running   0          2m32s   app=my-nginx-pods-label
replicaset-nginx-vpf8f   1/1     Running   0          2m32s   app=my-nginx-pods-label
replicaset-nginx-wmk88   1/1     Running   0          10s     app=my-nginx-pods-label
```

### 레플리카셋이 생성한 파드의 라벨 변경 ###
```
vagrant@master-node:~$ kubectl edit pod replicaset-nginx-kbbwr
pod/replicaset-nginx-kbbwr edited
```
```
 # labels:                      <= 파드의 라벨을 주석처리함
    # app: my-nginx-pods-label    
  name: replicaset-nginx-kbbwr
  namespace: default
```

### app: my-nginx-pods-label 라벨을 가진 새로운 파드가 생성 ###
```
vagrant@master-node:~$ kubectl get pod --show-labels
NAME                     READY   STATUS    RESTARTS   AGE     LABELS
replicaset-nginx-fl7cs   1/1     Running   0          106s    app=my-nginx-pods-label
replicaset-nginx-kbbwr   1/1     Running   0          7m18s   <none>
replicaset-nginx-vpf8f   1/1     Running   0          7m18s   app=my-nginx-pods-label
replicaset-nginx-wmk88   1/1     Running   0          4m56s   app=my-nginx-pods-label
```

### 레플리카셋을 삭제 -> 라벨이 일치하지 않는 파드는 삭제되지 않음 ###
```
vagrant@master-node:~$ kubectl delete rs replicaset-nginx
replicaset.apps "replicaset-nginx" deleted

vagrant@master-node:~$ kubectl get pod --show-labels
NAME                     READY   STATUS    RESTARTS   AGE     LABELS
replicaset-nginx-kbbwr   1/1     Running   0          9m48s   <none>
```

## 디플로이먼트(Deployment) ##

- 레플리카셋, 파드의 배포를 관리
- 애플리케이션의 **배포와 업데이트**를 편하게 하기 위해서 사용
- 쿠버네티스에서 **상태가 없는(stateless) 애플리케이션을 배포**할 때 사용하는 가장 기본적인 컨트롤러
- 디플로이먼트는 **스케일, 롤아웃, 롤백, 자동복구** 기능이 있음
  - 스케일 : 파드의 개수를 늘리거나 줄일 수 있음
  - 롤아웃, 롤백 : 서비스를 유지하면서 파드를 교체
  - 자동복구 : 노드 수준에서 장애가 발생했을 때 파드를 복구하는 것이 가능

## 디플로이먼트 생성 및 삭제 ##
https://kubernetes.io/ko/docs/concepts/workloads/controllers/deployment/

### 디플로이먼트 생성 ###
- /home/vagrant/deployment-nginx.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-nginx
  template:
    metadata:
      name: my-nginx-pod
      labels:
        app: my-nginx
    spec:
      containers:
      - name: nginx
        image: docker.io/nginx
        ports:
        - containerPort: 80
      imagePullSecrets:
      - name: regcred          

```

### 디플로이먼트, 레플리카셋, 파드 생성을 확인 ###
```
vagrant@master-node:~$ kubectl apply -f deployment-nginx.yaml
deployment.apps/my-nginx-deployment created

vagrant@master-node:~$ kubectl get deployments,replicasets,pods
NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-nginx-deployment   3/3     3            3           26s

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/my-nginx-deployment-66bcdb4565   3         3         3       25s

NAME                                       READY   STATUS    RESTARTS   AGE
pod/my-nginx-deployment-66bcdb4565-hjbtw   1/1     Running   0          25s
pod/my-nginx-deployment-66bcdb4565-kmgs2   1/1     Running   0          25s
pod/my-nginx-deployment-66bcdb4565-nv9bg   1/1     Running   0          25s
```

### 디플로이먼트를 삭제 -> 레플리카셋, 파드 또한 함께 삭제됨 ###
```
vagrant@master-node:~$ kubectl delete deployment my-nginx-deployment
deployment.apps "my-nginx-deployment" deleted

vagrant@master-node:~$ kubectl get deployments,replicasets,pods
No resources found in default namespace.
```

## 디플로이먼트 사용하는 이유 1) 스케일 ##
- 레플리카의 값을 변경해서 파드의 개수를 조절 -> 처리 능력을 높이고 낮추는 기능
- 파드의 개수를 늘리는 중에 쿠버네티스 클러스터의 자원(CPU, 메모리, ...)이 부족해지면 ... 노드를 추가하여 자원이 생길때까지 파드 생성을 보류

### 레플리카 값을 3으로 설정해서 디플로이먼트 생성 ###
- /home/vagrant/web-deployment-replicas-3.yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      name: my-nginx-pod
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: docker.io/nginx
        ports:
        - containerPort: 80
      imagePullSecrets:
      - name: regcred          
```

### 생성한 yaml 배포 후 확인 ###
```
vagrant@master-node:~$ kubectl apply -f web-deployment-replicas-3.yaml
deployment.apps/web-deploy created

vagrant@master-node:~$ kubectl get deploy,rs,po -o wide
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES            SELECTOR
deployment.apps/web-deploy   3/3     3            3           37s   nginx        docker.io/nginx   app=web

NAME                                    DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES            SELECTOR
replicaset.apps/web-deploy-59fbcf55f8   3         3         3       37s   nginx        docker.io/nginx   app=web,pod-template-hash=59fbcf55f8

NAME                              READY   STATUS    RESTARTS   AGE   IP              NODE            NOMINATED NODE   READINESS GATES
pod/web-deploy-59fbcf55f8-m8xjb   1/1     Running   0          37s   172.16.158.15   worker-node02   <none>           <none>
pod/web-deploy-59fbcf55f8-rpzsl   1/1     Running   0          37s   172.16.158.16   worker-node02   <none>           <none>
pod/web-deploy-59fbcf55f8-zmnrl   1/1     Running   0          37s   172.16.87.203   worker-node01   <none>           <none>
```

### 레플리카 값을 5로 변경해서 적용 ###
```
vagrant@master-node:~$ cp web-deployment-replicas-3.yaml web-deployment-replicas-5.yaml
```
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-deploy
spec:
  replicas: 5             => 3에서 5로 변경
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      name: my-nginx-pod
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: docker.io/nginx
        ports:
        - containerPort: 80
      imagePullSecrets:
      - name: regcred          
```
- 수정한 yaml 파일 배포
```
vagrant@master-node:~$ kubectl apply -f web-deployment-replicas-5.yaml
deployment.apps/web-deploy configured

vagrant@master-node:~$ kubectl get deploy,rs,po -o wide
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES            SELECTOR
deployment.apps/web-deploy   5/5     5            5           3m    nginx        docker.io/nginx   app=web

NAME                                    DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES            SELECTOR
replicaset.apps/web-deploy-59fbcf55f8   5         5         5       3m    nginx        docker.io/nginx   app=web,pod-template-hash=59fbcf55f8

NAME                              READY   STATUS    RESTARTS   AGE   IP              NODE            NOMINATED NODE   READINESS GATES
pod/web-deploy-59fbcf55f8-m8xjb   1/1     Running   0          3m    172.16.158.15   worker-node02   <none>           <none>
pod/web-deploy-59fbcf55f8-plj2g   1/1     Running   0          22s   172.16.87.204   worker-node01   <none>           <none>
pod/web-deploy-59fbcf55f8-rpzsl   1/1     Running   0          3m    172.16.158.16   worker-node02   <none>           <none>
pod/web-deploy-59fbcf55f8-rz9jw   1/1     Running   0          22s   172.16.158.17   worker-node02   <none>           <none>
pod/web-deploy-59fbcf55f8-zmnrl   1/1     Running   0          3m    172.16.87.203   worker-node01   <none>           <none>
```

### kubectl scale 명령으로  레플리카 값 변경 ###
```
vagrant@master-node:~$ kubectl get deploy,rs,po -o wide
NAME                         READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES            SELECTOR
deployment.apps/web-deploy   5/10    10           5           4m38s   nginx        docker.io/nginx   app=web

NAME                                    DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES            SELECTOR
replicaset.apps/web-deploy-59fbcf55f8   10        10        5       4m38s   nginx        docker.io/nginx   app=web,pod-template-hash=59fbcf55f8

NAME                              READY   STATUS              RESTARTS   AGE     IP              NODE            NOMINATED NODE   READINESS GATES
pod/web-deploy-59fbcf55f8-6g9cs   0/1     ContainerCreating   0          2s      <none>          worker-node01   <none>           <none>
pod/web-deploy-59fbcf55f8-84kg7   0/1     ContainerCreating   0          2s      <none>          worker-node01   <none>           <none>
pod/web-deploy-59fbcf55f8-969qn   0/1     ContainerCreating   0          2s      <none>          worker-node02   <none>           <none>
pod/web-deploy-59fbcf55f8-m8xjb   1/1     Running             0          4m38s   172.16.158.15   worker-node02   <none>           <none>
pod/web-deploy-59fbcf55f8-mgvhs   0/1     ContainerCreating   0          2s      <none>          worker-node02   <none>           <none>
pod/web-deploy-59fbcf55f8-plj2g   1/1     Running             0          2m      172.16.87.204   worker-node01   <none>           <none>
pod/web-deploy-59fbcf55f8-pxddn   0/1     ContainerCreating   0          2s      <none>          worker-node01   <none>           <none>
pod/web-deploy-59fbcf55f8-rpzsl   1/1     Running             0          4m38s   172.16.158.16   worker-node02   <none>           <none>
pod/web-deploy-59fbcf55f8-rz9jw   1/1     Running             0          2m      172.16.158.17   worker-node02   <none>           <none>
pod/web-deploy-59fbcf55f8-zmnrl   1/1     Running             0          4m38s   172.16.87.203   worker-node01   <none>           <none>
```

## 디플로이먼트를 사용하는 이유 2) 롤아웃, 롤백 ##
- 애플리케이션을 업데이트할 때 레플리카셋의 변경 사항을 저장하는 **리비전(revision)을 남겨 롤백을 가능**하게 해주고, 무중단 서비스를 위해 **파드의 롤링 업데이터 전력을 지정**할 수 있음

### --record 옵션을 추가해 디플로이먼트 생성 ###
```
vagrant@master-node:~$ kubectl apply -f deployment-nginx.yaml --record
Flag --record has been deprecated, --record will be removed in the future
deployment.apps/my-nginx-deployment created

vagrant@master-node:~$ kubectl get deploy,rs,po -o wide
NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES            SELECTOR
deployment.apps/my-nginx-deployment   3/3     3            3           22s   nginx        docker.io/nginx   app=my-nginx

NAME                                             DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES            SELECTOR
replicaset.apps/my-nginx-deployment-66bcdb4565   3         3         3       22s   nginx        docker.io/nginx   app=my-nginx,pod-template-hash=66bcdb4565

NAME                                       READY   STATUS    RESTARTS   AGE   IP              NODE            NOMINATED NODE   READINESS GATES
pod/my-nginx-deployment-66bcdb4565-c6dh9   1/1     Running   0          22s   172.16.158.21   worker-node02   <none>           <none>
pod/my-nginx-deployment-66bcdb4565-lvzxt   1/1     Running   0          22s   172.16.158.20   worker-node02   <none>           <none>
pod/my-nginx-deployment-66bcdb4565-zrdrj   1/1     Running   0          22s   172.16.87.208   worker-node01   <none>           <none>
vagrant@master-node:~$
```
### kubectl set image 명령으로 파드의 이미지를 변경 ###
```
vagrant@master-node:~$ kubectl set image deployments my-nginx-deployment nginx=docker.io/nginx:1.11 --record
Flag --record has been deprecated, --record will be removed in the future
deployment.apps/my-nginx-deployment image updated
```
- 앞에서 생성한 파드는 종료, 삭제되었고 새롭게 추가된 파드로 변경됨
- 새로 생성된 replicaset ID(595b6754f6)를 통해 확인 가능
```
vagrant@master-node:~$ kubectl get deploy,rs,po -o wide
NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE     CONTAINERS   IMAGES                 SELECTOR
deployment.apps/my-nginx-deployment   3/3     3            3           5m13s   nginx        docker.io/nginx:1.11   app=my-nginx

NAME                                             DESIRED   CURRENT   READY   AGE     CONTAINERS   IMAGES                 SELECTOR
replicaset.apps/my-nginx-deployment-595b6754f6   3         3         3       2m46s   nginx        docker.io/nginx:1.11   app=my-nginx,pod-template-hash=595b6754f6
replicaset.apps/my-nginx-deployment-66bcdb4565   0         0         0       5m13s   nginx        docker.io/nginx        app=my-nginx,pod-template-hash=66bcdb4565

NAME                                       READY   STATUS    RESTARTS   AGE     IP              NODE            NOMINATED NODE   READINESS GATES
pod/my-nginx-deployment-595b6754f6-6crcc   1/1     Running   0          2m28s   172.16.87.209   worker-node01   <none>           <none>
pod/my-nginx-deployment-595b6754f6-89bk7   1/1     Running   0          2m11s   172.16.158.23   worker-node02   <none>           <none>
pod/my-nginx-deployment-595b6754f6-dr47v   1/1     Running   0          2m46s   172.16.158.22   worker-node02   <none>           <none>
```
### 리비전 정보 확인 ###
- --record=true 옵션으로 디플로이먼트를 변경하면 변경 사항을 기록하여 해당 버전의 레플리카셋을 보존할 수 있음
```
vagrant@master-node:~$ kubectl rollout history deployment my-nginx-deployment
deployment.apps/my-nginx-deployment
REVISION  CHANGE-CAUSE
1         kubectl apply --filename=deployment-nginx.yaml --record=true
2         kubectl set image deployments my-nginx-deployment nginx=docker.io/nginx:1.11 --record=true
```

### 이전 버전의 레플리카셋으로 롤백 ###
```
vagrant@master-node:~$ kubectl rollout undo deployment my-nginx-deployment --to-revision=1
deployment.apps/my-nginx-deployment rolled back

vagrant@master-node:~$ kubectl get rs,pod
NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/my-nginx-deployment-595b6754f6   0         0         0       3d17h
replicaset.apps/my-nginx-deployment-66bcdb4565   3         3         3       3d17h

NAME                                       READY   STATUS    RESTARTS   AGE
pod/my-nginx-deployment-66bcdb4565-g42pg   1/1     Running   0          23s
pod/my-nginx-deployment-66bcdb4565-j6667   1/1     Running   0          26s
pod/my-nginx-deployment-66bcdb4565-lq95j   1/1     Running   0          18s
```
```
vagrant@master-node:~$ kubectl rollout history deployment my-nginx-deployment
deployment.apps/my-nginx-deployment
REVISION  CHANGE-CAUSE
2         kubectl set image deployments my-nginx-deployment nginx=docker.io/nginx:1.11 --record=true
3         kubectl apply --filename=deployment-nginx.yaml --record=true
```

### 디플로이먼트 상세 정보 출력 ###
```
vagrant@master-node:~$ kubectl describe deployment my-nginx-deployment
Name:                   my-nginx-deployment
Namespace:              default
CreationTimestamp:      Fri, 06 Oct 2023 06:43:53 +0000
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 3        <= 버전 정보 확인 가능
                        kubernetes.io/change-cause: kubectl apply --filename=deployment-nginx.yaml --record=true
Selector:               app=my-nginx
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  app=my-nginx
  Containers:
   nginx:
    Image:        docker.io/nginx
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  my-nginx-deployment-595b6754f6 (0/0 replicas created)
NewReplicaSet:   my-nginx-deployment-66bcdb4565 (3/3 replicas created)
Events:
```

### 스케일 변경 및 롤백 ###
```
vagrant@master-node:~$ kubectl scale --replicas=10 deployment my-nginx-deployment --record=true
Flag --record has been deprecated, --record will be removed in the future
deployment.apps/my-nginx-deployment scaled

vagrant@master-node:~$ kubectl get all
NAME                                       READY   STATUS    RESTARTS   AGE
pod/my-nginx-deployment-66bcdb4565-2vrr2   1/1     Running   0          17s
pod/my-nginx-deployment-66bcdb4565-4gcvk   1/1     Running   0          17s
pod/my-nginx-deployment-66bcdb4565-bn6b2   1/1     Running   0          17s
pod/my-nginx-deployment-66bcdb4565-fbfsv   1/1     Running   0          17s
pod/my-nginx-deployment-66bcdb4565-g42pg   1/1     Running   0          7m53s
pod/my-nginx-deployment-66bcdb4565-j6667   1/1     Running   0          7m56s
pod/my-nginx-deployment-66bcdb4565-lq95j   1/1     Running   0          7m48s
pod/my-nginx-deployment-66bcdb4565-ngdbr   1/1     Running   0          17s
pod/my-nginx-deployment-66bcdb4565-pd4xc   1/1     Running   0          17s
pod/my-nginx-deployment-66bcdb4565-xz7j8   1/1     Running   0          17s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   172.17.0.1   <none>        443/TCP   4d14h

NAME                                  READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/my-nginx-deployment   10/10   10           10          3d17h

NAME                                             DESIRED   CURRENT   READY   AGE
replicaset.apps/my-nginx-deployment-595b6754f6   0         0         0       3d17h
replicaset.apps/my-nginx-deployment-66bcdb4565   10        10        10      3d17h  <= 스케일 변갱해도 레플리카셋은 그대로 유지
```
### 모든 리소스 삭제 ###
```
vagrant@master-node:~$ kubectl delete deployment,rs,pod --all
deployment.apps "my-nginx-deployment" deleted
replicaset.apps "my-nginx-deployment-595b6754f6" deleted
replicaset.apps "my-nginx-deployment-66bcdb4565" deleted
pod "my-nginx-deployment-66bcdb4565-2vrr2" deleted
pod "my-nginx-deployment-66bcdb4565-4gcvk" deleted
pod "my-nginx-deployment-66bcdb4565-bn6b2" deleted
pod "my-nginx-deployment-66bcdb4565-fbfsv" deleted
pod "my-nginx-deployment-66bcdb4565-g42pg" deleted
pod "my-nginx-deployment-66bcdb4565-j6667" deleted
pod "my-nginx-deployment-66bcdb4565-lq95j" deleted
pod "my-nginx-deployment-66bcdb4565-ngdbr" deleted
pod "my-nginx-deployment-66bcdb4565-pd4xc" deleted
pod "my-nginx-deployment-66bcdb4565-xz7j8" deleted
```

## 디플로이먼트를 사용하는 이유 3) 자동 복구 ##
- 파드 내의 컨테이너가 종료되는 경우, 파드는 컨테이너 수준의 장애에 대해 자동 복구를 시도하고, 디플로이먼트는 파드 단위로 복구를 시도

### 30초 단위로 재기동하는 파드 정의 ###
- /home/vagrant/restart-pod.yaml 작성
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test1
spec:
  containers:
  - name: busybox
    image: docker.io/busybox:1
    command: ["sh", "-c", "sleep 30; exit 0"]
  restartPolicy: Always
  imagePullSecrets:
  - name: regcred
```
### 동일 사양의 파드를 4개 가동하는 드플로이먼트 정의 ###
- /home/vagrant/restart-deployment.yaml 작성
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: test2
spec:
  replicas: 4
  selector:
    matchLabels:
      app: test2
  template:
    metadata:
      labels:
        app: test2
    spec:
      containers:
      - name: busybox
        image: docker.io/busybox:1
        command: ["sh", "-c", "sleep 30; exit 0"]
      imagePullSecrets:
      - name: regcred
```

### 파드와 디플로이먼트를 각각 배포 ###
```
vagrant@master-node:~$ kubectl apply -f restart-pod.yaml
pod/test1 created

vagrant@master-node:~$ kubectl apply -f restart-deployment.yaml
deployment.apps/test2 created
```

### 노드 및 파드 동작 확인 ###
```
vagrant@master-node:~$ kubectl get node,pod -o wide
NAME                 STATUS   ROLES           AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
node/master-node     Ready    control-plane   4d15h   v1.27.1   10.0.0.10     <none>        Ubuntu 22.04.3 LTS   5.15.0-83-generic   cri-o://1.27.1
node/worker-node01   Ready    worker          4d15h   v1.27.1   10.0.0.11     <none>        Ubuntu 22.04.3 LTS   5.15.0-83-generic   cri-o://1.27.1
node/worker-node02   Ready    worker          4d15h   v1.27.1   10.0.0.12     <none>        Ubuntu 22.04.3 LTS   5.15.0-83-generic   cri-o://1.27.1

NAME                        READY   STATUS      RESTARTS      AGE   IP              NODE            NOMINATED NODE   READINESS GATES
pod/test1                   0/1     Completed   1 (39s ago)   76s   172.16.158.31   worker-node02   <none>           <none>
pod/test2-958d4f5c4-bjnn4   0/1     Completed   0             32s   172.16.158.33   worker-node02   <none>           <none>
pod/test2-958d4f5c4-frhjg   0/1     Completed   0             32s   172.16.158.32   worker-node02   <none>           <none>
pod/test2-958d4f5c4-rdsz2   1/1     Running     0             32s   172.16.87.217   worker-node01   <none>           <none>
pod/test2-958d4f5c4-xgh48   1/1     Running     0             32s   172.16.87.216   worker-node01   <none>           <none>
```
```
vagrant@master-node:~$ kubectl get node,pod -o wide
NAME                 STATUS   ROLES           AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
node/master-node     Ready    control-plane   4d15h   v1.27.1   10.0.0.10     <none>        Ubuntu 22.04.3 LTS   5.15.0-83-generic   cri-o://1.27.1
node/worker-node01   Ready    worker          4d15h   v1.27.1   10.0.0.11     <none>        Ubuntu 22.04.3 LTS   5.15.0-83-generic   cri-o://1.27.1
node/worker-node02   Ready    worker          4d15h   v1.27.1   10.0.0.12     <none>        Ubuntu 22.04.3 LTS   5.15.0-83-generic   cri-o://1.27.1

NAME                        READY   STATUS      RESTARTS      AGE    IP              NODE            NOMINATED NODE   READINESS GATES
pod/test1                   0/1     Completed   2 (49s ago)   116s   172.16.158.31   worker-node02   <none>           <none>
pod/test2-958d4f5c4-bjnn4   0/1     Completed   1 (41s ago)   72s    172.16.158.33   worker-node02   <none>           <none>
pod/test2-958d4f5c4-frhjg   0/1     Completed   1 (41s ago)   72s    172.16.158.32   worker-node02   <none>           <none>
pod/test2-958d4f5c4-rdsz2   0/1     Completed   1 (34s ago)   72s    172.16.87.217   worker-node01   <none>           <none>
pod/test2-958d4f5c4-xgh48   0/1     Completed   1 (36s ago)   72s    172.16.87.216   worker-node01   <none>           <none>
```
```
vagrant@master-node:~$ kubectl get node,pod -o wide
NAME                 STATUS   ROLES           AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
node/master-node     Ready    control-plane   4d15h   v1.27.1   10.0.0.10     <none>        Ubuntu 22.04.3 LTS   5.15.0-83-generic   cri-o://1.27.1
node/worker-node01   Ready    worker          4d15h   v1.27.1   10.0.0.11     <none>        Ubuntu 22.04.3 LTS   5.15.0-83-generic   cri-o://1.27.1
node/worker-node02   Ready    worker          4d15h   v1.27.1   10.0.0.12     <none>        Ubuntu 22.04.3 LTS   5.15.0-83-generic   cri-o://1.27.1

NAME                        READY   STATUS    RESTARTS      AGE     IP              NODE            NOMINATED NODE   READINESS GATES
pod/test1                   1/1     Running   3 (31s ago)   2m22s   172.16.158.31   worker-node02   <none>           <none>
pod/test2-958d4f5c4-bjnn4   1/1     Running   2 (37s ago)   98s     172.16.158.33   worker-node02   <none>           <none>
pod/test2-958d4f5c4-frhjg   1/1     Running   2 (37s ago)   98s     172.16.158.32   worker-node02   <none>           <none>
pod/test2-958d4f5c4-rdsz2   1/1     Running   2 (30s ago)   98s     172.16.87.217   worker-node01   <none>           <none>
pod/test2-958d4f5c4-xgh48   1/1     Running   2 (32s ago)   98s     172.16.87.216   worker-node01   <none>           <none>
```

### 단독으로 기동한 test1 파드가 배포된 노드를 정지 (test1 Running 상태에서) ###
```
C:\Users\User>cd c:\kubernetes\vagrant-kubeadm-kubernetes

c:\kubernetes\vagrant-kubeadm-kubernetes>vagrant halt node02
```
### 노드 및 파드 동작을 확인 ###
```
vagrant@master-node:~$ kubectl get node
NAME            STATUS     ROLES           AGE     VERSION
master-node     Ready      control-plane   4d15h   v1.27.1
worker-node01   Ready      worker          4d15h   v1.27.1
worker-node02   NotReady   worker          4d15h   v1.27.1
```
```
vagrant@master-node:~$ kubectl get pod -o wide
NAME                    READY   STATUS             RESTARTS        AGE     IP              NODE            NOMINATED NODE   READINESS GATES
test1                   0/1     Completed          5 (3m32s ago)   7m47s   172.16.158.50   worker-node02   <none>           <none>
test2-958d4f5c4-5p5v2   1/1     Running            5 (3m ago)      7m5s    172.16.158.51   worker-node02   <none>           <none>
test2-958d4f5c4-67fr6   0/1     CrashLoopBackOff   5 (63s ago)     7m5s    172.16.87.220   worker-node01   <none>           <none>
test2-958d4f5c4-hwzxv   0/1     CrashLoopBackOff   5 (64s ago)     7m5s    172.16.87.219   worker-node01   <none>           <none>
test2-958d4f5c4-pxj8j   1/1     Running            5 (3m7s ago)    7m5s    172.16.158.52   worker-node02   <none>  
```
- 노드 종료 후 6분 정도 경과하면 디플로이먼트로 배포한 test2는 활성화 노드에 대체 파드를 생성하는 반면, 단독으로 기동한 test1은 원래 노드에 위치하는 것을 확인할 수 있음
- 변경사항 확인 => kubectl get pod -o wide --watch
```
vagrant@master-node:~$ kubectl get pod -o wide
NAME                    READY   STATUS             RESTARTS      AGE     IP              NODE            NOMINATED NODE   READINESS GATES
test1                   1/1     Terminating        5 (12m ago)   17m     172.16.158.31   worker-node02   <none>           <none>
test2-958d4f5c4-bjnn4   0/1     Terminating        4 (12m ago)   16m     172.16.158.33   worker-node02   <none>           <none>
test2-958d4f5c4-cjhd4   0/1     CrashLoopBackOff   4 (88s ago)   5m29s   172.16.87.219   worker-node01   <none>           <none>
test2-958d4f5c4-frhjg   0/1     Terminating        4 (12m ago)   16m     172.16.158.32   worker-node02   <none>           <none>
test2-958d4f5c4-rdsz2   0/1     CrashLoopBackOff   7 (80s ago)   16m     172.16.87.217   worker-node01   <none>           <none>
test2-958d4f5c4-xgh48   0/1     CrashLoopBackOff   7 (88s ago)   16m     172.16.87.216   worker-node01   <none>           <none>
test2-958d4f5c4-z58bb   0/1     CrashLoopBackOff   4 (87s ago)   5m29s   172.16.87.218   worker-node01   <none>           <none>
```
```
vagrant@master-node:~$ kubectl get pod -o wide
NAME                    READY   STATUS             RESTARTS        AGE   IP              NODE            NOMINATED NODE   READINESS GATES
test1                   1/1     Terminating        5 (22m ago)     26m   172.16.158.31   worker-node02   <none>           <none>
test2-958d4f5c4-bjnn4   0/1     Terminating        4 (21m ago)     25m   172.16.158.33   worker-node02   <none>           <none>
test2-958d4f5c4-cjhd4   1/1     Running            7 (5m28s ago)   14m   172.16.87.219   worker-node01   <none>           <none>
test2-958d4f5c4-frhjg   0/1     Terminating        4 (21m ago)     25m   172.16.158.32   worker-node02   <none>           <none>
test2-958d4f5c4-rdsz2   0/1     CrashLoopBackOff   8 (5m8s ago)    25m   172.16.87.217   worker-node01   <none>           <none>
test2-958d4f5c4-xgh48   1/1     Running            9 (5m13s ago)   25m   172.16.87.216   worker-node01   <none>           <none>
test2-958d4f5c4-z58bb   0/1     Completed          7 (5m35s ago)   14m   172.16.87.218   worker-node01   <none>           <none>
```
### 노드 재기동 후 노드 및 파드 동작 확인 ###
- c:\kubernetes\vagrant-kubeadm-kubernetes>vagrant up node02
- 중지되었던 노드가 복원되면, Terminating 상태였던 파드가 모두 제거되는 것을 확인
- 노드와의 통신이 회복되어 상태가 불분명했던 파드의 상태가 확인되어 삭제되는 것으로 단독으로 기동한 파드는 완전히 소멸됨
- 일시적인 장애로 자동 복구가 발동되었을 때 더 불안한 상태로 빠지는 것을 막기 위해서, 디플로이먼트는 천천히 복구를 수행하도록 만들어짐
```
vagrant@master-node:~$ kubectl get node
NAME            STATUS   ROLES           AGE     VERSION
master-node     Ready    control-plane   4d15h   v1.27.1
worker-node01   Ready    worker          4d15h   v1.27.1
worker-node02   Ready    worker          4d15h   v1.27.1

vagrant@master-node:~$ kubectl get pod -o wide
NAME                    READY   STATUS             RESTARTS        AGE   IP              NODE            NOMINATED NODE   READINESS GATES
test2-958d4f5c4-cjhd4   0/1     CrashLoopBackOff   8 (2m11s ago)   22m   172.16.87.219   worker-node01   <none>           <none>
test2-958d4f5c4-rdsz2   0/1     CrashLoopBackOff   10 (98s ago)    33m   172.16.87.217   worker-node01   <none>           <none>
test2-958d4f5c4-xgh48   0/1     CrashLoopBackOff   10 (113s ago)   33m   172.16.87.216   worker-node01   <none>           <none>
test2-958d4f5c4-z58bb   0/1     CrashLoopBackOff   8 (2m13s ago)   22m   172.16.87.218   worker-node01   <none>           <none>
```

## 디플로이먼트 배포 전략 ##
https://dev.classmethod.jp/articles/ci-cd-deployment-strategies-kr/

### Recreate (재생성) ###
- /home/vagrant/sample-deployment-recreate.yaml 작성

![image](https://github.com/xodbs1123/Kubernetes/assets/61976898/8c3720d1-6c15-4dd5-9558-260990864670)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: 
  name: sample-deployment-recreate
spec:
  strategy:
    type: Recreate              <= 기존 레플리카셋의 레플라 수를 0으로 하고 리소스를 반환
  replicas: 3                      신규 레플리카셋을 생성하고 레플리카 수를 늘림
  selector:
    matchLabels:
      app: sample-app
  template: 
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
        - name: nginx-container
          image: docker.io/nginx:1.16
      imagePullSecrets:
       - name: regcred
```
```
vagrant@master-node:~$ kubectl apply -f sample-deployment-recreate.yaml
deployment.apps/sample-deployment-recreate created
```
```
vagrant@master-node:~$ kubectl get rs,pod
NAME                                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/sample-deployment-recreate-9ff76c956   3         3         3       87s

NAME                                             READY   STATUS    RESTARTS   AGE
pod/sample-deployment-recreate-9ff76c956-2hkdp   1/1     Running   0          87s
pod/sample-deployment-recreate-9ff76c956-9b5t9   1/1     Running   0          87s
pod/sample-deployment-recreate-9ff76c956-flx96   1/1     Running   0          87s
```
- 리소스 변경 전 상태 확인
```
vagrant@master-node:~$ kubectl get rs --watch
NAME                                   DESIRED   CURRENT   READY   AGE
sample-deployment-recreate-9ff76c956   3         3         3       2m18s
```
- 컨테이너 이미지 업데이트 후 리소스 상태 확인
```
vagrant@master-node:~$ kubectl set image deployment sample-deployment-recreate nginx-container=nginx:1.17
deployment.apps/sample-deployment-recreate image updated
```
```
vagrant@master-node:~$ kubectl get rs --watch
NAME                                   DESIRED   CURRENT   READY   AGE
sample-deployment-recreate-9ff76c956   3         3         3       2m18s
sample-deployment-recreate-9ff76c956   0         3         3       4m11s
sample-deployment-recreate-9ff76c956   0         3         3       4m11s
sample-deployment-recreate-9ff76c956   0         0         0       4m12s    <= 첫 번째 RS를 종료
sample-deployment-recreate-77dc8d9fb   3         0         0       0s       <= 새로운 RS를 생성
sample-deployment-recreate-77dc8d9fb   3         0         0       0s
sample-deployment-recreate-77dc8d9fb   3         3         0       0s
sample-deployment-recreate-77dc8d9fb   3         3         1       16s
sample-deployment-recreate-77dc8d9fb   3         3         2       25s
sample-deployment-recreate-77dc8d9fb   3         3         3       27s      <= 새로운 RS를 서비스
```
### RollingUpdate ###
- 배포전략을 따로 지정하지 않으면 디폴트로 적용
- /home/vagrant/sample-deployment-rollingupdate.yaml 작성
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-dployment-rollingupdate
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavaliable: 0      <= 업데이트 중 동시에 정지 가능한 최대 파드 수
      maxSurge: 1            <= 업데이트 중 동시에 생성할 수 있는 최대 파드 수
    replicas: 3                  maxUnavaliable과 maxSurge를 모두 0으로 설정할 수는 없음
    selector:
      matchlabels:
        app: sample-app
    template:
      metadata:
        labels:
          app: sample-app
      spec:
        containers:
          - name: nginx-container
            image: docker.io/nginx:1.16
        imagePullSecrets:
       - name: regcred    
```
```
vagrant@master-node:~$ kubectl apply -f sample-deployment-rollingupdate.yaml
deployment.apps/sample-deployment-rollingupdate created

vagrant@master-node:~$ kubectl get rs,pod
NAME                                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/sample-deployment-rollingupdate-9ff76c956   3         3         3       8s

NAME                                                  READY   STATUS    RESTARTS   AGE
pod/sample-deployment-rollingupdate-9ff76c956-4zmsc   1/1     Running   0          8s
pod/sample-deployment-rollingupdate-9ff76c956-rjvd4   1/1     Running   0          8s
pod/sample-deployment-rollingupdate-9ff76c956-zspdb   1/1     Running   0          8s
```
- watch 명령어로 상태 모니터링
```
vagrant@master-node:~$ kubectl get rs --watch
NAME                                        DESIRED   CURRENT   READY   AGE
sample-deployment-rollingupdate-9ff76c956   3         3         3       45s
```
- 이미지 업데이트
```
vagrant@master-node:~$ kubectl set image deployment sample-deployment-rollingupdate nginx-container=nginx:1.17
deployment.apps/sample-deployment-rollingupdate image updated
```
- watch 명령어로 리소스 변경 상태 모니터링
```
vagrant@master-node:~$ kubectl get rs --watch
NAME                                        DESIRED   CURRENT   READY   AGE
sample-deployment-rollingupdate-9ff76c956   3         3         3       45s
sample-deployment-rollingupdate-77dc8d9fb   1         0         0       0s
sample-deployment-rollingupdate-77dc8d9fb   1         0         0       0s
sample-deployment-rollingupdate-77dc8d9fb   1         1         0       0s
sample-deployment-rollingupdate-77dc8d9fb   1         1         1       0s
sample-deployment-rollingupdate-9ff76c956   2         3         3       87s
sample-deployment-rollingupdate-77dc8d9fb   2         1         1       0s
sample-deployment-rollingupdate-9ff76c956   2         3         3       87s
sample-deployment-rollingupdate-9ff76c956   2         2         2       87s
sample-deployment-rollingupdate-77dc8d9fb   2         1         1       1s
sample-deployment-rollingupdate-77dc8d9fb   2         2         1       1s
sample-deployment-rollingupdate-77dc8d9fb   2         2         2       1s
sample-deployment-rollingupdate-9ff76c956   1         2         2       88s
sample-deployment-rollingupdate-77dc8d9fb   3         2         2       1s
sample-deployment-rollingupdate-9ff76c956   1         2         2       88s
sample-deployment-rollingupdate-77dc8d9fb   3         2         2       1s
sample-deployment-rollingupdate-9ff76c956   1         1         1       88s
sample-deployment-rollingupdate-77dc8d9fb   3         3         2       1s
sample-deployment-rollingupdate-77dc8d9fb   3         3         3       2s
sample-deployment-rollingupdate-9ff76c956   0         1         1       89s
sample-deployment-rollingupdate-9ff76c956   0         1         1       89s
sample-deployment-rollingupdate-9ff76c956   0         0         0       89s
```
### Blue/Green ###
### Canary ###
