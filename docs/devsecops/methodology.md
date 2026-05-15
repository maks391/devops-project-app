# DevSecOps metodologija

Sigurnost je ugrađena na tri razine: kontejnerska slika, pipeline i infrastruktura.

## Razina 1 — Kontejnerska slika

- **Multi-stage build** — runtime slika ne sadrži razvojne alate
- **node:20-alpine** bazna slika (~10 MB Alpine, minimalna attack surface)
- **Non-root korisnik** — svi procesi izvršavaju se kao `appuser`
- **apk upgrade** — najnovije sigurnosne zakrpe pri svakom buildu

## Razina 2 — Pipeline

| Provjera                       | Alat          | Blokirajuća        |
|--------------------------------|---------------|--------------------|
| Build runtime slike            | Docker Buildx | Da                 |
| Skeniranje CRITICAL ranjivosti | Trivy         | Da — exit-code 1   |
| Skeniranje HIGH ranjivosti     | Trivy         | Ne — JSON artefakt |
| Smoke test health endpointa    | curl          | Da                 |
| Smoke test poslovnog tijeka    | curl + python | Da                 |
| Objava slika                   | GHCR          | Uvjetna            |

## Razina 3 — Infrastruktura

- **Secrets** — nikad hardkodirani; lokalno `.env`, produkcija Kubernetes Secret
- **RBAC** — prazna pravila (least-privilege, servisi ne trebaju K8s API pristup)
- **NetworkPolicy** — deny-all + eksplicitni allow samo za legitimne tokove
- **ServiceAccount** — automountanje tokena onemogućeno

## Zašto Trivy?

Trivy je odabran zbog:
- Native integracije s GitHub Actions (`aquasecurity/trivy-action`)
- Skeniranja OS paketa i npm ovisnosti u jednom koraku
- JSON izlaza pogodnog za artefakt pohranu i automatsku analizu
- Open-source licence bez ograničenja za CI/CD upotrebu
