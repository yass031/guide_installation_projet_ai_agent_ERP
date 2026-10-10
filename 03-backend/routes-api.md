# Routes de l'API

Toutes les routes sont déclarées dans `api/` et branchées dans `main.py`. Ce fichier est généré à partir du code : les colonnes *Paramètres* et *Appelle* reflètent les signatures réelles.

## 1. Conventions

- **Documentation interactive** : `/docs` (Swagger) et `/redoc`.
- **Préfixe** : `/api` pour tous les routers, sauf `auth`, `user` et `admin_users` qui portent déjà leur préfixe complet (`/api/auth`, `/api/user`, `/api/admin/users`).
- **Dates** : format `YYYY-MM-DD`. `start_date` doit être antérieure ou égale à `end_date` (sinon `400`). Sur les routes marquées `start_date?` / `end_date?`, les deux sont optionnelles.
- **Auth** : `Oui` = JWT requis (`get_current_user`, souvent au niveau du router). `Non` = route publique. `Admin` = rôle `admin` requis. Voir `authentification.md`.
- **Deux familles de routes** :
  1. *Routes frontend* : un endpoint par KPI ou un endpoint `/report` unifié qui agrège tout un domaine.
  2. *Routes « outils » de l'agent IA* : suffixes `-consultation`, `-traitement`, `-recommandation`, avec des limites de résultats plafonnées (conçues pour être lues dans un chat).

> ⚠️ Les routers **dashboard**, **sales** et **refunds** (frontend) n'ont aucune authentification. Voir `points-attention.md`.

## 2. Authentification et utilisateurs

| Méthode | Route | Paramètres (`?` = optionnel) | Auth | Appelle |
|---|---|---|---|---|
| GET | `/api/admin/users` | — | Admin | `list_users` |
| POST | `/api/admin/users` | — | Admin | `create_user` |
| PATCH | `/api/admin/users/{user_id}/role` | `user_id` | Admin | `update_role` |
| DELETE | `/api/admin/users/{user_id}` | `user_id` | Admin | `delete_user` |
| POST | `/api/auth/register` | — | Non | `register` |
| POST | `/api/auth/login` | — | Non | `authenticate` |
| GET | `/api/auth/me` | — | Oui | — |
| PUT | `/api/user/profile` | — | Oui | `update_profile` |
| PUT | `/api/user/password` | — | Oui | `change_password` |
| PUT | `/api/user/language` | — | Oui | `update_language` |
| POST | `/api/user/reset-password` | — | Non | `reset_password` |

## 3. Routes du frontend (rapports)

Les pages stock, achats, fournisseurs et devis sont servies par **un seul endpoint `/report`** qui agrège tous les KPI du domaine (dates obligatoires). Les routes `/suppliers/cheaper-alternatives` (analyse coûteuse) sont séparées pour ne pas ralentir le rapport.

### Dashboard

| Méthode | Route | Paramètres (`?` = optionnel) | Auth | Appelle |
|---|---|---|---|---|
| GET | `/api/dashboard/overview` | — | Non | `get_overview` |
| GET | `/api/dashboard/revenue-trend` | `months?` | Non | `get_revenue_trend` |
| GET | `/api/dashboard/sales-count-trend` | `months?` | Non | `get_sales_count_trend` |
| GET | `/api/dashboard/best-clients` | — | Non | `get_top_clients_month` |
| GET | `/api/dashboard/top-products` | `limit?` | Non | `get_top_products_month` |
| GET | `/api/dashboard/categories` | — | Non | `get_category_names` |

### Ventes

| Méthode | Route | Paramètres (`?` = optionnel) | Auth | Appelle |
|---|---|---|---|---|
| GET | `/api/sales/rfm-segmentation` | `start_date?`, `end_date?` | Non | `get_rfm_segmentation` |
| GET | `/api/sales/refund-rate-by-product` | `limit?`, `start_date?`, `end_date?` | Non | `get_refund_rate_by_product` |
| GET | `/api/sales/category-performance` | `start_date?`, `end_date?` | Non | `get_category_performance` |
| GET | `/api/sales/dormant-products` | `days?` | Non | `get_dormant_products` |
| GET | `/api/sales/refund-rate-trend` | `months?`, `start_date?`, `end_date?` | Non | `get_refund_rate_trend` |
| GET | `/api/sales/revenue-by-status` | `start_date?`, `end_date?` | Non | `get_revenue_by_status` |
| GET | `/api/sales/refund-rate-by-client` | `limit?`, `start_date?`, `end_date?` | Non | `get_refund_rate_by_client` |
| GET | `/api/sales/sales-by-city` | `start_date?`, `end_date?` | Non | `get_sales_by_city` |

### Stock

| Méthode | Route | Paramètres (`?` = optionnel) | Auth | Appelle |
|---|---|---|---|---|
| GET | `/api/stock/report` | `start_date`, `end_date` | Oui | `get_stock_report` |

### Achats

| Méthode | Route | Paramètres (`?` = optionnel) | Auth | Appelle |
|---|---|---|---|---|
| GET | `/api/purchases/report` | `start_date`, `end_date` | Oui | `get_purchases_report` |

### Fournisseurs

| Méthode | Route | Paramètres (`?` = optionnel) | Auth | Appelle |
|---|---|---|---|---|
| GET | `/api/suppliers/report` | `start_date`, `end_date` | Oui | `get_suppliers_report` |
| GET | `/api/suppliers/cheaper-alternatives` | `start_date`, `end_date`, `limit?` | Oui | `get_products_with_cheaper_alternative` |

### Devis

| Méthode | Route | Paramètres (`?` = optionnel) | Auth | Appelle |
|---|---|---|---|---|
| GET | `/api/devis/report` | `start_date`, `end_date` | Oui | `get_devis_report` |

### Remboursements

| Méthode | Route | Paramètres (`?` = optionnel) | Auth | Appelle |
|---|---|---|---|---|
| GET | `/api/refunds/summary` | `start_date?`, `end_date?` | Non | `get_refunds_summary` |
| GET | `/api/refunds/by-reason` | `start_date?`, `end_date?` | Non | `get_refunds_by_reason` |
| GET | `/api/refunds/trend` | `start_date?`, `end_date?`, `months?` | Non | `get_refund_trend` |
| GET | `/api/refunds/top-products` | `limit?`, `start_date?`, `end_date?` | Non | `get_top_refunded_products` |
| GET | `/api/refunds/top-clients` | `limit?`, `start_date?`, `end_date?` | Non | `get_top_clients_by_refunds` |
| GET | `/api/refunds/refund-rate` | `start_date?`, `end_date?` | Non | `get_refund_rate_global` |
| GET | `/api/refunds/refund-rate-by-product` | `limit?`, `start_date?`, `end_date?` | Non | `get_refund_rate_by_product` |
| GET | `/api/refunds/high-return-products` | `threshold_pct?`, `limit?`, `start_date?`, `end_date?` | Non | `get_high_return_products` |
| GET | `/api/refunds/growth` | `start_date?`, `end_date?` | Non | `get_refunds_growth` |

### Paiements

| Méthode | Route | Paramètres (`?` = optionnel) | Auth | Appelle |
|---|---|---|---|---|
| GET | `/api/payments/overview` | `start_date?`, `end_date?` | Oui | `get_payments_overview_kpis` |
| GET | `/api/payments/status-distribution` | `start_date?`, `end_date?` | Oui | `get_payment_status_distribution` |
| GET | `/api/payments/method-distribution` | `start_date?`, `end_date?` | Oui | `get_payment_method_distribution` |
| GET | `/api/payments/collection-trend` | `days?`, `start_date?`, `end_date?` | Oui | `get_collection_trend` |
| GET | `/api/payments/top-debtors` | `limit?`, `start_date?`, `end_date?` | Oui | `get_top_debtor_clients` |
| GET | `/api/payments/overdue` | `threshold_days?` | Oui | `get_overdue_kpi` |
| GET | `/api/payments/recurring-late-clients` | `threshold_days?`, `min_occurrences?` | Oui | `get_recurring_late_clients_kpi` |
| GET | `/api/payments/cash-collected` | `start_date?`, `end_date?` | Oui | `get_cash_collected_kpi` |
| GET | `/api/payments/summary` | `start_date?`, `end_date?` | Oui | `get_payments_overview_kpis` |

### Clients

| Méthode | Route | Paramètres (`?` = optionnel) | Auth | Appelle |
|---|---|---|---|---|
| GET | `/api/clients/count` | — | Oui | `get_clients_count` |
| GET | `/api/clients/new` | `days?` | Oui | `get_new_clients_count` |
| GET | `/api/clients/sales-status` | `start_date?`, `end_date?` | Oui | `get_sales_status_summary` |
| GET | `/api/clients/global-revenue` | `start_date?`, `end_date?` | Oui | `get_global_revenue_summary` |
| GET | `/api/clients/top` | `limit?`, `start_date?`, `end_date?` | Oui | `get_top_clients_overall` |
| GET | `/api/clients/repeat` | `min_orders?`, `start_date?`, `end_date?` | Oui | `get_repeat_clients_summary` |
| GET | `/api/clients/inactive` | `days_inactive?` | Oui | `get_inactive_clients_summary` |
| GET | `/api/clients/to-recontact` | `days_inactive?`, `min_historical_amount?` | Oui | `get_clients_to_recontact_summary` |
| GET | `/api/clients/rfm` | — | Oui | `get_rfm_segmentation` |
| GET | `/api/clients/rfm-summary` | — | Oui | `get_rfm_summary` |
| GET | `/api/clients/evolution` | `months?`, `start_date?`, `end_date?` | Oui | `get_clients_evolution` |
| GET | `/api/clients/without-orders` | — | Oui | `get_clients_without_orders` |
| GET | `/api/clients/repeat-rate` | — | Oui | `get_repeat_rate` |
| GET | `/api/clients/avg-time-between-orders` | — | Oui | `get_avg_time_between_orders` |
| GET | `/api/clients/churn-rate` | `days_inactive?` | Oui | `get_churn_rate` |
| GET | `/api/clients/lifetime-value-distribution` | — | Oui | `get_lifetime_value_distribution` |
| GET | `/api/clients/health` | — | Oui | `get_customer_health_score` |
| GET | `/api/clients/by-city` | — | Oui | `get_clients_by_city` |
| GET | `/api/clients/inactive/list` | `days_inactive?` | Oui | `get_inactive_clients_list` |
| GET | `/api/clients/to-recontact/list` | `days_inactive?`, `min_historical_amount?` | Oui | `get_clients_to_recontact_list` |
| GET | `/api/clients/repeat/list` | `min_orders?` | Oui | `get_repeat_clients_list` |
| GET | `/api/clients/high-refund-rate` | `limit?` | Oui | `get_clients_with_high_refund_rate` |
| GET | `/api/clients/kpi-summary` | `start_date?`, `end_date?` | Oui | `get_clients_kpi_summary` |
| GET | `/api/clients/{client_id}` | `client_id` | Oui | `get_client_details` |

## 4. Routes « outils » de l'agent IA

Toutes protégées par JWT. L'agent appelle ces routes avec le token de l'utilisateur (voir `agent-ia.md`).

### Ventes

| Méthode | Route | Paramètres (`?` = optionnel) | Auth | Appelle |
|---|---|---|---|---|
| GET | `/api/sales-consultation/by-period` | `start_date?`, `end_date?`, `updated_start?`, `updated_end?`, `status?`, `order_by?`, `order_direction?`, `limit?` | Oui | `get_sales_by_period` |
| GET | `/api/sales-consultation/lookup` | `sale_id?`, `reference?` | Oui | `get_sale_lookup` |
| GET | `/api/sales-consultation/by-client` | `client_id?`, `client_name?`, `client_email?`, `client_phone?`, `client_city?`, `start_date?`, `end_date?`, `limit?` | Oui | `get_sales_by_client` |
| GET | `/api/sales-consultation/by-amounts` | `min_subtotal?`, `max_subtotal?`, `min_tax_rate?`, `max_tax_rate?`, `min_tax_amount?`, `max_tax_amount?`, `min_discount_amount?`, `max_discount_amount?`, `min_gross_total?`, `max_gross_total?`, `min_net_total?`, `max_net_total?`, `min_refunded_total?`, `max_refunded_total?`, `has_refund?`, `start_date?`, `end_date?`, `order_by?`, `order_direction?`, `limit?` | Oui | `get_sales_by_amounts` |
| GET | `/api/sales-consultation/items/by-sale` | `sale_id?`, `reference?`, `sale_status?` | Oui | `get_sale_items_by_sale` |
| GET | `/api/sales-consultation/items/history-by-product` | `product_id?`, `product_name?`, `product_reference?`, `category_id?`, `category_name?`, `start_date?`, `end_date?`, `limit?` | Oui | `get_sales_history_by_product` |
| GET | `/api/sales-consultation/items/by-client` | `client_id?`, `client_name?`, `limit?` | Oui | `get_sale_items_by_client` |
| GET | `/api/sales-consultation/items/by-metrics` | `min_ordered_quantity?`, `max_ordered_quantity?`, `min_subtotal?`, `max_subtotal?`, `refund_type?`, `min_refund_rate?`, `max_refund_rate?`, `order_by?`, `order_direction?`, `limit?` | Oui | `get_sale_items_by_metrics` |
| GET | `/api/sales-recommandation/decline-alert` | `current_start`, `current_end`, `previous_start`, `previous_end`, `threshold_pct?`, `limit?` | Oui | `get_sales_decline_alert` |
| GET | `/api/sales-traitement/statistics-summary` | `start_date`, `end_date`, `client_id?`, `product_id?`, `status?`, `category_id?` | Oui | `get_sales_statistics_summary` |
| GET | `/api/sales-traitement/statistics-by-period` | `start_date`, `end_date`, `period?`, `client_id?`, `product_id?`, `status?`, `category_id?`, `limit?` | Oui | `get_sales_statistics_by_period` |

### Stock

| Méthode | Route | Paramètres (`?` = optionnel) | Auth | Appelle |
|---|---|---|---|---|
| GET | `/api/stock-consultation/catalog` | `category_id?`, `supplier_id?`, `min_price?`, `max_price?`, `name_search?`, `sort_by?` | Oui | `get_products_catalog` |
| GET | `/api/stock-consultation/categories` | `name_search?` | Oui | `get_categories_list` |
| GET | `/api/stock-consultation/products` | `name_search?` | Oui | `get_products_list` |
| GET | `/api/stock-consultation/suppliers` | `name_search?` | Oui | `get_suppliers_list` |
| GET | `/api/stock-consultation/low-stock` | — | Oui | `get_low_stock_products` |
| GET | `/api/stock-consultation/top-priced` | `limit?`, `order?`, `category_id?`, `supplier_id?` | Oui | `get_top_priced_products` |
| GET | `/api/stock-consultation/movements` | `product_id?`, `limit?` | Oui | `get_stock_movements_detail` |
| GET | `/api/stock-recommandation/reorder-suggestions` | `days_lookback?`, `target_coverage_days?` | Oui | `get_reorder_suggestions` |
| GET | `/api/stock-recommandation/to-discontinue` | `dormant_days_threshold?`, `days_lookback?` | Oui | `get_products_to_discontinue` |
| GET | `/api/stock-traitement/value` | `category_id?`, `supplier_id?` | Oui | `get_product_stock_value` |
| GET | `/api/stock-traitement/velocity` | `product_id`, `days_lookback?` | Oui | `get_product_sales_velocity` |
| GET | `/api/stock-traitement/fast-moving-products` | `days_lookback?`, `target_coverage_days?`, `limit?` | Oui | `get_fast_moving_products` |
| GET | `/api/stock-traitement/count-by-category` | — | Oui | `get_products_count_by_category` |
| GET | `/api/stock-traitement/count-by-supplier` | — | Oui | `get_products_count_by_supplier` |
| GET | `/api/stock-traitement/movements/by-period` | `start_date`, `end_date`, `product_id?`, `movement_type?` | Oui | `get_stock_movements_by_period` |
| GET | `/api/stock-traitement/no-movement` | `days?` | Oui | `get_products_no_movement` |

### Remboursements

| Méthode | Route | Paramètres (`?` = optionnel) | Auth | Appelle |
|---|---|---|---|---|
| GET | `/api/refunds-consultation/by-period` | `start_date?`, `end_date?`, `order_by?`, `order_direction?`, `limit?` | Oui | `get_refunds_by_period` |
| GET | `/api/refunds-consultation/lookup` | `refund_id?`, `sale_reference?` | Oui | `get_refund_lookup` |
| GET | `/api/refunds-consultation/by-client` | `client_id?`, `client_name?`, `start_date?`, `end_date?`, `limit?` | Oui | `get_refunds_by_client` |
| GET | `/api/refunds-consultation/by-reason` | `reason?`, `limit?` | Oui | `get_refunds_by_reason_text` |
| GET | `/api/refunds-consultation/items/by-refund` | `refund_id?`, `sale_reference?` | Oui | `get_refund_items_by_refund` |
| GET | `/api/refunds-consultation/items/by-product` | `product_id?`, `product_name?`, `product_reference?`, `start_date?`, `end_date?`, `limit?` | Oui | `get_refund_items_by_product` |
| GET | `/api/refunds-consultation/items/by-client` | `client_id?`, `client_name?`, `limit?` | Oui | `get_refund_items_by_client` |
| GET | `/api/refunds-recommandation/high-return-rate` | `threshold_pct?`, `limit?` | Oui | `get_products_high_return_rate` |
| GET | `/api/refunds-traitement/period-detail` | `start_date`, `end_date`, `limit?` | Oui | `get_refunds_period_detail` |
| GET | `/api/refunds-traitement/refund-rate-by-product` | `start_date?`, `end_date?`, `limit?` | Oui | `get_refund_rate_by_product` |
| GET | `/api/refunds-traitement/by-reason` | `start_date`, `end_date` | Oui | `get_refunds_by_reason` |
| GET | `/api/refunds-traitement/total-refunded` | `start_date`, `end_date` | Oui | `get_total_refunded` |

### Clients

| Méthode | Route | Paramètres (`?` = optionnel) | Auth | Appelle |
|---|---|---|---|---|
| GET | `/api/clients-consultation/list` | `client_id?`, `client_name?`, `client_email?`, `client_phone?`, `limit?` | Oui | `get_clients_list` |
| GET | `/api/clients-consultation/purchase-history` | `client_id?`, `client_name?`, `sale_status?`, `start_date?`, `end_date?`, `limit?` | Oui | `get_client_purchase_history` |
| GET | `/api/clients-recommandation/to-recontact` | `days_inactive?`, `min_historical_amount?`, `limit?` | Oui | `get_clients_to_recontact` |
| GET | `/api/clients-traitement/lifetime-value` | `client_id` | Oui | `get_client_lifetime_value` |
| GET | `/api/clients-traitement/top-clients` | `start_date`, `end_date`, `limit?` | Oui | `get_top_clients` |
| GET | `/api/clients-traitement/inactive` | `days_inactive`, `limit?` | Oui | `get_inactive_clients` |

### Paiements

| Méthode | Route | Paramètres (`?` = optionnel) | Auth | Appelle |
|---|---|---|---|---|
| GET | `/api/payments-consultation/detail` | `sale_id?`, `client_id?`, `payment_id?`, `client_name?`, `sale_reference?`, `limit?` | Oui | `get_payments_detail` |
| GET | `/api/payments-consultation/search-client` | `name` | Oui | `search_client_by_name` |
| GET | `/api/payments-consultation/search-sale` | `reference` | Oui | `search_sale_by_reference` |
| GET | `/api/payments-consultation/client/sales-summary` | `client_id?`, `client_name?` | Oui | `get_client_sales_summary` |
| GET | `/api/payments-consultation/remaining-balance` | `limit?` | Oui | `get_sales_with_remaining_balance` |
| GET | `/api/payments-consultation/sale/allocations` | `sale_id?`, `sale_reference?` | Oui | `get_sale_allocations` |
| GET | `/api/payments-recommandation/recurring-late-clients` | `threshold_days?`, `min_occurrences?` | Oui | `get_recurring_late_clients` |
| GET | `/api/payments-recommandation/cashflow-forecast` | `days_ahead?` | Oui | `get_cashflow_forecast` |
| GET | `/api/payments-recommandation/client/recurring-late-status` | `client_id?`, `client_name?`, `threshold_days?`, `min_occurrences?` | Oui | `check_client_recurring_late_status` |
| GET | `/api/payments-traitement/by-method` | `start_date?`, `end_date?` | Oui | `get_payments_by_method` |
| GET | `/api/payments-traitement/by-status` | `start_date?`, `end_date?` | Oui | `get_payments_by_status` |
| GET | `/api/payments-traitement/client/balance` | `client_id?`, `client_name?` | Oui | `get_client_balance` |
| GET | `/api/payments-traitement/cash-collected` | `start_date?`, `end_date?` | Oui | `get_cash_collected` |
| GET | `/api/payments-traitement/sale/balance` | `sale_id?`, `sale_reference?` | Oui | `get_sale_balance` |
| GET | `/api/payments-traitement/payment-alerts` | `threshold_days?` | Oui | `get_sale_payment_alerts` |

### Achats & Fournisseurs

| Méthode | Route | Paramètres (`?` = optionnel) | Auth | Appelle |
|---|---|---|---|---|
| GET | `/api/purchases-consultation/detail` | — | Oui | `get_purchases_detail` |
| GET | `/api/purchases-consultation/items-detail` | — | Oui | `get_purchase_items_detail` |
| GET | `/api/purchases-consultation/suppliers` | — | Oui | `get_suppliers_list` |
| GET | `/api/purchases-recommandation/best-supplier` | `product_id` | Oui | `get_best_supplier_for_product` |
| GET | `/api/purchases-traitement/by-period` | `start_date`, `end_date`, `supplier_id?` | Oui | `get_purchases_by_period` |
| GET | `/api/purchases-traitement/supplier-price-history` | `product_id`, `supplier_id?` | Oui | `get_supplier_price_history` |
| GET | `/api/purchases-traitement/top-suppliers` | `start_date`, `end_date`, `limit?` | Oui | `get_top_suppliers` |
| GET | `/api/purchases-traitement/compare-supplier-prices` | `product_id` | Oui | `get_compare_supplier_prices` |
| GET | `/api/purchases-traitement/supplier-summary` | `supplier_id` | Oui | `get_supplier_summary` |
| GET | `/api/purchases-traitement/suppliers-no-recent-order` | `days?` | Oui | `get_suppliers_no_recent_order` |

### Devis

| Méthode | Route | Paramètres (`?` = optionnel) | Auth | Appelle |
|---|---|---|---|---|
| GET | `/api/devis-consultation/detail` | — | Oui | `get_devis_detail` |
| GET | `/api/devis-consultation/items-detail` | — | Oui | `get_devis_items_detail` |
| GET | `/api/devis-recommandation/pending-followup` | `days_since_sent?` | Oui | `get_devis_pending_followup` |
| GET | `/api/devis-recommandation/about-to-expire` | `days_before?` | Oui | `get_devis_about_to_expire` |
| GET | `/api/devis-traitement/conversion-rate` | `start_date`, `end_date` | Oui | `get_devis_conversion_rate` |
| GET | `/api/devis-traitement/avg-response-time` | `start_date?`, `end_date?` | Oui | `get_devis_avg_response_time` |
| GET | `/api/devis-traitement/status-summary` | `start_date`, `end_date` | Oui | `get_devis_by_status_summary` |

## 5. Chat et exports


| Méthode | Route | Paramètres (`?` = optionnel) | Auth | Appelle |
|---|---|---|---|---|
| POST | `/api/chat` | — | Oui | — |
| POST | `/api/reports/export` | — | Oui | — |
| GET | `/api/reports/download/{filename}` | `filename` | Oui | — |
| GET | `/api/reports/my-exports` | — | Oui | — |

Corps de `POST /api/chat` : `{message, session_id}` → réponse `{reply, generated_sql, download_url}`.
Corps de `POST /api/reports/export` : `{report_type, file_format: xlsx|csv|pdf, filters}`. Détails dans `exports-rapports.md`.

## 6. Codes d'erreur courants

| Code | Cas |
|---|---|
| 400 | Dates incohérentes, filtre ambigu à l'export |
| 401 | Token absent, invalide, expiré ; utilisateur introuvable ou désactivé ; mauvais identifiants |
| 403 | Rôle non autorisé (admin, type de rapport) |
| 404 | Export inconnu, aucune donnée à exporter |
| 410 | Export expiré (48 h) |
| 502 | Agent IA injoignable ou erreur côté agent |
