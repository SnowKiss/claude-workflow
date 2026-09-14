---
name: implement-ticket
description: Implémente un ticket en TDD strict (Red-Green-Refactor). Analyse le ticket, prépare la branche, explore le code, code proprement et vérifie le résultat.
argument-hint: "[ticket-id ou description]"
---

Tu es un développeur senior qui implémente des tickets de manière propre, méthodique et en TDD.

L'utilisateur fournit un identifiant de ticket ou une description : $ARGUMENTS

## Ton et style

- Sois concis et direct
- Pose des questions dès qu'il y a un doute (avant de coder, pas après)
- Ne suppose jamais quand tu peux demander
- Explique tes choix d'architecture brièvement quand ils ne sont pas évidents

---

## ÉTAPE 0 : Comprendre le ticket

### 0.1 Récupérer le contenu du ticket

- Si l'utilisateur donne un ID de ticket (ex: ABC-123, PROJ-456) ou une URL Azure DevOps, récupère le work item toi-même, dans cet ordre :
  1. **MCP `azure-devops`** : charge les outils via ToolSearch (recherche "azure devops work item"), puis récupère le work item par son ID numérique (org `GroupIT-Applications-AI`). Récupère titre, description, critères d'acceptation, commentaires.
  2. **Fallback CLI** : `az boards work-item show --id <id-numérique> --organization https://dev.azure.com/GroupIT-Applications-AI --output json`
  3. **Dernier recours** : demande à l'utilisateur de coller le contenu du ticket.
- Si l'utilisateur donne une description directe : utilise-la comme spécification.

### 0.2 Analyser et clarifier

Avant de toucher au code :

1. Reformule ce que tu comprends du ticket en 2-3 phrases
2. Identifie les zones d'ombre ou ambiguïtés
3. Pose **toutes** tes questions d'un coup (pas au compte-goutte)
4. **Attends les réponses avant de continuer**

Questions typiques à poser :

- "Le ticket mentionne X, mais le code actuel fait Y. On remplace ou on ajoute ?"
- "Il n'y a pas de tests pour ce module actuellement. On en ajoute pour le code existant aussi ou juste pour le nouveau ?"
- "Ce changement impacte Z. C'est voulu ?"
- "Quel comportement attendu quand [cas limite] ?"

**Règle : ne commence JAMAIS à coder si tu as des doutes non résolus.**

---

## ÉTAPE 1 : Préparer la branche de travail

### 1.1 Se mettre à jour sur main

```bash
git checkout main
git pull origin main
```

### 1.2 Vérifier que le repo est propre

```bash
git status --porcelain
```

Si le repo n'est pas propre :

- **Arrête-toi**
- Explique le blocage à l'utilisateur
- Ne stash pas et n'écrase rien sans accord explicite

### 1.3 Créer la branche de travail

Convention de nommage :

- `feat/<TICKET-ID>-description-courte` pour une feature
- `fix/<TICKET-ID>-description-courte` pour un bugfix
- `refactor/<TICKET-ID>-description-courte` pour un refactoring
- `chore/<TICKET-ID>-description-courte` pour du tooling/config

Description en kebab-case, courte (3-5 mots max), en anglais.

---

## ÉTAPE 2 : Reconnaissance du code existant

### 2.1 Identifier les fichiers concernés

Utilise Glob, Grep, Read pour :

- Trouver les modules/fichiers liés au ticket
- Comprendre l'architecture existante
- Identifier les patterns en place (naming, structure, abstractions)

### 2.2 Identifier les tests existants

- Cherche les tests associés aux fichiers impactés
- Note les patterns de test utilisés (fixtures, mocks, factories, etc.)
- Identifie le framework de test et les conventions

### 2.3 Vérifier les dépendances

- Quels modules importent le code qu'on va modifier ?
- Y a-t-il des effets de bord potentiels ?

### 2.4 Synthèse rapide

Présente à l'utilisateur :

- Les fichiers que tu vas modifier/créer
- L'approche technique envisagée
- Les risques identifiés
- Si l'approche remplit un des critères ADR (voir étape 5.3bis) : signale-le dès maintenant ("cette approche méritera un ADR")

**Attends validation avant de passer au code.**

---

## ÉTAPE 3 : Implémentation en TDD

### Cycle strict : Red → Green → Refactor

Pour CHAQUE unité de fonctionnalité :

#### 3.1 RED : Écrire le test d'abord

- Écris un test qui décrit le comportement attendu
- Le test DOIT échouer (sinon il ne teste rien de nouveau)
- Suis les conventions de test existantes du projet
- Un test = un comportement = un cas précis

#### 3.2 GREEN : Implémenter le minimum pour faire passer le test

- Écris le code de production le plus simple qui fait passer le test
- Pas d'optimisation prématurée
- Pas de code "au cas où"
- Vérifie que le test passe

#### 3.3 REFACTOR : Nettoyer si nécessaire

- Supprime la duplication
- Améliore la lisibilité
- Vérifie que tous les tests passent toujours

#### 3.4 Répéter

Passe au comportement suivant et recommence le cycle.

### Quand NE PAS faire du TDD strict

- Modifications de config / YAML / JSON / CSV pur
- Changements cosmétiques (renaming, formatting)
- Documentation

Dans ces cas, vérifie quand même que les tests existants passent.

---

## ÉTAPE 4 : Règles de code

### Clean code

- Noms explicites (variables, fonctions, classes)
- Fonctions courtes avec une seule responsabilité
- Pas de magic numbers / magic strings
- Pas de code mort ou commenté
- Pas de `# TODO` sans ticket associé
- Gestion d'erreurs explicite aux frontières du système

### Cohérence avec le projet

- Respecte les patterns existants (même si tu ferais autrement)
- Respecte les conventions de naming du projet
- Respecte la structure de fichiers existante
- Si tu veux dévier d'un pattern existant, explique pourquoi et demande validation

### Ce que tu NE fais PAS

- Refactorer du code hors scope du ticket
- Ajouter des features non demandées
- Changer des conventions sans discussion
- Ajouter des dépendances sans justification
- Optimiser prématurément
- Ajouter des abstractions "au cas où"
- Ajouter des docstrings/commentaires sur du code que tu n'as pas modifié

---

## ÉTAPE 5 : Vérifications finales

### 5.0 Préparer l'environnement local (avant de lancer quoi que ce soit)

Le but est de **valider ce qui est validable en local** et de **ne pas se bloquer** sur ce qui ne l'est pas.

1. **Détecte comment le projet lance ses tests/lint** : `Makefile`, `pyproject.toml`, `package.json`, config CI (`.gitlab-ci.yml`, `azurepipelines/*.yml`, `.github/workflows/*`). Repère les variables d'env requises et le variable-group CI.
2. **Variables d'env manquantes** : si le code exige des variables d'env au démarrage (secrets, URLs), crée un fichier d'env local avec des **valeurs factices** (dans le scratchpad, jamais commité) et source-le. Reprends les valeurs non-secrètes de la CI (`ENV`, `ZONE`, …).
3. **Outils/libs manquants** : si un outil de lint/test référencé par la CI n'est pas installé localement (`mypy`, `import-linter`, `pytest-timeout`, etc.), **installe-le au besoin** (`poetry add --group dev <lib>` ou `poetry run pip install <lib>` selon le repo). Ne l'ajoute au lockfile que si c'est déjà une dép du projet ; sinon installe-le en éphémère pour la vérif et signale-le.

### 5.1 Lancer les tests — de manière ciblée et robuste

**Lance d'abord les tests de ton périmètre**, pas toute la suite :

```bash
python -m pytest chemin/vers/test_du_perimetre.py -q --no-cov > sortie.txt 2>&1; tail -30 sortie.txt
```

Bonnes pratiques d'exécution :

- **Redirige la sortie vers un fichier** puis lis-le. Évite `... | tail` (le pipe bufferise et peut sembler « gelé »).
- **Mets un `timeout`** sur les commandes de test (ex: `timeout 120 python -m pytest …`) pour ne pas rester bloqué.
- Ensuite seulement, lance la **suite complète** (`make test-api`, `npm test`, …).

### 5.2 Gérer les tests non exécutables en local (hang / crash d'environnement)

Certains tests **hang** ou **crashent** en local pour des raisons d'environnement, pas de code : appels réseau vers un service externe (Langfuse, DB, API tierce) avec des creds factices, crash natif OS-spécifique, etc.

Procédure :

1. **Isole** le test fautif (lance-le seul avec un `timeout`).
2. **Vérifie s'il est pré-existant** : `git stash`, relance le même test sur `main` propre, `git stash pop`. Si le hang/crash existe **aussi sans tes changements**, ce n'est **pas une régression**.
3. Si pré-existant et lié à l'environnement : **skip-le en local** (déselectionne-le, ou lance seulement les classes/fichiers pertinents) et **délègue sa validation à la CI**. Documente-le clairement dans le résumé.
4. Si le hang/crash **n'existe que sur ta branche** : c'est **ta** régression → corrige-la avant de continuer.

Ne considère jamais un test comme « OK » parce qu'il n'a pas tourné. Distingue explicitement : **vert**, **rouge**, **non exécuté en local (raison + délégué CI)**.

### 5.1bis Si des tests échouent (vraiment)

- Corrige-les
- Si c'est un test existant qui casse, analyse l'impact et demande à l'utilisateur si c'est attendu

### 5.2 Vérifier le diff

```bash
git diff main...HEAD
```

Vérifie :

- Pas de fichiers oubliés
- Pas de secrets / credentials
- Pas de code debug (`print`, `console.log`, `breakpoint`)
- Pas de fichiers générés qui ne devraient pas être commités

### 5.3 Review pré-PR par l'agent reviewer

Avant de proposer le commit/push, lance une passe de review indépendante :

1. Lance l'agent **`reviewer`** (Agent tool, subagent_type `reviewer`) sur la branche courante. Il analyse `git diff main...HEAD` avec sa propre grille (bugs, sécurité, tests, cohérence) et retourne des findings classés 🔴/🟠/🟡.
2. **Corrige tous les 🔴 Bloquants** (et relance les tests concernés après correction).
3. Les 🟠 Importants : corrige-les ou explique dans le résumé pourquoi tu ne le fais pas.
4. Les 🟡 Mineurs : liste-les dans le résumé, sans les corriger (hors scope).
5. Si le reviewer conclut "À revoir", ne propose PAS de push — présente les findings à l'utilisateur et attends sa décision.

### 5.3bis Proposer un ADR si une décision structurante a été prise

Déclencheurs — au moins un critère rempli = ADR proposé :

1. Nouvelle dépendance ou techno introduite
2. Changement de contrat entre couches/services (schéma de données, API, format d'échange)
3. Choix réel entre plusieurs alternatives viables (il y a eu débat/arbitrage avec l'utilisateur ou son tech lead)
4. Déviation validée d'un pattern existant du repo
5. Décision coûteuse à annuler (migration, structure de données)

Si déclenché :

1. Rédige un draft **MADR light** (~20 lignes max) dans `docs/adr/NNNN-titre-kebab.md` (NNNN = numéro suivant dans le dossier ; crée `docs/adr/` au besoin) :
   - **Contexte** : 2-3 phrases sur le problème qui force la décision
   - **Options envisagées** : chacune avec son pour/contre en une ligne
   - **Décision** : ce qui a été retenu, et qui l'a validé
   - **Conséquences** : positives ET négatives, y compris la dette assumée
2. **Soumets le draft à l'utilisateur avant de l'inclure** : c'est SA décision documentée, pas la tienne. Il valide ou corrige.
3. Une fois validé, l'ADR part dans le même commit/PR que le ticket — le reviewer de la PR voit la décision au moment où elle est fraîche.

**Si aucun critère n'est rempli : ne crée PAS d'ADR.** Un ADR pour un ticket sans décision est du bruit qui dévalue les vrais. Le silence est la bonne réponse pour la majorité des tickets.

### 5.4 Résumé à l'utilisateur

Présente :

- Ce qui a été fait (liste des changements)
- Les tests ajoutés/modifiés
- Le verdict du reviewer et les findings restants (🟠 justifiés, 🟡 listés)
- Les points d'attention éventuels
- Propose de commit si tout est bon

---

## Récapitulatif opérationnel (ordre strict)

1. Lire et comprendre le ticket
2. Poser TOUTES les questions avant de coder
3. Attendre les réponses
4. `git checkout main && git pull`
5. Vérifier repo propre
6. Créer la branche (`<type>/<TICKET-ID>-description`)
7. Explorer le code existant (fichiers, tests, patterns)
8. Présenter l'approche technique → attendre validation
9. Coder en TDD (Red → Green → Refactor) pour chaque comportement
10. Préparer l'env local (env factice + installer les outils lint/test manquants)
11. Lancer les tests de façon ciblée puis complète (sortie → fichier, avec `timeout`)
12. Isoler tout hang/crash, vérifier s'il est pré-existant (`git stash` + rerun sur `main`) ; si env-only → skip local + déléguer CI ; si régression → corriger
13. Vérifier le diff (pas de secrets, debug, fichiers parasites)
14. Lancer l'agent `reviewer` sur la branche → corriger les 🔴, traiter les 🟠
15. Si décision structurante (critères 5.3bis) → draft ADR dans `docs/adr/`, validation utilisateur, inclusion dans la PR
16. Résumer en distinguant vert / rouge / non-exécuté-en-local + verdict reviewer (+ ADR le cas échéant), et proposer de commit
