# Compte Rendu - Exercice Monitoring DevSecOps

**Cours** : DevSecOps - LiveCampus Bac+5
**Date** : 31 mars 2026
**Exercice** : Mise en place d'une stack de monitoring avec Docker Compose

---

## 1. Objectif de l'exercice

Deployer une infrastructure de monitoring complete a l'aide de Docker Compose permettant de :
- Heberger une application web WordPress comme cible a monitorer
- Collecter des metriques systeme, conteneur et base de donnees
- Stocker les metriques dans une base time-series
- Visualiser les metriques via des dashboards

---

## 2. Architecture deployee

### 2.1 Vue d'ensemble

L'infrastructure est composee de **7 services** containerises, communiquant sur un reseau Docker bridge dedie (`monitoring`) :

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Reseau Docker: monitoring                    │
│                                                                     │
│  ┌────────────┐  ┌────────────┐                                    │
│  │ WordPress  │  │   MySQL    │  <-- Application a monitorer       │
│  │   :8080    │  │   :3306    │                                     │
│  └────────────┘  └────────────┘                                    │
│                                                                     │
│  ┌────────────┐  ┌────────────┐  ┌────────────┐                   │
│  │   Node     │  │  cAdvisor  │  │   MySQL    │  <-- Exporters    │
│  │  Exporter  │  │   :8081    │  │  Exporter  │                    │
│  │   :9100    │  │            │  │   :9104    │                    │
│  └─────┬──────┘  └─────┬──────┘  └─────┬──────┘                   │
│        │               │               │                            │
│        └───────────────┼───────────────┘                            │
│                        ▼                                            │
│               ┌────────────────┐                                   │
│               │   Prometheus   │  <-- Collecte & stockage          │
│               │     :9090      │                                    │
│               └───────┬────────┘                                   │
│                       ▼                                            │
│               ┌────────────────┐                                   │
│               │    Grafana     │  <-- Visualisation                 │
│               │     :3000      │                                    │
│               └────────────────┘                                   │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 Tableau des services

| Service | Image | Port | Role |
|---------|-------|------|------|
| WordPress | `wordpress:6.5-apache` | 8080 | Application web cible |
| MySQL | `mysql:8.0` | 3306 (interne) | Base de donnees WordPress |
| Node Exporter | `prom/node-exporter:v1.8.1` | 9100 | Metriques systeme (CPU, RAM, disque) |
| cAdvisor | `gcr.io/cadvisor/cadvisor:v0.49.1` | 8081 | Metriques par conteneur |
| MySQL Exporter | `prom/mysqld-exporter:v0.15.1` | 9104 | Metriques base de donnees |
| Prometheus | `prom/prometheus:v2.53.0` | 9090 | Collecte et stockage des metriques |
| Grafana | `grafana/grafana:11.0.0` | 3000 | Visualisation et dashboards |

---

## 3. Technologies utilisees

### 3.1 Prometheus

**Role** : Systeme de monitoring et d'alerting open-source.

Prometheus fonctionne sur un modele **pull** : il va activement chercher (scraper) les metriques sur les cibles a intervalles reguliers (ici 15 secondes). Les donnees sont stockees dans une base de donnees time-series (TSDB) locale avec une retention de 15 jours.

**Points cles** :
- Modele pull (contrairement a des outils push comme Datadog)
- Langage de requete PromQL pour interroger les metriques
- Decouverte de cibles via la configuration statique ou dynamique
- Rechargement a chaud de la configuration via l'API lifecycle

### 3.2 Grafana

**Role** : Plateforme de visualisation et de dashboarding.

Grafana se connecte a Prometheus comme datasource et permet de creer des dashboards interactifs. Dans ce projet, la datasource est **provisionnee automatiquement** via un fichier YAML au demarrage, evitant toute configuration manuelle.

**Points cles** :
- Provisioning as code (datasources configurees via YAML)
- Dashboards pre-faits importables depuis grafana.com
- Alerting integre (non configure dans cet exercice)

### 3.3 Node Exporter

**Role** : Collecteur de metriques systeme pour les machines Linux.

Il expose les metriques hardware et OS : utilisation CPU, memoire, espace disque, trafic reseau, etc. Les volumes `/proc`, `/sys` et `/` sont montes en lecture seule dans le conteneur pour acceder aux donnees du systeme hote.

### 3.4 cAdvisor (Container Advisor)

**Role** : Collecteur de metriques pour les conteneurs Docker.

Developpe par Google, cAdvisor fournit des metriques par conteneur : CPU, memoire, reseau et I/O disque. C'est l'outil qui permet de monitorer specifiquement le conteneur WordPress et MySQL.

### 3.5 MySQL Exporter

**Role** : Exporteur de metriques MySQL pour Prometheus.

Il expose les metriques internes de MySQL : nombre de connexions, requetes par seconde, threads actifs, metriques InnoDB, etc. L'authentification se fait via un fichier `.my.cnf` monte dans le conteneur.

### 3.6 WordPress + MySQL

**Role** : Application cible du monitoring.

WordPress est l'application web que l'on cherche a monitorer. Elle est accompagnee de sa base de donnees MySQL. Le monitoring s'effectue a trois niveaux :
- **Conteneur** : via cAdvisor (CPU, RAM, reseau du conteneur WordPress)
- **Base de donnees** : via MySQL Exporter (sante de MySQL)
- **Systeme** : via Node Exporter (ressources de la machine hote)

---

## 4. Configuration detaillee

### 4.1 Fichier docker-compose.yaml

Le fichier orchestre les 7 services. Points importants :

- **depends_on** : definit l'ordre de demarrage (MySQL avant WordPress, exporters avant Prometheus, Prometheus avant Grafana)
- **volumes nommes** : 4 volumes (`prometheus_data`, `grafana_data`, `wordpress_data`, `mysql_data`) pour la persistance des donnees
- **reseau bridge** : un seul reseau `monitoring` permet la communication inter-conteneurs par nom de service
- **restart: unless-stopped** : les conteneurs redemarrent automatiquement sauf arret explicite

### 4.2 Fichier prometheus.yml

Definit 4 jobs de scraping :

| Job | Cible scrappee | Frequence | Metriques |
|-----|---------------|-----------|-----------|
| prometheus | prometheus:9090 | 15s | Metriques internes Prometheus |
| node-exporter | node-exporter:9100 | 15s | Metriques systeme |
| cadvisor | cadvisor:8080 | 15s | Metriques conteneurs |
| mysql-exporter | mysql-exporter:9104 | 15s | Metriques MySQL |

### 4.3 Provisioning Grafana

Le fichier `datasource.yml` configure automatiquement la datasource Prometheus dans Grafana :
- **access: proxy** : les requetes vers Prometheus sont faites cote serveur Grafana
- **isDefault: true** : Prometheus est la datasource par defaut

---

## 5. Resultats des tests

### 5.1 Etat des conteneurs

Tous les 7 conteneurs demarrent et restent en etat **Up** :

```
NAMES            STATUS
wordpress        Up
mysql            Up
node-exporter    Up
cadvisor         Up (healthy)
mysql-exporter   Up
prometheus       Up
grafana          Up
```

### 5.2 Sante des targets Prometheus

Les 4 targets sont en etat **UP** dans Prometheus :

```
Job              Health    Cible
prometheus       up        prometheus:9090
node-exporter    up        node-exporter:9100
cadvisor         up        cadvisor:8080
mysql-exporter   up        mysql-exporter:9104
```

### 5.3 Endpoints verifies

| Endpoint | Resultat |
|----------|----------|
| `http://localhost:9090/-/healthy` | "Prometheus Server is Healthy." |
| `http://localhost:3000/api/health` | `{"database": "ok", "version": "11.0.0"}` |
| `http://localhost:9100/metrics` | Metriques Node Exporter exposees |
| `http://localhost:9104/metrics` | Metriques MySQL exposees |
| `http://localhost:8080` | Page d'installation WordPress |
| `http://localhost:8081` | Interface cAdvisor |

### 5.4 Datasource Grafana

La datasource Prometheus est automatiquement provisionnee et fonctionnelle dans Grafana.

---

## 6. Problemes rencontres et resolutions

### 6.1 MySQL Exporter - Erreur d'authentification

**Probleme** : Le MySQL Exporter v0.15.1 ne supporte plus la variable d'environnement `DATA_SOURCE_NAME` pour l'authentification. Le conteneur redemarrait en boucle avec l'erreur :
```
failed to validate config: no user specified in section or parent
```

**Resolution** : Creation d'un fichier `.my.cnf` contenant les credentials MySQL, monte en volume dans le conteneur :
```ini
[client]
user=root
password=root_password
host=mysql
port=3306
```

### 6.2 Attribut `version` obsolete

**Probleme** : Docker Compose affiche un warning pour l'attribut `version: "3.8"`.

**Resolution** : L'attribut a ete supprime dans la version finale. Les versions recentes de Docker Compose n'en ont plus besoin.

---

## 7. Lien avec le DevSecOps

Cet exercice s'inscrit dans la demarche DevSecOps a plusieurs niveaux :

### Observabilite (pilier Ops)
- Monitoring proactif des applications et de l'infrastructure
- Detection d'anomalies via les metriques (pics CPU, fuite memoire, surcharge DB)
- Dashboards pour la visibilite operationnelle

### Infrastructure as Code (pilier Dev)
- Toute l'infrastructure est definie dans des fichiers YAML versionnables
- Reproductibilite : `docker compose up -d` deploie l'integalite du stack
- Configuration as code : provisioning Grafana automatise

### Securite (pilier Sec)
- Les volumes sont montes en lecture seule (`:ro`) quand possible
- Le reseau bridge isole les services du reste de l'environnement
- Les credentials sont separees dans des fichiers dedies (`.my.cnf`)
- L'inscription publique Grafana est desactivee

---

## 8. Pour aller plus loin

- **Alerting** : Configurer des regles d'alerte dans Prometheus (ex: alerte si CPU > 80%)
- **Alertmanager** : Ajouter Prometheus Alertmanager pour envoyer des notifications (Slack, email)
- **Loki** : Ajouter Grafana Loki pour la collecte et visualisation des logs
- **Reverse proxy** : Ajouter Traefik ou Nginx comme reverse proxy devant WordPress
- **TLS** : Securiser les communications avec des certificats SSL
- **Authentification** : Remplacer les credentials en dur par Docker Secrets ou un gestionnaire de secrets
