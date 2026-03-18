## 1️⃣  OneFS Best‑Practices – Résumé ultra‑condensé  

| Domaine | Principes clés (OneFS v9.x) | Pourquoi c’est important pour **media rush, VM / K8s backups** |
|---------|----------------------------|--------------------------------------------------------------|
| **Structure de répertoires** | • Pré ≤ 509 niveaux, longueur ≤ 1 023 car → pas de dépassement de path. <br>• ≤ 100 k fichiers **par répertoire** (idéal ≤ 10 k). <br>• ≤ 100 k sous‑répertoires **par répertoire**. | Les gros rushes (milliers de fichiers par séquence) et les snapshots‑volumes VM sont plus rapides à parcourir si les dossiers restent « thin ». |
| **Limites de fichiers** | • Taille max = 16 TB (OneFS ≥ 8.2.2). <br>• Pour activer : `isi_large_file -c` → `isi_large_file -e`. <br>• Un fichier 16 TB consomme ≤ 10 % d’un pool → **pool ≥ 160 TB**. | Vous pourrez placer de très gros fichiers vidéo (4 K/8 K) sans fragmenter le pool et sans saturer la capacité de protection. |
| **Protection des données** | • Protection suggestée : **+2d:1n** (ou +3d:1n1d) pour les pools hybrides. <br>• Virtual Hot Spare (VHS) **activé** (10 % d’espace réservé). <br>• Mirroring 2‑8 x uniquement pour métadonnées. | Garantit que la perte d’un disque ou d’un nœud n’interrompt pas les sauvegardes VM ou les flux de capture. |
| **Gestion de la capacité** | • Ne jamais dépasser **90 %** d’utilisation d’un pool (HDD + SSD). <br>• Planifier l’ajout de nœuds dès **75 %** d’usage. <br>• Spillover **activé** (défaut). | Vos ingest de rushes peuvent exploser rapidement ; le seuil de 90 % évite les ralentissements de Snapshots/SyncIQ. |
| **SmartPools (Tiering)** | • Utiliser **atime** (access‑time) comme critère – préférer `‑atime 1d`. <br>• ≤ 30 politiques, chaque politique ≤ 3 OR + 5 AND. <br>• Policy order = *first‑match*. <br>• Virtual Hot Spare **≥ 10 %** de chaque tier. <br>• Ne pas mélanger pools SSD + HDD dans la même tier. | Vous pouvez pousser les rushes « actifs » vers le tier SSD, puis les archiver automatiquement après 30 j dans le tier HDD/Archive. |
| **Stratégie SSD** | • **Metadata‑read‑write acceleration** (6‑10 % du pool en SSD) – optimise les Snapshots et les deletes. <br>• **Data‑on‑SSD** uniquement sur pools All‑Flash. <br>• L3 Cache (SmartFlash) : 1‑2 SSD grands ≈ 150 % *working‑set* (L2 + L3). | Les snapshots de VM et les snapshots de PVC (Velero) seront beaucoup plus rapides ; les métadonnées des millions de petits fichiers (K8s) restent en SSD. |
| **Accès aux données (Data‑Access Settings)** | • **Concurrency** = défaut – bon pour workloads mixtes (VM + media). <br>• **Streaming** = à appliquer aux dossiers contenant les gros flux vidéo (préférer `‑access‑pattern streaming`). <br>• **Random** = pour répertoires de petits fichiers (<128 KB). | Vous pouvez assigner le bon pattern à chaque pool : streaming pour les dossiers `/ifs/media`, concurrency pour les PV K8s, random pour les logs. |
| **Réseau / SmartConnect** | • 10 GbE ou 40 GbE **par nœud** (minimum 1 × 10 GbE). <br>• Jumbo Frames MTU 9000 (5 % plus). <br>• Round‑Robin **load‑balancing** (policy = Round‑Robin). <br>• IP‑pool size ≈ N × (N‑1) (ex. 5 nœuds → 20 IP). <br>• Ne pas mélanger interfaces 1 GbE/10 GbE dans le même SmartConnect pool. | La bande passante élevéeée entre les serveurs de capture, les nœuds PowerScale et les clients (NFS/SMB) reste stable même en cas de panne d’une interface. |
| **Protocoles** | **NFS** – async, rsize/wsize = 1 MiB (laisser défaut). <br>**SMB 3** – multi‑channel, 3 k sessions ≤ 27 k idle. <br>**S3** – activer **Access‑Time Tracking** (ATime = 1 d) si vous utilisez les buckets ONTAP / StorageGRID. | NFS sera le canal principal pour les PV K8s (CSI‑PowerScale) ; SMB pour les équipes post‑production; S3 pour Velero et les applications d’objets. |
| **Snapshots (SnapshotIQ)** | • Limite = 20 000 snapshots/cluster, ≤ 1 024 par répertoire. <br>• Depth ≤ 275 répertoires. <br>• **Ordered delete** recommandé → supprime les plus anciens d’abord. <br>• Activer **metadata‑read/write SSD acceleration** pour des deletes rapides. | Vous pouvez prendre des snapshots toutesques (ex. toutes les 4 h) des VM / PVC sans exploser l’espace. |
| **SyncIQ (rep)** | • ≤ max 50 jobs concurrents (≥ 4 nœuds). <br>• Workers par node = 3 (par défaut) – augmenter à 5‑8 si le réseau est sous‑utilisé. <br>• Priorité = high pour les politiques RPO < 30 min. <br>• Chiffrement obligatoire sur WAN (OneFS ≥ 8.2). | Vous pouvez répliquer les snapshots vers le site DR (PowerScale ou StorageGRID) en gardant un RPO ≤ 15 min pour les VM. |
| **Quotas** | • ≤ ≤ 500 k quotas/cluster (8.2+). <br>• Ne jamais créer de quota sur **/ifs** (impact performance). <br>• Utiliser **SmartQuotas** avec `--snaps=true` pour comptabiliser les snapshots. | Permet de limiter la consommation des équipes post‑production (ex. 5 TB max par projet media). |
| **Job Engine** | • Maximum 3 jobs simultanés (hors FlexProtect/AutoBalance). <br>• Planifier les jobs lourds (SmartPools, SmartDedupe, FSAnalyze) **hors‑heure de pointe**. <br>• Gardez **VHS** et **Spillover** activés. | Évite que les jobs de tiering ou de dé‑duplication n’impactent les sauvegardes en cours. |
| **Monitoring** | • Insight > 95 % → alerte (email/SNMP). <br>• Intégrer **InsightIQ** + **ClusterIQ**. <br>• Surveiller **NFS connections** (< 1 000 / node) et **SMB sessions** (< 3 000 / node). | Vous avez une visibilité en temps réel sur la santé du cluster pendant les gros ingest de rushes. |
| **Immutabilité (SmartLock)** | • Utiliser **Enterprise mode** uniquement si vous avez besoin de conformité WORM. | Pas obligatoire nécessaire pour votre cas d’usage (media + VM). |
| **Accès Zones** | • ≤ 50 zones, chaque zone ≤ 25 k users/groups. <br>• Ne pas chevaucher les chemins sous **/ifs**. | Vous pouvez créer une zone “media‑team” et une zone “k8s‑team” avec leurs propres DNS/Groupnet. |

---

## 2️⃣  Quelles bonnes pratiques sont **critiques** pour votre architecture ?

| Domaine | Pratique à appliquer (et comment) | Impact direct sur votre flux |
|---------|-----------------------------------|------------------------------|
| **Structure de répertoires** | Créez une hiérarchie **profonde** : <br>`/ifs/media/<year>/<project>/<shot>` <br>et <br>`/ifs/vm/<env>/<host>` <br>et <br>`/ifs/k8s/<ns>/<pvc>`.<br>Limitez chaque dossier à < 100 k fichiers. | Réduction du temps de **readdirplus**, améliore les performances NFS/SMB et évite les goulets d’énumération lors des sauveg‑snapshots. |
| **Large‑file support** | Exécutez `isi_large_file -c` → `isi_large_file -e` ; assurez‑vous que chaque pool a **≥ 160 TB** (incl. protection). | Vous pouvez stocker les rushes 4 K/8 K (jusqu’à 16 TB) sans que OneFS refuse la création. |
| **Protection + VHS** | Vérifiez que chaque pool affiche **+2d:1n** (ou +3d:1n1d) et que **VHS** est **activé** (10 % d’espace). | Garantit la continuité de service pendant les pannes de dis ou de nœud ; essentiel pour les sauvegardes VM en cours. |
| **SmartPools tiering** | • Policy 1 → *Performance tier* (SSD) : `‑access‑pattern streaming` + `‑atime 1d`. <br>• Policy 2 → *Archive tier* (HDD) : `‑access‑pattern concurrency` + `‑atime 30d`. <br>• Activez **Virtual Hot Spare** (10 %). <br>• Ne créez **pas plus de 30 politiques**. | Les rushes restent sur le tier SSD tant qu’ils sont actifs, puis migrent automatiquement vers le tier HDD après 30 j, libérant de l’espace SSD pour les prochains projets. |
| **SSD strategy – metadata‑rw** | Dans chaque pool contenant SSD, choisissez **Metadata read & write acceleration** (6‑10 % du pool). | Snapshots (VM, PVC) et deletes sont **3‑5× plus rapides** – crucial pour Velero qui crée de nombreux snapshots. |
| **Data‑Access Settings** | - **Streaming** sur les dossiers contenant les gros fichiers vidéo (`/ifs/media`). <br>- **Concurrency** sur les dossiers VM et K8s (`/ifs/vm`, `/ifs/k8s`). | Optimise le placement des stripes : streaming → plus de disques, concurrency → moins de pré‑fetch, meilleur pour les petits‑fichiers K8s. |
| **Réseau / SmartConnect** | • 10 GbE ou 40 GbE **par nœud** (minimum 1 × 10 GbE). <br>• MTU 9000 partout. <br>• Round‑Robin + IP‑pool = N × (N‑1). <br>• Séparez les SmartConnect pools : un pool dédié pour le trafic **media** (10 GbE), un autre pour **VM/K8s** (40 GbE). | Évite la congestion entre les flux de capture (gros débit) et les sauvegardes VM/K8s. |
| **NFS/SMB** | - **NFS** : async, `rsize/wsize=1M` (laisser défaut). <br>- **SMB3** : multi‑channel, désactivez SMB1. <br>- Limitez **NFS connections** < 1 000 / node, **SMB sessions** < 3 000 / node. | Garantit que les clients (CSI, éditeurs) ne saturent pas le serveur de fichiers. |
| **Snapshots** | - **Ordered delete** (ex. chaque jour à 22 h). <br>- **Retention metadata‑rw** pour accélérer les deletes. <br>- Limitez **snap snapshots ≤** à 1 024 / répertoire. | Vous pouvez prendre des snapshots toutes les 4 h pour les VM / PVC sans épuiser l’espace. |
| **SyncIQ** | - **Workers = 3** (défaut) → augmentez à 5‑8 si le réseau est sous‑utilisé. <br>- **Chiffrement**AN sur WAN. <br>- **Priorité = high** pour les politiques RPO < 30 min. | Réplication fiable vers le site DR (PowerScale secondaire ou StorageGRID) avec RPO ≈ 15 min. |
| **SmartQuotas** | Créez des quotas **au niveau des projets** (`/ifs/media/projectX`) avec **hard = 5 TB** et `--snaps=true`. | Empêche un projet de monopol consommer tout le pool et déclenche des alertes automatiques. |
| **Job Engine** | - Planifiez **SmartPools** et **SmartDedupe** à 02:00 – 04:00 (hors‑heure de pointe). <br>- Gardez **3 jobs** simultanés max. <br>- **VHS** et **Spillover** restent activés. | Les jobs de tiering n’interfèrent pas avec les sauvegardes en cours. |
| **Monitoring** | - Activez les alertes **cluster‑full > 90 %** (email/SNMP). <br>- Intégrez **InsightIQ** + **ClusterIQ** pour le suivi du débit d’ingestion (media) et du nombre de snapshots. | Vous avez une visibilité proactive ; vous pouvez ajouter des nœuds avant que le seuil critique ne soit atteint. |
| **CSI‑PowerScale (v2.16)** | - Montez les volumes via **NFS** (ou SMB) ; assurez‑vous que le **pool SSD** a la stratégie **metadata‑rw**. <br>- Activez **Access‑Time Tracking** (`‑atime 1d`) afin que les politiques SmartPools fonctionnent. | Les PVC K8s créés par le driver seront automatiquement tierés : les pods “hot” restent sur SSD, les volumes inactifs migrent vers HDD. |
| **CSI‑PowerStore (v2.16)** | - Utilisez le driver **iSCSI/FC** sur les pools **+2d:1n**. <br>- Configurez les **LUNs** avec **metadata‑rw** si vous avez des SSD dans le pool. | Les VM sauvegardées en block (via PowerStore) bénéficient de la même protection et de la même accélération SSD que les fichiers. |
| **Velero → S3** | - Point : **ONTAP S3 bucket** (hot) ou **StorageGRID A** (cold). <br>- Activez **metadata‑rw** sur le bucket (via `isi bucket set –metadata‑rw`). <br>- Planifiez les **backups** toutes les 4 h, rétention = 7 jours (hot) + 30 jours (cold). | Velero utilise le même bucket que les équipes post‑production ; les snapshots‑snapshots sont rapides grâce à la SSD‑metadata. |

---

## 3️⃣  Checklist rapide à appliquer **avant** le dé du projet

| ✅ | Action | Commande / UI |
|----|--------|---------------|
| **1** | Créez la hiérarchie de dossiers (`/ifs/media`, `/ifs/vm`, `/ifs/k8s`). | `isi mkdir /ifs/media /ifs/vm /ifs/k8s` |
| **2** | Activez le support **large‑file** et vérifiez la taille du pool. | `isi_large_file -c && isi_large_file -e` |
| **3** | Vérifiez/ajustez le **niveau protection** (`+2d:1n`). | WebUI → Data Management → Storage Pools → SmartPools → *Suggested* |
| **4** | Configurez **SmartPools** (policy streaming / concurrency, atime = 1 d). | `isi storagepool policy create …` (voir doc) |
| **5** | Appliquez la **SSD strategy** « metadata‑read/write ». | `isi storagepool modify –id <pool> –ssd‑strategy metadata‑rw` |
| **6** | Activez **Virtual Hot Spare** (10 %). | WebUI → Data Management → Storage Pools → *Virtual Hot Spare* |
| **7** | Configurez **SmartConnect** (Round‑Robin, IP‑pool = N×(N‑1)). | WebUI → Network → SmartConnect |
| **8** | Activez **Jumbo Frames** sur tous les switches et interfaces. | `isi network modify –interface eth0 –mtu 9000` |
| **9** | Créez les **SmartQuotas** projet‑par‑projet. | `isi quota quotas create /ifs/media/projectX directory --hard-threshold 5T --snaps true` |
| **10** | Planifiez les **jobs** (SmartPools, SmartDedupe, SnapshotIQ) hors‑heure de pointe. | `isi job schedule …` |
| **11** | Déployez **csi‑powerscale v2.16** (NFS) et **csi‑powerstore v2.16** (iSCSI/FC) dans vos manifests K8s. | Helm chart / Operator – voir docs NetApp. |
| **12** | Configurez **Velero** → bucket **ONTAP S3** (hot) + **StorageGRID A** (cold). | `velero install --provider aws --bucket <bucket> --prefix backups --secret-file credentials-velero` |
| **13** | Activez les **alertes** (email/SNMP) pour > 90 % utilisation, > 95 % pour VHS. | WebUI → Alerts → Create Rule |
| **14** | Intégrez **InsightIQ** (ou Grafana + Prometheus) pour le suivi du débit d’ingestion média. | `isi analytics enable` + exporter. |
| **15** | Testez le **plan de reprise** : <br>‑ Snapshots → restore <br>‑ SyncIQ → failover <br>‑ Velero → restore. | `isi snapshot create …` / `isi syncIQ …` / `velero restore create …` |

---

## 4️⃣  Récapitulatif des **gains** attendus

| Objectif | Comment OneFS + vos réglages le réalisent |
|----------|-------------------------------------------|
| **Performance ingestion vidéo** | *Streaming* access‑pattern + SSD‑metadata + 40 GbE + Jumbo Frames → débit ≥ 30 GB/s, latence minimale. |
| **Sauvegarde VM / PVC** | Snapshots rapides (metadata‑rw), **ordered delete**, **SmartDedupe** en off‑peak, **SyncIQ** chiffré → RPO ≤ 15 min, RTO ≤ 4 h. |
| **Tiering & coût** | SmartPools déplace automatiquement les rushes inactifs vers HDD, libérant SSD pour les prochains projets ; **Virtual Hot Spare** évite les arrêts en cas de panne. |
| **Scalabilité** | Ajout de nœuds (max 252) sans re‑partition manuelle, SmartConnect répartition dynamique, **Spillover** évite les erreurs « no space ». |
| **Visibilité & contrôle** | InsightIQ + alerts → détection précoce des goulets, capacité d’ajouter des nœuds avant 75 % d’usage. |
| **Interopérabilité K8s** | CSI‑PowerScale (NFS) + CSI‑PowerStore (block) → PVC provisionnés selon le même pool de protection, tout en profitant du même **policy‑based tiering**. |
| **Sau multi‑tenant** | Access Zones (≤ 50) + Groupnets → isolation des équipes média vs infra, DNS dynamique via SmartConnect. |

---

### 🎯  Prochaine étape recommandée

1. **Déployer un petit cluster de lab (4 nœuds F600 + SSD)** pour valider :  
   - SmartPools policy (streaming vs concurrency)  
   - CSI‑PowerScale v2.16 + Velero → S3 bucket ONTAP  
   - Snapshots + ordered delete  

2. **Mesurer** le débit d’ingestion (media) et le temps de snapshot/delete.  

3. **Ajuster** le nombre de workers SyncIQ et la taille du pool SSD (6‑10 % du pool) jusqu’à atteindre les SLA RPO/RTO.  

4. **Planifier** l’extension du cluster (ajout de nœuds F810 All‑Flash) dès que l’utilisation atteint **80 %** pour garder **> 10 %** de marge.

