 

 

 

#### \# 查看自动生成的config.json 文件

```
[root@k8s-master1 YAML-master1]# cat ~/.docker/config.json 

{

"auths": {

"docker-registry:5000": {

"auth": "amFja3k6MTIzNDU2"

}

},

"HttpHeaders": {

"User-Agent": "Docker-Client/19.03.8 (linux)"

}

}
```

 

#### \# 查看是否可以在master主机和node1、node2上正常下载私有镜像仓库images

```
[root@k8s-master1 YAML-master1]# docker pull docker-registry:5000/tomcat:latest

latest: Pulling from tomcat

50e431f79093: Pull complete 

dd8c6d374ea5: Pull complete 

c85513200d84: Pull complete 

55769680e827: Pull complete 

e27ce2095ec2: Pull complete 

5943eea6cb7c: Pull complete 

3ed8ceae72a6: Pull complete 

91d1e510d72b: Pull complete 

98ce65c663bc: Pull complete 

27d4ac9d012a: Pull complete 

Digest: sha256:7c7516e06cb75dd90941edc5705f0ef10ea48e60ebad2baaa5c57106c6789a17

Status: Downloaded newer image for docker-registry:5000/tomcat:latest

docker-registry:5000/tomcat:latest
```

 

#### \# 创建k8s用的密钥文件 regcred

```
[root@k8s-master1 YAML-master1]# kubectl create secret docker-registry regcred --docker-server=docker-registry --docker-username=jacky --docker-password=123456 --docker-email=zhangjianye@hotmail.com

secret/regcred created

[root@k8s-master1 YAML-master1]# kubectl get secret regcred --output=yaml

apiVersion: v1

data:

 .dockerconfigjson: eyJhdXRocyI6eyJkb2NrZXItcmVnaXN0cnkiOnsidXNlcm5hbWUiOiJqYWNreSIsInBhc3N3b3JkIjoiMTIzNDU2IiwiZW1haWwiOiJ6aGFuZ2ppYW55ZUBob3RtYWlsLmNvbSIsImF1dGgiOiJhbUZqYTNrNk1USXpORFUyIn19fQ==

kind: Secret

metadata:

 creationTimestamp: "2020-04-10T16:20:27Z"

 name: regcred

 namespace: default

 resourceVersion: "150102"

 selfLink: /api/v1/namespaces/default/secrets/regcred

 uid: 8192dd41-82ef-4183-9a08-603493461dc6

type: kubernetes.io/dockerconfigjson
```

 

 

 

#### \# 通过kubectl命令查询创建的regcred, 密钥文件是否正确

[root@k8s-master1 YAML-master1]# kubectl get secret regcred --output="jsonpath={.data.\.dockerconfigjson}" | base64 --decode

{"auths":{"docker-registry":{"username":"jacky","password":"123456","email":"zhangjianye@hotmail.com","auth":"amFja3k6MTIzNDU2"}}}

 

#### \# 反向查看生成密码是否正确

```
[root@k8s-master1 YAML-master1]# echo "amFja3k6MTIzNDU2" | base64 --decode

jacky:123456

 

[root@k8s-master1 YAML-master1]# ll

total 52

-rw-r--r-- 1 root root 745 Oct 2 2019 apiserver-to-kubelet-rbac.yaml

-rw-r--r-- 1 root root 283 Oct 4 2019 bs.yaml

-rw-r--r-- 1 root root 4283 Oct 2 2019 coredns.yaml

-rw-r--r-- 1 root root 373 Oct 4 2019 dashboard-adminuser.yaml

-rw-r--r-- 1 root root 7098 Oct 4 2019 dashboard.yaml

-rw-r--r-- 1 root root 5032 Oct 2 2019 kube-flannel.yaml

-rw-r--r-- 1 root root 389 Apr 10 02:34 my-deploy.yaml

-rw-r--r-- 1 root root 191 Apr 10 09:44 my-private-reg-pod.yaml

-rw-r--r-- 1 root root 500 Apr 10 09:55 registry-pull-secret.yaml

-rw-r--r-- 1 root root 431 Apr 10 12:05 tomcat-deployment.yaml

[root@k8s-master1 YAML-master1]# cat tomcat-deployment.yaml 

apiVersion: apps/v1

kind: Deployment

metadata:

 creationTimestamp: null

 labels:

  run: tomcat

 name: tomcat

spec:

 replicas: 2

 selector:

  matchLabels:

   run: tomcat

 strategy: {}

 template:

  metadata:

   creationTimestamp: null

   labels:

​    run: tomcat

  spec:

   containers:

   \- image: docker-registry:5000/tomcat:latest

​    name: tomcat

   imagePullSecrets:

   \- name: regcred
```

 

#### \# 查看之前创建的pod

```
[root@k8s-master1 YAML-master1]# kubectl get pod

NAME   READY  STATUS  RESTARTS  AGE

busybox  1/1   Running  16     15h
```

 

#### \# 创建新的pod

```
[root@k8s-master1 YAML-master1]# kubectl create -f tomcat-deployment.yaml 

deployment.apps/tomcat created
```

 

 

```
[root@k8s-master1 YAML-master1]# kubectl get pod

NAME           READY  STATUS       RESTARTS  AGE

busybox          1/1   Running      16     15h

tomcat-75d645776d-6wgrs  0/1   ImagePullBackOff  0     14s

tomcat-75d645776d-d8sn2  0/1   ImagePullBackOff  0     14s

[root@k8s-master1 YAML-master1]# kubectl describe pod tomcat-75d645776d-6wgrs

Name:     tomcat-75d645776d-6wgrs

Namespace:  default

Priority:   0

Node:     k8s-node2/192.168.31.232

Start Time:  Fri, 10 Apr 2020 12:25:22 -0400

Labels:    pod-template-hash=75d645776d

​       run=tomcat

Annotations: <none>

Status:    Pending

IP:      10.244.1.44

IPs:

 IP:      10.244.1.44

Controlled By: ReplicaSet/tomcat-75d645776d

Containers:

 tomcat:

  Container ID:  

  Image:     docker-registry:5000/tomcat:latest

  Image ID:    

  Port:      <none>

  Host Port:   <none>

  State:     Waiting

   Reason:    ImagePullBackOff

  Ready:     False

  Restart Count: 0

  Environment:  <none>

  Mounts:

   /var/run/secrets/kubernetes.io/serviceaccount from default-token-zh8vs (ro)

Conditions:

 Type       Status

 Initialized    True 

 Ready       False 

 ContainersReady  False 

 PodScheduled   True 

Volumes:

 default-token-zh8vs:

  Type:    Secret (a volume populated by a Secret)

  SecretName: default-token-zh8vs

  Optional:  false

QoS Class:    BestEffort

Node-Selectors: <none>

Tolerations:   node.kubernetes.io/not-ready:NoExecute for 300s

​         node.kubernetes.io/unreachable:NoExecute for 300s

Events:

 Type   Reason     Age        From        Message

 ----   ------     ----        ----        -------

 Normal  Scheduled    <unknown>     default-scheduler  Successfully assigned default/tomcat-75d645776d-6wgrs to k8s-node2

 Normal  SandboxChanged 71s        kubelet, k8s-node2 Pod sandbox changed, it will be killed and re-created.

 Normal  Pulling     28s (x3 over 72s) kubelet, k8s-node2 Pulling image "docker-registry:5000/tomcat:latest"

 Warning Failed     28s (x3 over 72s) kubelet, k8s-node2 Failed to pull image "docker-registry:5000/tomcat:latest": rpc error: code = Unknown desc = Error response from daemon: Get [https://docker-registry:5000/v2/tomcat/manifests/latest](https://docker-registry:5000/v2/tomcat/manifests/latest): no basic auth credentials

 Warning Failed     28s (x3 over 72s) kubelet, k8s-node2 Error: ErrImagePull

 Normal  BackOff     2s (x6 over 70s)  kubelet, k8s-node2 Back-off pulling image "docker-registry:5000/tomcat:latest"

 Warning Failed     2s (x6 over 70s)  kubelet, k8s-node2 Error: ImagePullBackOff

[root@k8s-master1 YAML-master1]# 
```

 

 

 

 
