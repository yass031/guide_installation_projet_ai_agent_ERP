# NextAct — Installer la base de données (PostgreSQL + pgAdmin 4)

Bienvenue ! Ce guide vous accompagne pas à pas, de la première à la dernière étape, pour installer la base de données de l'application NextAct. Vous n'avez besoin d'aucune connaissance particulière en SQL : vous allez **copier-coller** du code dans pgAdmin 4, et je vous explique à chaque étape ce que vous faites, ce que vous devez voir à l'écran, et ce qu'il faut absolument noter.

Cette documentation est la **première des quatre** : 1. Base de données (ici) → 2. Agent IA → 3. Backend → 4. Frontend.

## Ce que vous allez obtenir

Une base PostgreSQL nommée `nextact_ecom` contenant :

- les **17 tables de l'ERP** (produits, ventes, clients, paiements, devis, achats, remboursements, stock…) et leurs relations ;
- une table **`users`** avec deux comptes : un **administrateur** (vous) et le compte **« AI agent »** ;
- des **données de démonstration** (facultatives) pour tester tout de suite ;
- un schéma **`agent_reporting`** : 27 vues et 44 fonctions qui servent au tableau de bord, aux alertes et à l'agent IA ;
- deux **rôles de sécurité** : `agent_user` (l'agent IA, qui ne peut que lire les vues) et `backend_user` (le backend).

## Les fichiers de cette documentation

| Fichier | Contenu |
|---|---|
| `README.md` | Ce guide d'installation |
| `install_database.sql` | **Tout le code SQL à copier-coller, dans l'ordre** |
| `schema-erd.md` | Schéma de l'architecture et diagramme des relations entre tables |
| `tables.md` | Description de chaque table et de chaque colonne |
| `vues.md` | Les 27 vues |
| `fonctions.md` | Les 44 fonctions, avec leurs paramètres et un exemple pour chacune |

---

## Avant de commencer

Il vous faut :

- **PostgreSQL** et **pgAdmin 4** installés ;
- le **mot de passe de l'utilisateur `postgres`** (celui choisi à l'installation de PostgreSQL) ;
- le fichier **`install_database.sql`** (dans ce dossier) ;
- **Python 3.11 ou plus** (uniquement pour le petit test de connexion de la partie 3).

### ✍️ Les 7 informations que vous allez choisir

Au début de l'installation, vous choisirez 7 valeurs. **Prenez une feuille ou un gestionnaire de mots de passe et notez-les au fur et à mesure** : vous en aurez besoin dans les fichiers `.env` de l'agent IA et du backend, et pour vous connecter à l'application.

| N° | Ce que vous choisissez | Où vous le réutiliserez |
|:-:|---|---|
| 1 | Mot de passe du rôle PostgreSQL `agent_user` | `.env` de l'agent IA → `DB_PASS` |
| 2 | Mot de passe du rôle PostgreSQL `backend_user` | `.env` du backend → `DB_PASS` |
| 3 | E-mail du compte « AI agent » (vous pouvez garder `agent@system.internal`) | `.env` de l'agent IA → `AGENT_SERVICE_EMAIL` |
| 4 | Mot de passe du compte « AI agent » | `.env` de l'agent IA → `AGENT_SERVICE_PASSWORD` |
| 5 | Votre nom complet (prénom puis nom) | Affiché dans l'application |
| 6 | **Votre e-mail** d'administrateur | Identifiant de connexion à l'application |
| 7 | **Votre mot de passe** d'administrateur | Mot de passe de connexion à l'application |

> 💡 Choisissez des mots de passe solides et **différents** les uns des autres. N'utilisez pas le caractère `'` (apostrophe) dans les mots de passe.

---

## Partie 1 — Préparer pgAdmin

### 1.1 Ouvrir pgAdmin 4 et se connecter

1. Lancez **pgAdmin 4**. Au premier démarrage, il peut vous demander de créer un « master password » : c'est un mot de passe propre à pgAdmin, choisissez-en un.
2. Dans la colonne de gauche, dépliez **Servers**, puis cliquez sur votre serveur (par exemple *PostgreSQL 16*). Saisissez le mot de passe de l'utilisateur **`postgres`**.
3. Si aucun serveur n'apparaît : clic droit sur **Servers → Register → Server…**
   - onglet **General** → *Name* : `PostgreSQL local` ;
   - onglet **Connection** → *Host name/address* : `localhost`, *Port* : `5432`, *Username* : `postgres`, *Password* : votre mot de passe ;
   - cliquez sur **Save**.

✅ **Ce que vous devez voir** : le serveur est déplié et vous voyez **Databases** et **Login/Group Roles**.

> ⚠️ **Toute l'installation se fait connecté en tant que `postgres`.**

### 1.2 Créer la base de données `nextact_ecom`

1. Clic droit sur **Databases → Create → Database…**
2. Dans *Database*, écrivez exactement : `nextact_ecom`
3. *Owner* : `postgres`. Cliquez sur **Save**.

✅ **Ce que vous devez voir** : `nextact_ecom` apparaît dans la liste des bases.

> Gardez ce nom exact : les fichiers `.env` utilisent `DB_NAME=nextact_ecom`. Si vous choisissez un autre nom, pensez à le modifier partout.

### 1.3 Ouvrir l'éditeur SQL (Query Tool) sur la bonne base

1. Faites un **clic droit sur `nextact_ecom`** (la base, pas le serveur) → **Query Tool**.
2. Vérifiez le titre de l'onglet qui s'ouvre : il doit mentionner **`nextact_ecom`**.
3. Cliquez sur l'icône **dossier (Open File)** de la barre d'outils, et ouvrez **`install_database.sql`**. Vous pouvez aussi ouvrir le fichier dans un éditeur de texte et le coller dans la fenêtre.

> 🎯 **Point d'attention n°1 — un seul Query Tool.** Gardez **cet onglet ouvert et utilisez-le pour toute l'installation**. L'ÉTAPE 0 « mémorise » vos informations pour la durée de la session : si vous fermez l'onglet, vous devrez ré-exécuter l'ÉTAPE 0 (le script vous le dira par un message clair).
>
> 🎯 **Point d'attention n°2 — la bonne base.** Si l'onglet indique `postgres` au lieu de `nextact_ecom`, fermez-le et recommencez en faisant le clic droit sur `nextact_ecom`.

#### Comment exécuter une étape

Le fichier est découpé en **étapes numérotées de 0 à 8**. Chaque étape commence par un grand bandeau :

```
-- ======================================================================
-- ÉTAPE 2 — CRÉER LA TABLE users ...
```

Pour chaque étape :

1. **Sélectionnez** à la souris tout le texte **depuis le bandeau de l'étape jusqu'à juste avant le bandeau de l'étape suivante** ;
2. Appuyez sur **F5** (ou sur le bouton ▶) : seule la sélection est exécutée ;
3. Regardez l'onglet **Messages** en bas : vous devez lire **`Query returned successfully`** (ou voir un tableau de résultats) ;
4. Passez à l'étape suivante.

> 🎯 **Point d'attention n°3 — dans l'ordre, une seule fois.** Exécutez les étapes dans l'ordre, sans en sauter. Les étapes 1, 2, 5 et 6 ne doivent être exécutées **qu'une seule fois** ; en cas de doute ou d'erreur, voir la section *Dépannage*.

---

## Partie 2 — Exécuter le fichier SQL, étape par étape

Le fichier contient 9 étapes (de l'ÉTAPE 0 à l'ÉTAPE 8). Suivez-les dans l'ordre : chacune est expliquée ci-dessous.

### ÉTAPE 0 — Saisir VOS informations

C'est **la seule partie du fichier que vous modifiez**. Au début du fichier, vous trouvez 7 lignes qui ressemblent à ceci :

```sql
SELECT set_config('nextact.db_agent_password',     'CHANGE_ME_mot_de_passe_agent_user',   false);
SELECT set_config('nextact.db_backend_password',   'CHANGE_ME_mot_de_passe_backend_user', false);
SELECT set_config('nextact.agent_service_email',   'agent@system.internal',               false);
SELECT set_config('nextact.agent_service_password','CHANGE_ME_mot_de_passe_compte_agent', false);
SELECT set_config('nextact.admin_full_name',       'CHANGE_ME Prénom Nom',                false);
SELECT set_config('nextact.admin_email',           'CHANGE_ME@exemple.com',               false);
SELECT set_config('nextact.admin_password',        'CHANGE_ME_mot_de_passe_admin',        false);
```

Dans chaque ligne, **remplacez uniquement le texte entre les guillemets simples** (celui qui commence par `CHANGE_ME`) par votre valeur. Les numéros 1 à 7 correspondent au tableau « Les 7 informations » plus haut.

Exemple pour la dernière ligne, si votre mot de passe administrateur est `MonMotDePasse2026` :

```sql
SELECT set_config('nextact.admin_password',        'MonMotDePasse2026',        false);
```

Ne touchez ni aux guillemets, ni au mot `false`, ni au début de la ligne.

Sélectionnez ensuite **tout le bloc « ÉTAPE 0 »** (jusqu'au bandeau « ÉTAPE 1 ») et appuyez sur **F5**.

✅ **Ce que vous devez voir** : un petit tableau de **7 lignes** (les mots de passe sont masqués par des `********`). **Aucune ligne ne doit contenir `CHANGE_ME`.** Si c'est le cas, corrigez la ligne concernée et ré-exécutez l'étape 0.

> 🛡️ Une sécurité est prévue : si vous oubliez de remplacer une valeur, les étapes suivantes **s'arrêtent avec un message clair** au lieu de créer des comptes avec un mot de passe d'exemple.

### ÉTAPE 1 — Créer les 17 tables de l'ERP

Sélectionnez le bloc **ÉTAPE 1**, puis **F5**.

Ce que cela fait : crée les tables `categories`, `clients`, `suppliers`, `products`, `devis`, `devis_items`, `sales`, `sale_items`, `purchases`, `purchase_items`, `refunds`, `refund_items`, `payments`, `payment_allocations`, `stock_movements`, `invoices`, `purchase_invoices`, ainsi que leurs relations.

✅ **Vérification** (à taper dans le Query Tool, ou à sélectionner puis F5) :

```sql
SELECT count(*) FROM information_schema.tables WHERE table_schema = 'public';
```

Résultat attendu : **17**.

### ÉTAPE 2 — Créer la table des comptes `users`

Bloc **ÉTAPE 2**, puis **F5**. Cette étape active aussi l'extension `pgcrypto`, qui sert à chiffrer les mots de passe.

✅ Résultat attendu : `Query returned successfully`.

### ÉTAPE 3 — Charger les données de démonstration (facultatif)

Bloc **ÉTAPE 3**, puis **F5**. Cela remplit la base de catégories, clients, fournisseurs, produits, ventes, paiements, devis… pour pouvoir tester l'application immédiatement.

- **Si vous voulez une base vide** : sautez simplement cette étape.
- ✅ Vous verrez défiler plusieurs petits résultats `setval` : c'est normal (ils recalent les compteurs d'identifiants).

Vérification rapide :

```sql
SELECT count(*) FROM clients;   -- attendu : 21 avec les données de démonstration
```

### ÉTAPE 4 — Créer le schéma `agent_reporting`, les rôles et les droits

Bloc **ÉTAPE 4**, puis **F5**.

Ce que cela fait :

- crée le schéma `agent_reporting` ;
- crée le rôle **`agent_user`** avec **votre mot de passe n°1** : il pourra uniquement **lire les vues** et exécuter les fonctions. Il n'aura **aucun accès** aux tables brutes ;
- crée le rôle **`backend_user`** avec **votre mot de passe n°2** : lecture seule sur les tables de l'ERP, lecture et écriture sur `users`.

✅ Résultat attendu : `Query returned successfully`.

> Si vous obtenez le message « La valeur … n'est pas renseignée », retournez à l'étape 0 et ré-exécutez-la **dans ce même onglet**.

### ÉTAPE 5 — Créer les deux comptes de l'application

Bloc **ÉTAPE 5**, puis **F5**. Cette étape crée dans `users` :

- le compte **« AI agent »** (avec vos informations 3 et 4) ;
- le compte **administrateur** (avec vos informations 5, 6 et 7).

Les mots de passe sont **chiffrés** avant d'être enregistrés : personne ne pourra les relire dans la base.

Le code utilise temporairement le rôle `backend_user` pour insérer ces comptes : cela vérifie au passage que ses droits fonctionnent.

> ⚠️ Si cette étape affiche une erreur, tapez **`RESET ROLE;`** et exécutez-le, puis relisez le message d'erreur avant de recommencer.

### ÉTAPE 6 — Les rôles des utilisateurs

Bloc **ÉTAPE 6**, puis **F5**. Cette étape :

- crée la liste des rôles possibles (`ADMIN`, `USER_STOCK`, `USER_SALES`, `USER_PURCHASES`, `USER_SUPPLIERS`, `USER_DEVIS`, `USER_PAYMENTS`, `USER_CLIENTS`, `USER_REFUNDS`) ;
- sépare le nom en **prénom** et **nom** ;
- donne le rôle **`ADMIN`** à l'administrateur dont vous avez choisi l'e-mail **et** au compte « AI agent » (c'est avec ce compte que l'agent IA appelle le backend).

✅ **Ce que vous devez voir** : un tableau de **2 lignes** — le compte `AI agent` et **votre e-mail** — **toutes deux avec le rôle `ADMIN`**.

> 🎯 **Point d'attention n°4 — une seule fois.** Cette étape modifie la structure de `users` : ne l'exécutez pas deux fois.

### ÉTAPE 7 — Installer les vues et les fonctions

Bloc **ÉTAPE 7** (le plus long du fichier), puis **F5**. Patientez quelques secondes.

Cette étape installe, dans le bon ordre, tout le schéma `agent_reporting` : **27 vues, 44 fonctions, 2 tables** (`alert_notifications_sent` et `file_exports`) et les droits associés.

✅ Résultat attendu : `Query returned successfully`.

> En cas d'erreur, l'onglet **Messages** indique la ligne en cause ; cherchez au-dessus le commentaire `-- ---- [n/64] nom_du_fichier.sql` pour savoir quel fichier est concerné.

### ÉTAPE 8 — Vérifier que tout est en place

Bloc **ÉTAPE 8**. Exécutez les requêtes une par une (sélectionnez-en une, **F5**) et comparez avec les résultats attendus :

| Requête | Résultat attendu |
|---|---|
| nombre de vues | **27** |
| nombre de fonctions | **44** |
| tables de `agent_reporting` | `alert_notifications_sent`, `file_exports` |
| contenu de `users` | 2 lignes, toutes deux en `ADMIN` |
| `fn_get_all_active_alerts()` | s'exécute sans erreur |
| `v_sales_detail` | quelques lignes de ventes |

#### Vérifier la sécurité de `agent_user`

C'est le test le plus important : l'agent IA ne doit **pas** pouvoir lire les tables brutes.

1. Dans pgAdmin : clic droit sur **Servers → Register → Server…**
   - *General → Name* : `nextact-agent` ;
   - *Connection* → *Host* : `localhost`, *Port* : `5432`, *Maintenance database* : `nextact_ecom`, *Username* : `agent_user`, *Password* : **votre mot de passe n°1** ;
   - **Save**.
2. Ouvrez un **Query Tool** sur `nextact_ecom` de ce nouveau serveur, puis exécutez ces deux requêtes **séparément** :

```sql
SELECT count(*) FROM agent_reporting.v_clients;   -- doit FONCTIONNER
SELECT count(*) FROM public.clients;              -- doit ÉCHOUER : permission denied
```

✅ La première renvoie un nombre ; la seconde **doit** afficher `permission denied for schema public`. C'est exactement ce qu'on veut : l'agent IA ne voit que les vues.

---

## Partie 3 — Préparer les fichiers `.env` et tester la connexion

Il reste à reporter vos informations dans les fichiers `.env`. Voici la partie « base de données » de chacun (le reste de chaque `.env` est expliqué dans les documentations 2 et 3).

**`.env` de l'agent IA**

```dotenv
DB_HOST=localhost
DB_PORT=5432
DB_NAME=nextact_ecom
DB_USER=agent_user
DB_PASS=<votre information n°1>
AGENT_SERVICE_EMAIL=<votre information n°3>
AGENT_SERVICE_PASSWORD=<votre information n°4>
BACKEND_URL=http://localhost:8000/api
```

**`.env` du backend**

```dotenv
DB_HOST=localhost
DB_PORT=5432
DB_NAME=nextact_ecom
DB_USER=backend_user
DB_PASS=<votre information n°2>
```

> ⚠️ Ne publiez **jamais** un fichier `.env` (GitHub, messagerie, capture d'écran). Ajoutez `.env` à votre `.gitignore`.

### Tester la connexion avec Python

Le dossier `db/` du projet contient `connection.py` et `conn-test.py`.

1. Installez les dépendances : `pip install asyncpg python-dotenv`
2. Placez le `.env` de l'agent IA **deux dossiers au-dessus du dossier `db`** (le script le cherche à cet endroit : par exemple, si `db` est dans `agent/src/db/`, le fichier est `agent/.env`).
3. Ouvrez un terminal **dans le dossier `db`** et lancez :

```bash
python conn-test.py
```

✅ **Ce que vous devez voir** :

```
✅ Connexion réussie !
Version PostgreSQL : PostgreSQL ...
search_path actif : agent_reporting
```

Bravo : la base est prête. Vous pouvez passer à la **documentation 2/4 — Agent IA**.

---

## ✅ Check-list finale

- [ ] Base `nextact_ecom` créée
- [ ] Étape 0 remplie, sans aucun `CHANGE_ME` ; **les 7 informations sont notées**
- [ ] Étapes 1 à 7 exécutées sans erreur (17 tables, puis `users`, puis `agent_reporting`)
- [ ] Étape 8 : 27 vues, 44 fonctions, 2 comptes dans `users`, tous deux en `ADMIN`
- [ ] `agent_user` lit `v_clients` mais ne peut pas lire `public.clients`
- [ ] `.env` de l'agent IA et du backend renseignés
- [ ] `python conn-test.py` affiche « Connexion réussie »

---

## Dépannage

| Message ou symptôme | Cause | Solution |
|---|---|---|
| `La valeur "…" n'est pas renseignée` | Étape 0 non remplie, ou onglet fermé et rouvert | Remplacez les `CHANGE_ME`, ré-exécutez l'étape 0 **dans le même onglet**, puis relancez l'étape en échec |
| `syntax error at or near …` | Sélection incomplète lors du copier-coller | Sélectionnez bien **du bandeau de l'étape jusqu'au bandeau suivant** |
| `relation "…" already exists` / `type "userrole" already exists` / `column "…" already exists` | L'étape a déjà été exécutée | Ne la rejouez pas. Pour recommencer : supprimez la base dans pgAdmin (clic droit → Delete (Force)) et reprenez à la section 1.2 |
| `relation "…" does not exist` | Une étape précédente a été sautée | Reprenez l'ordre : étape 1, puis 2, puis la suite |
| `permission denied` lors de l'étape 5 | Le rôle est resté sur `backend_user` après une erreur | Exécutez `RESET ROLE;` puis relancez l'étape 5 |
| `permission denied for schema public` avec `agent_user` | **Normal** : c'est la protection voulue | Utilisez les vues `agent_reporting.v_…` |
| Python : `Variables de connexion DB manquantes` | `.env` introuvable ou incomplet | Vérifiez son emplacement et les variables `DB_NAME`, `DB_USER`, `DB_PASS` |
| Python : `password authentication failed` | Mot de passe du `.env` différent de l'information n°1 ou n°2 | Corrigez le `.env`, ou changez le mot de passe : `ALTER ROLE agent_user PASSWORD 'nouveau';` |
| Python : `database "…" does not exist` | `DB_NAME` incorrect | Utilisez `nextact_ecom` |
| Python : `connection refused` | PostgreSQL arrêté ou mauvais port | Démarrez le service PostgreSQL et vérifiez `DB_PORT` |
| Python : `ModuleNotFoundError: asyncpg` | Dépendance absente | `pip install asyncpg python-dotenv` |

---
