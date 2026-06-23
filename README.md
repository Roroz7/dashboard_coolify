# 📊 Monitoring Stack — VPS OVH + Coolify

Stack de monitoring complet pour VPS OVH tournant sous Coolify.  
Déployé via **GitHub → Coolify** (les fichiers de config sont clonés directement sur le VPS).

## 🧱 Services

| Service | Port hôte | Port interne | Rôle |
|---|---|---|---|
| **Grafana** | **3001** | 3001 | Dashboards & visualisation |
| **Prometheus** | **19090** | 9090 | Base de données métriques |
| **Node Exporter** | **19100** | 9100 | Métriques système |
| **cAdvisor** | **18080** | 8080 | Métriques conteneurs Docker |
| **Alertmanager** | **19093** | 9093 | Alertes Discord |

## 📁 Structure

```
monitoring-stack/
├── docker-compose.yml
├── .env.example              ← copier en .env sur le VPS (jamais commit)
├── .gitignore
├── prometheus/
│   ├── prometheus.yml        ← config scraping
│   ├── alerts.yml            ← règles d'alertes
│   └── alertmanager.yml      ← routing alertes Discord
└── grafana/
    ├── provisioning/
    │   ├── datasources/prometheus.yml
    │   └── dashboards/dashboards.yml
    └── dashboards/
        └── vps-ovh.json      ← dashboard auto-provisionné
```

---

## 🚀 Déploiement GitHub → Coolify (étape par étape)

### 1. Préparer le repo GitHub

```bash
# Sur ta machine locale — dézippe l'archive téléchargée, puis :
cd monitoring-stack
git init
git add .
git commit -m "feat: initial monitoring stack"

# Crée un repo PRIVÉ sur github.com, puis :
git remote add origin git@github.com:TON_USERNAME/monitoring-stack.git
git push -u origin main
```

### 2. Connecter GitHub à Coolify

Dans Coolify → **Settings** → **Source** → **GitHub App** :
- Clique **Install GitHub App**
- Autorise l'accès à ton repo `monitoring-stack`
- Coolify récupère une clé de déploiement automatiquement

### 3. Créer le service dans Coolify

1. **New Resource** → **Docker Compose**
2. Source : **GitHub** → sélectionne ton repo `monitoring-stack`
3. Branche : `main`
4. Docker Compose file : `docker-compose.yml` (détecté auto)
5. Clique **Continue**

### 4. Configurer les variables d'environnement

Dans l'onglet **Environment** du service Coolify, ajoute :

```
GRAFANA_USER=admin
GRAFANA_PASSWORD=un_mot_de_passe_solide
GRAFANA_DOMAIN=monitoring.ton-domaine.com
DISCORD_WEBHOOK_URL=https://discord.com/api/webhooks/XXX/YYY
```

> ⚠️ Ces variables remplacent le `.env` — ne mets jamais `.env` dans Git.

### 5. Configurer le domaine Grafana

Dans l'onglet **Domains** :
- Ajoute `monitoring.ton-domaine.com`
- Port : `3001`
- Active HTTPS (Let's Encrypt automatique)

### 6. Déployer

Clique **Deploy** → Coolify clone le repo sur le VPS et lance les conteneurs.

---

## 🔄 Mettre à jour (workflow normal)

```bash
# Modifie un fichier, ex: ajouter une alerte
nano prometheus/alerts.yml

git add .
git commit -m "feat: nouvelle alerte disque"
git push
```

Puis dans Coolify → clique **Redeploy** (ou active le webhook GitHub pour auto-deploy).

### Activer le déploiement automatique (recommandé)

Dans Coolify → ton service → **Settings** → **Auto Deploy** → ON  
Coolify installe un webhook GitHub qui redéploie automatiquement à chaque `git push`.

---

## ⚙️ Commandes utiles sur le VPS

```bash
# Voir les logs en direct
docker compose logs -f grafana
docker compose logs -f prometheus

# Recharger la config Prometheus sans restart
curl -X POST http://localhost:19090/-/reload

# Vérifier les targets (métriques bien collectées ?)
curl http://localhost:19090/targets

# Voir les alertes actives
curl http://localhost:19093/api/v1/alerts

# Mettre à jour les images manuellement
docker compose pull && docker compose up -d
```

---

## 📱 Configurer les alertes Discord

1. Discord → ton serveur → **Paramètres du salon** → **Intégrations** → **Webhooks**
2. Crée un webhook, copie l'URL
3. Colle-la dans `DISCORD_WEBHOOK_URL` dans Coolify (onglet Environment)

Alertes configurées :
- 🟡 **Warning** : CPU > 85%, RAM > 85%, Disque > 80%, Load élevé
- 🔴 **Critique** : CPU > 95%, RAM > 95%, Disque > 90%, VPS DOWN

---

## 🔒 Sécurité

- **Ne jamais exposer** Prometheus (19090) et Node Exporter (19100) sur internet
- Seul **Grafana (3001)** est exposé via Coolify/Traefik avec TLS
- Le `.env` est dans `.gitignore` — les secrets passent par les env vars Coolify
- Données Prometheus conservées **30 jours** (modifiable dans `docker-compose.yml`)
