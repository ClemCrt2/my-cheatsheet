## Programme de suivi & reporting du stockage  
**Environnement** – PowerScale (OneFS) & PowerStore (OS 7.x – 2026)  
**Objectif** – Assurer la disponibilité, la performance et la capacité du parc de stockage, détecter les besoins d’évolution et fournir à la direction une visibilité claire (coûts, risques, opportunités) à chaque horizon (quotidien, hebdomadaire, semestriel).

---

## 1.  Cadence générale & livrables

| Cadence | Livrable | Diffusion | Responsable(s) |
|---------|----------|-----------|----------------|
| **Quotidien** | **Daily Health Dashboard** (PDF / HTML) – état du cluster, alertes critiques, capacité résiduelle, I/O criticales. | Email → Direction IT + Team Ops | Storage Engineer / Shift Lead |
| **Hebdomadaire** | **Weekly Operations Report** – synthèse des incidents, évolution de la capacité, performance moyenne, actions d’optimisation, demandes d’achat (disques, licences). | Email → Direction IT, CFO (budget), CIO. | Storage Lead + Capacity Planner |
| **Semestriel** | **Strategic Storage Review** – analyse de tendance 6 mois, prévision 12‑24 mois, recommandations d’investissements (scale‑out, Cloud‑tiering, DR), tableau de ROI. | Présentation PowerPoint + Executive Summary (1‑page) | Storage Architect + Finance Business Partner |

> **Note** : les livrables sont produits à partir de scripts automatisés (Python / Bash + jq) qui interrogent les APIs REST de PowerScale et PowerStore. Les fichiers sont stockés dans un dépôtre Git (ex : *storage‑reports*), versionnés et archivés 12 mois.

---

## 2.  Check quotidien (≈ 30 min)

| N° | Action | Commande / API | KPI / Rés | Seuil d’alerte | Commentaire |
|---|---|---|---|---|---|
| 1 | **État global du cluster** | `isi status -j` (PowerScale) <br> `GET /api/v1/cluster` (PowerStore) | `cluster_status` (OK/WARN/CRIT) | CRIT → ticket Urgent | Ajoute le code couleur au dashboard. |
| 2 | **Alertes critiques** | `isi events list --severity critical -j` <br> `GET /api/v1/alert?severity=critical` | Nombre d’alertes critiques | > 0 → escalade immédiate | Inclure le lien vers le ticket ServiceNow. |
| 3 | **Capacité résiduelle** | `isi storage pool list -j` → `free` <br> `GET /api/v1/pools` → `capacity.free_bytes` | % libre par pool | < 10 % → **Warning** <br> < 5 % → **Critical** | Si < 5 % déclencher le workflow “Capacity Request”. |
| 4 | **Utilisation CPU / Mémoire** | `isi status -d -j` → `cpu.utilization` / `mem.utilization` <br> `GET /api/v1/cluster/metrics` | % CPU, % RAM | > 80 % (CPU) ou > 85 % (RAM) → **Warning** | Vérifier les jobs de backup ou de snapshot qui tournent. |
| 5 | **IOPS / Throughput moyen (last 5 min)** | `isi statistics view -j --metrics iops,throughput` <br> `GET /api/v1/metrics?interval=5m` | IOPS, MB/s | > 80 % du seuil de capacité → **Warning** | Utiliser les seuils définis par SLA (ex : 30 k IOPS). |
| 6 | **État des SnapMirror / Replication** | `isi snapmirror list -j` <br> `GET /api/v1/replication/pairs` | % de réplicas “healthy” | < 95 % → **Warning** | Vérifier les lags (`last_sync_time`). |
| 7 | **Vérification du service NFS/SMB/S3** | `isi nfs exports list -j` <br> `isi smb shares list -j` <br> `GET /api/v1/s3/buckets` | Nombre d’exports actifs | Aucun export → **Critical** | Confirmer que les services clients sont up. |
| 8 | **Contrôle des licences** | `isi licenses status -j` <br> `GET /api/v1/licenses` | Licences actives / expirées | Expiration < 30 j → **Warning** | Créer ticket “Renew licence renewal”. |
| 9 | **Snapshot / Quota** | `isi snapshots list -j` <br> `isi quota list -j` | Nombre de snapshots > 30 j | > 30 j → **Warning** (nettoyage) | Automatiser la purge des snapshots anciens. |
| 10| **Sauvegarde de la configuration** | `isi configuration export -f /ifs/backup/daily_cfg_$(date +%F).tgz` | Succès / échec | Échec → **Critical** | Conserver 7 copies. |

**Livrable** – Script `daily_report.sh` → génère `daily_health_YYYYMMDD.html` + `daily_dashboard_YYYYMMDD.pdf`.  
**Envoi** – 07 h00 (heure locale) via `mailx` ou **Microsoft Teams** webhook.

---

## 3.  Check hebdomadaire (≈ 2 h)

| N° | Action | Méthode | KPI / seuil | Détails du rapport |
|---|---|---|---|---|
| 1 | **Analyse des incidents** | Export ServiceNow tickets (last 7 j) + `isi events` | Nombre d’incidents, MTTR | Tableau incidents, cause racine, actions correctives. |
| 2 | **T de capacité** | Historique `free_bytes` (PowerScale) + `capacity.free_bytes` (PowerStore) sur 30 j | % de croissance jour‑à‑jour | Graphique trend, projection 3 mois (linear + saisonnalité). |
| 3 | **Performance moyenne** | `isi statistics view -j --metrics iops,throughput,latency` (avg 7 j) <br> PowerStore `GET /api/v1/metrics?interval=7d` | IOPS, MB/s, latency (ms) | Comparaison avec SLA (ex : latence < 5 ms). |
| 4 | **Utilisation des quotas** | `isi quota report -j` | % de quotas dépassés | Liste des utilisateurs/partages qui approchent leurs limites. |
| 5 | **Efficacité de la déduplication / compression** (PowerStore uniquement) | `GET /api/v1/filesystems/{fs_id}/deduplication` | Ratio déduplication | > 2 : 1 → bonnes économies, sinon vérifier les politiques. |
| 6 | **État des réplications inter‑site** | `isi snapmirror list -j` + PowerStore `GET /api/v1/replication/policies` | Lag (minutes) | Lag > 30 min → alerter DR‑owner. |
| 7 | **Vérification de la santé du hardware** | `isi hardware health list -j` <br> PowerStore `GET /api/v1/hardware` | Disques “failed”, PSU, fans | Tout composant “critical” → ticket HW. |
| 8 | **Consommation de licences** | `isi licenses list -j` + PowerStore licences API | % de capacité utilisée vs. licence | > 85 % → préparer demande d’extension. |
| 9 | **Back‑up / Restore** | Vérifier les jobs de backup (PowerScale Snapshots, PowerStore Protection) – succès/échec | % de jobs réussis | < 95 % → escalade. |
|10| **Revue des demandes de configuration** | `isi configuration diff` (comparaison avec backup de la semaine précédente) | Nombre de changements changements | Documenter les changements (ex: ajout d’un pool). |

**Livrable** – `weekly_report_YYYYWW.pdf` (4‑pages) :

1. **Executive Summary** (2 paragraphes)  
2. **KP santé & incidents**  
3. **Capacité & prévisions** (graphes)  
4. **Recommandations d’achat / optimisation** (disques, licences, upgrade).  

**Envoi** – Tous les lundis 09 h00 à la Direction IT, CFO, VP Ops.

---

## 4.  Check semestriel (≈ 1 jour)

### 4.1  Étapes de la revue (6 mois)

| Phase | Action | Outils / Sources | Output |
|---|---|---|---|
| **A – Collecte de données historiques** | Export complet des métriques (capacity, IOPS, latency, snapshots, licences) sur les 180 jours. | PowerScale CLI `isi statistics export --format csv --start <date‑180>` <br> PowerStore API `GET /api/v1/metrics?start=<date‑180>` | CSV / Parquet (Data Lake). |
| **B – Analyse de tendance** | Modélisation ARIMA ou Prophet pour capacité, IOPS, latence. | Python pandas + fbprophet | Prévision 12 mois (capacité, IOPS). |
| **C – Évaluation de la conformité SLA** | Comparaison mensuelle des KPI avec les SLA contractuels. | Tableau de bord PowerBI / Grafana | Score SLA (ex : 99,5 % temps‑up). |
| **D – Bilan de l’efficacité des politiques** | Déduplication, compression, tiering (PowerStore) ; SmartPools, tiering (PowerScale). | `isi storage tier list -j` ; PowerStore `GET /api/v1/tiering` | Ratio d’économie (TB économisés). |
| **E – Analyse de la dette technique** | Inventaire des objets “legacy” (NFS v3, SMB 1.0, snapshots > 180 j, pools non‑optimisés). | Scripts de recherche de version/protocole. | Liste d’actions de modernisation. |
| **F – Scénarios de croissance** | 3 scénarios (Conservateur, Baseline, Aggressif) – impact sur CAPEX/OPEX. | Excel / PowerBI “What‑If”. | Recommandation d’investissement. |
| **G – Plan de migration / évolution** | - **Scale‑out** (ajout nœuds PowerScale ou nouvelles appliances PowerStore). <br>- **Cloud‑tiering** (PowerScale CloudPools, PowerStore CloudSnap). <br>- **DR/BCP** (étendre SnapMirror ou PowerStore Replication vers un site secondaire ou vers le Cloud). | Diagrammes d’architecture. | Feuille de route 12‑24 mois. |
| **H – Business case** | Calcul ROI (TCO 3 ans) – économies grâce à la déduplication, réduction des licences, migration vers le Cloud. | Modèle financier (Excel). | Slide “Investment Recommendation”. |

### 4.2  Livrables semestriels

| Document | Contenu | Diffusion |
|---|---|---|
| **Strategic Storage Review – Executive Deck** (12 slides) | 1️⃣ Vision 2026‑2028 <br>2️⃣ KPI actuels <br>3️⃣ Trend capacité & performance <br>4️⃣ Analyse de la dette technique <br>5️⃣ Scénarios d’évolution <br>6️⃣ Recommandations d’achat (hardware, licences, Cloud) <br>7️⃣ ROI & budget prévisionnel <br>8️⃣ Plan d’action (Q3‑Q4 2026) | Présentation au Comité de Direction (CIO, CFO, VP Ops). |
| **Detailed Technical Annex** (PDF ≈ 30 pages) | Tables de données, scripts, logs d’audit, résultats de simulation. | Partagé avec l’équipe d’architecture, audit interne. |
| **Roadmap 12‑24 mois** (Gantt) | Jalons d’ajout de nœuds, de migration Cloud, de fin de licence, de tests DR. | Confluence / Project‑Management tool (Jira / MS Project). |

---

## 5.  Processus de remontée des besoins & projets

| Niveau | Déclencheur | Action | Responsable | Délai |
|--------|-------------|--------|-------------|-------|
| **A – Observation quotidienne** | Capacité < 10 % ou alerte critique | Créer ticket **“Capacity‑Request”** dans ServiceNow (type = “Request”). | Storage Engineer | < 2 h |
| **B – Analyse hebdomadaire** | Croissance > 5 % / mois ou plusieurs alertes similaires | Proposer **“Purchase Order”** (disques SSD/NVMe, licences). | Capacity Planner + Finance Business Partner | 1 semaine |
| **C – Revue semestrielle** | Projection capacité > 80 % dans 12 mois **ou** besoin de nouvelles fonctionnalités (ex : CloudPools, S3 gateway) | Rédiger **Business Case** (ROI, TCO) et le soumettre au Comité d’Investissement. | Storage Architect + PMO | 2 semaines avant la réunion du comité |
| **D – Décision de la direction** | Validation du budget | Lancer le **procurement** (RFQ, PO). | Procurement Lead | Selon procédure interne (30‑45 j). |
| **E – Implémentation** | Livraison matériel / licences | Plan de déploiement (design, test, cut‑over). | Project Manager + Engineering Team | Selon le projet (3‑6 mois). |

---

## 6.  Modèle de tableau de suivi des besoins (exemple)

| ID | Service | Type de besoin | Description | Impact (SLA) | Volume actuel | Projection prévision 12 mois | Priorité | Statut | Responsable | Date cible |
|----|---------|----------------|-------------|--------------|--------------|------------------------|----------|--------|-------------|------------|
| RQ‑2026‑001 | PowerScale – NFS | Capacity | Ajout 20 TB de SSD dans le pool **fast_pool** | Risque de saturation > 5 % | 4,2 PB used / 4,8 PB total | 5,5 TB | Haute | En cours | Storage Lead | 15‑Oct‑2026 |
| RQ‑2026‑002 | PowerStore – S3 | Feature | Déploiement de **CloudPools** vers Azure Blob (tiering) | Amélioration des coûts de cold froid | 12 TB d’objets “cold” | 30 TB | Moyenne | En attente de budget | Architecte | 30‑Nov‑2026 |
| RQ‑2026‑003 | PowerScale – DR | Infrastructure | Ajout d’un site secondaire (10 GbE) pour SnapMirror | RTO < 4 h, RPO < 30 min | 2 PB répliqué | 2,5 PB | Critique | Planifié | PMO | 01‑Mar‑2027 |

---

## 7.  Automatisation – Exemple de pipeline (GitLab CI)

```yaml
stages:
  - daily
  - weekly
  - monthly
  - semestrial

variables:
  POWERSTORE_URL: "https://ps01.corp.local"
  POWERSCALE_HOST: "pscale01.corp.local"
  REPORTS_REPO: "git@gitlab.corp.local:storage/reports.git"

daily_report:
  stage: daily
  script:
    - ./scripts/daily_report.sh > daily_report.html
    - pandoc daily_report.html -o daily_report.pdf
    - git clone $REPORTS_REPO repo
    - cp daily_report.* repo/daily/
    - cd repo && git add . && git commit -m "Daily report $(date +%F)" && git push
    - ./scripts/mail_report.sh daily_report.pdf "Daily Storage Dashboard – $(date +%F)"
  only:
    - schedules   # déclenché par le scheduler chaque jour à 06:30

weekly_report:
  stage: weekly
  script:
    - ./scripts/weekly_report.sh > weekly_report.pdf
    - git clone $REPORTS_REPO repo
    - cp weekly_report.pdf repo/weekly/
    - cd repo && git add . && git commit -m "Weekly report $(date +%V)" && git push
    - ./scripts/mail_report.sh weekly_report.pdf "Weekly Storage Operations – Week $(date +%V)"
  only:
    - schedules   # chaque lundi 08:00

semestral_review:
  stage: semestrial
  script:
    - ./scripts/6month_analysis_analysis.py   # exécute le modèle de prévision
    - ./scripts/generate_review.sh    # crée crée PowerPoint + annexes
    - git clone $REPORTS_REPO repo
    - cp review_* repo/semestral/
    - cd repo && git add . && git commit -m "Semestral review $(date +%Y%M)" && git push
    - ./scripts/mail_review.sh review_deck.pdf "Strategic Storage Review – H1 $(date +%Y)"
  only:
    - schedules   # 1er jour du semestre (01‑Jan, 01‑Jul)
```

> **Avantages** : toute la chaîne (collecte → analyse → livrable → diffusion) est versionnée, traçable et ré‑exécutable.

---

## 8.  Tableau récapitulatif de la charge de travail

| Cadence | Temps estimé (person‑hour) | Fréquence | Total h/mois |
|---------|---------------------------|-----------|--------------|
| **Daily** | 0,5 h (script + revue) | 22 j/mois | 11 h |
| **Weekly** | 2 h (analyse + rédaction) | 4 j/mois | 8 h |
| **Monthly** (rapports de conformité) | 1 h | 1 j/mois | 1 h |
| **Semestriel** | 40 h (collecte, modélisation, présentation) | 2 x/yr | 6,7 h/mo |
| **Ad‑hoc** (incidents, projets) | variable | – | – |

**Ressource dédiée** – **1 FTE Storage Engineer** (≈ 30 h/mois) + **0,5 FTE Capacity Planner** (pour les prévisions). Le reste (développement de scripts, présentations) peut être partagé avec l’équipe d’architecture.

---

## 9.  Mod de template de **Executive Summary** (à ins/coller dans le rapport hebdo)

```
**Executive Summary – Semaine 32 (08‑Nov‑2026)**  

- **Disponibilité** : 99,97 % (objectif 99,95 %). Aucun incident majeur.  
- **Capacité** : 4,12 PB utilisés / 5,00 PB totaux (17 % libre). Projection 12 mois → 4,9 PB, marge < 5 % de marge.  
- **Performance** : Latence moyenne 3,2 ms (cible ≤ 5 ms). IOPS moyen 28 k (cible 30 k).  
- **Alertes** : 2 alertes critiques résolues (disque #12 défectueux, remplacé).  
- **Licences** : Licence “SmartPools – Premium” expire le 12‑Jan‑2027 (90 jours). Action : renouvellement prévu Q4.  
- **Recommandations** :  
  1️⃣ Approvisionnement de 20 TB SSD (pool “fast_pool”) – budget 45 k USD.  
  2️⃣ Étude de CloudPools Azure (coût estimé 0,018 USD/GB/mois).  
  3️⃣ Pilotage d’un nouveau PowerStore 8‑node pour le workload “AI‑training”.  
```

---

## 10.  Points d’attention spécifiques

| Sujet | Risque | Mitigation |
|---|---|---|
| **Panne d’un nœud** | Perte de capacité & réplication. | HA → minimum 3 nœuds, alertes de santé, plan de remplacement matériel. |
| **Expiration de licence** | Service S. | Suivi mensuel des dates d’expiration, ticket pré‑emptif 60 j avant. |
| **Croissance inattendue (ex : logs)** | Saturation rapide. | Impl alertes de capacité à 15 % libre, automatisation de “scale‑out” via API. |
| **Obsolescence du firmware** | Bugs, perte de support. | Calendrier de mise à jour trimestriel (test en lab). |
| **Non‑conformité aux politiques de rétention** | Risque juridique. | Audits mensuels des snapshots & politiques S3 lifecycle. |
| **Coût Cloud** | Dépassement budget. | Simulations de coût CloudPools avant activation, alertes de consommation > 10 % du budget. |

---

## 11.  Checklist de mise en place (à réaliser avant le **)

1. **Créer les comptes API** (PowerScale `admin` + PowerStore `admin`) avec **role = read‑write** et **token** stocké dans un coffre (HashiCorp Vault).  
2. **Déployer les scripts** (`daily_report.sh`, `weekly_report.sh`, `6month_analysis.py`) sur un serveur de monitoring dédié (ex : `monitor01`).  
3. **Configurer le job scheduler** (cron ou GitLab CI) selon le tableau CI ci‑dessus.  
4. **Paramétrer les alertes** dans **ServiceNow** (cré → Ticket) et **Microsoft Teams** (webhook).  
5. **Valider les modèles de prévision** (test sur 3 mois de données historiques).  
6. **Former l’équipe** (atelier 2 h – “Lire le Daily Dashboard”).  
7. **Documenter** le processus dans Confluence (page “Storage Reporting SOP”).  

---

### 🎯 Résultat attendu

- **Visibilité continue** sur la santé du stockage (KPIs à jour chaque jour).  
- **Détection précoce** des besoins d’extension (capacité, licences, performance).  
- **Reporting structuré** qui alimente la prise de décision stratégique (budget, projets Cloud, DR).  
- **Processus automatisé** qui minimise le temps passé à collecter les données et maximise la fiabilité des informations transmises à la direction.  

--- 
