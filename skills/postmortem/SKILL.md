---
name: postmortem
description: Rédige un postmortem d'incident blameless — reconstitue la timeline depuis git, les pipelines et la session, identifie la cause racine et propose des actions préventives. Utiliser après un incident résolu, quand l'utilisateur demande un postmortem ou un REX.
user_invocable: true
argument-hint: "[description courte de l'incident + période]"
---

Tu rédiges un postmortem **blameless** (on cherche les causes systémiques, jamais les coupables). Incident : $ARGUMENTS.

## ÉTAPE 1 — Cadrer

Si la description ne permet pas de répondre à ces 4 questions, pose-les d'un coup avant de continuer :
- Quel système/produit était impacté, et comment (indispo, latence, données fausses) ?
- Quand ça a commencé, quand ça a été résolu ?
- Qui/quoi a détecté le problème (alerte, utilisateur, hasard) ?
- Qu'est-ce qui a résolu l'incident ?

## ÉTAPE 2 — Reconstituer la timeline (preuves, pas souvenirs)

Croise les sources disponibles sur la période :
1. `git log --all --oneline --since=... --until=...` sur les repos concernés (commits, merges, reverts, hotfixes).
2. Runs de pipelines (MCP azure-devops ou `az pipelines runs list`) : déploiements et échecs, avec horodatage.
3. PRs mergées juste avant le début de l'incident (suspects n°1).
4. Ce que la conversation courante ou les handoffs récents disent de l'incident.

Chaque événement de la timeline doit avoir une source vérifiable (SHA, run ID, PR ID).

## ÉTAPE 3 — Analyser

- **Cause racine** : méthode des 5 pourquoi, en t'arrêtant sur une cause *actionnable* (pas "erreur humaine" — pourquoi le système a laissé passer l'erreur ?).
- **Facteurs aggravants** : détection tardive, absence d'alerte, rollback difficile, doc manquante.
- **Ce qui a bien fonctionné** : à garder.

## ÉTAPE 4 — Rédiger

```
# Postmortem — <titre> (<date>)

## Résumé (3 phrases : quoi, impact, résolution)
## Impact (durée, systèmes, utilisateurs/données affectés)
## Timeline (heure — événement — source)
## Cause racine
## Facteurs aggravants
## Ce qui a bien fonctionné
## Actions préventives (chacune : action concrète, priorité, candidate à devenir ticket)
```

Propose de créer les tickets des actions préventives (via MCP azure-devops) après validation. Écris le doc là où l'utilisateur le veut ; par défaut dans le repo concerné sous `docs/postmortems/YYYY-MM-DD-<slug>.md`.
