# Troubleshooting Guide

> Complete record of every issue encountered during this Kubernetes WordPress deployment on AWS EC2, including root cause analysis and exact fixes applied.

---

## Quick Reference — Issues We Hit

| # | Problem | Symptom | Root Cause | Fix |
|---|---------|---------|------------|-----|
| 1 | NFS mount hanging | `sudo mount` never returns | Port 2049 blocked in Security Group | Open port 2049 TCP/UDP |
| 2 | WordPress DB error | "Error establishing a database connection" | Wrong password key in secret reference | Change `MYSQL_ROOT_PASSWORD` key to `MYSQL_PASSWORD` |
| 3 | Calico nodes 0/1 | BGP not established | Port 179 + Protocol 4 blocked | Open port 179 TCP and Custom Protocol 4 |
| 4 | kubectl exec timeout | `error dialing backend: i/o timeout` | Port 10250 blocked | Open port 10250 TCP |
| 5 | DNS resolution fails | `nslookup mysql-service: no servers could be reached` | Calico broken (Protocol 4 blocked) | Fix Calico — DNS auto-recovers |
| 6 | Pod stuck ContainerCreating | WordPress pod never starts | `nfs-common` not installed on worker | `sudo apt install -y nfs-common` |
| 7 | kubeadm join fails | `ERROR IsPrivilegedUser` | Ran without sudo | Use `sudo kubeadm join` |

---

## Issue 1 — NFS Mount Hanging Forever

### Symptom
Running this command on the worker node would hang indefinitely and never return:
```bash
sudo mount -t nfs 172.31.85.249:/var/nfs/wordpress /mnt
```
The terminal just froze. No error, no output. Only `Ctrl+C` could cancel it. Then got:
```
mount.nfs: Connection timed out for 172.31.85.249:/var/nfs/wordpress on /mnt
```

### Root Cause
Port **2049** (the main NFS data transfer port) was not open in the AWS Security Group on the NFS server instance. The TCP connection was being silently dropped by AWS before it even reached the NFS server process.

### Why Port 2049 Matters
NFS uses port 2049 for all file data transfer. When the worker node tries to mount the NFS share, it connects to the NFS server on port 2049. If that port is blocked, the connection hangs waiting for a response that never comes — hence the indefinite wait.

### Fix Applied
Opened the following rules in the AWS Security Group:

| Type | Protocol | Port | Source |
|------|----------|------|--------|
| NFS (Custom TCP) | TCP | 2049 | 172.31.0.0/16 |
| Custom UDP | UDP | 2049 | 172.31.0.0/16 |
| Custom TCP | TCP | 111 | 172.31.0.0/16 |

Port 111 (RPC portmapper) is also needed — NFS uses Remote Procedure Calls and portmapper helps clients discover which port to use.

Source `172.31.0.0/16` = entire VPC. Only EC2 instances inside the VPC can access NFS — not the public internet.

### Verification
After opening the ports, this worked immediately:
```bash
sudo mount -t nfs 172.31.85.249:/var/nfs/wordpress /mnt
ls /mnt         # returned empty — no error
sudo umount /mnt
```

---

## Issue 2 — "Error establishing a database connection"

### Symptom
WordPress site at `http://3.95.224.129:30007` showed:
```
Error establishing a database connection
```
Both pods were Running. Both PVCs were Bound. MySQL logs showed it was up and listening on 3306.

### Root Cause
MySQL has two users:
- `root` — password: `mypassword`
- `wordpress` — password: `wordpresspass`

WordPress was configured to connect as the `wordpress` user. But in `wp-deployment.yaml`, the password was being fetched from the Secret using the **wrong key**:

```yaml
# WRONG — was fetching root password
- name: WORDPRESS_DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: mysql-secret
      key: MYSQL_ROOT_PASSWORD   # ← this is "mypassword" (root user)
```

WordPress was connecting as user `wordpress` but presenting the root password `mypassword`. MySQL rejected it because the `wordpress` user's actual password is `wordpresspass`.

### Fix Applied

**Option A (quick fix):** Hardcode the correct password temporarily:
```yaml
- name: WORDPRESS_DB_PASSWORD
  value: wordpresspass
```

**Option B (production-grade fix):** Add `MYSQL_PASSWORD` to the Secret and reference it correctly:

Updated `mysql-secret.yaml`:
```yaml
data:
  MYSQL_ROOT_PASSWORD: bXlwYXNzd29yZA==      # base64("mypassword")
  MYSQL_PASSWORD: d29yZHByZXNzcGFzcw==        # base64("wordpresspass")
```

Updated `wp-deployment.yaml`:
```yaml
- name: WORDPRESS_DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: mysql-secret
      key: MYSQL_PASSWORD   # ← correct key now
```

Then redeployed:
```bash
kubectl delete deployment wordpress
kubectl apply -f wordpress/wp-deployment.yaml
```

### Verification
WordPress installation screen appeared at `http://3.95.224.129:30007` immediately after redeployment.

### Lesson Learned
Always match the user to their correct password. MySQL root password and the wordpress user password are different credentials. The Secret must contain both, and each deployment must reference the correct key for its user.

---

## Issue 3 — Calico Nodes Showing 0/1 (Not Ready)

### Symptom
```bash
kubectl get pods -n kube-system | grep calico
calico-node-pnjrp   0/1   Running   0   127m
calico-node-r4nfz   0/1   Running   0   123m
```

Pods were running but not ready. Checking events:
```bash
kubectl describe pod calico-node-pnjrp -n kube-system
```
Output showed:
```
Warning  Unhealthy  Readiness probe failed:
calico/node is not ready: BIRD is not ready: BGP not established with 172.31.83.196
```

### Root Cause
Calico uses two network mechanisms:
1. **BGP (Border Gateway Protocol)** on port **179** — for nodes to exchange pod routing information
2. **IP-in-IP (Protocol 4)** — to encapsulate pod traffic between nodes

Both were blocked by the AWS Security Group. Without BGP, Calico nodes could not exchange routes. Without IP-in-IP, pod-to-pod traffic across nodes was silently dropped.

This was the root cause of the DNS failure (Issue 5) and the "Error establishing a database connection" — WordPress pod could not reach MySQL pod across the cluster network.

### Fix Applied
Added two new rules to the Security Group:

| Type | Protocol | Port | Source |
|------|----------|------|--------|
| Custom TCP | TCP | 179 | 172.31.0.0/16 |
| Custom Protocol | 4 (IP-in-IP) | All | 172.31.0.0/16 |

> **Important:** Protocol 4 is NOT a TCP/UDP port. In the AWS Security Group dropdown, select **Custom Protocol** and enter `4` in the Protocol field. Leave Port range empty. This is IP-in-IP encapsulation — a separate IP protocol number, not a port.

Then restarted Calico:
```bash
kubectl rollout restart daemonset/calico-node -n kube-system
```

### Verification
```bash
kubectl get pods -n kube-system | grep calico
calico-node-pnjrp   1/1   Running   0   2m
calico-node-r4nfz   1/1   Running   0   2m
```
Both now showing `1/1 Running`.

---

## Issue 4 — kubectl exec Timing Out

### Symptom
When trying to exec into a pod:
```bash
kubectl exec -it wordpress-84bbcd6648-b7fpc -- env | grep WORDPRESS
```
Got:
```
Error from server: error dialing backend: dial tcp 172.31.83.196:10250: i/o timeout
```

### Root Cause
Port **10250** is the Kubelet API port. The master node uses this port to:
- Execute commands inside pods (`kubectl exec`)
- Fetch pod logs (`kubectl logs`)
- Port-forward (`kubectl port-forward`)

This port was blocked in the Security Group, so the master could not reach the kubelet on the worker node.

### Fix Applied
Added inbound rule to Security Group:

| Type | Protocol | Port | Source |
|------|----------|------|--------|
| Custom TCP | TCP | 10250 | 172.31.0.0/16 |

### Verification
```bash
kubectl exec -it <pod-name> -- env | grep WORDPRESS
# Now returns environment variables successfully
```

---

## Issue 5 — DNS Resolution Failing Inside Cluster

### Symptom
```bash
kubectl run test-pod --image=busybox --rm -it --restart=Never -- nslookup mysql-service
```
Returned:
```
;; connection timed out; no servers could be reached
pod "test-pod" deleted
pod default/test-pod terminated (Error)
```

WordPress could not resolve `mysql-service` to the MySQL pod IP.

### Root Cause
CoreDNS was running (both pods showing `1/1 Running`) but Calico networking was broken (Issue 3 — Protocol 4 blocked). Without working pod-to-pod networking, the test pod could not reach the CoreDNS pods to resolve DNS queries.

This was a cascading failure: Security Group blocking Protocol 4 → Calico broken → pod networking broken → DNS broken → WordPress cannot find MySQL → "Error establishing a database connection".

### Fix Applied
Same fix as Issue 3 — opening Protocol 4 in the Security Group fixed Calico, which fixed pod networking, which fixed DNS automatically.

No separate DNS fix was needed.

### Verification
After fixing Calico:
```bash
kubectl run test-pod --image=busybox --rm -it --restart=Never -- nslookup mysql-service
```
Returned:
```
Server:    10.96.0.10
Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local

Name:      mysql-service
Address 1: 192.168.4.65 mysql-86678984b7-n6hcd.mysql-service.default.svc.cluster.local
```

---

## Issue 6 — WordPress Pod Stuck in ContainerCreating

### Symptom
```bash
kubectl get pods -l app=wordpress
NAME                         READY   STATUS              RESTARTS   AGE
wordpress-84bbcd6648-b7fpc   0/1     ContainerCreating   0          15m
```
Pod stayed in `ContainerCreating` indefinitely. No clear error in logs since the container had never started.

```bash
kubectl describe pod wordpress-84bbcd6648-b7fpc
```
Events showed volume mount issues — the NFS volume could not be mounted.

### Root Cause
`nfs-common` was not installed on the worker node. When Kubernetes mounts an NFS PersistentVolume into a pod, it uses the **host machine's OS** (worker node Ubuntu) to perform the actual `mount -t nfs` command — NOT something inside the container.

The `nfs-common` package provides the NFS client tools (`mount.nfs`) that the worker node's Ubuntu OS needs. Without it, the mount fails silently and the pod stays in `ContainerCreating` forever with no clear error message.

### Fix Applied
On the **worker node**:
```bash
sudo apt install -y nfs-common
```

Then deleted the stuck pod so it would be recreated fresh:
```bash
kubectl delete pod wordpress-84bbcd6648-b7fpc
# Deployment automatically creates a replacement pod
```

### Verification
```bash
kubectl get pods -l app=wordpress
NAME                        READY   STATUS    RESTARTS   AGE
wordpress-fb5b94d49-4k87s   1/1     Running   0          37s
```

### Key Lesson
`nfs-common` must be installed on the worker node **before** deploying any YAML files. The container itself does not need NFS tools — the host node does. This is one of the most common Kubernetes NFS mistakes.

---

## Issue 7 — kubeadm join Failing (Not Root)

### Symptom
On the worker node:
```bash
kubeadm join 172.31.89.189:6443 --token ... --discovery-token-ca-cert-hash sha256:...
```
Returned:
```
error execution phase preflight: [preflight] Some fatal errors occurred:
    [ERROR IsPrivilegedUser]: user is not running as root
```

### Root Cause
`kubeadm join` requires root privileges. Running without `sudo` fails the preflight check.

### Fix Applied
```bash
sudo kubeadm join 172.31.89.189:6443 --token ... --discovery-token-ca-cert-hash sha256:...
```

### Verification
```
This node has joined the cluster:
* Certificate signing request was sent to apiserver and a response was received.
* The Kubelet was informed of the new secure secure connection details.
Run 'kubectl get nodes' on the control-plane to see this node join the cluster.
```

---

## Essential Debug Commands

### Check Everything at Once
```bash
kubectl get pods -A                          # all pods, all namespaces
kubectl get all                              # all resources in default namespace
kubectl get pv,pvc                           # storage status
kubectl get nodes                            # node health
```

### Debug a Specific Pod
```bash
kubectl describe pod <pod-name>              # events + full details
kubectl logs <pod-name>                      # container output
kubectl logs <pod-name> --previous           # logs from crashed container
kubectl exec -it <pod-name> -- bash          # shell inside container
```

### Check Specific Stacks
```bash
# MySQL
kubectl get pods -l app=mysql
kubectl logs -l app=mysql
kubectl get pvc mysql-pvc

# WordPress
kubectl get pods -l app=wordpress
kubectl logs -l app=wordpress
kubectl get pvc wp-pvc

# Calico
kubectl get pods -n kube-system | grep calico
kubectl describe pod <calico-pod> -n kube-system

# Dashboard
kubectl get pods -n kubernetes-dashboard
kubectl get svc -n kubernetes-dashboard
```

### Test Network/DNS
```bash
# Test DNS resolution inside cluster
kubectl run test-pod --image=busybox --rm -it --restart=Never -- nslookup mysql-service

# Test NFS mount from worker node
sudo mount -t nfs 172.31.85.249:/var/nfs/wordpress /mnt
ls /mnt
sudo umount /mnt

# Check NFS exports on NFS server
sudo exportfs -v
```

### Fix/Restart Commands
```bash
# Restart Calico after Security Group changes
kubectl rollout restart daemonset/calico-node -n kube-system

# Restart a deployment
kubectl rollout restart deployment/wordpress
kubectl rollout restart deployment/mysql

# Delete and recreate a stuck pod (Deployment auto-recreates it)
kubectl delete pod <pod-name>

# Regenerate join token on master
kubeadm token create --print-join-command

# Get new dashboard token
kubectl -n kubernetes-dashboard create token admin-user
```

---

## Common Error Messages — Quick Reference

| Error Message | Likely Cause | Fix |
|--------------|--------------|-----|
| `mount.nfs: Connection timed out` | Port 2049 blocked | Open 2049 TCP/UDP in Security Group |
| `Error establishing a database connection` | Wrong DB password or Calico broken | Check secretKeyRef key + fix Calico |
| `BGP not established with X.X.X.X` | Port 179 or Protocol 4 blocked | Open port 179 + Custom Protocol 4 |
| `error dialing backend: i/o timeout` | Port 10250 blocked | Open port 10250 TCP |
| `no servers could be reached` (nslookup) | Calico/pod networking broken | Fix Calico Protocol 4 |
| `ContainerCreating` (stuck) | nfs-common missing on worker | `sudo apt install -y nfs-common` |
| `ERROR IsPrivilegedUser` | kubeadm run without sudo | Use `sudo kubeadm join` |
| `CreateContainerConfigError` | ConfigMap or Secret missing | Apply ConfigMap/Secret before Deployment |
| `Pending` PVC | PV not found or values mismatch | Check storage size + access mode match PV exactly |

---

## Security Group Checklist

Before deploying, verify these ports are open:

- [ ] Port 22 TCP — SSH (0.0.0.0/0)
- [ ] Port 80 TCP — HTTP (0.0.0.0/0)
- [ ] Port 443 TCP — HTTPS (0.0.0.0/0)
- [ ] Port 6443 TCP — Kubernetes API (172.31.0.0/16)
- [ ] Port 10250 TCP — Kubelet (172.31.0.0/16)
- [ ] Port 179 TCP — Calico BGP (172.31.0.0/16)
- [ ] Custom Protocol 4 — IP-in-IP (172.31.0.0/16)
- [ ] Port 2049 TCP — NFS (172.31.0.0/16)
- [ ] Port 2049 UDP — NFS (172.31.0.0/16)
- [ ] Port 111 TCP — RPC (172.31.0.0/16)
- [ ] Port 30007 TCP — WordPress NodePort (0.0.0.0/0)
- [ ] Port 30001 TCP — Dashboard NodePort (0.0.0.0/0)

---

*Kubernetes WordPress Project | Troubleshooting Reference | AWS EC2 | NFS Storage*
