# Fonctions du schéma `agent_reporting`

Le schéma `agent_reporting` contient **44 fonctions SQL**, organisées en deux couches :

- **traitement** : calculs et statistiques (totaux, classements, agrégations par période) ;
- **recommandation** : détection des situations à traiter (clients à relancer, devis qui expirent, risques de rupture…). Elles alimentent le module d'alertes.

Toutes se consomment avec un `SELECT` (connecté avec `agent_user` ou `backend_user`) :

```sql
SELECT * FROM agent_reporting.fn_top_clients('2026-07-01', '2026-07-31', 5);
```

Les paramètres marqués « défaut » sont facultatifs. Les dates se donnent au format `'AAAA-MM-JJ'`. Les exemples ci-dessous utilisent des valeurs de la base de démonstration : adaptez-les à vos données.

> Les vues utilisées par ces fonctions sont décrites dans [vues.md](vues.md).

## Sommaire

- [Alertes](#alertes)
- [Clients](#clients)
- [Devis](#devis)
- [Paiements](#paiements)
- [Produits et stock](#produits-et-stock)
- [Achats et fournisseurs](#achats-et-fournisseurs)
- [Remboursements](#remboursements)
- [Ventes](#ventes)

## Alertes

### `fn_get_all_active_alerts`

Regroupe toutes les alertes actives du système dans un format unique (type, clé, message, sévérité `high` / `medium` / `low`). C'est la fonction utilisée par le module d'alertes e-mail.

- **Couche** : alerte — **Fichier** : `aletre_email.sql`
- **Paramètres** : aucun
- **Colonnes retournées** : `alert_type`, `alert_key`, `alert_message`, `severity`

```sql
SELECT * FROM agent_reporting.fn_get_all_active_alerts();
```

## Clients

### `fn_client_lifetime_value`

Valeur d'un client sur toute sa durée de vie : total dépensé, nombre de commandes, panier moyen.

- **Couche** : traitement — **Fichier** : `clients/traitement/fn_client_lifetime_value.sql`
- **Paramètres** :
  - `p_client_id` (integer)
- **Colonnes retournées** : `client_id`, `client_name`, `total_spent`, `nb_orders`, `avg_order_value`

```sql
SELECT * FROM agent_reporting.fn_client_lifetime_value(3);
```

### `fn_inactive_clients`

Clients sans achat depuis plus de N jours, avec la date de leur dernière vente.

- **Couche** : traitement — **Fichier** : `clients/traitement/fn_inactive_clients.sql`
- **Paramètres** :
  - `p_days_inactive` (integer)
- **Colonnes retournées** : `client_id`, `client_name`, `last_sale_date`, `days_since_last_sale`

```sql
SELECT * FROM agent_reporting.fn_inactive_clients(60);
```

### `fn_top_clients`

Meilleurs clients d'une période, classés par montant dépensé.

- **Couche** : traitement — **Fichier** : `clients/traitement/fn_top_clients.sql`
- **Paramètres** :
  - `p_start_date` (date)
  - `p_end_date` (date)
  - `p_limit` (integer, défaut 10)
- **Colonnes retournées** : `client_id`, `client_name`, `nb_orders`, `total_spent`

```sql
SELECT * FROM agent_reporting.fn_top_clients('2026-07-01', '2026-07-31');
```

### `fn_clients_to_recontact`

Clients inactifs depuis N jours (60 par défaut) à relancer, avec filtre facultatif sur le montant historique minimum.

- **Couche** : recommandation — **Fichier** : `clients/recommendation/fn_clients_to_recontact.sql`
- **Paramètres** :
  - `p_days_inactive` (integer, défaut 60)
  - `p_min_historical_amount` (numeric, défaut NULL)
- **Colonnes retournées** : `client_id`, `client_name`, `last_sale_date`, `total_spent`

```sql
SELECT * FROM agent_reporting.fn_clients_to_recontact();
```

## Devis

### `fn_devis_avg_response_time`

Délai moyen (min, max) de réponse des clients aux devis, en jours.

- **Couche** : traitement — **Fichier** : `devis/traitement/fn_devis_avg_response_time.sql`
- **Paramètres** :
  - `p_start_date` (date, défaut NULL)
  - `p_end_date` (date, défaut NULL)
- **Colonnes retournées** : `avg_response_days`, `min_response_days`, `max_response_days`, `responded_count`

```sql
SELECT * FROM agent_reporting.fn_devis_avg_response_time();
```

### `fn_devis_by_status_summary`

Nombre et montant des devis par statut sur une période.

- **Couche** : traitement — **Fichier** : `devis/traitement/fn_devis_by_status_summary.sql`
- **Paramètres** :
  - `p_start_date` (date)
  - `p_end_date` (date)
- **Colonnes retournées** : `status`, `devis_count`, `total_amount`

```sql
SELECT * FROM agent_reporting.fn_devis_by_status_summary('2026-07-01', '2026-07-31');
```

### `fn_devis_conversion_rate`

Taux de conversion des devis (acceptés / total) sur une période.

- **Couche** : traitement — **Fichier** : `devis/traitement/fn_devis_conversion_rate.sql`
- **Paramètres** :
  - `p_start_date` (date)
  - `p_end_date` (date)
- **Colonnes retournées** : `total_devis`, `accepted_count`, `rejected_count`, `pending_count`, `conversion_rate_pct`

```sql
SELECT * FROM agent_reporting.fn_devis_conversion_rate('2026-07-01', '2026-07-31');
```

### `fn_devis_about_to_expire`

Devis qui arrivent à expiration dans les N prochains jours (3 par défaut).

- **Couche** : recommandation — **Fichier** : `devis/recommendation/fn_devis_about_to_expire.sql`
- **Paramètres** :
  - `p_days_before` (integer, défaut 3)
- **Colonnes retournées** : `devis_id`, `devis_reference`, `client_id`, `client_name`, `total`, `expires_at`, `days_until_expiry`

```sql
SELECT * FROM agent_reporting.fn_devis_about_to_expire();
```

### `fn_devis_pending_followup`

Devis envoyés depuis N jours ou plus (7 par défaut) sans réponse du client.

- **Couche** : recommandation — **Fichier** : `devis/recommendation/fn_devis_pending_followup.sql`
- **Paramètres** :
  - `p_days_since_sent` (integer, défaut 7)
- **Colonnes retournées** : `devis_id`, `devis_reference`, `client_id`, `client_name`, `total`, `sent_at`, `days_since_sent`

```sql
SELECT * FROM agent_reporting.fn_devis_pending_followup();
```

## Paiements

### `fn_cash_collected`

Total encaissé, nombre de transactions et montant moyen, sur une période facultative.

- **Couche** : traitement — **Fichier** : `payments/traitement/fn_cash_collected.sql`
- **Paramètres** :
  - `start_date` (date, défaut NULL)
  - `end_date` (date, défaut NULL)
- **Colonnes retournées** : `total_collected`, `transaction_count`, `average_transaction`

```sql
SELECT * FROM agent_reporting.fn_cash_collected();
```

### `fn_client_balance`

Solde d'un client : ventes nettes, total payé et reste dû.

- **Couche** : traitement — **Fichier** : `payments/traitement/fn_client_balance.sql`
- **Paramètres** :
  - `p_client_id` (bigint)
- **Colonnes retournées** : `client_id`, `client_name`, `sales_count`, `total_net_sold`, `total_paid`, `balance_due`

```sql
SELECT * FROM agent_reporting.fn_client_balance(3);
```

### `fn_payments_by_method`

Répartition des encaissements par méthode de paiement (cash, stripe, virement).

- **Couche** : traitement — **Fichier** : `payments/traitement/fn_payments_by_method.sql`
- **Paramètres** :
  - `start_date` (date, défaut NULL)
  - `end_date` (date, défaut NULL)
- **Colonnes retournées** : `method`, `total_collected`, `transaction_count`, `percentage`

```sql
SELECT * FROM agent_reporting.fn_payments_by_method();
```

### `fn_payments_by_status`

Répartition des paiements par statut.

- **Couche** : traitement — **Fichier** : `payments/traitement/fn_payments_by_status.sql`
- **Paramètres** :
  - `start_date` (date, défaut NULL)
  - `end_date` (date, défaut NULL)
- **Colonnes retournées** : `status`, `payment_count`, `total_collected`, `percentage`

```sql
SELECT * FROM agent_reporting.fn_payments_by_status();
```

### `fn_sale_balance`

Reste à payer sur une vente, avec un message explicatif.

- **Couche** : traitement — **Fichier** : `payments/traitement/fn_sale_balance.sql`
- **Paramètres** :
  - `p_sale_id` (bigint)
- **Colonnes retournées** : `sale_id`, `sale_reference`, `client_id`, `client_name`, `net_total`, `total_applied`, `remaining_balance`, `payment_message`

```sql
SELECT * FROM agent_reporting.fn_sale_balance(24);
```

### `fn_cashflow_forecast`

Prévision d'encaissement sur N jours (30 par défaut) à partir du restant dû et du rythme d'encaissement des 30 derniers jours, avec un niveau de confiance.

- **Couche** : recommandation — **Fichier** : `payments/recommandation/fn_cashflow_forecast.sql`
- **Paramètres** :
  - `days_ahead` (integer, défaut 30)
- **Colonnes retournées** : `total_outstanding`, `avg_daily_collection_last_30d`, `projected_collection`, `projection_confidence`, `forecast_message`

```sql
SELECT * FROM agent_reporting.fn_cashflow_forecast();
```

### `fn_recurring_late_clients`

Clients en retard de paiement de façon récurrente (seuil de jours et nombre minimum d'occurrences), avec une recommandation.

- **Couche** : recommandation — **Fichier** : `payments/recommandation/fn_recurring_late_clients.sql`
- **Paramètres** :
  - `threshold_days` (integer, défaut 30)
  - `min_occurrences` (integer, défaut 2)
- **Colonnes retournées** : `client_id`, `client_name`, `client_phone`, `client_email`, `late_occurrences`, `fully_unpaid_count`, `partially_paid_count`, `total_unpaid_amount`, `concerned_sales`, `recommendation`

```sql
SELECT * FROM agent_reporting.fn_recurring_late_clients();
```

### `fn_sale_payment_alerts`

Ventes impayées ou partiellement payées depuis plus de N jours (30 par défaut), avec un message d'alerte.

- **Couche** : recommandation — **Fichier** : `payments/recommandation/fn_sale_payment_alerts.sql`
- **Paramètres** :
  - `threshold_days` (integer, défaut 30)
- **Colonnes retournées** : `sale_id`, `sale_reference`, `client_id`, `client_name`, `client_phone`, `sale_date`, `days_elapsed`, `net_total`, `total_paid`, `remaining_balance`, `alert_type`, `alert_message`

```sql
SELECT * FROM agent_reporting.fn_sale_payment_alerts();
```

## Produits et stock

### `fn_fast_moving_products`

Produits à forte rotation avec le risque de rupture et la quantité de réapprovisionnement suggérée.

- **Couche** : traitement — **Fichier** : `products_stock/traitement/fn_fast_moving_products.sql`
- **Paramètres** :
  - `p_days_lookback` (integer, défaut 30)
  - `p_target_coverage_days` (integer, défaut 30)
  - `p_limit` (integer, défaut 10)
- **Colonnes retournées** : `product_id`, `product_reference`, `product_name`, `current_stock`, `alert_stock_threshold`, `avg_daily_sales`, `days_until_stockout`, `suggested_reorder_qty`

```sql
SELECT * FROM agent_reporting.fn_fast_moving_products();
```

### `fn_product_sales_velocity`

Vitesse de vente d'un produit (ventes moyennes par jour, jours avant rupture).

- **Couche** : traitement — **Fichier** : `products_stock/traitement/fn_product_sales_velocity.sql`
- **Paramètres** :
  - `p_product_id` (bigint)
  - `p_days_lookback` (integer, défaut 30)
- **Colonnes retournées** : `product_id`, `product_reference`, `product_name`, `current_stock`, `alert_stock_threshold`, `avg_daily_sales`, `days_until_stockout`, `suggested_reorder_qty`

```sql
SELECT * FROM agent_reporting.fn_product_sales_velocity(4);
```

### `fn_product_stock_value`

Nombre de produits, unités en stock et valeur totale du stock, avec filtres facultatifs par catégorie et fournisseur.

- **Couche** : traitement — **Fichier** : `products_stock/traitement/fn_product_stock_value.sql`
- **Paramètres** :
  - `p_category_id` (bigint, défaut NULL)
  - `p_supplier_id` (bigint, défaut NULL)
- **Colonnes retournées** : `total_products`, `total_stock_units`, `total_stock_value`

```sql
SELECT * FROM agent_reporting.fn_product_stock_value();
```

### `fn_products_no_movement`

Produits sans mouvement de stock depuis N jours (60 par défaut).

- **Couche** : traitement — **Fichier** : `products_stock/traitement/fn_products_no_movement.sql`
- **Paramètres** :
  - `p_days` (integer, défaut 60)
- **Colonnes retournées** : `product_id`, `product_name`, `product_reference`, `current_stock`, `last_movement_date`, `days_since_last_movement`

```sql
SELECT * FROM agent_reporting.fn_products_no_movement();
```

### `fn_stock_movements_by_period`

Mouvements de stock sur une période, filtrables par produit et par type (`in` / `out`).

- **Couche** : traitement — **Fichier** : `products_stock/traitement/fn_stock_movements_by_period.sql`
- **Paramètres** :
  - `p_start_date` (date)
  - `p_end_date` (date)
  - `p_product_id` (bigint, défaut NULL)
  - `p_movement_type` (varchar, défaut NULL)
- **Colonnes retournées** : `movement_id`, `product_id`, `product_name`, `movement_quantity`, `movement_type`, `movement_source`, `movement_date`

```sql
SELECT * FROM agent_reporting.fn_stock_movements_by_period('2026-07-01', '2026-07-31');
```

### `fn_stock_report_core`

Fonction interne centrale du rapport de stock (stock courant, ventes moyennes par jour, jours avant rupture, quantité à commander). Utilisée par `fn_fast_moving_products`, `fn_product_sales_velocity` et `fn_reorder_suggestions`.

- **Couche** : traitement — **Fichier** : `products_stock/traitement/fn_stock_report_core.sql`
- **Paramètres** :
  - `p_product_id` (bigint)
  - `p_only_below_alert` (boolean)
  - `p_days_lookback` (integer)
  - `p_target_coverage_days` (integer)
- **Colonnes retournées** : `product_id`, `product_reference`, `product_name`, `current_stock`, `alert_stock_threshold`, `avg_daily_sales`, `days_until_stockout`, `suggested_reorder_qty`

```sql
SELECT * FROM agent_reporting.fn_stock_report_core(NULL, false, 30, 30);
```

### `fn_stock_value_by_category`

Valeur du stock regroupée par catégorie.

- **Couche** : traitement — **Fichier** : `products_stock/traitement/fn_stock_value_by_category.sql`
- **Paramètres** : aucun
- **Colonnes retournées** : `category_name`, `total_products`, `total_stock_units`, `total_stock_value`

```sql
SELECT * FROM agent_reporting.fn_stock_value_by_category();
```

### `fn_products_to_discontinue`

Produits dormants candidats à l'arrêt, avec marge, ventes moyennes, jours depuis la dernière vente et raison.

- **Couche** : recommandation — **Fichier** : `products_stock/recommandation/fn_products_to_discontinue.sql`
- **Paramètres** :
  - `p_dormant_days_threshold` (integer, défaut 90)
  - `p_days_lookback` (integer, défaut 90)
- **Colonnes retournées** : `product_id`, `product_name`, `current_stock`, `product_price`, `avg_purchase_price`, `margin`, `avg_daily_sales`, `days_since_last_sale`, `discontinue_reason`

```sql
SELECT * FROM agent_reporting.fn_products_to_discontinue();
```

### `fn_reorder_suggestions`

Suggestions de réapprovisionnement pour les produits sous leur seuil d'alerte.

- **Couche** : recommandation — **Fichier** : `products_stock/recommandation/fn_reorder_suggestions.sql`
- **Paramètres** :
  - `p_days_lookback` (integer, défaut 30)
  - `p_target_coverage_days` (integer, défaut 30)
- **Colonnes retournées** : `product_id`, `product_reference`, `product_name`, `current_stock`, `alert_stock_threshold`, `avg_daily_sales`, `days_until_stockout`, `suggested_reorder_qty`

```sql
SELECT * FROM agent_reporting.fn_reorder_suggestions();
```

## Achats et fournisseurs

### `fn_compare_supplier_prices`

Compare les prix des fournisseurs pour un produit (moyen, minimum, maximum, nombre d'achats).

- **Couche** : traitement — **Fichier** : `purchases-suppliers/traitement/fn_compare_supplier_prices.sql`
- **Paramètres** :
  - `p_product_id` (bigint)
- **Colonnes retournées** : `supplier_id`, `supplier_name`, `avg_price`, `min_price`, `max_price`, `purchase_count`, `last_purchase_date`

```sql
SELECT * FROM agent_reporting.fn_compare_supplier_prices(4);
```

### `fn_purchases_by_period`

Achats d'une période (nombre et montant), avec filtre facultatif par fournisseur.

- **Couche** : traitement — **Fichier** : `purchases-suppliers/traitement/fn_purchases_by_period.sql`
- **Paramètres** :
  - `p_start_date` (date)
  - `p_end_date` (date)
  - `p_supplier_id` (bigint, défaut NULL)
- **Colonnes retournées** : `total_purchases`, `total_amount`, `supplier_id`, `supplier_name`

```sql
SELECT * FROM agent_reporting.fn_purchases_by_period('2026-07-01', '2026-07-31');
```

### `fn_supplier_price_history`

Historique des prix d'achat d'un produit, avec filtre facultatif par fournisseur.

- **Couche** : traitement — **Fichier** : `purchases-suppliers/traitement/fn_supplier_price_history.sql`
- **Paramètres** :
  - `p_product_id` (bigint)
  - `p_supplier_id` (bigint, défaut NULL)
- **Colonnes retournées** : `purchase_date`, `supplier_id`, `supplier_name`, `unit_price`, `quantity`

```sql
SELECT * FROM agent_reporting.fn_supplier_price_history(4);
```

### `fn_supplier_summary`

Synthèse d'un fournisseur : montant acheté, nombre de commandes, nombre de produits, dernière commande.

- **Couche** : traitement — **Fichier** : `purchases-suppliers/traitement/fn_supplier_summary.sql`
- **Paramètres** :
  - `p_supplier_id` (bigint)
- **Colonnes retournées** : `supplier_id`, `supplier_name`, `total_purchased`, `total_purchase_orders`, `product_count`, `last_order_date`

```sql
SELECT * FROM agent_reporting.fn_supplier_summary(1);
```

### `fn_suppliers_no_recent_order`

Fournisseurs sans commande depuis N jours (90 par défaut).

- **Couche** : traitement — **Fichier** : `purchases-suppliers/traitement/fn_suppliers_no_recent_order.sql`
- **Paramètres** :
  - `p_days` (integer, défaut 90)
- **Colonnes retournées** : `supplier_id`, `supplier_name`, `last_order_date`, `days_since_last_order`

```sql
SELECT * FROM agent_reporting.fn_suppliers_no_recent_order();
```

### `fn_top_suppliers`

Principaux fournisseurs d'une période, classés par montant acheté.

- **Couche** : traitement — **Fichier** : `purchases-suppliers/traitement/fn_top_suppliers.sql`
- **Paramètres** :
  - `p_start_date` (date)
  - `p_end_date` (date)
  - `p_limit` (integer, défaut 10)
- **Colonnes retournées** : `supplier_id`, `supplier_name`, `total_purchases`, `total_amount`

```sql
SELECT * FROM agent_reporting.fn_top_suppliers('2026-07-01', '2026-07-31');
```

### `fn_best_supplier_for_product`

Meilleur fournisseur pour un produit (identifié par son id ou son nom), avec un score de recommandation.

- **Couche** : recommandation — **Fichier** : `purchases-suppliers/recommandation/fn_best_supplier_for_product.sql`
- **Paramètres** :
  - `p_product_id` (bigint, défaut NULL)
  - `p_product_name` (varchar, défaut NULL)
- **Colonnes retournées** : `supplier_id`, `supplier_name`, `avg_price`, `purchase_count`, `last_purchase_date`, `recommendation_score`

```sql
SELECT * FROM agent_reporting.fn_best_supplier_for_product();
```

## Remboursements

### `fn_products_high_return_rate_period`

Produits dont le taux de retour dépasse un seuil (10 % par défaut) sur une période facultative.

- **Couche** : traitement — **Fichier** : `refunds/traitement/fn_products_high_return_rate_period.sql`
- **Paramètres** :
  - `p_start_date` (date, défaut NULL)
  - `p_end_date` (date, défaut NULL)
  - `p_threshold_pct` (numeric, défaut 10)
- **Colonnes retournées** : `product_id`, `product_name`, `quantity_sold`, `quantity_refunded`, `refund_rate_pct`

```sql
SELECT * FROM agent_reporting.fn_products_high_return_rate_period();
```

### `fn_refund_rate_by_product`

Taux de remboursement par produit sur une période facultative.

- **Couche** : traitement — **Fichier** : `refunds/traitement/fn_refund_rate_by_product.sql`
- **Paramètres** :
  - `p_start_date` (date, défaut NULL)
  - `p_end_date` (date, défaut NULL)
- **Colonnes retournées** : `product_id`, `product_name`, `quantity_sold`, `quantity_refunded`, `refund_rate_pct`

```sql
SELECT * FROM agent_reporting.fn_refund_rate_by_product();
```

### `fn_refund_rate_global`

Taux de remboursement global (quantités vendues et remboursées) sur une période facultative.

- **Couche** : traitement — **Fichier** : `refunds/traitement/fn_refund_rate_global.sql`
- **Paramètres** :
  - `p_start_date` (date, défaut NULL)
  - `p_end_date` (date, défaut NULL)
- **Colonnes retournées** : `quantity_sold`, `quantity_refunded`, `refund_rate_pct`

```sql
SELECT * FROM agent_reporting.fn_refund_rate_global();
```

### `fn_refunds_by_reason`

Remboursements regroupés par motif : nombre et montant.

- **Couche** : traitement — **Fichier** : `refunds/traitement/fn_refunds_by_reason.sql`
- **Paramètres** :
  - `p_start_date` (date)
  - `p_end_date` (date)
- **Colonnes retournées** : `reason`, `refund_count`, `total_amount`

```sql
SELECT * FROM agent_reporting.fn_refunds_by_reason('2026-07-01', '2026-07-31');
```

### `fn_refunds_period_detail`

Détail des remboursements d'une période (vente, client, produit, quantités, montants).

- **Couche** : traitement — **Fichier** : `refunds/traitement/fn_refunds_period_detail.sql`
- **Paramètres** :
  - `p_start_date` (date)
  - `p_end_date` (date)
- **Colonnes retournées** : `sale_reference`, `client_name`, `refund_id`, `refund_total`, `refunded_at`, `reason`, `product_name`, `product_price`, `quantity`, `item_total`

```sql
SELECT * FROM agent_reporting.fn_refunds_period_detail('2026-07-01', '2026-07-31');
```

### `fn_total_refunded`

Montant total remboursé sur une période (renvoie un seul nombre).

- **Couche** : traitement — **Fichier** : `refunds/traitement/fn_total_refunded.sql`
- **Paramètres** :
  - `p_start_date` (date)
  - `p_end_date` (date)
- **Retourne** : un nombre (numeric)

```sql
SELECT agent_reporting.fn_total_refunded('2026-07-01', '2026-07-31');
```

### `fn_products_high_return_rate`

Produits dont le taux de retour dépasse un seuil (10 % par défaut) : sert aux alertes.

- **Couche** : recommandation — **Fichier** : `refunds/recommendation/fn_products_high_return_rate.sql`
- **Paramètres** :
  - `p_threshold_pct` (numeric, défaut 10)
- **Colonnes retournées** : `product_id`, `product_name`, `quantity_sold`, `quantity_refunded`, `refund_rate_pct`

```sql
SELECT * FROM agent_reporting.fn_products_high_return_rate();
```

## Ventes

### `fn_sales_by_period`

Ventes agrégées par période (`day`, `week`, `month`, `quarter` ou `year`) avec chiffre d'affaires brut, remboursé et net, et panier moyen. Filtres facultatifs : client, produit, statut, catégorie.

- **Couche** : traitement — **Fichier** : `sales/traitement/fn_sales_by_period.sql`
- **Paramètres** :
  - `p_start_date` (date)
  - `p_end_date` (date)
  - `p_period` (varchar)
  - `p_client_id` (bigint, défaut NULL)
  - `p_product_id` (bigint, défaut NULL)
  - `p_status` (varchar, défaut NULL)
  - `p_category_id` (bigint, défaut NULL)
- **Colonnes retournées** : `period`, `total_sales`, `gross_revenue`, `refunded_amount`, `net_revenue`, `average_order_value`

```sql
SELECT * FROM agent_reporting.fn_sales_by_period('2026-07-01', '2026-07-31', 'month');
```

### `fn_sales_statistics`

Statistiques globales de ventes sur une période : nombre de ventes, chiffre d'affaires brut / net, panier moyen, nombre de clients, taux de remboursement. Mêmes filtres facultatifs.

- **Couche** : traitement — **Fichier** : `sales/traitement/fn_sales_statistics.sql`
- **Paramètres** :
  - `p_start_date` (date)
  - `p_end_date` (date)
  - `p_client_id` (bigint, défaut NULL)
  - `p_product_id` (bigint, défaut NULL)
  - `p_status` (varchar, défaut NULL)
  - `p_category_id` (bigint, défaut NULL)
- **Colonnes retournées** : `total_sales`, `gross_revenue`, `refunded_amount`, `net_revenue`, `average_order_value`, `total_customers`, `refund_rate`

```sql
SELECT * FROM agent_reporting.fn_sales_statistics('2026-07-01', '2026-07-31');
```

### `fn_sales_alert_decline`

Produits dont les ventes baissent entre deux périodes au-delà d'un seuil (20 % par défaut), avec un niveau d'alerte.

- **Couche** : recommandation — **Fichier** : `sales/recommandation/fn_sales_alert_decline.sql`
- **Paramètres** :
  - `p_current_start` (date)
  - `p_current_end` (date)
  - `p_previous_start` (date)
  - `p_previous_end` (date)
  - `p_threshold_pct` (numeric, défaut 20)
- **Colonnes retournées** : `product_id`, `product_reference`, `product_name`, `previous_qty`, `current_qty`, `variation_pct`, `alert_level`

```sql
SELECT * FROM agent_reporting.fn_sales_alert_decline('2026-07-08', '2026-07-15', '2026-07-01', '2026-07-07');
```

