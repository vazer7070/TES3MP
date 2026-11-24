# Rapport d'audit — TES3MP (fork `vazer7070/tes3mp`, branche `0.8.1`)

*Audit réalisé le 24 novembre 2026 sur le commit `49be5b6` (13 février 2024).*
*Annexes : [liste détaillée des 157 constats](CONSTATS_DETAILLES.md), avec preuves, impact, correctif et contre-vérification · [patch de compilation testé](patches/0001-tes3mp-server-constexpr-fix.patch).*

---

## 1. En bref

**TES3MP fonctionne et va loin sur le plan fonctionnel, mais le projet est figé et sa surface d'attaque réseau est fragile.**

- **Fonctionnalités.** Le multijoueur de Morrowind est complet : synchronisation des joueurs, PNJ, objets, quêtes et monde, plus une API Lua serveur de 764 fonctions et 68 événements. Le code multijoueur représente environ 48 000 lignes de C++.
- **Base technique figée.**
  - Le code repose sur OpenMW 0.47, sorti en 2021. L'amont TES3MP est quasiment à l'arrêt : 2 commits en 2024, et une branche 0.8.2 jamais fusionnée.
  - Le fork `vazer7070/tes3mp` est strictement identique à l'amont : aucun commit propre, aucune issue, aucune release.
- **Build et CI.**
  - **Le serveur ne compile plus tel quel** avec les compilateurs actuels (GCC ≥ 9, clang).
  - La dépendance réseau CrabNet n'est pas épinglée et casse avec GCC 13.
  - **Aucune CI ne fonctionne.**
- **Sécurité.** Un client malveillant peut **faire planter le serveur à distance de plusieurs façons**, dont une **écriture mémoire hors bornes**. Un serveur malveillant peut aussi corrompre la mémoire d'un client.
- **Tests.** Il n'y a **aucun test** du code multijoueur.

### Les 5 priorités

| # | Action | Pourquoi | Effort |
|---|---|---|---|
| 1 | **Rendre le serveur compilable** : appliquer le patch fourni, épingler CrabNet, ajouter une CI GitHub Actions | Sans cela, on ne peut ni livrer ni vérifier aucun correctif | 1 à 3 jours |
| 2 | **Corriger les failles réseau critiques** : index d'équipement, preInit, `resize()` non bornés, protocole maître, chaînes de format | Crash ou corruption mémoire du serveur déclenchable par n'importe quel client | 1 à 2 semaines |
| 3 | **Fiabiliser le moteur Lua** : wrappers `noexcept`, timers, fuite de pile Lua, course de données du MasterClient | Une simple erreur de script ou un arrêt suffit à tuer le processus | ~1 semaine |
| 4 | **Prendre les gains de performance rentables** : positions en `UNRELIABLE_SEQUENCED`, marqueurs de carte, index d'acteurs, sérialisation unique | Bande passante, latence et FPS côté client | 2 à 4 semaines |
| 5 | **Choisir la stratégie long terme** : rester sur 0.8.1, converger vers un OpenMW récent ou rejoindre un fork actif | Les 162 fichiers OpenMW patchés ont tous changé en amont | Décision, puis plusieurs mois |

### Chiffres clés

| Indicateur | Valeur |
|---|---|
| Code multijoueur | serveur 22 260 l. · protocole partagé 9 047 l. · client `mwmp` 16 579 l. |
| Points de modification d'OpenMW | 734 balises « tes3mp » dans 162 fichiers, soit 24 % des fichiers du client |
| Paquets réseau | ~99 identifiants, 101 classes de paquets, 174 processeurs (80 côté serveur, 95 côté client) |
| API de script | 764 fonctions C++ exposées à Lua (dont 37 dépréciées) et 68 événements ; CoreScripts en utilise 671 |
| Tests du code multijoueur | **0** |
| CI fonctionnelle | **0** (Travis, AppVeyor et GitLab hérités d'OpenMW, tous inopérants) |
| Build serveur sur Ubuntu 24.04 (GCC 13 / clang 18) | **échec** : 34 unités de compilation, 69 erreurs ; **succès** une fois le patch fourni appliqué |
| Constats de l'audit | 157 bruts : 13 critiques, 29 hauts, 63 moyens, 52 bas ou info |

---

## 2. Méthode et limites

**Démarche.** L'audit s'est organisé en 9 axes analysés en parallèle :
- réseau et protocole ;
- cœur du serveur ;
- moteur de scripts ;
- client `mwmp` ;
- intégration dans OpenMW ;
- sécurité ;
- build, CI et dépendances ;
- qualité et tests ;
- écosystème et documentation.

**Travaux empiriques.**
- **Builds réels** du serveur et du serveur maître sous Ubuntu 24.04 avec GCC 13.3, clang 18, CMake 3.28 et CMake 4.4. Le serveur patché a été lancé avec un script Lua minimal.
- **Analyse statique** du code serveur et du code partagé avec clang-tidy 18 et cppcheck 2.13. Les sorties brutes ont été triées à la main.
- **Bancs d'essai** pour reproduire les défauts : harnais ASan sur les parseurs de paquets, micro-benchmarks, ThreadSanitizer sur le serveur maître. Les clones des tags OpenMW 0.47.0 et 0.50.0 ont servi à mesurer la divergence.

**Contre-vérification adversariale.** Chaque constat critique ou haut, ainsi que chaque bug ou faille de sévérité moyenne, a été relu par un vérificateur distinct chargé de le **réfuter**. Résultat sur 82 constats :
- 49 confirmés ;
- 33 partiellement confirmés, avec une sévérité corrigée (28 abaissées, 3 relevées) ;
- 0 réfuté.

Les 75 constats de sévérité basse n'ont pas été contre-vérifiés. J'ai en outre relu moi-même les extraits de code des constats critiques.

**Limites.**
- **Client non construit.** Le client complet n'a été ni lié ni lancé. Les 113 `.cpp` OpenMW modifiés ont bien été compilés en `-Wall -Wextra`, mais les constats sur le client et l'intégration viennent de la lecture du code et de bancs isolés.
- **Plateformes non testées** : Windows/MSVC et macOS.
- **Pas de test de charge** avec de vrais clients. Les chiffres de bande passante sont des **estimations** calculées à partir des tailles de paquets.
- **Passe de complétude interrompue.** Elle visait trois angles morts : la persistance de l'état serveur (qui vit dans CoreScripts, dépôt externe), les scripts CI Android et macOS, et les licences tierces. Ses agents ont été interrompus par une limite d'usage, et ces zones restent survolées.
- **Nuance sur l'échec de compilation.** Il s'agit d'un **problème connu en amont** (issue `openmw-tes3mp#552`). nixpkgs le contourne avec un patch local sur `Script/Types.hpp` et `-fpermissive`. Le dépôt, lui, n'a jamais été corrigé.

---

## 3. Ce que contient le dépôt

```
┌──────────────────── Client « tes3mp » = OpenMW 0.47 modifié ────────────────────┐
│ apps/openmw : 734 balises « Start of tes3mp » dans 162 fichiers                 │
│   └─ apps/openmw/mwmp (16,6 k l.) : LocalPlayer / DedicatedPlayer, autorité de  │
│      cellule sur les PNJ (LocalActor / DedicatedActor), ObjectList, GUI, chat   │
└──────────────────────────────────┬──────────────────────────────────────────────┘
                                   │ UDP fiable — CrabNet (fork RakNet, externe)
┌──────────────────────────────────┴──── components/openmw-mp (9 k l.) ───────────┐
│ BasePacket + RW() symétrique · 101 classes de paquets · 5 contrôleurs           │
│ structures Base* partagées · protocole maître · TimedLog                        │
└──────────────────────────────────┬──────────────────────────────────────────────┘
┌──────────────────────────────────┴─── tes3mp-server : apps/openmw-mp (22 k l.) ─┐
│ boucle mono-thread (Receive → processeurs → TimerAPI → sleep 1 ms)              │
│ CellController / Cell::sendToLoaded · 80 processeurs · thread MasterClient      │
│ Script/ : 764 fonctions C++ → Lua (LuaJIT + LuaBridge), 68 événements           │
│        └──▶ CoreScripts (dépôt EXTERNE) = toute la logique de jeu serveur       │
└──────────────────────────────────┬──────────────────────────────────────────────┘
                                   │ annonce toutes les ~15 s
          apps/master (maître RakNet + API REST)  ◀──  apps/browser (navigateur Qt5)
```

| Zone | Rôle | Taille et technologies |
|---|---|---|
| `apps/openmw-mp` | Serveur dédié `tes3mp-server` | 22 k lignes, C++17, LuaJIT/LuaBridge, CrabNet |
| `components/openmw-mp` | Protocole partagé : paquets, contrôleurs, structures | 235 fichiers, 9 k lignes |
| `apps/openmw/mwmp` | Client multijoueur greffé sur le moteur | 154 fichiers, 16,6 k lignes |
| `apps/openmw` (hors `mwmp`) | Moteur OpenMW 0.47 avec points d'accroche | 155 fichiers modifiés, +10 518 / −234 lignes |
| `apps/master` | Serveur maître (annuaire) RakNet et REST | 716 lignes, plus Simple-Web vendorisé (2016) |
| `apps/browser` | Navigateur de serveurs | Qt5, 1,7 k lignes |
| `apps/opencs`, `launcher`, `wizard`… | Outils OpenMW hérités | quasiment non modifiés |
| `apps/openmw_test_suite` | Tests gtest hérités d'OpenMW | 20 fichiers, **aucun sur le multijoueur** |
| `docs/`, `manual/` | Documentation Sphinx d'OpenMW | 74 fichiers, **aucun ne mentionne TES3MP** |
| `CI/`, `.travis.yml`, `appveyor.yml`, `.gitlab-ci.yml` | CI héritée | 2 218 lignes, inopérantes |
| `files/tes3mp/` | Configurations par défaut du client et du serveur, UI, icônes | — |

**Ce qui n'est pas dans le dépôt** :
- **CrabNet**, la bibliothèque réseau, à cloner et compiler soi-même ;
- **CoreScripts**, toute la logique de jeu serveur en Lua ;
- les données du jeu ;
- toute documentation propre à TES3MP : le wiki externe compte 7 pages et sa page d'accueil est vide.

---

## 4. Points forts

- **Sérialisation symétrique.** Une seule méthode `Packet(bs, send)` sert à la lecture et à l'écriture via `RW()`, ce qui évite les divergences de format. Les paquets ne dépendent pas du singleton réseau, donc ils sont **faciles à tester**. Un prototype de test aller-retour de 40 lignes fonctionne déjà.
- **Identité de l'émetteur fiable.** Le serveur l'identifie par `packet->guid` et ignore le GUID présent dans le contenu, ce qui bloque l'usurpation directe d'un autre joueur.
- **Envoi par deltas et seuils.** Seuls les index modifiés des stats, attributs, compétences et équipement sont envoyés. Les stats dynamiques ont des seuils de 2 à 4 points. Les positions ne partent que pendant un mouvement. Les mises à jour de PNJ sont regroupées par cellule.
- **Diffusion par cellule.** `Cell::sendToLoaded` cible les joueurs qui ont la cellule chargée, avec un **contrôle d'autorité** sur `ID_ACTOR_POSITION`.
- **Séparation moteur / logique.** Le C++ reste générique et le gameplay vit en Lua, ce qui permet de moder sans recompiler. Les signatures d'événements sont vérifiées à la compilation (`static_assert`, `Script.hpp:77`). Le dispatch d'un événement Lua coûte 57 à 71 ns : **ce n'est pas un goulot**.
- **API Lua très large et documentée en ligne.** 717 blocs `\brief` couvrent 726 déclarations, avec un nommage cohérent Get/Set/Send/Read/Clear.
- **Modifications d'OpenMW traçables.** Elles sont balisées et commentées de façon systématique : 164 des 170 fichiers modifiés portent une balise. Ce sont surtout des ajouts (575 ajouts pour 159 changements).
- **Quelques parseurs déjà défensifs** : `PacketPreInit`, `ActorPacket` et `ObjectPacket` (plafond de 3000), ainsi que les index de compétences, attributs et stats.
- **Code propre au sens du compilateur.** Hors du problème bloquant, le code compile avec peu d'avertissements : 53 avertissements GCC uniques, la plupart `-Wdeprecated-copy` venant de RakNet. On compte 0 tabulation, 0 CRLF et 4 TODO.

---

## 5. Problèmes critiques, à traiter en premier

### 5.1 Le serveur ne compile pas tel quel — `BLD-1`, `SCR-1`, `QUA-9`

`apps/openmw-mp/Script/Types.hpp:104` convertit un pointeur de fonction en `void*` dans un constructeur `constexpr`, utilisé par la table `static constexpr ScriptFunctionData functions[]`. Les compilateurs réagissent ainsi :

- GCC 9 à 14 rejettent `reinterpret_cast` dans une expression constante, et `-fpermissive` ne débloque rien.
- clang 18 rejette la branche `#else` de `Types.hpp:107`.

Sur Ubuntu 24.04, cela donne 34 unités de compilation en échec et 69 erreurs. Le code n'était sans doute accepté que par GCC 8, celui de l'ancienne CI Travis.

**Correctif testé** : [`patches/0001-tes3mp-server-constexpr-fix.patch`](patches/0001-tes3mp-server-constexpr-fix.patch), qui modifie 25 fichiers de façon essentiellement mécanique.
- L'adresse de la fonction passe par `template<auto F> void* ScriptAddrOf()`.
- Les 764 entrées de table s'écrivent désormais `SFP(Fonction)`.
- Les 2 sites d'appel utilisent `getAddr()`.

Avec ce patch, `tes3mp-server` compile avec GCC 13 et avec clang 18, et un appel réel Lua → C++ fonctionne. Le patch s'applique proprement avec `git apply`.

**Autres obstacles au build :**

| ID | Problème | Correctif |
|---|---|---|
| `BLD-2`, `ECO-4` | CrabNet est cloné sans version (`CI/before_install.linux.sh:10`). Son HEAD `19e6619` date de janvier 2020 et produit 11 erreurs avec GCC 13 (`intptr_t` non déclaré). | Épingler le commit (sous-module ou `FetchContent`) et appliquer le correctif CrabNet PR #18 (commit `3ec9a33`, celui qu'utilise nixpkgs) ou `-include cstdint`. |
| `BLD-3` | `cmake_minimum_required(3.1)` est placé **après** `project()`. CMake 4.x refuse de configurer. | Mettre `cmake_minimum_required(VERSION 3.16...3.31)` en tête, passer à `find_package(Boost CONFIG)` et utiliser `FetchContent_MakeAvailable`. Contournement immédiat : `-DCMAKE_POLICY_VERSION_MINIMUM=3.5`. |
| `BLD-6` | Le build du serveur seul exige SDL2, OpenGL, LZ4 et les en-têtes OSG, et télécharge recastnavigation. Rien de tout cela n'est nécessaire au serveur. | Conditionner ces `find_package` à `BUILD_OPENMW OR BUILD_OPENCS`, et isoler les composants dont le serveur a réellement besoin. |
| `BLD-7` | `FindRakNet.cmake` exige une bibliothèque Debug `RakNetLibStaticd`, ne lit que `$ENV{RAKNET_ROOT}` et n'est pas `SYSTEM`. | Réécrire le module avec une cible importée `RakNet::RakNet` et `find_package_handle_standard_args`, et rendre la bibliothèque Debug optionnelle. |
| `BLD-4`, `ECO-13` | La CI est morte. Travis est arrêté (et son `fi` orphelin est une erreur de syntaxe). GitLab et AppVeyor ne construisent ni CrabNet ni LuaJIT. `.github/` ne contient que `FUNDING.yml`. | Mettre en place une CI GitHub Actions (exemple en §9). |
| `ECO-12` | Simple-Web, vendorisé dans le serveur maître, date de 2016 et donne 22 erreurs en simulant Boost ≥ 1.87. | Mettre à jour Simple-Web ou remplacer ce serveur HTTP. |

### 5.2 Sécurité réseau : crashs et corruption mémoire déclenchables à distance

| # | Faille | Où | Déclenchement | Correctif |
|---|---|---|---|---|
| 1 | **Écriture hors bornes** via l'index d'équipement (`SA-1`, critique, confirmé) | `components/openmw-mp/Packets/Player/PacketPlayerEquipment.cpp:41-42` | Un client authentifié envoie `ID_PLAYER_EQUIPMENT` avec `equipmentIndex` < 0 ou ≥ 19. Le serveur écrit alors des données choisies par l'attaquant hors de `Item equipmentItems[19]`. Les paquets voisins (compétences, attributs) ont ce garde-fou, pas celui-ci. | `if (equipmentIndex < 0 \|\| equipmentIndex >= 19) { packetValid = false; return; }`, sur le modèle de `PacketPlayerSkill.cpp:41` |
| 2 | **Lecture hors bornes avant authentification** (`SEC-1`, `SRV-1`, critique, reproduit par SIGSEGV) | `apps/openmw-mp/Networking.cpp:261` et `PacketPreInit.cpp:58` | Un `ID_GAME_PREINIT` avec un plugin à `hashN=0` provoque la lecture de `dataFile->second[0]` sur un vecteur vide. Le seul « mot de passe » RakNet exigé est la chaîne de version, qui est publique. | Rejeter `hashN == 0` dans `PacketPreInit`. Ajouter aussi le `return false` manquant après `CloseConnection` (`Networking.cpp:231-238`, `SRV-7`), qui rend en outre la fonction indéfinie. |
| 3 | **Allocations sans borne** pilotées par des compteurs reçus (`NET-1`, `SRV-5`, `SEC-2`, `SA-2`, `QUA-1`, critique) | 24 paquets environ : Inventory, Spellbook, Journal, Topic, Book, CellState, Faction, QuickKeys, WorldMap, ClientScript*… | Sous ASan, un `ID_PLAYER_INVENTORY` de 8 octets utiles (`count=0x7FFFFFFF`) demande une allocation d'environ **171 Go**. Le `std::bad_alloc` lancé dans un `Process() noexcept` déclenche `std::terminate`, et le serveur tombe. | Écrire un helper unique `ReadCount(count, max)` qui vérifie aussi les bits restants (`GetNumberOfUnreadBits`), et plafonner **chaque** `resize()`. |
| 4 | **Compteurs signés du protocole maître** (`NET-2`, `SEC-3`, critique) | `components/openmw-mp/Master/ProxyMasterPacket.hpp:74` | Une annonce avec un compteur négatif aboutit à un `resize()` gigantesque, sans `try/catch`. Le serveur maître tombe. | Utiliser des compteurs non signés bornés et entourer `MasterServer::Thread` d'un `try/catch`. |
| 5 | **Échecs de lecture ignorés** et amplification (`NET-4`, `QUA-6`, haute) | `ObjectPacket.cpp:34-43` ; seul 1 retour de `RW()` sur 963 est testé | Un `ID_OBJECT_MOVE` d'environ 20 octets annonce 3000 objets. Le serveur crée 3000 objets non initialisés, fuit des octets de pile et diffuse environ 63 Ko à **chaque** joueur, soit une amplification d'environ 3000×. | Rendre l'échec de lecture « collant » dans `RW()`, faire passer `packetValid = false` au premier échec et faire tester `isPacketValid()` par **tous** les processeurs. Rejeter aussi les bits non lus. |
| 6 | **Chaînes de format contrôlées par l'extérieur** (`SEC-4`, `SCR-8`, `SA-3`, `QUA-3`) | client : `apps/openmw/mwmp/Networking.cpp:419` ; serveur : `Script/Script.hpp:103` ; `TimedLog.hpp:45` | Un serveur malveillant envoie des noms de plugins contenant `%s`, et `vsnprintf` côté client lit la pile ou plante. Côté serveur, le message d'erreur Lua sert de format. La cause commune est que `TimedLog::print` n'a pas d'attribut `format(printf)` et que `#pragma GCC system_header` masque les avertissements. | Écrire systématiquement `LOG(..., "%s", str)`. Ajouter `__attribute__((format(printf, 2, 3)))` et retirer le pragma, puis compiler avec `-Werror=format-security`. |
| 7 | **Diffusion vers des connexions non authentifiées** (`NET-5`, haute) | `components/openmw-mp/Packets/BasePacket.cpp:66` et 5 processeurs Object qui relaient en global | Un client qui passe PreInit sans jamais faire le handshake reste connecté indéfiniment et reçoit le chat global et les relais d'objets, même sur un serveur protégé par mot de passe. | Diffuser uniquement aux joueurs `POSTLOADED`, imposer un délai d'expiration au handshake (environ 10 s) et faire passer ces 5 processeurs par `sendToLoaded` avec un contrôle d'autorité. |
| 8 | **Client : pointeur vers une variable de pile** (`INT-1`, critique) | `apps/openmw/mwworld/store.cpp:810` | Quand le serveur surcharge une cellule intérieure, `mSharedInt` garde l'adresse d'une variable locale détruite, ce qui corrompt la mémoire du client. La fonction peut aussi se terminer sans `return`. | Pointer vers la copie stockée, jamais vers l'argument, et compiler avec `-Werror=return-type`. |
| 9 | **API REST du maître** (`ECO-1`, `ECO-2`, `SA-4`, haute) | `apps/master/RestServer.cpp:38` et `:100`, `MasterServer.cpp:63` | Un `POST` sans authentification injecte de faux serveurs : 6 entrées suffisent à afficher 4 996 joueurs. Un guillemet dans un nom casse le JSON pour tous les consommateurs. La map des serveurs est partagée entre deux threads sans verrou, ce que ThreadSanitizer confirme. | Supprimer les routes `POST`/`PUT` « < 0.6 », générer le JSON avec une bibliothèque, ajouter un `std::mutex` et limiter les annonces par IP. |

**Faiblesses structurelles** (moyennes, à planifier) :
- aucune limitation de débit par client, et `SetLimitIPConnectionFrequency` n'est jamais appelé (`SRV-14`, `SEC-6`) ;
- transport sans chiffrement : CrabNet est compilé avec `LIBCAT_SECURITY=0`, alors que l'option `CRABNET_ENABLE_LIBCAT_SECURITY` existe. Le mot de passe du serveur circule en clair et apparaît dans les journaux, le nombre d'essais est illimité, et le hachage côté client n'est pas salé (`NET-11`, `SEC-5`, `CLI-14`) ;
- modèle « client autoritaire » : position, stats et dégâts sont rediffusés sans validation (`SEC-7`). C'est **inhérent au design** ; les contrôles de vraisemblance peuvent se faire en partie dans CoreScripts.

> **Méthode recommandée.** Ces failles ont une cause commune : la désérialisation fait confiance au flux. Plutôt que de corriger site par site, il vaut mieux réunir dans `BasePacket` :
> - un échec de lecture collant ;
> - des compteurs bornés ;
> - des index vérifiés ;
> - une validation obligatoire avant `Do()`.
>
> Il faut ensuite **fuzzer** les ~100 parseurs avec libFuzzer et ASan. Ces parseurs sont testables isolément avec `peer=nullptr`.

### 5.3 Robustesse du serveur et du moteur de scripts

| ID | Problème | Conséquence | Correctif |
|---|---|---|---|
| `SCR-3` (haute) | Les wrappers Lua sont `noexcept` (`LangLua.cpp:61-95`). Les fonctions de l'API contiennent 229 `.at()` dans des fonctions `noexcept`, et les paramètres `const char*` nil ne sont pas testés. | `pcall(tes3mp.SendMessage, "abc", "x")` provoque `terminate` (code 134), et `pcall(tes3mp.SendMessage, 1, nil)` un **SIGSEGV** (code 139). **Une faute de frappe dans un script tue le serveur**, même sous `pcall` et avec `ignoreScriptErrors`. | Retirer `noexcept`, convertir les exceptions en `luaL_error`, refuser nil pour les paramètres de type `s` et garder les index. |
| `SCR-4`, `QUA-4` (haute) | `TimerAPI::CreateTimerLua` parcourt la map avec `for (auto timer : timers)`, ce qui travaille sur une **copie** (`TimerAPI.cpp:72`). | La réutilisation d'un slot libéré est perdue : fuite mémoire, timers fantômes, ID 0 renvoyé à tort. | Écrire `for (auto& timer : timers)` et ajouter un `break`. |
| `SCR-5` (moyenne) | Les timers ne sont jamais libérés et `Tick` est en O(N), environ 1 000 fois par seconde. | La charge CPU augmente avec l'uptime : 948 µs par tick avec 100 000 timers, soit environ 95 % d'un cœur. | Utiliser un tas trié par échéance, libérer automatiquement les timers à usage unique et passer à `steady_clock`. |
| `SCR-2` (haute, partiel) | `CreateTimerEx` conserve des `const char*` qui pointent vers des chaînes Lua (`LuaFunc.cpp:159`). | Use-after-free quand le timer se déclenche. Avec CoreScripts, le chemin restant est : mort puis déconnexion en moins de 5 s. | Copier les arguments en `std::string`. |
| `SCR-6` (moyenne) | Chaque événement laisse une valeur sur la pile Lua (`LangLua.cpp:269`). | Mesuré sur banc : `PANIC ... stack overflow` après environ 65 500 événements sans timer. | Ajouter `lua_pop` et un test de non-régression sur `lua_gettop`. |
| `SCR-7` | La gestion des erreurs est incohérente : les timers échappent à `ignoreScriptErrors`, `OnServerScriptCrash` peut se rappeler récursivement, et il n'y a pas de traceback. | Diagnostic difficile et arrêts imprévisibles. | Passer par un `pcall` unique avec `debug.traceback` et un garde de récursion. |
| `SRV-2`, `NET-7`, `QUA-5`, `BLD-5` | Le thread `MasterClient` parcourt la table des joueurs sans verrou. | Course de données, et **SIGSEGV à l'arrêt dans 3 cas sur 3** quand le serveur maître est activé. | Envoyer un instantané préparé par la boucle principale (un compteur atomique suffit), puis `join()` avant de détruire `Networking`. |
| `SRV-4` | La macro `GET_PLAYER` teste une variable codée en dur `player` et non son argument (`ScriptFunctions.hpp:37`). | Garde-fou inopérant à 1 endroit (`Mechanics.cpp:197`). | Utiliser le paramètre de la macro. |
| `SCR-12`, `SCR-11` | Les plugins natifs (`.so`/`.dll`) n'exécutent jamais leurs timers ni leurs fonctions publiques. La branche ARM est cassée depuis le refactor `953be7c`. | Fonctionnalités mortes. | Corriger ou retirer, et documenter. |
| `BLD-15`, `ECO-6`, `SRV-9` | Une erreur au démarrage se termine par `std::terminate` et un core dump. Les gestionnaires de signal ne sont pas *async-signal-safe*. | Arrêts sales, journaux trompeurs. | `try/catch` dans `main` et `std::atomic<bool>` / `sig_atomic_t`. |

### 5.4 Bugs du client

| ID | Problème | Conséquence |
|---|---|---|
| `CLI-1` (haute) | `apps/openmw/mwmp/Cell.cpp:91` fait `erase(it++)`, puis la ligne 101 fait `++it` sans condition. | **Le client autoritaire se fige** (boucle infinie) quand le dernier PNJ local d'une cellule est supprimé. Reproduit par un micro-test. |
| `CLI-2` (haute) | `runConsoleCommands` parcourt un `ObjectList` que `Console::execute` vide puis réémet (`ObjectList.cpp:960`). | Use-after-free et écho des commandes au serveur. |
| `CLI-3` | `PlayerList::getPlayer(guid)` utilise `operator[]`, qui insère des `nullptr` ; `getPlayersInCell` les déréférence ensuite. | SIGSEGV. |
| `CLI-11` | `setFactions` boucle sans fin si le rang reçu dépasse le rang maximal de la faction. | Gel du client par un serveur malveillant ou bogué. |
| `INT-2`, `INT-3`, `INT-4` | Pointeur pendant dans `mIdCache` après une réinitialisation de cellule ; `updatePtrsWithRefId` modifie le vecteur qu'il parcourt et fuit un `ManualRef` ; `ActionTeleport` utilise `newCellStore` sans l'initialiser. | Corruption mémoire et crashs intermittents. |
| `NET-6` | `WriteCompressed<float>` borne à [-1, 1] le poids et la qualité des enregistrements dynamiques. | Objets créés par le serveur avec des valeurs fausses. |
| `SA-5` | `reset()` vide le conteneur pendant une boucle *range-for* (`ObjectList.cpp:761`). | Invalidation d'itérateur. |

---

## 6. Ce qui manque

**Tests et qualité**
- Aucun test du code multijoueur : pas de test aller-retour des paquets, pas de test de l'API de script, des timers ni de `CellController`, pas de test de bout en bout avec des clients robots.
- Aucun fuzzing des parseurs réseau, aucun build ASan, UBSan ou TSan, aucun clang-tidy en CI. `BUILD_WITH_CODE_COVERAGE` existe mais n'est jamais utilisé.
- Aucun `.clang-format` (l'amont OpenMW en a adopté un), aucun `.clang-tidy`, pas de `-Werror` ciblé sur `return-type`, `format-security` ou `maybe-uninitialized`. Ces drapeaux auraient bloqué plusieurs bugs trouvés ici.

**Build, CI et livraison**
- Aucun workflow GitHub Actions, aucune release ni binaire, aucun Dockerfile ni image de serveur dédié, aucun paquet deb, rpm ou AppImage.
- Ni CMakePresets ni script de bootstrap. Aucune option pour figer le hash de compatibilité réseau : la compatibilité client-serveur dépend du hash git au moment du build (`BLD-9`), donc un build depuis une archive source n'est pas reproductible.
- Ni Dependabot ni Renovate. CrabNet date de 2020 et recastnavigation de 2021.

**Réseau**
- Pas de limitation de débit, pas de délai d'expiration pour le handshake, pas de budget de bande passante par client.
- Pas de canal non fiable pour le mouvement, pas de quantification, pas de numéro de séquence ni d'horodatage dans les positions, donc pas d'interpolation par snapshots.
- Aucune métrique réseau : `RakNetStatistics` n'est jamais lu.
- Pas de négociation explicite de version ou de capacités (`NET-13`). Les identifiants de paquets sont implicites sur un `uint8`, et il ne reste que 21 valeurs libres.
- Aucune documentation du format des paquets ni de la machine d'états de connexion.

**Exploitation du serveur**
- La console d'administration est morte : `handleInput.cpp` n'est pas compilé (`SRV-6`).
- Pas de rechargement à chaud des scripts, pas de métriques (temps de tick, paquets par seconde), pas de mode démon propre.
- Breakpad est désactivé et son sous-module n'est pas initialisé (`BLD-13`). Le crash catcher du client est coupé sur toutes les plateformes (`INT-14`).
- Les configurations par défaut publient automatiquement le serveur sur le maître public sans mot de passe (`ECO-7`). Côté client, la journalisation par défaut est en mode « spam » (`ECO-16`).

**Documentation**
- Aucun guide de compilation : comment obtenir CrabNet et LuaJIT, quelles versions d'OSG, MyGUI et Bullet utiliser.
- Aucun guide d'hébergement : port UDP 25565, pare-feu, installation de CoreScripts, service systemd, sauvegardes.
- Aucune référence générée de l'API Lua : ni HTML ni stubs LuaLS. **Les 68 callbacks ne sont pas documentés** (`ECO-10`).
- `CONTRIBUTING.md`, `CHANGELOG.md` et `CHANGELOG_PR.md` sont hérités d'OpenMW ; le dernier est encore un gabarit avec des placeholders (`ECO-11`). `.readthedocs.yaml` et Sphinx 1.8.5 sont obsolètes (`ECO-15`).
- Le README ne dit rien de ce fork : objectif, branche suivie, mainteneur.

**Gouvernance**
- Aucun modèle d'issue ou de PR, ni `SECURITY.md`, ni `CODE_OF_CONDUCT.md`.
- Aucun responsable identifié pour traiter des failles qui sont pourtant exploitables à distance.

---

## 7. Optimiser : performances

| Problème | Mesure ou constat | Action | Gain attendu | Effort |
|---|---|---|---|---|
| Positions envoyées en `RELIABLE_ORDERED` (`NET-8`, `CLI-5`) | 0 paquet non fiable. Une position perdue bloque toutes celles qui suivent (blocage en tête de file). | Passer positions et animations en `UNRELIABLE_SEQUENCED` sur un canal dédié. | Fluidité nettement meilleure en cas de perte, moins de retransmissions | S |
| Taille et fréquence des positions (`NET-9`, `CLI-5`, `CLI-9`) | Environ 58 octets utiles par paquet, 66,7 Hz. La cadence dépend du FPS. En l'air, 2 positions et 1 paquet d'animation partent à chaque tick. **Estimation** pour 32 joueurs dans la même cellule : environ 31 Mbit/s en sortie serveur pour les seules positions. | Tick réseau fixe à 20–30 Hz ; positions quantifiées relativement à la cellule et rotations sur 16 bits ; identifiant joueur sur 16 bits au lieu d'un GUID de 8 octets ; suppression des doublons en l'air. | Bande passante des positions probablement divisée par 3 à 5 (estimation à mesurer) | M |
| Interpolation naïve (`CLI-6`) | Lerp ×15 sans horodatage, `moveObject` appelé à chaque frame même à l'arrêt. | Tampon de snapshots horodatés (environ 100 ms) avec interpolation, extrapolation bornée, et aucune mise à jour à l'arrêt. | Mouvements fluides malgré la gigue | L |
| `Cell::readActorList` en O(n²) (`SRV-3`) | Copies de `BaseActor` (2 760 octets). Mesuré : **101,6 ms par paquet pour 500 acteurs**, sur le thread unique du serveur. | Index `unordered_map<(refNum, mpNum), size_t>` et accès par référence. | Moins de 1 ms mesuré | S–M |
| Marqueurs de carte (`CLI-4`) | Chaque position reçue détruit et recrée **tous** les widgets de marqueurs 4 fois, soit 200 à 260 reconstructions MyGUI par seconde **par joueur distant**, même carte fermée. | `moveMarker()` en place, à 2–4 Hz au plus, en réutilisant les widgets. | FPS client qui ne dépend plus du nombre de joueurs | S |
| Relais `sendToLoaded` (`NET-10`, `QUA-15`) | Le paquet est re-sérialisé pour chaque destinataire, et une `std::list` triée est reconstruite à chaque envoi. | Sérialiser une fois et envoyer le même `BitStream` à N destinataires. | CPU serveur divisé à peu près par le nombre de destinataires sur ce chemin | S |
| Index d'acteurs par chaînes (`CLI-8`) | Clés construites avec `ostringstream` pour chaque acteur à chaque frame : 282 ns contre 4,6 ns avec `uint64` et `unordered_map`. | Clé numérique `(refNum, mpNum)` sur 64 bits. | ×61 sur ce chemin | M |
| Copies profondes à chaque tick (`CLI-7`) | `NpcStats` et `CreatureStats` copiés, `BaseActor` passé par valeur. | Passage par `const&` et comparaison champ par champ. | Moins d'allocations | S |
| Recherches linéaires dans le monde (`INT-11`) | 23 appels à `CellStore::searchExact` dans `ObjectList.cpp`. | Index `(refNum, mpNum)` → `Ptr` par cellule. | Coût par paquet d'objet en O(1) | L |
| Timers (`SCR-5`) | `Tick` en O(N) environ 1 000 fois par seconde, timers jamais libérés. | Tas trié par échéance et libération automatique. | CPU stable sur la durée | S |
| Boucle serveur (`SRV-8`) | `sleep(1 ms)` fixe et sondage du clavier. | Attente sur socket avec délai (prochain timer), ou tick fixe. | Moins de latence et de CPU à vide | M |
| Portée de l'IA (`INT-9`) | Forcée à 204 800 unités au minimum, ce qui ignore le réglage utilisateur. | La rendre configurable par le serveur. | CPU client | S |
| Variables de scripts synchronisées (`CLI-10`, `INT-17`) | Un paquet fiable par modification. Liste blanche parcourue linéairement, avec une copie de `std::string` pour chaque script local à chaque frame. | `unordered_set` et envoi groupé par tick. | Moins de paquets et de CPU | S–M |

> L'appel d'un événement Lua (57 à 71 ns) et le dispatch des paquets ne sont **pas** des goulots : inutile de les optimiser en priorité.

---

## 8. Améliorer : architecture et maintenabilité

1. **Découpler OpenMW de `mwmp`** (`INT-5`).
   - Constat : on compte 298 appels `mwmp::Main::get()` dans 84 fichiers OpenMW, 289 `#include "../mwmp/…"` et 75 séquences `objectList->reset()` dupliquées. Rien ne permet de construire le client sans réseau.
   - Action : créer une interface `MWBase::MultiplayerManager`, enregistrée dans `MWBase::Environment` sur le modèle de `MWBase::LuaManager` en 0.50. Elle exposerait des événements sémantiques comme `onObjectStateChanged`, `shouldApplyLocally` ou `isLocalAuthority`.
   - Prévoir une implémentation nulle pour le mode solo et les tests.
   - Migrer sous-système par sous-système : `mwscript` d'abord, puis `mwclass`.

2. **Préparer une remise à niveau sur OpenMW** (`INT-6`).
   - Constat : les 162 fichiers patchés encore présents en 0.50 ont **tous** changé en amont (+41 683 / −33 691 lignes), et 8 n'existent plus. Entre-temps, OpenMW a introduit un scripting Lua natif (0.48), `ESM::RefId`, ESM4 et un formatage clang-format.
   - Stratégie par étapes :
     - a) appliquer le `.clang-format` amont à l'arbre TES3MP **avant** de rebaser, pour neutraliser les conflits de formatage ;
     - b) découper les modifications en séries de patchs par sous-système ;
     - c) rebaser sur 0.48, puis 0.49, puis 0.50 ;
     - d) porter les règles « disable unilateral X » vers les gestionnaires Lua d'OpenMW (`I.Activation.addHandlerForType`, etc.). La documentation Lua d'OpenMW 0.50 mentionne explicitement la compatibilité avec un futur multijoueur.

3. **Réduire le code répétitif.**
   - Constat : sur 174 processeurs, 65 des 79 processeurs serveur ont 3 lignes utiles ou moins. Il y a 9 `Process()` clonés et 4 % de code dupliqué (jscpd).
   - Action : remplacer ces processeurs par une table déclarative ou un template `RelayProcessor<ID, Portée, Callback>` (`QUA-12`, `SRV-13`).

4. **Découper les fonctions géantes.** `PacketRecordDynamic::Packet` compte 853 lignes de code pour une complexité cyclomatique de 156. Il faut un sérialiseur par type d'enregistrement (`QUA-13`).

5. **Clarifier la propriété de la mémoire.** On compte 325 `new`, 35 `delete` et 9 pointeurs intelligents, plus des `operator[]` qui insèrent des entrées nulles (`QUA-14`, `CLI-15`). Passer à `unique_ptr` et à des fonctions `find()` explicites.

6. **Contractualiser l'API de script** (`SCR-9`, `SCR-10`).
   - Générer depuis la table `ScriptFunctions` un manifeste JSON, des stubs LuaLS/EmmyLua et une référence HTML.
   - Exposer une version d'API interrogeable depuis Lua.
   - Vérifier le contrat contre CoreScripts en CI. CoreScripts appelle par exemple `SetExterior`, qui n'existe pas.
   - Remplacer l'appel par pointeur variadique `R(*)(...)`, qui est un comportement indéfini, par des wrappers typés.

7. **Versionner le protocole** (`NET-13`). Numéroter explicitement les identifiants de paquets, prévoir une négociation de version et de capacités au handshake, et cesser d'utiliser le hash git comme clé de compatibilité.

8. **Fiabiliser le balisage.** Corriger les 12 anomalies de balisage (`INT-7`) et ajouter un lint CI qui vérifie l'appariement Start/End. Remplacer les `return;` qui désactivent du code amont par des conditions explicites (`INT-10`).

9. **Récupérer les correctifs de la branche 0.8.2** (`ECO-3`, `INT-15`). Elle contient 10 commits de 2022 (sons d'objets, cooldowns, alliances) absents de 0.8.1.

---

## 9. Feuille de route proposée

### Phase 0 — Débloquer et sécuriser (≈ 1 à 2 semaines)

1. Appliquer [`patches/0001-tes3mp-server-constexpr-fix.patch`](patches/0001-tes3mp-server-constexpr-fix.patch). Épingler CrabNet et appliquer son correctif GCC 13. Corriger l'ordre `cmake_minimum_required` / `project`.
2. Ajouter une CI GitHub Actions qui construit le serveur et le maître avec GCC et clang, puis lance un **test de fumée** : script `serverCore.lua` minimal qui appelle `StopServer(0)`, et vérification du code de sortie.
3. Corriger les failles §5.2 n° 1 à 6 et 8 ainsi que la course de données du maître : borne de l'index d'équipement, `hashN == 0` et `return` dans preInit, `ReadCount` borné sur tous les compteurs, échec de lecture collant et `isPacketValid()` obligatoire, compteurs du maître, `"%s"` et attribut `format`, `Store<ESM::Cell>::override`, `std::mutex` dans le maître.
4. Corriger la robustesse §5.3 : wrappers Lua sans `noexcept`, `auto&` dans `TimerAPI`, copie des chaînes de `CreateTimerEx`, `lua_pop`, synchronisation du `MasterClient`.
5. Corriger les bugs client `CLI-1`, `CLI-2`, `CLI-3`, `CLI-11` et `INT-2` à `INT-4`.

### Phase 1 — Fiabiliser, mesurer, documenter (≈ 1 à 2 mois)

- **Tests**
  - tests aller-retour pour les ~100 paquets ;
  - cibles libFuzzer sur les parseurs ;
  - job CI en ASan/UBSan avec le test de fumée ;
  - tests de l'API Lua et de `TimerAPI` ;
  - clang-tidy en CI sur `apps/openmw-mp` et `components/openmw-mp`.
- **Réseau**
  - limitation de débit par client et `SetLimitIPConnectionFrequency` ;
  - délai d'expiration du handshake ;
  - diffusion réservée aux joueurs `POSTLOADED` ;
  - métriques exposées (paquets/s, octets/s et RTT par client).
- **Performances rapides** (§7) : `UNRELIABLE_SEQUENCED`, marqueurs de carte, index d'acteurs côté serveur, sérialisation unique, tas de timers.
- **Serveur maître** : suppression des routes REST héritées, JSON généré par une bibliothèque, configuration des ports, limites par IP.
- **Documentation et outillage**
  - `BUILDING.md` et `HOSTING.md` ;
  - Dockerfile du serveur ;
  - référence Lua générée et documentation des 68 callbacks ;
  - section « État du fork » dans le README ;
  - modèles d'issue et de PR, `SECURITY.md` ;
  - `.clang-format` et `.clang-tidy`.
- **Amont** : intégrer les commits de la branche 0.8.2 et proposer les correctifs de sécurité à TES3MP/TES3MP.

### Phase 2 — Moderniser (≈ 3 à 6 mois et plus)

- **Décision stratégique** entre trois voies :
  - (a) maintenir 0.8.1 en mode correctif ;
  - (b) converger vers OpenMW 0.50 ou plus (§8, point 2) ;
  - (c) rejoindre un fork existant. Par exemple DreamWeave, auteur de la PR #661 fusionnée en amont ; vérifier son activité avant de trancher.
- Interface `MWBase::MultiplayerManager` et découplage progressif.
- **Protocole v2** :
  - quantification et identifiants compacts ;
  - séquences et horodatages, interpolation par snapshots ;
  - gestion d'intérêt par distance ;
  - compression des gros paquets (`RecordDynamic`, `CellState`, `WorldMap`) ;
  - négociation de version ;
  - authentification par défi-réponse salé, et chiffrement optionnel (`CRABNET_ENABLE_LIBCAT_SECURITY` ou couche DTLS).
- Factorisation des processeurs et de `PacketRecordDynamic`, passage aux pointeurs intelligents.

### Exemple de point de départ pour la CI (à ajuster, non exécuté sur GitHub)

Ce workflow reprend la configuration validée pendant l'audit. Il **ne passera qu'une fois le patch `constexpr` appliqué**. Le chemin d'installation de CrabNet (`lib/` ou `lib64/`) reste à vérifier.

```yaml
# .github/workflows/server.yml
name: tes3mp-server
on: [push, pull_request]
jobs:
  linux:
    runs-on: ubuntu-24.04
    strategy:
      fail-fast: false
      matrix:
        include:
          - { cc: gcc,   cxx: g++ }
          - { cc: clang, cxx: clang++ }
    env:
      CC: ${{ matrix.cc }}
      CXX: ${{ matrix.cxx }}
      RAKNET_ROOT: ${{ github.workspace }}/crabnet-install
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }        # le hash git entre dans la compatibilité réseau
      - name: Dépendances
        run: |
          sudo apt-get update
          sudo apt-get install -y ninja-build ccache \
            libboost-filesystem-dev libboost-program-options-dev libboost-system-dev \
            libboost-iostreams-dev libluajit-5.1-dev liblz4-dev libsdl2-dev libgl-dev \
            libopenscenegraph-dev
      - uses: actions/cache@v4
        id: crabnet
        with: { path: crabnet-install, key: "crabnet-19e6619-${{ matrix.cc }}" }
      - name: CrabNet (épinglé)
        if: steps.crabnet.outputs.cache-hit != 'true'
        run: |
          git clone https://github.com/TES3MP/CrabNet crabnet-src
          git -C crabnet-src checkout 19e66190e83f53bcdcbcd6513238ed2e54878a21
          cmake -S crabnet-src -B crabnet-build -G Ninja -DCMAKE_BUILD_TYPE=Release \
            -DCRABNET_ENABLE_DLL=OFF -DCRABNET_ENABLE_SAMPLES=OFF \
            -DCMAKE_CXX_FLAGS="-include cstdint" -DCMAKE_INSTALL_PREFIX="$RAKNET_ROOT"
          cmake --build crabnet-build && cmake --install crabnet-build
      - name: Configuration (serveur + maître seulement)
        run: |
          cmake -S . -B build -G Ninja -DCMAKE_BUILD_TYPE=RelWithDebInfo \
            -DCMAKE_C_COMPILER_LAUNCHER=ccache -DCMAKE_CXX_COMPILER_LAUNCHER=ccache \
            -DBUILD_OPENMW=OFF -DBUILD_OPENCS=OFF -DBUILD_LAUNCHER=OFF -DBUILD_WIZARD=OFF \
            -DBUILD_BROWSER=OFF -DBUILD_BSATOOL=OFF -DBUILD_ESMTOOL=OFF -DBUILD_ESSIMPORTER=OFF \
            -DBUILD_MWINIIMPORTER=OFF -DBUILD_NIFTEST=OFF -DBUILD_UNITTESTS=OFF -DBUILD_MASTER=ON \
            -DRakNet_LIBRARY_DEBUG="$RAKNET_ROOT/lib/libRakNetLibStatic.a" \
            -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
      - run: cmake --build build --target tes3mp-server masterserver
      # Étapes suivantes à ajouter : test de fumée (StopServer(0)), job ASan/UBSan, clang-tidy, upload-artifact.
```

---

## 10. Annexes

- **[CONSTATS_DETAILLES.md](CONSTATS_DETAILLES.md)** : les 157 constats classés par axe. Pour chacun : emplacement `fichier:ligne`, preuve, impact, recommandation, effort et résultat de la contre-vérification. Le fichier peut servir directement de backlog.
- **[patches/0001-tes3mp-server-constexpr-fix.patch](patches/0001-tes3mp-server-constexpr-fix.patch)** : correctif de compilation du serveur (25 fichiers), testé avec GCC 13.3 et clang 18.1. Pour l'appliquer : `git apply audit/patches/0001-tes3mp-server-constexpr-fix.patch`. Il n'a **pas** été appliqué au code du dépôt.

**Correspondance des doublons entre axes**

| Sujet | Identifiants |
|---|---|
| `resize()` sans borne | NET-1, SRV-5, SEC-2, SA-2, QUA-1 |
| preInit | SEC-1, SRV-1, SRV-7, NET-12 |
| Chaînes de format | SEC-4, SCR-8, SA-3, QUA-3 |
| Course de données du `MasterClient` | SRV-2, NET-7, QUA-5, BLD-5 |
| Échec de compilation | BLD-1, SCR-1, QUA-9 |
| CrabNet | BLD-2, ECO-4 |
| Timers | SCR-4, QUA-4, SCR-5 |
| Positions `RELIABLE_ORDERED` | NET-8, CLI-5 |
| Relais `sendToLoaded` | NET-10, QUA-15 |
| Map du serveur maître | ECO-2, SA-4 |
| Mot de passe en clair | NET-11, SEC-5, CLI-14 |
| Code répétitif des processeurs | SRV-13, QUA-12 |
