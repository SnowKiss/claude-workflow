---
name: nom-en-kebab-case
description: >
  Une phrase qui dit CE QUE fait la skill, puis QUAND l'utiliser, avec les formulations
  réelles de l'utilisateur. C'est ce texte qui décide du déclenchement automatique —
  écris-le pour être reconnu, pas pour être joli.
user_invocable: true
argument-hint: "[ce qu'on peut passer en argument]"
---

Tu <verbe d'action> : $ARGUMENTS.

<!--
CONVENTIONS QUI COMPTENT

1. Adresse-toi à l'agent à la deuxième personne, à l'impératif. Pas de "cette skill permet
   de" : personne ne lit une skill, elle est exécutée.

2. Numérote les étapes et rends-les vérifiables. "Analyse le code" n'est pas une étape ;
   "lis le diff, puis ouvre en entier chaque fichier modifié" en est une.

3. Mets les garde-fous DANS la procédure, pas dans une section "attention" à la fin. Une
   règle placée après l'action qu'elle encadre arrive trop tard.

4. Dis explicitement ce que l'agent ne doit PAS faire, et ce qu'il doit faire à la place.
   Les interdits sans alternative sont contournés.

5. Prévois le cas où ça bloque : que faire quand une commande échoue, quand l'information
   manque, quand l'utilisateur n'est pas là pour arbitrer.
-->

## ÉTAPE 1 — Cadrer

Si <l'information nécessaire> manque, pose les questions d'un coup avant de continuer
plutôt que de deviner. Liste-les ici.

## ÉTAPE 2 — Collecter les faits

Commandes concrètes, avec ce qu'on en attend :

```bash
# une commande par chose à savoir
```

Chaque constat doit être rattaché à une source vérifiable (SHA, identifiant, chemin de
fichier). Ce qui n'est pas sourcé est une hypothèse et doit être annoncé comme telle.

## ÉTAPE 3 — Agir

Ce qui est autorisé, et ce qui exige une validation :

- **Sans demander** : <actions réversibles et locales>
- **Avec accord explicite dans le message courant** : <actions irréversibles ou visibles
  de l'extérieur — publication, déploiement, envoi, suppression>

Un accord donné plus tôt dans la conversation ne vaut pas pour l'action en cours.

## ÉTAPE 4 — Restituer

Format de sortie attendu, court et exploitable. Précise :
- ce qui est affiché à l'utilisateur
- ce qui est écrit sur disque, et où
- ce qu'il reste à décider, formulé pour être validable d'un mot

## Si ça bloque

<Quoi faire quand l'étape N échoue : s'arrêter et expliquer, ou continuer en dégradé.>
Ne jamais masquer un échec pour produire un résultat présentable.
