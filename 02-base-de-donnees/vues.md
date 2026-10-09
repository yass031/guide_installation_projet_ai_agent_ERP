# Vues du schéma `agent_reporting`

Le schéma `agent_reporting` contient **27 vues**. Ce sont les seuls objets de lecture exposés à l'agent IA : il n'a **aucun accès** aux tables brutes du schéma `public`.

- **Vues de base** (12) : reflet des tables de l'ERP, avec un nom de client ou de fournisseur ajouté quand c'est utile.
- **Vues de consultation** (15) : jointures déjà faites et montants calculés (brut, remboursé, net).

Exemple d'utilisation (connecté avec `agent_user` ou `backend_user`) :

```sql
SELECT * FROM agent_reporting.v_sales_detail ORDER BY sale_date DESC LIMIT 10;
```

Les relations entre les tables sources sont décrites dans [schema-erd.md](schema-erd.md), les colonnes des tables dans [tables.md](tables.md).

## Vues de base (`views.sql`)

### `v_categories`

Reflet de la table `categories`.

- **Fichier** : `views.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `id` | bigint |
| `name` | varchar(255) |
| `description` | text |
| `created_at` | timestamp |
| `updated_at` | timestamp |

### `v_clients`

Reflet de la table `clients`.

- **Fichier** : `views.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `id` | bigint |
| `name` | varchar(255) |
| `email` | varchar(255) |
| `phone` | varchar(255) |
| `address` | text |
| `created_at` | timestamp |
| `updated_at` | timestamp |

### `v_devis`

Devis avec le nom du client.

- **Fichier** : `views.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `id` | bigint |
| `reference` | varchar(255) |
| `client_id` | bigint |
| `client_name` | varchar(255) |
| `subtotal` | numeric(12,2) |
| `discount` | numeric(12,2) |
| `tax` | numeric(12,2) |
| `total` | numeric(12,2) |
| `status` | varchar(30) |
| `sent_at` | timestamp |
| `accepted_at` | timestamp |
| `rejected_at` | timestamp |
| `expires_at` | timestamp |
| `notes` | text |
| `created_by` | bigint |
| `created_at` | timestamp |
| `updated_at` | timestamp |

### `v_devis_items`

Reflet de la table `devis_items`.

- **Fichier** : `views.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `id` | bigint |
| `devis_id` | bigint |
| `product_id` | bigint |
| `quantity` | integer |
| `price` | numeric(12,2) |
| `total` | numeric(12,2) |
| `created_at` | timestamp |
| `updated_at` | timestamp |

### `v_payment_allocations`

Reflet de la table `payment_allocations` (affectation des paiements aux ventes).

- **Fichier** : `views.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `id` | bigint |
| `payment_id` | bigint |
| `payable_type` | varchar(255) |
| `payable_id` | bigint |
| `amount_applied` | numeric(12,2) |
| `created_at` | timestamp |
| `updated_at` | timestamp |

### `v_payments`

Paiements avec le nom du client.

- **Fichier** : `views.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `id` | bigint |
| `reference` | varchar(255) |
| `sale_id` | bigint |
| `client_id` | bigint |
| `client_name` | varchar(255) |
| `amount` | numeric(12,2) |
| `method` | varchar(30) |
| `status` | varchar(30) |
| `external_reference` | varchar(255) |
| `approved_by` | bigint |
| `approved_at` | timestamp |
| `notes` | text |
| `stripe_intent_id` | varchar(255) |
| `created_at` | timestamp |
| `updated_at` | timestamp |

### `v_purchase_items`

Reflet de la table `purchase_items`.

- **Fichier** : `views.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `id` | bigint |
| `purchase_id` | bigint |
| `product_id` | bigint |
| `price` | numeric(10,2) |
| `quantity` | integer |
| `total` | numeric(10,2) |
| `created_at` | timestamp |
| `updated_at` | timestamp |

### `v_purchases`

Achats avec le nom du fournisseur.

- **Fichier** : `views.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `id` | bigint |
| `supplier_id` | bigint |
| `supplier_name` | varchar(255) |
| `total` | numeric(10,2) |
| `status` | varchar(30) |
| `created_at` | timestamp |
| `updated_at` | timestamp |

### `v_refund_items`

Reflet de la table `refund_items`.

- **Fichier** : `views.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `id` | bigint |
| `refund_id` | bigint |
| `product_id` | bigint |
| `price` | numeric(10,2) |
| `quantity` | integer |
| `total` | numeric(10,2) |
| `created_at` | timestamp |
| `updated_at` | timestamp |

### `v_refunds`

Reflet de la table `refunds`.

- **Fichier** : `views.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `id` | bigint |
| `sale_id` | bigint |
| `total` | numeric(10,2) |
| `reason` | text |
| `created_at` | timestamp |
| `updated_at` | timestamp |

### `v_stock_movements`

Reflet de la table `stock_movements`.

- **Fichier** : `views.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `id` | bigint |
| `product_id` | bigint |
| `quantity` | integer |
| `type` | varchar(30) |
| `source` | varchar(30) |
| `reference_id` | bigint |
| `created_at` | timestamp |
| `updated_at` | timestamp |

### `v_suppliers`

Reflet de la table `suppliers`.

- **Fichier** : `views.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `id` | bigint |
| `name` | varchar(255) |
| `phone` | varchar(255) |
| `address` | text |
| `created_at` | timestamp |
| `updated_at` | timestamp |

## Vues de consultation

### `v_client_purchase_history`

Historique des achats par client : une ligne par vente, avec montants brut, remboursé et net.

- **Fichier** : `clients/consultation/v_client_purchase_history.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `client_id` | bigint |
| `client_name` | varchar(255) |
| `sale_id` | bigint |
| `sale_reference` | varchar(255) |
| `sale_date` | timestamp |
| `sale_status` | varchar(30) |
| `subtotal` | numeric(10,2) |
| `tax_amount` | numeric(10,2) |
| `discount_amount` | numeric(10,2) |
| `gross_total` | numeric |
| `net_total` | numeric |
| `refunded_total` | numeric |

### `v_clients_list`

Liste des clients (colonnes préfixées `client_`).

- **Fichier** : `clients/consultation/v_clients_list.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `client_id` | bigint |
| `client_name` | varchar(255) |
| `client_email` | varchar(255) |
| `client_phone` | varchar(255) |
| `client_address` | text |
| `client_created_at` | timestamp |

### `v_devis_items_detail`

Lignes de devis avec le nom du produit et du client.

- **Fichier** : `devis/consultation/v_devis_items_detail.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `id` | bigint |
| `devis_id` | bigint |
| `product_id` | bigint |
| `product_name` | varchar(255) |
| `client_id` | bigint |
| `client_name` | varchar(255) |
| `quantity` | integer |
| `price` | numeric(12,2) |
| `total` | numeric(12,2) |
| `created_at` | timestamp |
| `updated_at` | timestamp |

### `v_low_stock_products`

Produits dont le stock est bas, avec un niveau de sévérité (`stock_severity`).

- **Fichier** : `products_stock/traitement/v_low_stock_products.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `product_id` | bigint |
| `product_reference` | varchar(255) |
| `product_name` | varchar(255) |
| `product_description` | text |
| `product_price` | numeric(10,2) |
| `current_stock` | integer |
| `min_stock_threshold` | integer |
| `security_stock_threshold` | integer |
| `alert_stock_threshold` | integer |
| `product_created_at` | timestamp |
| `product_updated_at` | timestamp |
| `stock_severity` | text |

### `v_payments_detail`

Une ligne par affectation de paiement : détail du paiement, de la vente, du client, et solde restant de la vente.

- **Fichier** : `payments/consultation/v_payments_detail.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `allocation_id` | bigint |
| `payment_id` | bigint |
| `payment_reference` | varchar(255) |
| `method` | varchar(30) |
| `payment_status` | varchar(30) |
| `external_reference` | varchar(255) |
| `stripe_intent_id` | varchar(255) |
| `payment_notes` | text |
| `sale_id` | bigint |
| `sale_reference` | varchar(255) |
| `sale_status` | varchar(30) |
| `sale_date` | timestamp |
| `client_id` | bigint |
| `client_name` | varchar(255) |
| `client_email` | varchar(255) |
| `client_phone` | varchar(255) |
| `client_address` | text |
| `subtotal` | numeric(10,2) |
| `tax_rate` | numeric(10,2) |
| `tax_amount` | numeric(10,2) |
| `discount_amount` | numeric(10,2) |
| `gross_total` | numeric |
| `refunded_total` | numeric |
| `net_total` | numeric |
| `amount_applied` | numeric(12,2) |
| `allocation_date` | timestamp |
| `total_applied_on_sale` | numeric |
| `remaining_balance` | numeric |
| `payment_created_at` | timestamp |
| `payment_updated_at` | timestamp |

### `v_payments_pending_approval`

Paiements en attente d'approbation, avec la vente et le client concernés et le solde restant.

- **Fichier** : `payments/consultation/v_payments_pending_approval.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `payment_id` | bigint |
| `payment_reference` | varchar(255) |
| `method` | varchar(30) |
| `payment_status` | varchar(30) |
| `external_reference` | varchar(255) |
| `stripe_intent_id` | varchar(255) |
| `payment_notes` | text |
| `sale_id` | bigint |
| `sale_reference` | varchar(255) |
| `sale_status` | varchar(30) |
| `sale_date` | timestamp |
| `client_id` | bigint |
| `client_name` | varchar(255) |
| `client_email` | varchar(255) |
| `client_phone` | varchar(255) |
| `client_address` | text |
| `gross_total` | numeric |
| `refunded_total` | numeric |
| `net_total` | numeric |
| `total_applied` | numeric |
| `remaining_balance` | numeric |
| `created_at` | timestamp |
| `updated_at` | timestamp |

### `v_products_catalog`

Catalogue des produits enrichi avec la catégorie et le fournisseur.

- **Fichier** : `products_stock/consultation/v_products_catalog.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `product_id` | bigint |
| `product_reference` | varchar(255) |
| `product_name` | varchar(255) |
| `product_description` | text |
| `product_price` | numeric(10,2) |
| `current_stock` | integer |
| `min_stock_threshold` | integer |
| `security_stock_threshold` | integer |
| `alert_stock_threshold` | integer |
| `product_created_at` | timestamp |
| `product_updated_at` | timestamp |
| `category_id` | bigint |
| `category_name` | varchar(255) |
| `category_description` | text |
| `supplier_id` | bigint |
| `supplier_name` | varchar(255) |
| `supplier_phone` | varchar(255) |
| `supplier_address` | text |

### `v_purchase_items_detail`

Lignes d'achat avec produit, fournisseur et statut de l'achat.

- **Fichier** : `purchases-suppliers/consultation/v_purchase_items_detail.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `purchase_item_id` | bigint |
| `purchase_id` | bigint |
| `purchase_status` | varchar(30) |
| `purchase_date` | timestamp |
| `product_id` | bigint |
| `product_name` | varchar(255) |
| `product_reference` | varchar(255) |
| `unit_price` | numeric(10,2) |
| `quantity` | integer |
| `line_total` | numeric(10,2) |
| `supplier_id` | bigint |
| `supplier_name` | varchar(255) |

### `v_purchases_detail`

Achats avec les coordonnées du fournisseur.

- **Fichier** : `purchases-suppliers/consultation/v_purchases_detail.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `purchase_id` | bigint |
| `purchase_total` | numeric(10,2) |
| `purchase_status` | varchar(30) |
| `purchase_date` | timestamp |
| `supplier_id` | bigint |
| `supplier_name` | varchar(255) |
| `supplier_phone` | varchar(255) |
| `supplier_address` | text |

### `v_refund_items_detail`

Lignes de remboursement avec vente, client, produit et motif.

- **Fichier** : `refunds/consultation/v_refund_items_detail.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `refund_item_id` | bigint |
| `refund_id` | bigint |
| `sale_id` | bigint |
| `sale_reference` | varchar(255) |
| `client_id` | bigint |
| `client_name` | varchar(255) |
| `product_id` | bigint |
| `product_name` | varchar(255) |
| `product_reference` | varchar(255) |
| `quantity` | integer |
| `price` | numeric(10,2) |
| `total` | numeric(10,2) |
| `reason` | text |
| `refunded_at` | timestamp |

### `v_refunds_detail`

Remboursements avec la vente et le client concernés.

- **Fichier** : `refunds/consultation/v_refunds_detail.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `refund_id` | bigint |
| `sale_id` | bigint |
| `sale_reference` | varchar(255) |
| `client_id` | bigint |
| `client_name` | varchar(255) |
| `reason` | text |
| `refund_total` | numeric(10,2) |
| `refunded_at` | timestamp |

### `v_sale_items`

Lignes de vente détaillées : produit, catégorie, quantités vendues, remboursées et nettes, type et taux de remboursement.

- **Fichier** : `sales/consultation/v_sale_items.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `sale_id` | bigint |
| `sale_reference` | varchar(255) |
| `sale_date` | timestamp |
| `sale_status` | varchar(30) |
| `client_id` | bigint |
| `client_name` | varchar(255) |
| `client_email` | varchar(255) |
| `client_phone` | varchar(255) |
| `product_id` | bigint |
| `product_name` | varchar(255) |
| `product_reference` | varchar(255) |
| `category_id` | bigint |
| `category_name` | varchar(255) |
| `product_unit_price` | numeric(10,2) |
| `ordered_quantity` | integer |
| `product_subtotal_before_discount_tax` | numeric(10,2) |
| `refunded_quantity` | integer |
| `refunded_amount` | numeric(12,2) |
| `net_quantity_sold` | integer |
| `refund_type` | text |
| `refund_rate_percentage` | numeric |

### `v_sales_detail`

**Vue centrale des ventes** : une ligne par vente avec le client et les montants `gross_total` (brut), `refunded_total` et `net_total` (net). Utilisée par de nombreuses fonctions.

- **Fichier** : `sales/consultation/v_sales_detail.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `sale_id` | bigint |
| `sale_reference` | varchar(255) |
| `sale_date` | timestamp |
| `sale_updated_at` | timestamp |
| `status` | varchar(30) |
| `subtotal` | numeric(10,2) |
| `tax_rate` | numeric(10,2) |
| `tax_amount` | numeric(10,2) |
| `discount_amount` | numeric(10,2) |
| `gross_total` | numeric |
| `refunded_total` | numeric |
| `net_total` | numeric |
| `client_id` | bigint |
| `client_name` | varchar(255) |
| `client_email` | varchar(255) |
| `client_phone` | varchar(255) |
| `client_address` | text |

### `v_stock_movements_detail`

Mouvements de stock avec le nom et la référence du produit.

- **Fichier** : `products_stock/consultation/v_stock_movements_detail.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `movement_id` | bigint |
| `product_id` | bigint |
| `product_name` | varchar(255) |
| `product_reference` | varchar(255) |
| `movement_quantity` | integer |
| `movement_type` | varchar(30) |
| `movement_source` | varchar(30) |
| `source_reference_id` | bigint |
| `movement_date` | timestamp |

### `v_suppliers_list`

Liste des fournisseurs avec le nombre de produits fournis.

- **Fichier** : `purchases-suppliers/consultation/v_suppliers_list.sql`
- **Colonnes** :

| Colonne | Type |
|---|---|
| `supplier_id` | bigint |
| `supplier_name` | varchar(255) |
| `supplier_phone` | varchar(255) |
| `supplier_address` | text |
| `product_count` | bigint |

