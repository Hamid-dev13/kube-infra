# Repo infra — plateforme du cluster KUBE

Tout l'outillage du cluster (GitOps, monitoring, logs) est décrit ici et appliqué par
**Argo CD**. Rien ne s'installe à la main, à l'exception du bootstrap ci-dessous.

## Structure

```
infra/
├── ansible/                   rôle qui joue le bootstrap (à brancher sur le playbook du cluster)
│   ├── bootstrap-platform.yml
│   └── roles/platform_bootstrap/
├── bootstrap/                 appliqué UNE fois, par le rôle Ansible
│   ├── argocd-values.yaml       réglages d'installation d'Argo CD
│   └── root-app.yaml            Application racine (pattern app-of-apps)
├── apps/                      surveillé par l'Application racine
│   ├── kustomization.yaml       liste des composants installés
│   ├── monitoring.yaml          1 fichier = 1 Application = 1 composant
│   └── logs.yaml
└── components/                le contenu de chaque composant (kustomize + charts Helm)
    ├── monitoring/              kube-prometheus-stack + alertes maison
    └── logs/                    Loki + Alloy
```

Chaîne de déploiement : `root` lit `apps/` et crée une Application par fichier ; chaque
Application lit son dossier dans `components/` et le déploie dans son namespace.

## Bootstrap

### Par Ansible (cas normal)

Le rôle `platform_bootstrap` joue les 3 étapes ci-dessous sur un nœud du control plane,
une fois le cluster monté. Il est idempotent : le rejouer ne change rien.

```yaml
# à la fin du playbook du cluster
- import_playbook: <chemin vers ce repo>/infra/ansible/bootstrap-platform.yml
```

Prérequis sur le contrôleur Ansible : `ansible-galaxy collection install kubernetes.core`.
Le rôle installe lui-même `helm` et `python3-kubernetes` sur le nœud (Debian/Ubuntu).

| Variable | Défaut | Rôle |
|---|---|---|
| `platform_target` | `control_plane` | groupe d'inventaire visé (un seul nœud est utilisé) |
| `platform_kubeconfig` | `/etc/kubernetes/admin.conf` | kubeconfig admin sur ce nœud (kubeadm) |
| `platform_grafana_admin_password` | vide = généré une fois | à mettre dans le vault Ansible |
| `platform_install_deps` | `true` | `false` si `python3-kubernetes` est déjà installé |

Le rôle se termine en attendant que `root` soit `Synced` / `Healthy`.

### À la main (labo)

Prérequis : un cluster joignable par `kubectl`, et `helm`.

Argo CD lit ce repo depuis https://github.com/Hamid-dev13/kube-infra (public : aucun
identifiant à fournir). Ne jamais y commiter de secret en clair.

```bash
# 1. Argo CD (version du chart fixée pour que l'installation soit reproductible)
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd --version 10.11.1 \
  -n argocd --create-namespace -f infra/bootstrap/argocd-values.yaml --wait

# 2. Mot de passe admin de Grafana — provisoire, deviendra un SealedSecret
kubectl create namespace monitoring
kubectl -n monitoring create secret generic grafana-admin \
  --from-literal=admin-user=admin --from-literal=admin-password="$(openssl rand -base64 18)"

# 3. Application racine : à partir d'ici, tout vient de Git
kubectl apply -f infra/bootstrap/root-app.yaml
```

Vérifier : `kubectl -n argocd get applications` doit afficher `root`, `monitoring` et
`logs` en `Synced` / `Healthy` (compter quelques minutes au premier déploiement).

## Ajouter ou modifier un composant

- **Modifier** : éditer les values dans `components/<composant>/`, commiter. Argo CD
  applique le changement (interrogation de Git toutes les 3 min, ou bouton REFRESH).
- **Ajouter** : créer `components/<nouveau>/kustomization.yaml`, une Application
  `apps/<nouveau>.yaml` (sur le modèle de `logs.yaml`), et l'ajouter à `apps/kustomization.yaml`.
- **Retirer** : supprimer le fichier de `apps/` et sa ligne. Grâce au finalizer, Argo CD
  supprime aussi tout ce que l'Application avait déployé.
- **Prévisualiser** ce qu'un composant va créer, sans rien appliquer :
  `kubectl kustomize --enable-helm infra/components/<composant>`

## Choix d'outils

| Outil | Pourquoi |
|---|---|
| **Argo CD** (plutôt que Flux) | Son interface montre en direct la synchro, la santé et le diff de chaque ressource, ce qui rend la démo lisible. Il gère l'OIDC nativement ; Flux n'a pas d'interface intégrée. |
| **kustomize + `helmCharts`** | Le sujet impose kustomize et les outils sont distribués en charts Helm officiels : un dossier par composant, avec un rendu vérifiable en local avant tout commit. |
| **kube-prometheus-stack** (plutôt que VictoriaMetrics) | Le standard, livré avec Grafana, des dashboards et des alertes prêts à l'emploi ; ses CRDs (`ServiceMonitor`, `PrometheusRule`) permettent d'ajouter métriques et alertes par Git. VictoriaMetrics, plus léger, reste le plan B si la RAM manque. |
| **Loki en mode SingleBinary** | Un seul pod (~120 Mi) avec stockage disque : les modes distribués visent de gros volumes, pas 3 nœuds de ~4 Go. |
| **Alloy** (plutôt que Promtail) | Promtail est déprécié par Grafana au profit d'Alloy. Un pod par nœud lit `/var/log/pods` et ne remonte que les pods de son nœud, pour éviter les doublons. |

## Alertes maison

| Alerte | Déclenchement | Pourquoi |
|---|---|---|
| `PodNonPret` | Un pod `Running` mais pas `Ready` depuis 2 min | Rend visible une version défaillante qui ne prend pas le trafic. L'équivalent du chart (`KubePodNotReady`) attend 15 min. |

## Consommation mesurée (labo, au repos)

| Composant | RAM |
|---|---|
| Argo CD | ~350–490 Mi |
| Monitoring (Prometheus, Grafana, Alertmanager, exporters) | ~930 Mi |
| Logs (Loki + 3 Alloy) | ~385 Mi |

## Accès (en attendant l'Ingress et l'OIDC)

```bash
kubectl -n argocd port-forward svc/argocd-server 8080:443        # https://localhost:8080
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d

kubectl -n monitoring port-forward svc/kps-grafana 3001:80        # http://localhost:3001
kubectl -n monitoring get secret grafana-admin -o jsonpath='{.data.admin-password}' | base64 -d
```

## Reste à faire pour le cluster AWS

- [x] `repoURL` pointés sur le repo public `Hamid-dev13/kube-infra` (le repo Epitech est
      privé et une deploy key demande des droits admin que le groupe n'a pas).
- [x] Rôle Ansible `platform_bootstrap` (testé sur le labo k3d, idempotent).
- [ ] Le brancher à la fin du playbook du cluster (`import_playbook`) et tester sur AWS.
- [ ] Réactiver `kubeEtcd`, `kubeControllerManager`, `kubeScheduler`, `kubeProxy` dans
      `components/monitoring/values.yaml` (désactivés car k3s les embarque), et faire
      exposer leurs métriques dans la configuration kubeadm.
- [ ] Exposer Argo CD et Grafana en HTTPS (Ingress/Gateway + cert-manager).
- [ ] OIDC via dex sur Argo CD et Grafana ; décider si Argo CD garde son dex embarqué.
- [ ] `grafana-admin` en SealedSecret.
- [ ] Stockage de Loki : vérifier la StorageClass disponible (EFS limité à 20 Go pour le groupe).
