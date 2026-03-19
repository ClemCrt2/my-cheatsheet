### 🎯 Challenge : Questions techniques sur PowerScale, PowerStore et les architectures de migration vers Ceph  

> **Objectif** : vous mettre dans la peau d’un architecte / administrateur senior.  
> Répondez (ou notez vos pistes) à chaque question ; si vous avez besoin d’un indice, je suis là pour vous aider.  

---

## 1️⃣ PowerScale (OneFS) – Administration & Architecture  

| # | Question | Points à couvrir (pour vous guider) |
|---|----------|--------------------------------------|
| **1.1** | **Comment OneFS assure‑t‑il la tolérance aux pannes d’un nœud ?** | – rôle du **cluster‑wide namespace**, **CRUSH**, **RAID‑DP/TEC**, **HA‑pair**, **quator** (si 2‑node). |
| **1.2** | **Expliquez la différence entre un « SmartPool » et un « Tier ».** | – politique de placement, seuils de température, migration asynchrone, impact sur les performances. |
| **1.3** | **Quel est le rôle exact du service « MDS » (Metadata Server) ?** | – gestion du namespace, nombre de MDS recommand, scaling, impact sur les opérations. |
| **1.4** | **Vous devez ajouter un nœud à un cluster déjà en production. Quels sont les pré‑requis matériels et les étapes de validation avant le `isi cluster node add` ?** | – firmware, version OneFS, réseau interne, disque‑pool compatible, health du cluster, sauvegarde de la config. |
| **1.5** | **Comment configureriez‑vous un réseau interne à MTU 9000 sur un cluster PowerScale ESXi vs KVM ?** | – paramètres du vSwitch / SR‑IOV, jumbo‑frame sur le NIC, validation avec `isi network interface list`. |
| **1.6** | **Quelle est la différence pour migrer un export NFS v3 vers NFS v4.2 sans interruptionpre les clients ?** | – création du nouvel export, mise à jour du client, `exportfs -r`, gestion des ACL/IDMAP. |
| **1.7** | **Comment surveiller et diagnostiquer un « SnapMirror lag » qui dépasse les 30 minutes ?** | – `isi snapmirror list -j`, `last_sync_time`, `last_attempt` vs `now`, causes possibles (réseau, capacité, throttling). |
| **1.8** | **PowerScale propose un « S3 compatible gateway ». Quels sont les avantages et les limites de cette implémentation par rapport à un vrai S3 public ?** | – performances, compatibilité d’API (v4 signatures), politiques de cycle de vie, absence de multi‑region, facturation. |
| **1.9** | **Vous avez détecté un disque « broken » dans un agrégat RAID‑DP. Décrivez la procédure de remplacement sans perdre de données.** | – `isi storage disk replace`, `isi storage aggregate scrub`, vérification du rebuild, impact sur le pool. |
| **1.10** | **Quel mécanisme OneFS utilise‑t‑il pour garantir la cohérence des métadonnées lors d’un redémarrage brutal d’un nœud ?** | – journalisation du MDS, quorum, rôle du **Mediator**, récupération du journal. |

---

## 2️⃣ PowerStore (OS 7.x) – Administration & Architecture  

| # | Question | Points à couvrir |
|---|----------|-------------------|
| **2.1** | **PowerStore combine le modèle « all‑flash » et le modèle « hybrid ». Comment le système décide‑t‑il où placer un bloc ?** | – **Dynamic Tiering**, **SmartPools**, **Write‑Optimized vs Capacity‑Optimized**, politiques de température. |
| **2.2** | **Expliquez le fonctionnement du **Data Reduction Engine** (compression + deduplication) et son impact sur les performances d’écriture.** | – pipeline de données, seuils de “dedup‑ratio”, impact sur le CPU, utilisation de la **FlashCache**. |
| **2.3** | **Vous devez créer un groupe de réplication (SRDF‑like) entre deux PowerStore en différents sites. Quels paramètres devez‑vous configurer ?** | – **PowerStore Replication**, **RPO**, **Network QoS**, **Failover/Failback**, **Consistency Groups**. |
| **2.4** | **Comment PowerStore expose‑t‑il les volumes iSCSI et FC au travers d’un « Host Group » ?** | – création du **Host**, **Host Group**, **LUN mapping**, **CHAP/FC‑WWPN**, **Multipathing**. |
| **2.5** | **Quelle est la différence entre un **Snapshot** et un **Clone** sur PowerStore ?** | – copy‑on‑write, espace consommé, instantané vs volume modifiable, temps de création. |
| **2.6** | **Décrivez le processus de mise à jour du firmware via PowerStore Manager. Quels sont les risques si vous ne suivez pas la séquence recommandée ?** | – **Rolling Upgrade**, **maintenance mode**, **quorum**, impact sur la disponibilité. |
| **2.7** | **PowerStore propose un service d’Object (S3). Quels sont les scén de **bucket‑level lifecycle** disponibles et comment les configurer via l’API ?** | – **Expiration**, **Transition**, **Tag‑based policies**, **PUT Bucket Lifecycle**. |
| **2.8** | **Vous avez besoin d’un pool de 10 TB pour un workload de bases de données transactionnelles. Quels critères allez‑vous prior avant de le créer ?** | – **IOPS**, **latence**, **type de disque (NVMe vs SSD)**, **erasure coding vs replication**, **QoS policy**. |
| **2.9** | **Comment diagnostiqueriez‑vous une dégradation de performance soudaine sur un volume NFS ?** | – **PowerStore Insights**, **latency heatmaps**, **IOPS per client**, **network path**, **host‑side metrics**. |
| **2.10** | **PowerStore utilise un modèle de licence « capacity + feature. Comment calculer le besoin de licence lorsqu’on passe d’un pool 4 TB à 8 TB avec la même feature set ?** | – **Capacity Tier** (per‑TB), **Feature License** (S3, FC, etc.), **License‑Lock‑ID**, **renouvellement**. |

---

## 3️⃣ Migration : ONTAP Select → Ceph (Object / File / Block)

| # | Question | Points à couvrir |
|---|----------|-------------------|
| **3.1** | **Quel est le principal avantage de la réplication **RADOS Mirror** par rapport à SnapMirror ?** | – **Asynchrone vs synchrone**, **granularité**, **multi‑site**, **bandwidth throttling**, **compatibilité S3**. |
| **3.2** | **Vous devez migrer 15 TB de données S3 d’ONTAP Select vers Ceph RGW. Quelle stratégie de synchronisation choisiriez‑vous pour minimiser le temps d’indisponibilité ?** | – **Initial bulk copy** (rclone / awscli), **Delta sync**, **Versioning**, **Object lock**, **DNS cut‑over**. |
| **3.3** | **Comment mappez‑vous les permissions POSIX d’ONTAP (NFS export) vers CephFS via Ganesha ?** | – **CephFS POSIX ACL**, **Ganesha auth type**, **ID mapping**, **export options (sec=sys,anonuid)**. |
| **3.4** | **Quel est le rôle du **Ceph RBD‑Mirror** dans la continuité d’activité et comment le configurer pour un pool de bases de données ?** | – **RBD‑Mirror daemon**, **image‑level mirroring**, **sync mode (journal vs full)**, **failover script**. |
| **3.5** | **Dans un scénario de migration de LUN iSCSI d’ONTAP Select vers Ceph RBD‑iSCSI‑gateway, quels** sont les étapes critiques pour garantir l’intégrité des données ?** | – **Snapshot source**, **export RBD**, **map RBD**, **iSCSI‑gateway LUN creation**, **CHAP**, **test I/O**, **cut‑over**. |
| **3.6** | **Comment gérer les quotas d’utilisateurs ONTAP Select (quota‑tree) après migration vers Ceph FS ?** | – **CephFS quota**, **user‑id mapping**, **migration script** (export‑import), **vérification**. |
| **3.7** | **Vous avez un pool ONTAP Select avec RAID‑DP. Quel schéma de réplication recommanderiez‑vous sur Ceph pour obtenir un niveau de protection équivalent ?** | – **Replication factor = 3**, **Erasure coding (k+m)**, **CRUSH rules**, **Impact sur capacité**. |
| **3.8** | **Quel impact la différence de MTU (9 000 vs 1 500) aura‑t‑elle sur le trafic RGW et CephFS ?** | – **Jumbo frames**, **fragmentation**, **config du réseau interne Ceph**, **tuning du kernel**. |
| **3.9** | **Décrivez le processus de validation d’une migration d’un bucket S3 contenant des versions multiples d’objets.** | – **Checksum comparison**, **list‑object‑versions**, **rclone check –size‑only**, **audites de métadonnées**. |
| **3.10** | **Quel serait votre plan de rollback si, après la migration, un service critique (ex : ERP) rencontre des lat d’accès aux fichiers ?** | – **Snapshot pré‑migration**, **RGW DNS revert**, **Re‑mount NFS export ONTAP**, **run‑book**. |

---

## 4️⃣ Conception globale & Planification de capacité  

| # | Question | Points à couvrir |
|---|----------|-------------------|
| **4.1** | **Comment calculer le “usable capacity” d’un pool Ceph en tenant compte du facteur de réplication ?** | – `Usable = Raw × (Replication Factor / (Replication Factor + Overhead))`, **erasure‑coding** factor, **CR %**. |
| **4.2** | **Quel est le rôle du **CRUSH map** dans la résilience et la performance d’un cluster Ceph ?** | – **Placement rules**, **failure domains**, **weights**, **tuning (chooseleaf, straw2)**. |
| **4.3** | **Dans un environnement multi‑site, comment concevriez‑vous le réseau de réplication Ceph pour respecter un RPO ≤ 5 min ?** | – **Bandwidth**, **compression**, **asynchronous vs synchronous**, **zone‑aware CRUSH**, **tuning of `radosgw-admin zone add`**. |
| **4.4** | **Quel est l’impact d’un “slow OSD” sur les performances globales du cluster et comment le détecter ?** | – **OSD latency histogram**, **`ceph osd perf`**, **back‑pressure**, **re‑balancing**. |
| **4.5** | **Vous devez dimensionner un pool dédié pour un workload de sauvegarde (sequential writes, 10 TB / jour). Quels paramètres de pool choisiriez‑vous ?** | – **Erasure coding** (k=4, m=2) vs **replicated**, **pg_num**, **pgp_num**, **compression**, **BlueStore journal on SSD**. |
| **4.6** | **Comment implémenteriez‑vous une politique de “cold‑data tiering” depuis Ceph RGW vers un objet store public (ex : AWS S3) ?** | – **RADOSGW**, **Bucket‑policy lifecycle**, **Ceph Object Store tiering**, **S3 gateway**. |
| **4.7** | **Quel est le processus de mise à jour du kernel Ceph client sur un cluster KVM / VMware et quels risques assoc** | – **Rolling upgrade**, **compatibilité RBD version**, **downtime du client**, **fallback**. |
| **4.8** | **Comment sécuriser le trafic inter‑OSD (replication, heartbeats) ?** | – **TLS / SSL**, **IPsec**, **firewall rules**, **mutual authentication**. |
| **4.9** | **Quelle stratégie de sauvegarde recommanderiez‑vous pour les métadonnées Ceph (CRUSH map, monmap, mgr modules) ?** | – **`ceph mon dump > monmap`**, **`ceph osd getcrushmap -o crush.map`**, **snapshot du `/var/lib/ceph`**, **off‑site copy**. |
| **4.10** | **Dans un projet de migration, comment justifieriez‑vous le ROI d’une solution hybride PowerScale + Ceph vs une solution 100 % Ceph ?** | – **CAPEX vs OPEX**, **TCO**, **Performance (SSD vs NVMe)**, **Gestion des licences**, **Compétences**, **Road‑map produit**. |

---

## 5️⃣ Sécurité & Conformité  

| # | Question | Points à couvrir |
|---|----------|-------------------|
| **5.1** | **Comment activer le chiffrement au repos sur PowerScale et sur Ceph ?** | – **OneFS Encryption** (AES AES), **Ceph RBD encryption**, **RGW SSE‑S3/SSE‑C**, **Key‑Management** (KMIP, Vault). |
| **5.2** | **Décrivez le mécanisme d’authentification MFA disponible sur PowerStore et sur Ceph RGW.** | – **PowerStore MFA** (TOTP, Duo), **RGW STS / IAM with MFA**, **YubiKey**. |
| **5.3** | **Vous devez appliquer le GDPR GDPR sur les données stockées dans les buckets S3. Quels contrôles mettriez‑vous en place ?** | – **Data‑location tags**, **Retention policies**, **Encryption**, **Audit logs**, **Right‑to‑be‑forgotten (object delete)**. |
| **5.4** | **Comment impliez‑vous l’accès à un pool CephFS à un groupe AD spécifique ?** | – **CephFS auth with Kerberos**, **Ganesha auth type=krb5**, **mapping AD groups → CephFS IDs**. |
| **5.5** | **Quel est le risque d’utiliser le protocole NFS v3 dans un environnement multi‑tenant et comment le mitiger ?** | – **UID/GID collisions**, **lack of ACL**, **stateless**, **migration vers NFS v4 + Kerberos**. |
| **5.6** | **PowerScale propose le « SmartQuotas ». Comment les intégreriez‑vous à une politique de conformité ISO 27001 ?** | – **Quota‑based alerts**, **audit trail**, **reporting**, **retention**, **review périodique**. |
| **5.7** | **Dans Ceph, comment protéger les secrets (Keyring, admin key) contre le vol ?** | – **Vault / KMS**, **file permissions (600)**, **encryptedes de rotation**, **audit du fichier `/etc/ceph/ceph.client.admin.keyring`**. |
| **5.8** | **Quel est le processus de revocation d’un certificat client RGW compromis ?** | – **Revoke via `radosgw-admin`**, **update IAM policies**, **regénérer les clés d’accès**, **notify les applications**. |
| **5.9** | **Comment garantir la non‑répudiation des opérations d’administration sur PowerStore ?** | – **Audit logs**, **digital signatures**, **role‑based access**, **Syslog forwarding to SIEM**. |
| **5.10** | **Vous devez mettre en place un chiffrement de bout‑en‑bout (E2EE) pour des objets stockés dans Ceph RGW. Quels sont les choix possibles ?** | – **Client‑side encryption** (SSE‑C, AWS KMS), **Ceph Object‑Lock**, **Envelope encryption**, **SDK support**. |

---

## 6️⃣ Questions de mise en pratique (Scénario « Cas réel »)

| # | Scénario | Questions à résoudre |
|---|----------|-------------------|
| **6.1** | **Un client signale une latence élevée (> 200 ms) lors de l’accès à un répertoire partagé via SMB NFS sur PowerScale.** | 1️⃣ Quels logs consulter (`/var/log/isid.log`, `isi statistics`)? 2️⃣ Comment vérifier le **heat‑map** des I/O ? 3️⃣ Quelles actions de tuning (MDS, cache, réseau) proposeriez‑vous ? |
| **6.2** | **Après la migration d’un bucket S3 vers Ceph RGW, les objets ne sont plus accessibles via l’URL publique.** | 1️⃣ Vérifier la configuration du **virtual‑host** vs **path‑style**. 2️⃣ Contrôler les **CORS** et **bucket policy**. 3️⃣ Tester avec `aws s3api get-object` et analyser le code d’erreur. |
| **6.3** | **Un L de réplication (RAID‑DP) sur PowerStore montre une alerte « Degraded ».** | 1️⃣ Quelle est la procédure‑procédure de remplacement (sans perte de données) ? 2️⃣ Comment vérifier le **rebuild progress** ? 3️⃣ Impact sur les **Qoes de réplication** en cours ? |
| **6.4** | **Vous devez provisionner un pool Ceph pour un workload de machine‑learning (lecture‑GPU ) qui nécessite 2 TB de stockage NVMe avec IOPS > 200 k.** | 1️⃣ Choix du **type d’OSD** (NVMe BlueStore). 2️⃣ Configuration du **PG count** (formula `num_osd * 100`). 3️⃣ Activation du **bluestore cache tier** et du **RBD‑mirroring**. |
| **6.5** | **Une mise à jour du firmware du contrôleur RAID PowerScale a échoué, laissant le nœud en mode « offline ».** | 1️⃣ Comment récupérer le nœud en mode **maintenance** ? 2️⃣ Quelles vérifications de **health** effectuer avant de ré‑intégrer le nœud ? 3️⃣ Risques de **split‑brain** et comment les éviter ? |

---

## 7️⃣ Bonus – « What‑If » (pour pousser votre réflexion)

| # | Question | Pourquoi c’est intéressant |
|---|----------|-----------------------------|
| **7.1** | **Et si vous deviez réduire de 30 % la capacité de votre tout en conservant les mêmes SLA ?** | Analyse de la **déduplication**, **compression**, **tiering**, **archivage**. |
| **7.2** | **Comment géreriez‑vous la migration d’un workload de bases de données NoSQL (Cassandra) qui utilise directement le disque (raw device) sur PowerStore → Ceph RBD ?** | **RBD‑exclusive‑lock**, **journaling**, **consistency groups**, **latency impact**. |
| **7.3** | **Supposons que votre réseau interne passe de 10 GbE à 25 GbE. Quels paramètres Ceph devez‑vous retuner pour profiter de la bande passante supplémentaire ?** | **`osd_op_threads`**, **`ms_async_op_threads`**, **`filestore_max_sync_interval`**, **`rbd_cache`**. |
| **7.4** | **Vous devez garantir la conformité PCI‑DSS pour les données de cartes de paiement stockées dans un bucket S3 / RGW. Quels contrôles spécifiques** | **Encryption at rest**, **Network isolation**, **Access logging**, **Retentiones de tokenisation**, **audites de rétention**. |
| **7.5** | **Comment implémenteriez‑vous un « Zero‑Touch Provisioning » de nouveaux nœuds PowerScale via Ansible ?** | **`isi cluster node add`**, **`isi hardware inventory`**, **`isi network interface create`**, **`isi storage pool create`**, **`isi status`**. |

---

## 🧭 Comment procéder ?

1. **Choisissez un groupe de questions** (par ex. PowerScale #1‑#5) et essayez d’y répondre sans consulter la documentation.  
2. **Notez les points où vous avez besoin d’un indice** – je pourrai vous fournir un rappel ou un exemple de commande.  
3. **Passez aux scénarios** (section 6) : rédigez le run‑book que vous appliqueriez.  
4. **Terminez par un « What‑If »** pour tester votre capacité à penser en termes d’optimisation et de contraintes business.  
