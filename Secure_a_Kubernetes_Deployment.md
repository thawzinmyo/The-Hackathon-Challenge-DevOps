# Secure a Kubernetes Deployment

Let's go to the final task. Here is the architecture!

![Architecture](https://github.com/thawzinmyo/The-Hackathon-Challenge-DevOps/blob/master/image/Secure_a_Kubernetes_Deployment_Architecture.png)

let's enter our mail address with the same prvicious task.

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

Question on "pv" icon.

- A persistentVolume called 'alpha-pv' has already been created. Do not modify it and inspect the parameters used to create it.

Solution:

Quick verify persistentvolume named "alpha-pv".
```bash
$ kubectl get persistentvolume --namespace alpha
  NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS    REASON   AGE
  alpha-pv   1Gi        RWX            Delete           Available           local-storage            2m36s
```
---
---

Quesiton on "alpha-pvc" icon.

- 'alpha-pvc' should be bound to 'alpha-pv'. Delete and Re-create it if necessary.

Solution:

Okay, after quick verify persistentvolume, memorize some pv's information to create persistentvolumeclaim named "alpha-pvc". Let's create "YAML Manifest File for PVC".

<details>
<summary>PVC YAML Manifest file</summary>
<p>

```bash
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: alpha-pvc
  namespace: alpha
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

Let's verify! Is it bound? If it's not bound or success, delete that exiting pvc and create from the PVC manifest. It will complete successfully. 
```bash
$ kubectl --namespace alpha get persistentvolumeclaim
  NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS    AGE
  alpha-pvc   Bound    alpha-pv   1Gi        RWX            local-storage   30s
```

Yes, it's bound successfully.

---
---

Question on "apparmor" icon

- Move the AppArmor profile '/root/usr.sbin.nginx' to '/etc/apparmor.d/usr.sbin.nginx' on the controlplane node
- Load the 'AppArmor` profile called 'custom-nginx' and ensure it is enforced.

[AppArmor](https://kubernetes.io/docs/tutorials/security/apparmor/) is under covered Certified Kubernetes Security course. I am not that level. So, I don't even know what is this. After 1 hour searching and watching, I understand it - AppArmor is a Linux kernel security module that supplements the standard Linux user and group based permissions to confine programs to a limited set of resources. AppArmor can be configured for any application to reduce its potential attack surface and provide greater in-depth defense

Solution:

Okay, let's start with the given question. Let's move the AppArmor profile.
```bash
$ mv /root/usr.sbin.nginx /etc/apparmor.d/
```
Verify that task.
```bash
$ ls -al /etc/apparmor.d/ | grep -i nginx
  -rw-rw-r--   1 root root  1307 Mar 14 19:55 usr.sbin.nginx
```

Before load the "custom-nginx", let's check with "apparmor_status"

<details>
<summary>Apparmor_Status</summary>
<p>

```bash
$ apparmor_status 
  apparmor module is loaded.
  56 profiles are loaded.
  19 profiles are in enforce mode.
     /sbin/dhclient
     /usr/bin/lxc-start
     /usr/bin/man
     /usr/lib/NetworkManager/nm-dhcp-client.action
     /usr/lib/NetworkManager/nm-dhcp-helper
     /usr/lib/chromium-browser/chromium-browser//browser_java
     /usr/lib/chromium-browser/chromium-browser//browser_openjdk
     /usr/lib/chromium-browser/chromium-browser//sanitized_helper
     /usr/lib/connman/scripts/dhclient-script
     /usr/lib/snapd/snap-confine
     /usr/lib/snapd/snap-confine//mount-namespace-capture-helper
     /usr/sbin/tcpdump
     docker-default
     lxc-container-default
     lxc-container-default-cgns
     lxc-container-default-with-mounting
     lxc-container-default-with-nesting
     man_filter
     man_groff
```
</p>
</details>

There is no "custom-nginx" yet.

Okay, time to load "custom-nginx".

```bash
$ apparmor_parser /etc/apparmor.d/usr.sbin.nginx
```
What's about now?

<details>
<summary>After Load Apparmor "custom-nginx"</summary>
<p>

```bash
$ apparmor_status 
  apparmor module is loaded.
  57 profiles are loaded.
  20 profiles are in enforce mode.
     /sbin/dhclient
     /usr/bin/lxc-start
     /usr/bin/man
     /usr/lib/NetworkManager/nm-dhcp-client.action
     /usr/lib/NetworkManager/nm-dhcp-helper
     /usr/lib/chromium-browser/chromium-browser//browser_java
     /usr/lib/chromium-browser/chromium-browser//browser_openjdk
     /usr/lib/chromium-browser/chromium-browser//sanitized_helper
     /usr/lib/connman/scripts/dhclient-script
     /usr/lib/snapd/snap-confine
     /usr/lib/snapd/snap-confine//mount-namespace-capture-helper
     /usr/sbin/tcpdump
     custom-nginx
     docker-default
```
</p>
</details>

Here is the "custom-nginx". That apparmor profile will use in pod's manifest file later.


---
---

Question on "images" icon.

- Permitted images are: 'nginx:alpine', 'bitnami/nginx', 'nginx:1.13', 'nginx:1.17', 'nginx:1.16'and 'nginx:1.14'. Use 'trivy' to find the image with the least number of 'CRITICAL' vulnerabilities.

Solution:

So, how to find the image with the least number of vulnerabilities? Okay, there is a hint. What is [**trivy**](https://aquasecurity.github.io/trivy/v0.24.2/) in the question? After two hours searching and watching about trivy? I am understand a quite. **Trivy** is a simple and comprehensive vulnerability/misconfiguration scanner for containers and other artifacts.

 There are images in the given question. So, I need to try one by one with trivy that scan the images with least the vulnerability. I will not show the result. Its output is so long. I use this command
 ```bash
$ trivy image --serverity CRITICAL <image>
 ```

 <details>
 <summary>Scanning for "nginx:1.17"</summary>
 <p>
 
 ```bash
 $ trivy image --severity "CRITICAL" nginx:1.17
   2022-03-18T03:06:35.340Z        INFO    Detecting Debian vulnerabilities...
   2022-03-18T03:06:35.418Z        INFO    Trivy skips scanning programming language libraries because no supported file was detected

   nginx:1.17 (debian 10.4)
   ========================
   Total: 37 (CRITICAL: 37)

 ```
 </p>
 </details>

<details>
<summary>Scanning for "nginx:alpine"</summary>
<p>

```bash
$  trivy image --severity "CRITICAL" nginx:alpine
   2022-03-18T03:05:21.444Z        INFO    Need to update DB
   2022-03-18T03:05:21.445Z        INFO    Downloading DB...
   27.25 MiB / 27.25 MiB [----------------------------------------------------------------------------------------------------] 100.00% 13.03 MiB p/s 3s
   2022-03-18T03:05:25.761Z        WARN    This OS version is not on the EOL list: alpine 3.15
   2022-03-18T03:05:25.762Z        INFO    Detecting Alpine vulnerabilities...
   2022-03-18T03:05:25.768Z        INFO    Trivy skips scanning programming language libraries because no supported file was detected
   2022-03-18T03:05:25.769Z        WARN    This OS version is no longer supported by the distribution: alpine 3.15.1
   2022-03-18T03:05:25.770Z        WARN    The vulnerability detection may be insufficient because security updates are not provided

nginx:alpine (alpine 3.15.1)
============================
Total: 0 (CRITICAL: 0)
```
</p>
</details>
I found "nginx:alpine" is less the vulnerability than other images. So, I will use this image under deployment as a image later.


---
---

Question on "middleware" icon.

- Pod called 'middleware' is already deployed in the 'alpha' namespace. Inspect it but do not alter it in anyway!


Solution:

Just verify the pod named "middleware" with labels under namespace "alpha".
```bash
$ kubectl --namespace alpha get pod --show-labels
  NAME         READY   STATUS    RESTARTS   AGE   LABELS
  external     1/1     Running   0          18m   app=external
  middleware   1/1     Running   0          18m   app=middleware
```
It's runnig.

---
---

Question on "external" icon.

- Pod called 'external' is already deployed in the 'alpha' namespace. Inspect it but do not alter it in anyway!
- 'external' pod should NOT be able to connect to 'alpha-svc' on port 80

Solution:

Let's verify the pod with labels.
```bash
$ kubectl --namespace alpha get pod --show-labels
  NAME         READY   STATUS    RESTARTS   AGE   LABELS
  external     1/1     Running   0          18m   app=external
  middleware   1/1     Running   0          18m   app=middleware
```
Yeah, just leave it for that question. It's enough.

---
---

Question on "alpha-xyz" icon.

- Create a deployment called 'alpha-xyz' that uses the image with the least 'CRITICAL' vulnerabilities? (Use the sample YAML file located at '/root/alpha-xyz.yaml' to create the deployment. Please make sure to use the same names and labels specified in this sample YAML file!)
- Deployment has exactly '1' ready replica
- 'data-volume' is mounted at '/usr/share/nginx/html' on the pod


Solution:

Let's deploy and add the necessary info to that "alpha-xyz" under the given path.
```bash
$ vi /root/alpha-xyz.yaml
```
<details>
<summary>Alpha-XYZ YAML Manifest File</summary>
<p>

```bash
apiVersion: apps/v1
kind: Deployment
metadata:
  creationTimestamp: null
  labels:
    app: alpha-xyz
  name: alpha-xyz
  namespace: alpha
spec:
  replicas: 1
  selector:
    matchLabels:
      app: alpha-xyz
  strategy: {}
  template:
    metadata:
      creationTimestamp: null
      labels:
        app: alpha-xyz
      annotations:
        container.apparmor.security.beta.kubernetes.io/nginx: localhost/custom-nginx
    spec:
      volumes:
      - name: data-volume
        persistentVolumeClaim:
          claimName: alpha-pvc
      containers:
      - image: nginx:alpine
        name: nginx
        volumeMounts:
        - name: data-volume
          mountPath: /usr/share/nginx/html
```
</p>
</details>

Create the Deployment named "alpha-xyz".
```bash
$ kubectl create -f alpha-xyz.yaml
```

Let's verify. Is it running? 
```bash
$  kubectl --namespace alpha get pod
   NAME                         READY   STATUS    RESTARTS   AGE
   alpha-xyz-7c59b9c98f-p69jk   1/1     Running   0          15s
   external                     1/1     Running   0          21m
   middleware                   1/1     Running   0          21m
```

It's running successfully.

---
---

Question on "restrict-inbound" icon.

- Create a NetworkPolicy called 'restrict-inbound' in the 'alpha' namespace
Policy Type = 'Ingress'
Inbound access only allowed from the pod called 'middleware' with label 'app=middleware'
Inbound access only allowed to TCP port 80 on pods matching the policy

Solution:

Create "Network Policy" manifest file.
<details>
<summary>Restrict-Inbound YAML Manifest File</summary>
<p>

```bash
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: restrict-inbound
  namespace: alpha
spec:
  podSelector:
    matchLabels:
      app: alpha-xyz
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          namespace: alpha
      podSelector:
        matchLabels:
          app: middleware
    - podSelector:
        matchLabels:
          app: middleware
    ports:
    - protocol: TCP
      port: 80
```
</p>
</details>

Let's verify.
```bash
$ kubectl --namespace alpha get networkpolicy
  NAME               POD-SELECTOR    AGE
  restrict-inbound   app=alpha-xyz   19s
```

Let's describe that "restrict-inbound".
<details>
<summary>Describe "restrict-inbound"</summary>
<p>

```bash
$ kubectl --namespace alpha describe networkpolicy restrict-inbound
Name:         restrict-inbound
Namespace:    alpha
Created on:   2022-03-18 03:14:08 +0000 UTC
Labels:       <none>
Annotations:  <none>
Spec:
  PodSelector:     app=alpha-xyz
  Allowing ingress traffic:
    To Port: 80/TCP
    From:
      NamespaceSelector: namespace=alpha
      PodSelector: app=middleware
    From:
      PodSelector: app=middleware
  Not affecting egress traffic
  Policy Types: Ingress
```
</p>
</details>

I spent a lot of hour on this manifest file. There is a trick for me. I read [**Kubernetes Document**](https://kubernetes.io/docs/concepts/services-networking/network-policies/) again and again. Now, it's easy for you when you read this **Network Policy** YAML manifest file. For me, It's not easy. After that, I can solve it by elimating my mistake or misconfiguration.

---
---

Question on "alpha-service" icon.

- Expose the 'alpha-xyz' as a 'ClusterIP' type service called 'alpha-svc'
'alpha-svc' should be exposed on 'port: 80' and 'targetPort: 80'

Solution:

Let's expose the "alpha-xyz" deployment as a "ClusterIP" with named "alpha-svc".

```bash
$ kubectl --namespace alpha expose deployment alpha-xyz --name alpha-svc --port 80 --target-port 80
```

Let's verify the resource that created.
```bash
$ kubectl --namespace alpha get service
  NAME        TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
  alpha-svc   ClusterIP   10.111.92.13   <none>        80/TCP    12s
```
<details>
<summary>Alpha-SVC Describe</summary>
<p>

```bash
$ kubectl --namespace alpha describe service alpha-svc
Name:              alpha-svc
Namespace:         alpha
Labels:            app=alpha-xyz
Annotations:       <none>
Selector:          app=alpha-xyz
Type:              ClusterIP
IP Families:       <none>
IP:                10.111.92.13
IPs:               10.111.92.13
Port:              <unset>  80/TCP
TargetPort:        80/TCP
Endpoints:         10.50.0.6:80
Session Affinity:  None
Events:            <none>
```
</p>
</details>

---
---

Finally, time to press final "Check" button. Let's do it.
Bomb!!! Bomb!!! Bomb!!!
It's complete successfully.

![Challenge Complete!](https://github.com/thawzinmyo/The-Hackathon-Challenge-DevOps/blob/master/image/Secure_a_Kubernetes_Deployment_Complete.png)
