# 1.1 — Image Naïve

| Image        | ID           | Disk Usage | Content Size |
| ------------ | ------------ | ---------- | ------------ |
| tp2-naive:v1 | 7ae683cf0d09 | 281MB      | 64.1MB       |

# 1.2 — Image Multi-Stage

| Image      | ID           | Disk Usage | Content Size |
| ---------- | ------------ | ---------- | ------------ |
| tp2-app:v1 | 29ae6402a364 | 202MB      | 49.8MB       |

**Réduction de taille (Disk Usage) :** ~28.1%

**Question :** Pourquoi les outils de build (TypeScript, gcc, etc.) ne doivent-ils **pas** être présents dans l'image de production ?
**Réponse :**

1. **Sécurité :** Moins d'outils installés réduit la surface d'attaque (un attaquant ne peut pas compiler de malware sur le conteneur).
2. **Poids de l'image :** L'image est plus légère, donc plus rapide à télécharger et à déployer (cold start réduit).
3. **Performance :** L'image contient uniquement ce qui est nécessaire à l'exécution.

# 1.3 — .dockerignore

Le fichier `.dockerignore` a été créé pour exclure `node_modules`, `dist` et les fichiers de configuration Docker afin de ne pas alourdir le contexte de build.

# Partie 2 — Docker Compose : Stack App + PostgreSQL

## 2.2 — Orchestration

Le fichier `docker-compose.yml` configure deux services : `web` (l'application Node.js) et `db` (PostgreSQL 16).

**Question :** Pourquoi utilise-t-on `condition: service_healthy` plutôt que `condition: service_started` pour `depends_on` ?
**Réponse :** `service_started` vérifie seulement que le conteneur a démarré, mais la base de données peut prendre plusieurs secondes pour initialiser ses fichiers et accepter des connexions. `service_healthy` s'appuie sur le `healthcheck` (`pg_isready`) pour garantir que la DB est réellement opérationnelle avant de lancer l'application, évitant ainsi des erreurs de connexion au démarrage.

## 2.3 — Tests de la stack

Les tests effectués avec la stack active montrent que la communication entre l'application et la base de données fonctionne :

- **Route `/health`** :
  ```json
  { "status": "ok", "database": "connected" }
  ```
- **Route `/db` (Incrémentation)** :
  - Premier appel : `{ "total_visits": 1 }`
  - Appels suivants : `{ "total_visits": 4 }` (confirmé par les tests de validation)

### Commandes Docker Compose utilisées :

```bash
# Démarrer tous les services en arrière-plan
docker-compose up -d

# Vérifier l'état des services (doivent être "running" et "healthy")
docker-compose ps

# Tester l'application
curl http://localhost:8080/
curl http://localhost:8080/health
curl http://localhost:8080/db # Premier appel → total_visits: 1
curl http://localhost:8080/db # Second appel → total_visits: 2

# Voir les logs en temps réel
docker-compose logs -f

# Arrêter sans supprimer les volumes (données conservées)
docker-compose stop

# Arrêter ET supprimer les volumes (reset complet)
docker-compose down -v
```

# Partie 3 — Artifact Registry & Push de l'Image

Artifact Registry permet de stocker et gérer nos images Docker de manière privée sur GCP.

## 3.1 — Création du Repository

La création du dépôt Docker dans la région `europe-west9` :

```powershell
gcloud artifacts repositories create tp2-registry `
--repository-format=docker `
--location=europe-west9 `
--description="Registry TP2 YNOV"
```

## 3.2 — Authentification Docker

Configuration de Docker pour utiliser les identifiants GCP :

```powershell
gcloud auth configure-docker europe-west9-docker.pkg.dev
```

## 3.3 — Tag et Push

L'image optimisée a été taguée et poussée vers le registre :

```powershell
$PROJECT_ID = (gcloud config get-value project)
$IMAGE_TAG = "europe-west9-docker.pkg.dev/$PROJECT_ID/tp2-registry/tp2-app:v1"

docker tag tp2-app:v1 $IMAGE_TAG
docker push $IMAGE_TAG
```

**Vérification de l'image distantes :**

```powershell
$PROJECT_ID = (gcloud config get-value project)
gcloud artifacts docker images list "europe-west9-docker.pkg.dev/${PROJECT_ID}/tp2-registry/tp2-app"
```

# Partie 4 — Déploiement sur Cloud Run

Cloud Run permet de déployer des conteneurs de manière serverless.

## 4.1 — Déploiement du Service

Commande utilisée pour le déploiement initial :

```powershell
$PROJECT_ID = (gcloud config get-value project)
$IMAGE = "europe-west9-docker.pkg.dev/$PROJECT_ID/tp2-registry/tp2-app:v1"

gcloud run deploy tp2-service `
--image=$IMAGE `
--region=europe-west9 `
--platform=managed `
--allow-unauthenticated `
--port=8080 `
--memory=512Mi `
--cpu=1 `
--max-instances=3 `
--set-env-vars="APP_ENV=production"
```

## 4.2 — Tests du Déploiement Public

L'URL publique générée est : `https://tp2-service-856724546283.europe-west9.run.app/`

### Test

{"message":"Hello from YNOV Cloud TP2","version":"2.0.0","stage":"production"}

## 4.3 - Tests de performance

Measure-Command { Invoke-RestMethod https://tp2-service-856724546283.europe-west9.run.app/health }

Days : 0
Hours : 0
Minutes : 0
Seconds : 0
Milliseconds : 263
Ticks : 2636027
TotalDays : 3,05095717592593E-06
TotalHours : 7,32229722222222E-05
TotalMinutes : 0,00439337833333333
TotalSeconds : 0,2636027
TotalMilliseconds : 263,6027

# Partie 5 — Networking GCP : VPC & Firewall

Cette partie nous a permis de comprendre comment isoler et sécuriser nos ressources sur GCP.

## 5.1 — VPC & Subnets

Nous avons créé un VPC personnalisé pour un contrôle total sur l'adressage :

```powershell
# Création du VPC
gcloud compute networks create tp2-vpc --subnet-mode=custom

# Création des sous-réseaux
gcloud compute networks subnets create tp2-subnet-public `
--network=tp2-vpc --region=europe-west9 --range=10.10.1.0/24

gcloud compute networks subnets create tp2-subnet-private `
--network=tp2-vpc --region=europe-west9 --range=10.10.2.0/24
```

**Question :** Pourquoi sépare-t-on les ressources applicatives et les bases de données dans des sous-réseaux différents ?
**Réponse :** Pour appliquer le principe de **défense en profondeur**. En isolant la base de données dans un sous-réseau privé (sans IP publique), on s'assure qu'elle n'est pas accessible directement depuis Internet, même si le pare-feu est mal configuré. Seul le sous-réseau public (où se trouve l'application) peut communiquer avec elle.

## 5.2 — Firewall Rules

Configuration des accès entrants (INGRESS) :

```powershell
# Autoriser HTTP et HTTPS pour le tag 'http-server'
gcloud compute firewall-rules create tp2-allow-http --network=tp2-vpc --direction=INGRESS --action=ALLOW --rules=tcp:80 --source-ranges=0.0.0.0/0 --target-tags=http-server
gcloud compute firewall-rules create tp2-allow-https --network=tp2-vpc --direction=INGRESS --action=ALLOW --rules=tcp:443 --source-ranges=0.0.0.0/0 --target-tags=http-server

# Autoriser Postgres UNIQUEMENT depuis le subnet public
gcloud compute firewall-rules create tp2-allow-postgres --network=tp2-vpc --direction=INGRESS --action=ALLOW --rules=tcp:5432 --source-ranges=10.10.1.0/24 --target-tags=db-server
```

**Question :** Quelle est la différence entre un Security Group (AWS) et une Firewall Rule (GCP) ?
**Réponse :**

1. **Scope** : Un SG AWS est rattaché à une interface réseau (instance), alors qu'une règle GCP est définie au niveau du VPC et cible les instances via des **Network Tags**.
2. **Flexibilité** : GCP permet des règles d'autorisation (ALLOW) et de refus (DENY). AWS SG ne gère que l'autorisation (le refus est implicite).
3. **Ciblage** : GCP utilise les tags de manière très dynamique pour appliquer des règles à des groupes d'instances sans modifier leur configuration réseau.

## 6.1 - Création du bucket

```powershell
# Définir les variables pour ton projet
$PROJECT_ID = (gcloud config get-value project)
$BUCKET = "ynov-tp2-versioned-${PROJECT_ID}"

# Créer le bucket dans la région de Paris
gcloud storage buckets create gs://${BUCKET} --location=europe-west9 --uniform-bucket-level-access
```

```powershell
# Activer le versioning
gcloud storage buckets update gs://${BUCKET} --versioning
```

```powershell
# Vérifier que c'est bien activé
gcloud storage buckets describe gs://${BUCKET} --format="value(versioning.enabled)"
```

```powershell
# Créer un fichier de configuration local
echo "Version 1" > config.json

# Uploader vers le bucket (Version 1)
gcloud storage cp config.json gs://${BUCKET}/
```

```powershell
# Créer un fichier de configuration local
echo "Version 2" > config.json

# Uploader vers le bucket (Version 1)
gcloud storage cp config.json gs://${BUCKET}/
```

```powershell
# Créer un fichier de configuration local
echo "Version 3" > config.json

# Uploader vers le bucket (Version 1)
gcloud storage cp config.json gs://${BUCKET}/
```

```powershell
gcloud storage ls -a gs://${BUCKET}/config.json

gs://ynov-tp2-versioned-developper-pour-le-cloud/config.json#1775554485506440
gs://ynov-tp2-versioned-developper-pour-le-cloud/config.json#1775554499417884
gs://ynov-tp2-versioned-developper-pour-le-cloud/config.json#1775554505775205

```

## 6.3 - Règles de lifecycle
Le fichier `lifecycle.json` définit des règles automatiques pour optimiser le stockage :
```json
{
  "rule": [
    {
      "action": { "type": "Delete" },
      "condition": {
        "numNewerVersions": 3,
        "isLive": false
      }
    },
    {
      "action": {
        "type": "SetStorageClass",
        "storageClass": "NEARLINE"
      },
      "condition": {
        "age": 30,
        "isLive": true
      }
    }
  ]
}
```

**Question :** Expliquez les deux règles lifecycle configurées ci-dessus. Quel est l'intérêt économique de passer en classe NEARLINE après 30 jours ?
**Réponse :** 
- **Règle 1** : Supprime automatiquement les versions non-actuelles d'un objet (anciennes versions) dès qu'il existe plus de 3 versions plus récentes. Cela évite l'accumulation infinie de fichiers obsolètes.
- **Règle 2** : Change la classe de stockage des objets actuels en **NEARLINE** après 30 jours sans modification.
- **Intérêt économique** : La classe NEARLINE est moins coûteuse au Go/mois que la classe STANDARD. C'est idéal pour des données que l'on doit conserver (archives, backups) mais que l'on consulte peu souvent, permettant de réduire la facture Cloud de manière significative.

# Partie 7 — Gestion de trafic : Canary Release & Traffic Splitting

Le traffic splitting permet de tester une nouvelle version (Canary) sur une petite portion d'utilisateurs avant un déploiement global.

## 7.1 — Préparation de la V2
Nous avons modifié le code pour passer en version `2.1.0` et déployé cette nouvelle révision sans lui envoyer de trafic immédiatement :
```powershell
gcloud run deploy tp2-service --image=$IMAGE_V2 --no-traffic
```

## 7.2 — Configuration du Traffic Splitting
Nous avons réparti le trafic entre l'ancienne (80%) et la nouvelle (20%) révision :
```powershell
gcloud run services update-traffic tp2-service `
--to-revisions="tp2-service-v1=80,tp2-service-v2=20"
```

### Résultats des tests (10 requêtes) :
*   Version 2.0.0 (v1) : ~80-90% des réponses.
*   Version 2.1.0 (v2) : ~10-20% des réponses (Canary).

**Question :** Quel avantage voyez-vous à utiliser `--no-traffic` lors d'un déploiement ?
**Réponse :** Cela permet de déployer et de préparer l'infrastructure (vérifier que le conteneur démarre bien, faire des tests de fumée via l'URL directe de la révision) sans impacter les utilisateurs réels. On ne redirige le trafic que lorsque l'on est certain de la stabilité de la nouvelle version.

# Partie 8 — Cache Redis & Performance

L'ajout d'un cache Redis permet d'alléger la charge sur la base de données PostgreSQL et d'accélérer les temps de réponse.

## 8.1 — Docker Compose avec Redis
Nous avons ajouté un service `cache` utilisant l'image `redis:7-alpine` et configuré l'application pour s'y connecter.

## 8.2 — Implémentation du Cache (route /cached)
L'application utilise désormais le client `redis` pour stocker le nombre de visites avec un TTL de 10 secondes.

## 8.3 — Tests du cache
```powershell
# Première requête → source: "database" (cache froid)
Invoke-RestMethod http://localhost:8080/cached

# Requête suivante avant 10s → source: "cache" (cache chaud)
Invoke-RestMethod http://localhost:8080/cached
```

**Question :** Quel est l'intérêt du TTL (Time-To-Live) dans un cache ?
**Réponse :** Le TTL garantit que les données ne restent pas indéfiniment en mémoire. Cela permet de rafraîchir périodiquement le cache pour limiter l'obsolescence des données et de libérer de la RAM sur le serveur Redis en supprimant automatiquement les clés anciennes.

**Question :** Dans quelle situation l'utilisation d'un cache Redis peut-elle poser un problème de cohérence des données ?
**Réponse :** Si une donnée est modifiée dans PostgreSQL mais que l'ancienne valeur est encore stockée dans Redis avec un TTL actif, l'utilisateur verra une donnée erronée. Pour éviter cela, on peut utiliser une stratégie d'**invalidation de cache** (supprimer la clé Redis dès que la DB est mise à jour).

# Partie 9 — Nettoyage Final

Pour clôturer le TP et éviter toute facturation sur GCP, nous avons supprimé l'intégralité des ressources cloud :

```powershell
# 1. Supprimer le service Cloud Run
gcloud run services delete tp2-service --region=europe-west9 --quiet

# 2. Supprimer le registre d'images
gcloud artifacts repositories delete tp2-registry --location=europe-west9 --quiet

# 3. Supprimer le bucket et son contenu
gcloud storage rm --recursive gs://ynov-tp2-versioned-developper-pour-le-cloud

# 4. Supprimer le réseau (Firewall, Subnets, VPC)
gcloud compute firewall-rules delete tp2-allow-http tp2-allow-https tp2-allow-postgres --quiet
gcloud compute networks subnets delete tp2-subnet-public tp2-subnet-private --region=europe-west9 --quiet
gcloud compute networks delete tp2-vpc --quiet
```
