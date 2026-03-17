# Projet AWS : Alertes d’offres d’alternance automatisées

## Objectif

À chaque **nouvelle offre déposée** dans un **bucket S3** :

- La **stocker** dans **DynamoDB**
- **Envoyer** une notification via **SNS** aux étudiants concernés
- **Générer** des logs dans **CloudWatch Logs**

---

## Prérequis

- **AWS CLI** installé et configuré (`aws configure`)
- **Compte AWS** avec droits pour : S3, SNS, Lambda, DynamoDB, IAM
- **Python 3.11** ou supérieur pour Lambda

## Remarque

Ce lab doit être réalisé dans **AWS CloudShell**.

## Déploiement

### Créer le Topic SNS

```Bash
# Créer un Topic SNS pour notifications
TOPIC_ARN=$(aws sns create-topic \
    --name AlertesOffres \
    --query 'TopicArn' \
    --output text \
    --region eu-west-3)
echo "SNS Topic ARN : $TOPIC_ARN"

# S’abonner à ce topic (email réel à confirmer dans la boîte)
aws sns subscribe \
    --topic-arn $TOPIC_ARN \
    --protocol email \
    --notification-endpoint pape.lo@estiam.com \
    --region eu-west-3
```

SNS Topic ARN : `arn:aws:sns:eu-west-3:157125373775:AlertesOffres`

![image](https://hackmd.io/_uploads/r1Mi3uW5be.png)

![image](https://hackmd.io/_uploads/S101pOW9bg.png)

![image](https://hackmd.io/_uploads/BkC-p_Zqbl.png)

Confirmation

![image](https://hackmd.io/_uploads/B1lOmp_bcbl.png)

![image](https://hackmd.io/_uploads/HJ24pOZ9Wl.png)
![image](https://hackmd.io/_uploads/BkSDpd-cWe.png)

Si vous avez une liste d'étudiante

- Prépare le fichier `etudiants.txt` ou `csv` (un email par ligne).
- Lance cette boucle dans ton terminal :

```Bash
for email in $(cat etudiants.txt); do
  aws sns subscribe \
    --topic-arn $TOPIC_ARN \
    --protocol email \
    --notification-endpoint $email \
    --region eu-west-3
done

```

Chaque étudiant recevra alors un mail de confirmation d'AWS.

### Créer la table DynamoDB

```Bash
# Créer la table DynamoDB
aws dynamodb create-table \
    --table-name TableOffres \
    --attribute-definitions AttributeName=ID,AttributeType=S \
    --key-schema AttributeName=ID,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST \
    --region eu-west-3
```

![image](https://hackmd.io/_uploads/ByZ2pOWqbg.png)
![image](https://hackmd.io/_uploads/Sy-R6OWqWl.png)

### Créer la table étudiant

```Bash
aws dynamodb create-table \
    --table-name TableEtudiants \
    --attribute-definitions AttributeName=Email,AttributeType=S \
    --key-schema AttributeName=Email,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST \
    --region eu-west-3
```

![image](https://hackmd.io/_uploads/Sy5xA_Z5Zl.png)
![image](https://hackmd.io/_uploads/HywQ0OW9Zl.png)

### Insertion d'un étudiant

```Bash
aws dynamodb put-item \
    --table-name TableEtudiants \
    --item '{
        "Email": {"S": "pape.lo@estiam.com"},
        "Nom": {"S": "Pape Lo"},
        "Domaines": {"SS": ["DevOps", "Cloud"]}
    }' \
    --region eu-west-3
```

![image](https://hackmd.io/_uploads/rylv0u-9-l.png)
![image](https://hackmd.io/_uploads/By0oC_-cWg.png)

Si vous avez plusieurs étudiant avec différentes spécialités

- Crée un fichier nommé `etudiants.csv`
  Par exemple

```txt
daf.lo@estiam.com,Cloud
jean.dupont@estiam.com,Cybersecurity
marie.curie@estiam.com,Architecture
lucas.web@estiam.com,Web et Mobile
```

Utilise ce petit script pour envoyer toute ta liste dans DynamoDB en une seule fois. Ce script va créer une entrée pour chaque étudiant avec son domaine spécifique.

```Python
import boto3
import csv

dynamodb = boto3.resource('dynamodb', region_name='eu-west-3')
table = dynamodb.Table('TableEtudiants')

with open('etudiants.csv', mode='r') as f:
    reader = csv.reader(f)
    for row in reader:
        email, domaine = row[0], row[1]
        table.put_item(Item={
            'Email': email,
            'Nom': email.split('.')[0].capitalize(), # Extrait le prénom
            'Domaines': [domaine, 'Général']        # L'inscrit à son domaine + Général
        })
        print(f"Étudiant {email} ajouté en {domaine}")
```

#### Comment le système va réagir ?

Une fois la **table remplie**, voici comment l'infrastructure va fonctionner :

- Si on uploade : `Cloud_Offre_Amazon.pdf`

1. **Lambda** :
   - Scanne la **table DynamoDB**
   - Voit que **Marie** est en "Architecture" (elle n’est pas concernée)
   - Voit que **Daf** est en "Cloud" (il est concerné)

2. **SNS** :
   - Envoie l'**email uniquement à Daf**

## Mapping Domaine → Préfixe du fichier S3

| Domaine               | Préfixe du fichier S3 | Exemple de nom de fichier |
| --------------------- | --------------------- | ------------------------- |
| Cloud                 | Cloud\_               | Cloud_Architecte_AWS.pdf  |
| Cybersecurity         | Cyber\_               | Cyber_Analyste_SOC.pdf    |
| Architecture          | Archi\_               | Archi_Urbaniste_SI.pdf    |
| Web et Mobile         | Web\_                 | Web_Dev_Fullstack.pdf     |
| Autre (Tout le reste) | Offre_Generale.txt    | Offre_Generale.txt        |

### Créer un rôle IAM pour la Lambda

```Bash
# Créer la trust policy
cat > trust-policy.json << EOL
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": { "Service": "lambda.amazonaws.com" },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOL
```

![image](https://hackmd.io/_uploads/S1SJkK-5Wx.png)

```Bash
# Créer le rôle
aws iam create-role \
    --role-name LambdaOffreRole \
    --assume-role-policy-document file://trust-policy.json
```

![image](https://hackmd.io/_uploads/Bk_bJFZcbx.png)
![image](https://hackmd.io/_uploads/HkZ8yKZcWe.png)

```Bash
# Attacher les policies nécessaires
aws iam attach-role-policy --role-name LambdaOffreRole --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam attach-role-policy --role-name LambdaOffreRole --policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess
aws iam attach-role-policy --role-name LambdaOffreRole --policy-arn arn:aws:iam::aws:policy/AmazonSNSFullAccess
aws iam attach-role-policy --role-name LambdaOffreRole --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

![image](https://hackmd.io/_uploads/B1iQkYbqZl.png)

### Créer le Bucket S3 (déclencheur)

```Bash
BUCKET_NAME="offres-alternance-$(date +%s)"
aws s3 mb s3://$BUCKET_NAME --region eu-west-3
echo "Bucket créé : $BUCKET_NAME"
```

![image](https://hackmd.io/_uploads/rJeq1t-5Zg.png)
![image](https://hackmd.io/_uploads/ByvjyYZ9be.png)

### Créer la Lambda

Fichier `lambda_function.py`

```Bash
nano lambda_function.py
```

Copier ce code

```Python
import boto3
import uuid
import os
from datetime import datetime
from botocore.exceptions import ClientError
from botocore.config import Config

# CONFIGURATION ULTIME : On force le style d'adressage et la signature V4
s3_config = Config(
    region_name='eu-west-3',
    signature_version='s3v4',
    s3={'addressing_style': 'virtual'}
)

# Initialisation des clients avec la configuration renforcée
s3_client = boto3.client('s3', config=s3_config)
dynamodb = boto3.resource('dynamodb', region_name='eu-west-3')
sns = boto3.client('sns', region_name='eu-west-3')

TABLE_OFFRES = 'TableOffres'
TABLE_ETUDIANTS = 'TableEtudiants'

def lambda_handler(event, context):
    # 1. Extraction des données S3
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']

    # 2. Logique de domaine
    if '_' in key:
        domaine = key.split('_')[0]
    else:
        domaine = "Général"

    date_pub = datetime.utcnow().strftime("%d/%m/%Y à %H:%M UTC")

    # 3. Génération de l'URL Présignée (Correctif Signature)
    try:
        url_securisee = s3_client.generate_presigned_url('get_object',
            Params={'Bucket': bucket, 'Key': key},
            ExpiresIn=3600)
        print(f"URL générée pour {key}")
    except ClientError as e:
        print(f"Erreur S3: {e}")
        return {"status": "error"}

    # 4. Enregistrement DynamoDB
    table_offres = dynamodb.Table(TABLE_OFFRES)
    offre_id = str(uuid.uuid4())
    table_offres.put_item(Item={
        'ID': offre_id,
        'NomFichier': key,
        'URL': url_securisee,
        'DatePublication': date_pub,
        'Domaine': domaine
    })

    # 5. Scan des étudiants et notification
    table_etudiants = dynamodb.Table(TABLE_ETUDIANTS)
    etudiants = table_etudiants.scan().get('Items', [])

    count = 0
    for etudiant in etudiants:
        domaines_etudiant = etudiant.get('Domaines', [])
        nom_etudiant = etudiant.get('Nom', 'Étudiant')

        if domaine in domaines_etudiant:
            sujet = f"🚀 Nouvelle offre Alternance : {domaine}"

            corps_message = f"""
Bonjour {nom_etudiant},

Une nouvelle offre vient d'être publiée :

📌 Poste : {domaine}
📂 Fichier : {key}
🗓 Date : {date_pub}

🔗 LIEN DE TÉLÉCHARGEMENT (Valide 1h) :
{url_securisee}

Cordialement,
L'équipe Recrutement
"""
            sns.publish(
                TopicArn=os.environ['SNS_TOPIC_ARN'],
                Message=corps_message,
                Subject=sujet
            )
            count += 1

    return {"status": "success", "sent": count}

```

![image](https://hackmd.io/_uploads/ByDmlt-9-e.png)

### Déployer la Lambda

```Bash
# Zipper le fichier
zip function.zip lambda_function.py

# Récupérer l'ARN du rôle IAM
ROLE_ARN=$(aws iam get-role --role-name LambdaOffreRole --query 'Role.Arn' --output text)

# Créer la fonction Lambda
aws lambda create-function \
    --function-name NotifieurOffres \
    --zip-file fileb://function.zip \
    --handler lambda_function.lambda_handler \
    --runtime python3.11 \
    --role $ROLE_ARN \
    --environment "Variables={SNS_TOPIC_ARN=$TOPIC_ARN}" \
    --region eu-west-3
```

![image](https://hackmd.io/_uploads/SJpsgtZ5Zg.png)
![image](https://hackmd.io/_uploads/BkapxKW9Zl.png)
![image](https://hackmd.io/_uploads/rJRClKZ9bg.png)

### Connecter S3 → Lambda

```Bash
# Autoriser S3 à invoquer Lambda
aws lambda add-permission \
    --function-name NotifieurOffres \
    --statement-id s3-invoke \
    --action "lambda:InvokeFunction" \
    --principal s3.amazonaws.com \
    --source-arn arn:aws:s3:::$BUCKET_NAME \
    --region eu-west-3

# Configurer la notification S3
cat > notification.json << EOL
{
  "LambdaFunctionConfigurations": [
    {
      "LambdaFunctionArn": "$(aws lambda get-function --function-name NotifieurOffres --query "Configuration.FunctionArn" --output text)",
      "Events": ["s3:ObjectCreated:*"]
    }
  ]
}
EOL

aws s3api put-bucket-notification-configuration \
    --bucket $BUCKET_NAME \
    --notification-configuration file://notification.json
```

![image](https://hackmd.io/_uploads/Bk3ZbK-qZg.png)
![image](https://hackmd.io/_uploads/BJENZKb9bg.png)
![image](https://hackmd.io/_uploads/HyVubtb9-l.png)

### Test

#### Tester le SNS

```Bash
aws sns publish \
  --topic-arn arn:aws:sns:eu-west-3:157125373775:AlertesOffres \
  --message "Ceci est un test manuel. Si je reçois cet email, SNS fonctionne !" \
  --subject "TEST SNS DIRECT"
```

![image](https://hackmd.io/_uploads/r18xPtWqZe.png)
![image](https://hackmd.io/_uploads/BygWwKbqZe.png)

### Test envoie offre-1

```Bash
echo "Contenu offre" > DevOps_OffreJanvier.pdf
aws s3 cp DevOps_OffreJanvier.pdf s3://$BUCKET_NAME/
```

![image](https://hackmd.io/_uploads/HkM4utZqZe.png)
![image](https://hackmd.io/_uploads/H11LOFZqbl.png)

### Test envoie offre-2

```Bash
echo "Contenu offre" > Cloud_Architecte.pdf
aws s3 cp Cloud_Architecte.pdf s3://$BUCKET_NAME/
```

![image](https://hackmd.io/_uploads/Bycc_tbqbx.png)
![image](https://hackmd.io/_uploads/SJGadYZq-g.png)

Vérifier DynamoDB

```Bash
aws dynamodb scan --table-name TableOffres --region eu-west-3
```

![image](https://hackmd.io/_uploads/SkLktYbcWg.png)
![image](https://hackmd.io/_uploads/SJ6QtK-9Wl.png)

### Nettoyage

```Bash
# 1. Supprimer la fonction Lambda
aws lambda delete-function --function-name NotifieurOffres --region eu-west-3
```

```Bash
# 2. Supprimer les tables DynamoDB
aws dynamodb delete-table --table-name TableOffres --region eu-west-3
aws dynamodb delete-table --table-name TableEtudiants --region eu-west-3
```

```Bash
# 3. Supprimer le Topic SNS (remplace l'ARN si nécessaire)
aws sns delete-topic --topic-arn arn:aws:sns:eu-west-3:157125373775:AlertesOffres --region eu-west-3
```

```
# 4. Nettoyer les fichiers dans le bucket
aws s3 rm s3://$BUCKET_NAME/ --recursive
```

```Bash
# 5. Vider et supprimer le bucket S3 (Attention: définitif)
aws s3 rb s3://$BUCKET_NAME --force --region eu-west-3
```

```Bash
# Détacher les policies
aws iam detach-role-policy --role-name LambdaOffreRole --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam detach-role-policy --role-name LambdaOffreRole --policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess
aws iam detach-role-policy --role-name LambdaOffreRole --policy-arn arn:aws:iam::aws:policy/AmazonSNSFullAccess
aws iam detach-role-policy --role-name LambdaOffreRole --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Supprimer le rôle
aws iam delete-role --role-name LambdaOffreRole

```

```Bash
# 6. Supprimer les fichiers locaux
rm -f Cloud_Architecte.pdf DevOps_OffreJanvier.pdf function.zip lambda_function.py notification.json offre-test.txt response.json test-notification.txt test.txt trust-policy.json
```
