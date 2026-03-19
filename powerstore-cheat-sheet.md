# PowerScale (OneFS) – Cheat‑Sheet d’Administration de Base  
**Version 2026‑03 (OneFS 9.5.x – 9.7.x)** – les commandes ci‑dessous fonctionnent sur toutes les versions récentes (8.x → 9.x). Elles sont toutes exécutées depuis le **shell d’un nœud de gestion** (généralement le « cluster‑admin » ou tout nœud avec le rôle *cluster‑admin*).  

> **Rappel** :  
> - Le préfixe **`isi`** lance l’utilitaire de gestion OneFS.  
> - La plupart des sous‑commandes acceptent les options `-h` / `--help` pour afficherd l’aide détaillée.  
> - Les actions qui modifient la configuration (création de pools, modification de licences, upgrade…) exigent le rôle **`cluster_admin`** ou **`root`**.  
> - Les sorties sont en **JSON** (`-j`) ou **texte** (`-i`), ce qui facilite le scripting.  

---

## 1.  Découvrir la version et l’état du cluster

| Commande | Description | Exemple |
|---|---|---|
| `isi status` | Rés état global du cluster (OK / WARN / CRIT) + version OneFS. | `isi status` |
| `isi version` | Affiche la version exacte du logiciel OneFS et le build. | `isi version` |
| `isi status -d` | Détails sur chaque nœud (CPU, RAM, disque, état). | `isi status -d` |
| `isi hardware info` | Inventaire matériel (CPU, mémoire, contrôleurs, cartes réseau). | `isi hardware info` |
| `isi licenses list` | Liste les licences installées (capacity, protocoles, etc.). | `isi licenses list` |
| `isi cluster node list -i` | Liste les nœuds du cluster avec leurs adresses IP. | `isi cluster node list -i` |

---

## 2.  Gestion des nœuds (ajout / retrait / mise à jour)

| Action | Commande | Exemple |
|---|---|---|
| **Ajouter un nœud** | `isi cluster node add <IP‑node> --role <role>` | `isi cluster node add 10.0.1.12 --role storage` |
| **Retirer un nœud** | `isi cluster node remove <node‑name>` | `isi cluster node remove node03` |
| **Mettre en maintenance** (drain) | `isi cluster node maintenance start <node‑name>` | `isi cluster node maintenance start node02` |
| **Sortir de maintenance** | `isi cluster node maintenance stop <node‑name>` | `isi cluster node maintenance stop node02` |
| **Redémarrer un nœud** | `isi cluster node reboot <node‑name>` | `isi cluster node reboot node01` |
| **Mise à jour du firmware/OS** | `isi upgrade start --image <path‑to‑image>` | `isi upgrade start --image /ifs/upgrade/OneFS_9.5.0.1.iso` |
| **Vérifier l’état de mise à jour** | `isi upgrade status` | `isi upgrade status` |

---

## 3.  Gestion du stockage (pools, disques, agrégats)

| Action | Commande | Exemple |
|---|---|---|
| **Lister les pools** | `isi storage pool list -i` | `isi storage pool list -i` |
| **Créer un pool** | `isi storage pool create <pool‑name> --node <node‑list> --disk <disk‑list>` | `isi storage pool create pool01 --node node01,node02 --disk 0:0:0,0:0:1` |
| **Supprimer un pool** | `isi storage pool delete <pool‑name>` | `isi storage pool delete pool01` |
| **Lister les disques** | `isi storage disk list -i` | `isi storage disk list -i` |
| **V le détail d’un disque** | `isi storage disk view <disk‑id> -i` | `isi storage disk view 0:0:0 -i` |
| **Remplacer un disque défectueux** | `isi storage disk replace <old‑id> <new‑id>` | `isi storage disk replace 0:0:0 0:0:2` |
| **Vérifier la santé des agrégats** | `isi storage aggregate list -i` | `isi storage aggregate list -i` |
| **Créer un agrégat** (rare, généralement via pool) | `isi storage aggregate create <agg‑name> --pool <pool‑name> --disk <disk‑list>` | `isi storage aggregate create agg01 --pool pool01 --disk 0:0:0,0:0:1` |
| **Défragmenter un agrégat** | `isi storage aggregate scrub start <agg‑name>` | `isi storage aggregate scrub start agg01` |

---

## 4.  Gestion des espaces de fichiers (IFS)

| Action | Commande | Exemple |
|---|---|---|
| **Lister les répertoires racine** | `isi file list /ifs` | `isi file list /ifs` |
| **Créer un répertoire** | `isi file mkdir /ifs/data` | `isi file mkdir /ifs/data` |
| **Supprimer un répertoire** | `isi file rmdir /ifs/old` | `isi file rmdir /ifs/old` |
| **Copier un fichier** | `isi file cp /ifs/src/file.txt /ifs/dst/file.txt` | `isi file cp /ifs/a/b.txt /ifs/c/d.txt` |
| **Déplacer / renommer** | `isi file mv /ifs/oldname /ifs/newname` | `isi file mv /ifs/tmp /ifs/archive` |
| **Afficher les propriétés** | `isi file stat /ifs/path` | `isi file stat /ifs/projects` |
| **Changer les permissions POSIX** | `isi file chmod 770 /ifs/projectX` | `isi file chmod 770 /ifs/projectX` |
| **Changer le propriétaire** | `isi file chown <user>:<group> /ifs/path` | `isi file chown alice:dev /ifs/projectX` |
| **Activer la réplication (SnapMirror)** | `isi snapmirror create <src‑path> <dst‑path> --schedule <cron>` | `isi snapmirror create /ifs/projX /ifs/backup/projX --schedule "0 2 * * *"` |
| **Lister les SnapMirrors** | `isi snapmirror list -i` | `isi snapmirror list -i` |
| **Forcer une réplication** | `isi snapmirror sync <id>` | `isi snapmirror sync 12345` |

---

## 5.  Gestion des quotas

| Action | Commande | Exemple |
|---|---|---|
| **Lister les quotas** | `isi quota list -i` | `isi quota list -i` |
| **Créer un quota** (par ré ou répertoire) | `isi quota create --type <user|directory> --path <path> --size <size>` | `isi quota create --type directory --path /ifs/projects --size 5TB` |
| **Modifier un quota** | `isi quota modify <quota‑id> --size <new‑size>` | `isi quota modify 101 --size 6TB` |
| **Supprimer un quota** | `isi quota delete <quota‑id>` | `isi quota delete 101` |
| **Afficher l’utilisation d’un quota** | `isi quota report -i` | `isi quota report -i` |

---

## 6.  Gestion des partages (NFS, SMB, FTP, S3)

| Protocole | Commande | Exemple |
|---|---|---|
| **Lister les exports NFS** | `isi nfs exports list -i` | `isi nfs exports list -i` |
| **Créer un export NFS** | `isi nfs exports create /ifs/data --clients <IP‑range> --permissions rw` | `isi nfs exports create /ifs/data --clients 10.0.0.0/24 --permissions rw` |
| **Supprimer un export NFS** | `isi nfs exports delete <export‑id>` | `isi nfs exports delete 12` |
| **Lister les partages SMB** | `isi smb shares list -i` | `isi smb shares list -i` |
| **Créer un partage SMB** | `isi smb shares create <share‑name> /ifs/data --description "Projet X"` | `isi smb shares create projX /ifs/projectX --description "Share for Project X"` |
| **Supprimer un partage SMB** | `isi smb shares delete <share‑name>` | `isi smb shares delete projX` |
| **Lister les zones S3 (RGW)** | `isi s3 zones list -i` | `isi s3 zones list -i` |
| **Créer un bucket S3** | `isi s3 buckets create <bucket‑name> --zone <zone‑name>` | `isi s3 buckets create logs --zone default` |
| **Définir une politique de cycle de vie** | `isi s3 bucket lifecycle set <bucket‑name> --policy <json‑file>` | `isi s3 bucket lifecycle set logs --policy /ifs/policies/logs.json` |
| **Lister les utilisateurs S3** | `isi s3 users list -i` | `isi s3 users list -i` |
| **Créer un utilisateur S3** | `isi s3 users create <user‑name> --access-key <key> --secret-key <secret>` | `isi s3 users create alice --access-key AKIA… --secret-key wJalrXU…` |

---

## 7.  Gestion des utilisateurs / groupes (LDAP/AD, local)

| Action | Commande | Exemple |
|---|---|---|
| **Lister les utilisateurs** | `isi auth users list -i` | `isi auth users list -i` |
| **Créer un utilisateur local** | `isi auth users create <user> --password <pwd>` | `isi auth users create bob --password S3cure!` |
| **Supprimer un utilisateur** | `isi auth users delete <user>` | `isi auth users delete bob` |
| **Lister les groupes** | `isi auth groups list -i` | `isi auth groups list -i` |
| **Créer un groupe local** | `isi auth groups create <group>` | `isi auth groups create devops` |
| **Ajouter un utilisateur à un groupe** | `isi auth groups adduser <group> <user>` | `isi auth groups adduser devops alice` |
| **Configurer l’auth LDAP LDAP/AD** | `isi auth providers add --type ldap --name AD --host <dc> --domain <domain> --bind-dn <user> --bind-pw <pwd>` | `isi auth providers add --type ldap --name AD --host dc01.corp --domain corp.local --bind-dn "CN=svc,OU=Service,DC=corp,DC=local" --bind-pw Secret123` |
| **Vérifier la connexion**** | `isi auth providers test --name AD` | `isi auth providers test --name AD` |

---

## 8.  Surveillance & alertes

| Action | Commande | Exemple |
|---|---|---|
| **Afficher le tableau de bord** | `isi statistics view` | `isi statistics view` |
| **Lister les alarmes** | `isi events list -i` | `isi events list -i` |
| **Filtrer les alarmes par sévérité** | `isi events list --severity <critical|warning|info> -i` | `isi events list --severity critical -i` |
| **Exporter les métriques Prometheus** | `isi statistics export --format prometheus` | `isi statistics export --format prometheus > /etc/prometheus/onefs.prom` |
| **Configurer les notifications e‑mail** | `isi alerts email set --recipients <mail>` | `isi alerts email set --recipients admin@corp.com` |
| **Créer** | `isi alerts smtp set --server <smtp‑host> --port <port>` | `isi alerts smtp set --server smtp.corp.com --port 587` |

---

## 9.  Sauvegarde / restauration (Snapshots)

| Action | Commande | Exemple |
|---|---|---|
| **Créer un snapshot** (sur un répertoire) | `isi snapshots create /ifs/data --name <snap‑name>` | `isi snapshots create /ifs/projectX --name pre‑upgrade` |
| **Lister les snapshots** | `isi snapshots list -i` | `isi snapshots list -i` |
| **Restaurer un snapshot** (vers le même chemin) | `isi snapshots restore /ifs/projectX --snapshot <snap‑name>` | `isi snapshots restore /ifs/projectX --snapshot pre-upgrade` |
| **Cloner un snapshot** (nouveau répertoire) | `isi snapshots clone /ifs/projectX --snapshot <snap‑name> --target /ifs/projectX_clone` | `isi snapshots clone /ifs/projectX --snapshot pre-upgrade --target /ifs/projectX_clone` |
| **Supprimer un snapshot** | `isi snapshots delete /ifs/projectX --snapshot <snap‑name>` | `isi snapshots delete /ifs/projectX --snapshot pre-upgrade` |
| **Planifier la création de snapshots** | `isi snapshots schedule create --path /ifs/data --frequency daily --time 02:00 --retain 30` | `isi snapshots schedule create --path /ifs/data --frequency daily --time 02:00 --retain 30` |

---

## 10.  Gestion des licences

| Action | Commande | Exemple |
|---|---|---|
| **Lister les licences installées** | `isi licenses list -i` | `isi licenses list -i` |
| **Installer une licence** | `isi licenses add --file /ifs/license/OneFS_9.5.0.1.lic` | `isi licenses add --file /ifs/license/OneFS_9.5.0.1.lic` |
| **Supprimer une licence** | `isi licenses delete <license‑id>` | `isi licenses delete 3` |
| **Afficher** | `isi licenses status` | `isi licenses status` |

---

## 11.  Outils de scripting / automatisation

| Outil | Usage | Exemple |
|---|---|---|
| **`isi` + `jq`** | Parser le JSON produit par `-j`. | `isi storage pool list -j | jq '.pools[] | {name:.name,free:.free}'` |
| **`isi` + `awk`** | Rapports texte rapides. | `isi status -i | awk '/CPU/ {print $2,$3}'` |
| **`isi` + `cron`** | Planifier des snapshots, quotas, etc. | `crontab -e` → `0 2 * * * isi snapshots schedule run` |
| **`isi` + `ansible`** | Modules `onefs_*` (community.dellemc.onefs). | `- name: create NFS export`<br>`  dell_emc.onefs.onefs_nfs_export:`<br>`    path: /ifs/data`<br>`    clients: 10.0.0.0/24` |
| **`isi` + `python`** | SDK `onefs-sdk-python` (pip install onefs). | `from onefs import OneFS`<br>`c = OneFS(host='10.0.0.1', user='admin', password='…')`<br>`c.storage.pool.create('pool01', nodes=['node01','node02'], disks=['0:0:0','0:0:1'])` |

---

## 12.  Bonnes pratiques rapides (check‑list du jour)

| Vérification | Commande | Fréquence |
|---|---|---|
| **État du cluster** | `isi status` | À chaque login. |
| **Alertes critiques** | `isi events list --severity critical -i` | Toutes les heures (ou via alerting). |
| **Capacité restante** | `isi storage pool list -i` | Quotidien. |
| **Utilisation des quotas** | `isi quota report -i` | Quotidien. |
| **Snapshots planifiés** | `isi snapshots schedule list -i` | Hebdomadaire. |
| **Licences valides** | `isi licenses status` | Mensuel. |
| **Mises à jour disponibles** | `isi upgrade list` | Mensuel. |
| **Synchronisation SnapMirror** | `isi snapmirror list -i` | Toutes les 2 h (ou selon SLA). |
| **Vérifier les performances réseau** | `isi network interface list -i` + `isi network interface stats` | Hebdomadaire. |
| **Sauvegarde de la configuration** | `isi configuration export -f /ifs/backup/cluster_cfg_$(date +%F).tgz` | Avant chaque upgrade. |

---

## 13.  Ressources & Documentation officielle

| Lien | Contenu |
|---|---|
| **OneFS CLI Reference** | <https://www.delltechnologies.com/en-us/documentation/onefs/cli-reference/> |
| **OneFS Administration Guide (9.x)** | <https://www.dell.com/support/kbdoc/en-us/000210317/onefs-9-5-administration-guide> |
| **PowerScale SDK (Python)** | <https://github.com/dell/onefs-sdk-python> |
| **Ansible Collection – dell_emc.onefs** | <https://galaxy.ansible.com/dell_emc/onefs> |
| **PowerScale Community Forums** | <https://community.dell.com/t5/PowerScale/bd-p/PowerScale> |
| **Dell EMC Support – Knowledge Base** | Recherche (recherchez “OneFS <version> CLI”) |

---

### 📌 Points clés à retenir

1. **Toutes les actions d’administration passent par le binaire `isi`.**  
2. **Utilisez `-j` + `jq`** pour des scripts robustes et idempotents.  
3. **Gardvegardez la configuration** (`isi configuration export`) avant chaque changement majeur (upgrade, ajout de nœuds, modification de licences).  
4. **Gardez 3 nœuds MON** et **répliquez les pools à 3 copies** (ou 2 + EC) pour la tolérance aux pannes.  
5. **Planifiez les snapshots / SnapMirror** en dehors des heures de pointe pour éviter la surcharge.  
6. **Vérifiez les licences** (capacity, protocoles) dès le premier jour afin d’éviter les blocages de service.  

