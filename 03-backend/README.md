# 📘 Documentation Backend — Nextact Reporting API (FastAPI)

## 1. Vue d'ensemble

Backend REST API organisé par **domaine métier** (Stock, Purchases, Suppliers, Devis, Sales, Refunds, Clients, Payments), suivant le **pattern Repository**. Chaque domaine de rapport expose un endpoint `/report` unifié (filtrable par plage de dates) en plus d'endpoints individuels pour des besoins ponctuels.

Le backend sert trois consommateurs :

1. **Le frontend web** : tableaux de bord et pages de rapport par domaine.
2. **Un agent IA conversationnel** (service séparé) qui interroge les données via des routes « outils » et peut générer des exports xlsx / csv / pdf.
3. **Des traitements automatiques** : alertes métier envoyées par email, nettoyage des exports expirés.

Le backend **lit** une base PostgreSQL existante (vues et fonctions du schéma `agent_reporting`). Il n'écrit que dans trois tables : `users`, `file_exports`, `alert_notifications_sent`.

## 2. Stack technique

| Domaine | Technologie |
|---|---|
| Framework web | FastAPI, Uvicorn |
| Base de données | PostgreSQL (`psycopg2`) |
| Accès aux données | SQLAlchemy 2.x (ORM pour `users` / `file_exports`, SQL brut `text()` pour le reporting) |
| Configuration | `pydantic-settings` + `.env` |
| Authentification | JWT (`python-jose`), mots de passe `bcrypt` (`passlib`) |
| Tâches planifiées | APScheduler (`BackgroundScheduler`) |
| Emails | `smtplib` (SMTP + STARTTLS) |
| Exports | `pandas`, `openpyxl` (xlsx), `reportlab` (pdf) |
| Client HTTP | `httpx` (relais vers l'agent IA) |

## 3. Sommaire de la documentation

| Fichier | Contenu |
|---|---|
| [`structure-projet.md`](structure-projet.md) | Arborescence, rôle de chaque dossier, câblage de `main.py`, cycle d'une requête |
| [`configuration.md`](configuration.md) | Variables d'environnement, installation, lancement |
| [`base-de-donnees.md`](base-de-donnees.md) | Connexion, tables gérées, vues et fonctions SQL consommées |
| [`repositories.md`](repositories.md) | Pattern Repository + méthodes de chaque repository par domaine |
| [`routes-api.md`](routes-api.md) | Référence complète des routes (frontend, outils agent, chat, exports) |
| [`authentification.md`](authentification.md) | JWT, rôles, matrice de permissions, gestion des comptes, compte de service |
| [`services.md`](services.md) | Couche services : RFM, remboursements, emails, dashboard |
| [`agent-ia.md`](agent-ia.md) | Intégration avec l'agent IA (chat, outils, contrôle d'accès) |
| [`exports-rapports.md`](exports-rapports.md) | Génération de fichiers xlsx / csv / pdf, registre de rapports |
| [`jobs-planifies.md`](jobs-planifies.md) | Alertes email et nettoyage des exports |
| [`points-attention.md`](points-attention.md) | Anomalies observées dans le code et pistes d'amélioration |

## 4. Domaines couverts

| Domaine | Rapport frontend | Repository | Préfixes « outils » agent |
|---|---|---|---|
| Dashboard | `/api/dashboard/*` | `DashboardRepository` | — |
| Stock | `/api/stock/report` | `StockAnalyticsRepository` | `stock-consultation`, `stock-traitement`, `stock-recommandation` |
| Purchases | `/api/purchases/report` | `PurchasesAnalyticsRepository` | `purchases-consultation`, `purchases-traitement`, `purchases-recommandation` |
| Suppliers | `/api/suppliers/report` | `SuppliersAnalyticsRepository` | (partagés avec Purchases) |
| Devis | `/api/devis/report` | `DevisAnalyticsRepository` | `devis-consultation`, `devis-traitement`, `devis-recommandation` |
| Sales | `/api/sales/*` | `SalesRepository` | `sales-consultation`, `sales-traitement`, `sales-recommandation` |
| Refunds | `/api/refunds/*` | `RefundsRepository` | `refunds-consultation`, `refunds-traitement`, `refunds-recommandation` |
| Clients | `/api/clients/*` | `ClientsAnalyticsRepository` | `clients-consultation`, `clients-traitement`, `clients-recommandation` |
| Payments | `/api/payments/*` | `PaymentsAnalyticsRepository` | `payments-consultation`, `payments-traitement`, `payments-recommandation` |

## 5. Démarrage rapide

```bash
python -m venv venv && source venv/bin/activate
pip install -r requirements.txt pandas openpyxl reportlab
# créer le fichier .env (voir configuration.md)
uvicorn main:app --reload --port 8000
```

Swagger : `http://localhost:8000/docs`.

## 6. Ce qu'il faut savoir avant de commencer

- Les routers `dashboard`, `sales` et `refunds` (frontend) sont **publics** ; tous les autres routers de rapport exigent un JWT mais **aucun ne contrôle le rôle**. Voir [`points-attention.md`](points-attention.md).
- Plusieurs routes de gestion de compte sont cassées en l'état (`/register`, `/user/profile`, `/user/language`).
- Le code source de l'agent IA, les scripts SQL (vues et fonctions) et le frontend ne font pas partie de ce dépôt.
