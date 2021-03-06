
# Ingress 简介

```
 在Kubernetes中，服务和Pod的IP地址仅可以在集群网络内部使用，对于集群外的应用是不可见的。为了使外部的应用能够访问集群内的服务，在Kubernetes 目前 提供了以下几种方案：
NodePort
LoadBalancer
Ingress

```

## Ingress 组成
```
    ingress controller
　　将新加入的Ingress转化成Nginx的配置文件并使之生效
ingress服务
　　将Nginx的配置抽象成一个Ingress对象，每添加一个新的服务只需写一个新的Ingress的yaml文件即可
  
```

## Ingress 工作原理
```
1.ingress controller通过和kubernetes api交互，动态的去感知集群中ingress规则变化，
2.然后读取它，按照自定义的规则，规则就是写明了哪个域名对应哪个service，生成一段nginx配置，
3.再写到nginx-ingress-control的pod里，这个Ingress controller的pod里运行着一个Nginx服务，控制器会把生成的nginx配置写入/etc/nginx.conf文件中，
4.然后reload一下使配置生效。以此达到域名分配置和动态更新的问题。

```

## Ingress 可以解决什么问题

```

.动态配置服务
　　如果按照传统方式, 当新增加一个服务时, 我们可能需要在流量入口加一个反向代理指向我们新的k8s服务. 而如果用了Ingress, 只需要配置好这个服务, 当服务启动时, 会自动注册到Ingress的中, 不需要而外的操作.
2.减少不必要的端口暴露
　　配置过k8s的都清楚, 第一步是要关闭防火墙的, 主要原因是k8s的很多服务会以NodePort方式映射出去, 这样就相当于给宿主机打了很多孔, 既不安全也不优雅. 而Ingress可以避免这个问题, 除了Ingress自身服务可能需要映射出去, 其他服务都不要用NodePort方式
  
```

# 部署ingress

## 部署下载

```
  提供了两种方式 ：
　　默认下载最新的yaml：
　　　　wget  https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
　　指定版本号下载对应的yaml   
```

## 部署文件介绍

```
    1.namespace.yaml 
    创建一个独立的命名空间 ingress-nginx

    2.configmap.yaml 
    ConfigMap是存储通用的配置变量的，类似于配置文件，使用户可以将分布式系统中用于不同模块的环境变量统一到一个对象中管理；而它与配置文件的区别在于它是存在集群的“环境”中的，并且支持K8S集群中所有通用的操作调用方式。
    从数据角度来看，ConfigMap的类型只是键值组，用于存储被Pod或者其他资源对象（如RC）访问的信息。这与secret的设计理念有异曲同工之妙，主要区别在于ConfigMap通常不用于存储敏感信息，而只存储简单的文本信息。
    ConfigMap可以保存环境变量的属性，也可以保存配置文件。
    创建pod时，对configmap进行绑定，pod内的应用可以直接引用ConfigMap的配置。相当于configmap为应用/运行环境封装配置。
    pod使用ConfigMap，通常用于：设置环境变量的值、设置命令行参数、创建配置文件。

    3.default-backend.yaml 
    如果外界访问的域名不存在的话，则默认转发到default-http-backend这个Service，其会直接返回404：

    4.rbac.yaml 
    负责Ingress的RBAC授权的控制，其创建了Ingress用到的ServiceAccount、ClusterRole、Role、RoleBinding、ClusterRoleBinding

    5.with-rbac.yaml 
    是Ingress的核心，用于创建ingress-controller。前面提到过，ingress-controller的作用是将新加入的Ingress进行转化为Nginx的配置

```

## 部署ingress

准备镜像
(1) quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.23.0
(2) registry.cn-hangzhou.aliyuncs.com/sunnydou/defaultbackend:1.4


## 执行脚本yaml

service-nodeport 是一个service 是直接指向 nginx-ingress-controller的

了解了以上内容以后，这也就很好的说明了我为什么喜欢把负载均衡器部署为 Daemon Set；因为无论如何请求首先是被负载均衡器拦截的，所以在每个 node 上都部署一下，同时 hostport 方式监听 80 端口；那么就解决了其他方式部署不确定 负载均衡器在哪的问题，同时访问每个 node 的 80 都能正确解析请求；如果前端再 放个 nginx 就又实现了一层负载均衡

```
  kubectl apply -f mandatory.yaml default-backend.yaml service-nodeport.yaml
```

## 查看nginx-ingress组件状态

```
     [root@k8s-master deploy]# kubectl get all -n ingress-nginx
    NAME                                            READY   STATUS    RESTARTS   AGE
    pod/default-http-backend-647f778689-s6hn4       1/1     Running   0          17h
    pod/nginx-ingress-controller-74949d46bd-sxk26   1/1     Running   0          17h

    NAME                           TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
    service/default-http-backend   ClusterIP   10.254.200.36   <none>        80/TCP                       17h
    service/ingress-nginx          NodePort    10.254.58.75    <none>        80:29758/TCP,443:24841/TCP   92m

    NAME                                       READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/default-http-backend       1/1     1            1           17h
    deployment.apps/nginx-ingress-controller   1/1     1            1           17h

    NAME                                                  DESIRED   CURRENT   READY   AGE
    replicaset.apps/default-http-backend-647f778689       1         1         1       17h
    replicaset.apps/nginx-ingress-controller-74949d46bd   1         1         1       17h
    replicaset.apps/nginx-ingress-controller-7966d94d6c   0         0         0       17h
```

## 访问nginx-ingress

![nginx-ingress](./images/nginx-ingress.png)

可以看到，提示404，这个因为当前ingress-nginx服务现在还没有后端服务，这是正常的



# 创建后端服务

## 创建一个Service及后端Deployment(以tomcat为例)

```
 [root@k8s-master ingress-nginx]# cat tomcat-deploy.yaml 
apiVersion: v1
kind: Service
metadata:
  name: tomcat
  namespace: default
spec:
  selector:
    app: tomcat
    release: canary
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  - name: ajp
    port: 8009
    targetPort: 8009

---
apiVersion: apps/v1
kind: Deployment
metadata: 
  name: tomcat-deploy
spec:
  replicas: 3
  selector: 
    matchLabels:
      app: tomcat
      release: canary
  template:
    metadata:
      labels:
        app: tomcat
        release: canary
    spec:
      containers:
      - name: tomcat
        image: tomcat:7-alpine
        ports:
        - name: httpd
          containerPort: 8080
        - name: ajp
          containerPort: 8009 
```

将tomcat添加至ingress-nginx中，增加到ingress脚本中
app-nginx-ingress
```

cat nginx-ingress.yaml

apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: app-nginx-ingress
  namespace: default
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx
          servicePort: 80
    http:
      paths:
      - path: /examples
        backend:
          serviceName: tomcat
          servicePort: 8080
[root@k8s-master deploy]# 

```

192.168.1.160  foo.bar.com  添加到/etc/hosts,客户端的hosts也需要添加


## 查看2个服务

![nginx-ingress-service1](./images/nginx-ingress-service1.png)

![nginx-ingress-service2](./images/nginx-ingress-service2.png)


![nginx-ingress-rule](./images/nginx-ingress-rule.png)

![nginx-ingress-health](./images/nginx-ingress-health.png)


# 下面我们对tomcat服务添加httpds服务

## 创建私有证书及secret

```
  [root@k8s-master ingress-nginx]# openssl genrsa -out tls.key 2048
  Generating RSA private key, 2048 bit long modulus
  .......+++
  ..............................+++
  e is 65537 (0x10001)
  [root@k8s-master ingress-nginx]# openssl req -new -x509 -key tls.key -out tls.crt -subj /C=CN/ST=Beijing/L=Beijing/O=DevOps/CN=tomcat.magedu.com #注意域名要和服务的域名一致 
  [root@k8s-master ingress-nginx]# kubectl create secret tls tomcat-ingress-secret --cert=tls.crt --key=tls.key #创建secret
  secret "tomcat-ingress-secret" created
  [root@k8s-master ingress-nginx]# kubectl get secret
  NAME                    TYPE                                  DATA      AGE
  default-token-bf52l     kubernetes.io/service-account-token   3         9d
  tomcat-ingress-secret   kubernetes.io/tls                     2         7s
  [root@k8s-master ingress-nginx]# kubectl describe secret tomcat-ingress-secret
  Name:         tomcat-ingress-secret
  Namespace:    default
  Labels:       <none>
  Annotations:  <none>

  Type:  kubernetes.io/tls

  Data
  ====
  tls.crt:  1294 bytes  #base64加密
  tls.key:  1679 bytes
```
将证书应用至tomcat服务中

```
root@k8s-master01 ingress]# cat ingress-tomcat-tls.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-tomcat-tls
  namespace: default
  annotations: 
    kubernets.io/ingress.class: "nginx"
spec:
  tls:
  - hosts:
    - tomcat.magedu.com        #与secret证书的域名需要保持一致
    secretName: tomcat-ingress-secret   #secret证书的名称
  rules:
  - host: tomcat.magedu.com
    http:
      paths:
      - path: 
        backend:
          serviceName: tomcat
          servicePort: 8080
  

```

访问服务




