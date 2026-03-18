## Migration : d’ONTAP Select (S3 / File / Block) → Ceph (RGW / CephFS / RBD)

> **Objectif** – Passer l’ensemble des services de stockage fournis par NetApp ONTAP Select (Object S3, CIFS/NFS File System, iSCSI Block) vers une plateforme **Ceph** (RGW = S3‑compatible, **CephFS** = POSIX File, **RBD** = Block) tout en garantissant la continuité de service, la conformité aux SLA et la traçabilité des données.

---

## 1.  Faisabilité technique

| Aspect | ONTAP Select | Ceph (v4.x / v5.x) | Verdict |
|---|---|---|---|
| **Object (S3)** | ONTAP Select S3 via **S3 API** (bucket, versioning, lifecycle). | Ceph RGW expose **S3 compatible API** (v4 signatures, IAM, bucket policies). | ✅  Compatible – migration via **s3cmd / awscli / rclone** ou **S3‑sync**. |
| **File (NFS / CIFS)** | CIF NFS v3/v4, SMBS/SMB v3. | **CephFS** (POSIX) + **CephFS‑NFS‑Ganesha** (NFS v3/v4) / **CephFS‑SMB** (via Samba). | ✅  Possible – nécessite un **gateway** (Ganesha ou Samba) ou un **NFS‑gateway**. |
| **Block (iSCSI)** | iSCSI target LUNs, SnapMirror, SnapLock, etc. | **RBD** (block device) + **Ceph‑iSCSI‑gateway** (tgt‑adm) ou **Ceph‑iSCSI‑gateway** (LIO). | ✅  Possible – RBD → iSCSI gateway ou utilisation native RBD via KVM/containers. |
| **Snapshots / Clones** | Snapshots, clones, SnapMirror (DR). | Ceph Snapshots (RBD/FS), **RBD‑Mirror**, **CephFS‑snap**, **CRUSH‑based DR**. | ✅  Fonctionnalités équivalentes, mais les workflows changent. |
| **Performance (SSD/NVMe)** | RAID‑DP/TEC, NVMe over TCP, jumbo‑frames. | Ceph CRUSH + OSDs (SSD/NVMe) + **BlueStore**, **NVMe‑oF** support. | ✅  Peut égaler ou dépasser avec un pool bien dimensionné. |
| **Gestion** | ONTAP System Manager, CLI, AutoSupport. | Ceph Dashboard, CLI (`ceph`), Prometheus/Grafana, Ceph‑adm. | ✅  Nouveau tooling, formation requise. |
| **Licence** | Licence Capacity Tiers/Pool (per‑node). | Open‑source (GPL) – pas de licence capacity, mais besoin de **hardware adéquat**. | ✅  Économies de licence, mais coût CAPEX/OPS. |

**Conclusion** – La migration est **techniquement faisable**. Tous les services d’ONTAP Select trouventont des équivalents dans Ceph. Le principal défi‑point réside dans :

1. **Ré‑architecture du réseau** (RGW, CephFS‑Ganesha, iSCSI‑gateway).  
2. **Gestion‑migration des données** (largeur de bande, fenêtres de maintenance).  
3. **Adaptation des workflows** (snapshots, policies, DR).  
4. **Formation du personnel** (Ceph‑adm, CRUSH, RBD‑Mirror, RGW IAM).  

---

## 2.  Pré‑requis organisationnels & techniques

| Niveau | Pré‑requis | Détails |
|---|---|---|
| **Gouvernance** | **Sponsor exécutif** & **budget approuvé** | CAPEX pour serveurs, SSD/NVMe, switches, licences de support (Red Hat / SUSE, ou Ceph‑Enterprise). |
| **Architecture** | **Dimensionnement du cluster Ceph** (OSDs, MONs, MGRs, RGW, MDS, iSCSI‑gateway). | - Minimum 3 MONs (HA). <br>- 1 MGR (ou 2 pour HA). <br>- 1 RGW / zone par région (S3). <br>- 1 MDS par 10 TB de CephFS (ou plus selon charge). <br>- OSD‑ratio ≈ 1 : 2 (SSD / HDD) selon workload. |
| **Hardware** | **Serveurs** (x86‑64, BIOS VT‑d, 2 × NIC 10 GbE + 2 × NIC 25 GbE). <br>**Disques** : SSD ≥ 2 TB (BlueStore OSD), HDD NVMe ≥ 4 TB (journaux). <br>**Réseau** : VLAN internal (9 000 MTU) + VLAN external (1 500 MTU) + **RDMA** (optionnel). | Utiliser les mêmes serveurs que ceux qui hébergent ONTAP Select (re‑utilisation possible si capacité suffisante). |
| **Logiciel** | **OS** (RHEL 9 / Rocky 9 / SLES 15 SP4) – **Ceph Octopus / Pacific / Quincy** (v4.x‑v5.x). <br>**Ceph‑adm** (container‑based). <br>**RGW** + **IAM** (Keycloak ou Ceph‑native). <br>**Ganesha** / **Samba** pour NFS/SMB. <br>**tgt‑adm** ou **LIO** pour iSCSI. | Toutes‑installer via **ceph‑adm** (bootstrap, mon, mgr, osd, mds, rgw, iscsi‑gateway). |
| **Sécurité** | **TLS** pour RGW, **Kerberos**/AD pour CIFS, **CHAP** pour iSCSI. <br>**MFA** pour admin Ceph (Vault + OTP). | Impl‑configurer les **firewalls** (ports 6789, 7480, 80/443, 3260, 24007‑24008). |
| **Intégration** | **Active Directory** (pour CIFS), **IAM** (pour S3), **vCenter** (pour VM RBD). | Créer des **domains‑mappings** (AD‑bind, RGW‑policy). |
| **Plan de continuité** | **DR** (Ceph‑Mirror ou Multi‑site CRUSH), **Back‑up** (Ceph‑FS‑Snapshot → Object Store, RBD‑Export). | Prévoir une **zone secondaire** (ex. AWS S3, Azure Blob) pour réplication. |
| **Outils de migration** | **rclone**, **s3cmd**, **awscli**, **rsync**, **Ceph‑FS‑snapshot → RBD‑clone**, **iscsi‑target‑exporter**. | Tester d’automatisation (Ansible, Terraform). |
| **Tests & Validation** | **Lab** (2 nodes Ceph) pour valider chaque flux (S3, NFS, iSCSI). <br> **KPIs** (latence, débit, IOPS). | Définir **SLAs** à atteindre avant go‑live. |
| **Documentation** | **Run‑books** (provisioning, upgrade, fail‑over). <br> **Run‑book de rollback** (re‑activer ONTAP). | Stocker dans Confluence / Git. |

---

## 3.  Architecture cible Ceph

```
+-------------------+          +-------------------+          +-------------------+
|   RGW (S3)        |  <--->   |   Ceph Cluster    |  <--->   |   iSCSI‑Gateway   |
|  (HTTP/HTTPS 80/443)      |  (MON/MGR/OSD)    |          | (tgt‑adm / LIO)   |
+-------------------+          +-------------------+          +-------------------+
          ^                               ^                         ^
          |                               |                         |
          |                               |                         |
   +------v------+                +-------v------+          +-------v------+
   |   Clients   |                |   Clients   |          |   Clients   |
   | (S3 SDK,   |                | (NFS/SMB)   |          | (iSCSI initiator) |
   +------------+                +------------+          +-------------------+

CephFS (MDS) attaches to the same cluster – expose NFS via Ganesha
and SMB via Samba (or via CephFS‑kernel client for Linux).
```

- **Réseau interne** (CRUSH) : MTU 9 000, 10 GbE + 25 GbE, **RDMA** (optionnel) pour RBD/OSD.
- **Réseau externe** (clients) : 1 500 MTU, VLAN S3, VLAN NFS/SMB, VLAN iSCSI.
- **Sé RGW** : 1 zone / region (ou 2 zones pour DR).
- **Pools** :  
  - `rbd_pool` (replicated 3 × or erasure‑coded).  
  - `fs_data` (CephFS).  
  - `rgw_data` (RGW).  
  - `log_pool` (for RGW logs).  
  - `crash` (default).  

---

## 4.  Plan de migration détaillé (6 phases)

### Phase 0 – Pré‑préparation (2‑4 semaines)

| Action | Responsable | Livrable |
|---|---|---|
| **Kick‑off** – définir de projet, périmètre, SLA. | PM | Charte projet. |
| **Inventaire ONTAP** – volumes, buckets, LUNs, policies, Snap, ACLs. | Storage Engineer | Export CSV/JSON (via `volume show -fields`, `bucket list`, `lun show`). |
| **Dimensionnement Ceph** – calcul du **usable capacity** (incl. replication factor 3). | Architecte | Diagramme de capacité, BOM. |
| **Lab de validation** – déployer 2 nœuds Ceph (VM) et tester S3/NFS/iSCSI. | Lab Team | Rapport de validation. |
| **Définir les fenêtres de migration** – périodes de faible activité, plan de rollback‑back. | Ops | Calendrier. |
| **Plan de formation** – workshops Ceph‑adm, RGW IAM, Ganesha. | Training Lead | Sessions planifiées. |

### Phase 1 – Installation du cluster Ceph (2‑3 semaines)

1. **Bootstrap MON/MGR** (`cephadm bootstrap --mon-ip <IP>`).  
2. **Déployer OSDs** (BlueStore) – `ceph orch apply osd --placement="host1 host2 …"` – créer les **OSD‑cr** (SSD / NVMe).  
3. **Créer les pools** (`ceph osd pool create rbd_pool 128 128`, `ceph osd pool create fs_data 256 256`, `ceph osd pool create rgw_data 256 256`).  
4. **Activer les services** :  
   - `ceph mgr module enable dashboard` → config UI.  
   - `ceph orch apply mds cephfs --placement="host1"` → CephFS.  
   - `ceph orch apply rgw <zone> --realm‑placement="host2"` → RGW.  
   - `ceph orch apply iscsi <gateway> --placement="host3"` → iSCSI‑gateway.  
5. **Configurer le réseau** (MTU, VLAN, firewall).  
6. **Intégrer AD/LDAP** (RGW‑IAM, Ganesha‑auth).  
7. **Vérifier la santé** (`ceph -s`, `ceph health detail`).  

### Phase 2 – Migration des données **Object (S3)** (2‑4 semaines)

| Sous‑étape | Outils | Commentaire |
|---|---|---|
| **Export des métadonnées** (policies, versioning) | `aws s3api list-buckets`, `aws s3api get-bucket-policy` | Sauvegarder en JSON. |
| **Création des buckets RGW** | `radosgw-admin bucket create --bucket <name>` | Appliquer les policies (via `radosgw-admin bucket policy set`). |
| **Synchronisation initiale** | `rclone sync s3://<src‑bucket> rgw:<dst‑bucket> --transfers 32 --checkers 16` | Utiliser **multipart** + **checksum**. |
| **Vérification d’intégrité** | `rclone check`, `md5sum` sur objets aléatoires. |
| **Delta sync** (pendant la fenêtre de bascule) | `rclone sync` (second run) | Durée < 30 min si le trafic est limité. |
| **Basculement DNS / endpoint** | Modifier CNAME/Route53 → RGW endpoint. | Propagation < 5 min (TTL low). |
| **Tests d’application** | Exécution (apps) – upload/download. | Validation avant désactivation ONTAP S3. |

### Phase 3 – Migration des **File Services** (NFS / SMB) (3‑5 semaines)

| Sous‑étape | Outils | Détails |
|---|---|---|
| **Export des partages** (export‑path, ACL, quotas) | `exportfs -v`, `smbconf` → CSV. | Inclure **CIFS ACL** (AD groups). |
| **Création des CephFS volumes** | `ceph fs volume create <fs>` → `ceph fs subvolume create <fs> <subvol>`. |
| **Export NFS via Ganesha** | `ganesha.nfsd -F` – config `EXPORT` avec `FSAL` = `CEPH`. |
| **Export SMB via Samba** | `smb.conf` – backend `ceph`. |
| **Synchronisation initiale** | `rsync -avxHAX --numeric-ids --progress /mnt/oldshare/ /mnt/cephfs/subvol/` (ou **pNFS**). |
| **Vérification** | `diff -r`, `md5sum` sur un sous‑ensemble. |
| **Basculement des clients** | Modifier `/etc/fstab` (NFS) ou `net use` (SMB) vers les nouvelles exportations. |
| **Décommission ONTAP NFS/SMB** | Désactiver les exports, garderes. |
| **Tests de performance** | `fio`, `iozone` sur CephFS – comparer aux KP. |

### Phase 4 – Migration des **Block (iSCSI)** (4‑6 semaines)

| Sous‑étape | Outils | Remarques |
|---|---|---|
| **Inventaire LUNs** | `lun show -fields name,size,svm`. | Exporter en CSV. |
| **Cré les pools RBD** | `ceph osd pool create rbd_pool 128`. |
| **Créer les images RBD** | `rbd create <name> --size <GB> --pool rbd_pool`. |
| **Copie des données** | `rbd export <src> - | rbd import - <dst>` **ou** `dd if=/dev/mapper/<src> of=/dev/rbd/<pool>/<dst> bs=1M conv=noerror,sync`. |
| **Synchronisation delta** (si besoin) | `rbd diff <src> --from-snap <snap>` + `rbd import-diff`. |
| **Déployer iSCSI‑gateway** | `ceph iscsi create <gw> --pool rbd_pool`. <br> ` **targets** & **LUNs** (`ceph iscsi create-target`, `ceph iscsi create-lun`). |
| **Configurer CHAP / ACL** | `ceph iscsi auth set`. |
| **Basculement initiateurs** | Modif `iqn` ou `target portal` dans initiator config. |
| **Tests de connectivité** | `iscsiadm -m discovery`, `iscsiadm -m node -L all`. |
| **Retrait des LUNs ONTAP** | `lun delete`. |
| **Snapshot/Clone migration** | Convertir les snapshots SnapMirror → **RBD‑Mirror** (asynchrone) ou **CephFS‑**. |

### Phase 5 – Validation globale & bascule finale (1‑2 semaines)

| Action | Métriques | Acceptance Criteria |
|---|---|---|
| **Tests fonctionnels** (upload/download S3, read/write NFS/SMB, iSCSI I/O) | Latence < 5 ms (local), débit ≥ 80 % du benchmark ONTAP. | ✔‑OK. |
| **Tests de résilience** (MON fail, OSD fail, RGW restart, iSCSI‑gateway fail) | Aucun impact client > 30 s. | OK. |
| **Tests de DR/DR** (RBD‑Mirror, Ceph‑FS‑snapshot → Object) | RTO < 15 min, RPO ≤ 5 min. | OK. |
| **Audit de sécurité** (IAM, ACL, TLS) | Con les policies migrées, chiffrement en transit. | OK. |
| **Documentation & Run‑books** | Procédures de création de bucket, de partage, de LUN. | OK. |
| **Go/No‑Go** | Comité de validation (Ops, Sec, Biz). | Décision. |

### Phase 6 – Décommission d’ONTAP Select (1‑2 semaines)

| Étape | Action | Commentaire |
|---|---|---|
| **Sauvegarde finale** | Export complet des configurations ONTAP (`system configuration backup`). | Conserver 30 jours. |
| **Retrait du réseau** | Désactiver les VLAN internes ONTAP, libérer les IP. |
| **Destruction des VM** | Supprimer les VM ONTAP Select (ou les conserveriser). |
| **Re‑utilisation du hardware** | Re‑déployer les nœuds comme **OSD** supplémentaires (si souhaité). |
| **Clôture du projet** | Rapport de clôture, leçons apprises. |  |

---

## 5.  Risques & stratégies d’atténuation

| Risque | Impact | Mitigation |
|---|---|---|
| **Perte de données pendant sync** | Critique | Utiliser **checksum** (MD5/SHA256) + **double‑run** `rclone sync` + **snap‑backup** avant bascule. |
| **Incompatibilité d’ACL** (SMB/NFS) | Moyen | Mapper AD groups → CephFS‑Ganesha / Samba ACL via `ceph auth get-or-create client.<user>`. |
| **Dégradation de performance** | Moyen | Dimensionner les OSDs avec **SSD / NVMe** pour journal, activer **BlueStore** `bluestore_compression` si besoin. |
| **Bottleneck réseau interne** | Moyen | Utiliser **jumbo‑frames** (9 000) et **RDMA** (RoCE) pour RBD. |
| **Complexité du workflow de snapshots** | Moyen | Former les équipes à **RBD‑Mirror** et **CephFS‑snapshot** ; créer des scripts de migration de policies. |
| **Man des licences** (Capacité ONTAP → Ceph) | Faible | Faire‑audit de capacité (pas de licence) – prévoir marge hardware. |
| **Compétences du personnel** | Moyen | Programme de formation Ceph (Red Hat, SUSE, ou community). |
| **Interruption du service pendant bascule DNS** | Faible | Baisser TTL à 60 s **avant** migration. |
| **Défaillance du cluster Ceph en production** | Critique | Déployer **3 MONs** sur hôtes distincts, **OSD‑replication = 3**; test de **fail‑over** avant go‑live. |

---

## 6.  Estimation de planning (≈ 12‑16 semaines)

| Semaine | Activité principale |
|---|---|
| 1‑2 | Kick‑off, inventaire ONTAP, dimensionnement Ceph, lab de proof‑of‑concept. |
| 3‑4 | Installation du cluster Ceph (MON/MGR/OSD, pools, RGW, MDS, iSCSI‑gateway). |
| 5‑6 | Migration S3 (export, sync, validation, bascule DNS). |
| 7‑9 | Migration File (CephFS, Ganesha, Samba, rsync, tests). |
| 10‑13 | Migration Block (RBD, iSCSI‑gateway, delta sync, tests). |
| 14 | Validation globale, tests de résilience, audit sécurité. |
| 15 | Go/No‑Go, bascule finale, désactivation ONTAP. |
| 16 | Décommission ONTAP, documentation finale, formation post‑go‑live. |

> **Note** : Les fenêtres de migration peuvent être parallélisées (S3 et File) si la bande passante le permet. Le bloc (iSCSI) est généralement le plus long à cause du besoin de **consistance** des LUNs.

---

## 7.  Checklist de « Go‑Live »

- [ ] Tous les **buckets** RGW créés, policies appliquées, **IAM** testées.  
- [ ] CephFS exporté via **Ganesha** (NFS) et **Samba** (SMB) – ACL OK.  
- [ ] iSCSI‑gateway en production, CHAP fonctionnel, initiateurs connectés.  
- [ ] **RBD‑Mirror** (ou **CephFS‑snapshot**) configuré pour DR.  
- [ ] **Monitoring** (Prometheus + Grafana) et **Alertmanager** actifs.  
- [ ] **Backup** automatisé (Ceph‑FS → Object, RBD → S3).  
- [ ] **Documentation** mise à jour (run‑books, procédures de rollback).  
- [ ] **Formation** terminée pour l’équipe d’exploitation.  
- [ ] **Basculement DNS** réalisé, TTL < 60 s.  
- [ ] **Health** du cluster `ceph -s` = **HEALTH_OK** pendant 48 h.  

---

## 8.  Ressources utiles (links)

| Ressource | Description |
|---|---|
| **Ceph Documentation – RGW** | <https://docs.ceph.com/en/latest/radosgw/> |
| **CephFS – Ganesha Integration** | <https://docs.ceph.com/en/latest/cephfs/ganesha/> |
| **Ceph iSCSI‑gateway (tgt‑adm)** | <https://docs.ceph.com/en/latest/iscsi/> |
| **Rclone S3 ↔ RGW sync** | <https://rclone.org/ceph/> |
| **Ceph‑adm Quick‑Start** | <https://docs.ceph.com/en/latest/cephadm/> |
| **Ceph‑Mirror (RBD)** | <https://docs.ceph.com/en/latest/rbd/mirror/> |
| **NetApp ONTAP → Ceph migration guide** (community) | <https://github.com/netapp/ontap-to-ceph-migration> (exemple de scripts). |
| **Red Hat Ceph Storage – Architecture Guide** | <https://access.redhat.com/documentation/en-us/red_hat_ceph_storage/> |
| **SUSE Enterprise Storage – Best Practices** | <https://documentation.suse.com/ses/> |

---

## 9.  Résumé exécutif

1. **Oui, la migration est techniquement faisable** – chaque service ONTAP Select possède un équivalent natif dans Ceph (RGW ↔ S3, CephFS ↔ NFS/SMB, RBD + iSCSI‑gateway ↔ iSCSI).  
2. **Pré‑requis majeurs** :  
   - Dimensionnement d’un cluster Ceph (≥ 3 MON, OSDs SSD/NVMe, réseau dédié‑isolé).  
   - Outils de migration (rclone, rsync, rbd export/import).  
   - Intégration AD/LDAP, IAM, CHAP, TLS.  
   - Tests en laboratoire et validation des performances.  
3. **Plan de migration** en 6 phases (install, S3, File, Block, validation, décommission).  
4. **Risques** maîtrisables avec double‑run sync, checksum, tests de résilience, formation.  
5. **Livrables** : architecture Ceph, scripts d’automatisation, run‑books, documentation, tableau de bord de santé.  

