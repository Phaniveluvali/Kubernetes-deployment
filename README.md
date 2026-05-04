# Kubernetes Metrics Server Setup (Working Guide)

This guide documents the exact steps we followed to get **Metrics Server working** in our Kubernetes cluster (v1.30).

We ran into the common issue where metrics were not available, and this README captures what actually worked for us.

---

## 🧩 The Problem

When we tried to check node metrics:

```bash
kubectl top nodes
```

We got:

```
error: Metrics API not available
```

---

## 🔍 What was happening?

* Metrics Server was either not installed or not functioning properly
* Even after installing it, the pod was not becoming `Ready` (0/1)
* Root cause: Metrics Server couldn’t talk to Kubelet due to TLS validation issues (very common in on-prem clusters)

---

## ⚙️ What worked for us

### 1. Install Metrics Server

We used the official manifest:

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

### 2. Check deployment status

```bash
kubectl get deployments -n kube-system
```

At this point, we saw:

```
metrics-server   0/1   (not ready)
```

---

### 3. Fix the real issue (IMPORTANT)

This is the key step.

We edited the **deployment (not the pod!)**:

```bash
kubectl edit deployment metrics-server -n kube-system
```

Then added the following arguments under the container section:

```yaml
- --kubelet-insecure-tls
- --kubelet-preferred-address-types=InternalIP
```

So it looked like this:

```yaml
containers:
- name: metrics-server
  args:
    - --cert-dir=/tmp
    - --secure-port=10250
    - --kubelet-insecure-tls
    - --kubelet-preferred-address-types=InternalIP
```

---

### ⚠️ Important note

We initially tried modifying the pod directly — that didn’t work.

👉 The fix must be applied at the **Deployment level**, because pods are recreated automatically.

---

### 4. Restart the deployment

```bash
kubectl rollout restart deployment metrics-server -n kube-system
```

---

### 5. Verify

```bash
kubectl get pods -n kube-system
```

Now the metrics-server pod should be:

```
1/1 Running
```

---

### 6. Test metrics

After ~30 seconds:

```bash
kubectl top nodes
kubectl top pods -A
```

This finally worked ✅

---

## 🔧 If it still doesn’t work

Here are a few quick things to check:

### Check logs

```bash
kubectl logs -n kube-system deployment/metrics-server
```

---

### Check API service

```bash
kubectl get apiservice v1beta1.metrics.k8s.io -o yaml
```

Look for:

```
status: True
```

---

### Check node connectivity

```bash
kubectl get nodes -o wide
```

Make sure:

* Node Internal IPs are reachable
* Port **10250** is open (Kubelet)

---

## 🧠 Key takeaway

In most on-prem Kubernetes setups, Metrics Server fails because:

* Kubelet uses self-signed certificates
* Metrics Server tries to verify them and fails

Adding:

```yaml
--kubelet-insecure-tls
```

solves this problem.

---

## ✅ Final result

Once everything is fixed:

```bash
kubectl top nodes
```

You’ll see CPU and memory usage for nodes and pods.

---

## 👍 Summary

* Installed Metrics Server
* Deployment was not ready
* Fixed by adding kubelet flags in deployment
* Restarted deployment
* Metrics started working

---

This approach worked reliably in our environment and should help in similar setups.

---

