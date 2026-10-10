# 🧩 Pages et composants

## 1. Composants réutilisables

| Composant | Rôle |
|---|---|
| `Layout` | Cadre des pages authentifiées : sidebar sticky (260 px), zone `<Outlet/>`, chatbot |
| `Sidebar` | Navigation filtrée par `user.role`, bascule de thème, menu utilisateur (initiales, Paramètres, Déconnexion) |
| `PrivateRoute` | Garde d'authentification |
| `Chatbot` | Widget « Assistant ERP » (voir `services-api.md`) |
| `KpiCard` | Carte KPI : icône + valeur + libellé + sous-texte (utilisée par le Dashboard) |
| `DateRangeFilter` | Filtre de période (voir ci-dessous) |
| `DownloadReportButton` / `ReportDownloadButton` | Export PDF (voir ci-dessous) |
| `MoroccoSuppliersMap` / `SalesCityMap` | Cartes SVG du Maroc |
| `CustomBarTooltip` | Tooltips Recharts : `StockCategoryTooltip`, `PurchasesCountTooltip`, `SupplierAmountTooltip` (non utilisés : redéfinis localement dans les pages) |
| Sélecteurs | `GlobalPeriodFilter`, `MonthYearSelector`, `SingleMonthSelector`, `YearSelector`, `PeriodSelector`, `DaysSelector` (non utilisés) |

### DateRangeFilter

Filtre de plage de dates (start → end) avec presets rapides : *Ce mois, Mois dernier, Cette année, Année dernière, Tout*. Utilisé en haut de chaque page de rapport et sur le Dashboard.

```jsx
<DateRangeFilter startDate={range.startDate} endDate={range.endDate} onChange={setRange} />
```

- Principe : chaque page gère **un état `{ startDate, endDate }` unique** passé à tous les appels filtrables ; les données « snapshot » (valeur actuelle du stock) ignorent ce filtre.
- Exporte aussi `useDefaultDateRange()` : du 1er du mois à aujourd'hui.
- Les pages **Ventes, Remboursements, Clients et Paiements** utilisent une **version locale** du filtre (avec un preset supplémentaire « Aujourd'hui »).
- ⚠️ Les dates sont produites via `toISOString()` (UTC), ce qui peut décaler d'un jour en UTC+1.

### DownloadReportButton (par `id`)

Capture le contenu de la page (`html2canvas-pro` + `jsPDF`, A4, `scale: 3`) et télécharge un PDF. Nécessite un `id` sur le conteneur racine.

```jsx
<div id="stock-report-content">
  ...
  <DownloadReportButton targetId="stock-report-content" fileName="rapport_stock" />
</div>
```

Il attend le rendu des graphiques et adapte le fond au thème. Utilisé par Stock, Achats, Fournisseurs, Devis.

### ReportDownloadButton (par `ref`)

Variante pilotée par `targetRef` (+ `fileName`), `scale: 2`, fond blanc, avec `import()` dynamique des libs PDF au clic. Utilisé par Ventes, Remboursements, Clients, Paiements.

### Cartes du Maroc

Cartes **SVG** (pas de bibliothèque cartographique) : contour national, **12 régions** et points de villes positionnés par projection lat/lng (bornes lat 20,6 → 36,3 ; lng −17,5 → −0,8).

| | `MoroccoSuppliersMap` | `SalesCityMap` |
|---|---|---|
| Props | `data`, `showRegions` | `data`, `showRegions`, `metric` (`"revenue"` par défaut), `title` |
| Données | fournisseurs par ville | `{ city, lat, lng, nb_sales, total_revenue, nb_clients }` |
| Rendu | taille du point ∝ nb de fournisseurs ; **échelle de couleurs à 5 paliers** ; légende ; tooltip au survol | taille/couleur selon la métrique ; tooltip |
| Sans données | jeu de données par défaut | idem |

## 2. Hooks

### `useFetch(fetchFn, deps = [])`

```jsx
const { data, loading, error, refetch } = useFetch(fetchFunction, dependencyArray);
```

Recharge à chaque changement de `deps` — utilisé partout pour réagir aux filtres. `error` est une chaîne (`err.message`).

### `useChartData(fetchFn, defaultGranularity = "month")`

Charge les données d'un graphique indépendamment avec sa propre granularité (jour/mois/année) et un drapeau `cancelled`. Retourne `{ data, loading, error, granularity, setGranularity }`. *Non utilisé actuellement.*

## 3. Pages

Les pages de rapport partagent la même structure : en-tête (titre + période), filtre de dates, bouton d'export PDF, KPIs, graphiques Recharts, listes/tableaux, et gestion des états chargement/erreur/vide.

### Authentification

- **Login** : écran scindé (formulaire / visuel), affichage du mot de passe, liens « Mot de passe oublié » et « S'inscrire », bascule de thème. La case « Se souvenir de moi » est sans effet.
- **Register** : nom, e-mail, mot de passe + confirmation, validations côté client.
- **ForgotPw** : e-mail + nouveau mot de passe, écran de succès puis redirection vers `/login` après 2 s.

### Dashboard (`admin`)

KPIs globaux (CA du mois, clients, produits, meilleur client) + KPIs filtrables par domaine (valeur du stock, montant acheté, concentration top 3 fournisseurs, taux de conversion des devis) + graphique d'évolution CA/ventes sur 6 mois (double axe) + alertes transversales (stock critique, devis à relancer) + top 3 clients et produits du mois.

Données : `getDashboardData(startDate, endDate)` agrège `/dashboard/overview`, `/dashboard/revenue-trend`, `/dashboard/sales-count-trend`, `/dashboard/best-clients`, `/dashboard/top-products`, puis les 4 endpoints `/report` (stock, achats, fournisseurs, devis).

### RoleWelcome (non-admin, `/dashboard`)

Carte d'accueil par rôle (`ROLE_CONFIG` : icône, titre « Espace … », description, couleur, lien vers le rapport) ; message générique si le rôle n'est pas configuré.

### StockReport

- KPI cards (valeur du stock, unités, produits, alertes) — **snapshots**
- Stock dormant / produits à arrêter — filtrables par fenêtre glissante
- Bar chart stock par catégorie (label + tooltip enrichi)
- Donut répartition des alertes (`critical` / `low` / `warning`)
- Bar chart mouvements de stock (filtré par la plage globale)
- Listes : alertes de stock, produits à rotation rapide
- Tables : produits les moins/plus chers, fourchette de prix par catégorie

### PurchasesReport

- KPI cards : montant total, nb commandes, moyenne produits/commande, produits achetés 1 fois
- Line chart évolution du montant acheté (filtré par dates)
- Bar chart nb commandes (tooltip enrichi : montant, moyenne, fournisseur dominant)
- Donut répartition par catégorie
- Listes : produits les plus achetés, dernières commandes
- Section variations de prix (plus fortes hausses/baisses)

### SuppliersReport

- KPI cards : nb fournisseurs, inactifs 30 j+, concentration top 3, sans produit associé
- Bar chart montant acheté par fournisseur (tooltip enrichi)
- **Carte** `MoroccoSuppliersMap`
- Top 5 fournisseurs, liste des fournisseurs inactifs
- Section « Économies potentielles » (alternatives moins chères) **chargée à la demande** (bouton « Analyser »), car coûteuse en calcul

### DevisReport

- KPI cards : nb devis, montant total, taux de conversion, pipeline commercial
- Cartes relances/expiration (filtrables indépendamment par nombre de jours)
- Donut répartition par statut (`draft`, `sent`, `accepted`, `rejected`)
- Line chart évolution du taux de conversion
- Délai moyen de réponse, comparaison montants acceptés/refusés
- Top devis par montant, clients les plus sollicités, produits les plus proposés

### SalesReport *(nouveau par rapport à l'ancienne doc)*

KPIs : chiffre d'affaires, nombre de commandes, panier moyen, taux de remboursement. Sections : performance par catégorie, répartition par statut (donut), produits dormants (90 jours), **carte des ventes par ville**, remboursements par produit et par client, **segmentation RFM**.

### refundsReport *(nouveau)*

KPIs : nb de remboursements, montant total, montant moyen, taux de retour global (unités remboursées / vendues), croissance vs période précédente. Graphiques : évolution mensuelle, répartition par raison, produits les plus remboursés, taux de retour par produit. Listes : clients concernés, produits à taux élevé (seuil 10 %).

### ClientsReport *(nouveau)*

KPIs : total clients, nouveaux, CA total, panier moyen, taux de réachat, churn. Sections : évolution des clients actifs, segments RFM, clients inactifs et à relancer, carte géographique, top clients, distribution de la valeur vie client, santé client.

### PaymentsReport *(nouveau)*

Vue d'ensemble, répartition par statut et par méthode, tendance d'encaissement, principaux débiteurs, impayés (`overdue`), clients en retard récurrent, cash encaissé.

### SettingsPage

Trois cartes : **Profil** (nom, e-mail), **Mot de passe** (actuel / nouveau / confirmation, ≥ 8 caractères), **Langue** (`fr`, `en`, `ar`). La langue est enregistrée côté serveur mais l'interface n'est pas internationalisée.

### AdminUsersPage (`admin`)

Tableau des utilisateurs, changement de rôle en ligne, suppression avec confirmation, modale de création (prénom, nom, e-mail, rôle). Le mot de passe temporaire est **généré par le serveur** et affiché une seule fois.
