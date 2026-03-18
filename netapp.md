## 1. Présentation très courte de NetApp (2026)

| Domaine | Produit phare | Fonction principale | Positionnement S3 / Objet |
|---------|----------------|---------------------|---------------------------|
| **Syst de fichiers + block** | **ONTAP** (déployé sur les plates‑formes **AFF** – All‑Flash FAS, **FAS** – hybride, **E‑Series** – disque) | Gestion : NAS (NFS, SMB), SAN (iSCSI, FC) et **S3‑compatible object** via le **S3 API d’ONTAP**. | **ONTAP 9.12** (GA 2025) intègre un service S3 natif (déploiement « S3 Gateway » ou « S3 Native ») qui expose les volumes NFS/SMB comme des buckets S3. |
| **Object‑storage dédié** | **NetApp StorageGRID** (déploiement « on‑prem », « cloud » ou hybride) | Stockage d’objets à très grande échelle, compatible 100 % avec l’API **Amazon S3** (et S3‑compatible d’autres fournisseurs). | Version **StorageGRID 7.2** (GA 2025) – multi‑site, tiering vers vers le cloud (Azure Blob, AWS S3, Google Cloud Storage). |
| **Cloud‑native** | **Cloud Volumes ONTAP (CVO)** – service managé sur AWS, Azure, GCP | ONTAP en tant que service, expose **NFS/SMB/iSCSI** **et** une **API S3** (via le « S3 Gateway » intégré). | Version **CVO 3.12** (GA 2025). |
| **Data‑Protection** | **SnapMirror**, **SnapVault**, **SnapLock**, **NetApp Data Fabric** | Réplication, sauvegarde, archivage, conformité. | Tous ces services peuvent être utilisés avec les buckets S3 (ex. réplication inter‑site de StorageGRID). |
| **Gestion & Automation** | **NetApp ONTAP System Manager**, **Active (CLI)**, **REST API**, **PowerShell/Ansible modules** | Administration unifiée, monitoring, orchestration. | Les mêmes interfaces gèrent les volumes classiques et les buckets S3. |

> **En bref** : NetApp propose **deux voies** pour le stockage objet :  
> 1. **ONTAP S3 Native** – ajoute une couche S3 sur les volumes traditionnels (AFF/FAS/E‑Series). Idéal quand vous avez déjà un cluster ONTAP et que vous voulez exposer quelques buckets sans installer un nouveau produit.  
> 2. **StorageGRID** – solution objet « pure‑prem » ou hybride, conçue pour les gros volumes, la conformité et le tiering multi‑cloud.

---

## 2. Versions actuelles (mars 2026)

| Produit | Version majeure (et point de release le plus récent) | Date de GA |
|---------|------------------------------------------------------|------------|
| **ONTAP** | **9.12.1** (GA janvier 2026) – ajoute le *S3 Native* en mode **“S3‑only”** et améliore le *Data‑Efficiency Engine*. | 2026‑01‑14 |
| **StorageGRID** | **7.2.0** (GA février 2026) – support du *S3 Object Lock* et du *Multi‑Region Tiering*. | 2026‑02‑09 |
| **Cloud Volumes ONTAP (CVO)** | **3.12.2** (GA mars 2026) – intégr du *S3 Gateway* et du *Hybrid Cloud Tier*. | 2026‑03‑02 |

---

## 3. Comment connaître le contrat NetApp qui est en cours ?

| Étape | Action | Où la faire |
|------|--------|--------------|
| **1. Accès au portail client** | Connectez‑vous sur **My NetApp** (https://my.netapp.com). | Vous avez besoin du **Customer ID** ou du **login** fourni par votre interloc. |
| **2. Re du contrat** | Dans le menu **“Support & Services” → “Contracts & Entitlements”**. Vous y verrez : <br>• Numéro du contrat (ex. **N‑12345678**) <br>• Type (Standard, Premier, ProSupport) <br>• Périmètre (ONTAP, StorageGRID, CVO) <br>• Date d’expiration. | My NetApp |
| **3. Via le numéro de série du matériel** | Sur chaque appliance (AFF/FAS/StorageGRID) le **serial number** (ex. **A1B2C3D4**) peut être entré dans **My NetApp → “Asset Lookup”**. Le résultat indique le contrat qui couvre cet asset. | My NetApp |
| **4. En cas d’accès limité** | - **Contactez votre représentant** (ou le revendeur Dell / Tech Partner) – ils ont généralement le contrat dans leur CRM. <br> - **Appelez le support NetApp** (1‑800‑887‑2639 US / +33 1 70 44 70 00 FR) et fournissez le serial number ; ils peuvent vous confirmer le contrat. |
| **5. Documentation** | Conservez le **PDF du contrat** et le **Certificate of Coverage** (CoC) qui détaille les services inclus (HW, SW, support, licences S3, etc.). | My NetApp ou votre service juridique. |

> **Conseil** : créez un ticket interne « Contract Lookup » dans votre outil de suivi (Jira, ServiceNow…) afin de garder une trace de la demande et de la réponse du support.

---

## 4. Procédures de mise à jour (upgrade) – NetApp

### 4.1 ONTAP (cluster AFF/FAS/E‑Series)

| Étape | Action | Commande / UI |
|------|--------|----------------|
| **1. Vérifier la santé** | `cluster show` → **“healthy”** <br> `system node run -node <node> -command "system health"` | CLI ou **System Manager → Cluster → Health** |
| **2. Sauvegarde de la configuration** | `system configuration backup create -destination <path>` (ex. `/etc/ontap/backup/$(date +%F).tgz`) | |
| **3. Vérifier le **Upgrade Matrix** (PDF) | <https://www.netapp.com/support/upgrade-matrix> | |
| **4. Télécharger le package** | Via **Support Site** → *ONTAP 9.12.1* → **download** (ISO ou .tgz). Copiez‑le sur le **cluster** (ex. `/mnt/bootbank/upgrade`).`). | |
| **5. Lancer l’upgrade (rolling)** | ```cluster image modify -image /mnt/bootbank/upgrade/ONTAP_9.12.1.tgz -node <node> -rolling``` <br> Répétez pour chaque nœud ou utilisez le cluster le faire automatiquement. | |
| **6. Validation** | `cluster show` → **“healthy”** <br> `system version show` → 9.12.1 <br> `system health view` → aucune alerte critique. | |
| **7. Post‑upgrade** | Re‑activer les services désactivés (ex. `system services start nfs` <br> Vérifier les licences S3 (`system services license show`). | |

> **Remarque** : depuis ONTAP 9.11, le **S3 Native** peut être activé **sans** sans redémarrage : `vserver services s3 enable -vserver <sv>`. L’upgrade ne doit donc pas impacter les buckets existants.

### 4.2 StorageGRID

| Étape | Action | UI / CLI |
|------|--------|----------|
| **1. Health check** | **StorageGRID → System → Health** (Dashboard) ou `grid health show` (CLI). | |
| **2. Export de la config** | `grid config backup create -file /tmp/sg_backup_$(date +%F).tgz` | |
| **3. Vérifier la compatibilité** | Consultez le **StorageGRID Upgrade Matrix** (PDF) → version cible 7.2. | |
| **4. Télécharger le bundle** | Depuis le **Support Site** → *StorageGRID 7.2* → **.tgz**. Copiez‑le sur le **Management Node** (`/var/tmp`). | |
| **5. Lancer l’upgrade (rolling)** | ```grid upgrade start --file /var/tmp/StorageGRID_7.2.tgz --mode rolling``` <br> Le système met à jour les **nodes** un par un, le service reste disponible. | |
| **6. Validation** | `grid health view` → **OK** <br> `grid version show` → 7.2.0 | |
| **7. Post‑upgrade** | Re‑activer le **Tiering** si désactivé, vérifier les **Buckets** (`grid bucket list`). | |

---

## 5. Commandes de vérification de santé (CLI)

| Domaine | Commande | Description |
|---------|----------|--------------|
| **ONTAP – général** | `cluster show` | État du cluster (nodes, quorum). |
| | `system node run -node <node> -command "system health"` | Santé détaillée du nœud. |
| | `vserver services s3 show` | Liste les S3‑vservers et. |
| | `volume show -fields size,used,percent-used` | Utilisation des volumes. |
| | `statistics statistics show -object volume -interval 60` | IOPS/latence par volume. |
| **ONTAP – S3** | `vserver services s3 bucket list -vserver <sv>` | Liste les buckets S3. |
| | `vserver services s3 bucket show -vserver <sv> -bucket <name>` | Détails (policy, versioning, lock). |
| **StorageGRID – général** | `grid health view` | Vue globale (Critical/Warning). |
| | `grid node list` | Statut de chaque node (up/down). |
| | `grid bucket list` | Tous les buckets S3. |
| | `grid capacity show` | Capacité totale, used, tiering. |
| | `grid performance show -interval 60` | IOPS, bande passante, latence. |
| **StorageGRID – réplication** | `grid replication policy list` | Politiques de réplication inter‑site. |
| **CVO – Cloud** | `cvo system show` (via **cvo-cli**) | État du service cloud. |
| | `cvo s3 bucket list` | Buckets exposés par le S3‑Gateway. |

> **Astuce** : toutes les commandes ci‑dessus peuvent être exécutées en **JSON** (`-output json`) pour les intégrer dans des scripts de monitoring (Prometheus, Splunk, etc.).

---

## 6. Administration de base – S3

### 6.1 ONTAP – créer un bucket S3

```bash
# 1. Créez un vserver dédié S3 (si besoin)
vserver create -vserver sv_s3 -subtype S3 -aggregate aggr1

# 2. Activez le service S3 sur ce vserver
vserver services s3 enable -vserver sv_s3

# 3. Créez le bucket
vserver services s3 bucket create -vserver sv_s3 -bucket mybucket \
   -policy private -versioning enabled -object-lock enabled
```

> **Remarque** : vous pouvez associer le bucket à un **volume** existant (`-volume <vol_name>`) ou laisser ONTAP créer un **volume dédié** automatiquement.

### 6.2 StorageGRID – créer un bucket S3 (UI)

1. **StorageGRID → Buckets → Create Bucket**  
2. Choisissez le **namespace** (ex. `sg‑prod`) → **Bucket name** (`my‑bucket`).  
3. Options : *Versioning*, *Object Lock*, *Lifecycle policy*, *Replication* (si multi‑site).  
4. Validez → le bucket apparaît dans la liste et est immédiatement accessible via l’endpoint S3 (ex. `https://sg-prod.example.com:443`).

### 6.3 Gestion de bucket via CLI (StorageGRID)

```bash
grid bucket create -name mybucket -namespace sg-prod \
   -versioning enabled -object-lock enabled
```

### 6.4 Gestion des politiques de réplication (StorageGRID)

```bash
# Crée une politique qui réplique le bucket mybucket vers le site secondaire
grid replication policy create -name repl_mybucket \
   -source-bucket mybucket -dest-site site2 \
   -mode async -schedule daily
```

### 6.5 Contr de **policy lifecycle** (ONTAP)

```bash
vserver services s3 bucket lifecycle create create \
   -vserver sv_s3 -bucket mybucket \
   -rule-name "TransitionToGlacier" \
   -filter-prefix "" \
   -transition-days 30 \
   -storage-class glacier
```

---

## 7. Bonnes pratiques spécifiques à l’usage S3 avec NetApp

| ✅ | Bonnes pratiques |
|---|-------------------|
| **Séparer les workloads** | Créez des **vservers S3** distincts pour chaque domaine (prod, dev, sauvegarde) afin de limiter l’impact d’un incident. |
| **Activer l’Object Lock** | Si vous avez des exigences de conformité (WORM), activez **Object Lock** dès la création du bucket (ONTAP ou StorageGRID). |
| **Utiliser le tiering** | Sur StorageGRID, activez le **Tiering Policy** vers le cloud (Azure Blob, AWS S3) pour les objets « cold ». |
| **Contrôler les accès** | Utilisez **IAM‑style policies** (ONTAP → `vserver services s3 policy create`) ou **Bucket ACL** pour limiter les accès au minimum. |
| **Surveiller les coûts** | Configurez des alertes **S3 Error Rate > 1 %** ou **latence > 100 ms** dans **NetApp OnCommand Insight** ou **Grafana** via les métriques REST. |
| **Sauvegarder les métadonnées** | Exportez régulièrement les **bucket policies** et **replication rules** (`grid bucket export` ou `vserver services s3 bucket export`). |
| **Plan de reprise‑roll** | Conservez toujours le **bundle de version précédente** (ex. ONTAP 9.11) pendant **30 jours** au cas où un rollback serait nécessaire. |

---

## 8. Ressources officielles (2026)

| Ressource | Lien |
|-----------|------|
| **ONTAP Documentation** | <https://docs.netapp.com/ontap-9/index.html> |
| **ONTAP Upgrade Guide (9.12)** | <https://www.netapp.com/support/upgrade-guides/ontap-9-12-upgrade-guide.pdf> |
| **StorageGRID Documentation** | <https://docs.netapp.com/storagegrid-7/index.html> |
| **StorageGRID Upgrade Guide (7.2)** | <https://www.netapp.com/support/upgrade-guides/storagegrid-7-2-upgrade.pdf> |
| **S3 API – ONTAP** | <https://docs.netapp.com/ontap-9/reference/s3.html> |
| **Object Lock & Governance Compliance** | <https://docs.netapp.com/ontap-9/feature/object-lock.html> |
| **My NetApp – Contracts & Entitlements** | <https://my.netapp.com> |
| **NetApp Community** | <https://community.netapp.com> |
| **NetApp Support (ticket)** | <https://support.netapp.com> |
| **NetApp YouTube – “S3 on ONTAP”** | <https://www.youtube.com/playlist?list=PL6C9C8F2F0E0A2F1A> |

---

## 9. Synth rapide avant votre première intervention S3

1. **Identifier le produit** (ONTAP S3 Native vs StorageGRID).  
2. **Vérifier le contrat** via My NetApp (ou demander au revendeur).  
3. **Exporter la configuration** (`system configuration backup` ou `grid config backup`).  
4. **Vérifier la santé** (`cluster show`, `grid health view`).  
5. **Valider la version** (`system version show`, `grid version show`).  
6. **Planifier l’upgrade** (si besoin) – rolling, fenêtre de 2 h.  
7. **Créer/mettre à jour les buckets** avec les bonnes options (Versioning, Object Lock, Lifecycle).  
8. **Configurer le monitoring** (SNMP/REST → Grafana, OnCommand Insight).  
9. **Documenter** le processus (ticket, notes de version, scripts).  

---
