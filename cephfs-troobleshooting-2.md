
          # Analyse Détaillée des Performances MDS - Diagnostic Approfondi

## 📊 Synthèse de l'Analyse des Métriques

Après analyse de votre dump de performance, voici mon diagnostic :
<chat_kit_widget type=(skills) id="2267"><output>Python code interpreter...</output></chat_kit_widget>

<chat_kit_widget type=(skills) id="2267"><output>Python code interpreter...</output></chat_kit_widget>
## 🎯 Diagnostic Final et Causes Racines

### ✅ **Bonnes Nouvelles**
1. **Latence moyenne excellente** : 8.9 ms (bien en dessous du seuil de 100ms)
2. **Taux de slow replies très faible** : 0.000007% (46 sur 617M requêtes)
3. **Connexions bien équilibrées** : ~1017 clients actifs répartis uniformément
4. **Cache MDS sain** : 1.2M inodes, taux d'expiration normal

### 🔴 **Problèmes Identifiés**

#### 1. **Déséquilibre Critique du Worker-2 (PROBLÈME PRINCIPAL)**
- Le Worker-2 passe **5x plus de temps** en réception que le Worker-0
- Latence d'envoi de messages 2.5x supérieure (0.235ms vs 0.096ms)
- **Cause probable** : Clients lents ou réseau dégradé sur certaines connexions

#### 2. **Churn de Capabilities Massif**
```
Révocations : 438M
Attributions : 475M
Ratio : 0.92 (proche de 1:1)
```
**Interprétation** : Presque chaque capability accordée est ensuite révoquée, indiquant :
- Forte contention sur les fichiers/répertoires
- Clients multiples accédant aux mêmes inodes
- Pattern d'accès très dynamique (lecture/écriture concurrente)

#### 3. **Ratio Cap Release Anormal**
```
Releases clients → MDS : 3M
Releases traités internes : 261M
Ratio : 1:85
```
**Ce que cela signifie** : Le MDS traite 85x plus de releases en interne que ce que les clients confirment explicitement, suggérant :
- Des révocations forcées (timeouts)
- Des clients qui ne répondent pas assez vite
- Des évictions de cache automatiques

---

## 🔧 Plan d'Action Prioritaire

### **PHASE 1 : Diagnostic Immédiat (À faire MAINTENANT)**

#### A. Identifier les Clients sur Worker-2

```bash
# Connexions sur le MDS - identifier les clients problématiques
ceph daemon mds.node-05 session ls | jq -r '.[] | "\(.id) \(.inst) \(.num_caps) \(.recall_caps_throttle)"' | sort -k3 -rn | head -20

# Voir spécifiquement les clients avec beaucoup de caps
ceph daemon mds.node-05 session ls | jq '.[] | select(.num_caps > 5000)'

# Identifier les clients en timeout
ceph daemon mds.node-05 dump_blocked_ops
```

#### B. Vérifier les Hotspots de Contention

```bash
# Identifier les inodes les plus sollicités (hot inodes)
ceph daemon mds.node-05 dump_ops_in_flight | jq '.ops[] | select(.type_data.flag_point) | .type_data.reqid.client_t'

# Voir les locks les plus contestés
ceph daemon mds.node-05 objecter_requests

# Statistiques de cache détaillées
ceph daemon mds.node-05 cache status | jq '.cache_size, .num_caps, .num_inodes_with_caps'
```

#### C. Analyser les Patterns d'Accès

```bash
# Top des opérations par type
ceph daemon mds.node-05 perf dump | jq '.mds_server'

# Voir les opérations lentes actuelles
ceph daemon mds.node-05 ops | jq '.ops[] | select(.duration > 1)'

# Historique des opérations lentes
ceph daemon mds.node-05 dump_historic_ops | jq '.ops[] | select(.duration > 5) | {client: .client, type: .type_data.flag_point, duration}'
```

---

### **PHASE 2 : Actions Correctives Immédiates**

#### Option 1 : Augmenter les Timeouts (Recommandé en premier)

```bash
# Augmenter les timeouts pour donner plus de temps aux clients
ceph config set mds mds_cap_revoke_eviction_timeout 300  # 5 minutes (défaut: 60s)
ceph config set mds mds_session_timeout 900              # 15 minutes (défaut: 300s)
ceph config set mds mds_session_autoclose 1800           # 30 minutes

# Réduire l'agressivité des révocations
ceph config set mds mds_recall_max_decay_rate 1.5        # Plus conservateur (défaut: 2.5)
ceph config set mds mds_recall_warning_threshold 50000   # Augmenter le seuil
```

#### Option 2 : Optimiser le Cache MDS

```bash
# Augmenter la mémoire cache si disponible (vous avez 1.2M inodes, c'est gérable)
ceph config set mds mds_cache_memory_limit 34359738368   # 32 GB

# Ajuster les paramètres de trim
ceph config set mds mds_cache_trim_threshold 256000      # Garder plus en cache
ceph config set mds mds_cache_release_free_interval 30   # Libérer moins souvent

# Optimiser la gestion des capabilities
ceph config set mds mds_max_caps_per_client 2097152      # 2M caps par client
ceph config set mds mds_cap_acquisition_throttle 1000000 # Moins de throttling
```

#### Option 3 : Réduire la Contention sur les Fichiers Chauds

```bash
# Activer le cache distribué (si multi-MDS)
ceph config set mds mds_cache_reservation 0.1

# Améliorer la distribution des locks
ceph config set mds mds_scatter_nudge_interval 10        # Plus fréquent

# Optimiser les read-ahead pour réduire les aller-retours
ceph config set client client_readahead_max_bytes 8388608  # 8MB
ceph config set client client_readahead_max_periods 8
```

---

### **PHASE 3 : Investigation des Clients Lents**

Sur les **8 clients identifiés** dans vos logs (dvs68639, dvs84148, etc.) :

```bash
# Script à exécuter sur CHAQUE client problématique
#!/bin/bash
CLIENT_HOST=$1

echo "=== Diagnostic Client: $CLIENT_HOST ==="

# 1. Vérifier la latence réseau vers le MDS
echo "--- Latence réseau ---"
ping -c 100 node-05.eva.produhost.net | tail -n 2

# 2. Vérifier les processus en I/O wait
echo "--- Processus bloqués ---"
ps aux | awk '$8 ~ /D/'

# 3. Statistiques du mount CephFS
echo "--- Stats CephFS ---"
cat /sys/kernel/debug/ceph/*/mdsc

# 4. Vérifier les métriques du client
echo "--- Client metrics ---"
cat /sys/kernel/debug/ceph/*/caps

# 5. Charge système
echo "--- Load average ---"
uptime

# 6. Saturation réseau
echo "--- Network stats ---"
netstat -s | grep -E "retransmit|timeout|drop"
```

**Commandes spécifiques par client** :

```bash
# Depuis le MDS, pour chaque client_id problématique
for cid in 152718174 153052851 154745009 155623719 155854981 156250766 157328870 157328895; do
    echo "=== Client ID: $cid ==="
    ceph daemon mds.node-05 session ls | jq ".[] | select(.id == $cid)"
    ceph tell mds.node-05 client ls | jq ".[] | select(.id == $cid) | {id, inst, num_caps, client_metadata}"
done
```

---

### **PHASE 4 : Monitoring et Alerting**

Mettez en place ces métriques de surveillance :

```bash
# Script de monitoring à exécuter toutes les 5 minutes
#!/bin/bash

# 1. Latence de réponse
REPLY_LAT=$(ceph daemon mds.node-05 perf dump | jq '.mds.reply_latency.avgtime')
if (( $(echo "$REPLY_LAT > 0.1" | bc -l) )); then
    echo "ALERT: Reply latency > 100ms: ${REPLY_LAT}s"
fi

# 2. Slow replies
SLOW_REPLIES=$(ceph daemon mds.node-05 perf dump | jq '.mds.slow_reply')
if [ "$SLOW_REPLIES" -gt 10 ]; then
    echo "ALERT: Slow replies detected: $SLOW_REPLIES"
fi

# 3. Capabilities en attente
CAPS=$(ceph daemon mds.node-05 perf dump | jq '.mds_cache.num_caps')
if [ "$CAPS" -gt 1000000 ]; then
    echo "WARNING: High cap count: $CAPS"
fi

# 4. Déséquilibre des workers
ceph daemon mds.node-05 perf dump | jq -r '
    .["AsyncMessenger::Worker-0"].msgr_send_messages_queue_lat.avgtime,
    .["AsyncMessenger::Worker-1"].msgr_send_messages_queue_lat.avgtime,
    .["AsyncMessenger::Worker-2"].msgr_send_messages_queue_lat.avgtime
'

# 5. Sessions bloquées
ceph daemon mds.node-05 session ls | jq '[.[] | select(.recall_caps_throttle > 0)] | length'
```

**Seuils d'alerte recommandés** :

| Métrique | Normal | Warning | Critical |
|----------|--------|---------|----------|
| `reply_latency` | < 50ms | 50-100ms | > 100ms |
| `slow_reply` | 0-5 | 5-20 | > 20 |
| `num_caps` | < 500k | 500k-1M | > 1M |
| `worker latency ratio` | < 1.5x | 1.5-2x | > 2x |
| `cap revoke/grant ratio` | < 0.5 | 0.5-0.8 | > 0.9 |

---

## 🎯 Recommandations Stratégiques Long Terme

### 1. **Architecture Multi-MDS (Si Charge Élevée)**

Avec **~1000 clients actifs** et **12,500 req/s**, vous pourriez bénéficier de multi-MDS :

```bash
# Vérifier d'abord que vous avez des standby MDS
ceph fs status mycustomerfs

# Activer progressivement multi-MDS (TESTER EN PRÉ-PROD D'ABORD)
ceph fs set mycustomerfs max_mds 2
ceph fs set mycustomerfs allow_multimds true

# Surveiller la stabilité pendant 24h, puis augmenter si OK
ceph fs set mycustomerfs max_mds 3
```

**Avantages** :
- Distribution de la charge de capabilities
- Meilleure tolérance aux clients lents (isolation)
- Scaling horizontal

**Inconvénients** :
- Complexité accrue
- Nécessite une bonne répartition de l'arborescence
- Peut introduire de la latence sur les traversées de subtrees

### 2. **Optimisation de l'Application Côté Client**

Si possible, modifier les patterns d'accès des applications :

```bash
# Dans /etc/ceph/ceph.conf sur les clients
[client]
# Réduire la pression sur les capabilities
client_cache_size = 32768              # Augmenter le cache local
client_caps_release_delay = 10         # Retarder les releases (secondes)

# Optimiser les I/O
client_oc_size = 419430400             # 400MB object cache
client_oc_max_dirty = 209715200        # 200MB dirty data
client_oc_target_dirty = 83886080      # 80MB target

# Améliorer la reconnexion
client_reconnect_stale = true
client_mount_timeout = 300
```

### 3. **Séparation des Workloads**

Si vous avez des patterns d'accès très différents :

```bash
# Créer un second filesystem pour les workloads à haute contention
ceph fs new mycustomerfs-hot cephfs_metadata_hot cephfs_data_hot

# Ou utiliser des subtrees avec pinning
ceph fs set mycustomerfs allow_dirfrags true
# Puis distribuer manuellement les répertoires chauds sur différents MDS
```

### 4. **Tuning Réseau Avancé**

```bash
# Sur le serveur MDS (node-05)
# Augmenter les buffers réseau
sysctl -w net.core.rmem_max=134217728
sysctl -w net.core.wmem_max=134217728
sysctl -w net.ipv4.tcp_rmem="4096 87380 67108864"
sysctl -w net.ipv4.tcp_wmem="4096 65536 67108864"

# Optimiser pour faible latence
sysctl -w net.ipv4.tcp_low_latency=1
sysctl -w net.ipv4.tcp_sack=1

# Augmenter la queue des connexions
sysctl -w net.core.netdev_max_backlog=5000
```

---

## 📋 Checklist de Résolution

- [ ] **Immédiat** : Augmenter `mds_cap_revoke_eviction_timeout` à 300s
- [ ] **Immédiat** : Identifier les 10 clients avec le plus de caps
- [ ] **Court terme** : Vérifier la latence réseau vers les 8 clients problématiques
- [ ] **Court terme** : Analyser les processus en état D sur ces clients
- [ ] **Court terme** : Vérifier les logs système des clients (dmesg, syslog)
- [ ] **Moyen terme** : Optimiser la configuration client (`/etc/ceph/ceph.conf`)
- [ ] **Moyen terme** : Mettre en place le monitoring des métriques critiques
- [ ] **Long terme** : Évaluer multi-MDS en pré-production
- [ ] **Long terme** : Revoir l'architecture d'accès des applications

---

## 🔍 Questions de Suivi pour Affiner le Diagnostic

1. **Quel est le pattern d'accès** de vos applications sur CephFS ?
   - Lecture intensive / Écriture intensive / Mixte ?
   - Fichiers nombreux et petits ou gros fichiers ?
   - Accès concurrent sur les mêmes fichiers ?

2. **Configuration réseau** :
   - Latence typique entre clients et MDS ?
   - Utilisation d'un réseau dédié pour le stockage ?
   - MTU configuré (Jumbo frames activés ?) ?

3. **Charge des clients** :
   - Les 8 clients identifiés ont-ils une charge CPU/mémoire élevée ?
   - Utilisent-ils des montages kernel ou FUSE ?

4. **Historique** :
   - Le problème est-il récent ou récurrent ?
   - Corrélation avec une augmentation du nombre de clients ?
   - Changements récents dans les applications ?

Pouvez-vous me fournir la sortie de ces commandes pour affiner le diagnostic ?

```bash
ceph daemon mds.node-05 session ls | jq '[.[] | {id, inst, num_caps, recall_caps}] | sort_by(.num_caps) | reverse | .[0:10]'
ceph fs status mycustomerfs
ceph daemon mds.node-05 dump_ops_in_flight | jq '.ops | length'
``` 
