# 🏎️ Pit Wall — Gestion de préparation course & planning paddock

Application web de suivi de préparation d'une voiture de course : tableau de bord, planning (Gantt), interventions par département, checklists et gestion d'équipe. Front statique hébergé sur **GitHub Pages**, données et authentification gérées par **Supabase**.

---

## Sommaire

- [Fonctionnalités](#fonctionnalités)
- [Stack technique](#stack-technique)
- [Architecture](#architecture)
- [Rôles et permissions](#rôles-et-permissions)
- [Structure des fichiers](#structure-des-fichiers)
- [Installation (nouveau projet)](#installation-nouveau-projet)
- [Mise à jour d'une base existante](#mise-à-jour-dune-base-existante)
- [Sécurité](#sécurité)
- [Personnalisation](#personnalisation)
- [Limites connues & pistes](#limites-connues--pistes)

---

## Fonctionnalités

- **Tableau de bord** : jauge de préparation globale, compte à rebours jusqu'à la course, indicateurs clés (tâches ouvertes, points bloquants, retards, validées), avancement par département, répartition des statuts et panneau d'alertes (tâches critiques ou en retard).
- **Interventions** : tableau groupé par département, recherche, filtres (statut, criticité, département) et tri. Chaque tâche est dépliable et contient une **description** libre et une **checklist de sous-tâches**. Détection automatique des retards.
- **Diagramme de Gantt** : chronogramme avec axe temporel, repère « aujourd'hui » et repère « course », barres colorées par criticité avec avancement.
- **Gestion d'équipe** (ingénieur) : création de comptes, attribution de rôle, rattachement à un département, suppression.
- **Départements** : création, renommage, réordonnancement, suppression ; affiliation des membres.
- **Affectation** : chaque tâche peut être confiée à une personne (compte réel), avec son département affiché.
- **Cloisonnement par département** : un lecteur ou un contributeur ne voit que les tâches de son propre pôle.
- **Temps réel** : toute modification (tâche, sous-tâche, département, course) se propage automatiquement aux autres écrans connectés.
- **Confort** : écran de chargement avec squelettes, micro-animations, respect de `prefers-reduced-motion`, export **JSON** et **CSV**.

---

## Stack technique

| Couche | Technologie |
|---|---|
| Front | HTML + JavaScript (vanilla), [Tailwind CSS](https://tailwindcss.com/) (CDN), polices Oxanium + Inter, icônes SVG intégrées |
| Base de données | PostgreSQL (via Supabase) |
| Authentification | Supabase Auth (e-mail / mot de passe) |
| Temps réel | Supabase Realtime |
| Logique privilégiée | Supabase Edge Function (Deno / TypeScript) |
| Hébergement front | GitHub Pages (fichier statique) |

Aucune étape de build : le front est un unique fichier `index.html`.

---

## Architecture

```
   Navigateur (GitHub Pages)                 Supabase
 ┌───────────────────────────┐        ┌──────────────────────────┐
 │  index.html               │        │  PostgreSQL + RLS         │
 │  - UI, rendu, temps réel  │  HTTPS │  - profiles, departments, │
 │  - clé publishable        │◄──────►│    tasks, subtasks,       │
 │    (lecture/écriture       │        │    settings              │
 │     selon la RLS)         │        │  Auth (comptes, rôles)    │
 └─────────────┬─────────────┘        │  Realtime                 │
               │ appel signé           │  Edge Function            │
               └──────────────────────►│  « manage-users »         │
                 (gestion des comptes)  │  (clé service_role,       │
                                        │   côté serveur uniquement)│
                                        └──────────────────────────┘
```

- Le front utilise la **clé publishable** : sans danger dans le navigateur, car toute la sécurité est appliquée par la **RLS** côté base.
- La création / suppression de comptes exige des droits privilégiés (clé `service_role`) : elle passe par la fonction Edge **`manage-users`**, où la clé reste côté serveur et n'est jamais exposée.

---

## Rôles et permissions

Le rôle est stocké dans la table `profiles` et modifiable uniquement par un ingénieur (via la fonction Edge).

| Capacité | Ingénieur | Contributeur | Lecteur |
|---|:---:|:---:|:---:|
| Voir le tableau de bord / Gantt | ✅ (tous pôles) | ✅ (son pôle) | ✅ (son pôle) |
| Créer / modifier / supprimer une tâche | ✅ | ❌ | ❌ |
| Modifier le **lien de suivi** d'une tâche « ouverte » | ✅ | ✅ (son pôle) | ❌ |
| Gérer départements & réglage course | ✅ | ❌ | ❌ |
| Gérer les utilisateurs (créer, rôle, département) | ✅ | ❌ | ❌ |

- **Visibilité** : l'ingénieur voit tout ; le contributeur et le lecteur ne voient que les tâches de **leur** département. Un non-ingénieur **sans département** ne voit aucune tâche (message d'invitation à se faire rattacher).
- **Ouverture à la contribution** : l'ingénieur « ouvre » une tâche (verrou `link_open`) pour qu'un contributeur puisse en renseigner le lien de suivi — et rien d'autre.

Ces règles sont appliquées par la **RLS** (base), pas seulement par l'interface : elles restent valables même en interrogeant l'API directement.

---

## Structure des fichiers

```
index.html                     Application complète (front)
schema.sql                     Schéma complet de la base (installation from scratch)
manage-users.ts                Code de la fonction Edge « manage-users »
migrations/                    Modifications incrémentales (voir plus bas)
  migration-role-contributeur.sql
  migration-link-open.sql
  migration-task-content.sql
  migration-user-department.sql
  migration-assignee.sql
  migration-dept-visibility.sql
```

> Pour une **nouvelle** installation, `schema.sql` suffit : il contient déjà l'intégralité des tables, fonctions, politiques RLS et le temps réel. Les fichiers `migration-*.sql` ne servent qu'à faire évoluer une base déjà en place.

---

## Installation (nouveau projet)

### 1. Créer le projet Supabase
Sur [supabase.com](https://supabase.com), créer un projet (offre gratuite suffisante).

### 2. Créer la base
Dans **SQL Editor → New query**, coller tout le contenu de `schema.sql`, puis **Run**.
Résultat attendu : *Success. No rows returned*.

### 3. Brancher le front
Dans **Project Settings → API** (ou le bouton **Connect**), récupérer :
- le **Project URL** (`https://xxxx.supabase.co`) ;
- la clé **publishable** (`sb_publishable_…`).

Les renseigner en haut du second `<script>` de `index.html` :

```js
const SUPABASE_URL      = "https://xxxx.supabase.co";
const SUPABASE_ANON_KEY = "sb_publishable_xxxxxxxx";
```

### 4. Déployer la fonction Edge
Dans **Edge Functions → Deploy a new function → Via Editor** :
- nommer la fonction **exactement** `manage-users` ;
- coller le contenu de `manage-users.ts` ;
- **désactiver « Verify JWT »** (onglet *Settings* de la fonction) — la fonction fait sa propre vérification, et cela évite le blocage CORS de la requête *preflight* ;
- **Deploy**.

Vérification : ouvrir `https://xxxx.supabase.co/functions/v1/manage-users` doit renvoyer
`{"ok":false,"error":"Non authentifié."}` (la fonction tourne).

Aucune clé à configurer : `SUPABASE_URL` et `SUPABASE_SERVICE_ROLE_KEY` sont injectées automatiquement dans la fonction.

### 5. Créer le premier ingénieur
- **Authentication → Users → Add user** : e-mail + mot de passe, cocher **Auto Confirm User**.
- **Table Editor → `profiles` →** sur la ligne du compte, passer `role` à `ingenieur`.
- (Recommandé) Couper l'inscription libre : **Authentication → Sign In / Providers → « Allow new users to sign up »** sur *off* (les comptes se créent depuis l'application).

### 6. Publier le front
Pousser `index.html` sur le dépôt GitHub, activer **Settings → Pages**, puis partager l'URL à l'équipe.
Les autres comptes se créent ensuite directement depuis la section *Utilisateurs* de l'application.

---

## Mise à jour d'une base existante

Si la base a été créée avec une version antérieure, appliquer les migrations manquantes **dans l'ordre**, via SQL Editor. Chaque fichier est ré-exécutable sans casse.

1. `migration-role-contributeur.sql` — ajoute le rôle *contributeur*.
2. `migration-link-open.sql` — verrou « tâche ouverte à la contribution ».
3. `migration-task-content.sql` — description + sous-tâches.
4. `migration-user-department.sql` — rattachement d'un profil à un département.
5. `migration-assignee.sql` — responsable relié à un compte + durcissement du garde-fou contributeur.
6. `migration-dept-visibility.sql` — cloisonnement de la visibilité par département.

---

## Sécurité

- **Clé publishable dans le front** : normal et prévu. Elle ne donne accès qu'à ce que la RLS autorise.
- **Clé `service_role`** : jamais dans le front ni sur GitHub. Utilisée uniquement par la fonction Edge, côté serveur.
- **RLS activée** sur toutes les tables : lecture et écriture filtrées par rôle et par département.
- **Pas d'auto-promotion** : un utilisateur ne peut pas modifier son propre rôle ni son rattachement depuis le front ; ces changements passent par la fonction Edge, réservée aux ingénieurs.
- **Garde-fou contributeur** : au niveau base, un contributeur ne peut modifier que le champ *lien* d'une tâche ouverte de son département — toute autre modification est rejetée, même via l'API.

> ⚠️ Si une clé `service_role` (ou toute clé secrète) a été exposée par erreur, la révoquer immédiatement dans **Project Settings → API Keys**.

---

## Personnalisation

- **Course affichée** : modifiable dans l'application (panneau ingénieur → nom + date/heure de la course). Stockée dans la table `settings` (clé `race`).
- **Données de démonstration** : `schema.sql` insère quelques départements et tâches d'exemple. Supprimer le bloc « DONNÉES DE DÉMO » à la fin du fichier pour partir d'une base vide.
- **Départements, statuts, criticités** : les statuts (`À planifier`, `En cours`, `Prêt / Validé`) et criticités (`Normal`, `Important`, `Critique`) sont définis par des contraintes dans `schema.sql` et repris dans `index.html` ; les modifier impose de les changer aux deux endroits.
- **Renommer le rôle « contributeur »** : possible, mais l'identifiant (*slug*) doit rester identique dans `schema.sql`, `manage-users.ts` et `index.html` ; seuls les libellés affichés peuvent changer.

---

## Limites connues & pistes

- **Tailwind via CDN** : affiche un avertissement console « should not be used in production ». Sans impact fonctionnel ; pour un rendu « prod propre », compiler Tailwind.
- **Un seul département par personne** : le modèle actuel n'autorise pas l'appartenance multi-départements (extensible via une table de liaison).
- **Affichage mobile** : les tableaux défilent horizontalement sur petit écran ; une vue « carte » par tâche améliorerait le confort.
- **Réinitialisation de mot de passe** : non gérée dans l'app (comptes créés avec mot de passe par l'ingénieur) ; peut être ajoutée via l'e-mail de réinitialisation Supabase.

---

*Projet réalisé dans le cadre d'un cursus Cloud / Infrastructure & Sécurité.*
