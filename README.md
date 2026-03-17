# Alertes d’offres d’alternance automatisées (AWS S3 → Lambda → DynamoDB → SNS)

## 🖊️ Auteur

- **Nom :** LO 
- **Prénom :** Pape 
- **Email :** pape.lo@estiam.com  
- **GitHub :** [dspitech](https://github.com/dspitech)  

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

## Mapping Domaine → Préfixe du fichier S3

| Domaine | Préfixe attendu | Exemple |
| --- | --- | --- |
| Cloud | `Cloud_` | `Cloud_Architecte_AWS.pdf` |
| Cybersecurity | `Cyber_` | `Cyber_Analyste_SOC.pdf` |
| Architecture | `Archi_` | `Archi_Urbaniste_SI.pdf` |
| Web et Mobile | `Web_` | `Web_Dev_Fullstack.pdf` |
| Général (fallback) | *(pas de `_`)* | `OffreGenerale.pdf` |

---

## Prérequis

- **Compte AWS** avec droits : S3, Lambda, SNS, DynamoDB, IAM, CloudWatch
- **AWS CLI** configuré (`aws configure`)
- **Python 3.11+** (runtime Lambda)
- Recommandé : exécuter dans **AWS CloudShell** (tel que prévu dans le lab)
- Région utilisée dans le lab : **`eu-west-3`**

---

## Documentation (important)

- **Guide de lab (déploiement complet + commandes AWS CLI + code Lambda + tests + nettoyage)** : `LAB-024-S3-Lambda-SNS-Alertes-offres-alternance.md`

> Le `README` reste volontairement “vitrine projet”. Toute la configuration et les commandes doivent être suivies depuis le fichier de lab.

---

## Comment ça marche (logique fonctionnelle)

- **Détection du domaine** : le domaine est déduit du **préfixe** du fichier avant `_` (ex: `Cloud_...` → `Cloud`).  
  Sans `_`, l’offre est traitée comme **Général**.
- **Ciblage étudiants** : chaque étudiant possède une liste `Domaines` (ex: `["Cloud", "DevOps", "Général"]`).  
  La Lambda notifie uniquement si le domaine de l’offre est présent dans `Domaines`.
- **Traçabilité** : chaque offre est historisée dans `TableOffres` avec un identifiant unique + date + URL présignée.

---

## Configuration (résumé)

- **Région** : `eu-west-3`
- **Ressources AWS** :
  - **S3** : bucket déclencheur (ObjectCreated)
  - **Lambda** : traitement + notification
  - **DynamoDB** : `TableOffres`, `TableEtudiants`
  - **SNS** : topic email
- **Variable d’environnement Lambda** :
  - `SNS_TOPIC_ARN` : ARN du topic SNS

---

## Structure du dépôt

- `README.md` : présentation du projet (ce fichier)
- `LAB-024-S3-Lambda-SNS-Alertes-offres-alternance.md` : guide complet du lab (déploiement / tests / nettoyage)

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
## Roadmap (idées d’amélioration)

- Filtrer par **mots-clés** (ex: extraction texte PDF)
- Remplacer `Scan` par un modèle de données plus scalable (index / table de correspondance)
- Ajouter une **API** de consultation (API Gateway + Lambda)
- Ajouter des **alertes CloudWatch** (erreurs Lambda / throttling)

