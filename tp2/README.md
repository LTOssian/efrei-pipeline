# TP2 — Pipeline CI/CD avec Jenkins

## Étapes réalisées

### 1. Installation de VMWare
- Télécharger et installer VMWare Workstation/Fusion
- Créer une machine virtuelle pour l'environnement DevOps (Ubuntu Server recommandé)

### 2. Configuration de la VM
- Vérifier l'accès à internet : `ping google.com`
- Vérifier l'adresse IP : `ip addr show` ou `ifconfig`
- Réaliser un snapshot de la machine vierge

### 3. Installation de Docker Engine
```bash
sudo apt update
sudo apt install -y docker.io
sudo systemctl enable docker --now
sudo usermod -aG docker $USER
```

### 4. Création du réseau Docker "devops"
```bash
docker network create devops
```

### 5. Déploiement de Jenkins
```bash
docker run -d \
  --name jenkins \
  --network devops \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  jenkins/jenkins:lts
```

### 6. Finalisation de l'installation Jenkins
- Récupérer le mot de passe initial :
  ```bash
  docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
  ```
- Accéder à `http://<IP_VM>:8080`
- Suivre l'assistant d'installation (plugins suggérés)

### 7. Pipeline "Hello World"
- Créer un nouveau job Pipeline
- Ajouter un script simple :
  ```groovy
  pipeline {
      agent any
      stages {
          stage('Hello') {
              steps {
                  echo 'Hello World!'
              }
          }
      }
  }
  ```
- Lancer la pipeline

### 8. Pipeline CI/CD
- Ajouter les étapes : Checkout → Build → Test → Deploy
- Le `Jenkinsfile` de ce repo contient la pipeline complète

### 9. Fork du repository
- Forker ce repository sur GitHub
- Ajouter le `Jenkinsfile` à la racine (ou dans `tp2/`)

### 10. Agent Jenkins
- Configurer un nouvel agent dans Jenkins (Manage Jenkins → Nodes)
- Lancer une pipeline avec l'agent dédié

### 11. Tunnel ngrok
```bash
ngrok http 8080
```
- Récupérer l'URL publique générée
- Jenkins est accessible temporairement via cette URL

### 12. Webhook GitHub → Jenkins
- Dans GitHub : Settings → Webhooks → Add webhook
  - Payload URL : `https://<ngrok-url>/github-webhook/`
  - Content type : `application/json`
  - Events : Just the push event
- Dans Jenkins : configurer le job pour utiliser "GitHub hook trigger for GITScm polling"
- La pipeline se déclenche automatiquement à chaque push
