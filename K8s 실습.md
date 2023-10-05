## SSH Client ##
https://www.bitvise.com/ssh-client-download

![image](https://github.com/xodbs1123/Kubernetes/assets/61976898/2fe77caf-d52f-43a9-856f-a7f2f524f3c9)

## nginx 컨테이너로 구성된 파드 생성 ##
### CLI ###
```
root@master:/home/vagrant# kubectl run my-nginx-pod-cli --image=nginx:latest --port 80
pod/my-nginx-pod-cli created           ~~~~ 파드 이름 ~~~  ~~~~~~ 이미지 ~~~~~

root@master:/home/vagrant# kubectl get pod
NAME               READY   STATUS    RESTARTS   AGE
my-nginx-pod-cli   1/1     Running   0          46s
```
### YAML ###
```
root@master:/home/vagrant# vi nginx-pod.yaml
```
``` yaml
apiVersion: v1                  # YAML 파일에서 정의한 오브젝트의 API 버전
kind: Pod                       # 리소스의 종류 ( kubectl api-resources 명령의 KIND 항목)
metadata:                       # 라벨, 주석(annotation), 이름과 같은 리소스의 부가 정의
 name: my-nginx-pod
spec:                           # 리소스 생성을 위한 정보
 containers:
 - name: my-nginx-container
   image: nginx:latest
   ports:
   - containerPort: 80
     protocol: TCP
```
- 실행
```
root@master:/home/vagrant# kubectl apply -f nginx-pod.yaml

root@master:/home/vagrant# kubectl get pod
NAME               READY   STATUS    RESTARTS   AGE
my-nginx-pod       1/1     Running   0          25s
my-nginx-pod-cli   1/1     Running   0          11m
```

## kubectl get pod로 표시되는 STATUS ##
https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/

![image](https://github.com/xodbs1123/Kubernetes/assets/61976898/484bca09-4bd0-4b00-a2d6-0f49234aeaaf)


## 생성된 리소스의 자세한 정보를 확인 ##
```
root@master:/home/vagrant# kubectl describe pods my-nginx-pod

Name:             my-nginx-pod
Namespace:        default
Priority:         0
Service Account:  default
Node:             worker1/192.168.10.110
Start Time:       Thu, 05 Oct 2023 09:35:41 +0900
Labels:           <none>
Annotations:      cni.projectcalico.org/containerID: aff30174fe3b9e52006b41fcc0954f315ae44a5348b5903e958f0f0cba85cf69
                  cni.projectcalico.org/podIP: 10.233.105.131/32
                  cni.projectcalico.org/podIPs: 10.233.105.131/32
Status:           Running
IP:               10.233.105.131          <= 파드 IP (클러스터 내부에서만 접근 가능)
IPs:                                         쿠버네티스 외부 또는 내부에서 파드에 접근하려면 서비스 생성해야 함
  IP:  10.233.105.131
Containers:
  my-nginx-container:
    Container ID:   containerd://73054fc543af5d4cf8cb05ba3220161488030d691f77f83f8b65f6527a30c309
    Image:          nginx:latest
    Image ID:       docker.io/library/nginx@sha256:32da30332506740a2f7c34d5dc70467b7f14ec67d912703568daff790ab3f755
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Thu, 05 Oct 2023 09:35:52 +0900
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-rfqtp (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-rfqtp:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  10m   default-scheduler  Successfully assigned default/my-nginx-pod to worker1
  Normal  Pulling    11m   kubelet            Pulling image "nginx:latest"
  Normal  Pulled     10m   kubelet            Successfully pulled image "nginx:latest" in 10.263s (10.263s including waiting)
  Normal  Created    10m   kubelet            Created container my-nginx-container
  Normal  Started    10m   kubelet            Started container my-nginx-container
```

## nginx 동작 확인 ##
### 클러스터 내부에서 nginx 파드 동작을 확인 (클러스터 내의 노드에서 실행) ###
- 같은 클러스터 내부에서 생성된 파드는 같은 네트워크로 묶임
- 따라서 노드에서 파드에 바로 접근이 가능한 것
```
root@master:/home/vagrant# curl 10.233.105.131    <= 클러스터를 구성하고 있는 master 노드에서 파드 IP로 접근

<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

### 클러스터 내부에 테스트 용도의 파드를 임시로 생성해서 nginx 파드의 동작 확인 (클러스터 내의 노드에 접속이 어려운 경우) ###
```
root@master:/home/vagrant# kubectl run -it --rm debug --image=busybox --restart=Never sh
If you don't see a command prompt, try pressing enter.
/ #
/ # wget -q -O - http://10.233.125.2
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
/ #
```

### 다른 터미널에서 파드 상태를 확인 ###
- 노드의 경계를 넘어서 출력된 IP를 기반으로 피드간 통신이 가능
- 테스트용 파드를 빠져나오면 해당 파드는 삭제됨
```
root@master:/home/vagrant# kubectl get pod -o wide
NAME               READY   STATUS    RESTARTS   AGE     IP               NODE      NOMINATED NODE   READINESS GATES
debug              1/1     Running   0          3m42s   10.233.105.132   worker1   <none>           <none>
my-nginx-pod       1/1     Running   0          36m     10.233.105.131   worker1   <none>           <none>
my-nginx-pod-cli   1/1     Running   0          47m     10.233.125.2     worker2   <none>           <none>
```

### kubectl exec 명령으로 파드의 컨테이너에 명령어 전달 ###
```
root@master:/home/vagrant# kubectl exec -it my-nginx-pod -- /bin/bash
                                            ~~ 파드이름 ~~  ~~파드 내의 컨테이너에서 실행할 명령어 ~~
root@my-nginx-pod:/# ls /etc/nginx/
conf.d  fastcgi_params  mime.types  modules  nginx.conf  scgi_params  uwsgi_params
```

### kubectl logs 명령으로 파드의 로그를 확인 가능 ###
```
root@master:/home/vagrant# kubectl logs my-nginx-pod    <= 파드의 표준 출력 로그를 확인
/docker-entrypoint.sh: /docker-entrypoint.d/ is not empty, will attempt to perform configuration
/docker-entrypoint.sh: Looking for shell scripts in /docker-entrypoint.d/
/docker-entrypoint.sh: Launching /docker-entrypoint.d/10-listen-on-ipv6-by-default.sh
10-listen-on-ipv6-by-default.sh: info: Getting the checksum of /etc/nginx/conf.d/default.conf
10-listen-on-ipv6-by-default.sh: info: Enabled listen on IPv6 in /etc/nginx/conf.d/default.conf
/docker-entrypoint.sh: Sourcing /docker-entrypoint.d/15-local-resolvers.envsh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/20-envsubst-on-templates.sh
/docker-entrypoint.sh: Launching /docker-entrypoint.d/30-tune-worker-processes.sh
/docker-entrypoint.sh: Configuration complete; ready for start up
2023/10/05 00:35:52 [notice] 1#1: using the "epoll" event method
2023/10/05 00:35:52 [notice] 1#1: nginx/1.25.2
2023/10/05 00:35:52 [notice] 1#1: built by gcc 12.2.0 (Debian 12.2.0-14)
2023/10/05 00:35:52 [notice] 1#1: OS: Linux 5.15.0-84-generic
2023/10/05 00:35:52 [notice] 1#1: getrlimit(RLIMIT_NOFILE): 65535:65535
2023/10/05 00:35:52 [notice] 1#1: start worker processes
2023/10/05 00:35:52 [notice] 1#1: start worker process 28
2023/10/05 00:35:52 [notice] 1#1: start worker process 29
10.233.97.128 - - [05/Oct/2023:00:55:01 +0000] "GET / HTTP/1.1" 200 615 "-" "curl/7.81.0" "-"
10.233.105.132 - - [05/Oct/2023:01:12:54 +0000] "GET / HTTP/1.1" 200 615 "-" "Wget" "-"
```

### kubectl delete -f 명령으로 쿠버네티스 오브젝트 삭제 ###
```
root@master:/home/vagrant# kubectl delete -f nginx-pod.yaml   <= nginx-pod.yaml 파일에 정의된 오브젝트를 삭제
pod "my-nginx-pod" deleted                                       kubectl delete pod my-nginx-pod 와 동일

```

## 쿠버네티스가 파드를 사용하는 이유 ##
- 여러 리눅스 네임스페이스를 공유하는 **여러 컨테이너들을 추상화된 집합으로 사용**하기 위함
- /home/vagrant/nginx-pod-with-ubuntu.yaml 생성
```yaml
apiVersion: v1
kind: Pod
metadata:
 name: my-nginx-pod
spec:
 containers:
 - name: my-nginx-container
   image: nginx:latest
   ports:
   - containerPort: 80
     protocol: TCP
  - name: ubuntu-sidecar-container
    image: alicek106/rr-test:curl
    command: ['tail']        <= 컨테이너가 종료되지 않도록 tail -f /dev/null 실행
    args: ['-f', '/dev/null']
```
```
root@master:/home/vagrant# kubectl apply -f nginx-pod-with-ubuntu.yaml
pod/my-nginx-pod created

root@master:/home/vagrant# kubectl get pod
NAME               READY   STATUS    RESTARTS        AGE
my-nginx-pod       2/2     Running   0               5s
my-nginx-pod-cli   1/1     Running   1 (5m22s ago)   115m
```
```
root@master:/home/vagrant# kubectl exec -it my-nginx-pod -c ubuntu-sidecar-container -- /bin/bash
                                                            ~~ 파드 내 컨테이너 이름 ~~
root@my-nginx-pod:/# curl localhost
<!DOCTYPE html>
<html>				                      	⇒ 우분투 컨테이너 내부에서 localhost로의 요청에 대해 응답이 도착하는 것을 확인
<head>					                         우분투 컨테이너의 localhost에서 nginx 서버로 접근이 가능 
<title>Welcome to nginx!</title> 	       파드 내부의 컨테이너들은 네트워크와 같은 리눅스 네임스페이스를 공유함
<style>
html { color-scheme: light dark; }
body { width: 35em; margin: 0 auto;
font-family: Tahoma, Verdana, Arial, sans-serif; }
</style>
</head>
<body>
<h1>Welcome to nginx!</h1>
<p>If you see this page, the nginx web server is successfully installed and
working. Further configuration is required.</p>

<p>For online documentation and support please refer to
<a href="http://nginx.org/">nginx.org</a>.<br/>
Commercial support is available at
<a href="http://nginx.com/">nginx.com</a>.</p>

<p><em>Thank you for using nginx.</em></p>
</body>
</html>
```

## POD : 사이드카 패턴 ##
![image](https://github.com/xodbs1123/Kubernetes/assets/61976898/71a08885-d5ef-4fab-aece-b5d1d16ad23d)

### 하나의 파드에 웹 서버 컨테이너와 깃허브에서 최신 컨텐츠를 다운받는 컨테이너 조합 ###
- 깃 허브에서 정기적으로 컨텐츠를 다운받는 쉘 스크립트 작성(C:\kubernetes > mkdir sidecar)
```
C:\kubernetes> mkdir sidecar

C:\kubernetes> cd sidecar

C:\kubernetes\sidecar> code contents-cloner
```
```
#!/bin/bash

if [ -z $CONTENTS_SOURCE_URL]; then
    exit 1
fi

gjt clone $CONTENTS_SOURCE_URL /data

cd /data

while true
do 
    date
    sleep 60
    git pull

done
```

### Dockerfile 작성 ###
```
FROM        ubuntu:latest
RUN         apt-get update && apt-get install -y git
COPY        .\contents-cloner  /contents-cloner
RUN         chmod a+x /contents-cloner
WORKDIR     /
CMD         ["/contents-cloner"]
```

### 이미지 빌드 및 레지스트리에 등록 ###
```
c:\kubernetes\sidecar>docker image build --tag tyunme/contents-cloner:1.0 .

c:\kubernetes\sidecar>docker image ls
REPOSITORY                                                TAG                                                                          IMAGE ID       CREATED         SIZE
tyunme/contents-cloner                                    1.0                                                                          5bee7b44e413   6 seconds ago   198MB

c:\kubernetes\sidecar>docker login
Authenticating with existing credentials...
Login Succeeded
```
```
c:\kubernetes\sidecar>docker image push tyunme/contents-cloner:1.0
```

### 깃 허브 레포지토리 주소를 확인 ###
https://github.com/xodbs1123/sidecar.git

### 사이드카 패턴의 파드를 생성하는 매니페스트 작성 ###
- /home/vagrant/venv/webserver.yaml 작성
```yaml
apiVersion: v1
kind: Pod
metadata:
 name: webserver
spec:
 containers:
 - name: nginx
   image: nginx
   volumeMounts:
   - mountPath: /usr/share/nginx/html
     name: contents-vol
     readOnly: true
  - name: cloner
    image: tyunme/contents-cloner:1.0
    env:
     - name: CONTENTS_SOURCE_URL
       value: "https://github.com/xodbs1123/sidecar.git"
    volumeMounts:
    - mountPath: /data
      name: contens-vol
volumes:
 - name: contents-vol
   emptyDir: {}
```

### yaml 파일 실행 ###
```
root@master:/home/vagrant# kubectl apply -f webserver.yaml
pod/webserver created

root@master:/home/vagrant# kubectl get pod -o wide
NAME               READY   STATUS    RESTARTS      AGE    IP               NODE      NOMINATED NODE   READINESS GATES
webserver          2/2     Running   0             56s    10.233.105.134   worker1   <none>           <none>
```

### Docker Desktop Kubernetes를 활용해서 테스트 ###
- C:\kubernetes\webserver.yaml
- 명령을 입력할 클러스터를 선택
```
c:\kubernetes>kubectl config get-contexts
CURRENT   NAME             CLUSTER          AUTHINFO         NAMESPACE
          docker-desktop   docker-desktop   docker-desktop
*         minikube         minikube         minikube         default

c:\kubernetes>kubectl config use-context docker-desktop
Switched to context "docker-desktop".

c:\kubernetes>kubectl config get-contexts
CURRENT   NAME             CLUSTER          AUTHINFO         NAMESPACE
*         docker-desktop   docker-desktop   docker-desktop
          minikube         minikube         minikube         default
```
- yaml 파일 실행
```
c:\kubernetes>kubectl apply -f webserver.yaml
pod/webserver created
```
- pod 확인
```
c:\kubernetes>kubectl get pod -o wide
NAME                         READY   STATUS    RESTARTS   AGE   IP          NODE             NOMINATED NODE   READINESS GATES
hello-k8s-75797f94b4-crp5n   1/1     Running   0          28h   10.1.0.13   docker-desktop   <none>           <none>
webserver                    2/2     Running   0          65s   10.1.0.14   docker-desktop   <none>           <none>
```

### 대화형(=임시) 파드를 가동해서 초기 컨텐츠를 실행 ###
```
c:\kubernetes>kubectl run busybox --image=busybox --restart=Never --rm -it sh
If you don't see a command prompt, try pressing enter.
/ #
```
```
/ # wget -q -O - http://10.1.0.14
<!DOCTYPE html>
<html>
<head>
    <title>Kim Tae Yun Docker Project</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f4f4f4;
            margin: 0;
            padding: 0;
        }

        h1 {
            text-align: center;
            color: #333;
        }

        .container {
            max-width: 400px;
            margin: 0 auto;
            text-align: center;
            padding: 20px;
            background-color: #fff;
            border-radius: 5px;
            box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
        }

        button {
            background-color: #007BFF;
            color: #fff;
            border: none;
            padding: 10px 20px;
            border-radius: 5px;
            cursor: pointer;
        }

        button:hover {
            background-color: #0056b3;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Kim Tae Yun Docker Project</h1>
        <!-- 버튼 추가 -->
        <button onclick="goToDockerPage()">Go to Docker Page</button>
    </div>

    <script>
        // JavaScript 함수를 사용하여 페이지 이동 처리
        function goToDockerPage() {
            window.location.href = "docker.html"; // 이동할 페이지의 URL을 여기에 입력
        }
    </script>
</body>
</html>
```

### 깃 허브 리포지터리의 컨텐츠를 수정하고 커밋 ###
- wget을 통해 수정된 내용 확인 ( Go to Docker page -> Go to Taeyun Docker page )
```
/ # wget -q -O - http://10.1.0.14
<!DOCTYPE html>
<html>
<head>
    <title>Kim Tae Yun Docker Project</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background-color: #f4f4f4;
            margin: 0;
            padding: 0;
        }

        h1 {
            text-align: center;
            color: #333;
        }

        .container {
            max-width: 400px;
            margin: 0 auto;
            text-align: center;
            padding: 20px;
            background-color: #fff;
            border-radius: 5px;
            box-shadow: 0 0 10px rgba(0, 0, 0, 0.1);
        }

        button {
            background-color: #007BFF;
            color: #fff;
            border: none;
            padding: 10px 20px;
            border-radius: 5px;
            cursor: pointer;
        }

        button:hover {
            background-color: #0056b3;
        }
    </style>
</head>
<body>
    <div class="container">
        <h1>Kim Tae Yun Docker Project</h1>
        <!-- 버튼 추가 -->
        <button onclick="goToDockerPage()">Go to Taeyun Docker Page</button>
    </div>

    <script>
        // JavaScript 함수를 사용하여 페이지 이동 처리
        function goToDockerPage() {
            window.location.href = "docker.html"; // 이동할 페이지의 URL을 여기에 입력
        }
    </script>
</body>
</html>
```

### 파드 삭제 ###
```
c:\kubernetes>kubectl delete -f webserver.yaml
pod "webserver" deleted
```

## POD : 초기화 컨테이너 ##
https://kubernetes.io/docs/concepts/workloads/pods/init-containers/

- 파드의 앱 컨테이너들이 실행되기 전에 실행되는 특수한 컨테이너
- 앱 이미지에는 없는 셋업을 위한 유틸리티 또는 설정 스크립트 등을 포함할 수 있음
- 일반적인 컨테이너와 차이점
  - 앱 컨테이너의 리소스 상한, 볼륨, 보안 셋팅을 포함한 모든 필드와 기능을 지원
  - 파드가 준비 상태가 되기 전에 완료를 목표로 살행되어야 하므로, lifecycle, livenessProte, readinessProbe, startupProbe를 지원하지 않음

### C:\kubernetes\init-pod.yaml 생성 및 실행 ###
- 스토리지를 마운트할 때 스토리지 안에 새로운 디렉터리를 만들고, 소유자를 설정하는 초기화 기능을 수행하는 컨테이너를 포함
```
kind: Pod
metadata:
  name: init-sample
spec:
  containers:
  - name: main
    image: ubuntu
    command: ["/bin/bash"]
    args: ["-c", "tail -f /dev/null"]
    volumeMounts:
    - mountPath: /docs
      name: data-vol
      readOnly: false
  initContainers:
  - name: init
    image: alpine
    command: ["/bin/sh"]
    args: ["-c", "mkdir /mnt/html; chown 33:33 /mnt/html"]
    volumeMounts:
    - mountPath: /mnt
      name: data-vol
      readOnly: false
  volumes:
  - name: data-vol
    emptyDir: {}

```
```
c:\kubernetes> kubectl apply -f init-pod.yaml
pod/init-sample created

c:\kubernetes> kubectl get pods
NAME                         READY   STATUS     RESTARTS   AGE
init-sample                  0/1     Init:0/1   0          6s

c:\kubernetes> kubectl get pods
NAME                         READY   STATUS            RESTARTS   AGE
init-sample                  0/1     PodInitializing   0          11s

c:\kubernetes> kubectl get pods
NAME                         READY   STATUS    RESTARTS   AGE
init-sample                  1/1     Running   0          14s
```

```
c:\kubernetes> kubectl exec -it init-sample -- sh			⇐ 컨테이너를 지정하지 않은 경우, default로 접속
Defaulted container "main" out of: main, init (init)
# exit

c:\kubernetes> kubectl exec -it init-sample -c main -- sh		⇐ 컨테이너를 지정 
# exit

c:\kubernetes> kubectl exec -it init-sample -c init -- sh		⇐ init 컨테이너로는 접속이 불가
error: unable to upgrade connection: container not found ("init")

c:\kubernetes> kubectl exec -it init-sample -c main -- sh
```
```
c:\kubernetes> kubectl exec -it init-sample -c main -- sh
# 
# ls -al /docs/
total 12
drwxrwxrwx 3 root     root     4096 Oct  5 05:49 .
drwxr-xr-x 1 root     root     4096 Oct  5 05:49 ..
drwxr-xr-x 2 www-data www-data 4096 Oct  5 05:49 html		⇐ 디렉터리 생성 및 소유자와 그룹 변경을 확인
```

### pod 삭제 ###
```
c:\kubernetes>kubectl delete -f init-pod.yaml
pod "init-sample" deleted
```

## Pod의 헬스 체크 기능 ##
- 파드의 컨테이너에는 애플리케이션이 정상적으로 기동하고 있는지 확인하는 기능(=헬스 체크 기능)을 설정할 수 있으며, 이상이 감지되면 컨테이너를 강제 종료하고 재시작할 수 있음
- kubelet가 컨테이너의 헬스 체크를 담당
- kubelet의 헬스 체크는 활성 프로브(Liveness Probe)와 준비 상태 프로브(Readiness Probe)를 사용하여 실행 중인 파드의 컨테이너를 검사
### 활성 프로브(Liveness Probe) ###
  - 컨테이너의 애플리케이션이 정상적으로 실행 중인 것을 검사
  - 검사에 실패하면 파드 상의 컨테이너를 강제로 종료하고 재시작
  - 이 기능을 사용하기 위해서는 매니페스트에 명시적으로 설정해야 함
### 준비 상태 프로브(Readiness Probe) ###
  - 컨테이너의 애플리케이션이 요청을 받을 준비가 되었는지를 검사
  - 검사에 실패하면 서비스에 의한 요청 트래픽 전송을 중지
  - 파드가 기동하고 나서 준비가 될 때까지 요청이 전송되지 않기 위해 사용
  - 이 기능을 사용하기 위해서는 매니패스트에 명시적으로 설정해야함
### 프로브 대응 핸들러 ###
- exec
  - 컨테이너 내 커맨드를 실행
  - exit 코드 0으로 종료하면 진단 결과는 성공으로 간주, 그 외는 실패로 간주
- tcpSocket
  - 지정한 TCP 포트 번호로 연결할 수 있으면 성공으로 간주
- httpGet
  - 지정한 포트와 경로로 HTTP GET 요청을 정기적으로 실행
  - HTTP 상태 코드가 200 이상 499 미만이면 성공으로 간주하고, 그 외는 실패로 간주
  - 지정 포트가 열려 있지 않은 경우도 실패로 간주
## 실제 컨테이너를 기동하여 프로브 동작 확인 ##
- nodejs 설치
https://nodejs.org/ko/download

```
c:\kubernetes>mkdir webapp

c:\kubernetes>cd webapp

c:\kubernetes\webapp>
```
```
c:\kubernetes\webapp>npm init -y
Wrote to c:\kubernetes\webapp\package.json:

{
  "name": "webapp",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```
- C:\kubernetes\webapp\index.js 작성
```js
const express = require('express')
const app = express()
var start = Date.now()

app.get('/healthz', function(request, response) {
    var msec = Date.now() - start
    var code = 200
    if (msec > 40000) {
        code = 500
    }
    console.log('GET /healthz ' + code)
    response.status(code).send('OK')
});

app.get('/ready', function(request, response) {
    var msec = Date.now() - start
    var code = 500
    if (msec > 20000) {
        code = 200
    }
    console.log('GET /ready ' + code)
    response.status(code).send('OK')
});

app.get('/', function(request, response) {
    console.log('GET /')
    response.send('Hello from Node.js')
});

app.listen(3000);
```
- 애플리케이션 실행에 필요한 모듈 추가
```
c:\kubernetes\webapp>npm install express -y
```
- package.json에 의존 모듈이 추가딘 것을 확인
```
c:\kubernetes\webapp>type package.json
{
  "name": "webapp",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "express": "^4.18.2"
  }
}
```
- 로컬호스트 3000포트로 접근
```
c:\kubernetes\webapp>node index.js
```
```
c:\kubernetes\webapp>node index.js
GET /
GET /ready 200
GET /healthz 500
```
- C:\kubernetes\webapp\Dockerfile 작성
```
FROM        alpine:latest
RUN         apk update && apk add --no-cache nodejs npm
WORKDIR     /
ADD         ./package.json /
RUN         npm install
ADD         ./index.js  /
CMD         node /index.js
```
- 이미지 빌드 및 등록
```
c:\kubernetes\webapp>docker image build --tag tyunme/webapp:1.0 .

c:\kubernetes\webapp>docker image push tyunme/webapp:1.0
```

- 헬스 체크 설정 매니페스트 작성 (C:\kubernetes\webapp\webapp-pod.yaml)
```
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  containers:
  - name: webapp
    image: myanjini/webapp:1.0
    livenessProbe:                       # 어플리케이션이 살아 있는지 확인
      httpGet:
        path: /healthz
        port: 3000
      initialDelaySeconds: 3              # 검사 개시 대기 시간
      periodSeconds: 5                    # 검사 간격(주기)
    readinessProbe:                       # 어플리케이션이 요청을 처리할 준비되었는지 확인
      httpGet:
        path: /ready
        port: 3000
      initialDelaySeconds: 15
      periodSeconds: 6

```
- 파드를 배포하고 헬스 체크가 실행되어 READY 상태가 되는 과정 확인
```
c:\kubernetes\webapp>kubectl apply -f webapp-pod.yaml
pod/webapp created


c:\kubernetes\webapp> kubectl get pod
NAME                         READY   STATUS    RESTARTS   AGE
hello-k8s-75797f94b4-v8gvw   1/1     Running   0          30h
init-sample                  1/1     Running   0          88m
webapp                       0/1     Running   0          18s
                             ~
                             readinessProbe가 성공하지 못했기 때문에 READY 상태가 아닌 것으로 판단

c:\kubernetes\webapp>kubectl get pod
NAME                         READY   STATUS    RESTARTS   AGE
hello-k8s-75797f94b4-v8gvw   1/1     Running   0          30h
init-sample                  1/1     Running   0          88m
webapp                       1/1     Running   0          26s
```
```
c:\kubernetes\webapp> kubectl logs webapp			⇐ 컨테이너 로그에 기록된 헬스 체크 결과를 출력
GET /healthz 200						⇐ livenessProbe에 의해 액세스 
GET /healthz 200
GET /healthz 200
GET /ready 500							⇐ 20초가 지나기 전에 /ready가 500을 응답
GET /healthz 200
GET /ready 200
GET /healthz 200
GET /ready 200
GET /healthz 200
GET /healthz 200
GET /ready 200
GET /healthz 200
GET /ready 200
GET /healthz 500
GET /ready 200
GET /healthz 500
GET /ready 200
GET /healthz 500

c:\kubernetes\webapp> kubectl describe pods webapp
Name:             webapp
Namespace:        default
Priority:         0
Service Account:  default
Node:             docker-desktop/192.168.65.4
Start Time:       Thu, 05 Oct 2023 16:28:11 +0900
Labels:           <none>
Annotations:      <none>
Status:           Running
IP:               10.1.0.17
IPs:
  IP:  10.1.0.17
Containers:
  webapp:
    Container ID:   docker://74759edf46ca34ac57ef05f0b047a12018400e6ae8de601e2f24eb47890fdc08
    Image:          myanjini/webapp:1.0
    Image ID:       docker-pullable://myanjini/webapp@sha256:212c0445a2e31d42638802cd8d34ecc46c3f5f146ff4970df50f6099506677a2
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Thu, 05 Oct 2023 16:29:37 +0900
    Last State:     Terminated
      Reason:       Error
      Exit Code:    137
      Started:      Thu, 05 Oct 2023 16:28:12 +0900
      Finished:     Thu, 05 Oct 2023 16:29:37 +0900
    Ready:          True
    Restart Count:  1
    Liveness:       http-get http://:3000/healthz delay=3s timeout=1s period=5s #success=1 #failure=3
    Readiness:      http-get http://:3000/ready delay=15s timeout=1s period=6s #success=1 #failure=3
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-vrwx7 (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-vrwx7:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason     Age                  From               Message
  ----     ------     ----                 ----               -------
  Normal   Scheduled  2m40s                default-scheduler  Successfully assigned default/webapp to docker-desktop
  Normal   Pulled     75s (x2 over 2m40s)  kubelet            Container image "myanjini/webapp:1.0" already present on machine
  Normal   Created    75s (x2 over 2m40s)  kubelet            Created container webapp
  Normal   Started    75s (x2 over 2m40s)  kubelet            Started container webapp
  Warning  Unhealthy  58s (x2 over 2m22s)  kubelet            Readiness probe failed: HTTP probe failed with statuscode: 500
  Warning  Unhealthy  20s (x6 over 115s)   kubelet            Liveness probe failed: HTTP probe failed with statuscode: 500
  Normal   Killing    20s (x2 over 105s)   kubelet            Container webapp failed liveness probe, will be restarted
```
