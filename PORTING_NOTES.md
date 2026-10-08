# Notes de portage — différences importantes

## ⚠️ À lire avant de juger la complétude de quoi que ce soit : `runData` d'abord

`src/generated/resources/data/` est **gitignoré** (voir `.gitignore` ligne 36) : c'est la sortie du
générateur de données du projet (recettes, modèles de blocs/items, blockstates, tags,
advancements, loot tables...), jamais committée. Un `grep`/`find` sur l'arbre git (ce qu'a fait
l'analyse de Phase 1-4) **sous-estime massivement** le contenu réel : on avait conclu « 48 recettes
seulement » alors qu'une fois `./gradlew :1.20.1-forge:runData` lancé, **1535 recettes** sont
produites à partir de code déjà écrit (voir [docs/porting/ROADMAP.md](docs/porting/ROADMAP.md),
section « Correction majeure »). Avant toute conclusion sur l'état d'un système data-driven de ce
projet (recettes, loot tables, tags, modèles, advancements), lancer :

```bash
./gradlew :1.20.1-forge:runData
```

puis regarder dans `src/generated/resources/`, pas seulement dans `src/main/resources/`.

**Piège inverse à ne pas reproduire** : deux fichiers sous `src/generated/resources/` SONT suivis
par git malgré le `.gitignore` général du dossier —
`assets/hbm_m/lang/{en_us,ru_ru}.json` (probablement synchronisés via Crowdin, voir `crowdin.yml`
à la racine — ils contiennent des centaines de clés, dont des traductions de succès/boss, que le
`LanguageProvider` Java actuel ne réémet pas). Lancer `runData` **régénère ces deux fichiers avec
beaucoup moins de contenu** que la version committée (constaté : -3024/+384 lignes, suppression de
clés comme `advancements.hbm_m.boss_creeper.*`). **Ne jamais committer ces deux fichiers après un
`runData` sans comparer au préalable** (`git diff` avant `git add`) — dans le doute, `git restore`
pour revenir à la version committée, qui est la source de vérité pour ces deux-là spécifiquement.

## Ce dépôt n'est pas un portage écrit depuis zéro

Ce projet part d'un clone complet (historique git inclus) de
[Raptor324/HBM-Modernized](https://github.com/Raptor324/HBM-Modernized) (GPL-3.0), remote nommé
`upstream`. Ce choix est documenté et justifié dans
[docs/porting/BASE_DECISION.md](docs/porting/BASE_DECISION.md). Toute contribution « amont »
utile (corrections de bugs génériques, par exemple) devrait idéalement être proposée en PR à
`Raptor324/HBM-Modernized` plutôt que divergée silencieusement ici, sauf quand le changement est
spécifique à notre objectif de parité avec le contenu historique HBM.

## Architecture multi-version/multi-loader (Stonecutter)

La base utilise **Stonecutter** (`stonecutter.gradle.kts`, `settings.gradle.kts`) : un seul arbre
`src/` est compilé vers plusieurs couples version Minecraft/loader définis dans `versions/*`.
Déclarées actives dans `settings.gradle.kts` (`match(...)` non commenté) : `1.20.1-forge` (JDK 17)
et `1.21.1-neoforge` (JDK 21). Du scaffolding existe aussi pour `1.20.1-fabric`, `1.21.1-fabric`,
`1.21.11-fabric`, `1.21.11-neoforge` mais ces cibles sont commentées/inactives. Changer de cible
active se fait via `.sc_active_version` et les déclarations `match(...)` de `settings.gradle.kts`.

**⚠️ `1.21.1-neoforge` est déclarée active mais ne compile pas** (vérifié le 2026-10-07 :
`./gradlew :1.21.1-neoforge:compileJava` → 2943 erreurs, presque toutes des imports
`net.minecraftforge.*` non substitués par le préprocesseur `//? if forge`/`//? if neoforge`, plus
des API NeoForge manquantes). La CI amont (`.github/workflows/*.yml`) ne construit que
`1.20.1-forge` — c'est la seule cible à traiter comme réellement fonctionnelle aujourd'hui.
**Avant d'écrire du code en pensant « ça marchera aussi sur NeoForge », vérifier d'abord que
`1.21.1-neoforge` compile à l'instant présent** (il est probable que non, et que ce soit un chantier
séparé de remise en état, pas quelque chose qui se résout tout seul en continuant sur Forge).

**Implication pour le portage de contenu, en l'état actuel (NeoForge cassé)** : écrire du code
commun (sans API spécifique à un loader) reste une bonne pratique quand c'est naturel — mon ajout
de `SatelliteResonator` ne touche que des classes vanilla (`ServerLevel`, `ServerPlayer`,
`SoundEvents`, `Heightmap`) et compile sur Forge sans avoir pu être vérifié sur NeoForge puisque ce
dernier ne compile déjà pas sans ce changement. Mais ne pas se fixer comme objectif systématique
« doit compiler sur les deux cibles » tant que NeoForge lui-même ne compile pas : ce serait vérifier
une propriété qui ne peut pas encore être testée.

## Convention de nommage héritée du 1.7.10/1.12.2 vs conventions modernes

| 1.7.10 / CE (1.12.2) | HBM-Modernized (1.20.1+) |
|---|---|
| `TileEntity` | `BlockEntity` (paquet `blockentity/`) |
| Rendu OpenGL direct, enregistrement manuel | `DeferredRegister`, capabilities, datagen |
| Recettes codées en dur en Java | JSON de recettes + `Recipe<?>`/serializers, générées par datagen |
| NBT libre sur `ItemStack` | (à vérifier au fil du portage — en NeoForge 1.21+, NBT est remplacé par les *Data Components* ; HBM-Modernized cible aussi 1.20.1 Forge où NBT reste la norme — vérifier au cas par cas selon la cible active avant de porter une mécanique basée sur NBT) |

## Référence de contenu : original (1.7.10) vs Community Edition (1.12.2)

Quand une fonctionnalité existe différemment entre le 1.7.10 officiel et CE (1.12.2), **CE est la
référence par défaut** si le système en question a été explicitement amélioré dans CE par rapport
au 1.7.10 (assembly machine, chemical plant, particle accelerator, PWR/ZIRNOX/ICF/Research
Reactor, drones, armes — voir [docs/porting/RESEARCH.md §2](docs/porting/RESEARCH.md)), car sa
syntaxe/structure Java 8 moderne est plus proche de ce qu'on vise que le code 1.7.10 original. Pour
tout le reste, le 1.7.10 officiel reste la référence de comportement par défaut (c'est la version
qui reçoit encore des mises à jour aujourd'hui).

Toute reprise de logique, de recettes ou de ressources depuis CE ou l'original est consignée ici
avec le dépôt source, la révision, les fichiers d'origine et les fichiers de destination.

### Progression des combustibles PWR

- Source : `HbmMods/Hbm-s-Nuclear-Tech-GIT` à `ae673f8546ff5c27c51fc927569fbfa3792dccac`
  (GPL-3.0), `src/main/java/com/hbm/crafting/RodRecipes.java`, recettes `EnumPWRFuel` MEU à
  BFB_PU241. Les quinze recettes modernes réimplémentent les motifs historiques ; les recettes
  standard consomment deux billets et une plaque polymère pour une barre, et les deux BFB utilisent
  leurs ingrédients renforcés.
- Ressources : les quinze sprites `pwr_fuel.*.png` et `pwr_fuel_hot.png` de
  `src/main/resources/assets/hbm/textures/items/` dans la même révision GPL-3.0.
- Destination : `ModVanillaRecipeProvider.registerPwrFuelRecipe`, recettes BFB voisines,
  `ModItemModelProvider.registerPwrFuelItemModels` et les textures `assets/hbm_m/textures/item/`.
  Les modèles associent chaque ID moderne au sprite historique correspondant ; les formes chaudes
  utilisent le sprite chaud commun d'origine.

### Plaques épuisées du Research Reactor

- Source : même dépôt et même révision GPL-3.0 ; `com/hbm/tileentity/machine/TileEntityReactorResearch.java`
  associe chacun des sept combustibles à sa forme épuisée, et `com/hbm/hazard/HazardRegistry.java`
  donne les niveaux de radiation/chaleur historiques.
- Ressource ajoutée : `assets/hbm/textures/items/waste_plate_uranium.png`. Les formes U-233 et
  U-235 partagent ce sprite ; la forme Pu-239 partage le sprite MOX comme dans les ressources
  d'origine.
- Destination : nouveaux items `WASTE_PLATE_U233`, `WASTE_PLATE_U235` et `WASTE_PLATE_PU239`,
  modèles, traductions et entrées de danger ; `MachineResearchReactorBlockEntity.depletedFormOf`
  retourne désormais une forme épuisée pour chacun des sept combustibles.

### Recettes des plaques du Research Reactor

- **Source** : HBM Community Edition 1.12.2, `com/hbm/inventory/recipes/anvil/AnvilRecipes.java`
  (les sept recettes `plate_fuel_*`, tier 4).
- **Portage** : `AnvilRecipeGenerator` produit les sept recettes à l'enclume nucléaire : un lingot
  U-233, U-235, MOX, Pu-239 ou schrabidium, ou un billet Ra-226/Be ou Pu-238/Be, donne la plaque
  correspondante. Les niveaux et quantités suivent CE. Les deux recettes de table ajoutées
  antérieurement pour U-233/U-235 ont été remplacées par leur progression historique à l'enclume.
- **Assets** : les sept sprites historiques et les modèles sont déjà disponibles; aucune texture
  placeholder n'a été créée pour cette passe.

## Blocage Windows : chemins trop longs

Le clone initial de HBM-Modernized a échoué partiellement sur Windows (`Filename too long`) sur
des chemins d'assets profondément imbriqués (`textures/item/powders/tiny/...`). Corrigé en
activant `git config --global core.longpaths true` puis en reclonant. **Si ce dépôt est recloné
sur une autre machine Windows, exécuter cette commande avant le clone.** Envisager aussi
d'activer `LongPathsEnabled` au niveau de l'OS (clé de registre Windows) si d'autres outils que
git rencontrent le même problème — changement qui doit être fait par l'utilisateur lui-même, pas
par un agent, car c'est un réglage système.

## Outillage installé (Phase 5 — 2026-10-07)

- **JDK 17 et JDK 21** installés en **zip portable Temurin** (pas l'installeur MSI winget) sous
  `C:/Users/steep/jdks/jdk-17.0.20.1+1` et `C:/Users/steep/jdks/jdk-21.0.12.1+1`.
  **Pourquoi pas l'installeur winget/MSI** : `winget install EclipseAdoptium.Temurin.17.JDK
  --silent` s'est bloqué plusieurs minutes sans consommer de CPU — signe typique d'une invite UAC
  d'élévation qui ne peut pas être validée dans un shell non interactif. `--scope user` n'est pas
  supporté par ce paquet winget (« Aucun programme d'installation applicable n'a été trouvé »).
  Solution retenue : téléchargement direct du zip portable via l'API Adoptium
  (`api.adoptium.net/v3/assets/latest/<version>/hotspot?image_type=jdk&os=windows&architecture=x64`)
  puis extraction avec `Expand-Archive` — aucune élévation requise.
- **`org.gradle.java.home` corrigé localement** dans chaque `versions/*/gradle.properties` (le
  dépôt amont committe le chemin JDK 21 du poste personnel de l'auteur — `jdk-21.0.9.10-hotspot` —
  qui n'existe pas ailleurs). Mappage utilisé : JDK 17 pour `1.20.1-forge`/`1.20.1-fabric`, JDK 21
  pour toutes les cibles `1.21.x` (cohérent avec `ModPlatformPlugin.kt` : `>=1.20.6` → Java 21,
  sinon `>=1.18` → Java 17). **Ce correctif est spécifique à cette machine** — à refaire si le
  dépôt est cloné ailleurs, en pointant vers l'installation JDK locale de cette autre machine.

- **`~/.gradle/gradle.properties`** (hors dépôt git, propre à la machine) déclare
  `org.gradle.java.installations.paths` pour que le résolveur de toolchains Gradle trouve nos deux
  JDK sans dépendre du téléchargement automatique via foojay (qui a échoué une fois avec
  « No defined toolchain download url for WINDOWS on x86_64 architecture » avant qu'on enregistre
  les chemins explicitement).
- **Premier build réel réussi** : `JAVA_HOME=.../jdk-17.0.20.1+1 ./gradlew :1.20.1-forge:build` →
  `BUILD SUCCESSFUL`. Point d'attention : le script `gradlew` a besoin de `JAVA_HOME` **dans
  l'environnement du shell** pour démarrer (avant même de lire `gradle.properties`) — sans ça il
  échoue avec « JAVA_HOME is not set and no 'java' command could be found in your PATH », même si
  `org.gradle.java.home` est correctement configuré dans le projet.
- **Gradle 9.3.1** (wrapper fourni) : pas d'installation séparée nécessaire, se télécharge seul au
  premier lancement.
- `gh` CLI absent de cette session (recherches Phase 1 faites via l'API GitHub brute en `curl`).

### Attention au datagen des langues

Sur la cible Forge 1.20.1, `runData` réécrit les fichiers suivis `src/generated/resources/assets/hbm_m/lang/*.json`
à partir de `ModLanguageProvider`, qui ne contient pas encore toutes les traductions historiques.
La génération complète observée a supprimé plus de mille entrées existantes. Après chaque
`runData`, fusionner ses clés avec les catalogues suivis de la branche avant de garder ces fichiers :
les valeurs fournies par le générateur sont mises à jour, tandis que les traductions historiques
qu'il ne réémet pas sont conservées. Cette passe a aussi ajouté les nouveaux libellés au fournisseur
Java pour qu'ils survivent aux générations futures. Corriger le fournisseur pour couvrir le catalogue
complet reste un chantier séparé.

### Satellite Mapper moderne (2026-10-07)

- `SatelliteMapper` est enregistré sur l’item `SAT_MAPPER`, converti en `ItemSatChip` pour qu’il accepte une fréquence et soit déployable par le lanceur Soyouz existant.
- L’action du désignateur rapporte le chargement du chunk ciblé, X/Z, l’altitude du terrain et les joueurs à portée horizontale de 250 blocs (comportement issu de `com.hbm.saveddata.satellites.SatelliteMapper` de CE).
- Interface SAT_PANEL, requête de pollution/soot et affichage cartographique restent absents; le rapport textuel remplace temporairement le panneau interactif historique.
### Cœur ICF moderne (2026-10-07)

- `icf` est passé d'un bloc décoratif à un cœur jouable à bloc unique, avec le menu historique, trois
  réservoirs et le cycle de pellets/chaleur/refroidissement/flux stellaire porté de `TileEntityICF`.
- `icf_controller` reçoit l'énergie du réseau et envoie une impulsion au premier cœur visible dans
  son axe (48 blocs maximum). Le cœur ne consomme plus directement son propre buffer : l'impulsion
  arrive via `receiveLaserPulse`, comme le champ laser accumulé par la référence CE.
- Cela ne remplace pas l'assemblage CE connecté : flood-fill, cellules, émetteurs, condensateurs,
  turbochargeurs et calcul du laser selon les composants sont encore absents. Par prudence, le
  contrôleur moderne arrête le faisceau devant un bloc solide au lieu de le détruire.
- Le caloporteur est choisi par le slot d'identifiant de fluide et doit porter le trait
  `FT_Heatable` avec efficacité ICF; les fluides chaud et flux stellaire sont des sorties tuyautables.
- Compilation Forge 1.20.1 et jar réussis. Aucune vérification en monde de jeu n'a encore été faite.

### Assemblage du contrôleur laser ICF (2026-10-07)

- Port partiel de la validation flood-fill de `MachineICFController.assemble` et du calcul de puissance de `TileEntityICFController.getMaxPower` (HBM CE 1.12.2) vers `MachineICFControllerBlockEntity.assembleLaserParts`.
- Au clic, la modernisation vérifie un volume connecté (1 024 modules maximum), exige casing + port + cellule alignée derrière le contrôleur, puis compte les émetteurs, condensateurs et turbochargeurs reliés dans cet ordre. La capacité applique `sqrt(condensateurs) × 2 500 000 + sqrt(min(turbochargeurs, condensateurs)) × 5 000 000`.
- Simplifications : CE utilise un flood-fill connecté jusqu’à 1 024 blocs, pas un gabarit fixe 8×8. Les pièces modernes ne sont pas encore remplacées par les blocs d’assemblage CE; l’énergie passe par le contrôleur au lieu des ports, et le faisceau ne détruit pas de terrain ni ne blesse les entités. La validation au runtime de jeu reste à faire.
### Recettes de base du creuset (2026-10-07)

- **Source** : HBM original 1.7.10, `com/hbm/inventory/recipes/CrucibleRecipes.java` et
  `com/hbm/inventory/material/Mats.java` dans `work/references/hbm-original-1.7.10`.
- **Portage** : les 13 recettes de base sont enregistrées dans `MoltenAlloyRecipes`; les minerais
  d'hématite et de malachite sont d'abord fondus en intermédiaires, puis réagissent avec le flux
  pour produire fer/cuivre et laitier. Le flux et le laitier des recettes d'acier et de bronze ont
  également été conservés.
- **Formes de matériaux** : `MaterialType` fait le lien avec les ingots déjà inscrits dans
  `ModIngots` (U-238, cadmium, strontium, calcium, tungstène magnétisé, boue, ferrouranium et
  BSCCO). `flux_powder` fournit le flux. Le laitier n'avait pas de forme d'ingot moderne :
  `ModIngots.SLAG` crée `slag_ingot` et son modèle réutilise la texture existante
  `textures/block/block_slag.png`.
- Les trois recettes optionnelles `steelWrought`/`steelPig`/`steelMeteoric` restent absentes car
  elles ne sont activées que si GregTech 6 est installé.

## Licence et attribution

Ce projet est distribué sous **GPL-3.0**, héritée de la base HBM-Modernized et compatible avec
toutes les sources de contenu identifiées (original HBM, Community Edition, forks modernes — voir
[docs/porting/RESEARCH.md §6](docs/porting/RESEARCH.md)). Toute réutilisation de code externe doit
être attribuée dans ce fichier au moment où elle a lieu.

### Satellite Scanner moderne (2026-10-07)

- Port de la capacité `HAS_ORES` du `SatelliteScanner` CE : depuis le désignateur orbital, le Scanner compte les minerais vanilla par famille (charbon, fer, cuivre, or, redstone, lapis, diamant, émeraude) dans le chunk ciblé et sur toute la hauteur de construction.
- La lecture passe par `getChunkNow`; si le chunk n’est pas déjà chargé, le joueur reçoit un message et aucun chunk distant n’est chargé. Le résultat mesure les blocs de minerai, pas les veines ou les ressources HBM personnalisées.
- `sat_head_scanner` + `sat_base` produisent désormais l’item `satellite_scanner` à l’Arc Welder. `sat_scanner` reste le nom du bloc décoratif. L’item utilise `ItemSatChip`, est inscrit dans le registre orbital sous un nouvel ID ajouté en fin de liste, et possède son modèle, sa texture et ses traductions EN/RU/FR.
- Le panneau `SAT_PANEL` reste non porté; le rapport textuel est l’interface moderne de remplacement. `runData`, `compileJava` et `jar` Forge 1.20.1 réussis; seul l’essai en jeu reste à faire.
### Satellite Radar moderne (2026-10-07)

- `SAT_RADAR` devient une puce `ItemSatChip`, inscrite à la fin de `Satellite.register()` pour ne pas déplacer les IDs déjà sauvegardés. La recette existante assemble `sat_head_radar` et `sat_base` (600 ticks, 10 000 HE).
- Port de la portée horizontale CE de 1 000 blocs : l’action du désignateur résume les joueurs actifs de la dimension et les missiles du registre global HBM qui peuvent être détectés par le radar. La liste affiche jusqu’à huit contacts avec leurs coordonnées et conserve le compte total.
- Le registre global permet de rapporter les missiles même si leur chunk n’est pas chargé; la recherche ne force aucun chargement. La séparation du SAT_PANEL, les filtres et le détail des contacts personnalisés restent à porter.
### Satellite Laser moderne (2026-10-07)

- `SAT_LASER` est passé à `ItemSatChip` et inscrit à la fin de `Satellite.register()`; la recette `sat_head_laser` + `sat_base` préexistait (600 ticks, 10 000 HE).
- L’action du désignateur lance le MK5 à la hauteur de surface visée, avec la force de base CE (40) et le délai persistant de 5 minutes. L’API MK5 existante coupe les retombées mais conserve dégâts, destruction de terrain, changements de biome et irradiation instantanée, comme le Death Ray CE.
- Le `SAT_PANEL` est remplacé par l’action directe du désignateur. La configuration CE des dimensions paisibles n’a pas d’équivalent moderne; par défaut, le tir n’est donc pas restreint par dimension. `runData`, `compileJava` et `jar` Forge 1.20.1 réussis; essai en jeu restant à faire.

### Satellite Precision Laser moderne (2026-10-08)

- Le satellite utilise `ItemSatChip` et le tir au désignateur. Le délai CE de cinq secondes est
  sauvegardé dans les données du satellite; la recette moderne requiert tête laser, base et
  interface pour distinguer cette puce du Death Ray déjà existant.
- L’explosion CE est portée avec `ExplosionVNT` : rayon 5, traitement standard des blocs, dégâts
  laser croisés fixes de 1 000 avec perforation de 50 DT / 50 % DR et effets d’explosion HBM.
  Le type de dégâts `LASER` est relié à la catégorie énergie de la résistance Power Armor.
- La sélection d’entité par le panneau `SAT_COORD`, la sélection aléatoire à ±0,025 bloc et le
  rendu permanent du rayon ne sont pas reproduits; le désignateur moderne vise un bloc.
  `runData`, `compileJava` et `jar` Forge 1.20.1 réussis; essai en jeu restant à faire.

### Satellites miniers modernes (2026-10-08)

- `SatelliteMiner` et `SatelliteLunarMiner` sont ajoutés en fin du registre pour préserver les IDs
  orbitaux déjà sauvegardés. Les deux items sont `ItemSatChip`; le joueur les règle avant le vol
  Soyouz. Le tick serveur avance leur cycle sans charger de chunk.
- Les pools reprennent les 26 entrées pondérées du Miner d’astéroïdes CE et les sept entrées du
  Lunar Miner. Chaque cycle de 15 minutes produit 10 à 15 piles; le satellite attend que le joueur
  vide sa cargaison avant de relancer le cycle. Le NBT garde progrès et piles, et le désignateur
  livre le butin à l’inventaire sans perdre ce qui ne rentre pas.
- Le registre moderne n’avait pas `powder_power`; l’item et son sprite CE `powder_energy_alt` sont
  ajoutés pour garder cette sortie du pool. Les deux recettes Assembler reprennent les ingrédients
  modernes correspondant aux recettes CE; le baril de kérosène est adapté en carburant de fusée.
- `runData`, `compileJava` et `jar` Forge 1.20.1 réussis. Les modèles, sprite et deux JSON de
  recettes sont présents dans la sortie; l’essai en jeu reste à faire.

### Satellite RayScan moderne (2026-10-08)

- `SatelliteRayScan` est ajouté après les satellites miniers dans le registre pour préserver les
  IDs déjà sauvegardés. L’item `sat_ray_scan` est une puce réglable en fréquence, équipée du sprite
  CE; l’Assembler le fabrique en 1 200 ticks et 25 000 HE à partir de composants modernes.
- `RayEventRegistry` conserve jusqu’à 4 096 événements transitoires dans le serveur courant,
  fusionnés par dimension, position et type. Les entrées expirent selon l’horloge globale et le
  registre est vidé à l’arrêt du serveur.
- Le RayScan renvoie les signaux d’un rayon horizontal de 250 blocs, avec huit coordonnées au
  maximum. Les émissions neutroniques viennent des crayons RBMK et du PWR actif, les particules
  énergétiques du cœur ICF en fusion, et les ondes radar des pings de MachineRadar. `SAT_PANEL` et
  les événements radio du RTTY restent absents.
- `runData`, `compileJava` et `jar` Forge 1.20.1 réussis; classe, recette, modèle et texture
  vérifiés dans le JAR. Essai en jeu restant à faire.

### Satellite Detector moderne (2026-10-08)

- `SatelliteDetector` est ajouté après RayScan, sans déplacer les IDs orbitaux déjà sauvegardés.
  L’item `sat_detector` est une puce à fréquence avec le sprite CE; l’Assembler le fabrique en
  1 200 ticks / 25 000 HE avec les quantités CE de plaques d’or moulées, panneaux solaires, fils
  denses, circuits bismoïdaux, éléments basse densité et contrôleur avancé.
- `RadiationBurstRegistry` conserve au plus 4 096 impulsions transitoires, triées par dimension,
  avec expiration sur l’horloge serveur globale et purge à l’arrêt du serveur. Les durées et
  erreurs aléatoires CE sont reprises : faible 15 s / ±10 000 blocs, moyenne 10 ticks / ±2 500,
  forte 60 s / ±500. Le désignateur balaie toute la dimension, affiche le total et jusqu’à huit
  contacts intensité/coordonnées.
- Les explosions MK5 et Fleija publient une impulsion forte à leur premier tick; la mine Fat ajoute
  le signal faible CE; chaque ping du MachineRadar publie un signal moyen. Les coordonnées sont
  volontairement approximatives, comme dans la référence. Aucun chunk n’est chargé par le scan.
- `runData`, `compileJava` et `jar` Forge 1.20.1 réussis; essai en jeu restant à faire.

### Satellite Relay et RTTY modernes (2026-10-08)

- `SatelliteRelay` est ajouté après Detector pour préserver les IDs existants. L’item `sat_relay`
  devient une puce réglable; clic droit normal fait défiler les dimensions déjà actives du serveur,
  sneak+clic droit règle la fréquence, et le Soyuz copie la dimension choisie dans le satellite.
  Le designator envoie une trame de position dans le canal dont le nom est cette fréquence.
- `RttyMessageBus` reprend le principe RTTY CE : file d’émission différée, dernière valeur par
  paire dimension/canal et addition des valeurs numériques simultanées. Les canaux sont bornés à
  4 096 entrées; les messages expirent après cinq secondes et le bus est vidé à l’arrêt du serveur.
- `RTTY_PAGER` reçoit désormais les trames récentes du monde où se trouve le joueur; il utilise
  `ItemSatChip`, donc le Sat Linker peut copier son canal depuis une puce de satellite. Le pager se
  règle avec sneak+clic droit et affiche le dernier signal par clic droit. Le mécanisme CE de
  destruction du porteur sur le message `selfdestruct` n’est pas repris.
- La recette du Relais reprend la recette CE de l’Assembler (600 ticks, 100 HE); la bouteille
  d’hydrogène est représentée par une particule d’hydrogène, et le type de circuit basique par le
  circuit silicium existant. Modèles et recette présents dans le JAR; `runData`, compilation et
  `jar` réussis. Essai en jeu restant à faire.

### Satellite Science moderne (2026-10-08)

- `SatelliteScience` est ajouté en dernier dans le registre, donc les IDs antérieurs restent
  stables. `SatelliteManager.orbit` livre désormais un payload à `onPartDelivered` si cette
  fréquence est déjà occupée, comme le système legacy. Soyouz peut ainsi déposer les modules
  `sat_science_sensor`/`sat_science_assembler`, un disque vierge accordé, ou un assemblage orbital.
- La puce de laboratoire, les deux modules, l’assemblage orbital et les disques sont des items
  séparés avec les sprites CE. Le disque vierge est un payload accordable; les sorties Flight Data
  et Orbit Data sont des objets récupérés par le désignateur. Le capteur avance à un taux par
  module jusqu’à 100 heures, le vol produit au plus un disque toutes les 15 minutes, et chaque
  assembleur traite un assemblage orbital en trois minutes pour produire le circuit cristallin.
- Les compteurs de modules/disques, la progression capteur, la file d’assemblage et la cargaison
  sont sauvegardés en NBT. Les piles qui ne rentrent pas restent à bord. Les recettes Assembler
  modernes reprennent les composants CE disponibles; les composants manquants ont des substituts
  documentés dans le code. `runData`, `compileJava` et `jar` Forge 1.20.1 réussis; le SAT_PANEL,
  le dock orbital et l’essai en jeu ne sont pas portés/vérifiés.

### Plasma Forge : plaques avancées, pièces ICF, DFC et équipements orbitaux (2026-10-08)

- Le bloc décoratif devient une machine avec block entity, grille 4×3 (12 entrées), sortie protégée,
  menu, écran, stockage HE et réservoir de 16 000 mB. Les 11 soudages CE sont actifs : deux plaques
  moulées → une plaque soudée pour le fer, l’acier, le cuivre, le titane, le zirconium, l’aluminium,
  le TC alloy, le CD alloy, le tungstène, le CMB et l’osmiridium. Les plaques euphémium et
  dineutronium produisent quatre unités par lot. Durées et coûts de base repris de
  `PlasmaForgeRecipes`.
- La recette CE `plsm.hde` ajoute l’item `part_hde` : deux plaques moulées de bronze bismuthé,
  une plaque CMB soudée, un lingot CFT et 4 000 mB de flux stellaire donnent un HDE (600 ticks,
  25 000 000 HE/t, seuil de pulse 10 000 000 TU). Le sprite `part_generic.hde.png` provient de
  HBM Community Edition 1.12.2.
- La forge cherche un cœur ICF chargé dans un rayon de huit blocs et exige une impulsion d’au moins
  500 000 TU pour avancer. L’énergie consommée par recette vient du buffer de la machine. Le bloc
  possède une recette Assembler sans dépendance circulaire sur les plaques qu’il fabrique.
- Les fluides oxygène, gaz de reformage, xénon et flux stellaire arrivent par tuyaux; le handler
  externe est entrée seule. Le contenu est sauvegardé et restitué en baril à la casse. Les recettes
  multi-ingrédients couvrent la pièce HDE, les pièces ICF (laser, contrôleur, structures et cœur)
  et le noyau avec ses quatre périphériques DFC. La grille agrège les quantités d’un même ingrédient entre piles et accepte
  12 entrées pour les recettes CE réparties en piles. Les sauvegardes de l’ancienne grille à neuf
  entrées déplacent son ancien slot de sortie vers le nouvel index 12.
- La poudre chlorophyte est récupérée en recyclant une bobine de hadron chlorophyte (2 poudres,
  recette shapeless CE); la forge fabrique maintenant le noyau DFC avec 12 000 mB de flux stellaire.
- Deux équipements CE de fin de progression ont aussi leur recette : la batterie dineutronium
  Fensusan et le satellite Gerald. Fensusan emploie la cellule spark moderne comme substitut du
  quantum battery pack; Gerald consomme notamment 256 HDE, 64 détonateurs nucléaires, des plaques
  et du fil dense de schrabidium, un circuit quantique et la pièce UFO.
- La forge contient maintenant 33 recettes au total.
- Substitutions faute de formes modernes exactes : astatine pour At-209; tungstène soudé pour
  l’alliage résistant; fil dense avancé pour ND; fil dense de schrabidium pour DNT; tungstène moulé
  pour les plaques de métaux lourds; circuit quantique générique pour le circuit contrôleur; fil
  dense de schrabidium pour le BSCCO dense de Gerald; et
  singularité moderne pour la singularité étincelle. Le cœur DFC et l’émetteur directionnel sont
  maintenant fonctionnels : les deux catalyseurs et le cœur AMS se chargent par clic, les deux
  réservoirs acceptent les carburants CE, les impulsions consomment du carburant et remplissent le
  tampon HE extractible. L’émetteur consomme HE et cryogel; clic droit l’active, sneak+clic règle
  sa puissance. L’injecteur transfère jusqu’à 1 000 mB de chacun de ses deux carburants par tick
  sur une ligne dégagée de 15 blocs vers le noyau. Le récepteur convertit le faisceau en énergie
  exportable (5 000 HE/J), exige du cryogel et ne reçoit que par sa face avant. Le stabilisateur
  orientable utilise la lentille AMS, consomme `watts^4` HE par tick et transmet un champ au noyau
  sur 15 blocs; le noyau refuse les impulsions sans champ. La lentille a une durabilité CE de
  432 000 000 ticks; sa recette conserve le patron CE (plaques dineutronium, verre renforcé,
  blocs de diamant). L’interface graphique DFC, l’assemblage complet, la fusion, la radiation et
  le mécanisme de défaillance CE restent à
  porter; fonctionnement en jeu à vérifier. `runData`, `compileJava` et `jar` Forge 1.20.1 réussis;
  fonctionnement en jeu à vérifier.

### Assemblage de l’enceinte ICF (2026-10-08)

- `struct_icf_core` n’est plus un bloc décoratif : clic droit sur l’ancre vérifie le gabarit de
  `TileEntityICFStruct` de HBM CE 1.12.2. Le contrôle essaie les quatre orientations horizontales
  et compare les positions exactes des composants meta 0, 2 et 4 sur 17 longueurs et six niveaux.
- Les variantes CE sont mappées aux blocs modernes `icf_component`, `icf_component_vessel_welded`
  et `icf_component_structure_bolted`. Si toutes les pièces conviennent, l’ancre est remplacée par
  le cœur ICF fonctionnel; sinon, un message de structure invalide est affiché. Les chunks hors
  rayon déjà chargés ne sont jamais chargés par le vérificateur.
- L’enceinte de confinement est maintenant assemblable; cela ne porte pas le multibloc de fusion
  séparé ni le gabarit 8×8 de la chaîne laser. Build Forge et essai en monde consignés au suivi.
