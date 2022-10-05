# k8s-ingress-learning



第一步：部署K8S


第二步：部署MetalLB

参考文档：
 - https://metallb.universe.tf/configuration
 - https://www.51cto.com/article/717556.html

```
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.5/config/manifests/metallb-native.yaml
```
创建 IP pool 
```
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: first-pool
  namespace: metallb-system
spec:
  addresses:
  - 192.168.99.161-192.168.99.180
```
L2 Advertisement
```
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata:
  name: first-pool-l2
  namespace: metallb-system
```


第三步：部署Contour Ingress

```
kubectl apply -f https://projectcontour.io/quickstart/contour.yaml
kubectl get pods -n projectcontour -o wide
```

```
[root@centos7-1 k8s-ingress-learning]# kubectl get svc -A
NAMESPACE        NAME              TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                      AGE
default          kubernetes        ClusterIP      10.96.0.1       <none>           443/TCP                      46h
default          web-a-svc         ClusterIP      10.109.46.245   <none>           80/TCP                       22h
default          web-b-svc         ClusterIP      10.97.190.227   <none>           80/TCP                       22h
kube-system      kube-dns          ClusterIP      10.96.0.10      <none>           53/UDP,53/TCP,9153/TCP       46h
metallb-system   webhook-service   ClusterIP      10.103.188.17   <none>           443/TCP                      23h
projectcontour   contour           ClusterIP      10.111.53.113   <none>           8001/TCP                     69m
projectcontour   envoy             LoadBalancer   10.105.197.10   192.168.99.161   80:31445/TCP,443:32000/TCP   69m
```

第四步：创建Dockerfile
注意：需要创建两个文件夹  index-a 和 index-b

分别创建两个镜像 web-a  和  web-b

下面以web-a为例

创建index-a.html
```
<html>
<head>
  <title> welcome to Server A! </title>
</head>
<body>
  <p> Welcome to Server A! </p>
</body
```

创建Dockerfile
```
FROM ubuntu:latest
RUN apt-get update;\
apt-get install apache2 -y
COPY index-a.html /var/www/html/index-a.html
EXPOSE 80
ENTRYPOINT ["/usr/sbin/apache2ctl","-D","FOREGROUND"]
```

第五步：制作和上传镜像
```
docker build -t yxyj1919/ingress-demo-a .

docker login

docker push yxyj1919/ingress-demo-a:latest
```

验证
```
docker images

docker run -d -p 8080:80 -h ingress-demo.mydomain.com yxyj1919/ingress-demo-a:latest

docker rm XXX
```


第六步：创建Deployment和Service

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-a-deployment
  labels:
    app: web-a
spec:
  replicas: 1
  selector:
    matchLabels:
      app: web-a
  template:
    metadata:
      labels:
        app: web-a
    spec:
      containers:
      - name: web-a
        image: yxyj1919/ingress-demo-a
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  labels:
    app: web-a
  name: web-a-svc
spec:
  selector:
    app: web-a
  ports:
    - name: http
      protocol: TCP
      port: 80
  sessionAffinity: None
  type: ClusterIP
```

第七步：创建Ingress

```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
spec:
  rules:
  - host: web.mydomain.com
    http:
      paths:
      - path: /index-a.html
        pathType: Prefix
        backend:
          service:
            name: web-a-svc
            port:
              number: 80
      - path: /index-b.html
        pathType: Prefix
        backend:
          service:
            name: web-b-svc
            port:
              number: 80
```

第八步：部署/验证

```
[root@centos7-1 index-a]# kubectl get pod
NAME                                READY   STATUS    RESTARTS   AGE
web-a-deployment-857d856968-ndf5b   1/1     Running   0          22h
web-b-deployment-57d8d64655-q4sm9   1/1     Running   0          22h

[root@centos7-1 index-a]# kubectl get svc -A
NAMESPACE        NAME              TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)                      AGE
default          kubernetes        ClusterIP      10.96.0.1       <none>           443/TCP                      47h
default          web-a-svc         ClusterIP      10.109.46.245   <none>           80/TCP                       22h
default          web-b-svc         ClusterIP      10.97.190.227   <none>           80/TCP                       22h
kube-system      kube-dns          ClusterIP      10.96.0.10      <none>           53/UDP,53/TCP,9153/TCP       47h
metallb-system   webhook-service   ClusterIP      10.103.188.17   <none>           443/TCP                      23h
projectcontour   contour           ClusterIP      10.111.53.113   <none>           8001/TCP                     95m
projectcontour   envoy             LoadBalancer   10.105.197.10   192.168.99.161   80:31445/TCP,443:32000/TCP   95m

[root@centos7-1 index-a]# kubectl get ingress
NAME          CLASS    HOSTS              ADDRESS          PORTS   AGE
web-ingress   <none>   web.mydomain.com   192.168.99.161   80      22h
```
