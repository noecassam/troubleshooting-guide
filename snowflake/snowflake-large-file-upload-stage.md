# Uploader un fichier volumineux vers un stage Snowflake

Guide pour contourner la limite de taille de l'upload via Snowsight (interface web) et charger des fichiers de plusieurs centaines de Mo ou Go vers un stage interne.

## Contexte

L'upload via Snowsight est limité à 250 Mo par fichier. Pour les fichiers plus volumineux (datasets ML, exports SAP, archives), il faut passer par le client Python (Snowpark ou connecteur), qui :

- gère la reprise en cas d'erreur réseau
- ne tape pas la limite de l'UI

Aucun export, aucun transit par un bucket S3 intermédiaire : tout va directement vers le stage interne.

## Prérequis

```bash
pip install snowflake-snowpark-python
```

Et avoir :

- un accès SSO actif au compte Snowflake
- les droits `WRITE` sur le stage cible
- un warehouse démarré (ou en auto-resume)

## Étape 1 : récupérer ton `account identifier`

Dans Snowsight, en bas à gauche, clique sur le nom du compte, Account > View account details puis sur "Copy account identifier". Le format attendu est `<orgname>-<account_name>`, par exemple :

```
URGO-URGO_MEDICAL
```

Évite le locator legacy (type `JQ40854`), moins fiable avec `externalbrowser`.

## Étape 2 : récupérer ton `LOGIN_NAME`

Le `user` à passer dans la connexion est le `LOGIN_NAME` du user Snowflake, qui doit matcher l'identité renvoyée par l'IdP (généralement ton email pro).

Dans Snowsight :

```sql
DESC USER <ton_user_snowflake>;
```
(tu peux obtenir ton_user_snowflake en lançant dans Snwosight SELECT CURRENT_USER();)
Note la valeur exacte (ex. `N.CASSAM-CHENAI@FR.URGO.COM`).

## Étape 3 : script d'upload

```python
from snowflake.snowpark import Session

connection_parameters = {
    "account": "URGO-URGO_MEDICAL",
    "user": "N.CASSAM-CHENAI@FR.URGO.COM",
    "authenticator": "externalbrowser",
    "role": "SBX_NOE_ANALYST",
    "warehouse": "URGO_MEDICAL_QUERY_WH",
    "database": "SANDBOX",
    "schema": "SBX_NOE",
}

session = Session.builder.configs(connection_parameters).create()

session.file.put(
    "C:/Users/<toi>/Downloads/mon_fichier.zip",
    "@STG_RAW_DATASETS",
    auto_compress=False,   # mettre True si fichier non déjà compressé
    overwrite=True,
    parallel=4,            # 4 threads d'upload, ajuster selon la connexion
)
```

Une fenêtre de navigateur s'ouvre pour l'authentification SSO, puis l'upload démarre. Il faut attendre que le terminal ait fini, et ça devrait être bon.

## Étape 4 : vérification

```sql
LIST @STG_RAW_DATASETS;
```

Tu dois voir ton fichier avec sa taille en octets. 

## Bonnes pratiques

- **Compression** : si ton fichier est déjà compressé (`.zip`, `.gz`, `.parquet`), passe `auto_compress=False`. Sinon laisse `True`, Snowflake gzip-era automatiquement.
- **Parallel** : `parallel=4` à `8` accélère sensiblement sur les connexions rapides.
- **Stage temporaire** : pour un usage one-shot, crée un stage temporaire (`CREATE TEMPORARY STAGE`) qui se nettoie en fin de session.

## Pour aller plus loin

- Doc officielle : [PUT command](https://docs.snowflake.com/en/sql-reference/sql/put)
