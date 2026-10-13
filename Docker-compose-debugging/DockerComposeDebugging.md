# Docker Compose debugging

## Exercice 1 : WordPress + MySQL

**Avant de commencer**
```bash
time="2026-10-12T17:43:10+02:00" level=warning msg="C:\\Users\\picau\\Documents\\Code\\Entrainements\\ALT\\BetaTest-Pack---Dev-builder\\Docker-compose-debugging\\Ex1\\docker-compose.yml: the attribute `version` is obsolete, it will be ignored, please remove it to avoid potential confusion"
[+] Running 5/6
 ✔ Network ex1_default         Created        0.2s 
 ✔ Volume ex1_wordpress_data   Created        0.1s 
 ✔ Volume ex1_mysql_data       Created        0.0s 
 - Container ex1-mysql-1       Starting       0.9s 
 ✔ Container ex1-phpmyadmin-1  Created        0.2s 
 ✔ Container ex1-wordpress-1   Created        0.3s 
Error response from daemon: ports are not available: exposing port TCP 0.0.0.0:3306 -> 127.0.0.1:0: listen tcp 0.0.0.0:3306: bind: Only one usage of each socket address (protocol/network address/port) is normally permitted.
```
- Le port 3306 est déjà pris en local.
- `version: '3.8'` est obsolète -> supprimé.

- [x] **Identifier pourquoi MySQL ne démarre pas**

Logs MySQL :
```bash
2026-10-12 15:46:00+00:00 [ERROR] [Entrypoint]: Database is uninitialized and password option is not specified
You need to specify one of the following as an environment variable:
- MYSQL_ROOT_PASSWORD
- MYSQL_ALLOW_EMPTY_PASSWORD
- MYSQL_RANDOM_ROOT_PASSWORD
```
-> Il manque `MYSQL_ROOT_PASSWORD`.

Corrections :
- Ajout de `MYSQL_ROOT_PASSWORD` et des variables d'environnement
- Ajout de `restart: unless-stopped`
- Image officielle `phpmyadmin:latest` au lieu de `phpmyadmin/phpmyadmin`
- Healthcheck sur la BDD :
  ```yml
  healthcheck:
    test: ["CMD", "mysqladmin", "ping", "-h", "127.0.0.1"]
    interval: 10s
    timeout: 5s
    retries: 3
  ```

```bash
$ docker compose up -d
[+] Running 6/6
 ✔ Network ex1_default         Created    0.2s 
 ✔ Volume ex1_wordpress_data   Created    0.0s 
 ✔ Volume ex1_mysql_data       Created    0.0s 
 ✔ Container ex1-mysql-1       Healthy    29.9s 
 ✔ Container ex1-wordpress-1   Started   30.3s 
 ✔ Container ex1-phpmyadmin-1  Started       
```

- [x] **Corriger les problèmes de connexion WordPress ↔ MySQL**

Deux causes : MySQL ne démarrait pas (root password manquant), et WordPress démarrait avant que MySQL soit prêt (`depends_on` sans condition). Corrigé avec le healthcheck + `condition: service_healthy`.

- [x] **Sécuriser la configuration (mots de passe, réseau)**
- Mots de passe en dur -> variables dans `.env`
- Suppression des identifiants de connexion automatique de phpMyAdmin pour sécuriser l'accès à la BDD
- Suppression des ports de MySQL : les autres services y accèdent via le réseau Docker
- Réseau dédié `ex1_app-network`. En prod, j'irais plus loin : réseau backend en `internal: true` pour la base, et WordPress/phpMyAdmin derrière un reverse proxy Nginx, sans ports exposés directement.

- [x] **Tester l'accès à WordPress (port 8080) et PhpMyAdmin (port 8081)**

Fonctionnels.

---

## Exercice 2 : Nextcloud + PostgreSQL

**Avant de commencer**

Suppression de `version` (obsolète, cf. Ex1).
```bash
[+] Running 6/6
 ✔ Network ex2_default        Created  0.2s 
 ✔ Volume ex2_nextcloud_data  Created  0.0s 
 ✔ Volume ex2_postgres_data   Created  0.0s 
 ✔ Container ex2-redis-1      Started  3.1s 
 ✔ Container ex2-postgres-1   Started  3.1s 
 ✔ Container ex2-nextcloud-1  Started   

NAME              IMAGE              COMMAND                  SERVICE     CREATED              STATUS              PORTS
ex2-nextcloud-1   nextcloud:latest   "/entrypoint.sh apac…"   nextcloud   About a minute ago   Up About a minute   0.0.0.0:8080->80/tcp, [::]:8080->80/tcp
ex2-postgres-1    postgres:13        "docker-entrypoint.s…"   postgres    About a minute ago   Up About a minute   0.0.0.0:5432->5432/tcp, [::]:5432->5432/tcp
ex2-redis-1       redis:alpine       "docker-entrypoint.s…"   redis       About a minute ago   Up About a minute   0.0.0.0:6379->6379/tcp, [::]:6379->6379/tcp
```

- [x] **Identifier pourquoi Nextcloud affiche "Internal Server Error"**

Connexion sur Nextcloud réussie sans erreur avec le compose d'origine : l'erreur annoncée n'est pas reproduite. Cause probable quand elle survient : Nextcloud démarre avant que Postgres soit prêt (pas de healthcheck). Lors de mon test, la copie des fichiers de Nextcloud au 1er lancement a laissé le temps à Postgres de démarrer, mais rien ne le garantit.

Autres problèmes identifiés :
- `postgres:13` : la dernière version sur Docker Hub est la 19, mais `postgres:latest` plante au démarrage -> fixé sur `postgres:17`, la config du volume reste la même
- Ports 5432 et 6379 exposés sur l'hôte inutilement -> retirés
- Mots de passe en dur -> variables dans `.env` + réseau dédié

- [x] **Intégrer Redis comme cache pour Nextcloud**
- `REDIS_ARGS` (trouvé sur redis.io) ignoré par l'image : `redis-cli ping` répondait `PONG`, donc pas de mot de passe -> passage par `command: redis-server --requirepass ...`, le ping répond maintenant `NOAUTH`.
- Ajout de `REDIS_HOST` et `REDIS_HOST_PASSWORD` côté Nextcloud.

- [x] **Mettre en place les health checks appropriés**
- Healthchecks Postgres + Redis, conditions `service_healthy` sur Nextcloud.
- Le premier healthcheck `pg_isready` générait `FATAL: role "root" does not exist` dans les logs (connexion avec l'utilisateur par défaut) -> ajout de `-U` et `-d`.
- Healthcheck Redis avec authentification + `grep PONG`, sinon il passait "healthy" même en recevant `NOAUTH`.

- [x] **Tester l'accès à Nextcloud et la création d'un utilisateur admin**

Fonctionnels.
```bash
# Nextcloud
==> Using Redis as PHP session handler...
Initializing nextcloud 35.0.1.1 ...

# Redis
1:M 12 Oct 2026 17:38:37.144 * Background saving terminated with success
```

---

## Exercice 3 : Mattermost + PostgreSQL

**Avant de commencer**

Suppression de `version` (obsolète, cf. Ex1) + Postgres en 17 (cf. Ex2).
```bash
[+] Running 7/7
 ✔ Network ex3_default            Created   0.6s 
 ✔ Volume ex3_mattermost_data     Created   0.0s 
 ✔ Volume ex3_mattermost_logs     Created   0.0s 
 ✔ Volume ex3_mattermost_plugins  Created   0.0s 
 ✔ Volume ex3_postgres_data       Created   0.0s 
 ✔ Container ex3-postgres-1       Started   7.8s 
 ✔ Container ex3-mattermost-1     Started    
```

- [x] **Identifier pourquoi la connexion à la base de données échoue**

Logs Mattermost :
```bash
{"timestamp":"2026-10-13 12:19:09.831 Z","level":"error","msg":"failed to initialize platform: cannot create store: error setting up connections: pq: SSL is not enabled on the server","caller":"commands/server.go:75"}

Error: failed to initialize platform: cannot create store: error setting up connections: pq: SSL is not enabled on the server
```
-> Mattermost tente une connexion SSL que le Postgres du conteneur ne propose pas.

- [x] **Vérifier le format de la chaîne de connexion Mattermost**

Ajout des paramètres à la datasource (trouvés sur la doc Mattermost) : `?sslmode=disable&connect_timeout=10`

- [x] **Sécuriser la configuration avec des variables d'environnement**
- Mots de passe en dur -> variables dans `.env`
- Healthcheck sur Postgres + condition `service_healthy` sur Mattermost
- Réseau dédié
- Ajout de `restart: unless-stopped`
- Volume pour la config (vu sur la doc Mattermost)

- [x] **Tester la création d'un compte utilisateur sur Mattermost**

Fonctionnel, inscription réalisée !

---

## Exercice 4 : Stack ELK

**Avant de commencer**

Suppression de `version` (obsolète, cf. Ex1).

Il semble y avoir plusieurs erreurs à trouver (dont la mémoire), je lance donc les services un par un pour isoler les problèmes.

**Elasticsearch** : démarre seul sans crash mémoire.
```bash
elasticsearch-1 | {"@timestamp":"2026-10-13T13:04:54.511Z", "log.level": "INFO", "message":"adding index template [logs] for index patterns [logs-*-*]", ...}
```
```json
{
  "name": "36a5735e59ad",
  "cluster_name": "docker-cluster",
  "version": { "number": "8.11.0", "build_type": "docker" },
  "tagline": "You Know, for Search"
}
```

**Kibana** : accessible sur http://localhost:5601/, pas d'erreur dans les logs.

**Logstash** : pas d'erreur dans les logs, API OK sur http://localhost:9600.
```json
{
  "version": "8.11.0",
  "http_address": "0.0.0.0:9600",
  "status": "green",
  "pipeline": { "workers": 12, "batch_size": 125, "batch_delay": 50 }
}
```

**Filebeat** : démarre puis s'arrête aussitôt.
```bash
Exiting: error loading config file: config file ("filebeat.yml") can only be writable by the owner but the permissions are "-rwxrwxrwx" (to fix the permissions use: 'chmod go-w /usr/share/filebeat/filebeat.yml')
```
Sous Windows, les fichiers montés apparaissent en `rwxrwxrwx` et le `chmod` proposé n'a pas d'effet (permissions NTFS). Solution trouvée sur le repo GitHub d'abes-esr :
```yml
user: root
command: ["-e", "--strict.perms=false"]
```
Compromis : cette vérification empêche qu'un autre utilisateur modifie la config. Elle est désactivée ici car les permissions ne sont pas corrigeables sous Windows.

Redémarrage complet : toujours pas de crash mémoire, mais 2,47 Go de RAM utilisés et un pic à 668 % de CPU au démarrage.

- [x] **Résoudre les problèmes de mémoire d'Elasticsearch**

Crash non reproduit sur ma machine (assez de RAM dispo). Je pense qu'il dépend de la mémoire allouée à Docker : sur un poste limité à 2 Go, ES se fait tuer par le système. Correction préventive : `mem_limit` sur ES, valeurs dans le `.env` avec les heap pour les adapter à la machine.
```yml
mem_limit: 2048m
```

- [x] **S'assurer que les services démarrent dans le bon ordre**

Healthchecks + `condition: service_healthy` : Elasticsearch healthy -> Kibana et Logstash démarrent, Logstash healthy -> Filebeat démarre.
```bash
[+] Running 6/6
 ✔ Network ex4_app-network        Created      0.2s 
 ✔ Volume ex4_elasticsearch_data  Created      0.0s 
 ✔ Container ex4-elasticsearch-1  Healthy      49.9s 
 ✔ Container ex4-kibana-1         Started      48.2s 
 ✔ Container ex4-logstash-1       Healthy      79.5s 
 ✔ Container ex4-filebeat-1       Started      78.4s
```

- [x] **Tester l'ingestion de logs et leur visualisation dans Kibana**

Test via Dev Tools : Elasticsearch indexe bien et Kibana affiche les données.
```json
POST test-manuel/_doc
{
  "@timestamp": "2026-10-13T19:15:00",
  "message": "hello depuis Kibana"
}
```
```json
{
  "_index": "test-manuel",
  "result": "created",
  "_shards": { "total": 2, "successful": 1, "failed": 0 }
}
```

- [x] **Optimiser les performances et la sécurité**
- Healthchecks et `restart: unless-stopped` sur tous les services
- Variables d'environnement : limites mémoire (adaptables selon la machine) et URLs utilisées dans plusieurs fichiers (`ES_HOST`, `LOGSTASH_HOST`), lues aussi dans `logstash.conf` et `filebeat.yml`
- Monitoring legacy désactivé dans `logstash.yml` (déprécié en 8.x, source de warnings) :
  ```yml
  # xpack.monitoring.elasticsearch.hosts: [ "http://elasticsearch:9200" ]
  ```
  ```bash
  [2026-10-13T16:37:01,172][WARN ][logstash.outputs.elasticsearchmonitoring][.monitoring-logstash] Detected a 6.x and above cluster: the `type` event field won't be used to determine the document _type {:es_version=>8}
  [2026-10-13T16:37:01,199][WARN ][logstash.javapipeline ][.monitoring-logstash] 'pipeline.ordered' is enabled and is likely less efficient, consider disabling if preserving event order is not necessary
  ```
- Retrait du socket Docker et de `/var/lib/docker/containers` dans Filebeat (non utilisés, et le socket donne un accès quasi root à l'hôte)
- Retrait des ports 9300 et 5000 (non utilisés)
- Accès à ES ouvert ici (`xpack.security.enabled=false`). En prod, on activerait la sécurité avec des identifiants dans le `.env` (`ELASTIC_PASSWORD`, `KIBANA_PASSWORD`).

---

## Exercice 5 : Kong Gateway + microservices

**Avant de commencer**

Suppression de `version` (obsolète, cf. Ex1) + Postgres en 17 (cf. Ex2).

Pas de tâches dans la consigne ("à vous de jouer"), je me suis fixé celles-ci :

- [x] **Faire démarrer Kong de façon fiable**

1er essai avec le compose d'origine : tous les conteneurs démarrent, mais `kong` et `kong-migration` s'arrêtent aussitôt.
```bash
# kong-migration
Error: [PostgreSQL error] failed to retrieve PostgreSQL server_version_num: connection refused

# kong
nginx: [error] init_by_lua error: /usr/local/share/lua/5.1/kong/cmd/utils/migrations.lua:16: Database needs bootstrapping or is older than Kong 1.0.
To start a new installation from scratch, run 'kong migrations bootstrap'.
```
-> La migration part avant que Postgres soit prêt (`connection refused`) et échoue. Kong démarre alors sur une base vide (`Database needs bootstrapping`) et s'arrête.

Corrections :
- Réseau dédié
- Healthcheck sur Postgres, `kong-migration` attend `service_healthy`, Kong attend `service_completed_successfully` (la migration est un conteneur ponctuel : elle fait son travail puis s'arrête, donc pas de `restart` dessus)
- Healthcheck `kong health` sur Kong, `restart: unless-stopped` sur les services permanents
- Variables `KONG_PG_*` qui réutilisent celles de Postgres, pour éviter des valeurs différentes

```bash
✔ Container ex5-kong-database-1 Healthy 8.0s
✔ Container ex5-kong-migration-1 Exited 11.4s
✔ Container ex5-kong-1 Started 11.7s

# kong-migration
58 migrations processed
58 executed
Database is up-to-date
```
`kong-migration` termine avec succès (`Exited (0)`), Kong tourne.

- [x] **Configurer les routes vers les 3 services**

Kong démarre, mais aucune route :
- http://localhost:8000/ : `no Route matched with those values`
- http://localhost:8001/services : `{"data":[],"next":null}`, aucun service déclaré

Les `nginx.conf` définissent ce que chaque service sait répondre, mais Kong a sa propre table de routage, et aucun fichier fourni ne la remplit.

Note sur le 502 annoncé dans l'énoncé : il arrive quand un service Kong pointe vers `localhost:3001`. Dans le conteneur Kong, `localhost` c'est Kong lui-même. Il faut utiliser le nom du service Docker et son port interne : `http://user-service:80`.

Corrections :
- Pour garder Postgres et versionner la config, ajout d'un service decK (outil officiel de Kong) qui synchronise un fichier `config/kong.yml` dans la base. Inspiré d'un article, sans reprendre le `restart: always` sur le configurateur (il se relancerait à l'infini), son YAML (indentation cassée sur 3 services), les images en `latest` et le secret en dur.
- `strip_path: false` sur les routes : par défaut Kong retire le chemin avant de transmettre, Nginx reçoit `/` et répond 404.
- Les `nginx.conf` ont une route `/health` : je l'utilise pour les healthchecks des services, et decK attend que tout soit healthy. Quand il termine, la stack est prête à répondre, pas seulement configurée.

```bash
 ✘ Container ex5-product-service-1  Error     33.7s 
 ✘ Container ex5-order-service-1    Error     34.2s 
 ✘ Container ex5-user-service-1     Error     33.7s 
 ✔ Container ex5-kong-1             Healthy   24.2s 
dependency failed to start: container ex5-user-service-1 is unhealthy
```
Healthcheck en échec, vu dans Docker Desktop -> Inspect -> State -> Health : `wget: can't connect to remote host: Connection refused`.
-> Sous Alpine, `localhost` se résout en IPv6 (`::1`) alors que Nginx n'écoute qu'en IPv4 (`listen 80;`) -> passage en `127.0.0.1`, via la variable `APP_URL` pour uniformiser les healthchecks et simplifier le passage en prod.

- [x] **Tester les 3 routes via la gateway**

Fonctionnel : `/api/users`, `/api/products` et `/api/orders` répondent via http://localhost:8000.

- [x] **Sécuriser (ports, API admin, secrets)**
- Mots de passe en dur -> variables dans `.env`
- Ports 3001-3003 retirés : les services ne sont joignables que par Kong, sinon on contourne la gateway
- Montages `./services/*/html` retirés (dossiers inexistant)
- Redis commenté : il n'est utilisé par aucun service. S'il servait (cache, rate limiting), il faudrait un mot de passe et un volume