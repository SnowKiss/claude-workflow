---
name: respond-review
description: Traite les retours de reviewers sur une PR Azure DevOps — se met sur la bonne branche, trie chaque commentaire (corrige si pertinent, répond sinon), applique les corrections, puis poste les réponses et clôture les threads. À utiliser quand l'utilisateur veut "prendre en compte les retours" d'une review sur SA PR.
user_invocable: true
argument: PR ID ou URL
---

Tu prends en charge les retours de review sur une PR Azure DevOps dont l'utilisateur est l'auteur.
Contrairement à `review-pr` (qui reviewe la PR d'un autre), ici tu **réponds** aux reviewers :
tu corriges ce qui est pertinent, tu réponds à ce qui ne l'est pas, tu clôtures les threads.

L'utilisateur fournit une PR ID ou URL : $ARGUMENTS

## Principes

- Tutoie les reviewers, sois concis et direct, pas de blabla, réponds en français.
- **Trie chaque retour** : soit c'est pertinent → tu corriges ; soit ça ne l'est pas → tu réponds
  avec une justification claire. Ne corrige jamais "pour faire plaisir" si c'est faux ou hors scope.
- **Respecte les règles git de l'utilisateur (strictes) :**
  - Avant tout commit : lance les tests concernés (ou dis pourquoi il n'y en a pas) + résumé du diff.
  - Jamais de commit sur `main`/`qal`/`prod` : on reste sur la branche source de la PR.
  - **Demande l'accord explicite AVANT de commiter, pusher et poster des commentaires.** Un accord
    général ne suffit pas — c'est une action sortante.
  - Jamais de déploiement vers qal/prod.
- Outils : privilégie les outils MCP `azure-devops` (fiables, pas de token à gérer). `az devops invoke`
  en fallback si le MCP est indisponible.

## Variables par défaut

```
ORG_URL   = https://dev.azure.com/GroupIT-Applications-AI
PROJECT   = <YOUR_REPO>
REPO      = <YOUR_REPO>
ME        = <YOUR_EMAIL>
```
Adapte `PROJECT`/`REPO` si la PR est sur un autre autre repo de la même organisation (déduis-le de l'URL).

---

## ÉTAPE 0 — Récupérer la PR et les retours

1. Extrais le `PR_ID` depuis l'URL (`.../pullrequest/1038` → `1038`).
2. Récupère la PR : `repo_get_pull_request_by_id` (repositoryId=REPO, project=PROJECT, pullRequestId=PR_ID).
   Note : `sourceRefName`, `targetRefName`, `mergeStatus`, auteur.
3. Récupère les threads : `repo_list_pull_request_threads`. Pour un thread multi-commentaires,
   déplie avec `repo_list_pull_request_thread_comments`.
4. Récupère le diff si besoin de contexte : `repo_get_pull_request_changes`.

Ne retiens que les threads **de reviewers** (pas les threads système "Policy status…", pas tes propres
threads déjà résolus). Un thread avec `threadContext` est ancré sur un fichier/ligne ; un thread sans
`threadContext` est un commentaire général (souvent la synthèse).

---

## ÉTAPE 1 — Se positionner sur la bonne branche (obligatoire)

1. Vérifie que tu es dans le bon repo local (`git remote -v` doit pointer sur REPO).
   Sinon : arrête et explique.
2. Vérifie que le repo local est propre (`git status --porcelain`). S'il est sale : **n'écrase rien**,
   arrête et explique le blocage à l'utilisateur.
3. `git fetch --all --prune`, puis checkout de la branche source (`sourceRefName` sans `refs/heads/`) :
   ```bash
   git checkout <BRANCHE_SOURCE> && git pull --ff-only origin <BRANCHE_SOURCE>
   ```
   Fallback si absente en local : `git fetch origin <BRANCHE_SOURCE>:<BRANCHE_SOURCE>` puis checkout.
4. Si le checkout échoue : arrête et explique.

---

## ÉTAPE 2 — Trier chaque retour

Pour CHAQUE thread de reviewer, décide :

| Catégorie | Action |
|-----------|--------|
| **Bug réel / bloquant** (conflit de merge, régression, faille) | **Corriger** |
| **Amélioration pertinente** (refacto utile, test manquant, doc manquante) | **Corriger** si raisonnable et dans le scope |
| **Nit / cosmétique** | Corriger si trivial, sinon répondre |
| **Faux positif / trade-off assumé / hors scope** | **Répondre** avec justification, ne pas coder |
| **Question** | Répondre (et corriger si la réponse révèle un vrai souci) |

Avant de coder, **lis le code réel dans son contexte** (Read/Grep) pour confirmer que le retour est
fondé. Vérifie les faits toi-même (ex. si un reviewer parle d'un outillage de release, ouvre le fichier
de config au lieu de supposer).

**Cas fréquent — branche en retard / conflit de merge :** si `mergeStatus` indique un conflit ou si un
reviewer le signale, merge la cible dans la branche source et résous les conflits proprement
(`git merge origin/<TARGET>`), en réconciliant chaque fichier selon son intention réelle (ne "prends" pas
aveuglément un côté). Attention : un diff branche↔cible en 2-points peut faire apparaître de fausses
suppressions ; le vrai diff de la PR est en 3-points (`git diff origin/<TARGET>...HEAD`).

Produis une **table de tri** que tu montreras à l'utilisateur (thread → verdict corriger/répondre → 1 ligne).

---

## ÉTAPE 3 — Appliquer les corrections

- Fais les modifs par intention (helper, test, doc, résolution de conflit…).
- Regroupe logiquement : idéalement un commit de merge séparé du commit de corrections de revue.
- **Lance les tests concernés** (cible le module touché plutôt que toute la suite si possible).
  Sur ce repo (<YOUR_REPO> / API) les tests exigent des variables d'env fournies par la CI ; en local
  sans `.env`, exporte des valeurs factices pour les vars requises (`ENV`, `ZONE`, `REDIS_URL`,
  `AZURE_OPENAI_*`, `AZURE_SEARCH_KEY`, `LANGFUSE_*`, `PROMPT_SOURCE=local`) — les tests unitaires
  mockent la DB. Lance via `poetry run python -m pytest <fichier> -q`.
- Passe mypy/lint sur les fichiers modifiés si pertinent. **Distingue les erreurs préexistantes**
  (hors de ton diff) des erreurs que tu introduis : ne corrige que les tiennes, signale les autres.

---

## ÉTAPE 4 — Point d'arrêt : accord explicite

Montre à l'utilisateur :
1. La table de tri (corrigé / répondu, par thread).
2. Le résumé du diff (`git diff --stat` de tes fichiers, hors fichiers ramenés par un merge).
3. Le résultat des tests.

Puis **demande l'accord** sur la finalisation, en proposant les options :
- commit (merge + corrections) + push de la branche + réponses/clôtures sur les threads ;
- commit + push sans commentaires ;
- commit local seulement.

N'exécute la suite qu'après réponse explicite. (Utilise AskUserQuestion si utile.)

---

## ÉTAPE 5 — Commit & push (si accordé)

- Si un merge est en cours : commit du merge d'abord (`git commit --no-edit`), puis un commit dédié
  pour les corrections de revue avec un message clair `refactor/fix(...): address PR <ID> review feedback`
  et le co-author habituel.
- **Attention shell** : ce repo tourne sous Git Bash. Pour un message multi-ligne, utilise un heredoc
  bash `git commit -m "$(cat <<'EOF' … EOF)"` — **pas** la syntaxe PowerShell `@'…'@` (elle laisse un `@`
  parasite dans le message).
- Push sur la branche source uniquement : `git push origin <BRANCHE_SOURCE>`. Jamais sur main/qal/prod.

---

## ÉTAPE 6 — Répondre et clôturer les threads (si accordé)

Pour chaque thread traité, poste une réponse avec `repo_reply_to_comment`
(repositoryId=REPO, project=PROJECT, pullRequestId=PR_ID, threadId=…, content=…).

Contenu des réponses :
- **Corrigé** : dis précisément ce que tu as changé (nom du helper/test/fichier), en 1-3 lignes.
- **Répondu (pas corrigé)** : explique le trade-off ou pourquoi c'est hors scope / faux positif.
- Réponds même aux compliments/synthèses par une courte confirmation.

Puis mets à jour le statut du thread avec `repo_update_pull_request_thread` :
- `Fixed` — corrigé.
- `ByDesign` — comportement volontaire, non corrigé après justification.
- `WontFix` — valable mais on ne le fait pas (hors scope, dette assumée).
- Laisse `Active` un thread de synthèse ou une question ouverte qui appelle une suite.

---

## ÉTAPE 7 — Synthèse finale à l'utilisateur

Réponse courte :
- combien de threads corrigés vs répondus ;
- l'état des tests ;
- commit(s) + push (oui/non) ;
- points restants (ex. `mergeStatus` à reconfirmer côté Azure DevOps, erreurs préexistantes, follow-ups
  hors scope).

---

## Ce que tu NE fais PAS

- Commiter/pusher/poster sans accord explicite dans le message courant.
- "Corriger" un retour faux ou hors scope juste pour clore le thread — argumente et réponds.
- Résoudre un conflit de merge en prenant aveuglément un côté.
- Toucher à des erreurs lint/mypy préexistantes hors de ton diff (sauf demande explicite).
- Pusher sur main/qal/prod ou déployer quoi que ce soit.
