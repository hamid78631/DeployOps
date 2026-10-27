# DeployOps Agent — Plan détaillé des phases

Document de référence de l'équipe. Il donne à chacun la **même vue d'ensemble** :
ce qu'on construit, pourquoi, dans quel ordre, et ce que chaque phase doit livrer.

> À lire en entier une première fois, puis à consulter avant de commencer chaque phase.

---

## 1. Le projet en une page

### Ce qu'on construit
Un **agent IA** qui reçoit une demande en langage naturel, par exemple :

> « Déploie le modèle de churn avec 2 réplicas »

et qui, tout seul :

1. **analyse** le dépôt de l'application (modèle, dépendances, port, point d'entrée) ;
2. **génère** le Dockerfile, les manifestes Kubernetes et le workflow GitHub Actions ;
3. **vérifie** ses fichiers par un *dry-run* et **demande une validation humaine** ;
4. **applique** sur le cluster, **observe** le résultat et **se corrige** en cas d'échec ;
5. **rédige un rapport** et garde une **trace d'audit** de tout ce qu'il a fait.

### Ce qu'on ne construit PAS
L'application de churn n'est **pas** le projet : c'est une **donnée d'entrée** qui sert
à tester l'agent. Le livrable principal, c'est **l'agent**.

### Le schéma global

```
 Utilisateur ──► IHM Streamlit ──► Agent LangGraph ◄──► LLM Groq
                    ▲                    │
                    │ validation         │ outils encadrés (liste blanche)
                    │ humaine            ▼
                    │            docker build / kubectl (dry-run, apply, get,
                    │            describe, logs, rollout status)
                    │                    │
                    │                    ▼
                    │          Cluster Minikube
                    │          ├─ namespace deployops-target  (l'app churn déployée)
                    │          └─ namespace deployops-system  (agent, IHM, MinIO)
                    │                    │
                    └──── audit ◄────── MinIO (bucket "audit")
```

### Choix techniques

| Élément | Choix | Pourquoi |
|---|---|---|
| Application cible | Prédiction de **churn** (Telco Customer Churn), scikit-learn + FastAPI | Cas métier concret (exemple du cours S01), léger, données publiques |
| LLM | **Groq** (API distante, gratuite) | Rapide et gratuit. Seules la structure du dépôt et les erreurs K8s sont envoyées, **jamais de données métier** (argument à mettre dans le rapport) |
| Agent | **LangGraph** | Son graphe correspond exactement à la boucle planifier → agir → observer → corriger ; gère la pause pour validation humaine |
| Validation | **Pydantic** | Sorties du LLM structurées, arguments des outils contrôlés |
| Projet Python | **uv** | Environnement et versions identiques pour les 3 membres (`uv.lock`) |
| Cluster | **Minikube** | Vu en cours (S05), gratuit, local |
| Audit | **MinIO** | Stockage objet compatible S3, imposé par le cahier des charges |
| IHM | **Streamlit** | Simple, en Python, cité dans le cas d'usage du cours |
| CI/CD | **GitHub Actions** + **Trivy** | Imposé par le cahier des charges ; Trivy vu en S02 |

### Règles de sécurité (obligatoires, vérifiées par le prof)
- **Dry-run** (`kubectl apply --dry-run=server`) avant toute application.
- **Validation humaine** du plan avant tout `kubectl apply`.
- **Liste blanche** de commandes : `docker build`, `kubectl apply --dry-run=server`,
  `get`, `describe`, `logs`, `rollout status` (+ `apply` après validation). Rien d'autre.
- **RBAC minimal** : l'agent a son propre ServiceAccount, limité à **un seul namespace**.
- **Journal d'audit** : *qui, quoi, quand, pourquoi*, stocké dans MinIO.
- **Aucun secret dans Git** : la clé Groq est dans `.env` (ignoré) en local et dans un
  Secret Kubernetes sur le cluster.

---

## 2. L'équipe

| Membre | Rôle | Phases | En une phrase |
|---|---|---|---|
| **Hamid** | A — Infra & sécurité | 2, 6, 7 | Prépare le terrain où l'agent travaille, puis met l'agent lui-même en production |
| **Youssef** | B — Agent & IA | 3, 4 | Construit le cerveau (LangGraph + Groq) et ses mains (les outils) |
| **Ikram** | C — Interface & qualité | 1, 5, 8 | Fournit l'application à déployer, l'interface de pilotage, et mesure les résultats |
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

---

## 4. Ordre des phases et dépendances

```
Phase 0 (tous)
   │
   ├──► Phase 1 (Ikram)  ── application churn ──────────────┐
   ├──► Phase 2 (Hamid)  ── socle K8s + sécurité ───────────┤
   └──► Phase 3 (Youssef) ── outils encadrés ──► Phase 4 ───┤
                                              (Youssef)     │
                                                            ▼
                                     Phase 5 (Ikram) ── IHM Streamlit
                                                            │
                                     Phase 6 (Hamid) ── l'agent en production
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
**Responsable :** tous · **État :** presque terminée

**Objectif.** Avoir un dépôt commun et un poste de travail prêt chez chacun.

**Étapes.**
- [x] Dépôt GitHub `hamid78631/DeployOps` créé.
- [x] Projet Python initialisé avec `uv` (paquet `src/deployops`, Python 3.12).
- [x] `.gitignore` (secrets, Python, éditeurs) et `.env.example`.
- [x] README avec la description et les rôles.
- [ ] Youssef et Ikram : cloner le dépôt, vérifier `git config`, `uv sync`, `uv run deployops`.
- [ ] Chacun : vérifier les outils (Docker Desktop **lancé** avant) :
  ```bash
  docker version              # parties Client ET Server
  docker run --rm hello-world
  kubectl version --client
  minikube version
  minikube start
  kubectl get nodes           # → minikube   Ready   control-plane
  ```

**C'est terminé quand…** les 3 membres ont un `kubectl get nodes` qui répond `Ready`
et ont fait au moins un commit à leur nom.

---

### Phase 1 — L'application cible : prédiction de churn
**Responsable :** Ikram · **Dépend de :** phase 0

**Objectif.** Fournir une petite application ML réaliste que l'agent devra déployer,
ainsi que des **versions volontairement cassées** pour tester l'auto-correction.

**Pourquoi.** Sans application à déployer, on ne peut pas tester l'agent. Et sans
versions cassées, on ne peut pas prouver qu'il sait se corriger (exigence du cahier des charges).

**Étapes.**
1. Récupérer le dataset **Telco Customer Churn** (CSV public, ~7 000 clients).
   Le CSV n'est pas commité (ignoré par `.gitignore`) : prévoir une commande ou un
   script pour le télécharger.
2. `train.py` : préparation des données (colonnes catégorielles, valeurs manquantes),
   entraînement d'un modèle scikit-learn, sauvegarde en `model.joblib`.
3. `app.py` : API **FastAPI** avec
   - `GET /health` → `{"status": "ok"}` (servira aux sondes Kubernetes),
   - `POST /predict` → reçoit les caractéristiques d'un client, renvoie
     `{"churn": true/false, "probability": 0.73}`.
4. `requirements.txt` propre à l'application.
5. **Ne PAS écrire de Dockerfile ni de manifestes K8s** : c'est le travail de l'agent.
6. Créer les **variantes cassées**, chacune provoquant une panne précise :

   | Variante | Ce qu'on casse | Panne attendue |
   |---|---|---|
   | `missing-dep` | une bibliothèque retirée de `requirements.txt` | `CrashLoopBackOff` |
   | `wrong-port` | l'app écoute sur un autre port que celui annoncé | sonde en échec |
   | `low-memory` | la demande impose une limite mémoire trop basse | `OOMKilled` |
   | `missing-secret` | l'app exige une variable fournie par un Secret absent | Secret manquant |
   | `bad-image` | nom d'image erroné | `ImagePullBackOff` |

**Livrables.** Un dossier `target-app/` avec la version saine et les variantes,
plus un petit README qui explique chaque variante.

**C'est terminé quand…** l'application saine tourne en local (`uv run` ou `uvicorn`)
et répond à `/health` et `/predict`, et chaque variante est documentée.

---

### Phase 2 — Le socle Kubernetes et la sécurité
**Responsable :** Hamid · **Dépend de :** phase 0

**Objectif.** Préparer le cluster pour que l'agent ait **un terrain de jeu délimité**
et **le moins de droits possible**.

**Pourquoi.** Un agent IA peut se tromper ou être mal guidé. Les garde-fous ne doivent
pas reposer sur le LLM, mais sur Kubernetes lui-même : même si l'agent « veut » supprimer
un nœud, le cluster doit refuser.

**Étapes.**
1. **Deux namespaces** (des « dossiers » du cluster, vus en S05) :
   - `deployops-target` : le seul endroit où l'agent peut déployer ;
   - `deployops-system` : là où tournent l'agent, l'IHM et MinIO.
2. **RBAC** :
   - un **ServiceAccount** `deployops-agent` (l'identité de l'agent dans le cluster) ;
   - un **Role** dans `deployops-target` autorisant uniquement :
     Deployments, Services, ConfigMaps, Secrets (créer/modifier/lire),
     Pods et Events (lire), logs des Pods (lire) ;
   - un **RoleBinding** qui associe le Role au ServiceAccount.
3. **MinIO** déployé dans `deployops-system`, avec un volume persistant et un bucket `audit`.
4. **Tests de sécurité** (à garder pour le rapport) :
   ```bash
   kubectl auth can-i create deployments -n deployops-target \
     --as=system:serviceaccount:deployops-system:deployops-agent   # → yes
   kubectl auth can-i delete nodes \
     --as=system:serviceaccount:deployops-system:deployops-agent   # → no
   kubectl auth can-i create deployments -n default \
     --as=system:serviceaccount:deployops-system:deployops-agent   # → no
   ```

**Livrables.** Un dossier `k8s/` avec les manifestes (namespaces, RBAC, MinIO),
appliqués avec la méthode GMA du cours (générer, modifier, appliquer).

**C'est terminé quand…** les 3 tests `can-i` donnent les bons résultats et la console
MinIO affiche le bucket `audit`.

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
| `analyze_repo` | Lit le dépôt cible : framework, port, fichier du modèle, dépendances, point d'entrée |
| `docker_build` | Construit l'image à partir du Dockerfile généré |
| `kubectl_dry_run` | `kubectl apply --dry-run=server` : le cluster valide sans rien créer |
| `kubectl_apply` | Applique — **refuse** tant que le plan n'est pas validé par un humain |
| `kubectl_get` / `describe` / `logs` / `rollout_status` | Observer l'état du déploiement |
| `audit_log` | Enregistre chaque action dans MinIO (qui, quoi, quand, pourquoi) |

Règles communes à tous les outils :
- arguments validés par **Pydantic** (pas de chaîne de commande libre) ;
- namespace **imposé** (`deployops-target`), impossible à changer par le LLM ;
- **timeout** sur chaque commande ;
- chaque appel produit une entrée d'audit.

**Livrables.** Le code des outils dans `src/deployops/` et leurs **tests pytest**.

**C'est terminé quand…** chaque outil fonctionne seul, et un test prouve qu'une commande
hors liste blanche ou un autre namespace est **refusé**.

---

### Phase 4 — L'agent LangGraph + Groq
**Responsable :** Youssef · **Dépend de :** phase 3

**Objectif.** Construire le « cerveau » qui enchaîne les outils pour réaliser la demande.

**Le graphe :**
```
analyser ─► planifier ─► générer ─► dry-run ─► [VALIDATION HUMAINE] ─► appliquer ─► observer
                                                       │ refus                        │
                                                       ▼                     succès ◄─┤─► échec
                                                     arrêt                     │           │
                                                                               ▼           ▼
                                                                         rapport    diagnostiquer
                                                                                         │
                                                                                         ▼
                                                                    corriger (max N itérations) ─► dry-run
```

**Étapes.**
1. Connexion à Groq (clé lue depuis `.env`, modèle `llama-3.3-70b-versatile`).
2. Schémas **Pydantic** pour les sorties du LLM : le plan, les fichiers générés,
   le diagnostic. Le LLM doit répondre dans ce format, sinon on rejette.
3. Prompts : analyse du dépôt, génération des fichiers, diagnostic d'une panne.
4. **Diagnostic** : à partir de `describe` + `logs` + events, reconnaître
   `ImagePullBackOff`, `CrashLoopBackOff`, `OOMKilled`, sonde en échec, Secret manquant,
   puis proposer une correction.
5. **Pause de validation humaine** (mécanisme *interrupt* de LangGraph).
6. **Nombre maximal d'itérations** de correction (variable `MAX_ITERATIONS`).
7. **Rapport final** : statut, Pods, Services, corrections effectuées.
8. Une **CLI** pour tester sans IHM : `uv run deployops "déploie le modèle avec 2 réplicas"`.

**C'est terminé quand…** l'agent déploie la version saine de l'application churn de bout
en bout, et corrige au moins deux variantes cassées.

---

### Phase 5 — L'interface de pilotage (Streamlit)
**Responsable :** Ikram · **Dépend de :** phase 4

**Objectif.** Permettre à un humain de piloter l'agent sans terminal.

**Écrans.**
- **Chat** : saisir la demande en langage naturel.
- **Plan** : afficher le plan et les fichiers générés (Dockerfile, YAML, workflow),
  avec les boutons **Valider** / **Refuser**.
- **Suivi** : étapes de l'agent en direct, état des Pods.
- **Audit** : historique des actions, lu depuis MinIO.

**C'est terminé quand…** on peut faire tout le scénario de démo depuis le navigateur.

---

### Phase 6 — Conteneuriser et déployer l'agent lui-même
**Responsable :** Hamid · **Dépend de :** phases 4 et 5

**Objectif.** L'agent et son IHM doivent eux-mêmes tourner dans Kubernetes
(exigence : « agent conteneurisé et déployé »).

**Étapes.**
1. **Dockerfiles optimisés** selon les 7 règles du cours (S02) : version épinglée,
   ordre des couches, `.dockerignore`, image `slim`, build multi-étapes,
   utilisateur non-root, `HEALTHCHECK`.
2. **`docker-compose.yml`** pour lancer agent + IHM + MinIO en local (S04).
3. **Manifestes K8s** dans `deployops-system` : Deployment + Service pour l'agent et l'IHM,
   ConfigMap pour la configuration, **Secret pour la clé Groq**.
4. L'agent utilise le **ServiceAccount** de la phase 2.
5. Scan **Trivy** des images, correction des vulnérabilités critiques.

**C'est terminé quand…** l'IHM est accessible depuis le navigateur alors que tout tourne
dans Minikube.

---

### Phase 7 — CI/CD avec GitHub Actions
**Responsable :** Hamid · **En parallèle de :** phase 6

**Objectif.** Automatiser la vérification du projet à chaque push.

**Deux workflows distincts :**
1. **Celui de notre dépôt** (`.github/workflows/`) : lint, tests pytest, build des images,
   scan Trivy (échec si vulnérabilité CRITICAL).
2. **Celui de l'application cible** : il n'est pas écrit à la main, il est **généré par l'agent**
   (exigence du cahier des charges). Hamid peut fournir à Youssef un exemple de référence.

**C'est terminé quand…** une PR affiche un check vert, et un check rouge si on introduit
une erreur.

---

### Phase 8 — Benchmark, rapport et démonstration
**Responsable :** Ikram (benchmark), tous (rapport et démo)

**Benchmark** sur la version saine + les 5 variantes cassées, plusieurs essais chacune :

| Mesure | Question |
|---|---|
| Taux de succès | L'agent arrive-t-il à un déploiement fonctionnel ? |
| Itérations | Combien de corrections lui faut-il ? |
| Temps | Combien de temps de la demande au Pod `Running` ? |
| Sécurité | Combien d'actions interdites a-t-il tenté et qui ont été bloquées ? |
| Échecs | Pourquoi échoue-t-il quand il échoue ? |

**Rapport** (français ou anglais), structure imposée :
1. Introduction
2. Méthodologie adoptée
3. État de l'art (agents LLM pour le DevOps, LangGraph, smolagents, PydanticAI…)
4. Résultats et discussion (le benchmark)
5. Conclusion et perspectives

**Démonstration** (en direct, en classe) :
- un déploiement sain de bout en bout ;
- une panne détectée et corrigée par l'agent ;
- une action refusée par les garde-fous ;
- l'audit consulté dans l'IHM.
Prévoir une **vidéo de secours** au cas où le direct échoue.

**Bonus si le temps le permet.** Une 2ᵉ application cible (ex. analyse de sentiment
avec Flask, port 5000) pour prouver que l'agent est générique.

---

## 6. Lien avec le cours

| Séance | Notion | Où on l'utilise |
|---|---|---|
| S01 | Cloud, souveraineté, cycle de vie d'un modèle | Introduction du rapport, justification de Groq |
| S02 | Dockerfile, 7 règles, multi-étapes, Trivy | Phases 3, 6, 7 (et Dockerfiles générés par l'agent) |
| S04 | Docker Compose, réseau par nom de service, volumes | Phase 6 (développement local) |
| S05 | Cluster, Pod, état désiré, kubectl, manifestes, namespaces | Phases 2, 3, 4, 6 |
