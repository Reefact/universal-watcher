# Reefact.UniversalWatcher — Spécification V1

> Version consolidée. Elle remplace les documents antérieurs (`RepositoryWatcher V1`
> et `UniversalWatcher & RepositoryWatcher V1`), qui sont considérés comme des
> brouillons de conception.

---

## 0. Comment lire ce document

### 0.1. À qui il s'adresse

Ce document s'adresse à la personne qui va implémenter le produit. Il décrit **ce
qui doit exister et pourquoi**, pas comment l'écrire ligne à ligne. Les extraits
de code sont conceptuels : ils fixent des intentions, pas des signatures
définitives.

### 0.2. Structure

| Partie | Sections | Contenu |
|---|---|---|
| I | 1 – 4 | Le produit : ce qu'il fait, ce qu'il ne fait pas |
| II | 5 – 8 | L'architecture : projets, dépendances, responsabilités |
| III | 9 – 24 | Le modèle de RepositoryWatcher : état, snapshot, réconciliation |
| IV | 25 – 26 | L'écran : le tableau d'état |
| V | 27 – 34 | Les indicator devices : signal, file d'animations, sélection |
| VI | 35 – 38 | Le cycle de vie : démarrage, dégradation, arrêt |
| VII | 39 – 42 | Connexions et sécurité |
| VIII | 43 – 45 | Configuration |
| IX | 46 – 49 | Le CLI |
| X | 50 – 56 | Notes techniques, règles, décisions, points ouverts |

Trois sections méritent d'être lues en premier si le temps manque : **9** (le
modèle vivant), **24** (le récapitulatif du modèle) et **51** (les règles
essentielles).

### 0.3. Glossaire

Ces termes reviennent partout. Ils sont définis ici une fois pour toutes.

| Terme | Définition |
|---|---|
| **Host** | `UniversalWatcher` : le socle qui orchestre les modules. |
| **Module métier** | Une capacité de surveillance compréhensible par l'utilisateur. En V1 : `RepositoryWatcher`. |
| **Module technique** | Une capacité utilisée par le host ou les modules métier. En V1 : `Connections.GitHub`. |
| **Output** | Un module qui matérialise l'état hors du terminal. En V1 : `Outputs.Luxafor`. |
| **Indicator device** | Un point de sortie concret fourni par un output : un Orb, un buzzer, une fenêtre. Peut être physique ou virtuel. |
| **Snapshot** | Une lecture immuable de la vérité observée chez le fournisseur à un instant donné. |
| **State** | Le modèle vivant maintenu localement : ce qui est vrai maintenant. |
| **Event** | Ce qui vient de se produire, déduit de la comparaison entre deux observations. |
| **Reconcile** | L'opération qui confronte un state et un snapshot pour produire un nouveau state et des events. |
| **RunStatus** | Est-ce que la CI tourne en ce moment ? (`None` / `Running` / `Completed`) |
| **LastConclusion** | Quel a été le dernier verdict conclusif de la CI ? (`Unknown` / `Healthy` / `Broken`) |
| **WatchHealth** | La santé métier générique : `Unknown`, `Healthy`, `Warning`, `Critical`. |
| **Freshness** | Le degré de confiance dans la fraîcheur de ce qui est affiché. |
| **IndicatorSignal** | L'ordre résolu envoyé aux indicator devices : quoi rendre maintenant. |
| **Branche de référence** | La branche dont la casse est critique. Par défaut la branche par défaut du dépôt. |
| **Wizard** | Le dialogue interactif de sélection d'un ou plusieurs indicator devices au premier lancement. |

> **Attention à un mot piégeux : « provider ».** Le document l'emploie dans deux
> sens sans rapport. Pour éviter toute confusion, la terminologie est fixée
> ainsi :
>
> - **fournisseur** (ou *connection provider*) : GitHub, GitLab — la source des
>   données de dépôt ;
> - **`IIndicatorDeviceProvider`** : le composant qui découvre les indicator
>   devices d'un output.
>
> Le mot « provider » seul, en français, désigne toujours le premier.

---

# PARTIE I — LE PRODUIT

## 1. Vision

`UniversalWatcher` est un **host de surveillance local extensible par modules**,
distribué comme un unique .NET Tool. Il fournit une commande :

```bash
uwatch
```

Le premier module métier est `RepositoryWatcher`, qui surveille un dépôt Git.

```bash
uwatch repo just-dummies
```

### 1.1. Le besoin, en une phrase

> Savoir, à tout instant et sans effort, où en est mon dépôt — et être averti
> quand ça se dégrade.

Deux fonctions, pas trois :

```text
MONTRER    la vérité courante, en détail        → le tableau dans le terminal
ALERTER    quand quelque chose se dégrade       → les indicator devices
```

Le terminal et l'Orb répondent à **la même question** — quelle est la vérité
maintenant — à deux résolutions différentes. L'Orb dit « regarde », le terminal
dit « voilà quoi ».

### 1.2. Ce que le produit n'est pas

`UniversalWatcher` **n'est pas un outil de monitoring** au sens usuel du terme.
Cette distinction n'est pas cosmétique, elle détermine une grande partie du
design :

| Un outil de monitoring | UniversalWatcher |
|---|---|
| Collecte et conserve des séries temporelles | Ne conserve rien au-delà de la session |
| Raconte l'histoire d'un incident | Décrit la situation actuelle |
| Produit des rapports a posteriori | Produit un écran à regarder maintenant |
| Sert à diagnostiquer | Sert à savoir s'il faut aller diagnostiquer ailleurs |

Conséquence directe : **il n'y a pas de journal d'événements dans le produit.**
Un flux de logs répond à la question « qu'est-ce qui s'est passé pendant que je
ne regardais pas », qui n'est pas la question posée. Si un problème est apparu
puis a disparu, il n'y a rien à savoir. S'il est encore là, il est dans le
tableau.

### 1.3. Le principe directeur

```text
UniversalWatcher indique qu'un problème existe.
Le développeur ouvre GitHub s'il veut le détail.
```

## 2. Hors périmètre V1

Ces éléments sont explicitement exclus. La liste sert de garde-fou : elle
justifie la plupart des simplifications de ce document.

**Fonctionnalités produit**

- détail des jobs ou steps ayant échoué ;
- logs de CI ;
- diagnostic de la cause d'un échec ;
- issues, commentaires, reviews, stars, forks ;
- historique persistant, séries temporelles, statistiques d'instabilité ;
- dashboard web ;
- surveillance simultanée de plusieurs dépôts dans une seule instance ;
- webhooks, endpoint public, backend ;
- analyse du contenu du code.

**Techniques**

- système de plugins dynamiques : téléchargement, manifestes, installation à
  chaud, `AssemblyLoadContext` dédié, résolution de versions ;
- providers autres que GitHub effectivement implémentés (GitLab, Azure DevOps) ;
- outputs autres que Luxafor effectivement implémentés.

> **Note.** Le découpage architectural en modules est conservé (Partie II), mais
> il ne s'accompagne d'aucun mécanisme d'extension à l'exécution. Les modules
> sont connus à la compilation.

## 3. Nommage

Tous les projets de la solution sont préfixés `Reefact.`.

```text
Produit / solution     Reefact.UniversalWatcher
Commande installée     uwatch
Paquet NuGet publié    Reefact.UniversalWatcher
```

**Projets**

```text
Reefact.UniversalWatcher.Abstractions
Reefact.UniversalWatcher.Core
Reefact.UniversalWatcher.Cli

Reefact.UniversalWatcher.RepositoryWatcher            module métier
Reefact.UniversalWatcher.RepositoryWatcher.GitHub     adaptateur de lecture
Reefact.UniversalWatcher.Connections.GitHub           module technique
Reefact.UniversalWatcher.Outputs.Luxafor              output
```

Dans la suite du document, le préfixe `Reefact.` est omis pour la lisibilité.

## 4. Distribution

Un seul paquet, un seul tool, toutes les capacités V1 incluses.

```bash
dotnet tool install --global Reefact.UniversalWatcher
```

Le projet CLI porte :

```xml
<PackAsTool>true</PackAsTool>
<ToolCommandName>uwatch</ToolCommandName>
<PackageId>Reefact.UniversalWatcher</PackageId>
```

> Attention : le `PackageId` est `Reefact.UniversalWatcher`, **pas**
> `Reefact.UniversalWatcher.Cli`. C'est le nom que l'utilisateur tape pour
> installer.

Il n'existe pas de commande `uwatch install`, `uninstall` ou `update` pour des
modules internes.

---

# PARTIE II — ARCHITECTURE

## 5. Vue d'ensemble

L'architecture est modulaire, de type Clean / Hexagonale. Le principe est
qu'aucune règle métier ne doit connaître GitHub, Spectre ou Luxafor.

```text
┌──────────────────────────────────────────────────────────┐
│                          CLI                             │
│         composition root · Spectre · commandes           │
└──────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼───────────────────┐
        │                   │                   │
        ▼                   ▼                   ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────┐
│ RepositoryWa- │   │  Connections  │   │    Outputs    │
│    tcher      │   │    .GitHub    │   │   .Luxafor    │
│ module métier │   │   technique   │   │    output     │
└───────────────┘   └───────────────┘   └───────────────┘
        │                   │                   │
        └───────────────────┼───────────────────┘
                            ▼
                    ┌───────────────┐
                    │     Core      │
                    │  orchestration│
                    └───────────────┘
                            │
                            ▼
                    ┌───────────────┐
                    │ Abstractions  │
                    │   contrats    │
                    └───────────────┘
```

Et, transversalement, l'adaptateur de lecture :

```text
RepositoryWatcher.GitHub
    ├── dépend de RepositoryWatcher      (implémente IRepositorySource)
    └── dépend de Connections.GitHub     (obtient un client authentifié)
```

## 6. Pourquoi trois projets pour GitHub

C'est un point qui a besoin d'être explicité, car il n'est pas évident.

`Connections.GitHub` est un module **technique** : il sait authentifier auprès de
GitHub et produire un client utilisable. Il ne connaît ni pull request, ni check,
ni release — sinon, le jour où un futur `CertificateWatcher` l'utilisera pour
autre chose, il embarquerait du code de dépôt qui ne le concerne pas.

La requête qui lit les PR, les checks et les releases n'a de sens que pour
`RepositoryWatcher`. Elle vit donc dans un troisième projet :

```text
Connections.GitHub            « je sais me connecter à GitHub »
RepositoryWatcher             « je sais raisonner sur un dépôt »
RepositoryWatcher.GitHub      « je sais lire un dépôt sur GitHub »
```

C'est dans ce troisième projet que vit le mapping des conclusions de CI
(section 22).

## 7. Responsabilités

### 7.1. Abstractions

Contrats partagés, volontairement petits. Aucune dépendance vers Spectre,
GitHub, Luxafor ou les implémentations métier.

Contient notamment :

```text
WatchHealth              la santé métier générique
Freshness                l'état de confiance dans la fraîcheur
IndicatorSignal          l'ordre envoyé aux devices
IIndicatorDevice         le contrat d'un point de sortie
IIndicatorDeviceProvider la découverte des devices
IWatcherSession          le contrat qu'un module métier expose au host
ICredentialStore         le stockage sécurisé des secrets
BlinkDuration            value object de durée
Connection               le concept générique de connexion
ProviderException        taxonomie transitoire / définitive
```

### 7.2. Core

Responsabilités transverses, indépendantes du métier surveillé :

- **la résolution du signal** : file d'animations, priorités, horloge
  (Partie V) ;
- le registre des connection providers ;
- le registre des outputs et la découverte des devices ;
- le chargement et la validation de la configuration ;
- l'état de fraîcheur (`Live` / `Stale` / `Failed`) ;
- l'arrêt propre.

Core ne connaît ni GitHub ni Luxafor — **ni les pull requests**. Voir 7.6.

### 7.3. CLI

Adaptateur entrant et composition root. Contient Spectre.Console.Cli, les
commandes, le parsing, le rendu du tableau, et l'assemblage des modules.

**Le CLI ne contient aucune logique métier.** Il ne maintient pas le modèle : il
l'affiche.

### 7.4. RepositoryWatcher

Le module métier. Il maintient le modèle vivant du dépôt et produit les
événements. Il connaît les dépôts, les branches, les PR, les checks, les
releases. Il ne connaît ni Spectre, ni Luxafor, ni le stockage des secrets.

### 7.5. Les modules d'infrastructure

`Connections.GitHub` enregistre le fournisseur `github`.
`RepositoryWatcher.GitHub` implémente `IRepositorySource`.
`Outputs.Luxafor` fournit des `IIndicatorDevice`.
`Core.Credentials` implémente `ICredentialStore` pour chaque système
d'exploitation (voir section 41).

### 7.6. La frontière entre un module métier et le host

C'est le point d'articulation le plus important de l'architecture, et il doit
être explicite pour ne pas être implémenté à moitié.

**Qui possède la boucle de lecture ?** Le module métier. C'est
`RepositoryWatcher` qui lit, reconcilie et maintient son modèle. Core ne connaît
ni `RepositorySnapshot` ni `RepositoryState` — sinon il devrait dépendre du
module métier, ce qui inverserait le sens des dépendances.

**Que fait Core, alors ?** Il orchestre ce qui est transverse : le cycle de vie
de la session, la fraîcheur, la résolution du signal, l'arrêt.

La frontière est matérialisée par un contrat, exprimé en termes génériques :

```csharp
public interface IWatcherSession : IAsyncDisposable
{
    // Ce que le module métier expose au host
    WatchHealth CurrentHealth { get; }

    // Les signaux ponctuels produits par le métier
    IAsyncEnumerable<WatchSignal> Signals { get; }

    // Le cycle de vie, piloté par le host
    Task StartAsync(CancellationToken ct);
}
```

```text
RepositoryWatcher                          Core
─────────────────                          ────
lit, reconcilie, maintient le modèle
   │
   ├── CurrentHealth ────────────────────► base du signal
   └── Signals ──────────────────────────► file d'animations
                                                │
                                                ▼
                                          IndicatorSignal
                                                │
                                                ▼
                                             devices
```

Le CLI, lui, est composition root : il connaît `RepositoryWatcher` directement et
lit `RepositoryState` pour rendre le tableau. C'est son rôle, et introduire un
modèle de vue intermédiaire n'ajouterait qu'une couche de recopie.

> **Limitation V1 assumée.** Core gère **une seule** session à la fois. Il n'y a
> donc pas de règle d'agrégation entre plusieurs modules métier simultanés :
> une santé, une fraîcheur, un signal. Le contrat ci-dessus ne l'interdit pas
> pour l'avenir, mais rien n'est prévu pour ça et il ne faut pas supposer le
> contraire.

### 7.7. Taxonomie des erreurs

Core doit décider entre `Stale` et `Failed` (section 32) sans connaître HTTP.
C'est donc l'adaptateur, qui sait interpréter un code de réponse, qui porte
l'information dans le type d'exception.

```csharp
// Défini dans Abstractions
public abstract class ProviderException : Exception
{
    public abstract bool IsTransient { get; }
    public TimeSpan? RetryAfter { get; }     // si le fournisseur l'indique
}

TransientProviderException     réseau, 5xx, timeout, quota dépassé
PermanentProviderException     credential invalide, droits manquants, 404
```

Sans cette taxonomie, Core devrait inspecter des messages ou des types concrets
d'exception, ce qui casserait son indépendance.

## 8. Enregistrement à la compilation

Aucun switch central. Chaque module s'enregistre lui-même.

```csharp
services
    .AddRepositoryWatcher()            // enregistre la commande « repo »
    .AddGitHubConnectionProvider()     // enregistre le provider « github »
    .AddGitHubRepositorySource()       // enregistre l'adaptateur de lecture
    .AddLuxaforOutput();               // enregistre l'output
```

Les frontières doivent rester assez propres pour qu'un chargement dynamique
puisse être ajouté plus tard sans réécrire le domaine — mais cette capacité
n'est ni implémentée ni nécessaire en V1.

---

# PARTIE III — LE MODÈLE

## 9. Le principe : un modèle vivant

C'est le cœur du produit, et le point le plus important du document.

`RepositoryWatcher` ne pilote **jamais** l'affichage ni les devices à partir du
dernier événement reçu. Il maintient un modèle local de la vérité du dépôt, et
tout ce qui est montré est une **projection de ce modèle**.

Pourquoi c'est essentiel : imaginons trois PR cassées et la branche de référence
réparée à l'instant. Le dernier événement est vert. Si l'Orb suivait le dernier
événement, il serait vert — alors que trois PR sont cassées. La couleur doit
venir de l'état global, pas du dernier fait observé.

```text
        provider
           │
           ▼
      Snapshot        « ce que j'observe maintenant »
           │
           ▼
      Reconcile       ← State courant
           │
           ├──────────► nouveau State     « ce qui est vrai »
           └──────────► Event[]           « ce qui vient de changer »
                            │
                            ▼
                     signal aux devices
```

Le State alimente le tableau et la couleur stable.
Les Events alimentent les alertes.

## 10. RepositoryState

```csharp
RepositoryState
{
    TrackedBranch,      // la branche de référence
    PullRequests,       // les PR ouvertes
    LatestRelease
}
```

> **Renommage important.** Les brouillons parlaient de `Main` / `MainState` /
> `MainBroken`. Or `main` est un nom de branche, pas un concept. Beaucoup de
> dépôts utilisent `master`, `develop` ou `trunk`. Encoder un nom de branche dans
> le domaine est le même travers que d'y encoder une couleur RGB. Le concept
> générique est **la branche de référence** : celle dont la casse est critique.

### 10.1. TrackedBranchState

```text
Name                    ex. « main », « develop »
HeadSha
RunStatus               état d'exécution courant des checks
LastConclusion          dernier verdict conclusif connu
LastStatusChangedAt     quand LastConclusion a changé pour la dernière fois
```

### 10.2. PullRequestState

```text
Number
Title                   titre court, tronqué à l'affichage
HeadSha
IsDraft
Mergeable               Clean | Conflicting | Unknown
CreatedAt               fourni par le provider
RunStatus
LastConclusion
LastStatusChangedAt     calculé localement
```

### 10.3. ReleaseState

```text
Id
Tag
PublishedAt
```

## 11. RepositorySnapshot

Chaque lecture du provider produit un snapshot **immuable**.

```csharp
RepositorySnapshot
{
    ObservedAt,             // horodatage de la lecture
    TrackedBranch,
    PullRequests,
    LatestRelease
}
```

### 11.1. Pourquoi ObservedAt est nécessaire

Deux raisons, toutes deux importantes.

**La garde de monotonie.** Rien ne garantit que les snapshots arrivent dans
l'ordre où ils ont été produits — une requête lente peut être doublée par la
suivante. Un snapshot dont l'`ObservedAt` est antérieur ou égal au dernier
appliqué doit être **rejeté sans traitement**. Sans cette garde, le modèle peut
régresser vers un état périmé.

**L'horodatage des transitions.** Quand `Reconcile` détecte un changement d'état,
il doit dater ce changement. La date utilisée est celle du snapshot, pas
l'horloge locale au moment du traitement. Cela garde `Reconcile` **pur** : la
même paire (state, snapshot) produit toujours exactement le même résultat, ce qui
la rend testable sans horloge ni périphérique.

> `ObservedAt` est l'horloge du poste au moment de la lecture, pas celle du
> serveur. On ne s'en sert que pour ordonner des lectures locales entre elles,
> jamais pour comparer avec des dates fournies par le provider.

### 11.2. Un snapshot est tout ou rien

Un snapshot partiel est interdit. Si la lecture échoue, même partiellement,
l'adaptateur **ne publie pas de snapshot** : il lève.

C'est une précaution concrète, pas théorique. GraphQL peut renvoyer une réponse
« réussie » contenant un bloc `errors` et des champs à `null`. Si une telle
réponse était convertie en snapshot, les PR manquantes seraient interprétées
comme fermées, et le modèle produirait des transitions absurdes.

## 12. Deux façons d'entrer dans le modèle

Le domaine expose **exactement deux** opérations. Cette séparation est
structurelle : elle rend impossible la génération de faux événements au
démarrage.

```csharp
// Première observation : on construit. Aucun événement.
RepositoryState.CreateInitial(RepositorySnapshot snapshot)
    → RepositoryState

// Observations suivantes : on compare. Des événements peuvent naître.
state.Reconcile(RepositorySnapshot snapshot)
    → (RepositoryState NewState, RepositoryEvent[] Events)
```

### 12.1. Pourquoi deux opérations et pas une

Au démarrage, `RepositoryWatcher` découvre un dépôt qui a une histoire. Des PR
sont déjà cassées, une release a déjà été publiée. Ces faits ne sont **pas** des
événements : ils ne viennent pas de se produire, l'outil n'était simplement pas
là quand ils sont arrivés.

Si le premier snapshot passait par `Reconcile` sur un état vide, il produirait
« PR #438 vient de casser », « release v1.2.0 vient d'être publiée » — et
déclencherait des alertes pour des faits anciens.

`CreateInitial` initialise sans raconter. Le premier `LastStatusChangedAt` est
donc **inconnu** pour toutes les entités : on sait qu'une PR est cassée, on ne
sait pas depuis quand. C'est la contrepartie assumée de l'absence d'historique.

### 12.2. Reconcile est pure

Pas d'entrées/sorties, pas d'horloge interne, pas d'aléa. Deux snapshots
identiques produisent zéro événement. Cette propriété permet de tester la
totalité de la logique métier en table-driven, sans réseau ni périphérique.

### 12.3. Reconcile est le seul chemin

Il n'existe pas de mode « resynchronisation » distinct. Après une coupure
réseau, on reconcilie avec un snapshot frais, exactement comme d'habitude. Une
coupure de dix minutes et un intervalle de dix secondes sont le même phénomène à
deux échelles.

## 13. Le principe de non-invention

> **Le domaine ne raconte que les transitions entre deux points d'observation,
> jamais le chemin parcouru entre eux.**

C'est la règle qui découle de tout ce qui précède, et elle mérite d'être
comprise sur des exemples.

**Exemple 1 — un aller-retour invisible.** Entre deux lectures, une PR passe de
verte à rouge puis revient au vert, sans changement de commit. Les deux snapshots
sont identiques. `Reconcile` ne produit **aucun** événement. C'est correct : la
PR est verte, il n'y a rien à faire, et une alerte pour un problème déjà résolu
serait du bruit.

**Exemple 2 — plusieurs commits entre deux lectures.** Une PR verte reçoit un
commit, casse, reçoit un correctif, redevient verte. Le SHA a changé, donc les
snapshots diffèrent. Mais la transition se calcule **uniquement** sur la
comparaison des `LastConclusion` : verte avant, verte après, donc aucune
transition. Le changement de SHA ne produit rien à lui seul.

**Exemple 3 — une coupure réseau.** Pendant dix minutes hors ligne, la branche de
référence est réparée et deux PR cassent. À la reconnexion, le snapshot montre le
résultat. Le modèle est mis à jour, l'écran reflète la nouvelle réalité. Aucun
événement n'est fabriqué pour les transitions non observées.

> Limitation acceptée : un état qui apparaît et disparaît entièrement entre deux
> lectures peut ne jamais être observé. Ce n'est pas un défaut à corriger, c'est
> le comportement voulu.

## 14. Les deux mémoires des checks

Le modèle stocke **deux** informations distinctes sur l'état de la CI, et non
une seule. C'est ce qui permet de tenir la règle suivante :

> Une nouvelle exécution en cours ne répare jamais implicitement un état
> précédemment cassé.

```text
RunStatus         None | Running | Completed
                  « est-ce que ça tourne en ce moment ? »

LastConclusion    Unknown | Healthy | Broken
                  « quel a été le dernier verdict conclusif ? »
```

Illustration :

```text
PR #438 cassée
    → RunStatus = Completed, LastConclusion = Broken

un correctif est poussé, la CI redémarre
    → RunStatus = Running,   LastConclusion = Broken   ← inchangé !

la CI réussit
    → RunStatus = Completed, LastConclusion = Healthy  ← transition
```

Entre les deux, la PR **reste comptée comme cassée**. L'affichage montre bien que
ça tourne, mais la santé ne bouge pas tant qu'aucun verdict n'est tombé.

Même règle pour la branche de référence.

## 15. Agrégation des checks

Une PR a N check runs. La colonne d'affichage en montre **un seul** état agrégé.

L'agrégation porte uniquement sur les runs du **HEAD SHA courant**. Un run
encore en cours pour un SHA obsolète est ignoré : il teste du code déjà
remplacé.

Règle, appliquée dans cet ordre :

```text
1. au moins un « action required »        → action required
2. au moins un run en cours               → running
3. au moins un échec                      → failed
4. au moins un succès, aucun échec        → passed
5. tous skippés                           → skipped
6. tous annulés                           → cancelled
7. autre combinaison de non-concluants    → inconclusive
8. aucun run                              → none
```

**Pourquoi `action required` en premier.** Il signale qu'une intervention humaine
est attendue : tant que personne n'agit, rien n'avancera. C'est actionnable, au
contraire de `running` qui invite seulement à patienter.

**Pourquoi `running` avant `failed`.** La question la plus utile est « est-ce que
ça bouge ? ». Voir `running` dans la colonne CHECKS et `broken` dans la colonne
STATE dit exactement : un correctif est en cours de vérification sur une PR
cassée, attends avant de regarder.

Cette agrégation est **sans mémoire**. Elle décrit l'instant présent. Toute la
mémoire est dans `LastConclusion`.

## 16. Le lien entre agrégation et santé

Seuls deux résultats d'agrégation mettent à jour `LastConclusion` :

```text
passed   → LastConclusion = Healthy
failed   → LastConclusion = Broken

tout le reste → LastConclusion inchangé
```

C'est le point le plus subtil du modèle, et il mérite un exemple.

Une PR dont **tous** les jobs sont skippés : GitHub affiche un rond vert, car un
job skippé ne bloque pas le merge. Mais **rien n'a été testé**. Si on mappait ça
sur `Healthy`, une PR précédemment cassée serait considérée réparée sans qu'aucun
test n'ait tourné. On garde donc `LastConclusion` inchangé.

Ce qui donne des lignes comme :

```text
  SUBJECT   CHECKS      STATE
  #438      cancelled   broken     ← rien conclu cette fois, cassée avant
  #442      skipped     healthy    ← rien testé, était saine, le reste
  #446      running     broken     ← re-run en cours, toujours cassée
```

Les deux colonnes ensemble racontent quelque chose qu'une colonne unique ne
saurait pas dire.

## 17. Mapping des conclusions provider

Le domaine ne connaît pas les valeurs de GitHub. Le mapping vit dans
`RepositoryWatcher.GitHub`.

| Conclusion GitHub | Catégorie domaine |
|---|---|
| `success` | succès |
| `failure` | échec |
| `timed_out` | échec |
| `startup_failure` | échec |
| `action_required` | action requise |
| `cancelled` | annulé |
| `skipped` | skippé |
| `neutral` | non concluant |
| `stale` | non concluant |
| *(inconnue)* | non concluant |

**Note de vocabulaire.** Les catégories retenues (`skipped`, `cancelled`,
`action required`) portent des noms qui ressemblent à ceux de GitHub. Ce n'est
pas une fuite d'abstraction : ce sont des notions universelles de CI, présentes
chez GitLab, Azure DevOps ou Jenkins sous des noms voisins. Ce que le domaine
refuse, ce sont les valeurs arbitraires sans équivalent ailleurs
(`startup_failure`, `stale`), qui sont absorbées dans des catégories génériques.

Une valeur inconnue de l'adaptateur ne doit jamais faire échouer la lecture : on
la traite comme non concluante.

## 18. La santé globale

```csharp
public enum WatchHealth
{
    Unknown,
    Healthy,
    Warning,
    Critical
}
```

Règle de projection, dans cet ordre :

```text
la branche de référence est cassée
    → Critical

sinon, au moins une PR non-draft est cassée
    → Warning

sinon, la branche de référence n'a jamais eu de verdict
    → Unknown

sinon
    → Healthy
```

### 18.1. Pourquoi Unknown existe

Un dépôt dont les workflows sont entièrement conditionnels peut n'avoir jamais
produit de verdict sur sa branche de référence. Répondre `Healthy` serait mentir :
rien n'a été vérifié. `Unknown` est une réponse calculée légitime — le modèle
existe, il a été interrogé, et sa réponse est « je ne sais pas ».

> **À ne pas confondre avec le démarrage.** Pendant le bootstrap, il n'y a pas
> encore de modèle du tout : il n'y a pas de `WatchHealth`, même `Unknown`. Les
> deux situations produisent le même rendu (violet, voir section 29), mais elles
> sont différentes : l'une est une réponse, l'autre est une absence de réponse.

### 18.2. Les PR draft ne comptent pas

Une PR draft est du travail en cours. Ses checks échouent normalement pendant
l'itération. Si elle contribuait au `Warning`, l'Orb resterait orange en
permanence sur un dépôt actif, et le signal perdrait toute valeur.

Les drafts sont **affichées** dans le tableau avec leur état réel et un marqueur,
mais elles ne contribuent ni à `WatchHealth` ni aux alertes.

### 18.3. La priorité de la branche de référence est absolue

```text
branche de référence cassée
PR #42 cassée
PR #51 cassée

→ WatchHealth = Critical
```

L'Orb ne montre qu'un rouge global. Le tableau, lui, montre les trois problèmes.
C'est précisément la raison d'être du tableau.

## 19. Les événements métier

Le jeu d'événements est réduit à ce qui **produit un signal**. Les brouillons en
listaient une douzaine, dont la plupart n'existaient que pour alimenter un
journal aujourd'hui supprimé.

```text
TrackedBranchBroken
TrackedBranchRecovered
PullRequestBroken
PullRequestRecovered
ReleasePublished
```

Une PR nouvellement ouverte, un commit poussé, une CI qui démarre : tout cela
apparaît dans le tableau sans produire d'événement, parce que le tableau lit le
modèle et non un flux.

### 19.1. Règle de calcul des transitions

> Une transition se calcule **uniquement** sur le changement de
> `LastConclusion`. Jamais sur le changement de SHA, jamais sur le `RunStatus`.

Corollaire : une PR déjà cassée dont la CI échoue à nouveau ne produit **pas** de
nouvel événement `PullRequestBroken`. Elle était cassée, elle l'est toujours. Il
n'y a pas de nouvelle dégradation, donc pas de nouvelle alerte.

C'est ce qui évite d'être interrompu à chaque itération sur une PR qu'on est en
train de réparer.

### 19.2. Détection d'une nouvelle release

Une release n'est considérée comme nouvelle que si sa date de publication est
**strictement postérieure** à celle de la release connue.

Sans cette précision, la suppression d'une release provoquerait un faux positif :
la « dernière release » redeviendrait la précédente, qui diffère de l'état connu,
et un naïf comparateur annoncerait une publication.

## 20. Signaux génériques

`RepositoryWatcher` traduit ses événements métier en signaux génériques
compréhensibles par Core. C'est ce qui permet aux outputs de ne rien savoir des
pull requests.

```text
PullRequestBroken         → WarningRaised
TrackedBranchBroken       → CriticalRaised
ReleasePublished          → Celebration
*Recovered                → (aucun signal ponctuel)
```

Une réparation ne produit pas d'animation : elle produit simplement une nouvelle
projection de l'état courant. Il n'y a pas de « clignotement vert ».

Ces signaux sont l'**entrée** de Core. Ce que Core en fait est décrit en
Partie V.

## 21. Ordre déterministe

Un seul `Reconcile` peut produire plusieurs événements. Leur ordre doit être
déterministe : d'abord la branche de référence, puis les PR par numéro croissant.

Cela ne change rien à l'affichage, mais rend les tests reproductibles.

## 22. Single writer

`RepositoryState` a **un seul écrivain logique**. Aucune modification concurrente.

```text
polling
   │
   ▼
Channel<RepositorySnapshot>     (borné, capacité 1, DropOldest)
   │
   ▼
boucle de réconciliation        ← seul écrivain
   │
   ├─► RepositoryState
   └─► Event[]
```

Le canal est **borné à un élément** avec abandon du plus ancien : si un snapshot
est encore en attente de traitement quand le suivant arrive, le plus ancien est
inutile. Cela évite aussi qu'une accumulation ne fasse traiter des snapshots
périmés — et la garde de monotonie (section 11.1) rattrape le reste.

Cette boucle vit dans `RepositoryWatcher`, pas dans Core (section 7.6).

## 23. Les deux horloges

Trois usages du temps coexistent, et il faut éviter de les confondre.

| Usage | Source | Pourquoi |
|---|---|---|
| Estampiller `LastStatusChangedAt` | `snapshot.ObservedAt` | Garde `Reconcile` pure et déterministe |
| Calculer les échéances d'animation | Horloge de Core | Indépendante des lectures |
| Afficher « il y a 12m » dans le tableau | **Heure courante** au moment du rendu | Sinon l'affichage se fige entre deux lectures |

Le troisième point est important : les durées affichées sont recalculées à chaque
rendu par rapport à l'heure courante, pas par rapport au dernier snapshot. C'est
ce qui permet au bandeau `updated 4m ago` de continuer à s'incrémenter
précisément quand la lecture ne passe plus — c'est-à-dire au moment où
l'information est la plus utile.

L'horloge de Core doit être **injectable** (`TimeProvider`), pour que la file
d'animations, les durées minimales et les échéances soient testables sans
attendre trente secondes réelles.

## 24. Récapitulatif du modèle

Les quatorze sections précédentes se relient ainsi :

```text
        ┌──────────────────────────────────────────────┐
        │  Snapshot (immuable, complet, horodaté)       │
        │  ┌────────────────────────────────────────┐  │
        │  │ par sujet : HeadSha + N check runs      │  │
        │  └────────────────────────────────────────┘  │
        └──────────────────────┬───────────────────────┘
                               │
              agrégation des runs du HEAD SHA (§15)
                               │
                               ▼
              ┌────────────────────────────────┐
              │  running · failed · passed      │
              │  action required · skipped      │
              │  cancelled · inconclusive· none │
              └────────────────┬───────────────┘
                               │
        seuls « passed » et « failed » comptent (§16)
                               │
                               ▼
              ┌────────────────────────────────┐
              │  LastConclusion                 │
              │  Unknown | Healthy | Broken     │
              └────────────────┬───────────────┘
                               │
        transition = changement de LastConclusion
        uniquement — jamais le SHA, jamais le RunStatus (§19.1)
                               │
              ┌────────────────┴────────────────┐
              ▼                                 ▼
    ┌──────────────────┐            ┌──────────────────────┐
    │  WatchHealth     │            │  Event[]             │
    │  §18             │            │  §19                 │
    └────────┬─────────┘            └──────────┬───────────┘
             │                                 │
             ▼                                 ▼
    couleur stable                    signaux → file → alertes


  Ce que le tableau affiche :
      colonne CHECKS  ← l'agrégat, sans mémoire
      colonne STATE   ← LastConclusion, avec mémoire
```

La lecture en une phrase : *l'agrégat décrit l'instant, `LastConclusion` se
souvient, et seule la mémoire déclenche quelque chose.*

---

# PARTIE IV — L'ÉCRAN

## 25. Le rôle du terminal

Le terminal affiche **un tableau d'état, en continu**. Pas un flux, pas un
journal, pas un rapport ponctuel.

Cette décision remplace plusieurs artefacts des brouillons : il n'y a plus de
« rapport initial » ni de « nouveau rapport après reconnexion ». Le tableau
*est* le rapport, affiché en permanence.

Conséquence heureuse : la règle « le rapport doit exposer les problèmes masqués
par une criticité supérieure » n'a plus besoin d'être une règle. Elle est vraie
par construction, puisque toutes les lignes sont visibles tout le temps.

## 26. Le tableau

```text
  Reefact/just-dummies · personal · every 10s · orb-1

  SUBJECT    TITLE                      CHECKS     STATE     MERGE      AGE   CHANGED
  ● develop  —                          running    broken    —          —     4m
  ● #438     Add playground support     running    broken    clean      3d    12m
  ○ #442     Fix boundary condition     passed     healthy   clean      1d    1m
  ○ #451     Refactor snapshot types    passed     healthy   conflict   2d    5h
  ◌ #446     Decimal support            failed     broken    ~          2h    31s

  latest release  v1.2.0 · 3d

  HEALTH  CRITICAL                                        live · updated 2s ago
```

### 26.0. Les symboles

La pastille en tête de ligne reprend la santé de la ligne :

```text
●  rouge    la branche de référence est cassée
●  orange   une PR est cassée
○  vert     sain
◌  gris     draft, ou état inconnu — ne compte pas dans la santé
```

**Contrainte d'accessibilité.** L'information ne doit jamais reposer sur la
couleur seule. Le tableau écrit déjà `broken` / `healthy` / `unknown` en toutes
lettres dans la colonne STATE, et la forme du symbole distingue les cas
indépendamment de la couleur. Cette propriété doit être préservée par toute
évolution : une partie des utilisateurs ne distingue pas le rouge de l'orange ni
du vert.

Pour l'Orb, la marge est faible — mais c'est acceptable, puisque l'Orb ne porte
jamais d'information que le tableau ne donne pas.

### 26.1. Les colonnes

| Colonne | Contenu |
|---|---|
| **SUBJECT** | La branche de référence (son vrai nom) puis les PR par numéro. |
| **TITLE** | Titre court, tronqué à largeur fixe. Sans lui, un numéro de PR ne dit rien. |
| **CHECKS** | L'agrégat de la section 15. Ce que la CI a rendu au dernier passage. |
| **STATE** | `healthy` / `broken` / `unknown`. La santé métier, celle qui alimente la couleur. |
| **MERGE** | `clean` / `conflict` / `~` (calcul en cours chez le provider). |
| **AGE** | Depuis quand la PR existe. Répond à « celle-ci traîne ». |
| **CHANGED** | Depuis quand `STATE` n'a pas changé. Répond à « celle-ci vient de casser ». |

### 26.2. Pourquoi deux colonnes de temps

Elles répondent à des questions différentes et complémentaires :

```text
#438 · broken · AGE 3d · CHANGED 12m    → elle vient de casser, réagis
#451 · broken · AGE 3d · CHANGED 3d     → personne ne s'en occupe depuis 3 jours
```

`CHANGED` affiche `—` tant qu'aucune transition n'a été observée depuis le
démarrage (section 12.1). C'est le prix de l'absence d'historique, et il faut
l'assumer visiblement plutôt que d'inventer une date.

### 26.3. Pourquoi MERGE mérite une colonne

Parce que c'est actionnable. `passed` + `clean` se lit d'un coup d'œil comme
« je peux merger, go GitHub ». C'est le seul élément de contexte GitHub qui
déclenche une action et qui ne soit pas déjà dans les autres colonnes.

**Un conflit ne contribue pas à `WatchHealth`.** Un conflit n'est pas une
dégradation : c'est un fait normal du cycle de vie d'une PR. S'il rendait l'Orb
orange, un dépôt actif serait orange en permanence.

`Mergeable` est calculé de façon asynchrone chez GitHub : après un push sur la
base, la valeur passe temporairement à « inconnu » pendant le recalcul. Cet état
transitoire (`~`) ne doit jamais être traité comme un problème.

### 26.4. Ordre des lignes

Un dépôt peut avoir quarante PR ouvertes. Le tableau doit rester lisible sans
défilement — et surtout, **il ne doit pas bouger**.

> **Le piège à éviter.** Il serait tentant de trier par « ce qui a changé
> récemment », pour mettre l'actualité en haut. Ce serait une erreur : ce critère
> change à chaque lecture, donc les lignes se réordonneraient en permanence. Un
> tableau dont les lignes sautent est illisible — un comble pour un écran conçu
> pour être consulté d'un coup d'œil périphérique.

L'ordre retenu ne dépend donc que de critères **qui bougent rarement** :

```text
1. la branche de référence, toujours en première ligne
2. les PR par numéro décroissant (les plus récentes en haut)
```

C'est tout. Une PR ne change jamais de place tant qu'elle est ouverte. Une
nouvelle PR s'insère en haut, une PR fermée disparaît, et rien d'autre ne bouge.

L'actualité est portée par la colonne CHANGED, pas par la position.

### 26.5. Limite de lignes

Au-delà de la place disponible, une ligne de synthèse remplace le reste :

```text
  … 24 autres PR · 0 cassée
```

Les lignes conservées sont choisies dans cet ordre : la branche de référence,
puis les PR cassées, puis les plus récentes. Mais **l'affichage reste trié par
numéro décroissant** : la sélection décide qui est visible, pas dans quel ordre.

Les PR cassées ne sont **jamais** masquées : si elles ne tiennent pas, c'est le
reste qui disparaît.

### 26.6. Rythme de rendu

Le tableau se redessine **à son propre rythme**, indépendamment des lectures — de
l'ordre de la seconde.

C'est nécessaire, pas cosmétique. Les colonnes AGE et CHANGED affichent des
durées : si le rendu n'avait lieu qu'à la réception d'un snapshot, elles se
figeraient pendant tout l'intervalle. Et le bandeau `updated Xs ago`, dont
l'intérêt est précisément de dire depuis combien de temps rien n'est arrivé,
resterait figé au moment où la lecture ne passe plus — c'est-à-dire exactement
quand il devient utile.

```text
lecture      toutes les 10 s     → met à jour le contenu
rendu        toutes les 1 s      → met à jour l'affichage
```

### 26.7. Bandeau de bas de tableau

```text
HEALTH  CRITICAL                          live · updated 2s ago
```

Il porte la santé globale et l'état de fraîcheur (section 32). C'est ce qui
remplace les quatre lignes `SYSTEM` des brouillons : plutôt que de raconter
« connexion perdue », « reconnexion », « connexion rétablie », un seul indicateur
dit en permanence si ce qui est affiché est à jour.

```text
live · updated 2s ago              tout va bien
stale · last update 4m ago         la lecture ne passe plus
```

### 26.8. Contraintes techniques

Le tableau et le bandeau doivent être rendus dans une **seule** région
`AnsiConsole.Live()`. Spectre reprend la main sur sa région pour la redessiner ;
on ne peut pas y mêler des lignes écrites indépendamment. C'est aussi la raison
pour laquelle le diagnostic va dans un fichier (section 48).

La mise en page doit être **recalculée à chaque rendu**, pas figée au démarrage :
la largeur du terminal peut changer en cours de session, et la troncature des
titres doit suivre.

---

# PARTIE V — LES INDICATOR DEVICES

## 27. Rôle et statut

Un indicator device attire l'attention. Il ne porte pas d'information détaillée.

```text
LE TABLEAU     porte la vérité              → doit être exact
LES DEVICES    attirent l'attention         → une animation ratée n'est pas grave
```

Cette hiérarchie explique plusieurs choix qui suivent. On peut se permettre
qu'une animation soit écrasée ou raccourcie ; on ne peut pas se permettre que le
tableau mente.

Un device peut être **physique** (Orb Luxafor, buzzer) ou **virtuel** (une
fenêtre qui clignote). La distinction n'a pas d'importance : ce qui compte est
qu'il reçoive un signal et le rende perceptible.

**Plusieurs devices peuvent être actifs simultanément**, quels qu'ils soient :
trois Orbs, un Orb et un buzzer, un Orb d'une marque et un d'une autre. Ils
reçoivent tous le même signal et chacun le rend à sa façon.

## 28. IndicatorSignal

Core ne transmet pas d'ordres impératifs (« clignote », « fais un rainbow »). Il
publie un **état résolu** que chaque device applique.

```csharp
IndicatorSignal
{
    Base     : Unknown | Health(Healthy | Warning | Critical),
    Overlay  : None | BlinkWarning | BlinkCritical | Rainbow | Wave,
    Until    : Instant?          // échéance de l'overlay, null si permanent
}
```

### 28.1. Pourquoi un état résolu plutôt que des ordres

C'est une décision structurante, et voici le raisonnement.

Si les devices recevaient les signaux bruts (`WarningRaised`, `Celebration`…),
chacun devrait savoir : quelle est la santé de fond, quelle animation est en
cours, quelle est la table de priorités, combien de temps il reste. Autrement
dit, chaque device réimplémenterait la même logique — et avec trois devices
actifs, ils divergeraient.

En publiant un état résolu, **toute la logique vit une seule fois dans Core**.
Les devices deviennent des traducteurs sans mémoire.

Trois bénéfices concrets :

- **La priorité devient testable en pur unitaire.** On construit une séquence de
  signaux, on vérifie les `IndicatorSignal` produits. Aucun périphérique, aucune
  attente réelle de quinze secondes.
- **La reconnexion d'un device devient triviale.** Un Orb rebranché demande le
  signal courant et l'applique. Il n'a rien à reconstituer.
- **Une seule horloge.** Avec des ordres impératifs, chaque device aurait son
  chronomètre, et ils divergeraient.

## 29. Les trois axes du signal

La sémantique lumineuse tient en trois axes **orthogonaux**. C'est ce qui la rend
mémorisable sans documentation.

```text
COULEUR        ce que je sais
               violet = rien encore, et c'est normal
               vert / orange / rouge = la santé

WAVE           ce que je montre n'est pas fiable
               sur violet   = le démarrage ne se passe pas normalement
               sur une santé = c'était vrai, je ne peux plus le vérifier

CLIGNOTEMENT   quelque chose vient de se dégrader
```

Plus une quatrième animation, volontairement non prioritaire :

```text
RAINBOW        célébration d'une release
```

### 29.1. Le violet

Le violet est une couleur **hors palette de santé**. Elle signifie : « je réponds,
je n'ai pas d'information sur la santé du dépôt, et c'est normal à ce stade ».

Elle sert dans trois situations :

```text
identification d'un device pendant la sélection
bootstrap, avant le premier modèle
WatchHealth.Unknown
```

> **Pourquoi une couleur hors palette pour l'identification.** Si l'on utilisait
> le vert pour tester un périphérique, un utilisateur qui hésite deux secondes
> croirait lire un état de dépôt. Le violet est non ambigu par construction.

Les trois situations partagent le même rendu parce qu'elles disent la même chose
à l'utilisateur. Leurs origines diffèrent (absence de modèle vs modèle qui
répond « je ne sais pas »), mais cette différence est lisible dans le tableau,
pas nécessaire dans la lumière.

> **Une seule couleur hors palette.** Il serait tentant d'en ajouter une pour
> distinguer sélection et démarrage. On s'y refuse : la force de ce vocabulaire
> est de tenir en une phrase. Une couleur supplémentaire serait dépensée sans
> contrepartie, et le budget doit rester disponible pour un besoin réel futur.

### 29.2. Ce que la transition violet → couleur signifie

Le violet est continu de la sélection jusqu'au premier modèle complet. La bascule
vers une couleur de santé marque **l'instant où l'outil devient opérationnel**.
C'est un signal unique, net, sans extinction intermédiaire.

Cela remplace la règle des brouillons « l'Orb ne doit pas être allumé avant que
le modèle soit chargé ». La règle correcte est :

> Aucune **couleur de santé** ne doit être affichée avant qu'un modèle complet
> n'existe. Le violet, lui, peut s'allumer immédiatement — et c'est mieux, car il
> confirme tout de suite que le bon device répond.

## 30. La file d'animations

### 30.1. Le principe

Les signaux ponctuels (`WarningRaised`, `CriticalRaised`, `Celebration`) sont
**mis en file**, pas préemptés. Chaque élément a droit à un moment d'existence :
sans cela, un signal peut être écrasé par le suivant avant d'avoir été
perceptible.

```text
signal reçu
   │
   ▼
identique à celui en cours ?  ──oui──►  on prolonge l'échéance
   │ non                                (aucun changement visible)
   ▼
critique ?  ──oui──►  passe en tête de file
   │ non
   ▼
mis en file
```

### 30.2. Durées

```text
BlinkWarning     15 s par défaut, configurable
BlinkCritical    30 s par défaut, configurable
Rainbow           5 s
Durée minimale    3 s
```

La **durée minimale** s'applique à tout élément qui n'est pas le dernier de la
file : quand plusieurs signaux sont en attente, les précédents jouent 3 secondes
et on dépile, seul le dernier joue sa durée nominale. En dessous de 3 secondes,
une animation n'est pas perceptible du coin de l'œil — or c'est précisément
l'usage visé.

### 30.3. Fusion des signaux identiques

Deux signaux identiques consécutifs ne produisent qu'une animation. Ce n'est pas
une optimisation, c'est une constatation : redemander à un device l'animation
qu'il joue déjà ne change rien visuellement. On ne peut pas distinguer la fin
d'une pulsation orange du début de la suivante.

Concrètement, trois PR qui cassent en douze secondes produisent **un seul**
clignotement orange dont l'échéance recule.

```text
T+00   #42 casse   → blink orange jusqu'à T+15
T+08   #51 casse   → blink orange jusqu'à T+23
T+12   #57 casse   → blink orange jusqu'à T+27
```

> **Ce n'est pas une perte d'information.** On pourrait vouloir faire clignoter
> trois fois pour signaler trois PR. Ce serait une erreur : personne ne compte
> des pulsations du coin de l'œil, et l'Orb n'est pas le canal où lire une
> donnée. Il attire l'attention ; le tableau donne le compte exact.

La file ne prend donc son sens que lorsque les éléments **diffèrent**.

### 30.4. La wave n'est pas dans la file

Distinction importante :

```text
BLINK et RAINBOW   déclenchés par un événement, ont une durée   → file
WAVE               reflète un état, dure tant qu'il dure        → hors file
```

La wave n'attend pas son tour et ne s'épuise pas. Tant que la fraîcheur est
dégradée, elle est active et prend le pas sur tout le reste.

## 31. Priorités

```text
WAVE (fraîcheur dégradée)
    >
BLINK CRITICAL
    >
BLINK WARNING
    >
RAINBOW
    >
couleur stable
```

Trois conséquences :

**Le critique passe devant.** Si une PR casse puis la branche de référence une
seconde après, le rouge ne doit pas attendre la fin de l'orange. Le niveau
d'urgence n'est pas le même.

**Une célébration ne masque jamais un problème.** Un `Celebration` reçu alors
que `WatchHealth` n'est pas `Healthy` est **abandonné**, pas mis en file. On ne
célèbre pas sur un dépôt cassé. La release reste visible dans le tableau
(`latest release · v1.2.0 · 3m`), donc rien n'est perdu.

**Une dégradation interrompt le rainbow immédiatement.** Si une alerte survient
pendant l'animation de célébration, celle-ci est abandonnée et n'est pas
reprise.

## 32. La fraîcheur

Un état technique **orthogonal** à la santé métier. Une erreur réseau ne doit
jamais modifier artificiellement la santé du dépôt.

```csharp
public enum Freshness
{
    Starting,        // pas encore de modèle
    Live,            // le modèle est à jour
    Stale,           // la lecture ne passe plus, transitoire
    Failed           // irrécupérable, on s'arrête
}
```

> **Renommage.** Les brouillons parlaient de `MonitoringState`. Le terme est
> impropre : ce n'est pas l'état d'un système de monitoring, c'est le degré de
> confiance dans ce qui est affiché.

Les combinaisons sont libres :

```text
Healthy  + Live      Warning  + Live      Critical + Live
Healthy  + Stale     Warning  + Stale     Critical + Stale
```

`Critical + Stale` signifie exactement : *la dernière vérité connue est critique,
mais je ne peux plus vérifier qu'elle est encore exacte.*

### 32.1. Stale : ce qui le déclenche

Erreurs transitoires du provider : HTTP 5xx, timeout, perte réseau, rate limit
bloquant, réponse GraphQL partielle.

Effet :

- le dernier modèle connu est **conservé** ;
- `Freshness` passe à `Stale` ;
- une **wave** se déclenche dans la couleur de santé actuellement connue ;
- le bandeau du tableau indique depuis quand la lecture ne passe plus ;
- les tentatives de lecture continuent.

```text
Orb rouge   → wave rouge
Orb orange  → wave orange
Orb vert    → wave verte
```

La couleur reste celle du dernier état métier connu. Seul le mouvement dit que
l'information n'est plus garantie.

### 32.2. Failed : ce qui le déclenche

Erreurs **définitives**, où continuer d'essayer n'a aucun sens :

- token révoqué, expiré ou invalide (401) ;
- droits insuffisants sur le dépôt ;
- dépôt inexistant, supprimé ou renommé (404) ;
- connexion référencée mais absente de la configuration.

Effet : message explicite, **extinction des devices**, arrêt du processus avec un
code non nul.

> **Piège à ne pas manquer : le 403 est ambigu.** GitHub renvoie 403 aussi bien
> pour un **défaut de permission** (définitif) que pour un **dépassement de
> quota** (transitoire). Classer tout 403 en `Failed` tuerait le processus au
> premier rate limit — un événement de fonctionnement normal, pas un cas limite.
>
> La distinction doit se faire sur les en-têtes de réponse
> (`x-ratelimit-remaining` à zéro, `retry-after` présent), jamais sur le code
> seul. C'est l'adaptateur qui tranche et qui lève l'exception du bon type
> (section 7.7).

> **Pourquoi éteindre plutôt que passer en wave.** La wave dit « je ne peux plus
> vérifier », ce qui sous-entend que ça reviendra. Un token révoqué ne reviendra
> pas. Laisser une lumière allumée sur un état qu'on ne maintiendra plus est
> exactement ce que la règle d'arrêt propre interdit.

## 33. Les ports

Deux contrats distincts : découvrir, et commander.

### 33.1. Découverte

```csharp
public interface IIndicatorDeviceProvider
{
    string OutputId { get; }

    Task<IReadOnlyList<IndicatorDeviceDescriptor>> DiscoverAsync(
        CancellationToken ct);

    Task<IIndicatorDevice> OpenAsync(
        IndicatorDeviceId id,
        CancellationToken ct);
}
```

```csharp
IndicatorDeviceDescriptor
{
    Id,              // stable entre deux lancements, dans la mesure du possible
    OutputId,        // « luxafor »
    Label,           // libellé brut, souvent illisible
    Capabilities,
    IsAvailable      // faux si déjà verrouillé par une autre instance
}
```

**Format de l'identifiant** : `<outputId>:<deviceKey>`, par exemple
`luxafor:orb-1`. La partie gauche identifie l'output, ce qui garantit l'unicité
même si deux outputs numérotent leurs devices de la même façon.

`deviceKey` est produit par l'output. Il doit être aussi stable que le matériel
le permet — numéro de série s'il est exposé, index d'énumération sinon. Voir la
réserve en section 54.

**Interaction avec la ligne de commande** : `--device` **remplace** la sélection
enregistrée pour la session en cours, sans la modifier durablement. Cela permet
un usage ponctuel sur un autre device sans reconfigurer.

### 33.2. Commande

```csharp
public interface IIndicatorDevice : IAsyncDisposable
{
    IndicatorDeviceId    Id           { get; }
    IndicatorCapabilities Capabilities { get; }

    Task ApplyAsync(IndicatorSignal signal, CancellationToken ct);
    Task IdentifyAsync(CancellationToken ct);
    Task TurnOffAsync(CancellationToken ct);
}
```

Trois méthodes, parce que toute la logique est chez Core.

- `ApplyAsync` reçoit le signal résolu et le rend selon ses capacités. **Sans
  état interne** : le device ne se souvient pas de ce qu'il jouait avant.
- `IdentifyAsync` sert au wizard de sélection : le device se signale pour être
  reconnu.
- `TurnOffAsync` sert à l'arrêt et aux candidats rejetés pendant la sélection.

`IAsyncDisposable` porte la libération du verrou et de la ressource.

### 33.3. Capacités

```csharp
[Flags]
public enum IndicatorCapabilities
{
    None            = 0,
    SteadyColor     = 1,
    Blink           = 2,
    Wave            = 4,
    Rainbow         = 8,
    PointInTime     = 16    // ne sait que marquer un instant
}
```

**Chaque device décide lui-même de sa dégradation.** Core envoie toujours le
signal complet ; le device en prend ce qu'il sait rendre.

Exemple d'un buzzer, qui n'a que `PointInTime` :

```text
Base = Health(Critical)      → ignoré, rien à rendre en continu
Overlay = BlinkCritical      → double bip court
Overlay = Wave               → ignoré
Overlay = Rainbow            → ignoré
Overlay = None               → rien
IdentifyAsync                → bip distinctif
TurnOffAsync                 → rien à faire
```

Deux remarques que ce cas fait apparaître :

- Un device sans capacité continue ne peut pas répondre à « quel est l'état
  maintenant ». C'est sa nature, pas un défaut — le tableau reste la source
  d'état permanent.
- Un buzzer qui revient après une déconnexion ne doit **pas** rejouer les bips
  manqués. Il applique le signal courant : s'il n'y a pas d'overlay actif, il ne
  fait rien. Aucun traitement particulier n'est nécessaire.

Core doit avertir au démarrage si un signal important sera invisible sur tous les
devices sélectionnés — par exemple si aucun ne sait rendre la wave.

## 34. Sélection des devices

### 34.1. Ordre de décision au démarrage

```text
--no-signal              → aucune découverte n'est même tentée
                           les devices ne sont pas énumérés

sinon, découverte :
  0 device disponible    → avertissement, on continue sans signal
  1 device               → sélection silencieuse
  n, défaut valide       → on le prend, affiché dans le tableau
  n, pas de défaut       → wizard, et on enregistre le choix
```

### 34.2. Le wizard

Les libellés matériels sont souvent illisibles (`HID\VID_04D8&PID_F372`).
L'utilisateur identifie donc son device **en le regardant réagir**.

```text
Plusieurs indicator devices détectés :

  1. Luxafor Flag       HID\VID_04D8&PID_F372
  2. Luxafor Orb        HID\VID_04D8&PID_F372
  3. Luxafor Orb        HID\VID_04D8&PID_F372   (déjà utilisé)

Lequel voulez-vous utiliser ? [1-3, ou plusieurs séparés par une virgule] 2

  → le device 2 s'allume en violet

Est-ce le bon ? [o/N]
```

Points de conception :

- le test se fait **avant** la sélection définitive ;
- les candidats rejetés sont éteints immédiatement ;
- la sélection peut être **multiple** ;
- les devices déjà verrouillés par une autre instance sont listés mais non
  sélectionnables ;
- le choix est enregistré dans la configuration et ne sera plus redemandé.

> Il n'y a pas de mode non interactif. `uwatch` est un outil de poste de travail,
> lancé par un humain devant son terminal. Si l'entrée standard ne permet pas de
> poser la question, on affiche une erreur explicite.

**Accès matériel sous Linux.** Luxafor est un périphérique HID : l'accès sans
privilèges exige une règle udev. Sans elle, selon la configuration, la découverte
ne verra rien — ou verra le device mais échouera à l'ouvrir, ce qui est plus
déroutant.

Les deux cas doivent produire des messages distincts :

```text
No indicator device found.

Luxafor Orb found but not accessible.
On Linux, a udev rule is required. See <lien vers la documentation>.
```

### 34.3. Changer de choix

Le wizard ne se déclenche qu'une fois. Il faut donc un moyen de revenir sur la
sélection sans éditer le fichier de configuration à la main — on change de
bureau, on débranche un Orb pour un autre, on s'est trompé.

```bash
uwatch device select      # relance le wizard et enregistre le nouveau choix
```

C'est aussi utile pour changer de device à froid, sans lancer une surveillance.

### 34.4. Verrou

Le verrou est **par device**, pas global.

Deux instances peuvent tourner sur deux dépôts avec un device chacune : c'est un
usage prévu. Ce qui est interdit, c'est que deux instances pilotent le **même**
device — on obtiendrait des commandes concurrentes et des animations
indéterministes.

Le matériel ne protège pas de lui-même : plusieurs processus peuvent ouvrir le
même HID en écriture. Le verrou est donc **applicatif**.

**Il doit être implémenté par un fichier de verrou** ouvert en accès exclusif
(`FileShare.None`), pas par un mutex nommé.

> **Pourquoi cette précision.** Le réflexe naturel est le mutex nommé, mais son
> comportement n'est pas homogène entre Windows et Unix, et il se libère mal
> quand un processus meurt anormalement — laissant un device inutilisable
> jusqu'au redémarrage. Un fichier ouvert en exclusif est fiable sur les trois
> plateformes, et le système d'exploitation le relâche systématiquement à la mort
> du processus, y compris en cas de crash.

Si un device est déjà verrouillé, il apparaît dans la liste comme indisponible.
Si c'était le seul, on continue sans signal, avec un avertissement.

### 34.5. Reconnexion

La perte d'un device n'altère jamais le modèle. Le tableau continue.

À la reconnexion, le device applique **le signal courant**. Il n'a rien à
reconstituer et les animations manquées ne sont pas rejouées.

> Une panne du device ne peut évidemment pas être signalée par le device
> lui-même. Elle est indiquée dans le bandeau du tableau.

### 34.6. --no-signal

L'option court-circuite toute la découverte : aucune énumération, aucune latence,
aucune question.

Elle est **globale**, pas propre à la commande `repo`.

> **Différence avec « aucun device trouvé ».** Ce sont deux états distincts. Sans
> device trouvé, un device branché en cours de session est pris en compte. Avec
> `--no-signal`, brancher un device ne déclenche rien : l'utilisateur a déjà
> répondu.

L'implémentation propre est un `IIndicatorDevice` neutre, pas une condition
disséminée dans le code : le CLI ne doit pas avoir deux chemins d'exécution.

Le tableau indique toujours dans quel mode il tourne :

```text
orb-1                  un device actif, et lequel
no signal              désactivé volontairement
no device found        aucun trouvé
```

---

# PARTIE VI — CYCLE DE VIE

## 35. Démarrage

La séquence est courte. Les brouillons en comptaient treize étapes, dont
plusieurs n'existaient que pour orchestrer un rapport initial figé qui n'existe
plus.

```text
1.  Résoudre l'alias du dépôt
2.  Résoudre la connexion et le provider
3.  Sélectionner les devices          → violet fixe
4.  Se connecter au provider
5.  Lire un premier snapshot complet
6.  CreateInitial                     → le modèle existe
7.  Afficher le tableau
8.  Appliquer la couleur de santé     → fin du violet
9.  Boucle : lire, reconcilier, projeter
```

### 35.1. Progression affichée

Avant que le tableau ne prenne la main :

```text
UniversalWatcher

● Resolving repository
  just-dummies → Reefact/just-dummies

● Selecting indicator device
  orb-1 (Luxafor Orb)

● Connecting to GitHub
  Connection: personal · Account: Reefact
  Connected

● Loading repository state
  develop loaded
  6 open pull requests loaded
  Model ready
```

> Ce n'est pas un journal d'événements : c'est une progression de démarrage. Elle
> disparaît quand le tableau s'affiche.

### 35.2. Pas de buffering

Les brouillons prévoyaient un `BootstrapBuffer` : les événements arrivant pendant
l'affichage du rapport initial étaient mis de côté puis rejoués.

Ce mécanisme n'a plus d'objet. Il n'existe plus de rapport figé pendant lequel le
modèle serait en attente. La séquence est : on construit, on affiche, on
reconcilie en continu. Un snapshot qui arrive pendant l'affichage est simplement
le premier de la boucle.

### 35.3. Aucun faux événement au démarrage

Garanti structurellement par `CreateInitial` (section 12), pas par vigilance.

### 35.4. Démarrage anormal

Si le premier snapshot n'arrive pas (provider injoignable, timeout), l'Orb passe
en **violet + wave** : je réponds, je devrais savoir, je n'y arrive pas.

Si l'erreur est définitive (section 32.2), on éteint et on sort.

## 36. Boucle live

```text
     ┌──────────────────────────────────┐
     │  attendre l'intervalle           │
     └──────────────────────────────────┘
                   │
                   ▼
     ┌──────────────────────────────────┐
     │  lire un snapshot complet        │──── échec transitoire ──► Stale + wave
     └──────────────────────────────────┘                           on retente
                   │                     └──── échec définitif ────► Failed
                   ▼                                                 on sort
     ┌──────────────────────────────────┐
     │  garde de monotonie              │──── périmé ──► on ignore
     └──────────────────────────────────┘
                   │
                   ▼
     ┌──────────────────────────────────┐
     │  Reconcile                       │
     └──────────────────────────────────┘
              │            │
              ▼            ▼
        nouveau State   Event[]
              │            │
              ▼            ▼
          tableau     signaux → Core → IndicatorSignal → devices
```

## 37. Retour de Stale à Live

Rien de particulier : on lit un snapshot complet et on reconcilie. C'est le
chemin normal.

```text
lecture qui repasse
    ↓
Reconcile avec l'état conservé
    ↓
le tableau reflète la nouvelle réalité
    ↓
Freshness = Live, la wave s'arrête
    ↓
la couleur de santé reprend
```

Deux points :

- **Aucun événement n'est fabriqué** pour les transitions survenues pendant la
  coupure. Les changements apparaissent dans le tableau, sans alerte.
- **Les animations interrompues ne sont pas reprises.** La file est vidée : elle
  ne contenait que des faits antérieurs à la coupure.

Il n'existe pas de « nouveau rapport » à ce moment : le tableau était affiché
pendant toute la coupure, il continue de l'être.

## 38. Arrêt

Sur `Ctrl+C` :

```text
1. arrêter la boucle de lecture
2. laisser se terminer le traitement en cours
3. éteindre tous les devices
4. relâcher les verrous
5. libérer les ressources
6. afficher un message de fin
```

```text
Monitoring stopped.
```

> **Pourquoi éteindre les devices.** Une lumière qui reste allumée après l'arrêt
> du processus devient une information non maintenue — donc, avec le temps, une
> information fausse. C'est pire que pas d'information du tout.

L'arrêt doit être borné dans le temps : si un device ne répond pas, on ne bloque
pas indéfiniment la sortie du processus.

### 38.1. Traitement technique

Le comportement par défaut de .NET est de terminer le processus immédiatement.
Il faut donc intercepter le signal, **annuler la terminaison par défaut**, puis
déclencher un arrêt coordonné via un jeton d'annulation.

Il faut aussi prévoir le **second** `Ctrl+C` : si l'arrêt propre traîne — device
qui ne répond pas, requête en cours — l'utilisateur doit pouvoir forcer la
sortie.

```text
1er Ctrl+C   → arrêt propre, borné à quelques secondes
2e Ctrl+C    → sortie immédiate
dépassement  → sortie forcée
```

Un device laissé allumé par une sortie forcée est un défaut assumé : mieux vaut
une lumière orpheline qu'un processus qu'on ne peut pas arrêter.

---

# PARTIE VII — CONNEXIONS ET SÉCURITÉ

## 39. Le modèle de connexion

Une **connexion** représente un accès à un fournisseur. Elle appartient au host,
pas au module métier.

```text
Connection « personal »
├── ProviderId    github
├── Host          github.com
├── Account       Reefact
└── CredentialRef référence vers le stockage sécurisé
```

Le nom `personal` n'a aucune signification : c'est un alias utilisateur. `work`,
`client-a`, `perso` sont tout aussi valides.

### 39.1. Registre des providers

Le host ne code aucun provider en dur. Chaque module technique enregistre le
sien.

```text
ConnectionProviderRegistry
    github → GitHubConnectionProvider
```

`uwatch connection add personal --provider github` est valide parce que le module
GitHub est présent dans le tool. Ajouter GitLab plus tard n'exigera aucune
modification du modèle métier.

### 39.2. Chaîne de résolution

```text
alias de dépôt          just-dummies
    ↓
configuration           connection = personal
    ↓
connexion               provider = github
    ↓
registre                GitHubConnectionProvider
    ↓
client authentifié
    ↓
IRepositorySource       GitHubRepositorySource
```

Le modèle métier ne connaît que `IRepositorySource`.

## 40. Authentification GitHub

**Décision V1 : Personal Access Token fine-grained, saisi de façon masquée.**

### 40.1. Pourquoi ce choix

Trois options étaient envisageables.

| Option | Avantage | Inconvénient |
|---|---|---|
| **PAT fine-grained masqué** | Rien à créer côté serveur, permissions restreignables au dépôt | L'utilisateur doit créer le token à la main |
| OAuth Device Flow | Meilleure expérience, pas d'expiration à gérer | Exige d'enregistrer et **maintenir** une OAuth App ; les scopes OAuth classiques sont grossiers (`repo` inclut l'écriture) |
| Déléguer à `gh` | Zéro travail | Dépendance externe non déclarée, token dont on ne maîtrise ni la durée ni la portée |

Le Device Flow donne la meilleure expérience mais engage à faire vivre une
application OAuth pour un outil dont l'auteur est le premier utilisateur. Et
demander le scope `repo` — qui inclut l'**écriture** — pour un outil qui ne fait
que lire est difficile à justifier.

Le PAT fine-grained permet au contraire de restreindre en lecture seule, dépôt
par dépôt.

Le port `ICredentialStore` et le `ConnectionProvider` doivent rester conçus pour
que le mécanisme soit interchangeable : le Device Flow deviendra pertinent le
jour où le tool sera distribué largement.

### 40.2. Permissions requises

L'utilisateur doit être guidé précisément, car une permission oubliée produit un
tableau partiellement vide plutôt qu'une erreur franche.

```text
Metadata          lecture   (obligatoire)
Contents          lecture   branche de référence, SHA
Pull requests     lecture
Checks            lecture
```

### 40.3. Saisie

```bash
uwatch connection add personal --provider github
```

Le provider demande le token en **masquant la frappe**. Il n'est jamais affiché,
jamais écho.

Exemple interdit, refusé par le CLI :

```bash
uwatch connection add personal --provider github --token github_pat_xxx
```

Un argument en clair se retrouve dans l'historique du shell, dans la liste des
processus et dans les logs éventuels. L'option n'existe pas.

## 41. Stockage des secrets

```csharp
public interface ICredentialStore
{
    Task StoreAsync(CredentialRef reference, string secret, CancellationToken ct);
    Task<string?> GetAsync(CredentialRef reference, CancellationToken ct);
    Task RemoveAsync(CredentialRef reference, CancellationToken ct);
}
```

Règles absolues :

- aucun secret en clair dans le fichier de configuration ;
- aucun secret en argument de ligne de commande ;
- aucun secret dans le fichier de diagnostic (section 48) ;
- le fichier de configuration ne contient qu'une **référence** au secret.

Le tool étant un .NET Tool multiplateforme, l'implémentation dépend du système :

```text
Windows    Windows Credential Manager
macOS      Keychain
Linux      Secret Service (libsecret) via D-Bus
```

### 41.1. Le cas Linux sans session graphique, et WSL

C'est le point à ne pas sous-estimer. Secret Service suppose D-Bus et un agent de
trousseau. Sur un poste Linux de bureau, c'est acquis. **Dans WSL et sur une
machine sans session graphique, c'est généralement absent** — or WSL est un
environnement de développement très répandu.

Refuser silencieusement rendrait l'outil inutilisable pour ces utilisateurs sans
leur donner de porte de sortie. La règle retenue :

- si un magasin sécurisé est disponible, il est utilisé, sans question ;
- s'il est absent, l'outil **le dit explicitement**, nomme la cause, et indique
  comment installer un agent de trousseau ;
- il ne bascule **jamais** silencieusement vers un stockage en clair.

```text
✗ No secure credential store available on this system.

  uwatch stores credentials in the OS keyring and will not fall back to
  plain-text storage.

  On WSL or a headless Linux machine, install and start a Secret Service
  provider (for example gnome-keyring), then run this command again.
```

### 41.2. Le fichier de configuration

Il ne contient aucun secret, mais il révèle les comptes et les dépôts surveillés.
Il doit être créé avec des permissions restrictives (lecture/écriture pour le
propriétaire uniquement).

### 41.3. Le secret en mémoire

Le secret n'est conservé que le temps nécessaire à la requête. On ne prétend pas
davantage : `SecureString` est déprécié et n'offre pas de protection réelle sur
les plateformes modernes. Affirmer le contraire donnerait une fausse assurance.

## 42. Validation

### 42.1. À la création de la connexion

Un appel de vérification immédiat, qui affiche le compte authentifié. Sans cela,
l'erreur ne se découvre qu'au premier lancement, sans savoir si c'est le token,
les permissions ou le dépôt qui est en cause.

À ce stade, aucun dépôt n'est encore configuré : la validation ne peut porter que
sur l'identité et les permissions déclarées du token.

**L'ordre des opérations compte** :

```text
1. saisie masquée
2. appel de vérification
3. ── si échec ──► message d'erreur, RIEN n'est stocké, sortie
4. ── si succès ─► stockage du secret, puis écriture de la configuration
```

Stocker puis valider laisserait un credential invalide dans le magasin de secrets
après un échec.

### 42.2. Expiration du token

Les PAT fine-grained ont une expiration **obligatoire**, d'un an au maximum. Sans
anticipation, l'utilisateur découvrira l'échéance par un arrêt brutal du
processus, un matin, sans comprendre.

GitHub renvoie la date d'expiration dans un en-tête de réponse. Elle est donc
connue à chaque requête, et l'outil doit avertir à l'avance :

```text
⚠ The token for connection « personal » expires in 6 days (2026-08-24).
```

L'avertissement apparaît au démarrage et dans le bandeau du tableau à l'approche
de l'échéance.

### 42.3. À l'ajout d'un dépôt

C'est ici que la validation devient complète : on exécute la vraie lecture. Le
message doit distinguer les cas :

```text
✗ Repository Reefact/just-dummies not found.
  It may not exist, or the token « personal » may not have access to it.

✗ Connection « personal » cannot read checks on Reefact/just-dummies.
  The token is missing the « Checks: read » permission.
```

### 42.4. En cours de session

Un token révoqué ou expiré produit un `Freshness.Failed` (section 32.2) : message
explicite, extinction, sortie.

**Aucune nouvelle tentative n'est effectuée avec le même credential.** Ce n'est
pas seulement inutile : répéter des requêtes authentifiées avec un token invalide
peut déclencher les mécanismes de détection d'abus du fournisseur.

---

# PARTIE VIII — CONFIGURATION

## 43. Fichier

```json
{
  "connections": {
    "personal": {
      "provider": "github",
      "host": "github.com",
      "account": "Reefact",
      "credentialRef": "uwatch:connection:personal"
    }
  },
  "watchers": {
    "repo": {
      "repositories": {
        "just-dummies": {
          "connection": "personal",
          "remote": "Reefact/just-dummies",
          "branch": null
        }
      }
    }
  },
  "signals": {
    "warningBlinkSeconds": 15,
    "criticalBlinkSeconds": 30,
    "devices": ["luxafor:orb-1"]
  },
  "polling": {
    "intervalSeconds": 10
  }
}
```

Aucun secret réel n'apparaît dans ce fichier.

### 43.1. Pourquoi les durées ne sont pas sous « outputs »

C'est **Core** qui possède la file, l'horloge et les priorités (section 28.1).
Les durées sont donc une configuration du host, pas d'un output particulier. Les
placer sous `outputs.luxafor` laisserait entendre que chaque output a ses propres
durées — ce qui reproduirait exactement le problème que la centralisation
résout.

### 43.2. Branche de référence

`branch: null` signifie : utiliser la branche par défaut du dépôt, telle que le
provider la déclare. C'est le cas nominal et il ne demande aucune configuration.

Une valeur explicite permet de surveiller autre chose — typiquement `develop` sur
un dépôt dont le défaut est `main`.

## 44. Durées : le value object

```csharp
BlinkDuration
```

Le domaine ne manipule jamais un entier brut représentant des secondes.

```text
minimum    1 seconde
maximum    5 minutes
```

Ces bornes empêchent les valeurs négatives, le zéro, les erreurs d'unité et les
clignotements de plusieurs heures. La validation se fait **à l'entrée de
l'application**, pour qu'une durée invalide ne puisse jamais atteindre le
domaine.

Un seul value object couvre `WarningBlinkDuration` et `CriticalBlinkDuration` :
leurs invariants sont identiques, deux types distincts seraient du bruit.

## 45. Intervalle de lecture

```text
défaut      10 secondes
plancher     5 secondes
configurable
```

### 45.1. Pourquoi 10 et pas 5

Le quota GraphQL de GitHub se compte en **points**, pas en requêtes : 5000 points
par heure. Une requête qui remonte six PR avec leurs check runs coûte plusieurs
points.

```text
intervalle 5 s   → 720 requêtes/heure
                   à 5 points l'unité : 3600 points, soit 72 % du quota
                   deux instances dépassent
intervalle 10 s  → 360 requêtes/heure, marge confortable
```

GitHub renvoie le coût et le solde dans chaque réponse. **Le coût réel doit être
mesuré pendant l'implémentation** et la valeur par défaut ajustée si nécessaire.

Le plancher à 5 secondes empêche une configuration à 1 seconde qui épuiserait le
quota en quelques minutes.

### 45.2. Rate limit atteint

C'est un cas de `Stale`, avec une particularité utile : GitHub indique quand le
quota se réinitialise. Le bandeau peut donc afficher une information exploitable
plutôt qu'un vague « connexion perdue ».

```text
stale · rate limited until 14:32
```

---

# PARTIE IX — LE CLI

## 46. Grammaire

```text
uwatch <module> <target> [options]
```

```bash
uwatch repo just-dummies
```

Cette forme prépare de futurs modules sans les imposer :

```bash
uwatch http production-api
uwatch cert justdummies.io
```

Chaque module métier apporte son préfixe. `RepositoryWatcher` apporte `repo`.

## 47. Commandes V1

**Connexions**

```bash
uwatch connection add <name> --provider <provider>
uwatch connection list
uwatch connection remove <name>
uwatch connection providers
```

**Dépôts**

```bash
uwatch repo add <owner>/<repo> --connection <name> [--alias <alias>] [--branch <branch>]
uwatch repo list
uwatch repo remove <alias>
```

**Surveillance**

```bash
uwatch repo <alias>
uwatch repo <owner>/<repo> --connection <name>     # mode ponctuel
```

**Devices**

```bash
uwatch device list        # inventaire de tous les outputs, avec disponibilité
uwatch device select      # (re)lance le wizard et enregistre le choix
```

> Le terme retenu est **device**, pas `output`. Un output est un *module*
> (Luxafor, WinForms) ; un device est un *point de sortie concret* (cet Orb-ci,
> cette fenêtre-là). C'est bien la seconde notion que la commande liste. Le
> concept s'appelle `IndicatorDevice` dans le code ; `device` suffit en ligne de
> commande, où le contexte est donné.

**Options globales**

```bash
--no-signal              n'utiliser aucun indicator device
--device <id>            forcer un device (répétable)
--debug                  écrire un fichier de diagnostic
```

**Options de surveillance**

```bash
--interval 10s
```

### 47.1. Alias

```bash
uwatch repo add Reefact/just-dummies --connection personal
```

L'alias par défaut est le nom du dépôt :

```text
Reefact/just-dummies → just-dummies
```

Alias explicite :

```bash
uwatch repo add Reefact/just-dummies --alias jd --connection personal
uwatch repo jd
```

## 48. Diagnostic

Le tableau occupe l'écran : on ne peut pas y intercaler du diagnostic.

`--debug` écrit donc dans un **fichier**, dont le chemin est affiché au
démarrage.

```text
Debug log: /home/user/.uwatch/logs/2026-08-18-142233.log
```

Le fichier ne doit contenir **aucun secret**. « Faire attention » n'est pas une
règle applicable ; voici les règles mécaniques :

- les en-têtes de requête ne sont **jamais** journalisés, quelle qu'en soit la
  raison — c'est là que vit l'`Authorization` ;
- tout ce qui ressemble à un token dans un corps journalisé est expurgé par
  motif, avant écriture ;
- les réponses d'erreur brutes du fournisseur ne sont pas journalisées telles
  quelles : elles peuvent contenir des fragments de la requête.

Le fichier de diagnostic est créé avec des permissions restrictives, comme la
configuration.

## 49. Messages et cas d'erreur

Conventions standard : usage sur la sortie standard et code 0 quand l'utilisateur
n'a rien fait de mal ; message sur l'erreur standard et code non nul en cas
d'erreur réelle.

| Situation | Comportement |
|---|---|
| `uwatch` | Usage court, liste des commandes. Code 0. |
| `uwatch repo` sans dépôt configuré | Usage, plus deux lignes indiquant les commandes à taper pour démarrer. Code 0. |
| `uwatch repo` avec des dépôts configurés | Équivalent à `repo list`. Code 0. |
| `uwatch repo <alias-inconnu>` | Erreur, **et liste des alias connus**. Code non nul. |
| Connexion référencée mais supprimée | Erreur nommant la connexion manquante. Code non nul. |

Le seul cas où l'on va au-delà de la convention est le **premier lancement sans
aucune configuration** : un `--help` face à un état vide ne dit pas par où
commencer.

```text
No repository configured yet. To get started:

  uwatch connection add personal --provider github
  uwatch repo add <owner>/<repo> --connection personal
```

La liste des alias connus sur alias inconnu n'est pas de la pédagogie : c'est de
l'information factuelle qui économise un `repo list` après une faute de frappe.

---

# PARTIE X — GARANTIES, RÈGLES ET DÉCISIONS

## 50. Notes techniques

Ces points ne sont pas des décisions de conception, mais des pièges connus qui
coûteraient du temps s'ils étaient découverts tard.

| Sujet | Note |
|---|---|
| **Boucle de lecture** | `PeriodicTimer` plutôt qu'une boucle `Task.Delay` : pas d'accumulation de dérive, annulation propre. |
| **Horloge** | `TimeProvider` injectable dans Core, sinon la file d'animations n'est testable qu'en attendant trente secondes réelles. |
| **Canal** | Borné à 1 élément, `BoundedChannelFullMode.DropOldest`. |
| **Client GraphQL** | La bibliothèque GitHub la plus répandue en .NET est orientée REST et gère mal GraphQL. Le choix reste ouvert, mais il conditionne la détection des réponses partielles (section 11.2) : à faire tôt. |
| **Verrou** | Fichier ouvert en `FileShare.None`, pas de mutex nommé (section 34.4). |
| **Ctrl+C** | Interception explicite, annulation du comportement par défaut (section 38.1). |
| **HID Linux** | Règle udev nécessaire (section 34.2). |

## 51. Les règles essentielles

Ces règles sont la colonne vertébrale du produit. Une évolution qui en contredit
une doit être considérée comme une régression.

**Règle 1 — Le tableau porte la vérité, les devices attirent l'attention.**
Le tableau doit être exact. Une animation ratée n'est pas grave.

**Règle 2 — Ce qui est montré est toujours une projection de l'état, jamais du
dernier événement.**

**Règle 3 — Le produit dit qu'un problème existe ; le développeur ouvre GitHub
pour le détail.**

**Règle 4 — Le domaine ne raconte que les transitions entre deux observations,
jamais le chemin parcouru entre elles.**
Aucun événement n'est inventé pour une période non observée.

**Règle 5 — La couleur dit ce que je sais ; la wave dit que ce que je montre
n'est pas fiable.**
Un principe unique, valable au démarrage comme après une coupure.

**Règle 6 — Le clignotement représente une **nouvelle** dégradation.**
Une PR déjà cassée qui échoue encore ne clignote pas.

**Règle 7 — Une célébration ne prime jamais sur un problème.**

**Règle 8 — Une erreur technique ne modifie jamais la santé métier.**
Elle modifie la confiance dans la fraîcheur, ce qui est autre chose.

**Règle 9 — Aucun secret en clair, nulle part.**
Ni ligne de commande, ni configuration, ni fichier de diagnostic.

**Règle 10 — Le domaine ignore GitHub, Spectre et Luxafor.**
Y compris leurs valeurs, leurs couleurs et leurs noms de branche.

## 52. Robustesse attendue

- aucune corruption du modèle en cas d'erreur réseau ;
- aucun changement de santé métier causé par une erreur technique ;
- un seul écrivain du modèle ;
- snapshots immuables et complets, ou pas de snapshot du tout ;
- garde de monotonie sur les snapshots ;
- ordre déterministe des événements produits ;
- distinction stricte entre erreur transitoire et erreur définitive ;
- reconnexion automatique raisonnable des devices ;
- gestion d'un device temporairement absent ;
- verrou par device empêchant deux instances de piloter le même ;
- arrêt propre et borné dans le temps ;
- extinction des devices à l'arrêt.

## 53. Décisions figées

**Produit et distribution**

- produit `UniversalWatcher`, commande `uwatch` ;
- tous les projets préfixés `Reefact.` ;
- un seul paquet publié : `Reefact.UniversalWatcher` ;
- un seul .NET Tool, tous les modules à la compilation ;
- aucun système de plugins à l'exécution.

**Architecture**

- Abstractions / Core / CLI + modules métier, techniques, outputs ;
- trois projets pour GitHub (connexion, métier, adaptateur de lecture) ;
- **le module métier possède sa boucle de lecture** ; Core orchestre le
  transverse ;
- frontière matérialisée par `IWatcherSession` ;
- taxonomie d'erreurs transitoire / définitive portée par les exceptions ;
- Core possède la file d'animations, l'horloge et les priorités ;
- les devices sont des traducteurs sans état ;
- une seule session métier à la fois en V1.

**Modèle**

- `CreateInitial` et `Reconcile` sont les deux seuls points d'entrée ;
- `Reconcile` est pure et sert aussi après une coupure ;
- `ObservedAt` sur le snapshot, garde de monotonie ;
- `RunStatus` et `LastConclusion` séparés ;
- transitions calculées sur `LastConclusion` uniquement ;
- branche de référence configurable, par défaut celle du dépôt ;
- les PR draft sont affichées mais ne contribuent pas à la santé ;
- `WatchHealth` a quatre valeurs, dont `Unknown`.

**Écran**

- un tableau d'état permanent, pas de journal ;
- pas de rapport initial ni de rapport de reprise ;
- colonnes SUBJECT / TITLE / CHECKS / STATE / MERGE / AGE / CHANGED ;
- **ordre stable** : branche de référence, puis PR par numéro décroissant ;
- rendu à rythme propre (~1 s), indépendant des lectures ;
- les PR cassées ne sont jamais masquées ;
- l'information ne repose jamais sur la couleur seule.

**Signal**

- violet pour la sélection, le démarrage et `Unknown` ;
- une seule couleur hors palette ;
- file d'animations, durée minimale 3 s ;
- fusion des signaux identiques ;
- warning 15 s, critical 30 s, rainbow 5 s ;
- wave hors file, prioritaire, tant que la fraîcheur est dégradée ;
- plusieurs devices simultanés, verrou par device via **fichier de verrou** ;
- `uwatch device select` pour revenir sur le choix ;
- `--no-signal` court-circuite la découverte ;
- `Ctrl+C` éteint les devices, second `Ctrl+C` force la sortie.

**Sécurité**

- PAT fine-grained saisi masqué ;
- stockage dans le magasin sécurisé du système, **jamais** de repli en clair ;
- **valider avant de stocker** ;
- validation d'identité à la création, validation réelle à l'ajout de dépôt ;
- avertissement avant expiration du token ;
- token invalide en session = arrêt sans nouvelle tentative, pas wave ;
- 403 désambiguïsé par les en-têtes : quota = transitoire, permission =
  définitif.

**Réglages**

- intervalle 10 s par défaut, plancher 5 s ;
- `BlinkDuration` borné 1 s – 5 min.

## 54. Points ouverts

Aucun point ne bloque le démarrage de l'implémentation. Ceux-ci restent à
confirmer en cours de route :

- **Coût GraphQL réel** de la requête de lecture, à mesurer pour confirmer
  l'intervalle par défaut (section 45.1).
- **`StatusCheckRollup` ou agrégation manuelle** des check runs : le premier est
  plus simple mais laisse moins de contrôle sur le traitement des
  non-concluants (section 15).
- **Identité stable des devices** : deux Luxafor identiques peuvent n'exposer
  aucun numéro de série. Il faudra vérifier ce que le matériel expose réellement
  et, à défaut, assumer un identifiant faillible en l'affichant clairement.
- **Largeur du tableau** et politique exacte de troncature des titres selon la
  largeur du terminal.
- **Auteur et filtrage des PR.** Sur un dépôt d'équipe, la plupart des PR ne
  concernent pas l'utilisateur. Il n'y a ni colonne d'auteur ni filtre. Ce n'est
  pas critique pour le cas d'usage visé (dépôts personnels, peu de PR), mais ce
  sera vraisemblablement la première demande sur un dépôt actif.

### 54.1. Options examinées et écartées

Consignées ici pour éviter qu'elles ne soient reproposées sans le raisonnement.

**Un raccourci `uwatch <alias>` sans le préfixe `repo`.** Tranché : non, la
grammaire `uwatch <module> <target>` reste stricte, sans exception, y compris
pour le module unique de la V1.

> La commande la plus fréquente demande donc plus de frappe que ne le ferait un
> raccourci. C'est un coût assumé plutôt qu'un oubli : un raccourci créerait deux
> façons d'écrire la même commande, et sa résolution deviendrait dépendante du
> nombre de modules installés — silencieusement ambiguë le jour où un second
> module métier apparaîtrait avec un alias identique. La prévisibilité de la
> grammaire prime sur l'économie de frappe.

**Un mode « une seule lecture puis sortie ».** Tentant pour un coup d'œil rapide,
mais contraire au produit : l'outil est conçu pour rester ouvert dans un terminal
secondaire. Ajouter un mode ponctuel invitera ensuite à demander une sortie
machine, puis un mode CI — c'est-à-dire tout ce que le périmètre exclut.

**Garder les devices allumés après la fermeture.** Demande prévisible, mais une
lumière non maintenue devient une information fausse (section 38).

**Un compteur d'instabilité par PR.** Le nombre de fois qu'une PR a cassé et
s'est réparée serait une information utile — mais c'est de l'agrégation
temporelle, donc de l'historique, explicitement hors périmètre (section 2).

**Rejouer l'historique des exécutions entre deux lectures.** Techniquement
faisable — GitHub conserve chaque tentative — mais cela reviendrait à
reconstruire un journal, alors que le produit a délibérément renoncé au journal.

**Deux couleurs hors palette** pour distinguer sélection et démarrage
(section 29.1).

## 55. Critère de réussite V1

```bash
dotnet tool install --global Reefact.UniversalWatcher

uwatch connection add personal --provider github
uwatch repo add Reefact/just-dummies --connection personal
uwatch repo just-dummies
```

Puis :

1. la connexion est validée à la création, le compte est affiché ;
2. l'ajout du dépôt échoue avec un message précis si les permissions manquent ;
3. le device est sélectionné et passe au violet ;
4. le modèle complet est chargé sans produire d'événement ;
5. le tableau s'affiche avec toutes les PR ouvertes, leur état et leurs délais ;
6. le device prend la couleur de santé correcte ;
7. une PR qui casse produit un clignotement orange ;
8. une PR déjà cassée qui échoue encore ne produit **aucun** clignotement ;
9. la branche de référence qui casse produit un clignotement rouge prioritaire ;
10. plusieurs PR qui cassent coup sur coup produisent un seul clignotement
    prolongé ;
11. une release sur un dépôt sain produit un rainbow ; sur un dépôt cassé, rien ;
12. une coupure réseau produit une wave de la dernière couleur connue et un
    bandeau `stale` ;
13. **un dépassement de quota produit un `stale`, pas un arrêt** ;
14. la reprise met le tableau à jour sans fabriquer d'événement ;
15. un token révoqué éteint le device et arrête le processus avec un message
    clair, sans nouvelle tentative ;
16. deux instances sur deux dépôts fonctionnent avec un device chacune, et la
    seconde ne peut pas prendre le device de la première ;
17. les durées affichées continuent de s'incrémenter quand la lecture ne passe
    plus ;
18. l'ordre des lignes du tableau ne change pas d'une lecture à l'autre ;
19. `Ctrl+C` éteint les devices et relâche les verrous ; un second `Ctrl+C` force
    la sortie.

## 56. Le concept en trois lignes

```text
LE TABLEAU     la vérité du dépôt, en continu, en détail
LES DEVICES    « il se passe quelque chose, regarde »
GITHUB         le détail, quand on décide d'aller voir
```

Et la sémantique lumineuse, en trois axes orthogonaux :

```text
COULEUR        ce que je sais
WAVE           ce que je montre n'est pas fiable
CLIGNOTEMENT   quelque chose vient de se dégrader
```

Plus une célébration volontairement non prioritaire.
