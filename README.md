# Jenkins sur Kubernetes — Guide d'installation et configuration

> **Environnement** : Mac M1 / Rancher Desktop  
> **Date** : Mai 2026  
> **Stack** : Jenkins LTS, Helm, k8s (Rancher Desktop), DockerHub

---

## Prérequis

| Outil | Version testée | Notes |
|---|---|---|
| Rancher Desktop | 1.x | Fournit Docker + k8s local |
| Helm | 3.x | `brew install helm` |
| kubectl | inclus dans Rancher Desktop | |
| Docker | inclus dans Rancher Desktop | |
| Compte DockerHub | — | Pour pusher les images |
| Compte GitHub | — | Pour le repo et le webhook |

---

## 1. Créer le namespace Jenkins

```bash
kubectl create namespace jenkins
```

---

## 2. Ajouter le repo Helm Jenkins

```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update
```

---

## 3. Construire l'image Jenkins custom

L'image officielle `jenkins/jenkins:lts` ne contient pas Docker CLI, Python, ni kubectl.
On crée une image custom qui inclut les trois.

**`Dockerfile.jenkins` :**

```dockerfile
# Image Jenkins avec Docker CLI + Python + kubectl — ARM64/M1 compatible
FROM jenkins/jenkins:lts

USER root

# Installer Docker CLI + Python + kubectl
RUN apt-get update && \
    apt-get install -y ca-certificates curl python3 python3-pip && \
    # Docker CLI
    install -m 0755 -d /etc/apt/keyrings && \
    curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc && \
    chmod a+r /etc/apt/keyrings/docker.asc && \
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] \
      https://download.docker.com/linux/debian \
      $(. /etc/os-release && echo "$VERSION_CODENAME") stable" \
      > /etc/apt/sources.list.d/docker.list && \
    apt-get update && \
    apt-get install -y docker-ce-cli && \
    # kubectl
    curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && \
    install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl && \
    rm kubectl && \
    # Nettoyage
    apt-get clean && rm -rf /var/lib/apt/lists/*

USER jenkins
```

**Builder et pusher sur DockerHub :**

```bash
docker build --platform linux/amd64 \
  -f Dockerfile.jenkins \
  -t <ton-dockerhub-user>/jenkins-docker:lts .

docker push <ton-dockerhub-user>/jenkins-docker:lts
```

> ⚠️ `--platform linux/amd64` est obligatoire sur M1 — Jenkins tourne en amd64 dans la VM Rancher Desktop.

---

## 4. Configurer le values.yaml

Récupérer d'abord le fichier de référence :

```bash
helm show values jenkins/jenkins > jenkins-values-default.yaml
```

**`jenkins-values.yaml` :**

```yaml
# ============================================================
# Jenkins values.yaml — Mac M1 / Rancher Desktop
# ============================================================

controller:
  # ── Authentification ─────────────────────────────────────
  admin:
    username: "admin"
    password: "admin"             # à changer en production

  # ── Image custom (Docker CLI + Python + kubectl) ─────────
  image:
    registry: "docker.io"
    repository: "<ton-dockerhub-user>/jenkins-docker"
    tag: "lts"
    pullPolicy: "IfNotPresent"

  # ── Accès navigateur ─────────────────────────────────────
  serviceType: NodePort
  nodePort: 30080                 # http://localhost:30080

  # ── Communication agents JNLP ────────────────────────────
  agentListenerServiceType: "ClusterIP"
  agentListenerPort: 50000

  # ── Ressources (adapté M1 local) ─────────────────────────
  resources:
    requests:
      cpu: "250m"
      memory: "512Mi"
    limits:
      cpu: "2000m"
      memory: "1536Mi"            # Jenkins + builds simultanés

  javaOpts: "-Xms256m -Xmx768m"
  numExecutors: 2

  # ── Sécurité — root obligatoire pour accéder au docker.sock
  usePodSecurityContext: true
  runAsUser: 0
  fsGroup: 0
  containerSecurityContext:
    runAsUser: 0
    runAsGroup: 0
    readOnlyRootFilesystem: false
    allowPrivilegeEscalation: true

  # ── Startup probe — Jenkins prend ~3 min à démarrer ──────
  probes:
    startupProbe:
      failureThreshold: 60        # 60 × 10s = 10 minutes
      periodSeconds: 10
      timeoutSeconds: 10
    livenessProbe:
      failureThreshold: 10
      periodSeconds: 10
      timeoutSeconds: 10
    readinessProbe:
      failureThreshold: 10
      periodSeconds: 10
      timeoutSeconds: 10

  # ── Plugins installés au démarrage ───────────────────────
  installPlugins:
    - kubernetes:4423.vb_59f230b_ce53
    - workflow-aggregator:608.v67378e9d3db_1
    - git:5.10.1
    - configuration-as-code:2077.v41f1011a_5110

persistence:
  enabled: true
  storageClass: "local-path"      # fourni par Rancher Desktop
  size: "2Gi"

  # ── Monter le socket Docker de la VM Rancher Desktop ─────
  # ⚠️ Le socket est dans la VM Linux, pas sur le Mac host
  # Chemin vérifié avec : rdctl shell → ls -la /var/run/docker.sock
  volumes:
    - name: docker-sock
      hostPath:
        path: /var/run/docker.sock
        type: Socket

  mounts:
    - name: docker-sock
      mountPath: /var/run/docker.sock
```

---

## 5. Déployer Jenkins

```bash
helm install jenkins jenkins/jenkins \
  --namespace jenkins \
  --create-namespace \
  -f jenkins-values.yaml
```

**Vérifier que tout tourne :**

```bash
kubectl get pods -n jenkins      # jenkins-0 doit être 2/2 Running
kubectl get svc -n jenkins       # NodePort 30080
kubectl get pvc -n jenkins       # Bound
```

> ⏳ Jenkins prend **2 à 3 minutes** pour charger tous les plugins au premier démarrage.

**Vérifier l'accès au socket Docker :**

```bash
kubectl exec -n jenkins jenkins-0 -c jenkins -- ls -la /var/run/docker.sock
kubectl exec -n jenkins jenkins-0 -c jenkins -- docker version
```

**Accéder à Jenkins :**

```
http://localhost:30080
Login : admin / admin
```

---

## 6. Mettre à jour Jenkins (après modification du values.yaml)

```bash
helm upgrade jenkins jenkins/jenkins \
  --namespace jenkins \
  -f jenkins-values.yaml

kubectl rollout restart statefulset/jenkins -n jenkins
kubectl get pods -n jenkins -w
```

---

## 7. Configurer les credentials dans Jenkins

### Credential GitHub

```
Manage Jenkins → Credentials → System → Global → Add Credentials

Kind        : Username with password
Username    : <ton-username-github>
Password    : <ton-personal-access-token-github>
ID          : github-credentials
Description : GitHub token pour mon-app
```

**Générer le token GitHub :**
```
github.com → Settings → Developer settings
  → Personal access tokens → Tokens (classic)
    → Generate new token → scope : ✅ repo
```

### Credential DockerHub

```
Manage Jenkins → Credentials → System → Global → Add Credentials

Kind        : Username with password
Username    : <ton-username-dockerhub>
Password    : <ton-access-token-dockerhub>
ID          : dockerhub-secret
Description : DockerHub token
```

**Générer le token DockerHub :**
```
hub.docker.com → Account Settings → Personal access tokens
  → Generate new token → Read & Write
```

---

## 8. Créer le job Pipeline

```
localhost:30080 → + New Item

Name       : mon-app
Type       : Pipeline
→ OK

Section Pipeline :
  Definition  : Pipeline script from SCM
  SCM         : Git
  Repository  : https://github.com/<user>/<repo>.git
  Credentials : github-credentials
  Branch      : */main
  Script Path : Jenkinsfile

☐ Lightweight checkout   ← décocher obligatoirement
→ Save
```

> ⚠️ **Décocher "Lightweight checkout"** est obligatoire — sinon Jenkins échoue avec `fatal: not in a git directory`.

---

## 9. Problèmes connus et solutions

| Problème | Cause | Solution |
|---|---|---|
| `Startup probe failed` | Jenkins prend >2 min à démarrer | `failureThreshold: 60` dans le values.yaml |
| `docker: not found` | Image officielle sans Docker CLI | Utiliser l'image custom |
| `pip: not found` | Image officielle sans Python | Utiliser l'image custom |
| `kubectl: not found` | Image officielle sans kubectl | Utiliser l'image custom |
| `docker.sock: No such file` | Mauvaise syntaxe du mount | Utiliser `persistence.volumes` + `persistence.mounts` |
| `fatal: not in a git directory` | Lightweight checkout activé | Décocher dans la config du job |
| `controller.adminUser no longer exists` | Ancienne syntaxe du chart | Utiliser `controller.admin.username` |
| Mémoire insuffisante | Limite trop basse | `memory limits: 1536Mi` + `javaOpts: -Xmx768m` |

---

## 10. Commandes utiles

```bash
# État du pod Jenkins
kubectl get pods -n jenkins -w

# Logs Jenkins en temps réel
kubectl logs -n jenkins jenkins-0 -c jenkins -f

# Redémarrer Jenkins
kubectl rollout restart statefulset/jenkins -n jenkins

# Vérifier les volumes montés
kubectl get pod jenkins-0 -n jenkins -o yaml | grep -A 20 "volumes:"

# Se connecter à la VM Rancher Desktop
rdctl shell

# Vérifier le socket dans la VM
ls -la /var/run/docker.sock
```
