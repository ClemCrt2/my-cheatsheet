**Résumé synthétique du document « ONTAP Select documentation » (PDF – 10 mars 2026)**  

> Ce texte regroupe les informations essentielles pour planifier, déployer, administrer et exploiter ONTAP Select – la version logicielle d’ONTAP qui s’exécute sur des machines virtuelles (VM) / hyperviseurs (VMware ESXi ou Linux KVM).  
> Il couvre les versions 9.18.1 → 9.8, les nouveautés, les exigences matérielles, les modèles de licences, les architectures réseau et HA, ainsi que les procédures d’installation, d’extension, de mise à jour et de dépannage.

---

## 1.  Vue d’ensemble & concepts clés  

| Élément | Description |
|---|---|
| **ONTAP Select** | ONTAP « software‑only » déployé sous forme de VM. 1, 2, 4, 6, 8, 10 ou 12 nœuds.  Chaque nœud = VM avec l’image ONTAP. |
| **ONTAP Select Deploy** | VM Linux qui orchestre le provisionnement des nœuds, la gestion des licences, le suivi AutoSupport, les sauvegardes de configuration, etc. |
| **vNAS** | Mode « virtual NAS » : les nœuds utilisent un datastore externe (vSAN, VMFS, NFS, iSCSI) au lieu de stockage local. |
| **Capacity Tiers** | Licence per‑node (1 TB → 400 TB) – capacité verrouillée à chaque nœud. |
| **Capacity Pools** | Licence par pool partagé – la capacité est allouée dynamiquement aux nœuds d’un même cluster. |
| **HA (High‑Availability)** | Pair HA (2 nœuds) = réplication miroir des agrégats.  Le trafic HA passe par l’interface e0f (HA‑interconnect). |
| **Mediator** | Service intégré dans le Deploy qui stocke l’état HA des paires 2‑node et évite le split‑brain. |
| **RSM (RAID‑Sync‑Mirror)** | Réplication synchrone des blocs entre les nœuds d’une paire HA (interface e0e). |
| **Internal network** | Réseau dédié (jumbo MTU 7 500‑9 000 octets) utilisé uniquement pour le trafic cluster (RSM, HA‑IC, inter‑node). |
| **External network** | Réseau client (NFS/CIFS/iSCSI, gestion, SnapMirror, etc.) – MTU 1500 par défaut, VLAN/EST/VST/VGT possible. |

---

## 2.  Versions & nouveautés majeures  

| Version | Points forts |
|---|---|
| **9.18.1** (mars 2026) | • 2 nouveaux produits : *Deploy* et *Image* (remplace les 4 anciens). <br>• Expansion/contriction de cluster 4 → 12 nœuds (ESXi & KVM). <br>• Support KVM on Rocky Linux 10.1. |
| **9.17.1** | • RAID logiciel sur NVMe (KVM) via PCI‑passthrough. <br>• Expansion/contriction supportée sur KVM & ESXi (6 → 8 nœuds, …). <br>• SnapMirror cloud, SnapLock Select, vSAN ESA (multi‑node). <br>• Nouveau driver NDA (FreeBSD) remplace NVD. |
| **9.16.1** | • NLF (NetApp License File) mis à jour : ARP, ONTAP S3, S3 SnapMirror. <br>• Support ARP (mise à jour manuelle uniquement). <br>• ESXi 8.0 U3 supporté. |
| **9.15.1** | • Première/contriction uniquement sur ESXi (6 ↔ 8 nœuds). |
| **9.14.1** | • Support KVM rétabli (déprécié depuis 9.10). <br>• Fin du plug‑in vCenter Deploy. <br>• ESXi 8.0 U2 supporté. |
| **9.13.1** | • NVMe over TCP nécessite licence. <br>• ESXi 8.0 U1 GA supporté. <br>• MFA (YubiKey PIV/FIDO2) disponible. |
| **9.12.1 – 9.8** | • Pas de nouvelles fonctions majeures, uniquement correctifs et évolutions de compatibilité (ex. ESXi 7.0 U2, KVM RHEL 9.4, etc.). |

---

## 3.  Architecture de déploiement  

### 3.1  Topologie typique  

| Type | Nœuds | Réseaux | Stockage |
|---|---|---|---|
| **Single‑node** | 1 VM | 1 external (3 LIF e0a‑e0c) | DAS ou vNAS |
| **Multi‑node** | 2‑12 VM (pairs HA) | 2 networks : *internal* (e0c‑e0f) + *external* (e0a‑e0b‑e0g) | DAS (HW RAID RAID) **ou** vNAS (vSAN / external array) |
| **ROBO** | 1‑2 VM | 2 ports (1 int, 1 ext) | DAS ou vNAS |
| **MetroCluster SDS** | 2 VM (stretched) | 2 networks + médiateur distant | DAS (latence ≤ 5 ms, jitter ≤ 5 ms) |

> **Note** : chaque nœud doit être placé sur un **hyperviseur différent différent** (pas deux nœuds du même cluster sur le même hôte).  

### 3.2  Schéma réseau (multi‑node)  

```
+-------------------+      +-------------------+
|  VM (node A)     |      |  VM (node B)     |
|  e0a  → ext LIF  |←────▶|  e0a  → ext LIF  |
|  e0c/e0d → int   |      |  e0c/e0d → int   |
|  e0e  → RSM       |      |  e0e  → RSM       |
|  e0f  → HA‑IC     |      |  e0f  → HA‑IC     |
+-------------------+      +-------------------+

External network : VLAN 10 (mgmt) + VLAN 20 (data)  (MTU 1500)  
Internal network : VLAN 30 (cluster) (MTU 7 500‑9 000) – *Layer‑2 only*  
```

---

## 4.  Exigences matérielles  

| Élément | Minimum | Recommandé |
|---|---|---|
| **CPU** | Intel Xeon E5‑v3 ou plus (Sandy Bridge+) | 6 cœurs (small), 10 cœurs (medium), 18 cœurs (large) |
| **Mémoire** | 24 GiB (small), 72 GiB (medium), 136 GiB (large) | + 8 GiB réservés pour ONTAP |
| **Stockage local** | 8‑60 disques (HDD NL‑SAS/SATA/10K) **ou** 4‑60 SSD (premium) **ou** 4‑14 NVMe (premium XL) | RAID – garder 2 % de marge non‑licenciée |
| **Contrôleur RAID** | 12 Gbps, cache ≥ 512 MiB (BBU ou flash), mode *write‑back* | 24 Gbps, cache ≥ 1 GiB, support RAID 5/6/DP/TEC |
| **NIC** | 2 × 1 GbE (single‑node) <br>4 × 1 GbE ou 2 × 10 GbE (2‑node) <br>2 × 10 GbE (≥ 4‑node) | 2 × 25 GbE ou 2 × 40 GbE (haute performance) |
| **Hyperviseur** | ESXi 8.0 U3 / 9.0 **ou** KVM RHEL 10.1 / Rocky 10.1 | Dernière version supportée (voir tableau § 6) |
| **OS Deploy** | Linux 64‑bit (Debian 10, CentOS 8, etc.) | Ubuntu 22.04 LTS ou équivalent |

> **Remarque** : les nœuds utilisant le **RAID logiciel** nécessitent un **licence Premium ou Premium XL** et uniquement des SSD/NVMe. Le RAIDage PCI‑passthrough (VT‑d) doit être activé dans le BIOS.

---

## 5.  Modèles de licences  

| Modèle | Attribution | Particularités |
|---|---|---|
| **Capacity Tiers** | Licence par nœud (9 chiffres). | • Capacité verrouillée (1 TB → 400 TB). <br>• Licence perpétuelle (pas de renouvellement). <br>• Chaque nœud possède son propre *serial number*. |
| **Capacity Pools** | Licence par pool partagé (9 chiffres + License‑Lock‑ID). | • Capacité allouée à un **License Manager** (dé Deploy). <br>• Le pool peut être utilisé par plusieurs nœuds d’un même cluster. <br>• Durée de licence **définie** (renouvellement requis). <br>• Nécessite le **LLID** du Deploy pour générer le NLF. |
| **Plateformes** | Standard | Petite instance (4 CPU, 16 GiB). <br>Premium | Medium (8 CPU, 64 GiB) + SSD. <br>Premium XL | Large (16 CPU, 128 GiB) + SSD/NVMe, RAID. |
| **Évaluation** | 90 jours, licence auto‑générée, capacité maximale identique à la licence production correspondante. | Conversion → licence production possible (Tiers seul). |

### Calcul de capacité (exemple Tiers)  

- **Single‑node** : facteur = 1.13 → licence = `usable TB × 1.13` (arrondi au TB supérieur).  
- **HA‑pair** : facteur = 2.67 → licence = `usable TB × 2.67` (arrondi).  

### Calcul de capacité (exemple Pools)  

- Même facteur = 1.13 (single) ou 2.67 (HA).  
- La licence du pool doit couvrir **la somme** des licences requises par tous les nœuds du cluster.

---

## 6.  Compatibilité hyperviseur & OS  

| Hyperviseur | Versions supportées (au 10 mars 2026) |
|---|---|
| **VMware ESXi** | 9.0, 8.0 U3, 8.0 U2, 8.0 U1, 8.0 GA |
| **KVM – RHEL** | 10.1, 10.0, 9.7, 9.6, 9.5, 9.4, 9.2, 9.1, 9.0, 8.8, 8.7, 8.6 |
| **KVM – Rocky Linux** | 10.1, 10.0, 9.7, 9.6, 9.5, 9.4, 9.3, 9.2, 9.1, 9.0, 8.9, 8.8, 8.7, 8.6 |

> **Important** : Les versions ESXi 7.0 GA sont en **EOL** – migration obligatoire vers 8.0 ou 9.0.  

---

## 7.  Installation – étapes clés  

| Étape | Action principale | Points de vigilance |
|---|---|---|
| **1. Pré‑préparation de l’hôte** | • Installer OS (RHEL/Rocky) ou vérifier la version ESXi. <br>• Activer VT‑d, désactiver VMD / VMD‑V (pour NVMe). <br>• Configurer le réseau (IP statique ou DHCP). | Vérifier que le BIOS expose les ports NVMe en *passthrough*. |
| **2. Préparer le stockage** | • Créer un RAID RAID hardware (RAID 5/6/DP) ou préparer des LUNs. <br>• Pour RAID software : désactiver le contrôleur ou le mettre en mode HBA/JBOD. | Les LUNs doivent être **exclusifs** à ONTAP Select (pas de partage avec d’autres VM). |
| **3. Créer le pool de stockage (KVM)** | `virsh pool-define-as <pool> logical --source-dev /dev/sdb --target /dev/<pool>` <br> `virsh pool-build/start` <br> `virsh pool-start <pool>` | Le pool doit être **/dev/<pool>** et persister au boot (`autostart`). |
| **4. Télécharger & vérifier le OVA/TGZ Deploy** | • OVA signature (`openssl dgst -sha256 -verify …`) <br>• TGZ signature idem. | Utiliser OpenSSL 1.0.2 – 3.0. |
| **5. Déployer Deploy VM** | **ESXi** : *Deploy OVF Template* (wizard). <br>**KVM** : `virt-install … --import --disk path=ONTAPdeploy.raw …` | Choisir *VM hardware version 13* (obligatoire pour NVMe). |
| **6. Accéder à l’interface Web** | `https://<IP‑Deploy>` → admin / mot de passe. | Première connexion → changer le mot de passe admin, config AutoSupport. |
| **7. Ajouter licences** | **Licences → Capacity Tiers / Pools** → *Upload*. | Les licences doivent correspondre à la **taille d’instance** (small/medium/large). |
| **8. Enregistrer les hôtes** | *Hypervisor Hosts* → *Add* (direct ou via vCenter). | Le type (ESXi/KVM) doit être identique pour tous les nœuds du même cluster. |
| **9. Créer le cluster** | *Clusters → New* → définir > **Cluster Details**, **Node Setup**, **Network**, **Storage**. | - Les adresses IP des LIFs sont générées automatiquement (internal = link‑local). <br>- Vérifier le **Network Pre‑check** (MTU, LACP, VLAN). |
| **10. Finaliser** | Saisir le mot de passe ONTAP admin → *Create Cluster*. | Le processus dure 30‑45 min (selon taille). |
| **11. Post‑déploiement** | • Configurer SVM, volumes, policies via System Manager ou CLI. <br>• Sauvegarder la configuration Deploy (`Backup` → *Download*). | Activer AutoSupport, vérifier les **events**. |

---

## 8.  Gestion du cycle de vie  

| Action | Méthode (Web / CLI) | Remarques |
|---|---|---|
| **Upgrade d’une image de nœud** | *System → Upgrade* (dans ONTAP) **ou** `system node image modify` | Deploy ne gère pas les upgrades ; il faut d’abord mettre à jour l’image, puis redéployer le nœud. |
| **Conversion évaluation → production** | `Clusters → Modify License` (Web) ou `cluster license modify` (CLI) | Nécessite licence Capacity Tiers pour chaque nœud. |
| **Ajouter / retirer des nœuds (expansion)** | `Clusters` → *Expand Cluster* (Web) ou `cluster expand` (CLI) | Le réseau interne est vérifié avant le lancement. |
| **Contriction (réduction)** | *Contract Cluster* (Web) ou `cluster contract` (CLI) | Seules les tailles 12 → 10 → 8 → 6 → 4 sont permises. |
| **Redimensionner une instance (small → medium → large)** | `Clusters → Instance Resize` (Web) ou `cluster node modify -instance-type` (CLI) | **Large** uniquement sur ESXi (pas de support KVM). |
| **Remplacement d’un disque (RAID hardware)** | Via Deploy → *Edit Storage* (dé‑selection du disque) → *Apply*. | Le nœud redémarre automatiquement, la reconstruction démarre. |
| **Remplacement d’un disque (RAID software)** | 1️⃣ Identifier le disque (`storage disk show -fields serial`). <br>2️⃣ Sur l’hôte, détacher le disque (`virsh detach-disk … <br>3️⃣ Attacher le nouveau disque (`virsh attach-disk`).`). <br>4️⃣ Dans Deploy, *Edit Storage* → sélectionner le nouveau disque. | Nécessite un **spare** ou un **re‑build** manuel. |
| **Backup de la configuration Deploy** | *Administration → Backup* → *Create* (Web) ou `system configuration backup` (CLI) | Sauvegarder après chaque changement majeur (cluster create, licence, expansion). |
| **Restaurer la configuration Deploy** | *Restore* (Web) ou `system configuration restore` (CLI) | Après restauration, vérifier le **License Lock ID** (LLID) si vous utilisez Capacity Pools. |
| **Gestion** | | |

---

## 9.  Réseau – bonnes pratiques  

### 9.1  Configuration des vSwitches (ESXi)  

| Port‑group | Usage | Recommandations |
|---|---|---|
| **ONTAP‑External** | LIF e0a, e0b, e0g (data + mgmt) | *Load‑balancing* = **Route based on originating virtual port**. <br>MTU = 1500 (ou 9000 si le réseau le supporte). |
| **ONTAP‑Internal** | LIF e0c‑e0f (cluster, RSM, HA‑IC) | *Load‑balancing* = **Route based on IP hash** (ou **originating‑port**). <br>MTU = 7 500‑9 000. |
| **VM‑NIC assignment** | 4 VM‑NIC max. | **2 NIC** → External + Internal (actif/standby). <br>**4 NIC** → External (2) + Internal (2). |
| **LACP** (Distributed vSwitch) | Autorisé, mais **pas recommandé** pour le trafic interne (préférez des NIC‑team simples). | Si utilisé, configurer le LAG sur le switch avec **fast timer** (1 s). |

### 9.2  Tagging VLAN  

| Méthode | Où le tag est appliqué | Cas d’usage |
|---|---|---|
| **EST (External Switch Tagging)** | Sur le **switch physique** (port trunk). | Simplicité, pas besoin de VGT. |
| **VST (Virtual Switch Tagging)** | Sur le **port‑group** du vSwitch. | Le VM ne voit pas les VLANs – idéal pour l’**internal**. |
| **VGT (Virtual Guest Tagging)** | Dans l’OS invité (ONTAP) via **VLAN ID** sur les LIF. | Nécessaire quand plusieurs IPspaces/ VLANs sont requis sur le même port (ex. : 2 IPspaces). **Non supporté** sur les nœuds 2‑node + mediator (seulement EST/VST). |

### 9.3  MTU & Jumbo Frames  

- **Internal network** : 7 500‑9 000 bytes (défaut = 9 000).  
- **External network** : 1 500 bytes (ou 9 000 si le réseau le permet).  
- **Vérifier** : `network connectivity-check start --type internal` → *run‑id* → `show`.  

### 9.4  Redondance physique  

- **2‑node** : 1 × 10 GbE (ou 4 × 1 GbE) suffit.  
- **≥ 4‑node** : **2 × 10 GbE** (ou 4 × 25/40 GbE) recommandé.  
- **Switches** : config **PortFast** (STP) + **LACP fast** (si utilisé).  
- **Multi‑switch** : chaque NIC‑team connectée à un switch différent → protection contre un switch unique.  

---

## 10.  Haute disponibilité (HA)  

| Élément | Fonction |
|---|---|
| **HA‑pair** | Deux nœuds (mirrored aggregates). |
| **Mediator** | Médiateur (iSCSI target) stocke l’état HA – indispensable pour les paires 2‑node. |
| **RSM** | Réplication synchrone des blocs via LIF e0e (MTU 7 500‑9 000). |
| **HA‑IC** | Inter‑cluster heartbeat via LIF e0f (RDMA‑over‑Ethernet). |
| **Failover** | En cas de perte d’un nœud, le nœud survivant prend la charge (mirrored aggregate). |
| **Split‑brain** | Évité grâce au médiateur + quorum (mediator + 2 nœuds). |
| **MetroCluster SDS** | Deux nœuds séparés jusqu’à 10 km, latence ≤ 5 ms, jitter ≤ 5 ms, médiateur distant (IP ≤ 125 ms RTT). |

> **Prérequis** : chaque HA‑pair doit disposer d’un **spare** disque dans chaque agrégat (RAID‑DP/TEC) et d’une **licence** d’au moins la même taille que le disque le plus grand du nœud.

---

## 11.  Stockage – concepts & limites  

| Concept | Détails |
|---|---|
| **Aggregates** | RAID 4 (≤) ou RAID‑DP/TEC selon le nombre de disques. <br>Root = 68 GB (single) / 136 GB (HA). |
| **Data plexes** | Deux plexes (local + mirror) pour HA. |
| **Virtual disks (VMDK)** | Taille max = 8 TB (création) ; 16 TB (extension via *storage‑add*). |
| **NVRAM** | VMDK dédié, stocké sur le même datastore que le système (ou NVMe). <br>Sur hardware RAID, le cache du contrôleur agit comme NVRAM. |
| **Software RAID** | Né RAID 4/DP/TEC implémenté dans ONTAP – uniquement Premium XL + SSD/NVMe. |
| **vNAS** | Pas de RAID matériel – la résilience dépend du datastore externe (vSAN, VMFS, NFS). |
| **Capacity cap** | Optionnel – limite l’espace consommé par ONTAP (utile pour éviter d’utiliser la partie 2 % non‑licenciée). |
| **Buffer 2 %** | Réservé automatiquement sur chaque datastore – non‑licencié. |
| **Limite de 64 TB** | Taille maximale d’un **datastore** VMFS 5/6 (sauf vSAN). <br>Pour dépasser, créer plusieurs datastores ou utiliser vSAN. |
| **Maximum par nœud** | 400 TB de capacité brute (raw) – dépend du type de disque (SSD ≈ 16 TB / NVMe ≈ 16 TB). |
| **Maximum par cluster** | 12 nœuds × 400 TB = 4,8 PB (mais limité par licences et par le nombre de LUN/VMDK). |

---

## 12.  Sécurité & authentification  

| Fonction | Implémentation |
|---|---|
| **MFA** | Depuis 9.13.1 – YubiKey PIV ou **FIDO2** (YubiKey) via CLI (`security multifactor authentication enable`). |
| **Gestion** | Stockage chiffré AES‑256 + hash SHA‑256. |
| **Credential Store** | Base de données interne du Deploy – stocke les cred. hôte, vCenter, etc. |
| **Acc** | Accès admin via clé publique (ou MFA). |
| **AutoSupport** | Activé par défaut – envoie logs à NetApp. |
| **TLS 443** | Utilisé pour toutes les communications Deploy ↔ hyperviseur, Deploy ↔ nœuds. |
| **ICMP** | Utilisé uniquement pour les tests de connectivité (ping). |
| **Ports requis** | 443 (TLS), 902 (VMware VIX), 22 (SSH), 3260 (iSCSI), 80/443 (AutoSupport), 7200‑7400 (vSphere API). |

---

## 13.  Dépannage rapide (check‑list)  

| Symptom | Action recommandée |
|---|---|
| **Cluster stuck in *create_failed*** | Vérifier le **Network Pre‑check** (MTU, VLAN, LACP). <br>Consulter les **Events** → `cluster create`. |
| **Nœud offline après upgrade** | `system node reboot -node <name>` via CLI ou redémarrer la VM. |
| **RSM/HA‑IC down** | `network connectivity-check start --type internal` → analyser les logs. |
| **Licence expirée (Capacity)** | Renouveler la licence via le portail NetApp → *Upload* nouvelle licence. |
| **Disque en *broken*** | `storage disk fail -disk <id> -immediate` (simulation) → vérifier le **spare**. <br>Supprimer le label *broken* : `set advanced disk unfail -disk <id> -spare true`. |
| **Performance faible** | Vérifier la **MTU** du réseau interne, la **configuration NIC‑team**, la **charge du RAID controller** (cache). |
| **AutoSupport absent** | `settings & autosupport` → activer, vérifier le proxy (si présent). |
| **Mediator non‑accessible** | S’assurer que le **mediator** tourne a une IP statique et que le port 3260 est ouvert. |
| **VMware vMotion / DRS déplacé** | Après migration, **refresh** le cluster dans Deploy (`Cluster → Refresh`). |

---

## 14.  Points de vigilance & meilleures pratiques (récapitulatif)  

| Domaine | Best‑practice |
|---|---|
| **Planification** | • Définir le **modèle de licence** (Tiers vs Pools) avant le dimensionnement. <br>• Calculer la capacité avec le **facteur 1.13 / 2.67**. |
| **Hardware** | • Utiliser des **disques homogènes** (même type / vitesse). <br>• RAID controller = write‑back + BBU/FBWC. <br>• Pour NVMe → passthrough (VT‑d) et **pas de RAID hardware**. |
| **Réseau** | • Séparer **internal** et **external** (VLAN distinctes). <br>• MTU = 9 000 sur le réseau interne. <br>• Utiliser **NIC‑team** avec LACP fast ou act‑standby. |
| **HA** | • Toujours disposer d’un **mediator** actif (IP statique). <br>• Configurer **spare** dans chaque agrégat. |
| **Licences** | • Les licences **Pools** sont liées à l’**LLID** du Deploy – ne pas changer d’instance Deploy sans ré‑générer les licences. |
| **Mise à jour** | • Mettre à jour **Deploy** avant de mettre à jour les nœuds. <br>• Après upgrade, valider les **features de stockage** (dedup, compression). |
| **Sauvegarde** | • Exporter la configuration Deploy **après chaque changement majeur**. <br>• Conserver les backups hors du cluster (ex. NFS, S3). |
| **Monitoring** | • Activer **AutoSupport** et **Event logging**. <br>• Utiliser le **Network Connectivity Checker** avant chaque expansion. |
| **Sécurité** | • Activer **MFA** pour l’accès admin Deploy. <br>• Restreindre l’accès SSH (key‑only). |
| **vSAN ESA** | • Nécessite **Premium XL** + licence vSAN ESA. <br>• Utiliser des **datastores vSAN** de même performance que les autres nœuds. |

---

### 📌 En une phrase

> **ONTAP Select** offre la même richesse fonctionnelle qu’un système ONTAP matériel, mais nécessite sur une infrastructure virtualisée ; le succès du déploiement repose sur une **planification rigoureuse** (licences, capacité, réseau), le respect des **exigences matérielles** (RAID, NIC, NVMe) et l’utilisation du **Deploy utility** pour touteser le provisionnement, la mise à jour et la sauvegarde de la configuration.  

--- 

**Documents de référence complémentaires**  

- **Interoperability Matrix Tool (IMT)** – pour vérifier la compatibilité exactes hyperviseur/OS/firmware.  
- **Technical Report – ONTAP Select Product Architecture & Best Practices** – détails sur le réseau, le stockage et les performances.  
- **TR‑4647 – Multifactor Authentication in ONTAP** – procédure d’activation YubiKey.  

--- 
