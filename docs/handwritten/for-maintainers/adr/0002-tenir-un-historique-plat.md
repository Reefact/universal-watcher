# ADR-0002 | Tenir un historique plat sur main

| Champ | Valeur |
| - | - |
| Statut | Accepté |
| Proposé | le 2026-08-21 à 09:43 (UTC+02:00) |
| Accepté | le 2026-08-21 à 11:53 (UTC+02:00) |

<!-- Aucun amendement à ce jour. Au premier, décommenter ce bloc et
     ajouter la ligne a1 avec le numéro de sa PR.

### Amendements

| Amendement | Pull Request |
| - | - |
| — | [#83](https://github.com/Reefact/universal-watcher/pull/83) |

-->

## Contexte

Un seul développeur travaille sur ce dépôt, assisté d'agents. Plusieurs sessions
tournent parfois en parallèle, chacune sur sa propre branche, et une branche peut
vivre plusieurs jours avant d'atterrir. Rien n'écrit aujourd'hui comment le travail
atterrit sur `main`.

GitHub offre trois façons d'intégrer une pull request : écrire un commit de fusion
(*merge commit*), écraser la branche en un commit unique (*squash*), ou reporter
ses commits un à un sur la cible (*rebase*). Les deux premières inscrivent d'office
un renvoi vers la pull request dans le message qu'elles produisent et laissent une
entrée qui représente le lot ; la troisième conserve les messages d'origine tels
quels, donne aux commits reportés une nouvelle identité, et ne laisse aucune trace
du lot. Un dépôt peut n'autoriser qu'une de ces trois méthodes.

Le commit de fusion agrège dans son entrée unique l'ensemble des changements du
lot, et il se distingue des autres commits de l'historique. Reporter sur une cible
qui a bougé depuis suppose de reprendre la branche sur son nouvel état.

Un commit déjà présent sur `main` ne s'en retire qu'en réécrivant tous ceux qui le
suivent. Une branche, elle, n'est pas supprimée par son intégration : elle subsiste
jusqu'à une suppression explicite.

Aucune couche n'est chargée au démarrage d'une session d'agent.

## Décision

Une fois les pull requests fermées, il ne reste qu'une branche, `main`, et son
historique est plat : le travail y arrive par report des commits de la branche qui
l'a produit, jamais par un commit de fusion ; les branches de travail sont
supprimées une fois intégrées ; et ni leur nombre ni leur durée de vie ne sont
contraints.

## Justification

La décision porte sur le résultat, pas sur le chemin. Le travail se fait parfois
sur plusieurs branches à la fois, parfois pendant plusieurs jours : contraindre le
nombre ou la durée des branches interdirait une façon de travailler effectivement
pratiquée, et poserait une règle que l'usage contredirait la semaine suivante. Ce
qui est décidé ici ne concerne que ce qui reste une fois les pull requests
fermées — et reste donc vrai si la façon de travailler change encore.

Un historique plat se constate ; une discipline se raconte. Un commit de fusion
sur `main` se voit, sans avoir à connaître l'intention de qui l'a écrit. C'est ce
qui rend la règle opposable à une session qui n'a pas participé à la décision,
là où une convention de durée de branche ne serait jamais qu'une intention
déclarée.

Sur une ligne unique, l'unité de lecture devient le commit. Une branche de
plusieurs jours produit une pull request trop grosse pour se relire d'un bloc ; ce
qui se relit, se bissecte et se révoque alors, c'est le commit — tandis qu'un
commit de fusion, qui agrège tout un lot, renverrait la bissection sur un
changement qu'on ne peut pas lire. L'exigence n'est pas supprimée par cette
décision, elle est déplacée : de la taille du lot vers la granularité et la forme
des commits.

La ligne unique et la branche unique sont deux propriétés distinctes, portées par
deux surfaces distinctes : l'historique d'un côté, la liste des branches de l'autre.
Décider la première sans la seconde laisserait un dépôt à l'historique plat dont la
liste des branches accumule tout ce qui y a été intégré depuis le début. Or, avec
plusieurs branches vivantes à la fois, cette liste est le seul endroit où se lit ce
qui est en cours : une branche intégrée qui y subsiste ne s'y distingue pas d'un
travail en cours, et c'est exactement le signal qu'elle détruit.

Plusieurs branches vivantes n'empêchent pas la ligne unique. Elles obligent
seulement chaque branche à se reposer sur `main` avant d'atterrir : le coût est
réel, il peut aller jusqu'à retrancher des conflits, mais il est payé au moment du
merge et ne laisse rien dans l'historique.

## Alternatives envisagées

### Intégrer par commit de fusion

Envisagée parce qu'elle conserve la frontière du lot : une pull request reste une
entrée unique, qu'on peut relire ou révoquer d'un bloc, et son message renvoie
d'office vers elle.

Écartée : c'est précisément la ramification qu'on ne veut pas. L'historique devient
un graphe, la bissection atterrit sur des fusions, et la lecture à plat n'est plus
qu'une projection parmi d'autres.

### Écraser chaque branche en un commit unique

Envisagée parce qu'elle garantit la ligne unique sans rien demander à la branche —
le désordre y est absorbé au moment du merge — et parce que son message renvoie lui
aussi d'office vers la pull request.

Écartée : elle détruit la granularité qui est le but même. Une branche de plusieurs
jours deviendrait un commit unique, illisible, non bissectable et impossible à
révoquer par morceaux. Elle obtient la forme de l'historique en supprimant son
contenu.

### Conserver les branches intégrées

Envisagée parce que la branche est la seule chose qui garde le lot groupé une fois
l'historique aplati : son nom et la liste de ses commits disent ce que la pull
request contenait, là où `main` ne le dit plus.

Écartée : la pull request porte déjà cette information et ne disparaît pas avec la
branche. Ce que la conservation coûte, en revanche, est immédiat — la liste des
branches cesse de dire ce qui est en cours, ce qui est sa seule utilité quand
plusieurs sessions travaillent en parallèle.

### Contraindre la durée de vie et le nombre des branches

Envisagée parce qu'elle atteint le même résultat par une autre voie — c'est ce que le
trunk-based development impose : des branches assez courtes et assez peu nombreuses
pour ne pas diverger.

Écartée : elle interdirait une façon de travailler effectivement pratiquée, pour un
bénéfice que le report des commits obtient déjà sans rien contraindre en amont.

### Ne rien acter et s'en tenir à l'usage

Envisagée parce que la règle paraît évidente sur un dépôt à un seul développeur, et
parce qu'une décision facile à défaire ne mérite généralement pas d'ADR.

Écartée : l'historique, lui, ne se défait pas facilement — un commit de fusion
arrivé sur `main` ne s'en retire plus sans réécrire ce qui le suit. Et une session sans mémoire —
[ADR-0001](0001-adopter-la-pratique-adr.md) — redérive une convention à chaque
démarrage : une pratique non écrite ne s'oppose à rien.

## Conséquences

### Positives

* Un historique qui se lit comme une suite d'états, où la bissection atterrit sur
  un changement lisible et où révoquer reste une opération simple.
* Une propriété constatable plutôt qu'une discipline déclarée : un commit de fusion
  sur `main` se voit.
* La liste des branches ne montre que le travail en cours, et redevient donc lisible
  d'un coup d'œil même quand plusieurs sessions travaillent en parallèle.
* Le nombre de branches vivantes et leur durée restent libres — ce qui est décidé ne
  porte que sur ce qu'elles déposent.

### Négatives

* **La frontière du lot disparaît de l'historique.** Rien dans `main` ne dit où une
  pull request commence et finit ; cette vue n'existe plus que du côté de GitHub, et
  révoquer une pull request entière redevient une opération à composer commit par
  commit.
* **Le report n'inscrit aucun renvoi vers la pull request**, contrairement aux deux
  autres méthodes d'intégration. Hors de GitHub, rien dans un message de commit ne
  renvoie à ce qui l'a produit ; ce lien, s'il doit exister, s'écrit à la main.
* **Ce qu'une branche dépose sur `main` y arrive tel quel.** Aucune étape n'absorbe
  le désordre : commits d'essai, repentirs et corrections de corrections deviennent
  définitifs au moment du merge.
* **Une branche qui atterrit après une autre doit d'abord se reposer sur `main`.**
  Avec plusieurs branches vivantes, c'est une opération de routine, et elle peut
  demander de retrancher des conflits qu'aucune des deux ne voyait venir.
* **Les commits reportés ne sont pas, au sens de git, ceux qui vivaient sur la
  branche.** Ce qui a été vérifié là-bas ne l'a pas été sur `main`.
* **Reprendre un travail dont la branche a été supprimée suppose de la restaurer
  d'abord.** Son nom cesse par ailleurs d'être un repère disponible dans le dépôt.

### Risques

* **La linéarité est vérifiable, la lisibilité ne l'est pas.** Un historique plat
  fait de messages sans contenu satisfait cette décision sans servir ce qui la
  motive, et rien ne le signale.
* **Rien n'empêche mécaniquement un commit de fusion d'arriver sur `main`.** La
  méthode d'intégration proposée au moment du merge est une préférence, pas un
  interdit : l'historique peut être ramifié par une autre voie, et il ne se répare
  qu'en réécrivant ce qui suit.

## Actions de suivi

* **La forme des messages de commit n'est arrêtée nulle part** — suivi en
  [#80](https://github.com/Reefact/universal-watcher/issues/80). Cette décision
  garantit la ligne unique ; elle ne dit rien de ce qu'on y lit, alors que c'est ce
  qui la rend utile.
* **Rien ne met au propre ce qu'une branche dépose sur `main`** — suivi en
  [#81](https://github.com/Reefact/universal-watcher/issues/81). Le merge ne filtre
  plus : ce qui n'a pas été rangé avant devient définitif.
* **Rien ne rend cette règle opposable côté dépôt** — suivi en
  [#82](https://github.com/Reefact/universal-watcher/issues/82). Ce qu'un réglage de
  dépôt peut interdire, et ce qu'il ne peut qu'inciter, se décide là.
* **La règle n'est atteignable par aucune couche chargée au démarrage d'une
  session** — le travail est déjà suivi en
  [#74](https://github.com/Reefact/universal-watcher/issues/74), qui porte
  `CLAUDE.md`. Reste à trancher si cette règle y satisfait le critère d'admission
  que cette issue fixe.

## Références

* [ADR-0001](0001-adopter-la-pratique-adr.md).
* Issue [#74](https://github.com/Reefact/universal-watcher/issues/74).
