# Alertes d’offres d’alternance automatisées (AWS S3 → Lambda → DynamoDB → SNS)

Projet AWS serverless : à chaque **nouvelle offre** déposée dans un **bucket S3**, une **Lambda** est déclenchée pour **enregistrer l’offre dans DynamoDB** et **notifier via SNS** les étudiants dont le domaine correspond. Les exécutions et erreurs sont consultables dans **CloudWatch Logs**.

> Ce dépôt contient un guide de déploiement pas à pas (AWS CLI / CloudShell) : `LAB-024-S3-Lambda-SNS-Alertes-offres-alternance.md`.

---

## Fonctionnalités

- **Déclenchement automatique** sur upload S3 (`s3:ObjectCreated:*`)
- **Enregistrement** des offres dans **DynamoDB** (`TableOffres`)
- **Ciblage par domaine** des étudiants (table `TableEtudiants`)
- **Notification email** via **SNS** (Topic `AlertesOffres`)
- **Lien de téléchargement sécurisé** (URL présignée S3, valide 1h)
- **Observabilité** via **CloudWatch Logs**

---

## Architecture

1. **S3** reçoit un fichier (ex: `Cloud_Architecte_AWS.pdf`)
2. **S3 Event** déclenche la **Lambda**
3. La **Lambda** :
   - déduit le **domaine** à partir du nom de fichier (préfixe avant `_`)
   - génère une **URL présignée** pour téléchargement
   - écrit un enregistrement dans **DynamoDB** (`TableOffres`)
   - scanne `TableEtudiants` et envoie une **notification SNS** aux étudiants dont le domaine match
4. **SNS** distribue l’email aux abonnés du topic
5. **CloudWatch Logs** conserve les logs d’exécution


---

## Prérequis

- **Compte AWS** avec droits : S3, Lambda, SNS, DynamoDB, IAM, CloudWatch
- **AWS CLI** configuré (`aws configure`)
- **Python 3.11+** (runtime Lambda)
- Recommandé : exécuter dans **AWS CloudShell** (tel que prévu dans le lab)
- Région utilisée dans le lab : **`eu-west-3`**

---

## Déploiement (AWS CLI / CloudShell)

Les commandes détaillées sont dans `LAB-024-S3-Lambda-SNS-Alertes-offres-alternance.md`. Ci-dessous un condensé “quickstart”.

### 1) Créer le topic SNS + abonnement email

```bash
TOPIC_ARN=$(aws sns create-topic \
  --name AlertesOffres \
  --query 'TopicArn' \
  --output text \
  --region eu-west-3)

aws sns subscribe \
  --topic-arn "$TOPIC_ARN" \
  --protocol email \
  --notification-endpoint "ton.email@exemple.com" \
  --region eu-west-3

echo "Topic ARN: $TOPIC_ARN"
```

> Important : confirmer l’abonnement depuis l’email reçu.

### 2) Créer les tables DynamoDB

```bash
aws dynamodb create-table \
  --table-name TableOffres \
  --attribute-definitions AttributeName=ID,AttributeType=S \
  --key-schema AttributeName=ID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region eu-west-3

aws dynamodb create-table \
  --table-name TableEtudiants \
  --attribute-definitions AttributeName=Email,AttributeType=S \
  --key-schema AttributeName=Email,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region eu-west-3
```

### 3) Insérer des étudiants (exemple)

```bash
aws dynamodb put-item \
  --table-name TableEtudiants \
  --item '{
    "Email": {"S": "etudiant@exemple.com"},
    "Nom": {"S": "Etudiant"},
    "Domaines": {"SS": ["Cloud", "DevOps", "Général"]}
  }' \
  --region eu-west-3
```

### 4) Créer le rôle IAM de la Lambda

```bash
cat > trust-policy.json << 'EOL'
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

aws iam create-role \
  --role-name LambdaOffreRole \
  --assume-role-policy-document file://trust-policy.json

aws iam attach-role-policy --role-name LambdaOffreRole --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam attach-role-policy --role-name LambdaOffreRole --policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess
aws iam attach-role-policy --role-name LambdaOffreRole --policy-arn arn:aws:iam::aws:policy/AmazonSNSFullAccess
aws iam attach-role-policy --role-name LambdaOffreRole --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

> Bonne pratique : en production, remplacer ces policies larges par une policy **least privilege**.

### 5) Créer le bucket S3 (déclencheur)

```bash
BUCKET_NAME="offres-alternance-$(date +%s)"
aws s3 mb "s3://$BUCKET_NAME" --region eu-west-3
echo "Bucket: $BUCKET_NAME"
```

### 6) Créer & déployer la Lambda

Crée `lambda_function.py` (voir le code complet dans le lab), puis :

```bash
zip function.zip lambda_function.py

ROLE_ARN=$(aws iam get-role \
  --role-name LambdaOffreRole \
  --query 'Role.Arn' \
  --output text)

aws lambda create-function \
  --function-name NotifieurOffres \
  --zip-file fileb://function.zip \
  --handler lambda_function.lambda_handler \
  --runtime python3.11 \
  --role "$ROLE_ARN" \
  --environment "Variables={SNS_TOPIC_ARN=$TOPIC_ARN}" \
  --region eu-west-3
```

### 7) Connecter S3 → Lambda (event notification)

```bash
aws lambda add-permission \
  --function-name NotifieurOffres \
  --statement-id s3-invoke \
  --action "lambda:InvokeFunction" \
  --principal s3.amazonaws.com \
  --source-arn "arn:aws:s3:::$BUCKET_NAME" \
  --region eu-west-3

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
  --bucket "$BUCKET_NAME" \
  --notification-configuration file://notification.json
```

---

## Tests

### Tester SNS (envoi direct)

```bash
aws sns publish \
  --topic-arn "$TOPIC_ARN" \
  --message "Test SNS" \
  --subject "TEST SNS" \
  --region eu-west-3
```

### Tester le flux complet (upload S3)

```bash
echo "Contenu offre" > Cloud_Architecte.pdf
aws s3 cp Cloud_Architecte.pdf "s3://$BUCKET_NAME/"
```

Vérifier les écritures dans DynamoDB :

```bash
aws dynamodb scan --table-name TableOffres --region eu-west-3
```

Logs Lambda :
- Console AWS → **Lambda** → `NotifieurOffres` → **Monitor** → **View logs in CloudWatch**

---

## Variables d’environnement (Lambda)

- **`SNS_TOPIC_ARN`** *(obligatoire)* : ARN du topic SNS utilisé pour publier les notifications

---

## Nettoyage (éviter les coûts)

```bash
aws lambda delete-function --function-name NotifieurOffres --region eu-west-3

aws dynamodb delete-table --table-name TableOffres --region eu-west-3
aws dynamodb delete-table --table-name TableEtudiants --region eu-west-3

aws sns delete-topic --topic-arn "$TOPIC_ARN" --region eu-west-3

aws s3 rm "s3://$BUCKET_NAME/" --recursive
aws s3 rb "s3://$BUCKET_NAME" --force --region eu-west-3

aws iam detach-role-policy --role-name LambdaOffreRole --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess
aws iam detach-role-policy --role-name LambdaOffreRole --policy-arn arn:aws:iam::aws:policy/AmazonDynamoDBFullAccess
aws iam detach-role-policy --role-name LambdaOffreRole --policy-arn arn:aws:iam::aws:policy/AmazonSNSFullAccess
aws iam detach-role-policy --role-name LambdaOffreRole --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name LambdaOffreRole
```

---

## Sécurité & bonnes pratiques (recommandé)

- **Least privilege IAM** (S3 `GetObject` sur un bucket précis, DynamoDB `PutItem/Scan` sur tables ciblées, SNS `Publish` sur un topic)
- **Filtrer le trigger S3** (préfixe/suffixe) pour éviter les déclenchements non désirés
- **Éviter le `Scan` DynamoDB** à grande échelle : préférer une stratégie par index / partitionnement / table de correspondance
- **DLQ / retries** : configurer DLQ (SQS) ou destinations Lambda
- **Chiffrement** : SSE-S3/SSE-KMS pour S3, KMS pour SNS si nécessaire

---

## Troubleshooting

- **Je ne reçois pas d’email SNS** :
  - vérifier que l’abonnement est **Confirmed**
  - vérifier la région (`eu-west-3`) et l’ARN
- **La Lambda n’est pas déclenchée** :
  - vérifier la configuration `put-bucket-notification-configuration`
  - vérifier l’`add-permission` sur la Lambda (principal `s3.amazonaws.com`)
- **AccessDenied** :
  - revoir les policies du rôle `LambdaOffreRole`
  - vérifier que `SNS_TOPIC_ARN` est bien défini

---



