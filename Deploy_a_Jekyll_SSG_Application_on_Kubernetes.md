# Deploy a Jekyll SSG Application on Kubernetes

Here is the architecture for implementing a Jekyll SSG.

![Architecture](https://github.com/thawzinmyo/The-Hackathon-Challenge-DevOps/blob/master/image/Deploy_a_Jekyll_SSG_Application_On_Kubernetes_Architecture.png)

When you press each icon in the architecture, will see the questions.

Let's start with easy task by entering mail address.

Question in "user_email" icon.
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


Quesiton in "martin" icon.
- Build user information for martin in the default kubeconfig file: User = martin , client-key = /root/martin.key and client-certificate = /root/martin.crt
- Create a new context called 'developer' in the default kubeconfig file with 'user = martin' and 'cluster = kubernetes'

Solution:

```bash
$  kubectl config current-context 
   kubernetes-admin@kubernetes
```
I also check in .kube/config . There is a default context. So, I need to add the given context information for "martin" user...
<details>
<summary>Output</summary>
<p>

```bash
- context:
    cluster: kubernetes
    user: martin
  name: developer
current-context: kubernetes-admin@kubernetes
kind: Config
preferences: {}
users:
- name: martin
  user:
    client-certificate: /root/martin.crt
    client-key: /root/martin.key

```
</p>
</details>

Done!!! Leave it that!

---
---

Question in "developer-role" icon.
- 'developer-role', should have all(*) permissions for services in development namespace
- 'developer-role', should have all permissions(*) for persistentvolumeclaims in development namespace
- 'developer-role', should have all(*) permissions for pods in development namespace

Solution:

Let's create "developer-role" using declrative command

```bash
$  cat /root/developer-role.yaml
```
<details>
<summary>Developer-Role YAML Manifest File</summary>
<p>

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: development
  name: developer-role
rules:
- apiGroups: [""]
  resources: ["services", "persistentvolumeclaims", "pods"]
  verbs: ["*"]
```
</p>
</details>

```bash
$  kubectl create -f developer-role.yaml 
   role.rbac.authorization.k8s.io/developer-role created
```

Verify the resource.
<details>
<summary>Output</summary>
<p>

```bash
$  kubectl get role --namespace development
   NAME             CREATED AT
   developer-role   2022-03-15T11:31:34Z
```
</p>
</details>


---
---
Question in "developer-rolebinding" icon.
- create rolebinding = developer-rolebinding, role= 'developer-role', namespace = development
- rolebinding = developer-rolebinding associated with user = 'martin'

Solution:

Let's create rolebinding manifest file.

```bash
$  cat /root/developer-rolebinding.yaml
```
<details>
<summary>Developer-Role Binding YAML Manifest File</summary>
<p>

```bash
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: developer-rolebinding
  namespace: development
subjects:
- kind: User
  name: martin # "name" is case sensitive
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role #this must be Role or ClusterRole
  name: developer-role # this must match the name of the Role or ClusterRole you wish to bind to
  apiGroup: rbac.authorization.k8s.io
```
</p>
</details>

```bash
 $  kubectl create -f developer-rolebinding.yaml
    rolebinding.rbac.authorization.k8s.io/developer-rolebinding created
```

Verify the resource that created.
<details>
<summary>Output</summary>
<p>

```bash
$  kubectl get rolebinding --namespace development
   NAME                    ROLE                  AGE
   developer-rolebinding   Role/developer-role   6m25s
```
</p>
</details>


---
---
Question in "jekyll-pv" icon.
- jekyll-site pv is already created. Inspect it before you create the pvc

Solution:

There is no configure for this task. It's already created. Let's inspect before create pvc.
<details>
<summary>Output</summary>
<p>

```bash
$  kubectl --namespace development get persistentvolume
   NAME          CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS    REASON   AGE
   jekyll-site   1Gi        RWX            Delete           Available           local-storage            23m
```

</p>
</details>

---
---

Question in "jekyll-pvc" icon.
- Storage Request: 1Gi
- Access modes: ReadWriteMany
- pvc name = jekyll-site, namespace development
- 'jekyll-site' PVC should be bound to the PersistentVolume called 'jekyll-site'.
  
Solution:

Create manifest for PVC to bound with PV.
<details>
<summary>PVC YAML Manifest file</summary>
<p>

```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: jekyll-site
  namespace: development
spec:
  storageClassName: local-storage
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```
</p>
</details>

```bash
$  kubectl create -f pvc.yaml
persistentvolumeclaim/jekyll-site created
```

Let's verify the resource that created by checking "Output"

<details>
<summary>Output</summary>
</p>

```bash
$  kubectl --namespace development get persistentvolumeclaim
   NAME          STATUS   VOLUME        CAPACITY   ACCESS MODES   STORAGECLASS    AGE
   jekyll-site   Bound    jekyll-site   1Gi        RWX            local-storage   35s
```
</p>
</details>

PV and PVC are bounded. Let's move next task.

---
---

Question in "jekyll" icon.
- pod: 'jekyll' has an initContainer, name: 'copy-jekyll-site', image: 'kodekloud/jekyll'
- initContainer: 'copy-jekyll-site', command: [ "jekyll", "new", "/site" ] (command to run: jekyll new /site)
pod: 'jekyll', initContainer: 'copy-jekyll-site', mountPath = '/site'
- pod: 'jekyll', initContainer: 'copy-jekyll-site', volume name = 'site'
- pod: 'jekyll', container: 'jekyll', volume name = 'site'
- pod: 'jekyll', container: 'jekyll', mountPath = '/site'
- pod: 'jekyll', container: 'jekyll', image = 'kodekloud/jekyll-serve'
- pod: 'jekyll', uses volume called 'site' with pvc = 'jekyll-site'
- pod: 'jekyll' uses label 'run=jekyll'

Solution:

Let's create the Pod manifest file.
```bash
$  cat /etc/pod.yaml
```

<details>
<summary>Jekyll POD YAML Manifest File</summary>
<p>

```bash
apiVersion: v1
kind: Pod
metadata:
  name: jekyll
  namespace: development
  labels:
    run: jekyll
spec:
  volumes:
  - name: site
    persistentVolumeClaim:
      claimName: jekyll-site
  containers:
  - name: jekyll
    image: kodekloud/jekyll-serve 
    volumeMounts:
    - name: site
      mountPath: /site
  initContainers:
  - name: copy-jekyll-site
    image: kodekloud/jekyll
    command: ['bash', '-c', "jekyll new /site"]
    volumeMounts:
    - name: site
      mountPath: /site
```
</p>
</details>

```bash
$  kubectl create -f pod.yaml 
   pod/jekyll created
```

Let's verify the resource that created by checking "Output".
<details>
<summary>Running initContainer</summary>
<p>

```bash
$  kubectl --namespace development get pod
   NAME     READY   STATUS     RESTARTS   AGE
   jekyll   0/1     Init:0/1   0          13s
```
</p>
</details>

<details>
<summary>Running Container</summary>
<p>

```bash
$  kubectl --namespace development get pod
   NAME     READY   STATUS    RESTARTS   AGE
   jekyll   1/1     Running   0          2m55s
```
</p>
</details>

---
---

Now, Let's create the NodePort service for that POD named "Jekyll"

Question in "jekyll-node-service"
- Service 'jekyll' uses targetPort: '4000' , namespace: 'development'
Service 'jekyll' uses Port: '8080' , namespace: 'development'
- Service 'jekyll' uses NodePort: '30097' , namespace: 'development'

Solution:

Let's create the NodePort Service type using imperative command then edit "jekyll-svc.yaml" using VI.

```bash
$ kubectl --namespace development expose pod jekyll --type NodePort --name jekyll --port 8080 --target-port 4000 --dry-run=client -o yaml > jekyll-svc.yaml
```
Edit the "jekyll-svc.yaml" by adding "nodePort" using VI.
<details>
<summary>Output</summary>
<p>

```bash
apiVersion: v1
kind: Service
metadata:
  creationTimestamp: null
  labels:
    run: jekyll
  name: jekyll
  namespace: development
spec:
  ports:
  - port: 8080
    protocol: TCP
    targetPort: 4000
    nodePort: 30097
  selector:
    run: jekyll
  type: NodePort
status:
  loadBalancer: {}
```
</p>
</details>

```bash
$  kubectl --namespace development get svc 
   NAME     TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
   jekyll   NodePort   10.105.195.58   <none>        8080:30097/TCP   12s
```
  
Let's inspect the details of NodePort service.
<details>
<summary>Inspecting NodePort Service named "jekyll"
</summary>
<p>

```bash
$ kubectl --namespace development describe service jekyll
   Name:                     jekyll
   Namespace:                development
   Labels:                   run=jekyll
   Annotations:              <none>
   Selector:                 run=jekyll
   Type:                     NodePort
   IP Family Policy:         SingleStack
   IP Families:              IPv4
   IP:                       10.105.195.58
   IPs:                      10.105.195.58
   Port:                     <unset>  8080/TCP
   TargetPort:               4000/TCP
   NodePort:                 <unset>  30097/TCP
   Endpoints:                10.50.192.2:4000
   Session Affinity:         None
   External Traffic Policy:  Cluster 
   Events:                   <none>
```
</p>
</details>

Service is working.

---
---

Now, 
Question in "kube-config" icon.
- set context 'developer' with user = 'martin' and cluster = 'kubernetes' as the current context.

I created for "martin" user with cluster named "kubernetes". Now, according to the given question, switch the context using the following command.
```bash
$ kubectl config use-context developer
```

Let's verify the output using "current-context" flag.
```bash
$ kubectl config current-context
  developer
```

---
---
Now, press **Check** button on the left side.

Bomb!!! You will see "Complete" 

![Challenge Complete!](https://github.com/thawzinmyo/The-Hackathon-Challenge-DevOps/blob/master/image/Deploy_a_Jekyll_SSG_Application_On_Kubernetes_Architecture.png)

Let's break for a while.