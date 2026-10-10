# 🔌 Services API — contrat avec le backend

## 1. Wrapper HTTP (`services/api.js`)

- Base : `import.meta.env.VITE_API_URL || 'http://localhost:8000/api'`.
- En-têtes : `Content-Type: application/json` + `Authorization: Bearer <token>` si un token est présent.
- **401** : supprime le token, redirige vers `/#/login` et lève « Session expirée, veuillez vous reconnecter ».
- Autre erreur HTTP : `Error` avec `detail` du corps JSON (format FastAPI) ou `Erreur API: <status>`.
- Succès : `res.json()`.

## 2. Pattern des services

Chaque service suit le même pattern : **une fonction par endpoint**, via `request()`.

```javascript
// stockService.js
import request from './api';

export const getStockReport = (startDate, endDate) =>
  request(`/stock/report?start_date=${startDate}&end_date=${endDate}`);
```

Les services plus récents (ventes, remboursements, paiements, clients, fournisseurs) utilisent un helper `buildQuery(params)` qui ignore les valeurs vides, `null` ou `undefined` (dupliqué dans plusieurs fichiers). Certains agrègent plusieurs endpoints avec `Promise.all` (`getDashboardData`, `getSalesReport`, `getRefundsReport`).

## 3. Endpoints par service

Chemins relatifs à `VITE_API_URL`.

| Service | Fichier | Endpoints |
|---|---|---|
| Auth | `authService.js` | `POST /auth/login`, `POST /auth/register`, `GET /auth/me`, `POST /user/reset-password` |
| Paramètres | `settingsService.js` | `PUT /user/profile`, `PUT /user/password`, `PUT /user/language` |
| Admin | `adminService.js` | `GET/POST /admin/users`, `PATCH /admin/users/{id}/role`, `DELETE /admin/users/{id}` |
| Dashboard | `dashboardService.js` | `/dashboard/overview`, `/dashboard/revenue-trend?months=6`, `/dashboard/sales-count-trend?months=6`, `/dashboard/best-clients`, `/dashboard/top-products?limit=5` + `/stock/report`, `/purchases/report`, `/suppliers/report`, `/devis/report` |
| Stock | `stockService.js` | `/stock/report` |
| Purchases | `purchasesService.js` | `/purchases/report` |
| Suppliers | `suppliersService.js` | `/suppliers/report`, `/suppliers/cheaper-alternatives` (+ `limit`) |
| Devis | `devisService.js` | `/devis/report` |
| Ventes | `salesService.js` | `/sales/rfm-segmentation`, `/sales/refund-rate-by-product`, `/sales/category-performance`, `/sales/dormant-products?days=90`, `/sales/refund-rate-trend`, `/sales/revenue-by-status`, `/sales/refund-rate-by-client`, `/sales/sales-by-city` |
| Remboursements | `refundsService.js` | `/refunds/summary`, `/by-reason`, `/trend?months=12`, `/top-products`, `/top-clients`, `/refund-rate`, `/refund-rate-by-product`, `/high-return-products?threshold_pct=10`, `/growth` |
| Paiements | `paymentsService.js` | `/payments/summary`, `/overview`, `/status-distribution`, `/method-distribution`, `/collection-trend`, `/top-debtors`, `/overdue`, `/recurring-late-clients`, `/cash-collected` |
| Clients | `clientsService.js` | `/clients/kpi-summary` (utilisé par la page) + ~25 fonctions unitaires : count, new, sales-status, global-revenue, top, repeat, inactive, to-recontact, rfm, rfm-summary, evolution, repeat-rate, avg-time-between-orders, churn-rate, lifetime-value-distribution, health, by-city, `{id}`, high-refund-rate, without-orders |
| Chatbot | *(direct dans `Chatbot.jsx`)* | `POST /chat`, `GET /reports/download/<fichier>` |

Les rapports filtrables acceptent `start_date` et `end_date` (format `YYYY-MM-DD`).

## 4. Clés de réponse attendues (extraits)

| Endpoint | Clés utilisées par le frontend |
|---|---|
| `/auth/login` | `access_token` |
| `/auth/me` | `name`, `email`, `role`, `first_name`, `language` |
| `/dashboard/overview` | `revenue_this_month`, `sales_this_month`, `total_clients`, `total_suppliers`, `total_products`, `total_categories`, `best_client_this_month { client_name, total_spent }` |
| `/payments/*` (agrégé) | `overview_kpis`, `status_distribution`, `method_distribution`, `collection_trend`, `top_debtor_clients`, `overdue_kpi`, `recurring_late_clients_kpi`, `cash_collected_kpi` |
| `/refunds/*` | `summary { nb_refunds, total_refunded_amount, avg_refund_amount }`, `refund_rate { refund_rate_pct, quantity_refunded, quantity_sold }`, `growth { growth_pct, current_amount, previous_amount }` |
| `/clients/kpi-summary` | `clients_count`, `new_clients_count`, `global_revenue`, `sales_status_summary`, `top_clients`, `repeat_clients_summary`, `inactive_clients_summary`, `to_recontact_summary`, `rfm_summary`, `clients_evolution`, `repeat_rate`, `avg_time_between_orders` |
| `/sales/sales-by-city`, `/suppliers/*` (carte) | `{ city, lat, lng, nb_sales, total_revenue, nb_clients }` / fournisseurs par ville avec coordonnées |

## 5. Authentification et sécurité

- Login → `AuthContext.login()` → `authService.loginUser()` → `POST /auth/login`.
- Token stocké dans `localStorage` (clé `token`), attaché automatiquement par `api.js`.
- Réponse 401 → déconnexion automatique et redirection vers `/login`.
- ⚠️ Le contrôle d'accès par rôle côté frontend est cosmétique : **le backend doit vérifier le rôle sur chaque endpoint**.
- ⚠️ `POST /user/reset-password` n'envoie ni jeton ni ancien mot de passe : à sécuriser côté backend.

## 6. Chatbot (`Chatbot.jsx`)

Widget flottant (bouton en bas à droite, panneau 350×500 px agrandissable jusqu'à 600×750) qui envoie les messages à `POST /chat`, lequel relaie vers l'agent IA.

**Requête** : `{ message, session_id }` — `session_id` est un `crypto.randomUUID()` créé au montage et renouvelé par « Nouvelle conversation ».

**Réponse attendue** : `{ reply, generated_sql?, download_url? }`

- `reply` est rendu en **Markdown** (titres, listes, tableaux GFM, code).
- `generated_sql` : bouton « Voir le SQL ».
- `download_url` : bouton « Télécharger le fichier ». Un lien direct n'enverrait pas le JWT, donc le fichier est récupéré par `fetch` authentifié puis téléchargé depuis un blob ; les liens Markdown contenant `/reports/download/` sont interceptés de la même manière.
- Erreurs : 401 → retour au login ; 403 → nouvelle session + message ; autre → « Je n'ai pas pu contacter le serveur ».
- Suggestions rapides : « Produits en rupture de stock », « Chiffre d'affaires du mois », « Mes meilleurs clients ».

> ⚠️ L'URL de l'API est **codée en dur** (`http://localhost:8000/api`) dans ce composant : elle ne suit pas `VITE_API_URL`.
>
> 💡 Amélioration possible (reprise de l'ancienne doc) : si le backend renvoie des données structurées (`raw_data`), afficher un mini-tableau. Ce champ n'est pas utilisé aujourd'hui.
