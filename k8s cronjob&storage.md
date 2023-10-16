## 크론잡(cronjob) ##
- Job을 시간 기준으로 관리하도록 설정
- 지정한 시간에 한번만 Job을 실행하거나 지정한 시간동안 주기적으로 Job을 반복 실행
- Cron 형식 시간을 지정
- UNIX 애플리케이션 프로그램 데이터, 데이터베이스와 같이 **중요 데이터를 백업**하는데 사용
- 생성된 파드의 개수가 정해진 수를 넘어서면 가비지 수집 컨트롤러가 종료된 파드를 삭제

![image](https://github.com/xodbs1123/Kubernetes/assets/61976898/f77a0fa5-1e33-4be0-b1fe-9d0b859ac14e)

### 크론잡 사용하기 ####
- /home/vagrant/cronjob.yaml 작성
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *" # 분(0-59), 시(0-23), 일(1-31), 월(1-12), 요일(0-7)
  jobTemplate:
    spec:                 # job spec
      template:
        spec:             # pod spec
          containers:
          - name: hello
            image: docker.io/busybox
            args: 
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes Cluster
          imagePullSecrets:
          - name: regcred  
          restartPolicy: OnFailure
```
```
vagrant@master-node:~$ kubectl apply -f cronjob.yaml
cronjob.batch/hello created

vagrant@master-node:~$ kubectl describe cronjob.batch/hello
Name:                          hello
Namespace:                     default
Labels:                        <none>
Annotations:                   <none>
Schedule:                      */1 * * * *
Concurrency Policy:            Allow
Suspend:                       False
Successful Job History Limit:  3
Failed Job History Limit:      1
Starting Deadline Seconds:     <unset>
Selector:                      <unset>
Parallelism:                   <unset>
Completions:                   <unset>
Pod Template:
  Labels:  <none>
  Containers:
   hello:
    Image:      docker.io/busybox
    Port:       <none>
    Host Port:  <none>
    Args:
      /bin/sh
      -c
      date; echo Hello from the Kubernetes Cluster
    Environment:     <none>
    Mounts:          <none>
  Volumes:           <none>
Last Schedule Time:  Mon, 16 Oct 2023 01:32:00 +0000
Active Jobs:         <none>
Events:
  Type    Reason            Age    From                Message
  ----    ------            ----   ----                -------
  Normal  SuccessfulCreate  3m20s  cronjob-controller  Created job hello-28290329
  Normal  SawCompletedJob   3m14s  cronjob-controller  Saw completed job: hello-28290329, status: Complete
  Normal  SuccessfulCreate  2m20s  cronjob-controller  Created job hello-28290330
  Normal  SawCompletedJob   2m13s  cronjob-controller  Saw completed job: hello-28290330, status: Complete
  Normal  SuccessfulCreate  80s    cronjob-controller  Created job hello-28290331
  Normal  SawCompletedJob   75s    cronjob-controller  Saw completed job: hello-28290331, status: Complete
  Normal  SuccessfulCreate  20s    cronjob-controller  Created job hello-28290332
  Normal  SuccessfulDelete  15s    cronjob-controller  Deleted job hello-28290329
  Normal  SawCompletedJob   15s    cronjob-controller  Saw completed job: hello-28290332, status: Complete
				⇐ 1분 단위로 잡이 생성되는 것을 확인
```
```
vagrant@master-node:~$ kubectl get pod
NAME                   READY   STATUS      RESTARTS   AGE	
hello-28290333-2wsr9   0/1     Completed   0          2m33s	⇐ 종료된 파드 목록 기본적으로 3개까지 보존
hello-28290334-8mbfv   0/1     Completed   0          92s				   
hello-28290335-tpw6d   0/1     Completed   0          33s


vagrant@master-node:~$ kubectl delete cronjob hello
cronjob.batch "hello" deleted

vagrant@master-node:~$ kubectl get cronjob,job,pod			⇐ 크론잡을 삭제하면 생성했던 잡과 파드도 함께 삭제
No resources found in default namespace.
```
### 크론잡 설정 ###
- **spec.schedule**
  - cron 형식으로 스케줄을 기술
  - 분(0-59), 시(0-23), 일(1-31), 월(1-12), 요일(0-7, 0: 일요일, 7: 일요일)
- **spec.startingDeadlineSeconds**
  - 지정된 시간에 크론잡이 실행되지 못했을 때 필드 값으로 설정한 시간까지 지나면 크론잡이 실행되지 않게 함
- **spec.concurrentPolicy**
  - 크론잡이 실행하는 잡의 동시성을 관리
    - Allow => 기본값, 여러 개의 잡을 동시에 실행할 수 있게 함
    - Forbid => 동시 실행을 금지, 이전에 실행했던 잡이 정상적으로 종료되지 않고 실행 중인 상태에서 새로운 잡을 실행해야 할 시간이 되면, 해당 시간에 새로운 잡을 실행하지 않고 다음 지정된 시간에 잡을 실행
    - Replace => 이전에 실행했던 잡이 실행 중인 상태에서 실행할 시간이 되면, 이전에 실행 중이던 잡을 새로운 잡으로 대체
- **spec.succesfulJobHistoryLimit**
  - 정상 종료된 잡 내역의 보관 개수 (기본갑 : 3)
- **spec.failedJobHistoryLimit**
  - 비정상 종료된 잡 내역의 보관 개수 (기본값 : 1)
### 크론잡 설정 실습 ###
- /home/vagrant/cronjob-currency.yaml 작성
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-concurrency
spec:
  schedule: "*/1 * * * *"
  startingDeadlineSeconds: 600
  concurrencyPolicy: Forbid
  jobTemplate:
    spec:                 # job spec
      template:
        spec:             # pod spec
          containers:
          - name: hello
            image: docker.io/busybox
            args:
            - /bin/sh
            - -c
            - date; echo Hello from the Kubernetes Cluster; sleep 6000 # 6000초
          imagePullSecrets:
          - name: regcred  
          restartPolicy: OnFailure
```
- concurrencyPolicy: Forbid
```
vagrant@master-node:~$ kubectl apply -f cronjob-currency.yaml
cronjob.batch/hello-concurrency created

vagrant@master-node:~$ kubectl get cronjob,job,pod
NAME                              SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob.batch/hello-concurrency   */1 * * * *   False     1        68s             70s

NAME                                   COMPLETIONS   DURATION   AGE
job.batch/hello-concurrency-28290372   0/1           68s        68s		

NAME                                   READY   STATUS    RESTARTS   AGE
pod/hello-concurrency-28290372-crdvk   1/1     Running   0          68s

1분 간격으로 잡을 실행하도록 스케줄했으나, sleep 6000 코드로 인해 잡이 끝나지 않음
currencyPolicy 필드를 Forbid로 설정했기 때문에 잡을 동시에 실행하지 않고 기다림
⇒ 앞에 잡이 끝나지 않기 때문에 다음 잡을 진행할 수 없음
```
- concurrencyPolicy: Allow
```
vagrant@master-node:~$ kubectl edit cronjob hello-concurrency

spec:
  concurrencyPolicy: Allow			⇐ Forbid를 Allow로 변경하고 저장 후 종료 
  failedJobsHistoryLimit: 1
  jobTemplate:
    metadata:
      creationTimestamp: null

cronjob.batch/hello-concurrency edited


vagrant@master-node:~$ kubectl get cronjob,job,pod
NAME                              SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob.batch/hello-concurrency   */1 * * * *   False     2        32s             5m34s

NAME                                   COMPLETIONS   DURATION   AGE
job.batch/hello-concurrency-28290372   0/1           5m32s      5m32s		⇐ 기존 잡이 종료되지 않고 유지
job.batch/hello-concurrency-28290377   0/1           31s        31s		⇐ 새로운 잡이 생성(추가)

NAME                                   READY   STATUS    RESTARTS   AGE
pod/hello-concurrency-28290372-crdvk   1/1     Running   0          5m32s
pod/hello-concurrency-28290377-lptqs   1/1     Running   0          31s		⇐ 1분 마다 하나씩 추가	



vagrant@master-node:~$ kubectl get cronjob,job,pod
NAME                              SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob.batch/hello-concurrency   */1 * * * *   False     4        2s              7m4s

NAME                                   COMPLETIONS   DURATION   AGE
job.batch/hello-concurrency-28290372   0/1           7m2s       7m2s
job.batch/hello-concurrency-28290377   0/1           2m1s       2m1s
job.batch/hello-concurrency-28290378   0/1           62s        62s
job.batch/hello-concurrency-28290379   0/1           2s         2s

NAME                                   READY   STATUS              RESTARTS   AGE
pod/hello-concurrency-28290372-crdvk   1/1     Running             0          7m2s
pod/hello-concurrency-28290377-lptqs   1/1     Running             0          2m1s
pod/hello-concurrency-28290378-8xn9p   1/1     Running             0          62s
pod/hello-concurrency-28290379-lp9tx   0/1     ContainerCreating   0          2s
```
- concurrencyPolicy: Replace
```
vagrant@master-node:~$ kubectl edit cronjob hello-concurrency

spec:
  concurrencyPolicy: Replace				⇐ Allow를 Replace로 변경하고 저장 후 종료
  failedJobsHistoryLimit: 1
  jobTemplate:
    metadata:
      creationTimestamp: null

cronjob.batch/hello-concurrency edited



vagrant@master-node:~$ kubectl get cronjob,job,pod
NAME                              SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob.batch/hello-concurrency   */1 * * * *   False     1        5s              10m

NAME                                   COMPLETIONS   DURATION   AGE
job.batch/hello-concurrency-28290382   0/1           5s         5s

NAME                                   READY   STATUS        RESTARTS   AGE
pod/hello-concurrency-28290372-crdvk   1/1     Terminating   0          10m
pod/hello-concurrency-28290377-lptqs   1/1     Terminating   0          5m4s
pod/hello-concurrency-28290378-8xn9p   1/1     Terminating   0          4m5s
pod/hello-concurrency-28290379-lp9tx   1/1     Terminating   0          3m5s
pod/hello-concurrency-28290380-vh2c8   1/1     Terminating   0          2m5s
pod/hello-concurrency-28290381-m7gmh   1/1     Terminating   0          65s		⇐ 기존 잡을 모두 종료하고
pod/hello-concurrency-28290382-xp8ts   1/1     Running       0          5s		   새로운 잡을 실행

vagrant@master-node:~$ kubectl get cronjob,job,pod
NAME                              SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cronjob.batch/hello-concurrency   */1 * * * *   False     1        8s              11m

NAME                                   COMPLETIONS   DURATION   AGE
job.batch/hello-concurrency-28290383   0/1           8s         8s

NAME                                   READY   STATUS        RESTARTS   AGE
pod/hello-concurrency-28290382-xp8ts   1/1     Terminating   0          68s
pod/hello-concurrency-28290383-5kxbm   1/1     Running       0          8s
```
## 스토리지 ##
- 상태 없는(stateless) 애플리케이션 => 디플로이먼트의 각 파드는 별도의 데이터를 가지고 있지 않으며, 단순히 요청에 대한 응답만 변환
- 상태 있는(stateful) 애플리케이션 => 데이터베이스처럼 파드 내부에 특정 데이터를 보유해야 한느 애플리케이션

![image](https://github.com/xodbs1123/Kubernetes/assets/61976898/ed1bae29-73b1-4352-a71e-6913fdb06d59)


### 로컬볼륨 emptyDir => 파드 내의 컨테이너 간 임시 데이터 공유 ###
- /home/vagrant/emptydir-volume-pod.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-volume-pod
spec:
  containers:
  - name: content-creator
    image: docker.io/busybox
    args: ["tail", "-f", "/dev/null"]
    volumeMounts:
    - name: emptydir-volume
      mountPath: /data
  - name: webserver
    image: docker.io/httpd:2
    volumeMounts:
    - name: emptydir-volume
      mountPath: /usr/local/apache2/htdocs/
  imagePullSecrets:
  - name: regcred  
  volumes:
    - name: emptydir-volume
      emptyDir: {}
```
```
vagrant@master-node:~$ kubectl apply -f emptydir-volume-pod.yaml
pod/emptydir-volume-pod created

vagrant@master-node:~$ kubectl get pod
NAME                  READY   STATUS    RESTARTS   AGE
emptydir-volume-pod   2/2     Running   0          19s
```
```
vagrant@master-node:~$ kubectl exec -it emptydir-volume-pod -c content-creator -- /bin/sh
/ #                                     
/ # echo Hello, emptyDir!!! > /data/hello.html
/ # cat /data/hello.html
Hello, emptyDir!!!
/ # exit


vagrant@master-node:~$ kubectl exec -it emptydir-volume-pod -c webserver -- cat /usr/
local/apache2/htdocs/hello.html
Hello, emptyDir!!!


vagrant@master-node:~$ kubectl delete -f emptydir-volume-pod.yaml
pod "emptydir-volume-pod" deleted
```
### 로컬볼륨 hostPath => 워커 노드의 로컬 디렉터리를 볼륨으로 사용 ###
- /home/vagrant/hostpath-volume-pod.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-volume-pod
spec:
  containers:
  - name: busybox
    image: docker.io/busybox
    args: ["tail", "-f", "/dev/null"]
    volumeMounts:
    - name: hostpath-volume
      mountPath: /etc/data
  imagePullSecrets:
  - name: regcred  
  volumes:
    - name: hostpath-volume
      hostPath:
        path: /tmp
```
```
vagrant@master-node:~$ kubectl apply -f hostpath-volume-pod.yaml
pod/hostpath-volume-pod created

vagrant@master-node:~$ kubectl exec -it hostpath-volume-pod -- touch /etc/data/mydata

vagrant@master-node:~$ kubectl get pod -o wide
NAME                  READY   STATUS    RESTARTS   AGE   IP              NODE            NOMINATED NODE   READINESS GATES
hostpath-volume-pod   1/1     Running   0          71s   172.16.158.40   worker-node02   <none>           <none>
```
```
vagrant@master-node:~$ ssh vagrant@10.0.0.12			⇐ node02에 vagrant 계정으로 ssh 접속
vagrant@10.0.0.12's password: vagrant
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-83-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon Oct 16 03:20:20 AM UTC 2023

  System load:  0.04248046875      Users logged in:        1
  Usage of /:   36.6% of 30.34GB   IPv4 address for eth0:  10.0.2.15
  Memory usage: 22%                IPv4 address for eth1:  10.0.0.12
  Swap usage:   0%                 IPv4 address for tunl0: 172.16.158.0
  Processes:    158


This system is built by the Bento project by Chef Software
More information can be found at https://github.com/chef/bento
Last login: Sun Oct 15 23:11:39 2023 from 10.0.0.10
vagrant@worker-node02:~$

vagrant@worker-node02:~$  ls /tmp/
mydata
snap-private-tmp
systemd-private-4523bca7fdbb438fad9fe477c1a504c0-ModemManager.service-S4v1SG
systemd-private-4523bca7fdbb438fad9fe477c1a504c0-systemd-logind.service-qA564N
systemd-private-4523bca7fdbb438fad9fe477c1a504c0-systemd-resolved.service-2z7lR4
systemd-private-4523bca7fdbb438fad9fe477c1a504c0-upower.service-jRImBL

파드가 다른 노드에 생성되면 이전에 생성된 데이터를 참조할 수 없음
```
```
vagrant@worker-node02:~$ exit			⇐ node02로 SSH 연결을 종료(해제)
logout
Connection to 10.0.0.12 closed.


vagrant@master-node:~$ kubectl delete -f hostpath-volume-pod.yaml	⇐ 파드 삭제 후 
pod "hostpath-volume-pod" deleted						   node02로 접속해서 볼륨 디렉터리를 확인


vagrant@master-node:~$ ssh vagrant@10.0.0.12
vagrant@10.0.0.12's password:
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-83-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon Oct 16 03:20:20 AM UTC 2023

  System load:  0.04248046875      Users logged in:        1
  Usage of /:   36.6% of 30.34GB   IPv4 address for eth0:  10.0.2.15
  Memory usage: 22%                IPv4 address for eth1:  10.0.0.12
  Swap usage:   0%                 IPv4 address for tunl0: 172.16.158.0
  Processes:    158


This system is built by the Bento project by Chef Software
More information can be found at https://github.com/chef/bento
Last login: Mon Oct 16 03:20:21 2023 from 10.0.0.10
vagrant@worker-node02:~$ ls /tmp
mydata								⇐ 파드가 삭제되어도 생성한 파일이 유지되고 있는 것을 확인
snap-private-tmp
systemd-private-4523bca7fdbb438fad9fe477c1a504c0-ModemManager.service-S4v1SG
systemd-private-4523bca7fdbb438fad9fe477c1a504c0-systemd-logind.service-qA564N
systemd-private-4523bca7fdbb438fad9fe477c1a504c0-systemd-resolved.service-2z7lR4
systemd-private-4523bca7fdbb438fad9fe477c1a504c0-upower.service-jRImBL


동일 노드에 파드가 실행되면 해당 파일을 공유하게 됨
```
## 네트워크 볼륨 ##
### NFS 서버를 위한 디플로이먼트와 서비스를 생성 ###
- /home/vagrant/nfs-deployment-service.yaml
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-deployment
spec:
  selector:
    matchLabels:
      role: nfs-server
  template:
    metadata:
      labels:
        role: nfs-server
    spec:
      containers:
      - name: nfs-server-container
        image: gcr.io/google_containers/volume-nfs:0.8
        ports:
        - name: nfs
          containerPort: 2049
        - name: mountd
          containerPort: 20048
        - name: rpcbind
          containerPort: 111
        securityContext:
          privileged: true
      imagePullSecrets:
      - name: regcred          
---
apiVersion: v1
kind: Service
metadata:
  name: nfs-service
spec:
  ports:
  - name: nfs
    port: 2049
  - name: mountd
    port: 20048
  - name: rpcbind
    port: 111
  selector:
    role: nfs-server
```
```
vagrant@master-node:~$ kubectl apply -f nfs-deployment-service.yaml
deployment.apps/nfs-deployment created
service/nfs-service created

vagrant@master-node:~$ kubectl get all
NAME                                  READY   STATUS    RESTARTS   AGE
pod/nfs-deployment-65fd996696-jsvkw   1/1     Running   0          35m

NAME                  TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
service/kubernetes    ClusterIP   172.17.0.1      <none>        443/TCP                      10d
service/nfs-service   ClusterIP   172.17.11.120   <none>        2049/TCP,20048/TCP,111/TCP   35m

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nfs-deployment   1/1     1            1           35m

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/nfs-deployment-65fd996696   1         1         1       35m
```
- **모드 워크 노드에 nfs와 관련한 패키지 설치**
```
vagrant@master-node:~$ ssh vagrant@10.0.0.11
vagrant@10.0.0.11's password: vagrant
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-83-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon Oct 16 05:04:24 AM UTC 2023

  System load:  0.44580078125      Users logged in:        0
  Usage of /:   29.9% of 30.34GB   IPv4 address for eth0:  10.0.2.15
  Memory usage: 19%                IPv4 address for eth1:  10.0.0.11
  Swap usage:   0%                 IPv4 address for tunl0: 172.16.87.192
  Processes:    154


This system is built by the Bento project by Chef Software
More information can be found at https://github.com/chef/bento
Last login: Sun Oct 15 23:09:55 2023 from 10.0.0.10
vagrant@worker-node01:~$

vagrant@worker-node01:~$ sudo apt-get install -y nfs-common
                                                 ~~~~~~~~~~ 
                                              NFS 클라이언트를 지원한 리눅스 패키지 
                                              NFS 서버로부터 파일 및 디렉터리를 마우트하고 읽고 쓸 수 있도록 설정
                                              ~~~ 
                                              네트워크 파일 공유 프로토콜로, 
                                              여러 컴퓨터 간에 파일 및 디렉터리를 공유하고 액세스하는데 사용  


vagrant@worker-node01:~$ exit
logout
Connection to 10.0.0.11 closed.
```
```
vagrant@master-node:~$ ssh vagrant@10.0.0.12
vagrant@10.0.0.12's password: vagrant
Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.0-83-generic x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage

  System information as of Mon Oct 16 05:08:16 AM UTC 2023

  System load:  0.03173828125      Users logged in:        0
  Usage of /:   37.5% of 30.34GB   IPv4 address for eth0:  10.0.2.15
  Memory usage: 27%                IPv4 address for eth1:  10.0.0.12
  Swap usage:   0%                 IPv4 address for tunl0: 172.16.158.0
  Processes:    170


This system is built by the Bento project by Chef Software
More information can be found at https://github.com/chef/bento
Last login: Mon Oct 16 03:30:59 2023 from 10.0.0.10
vagrant@worker-node02:~$
vagrant@worker-node02:~$ sudo apt-get install -y nfs-common

vagrant@worker-node02:~$ exit
logout
Connection to 10.0.0.12 closed.
vagrant@master-node:~$
```
- **서비스의 CLUSTER-IP 확인**
```
vagrant@master-node:~$ kubectl get service
NAME          TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
kubernetes    ClusterIP   172.17.0.1      <none>        443/TCP                      10d
nfs-service   ClusterIP   172.17.11.120   <none>        2049/TCP,20048/TCP,111/TCP   85m
```
- **NFS 서버를 컨테이너에 마운트하는 파드 생성**
-  /home/vagrant/nfs-volume-pod.yaml
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nfs-volume-pod
spec:
  containers:
  - name: nfs-volume-container
    image: docker.io/busybox
    args: ["tail", "-f", "/dev/null"]
    volumeMounts:
    - name: nfs-volume
      mountPath: /mnt
  imagePullSecrets:
    - name: regcred
  volumes:
  - name: nfs-volume
    nfs:
      path: /
      server: 172.17.11.120   # nfs-service 서비스의 CLUSTER-IP 주소
```
```
vagrant@master-node:~$ kubectl apply -f nfs-volume-pod.yaml
pod/nfs-volume-pod created







vagrant@master-node:~$ kubectl delete -f nfs-volume-pod.yaml
pod "nfs-volume-pod" deleted

vagrant@master-node:~$ kubectl delete -f nfs-deployment-service.yaml
deployment.apps "nfs-deployment" deleted
```
## 퍼시턴트 볼륨과 퍼시스턴트 볼륨 클레임 ##
- 지금까지는 파드의 YAML 파일에 볼륨 정보를 직접 명시하는 방식
- 볼륨과 애플리케이션의 정의가 서로 밀접하게 연관되어 있어 서로 분리하기 어려움
- 네트워크 볼륨 타입과 별도의 YAML 파일 작성 필요
- PV, PVC 제공 => 파드가 볼륨의 세부적인 사항을 몰라도 볼륨을 사용할 수 있도록 추상화해 주는 역할

![image](https://github.com/xodbs1123/Kubernetes/assets/61976898/7738dde4-fef9-4299-94ce-28b9ba9d2bb4)


### NFS를 퍼스턴트 볼륨으로 사용 ###
- nfs 서버 디플로이먼트와 서비스를 생성
```
vagrant@master-node:~$ kubectl apply -f nfs-deployment-service.yaml
deployment.apps/nfs-deployment created
service/nfs-service created

vagrant@master-node:~$ kubectl get all
NAME                                  READY   STATUS    RESTARTS   AGE
pod/nfs-deployment-65fd996696-ggtcg   1/1     Running   0          62s

NAME                  TYPE        CLUSTER-IP    EXTERNAL-IP   PORT(S)                      AGE
service/kubernetes    ClusterIP   172.17.0.1    <none>        443/TCP                      10d
service/nfs-service   ClusterIP   172.17.6.97   <none>        2049/TCP,20048/TCP,111/TCP   62s

NAME                             READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nfs-deployment   1/1     1            1           62s

NAME                                        DESIRED   CURRENT   READY   AGE
replicaset.apps/nfs-deployment-65fd996696   1         1         1       62s
```
- **퍼시턴트 폴륨 생성**
- /home/vagrant/nfs-persistentvolume.yaml
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-persistentvolume
spec:
  capacity:
    storage: 1Gi          ## 볼륨의 크기 1G
  accessModes:
  - ReadWriteOnce         ## 하나의 파드(또는 인스턴스)에 의해서만 마운트될 수 있음
  nfs:
    path: /
    server: {CLUSTER-IP}
```
```
vagrant@master-node:~$ cat nfs-persistentvolume.yaml | sed "s/{CLUSTER-IP}/$(kubectl get service nfs-service -o jsonpath='{.spec.clusterIP}')/g" | kubectl apply -f -
persistentvolume/nfs-persistentvolume created

vagrant@master-node:~$ kubectl get pv
NAME                   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
nfs-persistentvolume   1Gi        RWO            Retain           Available                                   22s
                                                                  ~~~~~~~~~
Available = 사용 가능 = 아직 클레임에 바인딩되지 않은 사용할 수 있는 리소스
Bound = 바인딩 = 볼륨이 클레임에 바인딩됨                                                                               
Released = 릴리스 = 크레임이 삭제되었지만 클러스터에서 아직 리소를 반환하지 않음
Failed = 실패 = 볼륨이 자동 반환에 실패됨
```
- **퍼시턴트 볼륨 클레임과 파드 생성**
- /home/vagrant/nfs-persistentvolumeclaim-pod.yaml
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-persistentvolumeclaim
spec:
  storageClassName: ""
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: v1
kind: Pod
metadata:
  name: nfs-mount-container
spec:
  containers:
  - name: nfs-mount-container
    image: docker.io/busybox
    args: ["tail", "-f", "/dev/null"]
    volumeMounts:
    - name: nfs-volume
      mountPath: /mnt
  imagePullSecrets:
    - name: regcred
  volumes:
  - name: nfs-volume
    persistentVolumeClaim:
      claimName: nfs-persistentvolumeclaim
```
```
vagrant@master-node:~$ kubectl apply -f nfs-persistentvolumeclaim-pod.yaml
persistentvolumeclaim/nfs-persistentvolumeclaim created
pod/nfs-mount-container created

vagrant@master-node:~$ kubectl get pv,pvc,pod
NAME                                    CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM                               STORAGECLASS   REASON   AGE
persistentvolume/nfs-persistentvolume   1Gi        RWO            Retain           Bound    default/nfs-persistentvolumeclaim                           9m53s

NAME                                              STATUS   VOLUME                 CAPACITY   ACCESS MODES   STORAGECLASS   AGE
persistentvolumeclaim/nfs-persistentvolumeclaim   Bound    nfs-persistentvolume   1Gi        RWO                           21s

NAME                                  READY   STATUS              RESTARTS   AGE
pod/nfs-deployment-65fd996696-ggtcg   1/1     Running             0          21m
pod/nfs-mount-container               0/1     ContainerCreating   0          21s
```
- RECLAIM POLICY : 퍼시스턴트 볼륨 클레임이 삭제될 때 퍼시스턴트 볼륨과 연결되어 있던 저장소에 저장된 파일에 대한 처리 정책
  - Retain: 퍼시스턴트 볼륨 클레임이 삭제되어도 저장소에 있던 파일을 삭제하지 않는 정책
  - Delete: 퍼시스턴트 볼륨 클레임이 삭제되면 퍼시스턴트 볼륨과 연결된 저장소 자체를 삭제
  - Recycle: 퍼시스턴트 볼륨 클레임이 삭제되면 퍼시스턴트 볼륨과 연결된 저장소 데이터는 삭제하지만 저장소 볼륨 자체는 삭제하지 않고 유지


- ACCESS MODES
  - RWO = ReadWriteOnce : 하나의 노드에서 해당 볼륨이 읽기-쓰기로 마운트됨
  - ROX = ReadOnlyMay : 볼륨은 많은 노드에서 읽기 전용으로 마운트됨
  - RWX = ReadWriteMany : 볼륨은 많은 노드에서 읽기-쓰기로 마운트됨
  - RWOP = ReadWriteOncePod : 볼륨이 단일 파드에서 읽기-쓰기로 마운트됨. 전체 클러스터에서 단 하나의 파드만 해당 퍼시스턴트 볼륨 클레임을 읽거나 쓸 수 있어야 하는 경우에 사용
