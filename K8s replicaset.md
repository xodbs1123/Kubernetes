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
