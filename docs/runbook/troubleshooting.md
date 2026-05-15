# Troubleshooting runbook

## Opći postupak dijagnostike

```bash
kubectl get pods -n ticketing
kubectl get events -n ticketing
kubectl logs <pod> -n ticketing
kubectl describe pod <pod> -n ticketing
```

## Incident 1 — Pad baze podataka

**Simptomi:** API vraća 500, postgres pod u CrashLoopBackOff

**Dijagnoza:**
```bash
kubectl get pods -n ticketing
kubectl logs deploy/postgres -n ticketing
```

**Oporavak:**
```bash
kubectl rollout restart deployment/postgres -n ticketing
```

**Validacija:**
```bash
kubectl exec -n ticketing deploy/postgres -- pg_isready -U ticketing_user
curl http://localhost:8080/healthz
```

---

## Incident 2 — ImagePullBackOff

**Simptomi:** novi podovi ostaju u ImagePullBackOff nakon deploymenta

**Dijagnoza:**
```bash
kubectl describe pod -n ticketing -l app=api
```

**Oporavak:**
```bash
kubectl rollout undo deployment/api -n ticketing
```

**Validacija:**
```bash
kubectl rollout history deployment/api -n ticketing
kubectl get pods -n ticketing -l app=api
curl http://localhost:8080/healthz
```

---

## Incident 3 — Neispravan Secret

**Simptomi:** API i worker se ruše s greškama autentikacije prema bazi

**Dijagnoza:**
```bash
kubectl logs deploy/api -n ticketing --tail=20
kubectl get secret ticketing-secrets -n ticketing
```

**Oporavak:**
```bash
kubectl delete secret ticketing-secrets -n ticketing
kubectl create secret generic ticketing-secrets \
  --from-literal=POSTGRES_PASSWORD=ispravna_lozinka \
  --from-literal=POSTGRES_USER=ticketing_user \
  --from-literal=POSTGRES_DB=ticketing \
  -n ticketing
kubectl rollout restart deployment/api deployment/worker -n ticketing
```

**Validacija:**
```bash
kubectl get pods -n ticketing
kubectl logs deploy/api -n ticketing --tail=20
```

---

## Incident 4 — Liveness probe pada

**Simptomi:** pod u CrashLoopBackOff ubrzo nakon pokretanja

**Dijagnoza:**
```bash
kubectl describe pod -n ticketing -l app=api
```

**Oporavak:** povećaj `initialDelaySeconds` ili `failureThreshold` u manifestu, zatim:
```bash
kubectl apply -f infra/k8s/20-api.yaml -n ticketing
```

**Validacija:**
```bash
kubectl describe pod -n ticketing -l app=api
kubectl get pods -n ticketing
```

---

## Incident 5 — Worker ne procesira narudžbe

**Simptomi:** narudžbe ostaju u Redis queueu, status nikad nije `processed`

**Dijagnoza:**
```bash
kubectl logs deploy/worker -n ticketing --tail=50
kubectl exec -n ticketing deploy/redis -- redis-cli llen ticket_orders
```

**Oporavak:**
```bash
kubectl rollout restart deployment/worker -n ticketing
```

**Validacija:**
```bash
kubectl exec -n ticketing deploy/redis -- redis-cli llen ticket_orders
curl http://localhost:8080/tickets/orders
```

---

## Incident 6 — OOMKilled

**Simptomi:** pod se restartira s razlogom OOMKilled

**Dijagnoza:**
```bash
kubectl describe pod -n ticketing -l app=api
```

**Oporavak:** povećaj `resources.limits.memory` u manifestu, zatim:
```bash
kubectl apply -f infra/k8s/20-api.yaml -n ticketing
```

**Validacija:**
```bash
kubectl get pods -n ticketing
kubectl top pods -n ticketing
```
