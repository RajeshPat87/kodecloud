Complete documentation for all 10 Kubernetes tasks, inline for copy-paste.

---

# Runbook — Kubernetes Tasks: Pods, Deployments, Services, Troubleshooting

---

## Task 1 — Create a labeled Pod (`pod-httpd-t1q1`)

**Requirement:** Pod named `pod-httpd-t1q1`, image `httpd:latest`, label `app: httpd_app_t1q1`, container `httpd-container-t1q1`.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: pod-httpd-t1q1
  labels:
    app: httpd_app_t1q1
spec:
  containers:
    - name: httpd-container-t1q1
      image: httpd:latest
EOF
```

**Verify:**
```bash
kubectl get pod pod-httpd-t1q1 --show-labels
kubectl describe pod pod-httpd-t1q1 | grep -E "Name:|Image:|Labels:|State:"
```

**Requirement trace:**

| Requirement | Evidence |
|---|---|
| Pod name `pod-httpd-t1q1` | `metadata.name` |
| Image `httpd:latest` | `image: httpd:latest` |
| Label `app: httpd_app_t1q1` | `labels: app: httpd_app_t1q1` |
| Container `httpd-container-t1q1` | `name: httpd-container-t1q1` |

---

## Task 2 — Scale Deployment (`blue-app-t2q5`) replicas 1→3

**Requirement:** Change replicas from `1` to `3` without deleting the deployment.

```bash
# confirm current state + selector
kubectl get deployment blue-app-t2q5
kubectl get deployment blue-app-t2q5 -o jsonpath='{.spec.selector.matchLabels}'

# scale declaratively
kubectl get deployment blue-app-t2q5 -o yaml \
  | sed 's|replicas: 1|replicas: 3|g' \
  | kubectl apply -f -

# monitor
kubectl rollout status deployment/blue-app-t2q5

# verify
kubectl get deployment blue-app-t2q5
kubectl get pods -l app=blue-app-t2q5
```

**Requirement trace:**

| Requirement | Evidence |
|---|---|
| Deployment exists | `kubectl get deployment` |
| Replicas 1 → 3 | `sed replicas` + apply |
| All pods running | `READY 3/3` |

---

## Task 3 — Rollback Deployment (`nginx-deployment-t2q2`)

**Requirement:** Roll back to previous revision after a bad release.

```bash
# check history
kubectl rollout history deployment/nginx-deployment-t2q2

# check current image
kubectl describe deployment nginx-deployment-t2q2 | grep Image

# rollback
kubectl rollout undo deployment/nginx-deployment-t2q2

# monitor
kubectl rollout status deployment/nginx-deployment-t2q2

# verify image reverted
kubectl describe deployment nginx-deployment-t2q2 | grep Image
kubectl get pods | grep nginx-deployment-t2q2
```

**Key lesson:** `kubectl rollout undo` without `--to-revision` always reverts one revision. The old replicaset kept at 0 replicas is exactly for this purpose.

**Requirement trace:**

| Requirement | Evidence |
|---|---|
| Rolled back | `deployment rolled back` + `successfully rolled out` |
| Previous image restored | `describe` shows previous image |
| Pods running | `1/1 Running` |

---

## Task 4 — Pod with resource requests and limits (`httpd-pod-t3q6`)

**Requirement:** Pod `httpd-pod-t3q6`, container `httpd-container-t3q6`, image `httpd:latest`, requests memory `15Mi` CPU `100m`, limits memory `20Mi` CPU `100m`.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: httpd-pod-t3q6
spec:
  containers:
    - name: httpd-container-t3q6
      image: httpd:latest
      resources:
        requests:
          memory: "15Mi"
          cpu: "100m"
        limits:
          memory: "20Mi"
          cpu: "100m"
EOF
```

**Verify:**
```bash
kubectl get pod httpd-pod-t3q6
kubectl describe pod httpd-pod-t3q6 | grep -A6 "Requests:"
```

**Requirement trace:**

| Requirement | Evidence |
|---|---|
| Pod/container names | `metadata.name` + `containers[0].name` |
| Image `httpd:latest` | `image: httpd:latest` |
| Request memory `15Mi` / CPU `100m` | `requests` block |
| Limit memory `20Mi` / CPU `100m` | `limits` block |

---

## Task 5 — Create ReplicaSet (`httpd-replicaset-t3q4`)

**Requirement:** RS named `httpd-replicaset-t3q4`, image `httpd:latest`, labels `app: httpd_app_t3q4` + `type: front-end-t3q4`, container `httpd-container-t3q4`, replicas `4`.

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: httpd-replicaset-t3q4
  labels:
    app: httpd_app_t3q4
    type: front-end-t3q4
spec:
  replicas: 4
  selector:
    matchLabels:
      app: httpd_app_t3q4
      type: front-end-t3q4
  template:
    metadata:
      labels:
        app: httpd_app_t3q4
        type: front-end-t3q4
    spec:
      containers:
        - name: httpd-container-t3q4
          image: httpd:latest
EOF
```

**Verify:**
```bash
kubectl get replicaset httpd-replicaset-t3q4
kubectl get pods -l app=httpd_app_t3q4 --show-labels
```

**Key lesson:** Labels must appear in three places — `metadata.labels`, `selector.matchLabels`, and `template.metadata.labels`. All three must be consistent.

---

## Task 6 — Fix nginx+PHP-FPM pod (`nginx-phpfpm-t4q3`) — nginx mount wrong

**Root cause:** `nginx-container` mounted shared volume at `/usr/share/nginx/html` but nginx config served from `/var/www/html` → files not found.

```bash
# investigate
kubectl describe pod nginx-phpfpm-t4q3 | grep -A2 "Mounts:"
kubectl exec nginx-phpfpm-t4q3 -c nginx-container -- ls -la /var/www/html/
kubectl exec nginx-phpfpm-t4q3 -c php-fpm-container -- ls -la /var/www/html/

# export and fix
kubectl get pod nginx-phpfpm-t4q3 -o yaml > /tmp/nginx-phpfpm-t4q3.yaml
sed -i 's|mountPath: /usr/share/nginx/html|mountPath: /var/www/html|g' \
  /tmp/nginx-phpfpm-t4q3.yaml
grep "mountPath" /tmp/nginx-phpfpm-t4q3.yaml

# delete and recreate (pods are immutable)
kubectl delete pod nginx-phpfpm-t4q3
kubectl apply -f /tmp/nginx-phpfpm-t4q3.yaml
kubectl wait --for=condition=Ready pod/nginx-phpfpm-t4q3 --timeout=120s

# copy index.php (emptyDir resets on pod delete)
kubectl cp /home/thor/index.php nginx-phpfpm-t4q3:/var/www/html/index.php -c nginx-container

# verify
kubectl exec nginx-phpfpm-t4q3 -c nginx-container -- ls -l /var/www/html/index.php
kubectl exec nginx-phpfpm-t4q3 -c php-fpm-container -- ls -l /var/www/html/index.php
curl -s http://localhost:30008/index.php
```

**Root cause table:**

| Container | Before fix | After fix |
|---|---|---|
| `php-fpm-container` | `/var/www/html` ✓ | `/var/www/html` ✓ |
| `nginx-container` | `/usr/share/nginx/html` ❌ | `/var/www/html` ✓ |

---

## Task 7 — Fix Redis deployment (`redis-deployment-t4q2`) — two typos

**Root cause:** Two typos introduced during modification:
- Image: `redis:alpin` → `redis:alpine`
- ConfigMap name: `redis-cofig-t4q2` → `redis-config-t4q2`

```bash
# investigate
kubectl get deployment redis-deployment-t4q2
kubectl describe pod $(kubectl get pods | grep redis-deployment-t4q2 \
  | awk '{print $1}' | head -1) | grep -E "Image:|FailedMount|configmap"

# confirm real configmap name
kubectl get configmap | grep redis

# export and fix both typos
kubectl get deployment redis-deployment-t4q2 -o yaml > /tmp/redis-deployment.yaml
sed -i 's|image: redis:alpin$|image: redis:alpine|g' /tmp/redis-deployment.yaml
sed -i 's|redis-cofig-t4q2|redis-config-t4q2|g' /tmp/redis-deployment.yaml

# verify fixes
grep -E "image:|redis-co" /tmp/redis-deployment.yaml

# apply
kubectl apply -f /tmp/redis-deployment.yaml
kubectl rollout status deployment/redis-deployment-t4q2

# verify Redis
kubectl exec -it $(kubectl get pods | grep redis-deployment-t4q2 \
  | awk '{print $1}' | head -1) -- redis-cli ping
```

Expected: `PONG`

---

## Task 8 — Add label to service (`service-t5q6`)

**Requirement:** Add label `component: front-end-t5q6` to existing service.

```bash
# check existing labels
kubectl get service service-t5q6 --show-labels

# add label
kubectl label service service-t5q6 component=front-end-t5q6

# verify
kubectl get service service-t5q6 --show-labels
```

**Declarative alternative:**
```bash
kubectl get service service-t5q6 -o yaml > /tmp/service-t5q6.yaml
sed -i '/^  labels:/a\    component: front-end-t5q6' /tmp/service-t5q6.yaml
kubectl apply -f /tmp/service-t5q6.yaml
kubectl get service service-t5q6 --show-labels
```

---

## Task 9 — Fix service nodePort (`service-t5q3`) 30098→30099

**Requirement:** Update nodePort from `30098` to `30099`.

```bash
# check current nodePort
kubectl get service service-t5q3
kubectl describe service service-t5q3 | grep NodePort

# export and fix
kubectl get service service-t5q3 -o yaml > /tmp/service-t5q3.yaml
sed -i 's|nodePort: 30098|nodePort: 30099|g' /tmp/service-t5q3.yaml
grep "nodePort" /tmp/service-t5q3.yaml
kubectl apply -f /tmp/service-t5q3.yaml

# verify
kubectl get service service-t5q3
kubectl describe service service-t5q3 | grep NodePort
curl -s http://localhost:30099/
```

Expected: `80:30099/TCP`

---

## Task 10 — Fix nginx+PHP-FPM pod (`nginx-phpfpm`) — php-fpm mount wrong

**Root cause:** `php-fpm-container` mounted shared volume at `/usr/share/nginx/html` but nginx config passed `SCRIPT_FILENAME=/var/www/html/index.php` to PHP-FPM → "File not found".

```bash
# investigate
kubectl describe pod nginx-phpfpm | grep -A2 "Mounts:"
kubectl exec nginx-phpfpm -c php-fpm-container -- ls -la /var/www/html/
kubectl exec nginx-phpfpm -c php-fpm-container -- ls -la /usr/share/nginx/html/

# export and fix
kubectl get pod nginx-phpfpm -o yaml > /tmp/nginx-phpfpm.yaml
sed -i 's|mountPath: /usr/share/nginx/html|mountPath: /var/www/html|g' \
  /tmp/nginx-phpfpm.yaml
grep "mountPath" /tmp/nginx-phpfpm.yaml

# delete and recreate
kubectl delete pod nginx-phpfpm
kubectl apply -f /tmp/nginx-phpfpm.yaml
kubectl wait --for=condition=Ready pod/nginx-phpfpm --timeout=120s

# copy index.php
kubectl cp /home/thor/index.php nginx-phpfpm:/var/www/html/index.php -c nginx-container

# verify
kubectl exec nginx-phpfpm -c nginx-container -- ls -l /var/www/html/index.php
kubectl exec nginx-phpfpm -c php-fpm-container -- ls -l /var/www/html/index.php
curl -s http://localhost:30008/index.php
```

**Root cause table:**

| Container | Before fix | After fix |
|---|---|---|
| `nginx-container` | `/var/www/html` ✓ | `/var/www/html` ✓ |
| `php-fpm-container` | `/usr/share/nginx/html` ❌ | `/var/www/html` ✓ |

---

## General troubleshooting reference

```bash
# pod not starting
kubectl describe pod <name>           # check Events section

# check logs
kubectl logs <pod>                    # current logs
kubectl logs <pod> --previous         # crashed container logs
kubectl logs <pod> -c <container>     # specific container

# exec into container
kubectl exec -it <pod> -c <container> -- /bin/sh

# check all resources
kubectl get all
kubectl get all -A                    # all namespaces

# events sorted by time
kubectl get events --sort-by='.lastTimestamp'

# patch a label
kubectl label <resource> <name> key=value
kubectl label <resource> <name> key=value --overwrite

# scale
kubectl scale deployment <name> --replicas=N

# rollout
kubectl rollout status deployment/<name>
kubectl rollout history deployment/<name>
kubectl rollout undo deployment/<name>
```

## Common issue patterns

| Symptom | Likely cause | Fix |
|---|---|---|
| `ImagePullBackOff` | Image tag typo | fix image in deployment yaml |
| `ContainerCreating` stuck | Missing ConfigMap/Secret | check `FailedMount` in events; fix name typo |
| `File not found` (PHP-FPM) | Volume mount path mismatch | align both containers to same `mountPath` |
| `spec.selector: immutable` | Changed selector on apply | match existing selector exactly |
| `0/1 Running` | Multiple issues | check `describe pod` Events section |
| `CrashLoopBackOff` | Bad command/args or OOM | check logs + resource limits |
| Wrong nodePort | Typo in service spec | `get -o yaml + sed + apply` |
| Label missing | Not set at creation | `kubectl label service/pod <name> key=value` |