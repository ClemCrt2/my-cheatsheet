## 1️⃣  PowerStore – Synthèse des bonnes pratiques (PDF h18241)

| Thème | Principes clés du guide | Pourquoi c’est important pour **media rush + VM / K8s** |
|-------|------------------------|--------------------------------------------------------|
| **Modèles et mode de déploiement** | • PowerStore T = appliance « bare‑metal » (unifiée ou block‑only). <br>• PowerStore X = VM‑on‑PowerStore (AppsON) – les ressources CPU/MEM sont partagées avec les VM hébergées. <br>• Un mode **unified** (T) fournit **bloc + file** simultanément. | Vous avez besoin à la fois de **volumes block** (VM, PVC) **et** de **NAS** (rushes, partage d’équ). Le mode **unified T** est donc le plus adapté. |
| **Dimensionnement matériel** | • Les modèles diffèrent par nombre de cœurs, RAM et NVRAM (NVMe). <br>• Les séries 500 → 9200 offrent un **escalonement linéaire** d’IOPS/MBps. <br>• Les modèles ≥ 5000 ont **4 NVRAM** (meilleur débit d’écriture large‑bloc). | Pour les flux vidéo (GB/s) choisissez au minimum un modèle **5000 T** ou supérieur ; les modèles ≥ 7000 off plus de bande passante et de capacité de cache. |
| **Résilience interne (DRE)** | • Dynamic Resiliency Engine (DRE) crée automatiquement des **sets de résilience** (single‑ ou double‑drive failure). <br>• Pas besoin de hot‑spare dédié ; l’espace de reconstruction est réparti sur le set. <br>• Installation minimale : ≥ 10 disques (single‑failure) ou ≥ 19 disques (double‑failure). | Vous bénéficiez d’une **reconstruction rapide** même pendant les gros ingest de rushes ; aucune configuration manuelle de hot‑spare n’est requise. |
| **SCM vs SSD** | • SCM (Storage Class Memory) = latence ultra‑faible, idéal pour **petits blocs** (IOPS). <br>• Mix SSD + SCM : SCM ≥ 5 % de la capacité totale → métadonnées accélérées. <br>• All‑SCM = uniquement pour workloads très IOPS‑intensifs. | Pour les **VM / PVC** (petits blocs) un **mix SSD + SCM** (≥ 5 %) réduit réduit‑> améli la latence des snapshots et des clones. |
| **Réseau – Fibre Channel** | • 2 fabriques redondantes, **≥ 2 chemins** par hôte → **4 ports** par appliance. <br>• Limite : 8 chemins/volume. <br>• Utiliser les vitesses les plus élevées supportées (32 Gb/s, 16 Gb/s). | Garantit la **haute disponibilité** et le débit nécessaire aux sauvegardes VM (iSCSI/FC) et aux réplications asynchrones. |
| **Réseau – Ethernet** | • LACP / VLTi entre switches, **jumbo frames MTU 9000** partout. <br>• Ports Ether : les 2 premiers ports de chaque carte sont les plus performants (PCIe 8‑lane). <br>• User‑defined link‑aggregation (2‑4 ports) disponible depuis OS 3.0 (NAS uniquement). | Permet d’atteindre **25 GbE** (ou **100 GbE** avec la carte 2‑port) pour iSCSI, NVMe/TCP et NAS, indispensable pour les gros fichiers vidéo. |
| **Séparation des trafics** | • Utiliser **des ports physiques distincts** pour : <br>   – NAS (SMB/NFS) <br>   – iSCSI / NVMe‑TCP <br>   – réplication asynchrone. <br>• Créer des **réseaux de stockage** dédiés (ex. « Data », « Replication »). | Évite la contention entre les **ingests vidéo** (NAS) et les **sauvegardes block** (iSCSI/NVMe) ou la réplication DR. |
| **Block – Affinité dynamique** | • Par défaut, PowerStore utilise **ALUA/ANA** et répartit les I/O sur le nœud « optimisé ». <br>• Depuis OS 2.1, **Dynamic Node Affinity** ré‑équilibre automatiquement la charge entre les deux nœuds tant que l’affinité n’est pas fixée manuellement. | Vous n’avez pas à « manuellement » forcer les volumes sur un nœud ; le système garde les performances équilibrées même pendant les pics d’ingestion. |
| **Politiques de performance** | • Chaque volume possède une **policy** : Low / Medium / High. <br>• En cas de contention, les volumes **Low** reçoivent moins de ressources CPU. | Réservez **High** pour les volumes qui hébergent les **rushes** ou les bases de données de post‑production ; **Medium** pour les VM de sauvegarde ; **Low** uniquement pour les archives peu critiques. |
| **File (NAS) – équilibrage** | • Un NAS server s’exécute sur **un seul nœud**. <br>• Créer **au moins 2 NAS servers** (un sur chaque nœud) pour exploiter les deux CPU. <br>• Les NAS servers peuvent être déplacés manuellement si un nœud devient saturé. | Vous obtenez le double de capacité de calcul pour les partages SMB/NFS qui serviront les équipes montage vidéo. |
| **Réduction de données** | • Zero‑detect, **compression** et **deduplication** sont **toujours activées**. <br>• En période de forte écriture, la dé‑duplication peut être différée. | Vous bénéficz d’un **gain d’espace** (≈ 30 % – 50 %) sans impacter les performances critiques des ingest. |
| **Snapshots & Thin‑clones** | • Toutes les ressources sont **thin‑provisioned**. <br>• Créer = pointeur → création instantanée, aucun impact de performance. <br>• Thin‑clone = copie lecture/écriture indépendante, idéale. | Idéal pour **VM / PVC backup** : crée snapshots puis thin‑clones pour les restaurations ou les tests. |
| **Réplication asynchrone** | • Support : volumes, volume‑groups, thin‑clones, NAS servers, file systems. <br>• Protocole : iSCSI ou **TCP propriétaire Dell** (OS 3.0). <br>• Recommandation : **ports dédiés** pour le trafic de réplication. | Vous pouvez répliquer les snapshots de VM et les dossiers media vers un site DR (PowerStore secondaire ou StorageGRID) avec un **RPO ≤ 15 min**. |
| **Mise à jour OS** | • Upgrade **non‑disruptif** (rolling). <br>• Pendant la mise à jour, **50 %** des ressources sont indisponibles → préférez les fenêtres de faible activité. <br>• Exécuter le **Pre‑Upgrade Health Check** avant le déploiement. | Planifiez les upgrades pendant les nuits ou les week‑ends pour ne pas impacter les ingestes de rushes. |
| **Configuration hôte** | • MPIO : timeout & path‑checker adaptés. <br>• iSCSI : queue‑depth, désactiver *delayed ACK*. <br>• FC : queue‑depth. <br>• Jumbo frames + flow‑control sur les cartes NIC. <br>• Désactiver **UNMAP** côté OS (déjà géré par PowerStore). | Sans ces réglages, vous perdez jusqu’à 30 % de bande passante et vous la latence des sauvegardes. |
| **VMware / vSphere** | • Intégration native : vVols, SRM, SnapMirror‑like (replication). <br>• Pour AppsON (X‑model) : augmenter la profondeur de la file iSCSI interne, activer Jumbo frames. | Si vous utilisez **vVols** pour les VM, vous bénéficiez d’une granularité de politique (QoS) et d’une réplication au niveau VM. |
| **Cluster PowerStore** | • Tous les appliances d’un même cluster doivent être **identiques** (modèle, capacité). <br>• Le cluster ne regroupe que des **T‑models** ou que des **X‑models** (pas de mix). <br>• Un volume reste attaché à **un seul appliance** à la fois ; migration possible mais doit une fenêtre de maintenance. | Pour la scalabilité, ajoutez des appliances **T‑unified** de même modèle. Les volumes block migrent automatiquement, les NAS restent sur l’appliance d’origine. |
| **Gestion de capacité** | • Surveiller **utilisation ≤ 90 %** par pool. <br>• Planifier l’ajout de nœuds dès **75 %** d’usage. <br>• Les snapshots **VHS** (Virtual Hot Spare) sont **activés par défaut** – ne pas les désactiver. | Vous évitez les ralentissements de snapshots et de réplication lorsque le système approche de la saturation. |

---

## 2️⃣  Bonnes pratiques **les plus pertinentes** pour votre cas d’usage

| Domaine | Pratique à appliquer | Comment la mettre en œuvre (CLI/GUI) | Impact sur votre flux |
|---------|----------------------|---------------------------------------|-----------------------|
| **Mode de déploiement** | **Unified T‑model** (bloc + file). | Lors de l’installation, choisir *Unified* dans le wizard. | Vous avez les deux accès (iSCSI/NVMe + NFS/SMB) pour les rushes et les VM/PVC. |
| **Choix du modèle** | **PowerStore 5000 T** minimum (ou 7000 T si le budget le permet). | Vérifier le modèle dans *PowerStore Manager → System → Overview*. | Offre suffisamment de **NVRAM** (4 drives) pour les gros flux vidéo et les écritures parallèles. |
| **Disques** | **Mix SSD + SCM** : ≥ 5 % de capacité en SCM. | Dans *System → Storage → Drives* créer un **SCM pool** et associer les métadonnées. | Latence très faible pour les snapshots / clones des VM et des PVC. |
| **Réseau – Fibre Channel** | **2 fabrices redondantes**, **≥ 2 chemins**/hôte, **32 Gb/s** si possible. | Configurer les HBA et les switches, vérifier dans *PowerStore → Network → FC*. | Garant : débit constant pour les sauvegardes VM (iSCSI/FC) et réplication DR. |
| **Réseau – Ethernet** | **LACP** + **jumbo MTU 9000** sur toutes les liaisons. <br>**Ports dédiés** : <br>  – NAS (SMB/NFS) <br>  – iSCSI/NVMe‑TCP <br>  – Replication. | Dans *PowerStore → Network → Ethernet* créer **Storage Networks** séparés et assigner les ports. | Évite la contention entre les ingest de rushes (NAS) et les sauvegardes block. |
| **Performance policy** | **High** pour les volumes qui hébergent les **rushes** (ex. NAS export `/ifs/media`). <br>**Medium** pour les volumes VM de sauvegarde. <br>**Low** uniquement pour archives froides. | `pstcli volume modify --name <vol> --performance-policy High` (ou via UI). | Priorise les IOPS/MBps où ils sont le plus critiques. |
| **Node affinity** | **Laisser l’affinité dynamique activée** (pas de réglage manuel). | Vérifier `pstcli node affinity show` – doit être *auto*. | Le système ré‑équilibre automatiquement la charge pendant les pics d’ingestion. |
| **NAS servers** | Créer **2 NAS servers** (un sur chaque nœud) pour les partages media. | *PowerStore Manager → File → NAS Servers → Add* (choisir le nœud). | Double la puissance de calcul disponible pour les accès SMB/NFS des équipes post‑production. |
| **Snapshots / Thin‑clones** | Utiliser **snapshots** pour chaque sauvegarde VM/PVC, puis créer **thin‑clones** pour les tests de restauration. | `pstcli volume snapshot create --name <vol> --snapshot-name vm‑snap-<date>` <br>`pstcli volume clone create --source-snapshot <snap> --clone-name vm‑clone‑<date>` | Restauration quasi‑instantanée, aucun impact de performance sur le volume source. |
| **Replication** | Configurer **asynchronous replication** vers un site DR (PowerStore secondaire ou StorageGRID). Utiliser **ports dédiés** pour le trafic de réplication. | *PowerStore Manager → Replication → Add* → choisir **TCP‑Dell** ou **iSCSI**, assigner le **Storage Network “Replication”**. | Rée : RPO ≈ 15 min, RTO ≤ 4 h pour les VM et les dossiers media. |
| **Data reduction** | **Laisser compression + deduplication activées** (défaut). Sur les périodes de forte écriture, surveiller le **deduplication throttling** via InsightIQ. | Aucun réglage requis; surveiller via *PowerStore → Analytics*. | Gain d’espace ≈ 30‑50 % sans pénalité notable sur les flux vidéo. |
| **Gestion** | **Jumbo MTU 9000** sur tous les commutateurs, cartes NIC et sur le cluster (Cluster MTU). | `pstcli network set --mtu 9000` (ou UI → Network → Settings). | Améliore le débit des gros fichiers (rushes) de ~5 %. |
| **Host‑side tuning** | - **MPIO** : timeout = 30 s, path‑checker = 5 s. <br>- **iSCSI** : queue‑depth ≥ 64, désactiver *delayed‑*. <br>- **FC** : queue‑depth ≥ 64. <br>- **NVMe/TCP** : `max‑queues` = 4. | Voir le **PowerStore Host Configuration Guide** pour chaque OS (Windows, Linux, ESXi). | Évite les goulots d’étranglement côté serveur. |
| **Upgrade** | Planifier les upgrades **hors‑heure de pointe** (nuit/week‑end). Exécuter le **Pre‑Upgrade Health Check** (`pstcli health check run`). | `pstcli upgrade start --type rolling` (non‑disruptif). | Minimisation de l’impact sur les ingest de rushes et les sauvegardes en cours. |
| **Capacité** | **Surveiller** l’utilisation ≤ 90 % par pool, activer les alertes **95 %** et **99 %**. <br>Planifier l’ajout d’appliance dès **75 %** d’usage. | *PowerStore → Alerts → Alerts* → créer règle email. | Prévention des ralentissements de snapshots ou de réplication. |

---

## 3️⃣  Checklist d’implémentation (à cocher)

| ✅ | Action |
|----|--------|
| 1 | Choisir **PowerStore T – Unified** (≥ 5000 T). |
| 2 | Installer **mix SSD + SCM** (SCM ≥ 5 % du total). |
| 3 | Configurer **2 fabrices FC** redondantes, **≥ 2 chemins**/hôte, **32 Gb/s**. |
| 4 | Configurer **Ethernet LACP** avec **MTU 9000**; créer **3 Storage Networks** : NAS, Block, Replication. |
| 5 | Créer **2 NAS servers** (un par nœud) pour `/ifs/media`. |
| 6 | Appliquer **Performance Policy = High** aux volumes media, **Medium** aux volumes VM/PVC. |
| 7 | Laisser **Dynamic Node Affinity** activée (pas de réglage manuel). |
| 8 | Activer **Compression‑compression + deduplication** (défaut). |
| 9 | Configurer **Snapshots** et **Thin‑clones** pour chaque sauvegarde VM/PVC. |
|10| Mettre en place **Replication‑asynchrone** vers site DR sur ports dédiés. |
|11| Appliquer les **tuning host** (MPIO, iSCSI queue‑depth, FC queue‑depth, NVMe/TCP queues). |
|12| Activer les **alertes** (95 %/99 %) et surveiller via **InsightIQ**. |
|13| Planifier les **upgrades OS** pendant les fenêtres de faible activité. |
|14| Dès **Capacity planning** : ajouter un appliance dès 75 % d’utilisation. |
|15| Documenter les **paramètres réseau** (VLAN, MTU, LACP) et les **ports dédiés**. |

---

## 4️⃣  Exemple de flux de travail complet (PowerStore + PowerScale + NetApp)

```
┌─────────────────────┐          ┌─────────────────────┐
│   Serveurs de capture│          │   Hosts ESXi / K8s │
│   (NAS → SMB/NFS)   │          │   (iSCSI/NVMe‑TCP) │
└───────┬─────────────┘          └───────┬─────────────┘
        │  (NAS)                         │  (Block)
        ▼                                 ▼
   ┌───────────────┐               ┌───────────────┐
   │ PowerStore    │               │ PowerStore    │
   │ (Unified T)   │               │ (Unified T)   │
   │  - NAS Server │               │  - Volumes    │
   │  - SSD+SCM    │               │  - vVols      │
   └───────┬───────┘               └───────┬───────┘
           │                               │
   ┌───────▼───────┐               ┌───────▼───────┐
   │  Réplication  │               │  Snapshots/   │
   │  (TCP Dell)   │               │  Thin‑clones  │
   └───────┬───────┘               └───────┬───────┘
           │                               │
   ┌───────▼───────┐               ┌───────▼───────┐
   │  Site DR     │               │  Velero → S3  │
   │ (PowerStore  │               │ (ONTAP S3)    │
   │  ou Grid)    │               │               │
   └──────────────┘               └───────────────┘
```

*Les rushes sont stockés directement sur le **NAS** du PowerStore (SSD + SCM, policy = High).  
*Les VM et les PVC sont servis via **iSCSI/NVMe‑TCP** (policy = Medium).  
*Velero utilise le bucket **ONTAP S3** (ou le bucket StorageGRID) pour sauvegarder les snapshots / PVC.  
*La réplication asynchrone protège les deux types de données sur le site DR.*

---

## 5️⃣  Ressources complémentaires

| Document | Lien |
|----------|------|
| **PowerStore Info Hub** (guide d’installation, configuration réseau, host‑tuning) | <https://www.dell.com/support/kbdoc/en-us/000196877/powerstore-overview> |
| **PowerStore Host Configuration Guide** (MPIO, iSCSI, FC, NVMe/TCP) | <https://www.dell.com/support/kbdoc/en-us/000196878/powerstore-host-configuration> |
| **PowerStore CLI (PSTCLI) Reference** | <https://www.dell.com/support/kbdoc/en-us/000196879/powerstore-cli-reference> |
| **PowerStore InsightsIQ** (monitoring & capacity)) | <https://www.dell.com/support/kbdoc/en-us/000196880/powerstore-insightiq> |
| **PowerStore Replication (TCP‑Dell) Whitepaper** | <https://www.dell.com/support/kbdoc/en-us/000196881/powerstore-replication> |

---

### 🎯  En résumé
- **Déploiement** : PowerStore T unified, SSD + SCM, 2 fabric FC + LACP Ethernet, ports séparés (NAS / block / replication).  
- **Politiques** : High → media, Medium → VM/PVC, Low → archives.  
- **Snapshots/Thin‑clones** + **Replication** assurent les RPO/RTO requis.  
- **Réseau** : Jumbo MTU 9000, LACP, chemins multiples, séparation du trafic.  
- **Host tuning** : MPIO, queue‑depth, désactiver UNMAP, activer Jumbo frames.  
- **Surveillance** : alerts 95 %/99 %, InsightIQ, capacity planning à 75 %.  

