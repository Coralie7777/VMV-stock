# VMV Stock — contexte projet

App PWA mono-fichier (`index.html`) de gestion de stock aluminium/PVC pour VMV
(menuiserie alu/PVC, Firminy, Loire). Utilisée par Coco (responsable
administratif), Karim (manager), Philippe/Vincent (chefs de chantier),
Nabila (devis particuliers).

## Architecture

- **Fichier unique** : tout le code (HTML/CSS/JS) est dans `index.html`, pas
  de build step, pas de dépendances npm côté front — tout est chargé via CDN
  (pdf.js, ZXing) au besoin.
- **Deux profils utilisateurs** détectés par le nom d'utilisateur :
  `isAtelier()` (contient "atelier") ne voit que Stock + Scan ; le profil
  bureau voit tous les onglets (Stock, BL, Chantiers, Scan, Alertes, Export).

### Supabase — `rqhixlqtqmzlghguvmoy.supabase.co`

> ⚠️ L'ancien projet (`igzsvpotzpmfnmlqcjpi.supabase.co`) a été perdu/rendu
> inaccessible et reconstruit de zéro en septembre 2026. Ne jamais réutiliser
> cette ancienne URL.

Tables (RLS activé partout, policies anon/authenticated permissives — pas
d'auth utilisateur réelle dans l'app) :

- `stock` — `id, ref UNIQUE, qty, seuil, statut, created_at`. SELECT/INSERT/UPDATE.
- `lignes_chantier` — `id, ch_num, ch_client, ref, designation, unite, prevue, sortie, created_at`. SELECT/INSERT/UPDATE.
- `commandes_pdf` — `ch_id PK, url, updated_at`. SELECT/INSERT/UPDATE.
- `articles` (catalogue) — `ref PK, designation, famille, couleur, unite, categorie, created_at`.
  **SELECT + INSERT uniquement, pas d'UPDATE policy** — à ajouter si besoin
  d'éditer des fiches existantes un jour. 3811 lignes importées depuis
  l'ancien catalogue JS figé + complétion automatique via la fiche de
  fabrication (voir plus bas).
- Storage bucket `COMMANDES` (public, policies SELECT public + INSERT/UPDATE anon).

`sbFetch(path, opts)` est le wrapper générique pour les appels REST Supabase
— sauf `saveCommandeSupabase` qui utilise `fetch` brut avec
`Prefer: resolution=merge-duplicates,return=minimal` (le wrapper `sbFetch`
provoque des échecs silencieux sur réponses vides).

### Airtable

- **SUIVI VMV** (`appfx33OLfjgPOaji`) : chantiers pro (`tblNOOug2Z54IDZmQ`,
  numéro extrait par regex `^\d{2}\s\d{2}\s\d{2,3}$` sur les champs) et
  chantiers particuliers (`tblsvU8oIkwxT64At` — champ `Chantier` = numéro,
  champ `Nom` = client).
- **VMV BL** (`appvViFHwkQvcE35q`) : bons de livraison.
- **VMV Stock Aluminium** (`appZgeb2SHgS4i3ky`), **VMV Utilisateurs**.
- Auth : chaque utilisateur entre son propre Personal Access Token Airtable
  à la connexion (stocké en localStorage), pas de token partagé.
- **Quota gratuit Airtable : 1000 appels API/mois, partagés sur tout le
  workspace** (toutes bases confondues), 5 req/s. Très serré — la synchro
  automatique (`setInterval(autoSync,...)`) a été **retirée entièrement**
  en septembre 2026 pour ne pas le griller ; seul le bouton "🔄 Sync"
  manuel (`syncAll()`) déclenche des appels Airtable désormais. Ne pas
  réintroduire de polling automatique sans recalculer le budget.
- 403 sur Airtable = problème d'accès collaborateur sur la base précise
  (pas un souci de scope du token) → réinviter la personne comme
  collaborateur directement dans la base concernée.
- 429 `PUBLIC_API_BILLING_LIMIT_EXCEEDED` = quota mensuel épuisé, se
  réinitialise en début de mois suivant, rien à corriger côté code.
- Champ "Nom chantier" doit être type "Texte (ligne unique)", jamais
  "Texte long" (sinon absent des réponses API).
- Champ collaborateur "Qui" retourne `{name,email}` ou un tableau de ces
  objets.
- POST Airtable nécessite `typecast:true` pour accepter de nouvelles
  valeurs sur un champ `singleSelect`.

### Hébergement

- Repo GitHub : `Coralie7777/VMV-stock`.
- Déployé sur **Cloudflare** en tant que **projet Workers** (pas Pages
  classique) connecté au repo via intégration Git ; commande de build =
  `npx wrangler deploy`, déclenché automatiquement à chaque push.
- URL de production : `https://vmv-stock.crystalzenflow.workers.dev`. Le
  toggle "Production" sous l'onglet **Domaines** du projet doit être activé
  manuellement pour que l'URL soit publique (à vérifier si jamais l'URL
  semble down après un changement de config).
- Workflow : éditer `index.html` → push GitHub → attendre ~20-30s le build
  Cloudflare (visible dans l'onglet Déploiements) → rafraîchissement forcé
  navigateur (**Ctrl+F5**, la navigation privée ne fonctionne pas sur la
  machine de Coco pour vider le cache).

## Fonctionnalités principales

- **Stock** : recherche/filtre du catalogue (3811 articles), boutons +/-
  par référence, unité M (mètres) ou PC (pièces).
- **BL** : import PDF, extraction via pdf.js, rapprochement de conformité.
- **Chantiers** : liste pro + particuliers, détail par chantier avec lignes
  prévues/sorties, upload PDF de commande vers le bucket `COMMANDES`.
- **Scan** : douchette QR/code-barres via ZXing
  (`unpkg.com/@zxing/library@0.20.0/umd/index.min.js` — seule source CDN
  confirmée fonctionnelle, ne pas changer sans tester).
- **Fiche de fabrication → sortie de stock** (ajouté sept. 2026) : dans le
  détail d'un chantier, on peut déposer un PDF de fiche de fabrication
  Technal/TechDesign. Extraction fiable via **reconstruction positionnelle**
  (x/y des items pdf.js, ancrée sur les en-têtes de colonnes connus —
  `N°`/`Désignation`/`Trait. de surface`/`Qté`/`Long.`/`Coupe`) car le flux
  texte brut du PDF est **column-major** (toutes les réf, puis toutes les
  désignations, etc.) et inexploitable en concaténation naïve. Sections
  `Profilés`/`Profilés additionnels`/`Joints` → longueur en mètres
  (Σ qté×longueur/1000) ; `Quincailleries` → comptage en pièces. Un écran
  de confirmation liste chaque référence avec stock avant/après avant toute
  écriture. Les références absentes du catalogue sont **automatiquement
  ajoutées** à `articles` (réf/désignation/couleur extraites du PDF) en
  plus d'être décomptées du stock — le catalogue se complète ainsi tout
  seul au fil de l'usage plutôt que de bloquer les sorties.

## Pièges JS connus dans ce fichier

- Tirets unicode (U+2500), vrais retours à la ligne dans des littéraux de
  chaîne, BOM, et échappement de guillemets mixte dans de l'`innerHTML` ont
  déjà causé des erreurs de syntaxe silencieuses/difficiles à diagnostiquer
  — vérifier avec `node --check` sur le contenu du `<script>` principal
  après toute édition non triviale.

## Préférences de travail de Coco

- Corrections directes et simples plutôt qu'exploratoires.
- Avant toute modification touchant au stock ou au catalogue en masse,
  prévoir un écran de confirmation/prévisualisation plutôt qu'une écriture
  silencieuse (cf. fiche de fabrication).
- Toujours vérifier `node --check` sur le JS extrait avant de livrer un
  `index.html` modifié.
