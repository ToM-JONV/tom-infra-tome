# CLAUDE.md - Infrastructure ToM (Training of Managers)

## Vue d'ensemble

Infrastructure Kubernetes multi-composants pour la plateforme LumApps Learning (ToM).
L'architecture est organisée en clusters/groupes de services, avec des services externes, des composants par cluster (1/cluster) et des composants uniques (1 worldwide).

---

## Architecture applicative (flux entre services)

### ToM's APIs (coeur de la plateforme)

Les trois services principaux forment le coeur de la plateforme :

- **Userdata** : gestion des donnees utilisateurs et sessions
- **Centaurus** : backend principal (API, logique metier)
- **Mission Center** : interface d'administration (acces Admin via navigateur)

Userdata et Centaurus communiquent de maniere bidirectionnelle.

### Flux de donnees

```
Admin (navigateur) ──> Mission Center ──> Userdata <──> Centaurus
Learner (mobile)   ──────────────────────────────────> Centaurus

Centaurus ──> ToM Connect ──> Integrations (externes)
Centaurus <──> S3 (content)     -- stockage des contenus
Centaurus <──> S3 (analytics)   -- donnees analytiques
Centaurus ──> Curiosity         -- moteur de recherche/recommandation
Centaurus <──> Video transcoding -- transcodage video (service externe)

S3 (analytics) ──> Matillion ──> Snowflake ──> GoodData  -- pipeline BI
Curiosity <──> S3 (content)
```

### Legendes des composants

| Couleur | Type |
|---------|------|
| Jaune | ToM Software (developpe en interne) |
| Bleu | AWS Service |
| Rouge/Rose | Service externe |
| Vert | Composant client externe |

---

## Architecture infrastructure (AWS / Kubernetes)

### Hebergement

La plateforme est hebergee sur **AWS** avec l'architecture suivante :

```
Utilisateurs (navigateur / mobile)
       |
     HTTPS
       |
  Security Group
       |
  Load Balancer
       |
  Virtual Servers (EC2)
       |
  Kubernetes Cluster
       |
  ┌────────────────────────────────────┐
  │  MySQL  NGINX  .NET  Redis  sFTP   │
  │  PgSQL  PHP    ...                 │
  │                                    │
  │  Server 1  ...  Server n           │
  └────────────────────────────────────┘
```

### Composants infrastructure

- **Kubernetes** : orchestrateur de conteneurs, heberge tous les services applicatifs
- **Load Balancer** : repartition de charge entre les serveurs virtuels
- **Security Group** : pare-feu reseau (filtrage des acces entrants)
- **AWS S3** : stockage objet (contenus et analytics)
- **ADFS** (cote client) : federation d'identite pour l'authentification SSO (HTTPS)

### Services dans le cluster Kubernetes

| Service | Role |
|---------|------|
| MySQL | Base de donnees relationnelle (Gen2, legacy) |
| PgSQL | Base de donnees PostgreSQL (Centaurus, Connect, Curiosity, Addons) |
| Redis | Cache et sessions |
| NGINX | Reverse proxy / serveur web |
| PHP | Runtime PHP-FPM (MC/Udata, Gen2) |
| .NET | Services .NET (Centaurus) |
| sFTP | Transfert de fichiers securise |

### Connexions externes

| Destination | Protocole | Usage |
|-------------|-----------|-------|
| AWS S3 | HTTPS | Stockage de contenus et analytics |
| AWS SES | SMTP/TLS | Envoi d'emails transactionnels |
| Snowflake | HTTPS | Data warehouse (BI) |
| GoodData | HTTPS | Dashboards analytics |
| Freecaster | HTTPS | Streaming video |
| ADFS (client) | HTTPS | Authentification SSO |

---

## Composants Kubernetes detailles (par cluster)

### register
Gestion des instances clients (multi-tenant).
- `client1`, `client2`, `...` : instances clients isolees

### io
Couche d'entree/sortie fichiers.
- `sftp` : transfert de fichiers securise
- `nginx` : reverse proxy / serveur web
- `minio` : stockage objet compatible S3

### connect
Service de connexion / gateway.
- `gw` : gateway (point d'entree API)
- `wkr` : worker (traitement asynchrone)
- `pgsql` : base de donnees PostgreSQL

### Centaurus
Service backend principal (API + workers).
- `pgsql` : base PostgreSQL
- `redis x2` : cache / sessions (2 instances)
- `api` : serveur API
- `wkr x7` : workers (7 instances pour le traitement parallele)

### curiosity
Service d'analytics / recherche.
- `gw` : gateway
- `wkr` : worker
- `redis` : cache
- `neo4j` : base de donnees graphe
- `pgsql` : base PostgreSQL
- `elastic` : Elasticsearch (moteur de recherche)

### mods
Modules complementaires.
- `excel-expr` : export/import Excel
- `mgr-dashb` : dashboard manager

### Gen2 prod + stag
Service de production et staging de generation 2 (MC/Udata).
- `redis` : cache
- `schdr` : scheduler (taches planifiees)
- `php-fpm` : runtime PHP
- `wkr` : worker
- `mysql` : base de donnees MySQL
- `whk` : webhooks
- `run fixes` : execution de correctifs/migrations

### addons
Extensions et modules additionnels.
- `Build AOE` : build des add-ons
- `pgsql` : base PostgreSQL
- `php-fpm` : runtime PHP
- `wkr` : worker

### dump
Sauvegarde et export de donnees.
- `mysql x3` : dumps MySQL (3 instances)
- `pg x3` : dumps PostgreSQL (3 instances)
- `rclone` : synchronisation vers stockage distant

### tick
Stack de monitoring (metriques).
- `influxdb` : base de donnees de metriques time-series
- `telegraf` : collecteur de metriques

### drone
CI/CD et synchronisation.
- `runner` : execution des pipelines CI
- `listener` : ecoute des evenements
- `sync mgr-dashb` : synchronisation dashboard manager
- `sync isk` : synchronisation ISK
- `synk whk x2` : synchronisation webhooks (2 instances)
- `sync register` : synchronisation du registre

---

## Composants uniques (1 worldwide)

### kube
Orchestration Kubernetes globale.

### bastion
Serveur bastion pour l'acces securise a l'infrastructure.

### s3s
Stockage objet et services associes.
- `mcstats` : statistiques Mission Center
- `ToM IS` : ToM Information System
- `chronolog` : logs chronologiques
- `grafana` : dashboards de monitoring
- `gitlab` : gestion de code source
- `gitlab CI` : integration continue GitLab

### sysop
Operations systeme (maintenance).
- `schdr` : scheduler
- `redis` : cache
- `php-fpm` : runtime PHP
- `Logrotate` : rotation des logs
- `MC/udata` : services Mission Center et Userdata

### Jenkins
Pipeline CI/CD (builds, deploiements).

### Matillion
ETL (Extract, Transform, Load) pour les donnees analytics.

---

## Services externes

| Service | Usage |
|---------|-------|
| **GoodData** | Business Intelligence / Analytics |
| **Snowflake** | Data Warehouse |
| **intruder.io** | Scan de securite / vulnerabilites |
| **logz.io** | Centralisation de logs (ELK manage) |
| **Freecaster** | Streaming video |
| **Cloudflare** | CDN / DNS / protection DDoS |
| **OVH Horizon** | Cloud hosting (OpenStack) |
| **AWS SES** | Envoi d'emails transactionnels |

---

## Bases de donnees

| Type | Services |
|------|----------|
| **MySQL** | Gen2 prod+stag, dump (x3) |
| **PostgreSQL** | connect, Centaurus, curiosity, addons, dump (x3) |
| **Redis** | Centaurus (x2), curiosity, Gen2 prod+stag, sysop |
| **Neo4j** | curiosity (graphe de relations) |
| **Elasticsearch** | curiosity (recherche full-text) |
| **InfluxDB** | tick (metriques time-series) |
| **Snowflake** | Data warehouse (externe) |

---

## Monitoring & Observabilite

- **Grafana** (s3s) : dashboards de visualisation
- **InfluxDB + Telegraf** (tick) : collecte et stockage de metriques
- **logz.io** (externe) : centralisation des logs
- **GoodData** (externe) : analytics business

---

## CI/CD

- **Drone** : pipelines CI/CD par cluster (runners, listeners, syncs)
- **Jenkins** : pipeline CI/CD worldwide
- **GitLab CI** (s3s) : integration continue liee au code source

---

## Securite

- **Bastion** : point d'entree unique pour l'acces SSH a l'infrastructure
- **Cloudflare** : protection DDoS, WAF, DNS
- **intruder.io** : scans de vulnerabilites automatises
- **Security Group** : pare-feu reseau AWS (filtrage entrant)
- **ADFS** : federation d'identite (SSO client)
- **SFTP** (io) : transferts fichiers securises
