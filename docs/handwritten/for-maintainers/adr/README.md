# Journal des décisions d'architecture (ADR)

Des enregistrements datés des décisions techniques qui engagent ce dépôt : leur
contexte, l'option retenue, ses conséquences. Le journal est historique — une fois
un ADR accepté, sa décision n'est plus modifiée en place. On la révise en écrivant
un **nouvel** ADR qui remplace l'ancien.

## Quand un ADR est-il écrit ?

Chaque fois qu'une décision technique **engage la suite** et qu'on ne peut pas la
relire dans le code.

* **Prendre une dépendance** — bibliothèque GraphQL, framework de tests,
  bibliothèque d'assertions, conteneur d'injection, accès HID, magasin de secrets.
  Le choix se voit dans le `.csproj` ; ce qui a été comparé, et pourquoi celle-là a
  gagné, ne s'y voit pas.
* **Mettre en place ou changer un processus** — ce que la CI vérifie et ce qu'elle
  laisse passer, comment on publie, comment on nomme une branche, ce qui bloque une
  pull request et ce qui se contente d'avertir.
* **Poser une convention qui lie au-delà d'un fichier** — des frontières entre
  projets, un style rendu obligatoire par le build, une règle que d'autres suivront
  sans avoir participé à la discussion.
* **Configurer les agents** — ce que `CLAUDE.md` porte et ce qu'il ne porte pas, où
  vivent les règles, ce qu'un agent a le droit de faire seul.
* **Trancher un arbitrage de sécurité ou de compatibilité** — comment on
  s'authentifie, où vivent les secrets, quelle version plancher on s'impose.

Le test : *quelqu'un risque-t-il de refaire ou de défaire ce choix faute d'en
connaître la raison ?* Si oui, il mérite un ADR. Sinon, le commit suffit.

Corollaire : **une décision facile à défaire n'en mérite généralement pas.** Changer
de bibliothèque de formatage ne coûte rien ; changer de client GraphQL conditionne
la détection des réponses partielles, donc engage bien plus qu'une ligne de
dépendance. C'est l'engagement qui déclenche, pas l'importance apparente.

### Ce qui n'ajoute pas d'ADR

La plupart des changements : corriger un bug, ajouter un test, implémenter ce qui
est déjà décidé, renommer une variable. Une décision qui en remplace une **déjà
portée par un ADR** ne s'ajoute pas non plus — elle s'écrit comme un nouvel ADR qui
le supersède.

## Un ADR enregistre une décision, pas une spécification

Un ADR capture une **décision et son raisonnement**, jamais son implémentation. Le
code, la configuration, les options exactes, les déroulés pas à pas vivent dans le
code et dans la documentation vers laquelle l'ADR renvoie.

La **Justification** est un argumentaire, pas un document de conception : un
paragraphe qui explique *comment* une chose est construite, plutôt que *pourquoi* la
décision est la bonne, appartient ailleurs.

Le test : si l'implémentation change alors que la décision tient, l'ADR ne doit pas
avoir besoin d'être modifié.

## Ce journal et `specifications.md`

`specifications.md` est un **document de travail pré-développement**. Il énonce des
règles et des décisions, mais sans contexte isolé, sans alternatives envisagées,
sans conséquences, sans statut ni mécanisme de supersession — la forme qui permet de
les rouvrir proprement. Une section intitulée « décisions figées » signale seulement
qu'elles ne sont pas remises en cause à la date de rédaction.

Chacune pourra être portée dans un ADR le moment venu, quand une forme datée,
argumentée et supersédable devient utile — parce qu'elle est réexaminée, ou pour
pouvoir la superséder proprement plus tard. **Rien n'impose de le faire
préventivement**, et écrire un tel ADR ne modifie pas `specifications.md` : le
document de travail continue d'exister tel quel, l'ADR devient la forme faisant foi
pour cette décision précise.

L'articulation formelle entre les deux, notamment un versionnement de la
spécification, n'est pas tranchée à ce jour.

## Un ADR cite les précédents, il ne les redit pas

Le journal est **cumulatif** : ce qu'un ADR a posé est acquis pour tous les suivants.
Un ADR qui s'appuie sur un fait déjà enregistré le **cite là où il s'en sert** — il ne
le repose pas dans son Contexte.

La raison est celle qui interdit déjà de résumer `specifications.md` : un résumé diverge
de sa source dès la première évolution, et plus personne ne sait laquelle fait autorité.
La faute est la même à l'intérieur du journal, et elle s'aggrave avec le nombre d'ADR —
au douzième, le Contexte serait un digest des onze précédents, chacun étant une
divergence en attente.

Le test : **si le fait cité change, un seul fichier doit être à modifier.**

Corollaire pour la lecture croisée du Contexte et de la Justification : un argument qui
s'appuie sur un ADR antérieur n'est pas un argument sans fait. Son fait existe, il est
daté, il est ailleurs — et la citation suffit à le rendre atteignable.

**L'exception est la supersession**, et elle est explicite : un ADR qui en remplace un
autre résume ce que l'ancien décidait et pourquoi ça ne tient plus, parce que le lecteur
ne doit pas avoir à ouvrir l'ancien fichier pour comprendre celui qui le remplace.

## Qui décide quoi

Un agent **rédige, propose et recommande**. Il ne bascule aucun statut et ne crée
aucune issue de lui-même. Accepter, superséder, valider un amendement, ouvrir une
issue : ces actes appartiennent au mainteneur.

La raison tient au sens du statut : *Accepté* atteste d'une relecture humaine. Un
agent qui accepterait sa propre rédaction en ferait un champ rempli
automatiquement, et le journal perdrait ce qu'il est censé garantir.

La frontière est la même partout — **proposer oui, acter non**. Rédiger un ADR qui
en remplace un autre, ligne `Remplace` comprise, est du travail d'agent normal :
il naît en *Proposé*. Basculer l'**ancien** à *Remplacé* est l'acte qui acte.

Cette règle vaut **à tout moment, y compris sur une branche**. C'est même là que
toute acceptation s'écrit : un statut auto-accepté sur une branche serait figé par
le merge, sans que la relecture ait eu lieu.

## Ce que le merge fige

Tant qu'un ADR vit sur une branche, il est en rédaction : il se retouche librement,
statut et dates compris, sans amendement ni validation particulière. **C'est
l'arrivée sur `main` qui fige.**

À partir de là :

* **la section Décision ne bouge plus.** Si la décision change, c'est un nouvel ADR
  qui supersède — sans cette frontière, l'amendement deviendrait une porte dérobée
  pour réviser sans trace de supersession ;
* **le reste reste amendable** — une référence oubliée, une action de suivi
  manquante, une précision de contexte — mais jamais en silence ;
* **chaque amendement passe par une pull request** et ajoute sa ligne dans la table
  `Amendements` de l'ADR. La mécanique exacte est dans [`template.md`](template.md),
  en commentaires — voir « Conventions de fichier » pour les afficher.

La trace n'est pas négociable : un ADR atteste de *ce qu'on savait et pensait à cet
instant*. Un Contexte ou une Justification qu'on pourrait bonifier après coup, sans
que rien ne le signale, empêcherait de distinguer ce qui a fondé la décision de ce
qui a été ajouté ensuite pour la faire mieux paraître. La table `Amendements` dit en
outre *où* lire ce qui a changé, puisque la pull request en porte le titre, la
description et les commits.

## Superséder un ADR

Une décision révisée ne modifie jamais l'ADR qui la portait : elle s'écrit comme un
**nouvel** ADR qui remplace l'ancien. La mécanique — quelles lignes ajouter, dans
quel fichier — est dans [`template.md`](template.md), en commentaires : voir
« Conventions de fichier » pour les afficher.

Au niveau du journal, l'index ci-dessous passe le statut de l'ancien à *Remplacé* et
ajoute le nouvel ADR.

## Les actions de suivi et leurs issues

Une action de suivi écrite dans un ADR n'est suivie par rien : personne ne parcourt
le journal en cherchant du travail en attente. Une issue, si — elle apparaît dans le
backlog, elle se ferme, elle se retrouve. La traçabilité devient bidirectionnelle :
l'issue dit *pourquoi*, l'ADR dit *où c'est suivi*.

**Toute action de suivi a son issue.** Deux actions qui sont le même travail en
partagent une ; deux travaux sans rapport en prennent deux. L'ADR énonce le problème
et renvoie à l'issue ; le quoi et le comment vivent dans l'issue.

Une action qu'on ne veut pas mener maintenant a une issue elle aussi, **fermée
d'emblée en `not planned`** :

> Dans ce dépôt, `not planned` signifie *non planifiée*, pas *abandonnée*. L'issue
> énonce son critère de réouverture, et on la rouvre quand la condition survient.
> Elle n'encombre pas le backlog ouvert tout en restant trouvable.

Les créer **avant le merge** évite un amendement : leurs numéros s'écrivent alors
directement dans la rédaction, tant que l'ADR est encore librement modifiable.

## Conventions de fichier

Une décision par fichier, nommé `NNNN-résumé-en-kebab-case.md`, copié depuis
[`template.md`](template.md) — qui définit aussi la forme de l'en-tête, les statuts,
les dates et les tables, et qui se termine par le contrôle à passer sur le texte fini
avant de proposer un ADR. Ce que ce contrôle a changé se dit dans la description de la
pull request : un audit qu'on ne peut pas relire ne se distingue pas d'un audit annoncé.

> **Le gabarit paraît vide, et c'est voulu.** Toutes ses consignes sont en
> commentaires HTML : elles ne s'affichent pas dans la vue rendue de GitHub, ce qui
> évite qu'un ADR publié se retrouve à afficher son propre mode d'emploi. Pour les
> lire, basculez sur l'onglet **Code** en haut à droite du fichier (`Preview` ·
> **`Code`** · `Blame`), ou ouvrez [`template.md?plain=1`](template.md?plain=1).
> C'est là que vit la mécanique exacte à laquelle ce README renvoie.

## Index

| ADR | Titre | Statut |
|---|---|---|
| [ADR-0001](0001-adopter-la-pratique-adr.md) | Adopter la pratique ADR | Accepté |
| [ADR-0002](0002-tenir-un-historique-plat.md) | Tenir un historique plat sur `main` | Accepté |
