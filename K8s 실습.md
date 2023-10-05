## SSH Client ##
https://www.bitvise.com/ssh-client-download

![image](https://github.com/xodbs1123/Kubernetes/assets/61976898/2fe77caf-d52f-43a9-856f-a7f2f524f3c9)

## nginx 컨테이너로 구섣된 파드 생성 ##
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
