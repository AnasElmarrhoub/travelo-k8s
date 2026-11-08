# Guide Pratique & Cahier de Recette — Devoir 2 (Kubernetes)

**Module :** Conteneurisation des Applications — ASEDS INE2  
**Établissement :** Institut National des Postes et Télécommunications (INPT)  
**Enseignant :** Prof. Driss ALLAKI  
**Année universitaire :** 2026/2027  
**Projet :** Déploiement Automatisé et Mise à l’Échelle de l'Application Travelo sur Kubernetes  

---

> 🎯 **Objectif de ce document :**  
> Ce document est conçu pour vous guider pas à pas dans la réalisation du **Devoir 2**. Pour chaque question, vous trouverez :
> 1. Les **commandes de préparation et de vérification des versions** (ex: `kubectl version --client`, `minikube version`).
> 2. Les **commandes d'exécution et de déploiement**.
> 3. La commande exacte à lancer pour produire **la capture d'écran** demandée dans votre rapport.
> 4. Une **boîte explicative** prête à être insérée dans votre rapport LaTeX/Word.

---

## 📌 Sommaire des Étapes et Captures d'Écran

- [Étape 0 : Vérification Préalable des Outils](#étape-0--vérification-préalable-des-outils)
- [Question 1 : Création et Démarrage du Cluster Kubernetes (Minikube)](#question-1--création-du-cluster-kubernetes-local-minikube)
- [Question 2 : Création et Configuration du Namespace `travelo`](#question-2--création-et-configuration-du-namespace-travelo)
- [Question 3 : Versioning Git des Manifestes Kubernetes](#question-3--versioning-git-des-manifestes-kubernetes)
- [Question 3.a : Déploiement des Composants Applicatifs de Base](#question-3a--déploiement-des-composants-applicatifs-de-base)
  - [3.a.1 Secret MySQL](#3a1-secret-mysql)
  - [3.a.2 ConfigMap Travelo](#3a2-configmap-travelo)
  - [3.a.3 StatefulSet & Service Headless MySQL](#3a3-statefulset--service-headless-mysql)
  - [3.a.4 Deployment & Service Backend Spring Boot (2 Réplicas)](#3a4-deployment--service-backend-spring-boot)
  - [3.a.5 Deployment & Service Frontend React (2 Réplicas)](#3a5-deployment--service-frontend-react)
- [Question 3.b : Quotas de Ressources (ResourceQuota & LimitRange)](#question-3b--quotas-de-ressources-cpu--mémoire)
- [Question 3.c : Contrôle d'Accès Sécurisé (RBAC)](#question-3c--contrôle-daccès-sécurisé-rbac)
- [Question 3.d : Budget de Perturbation (PodDisruptionBudget)](#question-3d--budget-de-perturbation-poddisruptionbudget)
- [Question 3.e : Sondes de Santé (Startup, Liveness & Readiness Probes)](#question-3e--sondes-de-santé-startup-liveness--readiness)
- [Question 3.f : Ingress Controller & Routage par Nom de Domaine](#question-3f--ingress-controller--routage-par-nom-de-domaine)
- [Question 4 : Validation Finale, Métriques et Tests Navigateur](#question-4--validation-finale-métriques-et-tests-navigateur)
- [Partie Bonus (Operator, Skaffold, Helm)](#partie-bonus)
- [Tableau Récapitulatif de Toutes les Captures d'Écran](#tableau-récapitulatif-de-toutes-les-captures-décran)

---

## Étape 0 : Vérification Préalable des Outils

Avant de démarrer le cluster, il est essentiel de prouver dans le rapport que votre environnement hôte dispose des binaires appropriés.

### 💻 Commandes à exécuter
```powershell
docker --version
minikube version
kubectl version --client
```

### 📸 Capture d'Écran n°1
* **Commande :**
  ```powershell
  docker --version; minikube version; kubectl version --client
  ```
* **Ce que le screen doit montrer :** Les versions installées de Docker (Engine 28+), Minikube (v1.34+) et du client kubectl (v1.30+).
* **Texte pour le rapport :**
  > *« L'environnement local s'appuie sur le CLI kubectl pour interagir avec le cluster Kubernetes, orchestré localement par Minikube avec le runtime de conteneurs Docker. »*

---

## Question 1 : Création du Cluster Kubernetes Local (Minikube)

### 🎯 Objectif
Instancier un cluster Kubernetes avec un dimensionnement adapté (4 vCPU, 4 Go RAM) et activer les extensions nécessaires (*addons*) : le contrôleur Ingress Nginx et le serveur de métriques.

### 💻 Commandes d'exécution
```powershell
# 1. Démarrer le cluster Minikube avec le pilote Docker
minikube start --driver=docker --cpus=4 --memory=4096 --kubernetes-version=v1.30.0

# 2. Activer l'extension Ingress Nginx (requis pour la Q3.f)
minikube addons enable ingress

# 3. Activer l'extension metrics-server (requis pour le monitoring CPU/RAM)
minikube addons enable metrics-server
```

### 💻 Commandes de vérification pour les captures
```powershell
# État général de Minikube
minikube status

# Vérification du nœud du cluster
kubectl get nodes -o wide

# Vérification des addons actifs
minikube addons list | Select-String "ingress|metrics-server"
```

### 📸 Capture d'Écran n°2
* **Commande :** `minikube status; kubectl get nodes -o wide`
* **Ce que le screen doit montrer :** `host: Running`, `kubelet: Running`, `apiserver: Running` et le nœud `minikube` avec le statut `Ready`.

### 📸 Capture d'Écran n°3
* **Commande :** `minikube addons list | Select-String "ingress|metrics-server"`
* **Ce que le screen doit montrer :** Les addons `ingress` et `metrics-server` marqués comme `enabled`.

### 📝 Explication technique pour le rapport
> *« L'argument `--driver=docker` permet à Minikube d'isoler le control-plane et le worker K8s dans un conteneur Docker dédié, évitant l'empreinte lourde d'une machine virtuelle Hyper-V/VirtualBox. L'activation native de l'addon `ingress` déploie le contrôleur `ingress-nginx` sous le namespace `ingress-nginx`, assurant la terminaison HTTP et le routage applicatif de niveau 7. »*

---

## Question 2 : Création et Configuration du Namespace `travelo`

### 🎯 Objectif
Créer une frontière logique (*Namespace*) pour isoler toutes les ressources de notre projet Travelo et basculer le contexte actif de `kubectl` vers ce namespace.

### 📄 Manifeste : `k8s/00-namespace.yaml`
*(Fichier déjà généré dans votre dossier `k8s/`)*

### 💻 Commandes d'exécution
```powershell
# 1. Appliquer le manifeste du namespace
kubectl apply -f k8s/00-namespace.yaml

# 2. Configurer le namespace par défaut dans le contexte actif
kubectl config set-context --current --namespace=travelo
```

### 💻 Commandes de vérification
```powershell
# Vérifier la liste des namespaces
kubectl get namespaces

# Confirmer que le contexte courant pointe bien sur travelo
kubectl config view --minify | Select-String "namespace:"
```

### 📸 Capture d'Écran n°4
* **Commande :** `kubectl get namespaces; kubectl config view --minify | Select-String "namespace:"`
* **Ce que le screen doit montrer :** Le namespace `travelo` avec le statut `Active` et la ligne `namespace: travelo` dans la configuration active.

---

## Question 3 : Versioning Git des Manifestes Kubernetes

### 🎯 Objectif
Créer un dépôt Git dédié (sur GitHub ou GitLab), versionner l'ensemble des manifestes YAML de notre dossier `k8s/` et préparer le déploiement continu.

### 💻 Commandes d'exécution
```bash
git init
git add k8s/
git commit -m "feat: ajout des manifests Kubernetes de l'application Travelo"
git branch -M main
git remote add origin https://github.com/VOTRE_USER/travelo-k8s.git
git push -u origin main
```

### 💻 Commandes de vérification
```bash
git status
git log -1 --stat
git remote -v
```

### 📸 Capture d'Écran n°5
* **Commande :** `git status; git remote -v`
* **Ce que le screen doit montrer :** L'arbre de travail propre (*working tree clean*) et l'URL distante GitHub/GitLab pointant vers votre dépôt.

---

## Question 3.a : Déploiement des Composants Applicatifs de Base

### 🎯 Objectif
Déployer dans l'ordre strict les objets Kubernetes constituant l'architecture 3-tiers :
1. Les **Secrets** (mots de passe chiffrés)
2. Les **ConfigMaps** (URLs JDBC et constantes)
3. Le **StatefulSet** MySQL (avec volume persistant)
4. Le **Deployment** Backend Spring Boot (2 réplicas)
5. Le **Deployment** Frontend React (2 réplicas)

---

### 3.a.1 Secret MySQL

#### 💻 Commande d'application
```powershell
kubectl apply -f k8s/03-secrets.yaml
```

#### 💻 Commande de vérification
```powershell
kubectl get secret travelo-db-secret -n travelo
kubectl describe secret travelo-db-secret -n travelo
```

#### 📸 Capture d'Écran n°6
* **Commande :** `kubectl get secret travelo-db-secret -n travelo`
* **Ce que le screen doit montrer :** Le Secret de type `Opaque` contenant 2 entrées (`MYSQL_ROOT_PASSWORD`, `MYSQL_PASSWORD`).

---

### 3.a.2 ConfigMap Travelo

#### 💻 Commande d'application
```powershell
kubectl apply -f k8s/04-configmap.yaml
```

#### 💻 Commande de vérification
```powershell
kubectl get configmap travelo-config -n travelo
kubectl describe configmap travelo-config -n travelo
```

#### 📸 Capture d'Écran n°7
* **Commande :** `kubectl describe configmap travelo-config -n travelo`
* **Ce que le screen doit montrer :** Les paires clé-valeur : `MYSQL_DATABASE=travelo_db`, `MYSQL_HOST=mysql-svc`, et l'URL Spring JDBC.

---

### 3.a.3 StatefulSet & Service Headless MySQL

#### 💻 Commande d'application
```powershell
kubectl apply -f k8s/05-mysql-statefulset.yaml
```

#### 💻 Commande de vérification
```powershell
# Vérifier l'état du StatefulSet, du Pod et du Volume PVC
kubectl get statefulset,pods,svc,pvc -l app=mysql -n travelo
```

#### 📸 Capture d'Écran n°8
* **Commande :** `kubectl get statefulset,pods,svc,pvc -l app=mysql -n travelo`
* **Ce que le screen doit montrer :** 
  - `statefulset.apps/mysql 1/1`
  - Le pod nommé `mysql-0` avec statut `Running`
  - Le PVC `mysql-data-mysql-0` avec statut `Bound` (relié au stockage persistant).

#### 📝 Explication technique pour le rapport
> *« Contrairement à un Deployment standard stateless, un **StatefulSet** est impératif pour MySQL car il garantit : (1) un nom d'hôte stable et ordonné (`mysql-0`), (2) un Service Headless (`clusterIP: None`) permettant d'adresser directement le pod, et (3) un modèle de persistance `volumeClaimTemplates` liant de manière exclusive un PersistentVolumeClaim (PVC) au pod pour empêcher toute corruption des verrous InnoDB d'accès disque. »*

---

### 3.a.4 Deployment & Service Backend Spring Boot

#### 💻 Commande d'application
```powershell
kubectl apply -f k8s/06-backend-deployment.yaml
```

#### 💻 Commande de vérification
```powershell
kubectl get deployment backend-deployment -n travelo
kubectl get pods -l app=backend -n travelo -o wide
kubectl get svc backend-svc -n travelo
```

#### 📸 Capture d'Écran n°9
* **Commande :** `kubectl get deployment,pods,svc -l app=backend -n travelo -o wide`
* **Ce que le screen doit montrer :** 
  - `backend-deployment` avec **`READY 2/2`** (les 2 réplicas exigés).
  - Deux pods distincts exécutant l'image `anaselmarrhoub/travelo-backend:1.0`.
  - Le service `backend-svc` sur le port `8080/TCP`.

---

### 3.a.5 Deployment & Service Frontend React

#### 💻 Commande d'application
```powershell
kubectl apply -f k8s/07-frontend-deployment.yaml
```

#### 💻 Commande de vérification
```powershell
kubectl get deployment frontend-deployment -n travelo
kubectl get pods -l app=frontend -n travelo -o wide
kubectl get svc frontend-svc -n travelo
```

#### 📸 Capture d'Écran n°10
* **Commande :** `kubectl get deployment,pods,svc -l app=frontend -n travelo -o wide`
* **Ce que le screen doit montrer :** 
  - `frontend-deployment` avec **`READY 2/2`** (les 2 réplicas exigés).
  - Deux pods distincts exécutant l'image `anaselmarrhoub/travelo-frontend:1.0`.
  - Le service `frontend-svc` sur le port `80/TCP`.

---

## Question 3.b : Quotas de Ressources (CPU & Mémoire)

### 🎯 Objectif
Définir un **ResourceQuota** pour plafonner globalement la consommation du namespace `travelo`, et un **LimitRange** pour borner automatiquement chaque Pod individuel.

### 💻 Commande d'application
```powershell
kubectl apply -f k8s/02-resource-quota.yaml
```

### 💻 Commandes de vérification
```powershell
# 1. Vérifier la consommation par rapport au quota global
kubectl describe resourcequota travelo-quota -n travelo

# 2. Vérifier les contraintes min/max et par défaut du LimitRange
kubectl describe limitrange travelo-limit-range -n travelo
```

### 📸 Capture d'Écran n°11
* **Commande :** `kubectl describe resourcequota travelo-quota -n travelo`
* **Ce que le screen doit montrer :** Le tableau comparatif entre **Used** (ressources consommées actuellement par nos 5 pods) et **Hard** (plafond max : 2 CPU, 2Gi RAM).

### 📸 Capture d'Écran n°12
* **Commande :** `kubectl describe limitrange travelo-limit-range -n travelo`
* **Ce que le screen doit montrer :** Les valeurs `default`, `defaultRequest`, `min` et `max` pour les conteneurs.

### 📝 Explication technique pour le rapport
> *« Le **ResourceQuota** évite le déni de service à l'échelle du cluster en limitant le total agrégé de mémoire et de CPU dans le namespace `travelo`. Le **LimitRange** joue le rôle d'admission controller : si un conteneur est soumis sans spécifier de blocs `resources.requests` ou `resources.limits`, Kubernetes lui applique automatiquement les valeurs `defaultRequest` (100m CPU / 128Mi RAM) et `default` (500m CPU / 512Mi RAM). »*

---

## Question 3.c : Contrôle d'Accès Sécurisé (RBAC)

### 🎯 Objectif
Créer un compte de service (`ServiceAccount`) associé à un `Role` restreint aux ressources de `travelo` via un `RoleBinding`, et prouver l'efficacité des restrictions via `kubectl auth can-i`.

### 💻 Commande d'application
```powershell
kubectl apply -f k8s/01-rbac.yaml
```

### 💻 Commandes de vérification des droits
```powershell
# 1. Vérifier la création des objets RBAC
kubectl get sa,role,rolebinding -n travelo

# 2. Test 1 : Peut-il lister les pods dans travelo ? (RÉPONSE ATTENDUE : yes)
kubectl auth can-i list pods --as=system:serviceaccount:travelo:travelo-admin-sa -n travelo

# 3. Test 2 : Peut-il créer des déploiements dans travelo ? (RÉPONSE ATTENDUE : yes)
kubectl auth can-i create deployments --as=system:serviceaccount:travelo:travelo-admin-sa -n travelo

# 4. Test 3 : Peut-il supprimer un namespace dans le cluster ? (RÉPONSE ATTENDUE : no)
kubectl auth can-i delete namespaces --as=system:serviceaccount:travelo:travelo-admin-sa

# 5. Test 4 : Peut-il voir les nœuds du cluster ? (RÉPONSE ATTENDUE : no)
kubectl auth can-i get nodes --as=system:serviceaccount:travelo:travelo-admin-sa
```

### 📸 Capture d'Écran n°13
* **Commande :**
  ```powershell
  kubectl auth can-i list pods --as=system:serviceaccount:travelo:travelo-admin-sa -n travelo;
  kubectl auth can-i create deployments --as=system:serviceaccount:travelo:travelo-admin-sa -n travelo;
  kubectl auth can-i delete namespaces --as=system:serviceaccount:travelo:travelo-admin-sa;
  kubectl auth can-i get nodes --as=system:serviceaccount:travelo:travelo-admin-sa
  ```
* **Ce que le screen doit montrer :** 
  - `yes` pour les deux premières requêtes (autorisées au sein de `travelo`).
  - `no` pour les deux suivantes (interdites en dehors du namespace).

---

## Question 3.d : Budget de Perturbation (PodDisruptionBudget)

### 🎯 Objectif
Garantir la haute disponibilité lors de maintenances planifiées (mise à niveau d'un nœud, drainage du cluster). Le PDB interdit l'arrêt simultané de tous les pods d'un même service.

### 💻 Commande d'application
```powershell
kubectl apply -f k8s/08-pdb.yaml
```

### 💻 Commande de vérification
```powershell
kubectl get pdb -n travelo
kubectl describe pdb backend-pdb -n travelo
```

### 📸 Capture d'Écran n°14
* **Commande :** `kubectl get pdb -n travelo`
* **Ce que le screen doit montrer :** Les deux PDB (`backend-pdb` et `frontend-pdb`) affichant `MIN AVAILABLE: 1` et `ALLOWED DISRUPTIONS: 1`.

---

## Question 3.e : Sondes de Santé (Startup, Liveness & Readiness)

### 🎯 Objectif
Configurer et valider le cycle de vie auto-cicatrisant des conteneurs :
- **Startup Probe :** Attend l'initialisation complète de la JVM et d'Hibernate sans déclencher de faux redémarrages.
- **Readiness Probe :** Vérifie que l'endpoint `/api/flights/search` répond avant de router les requêtes utilisateurs.
- **Liveness Probe :** Détecte les blocages pour redémarrer le pod si nécessaire.

### 💻 Commandes de vérification
```powershell
# 1. Vérifier la configuration des sondes sur un pod backend
kubectl describe pod -l app=backend -n travelo | Select-String "Startup:|Readiness:|Liveness:|State:" -Context 0,2

# 2. Inspecter les événements récents (Pulling, Started, Passed probes)
kubectl get events -n travelo --sort-by='.metadata.creationTimestamp' | Select-Object -Last 15
```

### 📸 Capture d'Écran n°15
* **Commande :** `kubectl describe pod -l app=backend -n travelo | Select-String "Startup:|Readiness:|Liveness:" -Context 0,2`
* **Ce que le screen doit montrer :** Les blocs `Startup:`, `Readiness:` et `Liveness:` configurés sur le port 8080.

### 📸 Capture d'Écran n°16
* **Commande :** `kubectl get events -n travelo --sort-by='.metadata.creationTimestamp' | Select-Object -Last 10`
* **Ce que le screen doit montrer :** Les événements Kubernetes confirmant : `Scheduled`, `Pulled`, `Created`, `Started`.

---

## Question 3.f : Ingress Controller & Routage par Nom de Domaine

### 🎯 Objectif
Exposer l'application Travelo via un nom de domaine unifié `travelo.local` en routant :
- `/api/*` vers le Service `backend-svc:8080`
- `/*` vers le Service `frontend-svc:80`

### 💻 Commandes d'exécution et configuration DNS

```powershell
# 1. Récupérer l'adresse IP du cluster Minikube
minikube ip
```
*(Exemple retourné : `192.168.49.2`)*

#### Configuration du fichier `hosts` Windows :
Ouvrez PowerShell en tant qu'administrateur et exécutez :
```powershell
# Remplacez 192.168.49.2 par l'IP réelle donnée par 'minikube ip'
Add-Content -Path "C:\Windows\System32\drivers\etc\hosts" -Value "`n192.168.49.2 travelo.local"
```

#### Déploiement de l'Ingress :
```powershell
kubectl apply -f k8s/09-ingress.yaml
```

### 💻 Commandes de vérification
```powershell
# Vérifier l'état de l'Ingress (doit afficher l'IP dans ADDRESS)
kubectl get ingress travelo-ingress -n travelo

# Tester la résolution DNS locale
ping -n 1 travelo.local

# Tester la réponse HTTP du frontend via Ingress
curl -I http://travelo.local

# Tester la réponse HTTP du backend via Ingress
curl -s "http://travelo.local/api/flights/search?from=Casablanca&to=Paris&departureDate=2026-11-15"
```

### 📸 Capture d'Écran n°17
* **Commande :** `kubectl get ingress -n travelo; minikube ip`
* **Ce que le screen doit montrer :** L'Ingress `travelo-ingress` avec l'hôte `travelo.local` et l'adresse IP associée.

### 📸 Capture d'Écran n°18
* **Commande :** `Get-Content "C:\Windows\System32\drivers\etc\hosts" -Tail 3`
* **Ce que le screen doit montrer :** La ligne associant l'IP de Minikube à `travelo.local`.

---

## Question 4 : Validation Finale, Métriques et Tests Navigateur

### 🎯 Objectif
Valider l'ensemble du déploiement avec des commandes d'audit globales et des captures d'écran du navigateur en situation réelle.

### 💻 Commandes d'audit complètes
```powershell
# 1. Vue d'ensemble de tous les objets du namespace travelo
kubectl get all,pvc,ingress,quota,pdb,secret,configmap -n travelo

# 2. Mesure de la consommation réelle en CPU et RAM
kubectl top pods -n travelo
```

### 📸 Capture d'Écran n°19 : Vue d'ensemble du cluster
* **Commande :** `kubectl get all,pvc,ingress -n travelo`
* **Ce que le screen doit montrer :** Tous les Pods en statut `Running`, les Services, les Deployments avec `2/2`, le StatefulSet `1/1` et l'Ingress.

### 📸 Capture d'Écran n°20 : Interface Web dans le navigateur
* **Action :** Ouvrir votre navigateur sur **`http://travelo.local`**.
* **Ce que le screen doit montrer :** La page d'accueil de **Travelo** chargée correctement depuis Kubernetes via le nom de domaine.

### 📸 Capture d'Écran n°21 : Recherche de vols fonctionnelle
* **Action :** Remplir le formulaire (ex: *Casablanca* $\rightarrow$ *Paris*, date *2026-11-15*) et cliquer sur **Search**.
* **Ce que le screen doit montrer :** Les vols trouvés retournés par le Backend Spring Boot via l'Ingress `/api`.

### 📸 Capture d'Écran n°22 : Sélection de siège et réservation
* **Action :** Cliquer sur un vol, choisir un siège disponible et confirmer la réservation.
* **Ce que le screen doit montrer :** Le ticket de confirmation généré avec succès.

---

## Partie Bonus

### Bonus A : Gestion de la Base de Données via un Operator
Au lieu d'un StatefulSet natif géré manuellement, un **Kubernetes Operator** (ex: *Oracle MySQL Operator* ou *CloudNativePG*) apporte :
- Sauvegardes automatisées (*Scheduled Backups* vers S3/MinIO).
- Basculement automatique en cas de panne (*Failover Haute Disponibilité*).
- Gestion de répliques en lecture (*Read Replicas*).

### Bonus B : Développement Continu avec Skaffold
**Skaffold** élimine le cycle manuel `docker build -> push -> kubectl apply`.
- Installez Skaffold (`choco install skaffold` ou via binaire).
- Créez un fichier `skaffold.yaml` à la racine :
  ```yaml
  apiVersion: skaffold/v4beta11
  kind: Config
  metadata:
    name: travelo
  build:
    artifacts:
      - image: travelo-backend
        context: travelo-app/travelo_backend
      - image: travelo-frontend
        context: travelo-app/travelo_frontend
  manifests:
    rawYaml:
      - k8s/*.yaml
  ```
- Lancez `skaffold dev` : toute modification de code dans React ou Spring Boot est automatiquement recompilée et déployée dans Minikube !

### Bonus C : Packaging Helm Chart
Helm permet de déployer toute l'application Travelo en une seule commande :
```bash
helm create travelo-chart
helm install travelo-app ./travelo-chart -n helm --create-namespace
helm list -n helm
```

---

## Tableau Récapitulatif de Toutes les Captures d'Écran

Voici le tableau de bord à cocher au fur et à mesure pour votre rapport :

| N° | Description de la capture | Commande clé | Section |
|:--:|---|---|:--:|
| **1** | Versions des outils CLI | `kubectl version --client && minikube version` | Étape 0 |
| **2** | Minikube démarré et nœud Ready | `minikube status && kubectl get nodes` | Q1 |
| **3** | Addons ingress & metrics-server activés | `minikube addons list` | Q1 |
| **4** | Namespace `travelo` actif | `kubectl get ns travelo` | Q2 |
| **5** | Dépôt Git avec manifests | `git status && git remote -v` | Q3 |
| **6** | Secret MySQL | `kubectl get secret travelo-db-secret -n travelo` | Q3.a |
| **7** | ConfigMap Travelo | `kubectl describe configmap travelo-config -n travelo` | Q3.a |
| **8** | StatefulSet MySQL & PVC Bound | `kubectl get statefulset,pvc -n travelo` | Q3.a |
| **9** | Backend Deployment (2/2 réplicas) | `kubectl get deploy,pods -l app=backend -n travelo` | Q3.a |
| **10** | Frontend Deployment (2/2 réplicas) | `kubectl get deploy,pods -l app=frontend -n travelo` | Q3.a |
| **11** | ResourceQuota (Used vs Hard) | `kubectl describe resourcequota travelo-quota -n travelo` | Q3.b |
| **12** | LimitRange (Min / Max / Default) | `kubectl describe limitrange travelo-limit-range -n travelo` | Q3.b |
| **13** | Validation des permissions RBAC | `kubectl auth can-i ...` (4 tests) | Q3.c |
| **14** | PodDisruptionBudget actif | `kubectl get pdb -n travelo` | Q3.d |
| **15** | Configuration des sondes (Probes) | `kubectl describe pod -l app=backend -n travelo` | Q3.e |
| **16** | Événements récents des pods | `kubectl get events -n travelo` | Q3.e |
| **17** | Ingress configuré avec host | `kubectl get ingress -n travelo` | Q3.f |
| **18** | Entrée DNS dans le fichier `hosts` | `Get-Content C:\Windows\System32\drivers\etc\hosts` | Q3.f |
| **19** | Vue globale de toutes les ressources | `kubectl get all,pvc,ingress -n travelo` | Q4 |
| **20** | Navigateur : Page d'accueil Travelo | `http://travelo.local` | Q4 |
| **21** | Navigateur : Résultat recherche de vols | Formulaire de vol validé | Q4 |
| **22** | Navigateur : Réservation & Siège | Ticket généré | Q4 |
