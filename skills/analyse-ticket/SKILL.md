---
name: analyse-ticket
description: >-
  Analyse un ticket (Azure DevOps / Jira, ex. ABC-1234) du workspace de l'équipe et produit une
  spec Markdown + un prompt d'implémentation autonome par repo concerné, avec DoD précis.
  Déclenche ce skill DÈS QUE l'utilisateur colle le contenu d'un ticket, mentionne un identifiant
  de ticket (ABC-…, ou un titre "Description (What) / DoR / DoD"), ou demande d'« analyser un
  ticket », de « faire une spec », de « préparer les prompts pour les repos », même sans nommer le
  skill. À utiliser pour tout ticket touchant les projets <PRODUCT_A>, <PRODUCT_B> ou l'ingestion partagée
  (les repos de l'équipe). Ne PAS utiliser pour implémenter le ticket (c'est le rôle du skill implement-ticket) —
  ce skill produit uniquement l'analyse et les prompts.
---

# Analyse de ticket — workspace de l'équipe

Transforme un ticket brut en **spec + prompts d'implémentation prêts à exécuter**, un prompt par
repo concerné. Le livrable doit être self-contained : quelqu'un (ou le skill `implement-ticket`)
doit pouvoir coder à partir du prompt sans avoir vu ce chat ni le ticket original.

## Règle d'or : vérifier le ticket dans le code AVANT d'écrire la spec

Un ticket décrit une **intention**, pas la réalité du code — et il se trompe souvent sur l'état
actuel. Sur ABC-3419, le ticket affirmait que la whitelist était « dans un CSV » alors qu'elle
était **codée en dur** dans des listes Python, et la présentait comme une liste plate « URL + pays »
alors qu'elle portait 3 dimensions métier (catégorie / pays / contour). Écrire la spec sur la foi du
ticket aurait produit un mauvais schéma et cassé le comportement.

Donc, systématiquement : **localise le code réel concerné, lis-le, et confronte-le aux affirmations
du ticket.** Tout écart entre ce que dit le ticket et ce que fait le code est un constat à remonter
en tête de spec — c'est souvent l'apport le plus précieux de l'analyse.

**Le code est souvent EN AVANCE sur ce qu'on croit** (features déjà mergées entre-temps) : ex.
`is_pinned`, `useInfiniteQuery`, `preserve-scroll`, la pagination des messages étaient déjà
implémentés quand des tickets les supposaient absents. Ne jamais partir de la mémoire ou d'un ticket
antérieur : re-grep l'état courant. Corollaire fréquent : un fix « évident » cache un prérequis
(ex. ABC-3555 : passer à `key={message.id}` cassait React car le message user avait `id: ''` — il
fallait d'abord lui donner un id unique).

## Étape 0 — Charger la cartographie

Lis l'index mémoire du workspace pour savoir quels repos existent et à quoi ils servent :
`.claude-memory/MEMORY.md` (puis la fiche `repo-*.md` pertinente). Il y a 10 repos : 6 <PRODUCT_A>
(front, api, admin-ui, indexer, ingest, infra), 2 <PRODUCT_B> (api, front), 2 GROUPE_DIGITAL (airflow
partagé, pipelines-common). Consulte aussi l'audit existant par repo dans `<PRODUCT_A>/AuditFable/*.md` +
`tickets_audit.csv` : il contient souvent le contexte de dette technique et des conventions déjà
identifiés (ex. « pas d'Alembic, script SQL dans docs/ » pour l'Admin UI).

## Workflow

1. **Comprendre le ticket** : extraire l'intention (What), le DoR et le DoD. Noter l'identifiant
   (ex. `ABC-3419`) — il sert de nom de dossier.

2. **Investiguer le code** (règle d'or). Utilise Grep/Glob pour localiser le module, le CSV, la
   table, la config réels. Lis les fichiers clés. Pour un ticket qui ratisse large, lance des
   sous-agents `Explore` en parallèle plutôt que de tout lire toi-même. Identifie :
   - le/les repo(s) réellement touché(s) et les fichiers/zones précis ;
   - le pattern existant à réutiliser (ne réinvente pas : copie la façon dont le repo fait déjà la
     même classe de chose — accès DB, vue CRUD, cache, migration, tests) ;
   - les écarts entre le ticket et le code.

3. **Décider et remonter les points ouverts**. Si le DoR demande une validation (schéma, périmètre)
   ou si tu détectes une décision structurante, **propose une option recommandée** (motivée) et
   demande la validation de l'utilisateur via une question ciblée — ne bloque pas sur des détails à défaut
   conventionnel, mais ne tranche pas seul une décision qui lui revient (schéma de données,
   périmètre, contrat d'API). Une fois validé, mets à jour les fichiers pour retirer les « à valider ».

4. **Écrire les livrables** dans `specs/<TICKET-ID>/` à la racine du workspace :
   - `spec.md` — l'analyse complète (voir structure ci-dessous) ;
   - `prompt-<repo>.md` — un fichier **séparé par repo concerné** (ex. `prompt-api.md`,
     `prompt-front.md`, `prompt-admin-ui.md`, `prompt-infra.md`, `prompt-airflow.md`).
   - **Nom du dossier** : l'ID réel (`ABC-3419` ; `Fred#8` → `Fred-8`, pas de `#` dans les chemins).
     Sans ID, un slug parlant (`BUG-sidebar-infinite-scroll`, `PERF-sse-coalescing`).
   - **Ticket XL** : ajouter `subtickets.md` décomposant en sous-tâches trackables (What/Why/How + DoD,
     ordre, dépendances), comme `specs/Fred-8/`.

5. **Restituer** à l'utilisateur : rappel des repos concernés, des constats importants (écarts ticket/code),
   et des décisions à valider. Liens cliquables vers les fichiers produits.

### ⚠️ Le repo réel ≠ le repo annoncé
Le label du ticket ment souvent. Déterminer les VRAIS repos :
- un ticket « API » peut être **data/pipeline** (`<AIRFLOW_REPO>`, ex. Fred#8) ou **infra** (Terraform) ;
- un ticket « front » peut exiger un **changement de contrat backend** (ex. ABC-3558 feedback → `trace_id`
  émis par le stream + accepté au feedback ; rétention → pagination serveur des messages) ;
- attention **migration** : l'ingestion legacy (`<INGEST_REPO>`) est décommissionnée au profit
  du partagé `<AIRFLOW_REPO>` — vérifier lequel est vivant.
Quand c'est front+backend, écrire un prompt par repo, **API d'abord**, et **aligner le contrat** (mêmes
noms de champs) explicitement dans les deux prompts.

### Décisions structurantes → AskUserQuestion, puis acter
Pour un choix qui revient à l'utilisateur (schéma DB, contrat d'API, mécanisme, périmètre, front-only vs +backend,
XL à découper) : poser une **AskUserQuestion** avec l'option **recommandée en premier**. Une fois tranché,
**mettre à jour spec + prompts** : remplacer les « ⚠️ À VALIDER » par « ✅ VALIDÉ avec l'utilisateur (date) » et
retirer les conditionnels, pour que les prompts soient directement exécutables.

## Structure de `spec.md`

Adapte au ticket, mais couvre au minimum :

```
# <TICKET-ID> — <titre court>
## 1. Résumé (l'intention en 3-4 lignes)
## 2. État réel du code (⚠️ écarts vs description du ticket — fichiers + preuves)
## 3. Architecture cible (schéma texte du flux/composants)
## 4. Détail technique (schéma de données / contrat d'API / algo — marquer ✅ VALIDÉ ou ⚠️ À VALIDER)
## 5. Repos concernés (tableau : repo → rôle → lien vers le prompt)
## 6. Risques & décisions (régressions possibles, fallback, points ouverts)
## 7. Definition of Done consolidée (checklist vérifiable)
```

## Structure d'un `prompt-<repo>.md`

Chaque prompt est **autonome** et contient :

```
# Prompt d'implémentation — <TICKET-ID> — Repo <NOM_REPO>
> Où l'exécuter (chemin) + ordre conseillé vs autres repos + mention "enchaînable avec implement-ticket".
## Contexte
  - à quoi sert le repo, ce qu'il faut savoir de sa stack et de ses conventions (depuis la mémoire) ;
  - le/les fichiers et patterns exacts à réutiliser (chemins précis, fonctions/classes de référence).
## Ce qu'il faut faire
  - étapes concrètes et numérotées, référençant des fichiers réels ;
  - quand une décision technique reste ouverte, le dire explicitement + recommandation.
## Definition of Done
  - checklist vérifiable : comportement attendu, tests (dont un test anti-régression si on modifie
    un comportement existant), lint, CI.
## Garde-fous
  - iso-fonctionnel / ne pas casser X ; respecter la convention Y ; alignement de contrat inter-repos.
```

## Barre de qualité (ce qui rend un prompt bon)

- **Précision des chemins** : cite les fichiers réels (`src/lib/...:ligne`), pas des généralités.
- **Réutilisation** : pointe le pattern existant à copier plutôt que d'inventer.
- **Iso-fonctionnel d'abord** : si on migre/refactore un comportement, exiger un **test golden** qui
  prouve l'équivalence avec l'existant. C'est la meilleure protection anti-régression.
- **Fallback & robustesse** : pour tout accès réseau/DB ajouté, prévoir le cas indisponible.
- **Contrats inter-repos** : quand deux repos partagent une donnée (table, API), garder les prompts
  alignés et le dire dans chacun.
- **DoD vérifiable** : chaque case doit pouvoir être cochée objectivement (un test, un lint, un comportement).

## Catalogue d'anti-patterns récurrents (à repérer / vérifier)

Ces familles reviennent sans cesse dans <PRODUCT_A> — les chercher accélère l'analyse et évite les faux fixes :

- **`throw` avalé par un `catch` de parsing** : un `throw` intentionnel (ex. `{"type":"error"}`, 403
  `detail`) est dans le même `try` que le `JSON.parse`/`response.json()` → le `catch` le remplace par un
  message générique. Fix : n'entourer QUE le parse ; sortir le dispatch. (ABC-3554, ABC-3556)
- **Échec silencieux / `catch { /* silently fail */ }`** : rename/delete/feedback échouent sans signal.
  Fix : remonter via `showToast(t('..._failed'))` + refetch (état serveur). **Template dans le repo** :
  `handlePinToggle` (app-sidebar). (ABC-3557)
- **Fire-and-forget + succès présumé** : appel non `await`é + confirmation posée immédiatement → « merci »
  même si l'envoi a raté. Fix : await + confirmation conditionnelle. (ABC-3558)
- **Fail-open silencieux** : requête envoyée sans `Authorization` quand le token échoue (catch avale) →
  401 en cascade. Fix : **fail-closed** + reprise via `handle401Response`. (ABC-3559)
- **Succès inconditionnel ignorant un booléen** : endpoint renvoie 204/OK sans lire le `bool` du service
  (0 ligne / erreur avalée). Fix : mapper `False` → 404, exception → 500. (ABC-3565)
- **Placeholder Swagger envoyé tel quel** : `additionalProp1/2/3` dans un payload → junk côté backend.
- **i18n** : chaîne codée en dur (souvent en FR) au lieu de `t()` ; ou clé manquante dans une locale.
  ⚠️ **~12 locales** (`cs, el, en, en-GB, es, es-419, fr, nl-BE, nl-NL, pl, pt, pt-BR`) — toute
  modif/ajout de clé va dans **toutes**, sinon next-intl casse. (ABC-3556, 3560, 3561)
- **Perf streaming / rendu** : écriture d'état par token (coalescer, ~60-100 ms + flush final) ;
  `setQueryData` par token ; clé React à base d'`index` (utiliser `message.id`) ; reparse markdown/regex
  O(n²) ; `IntersectionObserver` sans `root` (donner le conteneur scrollable). (PERF-*, ABC-3555)
- **Troncature/limite silencieuse** : `substring(0, MAX)` sans avertir → prévenir/bloquer, pas couper. (ABC-3561)

## Conventions repo à connaître (rappel)

- **Poste Windows, pas de `make`** : lire le `Makefile` et lancer la commande réelle (pytest, etc.) ;
  timeout explicite 5 min pour les suites longues.
- **<API_REPO> (API)** : Poetry ; `make verify`=pytest ; prompts LLM en **Langfuse + fallback markdown**
  (`PROMPT_SOURCE=langfuse`) ; accès DB `user_management` en **asyncpg** ; cache TTL façon `get_context_lg2_code`.
- **Base `user_management` partagée, pas d'Alembic** : fournir un **script SQL** (`db/migrations/` ou `docs/`)
  + coordonner avec l'équipe plateforme pour l'exécuter DEV/QAL/PROD. Base checkpoints LangGraph séparée.
- **<FRONT_REPO>** : Next.js 15, **pnpm** (régénérer `pnpm-lock.yaml` si deps changent), Jest jsdom,
  next-intl, Zustand + TanStack Query ; streaming = fetch+ReadableStream JSONL (pas SSE strict).
- **Git** : jamais de commit direct sur `main`/`qal`/`prod` → branche + PR Azure DevOps ; jamais de
  déploiement qal/prod sans accord explicite ; ne jamais écrire de secret dans un fichier versionné.
- **Backlog d'audit** existant : `<PRODUCT_A>/AuditFable/*.md` + `tickets_audit.csv` (contexte de dette par repo).

## Réflexe préventif

Quand un bug révèle une cause racine systémique (ex. clés i18n divergentes entre locales, `except`
aveugles), le **signaler comme ticket adjacent** (ex. test de parité des clés i18n, règle ruff `BLE001`)
plutôt que de le traiter à la main à chaque occurrence — hors périmètre du ticket courant, mais noté.

## Exemple de référence (gold standard)

- `specs/ABC-3419/` — création de table + CRUD, front+backend, décision de schéma validée.
- `specs/ABC-3558/` — feedback : front + contrat backend (`trace_id`), décision de périmètre actée.
- `specs/Fred-8/` — ticket **XL** décomposé en `subtickets.md`.
- `specs/ABC-3565/` — fix backend précis (contrat HTTP), tests mis à jour.
Relis l'un d'eux pour caler le niveau de détail et le ton avant de produire un nouveau ticket.
