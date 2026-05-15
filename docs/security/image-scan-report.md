# Image Scan Report — Trivy

## Metodologija

Skeniranje se provodi Trivy alatom integriranim u GitHub Actions pipeline. Svaka runtime slika skenira se nakon izgradnje, prije objave u registar.

- **CRITICAL** ranjivosti → blokiraju pipeline (exit-code 1)
- **HIGH** ranjivosti → generiraju JSON artefakt, ne blokiraju

## Rezultati skeniranja

### api (node:20-alpine)

| Razina   | Broj |
|----------|------|
| CRITICAL |   0  |
| HIGH     |  11  |
| MEDIUM   |   2  |
| LOW      |   2  |

Pipeline prolazi quality gate (CRITICAL: 0).

### worker (node:20-alpine)

| Razina   | Broj |
|----------|------|
| CRITICAL |   0  |
| HIGH     |  11  |

### frontend (node:20-alpine)

| Razina   | Broj |
|----------|------|
| CRITICAL |   0  |
| HIGH     |  11  |

## Poznate HIGH ranjivosti i plan sanacije

| Paket           | CVE            | Plan                                            |
|-----------------|----------------|-------------------------------------------------|
| cross-spawn     | CVE-2024-21538 | Nadogradnja na 7.0.5+                           |
| glob            | CVE-2025-64756 | Nadogradnja na 11.1.0+                          |
| Ostale bez fixa |        -       | Prihvaćen rizik, reskeniranje pri svakom buildu |

Dugoročna mjera: uvođenje `npm audit --fix` kao koraka u pipeline prije izgradnje runtime slike.
