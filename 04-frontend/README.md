# 🖥️ 04 — Frontend (React)

> Documentation du client web **NextAct / ERPilot** : une SPA React (Vite) qui consomme l'API backend et propose des tableaux de bord et rapports par domaine métier, avec contrôle d'accès par rôle et assistant IA.

## Sommaire de ce dossier

| Fichier | Contenu |
|---|---|
| [`README.md`](./README.md) | Vue d'ensemble, stack, installation, scripts, déploiement, points d'attention |
| [`structure-projet.md`](./structure-projet.md) | Arborescence, architecture, flux de données, authentification, rôles, thème, conventions |
| [`pages-et-composants.md`](./pages-et-composants.md) | Détail des 15 pages, des composants réutilisables et des hooks |
| [`services-api.md`](./services-api.md) | Couche services : contrat avec le backend, endpoints, chatbot |

## 1. Vue d'ensemble

Dashboard React (Vite) consommant l'API backend. L'application est organisée autour de :

- un **Dashboard global** (réservé à l'`admin`) ;
- **8 rapports détaillés** par domaine : Stock, Achats (Purchases), Fournisseurs (Suppliers), Devis, **Ventes, Remboursements, Clients, Paiements** ;
- des pages **Paramètres** et **Gestion des utilisateurs** (admin) ;
- un **filtre de plage de dates** (start/end + presets) sur chaque rapport ;
- un **assistant IA** (chatbot flottant), l'**export PDF** des rapports et des **cartes du Maroc** (ventes et fournisseurs par ville) ;
- un thème clair/sombre, la devise **MAD (DH)** et une interface en **français**.

Le frontend **n'embarque aucune donnée** : tout vient de l'API (JWT). Il ne filtre ni ne trie les données lui-même : les périodes sont envoyées au backend en query params.

## 2. Stack technique

| Domaine | Technologie | Version |
|---|---|---|
| UI | React | ^19.2.7 |
| Build / dev server | Vite + `@vitejs/plugin-react` | ^8.1.1 / ^6.0.3 |
| Routage | `react-router-dom` (**HashRouter**) | ^7.18.1 |
| Style | Tailwind CSS v4 (`@tailwindcss/postcss`) | ^4.3.3 |
| Graphiques | Recharts | ^3.10.0 |
| Icônes | lucide-react | ^1.25.0 |
| Markdown (chatbot) | react-markdown + remark-gfm | ^10.1.0 / ^4.0.1 |
| Export PDF | jspdf + html2canvas-pro | ^4.2.1 / ^2.4.5 |
| Qualité | ESLint 10, config oxlint | — |

Projet en **JavaScript/JSX** (pas de TypeScript), `"type": "module"`.

## 3. Démarrage rapide

**Prérequis** : Node.js récent (18+), un backend accessible (par défaut `http://localhost:8000/api`).

```bash
npm install
echo "VITE_API_URL=http://localhost:8000/api" > .env
npm run dev          # http://localhost:5173
```

| Variable | Rôle | Défaut |
|---|---|---|
| `VITE_API_URL` | URL de base de l'API REST | `http://localhost:8000/api` |

> `.env` est dans `.gitignore`. Le README d'origine mentionne un `.env.example` qui n'est pas dans l'archive.

| Script | Action |
|---|---|
| `npm run dev` | Serveur de développement Vite |
| `npm run build` | Build de production (`dist/`) |
| `npm run preview` | Prévisualisation locale du build |
| `npm run lint` | ESLint |

**Déploiement** : le build produit des fichiers statiques. Grâce au **HashRouter** (`/#/dashboard`), aucune règle de réécriture d'URL n'est nécessaire. `VITE_API_URL` doit être défini **au moment du build** et le backend doit autoriser l'origine du frontend (CORS).

## 4. Compatibilité avec l'ancienne documentation

L'ancienne documentation fournie a été **comparée au code réel** de l'archive et intégrée lorsqu'elle était compatible.

| Élément de l'ancienne doc | Statut | Détail |
|---|---|---|
| Stack Vite + React, wrapper `api.js` (JWT, 401) | ✅ Conservé | Conforme au code |
| `DateRangeFilter` (props `startDate`, `endDate`, `onChange`, presets) | ✅ Conservé | Conforme ; exporte aussi `useDefaultDateRange()` |
| `DownloadReportButton` (`targetId`, `fileName`) | ✅ Conservé | Utilisé par Stock, Achats, Fournisseurs, Devis, Dashboard |
| Pattern des services (une fonction par endpoint via `request()`) | ✅ Conservé | Conforme |
| Pages Dashboard, Stock, Purchases, Suppliers, Devis et leur contenu | ✅ Conservé | Complété (voir `pages-et-composants.md`) |
| Auth (login → token `localStorage` → Bearer → 401 → `/login`) | ✅ Conservé | Complété (rôles, inscription, mot de passe oublié) |
| Thème clair/sombre (`dark` sur `<html>`) | ✅ Conservé | Précision : non persisté |
| `useFetch(fetchFn, deps)` | ✅ Conservé | Retourne aussi `refetch` |
| « 5 pages principales » | ⚠️ Mis à jour | Le projet compte **15 pages** dont **8 rapports** (ajout de Ventes, Remboursements, Clients, Paiements) |
| Arborescence `src/` | ⚠️ Mis à jour | Ajout de `RoleContext`, `RoleWelcome`, `PrivateRoute`, `SalesCityMap`, `ReportDownloadButton`, sélecteurs, `useChartData`, 6 services supplémentaires |
| `MoroccoSuppliersMap` « positions en %, pas de vraie lib » | ⚠️ Corrigé | Carte **SVG** avec contour du pays, 12 régions et projection lat/lng ; échelle de couleurs à 5 paliers + légende + tooltip |
| Dashboard « agrège overview, revenue-trend + 4 /report » | ⚠️ Complété | Appelle aussi `sales-count-trend`, `best-clients`, `top-products` (9 requêtes en parallèle) |
| Chatbot : « `raw_data` non exploité » | ⚠️ Remplacé | Le code lit `reply`, `generated_sql`, `download_url` ; `raw_data` n'est pas utilisé |
| Exemple `getStockReport` | ✅ Exact | Mais d'autres services utilisent un helper `buildQuery` |
| Rapports : filtre « global » sur toutes les pages | ⚠️ Nuancé | Sales/Refunds/Clients/Payments utilisent une **version locale** du filtre (avec preset « Aujourd'hui ») |

## 5. Points d'attention

Constats issus de la lecture du code (détails dans les autres fichiers) :

1. **Casse d'import** : `App.jsx` importe `./pages/RefundsReport` alors que le fichier est `refundsReport.jsx` → échec sous Linux (CI, Docker).
2. **Fuseau horaire** : les presets de dates utilisent `toISOString()` (UTC) ; en UTC+1 le « 1er du mois » peut devenir le dernier jour du mois précédent.
3. **Chatbot** : URL de l'API codée en dur (`http://localhost:8000/api`), ignore `VITE_API_URL`.
4. **`refreshUser`** appelé dans `SettingsPage` mais non exposé par `AuthContext` ; `register` n'est pas exporté par le provider.
5. **Mot de passe oublié** : seul `{ email, new_password }` est envoyé, sans jeton — à vérifier côté backend.
6. **JWT en `localStorage`** (exposé en cas de XSS) ; thème non persisté ; case « Se souvenir de moi » sans effet.
7. **Branding incohérent** : NextAct / ERPilot / StockPilot ; `index.html` avec `<title>frontend</title>` et `lang="en"`.
8. **Duplication et code mort** : `formatDH` défini dans ~12 fichiers, `DateRangeFilter` réimplémenté dans 4 pages, composants/hook non utilisés (`useChartData`, `CustomBarTooltip`, sélecteurs de période, `utils/format.js`, `data/views_column/stock_views.js` vide) ; `console.log` résiduels.
9. **Pas de tests** automatisés ; langue `fr/en/ar` sauvegardée mais sans i18n.

## 6. Glossaire

| Terme | Définition |
|---|---|
| **RFM** | Segmentation clients par Récence, Fréquence, Montant |
| **CLV** | *Customer Lifetime Value* (valeur vie client) |
| **Churn** | Part de clients devenus inactifs |
| **Pipeline (devis)** | Devis en cours (ni acceptés ni refusés) |
| **Stock dormant** | Produits sans mouvement/vente sur la période |
| **HashRouter** | Routage via le fragment d'URL (`/#/chemin`) |
