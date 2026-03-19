# My Cheatsheet — Storage / Ceph / NetApp / Dell (Notes & Command Snippets)

Collection de notes techniques, commandes et procédures (format Markdown) autour du **storage** et de l’**infrastructure** : **Ceph/CephFS**, **Dell (PowerScale/PowerStore)**, **NetApp (ONTAP / ONTAP Select)**, ainsi que des checklists de planning et du troubleshooting.

> Objectif : centraliser des “cheat sheets” rapides à consulter, avec des commandes prêtes à copier/coller et des rappels de bonnes pratiques.

---

## Table des matières

- [Contenu](#contenu)
- [Structure du dépôt](#structure-du-dépôt)
- [Comment utiliser ces notes](#comment-utiliser-ces-notes)
- [Conventions d’écriture](#conventions-décriture)
- [Contribuer](#contribuer)
- [Licence](#licence)
- [Disclaimer](#disclaimer)

---

## Contenu

### Ceph / CephFS
- **[`ceph.md`](./ceph.md)** — notes générales Ceph (commandes, rappels, ops)
- **[`migration-vers-ceph.md`](./migration-vers-ceph.md)** — éléments et étapes de migration
- **[`cephfs-troobleshooting-1.md`](./cephfs-troobleshooting-1.md)** — dépannage CephFS (partie 1)
- **[`cephfs-troobleshooting-2.md`](./cephfs-troobleshooting-2.md)** — dépannage CephFS (partie 2)

### Dell
- **[`dell.md`](./dell.md)** — notes Dell diverses
- **[`powerstore-cheat-sheet.md`](./powerstore-cheat-sheet.md)** — commandes / raccourcis PowerStore
- **[`powerstore-good-practices.md`](./powerstore-good-practices.md)** — bonnes pratiques PowerStore
- **[`powerscale-troobleshoot-1.md`](./powerscale-troobleshoot-1.md)** — dépannage PowerScale

### NetApp / ONTAP
- **[`netapp.md`](./netapp.md)** — notes NetApp diverses
- **[`onefs-good-practices.md`](./onefs-good-practices.md)** — bonnes pratiques OneFS (PowerScale)
- **[`ontap-select-good-practices.md`](./ontap-select-good-practices.md)** — bonnes pratiques ONTAP Select

### Checklists / divers
- **[`planning-storage-check.md`](./planning-storage-check.md)** — checklist de planning/storage assessment
- **[`questions-answers.md`](./questions-answers.md)** — Q/A, retours d’expérience, rappels
- **[`architecture-test.md`](./architecture-test.md)** — brouillon / essais d’architecture

---

## Structure du dépôt

Arborescence actuelle :

```text
.
├── architecture-test.md
├── cephfs-troobleshooting-1.md
├── cephfs-troobleshooting-2.md
├── ceph.md
├── dell.md
├── migration-vers-ceph.md
├── netapp.md
├── onefs-good-practices.md
├── ontap-select-good-practices.md
├── planning-storage-check.md
├── powerscale-troobleshoot-1.md
├── powerstore-cheat-sheet.md
├── powerstore-good-practices.md
├── questions-answers.md
└── README.md
```

---

## Comment utiliser ces notes

### Lecture rapide
- Utilise la section [Contenu](#contenu) pour naviguer par sujet.
- Chaque fichier est indépendant et peut être lu “à la demande”.

### Recherche (conseillé)
Recherche plein texte dans le dépôt :
```bash
grep -Rni "cephfs" .
```

Ou avec `ripgrep` (plus rapide) :
```bash
rg -n "quota|mds|health" .
```

### Usage “cheatsheet” dans le terminal
Tu peux prévisualiser un fichier directement :
```bash
sed -n '1,120p' ceph.md
```

---

## Conventions d’écriture

Pour garder les notes lisibles et cohérentes :

- Titres : `#`, `##`, `###` (éviter de sauter des niveaux)
- Commandes dans des blocs :
  ```bash
  # exemple
  ceph -s
  ```
- Préférer :
  - des listes courtes
  - des sections “Symptômes / Cause / Fix”
  - des exemples concrets
- Si une commande est dangereuse, la marquer clairement :
  > ⚠️ **Attention** : commande impactante / destructive, à exécuter avec prudence.

---

## Contribuer

Suggestions, corrections et ajouts sont bienvenus.

1. Fork le dépôt
2. Crée une branche :
   ```bash
   git checkout -b docs/ameliore-readme
   ```
3. Commit clair :
   ```bash
   git commit -m "docs: improve cephfs troubleshooting notes"
   ```
4. Ouvre une Pull Request

---

## Licence

- Partager sans contrainte : **CC BY 4.0**

---

## Disclaimer

Ces notes sont fournies **telles quelles**. Elles peuvent contenir des raccourcis, hypothèses de contexte ou commandes à fort impact.

Avant exécution en production :
- valider la pertinence pour votre environnement (versions, contraintes, politique sécurité),
- tester en environnement de pré-prod,
- relire les commandes potentiellement destructives.

---
