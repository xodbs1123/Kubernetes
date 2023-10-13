## 시크릿(Secret) ##
- 민감한 정보를 저장하기 위한 용도로 사용
- 네임스페이스에 종속되는 쿠버네티스 오브젝트

### 시크릿 생성 방법 ###
- **--from-literal 옵션을 이용**
  - 시크릿 종류를 명시하지 않을 경우 자동으로 설정되는 타입
  - 사용자가 정의하는 데이터를 저장하는 일반 목적의 시크릿
  - kubectl create secret 명령어로 시크릿을 생성할 때는 generic으로 명시
```
vagrant@master-node:~$ kubectl create secret generic my-password --from-literal password=p@ssw0rd
secret/my-password created

vagrant@master-node:~$ kubectl get secret

NAME          TYPE                             DATA   AGE
my-password   Opaque                           1      31s
regcred       kubernetes.io/dockerconfigjson   1      46h
```
- **--from file 옵션을 이용**
```
vagrant@master-node:~$ echo mypassword > pw1 && echo yourpassword > pw2

vagrant@master-node:~$ ls pw*
pw1  pw2
```
```
vagrant@master-node:~$ kubectl create secret generic our-password --from-file pw1 --from-file pw2
secret/our-password created

vagrant@master-node:~$ kubectl describe secret our-password
Name:         our-password
Namespace:    default
Labels:       <none>
Annotations:  <none>

Type:  Opaque

Data
====
pw1:  11 bytes        <= 파일 이름이 키로 사용되고 있고, 시크릿 키: 값의 크기(byte) 형식으로 출력
pw2:  13 bytes
```
```
vagrant@master-node:~$ kubectl get secret our-password -o yaml
apiVersion: v1
data:
  pw1: bXlwYXNzd29yZAo=          <= secret 값이 BASE64로 인코딩되어 있는 것을 확인 
  pw2: eW91cnBhc3N3b3JkCg==
kind: Secret
metadata:
  creationTimestamp: "2023-10-13T01:50:01Z"
  name: our-password
  namespace: default
  resourceVersion: "147442"
  uid: 2f61706b-7973-4ab4-a541-206ae694fa66
type: Opaque

vagrant@master-node:~$ echo bXlwYXNzd29yZAo= | base64 -d
mypassword
```

### 파드에서 시크릿 사용하는 방법 1) 시크릿의 값을 파드의 환경 변수로 사용 ###
- **시크릿에 저장된 모든 키-값 쌍을 파드의 환경 변수로 설정**
- /home/vagrant/secret-env-from-all.yaml 작성
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-from-all
spec:
  containers:
  - name: my-container
    image: docker.io/busybox
    args: ['tail', '-f', '/dev/null']
    envFrom:
    - secretRef:
        name: my-password
```
```
vagrant@master-node:~$ kubectl apply -f secret-env-from-all.yaml
pod/secret-env-from-all created
vagrant@master-node:~$ kubectl get pod
NAME                  READY   STATUS    RESTARTS   AGE
secret-env-from-all   1/1     Running   0          6s

vagrant@master-node:~$ kubectl exec secret-env-from-all -- env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
TERM=xterm
HOSTNAME=secret-env-from-all
password=p@ssw0rd
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://172.17.0.1:443
KUBERNETES_PORT_443_TCP=tcp://172.17.0.1:443
KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=172.17.0.1
KUBERNETES_SERVICE_HOST=172.17.0.1
HOME=/root
```
- **시크릿에 저장된 키-값 쌍 중 원하는 데이터만 파드의 환경 변수로 설정**
- /home/vagrant/secret-env-from-selective.yaml 작성
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-env-from-selective
spec:
  containers:
  - name: my-container
    image: docker.io/busybox
    args: ['tail', '-f', '/dev/null']
    env:
    - name: YOUR_PASSWORD           ## 새로운 환경변수 이름을 설정
      valueFrom:
        secretKeyRef:
          name: our-password        ## 값을 가져올 시크릿을 지정
          key: pw2                  ## 값을 가져올 시크릿의 키를 지정
```
```
vagrant@master-node:~$ kubectl apply -f secret-env-from-selective.yaml
pod/secret-env-from-selective created
vagrant@master-node:~$ kubectl exec secret-env-from-selective -- env
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
TERM=xterm
HOSTNAME=secret-env-from-selective
YOUR_PASSWORD=yourpassword

KUBERNETES_PORT_443_TCP_PROTO=tcp
KUBERNETES_PORT_443_TCP_PORT=443
KUBERNETES_PORT_443_TCP_ADDR=172.17.0.1
KUBERNETES_SERVICE_HOST=172.17.0.1
KUBERNETES_SERVICE_PORT=443
KUBERNETES_SERVICE_PORT_HTTPS=443
KUBERNETES_PORT=tcp://172.17.0.1:443
KUBERNETES_PORT_443_TCP=tcp://172.17.0.1:443
HOME=/root
```
### 파드에서 시크릿 사용하는 방법 2) 시크릿의 값을 파드 내부의 파일로 마운트해서 사용 ###
- **시크릿의 모든 키-값 데이터를 파드에 마운트**
- /home/vagrant/secret-volume-from-all.yaml 작성
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-from-all
spec:
  containers:
  - name: my-container
    image: docker.io/busybox
    args: ['tail', '-f', '/dev/null']
    volumeMounts:                     ## 볼륨을 컨테이너에 마운트(매핑)
    - name: secret-volume             ## 마운트할 볼륨 이름
      mountPath: /etc/secret          ## 컨테이너의 디렉터리
  volumes:
  - name: secret-volume               ## 볼륨 이름
    secret:
      secretName: our-password        ## 볼륨에 맵핑될 시크릿의 이름
```
```
vagrant@master-node:~$ kubectl apply -f secret-volume-from-all.yaml
pod/secret-volume-from-all created

vagrant@master-node:~$ kubectl exec secret-volume-from-all -- ls /etc/secret
pw1    <= our-password 시크릿의 각 항목의 키가 파일로 생성된 것을 확인
pw2
```
```
vagrant@master-node:~$ kubectl exec secret-volume-from-all -- cat /etc/secret/pw1
mypassword				⇐ 각 파일의 내용은 시크릿의 키에 해당하는 값이 설정된 것을 확인

vagrant@master-node:~$ kubectl exec secret-volume-from-all -- cat /etc/secret/pw2
yourpassword
```

- **시크릿에 존재하는 키-값 중 원하는 데이터만 파드에 마운트**
- /home/vagrant/secrete-volume-from-selective.yaml 작성
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-volume-from-all
spec:
  containers:
  - name: my-container
    image: docker.io/busybox
    args: ['tail', '-f', '/dev/null']
    volumeMounts:                     ## 볼륨을 컨테이너에 마운트(매핑)
    - name: secret-volume             ## 마운트할 볼륨 이름
      mountPath: /etc/secret          ## 컨테이너의 디렉터리
  volumes:
  - name: secret-volume               ## 볼륨 이름
    secret:
      secretName: our-password        ## 볼륨에 맵핑될 시크릿의 이름
      items:
      - key: pw1                      ## 시크릿에서 가져올 값의 키를 지정
      path: password1               ## 값을 저장할 파일 이름을 지정
```
```
vagrant@master-node:~$ kubectl apply -f secret-volume-from-selective.yaml
pod/secret-volume-from-selective created

vagrant@master-node:~$ kubectl get pod
NAME                           READY   STATUS    RESTARTS   AGE
secret-volume-from-selective   1/1     Running   0          6s

vagrant@master-node:~$ kubectl exec secret-volume-from-selective -- ls /etc/secret
password1

vagrant@master-node:~$ kubectl exec secret-volume-from-selective -- cat /etc/secret/password1
mypassword
```

## 시크릿 타입 ##
https://kubernetes.io/ko/docs/concepts/configuration/secret/#secret-types
- 시크릿 타입은 여러 종류의 기밀 데이터를 프로그래밍 방식으로 용이하게 처리하기 위해 사용

![image](https://github.com/xodbs1123/Kubernetes/assets/61976898/5ba0a69f-3562-4bfc-b109-1399da0e893c)

## Job과 CronJob ##
### 잡(Job) ###
- 일회성 또는 배치 작업을 실행하기 위한 객체
- 주로 한 번 실행하고 완료해야 하는 작업을 정의하는데 사용
- 파드의 모든 컨테이너가 정상적으로 종료할 때까지 재실행
- 특정 개수 만큼의 파드를 정상적으로 실행 종료되는 것을 보장

### 잡 컨트롤러의 특징 ###
- 지정한 실행 횟수와 병행 개수에 따라 한 개 이상의 파드를 실행
- 잡은 파드 내에 있는 모든 컨테이너가 정상 종료한 경우에 파드를 정상 종료한 것으로 취급
- 잡에 기술한 파드의 실행 횟수를 전부 정상 종료하면, 잡은 종료
- 파드의 비정상 종료에 따른 재실행 횟수의 상한에 도달해도 잡을 중단
- 노드 장애 등에 의해 잡의 파드가 제거된 경우, 다른 노드에서 파드를 재실행
- 잡의 의해 실행된 파드는 잡이 삭제될 때까지 유지하고, 잡을 삭제하면 모든 파드가 삭제

![image](https://github.com/xodbs1123/Kubernetes/assets/61976898/0c71fca1-a7a1-4c7b-913d-7c54d8f5a902)

### 잡 활용 예시 ###
- **동시 실행과 순차 실행**

![image](https://github.com/xodbs1123/Kubernetes/assets/61976898/809814c9-6f25-4b8a-bd43-4915ca942fc2)

- **파드를 실행할 노드를 선택**

![image](https://github.com/xodbs1123/Kubernetes/assets/61976898/6d10cc44-957c-4c86-b72b-1082ac897af3)

- **온라인 배치 처리 요청**

![image](https://github.com/xodbs1123/Kubernetes/assets/61976898/7e61c56b-40ee-4394-9443-72375e6e2e83)

- **정기 실행 배치 처리**
  - 크론잡은 설정한 시간에 정기적으로 잡을 실행 => 데이터의 백업이나 매시간 마다 실행되는 배치 처리 등을 실행

### 잡 병렬성 관리 ###
- 잡 하나가 동시에 실행할 파드를 지정
- .spec.parallelism 필드에 설정
- 기본값은 1, 잡을 정지하려면 0으로 설정

### 잡을 이용해서 정상적으로 종료되는 파드를 실행 ###

- **/home/vagrant/job-normal-end.yaml** 작성

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-normal-end
spec:
  template:
    spec:
      containers:
      - name: busybox
        image: docker.io/busybox
        command: ["sh", "-c", "sleep 5; exit 0"]
      restartPolicy: Never
  completions: 6      ## 총 실행 회수 (0 보다 큰 정수를 설정)
```
```
vagrant@master-node:~$ kubectl apply -f job-normal-end.yaml
job.batch/job-normal-end created

vagrant@master-node:~$ kubectl get job
NAME             COMPLETIONS   DURATION   AGE
job-normal-end   6/6           61s        70s

vagrant@master-node:~$ kubectl describe job job-normal-end
Name:             job-normal-end
Namespace:        default
Selector:         batch.kubernetes.io/controller-uid=907b69a1-cdf7-46c2-812d-d7ae69656245
Labels:           batch.kubernetes.io/controller-uid=907b69a1-cdf7-46c2-812d-d7ae69656245
                  batch.kubernetes.io/job-name=job-normal-end
                  controller-uid=907b69a1-cdf7-46c2-812d-d7ae69656245
                  job-name=job-normal-end
Annotations:      batch.kubernetes.io/job-tracking:
Parallelism:      1
Completions:      6
Completion Mode:  NonIndexed
Start Time:       Fri, 13 Oct 2023 03:44:38 +0000
Completed At:     Fri, 13 Oct 2023 03:45:39 +0000
Duration:         61s
Pods Statuses:    0 Active (0 Ready) / 6 Succeeded / 0 Failed
Pod Template:
  Labels:  batch.kubernetes.io/controller-uid=907b69a1-cdf7-46c2-812d-d7ae69656245
           batch.kubernetes.io/job-name=job-normal-end
           controller-uid=907b69a1-cdf7-46c2-812d-d7ae69656245
           job-name=job-normal-end
  Containers:
   busybox:
    Image:      docker.io/busybox
    Port:       <none>
    Host Port:  <none>
    Command:
      sh
      -c
      sleep 5; exit 0
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From            Message
  ----    ------            ----  ----            -------
  Normal  SuccessfulCreate  72s   job-controller  Created pod: job-normal-end-rczdg    <= 파드가 하나씩 순차적으로 실행되고 종료되는 것을 확인할 수 있음
  Normal  SuccessfulCreate  61s   job-controller  Created pod: job-normal-end-j6rtd
  Normal  SuccessfulCreate  51s   job-controller  Created pod: job-normal-end-8k7q8
  Normal  SuccessfulCreate  41s   job-controller  Created pod: job-normal-end-dbxvc
  Normal  SuccessfulCreate  31s   job-controller  Created pod: job-normal-end-4qklb
  Normal  SuccessfulCreate  21s   job-controller  Created pod: job-normal-end-cqfq8
  Normal  Completed         11s   job-controller  Job completed                         <= 잡이 종료되는 것을 확인
```

### parallelism을 설정해서 파드를 병렬로 실행 ###
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-normal-end
spec:
  template:
    spec:
      containers:
      - name: busybox
        image: docker.io/busybox
        command: ["sh", "-c", "sleep 5; exit 0"]
      restartPolicy: Never
  completions: 6      ## 총 실행 회수 (0 보다 큰 정수를 설정)
  parallelism: 2      ## 동시 실행할 파드의 개수
```
```
vagrant@master-node:~$ kubectl apply -f job-normal-end.yaml
job.batch/job-normal-end created

vagrant@master-node:~$ kubectl describe job job-normal-end
Name:             job-normal-end
Namespace:        default
Selector:         batch.kubernetes.io/controller-uid=22b7aa9a-c6f2-4c13-8766-bbcdebceb2c7
Labels:           batch.kubernetes.io/controller-uid=22b7aa9a-c6f2-4c13-8766-bbcdebceb2c7
                  batch.kubernetes.io/job-name=job-normal-end
                  controller-uid=22b7aa9a-c6f2-4c13-8766-bbcdebceb2c7
                  job-name=job-normal-end
Annotations:      batch.kubernetes.io/job-tracking:
Parallelism:      2
Completions:      6
Completion Mode:  NonIndexed
Start Time:       Fri, 13 Oct 2023 05:04:18 +0000
Completed At:     Fri, 13 Oct 2023 05:04:52 +0000
Duration:         34s
Pods Statuses:    0 Active (0 Ready) / 6 Succeeded / 0 Failed
Pod Template:
  Labels:  batch.kubernetes.io/controller-uid=22b7aa9a-c6f2-4c13-8766-bbcdebceb2c7
           batch.kubernetes.io/job-name=job-normal-end
           controller-uid=22b7aa9a-c6f2-4c13-8766-bbcdebceb2c7
           job-name=job-normal-end
  Containers:
   busybox:
    Image:      docker.io/busybox
    Port:       <none>
    Host Port:  <none>
    Command:
      sh
      -c
      sleep 5; exit 0
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type    Reason            Age   From            Message
  ----    ------            ----  ----            -------
  Normal  SuccessfulCreate  53s   job-controller  Created pod: job-normal-end-4s88w    <= 파드가 두 개씩 실행되고 종료된 것을 확인할 수 있음
  Normal  SuccessfulCreate  53s   job-controller  Created pod: job-normal-end-68jkb
  Normal  SuccessfulCreate  42s   job-controller  Created pod: job-normal-end-zhfvc
  Normal  SuccessfulCreate  42s   job-controller  Created pod: job-normal-end-kmm9h
  Normal  SuccessfulCreate  32s   job-controller  Created pod: job-normal-end-ps698
  Normal  SuccessfulCreate  32s   job-controller  Created pod: job-normal-end-rkngc
  Normal  Completed         19s   job-controller  Job completed
```
### 잡을 이용해서 비정상적으로 종료되는 파드를 실행 ###
- **/home/vagrant/job-abnormal-end.yaml**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-abnormal-end
spec:
  template:
    spec:
      containers:
      - name: busybox
        image: docker.io/busybox
        command: ["sh", "-c", "sleep 5; exit 1"]
      imagePullSecrets:
      - name: regcred
      restartPolicy: Never
  # completions: 1      ## 총 실행 회수 (기본값 1)
  # parallelism: 1      ## 동시 실행할 파드의 개수 (기본값 1)
  backoffLimit: 3       ## 실패했을 때 재실행하는 최대 실행 회수
```
```
vagrant@master-node:~$ kubectl apply -f job-abnormal-end.yaml
job.batch/job-normal-end created

vagrant@master-node:~$ kubectl get job,pod
NAME                         COMPLETIONS   DURATION   AGE
job.batch/job-abnormal-end   0/1           11m        11m

NAME                         READY   STATUS   RESTARTS   AGE
pod/job-abnormal-end-2cgpt   0/1     Error    0          11m
pod/job-abnormal-end-2xxs2   0/1     Error    0          11m
pod/job-abnormal-end-4nglp   0/1     Error    0          10m
pod/job-abnormal-end-dcj27   0/1     Error    0          11m

vagrant@master-node:~$ kubectl describe job job-abnormal-end
Name:             job-abnormal-end
Namespace:        default
Selector:         batch.kubernetes.io/controller-uid=a8ecd0b5-7a10-4e49-bdcf-5e68cda47357
Labels:           batch.kubernetes.io/controller-uid=a8ecd0b5-7a10-4e49-bdcf-5e68cda47357
                  batch.kubernetes.io/job-name=job-abnormal-end
                  controller-uid=a8ecd0b5-7a10-4e49-bdcf-5e68cda47357
                  job-name=job-abnormal-end
Annotations:      batch.kubernetes.io/job-tracking:
Parallelism:      1
Completions:      1
Completion Mode:  NonIndexed
Start Time:       Fri, 13 Oct 2023 05:19:07 +0000
Pods Statuses:    0 Active (0 Ready) / 0 Succeeded / 4 Failed
Pod Template:
  Labels:  batch.kubernetes.io/controller-uid=a8ecd0b5-7a10-4e49-bdcf-5e68cda47357
           batch.kubernetes.io/job-name=job-abnormal-end
           controller-uid=a8ecd0b5-7a10-4e49-bdcf-5e68cda47357
           job-name=job-abnormal-end
  Containers:
   busybox:
    Image:      docker.io/busybox
    Port:       <none>
    Host Port:  <none>
    Command:
      sh
      -c
      sleep 5; exit 1
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type     Reason                Age   From            Message
  ----     ------                ----  ----            -------
  Normal   SuccessfulCreate      12m   job-controller  Created pod: job-abnormal-end-2cgpt  <= 파드가 4번 실행(3번 재시작)
  Normal   SuccessfulCreate      12m   job-controller  Created pod: job-abnormal-end-2xxs2
  Normal   SuccessfulCreate      11m   job-controller  Created pod: job-abnormal-end-dcj27
  Normal   SuccessfulCreate      10m   job-controller  Created pod: job-abnormal-end-4nglp
  Warning  BackoffLimitExceeded  10m   job-controller  Job has reached the specified backoff limit
```
### 여러 컨테이너 중 일부가 이상 종료할 경우 => 파드는 비정상 종료로 처리 ###
- **/home/vagrant/job-container-failed.yaml**
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: job-container-failed
spec:
  template:
    spec:
      containers:
      - name: busybox-normal
        image: docker.io/busybox
        command: ["sh", "-c", "sleep 5; exit 0"]
      - name: busybox-abnormal
        image: docker.io/busybox
        command: ["sh", "-c", "sleep 5; exit 1"]        
      imagePullSecrets:
      - name: regcred
      restartPolicy: Never
  # completions: 1      ## 총 실행 회수 (기본값 1)
  # parallelism: 1      ## 동시 실행할 파드의 개수 (기본값 1)
  backoffLimit: 2       ## 실패했을 때 재실행하는 최대 실행 회수
```
```
vagrant@master-node:~$ kubectl apply -f job-container-failed.yaml
job.batch/job-container-failed created

vagrant@master-node:~$ kubectl describe job job-container-failed
Name:             job-container-failed
Namespace:        default
Selector:         batch.kubernetes.io/controller-uid=c349351a-9b35-4fa9-bb78-f42d822bf70b
Labels:           batch.kubernetes.io/controller-uid=c349351a-9b35-4fa9-bb78-f42d822bf70b
                  batch.kubernetes.io/job-name=job-container-failed
                  controller-uid=c349351a-9b35-4fa9-bb78-f42d822bf70b
                  job-name=job-container-failed
Annotations:      batch.kubernetes.io/job-tracking:
Parallelism:      1
Completions:      1
Completion Mode:  NonIndexed
Start Time:       Fri, 13 Oct 2023 05:38:15 +0000
Pods Statuses:    0 Active (0 Ready) / 0 Succeeded / 3 Failed
Pod Template:
  Labels:  batch.kubernetes.io/controller-uid=c349351a-9b35-4fa9-bb78-f42d822bf70b
           batch.kubernetes.io/job-name=job-container-failed
           controller-uid=c349351a-9b35-4fa9-bb78-f42d822bf70b
           job-name=job-container-failed
  Containers:
   busybox-normal:
    Image:      docker.io/busybox
    Port:       <none>
    Host Port:  <none>
    Command:
      sh
      -c
      sleep 5; exit 0
    Environment:  <none>
    Mounts:       <none>
   busybox-abnormal:
    Image:      docker.io/busybox
    Port:       <none>
    Host Port:  <none>
    Command:
      sh
      -c
      sleep 5; exit 1
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Events:
  Type     Reason                Age   From            Message
  ----     ------                ----  ----            -------
  Normal   SuccessfulCreate      64s   job-controller  Created pod: job-container-failed-hd7hp
  Normal   SuccessfulCreate      44s   job-controller  Created pod: job-container-failed-nlc8k
  Normal   SuccessfulCreate      14s   job-controller  Created pod: job-container-failed-kdp4p
  Warning  BackoffLimitExceeded  1s    job-controller  Job has reached the specified backoff limit
```
```
vagrant@master-node:~$ kubectl get pod
NAME                         READY   STATUS   RESTARTS   AGE
job-container-failed-5j4g5   0/2     Error    0          2m28s
job-container-failed-gqrvd   0/2     Error    0          117s
job-container-failed-xm47b   0/2     Error    0          2m49s
                                     ~~~~~
                                     정상종료 먼저되면 Completed가 되고, 비정상종료가 먼저되면 Error가 됨
```

## 잡 컨트롤러를 이용한 소수 계산 ##
### 소수를 만들어서 출력하는 프로그램을 numpy로 작성 ###
- C:\kubernetes\prime_number.py 작성
```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-


import os
import numpy as np
import math
np.set_printoptions(threshold='nan')




# 소수 판정 함수
def is_prime(n):
    if n % 2 == 0 and n > 2:
        return False
    return all(n % i for i in range(3, int(math.sqrt(n)) + 1, 2))




# 배열에 1부터 차례대로 숫자를 배치
nstart = eval(os.environ.get("A_START_NUM"))  
nsize  = eval(os.environ.get("A_SIZE_NUM"))    
nend   = nstart + nsize                        
ay     = np.arange(nstart, nend)              


# 소수 판정 함수를 벡터화
pvec = np.vectorize(is_prime)


# 배열 요소에 적용
primes_tf = pvec(ay)


# 소수만 추출하여 표시
primes = np.extract(primes_tf, ay)
print primes
```
- C:\kubernetes\requirements.txt
```
numpy==1.14.1
```

### Dockerfile 작성 ###
- 
```dokcerfile
FROM python:2
COPY ./requirements.txt /requirements.txt
COPY ./prime_number.py /prime_number.py
RUN pip install --no-cache-dir -r requirements.txt
CMD [ "python", "/prime_number.py" ]
```

### 이미지 빌드 및 도커허브 등록 ###
```
c:\kubernetes>docker image build -t primenumber_generator .

# 테스트
c:\kubernetes>docker container run -e A_START_NUM=1 -e A_SIZE_NUM=20 primenumber_generator
[ 1  2  3  5  7 11 13 17 19]

# 도커허브 등록
c:\kubernetes>docker image tag primenumber_generator tyunme/primenumber_generator:1.0

c:\kubernetes>docker login
Authenticating with existing credentials...
Login Succeeded

c:\kubernetes>docker image push tyunme/primenumber_generator:1.0
```

### 잡 컨트롤러의 매니페스트 파일 작성 ###
- **/home/vagrant/primenumber_job.yaml** 작성
```
apiVersion: batch/v1
kind: Job
metadata:
  name: primenumber-job
spec:
  template:
    spec:
      containers:
      - name: primenumber-generator
        image: docker.io/tyunme/primenumber_generator:1.0
        env:
        - name: A_START_NUM
          value: "2"
        - name: A_SIZE_NUM
          value: "10**6"
      restartPolicy: Never
      imagePullSecrets:
      - name: regcred  
  backoffLimit: 4
```

### 매니페스트 파일 실행 ###
```
vagrant@master-node:~$ kubectl apply -f primenumber_job.yaml
job.batch/primenumber-job created

vagrant@master-node:~$ kubectl get pod,job
NAME                        READY   STATUS      RESTARTS   AGE
pod/primenumber-job-xx648   0/1     Completed   0          16m

NAME                        COMPLETIONS   DURATION   AGE
job.batch/primenumber-job   1/1           70s        16m

vagrant@master-node:~$ kubectl logs pod/primenumber-job-xx648
```

## 메시지 브로커와 잡 컨트롤러를 조합 ##

![image](https://github.com/xodbs1123/Kubernetes/assets/61976898/70f48fee-1068-486a-8a73-09fb3d484784)

### 표준입력으로 계산범위를 전달받아서 소수를 계산하는 파이썬 어플리케이션 제작 ###
- **C:\kubernetes\prime_number_generator.py**
```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-

import sys
import numpy as np
import math
np.set_printoptions(threshold='nan')

# 소수 판정 함수
def is_prime(n):
    if n % 2 == 0 and n > 2:
        return False
    return all(n % i for i in range(3, int(math.sqrt(n)) + 1, 2))


# 소수 생성 함수
def prime_number_generater(nstart, nsize):
    nend = nstart + nsize
    ay = np.arange(nstart, nend)
    # 소수 판정 함수 벡터화
    pvec = np.vectorize(is_prime)
    # 배열 요소에 적용
    primes_t = pvec(ay)
    # 소수만 추출하여 표시
    primes = np.extract(primes_t, ay)
    return primes


if __name__ == '__main__':
    p = sys.stdin.read().split(",")
    print p
    print prime_number_generater(int(p[0]), int(p[1]))
```
- C:\kubernetes\requirements.txt
```
numpy==1.14.1
```
### Dockerfile 작성 ###
- C:\kubernetes\Dockerfile
```dockerfile
FROM ubuntu:16.04
RUN apt-get update && \
    apt-get install -y curl ca-certificates amqp-tools python python-pip
COPY ./requirements.txt /requirements.txt
COPY ./prime_number_generator.py /prime_number_generator.py
RUN pip install --no-cache-dir -r requirements.txt
CMD /usr/bin/amqp-consume --url=$BROKER_URL -q $QUEUE -c 1 /prime_number_generator.py
```
### 이미지 빌드 및 도커 허브 등록 ###
```
# 이미지 빌드
c:\kubernetes>docker build --tag tyunme/prime_number_generator:2.0 .

# 도커허브 등록
c:\kubernetes>docker image push tyunme/prime_number_generator:2.0
```

### 쿠버네티스 클러스터에 RabbitMQ 배포 ###
- **/home/vagrant/rabbitmq.yaml**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: taskque-deploy
spec:
  selector:
    matchLabels:
      app: taskque
  replicas: 1
  template:
    metadata:
      labels:
        app: taskque
    spec:
      containers:
      - image: docker.io/rabbitmq
        name: rabbitmq
        ports:
        - containerPort: 5672
        resources:
          limits:
            cpu: 100m
      imagePullSecrets:
      - name: regcred
---
apiVersion: v1
kind: Service
metadata:
  name: taskque-svc
spec:
  type: NodePort
  ports:
  - port: 5672
    nodePort: 31672
  selector:
    app: taskque
```
```
vagrant@master-node:~$ kubectl apply -f rabbitmq.yaml
deployment.apps/taskque-deploy created
service/taskque-svc created
```

### job-initiator.py 작성 ###
- **/home/vagrant/job-initiator.py**
```python
#!/usr/bin/env python
# -*- coding:utf-8 -*-


import yaml
import pika
from kubernetes import client, config


OBJECT_NAME = "pngen"
qname = 'taskque'


# 메시지 브로커 접속
def create_queue():
    qmgr_cred = pika.PlainCredentials('guest', 'guest')     
    qmgr_host = '10.0.0.10'                                 ## 노드 IP를 설정                          
    qmgr_port = '31672'                                     ## 노드 PORT        
    qmgr_pram = pika.ConnectionParameters(
        host=qmgr_host,
        port=qmgr_port,
        credentials=qmgr_cred)
    conn = pika.BlockingConnection(qmgr_pram)
    chnl = conn.channel()
    chnl.queue_declare(queue=qname)
    return chnl


# 잡 매니페스트 작성
def create_job_manifest(n_job, n_node):
    container = client.V1Container(
        name="pn-generator",
        image="tyunme/prime_number_generator:2.0",
        env=[
            client.V1EnvVar(name="BROKER_URL",
                            value="amqp://guest:guest@taskque-svc:5672"),
            client.V1EnvVar(name="QUEUE", value="taskque")
        ]
        
    )
    template = client.V1PodTemplateSpec(
        spec=client.V1PodSpec(containers=[container],
                              restart_policy="Never"
                              image_pull_secrets=[client.V1LocalObjectReference(name='regcred')]
                              ))
    spec = client.V1JobSpec(
        backoff_limit=4,
        template=template,
        completions=n_job,
        parallelism=n_node)
    job = client.V1Job(
        api_version="batch/v1",
        kind="Job",
        metadata=client.V1ObjectMeta(name=OBJECT_NAME),
        spec=spec)
    return job




if __name__ == '__main__':


    job_parms = [[1, 1000], [1001, 1000], [2001, 1000], [3001, 1000]]
    jobs = len(job_parms)
    nodes = 2


    queue = create_queue()
    for param_n in job_parms:
        param = str(param_n).replace('[', '').replace(']', '')
        queue.basic_publish(exchange='', routing_key=qname, body=param)


    config.load_kube_config()
    client.BatchV1Api().create_namespaced_job(
        body=create_job_manifest(jobs, nodes), namespace="default")
```

### 파이썬 모듈 설치 ###
```
vagrant@master-node:~$ sudo apt-get update
vagrant@master-node:~$ sudo apt-get install -y python3 python3-pip
vagrant@master-node:~$ pip3 install pika
vagrant@master-node:~$ pip3 install kubernetes
```

### job-initiator 실행 및 확인 ###
```
vagrant@master-node:~$ python3 job-initiator.py

vagrant@master-node:~$ kubectl get job,pod
NAME              COMPLETIONS   DURATION   AGE
job.batch/pngen   4/4           39s        98s

NAME                                  READY   STATUS      RESTARTS   AGE
pod/pngen-dn5xq                       0/1     Completed   0          98s
pod/pngen-lmc2w                       0/1     Completed   0          62s
pod/pngen-n295c                       0/1     Completed   0          98s
pod/pngen-zclw2                       0/1     Completed   0          66s
pod/taskque-deploy-77f7fdd8d6-7w9xv   1/1     Running     0          42m

vagrant@master-node:~$ kubectl logs pod/pngen-lmc2w
['3001', ' 1000']
[3001 3011 3019 3023 3037 3041 3049 3061 3067 3079 3083 3089 3109 3119
 3121 3137 3163 3167 3169 3181 3187 3191 3203 3209 3217 3221 3229 3251
 3253 3257 3259 3271 3299 3301 3307 3313 3319 3323 3329 3331 3343 3347
 3359 3361 3371 3373 3389 3391 3407 3413 3433 3449 3457 3461 3463 3467
 3469 3491 3499 3511 3517 3527 3529 3533 3539 3541 3547 3557 3559 3571
 3581 3583 3593 3607 3613 3617 3623 3631 3637 3643 3659 3671 3673 3677
 3691 3697 3701 3709 3719 3727 3733 3739 3761 3767 3769 3779 3793 3797
 3803 3821 3823 3833 3847 3851 3853 3863 3877 3881 3889 3907 3911 3917
 3919 3923 3929 3931 3943 3947 3967 3989]

vagrant@master-node:~$ kubectl logs pod/pngen-zclw2
['1001', ' 1000']
[1009 1013 1019 1021 1031 1033 1039 1049 1051 1061 1063 1069 1087 1091
 1093 1097 1103 1109 1117 1123 1129 1151 1153 1163 1171 1181 1187 1193
 1201 1213 1217 1223 1229 1231 1237 1249 1259 1277 1279 1283 1289 1291
 1297 1301 1303 1307 1319 1321 1327 1361 1367 1373 1381 1399 1409 1423
 1427 1429 1433 1439 1447 1451 1453 1459 1471 1481 1483 1487 1489 1493
 1499 1511 1523 1531 1543 1549 1553 1559 1567 1571 1579 1583 1597 1601
 1607 1609 1613 1619 1621 1627 1637 1657 1663 1667 1669 1693 1697 1699
 1709 1721 1723 1733 1741 1747 1753 1759 1777 1783 1787 1789 1801 1811
 1823 1831 1847 1861 1867 1871 1873 1877 1879 1889 1901 1907 1913 1931
 1933 1949 1951 1973 1979 1987 1993 1997 1999]
```
