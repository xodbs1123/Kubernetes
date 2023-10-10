## 데몬셋(DaemonSet) ##
- 각 노드에 파드를 하나씩 배치하는 리소스
- 레플리카 수를 지정할 수 없고 하나의 노드에 두 개의 파드를 배치할 수 없음
- 단, 파드를 배치하고 싶지 않은 노드가 있는 경우 nodeSelector 또는 노드 안티어피니티를 사용해서 예외 처리할 수 있음
- 노드를 늘렸을 때 데몬셋의 파드도 자동으로 늘어난 노드에서 기능
- 호스트 단위로 로그를 수집하는 경우, 리소스 사용 현황 및 노드 상태를 모니터링하는 경우 사용

### 데몬셋 정의 ###
- /home/vagrant/sample-daemonset.yaml 작성
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: sample-daemonset
spec:
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

### node01 중지 ###
- c:\kubernetes\vagrant-kubeadm-kubernetes>vagrant halt node01

### 데몬셋 배포 ###
```
vagrant@master-node:~$ kubectl apply -f sample-daemonset.yaml
daemonset.apps/sample-daemonset created

vagrant@master-node:~$ kubectl get daemonset,pod -o wide
NAME                              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE    CONTAINERS        IMAGES                 SELECTOR
daemonset.apps/sample-daemonset   1         1         1       1            1           <none>          108s   nginx-container   docker.io/nginx:1.16   app=sample-app

NAME                         READY   STATUS    RESTARTS   AGE    IP              NODE            NOMINATED NODE   READINESS GATES
pod/sample-daemonset-sw2k5   1/1     Running   0          108s   172.16.158.46   worker-node02   <none>           <none>
```

### node01 재시작 ###
- c:\kubernetes\vagrant-kubeadm-kubernetes>vagrant up node01

### 데몬셋 조회 ###
- node01에도 파드가 실행된 모습
```
vagrant@master-node:~$ kubectl get daemonset,pod -o wide
NAME                              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE     CONTAINERS        IMAGES                 SELECTOR
daemonset.apps/sample-daemonset   2         2         2       2            2           <none>          4m33s   nginx-container   docker.io/nginx:1.16   app=sample-app

NAME                         READY   STATUS    RESTARTS   AGE     IP              NODE            NOMINATED NODE   READINESS GATES
pod/sample-daemonset-nx9hr   1/1     Running   0          34s     172.16.87.226   worker-node01   <none>           <none>
pod/sample-daemonset-sw2k5   1/1     Running   0          4m33s   172.16.158.46   worker-node02   <none>           <none>
```

## 데몬셋 업데이트 전략 ##
### OnDelete ###
- 데몬셋 매니페스트가 변경되어도 기존 파드를 업데이트하지 않고, 파드가 다시 생성될 때 새로 정의한 파드를 생성
- /home/vagrant/sample-daemonset-ondelete.yaml 작성
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: sample-daemonset
spec:
  updateStrategy:
    type: OnDelete
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
- yaml 파일 배포
```
vagrant@master-node:~$ kubectl apply -f sample-daemonset-ondelete.yaml
daemonset.apps/sample-daemonset created
```
- 데몬셋 확인
```
vagrant@master-node:~$ kubectl get daemonset,pod -o wide
NAME                              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE   CONTAINERS        IMAGES                 SELECTOR
daemonset.apps/sample-daemonset   2         2         2       2            2           <none>          29s   nginx-container   docker.io/nginx:1.16   app=sample-app

NAME                         READY   STATUS    RESTARTS   AGE   IP              NODE            NOMINATED NODE   READINESS GATES
pod/sample-daemonset-9cfgv   1/1     Running   0          29s   172.16.87.227   worker-node01   <none>           <none>
pod/sample-daemonset-htmjp   1/1     Running   0          29s   172.16.158.47   worker-node02   <none>           <none>
```
- 이미지 변경( 파드가 변경되지 않은 것을 확인할 수 있음)
```
vagrant@master-node:~$ kubectl set image daemonset sample-daemonset nginx-container=nginx:1.17
daemonset.apps/sample-daemonset image updated

vagrant@master-node:~$ kubectl get daemonset,pod -o wide
NAME                              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE     CONTAINERS        IMAGES       SELECTOR
daemonset.apps/sample-daemonset   2         2         2       0            2           <none>          2m49s   nginx-container   nginx:1.17   app=sample-app

NAME                         READY   STATUS    RESTARTS   AGE     IP              NODE            NOMINATED NODE   READINESS GATES
pod/sample-daemonset-9cfgv   1/1     Running   0          2m49s   172.16.87.227   worker-node01   <none>           <none>
pod/sample-daemonset-htmjp   1/1     Running   0          2m49s   172.16.158.47   worker-node02   <none>           <none>
```
- 파드 삭제 후 학인 => 새로운 파드가 삭제된 파드와 동일한 노드에 생성된 것을 확인
```
vagrant@master-node:~$ kubectl get daemonset,pod -o wide
NAME                              DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE     CONTAINERS        IMAGES       SELECTOR
daemonset.apps/sample-daemonset   2         2         2       1            2           <none>          5m15s   nginx-container   nginx:1.17   app=sample-app

NAME                         READY   STATUS    RESTARTS   AGE     IP              NODE            NOMINATED NODE   READINESS GATES
pod/sample-daemonset-d6xnj   1/1     Running   0          7s      172.16.87.228   worker-node01   <none>           <none>
pod/sample-daemonset-htmjp   1/1     Running   0          5m15s   172.16.158.47   worker-node02   <none>           <none>
```
- 삭제 후 재실행한 파드 상세조회
```
vagrant@master-node:~$ kubectl describe pod sample-daemonset-d6xnj | grep Image:
    Image:          nginx:1.17
```
- 기존 실행하고 있던 파드 상세조회
```
vagrant@master-node:~$ kubectl describe pod sample-daemonset-htmjp | grep Image:
    Image:          docker.io/nginx:1.16
```

### RollingUpdate ###
- 즉시 파드를 업데이트할 때 사용 (기본값)
- maxSurge(동시에 생성할 수 있는 최대 파드 수)를 설정할 수 없으며, maxUnavaliable(동시에 정지 가능한 최대 파드 수)만 설정할 수 있음
- maxUnavaliable의 기본값은 1이며, 0으로 지정할 수 없음
- /home/vagrant/sample-daemonset-rollingupdate.yaml 작성
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: sample-daemonset-rollingupdate
spec:
  updateStrategy:
    type: RollingUpdate		
    rollingUpdate:				
      maxUnavailable: 1
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
- 데몬셋 생성 및 조회
```
vagrant@master-node:~$ kubectl apply -f sample-daemonset-rollingupdate.yaml
daemonset.apps/sample-daemonset-rollingupdate created

vagrant@master-node:~$ kubectl get daemonset,pod -o wide
NAME                                            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE   CONTAINERS        IMAGES                 SELECTOR
daemonset.apps/sample-daemonset-rollingupdate   2         2         2       2            2           <none>          21s   nginx-container   docker.io/nginx:1.16   app=sample-app

NAME                                       READY   STATUS    RESTARTS   AGE   IP              NODE            NOMINATED NODE   READINESS GATES
pod/sample-daemonset-rollingupdate-4hbpf   1/1     Running   0          21s   172.16.158.48   worker-node02   <none>           <none>
pod/sample-daemonset-rollingupdate-z87b4   1/1     Running   0          21s   172.16.87.229   worker-node01   <none>           <none>
```
- 파드 상태를 모니터링
```
vagrant@master-node:~$ kubectl get pod -o wide --watch
NAME                                   READY   STATUS    RESTARTS   AGE   IP              NODE            NOMINATED NODE   READINESS GATES
sample-daemonset-rollingupdate-4hbpf   1/1     Running   0          41s   172.16.158.48   worker-node02   <none>           <none>
sample-daemonset-rollingupdate-z87b4   1/1     Running   0          41s   172.16.87.229   worker-node01   <none>           <none>
```
- 데몬셋 이미지 변경
```
vagrant@master-node:~$ kubectl set image daemonset sample-daemonset-rollingupdate nginx-container=nginx:1.17
daemonset.apps/sample-daemonset-rollingupdate image updated
```
- 파드 상태 모니터링 변화 확인
```
vagrant@master-node:~$ kubectl get pod -o wide --watch
NAME                                   READY   STATUS    RESTARTS   AGE   IP              NODE            NOMINATED NODE   READINESS GATES
sample-daemonset-rollingupdate-4hbpf   1/1     Running   0          41s   172.16.158.48   worker-node02   <none>           <none>
sample-daemonset-rollingupdate-z87b4   1/1     Running   0          41s   172.16.87.229   worker-node01   <none>           <none>
sample-daemonset-rollingupdate-4hbpf   1/1     Terminating   0          96s   172.16.158.48   worker-node02   <none>           <none>
sample-daemonset-rollingupdate-4hbpf   1/1     Terminating   0          96s   172.16.158.48   worker-node02   <none>           <none>
sample-daemonset-rollingupdate-4hbpf   0/1     Terminating   0          96s   <none>          worker-node02   <none>           <none>
sample-daemonset-rollingupdate-p6gpv   0/1     Pending       0          0s    <none>          <none>          <none>           <none>
sample-daemonset-rollingupdate-p6gpv   0/1     Pending       0          0s    <none>          worker-node02   <none>           <none>
sample-daemonset-rollingupdate-p6gpv   0/1     ContainerCreating   0          0s    <none>          worker-node02   <none>           <none>
sample-daemonset-rollingupdate-p6gpv   0/1     ContainerCreating   0          1s    <none>          worker-node02   <none>           <none>
sample-daemonset-rollingupdate-4hbpf   0/1     Terminating         0          97s   172.16.158.48   worker-node02   <none>           <none>
sample-daemonset-rollingupdate-4hbpf   0/1     Terminating         0          97s   172.16.158.48   worker-node02   <none>           <none>
sample-daemonset-rollingupdate-4hbpf   0/1     Terminating         0          97s   172.16.158.48   worker-node02   <none>           <none>
sample-daemonset-rollingupdate-p6gpv   1/1     Running             0          2s    172.16.158.49   worker-node02   <none>           <none>
sample-daemonset-rollingupdate-z87b4   1/1     Terminating         0          98s   172.16.87.229   worker-node01   <none>           <none>
sample-daemonset-rollingupdate-z87b4   1/1     Terminating         0          98s   172.16.87.229   worker-node01   <none>           <none>
sample-daemonset-rollingupdate-z87b4   0/1     Terminating         0          98s   <none>          worker-node01   <none>           <none>
sample-daemonset-rollingupdate-dp7sz   0/1     Pending             0          0s    <none>          <none>          <none>           <none>
sample-daemonset-rollingupdate-dp7sz   0/1     Pending             0          0s    <none>          worker-node01   <none>           <none>
sample-daemonset-rollingupdate-dp7sz   0/1     ContainerCreating   0          0s    <none>          worker-node01   <none>           <none>
sample-daemonset-rollingupdate-dp7sz   0/1     ContainerCreating   0          1s    <none>          worker-node01   <none>           <none>
sample-daemonset-rollingupdate-z87b4   0/1     Terminating         0          99s   172.16.87.229   worker-node01   <none>           <none>
sample-daemonset-rollingupdate-z87b4   0/1     Terminating         0          99s   172.16.87.229   worker-node01   <none>           <none>
sample-daemonset-rollingupdate-z87b4   0/1     Terminating         0          99s   172.16.87.229   worker-node01   <none>           <none>
sample-daemonset-rollingupdate-dp7sz   1/1     Running             0          2s    172.16.87.230   worker-node01   <none>           <none>
```

## 스테이트풀셋(StatefulSet) ##
- 데이터베이스 등과 같은 스테이트풀(stateful)한 워크로드에 사용하기 위한 리소스
