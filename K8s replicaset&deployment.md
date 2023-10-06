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
