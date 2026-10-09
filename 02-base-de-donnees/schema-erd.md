# Schéma des relations (ERD) et architecture

## 1. Architecture générale

```mermaid
flowchart LR
    subgraph PUB["Schéma public (ERP)"]
        T["17 tables ERP"]
        U["users"]
    end
    subgraph AR["Schéma agent_reporting"]
        V["27 vues<br/>(consultation)"]
        F1["Fonctions de traitement<br/>(statistiques, KPIs)"]
        F2["Fonctions de recommandation<br/>(alertes, suggestions)"]
        A["fn_get_all_active_alerts()"]
        TA["alert_notifications_sent<br/>file_exports"]
    end
    T --> V --> F1 --> F2 --> A
    AGENT(["agent_user<br/>(agent IA)"]) -- "lecture / exécution" --> AR
    BACK(["backend_user<br/>(backend FastAPI)"]) -- "lecture seule" --> T
    BACK -- "lecture / écriture" --> U
    BACK -- "lecture / exécution" --> AR
    BACK -- "écriture" --> TA
    AGENT -. "aucun accès" .-x PUB
```

Les trois couches du schéma `agent_reporting` :

| Couche | Rôle | Dossier du projet |
|---|---|---|
| **consultation** | Vues « détail » : jointures déjà faites, montants nets calculés | `*/consultation/` et `views.sql` |
| **traitement** | Fonctions de calcul : statistiques, agrégations par période, classements | `*/traitement/` |
| **recommandation** | Fonctions qui détectent les situations à traiter (clients à relancer, devis qui expirent, ruptures…) | `*/recommendation/` ou `*/recommandation/` |

## 2. Diagramme entité-relation

Seules les colonnes clés (clé primaire **PK**, clé étrangère **FK**, unique **UK**) sont représentées. Toutes les colonnes sont détaillées dans [tables.md](tables.md).

```mermaid
erDiagram
    categories ||--o{ products : "category_id"
    clients |o--o{ devis : "client_id"
    clients |o--o{ payments : "client_id"
    clients ||--o{ sales : "client_id"
    devis ||--o{ devis_items : "devis_id"
    payments ||--o{ payment_allocations : "payment_id"
    products ||--o{ devis_items : "product_id"
    products ||--o{ purchase_items : "product_id"
    products ||--o{ refund_items : "product_id"
    products ||--o{ sale_items : "product_id"
    products ||--o{ stock_movements : "product_id"
    purchases ||--o{ purchase_invoices : "purchase_id"
    purchases ||--o{ purchase_items : "purchase_id"
    refunds ||--o{ refund_items : "refund_id"
    sales ||--o{ invoices : "sale_id"
    sales |o--o{ payments : "sale_id"
    sales ||--o{ refunds : "sale_id"
    sales ||--o{ sale_items : "sale_id"
    suppliers ||--o{ products : "supplier_id"
    suppliers ||--o{ purchases : "supplier_id"
    users ||--o{ file_exports : "user_id"
    users |o..o{ devis : "created_by (sans cle etrangere)"
    users |o..o{ payments : "approved_by (sans cle etrangere)"
    users ||--o{ file_exports : "user_id"
    categories {
        bigint id PK
    }
    clients {
        bigint id PK
    }
    suppliers {
        bigint id PK
    }
    products {
        bigint id PK
        varchar reference UK
        bigint category_id FK
        bigint supplier_id FK
    }
    devis {
        bigint id PK
        varchar reference UK
        bigint client_id FK
    }
    devis_items {
        bigint id PK
        bigint devis_id FK
        bigint product_id FK
    }
    sales {
        bigint id PK
        varchar reference UK
        bigint client_id FK
    }
    sale_items {
        bigint id PK
        bigint sale_id FK
        bigint product_id FK
    }
    purchases {
        bigint id PK
        bigint supplier_id FK
    }
    purchase_items {
        bigint id PK
        bigint purchase_id FK
        bigint product_id FK
    }
    refunds {
        bigint id PK
        bigint sale_id FK
    }
    refund_items {
        bigint id PK
        bigint refund_id FK
        bigint product_id FK
    }
    payments {
        bigint id PK
        varchar reference UK
        bigint sale_id FK
        bigint client_id FK
    }
    payment_allocations {
        bigint id PK
        bigint payment_id FK
    }
    stock_movements {
        bigint id PK
        bigint product_id FK
    }
    invoices {
        bigint id PK
        varchar invoice_number UK
        bigint sale_id FK,UK
    }
    purchase_invoices {
        bigint id PK
        varchar invoice_number UK
        bigint purchase_id FK,UK
    }
    users {
        integer id PK
        varchar email UK
    }
    file_exports {
        bigint id PK
        varchar filename UK
        integer user_id FK
    }
    alert_notifications_sent {
        integer id PK
    }
```

## 3. Liste des relations

| Table parent | Table enfant | Colonne (enfant) | Suppression du parent |
|---|---|---|---|
| `categories` | `products` | `category_id` | cascade |
| `clients` | `devis` | `client_id` | set null |
| `clients` | `payments` | `client_id` | set null |
| `clients` | `sales` | `client_id` | cascade |
| `devis` | `devis_items` | `devis_id` | cascade |
| `payments` | `payment_allocations` | `payment_id` | cascade |
| `products` | `devis_items` | `product_id` | cascade |
| `products` | `purchase_items` | `product_id` | cascade |
| `products` | `refund_items` | `product_id` | cascade |
| `products` | `sale_items` | `product_id` | cascade |
| `products` | `stock_movements` | `product_id` | cascade |
| `purchases` | `purchase_invoices` | `purchase_id` | cascade |
| `purchases` | `purchase_items` | `purchase_id` | cascade |
| `refunds` | `refund_items` | `refund_id` | cascade |
| `sales` | `invoices` | `sale_id` | cascade |
| `sales` | `payments` | `sale_id` | set null |
| `sales` | `refunds` | `sale_id` | cascade |
| `sales` | `sale_items` | `sale_id` | cascade |
| `suppliers` | `products` | `supplier_id` | cascade |
| `suppliers` | `purchases` | `supplier_id` | cascade |
| `users` | `file_exports` | `user_id` | cascade |
| `users` | `devis` | `created_by` | *(pas de clé étrangère)* |
| `users` | `payments` | `approved_by` | *(pas de clé étrangère)* |
| `users` | `file_exports` | `user_id` | supprime aussi ses exports (`cascade`) |

Liens logiques (non déclarés comme clés étrangères) :

- `payment_allocations.payable_type` + `payable_id` : désignent l'objet payé (la vente) ;
- `stock_movements.source` + `reference_id` : désignent l'achat, la vente ou le remboursement à l'origine du mouvement.

## 4. Lecture rapide du modèle

- **Ventes** : `clients` → `sales` → `sale_items` → `products`. Un paiement (`payments`) est affecté à une vente via `payment_allocations`. Un remboursement (`refunds` / `refund_items`) se rattache à une vente.
- **Achats** : `suppliers` → `purchases` → `purchase_items` → `products`.
- **Devis** : `clients` → `devis` → `devis_items` → `products`.
- **Stock** : chaque achat, vente ou remboursement génère une ligne dans `stock_movements`.
