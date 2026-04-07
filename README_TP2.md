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
$PROJECT_ID = (gcloud config get-value project)

> > gcloud artifacts docker images list "europe-west9-docker.pkg.dev/${PROJECT_ID}/tp2-registry/tp2-app"
> > Listing items under project developper-pour-le-cloud, location europe-west9, repository tp2-registry.

IMAGE DIGEST CREATE_TIME UPDATE_TIME SIZE
europe-west9-docker.pkg.dev/developper-pour-le-cloud/tp2-registry/tp2-app sha256:29ae6402a364b35b86cb5d071fd8f3929378f19f3e261a11561d39456215428b 2026-04-07T10:02:57 2026-04-07T10:02:57
