# Structure du projet

## 1. Arborescence

```
nextact-backend-only/
├── main.py                          # Création de l'app, CORS, routers, scheduler
├── requirements.txt                 # Dépendances (encodé en UTF-16, incomplet)
├── .env                             # Configuration locale (non versionné)
├── test.py                          # Script manuel : déclenche le job d'alertes
│
├── core/
│   ├── config.py                    # Settings pydantic (lit .env)
│   ├── database.py                  # Engine, SessionLocal, Base, get_db()
│   ├── security.py                  # hash/verify mot de passe, création/décodage JWT
│   ├── dependencies.py              # get_current_user, require_role
│   ├── role_permissions.py          # Matrice rôles → pages, tools, vues, exports
│   ├── moroccan_cities.py           # MOROCCAN_CITIES, CITY_COORDINATES
│   └── generate_hash.py             # Script utilitaire : hash bcrypt
│
├── models/
│   ├── user.py                      # User + enum UserRole (table users)
│   ├── file_export.py               # FileExport (agent_reporting.file_exports)
│   └── views.py                     # (vide)
│
├── schemas/                         # Schémas Pydantic
│   ├── auth.py                      # LoginRequest, Token, TokenData
│   ├── user.py                      # Profil, mot de passe, création/liste d'utilisateurs
│   ├── dashboard.py                 # Réponses du dashboard
│   ├── sales.py                     # ⚠️ copie de api/sales.py (mal rangé)
│   └── stock.py                     # (vide)
│
├── api/                             # Routers FastAPI
│   ├── auth.py                      # /api/auth/login, /register, /me
│   ├── user.py                      # /api/user/*
│   ├── admin_users.py               # /api/admin/users (admin)
│   ├── chat.py                      # /api/chat (relais vers l'agent IA)
│   ├── dashboard.py                 # /api/dashboard/*
│   ├── sales.py                     # /api/sales/*
│   ├── stock.py                     # /api/stock/report
│   ├── purchases.py                 # /api/purchases/report
│   ├── suppliers.py                 # /api/suppliers/*
│   ├── devis.py                     # /api/devis/report
│   ├── refunds.py                   # /api/refunds/*
│   ├── clients_analytics.py         # /api/clients/*
│   ├── payments_analytics.py        # /api/payments/*
│   ├── sales_api_tools/             # consultation.py · traitement.py · recommandation.py
│   ├── stock_api_tools/             # consultation.py · traitements.py · recommandation.py
│   ├── refunds_api_tools/           # consultation.py · traitement.py · recommandation.py
│   ├── clients_api_tools/           # consultation.py · traitement.py · recommandation.py
│   ├── payments_api_tools/          # consultation.py · traitement.py · recommandation.py
│   ├── purchases_suppliers_api_tools/  # purchases_consultation/_traitement/_recommandation.py
│   ├── devis_api_tools/             # devis_consultation/_traitement/_recommandation.py
│   └── reports_api_tools/
│       └── generation.py            # /api/reports/export · download · my-exports
│
├── repositories/                    # Requêtes SQL par domaine
│   ├── stock_repository.py          ├── purchases_repository.py
│   ├── suppliers_repository.py      ├── devis_repository.py
│   ├── sales_repository.py          ├── refunds_repository.py
│   ├── clients_repository.py        ├── payments_repository.py
│   ├── dashboard_repository.py      ├── user_repository.py
│   ├── admin_user_repository.py     ├── alert_notification_repository.py
│   └── test.py                      # Script manuel
│
├── services/
│   ├── auth_service.py              # Login, mot de passe, profil
│   ├── dashboard_service.py         # Mise en forme des KPI dashboard
│   ├── sales_service.py             # Segmentation RFM
│   ├── refunds_service.py           # Croissance des remboursements
│   ├── report_registry.py           # Registre report_type → repository
│   ├── file_generator.py            # Écriture xlsx / csv / pdf
│   ├── cleanup_job.py               # Suppression des exports expirés
│   ├── email_service.py             # Emails d'alertes (SMTP)
│   ├── pw_email_service.py          # Email de bienvenue (mot de passe temporaire)
│   ├── test_smtp.py                 # Script manuel
│   └── stock_service.py · purchases.py · suppliers.py   # (vides)
│
├── jobs/
│   └── alert_notifier_job.py        # Détection + envoi des alertes
│
└── temp_exports/                    # Fichiers exportés (créé à l'exécution)
```

## 2. Rôle des dossiers

| Dossier | Responsabilité |
|---|---|
| `core/` | Transversal : configuration, base, sécurité, permissions |
| `models/` | Mapping ORM — uniquement `users` et `file_exports` |
| `schemas/` | Validation des entrées/sorties (Pydantic) |
| `api/` | Routers : validation des paramètres, injection des dépendances, appel du repository ou du service |
| `repositories/` | Tout le SQL. Aucune logique HTTP |
| `services/` | Logique métier qui dépasse la simple requête, emails, génération de fichiers |
| `jobs/` | Traitements planifiés |

## 3. Cycle d'une requête

```
Client ──HTTP──▶ FastAPI
                  │  1. CORS (http://localhost:5173)
                  │  2. Router (api/…)   ── dependencies=[Depends(get_current_user)] ──▶ JWT valide ?
                  │  3. Validation Query/Body (dates, limites…)
                  │  4. Depends(get_db) → Session SQLAlchemy
                  │  5. Repository(db).méthode(...)   → SQL sur vues/fonctions agent_reporting
                  │  6. (optionnel) Service : mise en forme / calculs Python
                  ▼
                 JSON
```

Chaque router définit sa propre fabrique de dépendance, par exemple :

```python
def get_stock_repo(db: Session = Depends(get_db)) -> StockAnalyticsRepository:
    return StockAnalyticsRepository(db)
```

## 4. Câblage de `main.py`

- **Lifespan** : démarre un `BackgroundScheduler` avec deux jobs (alertes, nettoyage) et l'arrête à l'extinction. Voir `jobs-planifies.md`.
- **CORS** : seule l'origine `http://localhost:5173` (Vite) est autorisée, avec credentials.
- **Routers** :
  - Frontend : `dashboard`, `sales`, `stock`, `chat`, `refunds`, `clients_analytics`, `payments_analytics`, `purchases`, `suppliers`, `devis`, tous sous `/api`.
  - Agent IA : 21 routers `*_api_tools` sous `/api`.
  - `auth`, `user`, `admin_users` : préfixe complet défini dans le router.
  - `reports_router` sous `/api`.

> ⚠️ `admin_users.router` est inclus **deux fois** et les imports du haut de `main.py` sont dupliqués (voir `points-attention.md`).

## 5. Conventions

- Fichiers `*_api_tools/` : un fichier par famille — `consultation` (recherche/détail), `traitement` (calculs), `recommandation` (suggestions et alertes). Elles correspondent aux classes `*ConsultationRepository`, `*TraitementRepository`, `*RecommandationRepository`.
- Les rapports des pages frontend utilisent les classes `*AnalyticsRepository`.
- Les routers de rapport protègent toutes leurs routes avec `dependencies=[Depends(get_current_user)]` au niveau du `APIRouter`.
