# Deployment upute

## Lokalni razvoj

```bash
cp .env.example .env
podman-compose up --build
```

Validacija:
```bash
curl http://localhost:8080/healthz
curl http://localhost:3000/healthz
```

## Produkcija (Kubernetes)

### Preduvjeti

- Kubernetes klaster (minikube, k3s, ili managed)
- kubectl konfiguriran
- Pristup GitHub Container Registry

### Redoslijed primjene

```bash
kubectl apply -f infra/k8s/00-namespace.yaml
kubectl apply -f infra/k8s/01-configmap.yaml
```

Kreirati Secret s ispravnim vrijednostima (ne commitati!):
```bash
kubectl create secret generic ticketing-secrets \
  --from-literal=POSTGRES_PASSWORD=tvoja_lozinka \
  --from-literal=POSTGRES_USER=ticketing_user \
  --from-literal=POSTGRES_DB=ticketing \
  -n ticketing
```

Primjeniti ostale manifeste:
```bash
kubectl apply -f infra/k8s/ -n ticketing
```

### Validacija deploymenta

```bash
kubectl get pods -n ticketing
curl http://ticketing.local/healthz
```

### Rolling update

```bash
kubectl rollout status deployment/api -n ticketing
```

### Rollback

```bash
kubectl rollout undo deployment/api -n ticketing
kubectl rollout history deployment/api -n ticketing
```
