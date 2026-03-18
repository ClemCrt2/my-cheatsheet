
          # Analyse du Problème MDS - Capability Release Timeout

## 🔴 Diagnostic de la Situation

### Problème Identifié
Vous faites face à un problème classique mais critique de **"slow client"** ou **"unresponsive client"** dans CephFS. Le daemon MDS (Metadata Server) `mds.0` sur le nœud `node-05` ne parvient pas à récupérer les **capabilities** (permissions de cache) auprès de plusieurs clients.

### Explication Technique

Les **capabilities** dans CephFS sont des jetons de permission que le MDS accorde aux clients pour :
- Mettre en cache les métadonnées localement
- Effectuer des opérations en lecture/écriture
- Optimiser les performances en réduisant les round-trips

Quand le MDS a besoin de :
- Révoquer des permissions (changement de cohérence)
- Redistribuer des métadonnées
- Effectuer un failover ou un rebalancing

Il envoie des **capability release requests** aux clients. Si ces derniers ne répondent pas dans le délai imparti, vous obtenez ces messages d'erreur.

---

## 🔍 Causes Probables (par ordre de probabilité)

### 1. **Clients Surchargés ou en Détresse Réseau**
- Charge CPU/mémoire élevée sur les clients
- Latence réseau excessive
- Perte de paquets réseau
- Clients en swap intensif

### 2. **Configuration MDS Inadaptée**
- Timeouts trop courts pour votre environnement
- Cache MDS saturé
- Trop de clients actifs simultanément

### 3. **Problèmes de Performance du Cluster**
- OSDs lents (latence élevée)
- Réseau de stockage saturé
- Pool de métadonnées sous-dimensionné

### 4. **Clients Zombies ou Bloqués**
- Processus clients en état D (uninterruptible sleep)
- Montages NFS/Samba sur CephFS avec problèmes
- Applications avec I/O bloquantes

---

## 🔧 Actions de Diagnostic Immédiates

### 1. Vérifier l'État du MDS

```bash
# État global du MDS
ceph fs status mycustomerfs

# Métriques détaillées du MDS actif
ceph daemon mds.node-05 perf dump

# Performance counters critiques
ceph daemon mds.node-05 perf dump | grep -E "reply_latency|forward_latency|cap_revoke"

# Sessions clients problématiques
ceph daemon mds.node-05 session ls | jq '.[] | select(.num_caps > 10000)'
```

### 2. Analyser les Clients Problématiques

```bash
# Lister tous les clients avec leurs statistiques
ceph daemon mds.node-05 client ls

# Identifier les clients avec le plus de capabilities
ceph daemon mds.node-05 client ls | jq 'sort_by(.num_caps) | reverse | .[0:10]'

# Vérifier les clients spécifiques mentionnés dans les logs
for client_id in 152718174 153052851 154745009 155623719 155854981 156250766 157328870 157328895; do
    echo "=== Client $client_id ==="
    ceph daemon mds.node-05 session evict id=$client_id --dry-run
done
```

### 3. Vérifier la Santé du Pool de Métadonnées

```bash
# Performance du pool metadata
ceph osd pool stats cephfs.mycustomerfs.meta

# Latence des OSDs hébergeant les métadonnées
ceph osd perf

# IOPS et latence du pool
rados bench -p cephfs.mycustomerfs.meta 10 write --no-cleanup
rados bench -p cephfs.mycustomerfs.meta 10 rand
```

### 4. Diagnostic Réseau depuis le MDS

```bash
# Sur le nœud node-05, vérifier la connectivité vers les clients
for host in dvs68639 dvs84148 dvs12867 dvs60059 dvs47593 dvs39961 dvs21836; do
    echo "=== $host ==="
    ping -c 3 $host.eva.produhost.net
    mtr -r -c 10 $host.eva.produhost.net
done
```

### 5. Vérifier les Clients (sur chaque client problématique)

```bash
# Statistiques du client CephFS
ceph-fuse --client-mountpoint /mnt/cephfs --client-debug-mds 10

# Vérifier les I/O en attente
iostat -x 2 5

# Processus en état D (uninterruptible)
ps aux | awk '$8 ~ /D/ {print}'

# Latence réseau vers le MDS
ping -c 100 node-05.eva.produhost.net | tail -n 1
```

---

## ⚡ Actions Correctives Immédiates

### Option 1 : Augmenter les Timeouts (Solution Temporaire)

```bash
# Augmenter le timeout de capability release (défaut: 60s)
ceph config set mds mds_cap_revoke_eviction_timeout 180

# Augmenter le timeout de session (défaut: 300s)
ceph config set mds mds_session_timeout 600

# Augmenter le timeout de réponse client
ceph config set mds mds_session_autoclose 900
```

### Option 2 : Évincer les Clients Problématiques (Solution Radicale)

```bash
# Évincer un client spécifique (ATTENTION: interruption pour le client)
ceph tell mds.node-05 client evict id=152718174

# Ou utiliser la nouvelle syntaxe
ceph daemon mds.node-05 session evict id=152718174

# Évincer tous les clients d'un hôte spécifique
ceph tell mds.node-05 client evict client_metadata.hostname=dvs21836.eva.produhost.net
```

⚠️ **ATTENTION** : L'éviction forcera les clients à remonter le filesystem et peut causer des interruptions d'application.

### Option 3 : Limiter le Cache MDS

```bash
# Réduire la taille du cache MDS si saturé
ceph config set mds mds_cache_memory_limit 17179869184  # 16GB

# Forcer un trim du cache
ceph tell mds.node-05 cache drop 10000

# Ajuster le ratio de recall
ceph config set mds mds_recall_max_decay_rate 2.5
```

---

## 🎯 Recommandations de Bonnes Pratiques

### 1. **Tuning MDS pour Environnement de Production**

```bash
# Configuration optimale pour votre setup (à adapter)
ceph config set mds mds_cache_memory_limit 34359738368  # 32GB si vous avez la RAM
ceph config set mds mds_max_purge_ops 8192
ceph config set mds mds_max_purge_ops_per_pg 8
ceph config set mds mds_recall_max_caps 30000
ceph config set mds mds_recall_warning_threshold 30000

# Optimisation des sessions clients
ceph config set mds mds_session_cache_liveness_timeout 300
ceph config set mds mds_cap_acquisition_throttle 500000
```

### 2. **Monitoring Proactif**

Mettez en place des alertes sur :

```bash
# Nombre de clients lents
ceph daemon mds.node-05 perf dump | jq '.mds.slow_reply'

# Latence de réponse moyenne
ceph daemon mds.node-05 perf dump | jq '.mds.reply_latency'

# Nombre de capabilities en attente de révocation
ceph daemon mds.node-05 perf dump | jq '.mds_cache.num_caps'
```

**Seuils recommandés** :
- `reply_latency.avgtime` < 100ms (normal), > 500ms (problème)
- `slow_reply` = 0 (idéal), > 10 (investigation nécessaire)
- `num_caps` < 2M par MDS (dépend de votre RAM)

### 3. **Optimisation Côté Client**

Sur les clients, configurez :

```bash
# Dans /etc/ceph/ceph.conf
[client]
client_cache_size = 16384          # Limite du cache client
client_oc_size = 209715200         # 200MB object cache
client_oc_max_objects = 1000       # Nombre max d'objets en cache
client_reconnect_stale = true      # Reconnexion automatique
client_caps_release_delay = 5      # Délai avant release (secondes)
```

### 4. **Architecture Multi-MDS (si applicable)**

Si vous avez un seul MDS actif et beaucoup de clients :

```bash
# Vérifier la configuration actuelle
ceph fs get mycustomerfs

# Activer multi-MDS (ATTENTION: testez en pré-prod d'abord)
ceph fs set mycustomerfs max_mds 2

# Puis progressivement augmenter si stable
ceph fs set mycustomerfs max_mds 3
```

⚠️ **Note** : Multi-MDS nécessite une bonne répartition de la charge et peut introduire de la complexité.

---

## 📊 Commandes de Diagnostic Complémentaires

```bash
# Vue d'ensemble complète du filesystem
ceph fs dump

# Statistiques temps réel
ceph daemonperf mds.node-05

# Historique des ops lentes
ceph daemon mds.node-05 dump_historic_ops

# État du cache
ceph daemon mds.node-05 cache status

# Profiling (pour analyse approfondie)
ceph daemon mds.node-05 perf histogram dump
```

---

## 🚨 Scénario de Crise - Plan d'Action

Si le MDS devient complètement non-réactif :

```bash
# 1. Failover vers le MDS standby
ceph mds fail node-05

# 2. Le standby prendra le relais automatiquement
ceph fs status mycustomerfs

# 3. Une fois stabilisé, investiguer l'ancien MDS
ceph daemon mds.node-05 dump_mds_requests
```

---

## 📈 Métriques à Surveiller Long Terme

1. **Taux de révocation de capabilities** (doit être faible)
2. **Nombre de clients simultanés** (planifier le scaling)
3. **Latence réseau MDS ↔ Clients** (< 1ms idéal)
4. **Utilisation mémoire du MDS** (< 80% de la limite configurée)
5. **IOPS du pool metadata** (doit être rapide, SSD recommandé)

---

