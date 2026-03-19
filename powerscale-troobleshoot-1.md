## Scénario 6.1  
**« Un client signale une latence élevée (> 200 ms) lors de l’accès à un répertoire partagé via SMB/NFS sur PowerScale »**  

Nous allons décortiquer le problème en trois parties :  

1. **Collecte d’informations / logs**  
2. **Analyse de la charge I/O (heat‑map, métriques)**  
3. **Actions de tuning (MDS, cache, réseau, configuration export)**  

>### 1️⃣ Collecte d’informations – quels logs et quelles commandes ?

| Source | Pourquoi ? | Commande / Chemin |
|--------|------------|-------------------|
| **`isi statistics view`** | Donne les métriques agrégées (IOPS, Throughput, Latency) par service (NFS, SMB, MDS, OSD). | `isi statistics view -i --metrics latency,throughput,iops --duration 1h` |
| **`isi statistics view --detail`** | Permet de filtrer par *client IP* ou *export path* (ex. : `client_ip==10.0.5.23`). | `isi statistics view -i --metrics latency --filter "client_ip==10.0.5.23"` |
| **`isi event list`** (ou `isi events list -i`) | Recherche d’alertes liéeses (ex. : “MDS CPU high”, “Network congestion”). | `isi events list -i --severity warning,critical` |
| **`isi status -d`** | État détaillé des nœuds (CPU, RAM, disque, NIC) – utile pour repérer un nœud en surcharge. | `isi status -d` |
| **`isi hardware health list`** | Vérifie l’état du matériel (NIC, cartes, contrôleurs). | `isi hardware health list -i` |
| **Log système** | `/var/log/messages`, `/var/log/isid.log`, `/var/log/isi_mds.log` – contiennent les traces d’erreurs réseau, de timeout, de débordement de cache. | `tail -n 200 /var/log/isid.log` |
| **Log dmesg** (kernel) | Détecte des erreurs de driver NIC, perte de paquets, MTU mismatch. | `dmesg | grep -i eth` |
| **SMB audit logs** (si activé) | Permet de voir le temps de réponse côté protocole. | `isi smb audit list -i` |
| **NFS export log** (si `nfs.trace` activé) | Traces détaillées des appels NFS. | `isi nfs trace start` → `isi nfs trace stop` → `cat /var/log/isi_nfs_trace.log` |
| **`netstat -i` / `ethtool -S <iface>`** | Statistiques d’erreurs NIC (RX, TX, drops). | `netstat -i` ; `ethtool -S eth0` |

> **Bon à faire** : capturez les métriques pendant la période où le client voit la latence (ex. : 10 min). Utilisez `screen` ou `nohup` pour laisser tourner les commandes.

---

### 2️⃣ Analyse de la charge I/O – heat‑map et métriques détaillées  

#### 2.1  Heat‑map des I/O par export / client  

```bash
#  : obtenir la latence moyenne par export (last 30 min)
isi statistics view -i \
    --metrics latency \
    --duration 30m \
    --group-by path \
    --format csv > /tmp/latency_by_path.csv
```

*Importez le CSV dans Excel / Grafana pour visualiser les “hot spots”.*  

#### 2.2  Heat‑map par client IP  

```bash
isi statistics view -i \
    --metrics latency,iops,throughput \
    --duration 30m \
    --group-by client_ip \
    --format csv > /tmp/latency_by_ip.csv
```

Cher les clients qui dépassent le seuil 200 ms seront immédiatement repérables.

#### 2.3  Métriques MDS (metadata server)  

```bash
# Latence du MDS (metadata) – agrégée sur tout le cluster
isi statistics view -i --metrics mds_latency --duration 30m
```

Si la latence MDS > 5 ms, le goulot d’étranglement est côté métadonnées (ex. : trop de répertoires, ACL POSIX, ACL, quotas).

#### 2.4  Métriques réseau  

```bash
# Utilisation du NIC interne (jumbo) et externe
isi network interface list -i
isi network interface stats -i
```

Recher **TX/RX drops**, **errors**, **collisions**.  

#### 2.5  Utilisation du cache (client‑side & serveur‑side)

```bash
# Cache du MDS (metadata cache hit ratio)
isi statistics view -i --metrics mds_cache_hit_ratio --duration 30m
# Cache du serveur (file data cache)
isi statistics view -i --metrics cache_hit_ratio --duration 30m
```

Un **cache‑hit ratio** < 50 % indique que le serveur‑cache ne fonctionne pas correctement (ex. : trop de churn, fichiers très volatils).

---



---

### 3️⃣ Actions de tuning – que faire selon les constats ?

| Symptom / Constat | Action de tuning | Commande / Procédure |
|--------------------|------------------|-----------------------|
| **CPU MDS saturé (> 80 %)** | - **Ajouter un MDS secondaire** (scale‑out du namespace). <br>- **Ré‑équilibrer les exports** (déplacer certains répertoires vers un autre MDS). | `isi mdsmgr add --node <node> --role standby` <br>`isi mdsmgr rebalance start` |
| **Cache‑hit ratio faible** | - **Augmenter la taille du cache MDS** (`mds_cache_size`).`). <br>- **Activer le “metadata prefetch”** (`isi metadata prefetch enable`). <br>- **Vérifier les quotas** (les quotas mal configurés ralentissent les métadonnées). | `isi metadata cache set --size 8GB` <br>`isi metadata prefetch enable` |
| **Latence réseau (drops / errors)** | - **Vérifier MTU** sur le réseau interne (jumbo = 9000) et sur le client. <br>- **Activer le flow‑control** sur les NIC (`ethtool -A ethX tx on rx on`). <br>- **Re‑router le trafic** vers un autre switch si le port est saturé. | `ifconfig eth0 mtu 9000` <br>`ethtool -A eth0 tx on rx on` |
| **Export NFS/SMB mal configuré (sync vs async)** | - **Passer les exports en async** si la cohérence stricte n’est pas requise (réduit le round‑trip). <br>- **Ajuster les “read/write size”** (ex. : `rsize=1048576,wsize=1048576`). | `isi nfs exports modify <id> --sync false` <br>`isi smb shares modify <share> --write-cache true` |
| **Trop de petits fichiers (metadata churn)** | - **Activer “SmartQuotas”** pour limiter le nombre de fichiers par répertoire. <br>- **Utiliser “SmartPool”** pour placer les petits fichiers sur un pool SSD plus rapide. | `isi quota create --type directory --path /ifs/dir --files 50000` <br>`isi storage pool create fast_pool --disk <ssd‑list>` |
| **IOPS/Throughput limité par le pool** | - **Re‑balancer les OSDs** (`isi storage pool rebalance start`). <br>- **Augmenter le nombre de PG** (`isi storage pool set --pg-num <new>`). <br>- **Ajouter des OSD SSD** pour le “cache tier”. | `isi storage pool rebalance start -p pool01` |
| **Client Windows SMB 3.0** (latence > 200 ms) | - **Vérifier la version du dialect SMB** (`SMB 3.1.1` recommandé). <br>- **Activer le “SMB Direct (RDMA)”** si les NIC le supportent. <br>- **Désactiver le “Signing‑signing”** si non requis (améliore la latence). | `isi smb shares modify <share> --dialect SMB 3.1.1` <br>`ethtool -K <iface> rdma on` |
| **Client Linux NFS v4.2** (latence) | - **Activer le “NFS async”** (déjà par défaut). <br>- **Ajuster les paramètres** (`rsize`, `wsize`, `readdirsize`). <br>- **Vérifier les ID‑map** (mappage UID/GID) – un mapping lent peut ralentir les appels. | `mount -t nfs -o rsize=1048576,wsize=1048576,vers=4.2 server:/ifs/share /mnt` |
| **Snapshot / Quota impact** | - **Désactiver temporairement les quotas** sur le répertoire concerné pour vérifier l’impact. <br>- **Ré‑évaluer les quotas** (un quota plein peut engendrer des vérifications supplémentaires). | `isi snapshots delete <snap>` <br>`isi quota delete <quota-id>` |

---

### 4️⃣ Exemple de procédure pas‑à‑pas (run‑book)

1. **Capture des métriques pendant la période de latence**  
   ```bash
   mkdir -p /tmp/ps_diag/$(date +%F_%H%M)
   isi statistics view -i --metrics latency,iops,throughput --duration 15m > /tmp/ps_diag/latency.txt &
   isi status -d > /tmp/ps_diag/status.txt
   isi events list -i > /tmp/ps_diag/events.txt
   netstat -i > /tmp/ps_diag/netstat.txt
   dmesg > /tmp/ps_diag/dmesg.txt
   ```
2. **Analyse rapide**  
   - `grep -i high /tmp/ps_diag/latency.txt` → repérer les chemins / IP.  
   - `awk '{if($5>200)print}' /tmp/ps_diag/latency.txt` → extraire les dépassements.  
   - `grep -i error /tmp/ps_diag/isid.log` → erreurs réseau.  

3. **Vérifier le MDS**  
   ```bash
   isi mdsmgr status -i
   isi statistics view -i --metrics mds_latency --duration 15m
   ```
   - Si `mds_latency` > 5 ms → envisager un MDS supplémentaire.

4. **Vérifier le réseau**  
   ```bash
   ethtool -S eth0 | egrep "rx_dropped|tx_dropped|rx_errors|tx_errors"
   ping -c 5 <client_ip>
   traceroute <client_ip>
   ```
   - Si des drops > 0 → vérifier le switch, le câble, le MTU.

5. **Ajuster le export** (exemple NFS)  
   ```bash
   isi nfs exports modify <export-id> --sync false --rsize 1048576 --wsize 1048576
   isi nfs exports list -i | grep <export-id>
   ```

6. **Re‑balancer le pool (si besoin)**  
   ```bash
   isi storage pool rebalance start -p <pool-name>
   ```

7. **Valider** – refaire les mesures pendant 10 min et comparer :  
   - Latence moyenne < 100 ms ?  
   - Aucun nouveau warning dans `isi events`.  

8. **Documentation** – créer un ticket d’incident avec :  
   - **Symptôme** (latence 200 ms, client IP, export path).  
   - **Données collectées** (logs, CSV).  
   - **Actions effectu** (tuning appliqué, dates).  
   - **Résultat** (latence résolue ou escalade).  

---

### 5️⃣ Checklist de validation post‑intervention

| ✅ | Vérification |
|---|--------------|
| 1 | Latence moyenne du client < 100 ms (mesurée avec `fio` ou `dd` depuis le client). |
| 2 | Aucun **critical** event dans `isi events list` depuis l’intervention. |
| 3 | `isi statistics view` montre `mds_latency` < 5 ms, `cache_hit_ratio` > 80 %. |
| 4 | Aucun **drop** ou **error** sur les NIC (`ethtool -S`). |
| 5 | Export NFS/SMB toujours accessible, permissions intactes. |
| 6 | Documentation du run‑book mise à jour dans Confluence / ServiceNow. |

---

## 6️⃣ Ressources complémentaires (à garder à portée de main)

| Ressource | Lien |
|-----------|------|
| **OneFS CLI Reference – Export Management** | <https://www.dell.com/support/kbdoc/en-us/000210317/onefs-9-5-cli-reference#export> |
| **OneFS Performance Tuning Guide** | <https://www.dell.com/support/kbdoc/en-us/000210317/onefs-9-5-performance-tuning> |
| **SMB Performance Best Practices** | <https://www.dell.com/support/kbdoc/en-us/000210317/smb-performance-tuning> |
| **NFS Performance Guide** | <https://www.dell.com/support/kbdoc/en-us/000210317/nfs-performance-tuning> |
| **PowerScale Health Dashboard (Grafana)** | Exemple de tableau de bord partagé par Dell EMC : <https://github.com/dell/powerscale-grafana-dashboards> |

---

### 🎌 Résumé rapide pour votre direction (extrait d’un rapport)

> **Incident** : Latence > 200 ms sur le partage `/ifs/finance` (SMB + NFS) – client Windows 10, IP 10.0.5.23.  
> **Analyse** :  
> - MDS CPU = 92 % → surcharge des métadonnées.  
> - Cache‑hit = 42 % (faible).  
> - NIC eth1 (données) affichait 12 % de drops (jumbo‑frame désactivé).  
> **Actions** :  
> 1. Ajout d’un MDS secondaire (réplication du namespace).  
> 2. Augmentation du cache MDS à 8 GB et activation du prefetch.  
> 3. Passage du MTU à 9000 sur le switch et sur le serveur client.  
> 4. Reconfiguration de l’export NFS en mode **async**, `rsize/wsize=1M`.  
> **Résultat** : Latence moyenne tombée à 68 ms, MDS CPU à 45 %, aucun drop réseau.  

---

**Vous avez maintenant toutes les pièces** pour :

1. **Diagnostiquer** le problème (logs, métriques, heat‑map).  
2. **Identifier** le goulot d’étranglement (MDS, réseau, cache, export).  
3. **Appliquer** les réglages de tuning appropri.  
