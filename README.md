# Monitoring avec Prometheus, Grafana, Node Exporter et WordPress

## Structure du projet

```
monitoring-prometheus-grafana/
├── docker-compose.yaml                          # Orchestration des 7 services
├── prometheus/
│   └── prometheus.yml                           # Configuration de scraping Prometheus
├── grafana/
│   └── provisioning/
│       └── datasources/
│           └── datasource.yml                   # Auto-configuration de la datasource
├── mysql-exporter/
│   └── .my.cnf                                  # Credentials MySQL pour l'exporter
├── COMPTE_RENDU.md                              # Rapport de compte rendu du projet
└── README.md                                    # Ce fichier
```

## Acces aux services

| Service | URL | Identifiants |
|---------|-----|-------------|
| WordPress | http://localhost:8080 | (a configurer au premier lancement) |
| Prometheus | http://localhost:9090 | - |
| Grafana | http://localhost:3000 | admin / admin |
| Node Exporter | http://localhost:9100/metrics | - |
| MySQL Exporter | http://localhost:9104/metrics | - |
| cAdvisor | http://localhost:8081 | - |

## Commandes utiles

```bash
# Lancer le stack
docker compose up -d

# Voir les logs
docker compose logs -f

# Voir les logs d'un service specifique
docker compose logs -f wordpress

# Arreter le stack
docker compose down

# Arreter et supprimer les volumes (reset complet)
docker compose down -v
```

---

## Flux de donnees global

```
                                    ┌──────────────────┐
                                    │   Grafana :3000   │  <-- Visualisation
                                    └────────┬─────────┘
                                             │ PromQL
                                    ┌────────▼─────────┐
                                    │ Prometheus :9090  │  <-- Stockage & scraping
                                    └────────┬─────────┘
                          ┌──────────────────┼──────────────────┐
                          │                  │                  │
              ┌───────────▼──┐    ┌──────────▼───┐    ┌────────▼────────┐
              │ Node Exporter│    │   cAdvisor   │    │ MySQL Exporter  │
              │    :9100     │    │    :8081      │    │     :9104       │
              └───────┬──────┘    └──────┬───────┘    └────────┬────────┘
                      │                  │                     │
              Metriques systeme   Metriques conteneurs   Metriques BDD
              (CPU, RAM, disque)  (WordPress, MySQL...)  (queries, conn.)
```

---

## Explication partie par partie

### 1. docker-compose.yaml - L'orchestrateur (7 services)

#### Service `wordpress`

```yaml
image: wordpress:6.5-apache
ports:
  - "8080:80"
environment:
  WORDPRESS_DB_HOST: mysql:3306
  WORDPRESS_DB_NAME: wordpress
  WORDPRESS_DB_USER: wp_user
  WORDPRESS_DB_PASSWORD: wp_password
```

- Image officielle WordPress avec Apache integre.
- Expose sur le port **8080** de la machine hote.
- Se connecte a MySQL via le reseau Docker interne (`mysql:3306`).
- Les variables d'environnement configurent la connexion a la base de donnees.

#### Service `mysql`

```yaml
image: mysql:8.0
environment:
  MYSQL_DATABASE: wordpress
  MYSQL_USER: wp_user
  MYSQL_PASSWORD: wp_password
  MYSQL_ROOT_PASSWORD: root_password
```

- MySQL 8.0 comme base de donnees pour WordPress.
- Cree automatiquement la base `wordpress` et l'utilisateur `wp_user` au premier demarrage.
- Le volume `mysql_data` persiste les donnees entre les redemarrages.

#### Service `mysql-exporter`

```yaml
image: prom/mysqld-exporter:v0.15.1
volumes:
  - ./mysql-exporter/.my.cnf:/.my.cnf:ro
```

- Expose les metriques MySQL (connexions, requetes, latence, InnoDB) sur le port **9104**.
- Utilise un fichier `.my.cnf` monte en lecture seule pour l'authentification.

#### Service `cadvisor`

```yaml
image: gcr.io/cadvisor/cadvisor:v0.49.1
volumes:
  - /:/rootfs:ro
  - /var/run:/var/run:ro
  - /sys:/sys:ro
  - /var/lib/docker/:/var/lib/docker:ro
```

- **cAdvisor** (Container Advisor) collecte les metriques de **chaque conteneur** : CPU, memoire, reseau, I/O disque.
- Monte les repertoires systeme en lecture seule pour acceder aux cgroups et stats Docker.
- Expose sur le port **8081** (l'interface web native est aussi consultable).

#### Service `prometheus`

```yaml
image: prom/prometheus:v2.53.0
command:
  - "--storage.tsdb.retention.time=15d"
  - "--web.enable-lifecycle"
```

- Collecte et stocke toutes les metriques scrappees.
- Retention de 15 jours.
- `web.enable-lifecycle` : permet le rechargement a chaud de la config.

#### Service `grafana`

```yaml
image: grafana/grafana:11.0.0
environment:
  - GF_SECURITY_ADMIN_USER=admin
  - GF_SECURITY_ADMIN_PASSWORD=admin
```

- Plateforme de visualisation, connectee automatiquement a Prometheus via le provisioning.
- Login par defaut : `admin` / `admin`.

#### Service `node-exporter`

```yaml
image: prom/node-exporter:v1.8.1
```

- Collecte les metriques systeme du host : CPU, RAM, disque, reseau.

#### Volumes et Networks

- **4 volumes nommes** : `prometheus_data`, `grafana_data`, `wordpress_data`, `mysql_data` pour la persistance.
- **1 reseau bridge** `monitoring` : tous les services communiquent par nom de service.

---

### 2. prometheus/prometheus.yml - Les 4 jobs de scraping

| Job | Cible | Port | Metriques collectees |
|-----|-------|------|---------------------|
| `prometheus` | prometheus:9090 | 9090 | Metriques internes Prometheus |
| `node-exporter` | node-exporter:9100 | 9100 | CPU, RAM, disque, reseau du host |
| `cadvisor` | cadvisor:8080 | 8080 | CPU, RAM, reseau par conteneur |
| `mysql-exporter` | mysql-exporter:9104 | 9104 | Connexions, requetes, InnoDB MySQL |

---

### 3. grafana/provisioning/datasources/datasource.yml

Provisioning automatique de Prometheus comme datasource par defaut dans Grafana au demarrage.

---

### 4. mysql-exporter/.my.cnf

Fichier de credentials MySQL monte dans le conteneur mysqld-exporter. Contient l'utilisateur, le mot de passe et l'adresse du serveur MySQL.

---

## Utilisation de Grafana

### Premiers pas

1. Ouvrir http://localhost:3000
2. Se connecter avec `admin` / `admin`
3. Aller dans **Explore** pour tester des requetes PromQL

### Requetes PromQL utiles

#### Metriques systeme (Node Exporter)

| Requete | Description |
|---------|-------------|
| `up` | Voir quelles cibles sont actives (1 = up, 0 = down) |
| `node_cpu_seconds_total` | Temps CPU par mode |
| `rate(node_cpu_seconds_total{mode="idle"}[5m])` | Taux CPU idle sur 5 min |
| `node_memory_MemAvailable_bytes` | Memoire disponible |
| `node_filesystem_avail_bytes` | Espace disque disponible |

#### Metriques WordPress/conteneurs (cAdvisor)

| Requete | Description |
|---------|-------------|
| `container_cpu_usage_seconds_total{name="wordpress"}` | CPU utilise par WordPress |
| `container_memory_usage_bytes{name="wordpress"}` | RAM utilisee par WordPress |
| `rate(container_network_receive_bytes_total{name="wordpress"}[5m])` | Trafic reseau entrant WP |
| `container_memory_usage_bytes{name="mysql"}` | RAM utilisee par MySQL |

#### Metriques MySQL (MySQL Exporter)

| Requete | Description |
|---------|-------------|
| `mysql_global_status_connections` | Nombre total de connexions MySQL |
| `rate(mysql_global_status_queries[5m])` | Requetes par seconde |
| `mysql_global_status_threads_connected` | Threads actuellement connectes |
| `mysql_global_variables_max_connections` | Limite max de connexions |

### Dashboards pre-faits recommandes

| ID | Nom | Usage |
|----|-----|-------|
| `1860` | Node Exporter Full | Metriques systeme detaillees |
| `14282` | cAdvisor Exporter | Metriques des conteneurs Docker |
| `7362` | MySQL Overview | Metriques MySQL detaillees |

Pour importer : **Dashboards** > **Import** > entrer l'ID > selectionner **Prometheus** > **Import**
