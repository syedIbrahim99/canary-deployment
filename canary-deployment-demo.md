# Kubernetes Canary Deployment Lab 

## Objective

Understand how a Canary Deployment works in Kubernetes using:

* Version 1 (v1) application
* Version 2 (v2) application
* Kubernetes Service
* NodePort access
* Browser testing
* Curl-based testing

---

# Lab Architecture

We will deploy:

```text
v1 = 3 Pods
v2 = 1 Pod
```

Total:

```text
4 Pods
```

Traffic distribution will be approximately:

```text
3/4 = 75% → Version 1
1/4 = 25% → Version 2
```

Architecture:

```text
                        nginx-canary Service
                                |
         ------------------------------------------------
         |                |                |            |
      v1 Pod           v1 Pod           v1 Pod       v2 Pod
    VERSION 1        VERSION 1        VERSION 1    VERSION 2
```

---

# Environment

## Cluster Nodes

```bash
kubectl get nodes -o wide
```

Output:

```text
NAME       STATUS   ROLES           INTERNAL-IP
master     Ready    control-plane   192.168.30.171
worker-1   Ready                    192.168.30.172
```

---

# Step 1: Create Namespace

```bash
kubectl create namespace canary-demo
```

Verify:

```bash
kubectl get ns
```

---

# Step 2: Create Version 1 Deployment

Create file:

```bash
vi v1.yaml
```

Contents:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-v1
  namespace: canary-demo

spec:
  replicas: 3

  selector:
    matchLabels:
      app: nginx-canary
      version: v1

  template:
    metadata:
      labels:
        app: nginx-canary
        version: v1

    spec:
      containers:
      - name: nginx
        image: nginx:latest

        ports:
        - containerPort: 80

        command:
        - /bin/sh
        - -c
        - |
          echo '
          <html>
          <body style="font-family: Arial; text-align:center; margin-top:100px;">
          <h1>VERSION 1</h1>
          <h2>Canary Deployment Demo</h2>
          </body>
          </html>
          ' > /usr/share/nginx/html/index.html

          nginx -g 'daemon off;'
```

Apply:

```bash
kubectl apply -f v1.yaml
```

---

# Step 3: Create Version 2 Deployment

Create file:

```bash
vi v2.yaml
```

Contents:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-v2
  namespace: canary-demo

spec:
  replicas: 1

  selector:
    matchLabels:
      app: nginx-canary
      version: v2

  template:
    metadata:
      labels:
        app: nginx-canary
        version: v2

    spec:
      containers:
      - name: nginx
        image: nginx:latest

        ports:
        - containerPort: 80

        command:
        - /bin/sh
        - -c
        - |
          echo '
          <html>
          <body style="font-family: Arial; text-align:center; margin-top:100px;">
          <h1>VERSION 2</h1>
          <h2>Canary Deployment Demo</h2>
          </body>
          </html>
          ' > /usr/share/nginx/html/index.html

          nginx -g 'daemon off;'
```

Apply:

```bash
kubectl apply -f v2.yaml
```

---

# Step 4: Create NodePort Service

Create file:

```bash
vi service.yaml
```

Contents:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-canary
  namespace: canary-demo

spec:
  selector:
    app: nginx-canary

  ports:
  - port: 80
    targetPort: 80
    nodePort: 30080

  type: NodePort
```

Apply:

```bash
kubectl apply -f service.yaml
```

---

# Step 5: Verify Deployments

## Check Pods

```bash
kubectl get pods -n canary-demo -o wide
```

Output:

```text
NAME                        READY   STATUS    IP              NODE
nginx-v1-55589c4787-btbwm   1/1     Running   10.244.226.73   worker-1
nginx-v1-55589c4787-p7mwm   1/1     Running   10.244.226.72   worker-1
nginx-v1-55589c4787-z4pxb   1/1     Running   10.244.226.71   worker-1
nginx-v2-d784fc48f-7wsb9    1/1     Running   10.244.226.74   worker-1
```

---

## Check Service

```bash
kubectl get svc -n canary-demo -o wide
```

Output:

```text
NAME           TYPE       CLUSTER-IP    PORT(S)
nginx-canary   NodePort   10.111.63.6   80:30080/TCP
```

---

## Check Endpoints

```bash
kubectl get endpoints -n canary-demo
```

Output:

```text
NAME           ENDPOINTS
nginx-canary   10.244.226.71:80,
               10.244.226.72:80,
               10.244.226.73:80,
               10.244.226.74:80
```

Notice:

```text
3 v1 pod IPs
1 v2 pod IP
```

All pods are registered with the Service.

---

# Traffic Flow

When a request reaches:

```text
http://192.168.30.171:30080
```

Flow:

```text
Browser / Curl
       |
       v
NodePort Service
       |
       v
nginx-canary Service
       |
       +----> v1 Pod
       +----> v1 Pod
       +----> v1 Pod
       +----> v2 Pod
```

---

# Browser Test

Open:

```text
http://192.168.30.171:30080
```

Expected page:

```text
VERSION 1
Canary Deployment Demo
```

Refresh several times.

You may occasionally see:

```text
VERSION 2
Canary Deployment Demo
```

### Important Note

Browsers often:

* Reuse TCP connections
* Use HTTP Keep-Alive
* Cache requests

Therefore browser testing is **not always reliable** for observing load balancing.

You may continue seeing:

```text
VERSION 1
```

even though the Service is correctly distributing traffic.

---

# Reliable Test Using curl

Create a script:

```bash
vi script.sh
```

Contents:

```bash
for i in {1..30}
do
  curl -s http://192.168.30.171:30080 | grep VERSION
done
```

Give execute permission:

```bash
chmod +x script.sh
```

Run:

```bash
./script.sh
```

Sample Output:

```text
<h1>VERSION 1</h1>
<h1>VERSION 1</h1>
<h1>VERSION 2</h1>
<h1>VERSION 1</h1>
<h1>VERSION 2</h1>
<h1>VERSION 1</h1>
<h1>VERSION 2</h1>
<h1>VERSION 1</h1>
```

This proves:

```text
Traffic is reaching both versions.
```

---

# Why curl Works Better Than Browser

Each curl request creates a new connection:

```text
curl #1 --> v1
curl #2 --> v2
curl #3 --> v1
curl #4 --> v1
curl #5 --> v2
```

Therefore load balancing becomes visible.

---

# Understanding the Canary

Current deployment:

```text
v1 = 3 Pods
v2 = 1 Pod
```

Traffic split:

```text
75% → VERSION 1
25% → VERSION 2
```

This simulates a Canary Deployment.

---

# Simulating a Progressive Rollout

Current:

```text
v1 = 3
v2 = 1
```

Increase v2:

```bash
kubectl scale deployment nginx-v2 \
--replicas=2 \
-n canary-demo
```

Now:

```text
v1 = 3
v2 = 2
```

Approximate traffic:

```text
60% → v1
40% → v2
```

---

Increase again:

```bash
kubectl scale deployment nginx-v2 \
--replicas=3 \
-n canary-demo
```

Now:

```text
v1 = 3
v2 = 3
```

Traffic:

```text
50% → v1
50% → v2
```

---

Complete rollout:

```bash
kubectl scale deployment nginx-v1 \
--replicas=0 \
-n canary-demo
```

Now:

```text
v2 = 3
```

Traffic:

```text
100% → VERSION 2
```

---

# Key Learning

This lab demonstrates a **basic Kubernetes canary deployment** using replica counts.

```text
More v1 pods = More traffic to v1
More v2 pods = More traffic to v2
```

Example:

```text
3 v1 + 1 v2 = 75% / 25%
3 v1 + 2 v2 = 60% / 40%
3 v1 + 3 v2 = 50% / 50%
0 v1 + 3 v2 = 100% v2
```

In production environments, tools such as:

* Istio
* Argo Rollouts

are used to control traffic percentages precisely (e.g., 90% v1 and 10% v2) without depending on replica counts.
