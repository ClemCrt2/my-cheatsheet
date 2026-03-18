## 1. Présentation très succincte  

| Technologie | Positionnement | Nom commercial actuel | Système d’exploitation | Architecture principale |
|-------------|----------------|----------------------|------------------------|--------------------------|
| **Dell Isilon** | NAS scale‑out (fichiers) très performant, destiné aux gros volumes de données non structurées (media, génomique, sauvegarde, archives). | **PowerScale** (Isilon = ancienne marque, le produit a été rebaptisé PowerScale en 2020). | **OneFS** – système d’exploitation propriétaire, version 9.x (cluster‑wide). | Cluster de nœuds « scale‑out » interconnectés par un réseau 10/25 GbE ou InfiniBand. |
| **Dell PowerStore** | Système hybride flash / NVMe, orienté block et file (NFS/SMB) avec fonctions avancées de dé‑duplication, compression, snapshots, intégration cloud. | **PowerStore** (modèles : X‑Series – rack‑mount, T‑Series – 2U, 4U). | **PowerStore OS** – micro‑service, version 4.x (déployé comme conteneurs). | Architecture « software‑defined » avec des « nodes » (compute + storage) qui peuvent être ajoutés dynamiquement (scale‑out). |

---

## 2. Versions actuelles (mars 2026)

| Produit | Version majeure (et point de release le plus récent) | Date de GA (ou dernier patch) |
|--------|------------------------------------------------------|-------------------------------|
| **PowerScale OneFS** | **9.5.x** – 9.5.3 (GA mars 2026) – inclut le support complet de SMB 3.2, NFS v4.2, et les nouvelles fonctions d’auto‑tuning. | 2026‑03‑12 |
| **PowerStore OS** | **4.5.x** – 4.5.2 (GA février 2026) – amélioration du “Native Cloud Tier”, du “Data Reduction Engine” et du “Unified Automation”. | 2026‑02‑24 |

> **Note** : les versions exactes peuvent varier selon le modèle (X‑Series vs T‑Series) et le niveau de support (Standard vs Premier). Consultez le **Dell Technologies Support Site** (MyDellTech) pour le *Release Tracker* de chaque produit.

---

## 3. Procédures de mise à jour (upgrade)

### 3.1 PowerScale (OneFS)

| Étape | Action | Commentaire |
|------|--------|--------------|
| **1. Pré‑vérifications** | `isi status` → vérifier que le cluster est **Healthy** (healthy) et que **all nodes are up**. <br> Vérifier la version actuelle avec `isi version`. | Si des jobs de sauveg‑up ou de snapshot sont en cours, attendez qu’ils se terminent. |
| **2. Sauvegarde de la configuration** | `isi config export -f /ifs/.snapshot/upgrade-config-$(date +%F).tgz` | Le fichier peut être restauré avec `isi config import`. |
| **3. Vérifier la compatibilité du hardware** | `isi hardware check` ou consulter le *Hardware Compatibility Matrix* (HCM) sur le site Dell. | Certains |
| **4. Télécharger le package** | Via **Dell EMC Support** → *OneFS 9.5.x* → **download**. <br> Placez le fichier `.iso` ou `.tg` sur un nœud du cluster (ex. `/ifs/.snapshot/upgrade`). | |
| **5. Lancer l’upgrade** | ``` **Mode “rolling upgrade”** (recommandé) : <br>`isi upgrade start --rolling --file /ifs/.snapshot/upgrade/OneFS-9.5.3.iso` | Le processus met à jour les nœuds un par un, le cluster reste disponible. |
| **6. Validation** | `isi status` → **Healthy** <br> `isi version` → nouvelle version <br> `isi health view` → aucune alerte critique. | |
| **7. Post‑upgrade** | Exécuter les scripts de **post‑upgrade** fournis (ex. `isi services restart` <br> Vérifier les licences avec `isi licenses list`. | |

> **Bon à savoir** : PowerScale supporte les *upgrade à chaud* (sans arrêt) depuis OneFS 8.2. Si vous avez besoin d’un arrêt complet (ex. changement de firmware de la carte réseau), utilisez `isi upgrade start --offline`.

###### 3.2 PowerStore OS

| Étape | Action | Commentaire |
|------|--------|--------------|
| **1. Vérifier la santé du système** | `svc_storage system show` ou via **PowerStore Manager → System → Health**. | Tous les indicateurs **OK** avant de commencer. |
| **2. Snapshot** | **Snapshot du système de gestion** : `svc_storage snapshot create --name pre‑upgrade-$(date +%F)` (c’est un snapshot du *metadata*). |
| **3. Vérifier la version actuelle** | `svc_storage version show` (ex. `4.4.1`). |
| **4. Consulter le *Release Notes* et le *Upgrade Matrix*** | Disponible sur le **Dell EMC Support Site** → *PowerStore OS 4.5.x* → *Upgrade Guide*. |
| **5. Télécharger le package** | Depuis le portail, récupérez le **bundle** `PowerStore_OS_4.5.2.zip`. Copiez‑le sur le **Management Node** (ou utilisez le **PowerStore Manager** → **Software → Upload**). |
| **6. Lancer l’upgrade** | **Via UI** : *System → Software → Upgrade* → sélectionner le bundle → **Start**.<br>**Via CLI** (SSH sur le node de gestion) : <br>`svc_storage upgrade start --file /var/tmp/PowerStore_OS_4.5.2.zip --mode rolling` |
| **7. Suivi** | `svc_storage upgrade status` (affiche le pourcentage par node). |
| **8. Validation** | `svc_storage health view` → aucun **Critical**.<br>`svc_storage version show` → 4.5.2. |
| **9. Post‑upgrade** | Re‑activer les services désactivés (ex. **Replication**, **Cloud Tier**) via UI ou `svc_storage service enable <service>`. <br> Vérifier les licences (`svc_storage license list`). |

> **Remarque** : PowerStore utilise un **upgrade “rolling”** par défaut. Le cluster reste accessible pendant la mise à jour, sauf si vous choisissez le mode **offline** (rare, uniquement pour des changements de firmware majeurs).

---

## 4. Commandes de vérification de santé (CLI)

### 4.1 PowerScale (OneFS)

| Catégorie | Commande | Description |
|-----------|----------|--------------|
| **État général** | `isi status` | Retourume la santé du cluster (nodes, services, network). |
| **Vue détaillée** | `isi health view` | Affiche toutes les alertes (Critical, Warning, Info). |
| **Statistiques de performance** | `isi statistics view –type node –interval 60` | CPU, I/O, réseau par nœud. |
| **Pools** | `isi storagepool list` | Liste les pools, capacité, utilisation, taux de dé‑duplication. |
| **Network** | `isi network list` | Interfaces, agrégats, MTU, état. |
| **Jobs en cours** | `isi job list –state running` | Vérifier les répliques, snapshots, migrations. |
| **Licences** | `isi licenses list` | Vérifier que les licences (SMB, NFS, S3, etc.) sont actives. |
| **Version** | `isi version` | Version exacte du logiciel. |
| **Log** | `isi event view –type error –last 24h` | Dernières erreurs système. |

> **Astuce** : toutes les commandes peuvent être exécutées en mode **“quiet”** (`-q`) pour les scripts d’automatisation.

### 4.2 PowerStore

| Catégorie | Commande (SSH) | Description |
|-----------|----------------|--------------|
| **État général** | `svc_storage system show` | Affiche le statut du système (OK/Degraded). |
| **Health** | `svc_storage health view` | Toutes toutes les alertes (Critical/Warning). |
| **Version** | `svc_storage version show` | Version du OS et du firmware. |
| **Pools** | `svc_storage storagepool list` | Capacités, utilisation, taux de réduction. |
| **Volumes** | `svc_storage volume list` | Taille, type (block/NFS/SMB), snapshots. |
| **Réplicas** | `svc_storage replication list` | Statut des réplications inter‑site ou intra‑site. |
| **Performance** | `svc_storage performance show –interval 60` | IOPS, latence, bande passante par node. |
| **Network** | `svc_storage network list` | Interfaces, agrégats, VLAN, MTU. |
| **Jobs** | `svc_storage job list –state running` | Opérations de migration, snapshot, etc. |
| **Licences** | `svc_storage license list` | Licences activées (Data Reduction, Cloud Tier, etc.). |
| **Log** | `svc_storage log tail –lines 200` | Derniers messages du journal système. |

> **PowerStore Manager (UI)** : la plupart de ces informations sont disponibles sous **System → Health**, **Storage → Pools**, **Volumes**, **Performance**, etc. L’UI expose également une **REST API** (ex. `GET /api/rest/v1/system/health`) qui peut être intégrée à des outils de monitoring (Prometheus, Splunk).

---

## 5. Commandes d’administration de base

### 5.1 PowerScale

| Action | Commande | Exemple |
|--------|----------|---------|
| **Créer un pool de stockage** | `isi storagepool create <pool_name> -t <type> -d <disklist>` | `isi storagepool create poolA -t capacity -d node:1:disk1,node:1:disk2` |
| **Créer un fichier système (NFS/SMB)** | `isi zone create <zone_name>` <br> `isi nfs export create -path /ifs/data -clients * -access readwrite` | |
| **Snapshot un snapshot** | `isi snapshot create -path /ifs/data -name snap_$(date +%F)` | |
| **Déployer un nouveau nœud** | `isi node add -host <hostname> -ip <IP>` | |
| **Mettre à jour la configuration réseau** | `isi network modify -interface eth0 -mtu 9000` | |
| **Redémarrer un service** | `isi services restart <service>` | `isi services restart nfs` |
| **Supprimer un volume** | `isi rm /ifs/<volume>` | `isi rm /ifs/old_data` |

### 5.2 PowerStore

| Action | Commande (CLI) | Exemple |
|--------|----------------|---------|
| **Créer un pool** | `svc_storage storagepool create --name poolA --media-type SSD --size 50TB` | |
| **Créer un volume (NFS)** | `svc_storage volume create --name vol1 --size 5TB --protocol NFS --pool poolA` | |
| **Créer un volume (SMB)** | `svc_storage volume create --name vol2 --size 2TB --protocol SMB --pool poolA` | |
| **Snapshot d’un volume** | `svc_storage snapshot create --volume vol1 --name snap_$(date +%F)` | |
| **Ajouter un node (scale‑out)** | `svc_storage node add --host <IP> --role compute` | |
| **Modifier la bande passante d’une interface** | `svc_storage network modify --interface eth1 --mtu 9000` | |
| **Redémarrer un service** | `svc_storage service restart <service>` | `svc_storage service restart nfs` |
| **Supprimer un volume** | `svc_storage volume delete --name vol_old` | |
| **Déployer une réplication** | `svc_storage replication create --source vol1 --target <remote‑IP>:vol1_rep --type async` | |

> **Tip** : la plupart de ces actions peuvent être réalisées via **PowerStore Manager** (onglet *Storage → Volumes* ou *Pools*). L’interface graphique génère automatiquement les commandes REST correspondants.

---

## 6. Bonnes pratiques rapides (check‑list avant chaque mission)

| ✅ | Point à valider |
|---|-----------------|
| **Documentation** | Téléchargez le *Release Guide* et le *Upgrade Matrix* de la version cible. |
| **Sauvegarde de config** | Exportez la configuration (Isilon → `isi config export`; PowerStore → snapshot du metadata). |
| **Compatibilité hardware** | Vérifiez le HCM pour les modèles de nœuds, cartes réseau, firmware. |
| **Capacité de journal** | Assurez‑vous que `/var/log` (Isilon) ou `/var/log/` (PowerStore) a au moins 10 % d’espace libre. |
| **Jobs en cours** | Annulez ou terminez les migrations, snapshots, réplications avant upgrade. |
| **Licences** | Confirmez que les licences nécessaires (SMB, NFS, Cloud Tier, Data Reduction) sont actives. |
| **Plan de rollback** | Notez la procédure de la version précédente et gardez les packages d’installation à portée de main. |
| **Fenêtre de maintenance** | Même si les upgrades sont “rolling”, prévoyez une fenêtre de 2‑4 h pour les vérifications post‑upgrade. |
| **Monitoring** | Activez les alertes SNMP/Email pendant l’upgrade pour caper rapidement les problèmes. |
| **Test de performance** | Après upgrade, lancez un benchmark rapide (`fio` pour block, `dd` ou `iozone` pour file) afin de valider les IOPS/latence. |

---

## 7. Ressources utiles (2026)

| Ressource | Lien (à copier dans le navigateur) |
|-----------|-----------------------------------|
| **PowerScale OneFS Documentation** | `https://www.delltechnologies.com/en-us/documentation/powerscale/onefs.htm` |
| **PowerScale Upgrade Guide (9.5.x)** | `https://www.dell.com/support/kbdoc/en-us/000213450/powerscale-onefs-9-5-upgrade-guide` |
| **PowerStore OS Documentation** | `https://www.delltechnologies.com/en-us/documentation/powerstore.htm.htm` |
| **PowerStore Upgrade & Patch Guide (4.5.x)** | `https://www.dell.com/support/kbdoc/en-us/000214789/powerstore-os-4-5-upgrade-guide` |
| **Dell EMC Community – PowerScale** | `https://community.dell.com/t5/PowerScale/bd-p/PowerScale` |
| **Dell EMC Community – PowerStore** | `https://community.dell.com/t5/PowerStore/bd-p/PowerStore` |
| **REST API Reference (PowerStore)** | `https://developer.dell.com/api/PowerStore/` |
| **OneFS Command Reference** | `https://www.dell.com/support/kbdoc/en-us/000212976/onefs-command-reference` |

---

### En résumé

- **PowerScale (Isilon)** = NAS scale‑out, OneFS 9.5, upgrade *rolling* via `isi upgrade`.  
- **PowerStore** = Système hybride flash, PowerStore OS 4.5, upgrade via UI ou `svc_storage upgrade`.  
- Les deux plateformes offrent des CLI très riches (`isi …` / `svc_storage …`) qui permettent de vérifier la santé, de créer des pools/volumes, de gérer les snapshots et les réplications.  
- La procédure d’upgrade se résume à : **vérifier la santé → sauvegarder la config → valider la compatibilité → lancer l’upgrade rolling → valider le résultat**.  
