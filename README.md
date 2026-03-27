# KIND Kubernetes Demo 🚀

A hands-on Kubernetes learning project using KIND (Kubernetes IN Docker) on Mac.

---

## Cluster Setup

Multi-node cluster with port mappings for Ingress:
- 1 Control Plane
- 2 Worker Nodes
```bash
kind create cluster --name multi-node --config kind-config.yaml
```

---

## Project Structure
```
kind-demo/
├── kind-config.yaml   → multi-node cluster config with port mappings
├── namespace.yaml     → nginx-ns namespace
├── deployment.yaml    → nginx deployment with PVC + ConfigMap + Secret
├── service.yaml       → ClusterIP service
├── configmap.yaml     → app environment config
├── secret.yaml        → sensitive data (passwords, API keys)
├── pv.yaml            → persistent volume + persistent volume claim
├── cronjob.yaml       → automated backup every minute
└── ingress.yaml       → ingress routing rules
```

---

## Concepts Covered

### 1. Pods & Deployments
- Pods are temporary — Kubernetes recreates them automatically
- Never create pods directly — always use Deployments
- Deployment manages ReplicaSet which manages Pods
```bash
kubectl create deployment my-nginx --image=nginx --replicas=4
```

---

### 2. Services
Expose pods internally using ClusterIP:
```bash
kubectl expose deployment my-nginx --port=80 --type=ClusterIP
```

---

### 3. Namespaces
Isolate resources by environment or team:
```bash
kubectl apply -f namespace.yaml
kubectl get all -n nginx-ns
```

---

### 4. Scaling
Tell Kubernetes how many replicas you want — it handles the rest:
```bash
kubectl scale deployment my-nginx --replicas=6   # scale up
kubectl scale deployment my-nginx --replicas=2   # scale down
kubectl scale deployment my-nginx --replicas=0   # stop app
```

---

### 5. Rolling Updates & Rollbacks
Update app with zero downtime:
```bash
kubectl set image deployment/my-nginx nginx=nginx:1.25
kubectl rollout status deployment/my-nginx
```

Rollback if something goes wrong:
```bash
kubectl rollout undo deployment/my-nginx
kubectl rollout history deployment/my-nginx
```

---

### 6. ConfigMaps & Secrets
Pass config and passwords to pods without hardcoding:
```bash
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
```

Verify inside pod:
```bash
kubectl exec -it <pod-name> -n nginx-ns -- /bin/bash
echo $APP_ENV
echo $DB_PASSWORD
```

---

### 7. Volumes & PersistentVolumes

**Problem:** Pod data is lost when pod is deleted.

**emptyDir** — temporary, deleted with pod:
```yaml
volumes:
- name: myvolume
  emptyDir: {}
```

**PersistentVolume** — survives pod deletion:
```yaml
volumes:
- name: myvolume
  persistentVolumeClaim:
    claimName: nginx-pvc
```
```bash
kubectl get pv
kubectl get pvc -n nginx-ns
```

---

### 8. CronJob (Automated Backup)
Runs backup every minute (use `0 2 * * *` for daily in production):
```bash
kubectl apply -f cronjob.yaml
kubectl get cronjob -n nginx-ns
kubectl logs <backup-pod-name> -n nginx-ns
```

Suspend cronjob:
```bash
kubectl patch cronjob nginx-backup -n nginx-ns -p '{"spec":{"suspend":true}}'
```

---

### 9. Ingress

**Problem:** port-forward is temporary and manual.

**Solution:** Ingress = permanent smart front door for your cluster.

#### What We Did
1. Recreated cluster with port mappings (80 & 443)
2. Installed nginx ingress controller
3. Created ingress.yaml with routing rules
4. Added `nginx.local` to `/etc/hosts`

#### Traffic Flow
```
curl/browser (localhost:8080)
        ↓
port-forward → ingress-nginx-controller
        ↓
Ingress rule (Host: nginx.local)
        ↓
Service (nginx-service)
        ↓
Pod (nginx container) ✅
```

#### Test
```bash
kubectl port-forward -n ingress-nginx service/ingress-nginx-controller 8080:80
curl -H "Host: nginx.local" http://localhost:8080
```

#### Challenges Faced
| Challenge | Cause | Fix |
|---|---|---|
| Site can't be reached | Port 80 not mapped in old cluster | Recreated cluster with extraPortMappings |
| Ctrl+Z suspending command | Wrong keyboard shortcut | Use Ctrl+C to cancel, not Ctrl+Z |
| No active endpoints warning | Ingress started before pods were ready | Timing issue — fixed itself |
| Connection reset on Mac | Docker Desktop VM layer blocks direct port mapping | Used port-forward on ingress controller |

#### Real World vs Local
| Real World | Local (KIND on Mac) |
|---|---|
| Real domain (myapp.com) | Fake domain (nginx.local) |
| Cloud LoadBalancer (AWS ELB) | port-forward workaround |
| TLS/HTTPS | Plain HTTP |

---

## Apply Everything
```bash
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml
kubectl apply -f pv.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
kubectl apply -f cronjob.yaml
```

---

## Key Commands Reference
```bash
kubectl get all -n nginx-ns          # see everything in namespace
kubectl get nodes -o wide            # see all nodes with details
kubectl get pods -A                  # pods across all namespaces
kubectl describe pod <pod-name>      # deep detail of a pod
kubectl logs <pod-name>              # app logs inside pod
kubectl exec -it <pod-name> -- bash  # go inside a pod
kubectl rollout history deployment   # see deployment versions
```