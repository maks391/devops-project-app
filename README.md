# Secure Event Ticketing Platform

Projekt za kolegij **Uvod u DevOps - DevSecOps** (Sveučilište Algebra Bernays, 2026).
Demonstrira cijeli DevOps/DevSecOps ciklus: lokalni razvoj kroz Compose i produkcijski deployment kroz Kubernetes manifeste.

---

## Sadržaj

- [Arhitektura](#arhitektura)
- [CI/CD Pipeline](#cicd-pipeline)
- [DevSecOps](#devsecops)
- [Kontejneri vs. Virtualne Mašine](#kontejneri-vs-virtualne-mašine)
- [Lokalni razvoj](#lokalni-razvoj)
- [Validacija funkcionalnosti](#validacija-funkcionalnosti)
- [Sigurnosni elementi](#sigurnosni-elementi)

---

## Arhitektura

### Servisi i njihove uloge

| Servis       | Tehnologija               | Uloga                                                                                                                                                                |
|--------------|---------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `frontend`   | Node.js / Express + nginx | Korisničko sučelje za pregled evenata i kupnju karata. U dev modu pokreće Express server s hot-reloadom; u produkciji statičke datoteke servira nginx reverse proxy. |
| `api`        | Node.js / Express         | REST API za upravljanje eventima, narudžbama i health provjerama. Prima zahtjeve od frontenda i zapisuje narudžbe u PostgreSQL te ih šalje u Redis queue.            |
| `worker`     | Node.js                   | Pozadinska obrada poruka iz Redis queuea. Procesira narudžbe asinkrono kako bi API ostao responzivan.                                                                |
| `postgres`   | PostgreSQL 16             | Trajna relacijska pohrana narudžbi i stanja evenata.                                                                                                                 |
| `redis`      | Redis 7                   | Message queue između API-ja i Workera; može se koristiti i kao cache sloj.                                                                                           |

### Međuservisna komunikacija

```
Browser
   │
   ▼
[frontend :3000]
   │  /api/* proxy (dev: API_BASE_URL, prod: nginx proxy_pass)
   ▼
[api :8080] ──── PostgreSQL :5432  (narudžbe, upiti)
   │
   └──────────── Redis :6379       (RPUSH ticket_orders)
                      │
                 [worker]          (BLPOP ticket_orders → obradjuje → UPDATE postgres)
```

Svi servisi komuniciraju unutar izoliranog Docker/Podman mrežnog mosta `app-network`. Izvana su dostupni samo `frontend` (port 3000) i `api` (port 8080).

---

## Kontejneri vs. Virtualne Mašine

### Zašto kontejneri za ovaj projekt?

| Kriterij                  | Kontejneri (Podman/Docker)                          | Virtualne Mašine                                   |
|---------------------------|-----------------------------------------------------|----------------------------------------------------|
| **Veličina**              | Slike 50–200 MB (alpine baza)                       | Slika 1–20 GB (cijeli OS)                          |
| **Pokretanje**            | < 1 sekunda                                         | 30–120 sekundi (boot OS-a)                         |
| **Izolacija**             | Dijele kernel hosta; namespace + cgroup izolacija   | Potpuna kernel izolacija (hypervisor)              |
| **Potrošnja resursa**     | Minimalna (nema OS overhead-a po servisu)           | Svaka VM troši RAM/CPU za vlastiti OS              |
| **Portabilnost**          | Isti image radi lokalno, CI, produkcija             | VM image ovisi o hypervisoru i hardveru            |
| **Reproduktivnost**       | Deterministički build (Containerfile + lockfile)    | Teže garantirati identično okruženje               |
| **Skalabilnost**          | Kubernetes može pokrenuti stotine replica u sekundi | Sporije skaliranje zbog VM boota                   |
| **DevSecOps integracija** | Trivy skeniranje, image signing, layer audit        | Agent-based skeniranje, teže automatizirati        |

### Zašto su kontejneri bolji izbor za Secure Event Ticketing Platform?

1. **Mikroservisna arhitektura** — pet servisa s različitim runtime okruženjima (Node.js, PostgreSQL, Redis) lakše se kontejnerizira nego što se za svaki pokreće zasebna VM.
2. **Brzi development loop** — hot-reload s volume mountovima (`./src:/app/src`) znači da developer vidi promjene u milisekundama, ne minutama.
3. **Uniformnost** — isti `compose.yaml` radi na macOS, Windows (WSL2) i Linux bez prilagodbi.
4. **Sigurnosni pregled slika** — alati poput Trivyja skeniraju svaki layer image-a; kod VM-ova je to znatno kompleksnije.
5. **Kubernetes kompatibilnost** — produkcijski deploy koristi iste image-ove, samo ih orchestrira Kubernetes umjesto Compose-a.

### Kad su VM-ovi bolji izbor?

VM-ovi su prikladniji kad je potrebna potpuna kernel izolacija (npr. multi-tenant cloud, sigurnosno-kritični workloadi koji zahtijevaju fizičku separaciju), ili kad aplikacija zahtijeva specifični OS kernel koji se razlikuje od hosta.

---

## Lokalni razvoj

### Preduvjeti

- [Podman](https://podman.io/) >= 4.x s `podman-compose`, ili Docker >= 24.x s Docker Compose v2
- Git

### Postavljanje okruženja

```bash
# 1. Kloniraj repozitorij
git clone <repo-url>
cd devops-project-app

# 2. Kopiraj primjer konfiguracije
cp .env.example .env
# Pregledaj .env i prilagodi po potrebi (lozinke, portovi)
```

### Pokretanje (jednom naredbom)

```bash
podman-compose up --build
```

ili s Dockerom:

```bash
docker compose up --build
```

Stack pokreće sve servise u ispravnom redoslijedu (PostgreSQL i Redis prvo, API kad su baze zdrave, Frontend kad je API zdrav).

### Hot-reload (development)

Compose je konfiguriran s `target: dev` za frontend, API i Worker. Volumeni montiraju lokalni `src/` direktorij u kontejner:

- Promjena datoteke u `api/src/` → nodemon automatski restartira API proces
- Promjena datoteke u `worker/src/` → nodemon automatski restartira Worker
- Promjena datoteke u `frontend/src/` → nodemon automatski restartira Express dev server

Nema potrebe za `podman-compose restart` pri razvoju.

### Zaustavljanje stacka

```bash
# Zaustavi sve servise (zadrži volumene/podatke)
podman-compose down

# Zaustavi i ukloni volumene (briše podatke iz baze)
podman-compose down -v
```

### Korisne naredbe za razvoj

```bash
# Prati logove svih servisa
podman-compose logs -f

# Prati logove samo API-ja
podman-compose logs -f api

# Pokreni shell unutar API kontejnera
podman-compose exec api sh

# Provjeri status svih servisa
podman-compose ps
```

---

## Validacija funkcionalnosti

### Health provjere

```bash
curl http://localhost:8080/healthz
curl http://localhost:8080/readyz
```

### Dohvat evenata

```bash
curl http://localhost:8080/events
```

### Kupnja karata

```bash
curl -X POST http://localhost:8080/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{"eventId":"evt-1001","customerEmail":"student@example.com","quantity":2}'
```

### Pregled narudžbi

```bash
curl http://localhost:8080/tickets/orders
```

### Web sučelje

Otvori `http://localhost:3000` u pregledniku.

---

## Sigurnosni elementi

### Containerfile prakse (I2)

- **Multi-stage build** — builder stage instalira dependencies, runtime stage kopira samo potrebno; dev tools ne ulaze u produkcijsku sliku
- **Minimalna baza** — `node:20-alpine` i `nginx:1.27-alpine` (~50 MB vs ~900 MB za debian varijante)
- **Non-root korisnik** — svi servisi pokreću se kao `appuser` (uid != 0) ili ugrađeni `node` user
- **`apk upgrade`** — runtime stage primjenjuje najnovije sigurnosne zakrpe Alpine paketa pri buildu

### Upravljanje tajnama

- Lozinke i connection stringovi definirani su u `.env` (nije commitano u repozitorij)
- `.env.example` sadrži template s placeholder vrijednostima za onboarding novih developera

### Skeniranje ranjivosti

Trivy skeniranje image-ova dokumentirano je u [`docs/security/image-scan-report.md`](docs/security/image-scan-report.md).

### Produkcija (Kubernetes)

- Secret i ConfigMap objekti (bez hardkodiranih lozinki u manifestima)
- Liveness i Readiness probe za sve ključne servise
- Resource requests i limits
- ServiceAccount + RBAC
- NetworkPolicy segmentacija prometa
- Ingress / Route za vanjski pristup

---

## CI/CD Pipeline

GitHub Actions workflow (`.github/workflows/ci.yml`) automatski se pokreće pri svakom pushu na `main` i pri otvaranju PR-a.

### Tijek pipeline-a

```
push/PR
  │
  ├── build-and-test   → build svih runtime slika + smoke test API/frontend
  │
  ├── security-scan    → Trivy skeniranje svake slike
  │     CRITICAL → blokira pipeline
  │     HIGH     → upozorenje + JSON artefakt
  │
  └── publish          → push na GHCR (samo main, samo ako prethodni koraci prošli)
        :latest
        :sha-<commit>
```

### Objavljene slike (GHCR)

```
ghcr.io/matej-basic/devops-project-app/api:latest
ghcr.io/matej-basic/devops-project-app/api:sha-<commit>
ghcr.io/matej-basic/devops-project-app/worker:latest
ghcr.io/matej-basic/devops-project-app/worker:sha-<commit>
ghcr.io/matej-basic/devops-project-app/frontend:latest
ghcr.io/matej-basic/devops-project-app/frontend:sha-<commit>
```

Detaljan deployment postupak: [`docs/deployment/deployment.md`](docs/deployment/deployment.md)

---

## DevSecOps

Projekt primjenjuje DevSecOps principe kroz:

| Praksa                   | Implementacija                                               | 
|--------------------------|--------------------------------------------------------------|
| Shift-left skeniranje    | Trivy u CI pri svakom buildu                                 |
| Quality gate             | CRITICAL ranjivost blokira objavu slike                      |
| Minimalna attack surface | Multi-stage build, samo prod ovisnosti u runtime slici       |
| Non-root kontejneri      | Svi servisi pokreću se kao `appuser`                         |
| Reproducibilni buildovi  | `npm ci` + `package-lock.json`                               |
| Secrets bez hardcodinga  | GitHub Secrets u CI, `.env` lokalno, K8s Secret u produkciji |
| Audit trail              | Trivy JSON artefakti (30 dana), SHA image tagovi             |

Detaljna metodologija: [`docs/devsecops/methodology.md`](docs/devsecops/methodology.md)  
Izvještaj skeniranja: [`docs/security/image-scan-report.md`](docs/security/image-scan-report.md)

---

## Kubernetes deployment (produkcija)

Manifesti se nalaze u `infra/k8s/` i primjenjuju se u numeričkom redoslijedu:

```bash
kubectl apply -f infra/k8s/ -n ticketing
```

| Manifest                 | Sadržaj                                                         |
|--------------------------|-----------------------------------------------------------------|
| `00-namespace.yaml`      | Namespace `ticketing`                                           |
| `01-configmap.yaml`      | Nekritična konfiguracija (hostovi, portovi)                     |
| `02-secret.yaml`         | Upute za kreiranje Secreta s lozinkama                          |
| `03-serviceaccount.yaml` | ServiceAccount `ticketing-app` (least-privilege)                |
| `04-rbac.yaml`           | Role + RoleBinding                                              |
| `05-networkpolicy.yaml`  | Segmentacija prometa (deny-all + explicitni allow)              |
| `10-postgres-pvc.yaml`   | PersistentVolumeClaim (5Gi)                                     |
| `11-postgres.yaml`       | PostgreSQL Deployment + Service + init ConfigMap                |
| `12-redis.yaml`          | Redis Deployment + Service                                      |
| `20-api.yaml`            | API Deployment (2 replike, liveness/readiness probe) + Service  |
| `21-worker.yaml`         | Worker Deployment (1 replika)                                   |
| `22-frontend.yaml`       | Frontend Deployment (2 replike, liveness probe) + Service       |
| `30-ingress.yaml`        | Ingress (nginx) — `ticketing.local`                             |

Detaljan deployment postupak: [`docs/deployment/deployment.md`](docs/deployment/deployment.md)

---

## Troubleshooting runbook

Runbook pokriva 6 incidentnih scenarija s dijagnostikom, analizom uzroka i validacijom:

1. Pad baze podataka
2. ImagePullBackOff — neispravan image tag
3. Neispravan Secret — greška autentikacije
4. API liveness probe pada — CrashLoopBackOff
5. Worker ne procesira narudžbe — queue backlog
6. OOMKilled — prekoračenje memory limita

Runbook: [`docs/runbook/troubleshooting.md`](docs/runbook/troubleshooting.md)
