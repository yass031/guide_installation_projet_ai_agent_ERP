# Configuration et lancement

## 1. Prérequis

- Python 3.10 (fichiers `.pyc` en `cpython-310`)
- PostgreSQL avec le schéma `agent_reporting` en place (voir `base-de-donnees.md`)
- Un compte SMTP (par exemple Gmail avec mot de passe d'application)

## 2. Installation

```bash
python -m venv venv
source venv/bin/activate            # Windows : venv\Scripts\activate
pip install -r requirements.txt
pip install pandas openpyxl reportlab
```

> `requirements.txt` est encodé en **UTF-16** : le convertir en UTF-8 si pip le refuse. `pandas`, `openpyxl` et `reportlab` sont utilisés par le code mais **absents** du fichier. `jose==1.0.0` y est listé : c'est un paquet différent de `python-jose` qui expose le même nom de module `jose` — à retirer.

## 3. Variables d'environnement

Le fichier `.env` se place à la **racine du projet** (chemin calculé depuis `core/config.py`, donc indépendant du dossier de lancement).

| Variable | Obligatoire | Défaut | Description |
|---|---|---|---|
| `DB_HOST` | Oui | — | Hôte PostgreSQL |
| `DB_PORT` | Oui | — | Port |
| `DB_NAME` | Oui | — | Nom de la base |
| `DB_USER` | Oui | — | Utilisateur |
| `DB_PASS` | Oui | — | Mot de passe |
| `SECRET_KEY` | Oui | — | Clé de signature des JWT |
| `ALGORITHM` | Non | `HS256` | Algorithme JWT |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | Oui | — | Durée de vie du token |
| `SMTP_HOST` | Oui | — | Serveur SMTP |
| `SMTP_PORT` | Non | `587` | Port SMTP (STARTTLS) |
| `SMTP_USER` | Oui | — | Compte d'envoi |
| `SMTP_PASSWORD` | Oui | — | Mot de passe SMTP |
| `SMTP_FROM_NAME` | Non | `Nextact Alertes` | Nom d'expéditeur |
| `ALERT_CHECK_INTERVAL_MINUTES` | Non | `60` | Fréquence du scan d'alertes |
| `AGENT_SERVICE_EMAIL` | Non | — | Email du compte de service de l'agent |
| `AGENT_API_URL` | Non | `http://localhost:8080` | URL de l'agent IA (lue via `os.environ`) |
| `REPORTS_EXPORT_DIR` | Non | `temp_exports` | Dossier des exports |

### Exemple de `.env` (valeurs fictives)

```env
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ma_base
DB_USER=backend_user
DB_PASS=********
SECRET_KEY=<chaine-aleatoire-longue>
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=60

SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=alertes@example.com
SMTP_PASSWORD=********
SMTP_FROM_NAME=Nextact Alertes
ALERT_CHECK_INTERVAL_MINUTES=60

AGENT_SERVICE_EMAIL=agent@system.internal
AGENT_API_URL=http://localhost:8080
```

> ⚠️ `AGENT_API_URL` est lue avec `os.environ` à l'import de `api/chat.py`, **avant** que `load_dotenv()` de `pw_email_service.py` ne s'exécute : définissez-la dans l'environnement du processus plutôt que dans le `.env`. Les clés `SMTP_*` apparaissent deux fois dans le `.env` d'origine (bloc alertes et bloc email de bienvenue) : c'est le même compte.

## 4. Lancement

Depuis la racine du projet (les imports sont de la forme `from api import …`) :

```bash
uvicorn main:app --reload --port 8000
```

| URL | Usage |
|---|---|
| `/docs` | Swagger UI |
| `/redoc` | ReDoc |

Au démarrage, le log `[Alertes] Scheduler démarré, intervalle = N min` confirme le lancement des jobs.

## 5. Connexion à la base

`core/database.py` :

```python
engine = create_engine(settings.database_url, pool_pre_ping=True)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
```

`database_url` est construite sous la forme `postgresql://user:pass@host:port/name`. `get_db()` fournit une session par requête, fermée en fin de traitement.

## 6. CORS

Seule l'origine `http://localhost:5173` est autorisée (méthodes et headers : tous ; credentials : oui). À rendre configurable pour la production.

## 7. Scripts utilitaires

| Script | Usage |
|---|---|
| `core/generate_hash.py` | Génère un hash bcrypt (création manuelle d'un compte) |
| `test.py` | Déclenche manuellement le job d'alertes |
| `services/test_smtp.py` | Teste l'envoi SMTP |
| `repositories/test.py` | Essais manuels de repository |
