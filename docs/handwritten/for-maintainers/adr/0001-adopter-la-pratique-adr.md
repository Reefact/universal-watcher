# ADR-0001 | Adopter la pratique ADR

| Champ | Valeur |
| - | - |
| Statut | Accepté |
| Proposé | le 2026-08-20 à 18:21 (UTC+02:00) |
| Accepté | le 2026-08-21 à 09:25 (UTC+02:00) |

<!-- Aucun amendement à ce jour. Au premier, décommenter ce bloc et
     ajouter la ligne a1 avec le numéro de sa PR.

### Amendements

| Amendement | Pull Request |
| - | - |
| — | [#78](https://github.com/Reefact/universal-watcher/pull/78) |

-->

## Contexte

Le code de ce dépôt est écrit avec l'assistance d'une IA. Un agent ne conserve
aucune mémoire d'une session à l'autre : ce qui n'est pas écrit dans le dépôt
n'existe pas pour lui à la session suivante, et une décision qui n'y figure pas
est redérivée de zéro — sans garantie qu'elle le soit à l'identique. Il produit
par ailleurs une justification convaincante pour ce qu'il vient d'écrire, y
compris quand cela contredit un choix antérieur.

Le développement va produire des décisions techniques qui engagent la suite :
prendre une dépendance, arrêter ce que la CI vérifie, poser une convention qui
lie au-delà d'un fichier, trancher un arbitrage de sécurité. Le choix retenu se
lit dans le code ; ce qui a été comparé et pourquoi il a gagné ne s'y lit nulle
part.

`specifications.md` porte aujourd'hui les décisions déjà prises, faute d'un autre
endroit où les mettre : c'est le seul document que le projet possède. C'est un
document de travail écrit avant que le code n'existe, et le développement va le
confronter au réel — dans une approche itérative, ces décisions bougeront. Il les
énonce en prose continue, sans contexte isolé, sans alternatives, sans
conséquences, sans statut ni mécanisme de supersession.

## Décision

Ce dépôt tient un journal de décisions d'architecture (ADR) sous
`docs/handwritten/for-maintainers/adr/`, au gabarit défini dans `template.md`.

## Justification

Le gabarit sépare le fait (Contexte) de l'argument (Justification), ce qui rend
une décision relisible et rouvrable sans avoir à en reconstituer l'historique
ailleurs. Le statut *Proposé → Accepté → Remplacé* permet de faire évoluer une
décision — y compris une décision aujourd'hui énoncée dans `specifications.md` —
sans réécrire l'histoire : l'ancien ADR reste lisible, daté, et pointe vers
celui qui le remplace.

Un journal séparé porte ce que le code ne peut pas porter. Le choix se lit dans
un `.csproj` ou dans un workflow ; l'arbitrage qui l'a produit ne s'y lit nulle
part, et un commentaire qui le porterait disparaîtrait avec le refactor qu'il
annotait. Un ADR survit indépendamment du code qu'il décrit — ce qui est
exactement la propriété recherchée quand l'implémentation change alors que la
décision tient.

Face à un agent, ce journal joue un rôle que sa seule valeur d'archive ne
recouvre pas : il porte les décisions déjà prises sous une forme qu'il peut lire
au moment de coder, plutôt que de le laisser les retrancher à chaque session.
Une décision *datée d'avant* est aussi ce qui empêche qu'elle soit rouverte
implicitement — un agent argumente de façon convaincante en faveur de ce qu'il
vient d'écrire, et un enregistrement antérieur est ce à quoi on peut l'opposer.

## Alternatives envisagées

### Consigner les décisions techniques directement dans les issues GitHub

Envisagée parce que le suivi du travail se fait déjà par issues, sans outillage
supplémentaire à mettre en place.

Écartée : une issue se ferme et se perd dans l'historique ; rien n'indexe
« pourquoi tel choix a été fait » par sujet, et rien ne représente formellement
qu'une décision en remplace une autre.

### Un gabarit allégé, sans Alternatives ni Conséquences

Envisagée parce que les premières décisions attendues (bibliothèque GraphQL,
framework de tests) semblaient ne peser qu'une ou deux options.

Écartée : le gabarit complet a été explicitement retenu plutôt qu'une version
allégée, pour que la forme reste la même que le nombre d'alternatives réelles
soit une ou plusieurs.

### Ne tenir aucun registre séparé, tout modifier dans `specifications.md`

Envisagée parce que la spécification est déjà le document de référence du projet.

Écartée : un document de travail n'a pas de mécanisme de supersession — le
modifier efface la trace de ce qui était énoncé avant. C'est exactement le
problème que ce journal résout.

## Conséquences

### Positives

* Un mécanisme daté pour faire évoluer une décision, y compris une décision
  aujourd'hui énoncée dans `specifications.md`, sans perdre la trace de ce qui
  précédait.

### Négatives

* Une section supplémentaire du dépôt à tenir à jour.
* L'articulation formelle entre `specifications.md` et ce journal — en particulier
  un éventuel numéro de version de la spécification — reste à définir.

### Risques

* **Rien ne force un agent à consulter ce journal.** Aucun mécanisme ne vérifie
  qu'une décision enregistrée a été lue avant qu'on écrive du code qui la
  contredit. Le journal informe, il ne contraint pas : une décision correctement
  enregistrée peut être ignorée sans que rien ne le signale.

## Actions de suivi

* **Rendre ce journal atteignable depuis la couche chargée à chaque session** —
  suivi en [#74](https://github.com/Reefact/universal-watcher/issues/74). Un
  ADR n'est lu que si on va le chercher ; `CLAUDE.md` est le seul endroit où
  une obligation tient sans avoir été cherchée. Ce qu'il faut y mettre, et
  surtout ce qu'il ne faut pas y recopier, est spécifié dans l'issue.
* **Outiller la consultation du journal, le jour où il le faudra** — suivi en
  [#79](https://github.com/Reefact/universal-watcher/issues/79), fermée
  `not planned` à la rédaction de cet ADR : non planifiée, pas abandonnée. On
  n'outille pas contre un problème supposé ; on rouvre dès qu'une décision
  enregistrée est effectivement contredite. Les pistes envisagées et leur coût sont consignés
  dans l'issue, pour ne pas avoir à refaire ce raisonnement à ce moment-là.

## Références

* `specifications.md` §50 à §54.
* Issue [#75](https://github.com/Reefact/universal-watcher/issues/75).
* [ADR : Le chaînon manquant](https://youtu.be/LdmOaSDn000?si=k7wdXMvwzzbx-CT_)
