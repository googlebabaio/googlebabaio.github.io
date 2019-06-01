

#### 1.列出环境内所有的pv 并以 name字段排序（使用kubectl自带排序功能）

```
kubectl get pv --sort-by=.metadata.name
```

#### 2.列出指定pod的日志中状态为Error的行，并记录在指定的文件上
```
kubectl logs <podname> | grep bash > /opt/KUCC000xxx/KUCC000xxx.txt
```

#### 3.列出k8s可用的节点，不包含不可调度的 和 NoReachable的节点，并把数字写入到文件里
```
#笨方法，人工数
kubectl get nodes

#CheatSheet方法，应该还能优化JSONPATH
JSONPATH='{range .items[*]}{@.metadata.name}:{range @.status.conditions[*]}{@.type}={@.status};{end}{end}' \
 && kubectl get nodes -o jsonpath="$JSONPATH" | grep "Ready=True"
```

参考:kubectl命令熟悉程度
https://kubernetes.io/docs/reference/kubectl/cheatsheet/

#### 4.创建一个pod名称为nginx，并将其调度到节点为 disk=stat上
```
kubectl run nginx --image=nginx --restart=Never --dry-run > 4.yaml
#增加对应参数
vi 4.yaml
kubectl apply -f 4.yaml
```

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    env: test
spec:
  containers:
  - name: nginx
    image: nginx
    imagePullPolicy: IfNotPresent
  nodeSelector:
    disktype: ssd
```
参考:pod的调度
https://kubernetes.io/docs/concepts/configuration/assign-pod-node/

#### 5.提供一个pod的yaml，要求添加Init Container，Init Container的作用是创建一个空文件，pod的Containers判断文件是否存在，不存在则退出
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: apline
    image: nginx
    command: ['sh', '-c', 'if [目录下有work文件];then sleep 3600; else exit; fi;']
###增加init Container####
initContainers:
 - name: init
    image: busybox
    command: ['sh', '-c', 'touch /目录/work;']
```

参考:init容器
https://kubernetes.io/docs/concepts/workloads/pods/init-containers/

#### 6.指定在命名空间内创建一个pod名称为test，内含四个指定的镜像nginx、redis、memcached、busybox

```
kubectl run test --image=nginx --image=redis --image=memcached --image=buxybox --restart=Never -n <namespace>

```

#### 7.创建一个pod名称为test，镜像为nginx，Volume名称cache-volume为挂在在/data目录下，且Volume是non-Persistent的
```
apiVersion: v1
kind: Pod
metadata:
  name: test
spec:
  containers:
  - image: nginx
    name: test
    volumeMounts:
    - mountPath: /data
      name: cache-volume
  volumes:
  - name: cache-volume
    emptyDir: {}
```

参考:volumn
https://kubernetes.io/docs/concepts/storage/volumes/#emptydir

#### 8.列出Service名为test下的pod 并找出使用CPU使用率最高的一个，将pod名称写入文件中
```
#使用-o wide 获取service test的SELECTOR
kubectl get svc test -o wide
##获取结果我就随便造了
NAME              TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE       SELECTOR
test   ClusterIP   None         <none>        3306/TCP   50d       app=wordpress,tier=mysql

#获取对应SELECTOR的pod使用率，找到最大那个写入文件中
kubectl top test -l 'app=wordpress,tier=mysql'
```

参考:获取service selector，kubectl top监控pod资源
https://kubernetes.io/docs/tasks/debug-application-cluster/resource-usage-monitoring


#### 9.创建一个Pod名称为nginx-app，镜像为nginx，并根据pod创建名为nginx-app的Service，type为NodePort
```
kubectl run nginx-app --image=nginx --restart=Never --port=80
kubectl create svc nodeport nginx-app --tcp=80:80 --dry-run -o yaml > 9.yaml
#修改yaml，保证selector name=nginx-app
vi 9.yaml
```

```
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    app: nginx-app
  name: nginx-app
spec:
  ports:
  - name: 80-80
    port: 80
    protocol: TCP
    targetPort: 80
  selector:
#注意要和pod对应
    name: nginx-app
  type: NodePort
```

参考:service
https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types

#### 10.创建一个nginx的Workload，保证其在每个节点上运行，注意不要覆盖节点原有的Tolerations
```
#先创建一个deployment的yaml模板
kubectl run nginx --image=nginx --dry-run -o yaml > 10.yaml
#将yaml改成DaemonSet
vi 10.yaml
```

```
#修改apiVersion和kind
#apiVersion: extensions/v1beta1
#kind: Deployment
apiVersion:apps/v1
kind: DaemonSet
metadata:
  creationTimestamp: null
  labels:
    run: nginx
  name: nginx
spec:
#去掉replicas
# replicas: 1
  selector:
    matchLabels:
      run: nginx
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        resources: {}
status: {}
```

参考:daemonset
https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/


#### 11.将deployment为nginx-app的副本数从1变成4
```
#方法1
kubectl scale  --replicas=4 deployment nginx-app
#方法2，使用edit命令将replicas改成4
kubectl edit deploy nginx-app
```

参考:deployment的Scaling，搜索Scaling
https://kubernetes.io/docs/tasks/run-application/run-stateless-application-deployment/#scaling-the-application-by-increasing-the-replica-count

#### 12.创建nginx-app的deployment ，使用镜像为nginx:1.11.0-alpine ,修改镜像为1.11.3-alpine，并记录升级，再使用回滚，将镜像回滚至nginx:1.11.0-alpine
```
# 参考应该少了一个record
kubectl run nginx-app --image=nginx:1.11.0-alpine
kubectl set image deployment nginx-app --image=nginx:1.11.3-alpine
kubectl rollout undo deployment nginx-app
kubectl rollout status -w deployment nginx-app
```

参考:资源的更新
https://kubernetes.io/docs/reference/kubectl/cheatsheet/#updating-resources

#### 13.根据已有的一个nginx的pod、创建名为nginx的svc、并使用nslookup查找出service dns记录，pod的dns记录并分别写入到指定的文件中

```
#创建一个服务
kubectl create svc nodeport nginx --tcp=80:80
#创建一个指定版本的busybox，用于执行nslookup
kubectl create -f https://k8s.io/examples/admin/dns/busybox.yaml
#将svc的dns记录写入文件中
kubectl exec -ti busybox -- nslookup nginx > 指定文件
#获取pod的ip地址
kubectl get pod nginx -o yaml
#将获取的pod ip地址使用nslookup查找dns记录
kubectl exec -ti busybox -- nslookup <Pod ip>

```

参考:网络相关，DNS解析
https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/

#### 14.创建Secret 名为mysecret，内含有password字段，值为bob，然后 在pod1里 使用ENV进行调用，Pod2里使用Volume挂载在/data 下
```
#将密码值使用base64加密,记录在Notepad里
echo -n 'bob' | base64
```

14.secret.yaml
```
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  password: Ym9i
```

14.pod1.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: pod1
spec:
  containers:
  - name: mypod
    image: nginx
    volumeMounts:
    - name: mysecret
      mountPath: "/data"
      readOnly: true
  volumes:
  - name: mysecret
    secret:
      secretName: mysecret
```

14.pod2.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: pod2
spec:
  containers:
  - name: mycontainer
    image: redis
    env:
      - name: SECRET_PASSWORD
        valueFrom:
          secretKeyRef:
            name: mysecret
            key: password
```
参考:secret
https://kubernetes.io/docs/concepts/configuration/secret/

#### 15.使node1节点不可调度，并重新分配该节点上的pod
```
#直接drain会出错，需要添加--ignore-daemonsets --delete-local-data参数
kubectl drain node node1  --ignore-daemonsets --delete-local-data
```

参考：[Safely Drain a Node while Respecting Application SLOs]: （https://kubernetes.io/docs/tasks/administer-cluster/safely-drain-node/)


#### 16.使用etcd 备份功能备份etcd（提供enpoints，ca、cert、key）
```
ETCDCTL_API=3 etcdctl --endpoints https://127.0.0.1:2379 \
--cacert=ca.pem --cert=cert.pem --key=key.pem \
snapshot save snapshotdb
```

参考:etcd的集群的备份与恢复
https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster

#### 17.给出一个失联节点的集群，排查节点故障，要保证改动是永久的。
```
#查看集群状态
kubectl get nodes
#查看故障节点信息
kubectl describe node node1

#Message显示kubelet无法访问（记不清了）
#进入故障节点
ssh node1

#查看节点中的kubelet进程
ps -aux | grep kubelete
#没找到kubelet进程，查看kubelet服务状态
systemctl status kubelet.service
#kubelet服务没启动，启动服务并观察
systemctl start kubelet.service
#启动正常，enable服务
systemctl enable kubelet.service

#回到考试节点并查看状态
exit

kubectl get nodes #正常
```

参考:故障排查
https://kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/

#### 18.给出一个集群，排查出集群的故障
```
这道题没空做完。kubectl get node显示connection refuse，估计是apiserver的故障。
```

参考:故障排查
https://kubernetes.io/docs/tasks/debug-application-cluster/debug-cluster/

#### 19.给出一个节点，完善kubelet配置文件，要求使用systemd配置kubelet

#### 20.给出一个集群，将节点node1添加到集群中，并使用TLS bootstrapping
这道题没空做完，花费时间比较长，可惜了。

参考：TLS Bootstrapping
https://kubernetes.io/docs/reference/command-line-tools-reference/kubelet-tls-bootstrapping/

#### 21.创建一个pv，类型是hostPath，位于/data中，大小1G，模式ReadOnlyMany
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadOnlyMany
  hostPath:
    path: /data
```

参考:创建PV
https://kubernetes.io/docs/concepts/storage/persistent-volumes/
