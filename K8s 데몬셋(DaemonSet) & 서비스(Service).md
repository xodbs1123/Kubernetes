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

## 서비스(Service) ##
- 파드를 연결하고 외부에 노출
### 서비스 기능 ###
- 여러 개의 파드에 쉽게 접근할 수 있도록 고유한 도메인 이름을 부여
- 여러 개의 파드에 접근할 때, 요청을 분산하는 로드 밸런서 기능 수행
- 클라우드 플랫폼의 로드밸런서, 클러스터 노드의 포트 등을 통해 파드를 외부에 노출

### 서비스 타입 ###
- **ClusterIP**
  - 디폴트
  - 클러스터 내부에서 파드들에 접근할 때 사용
  - 외부로 파드를 노출하지 않기 때문에 클러스터 내부에서만 사용되는 파드에 적합
- **NodePort**
  - 파드에 접근할 수 있는 포트를 클러스터의 모든 노드에 동일하게 개방
  - **외부에서 파드에 접근할 수 있는 서비스 타입**
  - 접근할 수 있는 포트는 랜덤으로 정해지지만, 특정 포트로 접근하도록 설정 가능
- **LoadBalancer**
  - 클라우드 플랫폼에서 제공하는 로드밸런스를 동적으로 프로비저닝해 파드에 연결
  - NodePort 타입과 마찬가지로 **외부에서 파드에 접근할 수 있는 서비스 타입**
  - 일반적으로 AWS, GCP 등과 같은 클라우드 플랫폼 환경에서 사용
- **ExternalName**
  - 외부 서비스를 쿠버네티스 내부에서 호출하고자 할 때 사용
  - 클러스터 내의 파드에서 외부 IP 주소에 서비스의 이름으로 접근할 수 있음

### 서비스를 생성하는 방법 ###
- **매니페스트(YAML)을 이용해서 생성**
- **kubectl expose 명령을 사용해서 생성**

### 디플로이먼트로 파드 생성 및 요청 전달 ###
- **디플로이먼트 생성**
- /home/vagrant/hostname-deployment.yaml 작성
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hostname-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webserver
  template:
    metadata:
      labels:
        app: webserver
      name: my-webserver
    spec:
      containers:
      - name: my-webserver
        image: docker.io/alicek106/rr-test:echo-hostname
        ports:
        - containerPort: 80
      imagePullSecrets:                            
      - name: regcred
```
- **디플로이먼트 배포**
```
vagrant@master-node:~$ kubectl apply -f hostname-deployment.yaml
deployment.apps/hostname-deployment created

vagrant@master-node:~$ kubectl get pod -o wide
NAME                                   READY   STATUS    RESTARTS   AGE    IP              NODE            NOMINATED NODE   READINESS GATES
hostname-deployment-7d4f978855-qmsdd   1/1     Running   0          106s   172.16.158.50   worker-node02   <none>           <none>
hostname-deployment-7d4f978855-rn68f   1/1     Running   0          106s   172.16.87.231   worker-node01   <none>           <none>
hostname-deployment-7d4f978855-vg4p9   1/1     Running   0          106s   172.16.158.51   worker-node02   <none>           <none>
```
- **임시 파드를 실행해서 디플로이한 파드의 IP 주소로 요청을 전달**
```
vagrant@master-node:~$ kubectl run -it --rm debug --image=docker.io/busybox --restart=Never sh
If you don't see a command prompt, try pressing enter.
/ #
/ #
/ #
```
```
/ # wget -q -O - http://172.16.158.50 | grep Hello
        <p>Hello,  hostname-deployment-7d4f978855-qmsdd</p>     </blockquote>    <= 파드 이름이 제공되는 모습
```
### 파드 IP 주소를 기반으로 애플리케이션 파드에 접근했을 때 문제점 ###
- **파드가 수시로 삭제, 생성되는 과정에서 애플리케이션 파드의 IP 주소가 변경될 수 있음**
  - 클라이언트 파드가 변경된 애플리케이션 파드의 IP 주소를 알 수 없음
  - 따라서, 이름 기반으로 애플리케이션 파드에 접근할 수 있는 방안이 필요
- **클라이언트 파드는 자신이 알고 있는 애플리케이션 파드의 IP로만 접근**    
  - 부하가 특정 파드로 집중될 수 있음
  - 따라서, 로드밸런싱 기능이 필요

## ClusterIP 타입의 서비스 ##

![image](https://github.com/xodbs1123/Kubernetes/assets/61976898/a46cac20-0999-40e2-84e1-13329854bedd)

### 서비스 생성 ###
- /home/vagrant/hostname-service-clusterip.yaml 작성
```yaml
apiVersion: v1
kind: Service
metadata:
  name: hostname-service-clusterip
spec:
  type: ClusterIP       # 서비스 타입(기본값이 ClusterIP)
  selector:             # 어떤 라벨의 파드에 접근할 수 있게 만들 것인지 결정
    app: webserver     # 파드의 라벨
  ports:
    - name: web-port
      port: 8080        # 서비스의 IP에 접근할 때 사용할 포트
      targetPort: 80    # selector 항목에서 정의한 라벨에 의해 접근 대상이 된 파드 내부에서 사용하고 있는 포트

```
### 서비스 생성 및 확인 ###
```
vagrant@master-node:~$ kubectl apply -f hostname-service-clusterip.yaml
service/hostname-service-clusterip created

vagrant@master-node:~$ kubectl get service
NAME                         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
hostname-service-clusterip   ClusterIP   172.17.44.200   <none>        8080/TCP   18s
kubernetes                   ClusterIP   172.17.0.1      <none>        443/TCP    4d21h
```
- CLUSTER-IP => 쿠버네트스 클러스터에서만 사용할 수 있는 내부 IP로, 이 IP를 통해 서비스에 연결된 파드로 접근이 가능
### 임시 파드 생성 후 서비스의 CLUSTER-IP와 PORT로 요청 전송 ###
```
vagrant@master-node:~$  kubectl run -it --rm debug --image=docker.io/busybox --restart=Never sh
If you don't see a command prompt, try pressing enter.
/ #
```
```
/ # wget -q -O - http://172.17.44.200:8080 | grep Hello		⇐ 서비스의 CLUSTER-IP와 PORT로 접근
        <p>Hello,  hostname-deployment-7d4f978855-8h2m9</p>     	⇐ 서비스에 연결된 파드로 로드밸런싱되는 것을 확인
/ # wget -q -O - http://172.17.44.200:8080 | grep Hello
        <p>Hello,  hostname-deployment-7d4f978855-pzkgc</p>     </blockquote>
/ # wget -q -O - http://172.17.44.200:8080 | grep Hello
        <p>Hello,  hostname-deployment-7d4f978855-v7zlc</p>     </blockquote>
```
### 서비스 NAME과 PORT로 요청을 전달
- 쿠버네티스는 애플리케이션이 서비스나 파드를 쉽게 찾을 수 있도록 내부 DNS를 구동
```
/ # wget -q -O - http://hostname-service-clusterip:8080 | grep Hello
        <p>Hello,  hostname-deployment-7d4f978855-qmsdd</p>     </blockquote>
/ # wget -q -O - http://hostname-service-clusterip:8080 | grep Hello
        <p>Hello,  hostname-deployment-7d4f978855-rn68f</p>     </blockquote>
/ # wget -q -O - http://hostname-service-clusterip:8080 | grep Hello
        <p>Hello,  hostname-deployment-7d4f978855-vg4p9</p>     </blockquote>
```
### 파드 하나 삭제 ###
```
vagrant@master-node:~$ kubectl delete pod hostname-deployment-7d4f978855-qmsdd
pod "hostname-deployment-7d4f978855-qmsdd" deleted
```
```
vagrant@master-node:~$ kubectl get pod
NAME                                   READY   STATUS    RESTARTS   AGE
debug                                  1/1     Running   0          9m16s
hostname-deployment-7d4f978855-rn68f   1/1     Running   0          66m
hostname-deployment-7d4f978855-vg4p9   1/1     Running   0          66m
hostname-deployment-7d4f978855-zpnxd   1/1     Running   0          60s    <= 새로운 파드 추가
```
### 서비스로 접근했을 때 새로 생성된 파드로 요청이 전달되는 것을 확인 ###
```
/ # wget -q -O - http://hostname-service-clusterip:8080 | grep Hello
        <p>Hello,  hostname-deployment-7d4f978855-rn68f</p>     </blockquote>
/ # wget -q -O - http://hostname-service-clusterip:8080 | grep Hello
        <p>Hello,  hostname-deployment-7d4f978855-vg4p9</p>     </blockquote>
/ # wget -q -O - http://hostname-service-clusterip:8080 | grep Hello
        <p>Hello,  hostname-deployment-7d4f978855-zpnxd</p>     </blockquote>    <= 새로운 파드로 요청이 전달
```

### 대화형 파드에서 hostname-service-clusterip 서비스로 반복해서 요청을 전달하는 스크립트를 실행 ###
- 반환되는 호스트명이 라운드로빈 방식으로 출력되는 것을 확인
```
/ # while true; do wget -q -O - http://hostname-service-clusterip:8080 | grep Hello; sleep 1; done
        <p>Hello,  hostname-deployment-7d4f978855-vg4p9</p>     </blockquote>
        <p>Hello,  hostname-deployment-7d4f978855-vg4p9</p>     </blockquote>
        <p>Hello,  hostname-deployment-7d4f978855-zpnxd</p>     </blockquote>
        <p>Hello,  hostname-deployment-7d4f978855-rn68f</p>     </blockquote>
        <p>Hello,  hostname-deployment-7d4f978855-vg4p9</p>     </blockquote>
        <p>Hello,  hostname-deployment-7d4f978855-rn68f</p>     </blockquote>
        <p>Hello,  hostname-deployment-7d4f978855-vg4p9</p>     </blockquote>
        <p>Hello,  hostname-deployment-7d4f978855-zpnxd</p>     </blockquote>
        <p>Hello,  hostname-deployment-7d4f978855-zpnxd</p>     </blockquote>
```
### 서비스 생성 후 실행되는 파드에는 서비스와 관련한 환경 변수가 설정되어 있음 ###
```
/ # env | grep HOSTNAME_SERVICE_CLUSTERIP
HOSTNAME_SERVICE_CLUSTERIP_PORT_8080_TCP_ADDR=172.17.44.200
HOSTNAME_SERVICE_CLUSTERIP_SERVICE_HOST=172.17.44.200
HOSTNAME_SERVICE_CLUSTERIP_SERVICE_PORT_WEB_PORT=8080
HOSTNAME_SERVICE_CLUSTERIP_PORT_8080_TCP_PORT=8080
HOSTNAME_SERVICE_CLUSTERIP_PORT_8080_TCP_PROTO=tcp
HOSTNAME_SERVICE_CLUSTERIP_PORT=tcp://172.17.44.200:8080
HOSTNAME_SERVICE_CLUSTERIP_SERVICE_PORT=8080
HOSTNAME_SERVICE_CLUSTERIP_PORT_8080_TCP=tcp://172.17.44.200:8080
```

## 세션 어피니티(sessionAffinity) => 클라이언트 IP 별로 전송 파드를 고정 ##
### yaml 파일 수정 ###
- **/home/vagrant/hostname-service-clusterip.yaml 파일을 수정**
```
apiVersion: v1
kind: Service
metadata:
  name: hostname-service-clusterip
spec:
  type: ClusterIP
  selector:
    app: webserver
  ports:
  - name: web-port
    port: 8080
    targetPort: 80
  sessionAffinity: ClientIP # 클라이언트 IP 주소에 따라 요청을 처리할 파드가 결정
```
### 서비스 배포 ###
```
vagrant@master-node:~$ kubectl apply -f hostname-service-clusterip.yaml
service/hostname-service-clusterip configured

vagrant@master-node:~$ kubectl get pod,service
NAME                                       READY   STATUS    RESTARTS   AGE
pod/hostname-deployment-7d4f978855-rn68f   1/1     Running   0          82m
pod/hostname-deployment-7d4f978855-vg4p9   1/1     Running   0          82m
pod/hostname-deployment-7d4f978855-zpnxd   1/1     Running   0          16m

NAME                                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/hostname-service-clusterip   ClusterIP   172.17.44.200   <none>        8080/TCP   29m
service/kubernetes                   ClusterIP   172.17.0.1      <none>        443/TCP    4d21h
```
### 대화형 파드에서 hostname-service-clusterip 서비스로 반복해서 요청 전달 ###
```
vagrant@master-node:~$ kubectl run -it --rm debug --image=docker.io/busybox --restart=Never sh
If you don't see a command prompt, try pressing enter.
/ #
/ #
/ # ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
4: eth0@if27: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1480 qdisc noqueue qlen 1000
    link/ether 9e:8a:66:0f:55:c0 brd ff:ff:ff:ff:ff:ff
    inet 172.16.158.54/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::9c8a:66ff:fe0f:55c0/64 scope link
       valid_lft forever preferred_lft forever
```
```
/ # while true; do wget -q -O - http://hostname-service-clusterip:8080 | grep Hello; sleep 1; done
        <p>Hello,  hostname-deployment-7d4f978855-vg4p9</p>     </blockquote>    <= 동일한 파드가 응답하는 것을 확인
        <p>Hello,  hostname-deployment-7d4f978855-vg4p9</p>     </blockquote>
        <p>Hello,  hostname-deployment-7d4f978855-vg4p9</p>     </blockquote>
        <p>Hello,  hostname-deployment-7d4f978855-vg4p9</p>     </blockquote>
        <p>Hello,  hostname-deployment-7d4f978855-vg4p9</p>     </blockquote>
        <p>Hello,  hostname-deployment-7d4f978855-vg4p9</p>     </blockquote>
```

### 위 과정을 동일하게 진행 ###
```
vagrant@master-node:~$ kubectl run -it --rm debug2 --image=docker.io/busybox --restart=Never sh
If you don't see a command prompt, try pressing enter.
/ #
/ #
/ # ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: tunl0@NONE: <NOARP> mtu 1480 qdisc noop qlen 1000
    link/ipip 0.0.0.0 brd 0.0.0.0
4: eth0@if28: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1480 qdisc noqueue qlen 1000
    link/ether c6:1c:ba:06:2e:2d brd ff:ff:ff:ff:ff:ff
    inet 172.16.158.55/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::c41c:baff:fe06:2e2d/64 scope link
       valid_lft forever preferred_lft forever
```
```
/ # while true; do wget -q -O - http://hostname-service-clusterip:8080 | grep Hello; sleep 1; done
        <p>Hello,  hostname-deployment-7d4f978855-rn68f</p>     </blockquote>
        <p>Hello,  hostname-deployment-7d4f978855-rn68f</p>     </blockquote>
        <p>Hello,  hostname-deployment-7d4f978855-rn68f</p>     </blockquote>
        <p>Hello,  hostname-deployment-7d4f978855-rn68f</p>     </blockquote>
        <p>Hello,  hostname-deployment-7d4f978855-rn68f</p>     </blockquote>
```
