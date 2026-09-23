# template-ci-cd

Bibliothèque centrale de **workflows GitHub Actions réutilisables** pour la CI et la CD de tous mes projets.
Chaque projet appelle ces workflows au lieu de dupliquer sa propre configuration : une correction ou une mise à jour faite ici profite à tous les projets.

## Workflows disponibles

| Workflow | Rôle |
|---|---|
| [`node-ci.yml`](.github/workflows/node-ci.yml) | CI Node.js (npm / pnpm / yarn) : install, lint, typecheck, tests, build, matrice de versions |
| [`python-ci.yml`](.github/workflows/python-ci.yml) | CI Python avec uv : install, ruff, typecheck, pytest, matrice de versions |
| [`security.yml`](.github/workflows/security.yml) | Secrets (gitleaks), vulnérabilités et mauvaises configs (Trivy), revue des dépendances sur les PR |
| [`docker-build.yml`](.github/workflows/docker-build.yml) | Build d'image Docker (multi-arch possible), tags automatiques, push vers GHCR, scan Trivy |
| [`release.yml`](.github/workflows/release.yml) | Versionnage et CHANGELOG automatiques avec release-please (Conventional Commits) |
| [`deploy-render.yml`](.github/workflows/deploy-render.yml) | Déploiement sur Render via Deploy Hook, avec attente du statut `live` |

Chaque fichier documente ses `inputs`, `secrets` et `outputs` en tête.

## Utilisation dans un projet

Dans `<projet>/.github/workflows/ci.yml` :

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  ci:
    uses: thepja/template-ci-cd/.github/workflows/node-ci.yml@v1
    with:
      lint-command: npm run lint
      test-command: npm test

  security:
    uses: thepja/template-ci-cd/.github/workflows/security.yml@v1
```

Des pipelines complets prêts à copier sont dans [`examples/`](examples/) :

- [`examples/node-app.yml`](examples/node-app.yml) : CI → sécurité → release → image Docker → déploiement Render
- [`examples/python-app.yml`](examples/python-app.yml) : CI multi-versions → sécurité → image Docker

### Permissions

Un workflow réutilisable ne peut pas avoir plus de droits que le job appelant. Accorder dans le projet :

- `docker-build.yml` : `packages: write` (push GHCR)
- `release.yml` : `contents: write` et `pull-requests: write`

### Secrets à configurer dans les projets

| Secret | Utilisé par | Obligatoire |
|---|---|---|
| `RENDER_DEPLOY_HOOK` | `deploy-render.yml` (`deploy-hook-url`) | oui, pour déployer |
| `RENDER_API_KEY` | `deploy-render.yml` (`api-key`) | non : sans elle, pas d'attente du statut final |
| Licence gitleaks | `security.yml` (`gitleaks-license`) | seulement pour les dépôts d'organisation |

Le dépôt doit être **public**, ou, s'il est privé, autoriser l'accès sous *Settings → Actions → General → Access* pour que les autres dépôts puissent l'appeler.

## Versionnage de ce dépôt

Les projets référencent un tag majeur (`@v1`) :

- corrections et ajouts compatibles : on déplace le tag `v1` sur le nouveau commit (`git tag -f v1 && git push -f origin v1`) ;
- changement cassant (input renommé, comportement modifié) : nouveau tag `v2`.

Pour tester une modification avant de la publier, un projet peut pointer sur une branche : `@ma-branche`.

## Maintenance

- [`self-ci.yml`](.github/workflows/self-ci.yml) valide tous les workflows avec actionlint à chaque PR.
- Dependabot met à jour chaque semaine les versions des actions utilisées.

Pour valider en local : `actionlint .github/workflows/*.yml examples/*.yml`.
