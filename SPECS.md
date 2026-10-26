# Mowlana — Spécifications produit et techniques

Statut : spécification initiale, à implémenter. Date : 26 octobre 2026.

## 1. Finalité et promesse

Mowlana est un programme Rust 2024 qui reçoit une codebase locale et produit un
livre mdBook expliquant progressivement son fonctionnement. Chaque page du
parcours principal explique une ou plusieurs unités de code identifiées et situe
ces unités dans un graphe navigable. Les pages d'accueil, index et annexes sont
explicitement distinguées des pages explicatives.

Le lecteur doit pouvoir répondre à trois questions à chaque étape :

- Pourquoi ce mécanisme existe-t-il ?
- À quel code correspond cette explication et sur quelles preuves repose-t-elle ?
- Que comprend-il maintenant, et quelle est la prochaine étape accessible ?

Le parcours doit couvrir toutes les unités de code du périmètre annoncé, tout en
réduisant les prérequis inconnus, les répétitions, les changements de contexte et
la surcharge cognitive. Mowlana recherche un bon parcours selon des critères
explicites ; il ne promet ni optimum global ni pédagogie objectivement parfaite.

Artisan est le premier cas d'usage. Les phases P01–P50 servent de repères narratifs,
sans devenir des règles codées en dur ni limiter Mowlana à ce projet.

## 2. Utilisateurs et périmètre

Utilisateurs principaux : développeur qui découvre un dépôt, mainteneur qui veut
en transmettre les mécanismes et auteur qui révise une documentation exhaustive.
Le profil du lecteur indique ses connaissances, ses objectifs et la langue du
livre. Le profil Artisan initial suppose des connaissances Python/FastAPI et
DevOps, avec des bases de Rust à consolider.

### 2.1 Entrées

- Un répertoire local contenant le code, avec ou sans Git.
- Une configuration TOML facultative pour les valeurs par défaut.
- Des annotations éditoriales pour regrouper des unités, ajouter des prérequis,
  associer des phases et corriger les propositions automatiques.
- Les documents existants sélectionnés, traités comme sources avec provenance.
- Facultativement, un fournisseur de génération de texte configuré explicitement.

### 2.2 Sorties

- Un projet mdBook autonome : `book.toml`, `src/SUMMARY.md`, pages et ressources.
- Les inventaires, graphes, unités, parcours et preuves dans des fichiers JSON.
- Un rapport de couverture et les diagnostics bloquants ou non bloquants.
- Un manifeste de provenance permettant de retrouver l'entrée exacte de l'analyse.
- Facultativement, le rendu HTML validé par `mdbook build`.

### 2.3 Limites

La première version traite les sources Rust, leurs tests et leur configuration.
Tous les fichiers du dépôt sont inventoriés, mais un langage sans analyseur reste
explicitement non analysé. Une codebase mixte ne peut donc pas être déclarée
intégralement expliquée tant que ces sources ne sont pas prises en charge.

Les évolutions prévues couvrent TypeScript/Svelte, Python, Go et Shell, notamment
pour Artisan. Les configurations, SQL et formats embarqués demandent aussi une
explication lorsque leur comportement n'est pas couvert par l'unité englobante.

Mowlana n'exécute pas la codebase analysée, ses tests, ses scripts ni ses fichiers
de build. Il ne modifie pas ses sources, ne publie pas le livre automatiquement et
ne prend pas la couverture de tests pour une preuve de compréhension ou de sûreté.

## 3. Définitions et modèle de données

| Objet | Définition |
| --- | --- |
| Snapshot | Ensemble exact des fichiers sélectionnés et de leurs empreintes lors d'une analyse |
| Élément source | Élément technique extrait : module, type, fonction, méthode, champ, variante ou bloc |
| Unité explicative | Plus petit mécanisme cohérent que le lecteur doit comprendre ; lié à un ou plusieurs éléments ou fragments source |
| Concept | Connaissance préalable sans code propriétaire obligatoire, par exemple ownership ou quorum |
| Preuve | Référence localisée vers du code, un document, un test ou un résultat importé |
| Page explicative | Étape ordonnée qui explique des unités et introduit les concepts nécessaires |
| Phase | Repère de développement fourni par le projet, éventuellement relié à une révision historique |
| Annotation | Décision éditoriale persistante, distincte des résultats générés |

### 3.1 Références source

Une référence contient le chemin relatif normalisé, le nom qualifié lorsque
disponible, le genre d'élément, la plage source et l'empreinte du fragment. Les
plages utilisent des offsets d'octets, avec une fin exclusive ; lignes et colonnes
affichées sont calculées pour le snapshot. Les chemins absolus ne servent pas
d'identifiants persistants.

Les identifiants sont uniques, déterministes et fondés sur le chemin, le nom
qualifié et le genre, avec un discriminant pour les éléments anonymes ou les
implémentations multiples. Une modification de corps conserve l'identité si ces
attributs restent stables. Renommage et déplacement produisent une proposition de
correspondance à valider ; aucune stabilité parfaite n'est supposée.

Chaque unité possède au minimum : identifiant, titre, responsabilité, références
source, éléments possédés, concepts, phases, complexité estimée avec justification,
relations, état de validation et page propriétaire. L'origine automatique ou
éditoriale de chaque propriété est conservée.

### 3.2 Granularité

Une unité correspond à un mécanisme, pas automatiquement à un fichier ou une
fonction. Une grande fonction peut être répartie entre plusieurs unités ; plusieurs
helpers peuvent être regroupés lorsqu'ils expliquent un seul mécanisme. Toute
fusion ou scission conserve la traçabilité et la propriété des fragments.

Un grand élément source est couvert seulement lorsque tous ses fragments
obligatoires sont expliqués. Types, champs et variantes sont expliqués directement
ou au sein de l'unité qui les possède. Imports et syntaxe mécanique peuvent être
rattachés à leur module ; ils ne gonflent pas artificiellement le nombre d'unités.

Une découpe automatique est une proposition révisable. Le fichier entier constitue
une unité de secours pour un élément non compris, marquée incomplète ; cette
solution n'autorise pas une déclaration de couverture complète.

## 4. Inventaire et analyse statique

### 4.1 Inventaire exhaustif

L'inventaire porte sur tous les fichiers ordinaires du répertoire accessible et
attribue une catégorie explicite : source, test, configuration, documentation,
généré, vendorié, binaire, artefact de build, sensible ou exclu par configuration.
Les répertoires techniques ignorés sont enregistrés avec leur motif ; leur contenu
n'est pas assimilé à du code analysé. Une erreur de lecture est un diagnostic,
jamais une exclusion silencieuse.

La sélection ne dépend pas exclusivement de `git ls-files` : les fichiers non
suivis sont pris en compte selon la configuration. Les liens symboliques ne sont
pas suivis par défaut ; les cibles extérieures à la racine restent hors lecture.
Les exclusions apparaissent dans le rapport avec leurs motifs et leurs volumes.

Un fichier généré renvoie à son générateur lorsqu'il est identifié. Le code
vendorié est comptabilisé séparément. Un test fait partie du périmètre explicatif
lorsqu'il est sélectionné ; citer ce test ailleurs ne suffit pas à l'expliquer.

### 4.2 Analyse Rust initiale

L'analyse syntaxique couvre Rust 2024 : modules internes et externes, éléments
publics et privés, `impl`, traits, types, constantes, expressions importantes,
tests et conditions `cfg`. Elle parcourt les fichiers sans compiler le dépôt.

Les branches conditionnelles présentes dans les sources restent dans l'inventaire,
même lorsqu'elles ciblent Linux depuis un hôte macOS. Les conditions sont attachées
aux éléments ; aucune couverture multiplateforme d'exécution n'en est déduite.

Les appels de macros sont recensés, mais leurs expansions ne sont pas prétendues
résolues sans analyse dédiée. Les macros procédurales, appels dynamiques, noms
ambigus et sources générées absentes produisent des relations non résolues avec
un effet explicite sur le niveau de complétude.

Une analyse purement syntaxique ne fournit pas toute la résolution sémantique du
compilateur. Les relations portent un niveau `observed`, `inferred`, `editorial`
ou `unresolved`, avec preuve et origine. Les relations inférées ne sont pas
présentées comme certaines.

## 5. Les trois graphes

### 5.1 Graphe technique

Graphe orienté typé reliant éléments source et unités. Relations minimales :
`contains`, `calls`, `uses_type`, `produces`, `consumes`, `authorizes`, `owns_resource`
et `verified_by`. Les relations non déductibles de la syntaxe sont proposées ou
annotées, jamais fabriquées pour compléter artificiellement le graphe.

Chaque arête conserve sa provenance et son niveau de résolution. L'absence d'arête
n'est pas une preuve d'absence de dépendance. Toutes les composantes déconnectées,
y compris outils, tests et code potentiellement inutilisé, restent à parcourir.

### 5.2 Graphe pédagogique

Graphe reliant unités et concepts par des prérequis d'apprentissage. Ses relations
sont distinctes des appels techniques. Une unité peut être introduite par son
contrat avant que son implémentation soit expliquée.

Deux types de prérequis : `requires` obligatoire et `benefits_from` souhaitable.
Chaque prérequis précise le niveau attendu : concept introduit, contrat compris
ou mécanisme expliqué. Les acquis déclarés du lecteur satisfont les concepts
correspondants, mais ne couvrent aucune unité de code.

Les cycles techniques sont autorisés. Un cycle de prérequis obligatoires doit être
résolu par une introduction de contrat, un regroupement cohérent ou une correction
éditoriale. Le planificateur ne supprime pas une arête obligatoire pour avancer.
Si les budgets rendent le cycle insoluble, il rend un diagnostic localisé.

### 5.3 Graphe des pages

Chaque sommet représente une page. Les arêtes comprennent `next`/`previous` pour
le parcours principal et des renvois vers prérequis, rappels et approfondissements.
La lecture principale suit une séquence finie, sans impasse, visitant toutes les
unités sélectionnées au niveau expliqué. Les retours sont autorisés.

Une transition entre pages peut introduire un nouveau contexte sans suivre un
appel technique. Cette transition est explicitée. Mowlana ne cherche donc pas
uniquement une marche minimale dans le graphe d'appels : il résout conjointement
la couverture des unités, leur regroupement en pages et leur ordre pédagogique.

## 6. Planification du parcours

### 6.1 Contraintes obligatoires

- Chaque unité requise possède exactement une page propriétaire de son explication.
- Une page propriétaire contient une explication, pas uniquement un lien ou un nom.
- Chaque prérequis obligatoire est satisfait avant son utilisation, éventuellement
  par une section précédente de la même page, dont l'ordre est vérifiable.
- Chaque page explicative couvre au moins une unité de code et respecte les budgets
  configurés d'unités, de complexité estimée, de nouveaux concepts et de longueur.
- Toute unité hors du parcours est signalée avec son motif ; un oubli bloque la
  complétude, même lorsqu'il appartient à une composante déconnectée.
- Les rappels ne comptent pas comme nouvelles unités couvertes.

### 6.2 Critère d'optimisation

Les solutions qui violent une contrainte obligatoire sont rejetées. Parmi les
solutions admissibles, minimiser un score configurable composé de :

```text
coût = poids_contexte × changements_de_contexte
     + poids_prérequis × prérequis_souhaitables_différés
     + poids_répétition × explications_redondantes
     + poids_charge × déséquilibre_de_charge_entre_pages
     + poids_phases × écarts_à_la_progression_narrative
     + poids_longueur × nombre_de_pages
```

Les composants sont normalisés, documentés et exportés avec leurs poids. La charge
cognitive est une estimation, pas une mesure scientifique. Réduire le nombre de
pages reste secondaire aux contraintes de lisibilité. Une transition vers un autre
crate ou domaine n'est pas interdite, mais doit servir la progression.

### 6.3 Algorithme initial

1. Valider l'inventaire, la propriété des éléments et les annotations.
2. Détecter les cycles de prérequis obligatoires et proposer leur résolution.
3. Construire la liste des unités accessibles depuis les acquis du lecteur.
4. Choisir l'unité apportant le meilleur compromis entre couverture nouvelle,
   cohérence de contexte, progression narrative et coût de concepts.
5. Regrouper les unités compatibles dans une page selon les budgets.
6. Répéter jusqu'à épuisement, ou produire un diagnostic de blocage.
7. Améliorer localement l'ordre et les regroupements en préservant les contraintes.
8. Revalider la couverture complète et exporter le score obtenu.

Les égalités sont départagées par identifiant stable. À configuration et données
identiques, le planificateur produit le même parcours. Les limites d'itérations et
les motifs d'arrêt sont enregistrés. Une phase tronquée ne vaut pas couverture.

Le résultat est comparé à des parcours de référence simples, par exemple ordre des
modules ou phases puis dépendances. Un solveur exact reste une expérimentation
sur de petits graphes ; il n'est pas une dépendance obligatoire du produit.

## 7. Phases, histoire et documentation existante

Les phases sont importées depuis une configuration ou des documents explicitement
sélectionnés. Elles forment une relation plusieurs-à-plusieurs avec les unités :
une phase traverse plusieurs modules et un module peut contribuer à plusieurs phases.

Mode initial : expliquer le snapshot courant en suivant la progression des phases.
Un numéro de phase n'établit pas la date de création d'une fonction. Toute affirmation
historique nécessite une preuve attachée à une révision ou un diff identifié.

Un futur mode historique peut suivre des révisions successives. Il doit distinguer
le code ancien du code courant, y compris déplacements et suppressions. Git est
consulté en lecture seule ; une base sans historique reste une entrée valide.

Pour Artisan, les contrats restent dans PLAN, les formats livrés dans IMPLEMENTATION,
les frontières dans ARCHITECTURE et les preuves dans QUALIFICATION. Le livre explique
ces sources avec des renvois ; il ne devient pas une seconde autorité concurrente.
Une cible future n'est jamais présentée comme un comportement livré.

## 8. Contrat d'une page explicative

Chaque page contient, dans un ordre adapté au sujet :

1. La question concrète résolue et la raison d'être du mécanisme.
2. Les acquis supposés, concepts introduits et objectifs de compréhension.
3. Les unités expliquées, leurs identifiants et les liens vers leurs sources.
4. La position dans l'architecture et les flux entrants et sortants.
5. Le mécanisme détaillé, appuyé sur des extraits courts et exacts.
6. Les invariants, ressources, erreurs et conditions de reprise applicables.
7. Les tests ou preuves disponibles et ce qu'ils ne démontrent pas.
8. Les unités acquises, la progression cumulée et la transition vers la suite.

Les rubriques sans objet sont omises avec une justification dans les métadonnées.
Une analogie accompagne le raisonnement lorsqu'elle aide le profil du lecteur ;
ses limites sont indiquées. La prose explique les mécanismes et décisions plutôt
que de paraphraser chaque ligne.

Un bloc de code n'est pas une explication. Une explication propriétaire doit décrire
au moins le rôle de l'unité, son mécanisme et ses liens avec les autres unités.
Pour une unité métier ou système, elle expose aussi les invariants et échecs utiles.

Le livre ne dépend pas du dépôt original pour ses liens internes. Les extraits et
vues source autorisés sont générés depuis le snapshot et embarqués localement,
avec numéros de ligne et empreintes. Les liens Git sont optionnels, épinglés à une
révision ; pour un arbre modifié, les vues locales sont la référence exacte.

## 9. Production des explications

### 9.1 Séparer faits et propositions

L'extraction, les identifiants, la couverture et les liens sont calculés par le
programme. Un modèle de langage peut proposer la découpe pédagogique, les concepts,
les prérequis et la prose, mais ne décide pas seul de la vérité des références.

Chaque section rattache ses affirmations techniques à des preuves localisées.
Les assertions non étayées sont marquées à revoir. La présence d'un test prouve
l'existence de ce test ; seul un résultat importé avec son environnement, sa date
et son snapshot permet d'affirmer qu'il a été exécuté avec succès.

### 9.2 Fournisseur de génération

La génération complète exige un fournisseur configuré, local ou distant, ou un
contenu éditorial importé. Sans fournisseur, l'inventaire et le parcours restent
utilisables ; un export de brouillon est possible, mais n'est pas présenté comme
un livre intégralement expliqué.

Les demandes incluent seulement le contexte utile : unités, sources sélectionnées,
interfaces voisines, acquis du lecteur et références nécessaires. Le fournisseur
retourne un contenu structuré avec sections, identifiants et preuves. Mowlana
vérifie les références, les extraits et la conformité avant de publier la page.

Les budgets de requêtes, de tokens lorsque disponibles, de coût et de reprises sont
bornés et configurables. Un dépassement interrompt la génération proprement et
laisse un état reprenable. Les erreurs transitoires ont des reprises limitées ;
une erreur de configuration ou de contenu invalide n'est pas rejouée indéfiniment.

Les empreintes du contexte, la version des instructions, le modèle, les paramètres
et les réponses retenues sont enregistrés sans credentials. Le réemploi d'une
réponse validée est déterministe ; une nouvelle génération n'est pas garantie
identique, même avec les mêmes paramètres.

### 9.3 Revue éditoriale

États du contenu : `draft`, `validated`, `reviewed`, `stale`. `validated` signifie
que les contrôles structurels ont réussi ; `reviewed` représente une revue humaine
explicitement enregistrée. Aucun de ces états ne découle du seul avis du modèle.

Les corrections éditoriales sont stockées séparément du texte régénéré. Un changement
amont crée une proposition de mise à jour ou un conflit ; il n'écrase pas une correction.

## 10. Couverture et conditions de complétude

### 10.1 Indicateurs distincts

| Indicateur | Calcul et sens |
| --- | --- |
| Classification | Fichiers classifiés / fichiers inventoriés ; périmètre et dossiers ignorés publiés |
| Analyse | Éléments analysés et fichiers sources non analysés, ventilés par langage |
| Rattachement | Éléments source obligatoires possédés par des unités / éléments obligatoires extraits |
| Parcours | Unités requises présentes sur une page propriétaire / unités requises |
| Explication | Unités avec contenu propriétaire validé et frais / unités requises |
| Revue | Unités avec contenu humainement revu et frais / unités requises |
| Fraîcheur | Pages dont les sources et dépendances vérifiées correspondent au snapshot |

Les totaux par fichier, élément et unité sont publiés ensemble : modifier la
granularité ne doit pas masquer une régression. Les lignes ou octets expliqués
peuvent compléter le rapport, mais ne constituent pas seuls une mesure de qualité.
Un périmètre sans unité donne « non applicable », jamais un 100 % artificiel.

### 10.2 « Couverture complète du périmètre »

Cette mention exige une classification exhaustive, aucune source sélectionnée non
analysée, aucun élément obligatoire orphelin, toutes les unités expliquées et fraîches,
des prérequis satisfaits et aucun diagnostic bloquant. Les exclusions demeurent
visibles à côté du résultat. Une couverture d'un sous-périmètre Rust ne vaut pas
couverture intégrale d'un dépôt multilangage.

Les niveaux « structure vérifiée » et « revue humaine complète » sont annoncés
séparément. Mowlana ne certifie pas automatiquement la justesse pédagogique du livre.
L'expansion non résolue de macros ou un comportement généré inconnu limite le
périmètre certifiable et doit être signalé dans la déclaration de complétude.

### 10.3 États d'apprentissage

Une référence à une unité lui donne le niveau `mentioned`, un contrat présenté lui
donne `introduced`, une explication propriétaire conforme lui donne `explained`.
Seul `explained` contribue à la couverture explicative. La progression affichée
correspond au parcours prévu ; elle ne prétend pas mesurer la compréhension réelle
du lecteur. Un éventuel suivi local de lecture reste une préférence du navigateur.

## 11. Interface de commande cible

Les commandes suivantes définissent le contrat souhaité ; elles ne sont pas encore
implémentées. `--source` et `--output` désignent des chemins ; `--config` permet de
choisir une configuration. Aucun accès distant n'est activé implicitement.

```sh
# Pipeline complet ; fournisseur de texte défini dans la configuration.
mowlana generate --source ../artisan --output ../artisan-book --config mowlana.toml

# Étapes séparées, toutes reprenables.
mowlana scan --source ../artisan --output ../artisan-book
mowlana plan --output ../artisan-book --config mowlana.toml
mowlana write --output ../artisan-book --config mowlana.toml
mowlana verify --output ../artisan-book --require-complete
mowlana build --output ../artisan-book

# Recalcul après modification des sources.
mowlana update --source ../artisan --output ../artisan-book --config mowlana.toml
```

`generate` enchaîne inventaire, graphes, planification, rédaction, vérification et
export mdBook. Il n'exécute le rendu HTML que si `--build` est demandé. Sans
contenu éditorial complet ni fournisseur, il échoue explicitement ; `--draft`
autorise un export incomplet signalé comme tel.

`verify` n'utilise pas de modèle et ne modifie pas le contenu. `build` revérifie le
projet exporté puis appelle le binaire mdBook configuré, enregistre sa version et
propage son échec. Un build réussi ne remplace pas `verify --require-complete`.

Codes de sortie : `0` succès de la commande, `1` erreur d'exécution, `2` entrée ou
configuration invalide, `3` vérification/complétude demandée non satisfaite. Le rapport
distingue explicitement succès d'export d'un brouillon et complétude du livre.

### 11.1 Configuration minimale indicative

```toml
schema_version = 1

[book]
title = "Comprendre Artisan"
language = "fr"

[reader]
known_concepts = ["python", "fastapi", "linux-basics", "devops"]

[scope]
include = ["crates/**", "tests/**", "scripts/**", "dashboard/**", "Cargo.toml"]
exclude = ["target/**", "dashboard/dist/**"]

[planning]
max_units_per_page = 5
max_new_concepts_per_page = 3
max_page_words = 1800

[generation]
provider = "local"
network = false

[verification]
require_complete = true
require_human_review = false
```

Ces valeurs sont des points de départ à calibrer, pas des seuils pédagogiques
universels. Les chemins sont relatifs à la racine source pour `scope`, au fichier
de configuration pour les autres chemins configurés. Les clés inconnues, poids
invalides et budgets nuls sont refusés. Le rapport montre la configuration résolue,
les exclusions effectives et les langages encore non pris en charge.

Les annotations couvrent au minimum : fusions/scissions, titres, prérequis et
leur niveau, phases, épinglage relatif de pages et correspondances après renommage.
Une annotation visant un identifiant absent produit un diagnostic explicite.

## 12. Stockage et export mdBook

```text
artisan-book/
  book.toml
  src/
    SUMMARY.md
    index.md
    chapters/
    concepts/
    reference/             # Vues source autorisées, index unités/fichiers/phases.
    coverage.md
    assets/                # Graphe et navigation, ressources locales.
  .mowlana/
    manifest.json
    inventory.json
    elements.json
    units.json
    technical-graph.json
    pedagogical-graph.json
    route.json
    pages.json
    coverage.json
    diagnostics.json
    annotations.json
    generation/            # Réponses et provenance ; non exportées dans le livre.
  book/                    # HTML lorsque le rendu est demandé.
```

Les schémas JSON sont versionnés. Les références entre fichiers sont validées.
Le manifeste enregistre versions, empreintes d'entrée et configuration, révision
Git éventuelle, modifications locales et paramètres d'analyse. Les fichiers dont
le contenu change pendant la lecture sont relus ou signalés ; un snapshot incohérent
n'est pas exporté comme référence valide.

Le mdBook exporté contient un sommaire strictement valide et `create-missing = false`.
Le build doit échouer sur une page manquante plutôt que la créer silencieusement.
La conformité suit les références officielles
[SUMMARY.md](https://rust-lang.github.io/mdBook/format/summary.html) et
[book.toml](https://rust-lang.github.io/mdBook/format/configuration/general.html).

Le livre reste lisible sans JavaScript : ordre, liens, unités et progression sont
présents en Markdown. La navigation enrichie affiche un sous-graphe local, les
unités déjà abordées et la page suivante ; un index global permet de rechercher
un fichier, symbole, concept ou une phase. Les cycles et grandes cartes n'imposent
pas l'affichage permanent de tout le graphe.

Aucun CDN ni service distant n'est nécessaire à la consultation. Les sources
confidentielles autorisées ne sont pas ajoutées au rendu par accident : seules les
vues et extraits explicitement sélectionnés sont copiés depuis le snapshot.

## 13. Mise à jour et reproductibilité

Une mise à jour compare les snapshots, recalcule les éléments modifiés et invalide
les unités, relations et pages qui en dépendent. Une modification d'interface peut
invalider les pages consommatrices même lorsque leur propre extrait est inchangé.
En cas d'incertitude, l'invalidation est conservatrice et son motif est visible.

Les pages revues non affectées et les identifiants stables sont conservés. Un
changement de parcours respecte les dépendances et signale les modifications de
progression. Les disparitions, renommages et conflits éditoriaux sont présentés
avant de perdre leurs associations. Les anciennes pages périmées sont signalées
et ne contribuent plus à la couverture fraîche.

Inventaire, graphes et parcours sont reproductibles pour une même entrée, une
même configuration et une même version d'analyse. Les horodatages d'exécution sont
isolés de ces données. Le texte l'est à partir des réponses retenues et de leurs
empreintes, sans promesse de reproduire une nouvelle inférence distante.

Les écritures passent par des fichiers temporaires et une validation avant
remplacement. Une interruption laisse l'état antérieur utilisable ou un état
incomplet explicitement détectable ; aucun succès partiel n'est présenté comme une
publication complète. Une seule écriture simultanée est admise par répertoire de
sortie. Aucun fichier éditorial ou étranger n'est écrasé silencieusement.

## 14. Exécution, confidentialité et erreurs

La sortie doit être distincte de l'entrée et ne peut ni la contenir ni être incluse
dedans. Les chemins sont normalisés avant écriture. Le dépôt analysé reste en
lecture seule, y compris ses métadonnées Git. Une commande du projet analysé ou un
contenu documentaire ne devient jamais une instruction à exécuter par Mowlana.

Le réseau est désactivé par défaut. Un fournisseur distant exige une configuration
explicite de l'endpoint et du périmètre transmissible. Les credentials proviennent
de l'environnement ou d'un mécanisme local et n'entrent ni dans les journaux ni dans
les artefacts. Les fichiers sensibles connus sont exclus par défaut ; les filtres
ne sont pas présentés comme une garantie de détection de tous les secrets.

Les extraits et HTML sont échappés selon leur contexte. Les données JSON destinées
au navigateur ne permettent pas l'injection de script. Le livre n'exécute aucun
contenu provenant des commentaires du dépôt.

Les erreurs sont typées et contextualisées : fichier illisible, syntaxe non prise
en charge, référence absente, prérequis cyclique, génération invalide, budget épuisé,
sortie en conflit ou build mdBook échoué. Les journaux `tracing` indiquent étape,
identifiant et motif sans enregistrer les sources complètes par défaut. L'arrêt
utilisateur interrompt proprement les travaux et permet une reprise.

## 15. Architecture d'implémentation proposée

Commencer par un seul package Rust, avec bibliothèque et CLI fines. Les modules
suivent les étapes du pipeline : `inventory`, `analysis`, `units`, `graph`, `planning`,
`writing`, `verification`, `export` et `state`. Le modèle sérialisé commun reste
simple ; pas de service distribué, de base graphe ni de plugins dynamiques requis.

Choix initiaux à confirmer lors de l'implémentation : analyse syntaxique Rust avec
un parseur dédié compatible Rust 2024, Serde pour les contrats, TOML pour la
configuration, Clap pour la CLI, thiserror pour les erreurs et tracing pour les
journaux. Des listes d'adjacence et algorithmes explicites suffisent au premier
planificateur. Une dépendance graphe ne se justifie que par un besoin concret.

L'interface fournisseur de texte et les futurs analyseurs de langages constituent
les points d'extension identifiés. Ils ne nécessitent pas un framework générique
sur tout le pipeline. Les ressources de rendu sont distribuées localement.

L'inventaire et la validation ne chargent pas simultanément toutes les sources ou
réponses en mémoire. Le traitement d'un fichier ne déclenche pas de lecture réseau.
Les limites de taille et budgets sont configurables et tout dépassement est visible.
Le coût mémoire de base doit dépendre surtout des éléments et relations extraits.

## 16. Critères d'acceptation

| ID | Scénario | Résultat attendu |
| --- | --- | --- |
| A01 | Petit dépôt Rust, sans Git | Inventaire, graphe, parcours et mdBook générés avec provenance complète |
| A02 | Module privé et branche `cfg` inactive sur l'hôte | Tous deux présents dans l'inventaire et le périmètre explicatif |
| A03 | Unité jamais atteinte par les appels principaux | Incluse dans le parcours ou couverture explicitement incomplète |
| A04 | Prérequis obligatoires cycliques | Diagnostic des unités et arêtes ; aucune suppression silencieuse |
| A05 | Une page ne fait que citer une unité | L'unité n'atteint pas `explained` ; vérification complète en échec |
| A06 | Fonction scindée en trois unités, une oubliée | Élément source et livre déclarés incomplets |
| A07 | Fonction renommée ou supprimée | Références obsolètes signalées et annotations préservées pour réconciliation |
| A08 | Interface amont modifiée | Pages dépendantes marquées périmées avec justification |
| A09 | Dépôt Rust, Python et Svelte avec seul analyseur Rust | Langages manquants visibles ; aucune couverture intégrale annoncée |
| A10 | Génération interrompue ou budget épuisé | Reprise possible sans perdre les pages validées ni dépasser le budget |
| A11 | Même snapshot, configuration et réponses retenues | Inventaire, parcours et contenu stables hors métadonnées d'exécution |
| A12 | Test source présent sans résultat d'exécution | Le livre ne prétend pas que le test a réussi |
| A13 | Consultation sans réseau ni JavaScript | Texte, références, progression et navigation séquentielle utilisables |
| A14 | Page référencée dans SUMMARY mais absente | Vérification et build en échec explicite |
| A15 | Macro non résolue ou fichier illisible | Diagnostic localisé et niveau de complétude dégradé |
| A16 | Livre généré contenant du texte injectant HTML/script | Contenu affiché comme données, sans exécution du script |
| A17 | Sortie dans l'entrée, ou lien vers un chemin extérieur | Opération refusée sans modification du dépôt |
| A18 | Livre complet selon le périmètre déclaré | Toutes les contraintes vérifiées, rapport explicite, build mdBook réussi |

Tests du planificateur : chaque unité obligatoire possède un propriétaire, tous les
prérequis sont satisfaits dans l'ordre et chaque page respecte ses budgets. Tester
des graphes disjoints, cycles, fusions, scissions et égalités de score, pas seulement
un exemple linéaire. Les fournisseurs sont bouchonnés dans les tests ordinaires.

La revue pédagogique du pilote mesure la capacité d'un lecteur cible à suivre un
flux, expliquer un invariant et retrouver le code sans assistance. Les corrections
et difficultés observées servent à ajuster les budgets ; un score du planificateur
ne remplace pas cet essai.

## 17. Jalons de réalisation

1. **Socle traçable** : inventaire, snapshot, extraction Rust, identifiants, unités
   éditables et vérification des références. Aucun texte complet promis à ce stade.
2. **Parcours vérifiable** : graphes séparés, annotations, contraintes, heuristique,
   diagnostics de cycles et rapport de couverture structurelle.
3. **Livre explicatif minimal** : fournisseur de texte, preuves, contrôles des pages,
   export mdBook, index des sources et pilote Artisan autour de P01 et ses dépendances.
4. **Maintenance** : invalidation transitive, conservation des corrections, reprise,
   budgets et navigation locale du graphe.
5. **Artisan complet** : analyseurs supplémentaires, explication des tests/scripts,
   frontend/provider/configurations et couverture exhaustive du périmètre déclaré.
6. **Améliorations mesurées** : comparaison de parcours, calibrage par lecteurs et
   exploration optionnelle de l'histoire Git.

Chaque jalon doit produire un artefact utilisable et des preuves ciblées. Un pilote
P01 réussi ne clôture pas la couverture d'Artisan. La durée d'analyse, mémoire,
taille du livre, coût de génération et taux de reprise sont mesurés sur des entrées
de référence avant de fixer des objectifs chiffrés de performance.

## 18. Décisions retenues et questions à calibrer

Décisions : CLI Rust 2024, mdBook autonome, entrée locale en lecture seule,
inventaire exhaustif, distinction des trois graphes, couverture calculée hors modèle,
annotations persistantes, génération configurable et vérification indépendante.

À calibrer sur le pilote : granularité des unités, poids du score, longueur des
pages, nombre de nouveaux concepts et niveau de détail des vues source. À choisir
lors de l'implémentation : parseur Rust précis, version mdBook prise en charge et
premier fournisseur de texte. Ces choix seront enregistrés et testés ; aucune
dépendance distante particulière ni version de modèle n'est imposée par cette spec.
