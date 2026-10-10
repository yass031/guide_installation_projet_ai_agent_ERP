# 🗂️ Structure du projet

## 1. Arborescence

```
nextact-frontend-only/
├── index.html                   # Point d'entrée HTML (monte #root, charge src/main.jsx)
├── vite.config.js               # Plugin React
├── postcss.config.js            # Plugin @tailwindcss/postcss
├── eslint.config.js / .oxlintrc.json
├── public/                      # favicon.svg, icons.svg
└── src/
    ├── main.jsx                 # Rendu racine (StrictMode)
    ├── App.jsx                  # Providers + table des routes
    ├── index.css                # Tailwind, police Inter, mode sombre, scrollbars, focus
    ├── assets/NextAct.jpg       # Logo
    │
    ├── components/
    │   ├── Layout.jsx                # Layout authentifié (sidebar + contenu + chatbot)
    │   ├── Sidebar.jsx               # Navigation filtrée par rôle
    │   ├── PrivateRoute.jsx          # Garde d'authentification
    │   ├── Chatbot.jsx               # Widget chat agent IA
    │   ├── KpiCard.jsx               # Carte KPI réutilisable
    │   ├── DateRangeFilter.jsx       # Filtre de période (start/end + presets)
    │   ├── DownloadReportButton.jsx  # Export PDF par id de conteneur (targetId)
    │   ├── ReportDownloadButton.jsx  # Export PDF par ref React (targetRef)
    │   ├── MoroccoSuppliersMap.jsx   # Carte SVG : fournisseurs par ville
    │   ├── SalesCityMap.jsx          # Carte SVG : ventes par ville
    │   ├── CustomBarTooltip.jsx      # Tooltips enrichis (non utilisé)
    │   └── GlobalPeriodFilter / MonthYearSelector / SingleMonthSelector /
    │       YearSelector / PeriodSelector / DaysSelector   # (non utilisés)
    │
    ├── context/
    │   ├── AuthContext.jsx      # État d'authentification global
    │   ├── ThemeContext.jsx     # Thème clair/sombre
    │   └── RoleContext.jsx      # RoleRoute (garde par rôle)
    │
    ├── hooks/
    │   ├── useFetch.js          # Hook générique de fetching
    │   └── useChartData.js      # Fetch d'un graphique + granularité (non utilisé)
    │
    ├── services/
    │   ├── api.js               # Wrapper fetch (JWT, gestion 401)
    │   ├── authService.js       ├── settingsService.js    ├── adminService.js
    │   ├── dashboardService.js  ├── stockService.js       ├── purchasesService.js
    │   ├── suppliersService.js  ├── devisService.js       ├── salesService.js
    │   └── refundsService.js    ├── paymentsService.js    └── clientsService.js
    │
    ├── pages/                   # 15 pages (voir pages-et-composants.md)
    │   ├── Login · Register · ForgotPw
    │   ├── Dashboard · RoleWelcome
    │   ├── StockReport · PurchasesReport · SuppliersReport · DevisReport
    │   ├── SalesReport · refundsReport · ClientsReport · PaymentsReport
    │   └── SettingsPage · AdminUsersPage
    │
    ├── utils/format.js          # formatDH, formatDate (non utilisés)
    └── data/views_column/stock_views.js   # Fichier vide
```

## 2. Hiérarchie des composants

```
<StrictMode>
 └ <App>
    └ <ThemeProvider>            thème clair/sombre (classe .dark sur <html>)
       └ <AuthProvider>          utilisateur, token, login/logout
          └ <HashRouter>
             ├ /login, /register, /forgot-pw        (publiques)
             └ <PrivateRoute><Layout/></PrivateRoute>
                ├ <Sidebar/>      navigation filtrée par rôle
                ├ <main><Outlet/></main>   page courante
                └ <Chatbot/>      widget flottant
```

## 3. Flux de données

1. Une **page** appelle `useFetch(() => serviceFn(params), [deps])`.
2. Le **service** construit l'URL (et la query string) puis appelle `request()` de `services/api.js`.
3. `request()` ajoute `Authorization: Bearer <token>` (lu dans `localStorage`), gère les erreurs et renvoie le JSON.
4. Principe du filtre : chaque page gère **un état `{ startDate, endDate }` unique**, passé à tous les appels filtrables. Les données « snapshot » (ex. valeur actuelle du stock) ignorent ce filtre.
5. Certains services agrègent **plusieurs endpoints en parallèle** (`Promise.all`) : dashboard (9 requêtes), ventes, remboursements, clients.
6. `useFetch` recharge automatiquement dès que le tableau de dépendances change.

## 4. Authentification

```
Login.jsx → AuthContext.login() → authService.loginUser() → POST /auth/login
```

- Le token (`access_token`) est stocké dans `localStorage` (clé `token`).
- `GET /auth/me` alimente `user` (`name`, `email`, `role`, `first_name`, `language`) au login et à chaque rechargement (`loading = true` pendant ce temps).
- `api.js` attache `Authorization: Bearer <token>` à chaque requête.
- **Réponse 401** → suppression du token, redirection vers `/#/login`.
- **Inscription** : `POST /auth/register` ; validations côté client (mots de passe identiques, 8 caractères minimum).
- **Mot de passe oublié** : `POST /user/reset-password` `{ email, new_password }`.

### Gardes de route

| Garde | Comportement |
|---|---|
| `PrivateRoute` | Redirige vers `/login` sans utilisateur |
| `Layout` | Second contrôle (`isAuthenticated`) + spinner |
| `RoleRoute` | Redirige vers `/dashboard` si le rôle n'est pas dans `allowedRoles` |
| `/dashboard` | `DashboardOrWelcome` : **Dashboard** pour `admin`, **RoleWelcome** pour les autres |

> Ces gardes sont de l'ergonomie : la sécurité réelle doit être appliquée par le backend sur chaque endpoint.

## 5. Matrice des routes et rôles

| Route | Page | Rôles |
|---|---|---|
| `/dashboard` | Dashboard / RoleWelcome | tous (selon rôle) |
| `/rapport-ventes` | Rapport des ventes | `admin`, `user_sales` |
| `/rapport-stock` | Rapport de stock | `admin`, `user_stock` |
| `/rapport-purchases` | Rapport des achats | `admin`, `user_purchases` |
| `/rapport-suppliers` | Rapport fournisseurs | `admin`, `user_suppliers` |
| `/rapport-remboursements` | Rapport des remboursements | `admin`, `user_refunds` |
| `/clients-report` | Rapport clients | `admin`, `user_clients` |
| `/rapport-paiements` | Rapport des paiements | `admin`, `user_payments` |
| `/rapport-devis` | Rapport des devis | `admin`, `user_devis` |
| `/settings` | Paramètres | tout utilisateur connecté |
| `/admin/users` | Gestion des utilisateurs | `admin` |
| `/login`, `/register`, `/forgot-pw` | Auth | publiques |
| `/` et `*` | — | redirigent vers `/login` |

**Rôles** : `admin`, `user_stock`, `user_sales`, `user_purchases`, `user_suppliers`, `user_devis`, `user_payments`, `user_clients`, `user_refunds`.

## 6. Theming

Mode clair/sombre géré via une classe `dark` sur `<html>`, pilotée par `ThemeContext` (variante Tailwind `@custom-variant dark (&:where(.dark, .dark *))`). Toutes les classes Tailwind sont doublées avec le variant `dark:`. Le thème **n'est pas persisté** (retour au clair à chaque rechargement).

Design : police Inter, palette slate + bleu `#2563eb`, cartes `rounded-xl`, icônes lucide. Montants en MAD (`Intl.NumberFormat('fr-MA')`) ou « … DH ».

## 7. Export PDF

`html2canvas-pro` (et non `html2canvas`, qui ne lit pas les couleurs `oklch()` de Tailwind v4) capture le DOM, puis `jsPDF` découpe en pages A4. Fichier : `<nom>_<date>.pdf`. Deux composants coexistent (voir `pages-et-composants.md`).

## 8. Conventions observées

- Textes d'interface et commentaires en **français** ; identifiants de code en anglais.
- Alias snake_case → camelCase à la destructuration des réponses API.
- Chaque page de rapport a un conteneur d'export identifié par un `id` ou une `ref`.
- États *chargement / erreur / vide* gérés dans chaque page.

## 9. Ajouter un nouveau rapport

1. **Service** : `services/xxxService.js` (fonction appelant `request('/xxx/report?...')`).
2. **Page** : `pages/XxxReport.jsx` sur le modèle de `PurchasesReport.jsx` : `useState(useDefaultDateRange())`, `useFetch`, KPIs, graphiques, `<DownloadReportButton targetId="xxx-report-content" fileName="rapport_xxx" />` et un `<div id="xxx-report-content">` racine.
3. **Route** : dans `App.jsx`, `<RoleRoute allowedRoles={["admin","user_xxx"]}>`.
4. **Navigation** : entrée dans `Sidebar.navItems`.
5. **Rôle** : déclarer `user_xxx` dans `RoleWelcome.ROLE_CONFIG` et `AdminUsersPage.ROLES`, et le créer côté backend.
