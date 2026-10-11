# NextAct — Guide d'installation du projet (ERP + agent IA)

Documentation complète du projet **NextAct** : une application ERP de reporting (stock, ventes, achats, fournisseurs, clients, devis, paiements, remboursements) accompagnée d'un **agent IA** qui répond en langage naturel aux questions sur les données, génère des exports et envoie des alertes par e-mail.

Ce dépôt contient **uniquement la documentation** : l'installation pas à pas, l'architecture, et le détail de chaque partie (base de données, backend, frontend, agent IA).

## 🚀 Par où commencer ?

| Je veux… | J'ouvre… |
|---|---|
| **Installer et lancer tout le projet** | [**06-guide-installation.md**](06-guide-installation.md) |
| **Comprendre comment le projet est construit** | [01-architecture-globale.md](01-architecture-globale.md) |
| Détailler une partie en particulier | Le dossier correspondant, ci-dessous |

## Les quatre parties du projet

Elles s'installent **dans cet ordre** :

| Ordre | Partie | Ce que c'est | Documentation |
|:-:|---|---|---|
| 1 | **Base de données** | PostgreSQL : 17 tables ERP, 27 vues, 44 fonctions SQL, deux rôles de sécurité | [`02-base-de-donnees/`](02-base-de-donnees/README.md) |
| 2 | **Backend** | API REST FastAPI : authentification, rapports, exports, alertes e-mail | [`03-backend/`](03-backend/README.md) |
| 3 | **Agent IA** | Assistant conversationnel : 81 outils, contrôle d'accès par rôle, garde-fous | [`05-ai-agent/`](05-ai-agent/README.md) |
| 4 | **Frontend** | Application web React : dashboard, 8 rapports, administration, chatbot | [`04-frontend/`](04-frontend/README.md) |

> Les numéros de dossier (`02` à `05`) suivent l'organisation du dépôt ; l'**ordre d'installation** est celui du tableau ci-dessus.

## Contenu du dépôt

```
├── README.md                       ← vous êtes ici
├── 01-architecture-globale.md      ← vue d'ensemble : composants, parcours, rôles, configuration
├── 02-base-de-donnees/
│   ├── README.md                   ← guide d'installation de la base (pgAdmin 4)
│   ├── install_database.sql        ← tout le SQL, à copier-coller dans l'ordre
│   ├── schema-erd.md               ← schéma des relations
│   ├── tables.md                   ← description des tables et colonnes
│   ├── vues.md                     ← les 27 vues
│   └── fonctions.md                ← les 44 fonctions
├── 03-backend/
│   ├── README.md                   ← vue d'ensemble du backend
│   ├── configuration.md            ← variables d'environnement, installation, lancement
│   ├── structure-projet.md         ← arborescence et cycle d'une requête
│   ├── repositories.md             ← accès aux données par domaine
│   ├── routes-api.md               ← référence des routes
│   └── authentification.md         ← JWT, rôles, comptes, compte de service
├── 04-frontend/
│   ├── README.md                   ← installation et vue d'ensemble
│   ├── structure-projet.md         ← arborescence, flux de données, rôles
│   ├── pages-et-composants.md      ← pages, composants, hooks
│   └── services-api.md             ← contrat avec le backend, chatbot
├── 05-ai-agent/
│   ├── README.md                   ← guide d'installation de l'agent
│   ├── architecture-agent.md       ← architecture et logique de décision
│   ├── tools-et-schemas.md         ← les 81 outils et les droits par rôle
│   ├── system-prompt.md            ← les instructions données au modèle
│   └── securite.md                 ← protections, limites connues, correctifs
└── 06-guide-installation.md        ← installation complète, étape par étape
```

## Le projet en chiffres

| Élément | Valeur |
|---|---|
| Tables de l'ERP | 17 (+ `users`) |
| Vues / fonctions SQL de reporting | 27 / 44 |
| Outils de l'agent IA | 81 |
| Rôles utilisateurs | 9 (`admin` + 8 rôles de domaine) |
| Pages du frontend | 15, dont 8 rapports |
| Ports | PostgreSQL `5432` · backend `8000` · agent IA `8080` · frontend `5173` |

## Technologies

| Partie | Technologies |
|---|---|
| Base de données | PostgreSQL, pgAdmin 4 |
| Backend | Python, FastAPI, SQLAlchemy 2.x, JWT, bcrypt, APScheduler, pandas, openpyxl, reportlab |
| Frontend | React 19, Vite, Tailwind CSS v4, Recharts, react-router (HashRouter), jsPDF |
| Agent IA | Python, FastAPI, Groq (`qwen/qwen3.8-27b`), sentence-transformers (`multilingual-e5-small`), asyncpg |

## À savoir avant de commencer

- 🔒 **Les fichiers `.env` ne sont pas dans ce dépôt.** Mots de passe et clés d'API se saisissent à l'installation ; le guide explique où les obtenir et où les placer.
- 📝 **Gardez un carnet de configuration** pendant l'installation : plusieurs valeurs choisies à l'étape « base de données » sont réutilisées dans les autres parties (voir le [guide d'installation](06-guide-installation.md)).
- ⚠️ **Points de sécurité connus** : avant une mise en production, lisez la section « Sécurité » de l'[architecture globale](01-architecture-globale.md#7-sécurité--vue-densemble).
