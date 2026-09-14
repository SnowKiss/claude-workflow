---
name: self-audit
description: Audite l'utilisation de Claude Code par l'utilisateur sur la semaine écoulée (transcripts réels) et propose des améliorations concrètes — prompts, skills, agents, routines, CLAUDE.md. Boucle d'amélioration continue : relit les recommandations de l'audit précédent et vérifie si elles ont produit un effet. Utiliser quand l'utilisateur demande un audit de son utilisation, le bilan hebdo Claude, ou via la tâche planifiée du vendredi.
user_invocable: true
argument-hint: "[nb de jours, défaut 7]"
---

Tu audites la façon dont l'utilisateur a utilisé Claude Code sur les **$ARGUMENTS derniers jours** (défaut : 7). Objectif : lui faire gagner du temps la semaine prochaine, pas produire un joli rapport.

Emplacements :
- extracteur : `~/.claude/skills\self-audit\extract_usage.py`
- journal (mémoire de la boucle) : `~/.claude\audits\JOURNAL.md`
- rapports : `~/.claude\audits\YYYY-MM-DD.md`

## ÉTAPE 1 — Fermer la boucle précédente (à faire AVANT d'analyser)

Lis `JOURNAL.md`. Pour chaque recommandation encore ouverte de l'audit précédent :

1. **Vérifie factuellement** si elle a été appliquée — lis le fichier concerné (skill, `CLAUDE.md`, `settings.json`), ne te fie pas au journal.
2. **Mesure l'effet** sur le digest de cette semaine : la métrique visée a-t-elle bougé ? (ex. « erreurs PowerShell 19 % → 6 % »).
3. Classe : `appliquée + effet mesuré` / `appliquée, effet nul` / `non appliquée` / `devenue obsolète`.

Une reco « appliquée, effet nul » deux semaines de suite se **retire**, elle ne se répète pas. Une reco « non appliquée » deux fois de suite : demande à l'utilisateur si elle l'intéresse, sinon abandonne-la. C'est ce tri qui distingue une boucle d'un rapport hebdo.

## ÉTAPE 2 — Produire le digest

```
cd ~/.claude/skills\self-audit
python extract_usage.py --days 7 --out <scratchpad>/digest.json
```

Écris le digest dans le scratchpad de session, jamais dans un dossier de projet. Puis lis-le.

Ne lis **jamais** les transcripts `.jsonl` bruts directement (~350 Mo/semaine, ils feraient exploser le contexte). Si le digest est insuffisant sur un point précis, cible un `sessionId` et grep ce seul fichier.

## ÉTAPE 3 — Analyser

Le digest contient les prompts réels de l'utilisateur, les outils, les erreurs, les tokens. **Lis les prompts** — c'est la source la plus riche. Les champs `friction` sont des heuristiques regex, donc des hypothèses à confirmer, pas des faits : vérifie sur le texte du prompt avant d'en conclure quoi que ce soit.

Cherche sur les quatre axes :

**Friction & reprises** — Où l'utilisateur corrige, interrompt, refuse, ou répète une consigne. Un même rappel deux fois dans la semaine = une règle manquante dans `CLAUDE.md` ou dans une skill. Regarde aussi `denials` et `interruptions` : une action refusée révèle soit un défaut de cadrage de ma part, soit une permission à ajuster.

**Skills & routines** — Croise `skills_invoked` / `slash_commands` avec les skills existantes (`ls ~/.claude/skills/`) :
- skill jamais invoquée : mal nommée, description qui ne matche pas, ou inutile → renommer, réécrire la description, ou supprimer ;
- séquence de prompts répétée à l'identique sur plusieurs sessions (ex. « mets-toi sur main, pull ») → candidate à une skill ou un alias ;
- skill invoquée puis suivie de corrections → son prompt est à durcir.

**Coût & efficacité** — Sessions à fort volume d'appels pour peu de prompts (travail en boucle), outils appelés en rafale, ratio `cache_read` / `output_tokens`, compactions fréquentes (contexte saturé → tâche mal découpée). Croise `subagents_used` avec les agents disponibles (`ls ~/.claude/agents/`) : une exploration lourde faite en direct plutôt que déléguée à un subagent, c'est du contexte gaspillé.

**Qualité du résultat** — Tâches reprises plusieurs fois, allers-retours avant le bon résultat, outils à fort taux d'erreur (`error_rate`). Un outil à >10 % d'erreurs signale un usage à corriger (mauvaise syntaxe récurrente, mauvais outil pour le job) — regarde les `tool_error_samples` pour trancher.

Garde-fous d'honnêteté :
- 3 à 5 constats maximum, priorisés par temps réellement gagné. Pas de liste de 15 items tiède.
- Chaque constat s'appuie sur des **chiffres du digest** et au moins un exemple concret (session + extrait de prompt). Pas de constat à l'intuition.
- Une semaine calme est un résultat valide : écris « rien de significatif cette semaine » plutôt que de gratter du remplissage.
- Distingue « je n'ai pas la donnée » de « il n'y a rien ». L'extracteur ne voit pas ce qui se passe hors Claude Code.

## ÉTAPE 4 — Proposer des changements prêts à appliquer

Pour chaque constat, propose **le changement exact**, jamais un conseil vague. « Sois plus précis dans tes prompts » est inutilisable ; « ajouter cette ligne à `CLAUDE.md` : … » est actionnable.

Format par proposition :
- **Constat** — le fait + le chiffre.
- **Preuve** — session/projet + extrait.
- **Changement** — le diff : fichier visé et contenu exact (bloc de code pour une modif de skill ou de `CLAUDE.md`, commande pour une permission).
- **Gain attendu** — et la métrique qui le vérifiera la semaine prochaine.

**N'applique rien de toi-même.** Tu présentes, l'utilisateur valide. C'est sa règle : accord explicite dans le message courant. Une seule exception : écrire le rapport et mettre à jour le journal.

## ÉTAPE 5 — Écrire le rapport et le journal

1. Rapport dans `~/.claude/audits/YYYY-MM-DD.md` : bilan des recos précédentes (étape 1), métriques clés de la semaine, les constats et leurs propositions.
2. Mets à jour `JOURNAL.md` : une ligne par recommandation, avec son statut et la métrique de référence — c'est ce que l'audit suivant relira. Garde-le court (une ligne par reco, les recos closes archivées en fin de fichier).
3. Termine en présentant à l'utilisateur, en français et de façon ramassée : les recos précédentes qui ont marché ou pas, puis les 3-5 propositions de la semaine, chacune validable d'un mot.

Si la tâche tourne sans l'utilisateur devant (planifiée), le rapport reste sur disque et il le lira au démarrage suivant — dis-le explicitement en fin de rapport.
