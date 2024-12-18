###### 流量管理--创建 Ingress Gateway

使用提供的软件包 ServiceMesh.tar.gz 将 Bookinfo 应用部署到 default 命名空间下，使用 Istio Gateway 可 以实 现应 用程 序从 外部 访问， 请为 Bookinfo 应用创 建一 个名 为 bookinfo-gateway 的网关，指定所有 HTTP 流量通过 80 端口流入网格，然后将网关绑定到虚 拟服务 bookinfo 上。

```shell
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway # use istio default controller
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```



###### 服务网格--创建基于用户身份的路由

创建一个名为 reviews 路由，要求来自名为 Jason 的用户的所有流量将被路由到服务 reviews:v2。 

```shell
$ kubectl get virtualservice reviews -o yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
...
spec:
  hosts:
  - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1
```



###### 服务网格--创建请求路由

在default命名空间下创建一个名为reviews-route的虚拟服务，默认情况下，所有的HTTP 流量都会被路由到标签为 version:v1 的 reviews 服务的 Pod 上。此外，路径以/wpcatalog/或 /consumercatalog/开头的 HTTP 请求将被重写为/newcatalog，并被发送到标签为 version:v2 的 Pod 上。

```bash
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews-route
spec:
  #客户端访问服务的地址
  hosts:
  - reviews.prod.svc.cluster.local
  http:
  #如果请求的前缀为/wpcatalog、/consumercatalog，流量路由到v2版本的reviews中。
  - name: "reviews-v2-routes"
    match:
    - uri:
        prefix: "/wpcatalog"
    - uri:
        prefix: "/consumercatalog"
    rewrite:
      uri: "/newcatalog"
    route:
    - destination:
        host: reviews.prod.svc.cluster.local
        # 匹配的服务版本或子集为v2
        subset: v2
  #其它请求，流量路由到v1版本的reviews中。
  - name: "reviews-v1-route"
    route:
    - destination:
        host: reviews.prod.svc.cluster.local
        subset: v1
```

###### 服务网格：创建 DestinationRule

将 Bookinfo 应用部署到 default 命名空间下，为 Bookinfo 应用的四个微服务
设置默认目标规则，指定各个服务的可用版本。

```
apiVersion: networking.istio.io/v1beta3
kind: DestinationRule
metadata: 
  name: reviews
spec:
  host: reviews
  subsets: 
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
---
apiVersion: networking.istio.io/v1beta3
kind: DestinationRule
metadata:
  name: details
spec:
  host: details
  subsets: 
  - name: v1
    labels:
      version: v1
---
apiVersion: networking.istio.io/v1beta3
kind: DestinationRule
metadata:
  name: productpage
spec:
  host: productpage
  subsets:
  - name: v1
    labels:
      version: v1
---
apiVersion: networking.istio.io/v1beta3
kind: DestinationRule
metadata:
  name: ratings
spec:
  host: ratings
  subsets:
  - name: v1
    labels:
      version: v1
```

###### 服务网格：路由管理

将 Bookinfo 应用部署到 default 命名空间下，应用默认请求路由，将所有流
量路由到各个微服务的 v1 版本。然后更改请求路由 reviews，将指定比例的流量
从 reviews 的 v1 转移到 v3

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 50
    - destination:
        host: reviews
        subset: v3
      weight: 50
```

######  服务网格：流量镜像

将 Bookinfo 应用部署到 default 命名空间下，创建请求路由 reviews，将指定
的流量路由到 reviews 微服务的 v1 版本，然后将相同流量镜像到 reviews 微服务
的 v2 版本

```
[root@master ServiceMesh]# kubectl apply -f virtual-service-all-v1.yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
   - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 100
    mirror:
      host: reviews
      subset: v2
```

###### 服务网格：Sidecar 管理

在 default 命名空间下部署 Bookinfo 应用。创建 exam 命名空间，并声明一个
Sidecar 配置，允许向指定命名空间的公共服务输出流量。为所有指定标签的 Pod 声
明一个Sidecar 配置，接收和转发指定的流量。

```
apiVersion: networking.istio.io/v1alpha3
kind: Sidecar
metadata:
  name: default
  namespace: prod-us1
spec:
  egress:
  - hosts:
    - "prod-us1/*"
    - "prod-apis/*"
    - "istio-system/*"
```



###### 服务网格：创建 VirtualService

将 Bookinfo 应用部署到 default 命名空间下，为 Bookinfo 应用的四个微服务
设置默认版本的 VirtualService，将所有流量路由到每个微服务的 v1 版本

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: productpage
spec:
  hosts:
  - productpage
  http:
  - route:
    - destination:
        host: productpage
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: details
spec:
  hosts:
  - details
  http:
  - route:
    - destination:
        host: details
        subset: v1
```

```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: productpage
spec:
  hosts:
  - productpage
  http:
  - route:
    - destination:
        host: productpage
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: ratings
spec:
  hosts:
  - ratings
  http:
  - route:
    - destination:
        host: ratings
        subset: v1
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: details
spec:
  hosts:
  - details
  http:
  - route:
    - destination:
        host: details
        subset: v1
---
```



###### 容器云运维开发

一、管理Pod资源：使用SDK方式管理Pod服务

1.创建pod资源

创建pod需要提前将pod的yaml文件放到python程序的同级目录总，并引用yaml以及watch模块。

创建pod资源yaml文件如下：

```python
apiVersion: v1
kind: Pod
metadata:
  name: "my-nginx"
  namespace: default
  labels:
    app: "my-nginx"
spec:
  containers:
  - name: my-pod
    image: "nginx"
    ports:
    - containerPort: 80
      name: nginx
  restartPolicy: Always
```



创建pod的步骤为：

1、 创建config对象

2、 使用os模块打开yaml文件

3、 获取api对象

4、 使用create_namespaced_pod方法创建pod并返回结果。

案例代码如下：

```python
from  kubernetes import client,config,watch
import yaml
config.load_kube_config("config")
with open("nginx-pod.yaml",'r') as f:
    pod=yaml.safe_load(f)
    k8s_apps_v1=client.CoreV1Api()
    resp=k8s_apps_v1.create_namespaced_pod(body=pod,namespace="default")
    print("pod  status='%s'" % resp.metadata.name)
```

程序执行结果为：

```python
D:\My_ENV\Python39\python.exe E:\Project\PycharmProject\k8s_api\show_sdk.py 
pod  status='my-nginx'

Process finished with exit code 0
```

Client命令执行结果为：

```shell
[root@master ~]# kubectl get po
NAME       READY   STATUS    RESTARTS   AGE
my-nginx   1/1     Running   0          2m2s
[root@master ~]#
```

2.查看pod资源列表

跟nodes操作系统，list_namespaced_pod（）方法可对指定的namesapce中的pods做为json格式进行返回。使用遍历手段以及item方法将json数据转化为列表数据。

Pod_listsa案例代码如下：

```python
from kubernetes import client, config
def main():
    config.load_kube_config("config")
    api_instance = client.CoreV1Api()
    # Listing the cluster nodes
    node_list = api_instance.list_namespaced_pod(namespace="default")
    print("%s\t\t%s" % ("NAME", "LABELS"))
    # Patching the node labels
    for node in node_list.items:
        print("%s\t%s" % (node.metadata.name, node.metadata.labels))
if __name__ == '__main__':
    main()
```

执行结果如下：

```python
D:\My_ENV\Python39\python.exe E:\Project\PycharmProject\k8s_api\show_sdk.py 
NAME        LABELS
myjob-m5dks {'app': 'myjob', 'controller-uid': '1b126df2-5027-4d86-8712-3fe5bcc0328a', 'job-name': 'myjob'}

Process finished with exit code 0
```

这个结果是跟我们客户端的执行结果是一致的，如果想获得所有命名空间的pods列表需要将字段namespace设置为空。

代码案例如下：

```python
from kubernetes import client, config
def main():
    config.load_kube_config("config")
    api_instance = client.CoreV1Api()
    # Listing the cluster nodes
    node_list = api_instance.list_namespaced_pod(namespace="")
    print("%s\t\t%s" % ("NAME", "LABELS"))
    # Patching the node labels
    for node in node_list.items:
        print("%s\t%s" % (node.metadata.name, node.metadata.labels))
if __name__ == '__main__':
    main()
```

执行结果如下：

```python
D:\My_ENV\Python39\python.exe E:\Project\PycharmProject\k8s_api\show_sdk.py 
NAME        LABELS
myjob-m5dks {'app': 'myjob', 'controller-uid': '1b126df2-5027-4d86-8712-3fe5bcc0328a', 'job-name': 'myjob'}
kube-flannel-ds-qqnv6   {'app': 'flannel', 'controller-revision-hash': '975fb4779', 'pod-template-generation': '3', 'tier': 'node'}
kube-flannel-ds-wh8zc   {'app': 'flannel', 'controller-revision-hash': '975fb4779', 'pod-template-generation': '3', 'tier': 'node'}
coredns-6d8c4cb4d-2g76m {'k8s-app': 'kube-dns', 'pod-template-hash': '6d8c4cb4d'}
coredns-6d8c4cb4d-zgmj9 {'k8s-app': 'kube-dns', 'pod-template-hash': '6d8c4cb4d'}
etcd-master {'component': 'etcd', 'tier': 'control-plane'}
kube-apiserver-master   {'component': 'kube-apiserver', 'tier': 'control-plane'}
kube-controller-manager-master  {'component': 'kube-controller-manager', 'tier': 'control-plane'}
kube-proxy-b4r9b    {'controller-revision-hash': '8554974c5b', 'k8s-app': 'kube-proxy', 'pod-template-generation': '1'}
kube-proxy-cxczw    {'controller-revision-hash': '8554974c5b', 'k8s-app': 'kube-proxy', 'pod-template-generation': '1'}
kube-scheduler-master   {'component': 'kube-scheduler', 'tier': 'control-plane'}
dashboard-metrics-scraper-577dc49767-9dw9j  {'k8s-app': 'dashboard-metrics-scraper', 'pod-template-hash': '577dc49767'}
kubernetes-dashboard-78f9d9744f-rgw6c   {'k8s-app': 'kubernetes-dashboard', 'pod-template-hash': '78f9d9744f'}

Process finished with exit code 0
```

这样就可以将所有命名空间的pods列表打印出来。

3.删除pod资源

删除pod不需要使用yaml文件只需要使用delete_namespaced_pod方法即可。

```python
from  kubernetes import client,config,watch
import yaml
def delete_pod():
    config.load_kube_config("config")
    k8s_apps_v1=client.CoreV1Api()
    resp=k8s_apps_v1.delete_namespaced_pod(name="my-nginx",namespace="default")
    print(resp.metadata.name)
if __name__=='__main__':
    delete_pod()
```

执行结果为：

```python
D:\My_ENV\Python39\python.exe E:\Project\PycharmProject\k8s_api\show_sdk.py 
my-nginx

Process finished with exit code 0
```

通过client端执行命令查看结果为：

```shell
[root@master ~]# kubectl get pod
No resources found in default namespace.
```





二、管理job服务：使用SDK方式管理job服务。

1.job list 

系统中没有默认的job在运行，因此需要提前准备一个job来测试我们的joblist代码运行是否正常。

Job的yaml文件内容：

```yaml
# https://kubernetes.io/docs/concepts/workloads/controllers/job/
apiVersion: batch/v1
kind: Job
metadata:
  name: myjob
  namespace: default
  labels:
    app: myjob
spec:
  template:
    metadata:
      name: myjob
      labels:
        app: myjob
    spec:
      containers:
      - name: myjob
        image: nginx:latest
      restartPolicy: OnFailure
---
```

使用上面的yaml文件生成一个my-job的job工作。

List-job需要使用BatchV1Api来获取client对象进行list操作，具体代码案例如下：

```python
from kubernetes import  config,client
def list_job():
    config.load_kube_config("config")
    k8s_client=client.BatchV1Api()
    job_list=k8s_client.list_job_for_all_namespaces()
    for job in job_list.items:
        print(job.metadata.labels)
if __name__=="__main__":
   list_job()

```

执行结果为：

```python
D:\My_ENV\Python39\python.exe E:/Project/PycharmProject/k8s_api/k8s_restapi_demo.py
{'app': 'myjob'}

Process finished with exit code 0

```

2.创建job

将上面的yaml案例中的myjob更改为mydemo创建一个新的job。

```python
from kubernetes import  config,client
import yaml
def create_job():
    config.load_kube_config("config")
    f=yaml.safe_load(open("mydemo.yaml"))
    k8s_client=client.BatchV1Api()
    job_list=k8s_client.create_namespaced_job(body=f,namespace="default")
    print(job_list.metadata.labels)
if __name__=="__main__":
   create_job()

```

执行结果为：

```python
D:\My_ENV\Python39\python.exe E:/Project/PycharmProject/k8s_api/k8s_restapi_demo.py
{'app': 'mydemo'}

Process finished with exit code 0

```

Client端命令执行结果为：

```shell
[root@master ~]# kubectl get job
NAME     COMPLETIONS   DURATION   AGE
mydemo   1/1           65s        65s
[root@master ~]#

```

3.删除job

删除job需要调用delete_namespaced_job方法，传递参数为job的名字。具体代码案例如下：

```python
from kubernetes import  config,client
def create_job():
    config.load_kube_config("config")
    k8s_client=client.BatchV1Api()
    job_response=k8s_client.delete_namespaced_job(namespace="default",name="mydemo")
    print(job_response)
if __name__=="__main__":
   create_job()

```

代码执行结果为：

```python
D:\My_ENV\Python39\python.exe E:/Project/PycharmProject/k8s_api/k8s_restapi_demo.py
{'api_version': 'batch/v1',
 'code': None,
 'details': None,
 'kind': 'Job',
 'message': None,
 'metadata': {'_continue': None,
              'remaining_item_count': None,
              'resource_version': '58208',
              'self_link': None},
 'reason': None,
 'status': "{'startTime': '2023-03-24T02:39:15Z', 'active': 1, "
           "'uncountedTerminatedPods': {}}"}

Process finished with exit code 0

```

Client命令执行结果为：

```shell
[root@master ~]# kubectl get job
No resources found in default namespace.
[root@master ~]#

```







三、管理service服务：使用Restful APIs方式管理service服务

1.创建service

创建service需要使用CoreV1Api方法来返回client对象，以及创建serivce的yaml文件，yaml文件内容如下：

```yaml
kind: Service
apiVersion: v1
metadata:
  name:  mysrvice
spec:
  selector:
    app:  mysrvice
  type:  NodePort
  ports:
  - name:  name-of-the-port
    port:  80
    targetPort:  8080

```

创建service代码案例如下：

```python
from kubernetes import  config,client
import yaml
def create_service():
    f=yaml.safe_load(open("service.yaml"))
    config.load_kube_config("config")
    k8s_client=client.CoreV1Api()
    service=k8s_client.create_namespaced_service(namespace="default",body=f)
    print(service.metadata.name+" "+service.metadata.uid)
if __name__=="__main__":
   create_service()

```

代码执行结果：

```python
D:\My_ENV\Python39\python.exe E:/Project/PycharmProject/k8s_api/create_demo.py
mysrvice 7acceed7-28ea-48df-b1df-fd84e6b24960

Process finished with exit code 0

```

Client命令执行结果为：

```shell
[root@master ~]# kubectl get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP        33d
mysrvice     NodePort    10.96.243.171   <none>        80:32573/TCP   49s
[root@master ~]#

```

2.查看service列表

查看service list需要使用list_service_for_all_namespaces方法，具体代码案例如下：

```python
from kubernetes import  config,client
def service_cron_job():
    config.load_kube_config("config")
    k8s_client=client.CoreV1Api()
    service_list=k8s_client.list_service_for_all_namespaces()
    for service in  service_list.items:
        print(service.metadata.name)
if __name__=="__main__":
   service_cron_job()
```

代码执行结果为：

```python
D:\My_ENV\Python39\python.exe E:/Project/PycharmProject/k8s_api/list_demo.py
kubernetes
mysrvice
kube-dns
dashboard-metrics-scraper
kubernetes-dashboard

Process finished with exit code 0
```

Client命令执行结果为：

```shell
[root@master ~]# kubectl get svc -A
NAMESPACE              NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
default                kubernetes                  ClusterIP   10.96.0.1       <none>        443/TCP                  33d
default                mysrvice                    NodePort    10.96.243.171   <none>        80:32573/TCP             3m50s
kube-system            kube-dns                    ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP,9153/TCP   33d
kubernetes-dashboard   dashboard-metrics-scraper   ClusterIP   10.96.78.88     <none>        8000/TCP                 33d
kubernetes-dashboard   kubernetes-dashboard        NodePort    10.96.114.121   <none>        443:32040/TCP            33d

```

3.删除service

删除service需要使用delete_namespaced_service方法，具体代码案例如下：

```python
from kubernetes import  config,client
def delete_service():
    config.load_kube_config("config")
    k8s_client=client.CoreV1Api()
    service=k8s_client.delete_namespaced_service(name="mysrvice",namespace="default")
    print(service)
if __name__=="__main__":
   delete_service()
```

代码执行结果为：

```python
D:\My_ENV\Python39\python.exe E:/Project/PycharmProject/k8s_api/delete_demo.py
{'api_version': 'v1',
 'kind': 'Service',
 'metadata': {'annotations': None,
              'creation_timestamp': datetime.datetime(2023, 3, 24, 9, 57, 38, tzinfo=tzutc()),
              'deletion_grace_period_seconds': None,
              'deletion_timestamp': None,
              'finalizers': None,
              'generate_name': None,
              'generation': None,
              'labels': None,
              'managed_fields': [{'api_version': 'v1',
                                  'fields_type': 'FieldsV1',
                                  'fields_v1': {'f:spec': {'f:externalTrafficPolicy': {},
                                                           'f:internalTrafficPolicy': {},
                                                           'f:ports': {'.': {},
                                                                       'k:{"port":80,"protocol":"TCP"}': {'.': {},
                                                                                                          'f:name': {},
                                                                                                          'f:port': {},
                                                                                                          'f:protocol': {},
                                                                                                          'f:targetPort': {}}},
                                                           'f:selector': {},
                                                           'f:sessionAffinity': {},
                                                           'f:type': {}}},
                                  'manager': 'OpenAPI-Generator',
                                  'operation': 'Update',
                                  'subresource': None,
                                  'time': datetime.datetime(2023, 3, 24, 9, 57, 38, tzinfo=tzutc())}],
              'name': 'mysrvice',
              'namespace': 'default',
              'owner_references': None,
              'resource_version': '66376',
              'self_link': None,
              'uid': '7acceed7-28ea-48df-b1df-fd84e6b24960'},
 'spec': {'allocate_load_balancer_node_ports': None,
          'cluster_i_ps': ['10.96.243.171'],
          'cluster_ip': '10.96.243.171',
          'external_i_ps': None,
          'external_name': None,
          'external_traffic_policy': 'Cluster',
          'health_check_node_port': None,
          'internal_traffic_policy': 'Cluster',
          'ip_families': ['IPv4'],
          'ip_family_policy': 'SingleStack',
          'load_balancer_class': None,
          'load_balancer_ip': None,
          'load_balancer_source_ranges': None,
          'ports': [{'app_protocol': None,
                     'name': 'name-of-the-port',
                     'node_port': 32573,
                     'port': 80,
                     'protocol': 'TCP',
                     'target_port': 8080}],
          'publish_not_ready_addresses': None,
          'selector': {'app': 'mysrvice'},
          'session_affinity': 'None',
          'session_affinity_config': None,
          'type': 'NodePort'},
 'status': {'conditions': None, 'load_balancer': {'ingress': None}}}

Process finished with exit code 0
```

Client命令执行结果为：

```shell
[root@master ~]# kubectl get svc -A
NAMESPACE              NAME                        TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
default                kubernetes                  ClusterIP   10.96.0.1       <none>        443/TCP                  33d
kube-system            kube-dns                    ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP,9153/TCP   33d
kubernetes-dashboard   dashboard-metrics-scraper   ClusterIP   10.96.78.88     <none>        8000/TCP                 33d
kubernetes-dashboard   kubernetes-dashboard        NodePort    10.96.114.121   <none>        443:32040/TCP            33d
```

Mysrvice服务已经删除。







四、管理Deployment资源：使用SDK方式管理Deployment服务

1.创建Deployment资源

创建deployment使用create_namespaced_deployment方法，同时需要相应的yaml文件。

创建Deployment资源Yaml文件内容如下：

```yaml
# https://kubernetes.io/docs/concepts/workloads/controllers/deployment/
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myjob
  namespace: default
  labels:
    app: myjob
spec:
  selector:
    matchLabels:
      app: myjob
  replicas: 1
  template:
    metadata:
      labels:
        app: myjob
    spec:
      containers:
      - name: myjob
        image: nginx:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 80
          name: myjob
      restartPolicy: Always
---
```

案例代码：

```python
from kubernetes import  config,client
import yaml
def create_deyloyment():
    f=yaml.safe_load(open("deployment.yaml"))
    config.load_kube_config("config")
    k8s_client=client.AppsV1Api()
    dep_list=k8s_client.create_namespaced_deployment(body=f,namespace="default")
    print(dep_list.status)
if __name__=="__main__":
   create_deyloyment()
```

Client客户端执行结果为：

```shell
[root@master ~]# kubectl get deploy
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
myjob   1/1     1            1           2m11s
[root@master ~]#
```

2.查看Deployment资源列表

操作deployment需要使用client.AppsV1Api()方法去获取k8s返回结果进行操作而不是CoreV1Api（）方法，具体代码案例如下：

```python
from kubernetes import  config,client
def list_deyloyment():
    config.load_kube_config("config")
    k8s_client=client.AppsV1Api()
    dep_list=k8s_client.list_deployment_for_all_namespaces()
    for deploy in dep_list.items:
        print(deploy.metadata.name)
if __name__=="__main__":
    list_deyloyment()
```

代码执行结果为：

```python
D:\My_ENV\Python39\python.exe E:\Project\PycharmProject\k8s_api\k8s_restapi_demo.py 
coredns
dashboard-metrics-scraper
kubernetes-dashboard
Process finished with exit code 0
```

3.删除Deployment资源

删除deployment使用delete_namespaced_deployment方法，具体案例如下：

```python
from kubernetes import  config,client
import yaml
def delete_deyloyment():
    f=yaml.safe_load(open("deployment.yaml"))
    config.load_kube_config("config")
    k8s_client=client.AppsV1Api()
    dep_list=k8s_client.delete_namespaced_deployment(name="myjob",namespace="default")
    print(dep_list.status)
if __name__=="__main__":
   delete_deyloyment()
```

执行结果：

```python
D:\My_ENV\Python39\python.exe E:\Project\PycharmProject\k8s_api\k8s_restapi_demo.py 
Success

Process finished with exit code 0
```

Client命令执行结果为：

```shell
[root@master ~]# kubectl get deploy
No resources found in default namespace.
[root@master ~]#
```

