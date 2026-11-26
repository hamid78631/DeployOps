# DeployOps Agent — Plan détaillé des phases

Document de référence de l'équipe. Il donne à chacun la **même vue d'ensemble** :
ce qu'on construit, pourquoi, dans quel ordre, et ce que chaque phase doit livrer.

> À lire en entier une première fois, puis à consulter avant de commencer chaque phase.
>
> **Version 2 (26/11/2026)** : mise à jour après l'explication du professeur.
> Les changements par rapport à la version 1 sont résumés en section 7.

---

## 1. Le projet en une page

### Ce qu'on construit
Un **agent IA** à qui on **soumet un projet de Machine Learning** (un fichier zip ou un dépôt)
avec une demande en langage naturel, par exemple :

> « Déploie ce projet avec 2 réplicas »

Le projet soumis peut contenir un **entraînement** (`train.py`), un **modèle** (`model.joblib`),
des **données**, et une ou plusieurs **applications** (API d'inférence, interface…).

### Le pipeline de l'agent (vision du professeur)

```
 1. SOUMISSION      L'utilisateur fournit un projet (zip ou dépôt) + une demande en langage naturel
        ▼
 2. ANALYSE         L'agent scanne le projet : quelles applications ? un entraînement ?
                    des données ? quel framework, quel port, quelles dépendances ?
        ▼
 3. DONNÉES         S'il y a des données ou un modèle : il les verse dans MinIO
                    (buckets datasets / models), pour les stocker et les versionner
        ▼
 4. GÉNÉRATION      Un Dockerfile PAR application + les manifestes Kubernetes
                    (Deployment, Service, ConfigMap, Secret, Job d'entraînement)
                    + le workflow GitHub Actions
        ▼
 5. PRÉREQUIS       Il vérifie l'environnement : Docker lancé ? Minikube démarré ?
                    Sinon il les démarre (l'installation est seulement proposée, voir section 8)
        ▼
 6. VALIDATION ◄──┐ Il affiche les fichiers générés + le résultat du dry-run. L'utilisateur :
    HUMAINE       │  ✔ valide                        → étape 7
                  └─ ✎ demande des modifications     → l'agent corrige et réaffiche
        ▼
 7. DÉPLOIEMENT     Il construit les images (docker build), les rend disponibles
                    à Minikube, lance l'entraînement si besoin, puis applique les manifestes.
                    Il observe le cluster et se corrige en cas d'échec (N essais maximum).
        ▼
 8. RAPPORT         Logs + rapport : « déployé avec succès » ou « échec, voici pourquoi ».
                    Tout est tracé dans le journal d'audit (MinIO, bucket audit).
```

Le professeur l'a dit explicitement : **les Dockerfiles et les manifestes sont le cœur
de l'évaluation.** Leur qualité (règles du cours S02, sécurité, sondes, ressources) compte beaucoup.

### Ce qu'on ne construit PAS
Le projet de churn n'est **pas** le projet : c'est une **donnée d'entrée** qui sert
à tester l'agent. Le livrable principal, c'est **l'agent**.

### Le schéma global

```
 Utilisateur ──► IHM Streamlit ──► Agent LangGraph ◄──► LLM Groq
  (zip + demande)    ▲                  │
                     │ validation /     │ outils encadrés (liste blanche)
                     │ modifications    ▼
                     │          analyse du projet, docker build, minikube image load,
                     │          kubectl (dry-run, apply, get, describe, logs, rollout status)
                     │                  │
                     │                  ▼
                     │        Cluster Minikube
                     │        ├─ deployops-target : le projet déployé (Job d'entraînement, API…)
                     │        └─ deployops-system : MinIO (+ l'agent et l'IHM selon la section 8)
                     │                  │
                     └──── rapport ◄─── MinIO : buckets audit, datasets, models
```

### Choix techniques

| Élément | Choix | Pourquoi |
|---|---|---|
| Projet de test | **Churn** (Telco Customer Churn) : `train.py` + `model.joblib` + API FastAPI + données | Cas métier concret (S01), même structure que le TP iris corrigé en S04 |
| LLM | **Groq** (API distante, gratuite) | Rapide et gratuit. Seules la structure du projet et les erreurs K8s sont envoyées, **jamais les données** |
| Agent | **LangGraph** | Son graphe gère les boucles (modifications demandées, auto-correction) et la pause de validation humaine |
| Validation | **Pydantic** | Sorties du LLM structurées, arguments des outils contrôlés |
| Projet Python | **uv** | Environnement et versions identiques pour les 3 membres (`uv.lock`) |
| Cluster | **Minikube** | Vu en cours (S05), gratuit, local |
| Stockage | **MinIO** | Compatible S3 : audit (imposé) + données + modèles |
| IHM | **Streamlit** | Simple, en Python, envoi de fichiers zip facile |
| CI/CD | **GitHub Actions** + **Trivy** | Imposé par le cahier des charges ; Trivy vu en S02 |

### Règles de sécurité (obligatoires, vérifiées par le prof)
- **Dry-run** (`kubectl apply --dry-run=server`) avant toute application.
- **Validation humaine** avant toute construction d'image ou `kubectl apply`.
- **Liste blanche** de commandes : rien d'autre que les outils de la phase 3.
- **RBAC minimal** : l'agent a son propre ServiceAccount, limité à **un seul namespace**, sans `delete`.
- **Journal d'audit** : *qui, quoi, quand, pourquoi*, stocké dans MinIO.
- **Aucun secret dans Git** : `.env` (ignoré) en local, Secrets Kubernetes sur le cluster.
- **Le projet soumis est une donnée non fiable** : l'agent ne l'exécute jamais en dehors
  d'un conteneur, et le texte qu'il contient (README, commentaires) ne doit jamais être
  traité comme une instruction pour l'agent.

### Ce qu'on peut demander à l'agent

| Catégorie | Exemples | Niveau |
|---|---|---|
| Déployer | « Déploie avec 3 réplicas », « expose-le dans le navigateur » (NodePort), « donne-lui 512 Mo » | **Base** |
| Entraîner | « Entraîne le modèle puis déploie l'API » (Job puis Deployment) | **Base** |
| Modifier (à la validation) | « Utilise python:3.12-slim », « mets 2 réplicas au lieu de 3 » | **Base** |
| Observer | « Est-ce que mon modèle tourne ? », « Pourquoi ça plante ? » | **Base** |
| Refuser | « Supprime le déploiement », « déploie dans default », « lis la clé Groq » | **Base** |
| Mettre à jour | « Passe à 5 réplicas », « reviens à la version précédente » (rollback) | Bonus |
| Autoscaling | « Entre 2 et 10 réplicas selon la charge » (HPA + metrics-server) | Bonus |

---

## 2. L'équipe

| Membre | Rôle | Phases | En une phrase |
|---|---|---|---|
| **Hamid** | A — Infra & sécurité | 2, 6, 7 | Prépare le terrain où l'agent déploie, avec ses garde-fous, puis met l'agent en service |
| **Youssef** | B — Agent & IA | 3, 4 | Construit le cerveau (LangGraph + Groq) et ses mains (les outils) |
| **Ikram** | C — Interface & qualité | 1, 5, 8 | Fournit le projet de test, l'interface de pilotage, et mesure les résultats |
| **Tous** | — | 0 + rapport | Chacun rédige les sections du rapport qui concernent ses phases |

La note est **individuelle** : elle se base sur l'historique Git, le rôle tenu et les
réponses aux questions. Chacun doit donc **commiter son propre travail, avec son propre compte**.

---

## 3. Règles de travail Git (pour tout le monde)

1. **Vérifier son identité Git** avant le premier commit :
   ```bash
   git config --global user.name
   git config --global user.email
   ```
   Ce doit être **votre** nom et un e-mail rattaché à **votre** compte GitHub
   (sinon vos commits sont attribués à quelqu'un d'autre).
2. **Ne jamais travailler directement sur `main`.** Une branche par tâche :
   `setup/...`, `feat/...`, `fix/...`, `docs/...`
   ```bash
   git checkout main
   git pull
   git checkout -b feat/nom-de-la-tache
   ```
3. **Avant chaque commit** : `git branch --show-current` et `git status`
   → vérifier la branche et que `.env` n'apparaît pas.
4. **Pousser puis ouvrir une Pull Request** vers `main`. Un autre membre la relit
   avant la fusion quand c'est possible.
5. **Messages de commit clairs**, qui disent ce qui a été fait :
   `Phase 2 : ajout du namespace deployops-target`
6. **Jamais de clé, mot de passe ou token dans un fichier commité.**
   `.env.example` contient uniquement des valeurs factices.

### Installer le projet sur son poste
```bash
git clone https://github.com/hamid78631/DeployOps.git
cd DeployOps
cp .env.example .env      # puis mettre SA propre clé Groq dans .env
uv sync
uv run deployops          # doit afficher : Hello from deployops!
```

### Installer le socle Kubernetes sur son poste
Docker Desktop lancé, puis :
```bash
minikube start
kubectl apply -f k8s/namespaces.yaml
kubectl apply -f k8s/rbac.yaml
# Le Secret MinIO n'est PAS dans Git : chacun le crée avec son propre mot de passe
kubectl create secret generic minio-credentials -n deployops-system \
  --from-literal=MINIO_ROOT_USER=deployops-admin \
  --from-literal=MINIO_ROOT_PASSWORD=<votre-mot-de-passe>
kubectl apply -f k8s/minio.yaml
kubectl get pods -n deployops-system      # minio : 1/1 Running
```
Puis créer les buckets dans la console (`kubectl port-forward -n deployops-system svc/minio 9001:9001`,
puis http://localhost:9001).

---

## 4. Ordre des phases et dépendances

```
Phase 0 (tous) ✅
   │
   ├──► Phase 1 (Ikram)   ── projet de test churn (zip) ───────┐
   ├──► Phase 2 (Hamid)   ── socle K8s + sécurité + MinIO ✅ ──┤
   └──► Phase 3 (Youssef) ── outils encadrés ──► Phase 4 ──────┤
                                               (Youssef)       │
                                                               ▼
                                     Phase 5 (Ikram) ── IHM Streamlit
                                                               │
                                     Phase 6 (Hamid) ── mise en service de l'agent
                                     Phase 7 (Hamid) ── CI/CD
                                                               │
                                     Phase 8 (Ikram + tous) ── benchmark, rapport, démo
```

**Phases 1, 2 et 3 démarrent en parallèle** : personne n'attend personne au début.

---

## 5. Les phases en détail

Chaque phase est décrite avec la même structure :
**Objectif · Pourquoi · Étapes · Livrables · C'est terminé quand…**

---

### Phase 0 — Mise en place
**Responsable :** tous · **État :** ✅ terminée pour Hamid, à faire pour Youssef et Ikram

- [x] Dépôt GitHub `hamid78631/DeployOps` créé.
- [x] Projet Python initialisé avec `uv` (paquet `src/deployops`, Python 3.12).
- [x] `.gitignore` (secrets, Python, éditeurs) et `.env.example`.
- [x] README avec la description et les rôles.
- [ ] Youssef et Ikram : cloner le dépôt, vérifier `git config`, `uv sync`, `uv run deployops`.
- [ ] Youssef et Ikram : vérifier les outils (Docker Desktop **lancé** avant) :
  ```bash
  docker version              # parties Client ET Server
  docker run --rm hello-world
  kubectl version --client
  minikube version
  minikube start
  kubectl get nodes           # → minikube   Ready   control-plane
  ```
- [ ] Youssef et Ikram : installer le socle (section 3).

**C'est terminé quand…** les 3 membres ont un `kubectl get nodes` qui répond `Ready`
et ont fait au moins un commit à leur nom.

---

### Phase 1 — Le projet de test : prédiction de churn
**Responsable :** Ikram · **Dépend de :** phase 0

**Objectif.** Fournir un **projet ML complet**, comme un utilisateur le soumettrait à l'agent,
ainsi que des **versions volontairement cassées** pour tester l'auto-correction.

**Pourquoi.** Le professeur attend que l'agent reçoive un projet avec **entraînement et
inférence**, et qu'il génère **un Dockerfile par application**. Le projet de test doit donc
contenir les deux. C'est la même structure que le TP iris corrigé en S04.

**Structure attendue.**
```
churn-project/
├── data/telco_churn.csv        # données (versées dans MinIO par l'agent)
├── train.py                    # entraînement → model.joblib
├── app.py                      # API FastAPI : GET /health, POST /predict
├── requirements-train.txt      # dépendances de l'entraînement
├── requirements-api.txt        # dépendances de l'API
└── README.md                   # description du projet
```
**Pas de Dockerfile ni de manifeste** : c'est le travail de l'agent.

**Étapes.**
1. Récupérer le dataset **Telco Customer Churn** (CSV public, ~7 000 clients).
2. `train.py` : lit les données, prépare les colonnes, entraîne un modèle scikit-learn,
   écrit `model.joblib` à l'emplacement donné par la variable `MODEL_PATH`.
3. `app.py` : charge `model.joblib` (chemin dans `MODEL_PATH`) et expose
   - `GET /health` → `{"status": "ok"}` (servira aux sondes Kubernetes),
   - `POST /predict` → `{"churn": true/false, "probability": 0.73}`.
4. Vérifier que tout marche en local, sans Docker.
5. Produire le **zip** du projet : c'est ce fichier qu'on soumettra à l'agent.
6. Créer les **variantes cassées**, chacune provoquant une panne précise :

   | Variante | Ce qu'on casse | Panne attendue |
   |---|---|---|
   | `missing-dep` | une bibliothèque retirée des requirements | `CrashLoopBackOff` |
   | `wrong-port` | l'app écoute sur un autre port que celui annoncé | sonde en échec |
   | `low-memory` | la demande impose une limite mémoire trop basse | `OOMKilled` |
   | `missing-secret` | l'app exige une variable fournie par un Secret absent | Secret manquant |
   | `bad-image` | nom d'image erroné | `ImagePullBackOff` |

**Livrables.** Un dossier `target-app/` avec le projet sain, ses variantes, et un README.

**C'est terminé quand…** `train.py` produit le modèle, l'API répond à `/health` et `/predict`
en local, et chaque variante est documentée.

---

### Phase 2 — Le socle Kubernetes et la sécurité
**Responsable :** Hamid · **État :** ✅ socle terminé (PR #3, #4, #5) · compléments ci-dessous

**Objectif.** Préparer le cluster pour que l'agent ait **un terrain de jeu délimité**
et **le moins de droits possible**, ainsi que le **stockage** dont il a besoin.

**Pourquoi.** Un agent IA peut se tromper ou être manipulé (par exemple par un texte piégé
dans le projet soumis). Les garde-fous ne doivent pas reposer sur le LLM, mais sur
Kubernetes lui-même : même si l'agent « veut » supprimer un nœud, le cluster refuse.

**Fait.**
- [x] **Deux namespaces** : `deployops-target` (où l'agent déploie) et `deployops-system`
  (MinIO, secrets ; l'agent n'y a aucun droit).
- [x] **RBAC** : ServiceAccount `deployops-agent`, Role limité à `deployops-target`
  (Deployments, Services, ConfigMaps, Secrets ; lecture des Pods, logs, events, ReplicaSets ;
  **aucun `delete`**), RoleBinding.
- [x] **10 tests `kubectl auth can-i`** validés (dont : pas de suppression, pas d'accès à
  `default` ni à `deployops-system`, pas d'escalade de privilèges).
- [x] **MinIO** dans `deployops-system` : PVC 1 Go, Deployment sécurisé (non-root, sondes,
  ressources), Service ClusterIP, Secret créé hors Git, bucket `audit`.
  Image : `pgsty/minio` (les images officielles ne sont plus distribuées publiquement).

**À compléter (nouvelle vision).**
- [ ] **RBAC** : ajouter la création et la lecture des **Jobs** (groupe `batch`) pour
  l'entraînement, et des **PersistentVolumeClaims** pour stocker le modèle entraîné.
  Refaire les tests `can-i` correspondants.
- [ ] **MinIO** : créer les buckets **`datasets`** et **`models`** en plus de `audit`.
- [ ] **Images dans Minikube** : valider la méthode pour qu'une image construite par l'agent
  soit visible par le cluster (`minikube image load`, ou construction dans le Docker de Minikube).
- [ ] **Tests de sécurité automatisés** (pytest) pour remplacer les tests à la main.

**Livrables.** Le dossier `k8s/` (namespaces, RBAC, MinIO) et les tests de sécurité.

**C'est terminé quand…** les tests `can-i` (y compris Jobs) passent automatiquement et les
3 buckets existent.

---

### Phase 3 — Les outils encadrés
**Responsable :** Youssef · **Dépend de :** phase 0

**Objectif.** Écrire les **seules fonctions** par lesquelles l'agent peut agir sur le système.

**Pourquoi.** On ne donne **jamais** un terminal libre au LLM. Il ne peut qu'appeler ces
fonctions, qui vérifient tout avant d'exécuter quoi que ce soit. C'est la « liste blanche »
du cahier des charges.

**Étapes.** Écrire et tester **sans LLM** chacun de ces outils :

| Outil | Rôle |
|---|---|
| `extract_project` | Décompresse le zip soumis dans un dossier de travail isolé (refuser les chemins dangereux, ex. `../`) |
| `analyze_project` | Détecte les applications, l'entraînement, les données, le modèle, le framework, le port, les dépendances |
| `upload_to_minio` | Verse données et modèles dans les buckets `datasets` / `models` (versionnés par date ou empreinte) |
| `check_prerequisites` | Vérifie Docker et Minikube ; démarre Minikube si besoin ; **n'installe rien sans validation** |
| `docker_build` | Construit une image à partir d'un Dockerfile généré |
| `load_image` | Rend l'image disponible à Minikube (`minikube image load`) |
| `kubectl_dry_run` | `kubectl apply --dry-run=server` : le cluster valide sans rien créer |
| `kubectl_apply` | Applique — **refuse** tant que le plan n'est pas validé par un humain |
| `kubectl_get` / `describe` / `logs` / `rollout_status` | Observer l'état du déploiement et du Job d'entraînement |
| `audit_log` | Enregistre chaque action dans MinIO (qui, quoi, quand, pourquoi) |

Règles communes à tous les outils :
- arguments validés par **Pydantic** (pas de chaîne de commande libre) ;
- namespace **imposé** (`deployops-target`), impossible à changer par le LLM ;
- **timeout** sur chaque commande ;
- chaque appel produit une entrée d'audit.

**Livrables.** Le code des outils dans `src/deployops/` et leurs **tests pytest**.

**C'est terminé quand…** chaque outil fonctionne seul, et un test prouve qu'une commande
hors liste blanche, un autre namespace ou un zip piégé est **refusé**.

---

### Phase 4 — L'agent LangGraph + Groq
**Responsable :** Youssef · **Dépend de :** phase 3

**Objectif.** Construire le « cerveau » qui enchaîne les outils selon le pipeline de la section 1.

**Le graphe :**
```
 soumission ─► analyser ─► verser données (MinIO) ─► générer ─► vérifier prérequis ─► dry-run
                                                       ▲                                 │
                                                       │ modifications demandées          ▼
                                                       └──────────────────── [VALIDATION HUMAINE]
                                                                                         │ validé
                                                                                         ▼
                        rapport ◄── succès ◄── observer ◄── appliquer ◄── entraîner (Job) ◄── build + load images
                                                  │ échec
                                                  ▼
                                           diagnostiquer ─► corriger (max N essais) ─► dry-run
```

**Étapes.**
1. Connexion à Groq (clé lue depuis `.env`, modèle `llama-3.3-70b-versatile`).
2. Schémas **Pydantic** : la demande (applications, réplicas, exposition, ressources),
   l'analyse du projet, les fichiers générés, le diagnostic. Le LLM doit répondre dans ce
   format, sinon on rejette.
3. Prompts : analyse du projet, génération (**un Dockerfile par application**, manifestes,
   Job d'entraînement, workflow CI), prise en compte d'une **demande de modification**,
   diagnostic d'une panne.
4. **Qualité des fichiers générés** (point clé de l'évaluation) : appliquer les 7 règles du
   cours S02 (version épinglée, ordre des couches, image slim, multi-étapes, non-root,
   HEALTHCHECK, `.dockerignore`) et, pour les manifestes, sondes, ressources et labels.
5. **Boucle de validation** : *interrupt* LangGraph ; l'utilisateur valide **ou** écrit une
   demande de modification, l'agent régénère et réaffiche.
6. **Diagnostic** : à partir de `describe` + `logs` + events, reconnaître
   `ImagePullBackOff`, `CrashLoopBackOff`, `OOMKilled`, sonde en échec, Secret manquant,
   puis proposer une correction.
7. **Nombre maximal d'itérations** de correction (variable `MAX_ITERATIONS`).
8. **Rapport final** : statut, Pods, Services, corrections effectuées, où trouver le modèle.
9. Une **CLI** pour tester sans IHM : `uv run deployops churn-project.zip "déploie avec 2 réplicas"`.

**C'est terminé quand…** l'agent déploie le projet churn sain de bout en bout
(entraînement + API), accepte une demande de modification, et corrige au moins deux
variantes cassées.

---

### Phase 5 — L'interface de pilotage (Streamlit)
**Responsable :** Ikram · **Dépend de :** phase 4

**Objectif.** Permettre à un humain de piloter l'agent sans terminal.

**Écrans.**
- **Soumission** : envoyer le **zip** du projet et écrire la demande en langage naturel.
- **Plan** : afficher l'analyse et les fichiers générés (Dockerfiles, YAML, workflow)
  avec le résultat du dry-run, puis **Valider** ou **Demander des modifications** (zone de texte).
- **Suivi** : étapes de l'agent en direct, état du Job d'entraînement et des Pods.
- **Rapport** : résultat final (succès / échec et explication).
- **Audit** : historique des actions, lu depuis MinIO.

**C'est terminé quand…** on peut faire tout le scénario de démo depuis le navigateur.

---

### Phase 6 — Mise en service de l'agent
**Responsable :** Hamid · **Dépend de :** phases 4 et 5

**Objectif.** L'agent et son IHM doivent être conteneurisés et déployés
(exigence : « agent conteneurisé et déployé »). **La cible exacte dépend de la question 2
de la section 8.**

**Étapes.**
1. **Dockerfiles optimisés** de l'agent et de l'IHM selon les 7 règles du cours (S02).
2. **`docker-compose.yml`** pour lancer agent + IHM (+ MinIO si besoin) (S04).
3. Selon la décision de la section 8 :
   - **soit** l'agent tourne avec Docker Compose sur la machine (il peut alors vérifier et
     démarrer Docker et Minikube) ;
   - **soit** il tourne dans `deployops-system` (Deployment + Service, ServiceAccount de la
     phase 2, Secret pour la clé Groq), et la vérification des prérequis se limite au cluster.
4. Scan **Trivy** des images, correction des vulnérabilités critiques.

**C'est terminé quand…** l'IHM est accessible depuis le navigateur et pilote l'agent conteneurisé.

---

### Phase 7 — CI/CD avec GitHub Actions
**Responsable :** Hamid · **Peut commencer dès maintenant**

**Objectif.** Automatiser la vérification du projet à chaque push et à chaque PR.

**Deux workflows distincts :**
1. **Celui de notre dépôt** (`.github/workflows/ci.yml`), construit étape par étape :
   validation des manifestes (kubeconform), scan Trivy (secrets, failles, mauvaises
   configurations), lint et tests pytest, build des images.
2. **Celui du projet soumis** : il n'est pas écrit à la main, il est **généré par l'agent**
   (exigence du cahier des charges). Hamid fournit à Youssef un exemple de référence.

**Limite à connaître.** Les machines de GitHub ne peuvent pas joindre un Minikube local :
la CD s'arrête à la construction et à la publication des images.

**C'est terminé quand…** une PR affiche un check vert, et un check rouge si on introduit
une erreur.

---

### Phase 8 — Benchmark, rapport et démonstration
**Responsable :** Ikram (benchmark), tous (rapport et démo)

**Benchmark** sur le projet sain + les 5 variantes cassées, plusieurs essais chacune :

| Mesure | Question |
|---|---|
| Taux de succès | L'agent arrive-t-il à un déploiement fonctionnel ? |
| Itérations | Combien de corrections lui faut-il ? |
| Temps | Combien de temps de la soumission au Pod `Running` ? |
| Qualité | Les Dockerfiles et manifestes générés respectent-ils les règles du cours ? |
| Sécurité | Combien d'actions interdites a-t-il tentées et qui ont été bloquées ? |
| Échecs | Pourquoi échoue-t-il quand il échoue ? |

**Rapport** (français ou anglais), structure imposée :
1. Introduction
2. Méthodologie adoptée
3. État de l'art (agents LLM pour le DevOps, LangGraph, smolagents, PydanticAI…)
4. Résultats et discussion (le benchmark)
5. Conclusion et perspectives

**Démonstration** (en direct, en classe) :
- soumission du zip, génération des fichiers, **une demande de modification**, validation ;
- entraînement puis déploiement de bout en bout ;
- une panne détectée et corrigée par l'agent ;
- une action refusée par les garde-fous ;
- le rapport et l'audit consultés dans l'IHM.
Prévoir une **vidéo de secours** au cas où le direct échoue.

**Bonus si le temps le permet.** Un 2ᵉ projet de test (ex. analyse de sentiment avec Flask)
pour prouver que l'agent est générique.

---

## 6. Lien avec le cours

| Séance | Notion | Où on l'utilise |
|---|---|---|
| S01 | Cloud, souveraineté, cycle de vie d'un modèle, stockage objet | Introduction du rapport, Groq, MinIO |
| S02 | Dockerfile, 7 règles, multi-étapes, Trivy | Dockerfiles générés par l'agent (cœur de l'évaluation), phases 6, 7 |
| S04 | Docker Compose, Dockerfile.train / Dockerfile.infer, volumes | Structure du projet de test, phase 6 |
| S05 | Cluster, Pod, état désiré, kubectl, manifestes, namespaces, Job | Phases 2, 3, 4, 6 |

---

## 7. Ce qui a changé en version 2 (après l'explication du professeur)

| # | Sujet | Version 1 | Version 2 |
|---|---|---|---|
| 1 | Entrée de l'agent | Un dépôt avec une seule API | **Un projet complet (zip ou dépôt)**, avec entraînement possible |
| 2 | Dockerfiles | Un seul | **Un par application** (ex. entraînement + API) |
| 3 | Entraînement | Non prévu | **Job Kubernetes** lancé par l'agent |
| 4 | MinIO | Audit uniquement | Audit **+ données + modèles** |
| 5 | Prérequis | Non prévus | L'agent **vérifie et démarre** Docker / Minikube |
| 6 | Validation | Valider / Refuser | **Valider / Demander des modifications**, en boucle |
| 7 | Déploiement | Apply | **Build des images → chargement dans Minikube → entraînement → apply** |

Ce qui ne change pas : le socle de la phase 2, les règles de sécurité, les rôles,
l'auto-correction, le rapport, l'audit.

---

## 8. Questions ouvertes (à valider avec le professeur)

1. **Installation automatique de Docker / Kubernetes.** Installer des logiciels est une action
   lourde et risquée pour un agent. **Proposition :** l'agent vérifie et démarre
   (`minikube start`), mais l'installation est seulement **proposée**, avec validation humaine.
2. **Où tourne l'agent ?** S'il doit vérifier et démarrer Docker et Minikube, il doit tourner
   **sur la machine** (Docker Compose), pas dans un Pod du cluster (un Pod ne peut pas démarrer
   le cluster qui le contient). « Conteneurisé et déployé » peut-il vouloir dire « lancé avec
   Docker Compose » ?
3. **Format de soumission.** Zip uniquement, ou zip **et** dépôt Git ?
