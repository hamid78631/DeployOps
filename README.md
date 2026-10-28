# DeployOps Agent

Agent IA autonome qui transforme un modèle de Machine Learning en service
déployé sur Kubernetes, à partir d'une demande en langage naturel
(ex. « déploie ce modèle avec 2 réplicas »).

L'agent génère le Dockerfile, les manifestes Kubernetes et le workflow CI,
demande une validation humaine avant d'appliquer, puis corrige ses erreurs.

Projet du module *Cloud Computing et Déploiement des Modèles IA* (3ACI).

## Équipe

| Membre  | Rôle                    | Phases                                   |
|---------|-------------------------|------------------------------------------|
| Hamid   | Infra & sécurité        | Socle K8s, RBAC, MinIO, déploiement, CI  |
| Youssef | Agent & IA              | Outils encadrés, agent LangGraph + Groq  |
| Ikram   | Interface & qualité     | Application churn, IHM Streamlit, benchmark |

## Démarrage

Prérequis : Python 3.12, [uv](https://docs.astral.sh/uv/), Docker, Minikube, kubectl.

    cp .env.example .env    # puis renseigner GROQ_API_KEY
    uv sync
    uv run deployops

C