# devops-app-bookstack

Application BookStack conteneurisée, déployée via GitOps sur Kubernetes.

## Description

BookStack est une plateforme de documentation wiki. Ce repo contient :
- Le Dockerfile pour builder une image custom
- Les manifests Kubernetes (Kustomize)
- Les workflows GitHub Actions (CI/CD)

L'application tourne sur deux environnements (dev/prod) dans des namespaces séparés sur le même cluster AKS.

## Repos liés

- [devops-iac-azure](https://github.com/idarius/devops-iac-azure) → infrastructure Azure + bootstrap ArgoCD
- [devops-platform-k8s](https://github.com/idarius/devops-platform-k8s) → composants plateforme (ingress, monitoring, etc.)

## Structure

```
├── app/
│   ├── Dockerfile              # Image basée sur linuxserver/bookstack
│   └── src/
│       └── rncp.css            # CSS custom injecté dans BookStack
├── k8s/
│   ├── base/                   # Manifests communs
│   │   ├── bookstack-deployment.yaml
│   │   ├── bookstack-service.yaml
│   │   ├── bookstack-pvc.yaml
│   │   ├── bookstack-secrets.secret.sops.yaml  # Secrets chiffrés
│   │   ├── mariadb-deployment.yaml
│   │   ├── mariadb-service.yaml
│   │   └── mariadb-pvc.yaml
│   └── overlays/
│       ├── dev/                # Overlay environnement dev
│       │   ├── kustomization.yaml
│       │   ├── ingress.yaml
│       │   ├── networkpolicy.yaml
│       │   └── resourcequota.yaml
│       └── prod/               # Overlay environnement prod
│           └── (même structure)
└── .github/workflows/
    ├── ci.yaml                 # Build + scan + push + deploy dev
    ├── promote-prod.yaml       # Promotion d'une image vers prod
    └── reset-fallback.yaml     # Reset vers l'image publique
```

## Pipeline CI/CD

### ci.yaml (sur push dans `app/`)

1. **Gitleaks** : scan des secrets dans le code
2. **Build** : construction de l'image Docker
3. **Trivy** : scan de vulnérabilités (bloque sur CRITICAL)
4. **Push** : envoi vers ACR si les scans passent
5. **GitOps** : mise à jour du tag dans `k8s/overlays/dev/kustomization.yaml`

ArgoCD détecte le changement et déploie automatiquement en dev.

### promote-prod.yaml (manuel)

Workflow déclenché manuellement pour promouvoir une image testée en dev vers prod :
```
Actions → Run workflow → image_tag: sha-xxxxx
```

L'image n'est pas rebuild : on réutilise exactement celle validée en dev.

### reset-fallback.yaml (manuel)

Remet l'image par défaut (linuxserver/bookstack) pour les cas où l'ACR est détruit (cycle destroy/apply de l'infra).

## Déploiement

Le déploiement est 100% GitOps :
1. Push sur `main` (dossier `app/`) → CI build et update l'overlay dev
2. ArgoCD sync l'overlay dev → déploiement en `bookstack-dev`
3. Validation manuelle
4. Workflow `promote-prod` → update l'overlay prod sur branche `prod`
5. ArgoCD sync l'overlay prod → déploiement en `bookstack-prod`

## Secrets

Les secrets (mots de passe DB, APP_KEY, credentials admin) sont dans `k8s/base/bookstack-secrets.secret.sops.yaml`, chiffrés avec SOPS/Age.

Pour modifier :
```bash
sops k8s/base/bookstack-secrets.secret.sops.yaml
```

## Accès

| Environnement | URL |
|---------------|-----|
| Dev | https://bookstackdev.rncp.idarius.net |
| Prod | https://bookstackprod.rncp.idarius.net |

Les certificats TLS sont générés automatiquement par cert-manager (Let's Encrypt).

## Composants déployés

Par namespace :
- **bookstack** : 1 pod (PHP/Apache)
- **mariadb** : 1 pod (base de données)
- **PVC** : 2 volumes persistants (config BookStack + data MariaDB)
- **Ingress** : exposition via Traefik
- **NetworkPolicy** : isolation inter-namespaces
- **ResourceQuota** : limites de ressources par namespace
