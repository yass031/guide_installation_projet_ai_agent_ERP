# NextAct — Installer et lancer l'agent IA

Bienvenue ! Ce guide vous accompagne pas à pas pour installer, configurer et lancer **l'agent IA** de NextAct : l'assistant qui répond en langage naturel aux questions sur le stock, les ventes, les clients, les devis, les achats, les paiements et les remboursements.

Cette documentation est la **deuxième des quatre** : 1. Base de données → **2. Agent IA (ici)** → 3. Backend → 4. Frontend.

## Ce que fait l'agent

Vous écrivez *« quels produits sont en stock bas ? »* : l'agent choisit l'outil adapté parmi 81, interroge les données de l'entreprise, puis rédige la réponse. Il tient compte du **rôle** de chaque utilisateur (un utilisateur « ventes » ne voit que les ventes) et sait aussi générer des exports Excel, CSV ou PDF.

L'agent est un petit **service indépendant** (port `8080`) qui s'appuie sur trois éléments :

| Élément | Pourquoi |
|---|---|
| **La base PostgreSQL** (documentation 1) | Utilisée directement par le plan B de l'agent (requêtes SQL en lecture seule) |
| **Le backend** (documentation 3, port `8000`) | Vérifie l'identité des utilisateurs et fournit les données des 81 outils |
| **Groq** (service d'IA en ligne) | Le modèle de langage qui comprend les questions et rédige les réponses |

## Les fichiers de cette documentation

| Fichier | Contenu |
|---|---|
| `README.md` | Ce guide d'installation et de lancement |
| `architecture-agent.md` | L'architecture en détail : composants, parcours d'une question, logique de décision |
| `tools-et-schemas.md` | Les 81 outils, leurs paramètres et les droits de chaque rôle |
| `system-prompt.md` | Les instructions données au modèle, expliquées règle par règle |
| `securite.md` | Les protections, ce qui a été vérifié, les limites connues et leurs correctifs |

---

## Avant de commencer

Il vous faut :

- ✅ **La documentation 1 terminée** : la base `nextact_ecom` est installée. Vous avez besoin de trois des informations notées à l'étape 0 :
  - **n° 1** : le mot de passe du rôle `agent_user` ;
  - **n° 3** : l'e-mail du compte « AI agent » ;
  - **n° 4** : le mot de passe du compte « AI agent ».
- ✅ **Python 3.11 ou plus récent**. Pour vérifier, ouvrez un terminal et tapez `python --version`.
- ✅ **Une connexion Internet** (pour Groq, et pour le premier téléchargement du modèle de sélection des outils).
- ✅ **Le dossier du code de l'agent** (`ai-agent/`).
- ✅ **Un compte Groq** : la création est expliquée à l'étape 3.

> 💡 **Le backend (documentation 3) n'est pas nécessaire pour *démarrer* l'agent**, mais il l'est pour *discuter* avec lui : l'agent vérifie chaque utilisateur auprès du backend et lui demande toutes les données. Vous pouvez donc suivre les étapes 1 à 5 maintenant, et faire le test de conversation (étape 6) une fois le backend lancé.

### Ordre de démarrage de toute l'application

1. **PostgreSQL** (démarré en tâche de fond)
2. **Le backend** — port `8000`
3. **L'agent IA** — port `8080` (ce guide)
4. **Le frontend**

---

## Étape 1 — Vérifier la structure du dossier

Le dossier `ai-agent/` doit ressembler à ceci. **Le fichier `.env` n'y figure pas encore** : vous allez le créer à l'étape 3.

```
ai-agent/
├── .env                 ← à créer vous-même (étape 3)
└── src/
    ├── main.py          ← point de départ de l'agent
    ├── agent/           ← le cerveau de l'agent
    ├── tools/           ← les 81 outils
    └── db/
        ├── connection.py
        └── conn-test.py
```

> 🎯 **Point d'attention n°1 — l'emplacement du `.env`.** Le fichier `.env` se place **dans le dossier `ai-agent/`, à côté du dossier `src/`** (et non dans `src/`). C'est à cet endroit que `main.py` et `db/connection.py` le cherchent.

## Étape 2 — Installer Python et les bibliothèques

Ouvrez un terminal **dans le dossier `ai-agent/`**, puis créez un « environnement virtuel » (un espace isolé pour les bibliothèques du projet).

**Windows (PowerShell) :**

```powershell
cd ai-agent
python -m venv venv
venv\Scripts\activate
```

**macOS / Linux :**

```bash
cd ai-agent
python3 -m venv venv
source venv/bin/activate
```

✅ **Ce que vous devez voir** : `(venv)` au début de la ligne du terminal.

> Si PowerShell refuse d'activer l'environnement (message sur l'« exécution de scripts »), tapez d'abord : `Set-ExecutionPolicy -Scope Process -ExecutionPolicy RemoteSigned`, puis relancez `venv\Scripts\activate`.

Installez ensuite les bibliothèques :

```bash
pip install fastapi "uvicorn[standard]" python-dotenv httpx asyncpg groq sentence-transformers numpy pydantic
```

> Si le dossier contient un fichier `requirements.txt`, utilisez plutôt `pip install -r requirements.txt`.
>
> ⏳ L'installation de `sentence-transformers` télécharge aussi PyTorch : cela peut prendre plusieurs minutes. C'est normal.

## Étape 3 — Créer le fichier `.env`

Le fichier `.env` contient les **mots de passe et les clés** de l'agent. Il n'est **pas** dans le code (il ne doit jamais être publié sur GitHub) : vous devez le créer vous-même.

### 3.1 Créer le fichier

Dans le dossier `ai-agent/`, créez un nouveau fichier texte nommé **exactement** `.env` (avec le point au début, sans autre extension).

- **Avec VS Code** : clic droit dans l'explorateur → *New File* → tapez `.env`.
- **Avec le Bloc-notes (Windows)** : *Fichier → Enregistrer sous*, choisissez le type **« Tous les fichiers »**, nom : `.env` (sinon Windows ajoute `.txt`).

### 3.2 Y coller ce modèle

```dotenv
DB_HOST=localhost
DB_PORT=5432
DB_NAME=nextact_ecom
DB_USER=agent_user
DB_PASS=<mot de passe n°1 de la documentation 1>
GROQ_API_KEY=<votre clé Groq, voir 3.4>
BACKEND_URL=http://localhost:8000/api
AGENT_SERVICE_EMAIL=<e-mail n°3 de la documentation 1>
AGENT_SERVICE_PASSWORD=<mot de passe n°4 de la documentation 1>
```

Remplacez **chaque valeur entre `< >`** (et supprimez les `< >`) par votre valeur.

### 3.3 Que mettre dans chaque champ ?

| Champ | Obligatoire | Que mettre | Où le trouver |
|---|:-:|---|---|
| `DB_HOST` | non (défaut `localhost`) | `localhost` si PostgreSQL est sur la même machine que l'agent | Sinon, l'adresse du serveur PostgreSQL |
| `DB_PORT` | non (défaut `5432`) | `5432` | Le port choisi à l'installation de PostgreSQL. Dans pgAdmin : clic droit sur le serveur → *Properties* → onglet *Connection* → *Port* |
| `DB_NAME` | **oui** | `nextact_ecom` | Le nom de la base créée dans la documentation 1 |
| `DB_USER` | **oui** | `agent_user` | ⚠️ **Toujours `agent_user`, jamais `postgres`** : ce rôle n'a que le droit de lire les vues |
| `DB_PASS` | **oui** | Le mot de passe du rôle `agent_user` | **Information n° 1** choisie à l'étape 0 de la documentation 1 |
| `GROQ_API_KEY` | **oui** | Votre clé d'API Groq (elle commence par `gsk_`) | Voir 3.4 ci-dessous |
| `BACKEND_URL` | non (défaut `http://localhost:8000/api`) | L'adresse du backend, **terminée par `/api`**, sans `/` final | `http://localhost:8000/api` si le backend tourne sur votre machine avec le port par défaut |
| `AGENT_SERVICE_EMAIL` | **oui** | L'e-mail du compte « AI agent » | **Information n° 3** (par défaut `agent@system.internal`) |
| `AGENT_SERVICE_PASSWORD` | **oui** | Le mot de passe du compte « AI agent » | **Information n° 4** |

**Règles d'écriture :**

- une variable par ligne, sous la forme `NOM=valeur` ;
- les guillemets autour des valeurs sont facultatifs ; **utilisez-les** si la valeur contient des espaces ou le signe `#` ;
- aucune ligne en trop, aucun espace avant le nom.

> 🎯 **Point d'attention n°2 — des valeurs identiques à celles de la base.** `DB_PASS`, `AGENT_SERVICE_EMAIL` et `AGENT_SERVICE_PASSWORD` doivent être **exactement** ceux choisis à l'étape 0 de la documentation 1 : le compte « AI agent » a été créé avec ces valeurs.

### 3.4 Obtenir la clé Groq (`GROQ_API_KEY`)

Groq est le service qui héberge le modèle de langage utilisé par l'agent (`qwen/qwen3.8-27b`). Il faut une clé personnelle pour l'utiliser.

1. Ouvrez **https://console.groq.com** dans votre navigateur.
2. Cliquez sur **Sign up** (ou **Sign in** si vous avez un compte) et connectez-vous avec votre e-mail ou votre compte Google. Vérifiez votre e-mail si Groq vous le demande.
3. Allez sur la page des clés : **https://console.groq.com/keys** (menu **API Keys**).
4. Cliquez sur **Create API Key**.
5. Donnez-lui un nom, par exemple `nextact-agent`, puis validez.
6. **Copiez immédiatement la clé** (elle commence par `gsk_`) : **Groq ne l'affiche qu'une seule fois**. Si vous la perdez, il faudra en créer une nouvelle.
7. Collez-la dans votre fichier `.env`, à la place de `<votre clé Groq, voir 3.4>`.

> 🔒 **Ne partagez jamais cette clé** (message, capture d'écran, GitHub). Si elle a été exposée, supprimez-la sur la page des clés et créez-en une nouvelle.
>
> ℹ️ L'utilisation de Groq est soumise aux **limites de votre compte** (nombre de requêtes et de jetons par minute). Consultez-les sur https://console.groq.com/docs/rate-limits, et vérifiez que le modèle `qwen/qwen3.8-27b` est disponible pour votre compte.

## Étape 4 — Tester la connexion à la base de données

Ce petit test vérifie que `DB_NAME`, `DB_USER` et `DB_PASS` sont corrects. Dans le terminal (environnement activé) :

```bash
cd src/db
python conn-test.py
```

✅ **Ce que vous devez voir** :

```
✅ Connexion réussie !
Version PostgreSQL : PostgreSQL ...
search_path actif : agent_reporting
```

Si vous voyez `❌ Échec de connexion`, consultez la section *Dépannage* en bas de page.

## Étape 5 — Lancer l'agent

Depuis le dossier **`src`** (important : c'est de là que Python trouve les dossiers `agent`, `tools` et `db`) :

```bash
cd ..          # si vous venez de l'étape 4 (vous étiez dans src/db)
python main.py
```

Au **premier lancement**, l'agent télécharge le petit modèle de sélection des outils (`multilingual-e5-small`) puis le charge : **le démarrage peut durer plusieurs minutes**. Les lancements suivants sont bien plus rapides. Le démarrage normal se termine par une ligne du type :

```
INFO:     Uvicorn running on http://127.0.0.1:8080 (Press CTRL+C to quit)
```

> 💡 Vous pouvez aussi le lancer avec : `uvicorn main:app --host 127.0.0.1 --port 8080`.

**Vérification** : ouvrez **http://127.0.0.1:8080/docs** dans votre navigateur. Vous devez voir la page interactive de l'agent, avec la route **`POST /chat`**.

Pour arrêter l'agent : **Ctrl + C** dans le terminal. **Laissez ce terminal ouvert** tant que vous utilisez l'application.

> 🎯 **Point d'attention n°3 — deux conditions pour démarrer.** L'agent s'arrête immédiatement si `GROQ_API_KEY` est absente du `.env`, ou si `DB_NAME`, `DB_USER` ou `DB_PASS` manquent. Ce sont les deux premières erreurs à vérifier.

## Étape 6 — Tester une vraie conversation

⚠️ Cette étape nécessite que **le backend soit lancé** (documentation 3) et que le compte administrateur existe (documentation 1).

### 6.1 Vérifier le compte de l'agent

Le backend doit accepter la connexion du compte « AI agent ». Sur la page du backend (`http://localhost:8000/docs`), testez la route **`POST /api/auth/login`** avec l'e-mail et le mot de passe du compte « AI agent » (informations n° 3 et n° 4). La réponse doit contenir un `access_token`.

### 6.2 Obtenir un jeton de connexion pour vous

Faites de même avec **votre compte administrateur** (informations n° 6 et n° 7 de la documentation 1) et **copiez la valeur de `access_token`**.

En ligne de commande (macOS, Linux ou Git Bash) :

```bash
curl -X POST http://localhost:8000/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"votre-email@exemple.com","password":"votre-mot-de-passe"}'
```

### 6.3 Envoyer une question

Sur **http://127.0.0.1:8080/docs** :

1. Cliquez sur **`POST /chat`**, puis **Try it out**.
2. Dans le champ **authorization**, écrivez : `Bearer ` (avec une espace) suivi de votre jeton.
3. Dans le corps de la requête, écrivez par exemple :

```json
{
  "message": "Quels produits sont en stock bas ?",
  "session_id": "test-1"
}
```

4. Cliquez sur **Execute**.

La réponse a cette forme (exemple) :

```json
{
  "reply": "Voici les produits sous leur seuil d'alerte : ...",
  "tool_calls": ["get_low_stock_products"],
  "raw_data": [ ... ],
  "generated_sql": null,
  "download_url": null
}
```

- `reply` : la réponse pour l'utilisateur ;
- `tool_calls` : les outils utilisés ;
- `raw_data` : les données brutes du dernier outil ;
- `generated_sql` : rempli seulement si le plan B (SQL généré) a été utilisé ;
- `download_url` : un lien si un fichier a été généré.

> 🎯 **Point d'attention n°4 — un `session_id` par conversation.** Gardez le même `session_id` pour poser des questions de suite (l'agent se souvient des 20 derniers messages). Changez-le pour démarrer une conversation neuve. Un `session_id` appartient au premier utilisateur qui l'utilise.

### 6.4 Questions à essayer (avec les données de démonstration)

| Question | Ce que vous devriez observer |
|---|---|
| *Quels produits sont en stock bas ?* | Outil `get_low_stock_products` |
| *Quels sont mes 5 meilleurs clients ?* | Outil `get_top_clients` |
| *Donne-moi les ventes de juillet 2026.* | Outil `get_sales_by_period` |
| *Télécharge-moi les ventes de juillet 2026 en Excel.* | Outil `generate_report_file`, avec un `download_url` |
| *Quelle est la capitale de la France ?* | Refus poli : la question sort du rôle d'assistant ERP |
| *Merci !* | Réponse fixe, sans outil |

Pour tester les **droits**, connectez-vous avec un compte d'un autre rôle (par exemple `user_clients`) et posez une question sur un autre domaine : l'agent répond « Selon tes permissions actuelles, tu n'as accès qu'aux clients… ». Les droits de chaque rôle sont détaillés dans `tools-et-schemas.md`.

### 6.5 Lire la console

Pendant que vous discutez, le terminal de l'agent affiche des informations de diagnostic : la question, les **scores** de chaque outil (✅ = outil proposé au modèle), et les outils réellement appelés. C'est très utile pour comprendre pourquoi un outil a été choisi (voir `architecture-agent.md`, §5).

---

## ✅ Check-list finale

- [ ] Python 3.11 ou plus, environnement `venv` activé, bibliothèques installées
- [ ] Fichier `.env` créé dans `ai-agent/` (à côté de `src/`), toutes les valeurs remplies
- [ ] Clé Groq créée et copiée dans `GROQ_API_KEY`
- [ ] `python conn-test.py` affiche « Connexion réussie »
- [ ] `python main.py` (depuis `src/`) démarre sans erreur ; `http://127.0.0.1:8080/docs` s'affiche
- [ ] *(une fois le backend lancé)* `POST /chat` répond à une question de test

**Suite :** Documentation 3/4 — Backend.

---

## Dépannage

| Message ou symptôme | Cause probable | Solution |
|---|---|---|
| `KeyError: 'GROQ_API_KEY'` au démarrage | `.env` introuvable, mal placé, ou sans `GROQ_API_KEY` | Vérifiez que `.env` est dans `ai-agent/` (à côté de `src/`) et contient `GROQ_API_KEY=...` |
| `EnvironmentError: Variables de connexion DB manquantes` | `DB_NAME`, `DB_USER` ou `DB_PASS` absent | Complétez le `.env` |
| `password authentication failed for user "agent_user"` | `DB_PASS` différent du mot de passe n° 1 | Corrigez le `.env`, ou changez le mot de passe dans PostgreSQL : `ALTER ROLE agent_user WITH PASSWORD '...';` |
| `connection refused` ou `Connect call failed ... 5432` | PostgreSQL arrêté, ou mauvais `DB_PORT` | Démarrez le service PostgreSQL, vérifiez le port |
| `database "..." does not exist` | `DB_NAME` incorrect | Utilisez `nextact_ecom` |
| `ModuleNotFoundError: No module named '...'` | Environnement non activé, ou bibliothèque absente | Activez `venv` (le `(venv)` doit s'afficher) puis relancez la commande `pip install` de l'étape 2 |
| `No module named 'agent'` (ou `tools`, `db`) | Lancement depuis le mauvais dossier | Lancez `python main.py` **depuis `src/`** |
| `Address already in use` / port 8080 occupé | Un autre programme (ou une ancienne instance de l'agent) utilise le port | Fermez-le, ou changez le port dans `main.py` (et dans la configuration du backend) |
| Démarrage très long, téléchargement affiché | Premier lancement : téléchargement du modèle de sélection | Patientez ; les lancements suivants sont rapides. Une connexion Internet est nécessaire la première fois |
| `/chat` répond `422` (*Field required*) | En-tête `authorization` manquant | Ajoutez `Bearer <jeton>` dans le champ `authorization` |
| `/chat` répond `401` — *Token manquant ou mal formé* | L'en-tête ne commence pas par `Bearer ` | Écrivez `Bearer` + une espace + le jeton |
| `/chat` répond `401` — *Token invalide ou expiré* ou *Compte désactivé* | Jeton périmé, ou compte inactif | Reconnectez-vous au backend pour obtenir un nouveau jeton |
| `/chat` répond `502` — *Backend injoignable* | Le backend n'est pas lancé, ou `BACKEND_URL` est incorrect | Lancez le backend (documentation 3) ; vérifiez `BACKEND_URL` (terminé par `/api`) |
| `/chat` répond `403` — *Cette session n'appartient pas à cet utilisateur* | Ce `session_id` a déjà été utilisé par un autre compte | Utilisez un autre `session_id` |
| `/chat` répond `500` ; la console cite Groq (clé invalide, limite atteinte, modèle introuvable) | Clé Groq erronée, quota dépassé, ou modèle indisponible pour le compte | Lisez le message dans le terminal ; vérifiez la clé et les limites sur la console Groq |
| Réponse « Erreur backend (...) » ou « Impossible de contacter le backend » | Le backend est arrêté, ou refuse le compte de l'agent | Vérifiez l'étape 6.1 (connexion du compte « AI agent ») et que ce compte a bien le rôle `ADMIN` (documentation 1) |
| Réponse « Désolé, je n'ai pas pu traiter complètement ta demande » | Plus de 20 étapes d'outils nécessaires | Reformulez la question de façon plus précise |
| Réponse « Selon tes permissions actuelles… » | La question sort du domaine du rôle de l'utilisateur | C'est le comportement attendu ; utilisez un compte du bon rôle |
| Réponse « Cette question ne fait pas partie de mon rôle d'assistant ERP » | Question hors sujet, ou jugée hors périmètre | Posez une question sur le stock, les ventes, les clients, les devis, les fournisseurs… |
