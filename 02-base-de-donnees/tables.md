# Tables de la base de données

La base `nextact_ecom` contient deux schémas :

- **`public`** : les 17 tables de l'ERP + la table `users` de l'application (18 tables) ;
- **`agent_reporting`** : 2 tables propres au reporting, en plus des 27 vues et 44 fonctions (voir [vues.md](vues.md) et [fonctions.md](fonctions.md)).

Légende : « Obligatoire = oui » signifie que la colonne ne peut pas être vide (`NOT NULL`). Les identifiants (`id`) des tables ERP sont générés automatiquement.

> Pour le schéma des relations entre tables, voir [schema-erd.md](schema-erd.md).

## 1. Tables de l'ERP (schéma `public`)

Ces tables sont **en lecture seule** pour le backend (`backend_user`) et **inaccessibles** pour l'agent IA (`agent_user`), qui passe uniquement par les vues.

### `public.categories`

Catégories de produits (Phones, Laptops, Audio…).

| Colonne | Type | Obligatoire | Valeur par défaut | Remarques |
|---|---|:-:|---|---|
| `id` | bigint | oui | auto (identité) | **clé primaire** |
| `name` | varchar(255) | oui |  |  |
| `description` | text | non |  |  |
| `created_at` | timestamp | non |  |  |
| `updated_at` | timestamp | non |  |  |

### `public.clients`

Clients de l'entreprise : nom, e-mail, téléphone, adresse.

| Colonne | Type | Obligatoire | Valeur par défaut | Remarques |
|---|---|:-:|---|---|
| `id` | bigint | oui | auto (identité) | **clé primaire** |
| `name` | varchar(255) | oui |  |  |
| `email` | varchar(255) | oui |  |  |
| `phone` | varchar(255) | non |  |  |
| `address` | text | non |  |  |
| `created_at` | timestamp | non |  |  |
| `updated_at` | timestamp | non |  |  |

### `public.suppliers`

Fournisseurs auprès desquels l'entreprise achète ses produits.

| Colonne | Type | Obligatoire | Valeur par défaut | Remarques |
|---|---|:-:|---|---|
| `id` | bigint | oui | auto (identité) | **clé primaire** |
| `name` | varchar(255) | oui |  |  |
| `phone` | varchar(255) | non |  |  |
| `address` | text | non |  |  |
| `created_at` | timestamp | non |  |  |
| `updated_at` | timestamp | non |  |  |

### `public.products`

Catalogue des produits avec prix, stock courant et seuils de stock (`min_stock`, `security_stock`, `alert_stock`). Chaque produit appartient à une catégorie et à un fournisseur.

| Colonne | Type | Obligatoire | Valeur par défaut | Remarques |
|---|---|:-:|---|---|
| `id` | bigint | oui | auto (identité) | **clé primaire** |
| `reference` | varchar(255) | oui |  | unique |
| `name` | varchar(255) | oui |  |  |
| `description` | text | non |  |  |
| `image_path` | varchar(255) | non |  |  |
| `price` | numeric(10,2) | oui |  |  |
| `stock` | integer | oui | 0 |  |
| `min_stock` | integer | oui | 0 |  |
| `security_stock` | integer | oui | 0 |  |
| `alert_stock` | integer | oui | 0 |  |
| `category_id` | bigint | oui |  | → `categories.id` (on delete cascade) |
| `supplier_id` | bigint | oui |  | → `suppliers.id` (on delete cascade) |
| `created_at` | timestamp | non |  |  |
| `updated_at` | timestamp | non |  |  |

### `public.devis`

Devis commerciaux envoyés aux clients. Cycle de vie : `draft` → `sent` → `accepted` / `rejected` / `expired`. `created_by` contient l'identifiant de l'utilisateur créateur (sans clé étrangère).

| Colonne | Type | Obligatoire | Valeur par défaut | Remarques |
|---|---|:-:|---|---|
| `id` | bigint | oui | auto (identité) | **clé primaire** |
| `reference` | varchar(255) | oui |  | unique |
| `client_id` | bigint | non |  | → `clients.id` (on delete set null) |
| `subtotal` | numeric(12,2) | oui | 0.00 |  |
| `discount` | numeric(12,2) | oui | 0.00 |  |
| `tax` | numeric(12,2) | oui | 0.00 |  |
| `total` | numeric(12,2) | oui | 0.00 |  |
| `status` | varchar(30) | oui | draft | valeurs : `draft`, `sent`, `accepted`, `rejected`, `expired` |
| `sent_at` | timestamp | non |  |  |
| `accepted_at` | timestamp | non |  |  |
| `rejected_at` | timestamp | non |  |  |
| `expires_at` | timestamp | non |  |  |
| `notes` | text | non |  |  |
| `created_by` | bigint | oui |  |  |
| `created_at` | timestamp | non |  |  |
| `updated_at` | timestamp | non |  |  |

### `public.devis_items`

Lignes de produits d'un devis (quantité, prix, total).

| Colonne | Type | Obligatoire | Valeur par défaut | Remarques |
|---|---|:-:|---|---|
| `id` | bigint | oui | auto (identité) | **clé primaire** |
| `devis_id` | bigint | oui |  | → `devis.id` (on delete cascade) |
| `product_id` | bigint | oui |  | → `products.id` (on delete cascade) |
| `quantity` | integer | oui |  |  |
| `price` | numeric(12,2) | oui |  |  |
| `total` | numeric(12,2) | oui |  |  |
| `created_at` | timestamp | non |  |  |
| `updated_at` | timestamp | non |  |  |

### `public.sales`

Ventes. Les montants stockés sont `subtotal`, `tax_rate`, `tax_amount`, `discount_amount`, `total`. Statuts : `unpaid`, `partially_paid`, `paid`, `partial_refunded`, `refunded`.

| Colonne | Type | Obligatoire | Valeur par défaut | Remarques |
|---|---|:-:|---|---|
| `id` | bigint | oui | auto (identité) | **clé primaire** |
| `reference` | varchar(255) | oui |  | unique |
| `client_id` | bigint | oui |  | → `clients.id` (on delete cascade) |
| `subtotal` | numeric(10,2) | oui | 0.00 |  |
| `tax_rate` | numeric(10,2) | oui | 0.00 |  |
| `tax_amount` | numeric(10,2) | oui | 0.00 |  |
| `discount_amount` | numeric(10,2) | oui | 0.00 |  |
| `total` | numeric(10,2) | oui | 0.00 |  |
| `status` | varchar(30) | oui | unpaid | valeurs : `paid`, `partially_paid`, `unpaid`, `refunded`, `partial_refunded` |
| `created_at` | timestamp | non |  |  |
| `updated_at` | timestamp | non |  |  |

### `public.sale_items`

Lignes d'une vente. Contient aussi le suivi des remboursements ligne par ligne (`refund_quantity`, `refund_total`, `refund_status`).

| Colonne | Type | Obligatoire | Valeur par défaut | Remarques |
|---|---|:-:|---|---|
| `id` | bigint | oui | auto (identité) | **clé primaire** |
| `sale_id` | bigint | oui |  | → `sales.id` (on delete cascade) |
| `product_id` | bigint | oui |  | → `products.id` (on delete cascade) |
| `price` | numeric(10,2) | oui |  |  |
| `quantity` | integer | oui |  |  |
| `refund_quantity` | integer | oui | 0 |  |
| `total` | numeric(10,2) | oui |  |  |
| `refund_total` | numeric(12,2) | oui | 0.00 |  |
| `refund_status` | varchar(30) | oui | none | valeurs : `none`, `partial`, `refunded` |

### `public.purchases`

Achats (commandes) passés auprès des fournisseurs. Statuts : `pending`, `confirmed`.

| Colonne | Type | Obligatoire | Valeur par défaut | Remarques |
|---|---|:-:|---|---|
| `id` | bigint | oui | auto (identité) | **clé primaire** |
| `supplier_id` | bigint | oui |  | → `suppliers.id` (on delete cascade) |
| `total` | numeric(10,2) | oui | 0.00 |  |
| `status` | varchar(30) | oui | pending | valeurs : `pending`, `confirmed` |
| `created_at` | timestamp | non |  |  |
| `updated_at` | timestamp | non |  |  |

### `public.purchase_items`

Lignes de produits d'un achat (prix d'achat, quantité, total).

| Colonne | Type | Obligatoire | Valeur par défaut | Remarques |
|---|---|:-:|---|---|
| `id` | bigint | oui | auto (identité) | **clé primaire** |
| `purchase_id` | bigint | oui |  | → `purchases.id` (on delete cascade) |
| `product_id` | bigint | oui |  | → `products.id` (on delete cascade) |
| `price` | numeric(10,2) | oui |  |  |
| `quantity` | integer | oui |  |  |
| `total` | numeric(10,2) | oui |  |  |
| `created_at` | timestamp | non |  |  |
| `updated_at` | timestamp | non |  |  |

### `public.refunds`

Remboursements effectués sur une vente (en-tête : total et motif).

| Colonne | Type | Obligatoire | Valeur par défaut | Remarques |
|---|---|:-:|---|---|
| `id` | bigint | oui | auto (identité) | **clé primaire** |
| `sale_id` | bigint | oui |  | → `sales.id` (on delete cascade) |
| `total` | numeric(10,2) | oui | 0.00 |  |
| `reason` | text | non |  |  |
| `created_at` | timestamp | non |  |  |
| `updated_at` | timestamp | non |  |  |

### `public.refund_items`

Lignes de produits remboursés pour chaque remboursement.

| Colonne | Type | Obligatoire | Valeur par défaut | Remarques |
|---|---|:-:|---|---|
| `id` | bigint | oui | auto (identité) | **clé primaire** |
| `refund_id` | bigint | oui |  | → `refunds.id` (on delete cascade) |
| `product_id` | bigint | oui |  | → `products.id` (on delete cascade) |
| `price` | numeric(10,2) | oui |  |  |
| `quantity` | integer | oui |  |  |
| `total` | numeric(10,2) | oui |  |  |
| `created_at` | timestamp | non |  |  |
| `updated_at` | timestamp | non |  |  |

### `public.payments`

Paiements reçus. Méthodes : `cash`, `stripe`, `bank_transfer`. Statuts : `pending`, `approved`, `paid`, `failed`. `approved_by` contient l'identifiant de l'utilisateur qui a approuvé (sans clé étrangère).

| Colonne | Type | Obligatoire | Valeur par défaut | Remarques |
|---|---|:-:|---|---|
| `id` | bigint | oui | auto (identité) | **clé primaire** |
| `reference` | varchar(255) | non |  | unique |
| `sale_id` | bigint | non |  | → `sales.id` (on delete set null) |
| `client_id` | bigint | non |  | → `clients.id` (on delete set null) |
| `amount` | numeric(12,2) | oui |  |  |
| `method` | varchar(30) | oui |  | valeurs : `cash`, `stripe`, `bank_transfer` |
| `status` | varchar(30) | oui | pending | valeurs : `pending`, `approved`, `paid`, `failed` |
| `external_reference` | varchar(255) | non |  |  |
| `approved_by` | bigint | non |  |  |
| `approved_at` | timestamp | non |  |  |
| `notes` | text | non |  |  |
| `created_at` | timestamp | non |  |  |
| `updated_at` | timestamp | non |  |  |
| `stripe_intent_id` | varchar(255) | non |  |  |

### `public.payment_allocations`

Affectation d'un paiement à une vente : un paiement peut être réparti sur plusieurs ventes. `payable_type` / `payable_id` désignent l'objet payé (dans les données fournies : la vente, `App\Models\Sale`).

| Colonne | Type | Obligatoire | Valeur par défaut | Remarques |
|---|---|:-:|---|---|
| `id` | bigint | oui | auto (identité) | **clé primaire** |
| `payment_id` | bigint | oui |  | → `payments.id` (on delete cascade) |
| `payable_type` | varchar(255) | oui |  |  |
| `payable_id` | bigint | oui |  |  |
| `amount_applied` | numeric(12,2) | oui |  |  |
| `created_at` | timestamp | non |  |  |
| `updated_at` | timestamp | non |  |  |

### `public.stock_movements`

Journal des mouvements de stock : entrée (`in`) ou sortie (`out`), avec la source (`purchase`, `sale`, `refund`). `reference_id` est l'identifiant de l'achat, de la vente ou du remboursement concerné, selon `source`.

| Colonne | Type | Obligatoire | Valeur par défaut | Remarques |
|---|---|:-:|---|---|
| `id` | bigint | oui | auto (identité) | **clé primaire** |
| `product_id` | bigint | oui |  | → `products.id` (on delete cascade) |
| `quantity` | integer | oui |  |  |
| `type` | varchar(30) | oui |  | valeurs : `in`, `out` |
| `source` | varchar(30) | oui |  | valeurs : `purchase`, `sale`, `refund` |
| `reference_id` | bigint | non |  |  |
| `created_at` | timestamp | non |  |  |
| `updated_at` | timestamp | non |  |  |

### `public.invoices`

Factures de vente : une facture par vente (`sale_id` unique). Vide dans les données de démonstration.

| Colonne | Type | Obligatoire | Valeur par défaut | Remarques |
|---|---|:-:|---|---|
| `id` | bigint | oui | auto (identité) | **clé primaire** |
| `invoice_number` | varchar(255) | oui |  | unique |
| `sale_id` | bigint | oui |  | → `sales.id` (on delete cascade) · unique |
| `total` | numeric(10,2) | oui |  |  |
| `created_at` | timestamp | non |  |  |
| `updated_at` | timestamp | non |  |  |

### `public.purchase_invoices`

Factures d'achat : une facture par achat (`purchase_id` unique). Vide dans les données de démonstration.

| Colonne | Type | Obligatoire | Valeur par défaut | Remarques |
|---|---|:-:|---|---|
| `id` | bigint | oui | auto (identité) | **clé primaire** |
| `invoice_number` | varchar(255) | oui |  | unique |
| `purchase_id` | bigint | oui |  | → `purchases.id` (on delete cascade) · unique |
| `total` | numeric(10,2) | oui |  |  |
| `created_at` | timestamp | non |  |  |
| `updated_at` | timestamp | non |  |  |

## 2. Table de l'application (schéma `public`)

### `public.users`

Comptes de l'application (administrateur, employés, compte de l'agent IA). Le champ `role` définit les droits (voir ci-dessous). Les mots de passe sont stockés chiffrés (bcrypt).

| Colonne | Type | Obligatoire | Valeur par défaut | Remarques |
|---|---|:-:|---|---|
| `id` | integer | oui | auto (séquence) | **clé primaire** |
| `email` | varchar(255) | oui |  | unique |
| `password` | varchar(255) | oui |  |  |
| `is_active` | boolean | oui | true |  |
| `created_at` | timestamp | oui | now() |  |
| `updated_at` | timestamp | oui | now() |  |
| `first_name` | varchar(100) | oui |  |  |
| `last_name` | varchar(100) | oui |  |  |
| `role` | userrole | oui | USER_STOCK | valeurs : `ADMIN`, `USER_STOCK`, `USER_SALES`, `USER_PURCHASES`, `USER_SUPPLIERS`, `USER_DEVIS`, `USER_PAYMENTS`, `USER_CLIENTS`, `USER_REFUNDS` |

Comptes créés à l'installation :

| Compte | Rôle | Usage |
|---|---|---|
| `AI agent` (e-mail choisi à l'étape 0, par défaut `agent@system.internal`) | `USER_STOCK` (valeur par défaut) | Compte avec lequel l'agent IA s'authentifie auprès du backend |
| Administrateur (e-mail choisi à l'étape 0) | `ADMIN` | Connexion à l'application |

Rôles disponibles (`userrole`) : `ADMIN`, `USER_STOCK`, `USER_SALES`, `USER_PURCHASES`, `USER_SUPPLIERS`, `USER_DEVIS`, `USER_PAYMENTS`, `USER_CLIENTS`, `USER_REFUNDS`.

## 3. Tables de reporting (schéma `agent_reporting`)

### `agent_reporting.alert_notifications_sent`

Mémorise les alertes déjà envoyées par e-mail, pour ne pas renvoyer la même alerte à chaque contrôle. Contrainte `UNIQUE (alert_type, alert_key)`.

| Colonne | Type | Obligatoire | Valeur par défaut | Remarques |
|---|---|:-:|---|---|
| `id` | integer | oui | auto (séquence) | **clé primaire** |
| `alert_type` | varchar(50) | oui |  |  |
| `alert_key` | varchar(255) | oui |  |  |
| `sent_at` | timestamptz | oui | now() |  |

### `agent_reporting.file_exports`

Suivi des rapports téléchargeables générés (xlsx, csv, pdf) avec leur date d'expiration et l'utilisateur propriétaire.

| Colonne | Type | Obligatoire | Valeur par défaut | Remarques |
|---|---|:-:|---|---|
| `id` | bigint | oui | auto (séquence) | **clé primaire** |
| `filename` | varchar(255) | oui |  | unique |
| `filepath` | varchar(500) | oui |  |  |
| `report_type` | varchar(50) | oui |  |  |
| `file_format` | varchar(10) | oui |  |  |
| `original_name` | varchar(255) | oui |  |  |
| `user_id` | integer | oui |  | → `users.id` (on delete cascade) |
| `created_at` | timestamptz | oui | now() |  |
| `expires_at` | timestamptz | oui |  |  |

Types d'alertes pouvant apparaître dans `alert_notifications_sent.alert_type` (produits par `agent_reporting.fn_get_all_active_alerts()`) : `client_recontact`, `devis_expire`, `devis_followup`, `client_recurring_late`, `sale_payment_unpaid`, `sale_payment_partial`, `product_discontinue`, `reorder_suggestion`, `product_high_return_rate`, `sales_decline`.

Purge facultative pour permettre une nouvelle notification d'anciennes alertes :

```sql
DELETE FROM agent_reporting.alert_notifications_sent
WHERE sent_at < now() - INTERVAL '90 days';
```

## 4. Droits sur les tables

| Objet | `agent_user` (agent IA) | `backend_user` (backend) |
|---|---|---|
| 17 tables ERP (`public`) | aucun accès | lecture seule |
| `public.users` | aucun accès | lecture et écriture |
| Vues et fonctions `agent_reporting` | lecture / exécution | lecture / exécution |
| `agent_reporting.alert_notifications_sent` | lecture | lecture et écriture |
| `agent_reporting.file_exports` | lecture | lecture et écriture |
