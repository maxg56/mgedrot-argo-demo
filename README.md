# mgedrot-argo-demo

Démo de déploiement **GitOps** avec [Argo CD](https://argo-cd.readthedocs.io/).

Ce dépôt sert de source de vérité : il contient uniquement les manifestes Kubernetes d'une petite application. Argo CD surveille le dépôt et applique automatiquement sur le cluster tout changement poussé sur la branche `main`. Pour déployer une nouvelle version, il suffit donc de modifier le tag de l'image dans `deployment.yaml` et de pousser le commit : aucune commande `kubectl apply` n'est nécessaire.

## Contenu du dépôt

### `deployment.yaml`

Un `Deployment` (`apps/v1`) nommé `myapp`, dans le namespace `dev` :

- 1 réplica ;
- un conteneur `myapp` basé sur l'image `wil42/playground:v2` ;
- le conteneur écoute sur le port `8888` ;
- les pods portent le label `app: myapp`.

C'est le tag de l'image (`v1` ou `v2`) qu'on modifie pour changer de version. L'historique git montre cette bascule (`update to v1`, `update to v2`, ...). Un essai avec `v3` a été annulé car cette image n'existe pas.

### `service.yaml`

Un `Service` (`v1`) nommé `myapp-service`, dans le namespace `dev` :

- type `NodePort` ;
- sélectionne les pods `app: myapp` ;
- expose le port `8888` (vers le `targetPort` `8888` du conteneur) sur le `nodePort` `30099` de chaque nœud.

## Flux de synchronisation

```
 développeur ──git push──▶ GitHub (main)
                                │
                                │  Argo CD compare l'état du dépôt
                                ▼  à l'état du cluster
                           Argo CD (namespace argocd)
                                │
                                │  synchronisation (kubectl apply)
                                ▼
                    cluster ── namespace dev ── Deployment myapp
                                              └─ Service myapp-service (NodePort 30099)
```

1. On modifie un manifeste (par exemple `image: wil42/playground:v1`) et on pousse sur `main`.
2. Argo CD détecte que le dépôt diffère de ce qui tourne dans le cluster (application `OutOfSync`).
3. Avec la synchronisation automatique, Argo CD applique les manifestes : le `Deployment` lance un nouveau pod avec la nouvelle image.
4. L'application repasse `Synced` / `Healthy`.

## Reproduire la démo

Le dépôt ne contient que les manifestes de l'application : le cluster et Argo CD sont à installer à côté. Exemple avec [k3d](https://k3d.io/) (n'importe quel cluster Kubernetes convient).

### 1. Créer un cluster

```bash
# On publie le NodePort 30099 sur la machine hôte
k3d cluster create argo-demo -p "30099:30099@server:0"
```

### 2. Installer Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait -n argocd --for=condition=Ready pods --all --timeout=300s
```

Pour accéder à l'interface web :

```bash
# mot de passe initial de l'utilisateur admin
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

kubectl port-forward svc/argocd-server -n argocd 8080:443
# puis https://localhost:8080 (utilisateur : admin)
```

### 3. Créer le namespace de l'application

Les manifestes ciblent le namespace `dev` :

```bash
kubectl create namespace dev
```

(On peut aussi laisser Argo CD le créer avec l'option `CreateNamespace=true` ci-dessous.)

### 4. Créer l'application Argo CD

Ce manifeste n'est pas dans le dépôt : il est à appliquer sur le cluster, par exemple dans un fichier `application.yaml`.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/maxg56/mgedrot-argo-demo.git
    targetRevision: main
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

```bash
kubectl apply -f application.yaml
kubectl get applications -n argocd
kubectl get pods,svc -n dev
```

### 5. Tester l'application

```bash
curl http://localhost:30099/
```

### 6. Changer de version

```bash
sed -i 's/wil42\/playground:v2/wil42\/playground:v1/' deployment.yaml
git commit -am "update to v1"
git push
```

Argo CD vérifie le dépôt périodiquement (toutes les 3 minutes par défaut) ; on peut aussi forcer la synchronisation depuis l'interface. Une fois le nouveau pod démarré, `curl http://localhost:30099/` répond avec la nouvelle version.
