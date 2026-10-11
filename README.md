# SwitchCompagnie/.github

Dépôt spécial de l'organisation : profil public (`profile/README.md`), CI commune et modèles de workflows.

## CI Laravel commune

Le workflow réutilisable [`.github/workflows/laravel-ci.yml`](.github/workflows/laravel-ci.yml) sert à tous les projets Laravel de l'organisation. Il s'adapte au projet : une étape dont l'outil n'est pas installé est ignorée.

| Job | Contenu |
| --- | --- |
| `Code Style (Pint)` | `vendor/bin/pint --test` (ignoré si Pint n'est pas installé) |
| `Static Analysis (PHPStan)` | Larastan/PHPStan, seulement si `vendor/bin/phpstan` et un `phpstan.neon(.dist)` existent |
| `Frontend Build (Vite)` | `npm ci && npm run build` si `package.json` existe, le build est réutilisé par les tests |
| `PHP x.y Tests` | matrice PHP, cache Composer, `.env` depuis `.env.ci` ou `.env.example`, `key:generate`, base SQLite/MySQL/MariaDB/PostgreSQL, `migrate`, `passport:keys` si Passport est installé, compilation des vues Blade, `php artisan test` (`--parallel` si Paratest est installé) |
| `Security Audit` | `composer audit` et `npm audit --audit-level=high`, non bloquants par défaut |
| `Secret Scan (Gitleaks)` | optionnel (`run-secret-scan: true`), bloquant |

Permissions en lecture seule, actions épinglées sur leur version majeure.

### Utiliser la CI dans un projet

Crée `.github/workflows/ci.yml` :

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:

concurrency:
  group: ci-${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

permissions:
  contents: read

jobs:
  laravel:
    name: Laravel
    uses: SwitchCompagnie/.github/.github/workflows/laravel-ci.yml@main
    with:
      php-versions: '["8.4", "8.5"]'
```

Pour un nouveau dépôt, le plus simple : onglet **Actions**, puis **New workflow**, puis **Laravel CI (SwitchCompagnie)** dans la section de l'organisation.

### Paramètres (`with:`)

| Paramètre | Défaut | Rôle |
| --- | --- | --- |
| `php-versions` | `["8.4", "8.5"]` | versions PHP testées (JSON) |
| `php-version-tools` | `8.5` | PHP pour Pint, PHPStan et l'audit |
| `node-version` | `22` | Node.js du build Vite |
| `extensions` | bcmath, ctype, curl, dom, fileinfo, gd, intl, mbstring, pcntl, pdo, pdo_sqlite, sqlite3, xml, zip | extensions PHP (le pilote PDO MySQL/MariaDB/PostgreSQL est ajouté tout seul) |
| `database` | `sqlite` | `sqlite`, `mysql` (8.4), `mariadb` (11.4) ou `pgsql` (17) |
| `working-directory` | `.` | dossier du projet Laravel |
| `env-file` | auto | fichier copié en `.env` |
| `run-pint` | `true` | style de code |
| `run-larastan` | `true` | analyse statique si installée |
| `run-tests` / `run-migrations` / `run-seed` | `true` / `true` / `false` | tests, migrations, seeders |
| `parallel-tests` | `true` | `--parallel` si Paratest est installé |
| `lint-blade` | `true` | compile et vérifie les vues Blade |
| `run-npm-build` | `true` | build Vite |
| `run-audit` | `true` | `composer audit` + `npm audit` |
| `composer-audit-blocking` / `npm-audit-blocking` | `false` | rendre l'audit bloquant |
| `npm-audit-level` | `high` | niveau minimal de `npm audit` |
| `run-secret-scan` | `false` | Gitleaks sur tout l'historique |
| `timeout-minutes` | `20` | durée maximale par job |

Dans l'onglet Checks, les jobs s'affichent sous la forme `Laravel / Code Style (Pint)`, `Laravel / PHP 8.5 Tests`, etc. Pense à mettre ces noms à jour si une protection de branche exige des checks.

### Dependabot

Copie [`examples/dependabot.yml`](examples/dependabot.yml) dans `.github/dependabot.yml` du projet : mises à jour hebdomadaires groupées pour Composer et npm, mensuelles pour les GitHub Actions.

### Faire évoluer la CI

Une modification de `laravel-ci.yml` sur `main` s'applique à tous les projets dès leur prochain run. Teste d'abord sur une branche en pointant un projet sur `@ma-branche`, puis merge.
