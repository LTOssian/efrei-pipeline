# TP3 — Pipeline CI/CD avec AWS EC2 + Docker + Jenkins

**Équipe** : ALIMOU DIALLO
**DockerHub** : [ossian7](https://hub.docker.com/u/ossian7)
**EC2** : t3.small — Ubuntu 22.04 — IP `13.36.210.49`

---

## Étapes

### 1. Créer un compte AWS

- Aller sur [aws.amazon.com](https://aws.amazon.com) et créer un compte (navigation privée si besoin)
- Activer le Free Tier

### 2. Créer un user IAM + Access Key

```bash
aws iam create-user --user-name devops-cli
aws iam attach-user-policy --user-name devops-cli --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess
aws iam create-access-key --user-name devops-cli
```

📸 **Screenshot** : sortie de `create-access-key` avec AccessKeyId + SecretAccessKey

### 3. Configurer AWS CLI

```bash
aws configure
# AWS Access Key ID : <coller>
# AWS Secret Access Key : <coller>
# Default region : eu-west-3 (Paris)
# Default output : json
```

### 4. Créer la paire de clés SSH

```bash
aws ec2 create-key-pair --key-name devops-key --query 'KeyMaterial' --output text > devops-key.pem
chmod 400 devops-key.pem
```

### 5. Créer le Security Group + ouvrir les ports

```bash
aws ec2 create-security-group --group-name devops-sg --description "DevOps TP3 SG"

aws ec2 authorize-security-group-ingress --group-name devops-sg --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-name devops-sg --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-name devops-sg --protocol tcp --port 3000 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-name devops-sg --protocol tcp --port 8080 --cidr 0.0.0.0/0
```

📸 **Screenshot** : inbound rules du Security Group

### 6. Lancer l'instance EC2

```bash
# Trouver l'AMI Ubuntu 22.04
aws ec2 describe-images --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
  --query 'Images[*].[ImageId,CreationDate]' --output text | sort -k2 -r | head -1

# Lancer l'instance (t3.small, 20 GB)
aws ec2 run-instances \
  --image-id <AMI_ID> \
  --instance-type t3.small \
  --key-name devops-key \
  --security-groups devops-sg \
  --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":20}}]' \
  --count 1

# Récupérer l'IP publique
aws ec2 describe-instances --query 'Reservations[*].Instances[*].PublicIpAddress' --output text
```

📸 **Screenshot** : instance EC2 running dans la console AWS

### 7. Installer Docker sur l'EC2

```bash
ssh -i devops-key.pem ubuntu@<IP_EC2>
sudo apt update && sudo apt install -y docker.io
sudo usermod -aG docker ubuntu
exit
# Reconnecter pour que le groupe docker prenne effet
ssh -i devops-key.pem ubuntu@<IP_EC2>
```

📸 **Screenshot** : `docker --version` dans le terminal SSH

### 8. Lancer Jenkins sur l'EC2 (avec socket Docker)

```bash
ssh -i devops-key.pem ubuntu@<IP_EC2>

# Donner accès au socket Docker
sudo chmod 666 /var/run/docker.sock

# Lancer Jenkins
docker run -d \
  --name jenkins \
  -p 8080:8080 \
  -p 50000:50000 \
  -v jenkins_home:/var/jenkins_home \
  -v /var/run/docker.sock:/var/run/docker.sock \
  jenkins/jenkins:lts

# Récupérer le mot de passe admin
docker exec jenkins cat /var/jenkins_home/secrets/initialAdminPassword
```

- Accéder à `http://<IP_EC2>:8080`
- Installer les plugins suggérés, créer un compte admin

### 9. Construire et pousser les images Docker

```bash
# Sur l'EC2, cloner le repo
git clone https://github.com/LTOssian/efrei-pipeline.git
cd efrei-pipeline

# Login DockerHub
docker login -u ossian7

# Build + Push
docker build -t ossian7/landing tp3/landing
docker push ossian7/landing
docker build -t ossian7/api tp3/api
docker push ossian7/api
```

📸 **Screenshot** : `ossian7/landing` et `ossian7/api` sur DockerHub

### 10. Configurer la pipeline Jenkins

- Dans Jenkins (`http://<IP_EC2>:8080`), **New Item** → `tp3-pipeline` → **Pipeline**
- **Pipeline script from SCM** :
  - SCM : Git
  - Repository URL : `https://github.com/LTOssian/efrei-pipeline.git`
  - Script Path : `tp3/Jenkinsfile`
- **Save** → **Build Now**

📸 **Screenshot** : les 6 stages verts dans Jenkins

### 11. Vérifier le déploiement

```bash
# Landing page
curl http://<IP_EC2>

# API
curl http://<IP_EC2>:3000
```

📸 **Screenshot** : landing page dans le navigateur
📸 **Screenshot** : API → JSON `{"message":"Hello World"}`

---

## 📸 Checklist screenshots (8)

| # | Screenshot |
|---|---|
| 1 | IAM user + Access Key créés |
| 2 | Security Group inbound rules (22, 80, 3000, 8080) |
| 3 | Instance EC2 running avec IP |
| 4 | `docker --version` sur EC2 |
| 5 | Images `ossian7/landing` et `ossian7/api` sur DockerHub |
| 6 | Pipeline Jenkins 6 stages verts |
| 7 | Landing page dans le navigateur |
| 8 | API → JSON `{"message":"Hello World"}` |

## Pipeline Jenkins

```groovy
pipeline {
    agent any
    stages {
        stage('Checkout') { ... }
        stage('Build Landing') { ... }
        stage('Build API') { ... }
        stage('Push to DockerHub') { ... }
        stage('Deploy Landing') { ... }
        stage('Deploy API') { ... }
    }
}
```

Le Jenkinsfile complet se trouve dans `tp3/Jenkinsfile`.
