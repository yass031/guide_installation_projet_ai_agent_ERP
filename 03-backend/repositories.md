# Repositories

Ce document décrit la couche d'accès aux données (`repositories/`). Chaque fichier regroupe les requêtes SQL d'un domaine métier.

## 1. Principe général

```
Router (api/)  →  Service (services/, optionnel)  →  Repository (repositories/)  →  PostgreSQL
```

- Les repositories exécutent du **SQL paramétré** (`text()` + `:param`) sur les **vues** (`v_*`) et **fonctions** (`fn_*`) du schéma `agent_reporting`. Aucune logique d'écriture métier : seules les tables `users`, `file_exports` et `alert_notifications_sent` sont modifiées.
- Un repository reçoit une `Session` SQLAlchemy dans son constructeur (`Repository(db)`), injectée via `Depends(get_db)`.
- Les dates sont passées sous forme de chaîne `YYYY-MM-DD`.

### Les quatre familles de classes

Un domaine peut exposer jusqu'à quatre classes :

| Classe | Rôle | Consommée par |
|---|---|---|
| `*AnalyticsRepository` | KPI composés pour les pages de rapport du frontend | Routers `api/<domaine>.py` |
| `*ConsultationRepository` | Recherche et détail (lecture « brute », filtrable) | Routes `/<domaine>-consultation` (agent IA) et exports |
| `*TraitementRepository` | Calculs et agrégats | Routes `/<domaine>-traitement` (agent IA) |
| `*RecommandationRepository` | Suggestions et alertes | Routes `/<domaine>-recommandation` (agent IA) |

### Colonne « Type » des tableaux

Le type est déduit des paramètres de la méthode :

| Type | Signification |
|---|---|
| **Snapshot** | Photo de l'état actuel, sans paramètre de temps |
| **Filtrable** | Accepte `start_date` / `end_date` |
| **Fenêtre glissante** | Calculé depuis aujourd'hui sur N jours / mois (`days`, `months`, `threshold_days`…) |
| **Lookup** | Recherche ciblée par identifiant, nom ou référence |

> **Note importante (stock).** `get_fast_moving_products`, `get_dormant_products_count` et `get_discontinue_summary` restent en fenêtre glissante depuis aujourd'hui : elles dépendent du stock actuel, qui n'a pas d'historique en base.

> **Résolution nom → identifiant.** Les repositories de paiements, ventes et remboursements acceptent un nom de client ou une référence de vente. En cas d'ambiguïté, ils renvoient `{"resolved": false, ...}` plutôt qu'une liste, pour que l'agent demande une précision.

> **Sécurité du SQL dynamique.** Les rares fragments injectés par f-string (clause `ORDER BY`) passent par une liste blanche de colonnes (`allowed_order`) et un sens de tri normalisé en `ASC` / `DESC`.

## 2. Index des fichiers

| Fichier | Domaine |
|---|---|
| `repositories/stock_repository.py` | Stock |
| `repositories/purchases_repository.py` | Purchases (achats) |
| `repositories/suppliers_repository.py` | Suppliers (fournisseurs) |
| `repositories/devis_repository.py` | Devis |
| `repositories/sales_repository.py` | Sales (ventes) |
| `repositories/refunds_repository.py` | Refunds (remboursements) |
| `repositories/clients_repository.py` | Clients |
| `repositories/payments_repository.py` | Payments (paiements) |
| `repositories/dashboard_repository.py` | Dashboard |
| `repositories/user_repository.py` | Utilisateurs |
| `repositories/admin_user_repository.py` | Administration des utilisateurs |
| `repositories/alert_notification_repository.py` | Alertes |

## 3. Stock

Catalogue produits, valeur et mouvements de stock, alertes de rupture, rotation et produits dormants.

### `StockAnalyticsRepository` — Analytics (pages de rapport)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_stock_by_category()` | — | Snapshot | Stock total, valeur et nb de produits par catégorie |
| `get_stock_severity()` | — | Snapshot | Liste détaillée des produits en alerte de stock |
| `get_stock_severity_summary()` | — | Snapshot | Comptage des produits par niveau de sévérité |
| `get_cheapest_products()` | `limit` | Snapshot | Produits les moins chers |
| `get_most_expensive_products()` | `limit` | Snapshot | Produits les plus chers |
| `get_most_expensive_per_category()` | — | Snapshot | Produit le plus cher de chaque catégorie |
| `get_cheapest_per_category()` | — | Snapshot | Produit le moins cher de chaque catégorie |
| `get_total_stock_value()` | — | Snapshot | Valeur totale, nb d'unités et nb de produits |
| `get_stock_value_by_category()` | `limit` | Snapshot | Top catégories par valeur de stock |
| `get_movements_by_range()` | `start_date`, `end_date` | Filtrable | Mouvements de stock par type sur une plage de dates |
| `get_movements_over_time()` | `start_date`, `end_date` | Filtrable | Mouvements groupés par jour (graphique) |
| `get_dormant_products_count()` | `days` | Fenêtre glissante | Produits sans mouvement depuis X jours |
| `get_discontinue_summary()` | `dormant_days_threshold`, `days_lookback` | Fenêtre glissante | Résumé des produits candidats à l'arrêt |
| `get_fast_moving_products()` | `limit`, `days_lookback`, `target_coverage_days` | Fenêtre glissante | Produits à rotation rapide |
| `get_stock_report()` | `start_date`, `end_date` | Filtrable | **Endpoint résumé** : agrège toutes les méthodes du domaine |

### `StockConsultationRepository` — Consultation (outils agent)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_products_catalog()` | `category_id`, `supplier_id`, `min_price`, `max_price`, `name_search`, `sort_by` | Lookup | Catalogue produits filtrable (catégorie, fournisseur, prix, nom, tri) |
| `get_categories_list()` | `name_search` | Snapshot | Liste des catégories (recherche par nom) |
| `get_products_list()` | `name_search` | Snapshot | Liste des produits (recherche par nom) – résolveur nom → id |
| `get_suppliers_list()` | `name_search` | Snapshot | Liste des fournisseurs |
| `get_low_stock_products()` | — | Snapshot | Produits en stock bas |
| `get_top_priced_products()` | `limit`, `order`, `category_id`, `supplier_id` | Lookup | Produits les plus/moins chers, filtrables |
| `get_stock_movements_detail()` | `product_id`, `limit` | Lookup | Détail des mouvements de stock (produit, limite) |

### `StockTraitementRepository` — Traitement (outils agent)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_product_stock_value()` | `category_id`, `supplier_id` | Lookup | Valeur de stock par produit |
| `get_product_sales_velocity()` | `product_id`, `days_lookback` | Fenêtre glissante | Vitesse de vente d'un produit sur N jours |
| `get_fast_moving_products()` | `days_lookback`, `target_coverage_days`, `limit` | Fenêtre glissante | Produits à rotation rapide |
| `get_products_count_by_category()` | — | Snapshot | Nombre de produits par catégorie |
| `get_products_count_by_supplier()` | — | Snapshot | Nombre de produits par fournisseur |
| `get_stock_movements_by_period()` | `start_date`, `end_date`, `product_id`, `movement_type` | Filtrable | Mouvements par période (produit, type) |
| `get_products_no_movement()` | `days` | Fenêtre glissante | Produits sans mouvement depuis X jours |

### `StockRecommandationRepository` — Recommandation (outils agent)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_reorder_suggestions()` | `days_lookback`, `target_coverage_days` | Fenêtre glissante | Suggestions de réapprovisionnement |
| `get_products_to_discontinue()` | `dormant_days_threshold`, `days_lookback` | Fenêtre glissante | Produits à arrêter (dormants + faible vente) |

## 4. Purchases (achats)

Commandes fournisseurs : montants, volumes, évolution des prix d'achat. Contient aussi les méthodes « outils » partagées avec les fournisseurs.

### `PurchasesAnalyticsRepository` — Analytics (pages de rapport)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_purchases_detail()` | — | Snapshot | Détail de tous les achats |
| `get_purchase_items_detail()` | — | Snapshot | Détail ligne par ligne des achats |
| `get_purchases_by_period()` | `start_date`, `end_date`, `supplier_id` | Filtrable | Achats d'une période (fonction SQL `fn_purchases_by_period`) |
| `get_total_amount()` | `start_date`, `end_date` | Filtrable | Montant total sur la période |
| `get_total_count()` | `start_date`, `end_date` | Filtrable | Nombre d'éléments sur la période |
| `get_recent_purchases()` | `limit` | Snapshot | Dernières commandes (non filtré) |
| `get_avg_products_per_purchase()` | — | Snapshot | Nombre moyen de produits par commande |
| `get_amount_over_time()` | `start_date`, `end_date` | Filtrable | Montant acheté groupé par jour |
| `get_count_over_time()` | `start_date`, `end_date` | Filtrable | Nombre de commandes groupé par jour |
| `get_most_purchased_products()` | `limit`, `start_date`, `end_date` | Filtrable | Classement des produits par quantité achetée |
| `get_products_purchased_once()` | `start_date`, `end_date` | Filtrable | Produits achetés une seule fois |
| `get_purchases_by_category()` | `start_date`, `end_date` | Filtrable | Répartition des achats par catégorie |
| `get_products_biggest_price_change()` | `direction`, `limit`, `start_date`, `end_date` | Filtrable | Plus fortes variations de prix d'achat |
| `get_purchases_report()` | `start_date`, `end_date` | Filtrable | **Endpoint résumé** : agrège toutes les méthodes du domaine |

### `PurchasesConsultationRepository` — Consultation (outils agent)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_purchases_detail()` | — | Snapshot | Détail de tous les achats |
| `get_purchase_items_detail()` | — | Snapshot | Détail ligne par ligne des achats |
| `get_suppliers_list()` | — | Snapshot | Liste des fournisseurs |

### `PurchasesTraitementRepository` — Traitement (outils agent)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_purchases_by_period()` | `start_date`, `end_date`, `supplier_id` | Filtrable | Achats d'une période (fonction SQL `fn_purchases_by_period`) |
| `get_supplier_price_history()` | `product_id`, `supplier_id` | Lookup | Historique des prix d'un produit chez un fournisseur |
| `get_top_suppliers()` | `start_date`, `end_date`, `limit` | Filtrable | Classement des fournisseurs par volume d'achat |
| `get_compare_supplier_prices()` | `product_id` | Lookup | Comparaison des prix fournisseurs pour un produit |
| `get_supplier_summary()` | `supplier_id` | Lookup | Synthèse d'un fournisseur |
| `get_suppliers_no_recent_order()` | `days` | Fenêtre glissante | Fournisseurs sans commande récente |

### `PurchasesRecommandationRepository` — Recommandation (outils agent)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_best_supplier_for_product()` | `product_id` | Lookup | Meilleur fournisseur pour un produit |

## 5. Suppliers (fournisseurs)

Analyse des fournisseurs : classement, concentration, inactivité, géographie, économies potentielles.

### `SuppliersAnalyticsRepository` — Analytics (pages de rapport)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_suppliers_list()` | — | Snapshot | Liste des fournisseurs |
| `get_purchases_detail()` | — | Snapshot | Détail de tous les achats |
| `get_top_suppliers()` | `start_date`, `end_date`, `limit` | Filtrable | Classement des fournisseurs par volume d'achat |
| `get_best_supplier_for_product()` | `product_id` | Lookup | Meilleur fournisseur pour un produit |
| `get_suppliers_without_products()` | — | Snapshot | Fournisseurs sans aucun produit (structurel) |
| `get_suppliers_no_order_since()` | `days` | Fenêtre glissante | Fournisseurs inactifs depuis X jours |
| `get_supplier_concentration()` | `top_n`, `start_date`, `end_date` | Filtrable | % du volume d'achat chez les N premiers fournisseurs |
| `get_amount_by_supplier()` | `start_date`, `end_date` | Filtrable | Montant acheté par fournisseur |
| `get_suppliers_by_city()` | — | Snapshot | Distribution géographique (ville extraite de l'adresse) |
| `get_products_with_cheaper_alternative()` | `purchases_repo`, `limit`, `start_date`, `end_date` | Filtrable | Économies potentielles (croise achats + recommandation) – coûteux |
| `get_suppliers_report()` | `start_date`, `end_date` | Filtrable | **Endpoint résumé** : agrège tout sauf l'analyse coûteuse |

> ⚠️ **Logique métier à connaître.** `get_suppliers_by_city()` extrait la ville depuis le champ `address` (texte libre) en cherchant une correspondance avec la liste `MOROCCAN_CITIES` (`core/moroccan_cities.py`). Le résultat est approximatif si l'adresse est mal formatée ou si la ville n'est pas dans la liste.

## 6. Devis

Propositions commerciales (draft / sent / accepted / rejected) : conversion, pipeline, relances.

### `DevisAnalyticsRepository` — Analytics (pages de rapport)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_devis_detail()` | — | Snapshot | Liste des devis |
| `get_devis_items_detail()` | — | Snapshot | Détail ligne par ligne des devis |
| `get_devis_conversion_rate()` | `start_date`, `end_date` | Filtrable | Taux de conversion (fonction SQL `fn_devis_conversion_rate`) |
| `get_devis_avg_response_time()` | `start_date`, `end_date` | Filtrable | Délai moyen de réponse (SQL) |
| `get_devis_by_status_summary()` | `start_date`, `end_date` | Filtrable | Résumé par statut (SQL) |
| `get_devis_pending_followup()` | `days_since_sent` | Fenêtre glissante | Devis à relancer (SQL) |
| `get_devis_about_to_expire()` | `days_before` | Fenêtre glissante | Devis proches de l'expiration (SQL) |
| `get_total_count()` | `start_date`, `end_date` | Filtrable | Nombre d'éléments sur la période |
| `get_total_amount()` | `start_date`, `end_date` | Filtrable | Montant total sur la période |
| `get_devis_by_client()` | `limit`, `start_date`, `end_date` | Filtrable | Répartition des devis par client |
| `get_top_devis_by_amount()` | `limit`, `start_date`, `end_date` | Filtrable | Devis les plus importants |
| `get_most_proposed_products()` | `limit`, `start_date`, `end_date` | Filtrable | Produits les plus proposés |
| `get_conversion_summary()` | `start_date`, `end_date` | Filtrable | Taux de conversion (nombre + montant) |
| `get_conversion_over_time()` | `start_date`, `end_date` | Filtrable | Conversion groupée par jour |
| `get_status_breakdown()` | `start_date`, `end_date` | Filtrable | Répartition par statut |
| `get_accepted_vs_rejected_amounts()` | `start_date`, `end_date` | Filtrable | Montants acceptés vs refusés |
| `get_response_time_summary()` | `start_date`, `end_date` | Filtrable | Délai moyen de réponse |
| `get_pipeline_amount()` | `start_date`, `end_date` | Filtrable | Montant des devis non statués (draft + sent) |
| `get_followup_summary()` | `days_since_sent` | Fenêtre glissante | Devis à relancer (résumé) |
| `get_expiring_summary()` | `days_before` | Fenêtre glissante | Devis proches d'expirer (résumé) |
| `get_devis_report()` | `start_date`, `end_date` | Filtrable | **Endpoint résumé** : agrège toutes les méthodes du domaine |

### `DevisConsultationRepository` — Consultation (outils agent)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_devis_detail()` | — | Snapshot | Liste des devis |
| `get_devis_items_detail()` | — | Snapshot | Détail ligne par ligne des devis |

### `DevisTraitementRepository` — Traitement (outils agent)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_devis_conversion_rate()` | `start_date`, `end_date` | Filtrable | Taux de conversion (fonction SQL `fn_devis_conversion_rate`) |
| `get_devis_avg_response_time()` | `start_date`, `end_date` | Filtrable | Délai moyen de réponse (SQL) |
| `get_devis_by_status_summary()` | `start_date`, `end_date` | Filtrable | Résumé par statut (SQL) |

### `DevisRecommandationRepository` — Recommandation (outils agent)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_devis_pending_followup()` | `days_since_sent` | Fenêtre glissante | Devis à relancer (SQL) |
| `get_devis_about_to_expire()` | `days_before` | Fenêtre glissante | Devis proches de l'expiration (SQL) |

> ⚠️ **Distinction importante.** Un *devis* est une proposition commerciale non conclue, à ne pas confondre avec une *vente* (transaction réelle). Statuts possibles : `draft`, `sent`, `accepted`, `rejected`.

## 7. Sales (ventes)

Transactions de vente : segmentation RFM, performance catégories, ventes par ville, consultation détaillée.

### `SalesRepository`

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_rfm_raw()` | `start_date`, `end_date` | Filtrable | Données brutes RFM (récence, fréquence, montant) par client |
| `get_refund_rate_by_product()` | `start_date`, `end_date`, `limit` | Filtrable | Taux de remboursement par produit |
| `get_category_performance()` | `start_date`, `end_date` | Filtrable | Performance des catégories |
| `get_dormant_products()` | `days` | Fenêtre glissante | Produits dormants |
| `get_refund_rate_trend()` | `start_date`, `end_date`, `months` | Filtrable | Évolution mensuelle du taux de remboursement |
| `get_revenue_by_status()` | `start_date`, `end_date` | Filtrable | Chiffre d'affaires par statut de vente |
| `get_refund_rate_by_client()` | `start_date`, `end_date`, `limit` | Filtrable | Taux de remboursement par client |
| `get_sales_by_city()` | `start_date`, `end_date` | Filtrable | Ventes par ville (extraite de l'adresse) |

### `SalesConsultationRepository` — Consultation (outils agent)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_sales_by_period()` | `start_date`, `end_date`, `updated_start`, `updated_end`, `status`, `order_by`, `order_direction`, `limit` | Filtrable | Ventes d'une période (filtres statut, dates de MAJ, tri, limite) |
| `get_sale_lookup()` | `sale_id`, `reference` | Lookup | Recherche d'une vente par id ou référence |
| `get_sales_by_client()` | `client_id`, `client_name`, `client_email`, `client_phone`, `client_city`, `start_date`, `end_date`, `limit` | Filtrable | Ventes d'un client (id, nom, email, téléphone, ville) |
| `get_sales_by_amounts()` | `min_subtotal`, `max_subtotal`, `min_tax_rate`, `max_tax_rate`, `min_tax_amount`, `max_tax_amount`, `min_discount_amount`, `max_discount_amount`, `min_gross_total`, `max_gross_total`, `min_net_total`, `max_net_total`, `min_refunded_total`, `max_refunded_total`, `has_refund`, `start_date`, `end_date`, `order_by`, `order_direction`, `limit` | Filtrable | Ventes filtrées par fourchettes de montants/taxes/remises |
| `get_sale_items_by_sale()` | `sale_id`, `reference`, `sale_status` | Lookup | Lignes d'une vente |
| `get_sales_history_by_product()` | `product_id`, `product_name`, `product_reference`, `category_id`, `category_name`, `start_date`, `end_date`, `limit` | Filtrable | Historique des ventes d'un produit |
| `get_sale_items_by_client()` | `client_id`, `client_name`, `limit` | Lookup | Lignes de vente d'un client |
| `get_sale_items_by_metrics()` | `min_ordered_quantity`, `max_ordered_quantity`, `min_subtotal`, `max_subtotal`, `refund_type`, `min_refund_rate`, `max_refund_rate`, `order_by`, `order_direction`, `limit` | Snapshot | Lignes de vente filtrées par quantité, montant, taux de remboursement |

### `SalesTraitementRepository` — Traitement (outils agent)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_sales_statistics_summary()` | `start_date`, `end_date`, `client_id`, `product_id`, `status`, `category_id` | Filtrable | Statistiques de ventes (résumé, fonction SQL) |
| `get_sales_statistics_by_period()` | `start_date`, `end_date`, `period`, `client_id`, `product_id`, `status`, `category_id`, `limit` | Filtrable | Statistiques de ventes par jour/semaine/mois |

### `SalesRecommandationRepository` — Recommandation (outils agent)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_sales_decline_alert()` | `current_start`, `current_end`, `previous_start`, `previous_end`, `threshold_pct`, `limit` | Filtrable | Alerte de baisse des ventes entre deux périodes |

## 8. Refunds (remboursements)

Remboursements : taux, motifs, produits et clients à risque.

### `RefundsRepository`

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_refunds_summary()` | `start_date`, `end_date` | Filtrable | Résumé des remboursements |
| `get_refunds_by_reason()` | `start_date`, `end_date` | Filtrable | Remboursements par motif |
| `get_refund_trend()` | `start_date`, `end_date`, `months` | Filtrable | Évolution mensuelle des remboursements |
| `get_top_refunded_products()` | `limit`, `start_date`, `end_date` | Filtrable | Produits les plus remboursés |
| `get_top_clients_by_refunds()` | `limit`, `start_date`, `end_date` | Filtrable | Clients ayant le plus de remboursements |
| `get_refund_rate_global()` | `start_date`, `end_date` | Filtrable | Taux de remboursement global |
| `get_refund_rate_by_product()` | `limit`, `start_date`, `end_date` | Filtrable | Taux de remboursement par produit |
| `get_high_return_products()` | `threshold_pct`, `limit`, `start_date`, `end_date` | Filtrable | Produits dépassant un seuil de taux de retour |

### `RefundsConsultationRepository` — Consultation (outils agent)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_refunds_by_period()` | `start_date`, `end_date`, `order_by`, `order_direction`, `limit` | Filtrable | Remboursements d'une période (tri, limite) |
| `get_refund_lookup()` | `refund_id`, `sale_reference` | Lookup | Recherche d'un remboursement par id ou référence de vente |
| `get_refunds_by_client()` | `client_id`, `client_name`, `start_date`, `end_date`, `limit` | Filtrable | Remboursements d'un client |
| `get_refunds_by_reason_text()` | `reason`, `limit` | Snapshot | Remboursements par motif (recherche texte) |
| `get_refund_items_by_refund()` | `refund_id`, `sale_reference` | Lookup | Lignes d'un remboursement |
| `get_refund_items_by_product()` | `product_id`, `product_name`, `product_reference`, `start_date`, `end_date`, `limit` | Filtrable | Lignes de remboursement d'un produit |
| `get_refund_items_by_client()` | `client_id`, `client_name`, `limit` | Lookup | Lignes de remboursement d'un client |

### `RefundsTraitementRepository` — Traitement (outils agent)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_refunds_period_detail()` | `start_date`, `end_date`, `limit` | Filtrable | Détail des remboursements sur une période (SQL) |
| `get_refund_rate_by_product()` | `start_date`, `end_date`, `limit` | Filtrable | Taux de remboursement par produit |
| `get_refunds_by_reason()` | `start_date`, `end_date` | Filtrable | Remboursements par motif |
| `get_total_refunded()` | `start_date`, `end_date` | Filtrable | Montant total remboursé |

### `RefundsRecommandationRepository` — Recommandation (outils agent)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_products_high_return_rate()` | `threshold_pct`, `limit` | Snapshot | Produits à fort taux de retour |

## 9. Clients

Base clients : fidélité, churn, RFM, valeur vie, relances.

### `ClientsAnalyticsRepository` — Analytics (pages de rapport)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_clients_count()` | — | Snapshot | Nombre total de clients |
| `get_new_clients_count()` | `days` | Fenêtre glissante | Nouveaux clients sur N jours |
| `get_sales_status_summary()` | `start_date`, `end_date` | Filtrable | Ventes par statut |
| `get_global_revenue_summary()` | `start_date`, `end_date` | Filtrable | Chiffre d'affaires global |
| `get_top_clients_overall()` | `start_date`, `end_date`, `limit` | Filtrable | Meilleurs clients (CA) |
| `get_repeat_clients_summary()` | `start_date`, `end_date`, `min_orders` | Filtrable | Résumé des clients fidèles |
| `get_inactive_clients_summary()` | `days_inactive` | Fenêtre glissante | Résumé des clients inactifs |
| `get_clients_to_recontact_summary()` | `days_inactive`, `min_historical_amount` | Fenêtre glissante | Résumé des clients à relancer |
| `get_rfm_segmentation()` | — | Snapshot | Liste complète des clients avec segment RFM |
| `get_rfm_summary()` | — | Snapshot | Répartition des clients par segment RFM |
| `get_clients_evolution()` | `start_date`, `end_date`, `months` | Filtrable | Clients actifs par mois |
| `get_clients_without_orders()` | — | Snapshot | Clients sans aucune commande |
| `get_repeat_rate()` | — | Snapshot | Taux de réachat global (≥ 2 commandes) |
| `get_avg_time_between_orders()` | — | Snapshot | Délai moyen entre deux commandes |
| `get_churn_rate()` | `days_inactive` | Fenêtre glissante | Taux de churn (clients inactifs / actifs) |
| `get_lifetime_value_distribution()` | — | Snapshot | Distribution de la valeur vie client |
| `get_customer_health_score()` | — | Snapshot | Score de santé client (récence, fréquence, valeur) |
| `get_clients_by_city()` | — | Snapshot | Répartition géographique des clients |
| `get_clients_kpi_summary()` | `start_date`, `end_date` | Filtrable | Dashboard complet des KPI clients |
| `get_inactive_clients_list()` | `days_inactive` | Fenêtre glissante | Liste des clients inactifs |
| `get_clients_to_recontact_list()` | `days_inactive`, `min_historical_amount` | Fenêtre glissante | Liste des clients à relancer |
| `get_repeat_clients_list()` | `min_orders` | Snapshot | Liste des clients fidèles |
| `get_client_details()` | `client_id` | Lookup | Détails complets d'un client |
| `get_clients_with_high_refund_rate()` | `limit` | Snapshot | Clients au plus fort ratio remboursé / acheté |

### `ClientsConsultationRepository` — Consultation (outils agent)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_clients_list()` | `client_id`, `client_name`, `client_email`, `client_phone`, `limit` | Lookup | Recherche de clients (id, nom, email, téléphone) |
| `get_client_purchase_history()` | `client_id`, `client_name`, `sale_status`, `start_date`, `end_date`, `limit` | Filtrable | Historique d'achats d'un client |

### `ClientsTraitementRepository` — Traitement (outils agent)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_client_lifetime_value()` | `client_id` | Lookup | Valeur vie d'un client (SQL) |
| `get_top_clients()` | `start_date`, `end_date`, `limit` | Filtrable | Meilleurs clients sur une période (SQL) |
| `get_inactive_clients()` | `days_inactive`, `limit` | Fenêtre glissante | Clients inactifs (SQL) |

### `ClientsRecommandationRepository` — Recommandation (outils agent)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_clients_to_recontact()` | `days_inactive`, `min_historical_amount`, `limit` | Fenêtre glissante | Clients à relancer (SQL) |

## 10. Payments (paiements)

Encaissements, soldes, retards de paiement, prévision de trésorerie. Résout noms de clients / références de ventes en identifiants.

### `PaymentsAnalyticsRepository` — Analytics (pages de rapport)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_payments_overview_kpis()` | `start_date`, `end_date` | Filtrable | Vue d'ensemble : encaissé, restant dû, répartition |
| `get_payment_status_distribution()` | `start_date`, `end_date` | Filtrable | Répartition des paiements par statut |
| `get_payment_method_distribution()` | `start_date`, `end_date` | Filtrable | Répartition des paiements par méthode |
| `get_collection_trend()` | `start_date`, `end_date`, `days` | Filtrable | Évolution hebdomadaire des encaissements |
| `get_top_debtor_clients()` | `start_date`, `end_date`, `limit` | Filtrable | Clients classés par solde dû |
| `get_overdue_kpi()` | `threshold_days` | Fenêtre glissante | Nombre et montant des ventes en retard |
| `get_recurring_late_clients_kpi()` | `threshold_days`, `min_occurrences` | Fenêtre glissante | KPI clients récidivistes en retard |
| `get_cash_collected_kpi()` | `start_date`, `end_date` | Filtrable | KPI encaissement (total, nb, moyenne) |

### `PaymentsConsultationRepository` — Consultation (outils agent)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `resolve_client()` | `name` | Lookup | Résout un nom de client en `client_id` |
| `resolve_sale()` | `reference` | Lookup | Résout une référence de vente en `sale_id` |
| `search_client_by_name()` | `name` | Lookup | Recherche de client par nom (partiel) |
| `search_sale_by_reference()` | `reference` | Lookup | Recherche de vente par référence ou nom de client |
| `get_payments_detail()` | `sale_id`, `client_id`, `payment_id`, `client_name`, `sale_reference`, `limit` | Lookup | Détail brut des tranches versées |
| `get_client_sales_summary()` | `client_id`, `client_name` | Lookup | Détail des ventes d'un client avec soldes |
| `get_sales_with_remaining_balance()` | `limit` | Snapshot | Ventes ayant encore un solde à payer |
| `get_sale_allocations()` | `sale_id`, `sale_reference` | Lookup | Tranches versées pour une vente |

### `PaymentsTraitementRepository` — Traitement (outils agent)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_payments_by_method()` | `start_date`, `end_date` | Filtrable | Paiements par méthode |
| `get_payments_by_status()` | `start_date`, `end_date` | Filtrable | Paiements par statut |
| `get_client_balance()` | `client_id`, `client_name` | Lookup | Solde global d'un client |
| `get_cash_collected()` | `start_date`, `end_date` | Filtrable | Encaissements sur une période |
| `get_sale_balance()` | `sale_id`, `sale_reference` | Lookup | Solde restant d'une vente |
| `get_sale_payment_alerts()` | `threshold_days` | Fenêtre glissante | Ventes en retard avec message d'alerte |

### `PaymentsRecommandationRepository` — Recommandation (outils agent)

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_recurring_late_clients()` | `threshold_days`, `min_occurrences` | Fenêtre glissante | Clients récidivistes en retard de paiement |
| `get_cashflow_forecast()` | `days_ahead` | Fenêtre glissante | Prévision de trésorerie à N jours |
| `check_client_recurring_late_status()` | `client_id`, `client_name`, `threshold_days`, `min_occurrences` | Fenêtre glissante | Un client est-il récidiviste ? |

## 11. Dashboard

KPI de la page d'accueil.

### `DashboardRepository`

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_global_counts()` | — | Snapshot | Compteurs globaux |
| `get_current_month_summary()` | — | Snapshot | Résumé du mois courant |
| `get_category_names()` | — | Snapshot | Noms des catégories |
| `get_top_products_month()` | `limit` | Snapshot | Top produits du mois |
| `get_best_client_month()` | — | Snapshot | Meilleur client du mois |
| `get_top_clients_month()` | `limit` | Snapshot | Top clients du mois |
| `get_revenue_trend()` | `months` | Fenêtre glissante | Évolution du chiffre d'affaires |
| `get_sales_count_trend()` | `months` | Fenêtre glissante | Évolution du nombre de ventes |

## 12. Utilisateurs

Accès à la table `users` (authentification, profil).

### `UserRepository`

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_by_email()` | `email` | Lookup | Utilisateur par email |
| `get_by_id()` | `user_id` | Lookup | Utilisateur par id |
| `create()` | `name`, `email`, `hashed_password` | Lookup | Création (⚠️ incompatible avec le modèle, voir points-attention.md) |
| `change_password()` | `user`, `current_password`, `new_password` | Snapshot | Change le mot de passe (vérifie l'ancien) |
| `update_profile()` | `user`, `first_name`, `last_name`, `email` | Lookup | Met à jour prénom, nom, email |
| `reset_password()` | `email`, `new_password` | Lookup | Réinitialise le mot de passe d'un email |

> ⚠️ `AuthService.register` appelle `UserRepository.register`, qui n'existe pas (voir `points-attention.md`).

## 13. Administration des utilisateurs

CRUD réservé aux admins.

### `AdminUserRepository`

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `list_users()` | — | Snapshot | Liste des utilisateurs |
| `get_by_email()` | `email` | Lookup | Utilisateur par email |
| `create_user()` | `first_name`, `last_name`, `email`, `role` | Lookup | Crée un utilisateur avec mot de passe temporaire |
| `update_role()` | `user_id`, `role` | Lookup | Change le rôle |
| `delete_user()` | `user_id` | Lookup | Supprime un utilisateur |

## 14. Alertes

Alertes métier et historique de notification.

### `AlertNotificationRepository`

| Méthode | Paramètres | Type | Description |
|---|---|---|---|
| `get_active_alerts()` | — | Snapshot | Appelle `fn_get_all_active_alerts()` |
| `filter_unsent_alerts()` | `alerts` | Snapshot | Retire les alertes déjà notifiées |
| `mark_alerts_as_sent()` | `alerts` | Snapshot | Marque les alertes comme envoyées |
| `get_all_user_emails()` | — | Snapshot | Emails de tous les utilisateurs |
