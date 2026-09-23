# Template — commentaire Jira « Spec technique vN »

Coller tel quel dans l'éditeur Jira (les `## ` deviennent des titres, `1.`/`-` des listes).
Supprimer les sections vides sauf 1, 2, 3, 7, 8. Remplacer tout ce qui est entre chevrons.

```
Spec technique v<N> — <titre court du ticket>
v<N> : <si N>1 : « intègre la review de <prénom> — changements : … », sinon supprimer cette ligne>
Repos resynchronisés sur origin/<défaut> le <JJ/MM/AAAA>.

## 1. Ce que j'ai compris
<WHY/WHAT reformulés en 3 lignes. Périmètre : ce qui est dedans, ce qui est explicitement dehors.>
Hypothèses prises (en attendant les réponses du §9) :
- H1 : <hypothèse> → conséquence si fausse : <…>
- H2 : …

## 2. Existant vérifié dans le code
Écarts ticket / code :
- <ce que dit le ticket> ≠ <ce que fait le code> (<repo>/<chemin>:<ligne>)
Producteur : <pipeline / table / fichier, repo, chemin, cadence, cardinalité recomptée>
Consommateur(s) : <API / indexer / front, chemin complet jusqu'à l'usage, chemins de fichiers>
Patterns à réutiliser : <fichier de référence pour la même classe de problème>
Chiffres : <n contextes / lignes / locales, comptés sur origin/<défaut>>

## 3. Options étudiées et décision
| Option | Principe | Pour | Contre | Repos touchés |
|---|---|---|---|---|
| A | … | … | … | … |
| B | … | … | … | … |
| C | … | … | … | … |
Décision : Option <X> (validée avec l'utilisateur le <JJ/MM>). <Pourquoi X, pourquoi pas les autres, en 3-5 lignes.>
<Si structurant : « ADR à prévoir à l'implémentation : <sujet> ».>

## 4. Architecture cible
<Flux texte : source → transformation → stockage → API → front. Une ligne par étape, repo entre parenthèses.>

## 5. Détail technique
Modèle de données : <table / colonnes / types / contraintes, ou « inchangé »>
Contrat d'API / inter-repos : <endpoint, champs, types, valeurs par défaut, comportement si absent — identique dans les deux repos>
Algo / règle : <pseudo-code court ou règles numérotées>
Migration / coexistence : <scripts SQL dans docs/migrations, seed, période où ancien et nouveau coexistent, rollback>
Comportement dégradé : <donnée manquante, service indisponible : ce que voit l'utilisateur>

## 6. Sécurité
<Secrets (jamais dans le code), validation des entrées, moindre privilège, pas de nouveau point d'exposition — ou « aucun impact »>

## 7. Tests ↔ DoD
| Case du DoD | Test technique | Où |
|---|---|---|
| <case 1> | <test nommé, ce qu'il prouve> | <repo>/<chemin> |
| Aucune régression <…> | <test golden / non-régression> | … |
| <test manuel QAL si nécessaire> | <checklist> | hors CI |

## 8. Découpage de l'implémentation
<Par repo, dans l'ordre de déploiement. Une ligne = une tâche vérifiable.>
Repo <A> (d'abord) :
1. …
Repo <B> :
2. …
Ordre de déploiement : <A → B → validation QAL → nettoyage>. Coexistence : <…>.

## 9. Questions métier
@<Prénom Nom du reporter>
- Q1 : <question> — hypothèse prise en attendant : H1.
- Q2 : …

## 10. Pour le refinement
- <Repos réellement touchés + équipe plateforme si besoin → estimation actuelle <n> pts à revoir. Jamais bloquant.>

## 11. Tickets connexes suggérés (hors périmètre de ce ticket)
- <titre court> : <ce que ça impacte> ; <pourquoi hors ticket>.

## 12. Points ouverts pour les reviewers
- <ce sur quoi je veux explicitement un avis>
```

## Commentaire « Réponse à la review » (mode Répondre)

```
@<Prénom Nom du reviewer> merci pour la review. Réponses point par point, la v<N+1> complète suit.

Bloquant
1. <résumé du point> — Accepté : <ce qui change, renvoi §x de la v<N+1>>.
2. <résumé> — Refusé : <pourquoi, preuve chemin:ligne>.
Important
3. <résumé> — Accepté partiellement : <…>.
4. <résumé> — Question : <ce qu'il faut pour trancher>.
Mineur
5. <résumé> — Accepté.
```
