## 쿠버네티스(K8s, Kubernetes) ##
- 컨테이너 기반의 애플리케이션을 개발하고 배포할 수 있도록 설계된 오픈 소스 플랫폼
- 컨테이너 오케스트레이션 도구의 사실상 표준
- 쿠버네티스의 필요성과 활용
https://kubernetes.io/ko/docs/concepts/overview/#why-you-need-kubernetes-and-what-can-it-do

- 쿠버네티스 컴포넌트 
https://kubernetes.io/ko/docs/concepts/overview/components/

![image](https://github.com/xodbs1123/Kubernetes/assets/61976898/03f906dc-914c-409e-bf0b-403cccb0ce0c)

![image](https://github.com/xodbs1123/Kubernetes/assets/61976898/2db3af24-4f57-4c6b-a2f3-33240355c8d8)

## 쿠버네티스 설치 환경 ##
### 개발 용도의 쿠버네티스 설치(단일 노드) ###
- minikube
- Docker for Mac/Windows에 내장된 커부네티스

### 서비스 테스트 또는 운영 용도의 쿠버네티스 설치(다중 노드) ###
- kops
- kubespray
- kubeadm
- EKS, AKS, GKE 등의 매니지드 서비스

## Docker Desktop에서 Kubernetes 사용 ##
### 명령어 확인 ###
```
C:\Users\User>kubectl get nodes

NAME             STATUS   ROLES           AGE   VERSION
docker-desktop   Ready    control-plane   17m   v1.27.2
```
```
C:\Users\User>docker container ls

CONTAINER ID   IMAGE                       COMMAND                   CREATED          STATUS          PORTS     NAMES
830a67941cb8   556098075b3d                "/kube-vpnkit-forwar…"   18 minutes ago   Up 18 minutes             k8s_vpnkit-controller_vpnkit-controller_kube-system_70efb662-8b41-453a-80e6-536d4e9ba7ad_0
c3ebc617515b   99f89471f470                "/storage-provisione…"   18 minutes ago   Up 18 minutes             k8s_storage-provisioner_storage-provisioner_kube-system_b1ffe738-873e-4322-829f-b618e81c1f0c_0
974ba74113d8   registry.k8s.io/pause:3.9   "/pause"                  18 minutes ago   Up 18 minutes             k8s_POD_vpnkit-controller_kube-system_70efb662-8b41-453a-80e6-536d4e9ba7ad_0
25472c797a8b   registry.k8s.io/pause:3.9   "/pause"                  18 minutes ago   Up 18 minutes             k8s_POD_storage-provisioner_kube-system_b1ffe738-873e-4322-829f-b618e81c1f0c_0
5b89f7b76687   ead0a4a53df8                "/coredns -conf /etc…"   18 minutes ago   Up 18 minutes             k8s_coredns_coredns-5d78c9869d-l677b_kube-system_f2589dff-7566-45bd-80cb-46cb485d8d8c_0
9d7638218676   ead0a4a53df8                "/coredns -conf /etc…"   18 minutes ago   Up 18 minutes             k8s_coredns_coredns-5d78c9869d-bfg57_kube-system_4e628e20-6778-4a8a-a7b6-62df17d2bee2_0
c2b00ef45db0   registry.k8s.io/pause:3.9   "/pause"                  18 minutes ago   Up 18 minutes             k8s_POD_coredns-5d78c9869d-bfg57_kube-system_4e628e20-6778-4a8a-a7b6-62df17d2bee2_0
d85c69e909ad   registry.k8s.io/pause:3.9   "/pause"                  18 minutes ago   Up 18 minutes             k8s_POD_coredns-5d78c9869d-l677b_kube-system_f2589dff-7566-45bd-80cb-46cb485d8d8c_0
cc0c34b27e63   b8aa50768fd6                "/usr/local/bin/kube…"   18 minutes ago   Up 18 minutes             k8s_kube-proxy_kube-proxy-rlxtl_kube-system_80f805a6-1816-4d2b-b22d-968eb008b227_0
d30d2ae77084   registry.k8s.io/pause:3.9   "/pause"                  18 minutes ago   Up 18 minutes             k8s_POD_kube-proxy-rlxtl_kube-system_80f805a6-1816-4d2b-b22d-968eb008b227_0
38459f9a991d   ac2b7465ebba                "kube-controller-man…"   19 minutes ago   Up 19 minutes             k8s_kube-controller-manager_kube-controller-manager-docker-desktop_kube-system_6d2a77df9cc3ca29a2153e8e119160ab_0
23b35ff251fb   89e70da428d2                "kube-scheduler --au…"   19 minutes ago   Up 19 minutes             k8s_kube-scheduler_kube-scheduler-docker-desktop_kube-system_458a31e42f7d01ae485acb25e3254451_0
0c3eef02f337   86b6af7dd652                "etcd --advertise-cl…"   19 minutes ago   Up 19 minutes             k8s_etcd_etcd-docker-desktop_kube-system_a652e9966ea47b54b0275eed498b1cab_0
3403bae352a0   c5b13e4f7806                "kube-apiserver --ad…"   19 minutes ago   Up 19 minutes             k8s_kube-apiserver_kube-apiserver-docker-desktop_kube-system_2c5ac5deb80203a34d3d0ae47868dbee_0
5301d33aa8b3   registry.k8s.io/pause:3.9   "/pause"                  19 minutes ago   Up 19 minutes             k8s_POD_kube-controller-manager-docker-desktop_kube-system_6d2a77df9cc3ca29a2153e8e119160ab_0
42be46899193   registry.k8s.io/pause:3.9   "/pause"                  19 minutes ago   Up 19 minutes             k8s_POD_kube-apiserver-docker-desktop_kube-system_2c5ac5deb80203a34d3d0ae47868dbee_0
a81a2080d64f   registry.k8s.io/pause:3.9   "/pause"                  19 minutes ago   Up 19 minutes             k8s_POD_kube-scheduler-docker-desktop_kube-system_458a31e42f7d01ae485acb25e3254451_0
c744e252f9cf   registry.k8s.io/pause:3.9   "/pause"                  19 minutes ago   Up 19 minutes             k8s_POD_etcd-docker-desktop_kube-system_a652e9966ea47b54b0275eed498b1cab_0
```
### 애플리케이션 배포 ###
```
C:\Users\User>kubectl create deployment hello-k8s --image=k8s.gcr.io/echoserver:1.4
deployment.apps/hello-k8s created
```
### 애플리케이션 확인 ###
```
C:\Users\User>kubectl get deployment
NAME        READY   UP-TO-DATE   AVAILABLE   AGE
hello-k8s   1/1     1            1           13s
```
```
C:\Users\User>kubectl get deployment -o wide
NAME        READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                      SELECTOR
hello-k8s   1/1     1            1           56s   echoserver   k8s.gcr.io/echoserver:1.4   app=hello-k8s
```
```
C:\Users\User>kubectl get pod,replicaset,deployment -o wide
NAME                             READY   STATUS    RESTARTS   AGE   IP         NODE             NOMINATED NODE   READINESS GATES
pod/hello-k8s-75797f94b4-crp5n   1/1     Running   0          91s   10.1.0.8   docker-desktop   <none>           <none>

NAME                                   DESIRED   CURRENT   READY   AGE   CONTAINERS   IMAGES                      SELECTOR
replicaset.apps/hello-k8s-75797f94b4   1         1         1       91s   echoserver   k8s.gcr.io/echoserver:1.4   app=hello-k8s,pod-template-hash=75797f94b4

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES                      SELECTOR
deployment.apps/hello-k8s   1/1     1            1           91s   echoserver   k8s.gcr.io/echoserver:1.4   app=hello-k8s
```
```
C:\Users\User>kubectl get all
NAME                             READY   STATUS    RESTARTS   AGE
pod/hello-k8s-75797f94b4-crp5n   1/1     Running   0          3m38s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   28m

NAME                        READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/hello-k8s   1/1     1            1           3m38s

NAME                                   DESIRED   CURRENT   READY   AGE
replicaset.apps/hello-k8s-75797f94b4   1         1         1       3m38s
```
### 애플리케이션 삭제 ###
```
C:\Users\User>kubectl delete deployment hello-k8s
deployment.apps "hello-k8s" deleted
```

### 서비스 확인 ###
```
C:\Users\User>kubectl get service -o wide
NAME         TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE   SELECTOR
kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   27m   <none>
```

### NodePort 생성 ###
```
C:\Users\User>kubectl expose deployment hello-k8s --type=NodePort --port=8080
service/hello-k8s exposed
```
```
C:\Users\User>kubectl get service -o wide
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE   SELECTOR
hello-k8s    NodePort    10.103.208.92   <none>        8080:30963/TCP   17s   app=hello-k8s
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP          33m   <none>
```
### Port-forward ###
```
C:\Users\User>kubectl port-forward service/hello-k8s 9090:8080
Forwarding from 127.0.0.1:9090 -> 8080
Forwarding from [::1]:9090 -> 8080
```
![image](https://github.com/xodbs1123/Kubernetes/assets/61976898/7d7be733-2a98-4cac-81f6-39599a36d713)

## minikube를 활용한 단일 노드 쿠버네티스 클러스터 구성 ##
### 설치 방법 ###
https://minikube.sigs.k8s.io/docs/start/

![image](https://github.com/xodbs1123/Kubernetes/assets/61976898/be59d6f4-31c9-4caa-9dbf-3dbf7d13c8aa)

- 1번 2번 과정 수행 후 별도의 명렁 프롬프트 또는 파워쉘 실행

### 클러스터 시작 ###
```
PS C:\Users\User> minikube start
```

### 노드 확인 ###
```
PS C:\Users\User> kubectl get nodes
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   22s   v1.27.4
```

### Pods 확인 ###
```
PS C:\Users\User> kubectl get pods -A
NAMESPACE     NAME                               READY   STATUS    RESTARTS      AGE
kube-system   coredns-5d78c9869d-m69vs           1/1     Running   0             61s
kube-system   etcd-minikube                      1/1     Running   0             73s
kube-system   kube-apiserver-minikube            1/1     Running   0             76s
kube-system   kube-controller-manager-minikube   1/1     Running   0             73s
kube-system   kube-proxy-87llf                   1/1     Running   0             61s
kube-system   kube-scheduler-minikube            1/1     Running   0             75s
kube-system   storage-provisioner                1/1     Running   2 (60s ago)   72s
```

### 대시보드 실행 ###
```
PS C:\Users\User> minikube addons list
W1004 10:45:15.069064   13264 main.go:291] Unable to resolve the current Docker CLI context "default": context "default": context not found: open C:\Users\User\.docker\contexts\meta\37a8eec1ce19687d132fe29051dca629d164e2c4958ba141d5f4133a33f0688f\meta.json: The system cannot find the path specified.
|-----------------------------|----------|--------------|--------------------------------|
|         ADDON NAME          | PROFILE  |    STATUS    |           MAINTAINER           |
|-----------------------------|----------|--------------|--------------------------------|
| ambassador                  | minikube | disabled     | 3rd party (Ambassador)         |
| auto-pause                  | minikube | disabled     | minikube                       |
| cloud-spanner               | minikube | disabled     | Google                         |
| csi-hostpath-driver         | minikube | disabled     | Kubernetes                     |
| dashboard                   | minikube | disabled     | Kubernetes                     |
| default-storageclass        | minikube | enabled ✅   | Kubernetes                     |
| efk                         | minikube | disabled     | 3rd party (Elastic)            |
| freshpod                    | minikube | disabled     | Google                         |
| gcp-auth                    | minikube | disabled     | Google                         |
| gvisor                      | minikube | disabled     | minikube                       |
| headlamp                    | minikube | disabled     | 3rd party (kinvolk.io)         |
| helm-tiller                 | minikube | disabled     | 3rd party (Helm)               |
| inaccel                     | minikube | disabled     | 3rd party (InAccel             |
|                             |          |              | [info@inaccel.com])            |
| ingress                     | minikube | disabled     | Kubernetes                     |
| ingress-dns                 | minikube | disabled     | minikube                       |
| inspektor-gadget            | minikube | disabled     | 3rd party                      |
|                             |          |              | (inspektor-gadget.io)          |
| istio                       | minikube | disabled     | 3rd party (Istio)              |
| istio-provisioner           | minikube | disabled     | 3rd party (Istio)              |
| kong                        | minikube | disabled     | 3rd party (Kong HQ)            |
| kubevirt                    | minikube | disabled     | 3rd party (KubeVirt)           |
| logviewer                   | minikube | disabled     | 3rd party (unknown)            |
| metallb                     | minikube | disabled     | 3rd party (MetalLB)            |
| metrics-server              | minikube | disabled     | Kubernetes                     |
| nvidia-driver-installer     | minikube | disabled     | 3rd party (Nvidia)             |
| nvidia-gpu-device-plugin    | minikube | disabled     | 3rd party (Nvidia)             |
| olm                         | minikube | disabled     | 3rd party (Operator Framework) |
| pod-security-policy         | minikube | disabled     | 3rd party (unknown)            |
| portainer                   | minikube | disabled     | 3rd party (Portainer.io)       |
| registry                    | minikube | disabled     | minikube                       |
| registry-aliases            | minikube | disabled     | 3rd party (unknown)            |
| registry-creds              | minikube | disabled     | 3rd party (UPMC Enterprises)   |
| storage-provisioner         | minikube | enabled ✅   | minikube                       |
| storage-provisioner-gluster | minikube | disabled     | 3rd party (Gluster)            |
| volumesnapshots             | minikube | disabled     | Kubernetes                     |
|-----------------------------|----------|--------------|--------------------------------|
```
```
PS C:\Users\User> minikube dashboard
W1004 10:45:58.763280   12504 main.go:291] Unable to resolve the current Docker CLI context "default": context "default": context not found: open C:\Users\User\.docker\contexts\meta\37a8eec1ce19687d132fe29051dca629d164e2c4958ba141d5f4133a33f0688f\meta.json: The system cannot find the path specified.
* 대시보드를 활성화하는 중 ...
  - Using image docker.io/kubernetesui/dashboard:v2.7.0
  - Using image docker.io/kubernetesui/metrics-scraper:v1.0.8
* Some dashboard features require the metrics-server addon. To enable all features please run:

        minikube addons enable metrics-server


* Verifying dashboard health ...
* 프록시를 시작하는 중 ...
* Verifying proxy health ...
* Opening http://127.0.0.1:55372/api/v1/namespaces/kubernetes-dashboard/services/http:kubernetes-dashboard:/proxy/ in your default browser...
```
### helo-minikube 이름의 디플로이먼트 생성 ###
```
PS C:\Users\User> kubectl create deployment hello-minikube --image=k8s.gcr.io/echoserver:1.4
deployment.apps/hello-minikube created
```

![image](https://github.com/xodbs1123/Kubernetes/assets/61976898/f9ddb802-c999-40f6-ae08-35589fe18331)


### 클러스터 외부에서 파드로 접근 ###
- **1. NodePort 타입의 서비스 생성 후 해당 포트로 포트 포워딩**
```
PS C:\Users\User> kubectl expose deployment hello-minikube --type=NodePort --port=8080
service/hello-minikube exposed
```
```
PS C:\Users\User> kubectl get service -o wide
NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE   SELECTOR
hello-minikube   NodePort    10.99.152.121   <none>        8080:30807/TCP   22s   app=hello-minikube
kubernetes       ClusterIP   10.96.0.1       <none>        443/TCP          10m   <none>
```
```
PS C:\Users\User> kubectl port-forward service/hello-minikube 9090:8080
Forwarding from 127.0.0.1:9090 -> 8080
Forwarding from [::1]:9090 -> 8080
```
![image](https://github.com/xodbs1123/Kubernetes/assets/61976898/fdf12b33-b22b-40af-8446-0c3715c724e7)

- **2. LoadBalancer 타입의 서비스 생성 후 minikube turnnel 실행**
```
PS C:\Users\User> kubectl create deployment balanced --image=k8s.gcr.io/echoserver:1.4
deployment.apps/balanced created

PS C:\Users\User> kubectl expose deployment balanced --type=LoadBalancer --port=8080
service/balanced exposed
```
```
PS C:\Users\User> minikube tunnel
W1004 10:57:41.571393   15180 main.go:291] Unable to resolve the current Docker CLI context "default": context "default": context not found: open C:\Users\User\.docker\contexts\meta\37a8eec1ce19687d132fe29051dca629d164e2c4958ba141d5f4133a33f0688f\meta.json: The system cannot find the path specified.
* Tunnel successfully started

* NOTE: Please do not close this terminal as this process must stay alive for the tunnel to be accessible ...

* balanced 서비스의 터널을 시작하는 중
```
- 로드 밸런서에 IP 할당 확인
```
PS C:\Users\User> kubectl get service
NAME             TYPE           CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
balanced         LoadBalancer   10.110.47.56    127.0.0.1     8080:30968/TCP   107s
hello-minikube   NodePort       10.99.152.121   <none>        8080:30807/TCP   6m40s
kubernetes       ClusterIP      10.96.0.1       <none>        443/TCP          16m
```
![image](https://github.com/xodbs1123/Kubernetes/assets/61976898/c1dfb66e-c767-4def-8c5e-8b0c42333315)

### 클러스터 삭제 ###
```
PS C:\Users\User> minikube delete
W1004 10:59:25.137528    1360 main.go:291] Unable to resolve the current Docker CLI context "default": context "default": context not found: open C:\Users\User\.docker\contexts\meta\37a8eec1ce19687d132fe29051dca629d164e2c4958ba141d5f4133a33f0688f\meta.json: The system cannot find the path specified.
* docker 의 "minikube" 를 삭제하는 중 ...
```

## Kubespray를 이용한 멀티 노드 쿠버네티스 클러스터 구성 ##
https://myanjini.tistory.com/entry/Kubespray%EB%A5%BC-%ED%99%9C%EC%9A%A9%ED%95%9C-%EB%A9%80%ED%8B%B0-%EB%85%B8%EB%93%9C-%EC%BF%A0%EB%B2%84%EB%84%A4%ED%8B%B0%EC%8A%A4-%ED%81%B4%EB%9F%AC%EC%8A%A4%ED%84%B0-%EA%B5%AC%EC%84%B1

- vagrant 설치
https://www.vagrantup.com/

### Kubespray ###
- 앤서블(ansible)을 사용해 쿠버네티스 클러스터를 구축하는 도구

### Vagrant ###
- 가상화 환경을 구축하고 관리하기 위한 오픈 소스 도구
- 가상 머신의 생성, 시작, 정지, 삭제 등의 작업을 CLI을 통해 간단하게 수행
- 다양한 가상화 플랫폼을 지원(VirtualBox, Vmaware, Hyper-V, Docker 등)
```
vagrant init 		  Vagrantfile을 생성
vagrant up		    Vagrantfile을 기반으로 프로비저닝을 진행
vagrant halt		  가상머신을 종료
vagrant destroy	  가상머신을 삭제
vagrant ssh		    가상머신에 SSH 접속
vagrant provision	가상머신의 설정을 변경하고 적용
```

### Vagrant file, config.yaml 생성 ###
```
C:\Users\User>cd \

C:\>mkdir kubernetes

C:\>cd kubernetes

C:\kubernetes>code .

C:\kubernetes>
```

```Vagrantfile
require "yaml"  

CONFIG = YAML.load_file(File.join(File.dirname(__FILE__), "config.yaml"))

Vagrant.configure("2") do |config|
  # Use the same SSH key for all machines
  config.ssh.insert_key = false

  # masters
  CONFIG["masters"].each do |master|
    config.vm.define master["name"] do |cfg|
      cfg.vm.box = master["box"]
      cfg.vm.network "private_network", ip: master["ip"], virtualbox_intnet: true
      cfg.vm.hostname = master["hostname"]
      
      cfg.vm.provider "virtualbox" do |v|
        v.memory = master["memory"]
        v.cpus = master["cpu"]
        v.name = master["name"]
      end
      cfg.vm.provision "shell", inline: <<-SCRIPT
        sed -i -e "s/PasswordAuthentication no/PasswordAuthentication yes/g" /etc/ssh/sshd_config
        systemctl restart sshd
      SCRIPT

      # set timezone & disable swap memory, ufw & enable ip forwarding
      cfg.vm.provision "shell", inline: <<-SCRIPT
        sudo apt-get update
        sudo timedatectl set-timezone "Asia/Seoul"
        sudo swapoff -a
        sudo sed -i "/swap/d" /etc/fstab
        sudo systemctl stop ufw
        sudo systemctl disable ufw
        sudo sed -i "s/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/" /etc/sysctl.conf
        sudo sysctl -p
      SCRIPT

      # install python 
      cfg.vm.provision "shell", inline: <<-SCRIPT
        sudo apt install python3-pip python3-setuptools virtualenv -y
      SCRIPT
    end
  end
    
  # worker nodes
  CONFIG["workers"].each do |worker|
    config.vm.define worker["name"] do |cfg|
      cfg.vm.box = worker["box"]
      cfg.vm.network "private_network", ip: worker["ip"], virtualbox_intnet: true
      cfg.vm.hostname = worker["hostname"]
      
      cfg.vm.provider "virtualbox" do |v|
        v.memory = worker["memory"]
        v.cpus = worker["cpu"]
        v.name = worker["name"]
      end
      cfg.vm.provision "shell", inline: <<-SCRIPT
        sed -i -e "s/PasswordAuthentication no/PasswordAuthentication yes/g" /etc/ssh/sshd_config
        systemctl restart sshd
      SCRIPT

      # set timezone & disable swap memory & ufw & enable ip forwarding
      cfg.vm.provision "shell", inline: <<-SCRIPT
        sudo apt-get update
        sudo timedatectl set-timezone "Asia/Seoul"
        sudo swapoff -a
        sudo sed -i "/swap/d" /etc/fstab
        sudo systemctl stop ufw
        sudo systemctl disable ufw
        sudo sed -i "s/#net.ipv4.ip_forward=1/net.ipv4.ip_forward=1/" /etc/sysctl.conf
        sudo sysctl -p
      SCRIPT
    end
  end
end
```

```yaml
masters:
  - name: master
    box: generic/ubuntu2204
    hostname: master
    ip: 192.168.10.100
    memory: 2048
    cpu: 2
  
workers:
  - name: worker-1
    box: generic/ubuntu2204
    hostname: worker-1
    ip: 192.168.10.110
    memory: 2048
    cpu: 2

  - name: worker-2
    box: generic/ubuntu2204
    hostname: worker-2
    ip: 192.168.10.120
    memory: 2048
    cpu: 2
```

### virtualbox 세팅 ###
![image](https://github.com/xodbs1123/Kubernetes/assets/61976898/d6db0103-f883-4b45-a0a8-c25c1aac7587)

### vagrant up ###
```
C:\kubernetes>vagrant up
```
![image](https://github.com/xodbs1123/Kubernetes/assets/61976898/ef555ff9-05af-4210-937a-70df28392a62)

### 마스터 노드에 SSH 접속 ###
```
C:\kubernetes>vagrant ssh master
vagrant@master:~$
```

### RSA 방식 암호화키 생성 (pw:vagrant) ###
```
C:\kubernetes>vagrant ssh master
vagrant@master:~$ ssh-keygen -t rsa
Generating public/private rsa key pair.
Enter file in which to save the key (/home/vagrant/.ssh/id_rsa):
Enter passphrase (empty for no passphrase):
Enter same passphrase again:
Your identification has been saved in /home/vagrant/.ssh/id_rsa
Your public key has been saved in /home/vagrant/.ssh/id_rsa.pub
The key fingerprint is:
SHA256:Aos98AhiDQZGI8XxkM1um6aTBVGmYCQjMoeX2zGdyTk vagrant@master
The key's randomart image is:
+---[RSA 3072]----+
|&%=Boo +         |
|X=X== E          |
|ooo*oo .         |
|o.o*+o           |
|  oo=o. S        |
|    =. .         |
|   =             |
|  +              |
|   .             |
+----[SHA256]-----+
```

### 키 생성 확인 ###
```
vagrant@master:~$ ls -al .ssh/
total 20
drwx------ 2 vagrant vagrant 4096 Oct  4 12:20 .
drwxr-x--- 4 vagrant vagrant 4096 Sep 24 12:20 ..
-rw------- 1 vagrant vagrant  409 Sep 24 12:20 authorized_keys
-rw------- 1 vagrant vagrant 2655 Oct  4 12:20 id_rsa
-rw-r--r-- 1 vagrant vagrant  568 Oct  4 12:20 id_rsa.pub
```

### 마스터 및 work 노드에 공개키 배포 ###
```
vagrant@master:~$ ssh-copy-id vagrant@192.168.10.100

vagrant@master:~$ ssh-copy-id vagrant@192.168.10.110

vagrant@master:~$ ssh-copy-id vagrant@192.168.10.120
```

### 스냅샷 생성 ( 많이 해놓으면 좋음 ) ###
```
C:\kubernetes>vagrant snapshot save 01-init-state
==> master: Snapshotting the machine as '01-init-state'...
==> master: Snapshot saved! You can restore the snapshot at any time by
==> master: using `vagrant snapshot restore`. You can delete it using
==> master: `vagrant snapshot delete`.
==> worker-1: Snapshotting the machine as '01-init-state'...
==> worker-1: Snapshot saved! You can restore the snapshot at any time by
==> worker-1: using `vagrant snapshot restore`. You can delete it using
==> worker-1: `vagrant snapshot delete`.
==> worker-2: Snapshotting the machine as '01-init-state'...
==> worker-2: Snapshot saved! You can restore the snapshot at any time by
==> worker-2: using `vagrant snapshot restore`. You can delete it using
==> worker-2: `vagrant snapshot delete`.
```
```
C:\kubernetes>vagrant snapshot list
==> master:
01-init-state
==> worker-1:
01-init-state
==> worker-2:
01-init-state
```

### master 노드에 파이썬 가상환경 설치 ###
```
vagrant@master:~$ virtualenv --python=python3 venv
created virtual environment CPython3.10.12.final.0-64 in 298ms
  creator CPython3Posix(dest=/home/vagrant/venv, clear=False, no_vcs_ignore=False, global=False)
  seeder FromAppData(download=False, pip=bundle, setuptools=bundle, wheel=bundle, via=copy, app_data_dir=/home/vagrant/.local/share/virtualenv)
    added seed packages: pip==22.0.2, setuptools==59.6.0, wheel==0.37.1
  activators BashActivator,CShellActivator,FishActivator,NushellActivator,PowerShellActivator,PythonActivator
vagrant@master:~$ . venv/bin/activate
```

### kuberspray 설치 ###
```
(venv) vagrant@master:~$ git clone https://github.com/kubernetes-sigs/kubespray
```
```
(venv) vagrant@master:~$ cd kubespray
(venv) vagrant@master:~/kubespray$ pip install -r requirements.txt
```

### ansible 설치 확인 ###
```
(venv) vagrant@master:~/kubespray$ ansible --version
ansible [core 2.14.10]
  config file = /home/vagrant/kubespray/ansible.cfg
  configured module search path = ['/home/vagrant/kubespray/library']
  ansible python module location = /home/vagrant/venv/lib/python3.10/site-packages/ansible
  ansible collection location = /home/vagrant/.ansible/collections:/usr/share/ansible/collections
  executable location = /home/vagrant/venv/bin/ansible
  python version = 3.10.12 (main, Jun 11 2023, 05:26:28) [GCC 11.4.0] (/home/vagrant/venv/bin/python)
  jinja version = 3.1.2
  libyaml = True
```

### master 노드에서 ansible 인벤토리 설정 ###
```
(venv) vagrant@master:~/kubespray$ cp -rfp inventory/sample inventory/mycluster
(venv) vagrant@master:~/kubespray$ ls inventory/mycluster/
group_vars  inventory.ini  patches
```
```
(venv) vagrant@master:~/kubespray$ CONFIG_FILE=inventory/mycluster/hosts.yaml python3 contrib/inventory_builder/inventory.py ${IPS[@]}
```
```
(venv) vagrant@master:~/kubespray$ vi inventory/mycluster/hosts.yaml

all:
  hosts:
    master:
      ansible_host: 192.168.10.100
      ip: 192.168.10.100
      access_ip: 192.168.10.100
    worker1:
      ansible_host: 192.168.10.110
      ip: 192.168.10.110
      access_ip: 192.168.10.110
    worker2:
      ansible_host: 192.168.10.120
      ip: 192.168.10.120
      access_ip: 192.168.10.120
  children:
    kube_control_plane:
      hosts:
        master:
    kube_node:
      hosts:
        worker1:
        worker2:
    etcd:
      hosts:
        master:
    k8s_cluster:
      children:
        kube_control_plane:
        kube_node:
    calico_rr:
      hosts: {}
```
```
(venv) vagrant@master:~/kubespray$ ansible-playbook -i inventory/mycluster/hosts.yaml --become --become-user=root cluster.yml

중간에 나오는 password 입력 (vagrant)
```
