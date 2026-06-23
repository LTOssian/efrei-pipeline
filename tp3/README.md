# TP3 — Pipeline CI/CD avec AWS EC2 + Docker + Jenkins

**Équipe** : ALIMOU DIALLO
**DockerHub** : [ossian7](https://hub.docker.com/u/ossian7)

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

📸 **Screenshot** : la sortie de `create-access-key` avec AccessKeyId et SecretAccessKey

### 3. Configurer AWS CLI

```bash
aws configure
# AWS Access Key ID : <coller>
# AWS Secret Access Key : <coller>
# Default region : eu-west-3 (Paris) ou eu-west-1 (Ireland)
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

📸 **Screenshot** : les inbound rules du Security Group

### 6. Lancer l'instance EC2 t2.micro

```bash
# Trouver l'AMI Ubuntu 22.04
aws ec2 describe-images --owners 099720109477 \
  --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
  --query 'Images[*].[ImageId,CreationDate]' --output text | sort -k2 -r | head -1

# Lancer l'instance
aws ec2 run-instances \
  --image-id <AMI_ID> \
  --instance-type t2.micro \
  --key-name devops-key \
  --security-groups devops-sg \
  --count 1

# Récupérer l'IP publique
aws ec2 describe-instances --query 'Reservations[*].Instances[*].PublicIpAddress' --output text
```

📸 **Screenshot** : instance EC2 running dans la console AWS

### 7. Installer Docker sur l'EC2

```bash
ssh -i devops-key.pem ubuntu@<IP_EC2>
sudo apt update && sudo apt install -y docker.io
sudo usermod -aG docker $USER
exit
```

📸 **Screenshot** : `docker --version` dans le terminal SSH

### 8. Construire et pousser les images Docker

```bash
# Landing page
docker build -t ossian7/landing tp3/landing
docker push ossian7/landing

# API
docker build -t ossian7/api tp3/api
docker push ossian7/api
```

📸 **Screenshot** : `ossian7/landing` et `ossian7/api` sur DockerHub

### 9. Configurer Jenkins

- Dans Jenkins, créer un nouveau job Pipeline nommé `tp3-pipeline`
- **Pipeline script from SCM** :
  - SCM : Git
  - Repository URL : `https://github.com/LTOssian/efrei-pipeline.git`
  - Script Path : `tp3/Jenkinsfile`
- Modifier la variable `EC2_HOST` dans le Jenkinsfile avec l'IP de ton EC2
- **Build Now**

📸 **Screenshot** : les 6 stages verts dans Jenkins

### 10. Vérifier le déploiement

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
