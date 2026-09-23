# template-ci-cd

Bibliothèque centrale de **workflows GitHub Actions réutilisables** (`workflow_call`) pour la CI et la CD de mes projets.
Chaque projet appelle ces workflows au lieu de dupliquer sa configuration : une correction faite ici profite à tous.

Projet de référence : [thepja/first-app](https://github.com/thepja/first-app) (Java + Angular, Docker, Helm, Render).

## Principes

- **Construire une fois, déployer partout** : l'image est construite une seule fois (`docker-build`), testée, scannée, puis ce même binaire est publié (`docker-publish`) et déployé.
- **Tags immuables** : on déploie `sha-xxxxxxx` ou `X.Y.Z`, jamais `latest`.
- **Chaîne d'approvisionnement** : actions épinglées par SHA (mises à jour par Dependabot), attestation de provenance SLSA des images (dépôts publics).
- **Déploiements sûrs** : `helm upgrade --atomic`, `helm test`, un seul déploiement à la fois par environnement, secrets portés par les environnements GitHub.
- **Permissions minimales** : chaque workflow déclare uniquement les droits dont il a besoin.

## Workflows

| Workflow | Rôle | Permissions à accorder |
|---|---|---|
| [`java-maven-ci.yml`](.github/workflows/java-maven-ci.yml) | Build Maven (`verify`), version depuis le tag, rapports et JAR en artefacts | — |
| [`node-ci.yml`](.github/workflows/node-ci.yml) | npm / pnpm / yarn : install, lint, typecheck, tests, build, matrice de versions | — |
| [`python-ci.yml`](.github/workflows/python-ci.yml) | uv : ruff, typecheck, pytest, matrice de versions | — |
| [`helm-lint.yml`](.github/workflows/helm-lint.yml) | `helm lint --strict` + kubeconform pour chaque fichier de valeurs | — |
| [`docker-build.yml`](.github/workflows/docker-build.yml) | Build de l'image, smoke test, scan Grype ou Trivy, image conservée en artefact | — |
| [`k8s-test.yml`](.github/workflows/k8s-test.yml) | Déploiement du chart sur un cluster kind éphémère, test de rolling update, `helm test` | — |
| [`docker-publish.yml`](.github/workflows/docker-publish.yml) | Push de l'image testée sur GHCR, tags auto, attestation SLSA ; sortie `tag` immuable | `packages: write`, `id-token: write`, `attestations: write` |
| [`deploy-helm.yml`](.github/workflows/deploy-helm.yml) | Déploiement Helm dans un environnement GitHub (secret `KUBE_CONFIG`) | — |
| [`deploy-render.yml`](.github/workflows/deploy-render.yml) | Déploiement Render via Deploy Hook (par commit ou par image), attente de la version | — |
| [`github-release.yml`](.github/workflows/github-release.yml) | GitHub Release sur un tag `v*`, notes générées, livrables joints | `contents: write` |
| [`release-please.yml`](.github/workflows/release-please.yml) | Versionnage et CHANGELOG automatiques (Conventional Commits) | `contents: write`, `pull-requests: write` |
| [`codeql.yml`](.github/workflows/codeql.yml) | Analyse CodeQL (ignorée sur les dépôts privés) | `security-events: write`, `actions: read` |
| [`security.yml`](.github/workflows/security.yml) | gitleaks, Trivy (dépendances et configs), dependency-review sur les PR | — |

Chaque fichier documente ses `inputs`, `secrets` et `outputs` en tête. Un workflow réutilisable ne peut pas avoir plus de droits que le job appelant : accorder les permissions de la dernière colonne sur le job qui l'appelle.

## Utilisation

```yaml
jobs:
  build:
    uses: thepja/template-ci-cd/.github/workflows/java-maven-ci.yml@v1
    with:
      artifact-path: target/mon-app.jar
```

Pipelines complets :

- [first-app `ci-cd.yml`](https://github.com/thepja/first-app/blob/HEAD/.github/workflows/ci-cd.yml) : Java + Angular → image → kind → GHCR → Render, staging, release et production ;
- [`examples/node-app.yml`](examples/node-app.yml) : Node → release-please → image → Render ;
- [`examples/python-app.yml`](examples/python-app.yml) : Python → image → GHCR.

### Secrets et variables (dans le dépôt appelant)

Les workflows de déploiement lisent les secrets de l'**environnement GitHub** qu'ils ciblent : appeler avec `secrets: inherit`.

| Nom | Type | Environnement | Utilisé par |
|---|---|---|---|
| `KUBE_CONFIG` | secret | `staging`, `production`… | `deploy-helm.yml` : kubeconfig en base64. Sans lui, déploiement ignoré avec un avertissement |
| `RENDER_DEPLOY_HOOK_URL` | secret | `render` | `deploy-render.yml`. Sans lui, déploiement ignoré avec un avertissement |
| `RENDER_SERVICE_URL` | variable | `render` | `deploy-render.yml` : vérification de la version en ligne (optionnelle) |
| `RENDER_API_KEY` | secret | `render` | `deploy-render.yml` : attente du statut `live` quand la version n'est pas vérifiable (optionnel) |

### Visibilité

Un dépôt **public** (comme first-app) ne peut appeler que des workflows d'un dépôt **public** : ce dépôt doit donc être public. Il ne contient aucun secret.
S'il devait rester privé, seuls des dépôts privés du même compte pourraient l'appeler, après avoir autorisé l'accès dans *Settings → Actions → General → Access*.

## Versionnage de ce dépôt

Les projets référencent un tag majeur (`@v1`) ou, pour une sécurité maximale, un SHA de commit avec le tag en commentaire (`@<sha> # v1.2.0`), mis à jour par Dependabot.

- évolution compatible : nouveau tag `v1.x.y` et déplacement de `v1` (`git tag -f v1 && git push -f origin v1`) ;
- changement cassant (input renommé ou supprimé, comportement modifié) : nouveau majeur `v2`.

Pour tester une modification avant publication, un projet peut pointer sur une branche : `@ma-branche`.

## Maintenance

- [`self-ci.yml`](.github/workflows/self-ci.yml) valide tous les workflows avec actionlint (inclut shellcheck) à chaque push et PR.
- Dependabot met à jour chaque semaine les actions utilisées.

En local : `actionlint .github/workflows/*.yml examples/*.yml`.
