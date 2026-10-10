# Authentification et autorisations

## 1. Vue d'ensemble

```
POST /api/auth/login  →  JWT  →  Authorization: Bearer <token>  →  get_current_user  →  require_role (optionnel)
```

| Élément | Fichier |
|---|---|
| Hash et JWT | `core/security.py` |
| Dépendances FastAPI | `core/dependencies.py` |
| Matrice de permissions | `core/role_permissions.py` |
| Logique de login | `services/auth_service.py` |
| Accès utilisateurs | `repositories/user_repository.py`, `repositories/admin_user_repository.py` |
| Modèle | `models/user.py` |

## 2. Routes d'authentification

```
POST /api/auth/login      → retourne un JWT        (public)
POST /api/auth/register   → création de compte      (public, ⚠️ non fonctionnelle)
GET  /api/auth/me         → infos utilisateur courant
```

Toutes les routes de reporting sont protégées via `Depends(get_current_user)`. Le schéma OAuth2 (`OAuth2PasswordBearer`) pointe sur `/api/auth/login`, ce qui permet le bouton « Authorize » de Swagger.

## 3. Le JWT

| Propriété | Valeur |
|---|---|
| Bibliothèque | `python-jose` |
| Algorithme | `HS256` par défaut (`ALGORITHM`) |
| Clé | `SECRET_KEY` |
| Durée | `ACCESS_TOKEN_EXPIRE_MINUTES` |
| Contenu | `sub` = id de l'utilisateur (chaîne), `exp` |
| Rôle | **Absent du token** : relu en base à chaque requête |

Conséquence : un changement de rôle ou une désactivation de compte est **immédiat**, sans attendre l'expiration du token.

### Réponse du login

```json
{ "access_token": "<jwt>", "token_type": "bearer" }
```

## 4. Dépendances FastAPI

### `get_current_user`

1. Décode le token ; `401 Token invalide ou expiré` si invalide ou sans `sub`.
2. Recharge l'utilisateur par id ; `401 Utilisateur introuvable` s'il n'existe plus.
3. `401 Compte désactivé` si `is_active` est faux.

### `require_role(*roles)`

Fabrique une dépendance qui renvoie `403 Accès non autorisé` si le rôle de l'utilisateur n'est pas dans la liste. Utilisée par `admin_users.py` (`require_role("admin")`).

## 5. Rôles

L'enum `UserRole` définit **9 rôles** : `admin` et 8 rôles de domaine. Valeur par défaut d'un nouveau compte : `user_stock`.

### Matrice de permissions

| Rôle | Page frontend | Type d'export | Vues SQL (NL-to-SQL) |
|---|---|---|---|
| `admin` | Toutes | Tous | Toutes |
| `user_stock` | `/rapport-stock` | `stock` | `v_products_catalog`, `v_low_stock_products`, `v_categories`, `v_stock_movements` |
| `user_sales` | `/rapport-ventes` | `sales` | `v_sales_detail`, `v_sale_items` |
| `user_purchases` | `/rapport-purchases` | `purchases` | `v_purchases`, `v_purchase_items` |
| `user_suppliers` | `/rapport-suppliers` | `suppliers` | `v_suppliers`, `v_purchases` |
| `user_devis` | `/rapport-devis` | `devis` | `v_devis`, `v_devis_items` |
| `user_payments` | `/rapport-paiements` | `payments` | `v_payments`, `v_payment_allocations` |
| `user_clients` | `/clients-report` | `clients` | `v_clients` |
| `user_refunds` | `/rapport-remboursements` | `refunds` | `v_refunds`, `v_refund_items` |

### Structures exposées par `role_permissions.py`

| Structure / fonction | Rôle |
|---|---|
| `ROLE_PAGES` | Pages frontend autorisées (miroir côté frontend). `None` = tout |
| `ROLE_TOOLS` | Liste blanche des noms de tools de l'agent par rôle. Chaque rôle de domaine reçoit les tools de son domaine, des « résolveurs » (nom → id) et `generate_report_file` |
| `ROLE_VIEWS` | Vues SQL autorisées pour le fallback NL-to-SQL de l'agent |
| `ROLE_REPORT_TYPES` | `report_type` exportable/téléchargeable par rôle |
| `filter_tools_by_role()` · `get_views_for_role()` · `is_report_type_allowed_for_role()` | Fonctions d'aide |

Un rôle inconnu n'obtient **aucun** droit (*deny by default*). Dans ce backend, seule `is_report_type_allowed_for_role` et `ROLE_REPORT_TYPES` sont utilisées (module d'export) ; le filtrage des tools et des vues est destiné à l'agent IA.

## 6. Gestion des comptes

### Création par un admin — `POST /api/admin/users`

Corps : `{first_name, last_name, email, role}`.

1. Contrôle que l'email n'existe pas.
2. Génère un **mot de passe temporaire** de 12 caractères (`secrets`).
3. Crée l'utilisateur (bcrypt) puis envoie un email de bienvenue en **tâche de fond** (`pw_email_service.py`).
4. Retourne `{user, temp_password}`.

Autres routes admin : liste, changement de rôle (`PATCH …/{id}/role`), suppression (un admin ne peut pas se supprimer lui-même).

### Opérations de l'utilisateur connecté

| Route | Comportement |
|---|---|
| `PUT /api/user/password` | Vérifie l'ancien mot de passe ; le champ `email` du corps est **ignoré** (seul le JWT identifie le compte) ; minimum 8 caractères |
| `PUT /api/user/profile` | Met à jour prénom, nom, email (⚠️ cassée, voir §8) |
| `PUT /api/user/language` | Langue `fr` / `en` / `ar` (⚠️ cassée, voir §8) |

### Réinitialisation — `POST /api/user/reset-password`

Corps : `{email, new_password}`. ⚠️ **Route publique sans jeton de vérification** (voir §8).

## 7. Compte de service de l'agent IA

L'agent IA s'authentifie comme un utilisateur normal, avec un compte dédié. D'après les commentaires du code (`api/reports_api_tools/generation.py`), ce compte doit avoir le rôle **`admin`** en base : sinon `/api/reports/export` bloquerait tous les exports du chatbot au domaine du rôle du compte de service.

```sql
INSERT INTO users (first_name, last_name, email, password, role, is_active, created_at, updated_at)
VALUES ('AI', 'Agent', 'agent@system.internal', '<hash_bcrypt>', 'admin', true, NOW(), NOW());
```

- Le hash se génère avec `core/generate_hash.py` (bcrypt).
- L'email doit être **identique** à `AGENT_SERVICE_EMAIL` côté agent ; côté backend, le champ de configuration `agent_service_email` existe mais n'est pas encore exploité dans `generation.py`.
- Le vrai contrôle par rôle de l'utilisateur humain s'effectue **côté agent**, avant l'appel HTTP.
- Pour le chat, l'agent réutilise le **token de l'utilisateur** (relayé dans le header `Authorization`) ; le compte de service sert surtout aux exports. Voir `agent-ia.md`.

## 8. Points de sécurité connus

| Gravité | Constat |
|---|---|
| 🔴 Haute | `POST /api/user/reset-password` : n'importe qui peut changer le mot de passe d'un email connu |
| 🔴 Haute | Routers `dashboard`, `sales`, `refunds` sans authentification |
| 🟠 Moyenne | Aucun router de rapport ne vérifie le rôle : un `user_stock` authentifié peut appeler les routes des autres domaines |
| 🟠 Moyenne | `/api/auth/register` public ; le mot de passe temporaire est renvoyé dans la réponse **et** par email, sans changement forcé à la première connexion |
| 🟡 Basse | `authenticate` ne contrôle pas `is_active` (le blocage intervient à la requête suivante, dans `get_current_user`) |
| 🟡 Basse | Pas de limitation de tentatives de connexion, pas de révocation de token |

Détails et correctifs : [`points-attention.md`](points-attention.md).
