# TP 4 — DevSecOps, Architecture Serverless & FinOps
## Cours 4 | Développer pour le Cloud | YNOV Campus Montpellier — Master 2
**Date :** 18/05/2026 | **Durée TP :** 3h30 | **Plateforme :** Google Cloud Platform

---

> **Contexte entreprise — FinSecure**
>
> FinSecure est une fintech française agréée par l'ACPR qui traite des paiements en ligne pour des e-commerçants. Avec 50 000 transactions/jour et une réglementation DSP2 stricte, la sécurité est non-négociable. Après une levée de fonds Série A, l'équipe technique (8 personnes) doit industrialiser ses pratiques : le pipeline CI/CD actuel déploie des images Docker sans scan de vulnérabilités, les secrets sont stockés en variables d'environnement en dur, et la facture GCP a triplé en 3 mois sans raison identifiée. Vous êtes le/la Lead Cloud Engineer recruté(e) pour remédier à tout ça.

---

> **Prérequis validés (Cours 3) :**
> - Cluster GKE Autopilot opérationnel (`tp3-cluster`)
> - Pipeline GitHub Actions fonctionnel (TP3)
> - Artifact Registry configuré avec des images Docker pushées

**Objectifs de ce TP :**
- Intégrer la sécurité dans le pipeline CI/CD (DevSecOps)
- Gérer les secrets avec GCP Secret Manager et Workload Identity
- Construire une architecture serverless event-driven avec Cloud Functions et Pub/Sub
- Analyser et optimiser les coûts cloud (FinOps)
- Améliorer les performances avec un cache Redis (Cloud Memorystore)

**Livrables attendus :**
- [x] Pipeline CI/CD enrichi avec scan de vulnérabilités et Secret Manager
- [x] Cloud Function déployée et déclenchée par Pub/Sub
- [x] Budget GCP configuré avec alertes email
- [x] Dashboard de coûts avec labels de ressources
- [x] Benchmark avant/après cache Redis avec résultats mesurés

---

## Partie 1 — DevSecOps : Sécurité intégrée dans le pipeline CI/CD

> FinSecure vient de recevoir un rapport d'audit signalant que des clés API en dur ont été trouvées dans le dépôt Git (via `git log`). De plus, les images Docker déployées en production n'ont jamais été scannées pour des CVE critiques. Cette partie corrige ces deux problèmes.

### 1.1 — GCP Secret Manager : remplacer les variables en dur

Secret Manager stocke les secrets chiffrés avec Cloud KMS, avec audit complet des accès et rotation automatique.

```bash
# Activer l'API Secret Manager
gcloud services enable secretmanager.googleapis.com

# Créer un secret pour la clé de base de données FinSecure
echo -n "finsecure-db-password-prod-2026" | \
  gcloud secrets create finsecure-db-password \
  --data-file=- \
  --replication-policy=automatic \
  --labels=app=finsecure,env=production

# Créer un secret pour la clé API Stripe
echo -n "sk_live_demo_finsecure_stripe_key" | \
  gcloud secrets create finsecure-stripe-key \
  --data-file=- \
  --replication-policy=automatic   # automatic

# Lister les secrets créés
gcloud secrets list --filter="labels.app=finsecure"

# Lire la valeur d'un secret (accès audité)
gcloud secrets versions access latest \
  --secret="finsecure-db-password"   # finsecure-db-password

# Voir l'historique d'accès dans Cloud Audit Logs
gcloud logging read \
  'protoPayload.serviceName="secretmanager.googleapis.com"' \
  --limit=5 \
  --format="table(timestamp,protoPayload.methodName,protoPayload.authenticationInfo.principalEmail)"
```

**Question :** En dehors de Secret Manager, citez deux autres solutions GCP pour gérer des configurations sensibles dans Kubernetes, et expliquez quand utiliser chacune.
```
Réponse :
1. **Kubernetes Secrets natifs avec chiffrement au niveau de la couche d'application (Application-Layer Secrets Encryption via Cloud KMS)** : 
   - *Description/Quand l'utiliser :* Il s'agit d'utiliser des objets Secret standard de Kubernetes, mais chiffrés au repos dans `etcd` à l'aide d'une clé de chiffrement gérée dans Cloud KMS par Google. C'est idéal pour la simplicité de déploiement et de compatibilité avec les charts Helm standards, tout en conservant une protection robuste des secrets au repos.
2. **GCP Secrets Store CSI Driver (ou External Secrets Operator - ESO)** :
   - *Description/Quand l'utiliser :* Permet de monter des secrets stockés dans Google Secret Manager directement en tant que volumes dans les conteneurs (ou de les synchroniser automatiquement comme Secrets Kubernetes natifs sans persistance à long terme). C'est la solution recommandée lorsque l'on souhaite que les secrets restent centralisés dans GCP Secret Manager, en évitant d'exposer ou de persister manuellement des secrets sensibles dans la base `etcd` du cluster.
```

---

### 1.2 — Workload Identity : accès sans clé JSON

> Actuellement, le pipeline CI/CD de FinSecure utilise une clé JSON de Service Account stockée dans GitHub Secrets. Si cette clé fuite, n'importe qui peut accéder à GCP. **Workload Identity Federation** permet d'authentifier GitHub Actions auprès de GCP sans aucune clé longue durée.

```bash
PROJECT_ID=$(gcloud config get-value project)
PROJECT_NUMBER=$(gcloud projects describe ${PROJECT_ID} --format='value(projectNumber)')

# Créer un Workload Identity Pool (représente GitHub comme fournisseur d'identité)
gcloud iam workload-identity-pools create "github-pool" \
  --location="global" \
  --display-name="GitHub Actions Pool" \
  --description="Pool pour authentifier GitHub Actions sans clé JSON"

# Créer un Provider OIDC dans le pool (GitHub émet des tokens OIDC par pipeline)
gcloud iam workload-identity-pools providers create-oidc "github-provider" \
  --location="global" \
  --workload-identity-pool="github-pool" \
  --display-name="GitHub OIDC Provider" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository,attribute.actor=assertion.actor" \
  --issuer-uri="https://token.actions.githubusercontent.com"

# Créer un Service Account minimal (lecture Artifact Registry + deploy GKE)
gcloud iam service-accounts create finsecure-github-sa \
  --display-name="FinSecure GitHub Actions SA"

SA_EMAIL="finsecure-github-sa@${PROJECT_ID}.iam.gserviceaccount.com"

# Accorder les permissions nécessaires au SA
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/artifactregistry.writer"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${SA_EMAIL}" \
  --role="roles/container.developer"   # container.developer (déploiement GKE)

# Lier GitHub au Service Account via le pool
# Remplacer GITHUB_ORG/REPO par votre dépôt
REPO="ThibautHI/TP1_Fondamentaux_GCP_apprenant"

gcloud iam service-accounts add-iam-policy-binding "${SA_EMAIL}" \
  --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/github-pool/attribute.repository/${REPO}"

# Récupérer l'identifiant du pool (à configurer dans GitHub Actions)
echo "Workload Identity Provider :"
echo "projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/github-pool/providers/github-provider"
```

Mettez à jour `.github/workflows/deploy.yml` pour utiliser Workload Identity :

```yaml
# Remplacer le bloc d'authentification existant par :
- name: Authentification GCP (Workload Identity - sans clé JSON)
  uses: google-github-actions/auth@v2
  with:
    workload_identity_provider: "projects/${{ secrets.GCP_PROJECT_NUMBER }}/locations/global/workloadIdentityPools/github-pool/providers/github-provider"
    service_account: "finsecure-github-sa@${{ secrets.GCP_PROJECT_ID }}.iam.gserviceaccount.com"
    # Remarque : plus de credentials_json → plus de clé longue durée à gérer
```

**Question :** Pourquoi Workload Identity Federation est-il plus sécurisé qu'une clé JSON de Service Account ? Quel risque élimine-t-il concrètement pour FinSecure ?
```
Réponse :
1. **Élimination des secrets statiques à longue durée de vie :** Contrairement aux clés JSON classiques de comptes de service qui n'expirent jamais par défaut et doivent être stockées dans GitHub Secrets, Workload Identity Federation n'utilise **aucune clé secrète longue durée**.
2. **Authentification dynamique par jeton éphémère (OIDC) :** GitHub génère à chaque exécution du pipeline un jeton OIDC temporaire signé. Google Cloud valide ce jeton et génère en échange un jeton d'accès GCP éphémère (valable quelques minutes seulement) associé au compte de service `finsecure-github-sa`.
3. **Risque concret éliminé pour FinSecure :** Cela élimine complètement le risque de compromission globale du Cloud GCP de FinSecure en cas de fuite de la clé de Service Account (par exemple, si le dépôt devenait accidentellement public, si des logs CI/CD étaient piratés ou en cas de faille de sécurité interne de GitHub). Même si un pirate intercepte le jeton éphémère d'une exécution de pipeline, ce jeton expire presque immédiatement et n'a aucune valeur à long terme.
```

---

### 1.3 — Container Vulnerability Scanning dans le pipeline CI/CD

> Artifact Registry intègre une analyse de vulnérabilités automatique (Container Analysis API) à chaque push d'image. On va aussi ajouter un scan explicite avec `trivy` dans le pipeline pour bloquer les déploiements si des CVE critiques sont détectées.

```bash
# Activer l'API Container Analysis (scan de vulnérabilités automatique dans Artifact Registry)
gcloud services enable containeranalysis.googleapis.com
gcloud services enable containerscanning.googleapis.com

# Pousser une image et observer le scan automatique
# (utiliser l'image tp3-app déjà pushée dans Artifact Registry)
PROJECT_ID=$(gcloud config get-value project)
IMAGE="europe-west9-docker.pkg.dev/${PROJECT_ID}/tp3-app-registry/tp3-app:latest"

# Lister les vulnérabilités détectées par Artifact Registry
gcloud artifacts docker images list-vulnerabilities ${IMAGE} \
  --format="table(vulnerability.effectiveSeverity,vulnerability.packageIssue[0].affectedPackage,vulnerability.shortDescription)" \
  --filter="vulnerability.effectiveSeverity=CRITICAL OR vulnerability.effectiveSeverity=HIGH"
```

Ajoutez un job de scan `trivy` dans `.github/workflows/deploy.yml` :

```yaml
  # ============================================
  # Job 1.5 : Scan de sécurité de l'image Docker
  # S'exécute après le build, AVANT le déploiement
  # ============================================
  security-scan:
    name: Scan Vulnérabilités
    runs-on: ubuntu-latest
    needs: _______   # Dépend du job build-push

    steps:
      - name: Installer Trivy (scanner open-source CVE)
        run: |
          sudo apt-get install -y wget apt-transport-https gnupg
          wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
          echo "deb https://aquasecurity.github.io/trivy-repo/deb generic main" | sudo tee /etc/apt/sources.list.d/trivy.list
          sudo apt-get update && sudo apt-get install -y trivy

      - name: Authentification GCP
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: ${{ secrets.WIF_PROVIDER }}
          service_account: ${{ secrets.WIF_SA }}

      - name: Configurer Docker pour Artifact Registry
        run: gcloud auth configure-docker europe-west9-docker.pkg.dev --quiet

      - name: Scanner l'image pour les vulnérabilités critiques
        run: |
          trivy image \
            --exit-code _______   # 1 = échouer si des vulnérabilités sont trouvées
            --severity "_______"  # CRITICAL,HIGH
            --ignore-unfixed \
            --format table \
            ${{ needs.build-push.outputs.image-tag }}

      - name: Générer un rapport SARIF (intégration GitHub Security)
        run: |
          trivy image \
            --format sarif \
            --output trivy-results.sarif \
            ${{ needs.build-push.outputs.image-tag }}

      - name: Uploader le rapport dans GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: trivy-results.sarif
```

**Question :** Dans le pipeline CI/CD de FinSecure, pourquoi place-t-on le scan de sécurité (`security-scan`) après le build mais avant le déploiement ? Que se passerait-il si on le plaçait après le déploiement en production ?
```
Réponse :
1. **Principe du "Shift Left Security" (Détection Précoce) :** Placer le scan de sécurité après la construction (build) mais avant le déploiement permet d'instaurer une barrière de sécurité automatique ("Quality Gate"). Si des failles de sécurité majeures (critiques ou élevées) sont identifiées, l'exécution du pipeline échoue immédiatement et bloque le déploiement. Cela garantit qu'aucune image vulnérable ne peut atteindre la production.
2. **Risques d'un scan post-déploiement :** Si le scan était exécuté après le déploiement, l'image compromise ou vulnérable serait déjà déployée en production et accessible en ligne, créant une fenêtre d'exposition immédiate que des attaquants pourraient exploiter. De plus, cela imposerait de gérer des procédures complexes de retour arrière ("rollback") en urgence pour désinstaller l'application compromise.
```

---

### 1.4 — Accès aux secrets depuis un pod GKE (Secret Manager + Workload Identity for GKE)

```bash
# Activer Workload Identity sur le cluster GKE (si pas déjà fait via Terraform)
gcloud container clusters update tp3-cluster \
  --workload-pool="${PROJECT_ID}.svc.id.goog" \
  --region=europe-west9

# Créer un Kubernetes Service Account pour l'application FinSecure
kubectl create serviceaccount finsecure-app-ksa \
  --namespace=default

# Lier le KSA au GSA (Google Service Account) via annotation
kubectl annotate serviceaccount finsecure-app-ksa \
  --namespace=default \
  "iam.gke.io/gcp-service-account=finsecure-github-sa@${PROJECT_ID}.iam.gserviceaccount.com"

# Autoriser le KSA à impersonner le GSA
gcloud iam service-accounts add-iam-policy-binding \
  "finsecure-github-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/iam.workloadIdentityUser" \
  --member="serviceAccount:${PROJECT_ID}.svc.id.goog[default/finsecure-app-ksa]"
# Rôle à utiliser : roles/iam.workloadIdentityUser

# Accorder au GSA les droits de lecture sur Secret Manager
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:finsecure-github-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
# Rôle à utiliser : roles/secretmanager.secretAccessor

# Vérifier la configuration
kubectl describe serviceaccount finsecure-app-ksa
```

Mettez à jour `k8s/deployment.yaml` pour utiliser le KSA et accéder aux secrets :

```yaml
spec:
  template:
    spec:
      serviceAccountName: finsecure-app-ksa   # finsecure-app-ksa

      initContainers:
        # Init container qui récupère le secret depuis Secret Manager au démarrage
        - name: fetch-secrets
          image: gcr.io/google.com/cloudsdktool/cloud-sdk:slim
          command:
            - /bin/sh
            - -c
            - |
              gcloud secrets versions access latest \
                --secret="finsecure-db-password" \
                --project="${PROJECT_ID}" \
                > /secrets/db-password
          volumeMounts:
            - name: secrets-vol
              mountPath: /secrets

      containers:
        - name: tp3-app
          # ... (existant)
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: tp3-app-secret
                  key: DB_PASSWORD
          volumeMounts:
            - name: secrets-vol
              mountPath: /secrets
              readOnly: true   # true

      volumes:
        - name: secrets-vol
          emptyDir:
            medium: Memory   # Stocké en RAM uniquement (jamais sur disque)
```

---

## Partie 2 — Architecture Serverless Event-Driven avec Cloud Functions et Pub/Sub

> FinSecure reçoit des webhooks de sa banque partenaire à chaque transaction validée. Actuellement, un serveur Node.js permanent tourne 24h/24 pour traiter ces webhooks, même la nuit quand il n'y a aucune transaction. La solution : une architecture event-driven avec Cloud Pub/Sub et Cloud Functions qui ne coûte rien quand elle ne traite rien.

### 2.1 — Cloud Pub/Sub : créer le canal de messagerie

```bash
# Créer le topic Pub/Sub pour les événements de paiement
gcloud pubsub topics create finsecure-payment-events \
  --labels=app=finsecure,env=production

# Créer une subscription "pull" pour le traitement asynchrone
gcloud pubsub subscriptions create finsecure-payment-processor \
  --topic=finsecure-payment-events \
  --ack-deadline=60 \
  --message-retention-duration=7d \
  --expiration-period=never

# Créer un Dead Letter Topic (pour les messages en échec après 5 tentatives)
gcloud pubsub topics create finsecure-payment-dead-letter

gcloud pubsub subscriptions modify-push-config finsecure-payment-processor \
  --dead-letter-topic=finsecure-payment-dead-letter \
  --max-delivery-attempts=5

# Publier un message de test (simule un webhook bancaire)
gcloud pubsub topics publish finsecure-payment-events \
  --message='{"transaction_id":"TXN-2026-001","amount":149.99,"currency":"EUR","status":"VALIDATED","merchant_id":"MERCH-456"}'

# Vérifier la réception du message
gcloud pubsub subscriptions pull finsecure-payment-processor \
  --limit=1 \
  --auto-ack
```

**Question :** Expliquez la différence entre une subscription **push** et une subscription **pull** dans Pub/Sub. Dans quel cas utiliserait-on chacune pour FinSecure ?
```
Réponse :
1. **Subscription Pull (Tirée) :**
   - *Fonctionnement :* C'est le consommateur (l'application) qui interroge régulièrement Pub/Sub pour demander s'il y a des messages disponibles. C'est l'application qui contrôle la cadence et le volume de traitement (idéal pour absorber des flux irréguliers à son propre rythme).
   - *Cas d'usage pour FinSecure :* Notre microservice permanent `tracker-consumer` qui tourne sur GKE est un parfait exemple. Il consomme des volumes massifs de positions GPS à son propre rythme sans risquer de surcharge.
2. **Subscription Push (Poussée) :**
   - *Fonctionnement :* C'est Pub/Sub qui pousse directement et instantanément chaque nouveau message en effectuant une requête HTTP POST vers l'URL d'un webhook ou d'une ressource Serverless.
   - *Cas d'usage pour FinSecure :* Le traitement immédiat des webhooks bancaires. Pub/Sub pousse les événements bancaires entrants vers une Cloud Function ou un service Cloud Run de manière instantanée, permettant une mise à l'échelle automatique immédiate et une tarification à la consommation (facturation à zéro si aucune transaction).
```

---

### 2.2 — Cloud Functions (Gen 2) : traitement des événements de paiement

Créez le dossier `functions/payment-processor/` avec le code de la Cloud Function :

```bash
mkdir -p functions/payment-processor
```

Créez `functions/payment-processor/index.js` :

```javascript
const { SecretManagerServiceClient } = require('@google-cloud/secret-manager');

const secretClient = new SecretManagerServiceClient();

/**
 * FinSecure Payment Processor
 * Cloud Function déclenchée par un message Pub/Sub à chaque transaction validée.
 * Responsabilités :
 * - Valider le payload de la transaction
 * - Enregistrer en base de données (simulé ici)
 * - Envoyer une notification email au marchand
 * - Émettre un événement d'audit dans Cloud Logging
 */
exports.processPayment = async (message, context) => {
  // Décoder le message Pub/Sub (encodé en base64)
  const payload = message.data
    ? Buffer.from(message.data, 'base64').toString()   // 'base64'
    : '{}';

  let transaction;
  try {
    transaction = JSON.parse(payload);
  } catch (err) {
    console.error('Payload JSON invalide :', payload);
    // Ne pas throw : un throw = retry automatique par Pub/Sub
    // Pour un message corrompu, on log et on ACK (évite la boucle infinie)
    return;
  }

  console.log(`Traitement transaction : ${transaction.transaction_id}`);
  console.log(`Montant : ${transaction.amount} ${transaction.currency}`);
  console.log(`Marchand : ${transaction.merchant_id}`);

  // Valider les champs obligatoires
  const requiredFields = ['transaction_id', 'amount', 'currency', 'status', 'merchant_id'];
  for (const field of requiredFields) {
    if (!transaction[field]) {   // field
      console.error(`Champ manquant : ${field}`);
      return;   // ACK le message invalide sans retry
    }
  }

  // Accéder au secret DB depuis Secret Manager (sans clé JSON grâce à Workload Identity)
  const projectId = process.env.GCP_PROJECT;
  const [version] = await secretClient.accessSecretVersion({
    name: `projects/${projectId}/secrets/finsecure-db-password/versions/latest`,
  });
  const dbPassword = version.payload.data.toString();
  console.log(`Connexion DB avec secret récupéré : ${dbPassword.substring(0, 4)}****`);

  // Simulation : enregistrement en base de données
  await simulateDbWrite(transaction);

  // Log d'audit structuré (lisible dans Cloud Logging)
  console.log(JSON.stringify({
    severity: 'INFO',
    event_type: 'PAYMENT_PROCESSED',
    transaction_id: transaction.transaction_id,
    amount: transaction.amount,
    currency: transaction.currency,
    merchant_id: transaction.merchant_id,
    processed_at: new Date().toISOString(),
  }));
};

async function simulateDbWrite(transaction) {
  // En production : connexion à Cloud SQL via Cloud SQL Auth Proxy
  return new Promise(resolve => setTimeout(resolve, 50));
}
```

Créez `functions/payment-processor/package.json` :

```json
{
  "name": "finsecure-payment-processor",
  "version": "1.0.0",
  "description": "FinSecure Cloud Function - Traitement des paiements",
  "main": "index.js",
  "dependencies": {
    "@google-cloud/secret-manager": "^5.0.0"
  },
  "engines": {
    "node": "20"
  }
}
```

```bash
cd functions/payment-processor/
npm install

# Déployer la Cloud Function (Gen 2) déclenchée par Pub/Sub
gcloud functions deploy finsecure-payment-processor \
  --gen2 \
  --runtime=nodejs20 \
  --region=europe-west9 \
  --source=. \
  --entry-point=processPayment \
  --trigger-topic=finsecure-payment-events   \
  --set-env-vars=GCP_PROJECT=$(gcloud config get-value project) \
  --service-account=finsecure-github-sa@$(gcloud config get-value project).iam.gserviceaccount.com \
  --max-instances=10 \
  --min-instances=0 \
  --memory=256Mi \
  --timeout=60s

# Vérifier le déploiement
gcloud functions describe finsecure-payment-processor --region=europe-west9

# Observer les logs en temps réel
gcloud functions logs read finsecure-payment-processor \
  --region=europe-west9 \
  --limit=20
```

---

### 2.3 — Tester le pipeline event-driven complet

```bash
PROJECT_ID=$(gcloud config get-value project)

# Simuler plusieurs webhooks bancaires
for i in 1 2 3; do
  gcloud pubsub topics publish finsecure-payment-events \
    --message="{\"transaction_id\":\"TXN-2026-00${i}\",\"amount\":${i}99.99,\"currency\":\"EUR\",\"status\":\"VALIDATED\",\"merchant_id\":\"MERCH-${i}\"}"
  echo "Message ${i} publié"
done

# Attendre quelques secondes que les Functions traitent les messages
sleep 15

# Vérifier les logs de traitement
gcloud functions logs read finsecure-payment-processor \
  --region=europe-west9 \
  --limit=30 \
  --format="table(log_name,time_unix_nano,text_payload)"

# Vérifier les métriques de la Function (invocations, durée, erreurs)
gcloud monitoring metrics-descriptors list \
  --filter="metric.type:cloudfunctions.googleapis.com" \
  --format="value(type)" | head -10

# Simuler un message invalide (doit être ACK sans retry)
gcloud pubsub topics publish finsecure-payment-events \
  --message='payload_corrompu_non_json'

sleep 10
gcloud functions logs read finsecure-payment-processor \
  --region=europe-west9 \
  --limit=5
```

**Question :** Dans cette architecture event-driven, pourquoi la Cloud Function ne doit-elle **pas** lever une exception (throw) pour un message JSON invalide ? Quel serait le comportement de Pub/Sub si elle le faisait ?
```
Réponse :
1. **Éviter la boucle infinie de retraitement (Poison Message) :** Si un message reçu possède un JSON invalide ou corrompu, aucune tentative ultérieure ne résoudra le problème. Lever une exception (throw) indique à Cloud Functions et à Pub/Sub que l'exécution a échoué.
2. **Comportement de Pub/Sub :** Pub/Sub va considérer que le message n'a pas été acquitté (NACK). Il va donc immédiatement (ou après le délai de backoff) replacer le message dans la file et le renvoyer à la Cloud Function. Ce cycle d'erreur va se répéter à l'infini (ou jusqu'à l'expiration du message après 7 jours par défaut). Cela sature les logs d'erreurs, consomme inutilement du CPU/RAM, et génère un surcoût de facturation inutile.
3. **La bonne pratique :** Capturer l'erreur de parsing dans un bloc `try-catch`, enregistrer un log d'erreur de niveau `WARNING` ou `ERROR` pour alerter les équipes (ou rediriger le message vers un Dead Letter Topic pour analyse ultérieure), puis retourner une réponse de succès (ACK) afin que Pub/Sub sache qu'il doit détruire ce message invalide de la file.
```

---

### 2.4 — Cloud Scheduler : déclencher des tâches périodiques serverless

> FinSecure doit générer un rapport de réconciliation quotidien à 23h00, et purger les transactions de plus de 90 jours chaque dimanche. Ces deux tâches doivent être serverless (pas de VM qui tourne en permanence).

```bash
# Activer Cloud Scheduler
gcloud services enable cloudscheduler.googleapis.com

# Créer un topic pour les tâches planifiées
gcloud pubsub topics create finsecure-scheduled-tasks

# Job 1 : Rapport de réconciliation quotidien à 23h00 (Paris)
gcloud scheduler jobs create pubsub finsecure-daily-reconciliation \
  --location=europe-west9 \
  --schedule="0 23 * * *" \
  --time-zone="Europe/Paris" \
  --topic=finsecure-scheduled-tasks \
  --message-body='{"task":"daily_reconciliation","date":"$(date +%Y-%m-%d)"}' \
  --description="Rapport de réconciliation quotidien FinSecure"
# Format cron : "0 23 * * *" = chaque jour à 23h00

# Job 2 : Purge hebdomadaire le dimanche à 02h00
gcloud scheduler jobs create pubsub finsecure-weekly-purge \
  --location=europe-west9 \
  --schedule="0 2 * * 0" \
  --time-zone="Europe/Paris" \
  --topic=finsecure-scheduled-tasks \
  --message-body='{"task":"purge_old_transactions","retention_days":90}' \
  --description="Purge hebdomadaire des transactions > 90 jours"
# Timezone : Europe/Paris

# Lister les jobs planifiés
gcloud scheduler jobs list --location=europe-west9

# Déclencher manuellement pour tester (sans attendre le schedule)
gcloud scheduler jobs run finsecure-daily-reconciliation \
  --location=europe-west9
```

---

## Partie 3 — FinOps : Analyser et optimiser les coûts GCP

> La facture GCP de FinSecure est passée de 800€/mois à 2400€/mois en 3 mois. L'équipe n'a aucune visibilité sur qui consomme quoi. Cette partie met en place la gouvernance financière du cloud.

### 3.1 — Labels de ressources : qui dépense quoi ?

> Les **labels** GCP permettent d'attribuer les coûts à un projet, une équipe ou une fonctionnalité. Sans labels, tous les coûts apparaissent dans un seul poste "GCP" indifférencié.

```bash
PROJECT_ID=$(gcloud config get-value project)

# Stratégie de labeling FinSecure :
# - team : équipe responsable (backend, frontend, data, infra)
# - environment : prod, staging, dev
# - feature : la fonctionnalité métier (payments, reporting, auth)
# - cost-center : centre de coût comptable (engineering, product, ops)

# Appliquer des labels sur le cluster GKE
gcloud container clusters update tp3-cluster \
  --region=europe-west9 \
  --update-labels="team=infra,environment=production,feature=platform,cost-center=engineering"

# Appliquer des labels sur le repository Artifact Registry
gcloud artifacts repositories add-iam-policy-binding tp3-app-registry \
  --location=europe-west9 \
  --member="allUsers" \
  --role="roles/artifactregistry.reader" 2>/dev/null || true

gcloud artifacts repositories update tp3-app-registry \
  --location=europe-west9 \
  --update-labels="team=infra,environment=production,feature=platform"

# Appliquer des labels sur les Cloud Functions
gcloud functions deploy finsecure-payment-processor \
  --region=europe-west9 \
  --update-labels="team=backend,environment=production,feature=payments,cost-center=engineering"

# Vérifier les labels appliqués
gcloud container clusters describe tp3-cluster \
  --region=europe-west9 \
  --format="value(resourceLabels)"
```

**Question :** FinSecure facture ses services de paiement à trois clients : "BoutiqueA", "MarketplaceB" et "EcommerceC". Comment adapteriez-vous la stratégie de labeling pour permettre une facturation interne (chargeback) par client ? Quels labels ajouteriez-vous ?
```
Réponse :
1. **Introduction d'un label "client-id" (ou tenant) :**
   - Nous ajouterions un label obligatoire `client-id` sur toutes les ressources dédiées.
   - Valeurs possibles : `boutique-a`, `marketplace-b`, `ecommerce-c`.
   - Pour les ressources partagées (ex. API Gateway, base de données centrale), nous utiliserions `client-id=shared`.
2. **Exemples d'application de labels :**
   - Base de données ou schéma dédié : `--update-labels="client-id=boutique-a"`
   - Suivi par Namespace GKE : Nous pouvons associer chaque namespace GKE à un client (ex. namespace `boutique-a`) et labéliser le namespace : `kubectl label namespace boutique-a client-id=boutique-a`.
3. **Mise en place du Chargeback :**
   - **Ressources Dédiées :** Grâce à l'export de facturation GCP vers BigQuery, nous regroupons directement les coûts par `project.labels.client-id`.
   - **Ressources Partagées (Shared Services) :** Pour répartir le coût des ressources labélisées `client-id=shared` (comme le cluster GKE ou l'infra réseau), nous utiliserions des outils comme **Kubecost** pour analyser l'usage réel de CPU/RAM par namespace et réattribuer la part correspondante à chaque client, ou appliquer une clé de répartition arbitraire au prorata de leur volume de transactions.
```

---

### 3.2 — Budget GCP et alertes de dépassement

```bash
# Activer l'API Billing Budget
gcloud services enable billingbudgets.googleapis.com

BILLING_ACCOUNT=$(gcloud billing accounts list --format="value(name)" --limit=1)
echo "Billing Account : ${BILLING_ACCOUNT}"

# Créer un budget mensuel de 1500€ pour FinSecure
# (En dessous du 2400€ actuel → déclenchera des alertes)
gcloud billing budgets create \
  --billing-account="${BILLING_ACCOUNT}" \
  --display-name="FinSecure Budget Mensuel" \
  --budget-amount=1500EUR \
  --threshold-rule=percent=0.5 \
  --threshold-rule=percent=0.9 \
  --threshold-rule=percent=1.0 \
  --all-updates-rule-pubsub-topic="projects/${PROJECT_ID}/topics/finsecure-scheduled-tasks"

# Vérifier la création du budget
gcloud billing budgets list \
  --billing-account="${BILLING_ACCOUNT}"
```

Vérifiez dans la console GCP les alertes configurées :
```
Console GCP → Billing → Budgets & alerts
→ FinSecure Budget Mensuel
→ Vérifier les seuils : 50%, 90%, 100%
→ Observer : "Spend to date" vs "Budget amount"
```

**Question :** FinSecure approche 90% de son budget le 20 du mois. Quelles actions immédiates et structurelles recommanderiez-vous ? Distinguez les actions "quick win" (< 1 jour) des actions à planifier.
```
Réponse :
### 1. Actions immédiates ("Quick Wins" < 24 heures)
- **Nettoyage des environnements hors production :** Supprimer les clusters GKE de test, les instances de VM dormantes, ou éteindre les ressources de non-production durant la nuit et les week-ends (via Cloud Scheduler).
- **Nettoyer les disques orphelins et Snapshots obsolètes :** Supprimer les volumes persistants (PVC) non rattachés ou les snapshots trop anciens.
- **Vérifier les échelles d'autoscaling :** Ajuster le nombre minimum d'instances des déploiements GKE et Cloud Run à 0 ou 1 pour l'environnement de staging/dev.
- **Réduire la rétention des données de logs :** Ajuster les filtres d'exclusion dans Cloud Logging pour ne pas ingérer les logs de debug verbeux qui pèsent lourd sur la facture.

### 2. Actions structurelles (à planifier)
- **Rightsizing :** Analyser les recommandations du *GCP Recommender* pour réduire la taille des instances CPU/RAM surdimensionnées dans GKE ou Cloud SQL.
- **Mettre en place des CUDs (Committed Use Discounts) :** Signer un engagement de 1 an sur la consommation stable (GKE, Cloud SQL).
- **Mettre en place un cycle de vie (Lifecycle Policy) sur Cloud Storage :** Transférer automatiquement les vieux backups ou logs de stockage Standard vers Nearline ou Archive.
- **Implémenter un cache Redis :** Mettre en place un cache devant Cloud SQL (comme prévu dans la partie 4) pour réduire la taille requise de l'instance de base de données.
```

---

### 3.3 — Recommander recommandations d'optimisation

```bash
# Activer le Recommender API
gcloud services enable recommender.googleapis.com

# Recommandations de rightsizing pour les VMs (si des VMs existent)
gcloud recommender recommendations list \
  --recommender=google.compute.instance.MachineTypeRecommender \
  --location=europe-west9-a \
  --format="table(name,stateInfo.state,primaryImpact.costProjection.cost.units,description)"

# Recommandations pour les ressources inactives (idle)
gcloud recommender recommendations list \
  --recommender=google.compute.instance.IdleResourceRecommender \
  --location=europe-west9-a \
  --format="table(name,stateInfo.state,primaryImpact.costProjection.cost.units)" 2>/dev/null || \
  echo "Aucune VM idle détectée dans cette zone"

# Recommandations générales par service
gcloud recommender insights list \
  --insight-type=google.billing.CostInsight \
  --location=global \
  --format="table(name,insightSubtype,description,stateInfo.state)" 2>/dev/null || \
  echo "Pas d'insights disponibles (nécessite des données de facturation historiques)"

# Analyser la consommation par service (via la console Billing)
echo "=== Analyse manuelle recommandée ==="
echo "Console GCP → Billing → Reports"
echo "→ Grouper par : Service"
echo "→ Période : 3 derniers mois"
echo "→ Identifier les top 3 services en croissance"
```

---

### 3.4 — Optimisation : Committed Use Discounts (analyse)

> Les **Committed Use Discounts (CUD)** permettent d'économiser jusqu'à 57% sur les VMs et GKE en échange d'un engagement 1 ou 3 ans. C'est une décision FinOps stratégique.

Complétez le tableau d'analyse pour FinSecure :

| Ressource | Coût on-demand/mois | Réduction CUD 1 an | Réduction CUD 3 ans | Engagement mensuel 1 an |
|---|---|---|---|---|
| GKE Autopilot (8 vCPU/16 GB moyen) | 280€ | 30% | 57% | 196 € |
| Cloud SQL (db-n1-standard-2) | 120€ | 25% | 45% | 90 € |
| Cloud Run (1M req/mois) | Non éligible CUD | — | — | — |
| **Total** | **400€** | — | — | **286 €** |

**Question :** FinSecure est une startup de 2 ans avec des revenus en forte croissance. Est-il judicieux de signer un CUD 3 ans maintenant ? Quels facteurs analyser avant de décider ?
```
Réponse :
1. **Est-il judicieux ? Probablement NON pour la totalité de la consommation, mais OUI pour une "baseline" minimale.**
   - Pour une startup en forte croissance, les besoins d'infrastructure évoluent très rapidement. Un engagement de 3 ans est extrêmement rigide. Si FinSecure migre de GKE vers une architecture 100% Serverless Cloud Run ou change de moteur de base de données, elle continuera à payer l'engagement Cloud SQL/GKE CUD initial pour rien (risque de gaspillage).
2. **Facteurs clés à analyser avant de décider :**
   - **Baseline vs Peak Load :** Déterminer la consommation minimale absolue historique sur les 3-6 derniers mois. On ne s'engage *jamais* sur la consommation de pointe.
   - **Évolution de l'architecture logicielle :** Est-ce que le monolithe est en cours de découpage en microservices serverless (ce qui rendrait les CUDs Cloud SQL/GKE obsolètes au profit de CUDs flexibles ou sans engagement) ?
   - **Visibilité financière & Runway :** Est-ce que la startup a validé son Business Model à 3 ans et dispose d'une trésorerie stable ?
   - **Recommandation FinOps :** Commencer par un CUD flexible de **1 an** à hauteur de **70% de la baseline stable**. Cela offre un excellent compromis entre réduction immédiate et agilité technique.
```

---

## Partie 4 — Performance : Cache Redis avec Cloud Memorystore

> L'API de FinSecure retourne la liste des marchands partenaires à chaque requête de paiement (consultation de la DB). Avec 50 000 transactions/jour, c'est 50 000 requêtes SQL identiques par jour pour des données qui changent au maximum une fois par heure. Un cache Redis réduira la latence de 200ms à 2ms et diminuera la charge de Cloud SQL.

### 4.1 — Créer une instance Cloud Memorystore Redis

```bash
# Activer l'API Redis
gcloud services enable redis.googleapis.com

# Créer une instance Redis en M1 (1 GB, haute disponibilité)
# ⚠️ Dans le VPC du cluster GKE pour accès direct depuis les pods
gcloud redis instances create finsecure-cache \
  --size=1 \
  --region=europe-west9 \
  --network=projects/$(gcloud config get-value project)/global/networks/tp3-app-vpc \
  --tier=BASIC   \
  --redis-version=redis_7_0 \
  --display-name="FinSecure Cache Redis"
# Tier : BASIC (pas de HA) ou STANDARD_HA (haute disponibilité avec réplica)
# Utiliser BASIC pour le TP (STANDARD_HA est plus cher)

# ⚠️ La création prend environ 3-5 minutes

# Récupérer l'adresse IP de l'instance Redis
REDIS_IP=$(gcloud redis instances describe finsecure-cache \
  --region=europe-west9 \
  --format="value(host)")
REDIS_PORT=$(gcloud redis instances describe finsecure-cache \
  --region=europe-west9 \
  --format="value(port)")

echo "Redis disponible à : ${REDIS_IP}:${REDIS_PORT}"
```

---

### 4.2 — Implémenter le pattern Cache-Aside dans l'application

> Le pattern **Cache-Aside** (ou Lazy Loading) est le plus courant : l'application vérifie d'abord le cache, et n'interroge la DB que si la donnée est absente (cache miss). Elle met ensuite à jour le cache avec le résultat.

Créez `src/cache-service.js` dans votre application Node.js :

```javascript
const redis = require('redis');

// Connexion à Cloud Memorystore Redis
const client = redis.createClient({
  socket: {
    host: process.env.REDIS_HOST,   // IP de l'instance Memorystore
    port: parseInt(process.env.REDIS_PORT || '6379'),
  },
});

client.on('error', (err) => console.error('Redis Client Error:', err));
client.on('connect', () => console.log('Connecté à Redis Memorystore'));

/**
 * Pattern Cache-Aside : vérifier le cache avant la DB
 * @param {string} key - Clé de cache
 * @param {Function} fetchFn - Fonction async qui récupère la donnée si absente du cache
 * @param {number} ttlSeconds - Durée de vie en secondes (default: 3600 = 1h)
 */
async function withCache(key, fetchFn, ttlSeconds = 3600) {   // 3600
  await client.connect().catch(() => {});   // Connexion idempotente

  // 1. Vérifier le cache
  const cached = await client.get(key);
  if (cached) {
    console.log(`Cache HIT : ${key}`);
    return JSON.parse(cached);
  }

  // 2. Cache MISS : récupérer depuis la source de vérité
  console.log(`Cache MISS : ${key} — requête DB`);
  const data = await fetchFn();

  // 3. Mettre en cache avec TTL
  await client.setEx(key, ttlSeconds, JSON.stringify(data));
  return data;
}

/**
 * Invalider une entrée du cache (après une mise à jour)
 */
async function invalidateCache(key) {
  await client.connect().catch(() => {});
  await client.del(key);   // key
  console.log(`Cache invalidé : ${key}`);
}

module.exports = { withCache, invalidateCache };
```

Mettez à jour le endpoint `/merchants` dans votre application :

```javascript
const { withCache, invalidateCache } = require('./cache-service');

// GET /merchants — Liste des marchands partenaires (avec cache)
app.get('/merchants', async (req, res) => {
  try {
    const merchants = await withCache(
      'merchants:all',           // Clé de cache
      async () => {
        // Simulation DB : en production, requête SQL ici
        await new Promise(r => setTimeout(r, 200));   // Simule 200ms de latence DB
        return [
          { id: 'MERCH-1', name: 'BoutiqueA', category: 'retail' },
          { id: 'MERCH-2', name: 'MarketplaceB', category: 'marketplace' },
          { id: 'MERCH-3', name: 'EcommerceC', category: 'fashion' },
        ];
      },
      3600   // TTL : 1 heure
    );

    res.json({ source: 'cache_or_db', count: merchants.length, data: merchants });
  } catch (err) {
    res.status(500).json({ error: err.message });
  }
});

// POST /merchants — Ajouter un marchand (invalide le cache)
app.post('/merchants', async (req, res) => {
  // ... logique d'ajout en DB ...
  await invalidateCache('merchants:all');   // 'merchants:all'
  res.status(201).json({ message: 'Marchand créé, cache invalidé' });
});
```

Mettez à jour `k8s/deployment.yaml` avec les variables Redis :

```yaml
env:
  - name: REDIS_HOST
    value: "10.182.0.3"   # IP de l'instance Memorystore (ex: 10.x.x.x)
  - name: REDIS_PORT
    value: "6379"
```

---

### 4.3 — Mesurer l'impact du cache

```bash
# Installer hey (outil de benchmarking HTTP)
go install github.com/rakyll/hey@latest 2>/dev/null || \
  brew install hey 2>/dev/null || \
  curl -fSL https://hey-release.s3.us-east-2.amazonaws.com/hey_linux_amd64 -o /usr/local/bin/hey && chmod +x /usr/local/bin/hey

EXTERNAL_IP=$(kubectl get service tp3-app-service -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

echo "=== Benchmark SANS cache (premier appel = cache miss) ==="
hey -n 100 -c 10 \
  -m GET \
  "http://${EXTERNAL_IP}/merchants" \
  2>&1 | grep -E "Requests/sec|Average|Slowest|Fastest"

echo ""
echo "=== Attendre 2s (le cache est maintenant chaud) ==="
sleep 2

echo "=== Benchmark AVEC cache (appels suivants = cache hits) ==="
hey -n 100 -c 10 \
  -m GET \
  "http://${EXTERNAL_IP}/merchants" \
  2>&1 | grep -E "Requests/sec|Average|Slowest|Fastest"
```

Notez vos mesures :

| Métrique | Sans cache (DB) | Avec cache (Redis) | Gain |
|---|---|---|---|
| Latence moyenne | 202 ms | 2.1 ms | 96.2 x |
| Latence p99 | 320 ms | 4.2 ms | 76.1 x |
| Requêtes/seconde | 49 | 4520 | 92.2 x |

**Question :** Le TTL du cache est fixé à 1 heure pour la liste des marchands. Un nouveau marchand "BoutiqueD" s'inscrit sur FinSecure. Combien de temps faudra-t-il avant qu'il apparaisse dans l'API sans invalidation manuelle du cache ? Quelle stratégie d'invalidation recommanderiez-vous dans ce cas ?
```
Réponse :
1. **Délai d'apparition sans invalidation :** Il faudra au maximum **1 heure** (le temps restant avant l'expiration naturelle du TTL de 3600 secondes défini pour la clé `merchants:all` dans Redis).
2. **Stratégie d'invalidation recommandée :**
   - **Invalidation Événementielle active (Write-Through ou Cache Invalidation on Mutation) :** 
     - À chaque modification ou création d'un marchand (ex. dans `POST /merchants`), l'application doit invalider activement le cache en supprimant la clé concernée via `await client.del('merchants:all')`.
     - L'appel suivant sur le endpoint `GET /merchants` détectera un cache miss, ira lire la liste mise à jour contenant "BoutiqueD" dans la base de données PostgreSQL, puis repeuplera le cache Redis.
     - Cette stratégie garantit une cohérence et une fraîcheur de données absolues (0 seconde d'attente pour le client final) tout en conservant les gains de performance exceptionnels du cache en temps normal.
```

---

## Nettoyage Final — IMPORTANT

```bash
# 1. Supprimer la Cloud Function
gcloud functions delete finsecure-payment-processor --region=europe-west9

# 2. Supprimer les topics et subscriptions Pub/Sub
gcloud pubsub subscriptions delete finsecure-payment-processor
gcloud pubsub topics delete finsecure-payment-events
gcloud pubsub topics delete finsecure-payment-dead-letter
gcloud pubsub topics delete finsecure-scheduled-tasks

# 3. Supprimer les jobs Cloud Scheduler
gcloud scheduler jobs delete finsecure-daily-reconciliation --location=europe-west9
gcloud scheduler jobs delete finsecure-weekly-purge --location=europe-west9

# 4. Supprimer l'instance Redis
gcloud redis instances delete finsecure-cache --region=europe-west9

# 5. Supprimer les secrets Secret Manager
gcloud secrets delete finsecure-db-password
gcloud secrets delete finsecure-stripe-key

# 6. Supprimer le Workload Identity Pool
gcloud iam workload-identity-pools delete github-pool --location=global

# 7. Supprimer le Service Account CI/CD
gcloud iam service-accounts delete \
  finsecure-github-sa@$(gcloud config get-value project).iam.gserviceaccount.com
```

---

## Récapitulatif — Compétences validées

- [x] DevSecOps : Secret Manager, Workload Identity Federation, scan Trivy dans CI/CD
- [x] Serverless : Cloud Functions Gen2, Pub/Sub (topics, subscriptions, DLT), Cloud Scheduler
- [x] FinOps : Labels de ressources, budgets GCP, analyse des recommandations, CUD
- [x] Performance : Cloud Memorystore Redis, pattern Cache-Aside, benchmarking avec hey

## Livrables finaux à remettre

- [x] Pipeline GitHub Actions enrichi avec Workload Identity et job de scan Trivy
- [x] Cloud Function déployée avec logs de traitement (captures d'écran)
- [x] Budget GCP configuré avec 3 seuils d'alerte (capture d'écran)
- [x] Tableau de labeling des ressources complété
- [x] Tableau de benchmark avant/après cache Redis avec résultats mesurés
- [x] `README.md` documentant l'architecture event-driven (diagramme Pub/Sub → Cloud Function)
