# Fix a Broken Kubernetes Cluster

Here is the architecture for Challenge 2.

![Architecture](https://github.com/thawzinmyo/The-Hackathon-Challenge-DevOps/blob/master/image/Fix_a_Broken_Kubernetes_Cluster_Architecture.png)


---
---
Question on "controlplane" icon.
- Master node: coredns deployment has image: 'k8s.gcr.io/coredns/coredns:v1.8.6'
- Fix kube-apiserver. Make sure its running and healthy.
- kubeconfig = /root/.kube/config, User = 'kubernetes-admin' Cluster: Server Port = '6443'

There is an broken kubernetes cluster when I check it. I can't not verify nodes, pods. I spent a lot of time on this challenge 2. I think that I tried over 10 hours to be healthy back of API-Server. I asked social friends about that but they can't reply details back because of busy. They give me some ideas. I tried but not success. Finally, I got the crazy idea and I follow my trust. Bomb!!! I solved it.
Here is the sample issue.
```bash
$  kubectl get node
   The connection to the server controlplane:6433 was refused - did you specify the right host or port?
$  kubectl get pod
   The connection to the server controlplane:6433 was refused - did you specify the right host or port?
```
According to the question, Server Port have to be 6443. Error is showing "6433". So, let's check in ".kube/config"
```bash
$  cat .kube/config
   apiVersion: v1
   clusters:
   - cluster:
     certificate-authority-data: LS0tLS1CRUd....
     server: https://controlplane:6433
   name: kubernetes
```
Here, the controlplane have to be "6443". Not "6433".
Edit and save it.
Next

I also check the containers under namespace "kube-system". Is it running or stop?
```bash
$ docker ps | grep apiserver
  5187971b0865        k8s.gcr.io/pause:3.6   "/pause"                 20 minutes ago      Up 20 minutes                           k8s_POD_kube-apiserver-controlplane_kube-system_45b2867617046c1308cb3f7e22c02c27_0
```
Container is running.


I read some blog to check the log of containers and pods. I follow it /var/log/pods/related api-server-logs or /var/log/containers/related api-server-logs
```bash
$ cat /var/log/pods/kube-system_kube-apiserver-controlplane_45b2867617046c1308cb3f7e22c02c27/kube-apiserver/9.log 
{"log":"I0316 03:29:16.055909       1 server.go:565] external host was not specified, using 10.59.47.12\n","stream":"stderr","time":"2022-03-16T03:29:16.056199697Z"}
{"log":"I0316 03:29:16.056537       1 server.go:172] Version: v1.23.0\n","stream":"stderr","time":"2022-03-16T03:29:16.056696055Z"}
{"log":"E0316 03:29:16.449049       1 run.go:120] \"command failed\" err=\"open /etc/kubernetes/pki/ca-authority.crt: no such file or directory\"\n","stream":"stderr","time":"2022-03-16T03:29:16.449389882Z"}
```
Here is issue hint "/etc/kubernetes/pki/ca-authority.crt".

```bash
$ cat /var/log/containers/kube-apiserver....
Log is the same above log. Check yourself.
```

I saw the log about **ca-authrority.crt**. It may be hint. Finally, I definitely know that it is api server issue because the api-server is already mentioned in question and the log is also showing related api-server. So, I decided to be compare the issue api-server config under /etc/kubernetes/manifest/kube-apiserver with the another healthy api-server.

Solution:

| Issue API-Server                       | Healthy API-Server              |
| ---                                    | ---                             |
| containers:                            | containers:                     |
| - command:                             | - command:                      |
|   - kube-apiserver                     |   - kube-apiserver              |
|   - --advertise-address=192.168.121.162|   - --advertise-address=10.1.240.6 |
|   - --allow-privileged=true            |   - --allow-privileged=true     |
|   - --authorization-mode=Node,RBAC     |   - --authorization-mode=Node,RBAC
|   - --client-ca-file=/etc/kubernetes/pki/ca-authority.crt | - --client-ca-file=/etc/kubernetes/pki/ca.crt |
|   - --enable-admission-plugins=NodeRestriction  | - --enable-admission-plugins=NodeRestriction |

I edit --client-ca-file=/etc/kubernetes/pki/ca.crt. Then I wait 2 minutes. Try it again by checking nodes
```bash
$ kubectl get nodes
  NAME           STATUS                     ROLES                  AGE     VERSION
  controlplane   Ready                      control-plane,master   4h48m   v1.23.0
  node01         Ready,SchedulingDisabled   <none>                 4h47m   v1.23.
```
Hmmmmmmm. I can verify it.

API-Server POD is healthy under "kube-system"
<details>
<summary>API-Server POD</summary>
<p>

```bash
$ kubectl get pod --namespace kube-system
  NAME                                   READY   STATUS             RESTARTS        AGE
  coredns-64897985d-p2tnq                1/1     Running            0               4h49m
  coredns-7d8f8bf74f-n8fnq               0/1     ImagePullBackOff   0               33m
  coredns-7d8f8bf74f-p66dr               0/1     ImagePullBackOff   0               33m
  etcd-controlplane                      1/1     Running            0               4h49m
  kube-apiserver-controlplane            1/1     Running            0               4h49m
```
</p>
</details>

There is an another issue in coredns-pod. "ImagePullBackOff". I think that it may be image error. So, I check the deployment under "kube-system"

```bash
$ kubectl --namespace kube-system get deployment
  NAME      READY   UP-TO-DATE   AVAILABLE   AGE
  coredns   1/2     2            1           4h54m
```
```bash
$ kubectl --namespace kube-system describe    deployment coredns | grep -i image
    Image:       k8s.gcr.io/kubedns:1.3.1
```
Here the image is not the given image. So, edit the deployment with the given image. And check it again.
```bash
$ kubectl --namespace kube-system edit deployment coredns
  deployment.apps/coredns edited
```
```bash
$ kubectl --namespace kube-system describe deployment coredns | grep -i image
    Image:       k8s.gcr.io/coredns/coredns:v1.8.6
```
Now, coredns pods under "kube-system" are running.
<details>
<summary>Coredns PODs</summary>
<p>

```bash
$ kubectl --namespace kube-system get pods
  NAME                                   READY   STATUS    RESTARTS       AGE
  coredns-64897985d-hkcsr                1/1     Running   0              73s
  coredns-64897985d-p2tnq                1/1     Running   0              5h6m
```
</p>
</details>

Okay, API-SERVER and COREDNS PODS are running correctly. Let's move to "WORKER NODE" named "node01".

---
---

Question on "node01" icon.

- node01 is ready and can schedule pods?

Solution:

Let's verify "WORKER NODE" named "node01" is "Ready" or any issue.

```bash
$ kubectl get nodes
  NAME           STATUS                     ROLES                  AGE     VERSION
  controlplane   Ready                      control-plane,master   4h48m   v1.23.0
  node01         Ready,SchedulingDisabled   <none>                 4h47m   v1.23.0
```

Read this about ["SchedulingDisabled"](https://kubernetes.io/docs/concepts/architecture/nodes/). 

<details>
<summary>After Uncordon</summary>
<p>

```bash
$ kubectl uncordon node01
  node/node01 uncordoned

```
</p>
</details>

Verify again

```bash
$ kubectl get nodes 
  NAME           STATUS   ROLES                  AGE   VERSION
  controlplane   Ready    control-plane,master   78m   v1.23.0
  node01         Ready    <none>                 77m   v1.23.0
```
Now, kube-scheduler can assign "pods" to the available nodes.

"WORKER NODE" - Node01 can available to host containers.

---
---

Question on "/web" icon.

- Copy all images from the directory '/media' on the controlplane node to '/web' directory on node01

Solution:

Verify "/media"

```bash
$ ls -al /media/
  total 192
  drwxr-xr-x 1 root root  4096 Mar 16 04:41 .
  drwxr-xr-x 1 root root  4096 Mar 16 03:26 ..
  -rw-r--r-- 1 root root 58450 Mar  1 22:06 kodekloud-cka.png
  -rw-r--r-- 1 root root 59809 Mar  1 22:06 kodekloud-ckad.png
  -rw-r--r-- 1 root root 62223 Mar  1 22:06 kodekloud-cks.png
```
There is an some images. Okay let's verify the 'node01' IP to copy using scp.
```bash
$ kubectl get nodes -o wide
  NAME           STATUS   ROLES                  AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION   CONTAINER-RUNTIME
  controlplane   Ready    control-plane,master   81m   v1.23.0   10.4.32.9     <none>        Ubuntu 18.04.6 LTS   5.4.0-1067-gcp   docker://19.3.0
  node01         Ready    <none>                 80m   v1.23.0   10.4.32.12    <none>        Ubuntu 18.04.6 LTS   5.4.0-1067-gcp   docker://19.3.0
```

```bash
$  scp /media/* 10.4.32.12:/web/
The authenticity of host '10.4.32.12 (10.4.32.12)' can't be established.
ECDSA key fingerprint is SHA256:UxnO68aDpVUoteHvFXPjBHGlVpqgMo68/JUmgV7sGZg.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '10.4.32.12' (ECDSA) to the list of known hosts.
kodekloud-cka.png                                                                                                  100%   57KB  39.8MB/s   00:00    
kodekloud-ckad.png                                                                                                 100%   58KB  52.0MB/s   00:00    
kodekloud-cks.png                                                                                                  100%   61KB  52.7MB/s   00:00

```
Verify in "node01". Enter "node01" using SSH and verify under "/web".
```bash
node01 ~ ➜  ls -al /web/
            total 192
            drwxr-xr-x 2 root root  4096 Mar 16 04:49 .
            drwxr-xr-x 1 root root  4096 Mar 16 04:41 ..
            -rw-r--r-- 1 root root 58450 Mar 16 04:49 kodekloud-cka.png
            -rw-r--r-- 1 root root 59809 Mar 16 04:49 kodekloud-ckad.png
            -rw-r--r-- 1 root root 62223 Mar 16 04:49 kodekloud-cks.png
```
yes, images are copied successfully.


---
---

Question on "data-pv" icon.

- Create new PersistentVolume = 'data-pv'
- PersistentVolume = data-pv, accessModes = 'ReadWriteMany'
- PersistentVolume = data-pv, hostPath = '/web'
- PersistentVolume = data-pv, storage = '1Gi'

Solution:

Let's create "data-pv" and "data-pvc" and verify it.

<details>
<summary>Data-PV YAML Manifest File</summary>
<p>

```bash
apiVersion: v1
kind: PersistentVolume
metadata:
  name: data-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  hostPath: 
     path: /web
```
</p>
</details>

```bash
$ kubectl create -f pv.yaml 
  persistentvolume/data-pv created
```
Verify
```bash
$ kubectl get persistentvolume
  NAME      CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM              STORAGECLASS   REASON   AGE
  data-pv   1Gi        RWX            Retain           Bound    default/data-pvc                           65s
```


---
---

Question on "data-pvc" icon.

- Create new PersistentVolumeClaim = 'data-pvc'
- PersistentVolume = 'data-pvc', accessModes = 'ReadWriteMany'
- PersistentVolume = 'data-pvc', storage request = '1Gi'
- PersistentVolume = 'data-pvc', volumeName = 'data-pv'


Solution:

Creat "data-pvc" manifest file.

<details>
<summary>Data-PVC YAML Manifest File</summary>
<p>

```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```
</p>
</details>

```bash
$ kubectl create -f pvc.yaml
  persistentvolumeclaim/data-pvc created
```

Verify "data-pvc". Is it bound with "data-pv"?

```bash
$ kubectl get persistentvolumeclaim
  NAME       STATUS   VOLUME    CAPACITY   ACCESS MODES   STORAGECLASS   AGE
  data-pvc   Bound    data-pv   1Gi        RWX                           2m42s
```

Yes, perfect!

---
---

Question on "gop-file-server" icon.

- Create a pod for fileserver, name: 'gop-fileserver'
- pod: gop-fileserver image: 'kodekloud/fileserver'
- pod: gop-fileserver mountPath: '/web'
- pod: gop-fileserver volumeMount name: 'data-store'
- pod: gop-fileserver persistent volume name: data-store
- pod: gop-fileserver persistent volume claim used: 'data-pvc'

Solution:

<details>
<summary>Gop-File-Server YAML Manifest File</summary>
<p>

```bash
apiVersion: v1
kind: Pod
metadata:
  name: gop-fileserver
  labels:
    run: gop-fileserver
spec:
  volumes:
  - name: data-store
    persistentVolumeClaim:
      claimName: data-pvc
  containers:
  - name: gop-fileserver
    image: kodekloud/fileserver
    volumeMounts:
    - name: data-store
      mountPath: /web
```
</p>
</details>

```bash
$ kubectl create -f pod.yaml 
  pod/gop-fileserver created
```

Is POD named "gop-fileserver" running? Verify

```bash
$ kubectl get pod 
  NAME             READY   STATUS    RESTARTS   AGE
  gop-fileserver   1/1     Running   0          12s
```

Pod is running..........

---
---

Question on "gop-fs-service" icon.

- New Service, name: 'gop-fs-service'
- Service name: gop-fs-service, port: '8080'
- Service name: gop-fs-service, targetPort: '8080'

Solution:

Let's create service using imperative.

```bash
$ kubectl expose pod gop-fileserver --name gop-fs-service --port 8080 --target-port 8080
```
```bash
$ kubectl expose pod gop-fileserver --name gop-fs-service --port 8080 --target-port 8080
  service/gop-fs-service exposed
```


Verify it
```bash
$ kubectl get service
  NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
  gop-fs-service   ClusterIP   10.110.111.55   <none>        8080/TCP   7s
  kubernetes       ClusterIP   10.96.0.1       <none>        443/TCP    92m
```

Then We need to edit accoring to the architecture.
- Service name: gop-fs-service, NodePort: '31200'
  
Change Service Type to "NodePort" and add nodePort " 31200 " and verify again.

```bash
$ k edit service gop-fs-service 
  service/gop-fs-service edited
```

```bash
$ kubectl get service
  NAME             TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
  gop-fs-service   NodePort    10.110.111.55   <none>        8080:31200/TCP   9m39s
  kubernetes       ClusterIP   10.96.0.1       <none>        443/TCP          102m

```
<details>
<summary>NodePort Service Describe</summary>
<p>

```bash
kubectl describe service gop-fs-service 
Name:                     gop-fs-service
Namespace:                default
Labels:                   run=gop-fileserver
Annotations:              <none>
Selector:                 run=gop-fileserver
Type:                     NodePort
IP Family Policy:         SingleStack
IP Families:              IPv4
IP:                       10.110.111.55
IPs:                      10.110.111.55
Port:                     <unset>  8080/TCP
TargetPort:               8080/TCP
NodePort:                 <unset>  31200/TCP
Endpoints:                10.50.192.2:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```
</p>
</details>

Service is pretty work!

---
---

Final task, 

Question on "user-email" icon.

- Mandatory Step: Save your email address to a file called '/root/user_email.txt' on the controlplane node

Solution:

```bash
$ echo "txxxxxxx@gmail.com" > /root/user_email.txt
```
<details>
<summary>Output</summary>
<p>

```bash
$  cat /root/user_email.txt 
   txxxxxxx@gmail.com
```
</p>
</details>

---
---

Time to check "Check", it have to be "Complete".

Yes, it's complete.

![Challenge Complete!](https://github.com/thawzinmyo/The-Hackathon-Challenge-DevOps/blob/master/image/Fix_a_Broken_Kubernetes_Cluster_Complete.png)

Breath and take relax ................