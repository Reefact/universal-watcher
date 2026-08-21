# ADR-{numéro} | {Titre court}

<!-- Un ADR enregistre une DÉCISION et son RAISONNEMENT, jamais son
     implémentation. Code, configuration, YAML, flags exacts, extraits de
     commande et déroulés « comment c'est construit » n'ont leur place dans
     AUCUNE section — c'est de la spécification : elle vit dans le code et
     dans specifications.md, vers lesquels cet ADR renvoie plutôt que de
     les redire.
     Test : si l'implémentation change mais que la décision tient, cet ADR
     ne devrait pas avoir besoin d'être modifié.

     TOUTE référence à une issue ou à une PR s'écrit en LIEN COMPLET,
     jamais en `#NN` nu : GitHub n'autolie pas les numéros dans un fichier
     `.md` du dépôt, et un numéro qui ne clique pas oblige le lecteur à
     aller le chercher. Forme : [#74](https://github.com/Reefact/universal-watcher/issues/74).

     La liste de sections ci-dessous est FERMÉE : exactement ces sections,
     dans cet ordre, aucune renommée, aucune ajoutée. Si une section n'a
     rien de réel à dire, dites-le avec la formule exacte prévue pour elle
     — jamais un contenu inventé, jamais une suppression silencieuse, et
     surtout jamais une anticipation sur une décision qui n'a pas encore
     été prise, y compris celle d'un futur ADR :

       Conséquences > Positives   Aucune identifiée.
       Conséquences > Négatives   Aucune identifiée.
       Conséquences > Risques     Aucun identifié.
       Actions de suivi           Aucune identifiée.
       Références                 Aucune.

     Alternatives envisagées n'a pas de formule : une décision sans
     alternative réelle mérite qu'on se demande si elle vaut un ADR.

     Voir docs/handwritten/for-maintainers/adr/README.md. -->

| Champ | Valeur |
| - | - |
| Statut | Proposé · Accepté · Remplacé |
| Remplace — si cet ADR en remplace un autre, sinon ligne absente | [ADR-NNNN](NNNN-....md) |
| Remplacé par — si cet ADR a été remplacé, sinon ligne absente | [ADR-NNNN](NNNN-....md) |
| Proposé | le AAAA-MM-JJ à HH:MM (UTC+0X:00) |
| Accepté — `-` tant que le statut n'est pas Accepté | le AAAA-MM-JJ à HH:MM (UTC+0X:00) ou `-` |

<!-- Section Amendements — absente tant qu'il n'y a aucun amendement, et
     donc laissée en commentaire ici. La ligne d'origine est néanmoins
     remplie dès maintenant, avec le numéro de la PR qui amène cet ADR sur
     `main` : c'est le seul moment où il est connu sans avoir à fouiller
     l'historique. Au premier amendement, décommenter le bloc et ajouter
     la ligne a1.

### Amendements

| Amendement | Pull Request |
| - | - |
| — | [#NN](https://github.com/Reefact/universal-watcher/pull/NN) |

-->

<!-- Les dates sont à l'heure de Paris, en 24 h, annotées de leur décalage
     effectif : UTC+02:00 l'été, UTC+01:00 l'hiver. Le décalage plutôt que
     CEST/CET, parce qu'il reste vrai sans connaître la saison et qu'une
     erreur de recopie saute aux yeux — un UTC+02:00 en janvier se voit,
     un CEST en janvier non.

     L'heure n'est pas décorative : elle rend visible l'intervalle entre
     proposition et acceptation, donc le fait qu'une relecture ait eu lieu.
     Deux dates sans heure sur le même jour se lisent comme un intervalle
     nul, c'est-à-dire comme une ratification à la volée.

     Remplir `-` par une date n'est pas réécrire une date : `-` n'en était
     pas une. Une fois une vraie date écrite, en revanche, elle n'est plus
     jamais modifiée — ni Proposé, ni Accepté une fois rempli.

     Remplace / Remplacé par vont toujours par paire de fichiers, jamais
     dans le même : Remplace apparaît dans le NOUVEL ADR (celui qui
     remplace), Remplacé par dans l'ANCIEN (celui qu'on remplace, dont le
     Statut passe alors à Remplacé). Les deux fichiers restent en place
     sous leur nom d'origine — aucun n'est renommé ni supprimé.

     Mettre à jour l'index de README.md fait partie de la même opération :
     statut de l'ancien à Remplacé, nouvel ADR ajouté.

     LA TABLE AMENDEMENTS n'existe qu'à partir du premier amendement — un
     ADR intact n'a pas à porter une table vide, qui ferait passer
     l'amendement pour la norme. Sa première ligne, notée `—`, n'est pas
     un amendement : c'est la PR d'origine, celle qui a amené l'ADR sur
     `main`. Les suivantes sont numérotées a1, a2, et ainsi de suite, et
     ne sont jamais réécrites.
     UNE SUPERSESSION N'Y FIGURE PAS. Elle modifie pourtant le fichier —
     le Statut passe à Remplacé — mais ce n'est pas un amendement, et la
     ligne `Remplacé par` porte déjà toute l'information : elle pointe
     l'ADR remplaçant, dont la propre table pointe sa PR. La trace existe
     à un saut près ; lui donner une ligne ici salirait le mot.
     La cellule ne porte QUE le numéro de PR : ce qui a changé se lit dans
     la PR elle-même, titre, description et commits. Le recopier ici
     créerait une divergence en attente.
     Le numéro est connu avant le merge, d'où l'ordre : ouvrir la PR,
     ajouter la ligne, pousser, merger.

     TANT QU'UN ADR VIT SUR UNE BRANCHE, il est encore en rédaction : il
     se retouche librement, statut et dates compris, sans amendement
     ni validation particulière. C'est l'arrivée sur `main` qui fige —
     rien de ce qui suit SUR L'AMENDEMENT ne s'applique avant.
     Cette souplesse ne touche PAS la règle d'autorité, qui vaut à tout
     moment, branche comprise : c'est justement sur une branche qu'une
     acceptation s'écrit, et un agent qui s'y accepterait lui-même ferait
     ensuite figer ce statut par le merge.

     UNE FOIS SUR `main`, l'ADR reste amendable — référence oubliée,
     action de suivi manquante, précision de contexte — mais jamais en
     silence : l'amendement passe par une PR, qui ajoute sa ligne dans la
     table Amendements.
     La section DÉCISION, elle, ne s'amende alors plus jamais : si la
     décision change, c'est un nouvel ADR qui supersède, sinon
     l'amendement devient une porte dérobée pour réviser sans trace de
     supersession.
     Voir README.md, « Ce que le merge fige ».

     Un agent rédige le NOUVEL ADR — en Proposé, ligne Remplace comprise —
     mais ne bascule jamais l'ANCIEN à Remplacé de lui-même, pas plus qu'il
     n'accepte : proposer oui, acter non. Voir README.md, « Qui décide
     quoi ». -->

## Contexte

<!-- Des faits, rien d'autre — aucune justification de la solution choisie.
     Tout ce que la Justification argumentera doit être posé ici d'abord :
     contraintes techniques ou d'architecture, limites déjà connues,
     dépendances externes, ce que specifications.md dit ou laisse ouvert sur
     le sujet. Quelqu'un qui découvre le projet doit comprendre pourquoi une
     décision était nécessaire.
     SAUF ce qu'un ADR antérieur a déjà posé : ça se cite dans la
     Justification, à l'endroit qui s'en sert, et ça ne se repose pas ici.
     Le journal est cumulatif ; redire diverge. Voir README.md, « Un ADR
     cite les précédents, il ne les redit pas ».
     Si cet ADR en remplace un autre, résumez ici ce que l'ancien décidait
     et pourquoi ça ne tient plus — le lecteur ne doit pas avoir à ouvrir
     l'ancien fichier pour comprendre le nouveau. -->

## Décision

<!-- Une seule phrase. Aucune justification, aucune alternative, aucun
     détail d'implémentation à moins qu'il ne fasse partie de la décision
     elle-même. -->

## Justification

<!-- Argumentaire seul — pourquoi cette décision est le meilleur choix compte
     tenu du Contexte.

     LISEZ LES DEUX SECTIONS L'UNE CONTRE L'AUTRE, DANS LES DEUX SENS,
     jusqu'à ce qu'elles se répondent :
     - un ARGUMENT sans fait correspondant est toujours un défaut — soit
       il manque un fait au Contexte, soit l'argument ne tient pas. Un
       argument adossé à un ADR antérieur fait exception : son fait est
       posé, daté, ailleurs — il se cite ici et n'a pas à remonter au
       Contexte ;
     - un FAIT sans argument n'en est pas forcément un : il peut poser le
       décor, ou être traité directement par la Décision. Mais c'est un
       signal à vérifier, pas à laisser filer.
     Les deux sections s'améliorent ainsi mutuellement, et il faut
     plusieurs passes avant qu'elles se répondent vraiment.

     MÉFIEZ-VOUS DU FAIT QUI CONCLUT. « X n'a pas de mécanisme de
     supersession » est un fait ; « donc le modifier effacerait la trace »
     est un argument. Le second appartient ici ou aux Alternatives, pas au
     Contexte — un Contexte qui plaide en décrivant est le défaut le plus
     difficile à repérer.
     N'y mettez PAS de détail d'implémentation — pas de code, de
     configuration, de flags exacts, de déroulé pas à pas. C'est de la
     spécification : renvoyez vers le code ou vers specifications.md.
     Nommer le rôle d'un mécanisme et pourquoi il existe est un argument
     (à garder ici) ; documenter comment il est câblé est de la
     spécification (à renvoyer ailleurs). -->

## Alternatives envisagées

### {Alternative 1}

<!-- Pourquoi elle a été envisagée. -->

<!-- Pourquoi elle a finalement été écartée. -->

### {Alternative 2}

...

## Conséquences

### Positives

* ...

### Négatives

* ...

### Risques

* ...

## Actions de suivi

<!-- LA RÈGLE « PAS DE SPÉCIFICATION DANS UN ADR » VAUT ICI AUSSI. Une
     action de suivi énonce le PROBLÈME à traiter et renvoie à son issue ;
     le QUOI et le COMMENT vivent dans l'issue. Même test que pour le
     reste : si le détail change alors que la décision tient, l'ADR ne
     doit pas avoir à être amendé. Écrire le détail aux deux endroits
     garantit une divergence.

     TOUTE action a son issue, dont le numéro s'écrit sur sa ligne — sans
     ça elle n'est suivie par rien, personne ne parcourt le journal en
     cherchant du travail en attente. Deux actions qui sont le même
     travail partagent une issue ; deux travaux sans rapport en prennent
     deux.
     Une action qu'on ne veut pas mener maintenant a une issue elle aussi,
     FERMÉE D'EMBLÉE EN `not planned` — non planifiée, pas abandonnée —
     portant son critère de réouverture. Rien à trancher au cas par cas,
     donc rien qui se décide différemment d'une fois sur l'autre.
     Un agent RECOMMANDE les issues à ouvrir ; il ne les crée pas de
     lui-même.
     Créer les issues AVANT le merge évite un amendement : leurs numéros
     s'écrivent alors directement dans la rédaction. -->

* ...

## Références

<!-- Optionnel : autres ADR, specifications.md, issues, pull requests. -->

* ...
