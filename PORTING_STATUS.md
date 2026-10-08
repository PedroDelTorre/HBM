# État du portage HBM Nuclear Tech → moderne

Dernière mise à jour : 2026-10-08 (satellites 13/13; Plasma Forge à 33 recettes dont Fensusan et Gerald; enceinte ICF CE assemblable; les cinq blocs DFC ont une première mécanique; build Forge vérifié; essais en jeu restant à faire; Phase 5 en cours).

## Avancement global

**Estimation de travail : environ 74 %** (fourchette prudente : **66–83 %**, confiance faible à
moyenne). Ce chiffre résume la couverture des grands systèmes, leur fonctionnement réel et la
validation disponible. Il ne correspond ni au nombre de fichiers/objets enregistrés ni aux ≈56 %
de recettes estimées, qui ne mesurent que la couverture des recettes. L'inventaire historique et
les vérifications en jeu restent incomplets. Cette estimation sera réévaluée à chaque tranche
terminée.

## Où on en est

- **Phase 1-4** : terminées. Voir [docs/porting/RESEARCH.md](docs/porting/RESEARCH.md),
  [BASE_DECISION.md](docs/porting/BASE_DECISION.md), [CONTENT_MATRIX.md](docs/porting/CONTENT_MATRIX.md),
  [ROADMAP.md](docs/porting/ROADMAP.md).
- **Toolchain installée et le projet compile réellement** : JDK 17 (Temurin, portable, sans
  élévation admin) et JDK 21 installés sous `C:/Users/steep/jdks/`, enregistrés auprès de Gradle
  (`~/.gradle/gradle.properties`). `./gradlew :1.20.1-forge:build` → **BUILD SUCCESSFUL** (1m 24s,
  5453 fichiers source compilés, jar produit :
  `versions/1.20.1-forge/build/libs/hbm_m-0.2.1-alpha+1.20.1-forge.jar`). Dernière validation :
  `runData` réussi après la recette CE de lentille AMS; `compileJava` et `jar` réussis après le
  portage des cinq machines DFC. `test` est
  `NO-SOURCE` (aucun test n'est déclaré dans ce dépôt). Javadoc signale
  67 références/tags cassés et 100 avertissements, non bloquants par configuration
  (`isFailOnError = false`) ; ce ne sont pas des erreurs de compilation Java.
  **C'est la première compilation réelle de ce projet dans notre session — on dépasse déjà le
  stade où l'tentative `scarrymany` s'était arrêtée (jamais compilé).**
- **Phase 5 (portage réel de contenu)** : en cours. PWR, Research Reactor, son réacteur de
  reproduction, RadGen et plusieurs satellites sont intégrés ; voir
  [docs/porting/ROADMAP.md](docs/porting/ROADMAP.md) pour les prochains systèmes.

## Fonctionnalités portées

- **Crucible — recettes de base HBM rétablies.** Les 13 recettes par défaut de
  `CrucibleRecipes.registerDefaults()` sont maintenant actives : steel, hematite, malachite,
  redcopper, HSS, ferrouranium, tcalloy, cdalloy, bismuth bronze, arsenic bronze, combine steel,
  magnetized tungsten et BSCCO. Les recettes de réduction des minerais consomment le flux et
  produisent le laitier comme dans la référence; les recettes de bronze ont aussi retrouvé leur
  flux et leur sous-produit. Les items manquants dans l'ancien audit existaient déjà dans le
  registre générique `ModIngots`; le seul nouvel item est le lingot de laitier, dont le modèle
  réutilise la texture du bloc de laitier. Les recettes facultatives dépendant de GregTech 6
  restent hors du jeu de base.
  - **Sources historiques** : `CrucibleRecipes.java` de HBM 1.7.10; montants, fréquences et
    entrées/sorties comparés à la référence avant portage. Le détail du pont entre items et
    matériaux fondus est dans `PORTING_NOTES.md`.
  - **Validation** : `:1.20.1-forge:runData`, `:1.20.1-forge:compileJava` et `:1.20.1-forge:jar`
    terminent avec succès. Le modèle `slag_ingot` et les tags `ingots/slag` et `ingots/u238` ont
    été vérifiés dans la sortie. Le test en jeu reste à faire.

- **Satellite Resonator (`sat_resonator`, "Xenium Resonator")** — port de
  `com.hbm.saveddata.satellites.SatelliteResonator` (CE/1.7.10) :
  [SatelliteResonator.java](src/main/java/com/hbm_m/satellite/SatelliteResonator.java). Relais
  orbital réutilisable : une fois en orbite (lancé via le Soyuz Launcher comme Gerald), un
  `ItemSatDesignator` réglé sur la même fréquence téléporte son porteur aux coordonnées visées
  (Y auto-détecté à la surface si la cible est le ciel ouvert, comme en 1.7.10). Contrairement à
  Gerald (`SatelliteHorizons`), pas de drapeau « usage unique » — reproductible à volonté, comme
  l'original.
  - Item `SAT_RESONATOR` passé de `Item` générique à `ItemSatChip` (nécessaire pour porter une
    fréquence réglable par sneak+clic-droit — le mécanisme existait déjà pour Gerald, juste pas
    branché sur cet item).
  - Enregistré dans `Satellite.register()`, actif sur toutes les cibles (Forge/NeoForge/Fabric —
    code commun, aucune API spécifique à un loader).
  - Traduction anglaise ajoutée (`item.hbm_m.sat_resonator`) dans le fichier Crowdin-suivi
    `src/generated/resources/assets/hbm_m/lang/en_us.json` (pas de regénération — édition
    manuelle minimale pour ne pas régresser le reste du fichier, voir PORTING_NOTES.md).
  - **Compile-vérifié** : `./gradlew :1.20.1-forge:compileJava` → BUILD SUCCESSFUL.
  - **Non vérifié en jeu** (pas de `runClient` dans cette session) : le mapping exact
    `ENDERMAN_TELEPORT` + `stopRiding()` + `teleportTo()` n'a pas été testé avec un vrai joueur en
    train de monter une entité ou de traverser une dimension. Comportement attendu correct par
    lecture de l'API, pas observé.
  - **Vérifié sur Forge 1.20.1 seulement** — voir la correction majeure ci-dessous : la cible
    NeoForge 1.21.1 ne compile pas du tout actuellement, indépendamment de ce changement.
  - **Les 13 classes satellites CE sont maintenant actives**, avec quelques limites d’interface
    consignées ci-dessous. La gestion des payloads répétés par fréquence permet aussi d’ajouter les
    capteurs, assembleurs et disques de Science après la mise en orbite.

- **Satellite Precision Laser (`sat_precision_laser`) — frappe orbitale ciblée.** La puce à
  fréquence est fabriquée à l'Arc Welder avec la tête laser, la base et l'interface satellite.
  L'action du désignateur reproduit le projectile CE via VNT : rayon d'explosion 5, dégâts laser
  croisés fixes jusqu'à 1 000, perforation d'armure 50 DT / 50 % DR, destruction des blocs et
  délai de recharge sauvegardé de cinq secondes. La cible d'entité du panneau `SAT_COORD` et son
  rendu de rayon ne sont pas portés; la cible moderne reste le bloc visé par le désignateur.
  `runData`, `compileJava` et `jar` Forge 1.20.1 réussis; le JAR contient la classe, le type de
  dégâts, le modèle et la texture. Essai en jeu restant à faire.

- **Satellite Miner (`sat_miner`) et Lunar Miner (`sat_lunar_miner`).** Les deux puces règlent
  leur fréquence, se lancent par Soyouz et progressent en orbite sans charger de chunk. Elles
  reprennent la durée CE de 15 minutes et leurs pools pondérés de 10 à 15 piles; le butin et le
  progrès sont sauvegardés dans les données du satellite. Le désignateur récupère les objets dans
  l’inventaire du joueur; si celui-ci est plein, le reliquat reste en orbite. Le Lunar Miner ajoute
  le régolithe lunaire, la poudre de lithium et les cristaux de fer/lithium. Les recettes de
  l’Assembler reprennent les composants CE existants; le réservoir de kérosène CE est représenté
  par du carburant de fusée moderne. `powder_power`, absent du registre moderne, a été ajouté avec
  son sprite CE. `runData`, `compileJava` et `jar` Forge 1.20.1 réussis; essai en jeu restant à faire.

- **Satellite RayScan (`sat_ray_scan`) — détecteur de sources à bande étroite.** La puce est
  assemblée à l’Assembler (1 200 ticks, 25 000 HE) avec la recette CE adaptée aux pièces modernes.
  Le désignateur rapporte les signaux `NEUTRON_EMISSION`, `HIGH_ENERGY_PARTICLES` et
  `RADAR_WAVES` vus dans un rayon horizontal de 250 blocs, en montrant jusqu’à huit contacts.
  Les événements PWR/RBMK, ICF et radar expirent après 200–300 ticks et sont fusionnés par source.
  L’interface `SAT_PANEL` est remplacée par un rapport textuel, aucun chunk distant n’est chargé.
  `runData`, `compileJava` et `jar` Forge 1.20.1 réussis; essai en jeu restant à faire.

- **Satellite Detector (`sat_detector`) — détecteur large bande des impulsions radio.** Sa puce
  réglable se fabrique à l’Assembler en 1 200 ticks / 25 000 HE avec la recette CE adaptée aux
  composants modernes. Le désignateur recense les impulsions encore actives dans la dimension,
  affiche leur intensité et jusqu’à huit coordonnées; l’incertitude de position dépend de
  l’intensité, comme dans CE. Les explosions MK5 et Fleija émettent des impulsions fortes, la mine
  nucléaire ajoute l’impulsion faible CE, et le radar produit une impulsion moyenne. Le registre
  est limité à 4 096 entrées et vidé à l’arrêt du serveur. `runData`, `compileJava` et `jar` Forge
  1.20.1 réussis; l’essai en jeu reste à faire.

- **Satellite Relay (`sat_relay`) et RTTY — relais radio interdimensionnel.** La puce règle la
  fréquence avec sneak+clic droit et parcourt les dimensions serveur chargées par clic droit avant
  le lancement; ce réglage est transféré et sauvegardé en orbite. Le désignateur envoie un rapport
  de position sur le canal égal à la fréquence du satellite. Le `RTTY_PAGER` moderne peut lire ce
  même canal et se synchronise avec une puce au Sat Linker. `RttyMessageBus` garde les canaux par
  dimension, publie au tick suivant et expire les messages après cinq secondes; aucun chunk distant
  n’est chargé. La recette Assembler (600 ticks, 100 HE) suit les composants CE avec un substitut
  moderne à la bouteille d’hydrogène. `runData`, `compileJava` et `jar` Forge réussis; essai en jeu
  restant à faire.

- **Satellite Science (`sat_science`) — laboratoire orbital, capteurs et assembleur 0G.** Le
  laboratoire accepte par Soyouz les modules sensor/assembler, les disques vierges et les entrées
  orbitales réglés à sa fréquence. Il transforme un disque vierge en données de vol toutes les
  15 minutes; chaque capteur produit des données orbitales après 100 heures. L’assembleur traite
  les assemblages orbitaux en circuits cristallins en trois minutes. Les modules, compteurs,
  progression, file de production et cargaison sont sauvegardés dans le satellite; le désignateur
  récupère les résultats sans perte si l’inventaire est plein. Les recettes Assembler et sprites CE
  ont été ajoutés, avec adaptation de composants absents. `runData`, `compileJava` et `jar` Forge
  réussis; essai en jeu restant à faire.

- **Satellite Laser (`sat_laser`) — Death Ray orbital.** La puce se règle en fréquence, se
  fabrique avec la tête et la base à l’Arc Welder, puis se lance avec Soyouz. Le désignateur cible
  la surface et déclenche le MK5 existant (force CE 40), avec un délai de cinq minutes sauvegardé
  dans les données orbitales. Les retombées sont désactivées, mais les dégâts, la destruction de
  terrain et l’irradiation MK5 sont conservés. Le clic de carte SAT_PANEL est remplacé par l’action
  du désignateur; la liste de dimensions paisibles CE n’existe pas encore en configuration moderne.
  `runData`, `compileJava` et `jar` Forge 1.20.1 réussis; essai en jeu restant à faire.
- **Satellite Radar (`sat_radar`) — contacts orbitaux à longue portée.** La puce est maintenant
  réglable en fréquence et déployable par Soyouz. Le désignateur demande les joueurs actifs et les
  missiles du registre radar global dans un rayon horizontal de 1 000 blocs; la réponse donne leur
  position et le nombre total, avec huit fiches de contact au maximum. Le SAT_PANEL, filtres et
  types personnalisés en dehors des missiles radar ne sont pas repris. Pas de chargement de chunk.
  `runData`, compilation et JAR Forge 1.20.1 réussis; essai en jeu restant à faire.
- **Satellite Scanner (`sat_scanner`) — relevé orbital des minerais.** La tête `sat_head_scanner`
  et la base s’assemblent à l’Arc Welder en puce réglable en fréquence. Lancé par Soyouz, le
  Scanner compte les blocs de minerai vanilla (charbon, fer, cuivre, or, redstone, lapis, diamant,
  émeraude) sur toute la hauteur du chunk ciblé, uniquement s’il est déjà chargé. Le balayage ne
  charge pas les chunks à distance. Il remplace l’action `HAS_ORES` du panneau orbital CE par un
  rapport textuel à la demande (item `satellite_scanner`; `sat_scanner` reste le bloc décoratif); le SAT_PANEL et les minerais HBM spécifiques ne sont pas portés.
  `runData`, `compileJava` et `jar` Forge 1.20.1 réussis; essai en jeu restant à faire.
- **Satellite Mapper (`sat_mapper`) — puce et action orbitale fonctionnelles.** Le satellite déjà
  fabriqué à l’Assembler porte maintenant une fréquence, part avec Soyouz et répond au
  `ItemSatDesignator` correspondant. Son rapport affiche le statut de chargement du chunk ciblé,
  les coordonnées/hauteur et les joueurs à moins de 250 blocs, selon le comportement Mapper CE.
  L’interface graphique/panneau de cartes, la mesure de pollution et la visualisation cartographique
  ne sont pas portés. **Compile-vérifié** sur Forge 1.20.1; essai en orbite réelle restant à faire.
- **PWR (réacteur à eau pressurisée) — système complet, pas une recette isolée.** Port de
  `com.hbm.blocks.machine.{BlockPWR,MachinePWRController}` et
  `com.hbm.tileentity.machine.TileEntityPWRController` (CE/1.7.10) :
  [MachinePWRControllerBlock.java](src/main/java/com/hbm_m/block/machines/pwr/MachinePWRControllerBlock.java),
  [MachinePWRControllerBlockEntity.java](src/main/java/com/hbm_m/blockentity/machines/pwr/MachinePWRControllerBlockEntity.java),
  [PWRGenericBlock/BlockEntity](src/main/java/com/hbm_m/block/machines/pwr/PWRGenericBlock.java),
  [PWRFuelItem.java](src/main/java/com/hbm_m/item/industrial/PWRFuelItem.java),
  [Function.java](src/main/java/com/hbm_m/util/Function.java).
  - **Découverte en cours de route** : les 10 blocs PWR (casing, reflector, port, fuel, control,
    channel, heatex, heatsink, neutron_source, controller) **existaient déjà** comme placeholders
    `Block` génériques dans `ModBlocks.java`, avec blockstates/modèles/noms anglais **et 8 recettes
    Assembler déjà générées** (`datagen/recipes/custom/AssemblerRecipeGenerator.java`) — seule la
    logique de jeu manquait. Les 15 noms de traduction des 15 variantes de combustible
    (`pwr_fuel_meu`...`pwr_fuel_bfb_pu241`) existaient aussi déjà dans `en_us.json`. Ça confirme,
    une fois de plus, que ce projet a un pipeline d'assets très en avance sur le code (même
    schéma que les drones et les satellites).
  - **Mécanique portée fidèlement** : assemblage par flood-fill (clic-droit sur le contrôleur,
    les blocs casing/reflector/port bornent le cœur, les blocs internes valides sont balayés et
    remplacés par la coquille IO partagée `PWR_BLOCK`) ; simulation de chaleur cœur/coque,
    consommation de combustible avec courbe de rendement par palier (`Function` —
    `FunctionSqrt`/`FunctionLogarithmic`, réutilise `BobMathUtil.squirt` déjà présent dans la
    base) ; refroidissement par `hbm_m:coolant` → `coolant_hot` via le trait `FT_Heatable`
    existant (`HeatingType.PWR`, déjà configuré sur plusieurs fluides dans
    `ModFluidTraitsBootstrap.java`) ; fusion du cœur (barres → `CORIUM`, nouveau bloc, explosion,
    radiation via `ExplosionNukeGeneric`) ; nettoyage automatique des coquilles orphelines si le
    contrôleur est détruit/désassemblé (port de `TileEntityBlockPWR.update()`).
  - **Les 15 paliers de combustible historiques** ont leurs valeurs de chaleur/rendement/durée et
    leurs formes chaudes ; les 15 recettes de barres suivent maintenant `RodRecipes.java` de HBM
    1.7.10 : billet adapté + plaque polymère, une barre par craft. Les deux variantes BFB gardent
    leur recette renforcée avec bismuth et pépites de combustible.
  - **Affichage des combustibles PWR rétabli** : 15 sprites historiques distincts et le sprite
    partagé des barres chaudes sont copiés sous `assets/hbm_m/textures/item/`; datagen crée les 30
    modèles d'objets correspondants. L'accès complet aux ingrédients et à leurs propres recettes
    reste à auditer dans la comparaison de progression.
  - **Recettes de craft ajoutées** : contrôleur, bloc combustible (slot) et les 15 barres — les 8
    autres blocs structurels avaient déjà leur recette Assembler.
  - **Simplifié vs CE, documenté** (voir javadoc de `MachinePWRControllerBlockEntity` et
    PORTING_NOTES.md) : taille de réacteur fixe (16 barres/20+8 connexions/8 échangeurs/8
    canaux/8 puits/2 sources — un futur gabarit plus grand est un prolongement naturel), pas de
    redstone-over-radio/OpenComputers/détection satellite (API absentes de cette base), pas de
    son de fonctionnement, pas de connexion directe aux tuyaux de fluide externes (capability
    Forge non câblée — le réservoir reste utilisable via le menu).
  - **Compile-vérifié** (Forge 1.20.1) : `compileJava` ✅, `runData` ✅ (recettes générées,
    confirmées sur disque), `build` ✅ (jar assemblé). **Non vérifié en jeu** (pas de `runClient`) :
    l'algorithme de flood-fill, le rendu du contrôleur/coquilles, et l'équilibrage chaleur/flux
    n'ont jamais tourné dans un monde réel.

- **Research Reactor — bloc unique, pas de multiblock.** Port de
  `com.hbm.blocks.machine.ReactorResearch` et `com.hbm.tileentity.machine.TileEntityReactorResearch`
  (CE/1.7.10) :
  [MachineResearchReactorBlock.java](src/main/java/com/hbm_m/block/machines/reactor/MachineResearchReactorBlock.java),
  [MachineResearchReactorBlockEntity.java](src/main/java/com/hbm_m/blockentity/machines/reactor/MachineResearchReactorBlockEntity.java),
  [PlateFuelItem.java](src/main/java/com/hbm_m/item/industrial/PlateFuelItem.java).
  - **Même découverte que PWR** : le bloc (`MACHINE_REACTOR_SMALL`, « Research Reactor » en
    anglais) et les 7 plaques de combustible + leurs déchets existaient déjà. Les sept sprites
    historiques et modèles d'objets sont présents. Le bloc jumeau
    `MACHINE_REACTOR` (« Breeding Reactor », CE `MachineReactorBreeding`) existe aussi en
    placeholder mais n'est **pas** porté dans cette passe — il dépend du flux produit par ce
    réacteur, donc logiquement il vient après, pas avant (bon candidat pour la prochaine fois,
    ~344 lignes côté CE).
  - **Mécanique portée** : grille 4×3 de plaques de combustible, chaque plaque transforme le flux
    reçu de ses voisines en flux de sortie via une des 5 courbes de réactivité
    (`PlateFuelItem.ReactivityFunction` — log/racine/quadratique négative/linéaire/passive, valeurs
    de réactivité et durée de vie exactes de `ItemPlateFuel`/`ModItems` de CE), chauffe, refroidie
    par l'eau réelle autour du bloc, fusion du cœur → `CORIUM` + `DECO_STEEL` + explosion +
    radiation si la chaleur dépasse le maximum, contrôle des barres par curseur (0-100%, animé).
  - **Simplifié vs CE, documenté** : adjacence de grille standard haut/bas/gauche/droite au lieu de
    la géométrie exacte (non documentée) de CE ; points d'échantillonnage de l'eau simplifiés
    (4 voisins horizontaux + dessus/dessous, 0-6 au lieu de l'échelle 0-12 de CE) ; pas
    d'OpenComputers. Les 7 recettes de plaques sont maintenant rétablies à l'enclume nucléaire,
    niveau 4, d'après `AnvilRecipes.java` de CE : un lingot de combustible produit une plaque.
    Les deux anciennes recettes directes de table (U233/U235) ont été retirées pour garder cette
    progression. Les sept plaques produisent leur plaque épuisée correspondante quand elles sont
    consommées.
  - **Compile-vérifié** (Forge 1.20.1) : `compileJava` ✅, `runData` ✅ (7 modèles de déchets,
    textures et traductions vérifiés), `build` ✅. **Non vérifié en jeu.**

- **Breeding Reactor — réacteur de reproduction alimenté par le Research Reactor.** Port de
  `com.hbm.tileentity.machine.TileEntityMachineReactorBreeding` et
  `com.hbm.inventory.recipes.BreederRecipes` (HBM CE 1.12.2), raccordé au bloc placeholder
  `MACHINE_REACTOR` : [MachineBreedingReactorBlockEntity.java](src/main/java/com/hbm_m/blockentity/machines/reactor/MachineBreedingReactorBlockEntity.java),
  [MachineBreedingReactorBlock.java](src/main/java/com/hbm_m/block/machines/reactor/MachineBreedingReactorBlock.java).
  - Le réacteur additionne le flux des réacteurs de recherche voisins sur ses quatre côtés.
    Dix conversions historiques sont actives (lithium→tritium, cobalt→Co-60, Ra-226→Ac-227,
    Th-232→combustible thorium, U-235→Np-237, Np-237→Pu-238, Pu-238→Pu-239, U-238/uranium→RGP,
    RGP→déchet). Les formes simple/double/quadruple ont les seuils de flux historiques (×1/×2/×3)
    et la vitesse de progression CE.
  - Les 17 matériaux × 3 formes sont enregistrés avec leurs sprites historiques (51 textures),
    modèles, traductions et entrées créatives. Les recettes de tiges vides, chargées et déchargées
    couvrent les billets historiques; les tiges de tritium se transforment en cellules de tritium.
    La recette d'assemblage historique est active; l'alliage résistant absent du registre dédié est
    représenté par le Staballoy existant.
  - **Simplifié vs CE, documenté** : le bloc placeholder moderne est un seul bloc, alors que le
    bloc CE utilisait une structure de trois blocs de haut. Le réseau d'inventaire externe CE et
    OpenComputers ne sont pas portés.
  - **Compile-vérifié** (Forge 1.20.1) : `runData`, `compileJava` et `jar` réussis; 99 recettes
    de tiges chargées/déchargées et six recettes de formes vides produites. **Non vérifié en jeu** :
    la progression flux→isotope et le rendu de l'interface n'ont pas encore tourné dans un monde réel.

- **Radiation-Powered Engine / RadGen — générateur passif.** Port de
  `com.hbm.blocks.machine.MachineRadGen` et `com.hbm.tileentity.machine.TileEntityMachineRadGen`
  (CE/1.7.10) :
  [MachineRadGenBlock.java](src/main/java/com/hbm_m/block/machines/MachineRadGenBlock.java),
  [MachineRadGenBlockEntity.java](src/main/java/com/hbm_m/blockentity/machines/MachineRadGenBlockEntity.java).
  - **Troisième système d'affilée avec la même découverte** : `MACHINE_RADGEN` (« Radiation-Powered
    Engine ») et tous les items de combustible (déchets nucléaires courts/longs + tiny + déchet
    nucléaire + gemme radioactive) existaient déjà, sans aucune logique derrière.
  - **Mécanique portée** : 12 files de décroissance parallèles (12 emplacements combustible + 12
    sorties), chaque file convertit indépendamment un déchet en énergie (FE) sur une durée fixe
    avant de produire sa forme épuisée (valeurs de puissance/durée/sortie exactes de CE). Sortie
    d'énergie via le réseau d'énergie déjà existant dans cette base
    (`BaseMachineBlockEntity`/`ensureNetworkInitialized`), comme `MachineWoodBurnerBlockEntity` —
    pas besoin de reproduire le port directionnel manuel de CE.
  - **Simplifié vs CE, documenté** : pas d'empreinte multiblock fictive (CE utilisait des blocs
    factices uniquement pour la boîte de rendu visuelle, non reproduit — bloc simple à la place) ;
    pas de distinction par sous-variante de déchet (`WasteClass`) puisque cette base n'a qu'un item
    générique par type de déchet, pas des variantes à dommage comme CE.
  - **Compile-vérifié** (Forge 1.20.1) : `compileJava` ✅ (premier coup), `runData` ✅, `build` ✅.
    **Non vérifié en jeu.**

- **ICF Fuel Pellet Press — préparation de combustible de fusion.** Port du cycle de
  `TileEntityICFPress` et `ItemICFPellet` de HBM CE 1.12.2 dans
  [MachineICFPressBlockEntity.java](src/main/java/com/hbm_m/blockentity/machines/MachineICFPressBlockEntity.java)
  et [ICFPelletItem.java](src/main/java/com/hbm_m/item/industrial/ICFPelletItem.java). La presse utilise
  le bloc `machine_icf_press` qui était déjà enregistré : elle combine une capsule vide et deux
  combustibles **différents** pour produire une pastille avec sa composition conservée en NBT.
  - Les 13 combustibles CE sont modélisés avec leurs multiplicateurs de réaction, vitesses de
    consommation et seuils d’allumage. Huit fluides peuvent être reçus par barils ou tuyaux; cinq
    combustibles solides déjà disponibles se placent directement. Deux réservoirs distincts
    gardent les deux composants séparés.
  - La pastille garde son niveau d’usure et sa composition dans l’objet. Une charge de muons
    accélère l’allumage et s’applique à 16 fabrications, conformément au compteur CE. L’inventaire,
    les fluides, le menu et les jauges GUI sont raccordés; les fluides sont restitués dans un baril
    si la presse est démontée.
  - La recette historique de capsule vide (treillis de huit fils fins) est générée. La fabrication
    de la presse reste à l’Assembler avec une recette moderne, tant que sa recette de progression
    CE au Plasma Forge n’a pas été intégrée à la table de cette machine.
  - **Limite de portée** : le contrôleur laser multibloc CE et sa chaîne de condensateurs restent à
    faire; le port actuel de contrôleur directionnel est documenté avec le cœur ci-dessous.
  - **Compile-vérifié** (Forge 1.20.1) : `runData`, `compileJava` et `jar` réussis. **Non vérifié en
    jeu** : menus, remplissage réel des tanks, NBT de la pastille et barre d’usure n’ont pas encore
    été observés dans un monde.

- **ICF Core — cœur de fusion utilisable.** Le bloc historique `icf`, jusque-là
  décoratif, possède maintenant un block entity, une interface, des réservoirs et la boucle de fusion
  issue de `TileEntityICF` de HBM CE. Les cinq emplacements d’alimentation chargent la pastille dans le
  cœur actif; une pastille épuisée rejoint l’un des cinq emplacements de sortie.
  - Le cœur reprend le seuil d’allumage et la consommation des pastilles, leur rendement thermique,
    l’échauffement résiduel du laser et la décroissance de chaleur de 0,999 par tick. Il limite la
    chaleur à 1 000 milliards, comme la référence.
  - Trois réservoirs raccordés aux tuyaux : caloporteur ICF en entrée (512 000 mB), fluide chaud en
    sortie (512 000 mB) et flux stellaire en sortie (24 000 mB). Le caloporteur accepte les fluides
    avec trait ICF et peut être ciblé par l’identifiant de fluide; les équations de conversion
    reprennent `FT_Heatable`.
  - L’écran utilise l’illustration historique `gui_icf.png`; l’inventaire conserve les positions du
    menu CE. Les trois réservoirs et la chaleur sont sauvegardés, et leurs liquides sont rendus en
    barils si le bloc est démonté.
  - **Écart assumé** : le cœur actif reste un bloc unique après assemblage de son enceinte historique
    CE 17×6. Le contrôleur directionnel ci-dessous lui envoie son pulse; le modèle CE ajoute un
    assemblage laser connecté, limité à 1 024 blocs, avec chaîne de condensateurs et d’émetteurs.
    Le réseau plasma dédié reste à porter; aucune validation en jeu n’a encore été faite.
  - **Compile-vérifié** (Forge 1.20.1) : `runData`, `compileJava` et `jar` réussis après l’enregistrement du contrôleur et de son modèle orientable. Le test en jeu reste à faire.

- **ICF Laser Controller — impulsions directionnelles actives.** Le bloc `icf_controller` reçoit
  l’énergie du réseau FE/HBM, suit son orientation et cherche un cœur ICF sur une ligne dégagée
  jusqu’à 48 blocs. S’il en trouve un, il consomme une impulsion de son buffer et la remet au cœur;
  plusieurs contrôleurs peuvent donc contribuer au même réacteur.
  - Le cœur n’accepte plus de puissance directe : son laser vient maintenant du contrôleur, comme
    dans le modèle CE. Le buffer et la position du faisceau sont sauvegardés.
  - **Simplifié vs CE** : l’assemblage par flood-fill, la vérification de la coque/du port et la chaîne cellule → émetteur → condensateur sont maintenant actives au clic sur le contrôleur. La capacité reprend la formule CE basée sur la racine du nombre de condensateurs et de turbochargeurs. Le contrôleur reste alimenté directement par le réseau FE/HBM; la destruction des blocs faibles sur le trajet reste désactivée. Le faisceau moderne s’arrête devant tout bloc
    plein afin de ne pas creuser le monde. La portée de 48 blocs suit la recherche historique.
  - **Compile-vérifié** (Forge 1.20.1) : `runData`, `compileJava` et `jar` réussis; le fonctionnement réel reste à observer en jeu.

- **Modules matériels du laser ICF — assemblage connecté actif.** Les six blocs (coque, port, cellule,
  émetteur, condensateur, turbochargeur) ont leurs textures historiques, modèles et traductions, et
  sont listés dans l’onglet Machines. Un clic sur le contrôleur parcourt le volume connecté (limite
  1 024 pièces), vérifie la coque, le port, la ligne de cellules et les composants adjacents, puis
  applique la formule CE de capacité aux condensateurs correctement reliés.
  - Le bloc `struct_icf_core` assemble maintenant l’enceinte ICF CE : il vérifie les composants
    orientés sur 17 longueurs et six niveaux, sans charger les chunks non chargés, puis transforme
    l’ancre en cœur actif. Un clic sur une structure incomplète affiche son état.
  - **Limites actuelles du laser** : CE accepte un assemblage connecté jusqu’à 1 024 pièces, sans
    gabarit fixe 8×8. La modernisation valide les pièces mais ne les convertit pas encore en blocs
    d’assemblage avec ports énergétiques; le faisceau ne détruit pas non plus le terrain. Le Plasma
    Forge fabrique les pièces ICF et DFC.
  - L’assemblage et ses compteurs sont sauvegardés. **Compile-vérifié** (Forge 1.20.1) :
    `runData`, `compileJava` et `jar` réussis; fonctionnement en jeu restant à observer.

- **Plasma Forge — soudage, assemblage ICF, DFC et équipements orbitaux.** L’interface dispose maintenant
  de douze entrées, d’une sortie protégée, d’un tampon HE et d’une jauge de progression. Les 11
  soudages CE (fer, acier, cuivre, titane, zirconium, aluminium, TC alloy, CD alloy, tungstène,
  CMB, osmiridium) et deux plaques avancées (euphémium, dineutronium) sont actifs. La forge
  fabrique aussi les six pièces laser, le contrôleur, le cœur structurel ICF, trois composants de
  structure/vaisseau, la presse ICF, l’élément HDE historique, le noyau DFC et ses quatre
  périphériques, la batterie dineutronium de Fensusan et le satellite Gerald. La poudre chlorophyte du noyau est récupérée depuis la bobine de hadron par la
  recette de recyclage CE. Les recettes à plusieurs
  ingrédients agrègent leurs coûts dans toute la grille, comme les entrées CE réparties en piles.
  La progression, l’énergie et le fluide sont persistés; le bloc se fabrique dans l’Assembler.
  Les mondes enregistrés avec l’ancienne grille déplacent automatiquement leur ancienne sortie
  vers le nouvel emplacement de sortie.
  - L’allumage vérifie une impulsion laser d’au moins 500 000 TU sur un cœur ICF chargé dans un
    rayon de huit blocs. Un réservoir d’entrée de 16 000 mB reçoit oxygène, gaz de reformage, xénon
    et flux stellaire par tuyaux; les
    réactifs ne peuvent pas être extraits et le contenu revient en baril à la casse.
  - **Substitutions documentées** : poudre d’astatine pour la poussière At-209 CE; tungstène soudé
    pour l’alliage résistant; fil dense d’alliage avancé pour le fil ND; fil dense de schrabidium
    pour le fil DNT; tungstène moulé pour les plaques de métaux lourds; circuit quantique générique
    pour le circuit contrôleur et singularité moderne pour la singularité étincelle DFC. Les formes
    exactes absentes restent des limites de parité. La recette Fensusan emploie la cellule spark
    moderne comme substitut du quantum battery pack; Gerald utilise le fil dense de schrabidium
    comme substitut du BSCCO dense et un circuit quantique générique pour son contrôleur quantique.
  - **Limites restantes** : réseau/assemblage DFC complet,
    multibloc de fusion et réseau plasma/neutron CE. Le cœur DFC produit une sortie HE, mais la
    fusion, la radiation et le mécanisme CE de défaillance ne sont pas portés.
  - **Compile-vérifié** (Forge 1.20.1) : `runData`, `compileJava` et `jar` réussis; classe, menu,
    modèle, recette de fabrication et traductions sont dans le JAR. Essai en jeu restant à faire.
- **DFC — les cinq blocs ont une première mécanique jouable.** Le noyau accepte deux catalyseurs,
  un cœur AMS et deux carburants tuyautables; il transforme les impulsions en sortie HE. L’émetteur
  directionnel consomme HE/cryogel et envoie un faisceau jusqu’à 50 blocs. L’injecteur transfère
  jusqu’à 1 000 mB de carburant par réservoir et par tick vers le noyau sur une ligne dégagée de
  15 blocs. Le récepteur orienté convertit les joules en HE (5 000 HE/J) avec cryogel. Le
  stabilisateur use une lentille AMS et maintient le champ du noyau; sans champ, le noyau n’accepte
  pas les impulsions. La lentille a la durabilité CE (432 000 000 ticks) et sa recette CE est
  générée. L’assemblage complet du réseau, les interfaces, la fusion, la radiation et la
  défaillance explosive ne sont pas encore portés. **Compile-vérifié** (Forge 1.20.1) :
  `runData`, `compileJava` et `jar` réussis; essai en jeu restant à faire.

- **Fusion Breeder — conversions de fluides activées.** La structure `BREEDER` et son contrôleur
  existaient déjà, mais `canProcess()` retournait toujours `false`. Le traitement accepte maintenant
  les recettes historiques de `FluidBreederRecipes` : gaz → syngaz, huile légère → gaz de reformage
  et huile légère craquée → gaz de reformage, à raison de 1 000 mB vers 1 000 mB.
  - Deux réservoirs de 8 000 mB séparent entrée et sortie. La capability fluide remplit l’entrée et
    ne vide que la sortie; les contenants peuvent être chargés et vidés depuis le menu. Les deux
    réservoirs sont sérialisés, et leurs contenus sont rendus en barils lorsque la machine est
    démontée.
  - **Écart assumé avec CE** : la machine moderne utilise le réseau FE et une cadence de 600 ticks,
    faute du réseau plasma/neutron du réacteur de fusion historique. Les conversions solides par
    `OutgasserRecipes`, l’alimentation en flux depuis un réacteur et le raccordement graphique au
    multiblock de fusion ne sont pas encore portés. Ce n’est donc pas encore un cycle de fusion
    complet.
  - **Compile-vérifié** (Forge 1.20.1) après `runData`, `compileJava` et `jar`. **Non vérifié en
    jeu** : la structure, les deux tanks, l’automatisation par tuyaux et la conservation au démontage
    doivent encore être observées dans une partie.

## ⚠️ Correction majeure (2026-10-07, après compilation) : les recettes ne sont PAS le chantier n°1

Le diagnostic « 48 recettes JSON, chantier n°1 » (posé en Phase 1-4 par simple `grep`/`find` sur
l'arbre git) était **incomplet, pas faux pour les mauvaises raisons** : `.gitignore` exclut
`src/generated/resources/data/`, où vit la sortie du générateur de données du projet. En lançant
`./gradlew :1.20.1-forge:runData`, **1535 recettes JSON ont été produites** à partir de code Java
déjà écrit et fonctionnel (`ModVanillaRecipeProvider.java`, 1752 lignes ; sept générateurs dédiés
dans `datagen/recipes/custom/`, 6405 lignes au total). Comparé à l'estimation CE (~2730
sites d'appel, elle-même approximative), on est à **≈56 %**, pas ≈2 %. Détail complet et nouvelle
priorisation : [docs/porting/ROADMAP.md](docs/porting/ROADMAP.md).

**Leçon pour la suite** : ne jamais juger la complétude d'un système data-driven de ce projet sans
avoir lancé `runData` au préalable. Voir [PORTING_NOTES.md](PORTING_NOTES.md).

## Audit complet de `com.hbm_m.recipe` (2026-10-07, après la correction ci-dessus)

Fait suite à une découverte troublante : le cyclotron, classé « à créer » après la première
correction, s'est révélé être un système **déjà complet** (registre Java codé en dur fidèle au
1.7.10/CE, GUI/menu/JEI inclus) — je l'avais manqué car ce projet utilise un 3ᵉ mécanisme de
recette en plus du datagen JSON. Audit complet refait en croisant deux méthodes (lecture de
chaque fichier + vérification dans la séquence d'init réelle `MainRegistry.commonSetup()`) :
détail dans [docs/porting/ROADMAP.md](docs/porting/ROADMAP.md).

**Résultat (fiable cette fois, vérifié par deux méthodes croisées) :**
- **Complets et actifs** : Centrifuge, Cyclotron, ChemicalPlant, Crystallizer, CrucibleSmelting,
  MoltenAlloy, Mixer, ArcWelder, Soldering (tous registres Java codés en dur) + Anvil/Assembler/
  BlastFurnace/Press/Shredder/crafting vanilla (datagen, 1535 recettes).
- **Stubs explicitement vides, documentés comme tels par l'équipe amont** (pas une découverte) :
  `CrucibleAlloyingRecipes`, `CrucibleMoldRecipes` — en attente du système MaterialStack/Mats.
- **Satellites** : les treize classes CE sont enregistrées maintenant (`SatelliteHorizons`,
  `SatelliteResonator`, `SatelliteMapper`, `SatelliteScanner`, `SatelliteRadar`, `SatelliteLaser`,
  `SatellitePrecisionLaser`, `SatelliteMiner`, `SatelliteLunarMiner`, `SatelliteRayScan`,
  `SatelliteDetector`, `SatelliteRelay` et `SatelliteScience`) sur `Satellite.register()`.
- **Confirmé absent, par recherche exhaustive ET absence de la séquence d'init** : RBMK, PWR,
  ZIRNOX, ICF/fusion, DFC, drones de logistique. **Correction n°3 (voir ROADMAP.md) : RBMK et
  ZIRNOX se sont en réalité révélés très avancés** (simulation complète codée dans leurs
  `BlockEntity`, pas une classe de recette — un 4ᵉ pattern qu'aucun audit par recette ne pouvait
  voir). **PWR a depuis été porté dans cette session** (voir plus bas). ICF/fusion, DFC et les
  drones restent confirmés absents.
- Systèmes de sécurité nucléaire (SCRAM, alarmes) : non localisés (non ré-audité avec la même
  rigueur — à refaire avant d'agir dessus)

## ⚠️ Correction majeure : NeoForge 1.21.1 ne compile pas (vérifié le 2026-10-07)

`BASE_DECISION.md` affirmait à tort que « 1.21.1 NeoForge » était « déjà fonctionnelle dans le même
arbre de sources ». En testant réellement (`./gradlew :1.21.1-neoforge:compileJava`), **2943
erreurs** — presque toutes des imports `net.minecraftforge.*` non substitués par le préprocesseur
Stonecutter pour cette cible, plus des API NeoForge manquantes (`FluidStorage`, `ModBlockEntities`,
`QuadBakingVertexConsumer`). La CI amont elle-même ne construit que `1.20.1-forge`. **Aujourd'hui,
seul Forge 1.20.1 est une cible réellement fonctionnelle** ; le détail est dans
[docs/porting/BASE_DECISION.md](docs/porting/BASE_DECISION.md) et
[PORTING_NOTES.md](PORTING_NOTES.md). Ça ne remet pas en cause le choix de base (toujours le
portage 1.20+ le plus riche), mais toute affirmation de compatibilité multi-loader doit désormais
être vérifiée par compilation, pas supposée à partir de la présence du dossier `versions/`.

## Fonctionnalités restantes (non confirmées présentes du tout)

Cf. ROADMAP.md. Pas encore d'extraction exhaustive registre-par-registre (seulement un comptage
de sites d'appel) — à affiner si besoin avant de porter un système précis.

## Bugs connus

Aucun bug fonctionnel constaté — seulement les avertissements Javadoc mentionnés ci-dessus
(cosmétiques, non bloquants, préexistants dans la base).

## Systèmes à tester

Le jar compile et s'assemble ; reste à lancer un client/serveur Minecraft réel pour valider le
comportement en jeu (pas fait dans cette session — nécessiterait de lancer `runClient`, qui
télécharge des assets clients supplémentaires et démarre une fenêtre de jeu).

## Dépendances

- **JDK 17** (Temurin 17.0.20.1) installé sous `C:/Users/steep/jdks/jdk-17.0.20.1+1` — utilisé
  pour `1.20.1-forge`/`1.20.1-fabric`.
- **JDK 21** (Temurin 21.0.12.1) installé sous `C:/Users/steep/jdks/jdk-21.0.12.1+1` — utilisé pour
  les cibles `1.21.x`.
- Les deux installés **sans droits admin** (zip portable Adoptium, pas l'installeur MSI — voir
  PORTING_NOTES.md pour pourquoi l'installeur winget a été abandonné).
- `org.gradle.java.home` corrigé dans chaque `versions/*/gradle.properties` (pointait vers le poste
  de l'auteur d'origine).
- `~/.gradle/gradle.properties` (hors dépôt, propre à cette machine) enregistre les deux JDK auprès
  du résolveur de toolchains Gradle.
- Gradle 9.3.1 (wrapper) — télécharge et fonctionne correctement depuis cette session.

## Décisions architecturales

Voir [docs/porting/BASE_DECISION.md](docs/porting/BASE_DECISION.md).
