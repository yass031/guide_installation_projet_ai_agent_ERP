# Tools et schémas

Cette page décrit **les 81 outils (« tools »)** que l'agent peut utiliser pour répondre, la façon dont ils sont déclarés au modèle de langage (les « schémas »), et les droits d'accès de chaque rôle.

- Fonctionnement général de l'agent : [architecture-agent.md](architecture-agent.md)
- Règles de sécurité : [securite.md](securite.md)
- Instructions données au modèle : [system-prompt.md](system-prompt.md)

## 1. Qu'est-ce qu'un tool ?

Un tool est une **action précise que l'agent a le droit de déclencher** (par exemple « donner les produits en stock bas »). Le modèle de langage ne lit jamais la base de données lui-même : il **choisit un tool et ses paramètres**, l'agent exécute le tool, puis le modèle rédige la réponse à partir du résultat.

Chaque tool est composé de **trois éléments** :

| Élément | Où | Rôle |
|---|---|---|
| **La fonction** (`handler`) | `src/tools/<domaine>/<couche>_tools.py` | Code Python asynchrone qui appelle le backend et renvoie le résultat |
| **Le schéma** (`schema`) | même fichier | Description JSON envoyée au modèle : nom, description, paramètres (types, valeurs autorisées, obligatoires) |
| **La fiche de sélection** | `src/tools/tool_examples.py` | 1 description + 3 exemples de questions, utilisés pour retrouver le bon tool (voir architecture §5) |

Tous les tools sont rassemblés dans **`src/tools/registry.py`** : *rien n'est accessible à l'agent s'il n'est pas listé dans ce registre*.

### Exemple : le tool `get_clients_list`

La fonction :

```python
async def get_clients_list(client_id=None, client_name=None,
                           client_email=None, client_phone=None, limit=15) -> dict:
    rows = await backend_get("/clients-consultation/list", {
        "client_id": client_id, "client_name": client_name,
        "client_email": client_email, "client_phone": client_phone, "limit": limit,
    })
    return format_tool_response(data=rows)
```

Son schéma, tel que le modèle le reçoit (format *function calling*) :

```json
{
  "type": "function",
  "function": {
    "name": "get_clients_list",
    "description": "Recherche des clients par id, nom, email ou téléphone. Renvoie les informations de base (nom, email, téléphone, adresse, date de création). Utilise ce tool pour retrouver un client précis ou lister les clients correspondant à un critère.",
    "parameters": {
      "type": "object",
      "properties": {
        "client_id": {
          "type": "integer"
        },
        "client_name": {
          "type": "string"
        },
        "client_email": {
          "type": "string"
        },
        "client_phone": {
          "type": "string"
        },
        "limit": {
          "type": "integer",
          "description": "Défaut 15, max 20"
        }
      },
      "required": []
    }
  }
}
```

Le champ `description` est essentiel : c'est lui qui permet au modèle de savoir **quand** utiliser le tool. Le champ `required` indique les paramètres obligatoires (ici aucun).

## 2. Format des réponses

Tous les tools renvoient la **même enveloppe** (`format_tool_response`) :

```json
{ "data": [ ... ], "alerts": [ ... ], "row_count": 12 }
```

| Champ | Contenu |
|---|---|
| `data` | Les résultats (liste de lignes ou objet) |
| `alerts` | Alertes éventuelles (`type`, `message`, `severity`), que le modèle doit mentionner dans sa réponse |
| `row_count` | Nombre de lignes si `data` est une liste |

En cas d'échec, l'exécuteur renvoie `{ "error": "...", "data": null, "alerts": [] }` : erreur du backend (`ToolExecutionError`), permission refusée, ou erreur inattendue. Le modèle explique alors le problème à l'utilisateur.

Avant l'appel, les arguments reçus du modèle (souvent des textes) sont **convertis au bon type** Python d'après les annotations de la fonction (`int`, `float`, `bool`) ; une valeur non convertible devient `None`.

## 3. Comment les tools accèdent aux données

**Les 81 tools passent par l'API du backend** (via `backend_get` ou `backend_post` dans `src/tools/base.py`). Le backend exécute ensuite lui-même les requêtes sur la base. Conséquences :

- **le backend doit être démarré** pour que l'agent puisse répondre ;
- l'agent s'authentifie auprès du backend avec son **compte de service** (`AGENT_SERVICE_EMAIL` / `AGENT_SERVICE_PASSWORD`, rôle `ADMIN`), avec un jeton mis en cache et renouvelé automatiquement (nouvelle tentative en cas de `401`) ;
- les paramètres vides (`None`) ne sont pas envoyés ;
- délai maximal : 10 s pour une lecture (`GET`), 30 s pour une action (`POST`).

La connexion directe à PostgreSQL (rôle `agent_user`) n'est utilisée que par le **mécanisme de secours NL→SQL** (voir architecture §7).

## 4. Les 81 tools par domaine

Les tools sont organisés comme les fonctions SQL de la base, en **trois couches** :

- **consultation** : lire, rechercher, lister (une vue détaillée) ;
- **traitement** : calculer (totaux, statistiques, classements, agrégations par période) ;
- **recommandation** : détecter ce qui demande une action (relances, ruptures, retards…).

Dans la colonne « Paramètres », les paramètres **en gras** sont obligatoires. Toutes les dates se donnent au format `AAAA-MM-JJ`. Les valeurs autorisées de chaque paramètre (par exemple les statuts) sont définies dans le schéma, dans le fichier `*_tools.py` du tool.

### Produits et stock (15 tools)

| Tool | Couche | Endpoint du backend | Paramètres | À quoi il sert |
|---|---|---|---|---|
| `get_products_catalog` | consultation | `GET /stock-consultation/catalog` | category_id (entier), supplier_id (entier), min_price (nombre), max_price (nombre), name_search (texte), sort_by (texte) | Catalogue produits filtrable (catégorie, fournisseur, prix, nom). |
| `get_categories_list` | consultation | `GET /stock-consultation/categories` | name_search (texte) | Retourne la liste des catégories de produits avec leur ID et leur nom. |
| `get_products_list` | consultation | `GET /stock-consultation/products` | name_search (texte) | Retourne la liste des produits avec leur ID et leur nom. |
| `get_low_stock_products` | consultation | `GET /stock-consultation/low-stock` | — | Produits sous le seuil d'alerte, avec sévérité. |
| `get_top_priced_products` | consultation | `GET /stock-consultation/top-priced` | limit (entier), order (texte), category_id (entier), supplier_id (entier) | Classement des produits par prix, filtrable par catégorie et/ou fournisseur. |
| `get_stock_movements_detail` | consultation | `GET /stock-consultation/movements` | product_id (entier), limit (entier) | Retourne l'historique des mouvements de stock (entrées/sorties) pour un produit ou tous les produits. |
| `get_product_stock_value` | traitement | `GET /stock-traitement/value` | category_id (entier), supplier_id (entier) | Valeur totale du stock (qté × prix), filtrable par catégorie/fournisseur. |
| `get_product_sales_velocity` | traitement | `GET /stock-traitement/velocity` | **product_id (entier)**, days_lookback (entier) | Rythme de vente, stock actuel et jours avant rupture pour UN produit précis. |
| `get_fast_moving_products` | traitement | `GET /stock-traitement/fast-moving-products` | days_lookback (entier), target_coverage_days (entier), limit (entier) | Produits les plus vendus récemment, avec quantité suggérée de réapprovisionnement proactif. |
| `get_products_count_by_category` | traitement | `GET /stock-traitement/count-by-category` | — | Nombre de produits par catégorie, triés par quantité décroissante. |
| `get_products_count_by_supplier` | traitement | `GET /stock-traitement/count-by-supplier` | — | Nombre de produits par fournisseur, triés par quantité décroissante. |
| `get_products_no_movement` | traitement | `GET /stock-traitement/no-movement` | days (entier) | Retourne les produits n'ayant eu aucun mouvement de stock depuis X jours (stock dormant). |
| `get_stock_movements_by_period` | traitement | `GET /stock-traitement/movements/by-period` | **start_date (texte)**, **end_date (texte)**, product_id (entier), movement_type (texte) | Retourne les mouvements de stock sur une période donnée, filtrable par produit et type de mouvement (entrée, sortie, ajustement...). |
| `get_products_to_discontinue` | recommandation | `GET /stock-recommandation/to-discontinue` | dormant_days_threshold (entier), days_lookback (entier) | Retourne les produits candidats à l'arrêt de vente : marge négative (perte à chaque vente) et/ou stock dormant (pas vendu depuis longtemps). |
| `get_reorder_suggestions` | recommandation | `GET /stock-recommandation/reorder-suggestions` | days_lookback (entier), target_coverage_days (entier), urgent_threshold_days (entier) | Produits sous seuil d'alerte, triés par urgence, avec quantité suggérée à commander. |

### Ventes (11 tools)

| Tool | Couche | Endpoint du backend | Paramètres | À quoi il sert |
|---|---|---|---|---|
| `get_sales_by_period` | consultation | `GET /sales-consultation/by-period` | **start_date (texte)**, **end_date (texte)**, updated_start (texte), updated_end (texte), status (texte), order_by (texte), order_direction (texte), limit (entier) | Liste des ventes filtrée par période de création/modification et statut de paiement, triable. |
| `get_sale_lookup` | consultation | `GET /sales-consultation/lookup` | sale_id (entier), reference (texte) | Recherche une vente précise par son ID ou sa référence exacte (ex: SAL-2026-0004). |
| `get_sales_by_client` | consultation | `GET /sales-consultation/by-client` | client_id (entier), client_name (texte), client_email (texte), client_phone (texte), client_city (texte), start_date (texte), end_date (texte), limit (entier) | Liste des ventes d'un client identifié par ID, nom, email, téléphone ou ville. |
| `get_sales_by_amounts` | consultation | `GET /sales-consultation/by-amounts` | min_subtotal (nombre), max_subtotal (nombre), min_tax_rate (nombre), max_tax_rate (nombre), min_tax_amount (nombre), max_tax_amount (nombre), min_discount_amount (nombre), max_discount_amount (nombre), min_gross_total (nombre), max_gross_total (nombre), min_net_total (nombre), max_net_total (nombre), min_refunded_total (nombre), max_refunded_total (nombre), has_refund (booléen), start_date (texte), end_date (texte), order_by (texte), order_direction (texte), limit (entier) | Ventes filtrées par plage de montant : subtotal, tax_rate, tax_amount, discount_amount, gross_total (après remise/taxe avant remboursement), net_total (réellement conservé), refunded_total, ou présence de remboursement. |
| `get_sale_items_by_sale` | consultation | `GET /sales-consultation/items/by-sale` | sale_id (entier), reference (texte), sale_status (texte) | Liste des produits contenus dans une vente précise, identifiée par ID ou référence. |
| `get_sales_history_by_product` | consultation | `GET /sales-consultation/items/history-by-product` | product_id (entier), product_name (texte), product_reference (texte), category_id (entier), category_name (texte), start_date (texte), end_date (texte), limit (entier) | Historique des ventes d'un produit précis ou d'une catégorie, filtrable par période. |
| `get_sale_items_by_client` | consultation | `GET /sales-consultation/items/by-client` | client_id (entier), client_name (texte), limit (entier) | Liste des produits achetés par un client précis (identifié par ID ou nom), toutes ventes confondues. |
| `get_sale_items_by_metrics` | consultation | `GET /sales-consultation/items/by-metrics` | min_ordered_quantity (nombre), max_ordered_quantity (nombre), min_subtotal (nombre), max_subtotal (nombre), refund_type (texte), min_refund_rate (nombre), max_refund_rate (nombre), order_by (texte), order_direction (texte), limit (entier) | Lignes de vente filtrées par quantité commandée, montant, ou remboursement (type: none/partial/full, ou taux de retour en %). |
| `get_sales_statistics_by_period` | traitement | `GET /sales-traitement/statistics-by-period` | **start_date (texte)**, **end_date (texte)**, period (texte), client_id (entier), product_id (entier), status (texte), category_id (entier), limit (entier) | Statistiques de ventes agrégées par période (jour, semaine, mois, trimestre, année). |
| `get_sales_statistics_summary` | traitement | `GET /sales-traitement/statistics-summary` | **start_date (texte)**, **end_date (texte)**, client_id (entier), product_id (entier), status (texte), category_id (entier) | Résumé global des ventes : nombre de ventes, CA brut/net/remboursé, panier moyen, clients distincts, taux de remboursement. |
| `get_sales_decline_alert` | recommandation | `GET /sales-recommandation/decline-alert` | **current_start (texte)**, **current_end (texte)**, **previous_start (texte)**, **previous_end (texte)**, threshold_pct (nombre), limit (entier) | Détecte les produits dont la quantité vendue a baissé entre deux périodes (période actuelle vs période de référence). |

### Achats et fournisseurs (14 tools)

| Tool | Couche | Endpoint du backend | Paramètres | À quoi il sert |
|---|---|---|---|---|
| `get_suppliers_list` | consultation | `GET /purchases-consultation/suppliers` | — | Retourne la liste des fournisseurs avec leur nombre de produits associés. |
| `get_purchases_detail` | consultation | `GET /purchases-consultation/detail` | — | Retourne la liste détaillée des achats (fournisseur, statut, total). |
| `get_purchase_items_detail` | consultation | `GET /purchases-consultation/items-detail` | — | Retourne le détail ligne par ligne des achats, avec produit et quantité. |
| `get_purchases_by_period` | traitement | `GET /purchases-traitement/by-period` | **start_date (texte)**, **end_date (texte)**, supplier_id (entier) | Calcule le total acheté sur une période, filtrable par fournisseur. |
| `get_supplier_price_history` | traitement | `GET /purchases-traitement/supplier-price-history` | **product_id (entier)**, supplier_id (entier) | Retourne l'évolution du prix d'achat d'UN produit dans le temps, filtrable par fournisseur. |
| `get_top_suppliers` | traitement | `GET /purchases-traitement/top-suppliers` | **start_date (texte)**, **end_date (texte)**, limit (entier) | Classement des fournisseurs par volume d'achat sur une période. |
| `get_compare_supplier_prices` | traitement | `GET /purchases-traitement/compare-supplier-prices` | **product_id (entier)** | Compare le prix d'achat d'UN produit entre tous les fournisseurs qui le fournissent. |
| `get_supplier_summary` | traitement | `GET /purchases-traitement/supplier-summary` | **supplier_id (entier)** | Résumé d'UN fournisseur précis : total acheté, nombre de produits, dernière commande. |
| `get_suppliers_no_recent_order` | traitement | `GET /purchases-traitement/suppliers-no-recent-order` | days (entier) | Retourne les fournisseurs sans commande depuis X jours. |
| `get_current_month_purchases` | traitement | `GET /purchases/current-month` | supplier_id (entier) | Retourne la liste des achats (commandes fournisseurs) effectués depuis le début du mois en cours jusqu'à aujourd'hui. |
| `get_product_price_evolution` | traitement | `GET /purchases/price-evolution` | **product_id (entier)**, supplier_id (entier) | Retourne l'historique d'évolution du prix d'achat d'un produit dans le temps, à partir des commandes passées auprès des fournisseurs. |
| `get_most_purchased_products` | traitement | `GET /purchases/most-purchased-products` | **limit (entier)** | Retourne le classement des produits les plus achetés, triés par quantité (ou volume d'achat) décroissante, tous fournisseurs confondus. |
| `get_supplier_concentration` | traitement | `GET /purchases/supplier-concentration` | **top_n (entier)** | Retourne la concentration des achats par fournisseur, c'est-à-dire la part (en montant ou en volume) que représente chaque fournisseur dans le total des achats de l'entreprise. |
| `get_best_supplier_for_product` | recommandation | `GET /purchases-recommandation/best-supplier` | product_id (entier), product_name (texte) | Recommande le meilleur fournisseur pour UN produit précis, identifié soit par son product_id (ID numérique), soit par son nom (product_name). |

### Remboursements (12 tools)

| Tool | Couche | Endpoint du backend | Paramètres | À quoi il sert |
|---|---|---|---|---|
| `get_refunds_by_period` | consultation | `GET /refunds-consultation/by-period` | **start_date (texte)**, **end_date (texte)**, order_by (texte), order_direction (texte), limit (entier) | Liste des remboursements filtrée par période de création, triable par date ou montant. |
| `get_refund_lookup` | consultation | `GET /refunds-consultation/lookup` | refund_id (entier), sale_reference (texte) | Recherche un remboursement précis par son ID ou la référence de la vente associée (ex: SAL-2026-0004). |
| `get_refunds_by_client` | consultation | `GET /refunds-consultation/by-client` | client_id (entier), client_name (texte), start_date (texte), end_date (texte), limit (entier) | Liste des remboursements d'un client identifié par ID ou nom. |
| `get_refunds_by_reason_text` | consultation | `GET /refunds-consultation/by-reason` | reason (texte), limit (entier) | Recherche libre des remboursements dont la raison contient un texte donné (ex: 'défectueux', 'erreur livraison'). |
| `get_refund_items_by_refund` | consultation | `GET /refunds-consultation/items/by-refund` | refund_id (entier), sale_reference (texte) | Liste des produits remboursés dans un remboursement précis, identifié par ID ou référence de vente. |
| `get_refund_items_by_product` | consultation | `GET /refunds-consultation/items/by-product` | product_id (entier), product_name (texte), product_reference (texte), start_date (texte), end_date (texte), limit (entier) | Historique des remboursements d'un produit précis, filtrable par période. |
| `get_refund_items_by_client` | consultation | `GET /refunds-consultation/items/by-client` | client_id (entier), client_name (texte), limit (entier) | Liste des produits remboursés à un client précis (identifié par ID ou nom), tous remboursements confondus. |
| `get_refunds_period_detail` | traitement | `GET /refunds-traitement/period-detail` | **start_date (texte)**, **end_date (texte)**, limit (entier) | Détail complet des remboursements sur une période : une ligne par produit remboursé, avec la référence de vente, le nom du client, le total du remboursement, la date, le produit, son prix unitaire remboursé et la quantité. |
| `get_refund_rate_by_product` | traitement | `GET /refunds-traitement/refund-rate-by-product` | start_date (texte), end_date (texte), limit (entier) | Taux de retour par produit (quantité remboursée / quantité vendue, en %), classé du plus élevé au plus faible. |
| `get_refunds_by_reason` | traitement | `GET /refunds-traitement/by-reason` | **start_date (texte)**, **end_date (texte)** | Répartition agrégée des remboursements par raison sur une période : nombre de remboursements et montant total par raison, classé par montant décroissant. |
| `get_total_refunded` | traitement | `GET /refunds-traitement/total-refunded` | **start_date (texte)**, **end_date (texte)** | Montant total remboursé sur une période donnée, tous remboursements confondus. |
| `get_products_high_return_rate` | recommandation | `GET /refunds-recommandation/high-return-rate` | threshold_pct (nombre), limit (entier) | Détecte les produits dont le taux de retour (quantité remboursée / quantité vendue) dépasse un seuil donné, signal potentiel de problème qualité. |

### Clients (6 tools)

| Tool | Couche | Endpoint du backend | Paramètres | À quoi il sert |
|---|---|---|---|---|
| `get_clients_list` | consultation | `GET /clients-consultation/list` | client_id (entier), client_name (texte), client_email (texte), client_phone (texte), limit (entier) | Recherche des clients par id, nom, email ou téléphone. |
| `get_client_purchase_history` | consultation | `GET /clients-consultation/purchase-history` | client_id (entier), client_name (texte), sale_status (texte), start_date (texte), end_date (texte), limit (entier) | Historique détaillé des ventes d'un client (référence de vente, date, statut, montants). |
| `get_client_lifetime_value` | traitement | `GET /clients-traitement/lifetime-value` | **client_id (entier)** | Valeur vie d'un client (lifetime value) : total dépensé, nombre de commandes, panier moyen. |
| `get_top_clients` | traitement | `GET /clients-traitement/top-clients` | **start_date (texte)**, **end_date (texte)**, limit (entier) | Classement des meilleurs clients par montant total d'achat sur une période donnée. |
| `get_inactive_clients` | traitement | `GET /clients-traitement/inactive` | **days_inactive (entier)**, limit (entier) | Liste des clients n'ayant fait aucun achat depuis un nombre de jours donné. |
| `get_clients_to_recontact` | recommandation | `GET /clients-recommandation/to-recontact` | days_inactive (entier), min_historical_amount (nombre), limit (entier) | Détecte les clients à relancer selon leur inactivité (jours sans achat) et, optionnellement, un montant historique minimal dépensé. |

### Paiements (15 tools)

| Tool | Couche | Endpoint du backend | Paramètres | À quoi il sert |
|---|---|---|---|---|
| `get_payments_detail` | consultation | `GET /payments-consultation/detail` | sale_id (entier), client_id (entier), payment_id (entier), client_name (texte), sale_reference (texte), limit (entier) | Détail brut des tranches de paiement versées (montant appliqué, date d'allocation, méthode de règlement), filtrable par vente (sale_id OU sale_reference), client (client_id OU client_name) ou paiement précis. |
| `search_client_by_name` | consultation | `GET /payments-consultation/search-client` | **name (texte)** | Recherche un client par nom (recherche partielle, insensible à la casse) et renvoie toutes les correspondances. |
| `search_sale_by_reference` | consultation | `GET /payments-consultation/search-sale` | **reference (texte)** | Recherche une vente par référence (ex: SAL-2026-0004) ou par nom de client, recherche partielle, et renvoie toutes les correspondances. |
| `get_client_sales_summary` | consultation | `GET /payments-consultation/client/sales-summary` | client_id (entier), client_name (texte) | Détail complet des ventes d'UN client, identifié par client_id OU par client_name (pas besoin de chercher l'id avant, la résolution est automatique) : pour chaque vente, montant total, total payé, reste à payer… |
| `get_sales_with_remaining_balance` | consultation | `GET /payments-consultation/remaining-balance` | limit (entier) | Liste TOUTES les ventes ayant encore un solde à payer (partiellement payées ou pas payées du tout), tous clients confondus, triées par solde restant décroissant. |
| `get_sale_allocations` | consultation | `GET /payments-consultation/sale/allocations` | sale_id (entier), sale_reference (texte) | Liste chronologique des tranches de paiement versées pour UNE vente, identifiée par sale_id OU par sale_reference (référence de vente ou nom de client). |
| `get_payments_by_method` | traitement | `GET /payments-traitement/by-method` | start_date (texte), end_date (texte) | Répartition des paiements encaissés par méthode de règlement (espèces, carte, virement, etc.) sur une période donnée. |
| `get_payments_by_status` | traitement | `GET /payments-traitement/by-status` | start_date (texte), end_date (texte) | Répartition des paiements par statut (ex. |
| `get_client_balance` | traitement | `GET /payments-traitement/client/balance` | client_id (entier), client_name (texte) | Solde global d'UN client, identifié par client_id OU par client_name (résolution automatique du nom), toutes ventes confondues (total vendu net - total payé). |
| `get_cash_collected` | traitement | `GET /payments-traitement/cash-collected` | start_date (texte), end_date (texte) | Montant total encaissé sur une période donnée, avec nombre de transactions et montant moyen par transaction. |
| `get_sale_balance` | traitement | `GET /payments-traitement/sale/balance` | sale_id (entier), sale_reference (texte) | Solde restant dû pour UNE vente, identifiée par sale_id OU par sale_reference (référence de vente ou nom de client) — couvre aussi le cas où aucun paiement n'a été reçu. |
| `get_sale_payment_alerts` | traitement | `GET /payments-traitement/payment-alerts` | threshold_days (entier) | Liste des ventes en retard de paiement (solde restant > 0 depuis au moins threshold_days jours), avec message distinguant 'rien payé' et 'payé partiellement'. |
| `get_recurring_late_clients` | recommandation | `GET /payments-recommandation/recurring-late-clients` | threshold_days (entier), min_occurrences (entier) | Détecte les clients récidivistes ayant plusieurs ventes en retard de paiement (>= min_occurrences ventes en retard de plus de threshold_days jours, payées partiellement ou pas du tout), avec le montant total impayé et un texte… |
| `get_cashflow_forecast` | recommandation | `GET /payments-recommandation/cashflow-forecast` | days_ahead (entier) | Prévision de trésorerie attendue sur les days_ahead prochains jours, basée sur les soldes restant dus. |
| `check_client_recurring_late_status` | recommandation | `GET /payments-recommandation/client/recurring-late-status` | client_id (entier), client_name (texte), threshold_days (entier), min_occurrences (entier) | Vérifie si UN client, identifié par client_id OU par client_name (résolution automatique du nom), fait partie des récidivistes de retard de paiement. |

### Devis (7 tools)

| Tool | Couche | Endpoint du backend | Paramètres | À quoi il sert |
|---|---|---|---|---|
| `get_devis_detail` | consultation | `GET /devis-consultation/detail` | — | Retourne la liste des devis (propositions commerciales PAS ENCORE conclues), avec client, statut (draft/sent/accepted/rejected), montants. |
| `get_devis_items_detail` | consultation | `GET /devis-consultation/items-detail` | — | Retourne le détail ligne par ligne des devis, avec produit et quantité proposés. |
| `get_devis_conversion_rate` | traitement | `GET /devis-traitement/conversion-rate` | **start_date (texte)**, **end_date (texte)** | Calcule le taux de conversion des devis en ventes (% accepté) sur une période. |
| `get_devis_avg_response_time` | traitement | `GET /devis-traitement/avg-response-time` | start_date (texte), end_date (texte) | Délai moyen (en jours) entre l'envoi d'un devis et la réponse du client (accepté ou refusé). |
| `get_devis_by_status_summary` | traitement | `GET /devis-traitement/status-summary` | **start_date (texte)**, **end_date (texte)** | Répartition des devis par statut (draft/sent/accepted/rejected) avec montants, sur une période. |
| `get_devis_pending_followup` | recommandation | `GET /devis-recommandation/pending-followup` | days_since_sent (entier) | Retourne les devis envoyés (status sent) sans réponse depuis X jours, à relancer auprès du client. |
| `get_devis_about_to_expire` | recommandation | `GET /devis-recommandation/about-to-expire` | days_before (entier) | Retourne les devis proches de leur date d'expiration (pas encore acceptés/refusés), à relancer en urgence avant qu'ils n'expirent. |

### Rapports téléchargeables (1 tool)

| Tool | Couche | Endpoint du backend | Paramètres | À quoi il sert |
|---|---|---|---|---|
| `generate_report_file` | génération | `POST /reports/export` | **report_type (texte)**, file_format (texte), start_date (texte), end_date (texte), client_id (entier), client_name (texte), client_email (texte), client_phone (texte), supplier_id (entier), status (texte), limit (entier) | Génère un fichier téléchargeable (Excel, CSV ou PDF) pour un rapport donné (ventes, paiements, stock, remboursements, clients, achats, fournisseurs, devis) et retourne un lien de téléchargement. |

### Le tool d'export `generate_report_file`

Ce tool est le seul à utiliser `POST` (`/reports/export`). Il génère un fichier **Excel (`xlsx`), CSV ou PDF** et renvoie un lien de téléchargement (`download_url`) que l'agent transmet dans sa réponse.

- `report_type` (obligatoire) : `sales`, `payments`, `stock`, `refunds`, `clients`, `purchases`, `suppliers` ou `devis` ;
- `file_format` : `xlsx` (défaut), `csv` ou `pdf` ;
- filtres facultatifs : `start_date`, `end_date`, `client_id`, `client_name`, `client_email`, `client_phone`, `supplier_id`, `status`, `limit`. Chaque filtre ne s'applique qu'à certains types de rapport (précisé dans le schéma) ;
- il ne doit être utilisé que sur **demande explicite** d'un export ou d'un fichier ;
- **le type de rapport est contrôlé selon le rôle** de l'utilisateur avant l'appel (voir ci-dessous et [securite.md](securite.md)).

## 5. Droits d'accès par rôle

Le rôle de l'utilisateur est fourni par le backend (`GET /auth/me`). Il détermine **trois choses** : les tools autorisés, les vues SQL autorisées pour le mécanisme de secours, et les types de rapport exportables.

| Rôle | Nb de tools | Vues SQL autorisées (secours NL→SQL) | Types de rapport |
|---|:-:|---|---|
| `admin` | 81 | toutes les vues (16) | tous (8) |
| `user_stock` | 17 | `v_products_catalog`, `v_low_stock_products`, `v_categories`, `v_stock_movements` | `stock` |
| `user_sales` | 14 | `v_sales_detail`, `v_sale_items` | `sales` |
| `user_purchases` | 10 | `v_purchases`, `v_purchase_items` | `purchases` |
| `user_suppliers` | 10 | `v_suppliers`, `v_purchases` | `suppliers` |
| `user_devis` | 8 | `v_devis`, `v_devis_items` | `devis` |
| `user_payments` | 16 | `v_payments`, `v_payment_allocations` | `payments` |
| `user_clients` | 8 | `v_clients` | `clients` |
| `user_refunds` | 14 | `v_refunds`, `v_refund_items` | `refunds` |

Précisions :

- `admin` a accès à **tous** les tools, toutes les vues et tous les types de rapport.
- Chaque rôle `user_*` reçoit les tools de **son domaine**, plus le tool d'export (limité à son type de rapport) et quelques tools « outils de résolution » qui permettent de convertir un nom en identifiant :
  - `get_products_list` : `user_stock`, `user_sales`, `user_purchases`, `user_suppliers`, `user_refunds` ;
  - `get_suppliers_list` : `user_stock`, `user_purchases`, `user_suppliers` ;
  - `get_categories_list` : `user_stock`, `user_sales` ;
  - `get_supplier_price_history` : `user_purchases`, `user_suppliers` ;
  - `search_client_by_name` : `user_payments`, `user_clients`.
- Un rôle inconnu n'obtient **aucun** tool ni aucune vue (principe « fail-closed »).
- Les valeurs de rôle sont en **minuscules** côté agent (`user_stock`), alors que la base PostgreSQL stocke `USER_STOCK` : c'est ce que renvoie `GET /auth/me`.
- ⚠️ Ces permissions sont **le miroir manuel** de `core/role_permissions.py` du backend (indiqué dans l'en-tête du fichier). Si vous ajoutez un tool, une vue, un type de rapport ou un rôle d'un côté, **répliquez-le de l'autre côté**.

### Cohérence vérifiée

Lors de la rédaction de cette documentation, le code a été contrôlé : chacun des 81 tools du registre est rattaché à au moins un groupe de rôle, aucun nom listé dans les rôles n'est absent du registre, et chacun des 81 tools possède sa fiche de sélection dans `tool_examples.py`.

## 6. Autres fichiers du dossier `tools/`

| Fichier | Rôle |
|---|---|
| `registry.py` | Rassemble tous les tools (`ALL_TOOLS`), fournit les schémas au modèle (`get_all_schemas`), retrouve la fonction d'un tool (`get_tool_function`) et liste les 16 vues autorisées pour le secours (`get_view_names`) |
| `base.py` | Socle commun : `backend_get`, `backend_post`, `run_query`, `format_tool_response`, `build_alert`, `ToolExecutionError` |
| `role_permissions.py` | Droits par rôle et messages de refus (« tu n'as accès qu'aux ventes… ») |
| `tool_selector.py` | Sélection des tools les plus pertinents par similarité sémantique |
| `tool_examples.py` | Fiches de sélection (`TOOL_KNOWLEDGE`) |
| `column_registry.py` | Liste des colonnes valides pour certains tools : mécanisme de **réduction de colonnes**, actuellement **désactivé** (appel commenté dans `tool_executor.py`) |
| `<domaine>/__init__.py` | Regroupe les tools d'un domaine dans un dictionnaire (`STOCK_TOOLS`, `SALES_TOOLS`, `CLIENTS_TOOLS`…) |

## 7. Ajouter un nouveau tool (pas à pas)

1. **Côté backend** : créer (ou vérifier) l'endpoint qui fournit les données.
2. **Écrire la fonction et son schéma** dans le fichier `<couche>_tools.py` du domaine concerné (`async def ...` + dictionnaire `..._TOOL`). Soigner la `description` du schéma : elle guide le modèle.
3. **L'exporter** dans `src/tools/<domaine>/__init__.py` : ajouter une ligne `"nom_du_tool": {"handler": ..., "schema": ...}` au dictionnaire du domaine. Le registre le prend ensuite automatiquement.
4. **Ajouter sa fiche** dans `src/tools/tool_examples.py` : **une description et exactement 3 exemples** de questions. Le nombre est vérifié au démarrage (voir architecture §5) ; un autre nombre arrête l'application.
5. **Donner l'accès aux rôles** : ajouter le nom du tool dans la liste du domaine (`_STOCK_TOOLS`, `_SALES_TOOLS`…) de `src/tools/role_permissions.py`, et répliquer dans `core/role_permissions.py` du backend.
6. **Si le tool s'appuie sur une nouvelle vue** pour le secours NL→SQL : l'ajouter dans `get_view_names()` (`registry.py`), dans `ROLE_VIEWS` et `VIEW_DOMAIN` (`role_permissions.py`) et dans la documentation des vues de `nl_to_sql_fallback.py`.
7. **Redémarrer l'agent** et tester avec une question typique (la console affiche les scores de sélection).
