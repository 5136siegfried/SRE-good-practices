# Tester ses GitHub Actions en local avec `act`

> **TL;DR** : `act` exécute tes workflows GitHub Actions dans des containers Docker locaux. Tu valides ta CI avant de pousser, sans polluer l'historique de runs GitHub et sans attendre les runners.

---

## Pourquoi act ?

Le cycle classique sans `act` ressemble à ça :

```
modifier le workflow → git push → attendre 2-3 min → lire les logs → corriger → recommencer
```

Avec `act` :

```
modifier le workflow → act push → lire les logs localement → corriger
```

Pas de commit inutile, pas de runners qui tournent pour rien, feedback en quelques secondes.

---

## Installation

```bash
# macOS
brew install act

# Linux
curl -s https://raw.githubusercontent.com/nektos/act/master/install.sh | sudo bash

# Vérifier
act --version
```

`act` nécessite Docker en cours d'exécution.

---

## Premiers pas

```bash
# Depuis la racine du repo
# Lister les workflows et jobs disponibles
act --list

# Simuler un push sur main (déclenche les workflows `on: push`)
act push

# Simuler une pull request
act pull_request

# Lancer un job spécifique
act push --job lint
act push --job deploy
```

---

## Gestion des secrets

Pour les workflows qui utilisent `${{ secrets.XXX }}`, créer un fichier `.secrets` à la racine (gitignored) :

```bash
cat > .secrets << 'EOF'
VPS_HOST=1.2.3.4
VPS_USER=deploy
VPS_PORT=2222
VPS_SSH_KEY=
EOF
```

```bash
act push --secret-file .secrets
```

Ne jamais commiter `.secrets`. L'ajouter au `.gitignore` :

```
.secrets
```

---

## Apple Silicon (M1/M2/M3)

Par défaut, `act` peut avoir des problèmes sur ARM. Forcer l'architecture x86 :

```bash
act push --container-architecture linux/amd64
```

Ou le mettre dans un fichier de config `.actrc` à la racine :

```
--container-architecture linux/amd64
```

---

## Images Docker

`act` propose trois niveaux d'images Ubuntu :

| Image | Taille | Usage |
|-------|--------|-------|
| `catthehacker/ubuntu:act-latest` | ~1.5GB | Défaut, bon équilibre |
| `catthehacker/ubuntu:full-latest` | ~17GB | Très proche de GitHub runners |
| `node:16-buster-slim` | ~200MB | Ultra léger, dépendances minimales |

Si un job plante en local mais passe sur GitHub, c'est souvent un outil manquant dans l'image légère. Passer sur `full-latest` pour déboguer.

---

## Cas d'usage SRE

### Valider un playbook Ansible avant merge

```bash
act push --job lint        # yamllint + ansible-lint
act push --job check       # syntax-check ansible
```

### Tester un workflow de déploiement sans toucher le VPS

```bash
act push --job deploy --secret-file .secrets --dry-run
```

### Rejouer uniquement ce qui a changé

```bash
act push --job lint --reuse   # réutilise le container existant, plus rapide
```

### Déboguer un job qui plante

```bash
act push --job deploy -v       # verbose
act push --job deploy --artifact-server-path /tmp/artifacts
```

---

## Limitations connues

- Les **services Docker** (`services:` dans le workflow) ne sont pas toujours bien supportés
- Les **actions GitHub Marketplace** nécessitent une connexion internet pour être téléchargées
- Le **cache GitHub Actions** (`actions/cache`) ne fonctionne pas en local
- Certains **contextes GitHub** (`github.sha`, `github.ref`) ont des valeurs factices
- Le job `deploy` vers un vrai VPS échouera sur le SSH — c'est normal et attendu en local

---

## Intégration dans le workflow SRE

```
Feature branch
    │
    ├── act push --job lint      ← avant chaque commit (rapide, < 30s)
    │
    ├── act push                 ← avant chaque PR (tous les jobs)
    │
    └── git push / PR → GitHub Actions (validation finale sur vrais runners)
```

Utiliser `act` pour la boucle rapide, laisser GitHub Actions pour la validation officielle avant merge.

---

## Ressources

- [nektos/act — GitHub](https://github.com/nektos/act)
- [Documentation act](https://nektosact.com)
- [catthehacker/ubuntu images](https://github.com/catthehacker/docker_images)
