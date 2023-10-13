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
