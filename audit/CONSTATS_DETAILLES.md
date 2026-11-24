# Annexe — Constats détaillés de l'audit TES3MP

Liste exhaustive des **157 constats bruts** produits par les 9 axes d'analyse (plus l'analyse statique rattachée à l'axe Build). Des doublons existent entre axes ; le [rapport principal](RAPPORT_AUDIT.md) les fusionne et les priorise.

- **Vérification** : chaque constat critique/haute, et chaque bug ou faille de sévérité moyenne, a été relu par un vérificateur sceptique distinct chargé de le réfuter. Résultat : 49 confirmés, 33 partiellement confirmés (sévérité corrigée), 0 réfuté. Les 75 autres n'ont pas été contre-vérifiés.
- **Sévérité** : celle affichée est la sévérité *après* vérification ; la sévérité initiale est indiquée quand elle diffère.
- `<scratchpad-audit>` désigne le répertoire de travail temporaire de l'audit (bancs d'essai, logs de build), non conservé.

## Sommaire

1. [Réseau & protocole](#réseau--protocole) — 18 constats
2. [Serveur dédié (cœur)](#serveur-dédié-cœur) — 14 constats
3. [Moteur de scripts serveur (Lua/natif)](#moteur-de-scripts-serveur-luanatif) — 17 constats
4. [Client multijoueur (mwmp)](#client-multijoueur-mwmp) — 16 constats
5. [Intégration dans OpenMW & divergence amont](#intégration-dans-openmw--divergence-amont) — 17 constats
6. [Sécurité (modèle de menace transverse)](#sécurité-modèle-de-menace-transverse) — 11 constats
7. [Build, CI, dépendances (empirique)](#build-ci-dépendances-empirique) — 28 constats
8. [Qualité de code & tests](#qualité-de-code--tests) — 18 constats
9. [Écosystème, documentation & santé du projet](#écosystème-documentation--santé-du-projet) — 18 constats

## Réseau & protocole

| ID | Sévérité | Catégorie | Titre | Emplacement | Vérif. |
|---|---|---|---|---|---|
| [NET-1](#net-1) | 🔴 critique | securite | Allocations sans borne pilotées par des compteurs reçus : crash du serveur à distance (Process noexcept) | `components/openmw-mp/Packets/Player/PacketPlayerInventory.cpp:27` | confirmé |
| [NET-2](#net-2) | 🔴 critique | securite | Protocole maître : compteurs signés non validés, crash ou blocage du serveur maître par une annonce | `components/openmw-mp/Master/ProxyMasterPacket.hpp:74` | confirmé |
| [NET-4](#net-4) | 🟠 haute | securite | Échecs de lecture ignorés : paquets tronqués traités, amplification d'environ 3000× via ObjectPacket | `components/openmw-mp/Packets/Object/ObjectPacket.cpp:35` | confirmé |
| [NET-5](#net-5) | 🟠 haute | securite | Broadcast RakNet vers des connexions non authentifiées et relais global sans filtre par cellule | `components/openmw-mp/Packets/BasePacket.cpp:66` | confirmé |
| [NET-6](#net-6) | 🟠 haute | bug | Poids et qualité des enregistrements dynamiques bornés à [-1,1] par WriteCompressed<float> | `components/openmw-mp/Packets/Worldstate/PacketRecordDynamic.cpp:453` | confirmé |
| [NET-8](#net-8) | 🟠 haute | performance | Tous les paquets en RELIABLE_ORDERED, y compris les positions : blocage en tête de file et retransmissions inutiles | `components/openmw-mp/Packets/Player/PacketPlayerPosition.cpp:10` | confirmé |
| [NET-3](#net-3) | 🟡 moyenne | bug | BitStream construit avec un octet de trop : lecture d'un octet hors du tampon sur chaque paquet reçu | `apps/openmw-mp/Networking.cpp:559` | partiel (sévérité corrigée) |
| [NET-7](#net-7) | 🟡 moyenne | bug | Concurrence non protégée : le thread MasterClient parcourt la table des joueurs sans verrou | `apps/openmw-mp/MasterClient.cpp:198` | partiel (sévérité corrigée) |
| [NET-9](#net-9) | 🟡 moyenne | performance | Paquet de position surdimensionné : GUID de 8 octets, floats bruts, 66,7 Hz | `components/openmw-mp/Packets/Player/PacketPlayerPosition.cpp:17` | non soumis |
| [NET-10](#net-10) | 🟡 moyenne | performance | sendToLoaded re-sérialise le paquet pour chaque destinataire et reconstruit une std::list à chaque envoi | `apps/openmw-mp/Cell.cpp:207` | non soumis |
| [NET-11](#net-11) | 🟡 moyenne | securite | Mot de passe serveur en clair sur le réseau, écrit dans les journaux, essais illimités | `apps/openmw-mp/Networking.cpp:121` | confirmé |
| [NET-13](#net-13) | 🟡 moyenne | architecture | Versionnage fragile : mot de passe RakNet = version + proto + hash de commit, IDs implicites sur uint8 | `apps/openmw/mwmp/Networking.cpp:286` | non soumis |
| [NET-14](#net-14) | 🟡 moyenne | tests | Aucun test ni fuzzing de la sérialisation des ~100 paquets | `apps/openmw_test_suite/CMakeLists.txt` | non soumis |
| [NET-12](#net-12) | ⚪ basse | bug | preInit continue après un premier paquet invalide (return manquant) | `apps/openmw-mp/Networking.cpp:238` | partiel (sévérité corrigée) |
| [NET-15](#net-15) | ⚪ basse | performance | Dispatch des paquets en parcours linéaire et GetPacket via operator[] (insertion d'entrées nulles) | `components/openmw-mp/Controllers/PlayerPacketController.cpp:115` | non soumis |
| [NET-16](#net-16) | ⚪ basse | bug | Chaînes de 65 536 octets transmises vides (repli de la longueur uint16) et tronquées au premier NUL | `components/openmw-mp/Packets/BasePacket.hpp:87` | non soumis |
| [NET-17](#net-17) | ⚪ basse | performance | Client : attente active pendant la connexion et RakSleep(500) pendant PreInit | `apps/openmw/mwmp/Networking.cpp:290` | non soumis |
| [NET-18](#net-18) | ⚪ basse | bug | MasterClient::SetUpdateRate borne la mauvaise variable | `apps/openmw-mp/MasterClient.cpp:252` | non soumis |

### NET-1

**Allocations sans borne pilotées par des compteurs reçus : crash du serveur à distance (Process noexcept)**

- **Sévérité** : 🔴 critique · **Catégorie** : securite · **Effort** : M
- **Emplacement** : `components/openmw-mp/Packets/Player/PacketPlayerInventory.cpp:27`

**Preuve**

> PacketPlayerInventory.cpp:17-27 : `uint32_t count; ... RW(count, send); ... player->inventoryChanges.items.resize(count);` (count non initialisé, retour de RW ignoré, aucune borne). apps/openmw-mp/processors/PlayerProcessor.cpp:9 : `bool PlayerProcessor::Process(RakNet::Packet &packet) noexcept`. Même motif dans 24 sites (Spellbook, Journal, Topic, Book, CellState, Faction, Ally, QuickKeys, Cooldowns, StatsDynamic, Skill, Attribute, Equipment, SpellsActive, WorldMap, KillCount, ClientScriptLocal/Global/Settings, CellReset, CollisionOverride, ActorSpellsActive).

**Impact** — Un client connecté envoie ID_LOADED puis un ID_PLAYER_INVENTORY d'environ 20 octets avec count=0xFFFFFFFF. resize() lève std::length_error ou std::bad_alloc dans une fonction noexcept, ce qui appelle std::terminate et fait tomber le serveur pour tous les joueurs. Sur un serveur sans mot de passe, n'importe qui peut le faire. Un paquet simplement tronqué donne une taille aléatoire, car count n'est pas initialisé. Dans l'autre sens, un serveur malveillant listé dans le navigateur peut faire planter les clients.

**Recommandation** — Ajouter dans BasePacket un helper `bool RWCount(uint32_t &count, uint32_t max, uint32_t minBitsPerElem)`. Il lit le compteur, vérifie `count <= max && count * minBitsPerElem <= bs->GetNumberOfUnreadBits()`, et sinon met packetValid=false et count=0. Remplacer les 24 sites par ce helper, avec des maxima par type (ex. 1000 objets d'inventaire, 3 index de stats dynamiques). Dans les 4 Process() du serveur, tester `myPacket->isPacketValid()` avant `Do()`, attraper les exceptions dans la fonction (try/catch interne) et expulser l'émetteur.

**Contre-vérification (confirmé)** — PacketPlayerInventory.cpp:17-27 : `uint32_t count;` n'est pas initialisé, le retour de RW est ignoré et `items.resize(count)` n'a aucune borne. La seule barrière avant l'appel est la séquence handshake puis ID_LOADED (Networking.cpp:142-185) ; après elle, PlayerProcessor::Process (PlayerProcessor.cpp:9, noexcept) appelle Read() sans try/catch. Avec count=0xFFFFFFFF, la demande est d'environ 4,3 G × sizeof(Item), soit plusieurs centaines de Go : bad_alloc dans une fonction noexcept, donc std::terminate. Le même motif existe dans 31 appels resize() sous components/openmw-mp/Packets (grep), dont PacketPreInit.cpp:76, accessible avant même l'authentification.

### NET-2

**Protocole maître : compteurs signés non validés, crash ou blocage du serveur maître par une annonce**

- **Sévérité** : 🔴 critique · **Catégorie** : securite · **Effort** : S
- **Emplacement** : `components/openmw-mp/Master/ProxyMasterPacket.hpp:74`

**Preuve**

> l.65-74 : `int32_t playersCount = ...; packet->RW(playersCount, send); if (playersCount > QueryData::maxPlayers) playersCount = 0; ... server.players.resize(playersCount);`. l.35 : `while (rulesSize--)`. apps/master/MasterServer.cpp:131 : `pma.Read();` sans try/catch. Version.hpp:8 : `#define TES3MP_MASTERSERVER_PASSW "12345"`.

**Impact** — N'importe quel hôte se connecte au maître avec le mot de passe RakNet public et envoie ID_MASTER_ANNOUNCE avec playersCount=-1. resize(SIZE_MAX) lève std::length_error, qui n'est pas rattrapée : le maître plante et la liste des serveurs disparaît pour toute la communauté. Avec rulesSize=-1, la boucle fait environ 4,3 milliards d'itérations (débordement signé, comportement indéfini) et le maître se fige. Le même code lit les réponses du maître dans le navigateur et le serveur. En plus, l.55 `packet->RW(rule->str, send, QueryData::maxStringLength)` passe 256 comme argument `compress` : la limite de 256 caractères n'est jamais appliquée.

**Recommandation** — Passer les compteurs en uint32_t et rejeter le paquet (marquer invalide, ignorer le serveur) quand `count > max`, au lieu de ramener la valeur à 0, ce qui désynchronise aussi le flux. Corriger l.55 en `packet->RW(rule->str, send, false, QueryData::maxStringLength)`. Entourer pma.Read() et pmq/pmu d'un try/catch dans MasterServer::Thread, puis fermer la connexion fautive.

**Contre-vérification (confirmé)** — ProxyMasterPacket.hpp:65-74 ne vérifie que la borne haute (`playersCount > maxPlayers`). Avec -1, on obtient `resize(SIZE_MAX)` sur un vector<string>, qui lève length_error. MasterServer.cpp:131 appelle `pma.Read()` dans le std::thread du maître sans aucun try/catch (grep : les seuls catch de apps/master sont dans RestServer et SimpleWeb), donc terminate. Le mot de passe RakNet est public (Version.hpp:8, MasterServer.cpp:24). `while (rulesSize--)` (l.35) avec -1 boucle jusqu'au débordement signé. À la l.55, 256 est bien passé comme argument `compress` de `RW(std::string&, bool, bool, size_type)` (BasePacket.hpp:89), donc maxSize garde sa valeur par défaut de 64 Kio.

### NET-4

**Échecs de lecture ignorés : paquets tronqués traités, amplification d'environ 3000× via ObjectPacket**

- **Sévérité** : 🟠 haute · **Catégorie** : securite · **Effort** : M
- **Emplacement** : `components/openmw-mp/Packets/Object/ObjectPacket.cpp:35`

**Preuve**

> l.34-43 : `BaseObject baseObject; for (unsigned int i = 0; i < objectList->baseObjectCount; i++) { ... Object(baseObject, send); if (!send) objectList->baseObjects.push_back(baseObject);`. apps/openmw-mp/processors/ObjectProcessor.cpp:11 : `packet.Send(true);` (Do par défaut, utilisé tel quel par ProcessorObjectMove). Mesure : 963 appels RW, 1 seul retour testé.

**Impact** — Un ID_OBJECT_MOVE d'environ 20 octets annonce baseObjectCount=3000 (le maximum) sans aucune donnée. Toutes les lectures échouent en silence et 3000 objets par défaut sont créés, avec refNum et mpNum non initialisés (BaseObject.hpp:30-31), donc des octets de pile du serveur sont relayés. Le serveur re-sérialise ensuite environ 21 octets par objet, soit environ 63 Ko, et les diffuse en fiable-ordonné à chaque joueur. L'amplification est d'environ 3000× par destinataire, sans aucun débit limite. Plus généralement, n'importe quel paquet tronqué est traité avec des valeurs partielles.

**Recommandation** — Faire retourner bool à Packet()/Object()/Actor() et sortir à la première lecture échouée (`if (!RW(...)) { packetValid = false; return; }`, ou un drapeau collant dans RW). Avant la boucle, vérifier `bs->GetNumberOfUnreadBits() >= count * bitsMinParObjet`. Initialiser BaseObject (refNum=0, mpNum=0). Rejeter un paquet qui laisse des bits non lus. Dans ObjectProcessor/ActorProcessor, ignorer le paquet si `!myPacket->isPacketValid()`.

**Contre-vérification (confirmé)** — ObjectPacket.cpp:62 borne baseObjectCount à 3000, mais la boucle des l.34-43 ignore les échecs de lecture. `BaseObject baseObject;` est déclaré une fois hors de la boucle, avec refNum et mpNum non initialisés (BaseObject.hpp:30-31), puis il est poussé 3000 fois. ProcessorObjectMove.hpp ne redéfinit pas Do, donc ObjectProcessor.cpp:11 `packet.Send(true)` re-sérialise refId+refNum+mpNum+pos (PacketObjectMove.cpp:13-15) et diffuse l'ensemble à tous les autres pairs. Le seul prérequis est l'état POSTLOADED (Networking.cpp:211), à la portée de tout client admis.

### NET-5

**Broadcast RakNet vers des connexions non authentifiées et relais global sans filtre par cellule**

- **Sévérité** : 🟠 haute · **Catégorie** : securite · **Effort** : M
- **Emplacement** : `components/openmw-mp/Packets/BasePacket.cpp:66`

**Preuve**

> BasePacket.cpp:66 : `return peer->Send(bsSend, priority, reliability, orderChannel, guid, toOther);`. Script/Functions/Chat.cpp:24 : `packet->Send(true);`. Processeurs ObjectMove/ObjectRotate/ObjectAnimPlay/MusicPlay/ScriptMemberShort sans surcharge de Do(), donc ObjectProcessor.cpp:11 `packet.Send(true);`. Player.cpp:88-105 : handshakeCounter n'avance qu'à la réception de paquets, sans délai d'expiration.

**Impact** — Un broadcast RakNet part vers tous les systèmes connectés, y compris ceux qui n'ont pas fini le handshake. Sur un serveur protégé par mot de passe, un client modifié passe PreInit (le serveur lui renvoie la liste de plugins attendue), n'envoie jamais ID_SYSTEM_HANDSHAKE et reste connecté indéfiniment. Il reçoit alors le chat global et les relais Object, ce qui annule la confidentialité promise par le mot de passe. Par ailleurs, 5 types de paquets Object sont relayés à tout le serveur quelle que soit la cellule, sans validation ni callback Lua.

**Recommandation** — Remplacer `Send(true)` côté serveur par une diffusion explicite sur les joueurs `isHandshaked() && getLoadState()==Player::POSTLOADED`, en sérialisant une seule fois. Ajouter un délai d'expiration du handshake (ex. 10 s, horodatage dans Player vérifié dans mainLoop, puis kickPlayer). Surcharger Do() des 5 processeurs pour passer par `Cell::sendToLoaded` avec un contrôle d'autorité, comme ProcessorActorPosition.

**Contre-vérification (confirmé)** — BasePacket.cpp:66 appelle `peer->Send(..., guid, toOther)` avec broadcast=true. CrabNet RakPeer.cpp:4173 envoie alors à tout remoteSystem `isActive`, sans notion de handshake TES3MP. Le mot de passe RakNet du serveur vaut seulement version+commit (main.cpp:222-229), il est donc public. Le compteur de handshake n'augmente qu'à la réception d'un paquet joueur (Networking.cpp:142-153) et aucun délai d'expiration n'existe. Une connexion qui ne finit jamais le handshake, ou n'envoie même pas PreInit, reçoit donc le chat (Chat.cpp:24) et les relais Object. Les cinq processeurs cités (ProcessorObjectMove, Rotate, AnimPlay, MusicPlay, ScriptMemberShort) n'ont pas de Do propre.

### NET-6

**Poids et qualité des enregistrements dynamiques bornés à [-1,1] par WriteCompressed<float>**

- **Sévérité** : 🟠 haute · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `components/openmw-mp/Packets/Worldstate/PacketRecordDynamic.cpp:453`

**Preuve**

> l.453 : `RW(recordData.mData.mWeight, send, true);` et l.455 `RW(recordData.mData.mQuality, send, true);` (idem l.698, 734, 736, 820, 822, 851, 853). CrabNet BitStream.h:1270-1280 : `/// For values between -1 and 1 ... if (varCopy > 1.0f) varCopy = 1.0f; Write((uint16_t) ((varCopy + 1.0f) * 32767.5f));`

**Impact** — Neuf champs float (mWeight et mQuality des types Apparatus, Light, Lockpick, Probe, Repair ; esm/loadappa.hpp:33-34, loadligh.hpp:38, etc.) sont bornés à 1,0 et quantifiés sur 16 bits. Un objet personnalisé créé par le serveur avec un poids de 3 ou une qualité de 1,5 arrive chez tous les clients avec 1,0. En build Debug, RakAssert interrompt le processus. Les autres types (armures, armes : l.184-382) sérialisent mWeight sans compression, d'où l'incohérence.

**Recommandation** — Retirer `, true` sur ces 9 lignes. Mieux : ajouter dans BasePacket une surcharge `bool RW(float &data, bool write, bool compress = false)` qui ignore `compress`, pour empêcher toute réapparition. Ajouter un test aller-retour par type d'enregistrement avec des valeurs supérieures à 1.

**Contre-vérification (confirmé)** — PacketRecordDynamic.cpp:453/455, 698, 734/736, 820/822 et 851/853 utilisent `RW(..., send, true)` sur des float (esm/loadappa.hpp:33-34, loadligh.hpp:38, loadlock.hpp:20/23, loadprob.hpp:20/23, loadrepa.hpp:20/24). Dans CrabNet BitStream.h:1272-1280, la spécialisation WriteCompressed<float> borne à [-1,1] avec un RakAssert. Les armes et armures (l.184-382) écrivent mWeight sans compression. La corruption des valeurs supérieures à 1 est systématique dès que le serveur envoie ces types d'enregistrements.

### NET-8

**Tous les paquets en RELIABLE_ORDERED, y compris les positions : blocage en tête de file et retransmissions inutiles**

- **Sévérité** : 🟠 haute · **Catégorie** : performance · **Effort** : M
- **Emplacement** : `components/openmw-mp/Packets/Player/PacketPlayerPosition.cpp:10`

**Preuve**

> l.9-10 : `priority = MEDIUM_PRIORITY; //reliability = UNRELIABLE_SEQUENCED;`. PlayerPacket.cpp:12-13 : `reliability = RELIABLE_ORDERED; orderChannel = CHANNEL_PLAYER;`. `grep -rn UNRELIABLE` ne renvoie que cette ligne commentée.

**Impact** — Les positions envoyées à 66,7 Hz (joueur) et 40 Hz (acteurs) sont retransmises même quand elles sont périmées. ID_PLAYER_POSITION partage CHANNEL_PLAYER avec la quarantaine d'autres paquets joueur (inventaire, équipement, stats…). Une seule perte de datagramme bloque tout le canal jusqu'à la retransmission (au moins un RTT). Résultat en cas de perte : saccades élastiques, rafales de rattrapage et hausse de la bande passante.

**Recommandation** — Passer ID_PLAYER_POSITION, ID_ACTOR_POSITION, ID_PLAYER_ANIM_FLAGS, ID_ACTOR_ANIM_FLAGS et ID_PLAYER_MOMENTUM en UNRELIABLE_SEQUENCED sur des canaux dédiés (ex. CHANNEL_PLAYER_MOVE=6, CHANNEL_ACTOR_MOVE=7). Garder un envoi fiable de la position finale à l'arrêt (la logique posWasChanged de LocalPlayer.cpp:389-401 le permet). Mesurer avant et après avec `peer->GetStatistics()` (octets retransmis, perte) sous `ApplyNetworkSimulator(0.02f, 100, 20)`.

**Contre-vérification (confirmé)** — PacketPlayerPosition.cpp:9-10 laisse `//reliability = UNRELIABLE_SEQUENCED;` en commentaire, et le paquet hérite donc de RELIABLE_ORDERED/CHANNEL_PLAYER (PlayerPacket.cpp:12-13). grep confirme qu'aucun paquet n'utilise UNRELIABLE. Les périodes d'envoi sont bien de 0,015 s (LocalPlayer.cpp:104) et 0,025 s (Cell.cpp:37). Les positions partagent l'ordonnancement du canal avec tous les paquets joueur, si bien qu'une perte bloque le canal jusqu'à la retransmission.

### NET-10

**sendToLoaded re-sérialise le paquet pour chaque destinataire et reconstruit une std::list à chaque envoi**

- **Sévérité** : 🟡 moyenne · **Catégorie** : performance · **Effort** : M
- **Emplacement** : `apps/openmw-mp/Cell.cpp:207`

**Preuve**

> Cell.cpp:189-207 : `std::list <Player*> plList; ... plList.sort(); plList.unique(); for (auto pl : plList) { ... actorPacket->Send(pl->guid);`. BasePacket::Send(AddressOrGUID) (BasePacket.cpp:56-59) refait `ResetWritePointer(); Packet(bsSend, true);` à chaque appel. Même motif dans Player.cpp:132-148.

**Impact** — Pour un lot d'acteurs (jusqu'à 3000 acteurs, encodage Huffman des chaînes compris) et N joueurs dans la cellule, le thread unique du serveur ré-encode N fois. Il alloue, trie et dédoublonne aussi une liste chaînée à chaque paquet, 40 fois par seconde et par cellule active. Le coût CPU est en O(N × taille) et devient le goulot du serveur quand les joueurs se regroupent.

**Recommandation** — Sérialiser une seule fois dans un BitStream local (`packet->Packet(&bs, true)`), puis appeler `peer->Send(&bs, priority, reliability, channel, guid, false)` pour chaque destinataire. Ajouter `BasePacket::SendTo(const std::vector<RakNetGUID>&)`. Tenir dans Cell un std::vector de destinataires mis à jour à l'entrée et à la sortie de cellule, au lieu de reconstruire list+sort+unique.

### NET-11

**Mot de passe serveur en clair sur le réseau, écrit dans les journaux, essais illimités**

- **Sévérité** : 🟡 moyenne · **Catégorie** : securite · **Effort** : M
- **Emplacement** : `apps/openmw-mp/Networking.cpp:121`

**Preuve**

> l.121-122 : `LOG_MESSAGE_SIMPLE(TimedLog::LOG_WARN, "Wrong server password %s used by client at %s", baseSystem.serverPassword.c_str(), ...)`. PacketSystemHandshake.cpp:17 : `RW(system->serverPassword, send, true, maxPasswordLength)` (codage Huffman, pas un chiffrement). CrabNet NativeFeatureIncludes.h:55 : `LIBCAT_SECURITY 0`. Aucun appel à InitializeSecurity ni à SetLimitIPConnectionFrequency.

**Impact** — Tout observateur du trafic UDP lit le mot de passe. Les mots de passe erronés, souvent réutilisés ailleurs, se retrouvent dans les journaux. Comme une erreur entraîne seulement une expulsion suivie d'une reconnexion immédiate, rien n'empêche la force brute.

**Recommandation** — Ne plus journaliser la valeur du mot de passe. Appeler `peer->SetLimitIPConnectionFrequency(true)` dans apps/openmw-mp/main.cpp après Startup, et bannir temporairement une IP après K échecs. Remplacer l'envoi du mot de passe par un défi-réponse : nonce serveur dans la réponse PreInit, le client renvoie HMAC-SHA256(motdepasse, nonce) (PicoSHA2 est déjà dans extern/). À terme, chiffrer le transport.

**Contre-vérification (confirmé)** — Networking.cpp:121-122 écrit dans les journaux le mot de passe erroné en clair. PacketSystemHandshake.cpp:17 ne fait qu'un codage Huffman (compress=true) et CrabNet NativeFeatureIncludes.h:55 fixe `LIBCAT_SECURITY 0`. Aucun appel à InitializeSecurity ni à SetLimitIPConnectionFrequency n'apparaît dans apps/ ou components/ (grep). Une erreur provoque seulement un kickPlayer, et la reconnexion est immédiate.

### NET-13

**Versionnage fragile : mot de passe RakNet = version + proto + hash de commit, IDs implicites sur uint8**

- **Sévérité** : 🟡 moyenne · **Catégorie** : architecture · **Effort** : M
- **Emplacement** : `apps/openmw/mwmp/Networking.cpp:286`

**Preuve**

> l.279-286 : `sstr << TES3MP_VERSION; sstr << TES3MP_PROTO_VERSION; ... sstr << commitHashString; if (peer->Connect(..., sstr.str().c_str(), ...)`. l.306 : `errmsg = "Version mismatch!\nYour client is on version " TES3MP_VERSION`. NetworkMessages.hpp:8 : `_ID_UNUSED = ID_USER_PACKET_ENUM+1,` (énumération implicite).

**Impact** — La compatibilité dépend du hash de commit exact : ce fork, qui a ses propres commits, ne peut pas rejoindre un serveur 0.8.1 officiel même avec un protocole identique. Le client ne connaît jamais la version du serveur et affiche un message générique. Les IDs sont implicites : ID_PLACEHOLDER vaut 234, il ne reste que 21 valeurs, et ID_MASTER_ANNOUNCE (136) partage la valeur de ID_USER_MYID. Insérer un ID au milieu décale silencieusement tous les suivants.

**Recommandation** — Réduire le mot de passe RakNet à TES3MP_PROTO_VERSION seul. Transporter la version, le hash et un champ de capacités (bits) dans PacketPreInit, dans les deux sens, pour afficher une erreur précise. Fixer explicitement la valeur numérique de chaque ID et ajouter `static_assert(ID_PLACEHOLDER <= 255)`. Réserver un ID d'extension (ID_EXTENDED suivi d'un uint16) avant la saturation.

### NET-14

**Aucun test ni fuzzing de la sérialisation des ~100 paquets**

- **Sévérité** : 🟡 moyenne · **Catégorie** : tests · **Effort** : M
- **Emplacement** : `apps/openmw_test_suite/CMakeLists.txt`

**Preuve**

> `grep -rln "openmw-mp\|mwmp\|BitStream" apps/openmw_test_suite | wc -l` renvoie 0. Aucune cible de fuzzing dans les CMakeLists.txt hors extern.

**Impact** — Les fonctions RW lecture/écriture, écrites à la main, ne sont jamais vérifiées. Les défauts NET-1, NET-3, NET-4 et NET-6 auraient été détectés immédiatement par un test aller-retour ou par quelques minutes de fuzzing sous ASan/UBSan. Toute évolution du protocole se fait donc à l'aveugle.

**Recommandation** — Créer apps/openmw_test_suite/mwmp/ avec, pour chaque paquet de chaque contrôleur, un test qui remplit un BasePlayer, BaseObjectList, BaseActorList ou BaseWorldstate, écrit, relit, puis compare les valeurs et vérifie qu'aucun bit ne reste. Ajouter une cible libFuzzer `fuzz_mwmp_packets` (-fsanitize=fuzzer,address,undefined) qui passe des octets aléatoires à `Packet(&bs,false)` pour chaque ID, et l'exécuter en CI pendant 60 s.

### NET-3

**BitStream construit avec un octet de trop : lecture d'un octet hors du tampon sur chaque paquet reçu**

- **Sévérité** : 🟡 moyenne (initialement haute) · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw-mp/Networking.cpp:559`

**Preuve**

> `RakNet::BitStream bsIn(&packet->data[1], packet->length, false);`, présent aussi dans apps/openmw/mwmp/processors/{Player,Actor,Object,System,Worldstate}Processor.cpp:17-18 et apps/browser/netutils/Utils.cpp:44. CrabNet BitStream.cpp:84 : `numberOfBitsUsed = lengthInBytes << 3;`.

**Impact** — Le flux annonce `length` octets à partir de data+1, alors que seuls length-1 octets existent. Les contrôles de bornes de BitStream (ReadBits : `readOffset + n > numberOfBitsUsed`) autorisent donc la lecture d'un octet hors du tas, ce qu'un émetteur peut provoquer à volonté (chaîne ou valeur qui finit sur la dernière position). AddressSanitizer le signalerait comme heap-buffer-overflow. Le calcul de taille de PacketPreInit (headerSize inclut l'octet d'ID) dépend implicitement de ce décalage.

**Recommandation** — Aux 7 endroits, vérifier `packet->length >= 1 + RakNetGUID::size()`, puis construire `BitStream bsIn(packet->data, packet->length, false); bsIn.IgnoreBytes(1);`. Adapter PacketPreInit::Packet (comparer à packetSize + 1 ou utiliser GetNumberOfUnreadBits). Valider la correction par un test ASan.

**Contre-vérification (partiel (sévérité corrigée))** — C'est réel. Networking.cpp:559 et les cinq processeurs client (ActorProcessor.cpp:17, etc.) construisent `BitStream(&data[1], length, false)`, et CrabNet BitStream.cpp:84 fixe `numberOfBitsUsed = lengthInBytes << 3`. RakPeer.cpp:163 alloue exactement `malloc(dataSize)`, ce qui rend la lecture hors bornes d'un octet possible. En pratique, cet octet tombe presque toujours dans la marge d'alignement du bloc malloc : on obtient au plus un octet de tas dans une valeur lue, sans crash ni exploitation crédible. Le bug est à corriger (détectable par ASan), mais « haute » est exagéré.

### NET-7

**Concurrence non protégée : le thread MasterClient parcourt la table des joueurs sans verrou**

- **Sévérité** : 🟡 moyenne (initialement haute) · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw-mp/MasterClient.cpp:198`

**Preuve**

> l.193-198 : `auto *players = Players::getPlayers(); while (sRun) { SetPlayers((int) players->size()); auto pIt = players->begin();`, puis l.208 `pIt->second->npc.mName`. Pendant ce temps, Player.cpp:19 `players.erase(guid);` et :27 `players[guid] = new Player(guid);` s'exécutent dans le thread principal, sans mutex. `static bool sRun` n'est pas atomique.

**Impact** — Le thread d'annonce parcourt std::map<RakNetGUID, Player*> et lit npc.mName pendant que la boucle principale ajoute ou supprime des joueurs à chaque connexion ou déconnexion. Cela invalide les itérateurs et peut faire lire un Player déjà détruit : comportement indéfini, crash aléatoire d'autant plus probable que le serveur est fréquenté. queryData.players est aussi modifié hors de mutexData.

**Recommandation** — Ne plus lire Players depuis ce thread. Le thread principal publie un instantané (nombre de joueurs et noms) via une méthode `MasterClient::SetPlayerNames(std::vector<std::string>)` protégée par mutexData, appelée dans newPlayer, disconnectPlayer et à la réception de ID_PLAYER_BASEINFO. Déclarer `std::atomic<bool> sRun`. Tenir mutexData pendant toute la préparation du paquet dans Thread().

**Contre-vérification (partiel (sévérité corrigée))** — La course est réelle. MasterClient.cpp:193-221 parcourt `Players::getPlayers()`, qui renvoie la map statique (Player.cpp:57-60), et lit `npc.mName` sans verrou, pendant que le thread principal fait `players.erase`/`players[guid] = new` (Player.cpp:19-27). `static bool sRun` n'est pas atomique (MasterClient.hpp:40). En revanche, la boucle ne tourne que toutes les `timeout` ms (15000 par défaut, MasterClient.cpp:21, minimum 1000) et la fenêtre critique dure quelques microsecondes. Le comportement est indéfini, mais un crash est peu probable : sévérité moyenne plutôt que haute.

### NET-9

**Paquet de position surdimensionné : GUID de 8 octets, floats bruts, 66,7 Hz**

- **Sévérité** : 🟡 moyenne · **Catégorie** : performance · **Effort** : L
- **Emplacement** : `components/openmw-mp/Packets/Player/PacketPlayerPosition.cpp:17`

**Preuve**

> l.17-18 : `RW(player->position, send, 1); RW(player->direction, send, 1);`. BasePacket.cpp:24-25 : `bs->Write(packetID); bs->Write(guid);`. LocalPlayer.cpp:104 : `const float timeoutSec = 0.015;`. Networking.cpp:560 (serveur) : `bsIn.IgnoreBytes(... RakNetGUID::size()); // Ignore GUID`.

**Impact** — Le WriteCompressed générique de CrabNet (BitStream.cpp:467-478) ne supprime que les octets de poids fort nuls. Pour 6 floats où le lacet n'est presque jamais nul, il ne gagne rien et ajoute un bit qui désaligne le flux. Le paquet fait environ 58 octets (1 + 8 + ~25 + ≤25) à 66,7 Hz, soit environ 3,9 Ko/s par couple émetteur/récepteur. Avec 32 joueurs dans une cellule, cela donne environ 3,8 Mo/s (≈ 31 Mbit/s) de sortie serveur pour les seules positions, hors en-têtes RakNet/UDP. Les 8 octets de GUID envoyés par le client sont jetés par le serveur.

**Recommandation** — (1) Supprimer le GUID des paquets client→serveur ; côté serveur→client, le remplacer par l'identifiant de slot sur 16 bits (Player::getId). (2) Quantifier : position en int32 en point fixe au 1/8 d'unité, ou int24 relatif à l'origine de la cellule ; lacet et tangage en uint16 ; roulis omis ; direction sur 2 bits par axe. (3) Envoyer à 20 Hz avec un numéro de séquence sur 16 bits, et ajouter un tampon d'interpolation d'environ 100 ms dans DedicatedPlayer::move. Cible : environ 18 octets à 20 Hz, soit environ 0,36 Ko/s par couple, à peu près 10× moins. Valider avec GetStatistics.

### NET-12

**preInit continue après un premier paquet invalide (return manquant)**

- **Sévérité** : ⚪ basse (initialement moyenne) · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw-mp/Networking.cpp:238`

**Preuve**

> l.234-238 : `if (packet->data[0] != ID_GAME_PREINIT) { LOG_MESSAGE_SIMPLE(...); peer->CloseConnection(packet->systemAddress, true); }`, suivi sans return de l.244-247 : `PacketPreInit packetPreInit(peer); ... packetPreInit.Read();`

**Impact** — Un premier paquet d'un autre type est quand même décodé comme PreInit. S'il passe le contrôle, ou si la vérification des fichiers de données est désactivée, Players::newPlayer() est appelé et ID_SYSTEM_HANDSHAKE est demandé à une connexion déjà en cours de fermeture. L'état devient incohérent et un slot reste occupé jusqu'à la notification de déconnexion.

**Recommandation** — Ajouter `return false;` juste après `peer->CloseConnection(...)` à la ligne 238.

**Contre-vérification (partiel (sévérité corrigée))** — Le return manque bien après `CloseConnection` (Networking.cpp:234-238), et la lecture PreInit se poursuit aux l.244-247. Cependant, un paquet d'un autre type échoue presque toujours au test `!isPacketValid() || dataFiles.empty()` (l.249) et la fonction renvoie false. Un attaquant capable de forger un corps PreInit valide peut aussi bien envoyer un vrai PreInit, et le Player créé est supprimé à la notification de déconnexion (Networking.cpp:548-553). Il s'agit d'une incohérence d'état mineure, pas d'un bug moyen.

### NET-15

**Dispatch des paquets en parcours linéaire et GetPacket via operator[] (insertion d'entrées nulles)**

- **Sévérité** : ⚪ basse · **Catégorie** : performance · **Effort** : S
- **Emplacement** : `components/openmw-mp/Controllers/PlayerPacketController.cpp:115`

**Preuve**

> l.115-119 : `bool ...ContainsPacket(RakNet::MessageID id) { for(const auto &packet : packets) { if (packet.first == id)`. l.106 : `return packets[(unsigned char)id].get();`. l.111-112 : SetStream appelle `packet.second->SetStreams(...)` sur tous les paquets.

**Impact** — Pour chaque paquet reçu, le serveur peut parcourir linéairement jusqu'à 5 contrôleurs (1+43+15+25+11 entrées), appeler SetStream sur tous les paquets du contrôleur, puis parcourir linéairement les processeurs (PlayerProcessor.cpp:11). Le coût reste modéré, mais il s'applique à chaque paquet. Un GetPacket sur un ID inconnu insère un unique_ptr nul que SetStream déréférencerait ensuite.

**Recommandation** — Remplacer les cinq maps par une table `std::array<BasePacket*, 256>` indexée par MessageID qui donne aussi la famille de paquet. Utiliser `find()` au lieu de operator[] et des boucles. N'appeler SetReadStream que sur le paquet ciblé. Faire de même pour `processors`.

### NET-16

**Chaînes de 65 536 octets transmises vides (repli de la longueur uint16) et tronquées au premier NUL**

- **Sévérité** : ⚪ basse · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `components/openmw-mp/Packets/BasePacket.hpp:87`

**Preuve**

> l.87 : `const static uint32_t maxStrSize = 64 * 1024; // 64 KiB` et l.99 `rstr.AppendBytes(str.c_str(), str.size() > maxSize ? maxSize : str.size());`. CrabNet RakString.cpp:1194 : `auto l = (unsigned short) strlen(str);`

**Impact** — Une chaîne d'au moins 65 536 octets (commande console, texte ou script fourni par Lua) est tronquée à 65 536, puis sa longueur se replie à 0 sur uint16 : le récepteur reçoit une chaîne vide. strlen() coupe aussi au premier octet nul. Les données sont perdues sans aucun journal.

**Recommandation** — Fixer maxStrSize à 65535 et journaliser toute troncature. Pour les gros contenus (scripts, records), ajouter un type blob avec longueur uint32 bornée explicitement, écrit via WriteAlignedBytes.

### NET-17

**Client : attente active pendant la connexion et RakSleep(500) pendant PreInit**

- **Sévérité** : ⚪ basse · **Catégorie** : performance · **Effort** : S
- **Emplacement** : `apps/openmw/mwmp/Networking.cpp:290`

**Preuve**

> l.290-292 : `while (queue) { for (RakNet::Packet *packet = peer->Receive(); packet; ...)` sans aucune pause. l.388-391 : `if (!packet) { RakSleep(500); continue; }`

**Impact** — La boucle de connexion occupe 100 % d'un cœur pendant toute la tentative (3 essais de 500 ms). L'attente de la réponse PreInit ajoute jusqu'à 500 ms de latence. Aucun délai maximal propre : si le serveur ne répond jamais au PreInit, on dépend du timeout RakNet.

**Recommandation** — Ajouter `RakSleep(5)` dans la boucle de connexion. Remplacer RakSleep(500) par RakSleep(5) dans preInit. Ajouter un délai maximal explicite (ex. 10 s) avec un message d'erreur clair.

### NET-18

**MasterClient::SetUpdateRate borne la mauvaise variable**

- **Sévérité** : ⚪ basse · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw-mp/MasterClient.cpp:252`

**Preuve**

> l.252-256 : `if (timeout < min_rate) timeout = min_rate; else if (timeout > max_rate) timeout = max_rate; timeout = rate;`

**Impact** — Les bornes min_rate et max_rate ne s'appliquent jamais à `rate`. Quand le maître répond FUNCTION_DELETE, Process() abaisse timeout de step_rate sans plancher réel. De plus, timeout est écrit par le thread principal et lu par le thread d'annonce sans synchronisation.

**Recommandation** — Écrire `timeout = std::clamp(rate, min_rate, max_rate);` et déclarer `timeout` en `std::atomic<unsigned>`.

## Serveur dédié (cœur)

| ID | Sévérité | Catégorie | Titre | Emplacement | Vérif. |
|---|---|---|---|---|---|
| [SRV-1](#srv-1) | 🔴 critique | securite | Lecture hors-bornes dans preInit déclenchable à distance avant authentification | `apps/openmw-mp/Networking.cpp:261` | confirmé |
| [SRV-5](#srv-5) | 🔴 critique | securite | resize(count) non borné dans 16 lecteurs de paquets joueurs | `components/openmw-mp/Packets/Player/PacketPlayerSpellbook.cpp:27` | partiel (sévérité corrigée) |
| [SRV-3](#srv-3) | 🟠 haute | performance | Cell::readActorList en O(n²) avec copies de BaseActor (2760 o) sur la boucle principale | `apps/openmw-mp/Cell.cpp:112` | confirmé |
| [SRV-2](#srv-2) | 🟡 moyenne | bug | Course de données entre le thread MasterClient et la boucle principale sur la map des joueurs | `apps/openmw-mp/MasterClient.cpp:206` | partiel (sévérité corrigée) |
| [SRV-4](#srv-4) | 🟡 moyenne | bug | Macro GET_PLAYER teste une variable codée en dur "player" au lieu de son argument | `apps/openmw-mp/Script/ScriptFunctions.hpp:37` | partiel (sévérité corrigée) |
| [SRV-7](#srv-7) | 🟡 moyenne | bug | preInit continue le traitement après un mauvais premier paquet (return manquant) | `apps/openmw-mp/Networking.cpp:234` | confirmé |
| [SRV-8](#srv-8) | 🟡 moyenne | performance | Boucle principale à sleep fixe 1 ms + sondage clavier, non adaptative | `apps/openmw-mp/Networking.cpp:572` | non soumis |
| [SRV-14](#srv-14) | 🟡 moyenne | securite | Aucun contrôle de flux / anti-flood par joueur dans le dispatch | `apps/openmw-mp/Networking.cpp:305` | confirmé |
| [SRV-6](#srv-6) | ⚪ basse | bug | Console d'administration morte : handleInput.cpp non compilé et jamais appelé | `apps/openmw-mp/handleInput.cpp:9` | partiel (sévérité corrigée) |
| [SRV-9](#srv-9) | ⚪ basse | bug | killLoop est un bool global non atomique écrit depuis un gestionnaire de signal | `apps/openmw-mp/Networking.cpp:35` | non soumis |
| [SRV-10](#srv-10) | ⚪ basse | bug | getLastPlayerId déréférence slots.rbegin() sans vérifier la vacuité | `apps/openmw-mp/Player.cpp:64` | non soumis |
| [SRV-11](#srv-11) | ⚪ basse | bug | MasterClient::SetUpdateRate borne la mauvaise variable puis l'écrase | `apps/openmw-mp/MasterClient.cpp:250` | non soumis |
| [SRV-12](#srv-12) | ⚪ basse | bug | Player::id non initialisé ; pas de garde si aucun slot libre | `apps/openmw-mp/Player.cpp:171` | non soumis |
| [SRV-13](#srv-13) | ⚪ basse | maintenabilite | ≈90 processeurs quasi identiques : boilerplate massif à faible valeur | `apps/openmw-mp/processors/object/ProcessorObjectSound.hpp:16` | non soumis |

### SRV-1

**Lecture hors-bornes dans preInit déclenchable à distance avant authentification**

- **Sévérité** : 🔴 critique · **Catégorie** : securite · **Effort** : S
- **Emplacement** : `apps/openmw-mp/Networking.cpp:261`

**Preuve**

> LOG_APPEND(..., dataFile->second[0], dataFile->first.c_str()); — dataFile->second est la liste de hash du client. PacketPreInit::Read (components/openmw-mp/Packets/PacketPreInit.cpp:83-85) fait checksum.second.resize(numberOfHashesIt->hashN) et n'interdit que hashN > maxHashes ; hashN=0 est accepté, donc second est un vector vide.

**Impact** — Un client (ou un paquet forgé) qui envoie autant de fichiers de données que le serveur en exige (samples.size()==dataFiles.size(), cas du config par défaut : 3 ESM) mais avec une liste de hash vide provoque un operator[] sur un vector vide (pointeur interne nul) → déréférencement nul / segfault. Crash distant déclenchable AVANT le handshake et avant toute authentification : déni de service trivial du serveur.

**Recommandation** — Dans PacketPreInit::Packet, rejeter nas.hashN == 0 (packetValid=false) au même endroit que le test > maxHashes (PacketPreInit.cpp:57). Et/ou dans Networking.cpp:261 ne journaliser dataFile->second[0] que si !dataFile->second.empty(). Corriger les deux pour défense en profondeur.

**Contre-vérification (confirmé)** — PacketPreInit.cpp:53-59 ne rejette que hashN > maxHashes, donc hashN=0 passe et checksum.second.resize(0) (l.83) produit un vector vide. Networking.cpp:261 (et l.269 find(..., dataFile->second[0])) fait alors operator[] sur ce vector vide, dès que samples.size()==dataFiles.size(). preInit est appelé pour tout GUID inconnu (Networking.cpp:566), donc avant handshake et sans authentification, et aucun garde-fou n'existe entre les deux.

### SRV-5

**resize(count) non borné dans 16 lecteurs de paquets joueurs**

- **Sévérité** : 🔴 critique (initialement moyenne) · **Catégorie** : securite · **Effort** : M
- **Emplacement** : `components/openmw-mp/Packets/Player/PacketPlayerSpellbook.cpp:27`

**Preuve**

> uint32_t count; RW(count, send); ... player->spellbookChanges.spells.clear(); player->spellbookChanges.spells.resize(count); — count vient du réseau, aucun maximum. Même motif dans 16 fichiers (grep resize(count) : Skill, StatsDynamic, Equipment, Faction, Book, Cooldowns, QuickKeys...).

**Impact** — Un client malveillant envoie un count énorme (jusqu'à 2^32-1) : resize tente une allocation gigantesque → std::bad_alloc (non rattrapé dans le processeur) ou amplification mémoire/CPU. Déni de service après authentification. À la différence de maxActors=3000 et des index bornés à 8, ces champs n'ont pas de garde.

**Recommandation** — Définir une borne par type (ex. maxSpells, maxItems) et, après RW(count), si count > borne mettre packetValid=false et retourner — le processeur rejette alors via le test isValid/isPacketValid existant. Factoriser un helper readBoundedCount.

**Contre-vérification (partiel (sévérité corrigée))** — Le motif est confirmé : PacketPlayerSpellbook.cpp:27-28 fait resize(count) avec un count lu sur le réseau, sans maximum, et le motif apparaît dans 16 fichiers. L'impact est sous-estimé : PlayerProcessor::Process est déclaré noexcept (PlayerProcessor.cpp:9) et appelle myPacket->Read() (l.20). Un std::bad_alloc ou length_error provoque donc std::terminate, soit un crash immédiat du serveur. Le paquet est traité dès l'état POSTLOADED (Networking.cpp:180-191), après un simple handshake sans mot de passe sur un serveur public, donc bien avant le login Lua.

### SRV-3

**Cell::readActorList en O(n²) avec copies de BaseActor (2760 o) sur la boucle principale**

- **Sévérité** : 🟠 haute · **Catégorie** : performance · **Effort** : M
- **Emplacement** : `apps/openmw-mp/Cell.cpp:112`

**Preuve**

> containsActor fait pour chaque acteur reçu : mwmp::BaseActor actor = cellActorList.baseActors.at(i); (copie complète de 2760 octets) dans une boucle sur tous les acteurs de la cellule, puis getActor refait la même boucle. readActorList les appelle pour chaque acteur du paquet entrant (Cell.cpp:84-86).

**Impact** — Benchmark reproduisant ce code : 500 acteurs dans une cellule = 101.6 ms par paquet ID_ACTOR_POSITION, exécuté sur le thread unique du serveur, gelant toute la boucle (réception, autres joueurs, timers). Une cellule très peuplée (ville, horde d'invocations) ou un client envoyant de grandes listes dégrade massivement la latence de tous les joueurs.

**Recommandation** — Remplacer la recherche linéaire par un index std::unordered_map<clé(refNum,mpNum), size_t> dans Cell, et prendre les acteurs par référence (const auto&) au lieu de les copier dans containsActor/getActor/removeActors. Gain mesuré : de ~100 ms à <1 ms pour 500 acteurs.

**Contre-vérification (confirmé)** — Cell.cpp:112 copie bien chaque BaseActor par valeur (mwmp::BaseActor actor = ...at(i)) dans containsActor, puis getActor (l.122) refait le parcours. readActorList (l.76-80) le fait pour chaque acteur reçu, et ce chemin est atteint par ProcessorActorPosition.hpp:23 et ProcessorActorStatsDynamic. Le seul garde-fou est la vérification que l'émetteur est l'autorité de cellule (ProcessorActorPosition.hpp:21) ; il ne limite ni le coût ni la croissance de cellActorList, car les refNum inconnus sont ajoutés par push_back (l.104). Le chiffre de 101,6 ms n'a pas été reproduit ici, mais la complexité O(n²) avec copies est avérée.

### SRV-14

**Aucun contrôle de flux / anti-flood par joueur dans le dispatch**

- **Sévérité** : 🟡 moyenne · **Catégorie** : securite · **Effort** : L
- **Emplacement** : `apps/openmw-mp/Networking.cpp:305`

**Preuve**

> update() route inconditionnellement chaque paquet vers le PacketController puis le processeur (et donc les callbacks Lua). Aucun compteur de débit, taille cumulée ou quota par GUID ; seul le handshake a un compteur de tentatives (processPlayerPacket l.145-152).

**Impact** — Un joueur authentifié peut inonder le serveur (ID_PLAYER_POSITION, ID_CHAT_MESSAGE, grandes listes d'acteurs/objets). Combiné à SRV-3 et SRV-5, un seul client peut monopoliser la boucle mono-thread et dégrader tous les autres (déni de service applicatif).

**Recommandation** — Ajouter un limiteur par joueur (jetons/intervalle minimal par type de paquet, et/ou octets/s) avant le dispatch dans update(), avec seuils configurables et déconnexion/kick au dépassement. Exposer des compteurs aux scripts pour politique fine.

**Contre-vérification (confirmé)** — Networking::update (l.305-330) route chaque paquet sans quota. Le seul compteur concerne les tentatives de handshake (l.145-152). grep ne trouve aucun SetLimitIPConnectionFrequency, limite de débit ou anti-flood dans apps/openmw-mp ni dans components/openmw-mp ; seul SetMaximumIncomingConnections existe (main.cpp:266). La boucle principale est mono-thread (mainLoop l.517-570).

### SRV-2

**Course de données entre le thread MasterClient et la boucle principale sur la map des joueurs**

- **Sévérité** : 🟡 moyenne (initialement haute) · **Catégorie** : bug · **Effort** : M
- **Emplacement** : `apps/openmw-mp/MasterClient.cpp:206`

**Preuve**

> MasterClient::Thread fait auto *players = Players::getPlayers(); puis while(sRun){... for (int i=0; pIt != players->end(); i++, pIt++) if (queryData.players[i] != pIt->second->npc.mName) ... for (auto player : *players) ... player.second->npc.mName ...}. Le mutexData ne protège que queryData (SetPlayers etc.), jamais le conteneur players.

**Impact** — Pendant que ce thread itère la map et lit pIt->second->npc.mName, la boucle principale appelle Players::newPlayer (players[guid]=new, Player.cpp:131) et Players::deletePlayer (delete + players.erase, Player.cpp:122-123). Itération concurrente + mutation de std::map et lecture d'un Player détruit = use-after-free / corruption → crash non déterministe. Actif par défaut (MasterServer enabled=true dans tes3mp-server-default.cfg).

**Recommandation** — Introduire un verrou commun protégeant la map Players (ou faire produire par la boucle principale un instantané des noms sous verrou, consommé par le thread). Au minimum, remplacer l'accès direct par une copie des noms poussée depuis le thread réseau. Envisager de supprimer le thread au profit d'un envoi cadencé dans mainLoop.

**Contre-vérification (partiel (sévérité corrigée))** — La course est réelle. MasterClient::Thread (MasterClient.cpp:193-226) itère *players et lit npc.mName sans verrou, et mutexData ne protège que queryData. Pendant ce temps, Players::newPlayer/deletePlayer modifient la map et font delete sur le thread principal (Player.cpp:18-19 et 27, et non 122-131 comme indiqué). Le thread est démarré par défaut (main.cpp:300, enabled=true). Le comportement est indéfini, mais la fenêtre est étroite : un parcours court toutes les 10-15 s (rate=10000, MasterClient.cpp:21). Le crash est donc rare et non systématique, d'où une sévérité moyenne plutôt que haute.

### SRV-4

**Macro GET_PLAYER teste une variable codée en dur "player" au lieu de son argument**

- **Sévérité** : 🟡 moyenne (initialement haute) · **Catégorie** : bug · **Effort** : M
- **Emplacement** : `apps/openmw-mp/Script/ScriptFunctions.hpp:37`

**Preuve**

> #define GET_PLAYER(pid, pl, retvalue) pl = Players::getPlayer(pid); if (player == 0) {... return retvalue;} — le garde-fou teste 'player', pas le paramètre 'pl'.

**Impact** — Dans Mechanics.cpp:197 AddAlliedPlayerForPlayer fait GET_PLAYER(alliedPlayerPid, alliedPlayer, ) : alliedPlayer reçoit getPlayer(alliedPlayerPid) mais le test porte sur 'player' (déjà validé juste avant), jamais sur alliedPlayer. Si un script passe un alliedPlayerPid invalide/obsolète, alliedPlayer est nullptr et player->alliedPlayers.push_back(alliedPlayer->guid) (ligne 199) déréférence un pointeur nul → crash serveur déclenchable depuis Lua. La macro est aussi un piège pour toute nouvelle fonction n'utilisant pas le nom 'player'.

**Recommandation** — Corriger la macro pour tester son propre paramètre : if ((pl) == nullptr) {...}. Recompiler et auditer les 278 sites d'appel (ici un seul diverge, mais le risque est permanent).

**Contre-vérification (partiel (sévérité corrigée))** — La macro est bien défectueuse : ScriptFunctions.hpp:37-39 teste 'player' et non 'pl'. Mechanics.cpp:197-199 l'utilise avec alliedPlayer, puis déréférence alliedPlayer->guid sans vérification. C'est le seul usage avec un autre nom (grep). Le déclencheur est un appel Lua serveur avec un pid invalide, donc une erreur de script et non une entrée réseau directe : sévérité moyenne plutôt que haute.

### SRV-7

**preInit continue le traitement après un mauvais premier paquet (return manquant)**

- **Sévérité** : 🟡 moyenne · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw-mp/Networking.cpp:234`

**Preuve**

> if (packet->data[0] != ID_GAME_PREINIT) { LOG_...; peer->CloseConnection(packet->systemAddress, true); } — pas de return. Le flux continue ensuite en loggant « Received ID_GAME_PREINIT » et en parsant le paquet comme un PacketPreInit.

**Impact** — Une connexion dont le premier paquet n'est pas ID_GAME_PREINIT est fermée, mais le code poursuit la lecture/validation et, si le contenu passe les tests, exécute Players::newPlayer + demande de handshake sur une connexion déjà fermée : état incohérent (joueur fantôme) et traitement inutile d'un paquet attaquant.

**Recommandation** — Ajouter return false; juste après le CloseConnection dans la branche du mauvais premier paquet (Networking.cpp:238).

**Contre-vérification (confirmé)** — Networking.cpp:234-239 ferme la connexion sans return, puis lit le corps comme un PacketPreInit sans vérifier l'ID. BasePacket::Read ne contrôle pas packetID. Si le corps est valide, Players::newPlayer (l.297) crée un joueur et lui attribue un slot (Player.cpp:36-44). Comme CloseConnection ne génère pas d'ID_DISCONNECTION_NOTIFICATION locale, disconnectPlayer n'est jamais appelé pour lui : un joueur fantôme occupe le slot de façon permanente, ce que l'attaquant peut répéter.

### SRV-8

**Boucle principale à sleep fixe 1 ms + sondage clavier, non adaptative**

- **Sévérité** : 🟡 moyenne · **Catégorie** : performance · **Effort** : M
- **Emplacement** : `apps/openmw-mp/Networking.cpp:572`

**Preuve**

> while (running && !killLoop) { ... kbhit()/getch() ...; for(packet=peer->Receive(); ...) ...; TimerAPI::Tick(); std::this_thread::sleep_for(std::chrono::milliseconds(1)); }. sigaction(SIGTERM/SIGINT) est aussi réenregistré à chaque tour (l.513-514).

**Impact** — Plancher de latence d'environ 1 ms par cycle même sans charge, réveils permanents (coût CPU à vide, mauvais pour les hébergements multi-serveurs) ; aucune régulation : en pic de paquets, la boucle peut traiter sans relâche puis dormir arbitrairement. L'appel kbhit (select sur fd 0) à chaque tour est superflu et indéfini sans tty.

**Recommandation** — Remplacer le sleep fixe par une attente bornée pilotée par la disponibilité réseau (ou cadence cible configurable). Sortir l'enregistrement des signaux de la boucle (une fois avant la boucle). Rendre le sondage clavier conditionnel à la présence d'un tty.

### SRV-10

**getLastPlayerId déréférence slots.rbegin() sans vérifier la vacuité**

- **Sévérité** : ⚪ basse · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw-mp/Player.cpp:64`

**Preuve**

> unsigned short Players::getLastPlayerId() { return slots.rbegin()->first; } — si slots est vide, rbegin()==rend() et le déréférencement est un comportement indéfini.

**Impact** — MiscellaneousFunctions::GetLastPlayerId (Miscellaneous.cpp:41) expose cette fonction aux scripts Lua. Appelée avant toute connexion (ex. dans OnServerPostInit), elle lit une mémoire invalide. slots n'étant jamais purgé (les entrées sont mises à 0 mais pas erase), le cas ne survient qu'avant le premier joueur, mais reste un UB exploitable par script.

**Recommandation** — Retourner une valeur sentinelle (ou 0) si slots.empty(), ou documenter/garder côté API : if (slots.empty()) return 0;

### SRV-11

**MasterClient::SetUpdateRate borne la mauvaise variable puis l'écrase**

- **Sévérité** : ⚪ basse · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw-mp/MasterClient.cpp:250`

**Preuve**

> void SetUpdateRate(unsigned int rate){ if (timeout < min_rate) timeout = min_rate; else if (timeout > max_rate) timeout = max_rate; timeout = rate; } — le clamp agit sur 'timeout' (ancienne valeur) puis timeout est écrasé par 'rate' non borné.

**Impact** — Le bornage min_rate/max_rate (1000..60000) est du code mort : une cadence hors plage passée en config contourne les limites. Atténué par main.cpp:287 qui force updateRate >= 8000, mais la fonction publique ne protège rien.

**Recommandation** — Borner 'rate' puis l'assigner : rate = std::clamp(rate, min_rate, max_rate); timeout = rate;

### SRV-12

**Player::id non initialisé ; pas de garde si aucun slot libre**

- **Sévérité** : ⚪ basse · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw-mp/Player.cpp:171`

**Preuve**

> Player::Player(guid){ handshakeCounter=0; loadState=NOTLOADED; } — 'id' n'est pas initialisé. Dans Players::newPlayer (l.36-44) setId(i) n'est appelé que si un slot libre est trouvé dans [0, maxConnections[.

**Impact** — Si la boucle de slots ne trouve aucune place (incohérence entre nombre de joueurs et maxConnections, ou slots non purgés), le Player garde un id non initialisé, ensuite utilisé comme clé/identifiant côté scripts et paquets → comportement erratique. Même sans ce cas, lire un membre non initialisé est fragile.

**Recommandation** — Initialiser id (ex. à un sentinel invalide) dans le constructeur et, dans newPlayer, journaliser/refuser proprement si aucun slot n'est disponible.

### SRV-13

**≈90 processeurs quasi identiques : boilerplate massif à faible valeur**

- **Sévérité** : ⚪ basse · **Catégorie** : maintenabilite · **Effort** : L
- **Emplacement** : `apps/openmw-mp/processors/object/ProcessorObjectSound.hpp:16`

**Preuve**

> 59 des 79 en-têtes de processeurs ont <=2 lignes utiles ; 5 sont vides (Do ne fait rien). Motif répété : Do(){ LOG_... ; Script::Call<CallbackIdentity("OnX")>(player.getId(), objectList.cell.getShortDescription().c_str()); } — seul l'ID de paquet et le nom du callback changent.

**Impact** — Chaque nouveau type de paquet impose de créer un fichier, l'inclure et l'enregistrer dans ProcessorInitializer.cpp (3 endroits), avec risque d'oubli/incohérence. Volume de code à maintenir disproportionné par rapport à la logique réelle ; compilation alourdie (tout est en en-têtes inclus dans une seule TU d'init).

**Recommandation** — Remplacer les processeurs triviaux par une table déclarative (ID de paquet → nom de callback → mode de diffusion : none / sendToLoaded / broadcast) pilotée par données, en ne gardant des classes dédiées que pour les cas à logique propre (CellChange, ActorList...). Réduit des dizaines de fichiers à quelques lignes de table.

### SRV-6

**Console d'administration morte : handleInput.cpp non compilé et jamais appelé**

- **Sévérité** : ⚪ basse (initialement moyenne) · **Catégorie** : bug · **Effort** : M
- **Emplacement** : `apps/openmw-mp/handleInput.cpp:9`

**Preuve**

> mwmp_input::handler() bufferise la saisie et appelle OnServerWindowInput, mais handleInput.cpp n'apparaît pas dans apps/openmw-mp/CMakeLists.txt (grep vide) et handler() n'a aucun appelant (grep sur tout le dépôt). La vraie boucle fait seulement : if (kbhit() && getch() == '\n') break; (Networking.cpp:516).

**Impact** — Aucune commande console côté serveur n'existe réellement : le mécanisme OnServerWindowInput est du code mort. Pire, dans mainLoop toute entrée est ignorée sauf la touche Entrée, qui ARRÊTE le serveur. Un opérateur qui tape une commande puis Entrée tue le serveur. Fonctionnalité d'admin absente et piège d'exploitation.

**Recommandation** — Soit câbler mwmp_input::handler() dans mainLoop (et l'ajouter au CMake) pour router la saisie vers OnServerWindowInput, soit supprimer handleInput.cpp. Dissocier « Entrée = arrêt » d'une vraie ligne de commande (ex. commande 'quit').

**Contre-vérification (partiel (sévérité corrigée))** — Le code mort est confirmé : handleInput.cpp est absent de la liste de sources de apps/openmw-mp/CMakeLists.txt, handler() n'a aucun appelant, et OnServerWindowInput n'apparaît nulle part ailleurs. En revanche, Networking.cpp:516 (if (kbhit() && getch() == '\n') break;) est un arrêt volontaire et propre de la boucle (puis TimerAPI::Terminate), comparable à Ctrl+C. Il s'agit d'une fonctionnalité d'administration absente et d'un piège d'ergonomie local, pas d'un bug de sévérité moyenne.

### SRV-9

**killLoop est un bool global non atomique écrit depuis un gestionnaire de signal**

- **Sévérité** : ⚪ basse · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw-mp/Networking.cpp:35`

**Preuve**

> bool killLoop = false; ... void signalHandler(int signum){ ... killLoop = true; } puis while (running && !killLoop). De même 'running' est un membre bool simple modifié par stopServer().

**Impact** — Écrire/lire un bool non volatile depuis un handler de signal est techniquement un comportement indéfini (le standard exige volatile std::sig_atomic_t ou std::atomic). En pratique l'arrêt fonctionne, mais sans garantie que la boucle voie le changement selon l'optimisation du compilateur.

**Recommandation** — Déclarer killLoop comme volatile std::sig_atomic_t (ou std::atomic<bool>) et 'running' comme std::atomic<bool>. Enregistrer les handlers une seule fois.

## Moteur de scripts serveur (Lua/natif)

| ID | Sévérité | Catégorie | Titre | Emplacement | Vérif. |
|---|---|---|---|---|---|
| [SCR-1](#scr-1) | 🔴 critique | build-ci | La couche de script ne compile pas avec GCC ≥ 9 ni avec clang : reinterpret_cast dans un constexpr | `apps/openmw-mp/Script/Types.hpp:104` | confirmé |
| [SCR-2](#scr-2) | 🟠 haute | securite | Arguments chaîne de CreateTimerEx gardés en const char* vers des chaînes Lua : use-after-free au déclenchement | `apps/openmw-mp/Script/LangLua/LuaFunc.cpp:159` | partiel (sévérité corrigée) |
| [SCR-3](#scr-3) | 🟠 haute | bug | Wrappers Lua noexcept : une erreur d'argument, un nil ou une exception C++ tue le processus (terminate / SIGSEGV) | `apps/openmw-mp/Script/LangLua/LangLua.cpp:87` | confirmé |
| [SCR-4](#scr-4) | 🟠 haute | bug | TimerAPI : réutilisation d'ID cassée (copie de paire), timers fantômes et pointeurs pendants | `apps/openmw-mp/Script/API/TimerAPI.cpp:72` | confirmé |
| [SCR-7](#scr-7) | 🟠 haute | bug | Gestion d'erreurs incohérente : timers hors ignoreScriptErrors, récursion d'OnServerScriptCrash, pas de traceback | `apps/openmw-mp/Script/Script.hpp:104` | confirmé |
| [SCR-5](#scr-5) | 🟡 moyenne | performance | Timers jamais libérés et Tick en O(N) à ~1 kHz : la charge CPU augmente avec l'uptime | `apps/openmw-mp/Script/API/TimerAPI.cpp:196` | partiel (sévérité corrigée) |
| [SCR-6](#scr-6) | 🟡 moyenne | bug | Une valeur fuit sur la pile Lua à chaque événement : panique LuaJIT après ~65 500 appels sans timer | `apps/openmw-mp/Script/LangLua/LangLua.cpp:269` | partiel (sévérité corrigée) |
| [SCR-8](#scr-8) | 🟡 moyenne | securite | Message d'erreur Lua utilisé comme chaîne de format printf | `apps/openmw-mp/Script/Script.hpp:103` | confirmé |
| [SCR-9](#scr-9) | 🟡 moyenne | manque | Aucun contrat d'API vérifié entre le serveur et CoreScripts : fonction appelée inexistante, aucun manifeste, aucun test | `apps/openmw-mp/Script/Functions/Cells.hpp:21` | non soumis |
| [SCR-10](#scr-10) | 🟡 moyenne | maintenabilite | Toutes les fonctions de l'API sont appelées via un pointeur variadique R(*)(...) : comportement indéfini, float silencieusement mal typé | `apps/openmw-mp/Script/LangLua/LangLua.cpp:81` | non soumis |
| [SCR-11](#scr-11) | 🟡 moyenne | build-ci | La branche ARM est cassée par le refactor 953be7c, et les wrappers spéciaux dépendent de l'ordre de la table | `apps/openmw-mp/Script/LangLua/LangLua.cpp:129` | non soumis |
| [SCR-12](#scr-12) | 🟡 moyenne | bug | Les plugins natifs (.so/.dll) ne fonctionnent pas : timers et publics C++ jamais exécutés | `apps/openmw-mp/Script/ScriptFunction.cpp:38` | confirmé |
| [SCR-13](#scr-13) | ⚪ basse | architecture | API à état global implicite : pointeurs read* non initialisés et tampons statiques partagés | `apps/openmw-mp/Script/Functions/Objects.cpp:49` | non soumis |
| [SCR-14](#scr-14) | ⚪ basse | bug | PublicFnAPI : doublons ignorés en silence, DeleteAll invalide son itérateur, retour chaîne pendant | `apps/openmw-mp/Script/API/PublicFnAPI.cpp:62` | non soumis |
| [SCR-15](#scr-15) | ⚪ basse | bug | Messages d'erreur construits par arithmétique de pointeur, et index hors bornes | `apps/openmw-mp/Script/LangLua/LangLua.cpp:263` | non soumis |
| [SCR-16](#scr-16) | ⚪ basse | performance | Dispatch d'événements : coût négligeable mais cache de présence figé et recherches par nom | `apps/openmw-mp/Script/Script.hpp:38` | non soumis |
| [SCR-17](#scr-17) | ℹ️ info | securite | Aucun sandbox ni budget d'exécution, et type de script détecté par sous-chaîne | `apps/openmw-mp/Script/LangLua/LangLua.cpp:37` | non soumis |

### SCR-1

**La couche de script ne compile pas avec GCC ≥ 9 ni avec clang : reinterpret_cast dans un constexpr**

- **Sévérité** : 🔴 critique · **Catégorie** : build-ci · **Effort** : M
- **Emplacement** : `apps/openmw-mp/Script/Types.hpp:104`

**Preuve**

> Types.hpp:104 : `constexpr ScriptFunctionPointer(Function<R, Types...> addr) : ScriptIdentity(addr), addr((void*)(addr)) {}`. Ce constructeur sert à initialiser `static constexpr ScriptFunctionData functions[]` (ScriptFunctions.hpp:120). Avec g++ 9.5, 11.5 et 13.3 (-std=gnu++17, comme CMake avec CXX_EXTENSIONS YES) : `Types.hpp:104:94: error: 'reinterpret_cast' is not a constant expression` (-fpermissive n'y change rien). Avec clang 18 (branche #else, l.107) : `cannot initialize a member subobject of type 'void *' with an lvalue of type 'Function<...>'`. La seule CI qui compile avec GCC (.travis.yml:52-54) utilise g++-8 sur bionic.

**Impact** — tes3mp-server ne se compile qu'avec un ancien GCC laxiste (probablement ≤ 8). En effet, toute unité qui inclut ScriptFunctions.hpp échoue : Script.cpp, LangLua.cpp, Functions/*.cpp, TimerAPI.cpp. La compilation est donc impossible sur les distributions actuelles, avec clang/macOS et pour l'analyse statique (clang-tidy). Cela bloque toute maintenance du serveur.

**Recommandation** — Ne plus stocker de void* dans un constexpr. Deux options :
- remplacer la table par un enregistrement typé C++17 `template<auto F> int LuaWrap(lua_State*)`, qui déduit la signature de F, par exemple `{"GetName", &LuaWrap<&StatsFunctions::GetName>}` ;
- ou garder ScriptIdentity constexpr (types/ret/numargs) et placer les adresses dans un tableau non constexpr rempli au démarrage.

Ajouter ensuite un workflow GitHub Actions qui compile tes3mp-server avec gcc-13 et clang-18.

**Contre-vérification (confirmé)** — Reproduit sans rien modifier, en `-fsyntax-only` sur apps/openmw-mp/Script/ScriptFunctions.cpp. g++ 13.3 donne `Types.hpp:104:94: error: 'reinterpret_cast' is not a constant expression` (appelé depuis ScriptFunctions.hpp:153). clang++ 18 donne `Types.hpp:107:89: cannot initialize a member subobject of type 'void *'`. Les fichiers repro/g++-9.err à g++-14.err montrent la même erreur. La seule CI GCC est .travis.yml:52-54 (gcc-8, bionic), et le commit gcc-13 7921f71 n'a touché que des includes `<cstdint>` : le serveur ne compile pas avec les toolchains actuelles.

### SCR-2

**Arguments chaîne de CreateTimerEx gardés en const char* vers des chaînes Lua : use-after-free au déclenchement**

- **Sévérité** : 🟠 haute (initialement critique) · **Catégorie** : securite · **Effort** : S
- **Emplacement** : `apps/openmw-mp/Script/LangLua/LuaFunc.cpp:159`

**Preuve**

> LuaFunc.cpp:159 : `args.emplace_back(luabridge::Stack<const char*>::get(lua, i));`. Le pointeur est conservé dans Timer::args jusqu'au tir (TimerAPI.cpp:35 `Call(args);`), puis repoussé dans Lua par LangLua::Call (LangLua.cpp:308). CoreScripts appelle `tes3mp.CreateTimerEx("OnLoginTimeExpiration", time.seconds(config.loginTime), "is", pid, Players[pid].accountName)` (eventHandler.lua:613, loginTime=60), avec le même motif pour la mort (player/base.lua:561). Le banc LuaJIT qui reproduit ce motif affiche 'Joueur_AAAA…1795502302' avant déconnexion, puis 'zz38' après `Players[0]=nil; collectgarbage()`.

**Impact** — Un client peut laisser un pointeur pendant de deux façons : se connecter puis se déconnecter avant 60 s, ou mourir puis se déconnecter avant deathTime=5 s. Au tir, lua_pushstring lit de la mémoire libérée ou réutilisée. Le script reçoit alors un argument corrompu (le mauvais nom de compte) ; strlen peut lire au-delà de la zone et un crash est possible. N'importe quel client distant peut déclencher le problème.

**Recommandation** — Copier les chaînes au lieu de garder le pointeur :
- dans LangLua::CreateTimerEx et DefToVec, stocker `std::string(luabridge::Stack<const char*>::get(lua, i))` dans le boost::any ;
- dans LangLua::Call(vector), pousser `boost::any_cast<std::string>(…)` avec lua_pushlstring.

Autre option : garder une référence registre (luaL_ref) par argument et la libérer dans ~Timer.

**Contre-vérification (partiel (sévérité corrigée))** — Le défaut C++ est réel. LuaFunc.cpp:159 garde un `const char*` Lua sans ancrage (ni luaL_ref ni copie std::string) jusqu'au push dans LangLua.cpp:308. Le chemin « connexion puis déconnexion » est en revanche neutralisé : BasePlayer:Destroy (CoreScripts player/base.lua:141-144), appelé aussi par Kick (l.150-151), fait `tes3mp.StopTimer(self.loginTimerId)`, donc ce timer ne tire jamais. Seul le chemin « mort puis déconnexion en moins de 5 s » reste ouvert (resurrectTimerId, base.lua:561, n'est pas arrêté). Il faut en plus que le GC ait libéré la chaîne : le plus probable est une lecture de mémoire libérée qui échoue à la comparaison d'eventHandler.lua:1989, un crash restant possible mais non démontré, d'où « haute » plutôt que « critique ».

### SCR-3

**Wrappers Lua noexcept : une erreur d'argument, un nil ou une exception C++ tue le processus (terminate / SIGSEGV)**

- **Sévérité** : 🟠 haute · **Catégorie** : bug · **Effort** : M
- **Emplacement** : `apps/openmw-mp/Script/LangLua/LangLua.cpp:87`

**Preuve**

> LangLua.cpp:61, 77, 87 et 95 : `Dispatch(...) noexcept` et `LuaFunctionWrapper(lua_State* lua) noexcept` appellent luabridge::Stack<T>::get. Celle-ci passe par luaL_checkinteger/luaL_checkstring, qui lèvent une erreur Lua, propagée comme exception sous LuaJIT x64. Stack<const char*>::get renvoie 0 pour nil, puis Chat.cpp:14 fait `player->chatMessage = message;`. Banc :
> - `pcall(tes3mp.SendMessage, "abc", "x")` donne `terminate called without an active exception` (exit 134) ;
> - `pcall(tes3mp.SendMessage, 1, nil)` donne `Segmentation fault` (exit 139).
> 
> De plus, Functions/*.cpp contient 229 appels .at() dans des fonctions noexcept, par exemple Objects.cpp:94 `return readObjectList->baseObjects.at(index).refId.c_str();` sans contrôle de borne.

**Impact** — Une faute de type, un index hors bornes ou une variable nil dans un script arrête brutalement le serveur. C'est vrai même si la valeur vient de données réseau, même sous pcall Lua et même avec ignoreScriptErrors=true. Aucun message exploitable n'est produit, au lieu d'une erreur Lua récupérable.

**Recommandation** — - Retirer noexcept de LuaFunctionWrapper, de Dispatch et des fonctions LangLua::MakePublic et LangLua::CreateTimer.
- Entourer l'appel C++ d'un try/catch qui convertit std::exception en `luaL_error(L, "tes3mp.%s: %s", name, e.what())`.
- Pour les paramètres 's', rejeter nil avec un message explicite (lua_type(L, i) != LUA_TSTRING).
- Remplacer les `.at(index)` non gardés par `if (index >= size()) { LOG; return valeur_neutre; }`.

**Contre-vérification (confirmé)** — Les wrappers sont bien noexcept (LangLua.cpp:61, 77, 87, 95). Stack<char const*>::get vaut `lua_isnil ? 0 : luaL_checkstring` (extern/LuaBridge/detail/Stack.h:339-341), puis Chat.cpp:14 fait `player->chatMessage = message;` avec ce nullptr. Le banc scr/noexcept_err rejoué donne exit 134 (`terminate called without an active exception`) pour `pcall(tes3mp.SendMessage,"abc","x")` et exit 139 pour `(1,nil)`. Il y a 229 `.at(` dans Functions/*.cpp, par exemple Objects.cpp:94 dans une fonction noexcept. Seule nuance : l'origine « données réseau » est surtout indirecte (le déclencheur réel est une erreur de script).

### SCR-4

**TimerAPI : réutilisation d'ID cassée (copie de paire), timers fantômes et pointeurs pendants**

- **Sévérité** : 🟠 haute · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw-mp/Script/API/TimerAPI.cpp:72`

**Preuve**

> TimerAPI.cpp:72-76 : `for (auto timer : timers) { if (timer.second != nullptr) continue; timer.second = new Timer(lua, callback, msec, def, args); id = timer.first; }`. La boucle modifie une copie de la paire, sans break (même code l.96-100). Le banc reproduisant ce code affiche `ID libere=0 ; CreateTimer suivants -> 0 puis 0 ; timers[0]=(nil)`. TimerAPI.cpp:135 `timers.at(timerid)->Restart(msec);` déréférence ensuite ce nullptr. Terminate() (l.186-191) fait delete via une copie et laisse les pointeurs dans la map.

**Impact** — Après un seul FreeTimer, chaque CreateTimer renvoie le même ID mort :
- StartTimer affiche 'Timer not found!' et le callback ne part jamais ;
- RestartTimer/StopTimer/IsTimerElapsed sur cet ID provoquent un SIGSEGV, que catch(...) n'attrape pas ;
- chaque création fuit un Timer par slot libre.

Après Terminate, un handler OnServerExit qui manipule un timer fait un use-after-free. Cela explique que CoreScripts n'appelle jamais FreeTimer.

**Recommandation** — - Correctif minimal : itérer par référence (`for (auto &timer : timers)`) avec un `break`.
- Correctif complet : `std::vector<std::unique_ptr<Timer>>` plus une pile d'ID libres.
- Terminate() doit faire `timers.clear()`.
- Dans Tick, collecter d'abord les timers échus, puis appeler leurs callbacks hors de l'itération, pour éviter l'invalidation d'itérateurs si un callback crée un timer.
- Ajouter des tests gtest Create/Free/Create/Restart.

**Contre-vérification (confirmé)** — TimerAPI.cpp:72-78 et 96-102 : `for (auto timer : timers)` modifie une copie de la paire, sans break. Après FreeTimer (l.119-122 met timers[id]=nullptr), chaque création renvoie cet id mort et fuit le Timer alloué. StartTimer le refuse (l.148), mais ResetTimer (l.135), StopTimer (l.162) et IsTimerElapsed (l.175) déréférencent le nullptr, ce que catch(...) n'intercepte pas. Terminate (l.186-191) supprime les timers via une copie, puis ~Networking appelle OnServerExit (Networking.cpp:70) après Terminate (l.575) : pointeurs pendants. Le défaut reste latent avec CoreScripts, qui n'appelle aucun FreeTimer (grep vide), mais il touche toute API publique utilisée par des scripts tiers.

### SCR-7

**Gestion d'erreurs incohérente : timers hors ignoreScriptErrors, récursion d'OnServerScriptCrash, pas de traceback**

- **Sévérité** : 🟠 haute · **Catégorie** : bug · **Effort** : M
- **Emplacement** : `apps/openmw-mp/Script/Script.hpp:104`

**Preuve**

> Script.hpp:101-107 : `catch (std::exception &e) { LOG_MESSAGE_SIMPLE(...); Script::Call<Script::CallbackIdentity("OnServerScriptCrash")>(e.what()); if (!…getScriptErrorIgnoringState()) throw; }`. À l'inverse, Timer::Tick (TimerAPI.cpp:35 `Call(args);`), TimerAPI::Tick et Networking::mainLoop n'ont aucun try/catch : l'exception remonte jusqu'à main.cpp:309. Le pcall n'a pas de gestionnaire (LangLua.cpp:269, msgh=0). Côté CoreScripts, OnServerScriptCrash déclenche les handlers 'OnServerExit' personnalisés (serverCore.lua:289-291).

**Impact** — 1. Une erreur Lua dans un callback de timer arrête le serveur, même avec ignoreScriptErrors=true.
2. Si le handler OnServerScriptCrash échoue lui-même (handler personnalisé fautif), Script::Call se rappelle indéfiniment, jusqu'au débordement de pile C et au SIGSEGV.
3. Les erreurs ne donnent que 'fichier:ligne: message' sans pile d'appel.
4. Avec ignoreScriptErrors, chaque erreur déclenche les hooks OnServerExit alors que le serveur continue de tourner.

**Recommandation** — - Centraliser tout appel Lua (événements, timers, publics) dans une fonction `SafeCall`.
- Y faire un pcall avec un gestionnaire qui appelle luaL_traceback.
- Y logger via "%s" et y appliquer partout la politique ignoreScriptErrors.
- Empêcher la réentrance d'OnServerScriptCrash avec un drapeau statique `inCrashHandler`.
- Émettre un événement distinct `OnServerScriptError` quand l'erreur est ignorée.

**Contre-vérification (confirmé)** — Timer::Tick appelle `Call(args)` (TimerAPI.cpp:35), et ni TimerAPI::Tick ni mainLoop n'ont de try/catch (grep try/catch vide dans Networking.cpp). Une LuaException (std::exception, LuaException.h:37) remonte donc jusqu'à main.cpp:309-313, qui relance l'exception sans consulter ignoreScriptErrors. Dans Script.hpp:101-104, le catch appelle Script::Call<OnServerScriptCrash> avant le test de la l.106 : un handler de crash qui échoue relance la même branche, d'où une récursion infinie. pcall a msgh=0 (LuaException.h:96-98 via LangLua.cpp:269), donc pas de traceback. serverCore.lua:289-291 déclenche bien les hooks OnServerExit.

### SCR-10

**Toutes les fonctions de l'API sont appelées via un pointeur variadique R(*)(...) : comportement indéfini, float silencieusement mal typé**

- **Sévérité** : 🟡 moyenne · **Catégorie** : maintenabilite · **Effort** : M
- **Emplacement** : `apps/openmw-mp/Script/LangLua/LangLua.cpp:81`

**Preuve**

> LangLua.cpp:81 : `return reinterpret_cast<FunctionEllipsis<ReturnType>>(functionData.func.addr)(std::forward<Args>(args)...);`, avec FunctionEllipsis = `R(*)(...)` (Types.hpp:63-64). Script.hpp:93 fait de même pour les callbacks natifs. Dans Types.hpp:30-32, `TypeChar<T, sizeof(uint32_t)>` capture aussi float : `static_assert(TypeChar<float,sizeof(float)>::value == 'q')` compile (vérifié).

**Impact** — Appeler une fonction non variadique via un pointeur variadique est un comportement indéfini. Cela ne fonctionne que par chance sur x86-64 SysV/Win64, parce qu'aucune fonction exportée ne prend de float. Ajouter un paramètre float ferait lire la valeur comme un entier puis la transmettre promue en double : résultat faux, sans erreur. Sur l'ABI Apple arm64 (arguments variadiques passés sur la pile), tous les appels seraient faux.

**Recommandation** — Appeler chaque fonction avec son type exact. Par exemple, un wrapper `template<auto F> int LuaWrap(lua_State* L)` qui lit les arguments dans un std::tuple déduit de F, puis appelle `std::apply(F, args)`. À défaut, ajouter dans TypeChar un `static_assert` qui interdit float et long double.

### SCR-11

**La branche ARM est cassée par le refactor 953be7c, et les wrappers spéciaux dépendent de l'ordre de la table**

- **Sévérité** : 🟡 moyenne · **Catégorie** : build-ci · **Effort** : S
- **Emplacement** : `apps/openmw-mp/Script/LangLua/LangLua.cpp:129`

**Preuve**

> Le refactor a renommé LuaFuctionData en LuaFunctionData et F_ en LuaFunctionDefinition. Pourtant, `#ifdef __arm__` garde `LuaFuctionData *functions(indices<Indices...>)` et `F_<Indices>::F...` (LangLua.cpp:129-133), et appelle `GetLuaFunctions(IndicesFor<functions_n>{})` (l.189), fonction inexistante. LangLua.cpp:112-115 spécialisent en dur les indices 0 à 3 : `template<> struct LuaFunctionDefinition<0> { … {"CreateTimer", LangLua::CreateTimer}; };`.

**Impact** — Le serveur ne compile plus sur ARM 32 bits (Raspberry Pi, cible courante des petits serveurs), et personne ne s'en est aperçu faute de CI. Si l'on réordonne les quatre premières entrées de ScriptFunctions.hpp:121-124, le mauvais wrapper est lié au mauvais nom, sans erreur.

**Recommandation** — - Supprimer la branche __arm__ : le chemin générique fonctionne en C++17.
- Remplacer les spécialisations par indice par une surcharge par nom, ou par un tableau construit via std::index_sequence avec un static_assert sur les noms des quatre entrées spéciales.
- Ajouter un job de compilation croisée armhf/aarch64.

### SCR-12

**Les plugins natifs (.so/.dll) ne fonctionnent pas : timers et publics C++ jamais exécutés**

- **Sévérité** : 🟡 moyenne · **Catégorie** : bug · **Effort** : M
- **Emplacement** : `apps/openmw-mp/Script/ScriptFunction.cpp:38`

**Preuve**

> ScriptFunction::Call (ScriptFunction.cpp:35-38) : `if (def.length() != args.size()) throw …; else if (script_type == SCRIPT_LUA) {…}`. Il n'existe aucune branche SCRIPT_CPP : fCpp n'est jamais appelé. LangNative::Call renvoie nullptr. LangNative.cpp:42-43 fait `const char *prefix = SystemInterface<const char *>(lib, "prefix").result; std::string pf(prefix);` sans tester nullptr. Le chargement se fait par `dlopen(filename, RTLD_LAZY)` (l.33), sans contrôle de version ni d'ABI.

**Impact** — Un plugin natif se charge, mais ses timers et ses fonctions publiques ne s'exécutent jamais, sans aucun message. Un plugin sans symbole 'prefix' déclenche une exception ou un comportement indéfini. Toute évolution de signature casse les plugins sans diagnostic. Le code mort qui en résulte (union manuelle dans ScriptFunction, SystemInterface, CallPublic variadique) alourdit la maintenance.

**Recommandation** — Trancher entre deux options :
- retirer le backend natif (LangNative, SCRIPT_CPP, variantes va_list de CreateTimerEx/CallPublic) ;
- ou le terminer : ajouter la branche SCRIPT_CPP qui appelle fCpp, exiger un symbole `tes3mp_api_version` vérifié au chargement, tester `prefix != nullptr` et fournir un plugin d'exemple compilé en CI.

**Contre-vérification (confirmé)** — ScriptFunction::Call (ScriptFunction.cpp:35-68) n'a qu'une branche SCRIPT_LUA et renvoie un `result` vide pour SCRIPT_CPP. fCpp n'est jamais appelé, alors que Functions/Timer.cpp:11,21 et ScriptFunctions.cpp:16 créent des timers et des publics C++. LangNative::Call renvoie nullptr (LangNative.cpp:71-79). Le symbole `prefix` est lu sans test (l.42-43). Avec libstdc++, `std::string(nullptr)` lève logic_error, attrapée par le catch(...) de la l.49 : on obtient donc une exception peu claire plutôt qu'un UB. dlopen se fait en RTLD_LAZY sans contrôle d'ABI (l.33).

### SCR-5

**Timers jamais libérés et Tick en O(N) à ~1 kHz : la charge CPU augmente avec l'uptime**

- **Sévérité** : 🟡 moyenne (initialement haute) · **Catégorie** : performance · **Effort** : M
- **Emplacement** : `apps/openmw-mp/Script/API/TimerAPI.cpp:196`

**Preuve**

> TimerAPI.cpp:196-200 : `for (auto timer : timers) if (timer.second != nullptr) timer.second->Tick();`. Cette boucle s'exécute à chaque tour de mainLoop (Networking.cpp:571, suivi d'un sleep de 1 ms). CoreScripts crée un timer par connexion (eventHandler.lua:613) et un par mort (player/base.lua:561), et n'appelle jamais FreeTimer. Banc : Tick prend 2,2 µs avec 1 000 timers, 20 µs avec 10 000 et 948 µs avec 100 000 (environ 95 % d'un cœur). CreateTimer balaie aussi toute la map (O(N)) et Timer::Tick appelle system_clock::now() (l.29).

**Impact** — La map grossit sans limite : chaque création fuit un Timer, deux std::string et un vector<boost::any>. Le coût du tick croît linéairement. Un serveur public qui tourne longtemps finit par consacrer une part croissante d'un cœur à parcourir des timers terminés, ce qui retarde le traitement réseau. L'usage de system_clock fait aussi tirer les timers trop tôt ou trop tard en cas de saut d'horloge (NTP).

**Recommandation** — - Libérer automatiquement un timer one-shot après son tir, ou le réutiliser via RestartTimer côté CoreScripts (loginTimerId/resurrectTimerId).
- Remplacer le balayage par une file de priorité (min-heap sur l'échéance) basée sur std::chrono::steady_clock : Tick devient O(1) quand rien n'est échu.
- Exposer tes3mp.GetTimerCount() pour la supervision.

**Contre-vérification (partiel (sévérité corrigée))** — Le défaut est réel. Tick parcourt toute la map à chaque tour (TimerAPI.cpp:196-200, Networking.cpp:571-572 avec un sleep de 1 ms), CreateTimer la balaie aussi (l.72), et CoreScripts crée des timers à chaque connexion (eventHandler.lua:613) et à chaque mort (base.lua:561) sans jamais appeler FreeTimer, avec system_clock en l.29 et l.59. L'impact « haute » est exagéré : il faut des dizaines de milliers de connexions ou de morts pour atteindre environ 20 µs par tick (≈2 % d'un cœur selon le banc), et chaque Timer ne pèse que quelques centaines d'octets. C'est une fuite lente et non bornée plutôt qu'un goulot majeur.

### SCR-6

**Une valeur fuit sur la pile Lua à chaque événement : panique LuaJIT après ~65 500 appels sans timer**

- **Sévérité** : 🟡 moyenne (initialement haute) · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw-mp/Script/LangLua/LangLua.cpp:269`

**Preuve**

> LangLua.cpp:269-270 : `luabridge::LuaException::pcall(lua, n_args, 1); return boost::any(luabridge::LuaRef::fromStack(lua, -1));`. fromStack(L, index) fait lua_pushvalue puis luaL_ref, et ne retire pas l'original (LuaRef.h:828 « The stack item is not popped »). Script::Call ignore le résultat. Banc LuaJIT 2.1 : `apres 60000 appels : lua_gettop=60000`, puis `PANIC: unprotected error in call to Lua API (stack overflow)`. Seul ScriptFunction.cpp:64 `lua_settop(fLua.lua, 0);`, exécuté au tir d'un timer ou d'un public, vide la pile.

**Impact** — Le serveur ne tient que grâce à un effet de bord : le timer UpdateTime de CoreScripts (serverCore.lua:197), qui tire chaque seconde, remet la pile à zéro. Si un jeu de scripts n'a pas de timer périodique, ou si ce timer est arrêté, le processus s'arrête (abort) après ~65 500 événements. Les messages d'erreur de pcall restent aussi empilés.

**Recommandation** — - Pour les événements, appeler `lua_pcall(L, n, 0, msgh)`, puisque le résultat est ignoré.
- Dans Call(vector), lire le résultat puis faire `lua_pop(L, 1)`.
- Ajouter un garde RAII `struct StackGuard { lua_State* L; int top; ~StackGuard(){ lua_settop(L, top); } }` dans chaque Call, avec un assert en debug.
- Supprimer ensuite le `lua_settop(…, 0)` global de ScriptFunction::Call, qui masque le défaut.

**Contre-vérification (partiel (sévérité corrigée))** — La fuite est confirmée. LangLua.cpp:269-270 fait pcall(n,1) puis `LuaRef::fromStack(lua,-1)` (pushvalue puis ref, original non dépilé, LuaRef.h:828-834), et LuaException::whatFromStack ne dépile pas le message d'erreur. Le banc scr/stackleak rejoué donne `apres 60000 appels : lua_gettop=60000` puis `PANIC ... stack overflow`. Avec CoreScripts, UpdateTime fait RestartTimer chaque seconde hors de toute condition (serverCore.lua:165, après le `end` du if de la l.132), et ScriptFunction.cpp:64 remet la pile à 0 à chaque tir. Il faudrait donc environ 65 500 événements en moins d'une seconde : défaut latent, qui ne devient critique que pour des jeux de scripts sans timer périodique.

### SCR-8

**Message d'erreur Lua utilisé comme chaîne de format printf**

- **Sévérité** : 🟡 moyenne · **Catégorie** : securite · **Effort** : S
- **Emplacement** : `apps/openmw-mp/Script/Script.hpp:103`

**Preuve**

> Script.hpp:103 `LOG_MESSAGE_SIMPLE(TimedLog::LOG_ERROR, e.what());`, et main.cpp:311 fait de même. TimedLog.cpp:94 fait `sstr << message;`, puis TimedLog.cpp:103 `vsnprintf(buf.data(), buf.size(), sstr.str().c_str(), args);`. À l'inverse, ServerFunctions::LogMessage (Server.cpp:18) utilise correctement `"[Script]: %s"`.

**Impact** — Le texte d'une erreur Lua est interprété comme un format printf. Ce texte peut contenir des données de joueur, par exemple `error("commande invalide: " .. message)` ou un nom de compte. Des séquences '%s' ou '%n' provoquent alors des lectures de pile ou de registres, un crash, voire une écriture mémoire. C'est déclenchable à distance dès qu'un script inclut une entrée joueur dans une erreur.

**Recommandation** — - Écrire `LOG_MESSAGE_SIMPLE(TimedLog::LOG_ERROR, "%s", e.what());` aux deux endroits.
- Annoter `TimedLog::print` avec `__attribute__((format(printf, 6, 7)))`.
- Compiler avec -Werror=format-security pour attraper les cas futurs.

**Contre-vérification (confirmé)** — Script.hpp:103 et main.cpp:311 passent `e.what()` comme format à LOG_MESSAGE_SIMPLE (TimedLog.hpp:24). TimedLog.cpp:94 le concatène, puis l'utilise comme format de vsnprintf (l.100 et l.103), alors que Server.cpp:18 utilise bien `"[Script]: %s"`. Le déclenchement à distance dépend d'un script qui incorpore une donnée joueur dans le texte de l'erreur, d'où « moyenne » et non plus.

### SCR-9

**Aucun contrat d'API vérifié entre le serveur et CoreScripts : fonction appelée inexistante, aucun manifeste, aucun test**

- **Sévérité** : 🟡 moyenne · **Catégorie** : manque · **Effort** : M
- **Emplacement** : `apps/openmw-mp/Script/Functions/Cells.hpp:21`

**Preuve**

> CoreScripts 0.8.1 (commandHandler.lua:270) appelle `tes3mp.SetExterior(pid, cmd[2], cmd[3])`, alors que l'API n'expose que `{"SetExteriorCell", CellFunctions::SetExteriorCell}` (Cells.hpp:21). La comparaison scriptée donne :
> - 764 fonctions exportées, dont 671 utilisées par CoreScripts et 93 jamais utilisées ;
> - 1 fonction appelée qui n'existe pas ;
> - 37 fonctions dépréciées toujours exportées.
> 
> Apps/openmw_test_suite ne contient aucune référence au moteur de script.

**Impact** — La commande admin /setext lève 'attempt to call field SetExterior (a nil value)'. Avec la valeur par défaut config.ignoreScriptErrors=false (config.lua:451), le serveur s'arrête. Plus généralement, rien ne garantit que CoreScripts (dépôt séparé) et le serveur exposent la même API. Les signatures ne sont documentées qu'en C++, pas en Lua.

**Recommandation** — - Générer depuis ScriptFunctions::functions un manifeste via `tes3mp-server --dump-api=json` (nom, types 'iqsf…', type de retour, dépréciation).
- En dériver des stubs LuaLS/EmmyLua (`---@meta tes3mp`) publiés avec chaque release.
- Exposer `tes3mp.GetApiVersion()`.
- Dans CoreScripts, ajouter une CI LuaLS/luacheck qui échoue sur toute fonction tes3mp.* inconnue.
- Logger un WARN au premier appel d'une fonction dépréciée.

### SCR-13

**API à état global implicite : pointeurs read* non initialisés et tampons statiques partagés**

- **Sévérité** : ⚪ basse · **Catégorie** : architecture · **Effort** : L
- **Emplacement** : `apps/openmw-mp/Script/Functions/Objects.cpp:49`

**Preuve**

> Objects.cpp:13 déclare `BaseObjectList *readObjectList;`, qui n'est initialisé que par ReadReceivedObjectList (l.24). Objects.cpp:49 fait `return readObjectList->baseObjectCount;` sans test ; Actors.cpp:15 suit le même schéma. RecordsDynamic.cpp:14-31 définit environ 20 temporaires globaux (tempSpell, tempPotion…). Les fonctions renvoient des pointeurs vers `static std::string tempCellDescription` (Cells.cpp:12, Actors.cpp:22, Mechanics.cpp:11).

**Impact** — Le modèle Read…/Clear…/Set…/Send… est un automate à états implicite : appeler un getter avant Read… déréférence nullptr (SIGSEGV), et l'ordre d'appel n'est qu'une convention. Les const char* renvoyés sont écrasés à l'appel suivant : sans danger pour Lua, qui copie, mais dangereux pour le natif. Lire un paquet demande des centaines d'allers-retours Lua↔C (159 fonctions rien que pour Objects).

**Recommandation** — - Initialiser readObjectList et readActorList sur une liste vide statique.
- À moyen terme, ajouter des fonctions par lots qui renvoient des tables Lua, par exemple `tes3mp.GetReceivedObjects()` → tableau de {refId, refNum, mpNum, count, …} construit avec luabridge::newTable.
- Ajouter des fonctions d'envoi qui prennent une table, en conservant les getters unitaires pour la compatibilité.

### SCR-14

**PublicFnAPI : doublons ignorés en silence, DeleteAll invalide son itérateur, retour chaîne pendant**

- **Sévérité** : ⚪ basse · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw-mp/Script/API/PublicFnAPI.cpp:62`

**Preuve**

> PublicFnAPI.cpp:60-62 : `delete _public; publics.erase(it);`, suivi de `it++` dans le for, donc sur un itérateur invalidé. DeleteAll n'est jamais appelé (Script.cpp:82 `//Public::DeleteAll();`). `publics.emplace(name, this)` (l.13 et 19) ignore un doublon, et l'objet créé par `new Public(...)` (PublicFnAPI.hpp:23) fuit. ScriptFunction.cpp:55 renvoie `cast<const char*>()`, puis la ligne 64 vide la pile. ScriptFunctions.cpp:30 contient `catch (...) {}`.

**Impact** — Un MakePublic sur un nom déjà pris est ignoré sans message : le premier script garde la main. Une valeur de retour chaîne d'un public pointe vers une chaîne Lua qui n'est plus référencée. Les erreurs d'un CallPublic natif disparaissent. Ces défauts empêchent aussi tout futur rechargement de scripts.

**Recommandation** — - Utiliser `std::unordered_map<std::string, std::unique_ptr<Public>>`.
- Logger un WARN (ou lever une erreur Lua) en cas de doublon.
- Convertir les retours 's' en std::string avant lua_settop.
- Remplacer DeleteAll par `publics.clear()`.
- Journaliser les exceptions dans ScriptFunctions::CallPublic.

### SCR-15

**Messages d'erreur construits par arithmétique de pointeur, et index hors bornes**

- **Sévérité** : ⚪ basse · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw-mp/Script/LangLua/LangLua.cpp:263`

**Preuve**

> LangLua.cpp:263 et 315 : `throw std::runtime_error("C++ call: Unknown argument identifier " + argl[index]);`. ScriptFunction.cpp:61 : `"Lua call: Unknown return type" + ret_type`, signalé par clang : `adding 'char' to a string does not append to the string [-Wstring-plus-int]`. LuaFunc.cpp:53 et 166 lisent `types[i]` avec i égal à l'index de pile (décalé de 2 ou 4) au lieu de `types[i - args_begin]`.

**Impact** — Dans les chemins d'erreur, par exemple un type 'b', 'l' ou 'w' passé à CreateTimerEx ou CallPublic (cas commentés en LuaFunc.cpp:26-36), le message se construit à partir d'un pointeur décalé dans le littéral, ou au-delà, et d'un caractère hors chaîne. Le message est illisible ou implique une lecture hors limites, précisément au moment où un diagnostic serait utile.

**Recommandation** — - Écrire `std::string("… ") + argl[index]` et `types[i - args_begin]`.
- Gérer 'b', 'l' et 'w', ou les rejeter explicitement dans CreateTimerEx.
- Activer -Wall -Wextra -Werror pour apps/openmw-mp, et -Wstring-plus-int avec clang.

### SCR-16

**Dispatch d'événements : coût négligeable mais cache de présence figé et recherches par nom**

- **Sévérité** : ⚪ basse · **Catégorie** : performance · **Effort** : S
- **Emplacement** : `apps/openmw-mp/Script/Script.hpp:38`

**Preuve**

> Script.hpp:38 : `return reinterpret_cast<R>(lang->IsCallbackPresent(name));`, un bool converti en pointeur de fonction et mémorisé à vie. Script.hpp:84-87 enchaîne `callbacks_.count(I)`, `emplace` puis `callbacks_[I]`, soit trois recherches de hachage. LangLua::Call fait ensuite `lua_getglobal(lua, name)`. Banc LuaJIT : 57-71 ns par événement contre 29-35 ns avec une référence registre (luaL_ref + lua_rawgeti).

**Impact** — Le coût du dispatch n'est pas un goulot face à la logique de CoreScripts. En revanche :
- un handler défini après le premier événement (require tardif, chargement différé) n'est jamais appelé ;
- un handler retiré ensuite donne 'attempt to call a nil value', ce qui mène au chemin de crash (voir SCR-7).

**Recommandation** — Ne pas en faire une priorité. Lors du refactor SafeCall :
- faire une seule recherche avec `auto it = callbacks_.find(I)` ;
- remplacer le bool réinterprété par un état explicite (enum, ou std::optional<int> de luaL_ref) ;
- ou refaire le test de présence à chaque appel (lua_getglobal + lua_isfunction), qui ne coûte que quelques dizaines de ns.

### SCR-17

**Aucun sandbox ni budget d'exécution, et type de script détecté par sous-chaîne**

- **Sévérité** : ℹ️ info · **Catégorie** : securite · **Effort** : M
- **Emplacement** : `apps/openmw-mp/Script/LangLua/LangLua.cpp:37`

**Preuve**

> LangLua.cpp:37 : `luaL_openlibs(lua); // load all lua std libs`. Sous LuaJIT, cela charge io, os, ffi et jit, et aucun lua_sethook ni limite mémoire n'est posé. Script.cpp:23-30 choisit le langage par `strstr(path, ".so")`, puis `strstr(path, ".lua") || strstr(path, ".t")`, sur le chemin complet.

**Impact** — Les scripts, y compris les paquets tiers que les admins ajoutent, ont un accès complet au système et à la mémoire via FFI. Une boucle infinie fige le serveur sans recours. Si le chemin d'installation contient '.so' (par exemple /srv/tes3mp.solo/), serverCore.lua est chargé comme une bibliothèque native et le serveur ne démarre pas.

**Recommandation** — - Documenter le modèle de confiance des scripts.
- Ajouter une option `[Plugins] sandbox = true` qui retire ffi, os.execute et io.popen.
- Poser un hook LUA_MASKCOUNT qui émet un WARN ou une erreur au-delà d'un budget d'instructions par événement.
- Détecter le type de script par `std::filesystem::path(path).extension()`.

## Client multijoueur (mwmp)

| ID | Sévérité | Catégorie | Titre | Emplacement | Vérif. |
|---|---|---|---|---|---|
| [CLI-1](#cli-1) | 🟠 haute | bug | Cell::updateLocal incrémente deux fois l'itérateur après erase : boucle infinie, client autoritaire figé | `apps/openmw/mwmp/Cell.cpp:91` | confirmé |
| [CLI-2](#cli-2) | 🟠 haute | bug | runConsoleCommands parcourt un ObjectList que Console::execute vide puis réémet : use-after-free et écho au serveur | `apps/openmw/mwmp/ObjectList.cpp:960` | confirmé |
| [CLI-4](#cli-4) | 🟠 haute | performance | Chaque paquet de position d'un joueur distant détruit et recrée tous les widgets de marqueurs, 4 fois | `apps/openmw/mwmp/DedicatedPlayer.cpp:556` | confirmé |
| [CLI-3](#cli-3) | 🟡 moyenne | bug | PlayerList::getPlayer(guid) insère des nullptr via operator[] ; getPlayersInCell les déréférence (SIGSEGV) | `apps/openmw/mwmp/PlayerList.cpp:64` | partiel (sévérité corrigée) |
| [CLI-5](#cli-5) | 🟡 moyenne | performance | Positions et états envoyés en RELIABLE_ORDERED, à une cadence qui dépend du FPS (blocage en tête de file) | `apps/openmw/mwmp/LocalPlayer.cpp:104` | partiel (sévérité corrigée) |
| [CLI-6](#cli-6) | 🟡 moyenne | performance | Interpolation naïve des joueurs et PNJ distants, et moveObject appelé à chaque frame même à l'arrêt | `apps/openmw/mwmp/DedicatedPlayer.cpp:133` | non soumis |
| [CLI-7](#cli-7) | 🟡 moyenne | performance | Copies profondes à chaque tick : NpcStats et CreatureStats copiés, BaseActor (2 760 o) passé par valeur | `apps/openmw/mwmp/LocalPlayer.cpp:601` | non soumis |
| [CLI-8](#cli-8) | 🟡 moyenne | performance | Index d'acteurs à base de chaînes construites par ostringstream, sur le chemin chaud (par acteur et par frame) | `apps/openmw/mwmp/CellController.cpp:386` | non soumis |
| [CLI-9](#cli-9) | 🟡 moyenne | performance | En l'air, le joueur envoie 2 positions et 1 paquet de drapeaux d'animation à chaque tick | `apps/openmw/mwmp/LocalPlayer.cpp:629` | non soumis |
| [CLI-10](#cli-10) | 🟡 moyenne | performance | Variables de scripts synchronisées : un paquet fiable par modification, et une liste blanche à recherche linéaire sensible à la casse | `apps/openmw/mwmp/Main.cpp:248` | non soumis |
| [CLI-11](#cli-11) | 🟡 moyenne | bug | setFactions boucle indéfiniment si le rang reçu dépasse le rang maximal de la faction | `apps/openmw/mwmp/LocalPlayer.cpp:1374` | confirmé |
| [CLI-14](#cli-14) | 🟡 moyenne | securite | Identifiants : mot de passe serveur transmis en clair, et hachage client sans sel donc rejouable | `apps/openmw/mwmp/GUIController.cpp:180` | confirmé |
| [CLI-15](#cli-15) | 🟡 moyenne | maintenabilite | Aucun test, état global dans des variables static, mémoire gérée à la main : un frein majeur à l'évolution | `apps/openmw/mwmp/LocalPlayer.cpp:369` | non soumis |
| [CLI-12](#cli-12) | ⚪ basse | bug | Exceptions sur données réseau : paquet RakNet jamais libéré, frame interrompue, lecture hors limites d'un octet | `apps/openmw/mwmp/Networking.cpp:228` | partiel (sévérité corrigée) |
| [CLI-13](#cli-13) | ⚪ basse | bug | Connexion : attente active sans délai maximal, gel possible au démarrage | `apps/openmw/mwmp/Networking.cpp:290` | partiel (sévérité corrigée) |
| [CLI-16](#cli-16) | ⚪ basse | bug | Marqueurs fantômes après déconnexion et membres de DedicatedPlayer jamais initialisés | `apps/openmw/mwmp/PlayerList.cpp:49` | non soumis |

### CLI-1

**Cell::updateLocal incrémente deux fois l'itérateur après erase : boucle infinie, client autoritaire figé**

- **Sévérité** : 🟠 haute · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw/mwmp/Cell.cpp:91`

**Preuve**

> Cell.cpp:91 `localActors.erase(it++);` est suivi de Cell.cpp:101 `++it;`, exécuté aussi dans cette branche (else de la ligne 80). Le micro-test reproduisant le motif sur une std::map à 1 élément (scratchpad/cli/map_erase.cpp) ne termine pas : `timeout 5 ./map_erase; echo exit=$?` donne `exit=124`.

**Impact** — Le cas se produit quand une LocalActor encore activée est supprimée : cadavre « disposé », ID_OBJECT_DELETE envoyé par le serveur ou invocation expirée. Si c'était le dernier acteur local de la cellule, `++it` sur end() d'une map vide boucle indéfiniment dans _Rb_tree_increment et le client autoritaire se fige (builds libstdc++). Sinon, l'acteur suivant est sauté ou retraité pendant ce tick, ce qui reste un comportement indéfini.

**Recommandation** — Dans la branche isDeleted(), écrire `cellController->removeLocalActorRecord(mapIndex); delete actor; it = localActors.erase(it); continue;` et ne garder `++it` que pour les chemins qui ne suppriment rien. Ajouter un test unitaire de Cell::updateLocal avec un acteur supprimé seul dans la cellule.

**Contre-vérification (confirmé)** — Vérifié dans Cell.cpp : la boucle `for (auto it = localActors.begin(); it != localActors.end();)` (ligne 50) exécute `localActors.erase(it++);` (ligne 91, branche isEnabled && isDeleted), puis le `++it;` inconditionnel de la ligne 101. Rien en amont ne retire l'acteur avant ce point : seul Cell.cpp:89-91 appelle removeLocalActorRecord pour ce cas. Le chemin est atteignable, car une référence supprimée garde isEnabled() vrai. Si l'acteur était le dernier de la map, on incrémente end(), ce qui donne une boucle infinie sur une map vide sous libstdc++.

### CLI-2

**runConsoleCommands parcourt un ObjectList que Console::execute vide puis réémet : use-after-free et écho au serveur**

- **Sévérité** : 🟠 haute · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw/mwmp/ObjectList.cpp:960`

**Preuve**

> ObjectList.cpp:960 `for (const auto &baseObject : baseObjects)` appelle :980 `windowManager->executeCommandInConsole(consoleCommand);`, qui exécute mwgui/console.cpp:226 `objectList->reset();` sur la même instance Networking::objectList, puis :234 addObjectGeneric et :237 sendConsoleCommand. Reproduction du motif sous ASan (scratchpad/cli/vec_clear.cpp) : `ERROR: AddressSanitizer: heap-use-after-free ... READ of size 2`.

**Impact** — Dès qu'un script serveur exécute une commande console (disable, enable, lock…) sur au moins 2 objets, les itérations suivantes lisent des BaseObject détruits. Pour un refId de plus de 15 caractères, c'est de la mémoire libérée : commande appliquée au mauvais objet, ou crash. De plus, chaque commande reçue du serveur lui est renvoyée en ID_CONSOLE_COMMAND (trafic parasite, risque de boucle côté scripts).

**Recommandation** — Dans runConsoleCommands, copier `const auto objects = baseObjects; const std::string cmd = consoleCommand;` avant la boucle. Ajouter `bool fromServer` à Console::execute et à executeCommandInConsole pour ne pas appeler reset() ni envoyer ID_CONSOLE_COMMAND. Plus largement, séparer le tampon de réception de celui d'émission : getObjectList() ne doit pas renvoyer l'objet passé à ObjectProcessor::Process.

**Contre-vérification (confirmé)** — Networking.cpp:448 passe le membre `objectList` au processeur, et getObjectList() (Networking.cpp:500) renvoie `&objectList` : c'est donc la même instance. Console::execute (mwgui/console.cpp:225-237) appelle sans condition `objectList->reset()`, qui fait `baseObjects.clear()` (ObjectList.cpp:62), puis addObjectGeneric et sendConsoleCommand. Le range-for d'ObjectList.cpp:960 lit donc des éléments détruits dès le 2e objet. Chaque commande reçue est en outre renvoyée au serveur, et aucun garde-fou n'a été trouvé.

### CLI-4

**Chaque paquet de position d'un joueur distant détruit et recrée tous les widgets de marqueurs, 4 fois**

- **Sévérité** : 🟠 haute · **Catégorie** : performance · **Effort** : M
- **Emplacement** : `apps/openmw/mwmp/DedicatedPlayer.cpp:556`

**Preuve**

> ProcessorPlayerPosition.hpp:31 `->updateMarker();` appelle DedicatedPlayer.cpp:556-558 `deleteMarker(marker); marker = gui->createMarker(guid); addMarker(marker);`, avec PlayerMarkerCollection.hpp:18 `bool triggerEvent = true`. Chaque événement appelle updatePlayersMarkers, qui fait `destroyWidget` (GUIController.cpp:332), recrée les widgets puis `redraw()` (ligne 371). Deux LocalMapBase sont abonnés (mapwindow.cpp:196) : le HUD (hud.cpp:110) et MapWindow (mapwindow.cpp:704).

**Impact** — Chaque joueur distant en mouvement envoie environ 50 à 66 positions par seconde, soit de l'ordre de 200 à 260 reconstructions complètes de marqueurs par seconde. Avec 10 joueurs visibles, on dépasse 2 000 reconstructions MyGUI par seconde sur le thread principal, même carte fermée puisque la minimap du HUD est toujours abonnée. Le FPS baisse proportionnellement au nombre de joueurs.

**Recommandation** — Ne mettre à jour un marqueur que si sa cellule ou sa position en pixels sur la carte change, à 2 à 4 Hz au plus. Ajouter PlayerMarkerCollection::moveMarker(guid, x, y), qui modifie en place et ne déclenche qu'un seul eventMarkersChanged. Dans updatePlayersMarkers, réutiliser les widgets existants (setCoord) au lieu de les détruire et recréer. À défaut, appeler addMarker(marker, false) après deleteMarker.

**Contre-vérification (confirmé)** — ProcessorPlayerPosition.hpp appelle updateMarker() pour chaque paquet d'un joueur distant. DedicatedPlayer.cpp:556-558 enchaîne alors deleteMarker, qui déclenche eventMarkersChanged (PlayerMarkerCollection.cpp:23), puis addMarker avec triggerEvent=true par défaut (PlayerMarkerCollection.cpp:10-11), soit 2 événements. Les deux LocalMapBase (HUD et MapWindow) sont abonnés (mapwindow.cpp:196), d'où 4 appels à updatePlayersMarkers. Chacun détruit puis recrée tous les widgets de marqueurs et fait un redraw (GUIController.cpp:330-371). Le serveur relaie les positions à tous les joueurs des cellules chargées (openmw-mp ProcessorPlayerPosition.hpp:17 sendToLoaded), donc le coût croît en O(N²) sur le thread principal.

### CLI-10

**Variables de scripts synchronisées : un paquet fiable par modification, et une liste blanche à recherche linéaire sensible à la casse**

- **Sévérité** : 🟡 moyenne · **Catégorie** : performance · **Effort** : M
- **Emplacement** : `apps/openmw/mwmp/Main.cpp:248`

**Preuve**

> mwscript/interpretercontext.cpp:224-230 exécute `if (sendPackets) { ... objectList->reset(); ... sendClientScriptLocal(); }` à chaque changement d'un short ou d'un long. Main.cpp:248 `bool Main::isValidPacketScript(std::string scriptId)` utilise Utils::vectorContains (std::find, comparaison exacte) et est appelé pour chaque script local à chaque frame (engine.cpp:252).

**Impact** — Un script autorisé qui incrémente un compteur à chaque frame (motif courant `set timer to timer + 1`) émet environ 60 ID_CLIENT_SCRIPT_LOCAL par seconde et par objet, tous fiables et ordonnés. La liste blanche est copiée et parcourue linéairement pour chaque script local à chaque frame. Un identifiant dont la casse diffère n'est jamais synchronisé.

**Recommandation** — Accumuler les variables modifiées dans un ensemble « dirty » vidé une fois par tick réseau, avec un paquet groupé par cellule. Limiter la fréquence par variable, par exemple à 10 Hz. Stocker les listes blanches dans des std::unordered_set en minuscules et passer les identifiants en const std::string&.

### CLI-11

**setFactions boucle indéfiniment si le rang reçu dépasse le rang maximal de la faction**

- **Sévérité** : 🟡 moyenne · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw/mwmp/LocalPlayer.cpp:1374`

**Preuve**

> LocalPlayer.cpp:1374 `while (faction.rank != ptrNpcStats.getFactionRanks().at(faction.factionId))`. Or mwmechanics/npcstats.cpp:87 `if (it->second+1 < 10 && !factionPtr->mRanks[it->second+1].empty()) it->second += 1;` ne fait rien au rang maximal, et lowerRank efface l'entrée en dessous de 0 (lignes 99-102).

**Impact** — Un ID_PLAYER_FACTION dont le rang dépasse le maximum (script serveur bogué, commande admin) fige définitivement le client. Un rang négatif, ou un factionId qui n'est pas en minuscules (les clés de la map le sont), fait lever std::out_of_range et interrompt la frame.

**Recommandation** — Mettre factionId en minuscules et borner le rang à [0, dernier rang non vide]. L'affecter directement via une nouvelle méthode NpcStats::setFactionRank au lieu de boucler sur raiseRank et lowerRank. À défaut, limiter la boucle à 10 itérations.

**Contre-vérification (confirmé)** — Rien ne borne faction.rank entre la lecture (PacketPlayerFaction.cpp:35, `RW(faction.rank, send)`) et la boucle de LocalPlayer.cpp:1374. raiseRank (npcstats.cpp:87) ne fait rien au rang maximal ni au-delà de 9 : la boucle devient infinie. lowerRank efface l'entrée sous 0 (npcstats.cpp:99-102), et `.at(faction.factionId)` lève alors out_of_range. Il en va de même pour un factionId qui n'est pas en minuscules, puisque joinFaction et raiseRank passent les clés en minuscules alors que at() reçoit l'identifiant brut.

### CLI-14

**Identifiants : mot de passe serveur transmis en clair, et hachage client sans sel donc rejouable**

- **Sévérité** : 🟡 moyenne · **Catégorie** : securite · **Effort** : L
- **Emplacement** : `apps/openmw/mwmp/GUIController.cpp:180`

**Preuve**

> GUIController.cpp:180-181 : `textInput = picosha2::hash256_hex_string(textInput);` puis `hash256_hex_string(textInput + hash256_hex_string(hash256_hex_string(textInput)))`. PacketSystemHandshake.cpp:17 : `RW(system->serverPassword, send, true, maxPasswordLength)`. Une recherche d'InitializeSecurity dans apps et components ne renvoie rien.

**Impact** — Le mot de passe du serveur et l'empreinte du mot de passe du joueur circulent en clair sur UDP (compression RakString, sans chiffrement). L'empreinte est déterministe et sans sel : elle vaut le mot de passe pour quiconque l'intercepte, et se casse par dictionnaire.

**Recommandation** — Activer le chiffrement du transport de CrabNet/RakNet, ou DTLS. Remplacer l'envoi de l'empreinte par un challenge-response (nonce serveur plus HMAC). Côté serveur, stocker un hachage lent et salé (Argon2id ou bcrypt).

**Contre-vérification (confirmé)** — GUIController.cpp:180-181 calcule un double SHA-256 déterministe sans sel ni défi serveur ; l'empreinte est rejouable telle quelle. PacketSystemHandshake.cpp:17 transmet serverPassword en RakString, et aucun InitializeSecurity/SetEncryption n'existe dans apps/ ni dans components/, donc il n'y a pas de chiffrement. De plus, le serveur journalise en clair le mot de passe erroné (apps/openmw-mp/Networking.cpp:121, `Wrong server password %s`).

### CLI-15

**Aucun test, état global dans des variables static, mémoire gérée à la main : un frein majeur à l'évolution**

- **Sévérité** : 🟡 moyenne · **Catégorie** : maintenabilite · **Effort** : L
- **Emplacement** : `apps/openmw/mwmp/LocalPlayer.cpp:369`

**Preuve**

> LocalPlayer.cpp:369 `static bool posWasChanged = false;` et LocalPlayer.cpp:216 `static MWMechanics::DynamicStat<float> oldHealth(...)` ; mwmp compte 21 static locales, dont 17 dans LocalPlayer.cpp. `grep -rl mwmp apps/openmw_test_suite | wc -l` renvoie 0. clang : `CellController.cpp:451:10: warning: reference cannot be bound to dereferenced null pointer ... [-Wtautological-undefined-compare]`.

**Impact** — L'état de détection des changements vit dans des static partagées, que rien ne réinitialise lors d'un changement de personnage ou d'un reset serveur, et il ne peut pas être testé isolément. Les 15 new bruts et les pointeurs bruts vers des objets du monde rendent faciles des bugs de durée de vie comme CLI-1 à CLI-3. RecordHelper.cpp répète 25 fois le même schéma overrideRecord (1 598 lignes). Le test `&cell == nullptr` est supprimé par le compilateur, et GUILogin.cpp est du code mort.

**Recommandation** — Déplacer les static dans des membres de LocalPlayer et passer PlayerList, Cell et CellController à std::unique_ptr. Factoriser RecordHelper avec un template et des traits par type d'enregistrement. Supprimer GUILogin et le test de nullité sur une référence. Créer une cible gtest qui couvre CellController, Cell, PlayerList et la détection de changements de LocalPlayer, avec un monde factice, plus un rejeu de paquets capturés. Compiler mwmp en CI avec -Wall -Wextra, -Werror sur une sélection d'avertissements, et ASan.

### CLI-3

**PlayerList::getPlayer(guid) insère des nullptr via operator[] ; getPlayersInCell les déréférence (SIGSEGV)**

- **Sévérité** : 🟡 moyenne (initialement haute) · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw/mwmp/PlayerList.cpp:64`

**Preuve**

> PlayerList.cpp:64 `return playerList[guid];` insère une entrée nulle pour tout GUID inconnu. Cette fonction est appelée par PlayerProcessor.cpp:37, MechanicsHelper::getPlayerPtr et processAttack. PlayerList.cpp:108 `isSameCell(cell, playerEntry.second->cell)` ne teste pas la nullité ; elle est appelée par Worldstate.cpp:491 (ID_CELL_RESET).

**Impact** — Un paquet tardif qui référence un joueur déconnecté (cible d'attaque, lanceur d'un sort actif, IA d'un PNJ) laisse une entrée nulle dans la map. Le prochain reset de cellule fait alors planter le client par SIGSEGV, que le try/catch de Engine::frame ne rattrape pas. deletePlayer (lignes 49-53) a le même défaut pour un GUID inconnu, et la map grossit sans limite.

**Recommandation** — Écrire `auto it = playerList.find(guid); return it == playerList.end() ? nullptr : it->second.get();` et stocker des `std::unique_ptr<DedicatedPlayer>`. Ajouter un test de nullité dans getPlayersInCell et deletePlayer, et utiliser find() partout.

**Contre-vérification (partiel (sévérité corrigée))** — PlayerList.cpp:64 fait bien `return playerList[guid];` et insère nullptr. PlayerProcessor.cpp:37 l'appelle pour tout GUID distant : c'est le cas normal avant ID_PLAYER_BASEINFO (ProcessorPlayerBaseInfo.hpp:39-42 écrase ensuite l'entrée), mais une entrée nulle persiste pour un GUID jamais créé ou déjà supprimé. getPlayersInCell (PlayerList.cpp:108) est la seule boucle sans test de nullité : enableMarkers (ligne 134) et getPlayer(ptr) (ligne 71) en ont un, ce qui prouve que des nullptr existent bien. Le crash demande en revanche un ID_CELL_RESET, une commande serveur rare, et deletePlayer n'est appelé que si player != 0 (ProcessorUserDisconnected.hpp:25,41). La sévérité est donc moyenne.

### CLI-5

**Positions et états envoyés en RELIABLE_ORDERED, à une cadence qui dépend du FPS (blocage en tête de file)**

- **Sévérité** : 🟡 moyenne (initialement haute) · **Catégorie** : performance · **Effort** : M
- **Emplacement** : `apps/openmw/mwmp/LocalPlayer.cpp:104`

**Preuve**

> LocalPlayer.cpp:104-108 : `const float timeoutSec = 0.015;` puis `updateTimer = 0;`. PacketPlayerPosition.cpp:10 : `//reliability = UNRELIABLE_SEQUENCED;`. PlayerPacket.cpp:12 et ActorPacket.cpp:12 : `reliability = RELIABLE_ORDERED;`, avec un seul canal ordonné par famille (CHANNEL_PLAYER, CHANNEL_ACTOR).

**Impact** — Le joueur local émet environ 48 à 66 positions par seconde : 60 Hz à 60 FPS mais 48 Hz à 144 FPS, car le timer est remis à zéro au lieu d'être décrémenté. Les PNJ émettent à 40 Hz par cellule. Tout est fiable et ordonné, donc une seule perte bloque les positions, stats, chat et autres paquets du même canal jusqu'à la retransmission (au moins 1 RTT), puis les livre en rafale. Résultat : à-coups et rubber-banding chez tous les observateurs, avec un débit relayé par le serveur en O(N²).

**Recommandation** — Passer ID_PLAYER_POSITION, ID_ACTOR_POSITION et ID_*_ANIM_FLAGS en UNRELIABLE_SEQUENCED sur un canal dédié, avec un envoi fiable « keyframe » à l'arrêt et au changement de cellule. Fixer un tick réseau de 20 à 30 Hz avec un accumulateur (`updateTimer -= timeoutSec`). Ajouter une séquence et un horodatage au paquet, ce qui demande d'incrémenter TES3MP_PROTO_VERSION et d'adapter le serveur.

**Contre-vérification (partiel (sévérité corrigée))** — Les faits sont exacts. LocalPlayer.cpp:103-107 contient `timeoutSec = 0.015` et `updateTimer = 0`, ce qui donne environ 60 Hz à 60 FPS et 48 Hz à 144 FPS. PacketPlayerPosition.cpp:10 contient `//reliability = UNRELIABLE_SEQUENCED;`, et PlayerPacket.cpp/ActorPacket.cpp fixent RELIABLE_ORDERED sur un canal unique. Cell.cpp:37 confirme 40 Hz pour les PNJ. En revanche, l'envoi n'a lieu que pendant le mouvement (LocalPlayer.cpp:389, posIsChanging || posWasChanged). Le blocage en tête de file ne se manifeste qu'en cas de perte, et l'impact n'est pas mesuré : c'est un choix de conception coûteux plutôt qu'un goulot majeur avéré.

### CLI-6

**Interpolation naïve des joueurs et PNJ distants, et moveObject appelé à chaque frame même à l'arrêt**

- **Sévérité** : 🟡 moyenne · **Catégorie** : performance · **Effort** : L
- **Emplacement** : `apps/openmw/mwmp/DedicatedPlayer.cpp:133`

**Preuve**

> DedicatedPlayer.cpp:123 `const int maxInterpolationDistance = 80;`. DedicatedPlayer.cpp:133-136 `timeMultiplier = 15; ... getLinearInterpolation(refPos.asVec3(), position.asVec3(), dt * timeMultiplier); world->moveObject(...)`. DedicatedActor.cpp:82 fixe un seuil de 40, et DedicatedActor.cpp:97/106 appellent moveObject et rotateObject sans condition à chaque frame.

**Impact** — Le lissage est exponentiel vers le dernier échantillon, sans horodatage ni tampon. Le retard visuel est d'environ 67 ms et la vitesse apparente varie avec le FPS. Le facteur dépasse 1 quand dt > 0,067 s (dépassement et oscillation). L'entité est téléportée dès 80 ou 40 unités d'écart sur un axe, ce qui arrive en course rapide ou après une rafale de retransmissions. En parallèle, `direction` est injectée dans les MovementSettings, donc le contrôleur de personnage déplace aussi l'acteur : deux sources de mouvement se concurrencent et produisent de la gigue. Enfin, chaque PNJ dédié immobile coûte un moveObject (physique, scène) et un rotateObject par frame.

**Recommandation** — Utiliser un tampon de 2 ou 3 snapshots horodatés et afficher avec un délai fixe d'environ 100 ms, en interpolant entre deux snapshots. Ajouter une extrapolation bornée à 200 ms à partir de la vitesse. Borner le facteur d'interpolation à [0,1] et rendre le seuil de téléportation proportionnel à la vitesse. Sauter moveObject et rotateObject si le déplacement est inférieur à 0,5 unité et la rotation inférieure à 0,001 rad.

### CLI-7

**Copies profondes à chaque tick : NpcStats et CreatureStats copiés, BaseActor (2 760 o) passé par valeur**

- **Sévérité** : 🟡 moyenne · **Catégorie** : performance · **Effort** : S
- **Emplacement** : `apps/openmw/mwmp/LocalPlayer.cpp:601`

**Preuve**

> LocalPlayer.cpp:601 `MWMechanics::NpcStats ptrNpcStats = ptrPlayer.getClass().getNpcStats(ptrPlayer);`. LocalActor.cpp:116 `MWMechanics::CreatureStats ptrCreatureStats = ptr.getClass().getCreatureStats(ptr);`. ActorList.cpp:51 `void ActorList::addPositionActor(BaseActor baseActor)` puis ActorList.cpp:144 `baseActors = positionActors;`. Tailles mesurées : NpcStats 1 672 o, CreatureStats 952 o, BaseActor 2 760 o.

**Impact** — À environ 60 Hz, la copie de NpcStats duplique Spells, ActiveSpells, MagicEffects, AiSequence (avec clonage des paquets d'IA), les maps de factions et mUsedIds, qui grossit pendant toute la partie. Même chose pour chaque PNJ local toutes les 25 ms. Chaque PNJ ajouté à ActorList est en outre copié 3 fois (paramètre, push_back, affectation du vecteur), avec environ 45 std::string. Cela crée une pression d'allocation proportionnelle au nombre de PNJ ; clang-tidy signale 55 performance-unnecessary-value-param.

**Recommandation** — Remplacer par `const auto& stats = ptr.getClass().getNpcStats(ptr);` (et l'équivalent pour CreatureStats). Prendre `const BaseActor&` dans les add*Actor. Remplacer `baseActors = positionActors` par `baseActors.swap(positionActors)`. À terme, définir des structures de paquet légères par type, par exemple une position = refNum, mpNum et 2 ESM::Position.

### CLI-8

**Index d'acteurs à base de chaînes construites par ostringstream, sur le chemin chaud (par acteur et par frame)**

- **Sévérité** : 🟡 moyenne · **Catégorie** : performance · **Effort** : M
- **Emplacement** : `apps/openmw/mwmp/CellController.cpp:386`

**Preuve**

> CellController.cpp:386 `mapIndex = Utils::toString(refNum) + "-" + Utils::toString(mpNum);`, où Utils.cpp:113 crée un `std::ostringstream`. Cette clé sert à isLocalActor et getLocalActor (lignes 307/324), appelés par CharacterController::update (character.cpp:2147-2149) et Actors::update (actors.cpp:2177-2178). Benchmark : `string+ostringstream+map: 282 ns/lookup ; uint64+unordered_map: 4.6 ns/lookup ; ratio x61`.

**Impact** — Chaque acteur subit environ 6 recherches par frame, car isX puis getX reconstruisent la chaîne puis font un .at(). Pour 100 acteurs à 60 FPS, cela fait environ 36 000 ostringstream par seconde, soit à peu près 10 ms de CPU par seconde. S'y ajoutent PlayerList::isDedicatedPlayer (parcours linéaire avec comparaison de refId, appelé dans des boucles acteur × acteur de getActorsSidingWith) et CellStore::searchExact, qui balaie toutes les références de la cellule pour chaque acteur ou objet reçu (Cell.cpp:548 et 23 appels dans ObjectList.cpp).

**Recommandation** — Utiliser une clé `uint64_t key = (uint64_t)refNum << 32 | mpNum` dans un `std::unordered_map<uint64_t, LocalActor*>` direct, et exposer une seule API `findLocalActor(const Ptr&)` qui renvoie nullptr si absent. Marquer les Ptr des DedicatedPlayer (drapeau dans RefData ou ensemble de LiveCellRefBase*). Indexer (refNum, mpNum) vers LiveCellRefBase* dans CellStore.

### CLI-9

**En l'air, le joueur envoie 2 positions et 1 paquet de drapeaux d'animation à chaque tick**

- **Sévérité** : 🟡 moyenne · **Catégorie** : performance · **Effort** : S
- **Emplacement** : `apps/openmw/mwmp/LocalPlayer.cpp:629`

**Preuve**

> LocalPlayer.cpp:629 déclenche l'envoi sur la condition `wasJumping || isJumping || ...`, puis LocalPlayer.cpp:655 appelle `updatePosition(true); // fix position after jump;`, alors que update() a déjà appelé updatePosition() au même tick (ligne 110).

**Impact** — Pendant tout un saut ou une chute, et plus largement tant que isOnGround est faux sans vol (peut-être aussi à la nage), le client émet environ 3 paquets fiables par tick, soit 150 à 200 par seconde, au lieu d'un seul. Le serveur les relaie à tous les joueurs concernés.

**Recommandation** — N'envoyer ID_PLAYER_ANIM_FLAGS qu'aux transitions : début et fin de saut, ou changement de drapeaux. Ne pas forcer updatePosition(true) si une position a déjà été envoyée pendant ce tick (drapeau positionSentThisTick).

### CLI-12

**Exceptions sur données réseau : paquet RakNet jamais libéré, frame interrompue, lecture hors limites d'un octet**

- **Sévérité** : ⚪ basse (initialement moyenne) · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw/mwmp/Networking.cpp:228`

**Preuve**

> Networking.cpp:228 `for (packet=peer->Receive(); packet; peer->DeallocatePacket(packet), packet=peer->Receive())` n'a pas de try/catch autour de receiveMessage. Cell.cpp:574 `delete dedicatedActors.at(mapIndex);` lève une exception pour les acteurs ignorés par initializeDedicatedActors. PlayerProcessor.cpp:17 `RakNet::BitStream bsIn(&packet.data[1], packet.length, false);`, et le même motif se retrouve dans les 4 autres processeurs.

**Impact** — Toute exception remonte jusqu'à Engine::frame (« Error in frame ») ; mwmp contient 32 .at(), dont certains sur des clés issues du réseau. Le paquet en cours n'est alors jamais libéré, et les paquets suivants comme la mise à jour du monde sont sautés pour cette frame. Par ailleurs, le BitStream déclare length octets à partir de data[1] : un paquet malformé peut faire lire un octet au-delà du tampon.

**Recommandation** — Entourer receiveMessage d'un try/catch à l'intérieur de la boucle, et libérer le paquet par RAII (`std::unique_ptr<RakNet::Packet, Deleter>` dont le deleter appelle DeallocatePacket). Remplacer les .at() sur des clés réseau par find() plus un log. Construire le BitStream avec `packet.length - 1`.

**Contre-vérification (partiel (sévérité corrigée))** — Confirmé : Networking.cpp:228-257 n'a pas de try/catch, donc l'exception remonte jusqu'à Engine::frame (engine.cpp:456-458) et le paquet en cours n'est jamais désalloué. Cell.cpp:574 `.at()` peut lever une exception via readEquipment → uninitializeDedicatedActors(actorList) (Cell.cpp:277-278) quand initializeDedicatedActors a sauté l'acteur (`if (!ptrFound) continue;`, ligne 550). Le BitStream de PlayerProcessor.cpp:17 déclare bien un octet de trop. L'impact est toutefois exagéré : les paquets suivants restent dans la file RakNet et sont traités à la frame suivante, pas perdus. Il en résulte une fuite d'un paquet, une frame tronquée et une lecture d'un octet sur un paquet émis par le serveur.

### CLI-13

**Connexion : attente active sans délai maximal, gel possible au démarrage**

- **Sévérité** : ⚪ basse (initialement moyenne) · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw/mwmp/Networking.cpp:290`

**Preuve**

> Networking.cpp:286 `if (peer->Connect(...) != RakNet::CONNECTION_ATTEMPT_STARTED)` renseigne seulement errmsg. La boucle Networking.cpp:290 `while (queue)` ne s'arrête que sur réception d'un paquet, et ne fait aucune pause. Le pré-init, Networking.cpp:386-391 `while (!done) { ... RakSleep(500); continue; }`, n'a pas d'échéance. Main.cpp:149 : `pMain->port = atoi(address.substr(delimPos + 1).c_str());`.

**Impact** — Si Connect échoue immédiatement (CANNOT_RESOLVE_DOMAIN_NAME, INVALID_PARAMETER), aucun paquet n'arrivera : la boucle tourne indéfiniment à 100 % d'un cœur et l'erreur n'est jamais affichée. Même en fonctionnement normal, l'attente consomme un cœur. Le pré-init attend indéfiniment un serveur muet et ajoute jusqu'à 500 ms de latence. Enfin, `--connect=host` sans port donne port = atoi(host), par exemple 127 pour 127.0.0.1.

**Recommandation** — Sortir de la boucle dès que errmsg est renseigné, ajouter une échéance de 10 s et un RakSleep(10) entre deux Receive. Faire de même pour preInit, avec délai maximal et message d'erreur. Parser host[:port] avec 25565 comme port par défaut et accepter la forme IPv6 [adresse]:port.

**Contre-vérification (partiel (sévérité corrigée))** — Confirmé pour Networking.cpp:286-290 : un échec immédiat de Connect renseigne seulement errmsg, et `while (queue)` tourne sans pause ni échéance. Main.cpp:149 donne atoi(host) quand le port manque, par exemple port 0 pour `--connect=localhost`, ce qui rend plausible un gel au démarrage. La partie sur le pré-init est en revanche réfutée : Networking.cpp:400-404 termine la boucle sur ID_CONNECTION_LOST, que RakNet émet à l'expiration de son délai, donc l'attente n'est pas infinie. Le cas reste limité au démarrage et à une saisie locale.

### CLI-16

**Marqueurs fantômes après déconnexion et membres de DedicatedPlayer jamais initialisés**

- **Sévérité** : ⚪ basse · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw/mwmp/PlayerList.cpp:49`

**Preuve**

> PlayerList.cpp:49-52 `if (playerList[guid]->reference) playerList[guid]->deleteReference(); delete playerList[guid];` n'appelle pas removeMarker(), dont le seul appel est DedicatedPlayer.cpp:422. DedicatedPlayer.cpp lit `isLevitationPurged` ligne 201, `markerEnabled` ligne 547 et `previousDisplayCreatureName` ligne 332, alors que le constructeur (lignes 43-68) ne les initialise pas.

**Impact** — Les joueurs déconnectés restent sur la carte locale et la minimap jusqu'à la fermeture du client. Lire ces booléens non initialisés est un comportement indéfini : la lévitation peut être purgée ou non au premier tick, le marqueur actif ou non, et le déguisement en créature réappliqué à tort.

**Recommandation** — Appeler removeMarker() dans deletePlayer avant le delete, ou dans ~DedicatedPlayer. Initialiser ces membres dans la déclaration : `bool markerEnabled = false; bool isLevitationPurged = false; bool previousDisplayCreatureName = false;`.

## Intégration dans OpenMW & divergence amont

| ID | Sévérité | Catégorie | Titre | Emplacement | Vérif. |
|---|---|---|---|---|---|
| [INT-1](#int-1) | 🔴 critique | bug | Store<ESM::Cell>::override stocke un pointeur vers une variable de pile et peut sortir sans valeur de retour | `apps/openmw/mwworld/store.cpp:810` | confirmé |
| [INT-2](#int-2) | 🟠 haute | bug | La réinitialisation de cellule laisse un pointeur pendant dans mIdCache, et clearMovesToCells n'efface rien | `apps/openmw/mwworld/cells.cpp:107` | confirmé |
| [INT-3](#int-3) | 🟠 haute | bug | updatePtrsWithRefId modifie le vecteur qu'il itère, et fuit un ManualRef par objet | `apps/openmw/mwworld/worldimp.cpp:867` | confirmé |
| [INT-4](#int-4) | 🟠 haute | bug | ActionTeleport : newCellStore déréférencé sans initialisation quand le suiveur est en combat avec le joueur | `apps/openmw/mwworld/actionteleport.cpp:126` | confirmé |
| [INT-5](#int-5) | 🟠 haute | architecture | Couplage direct et massif d'OpenMW au singleton mwmp::Main, sans interface de hooks | `apps/openmw/mwscript/miscextensions.cpp:212` | confirmé |
| [INT-6](#int-6) | 🟠 haute | maintenabilite | Divergence amont non soutenable : les 162 fichiers patchés encore présents ont tous changé entre 0.47 et 0.50 | `apps/openmw/mwworld/worldimp.cpp:836` | confirmé |
| [INT-7](#int-7) | 🟡 moyenne | maintenabilite | Balisage incohérent : balises non appariées, variantes non canoniques et modifications non balisées | `apps/openmw/mwworld/actionteleport.cpp:109` | non soumis |
| [INT-8](#int-8) | 🟡 moyenne | bug | setPhysicsFramerate est sans effet avec la physique multithread de 0.47 | `apps/openmw/mwphysics/physicssystem.cpp:189` | confirmé |
| [INT-9](#int-9) | 🟡 moyenne | performance | La portée de traitement de l'IA est forcée à 204 800 unités au minimum, ce qui ignore le réglage utilisateur et rouvre le bug #1876 | `apps/openmw/mwmechanics/actors.cpp:1718` | non soumis |
| [INT-10](#int-10) | 🟡 moyenne | maintenabilite | Comportements OpenMW coupés sans condition (commentaires et « return; »), d'où du code amont mort et une incompatibilité avec les mods | `apps/openmw/mwstate/statemanagerimp.cpp:327` | non soumis |
| [INT-11](#int-11) | 🟡 moyenne | performance | Identité des objets fondée sur (refNum.mIndex, mpNum) sans mContentFile, avec recherche linéaire à chaque paquet | `apps/openmw/mwworld/cellstore.cpp:618` | non soumis |
| [INT-12](#int-12) | 🟡 moyenne | architecture | Logique réseau et gameplay intégrée à la couche rendu (mwrender) | `apps/openmw/mwrender/globalmap.cpp:636` | non soumis |
| [INT-13](#int-13) | 🟡 moyenne | tests | Aucun test automatisé des modifications TES3MP dans OpenMW | `apps/openmw_test_suite/CMakeLists.txt` | non soumis |
| [INT-14](#int-14) | ⚪ basse | manque | Crash catcher OpenMW désactivé sur toutes les plateformes, sans balise | `components/crashcatcher/crashcatcher.hpp:7` | non soumis |
| [INT-15](#int-15) | ⚪ basse | maintenabilite | Branches fragmentées : 0.8.2 non fusionnée, branche VR figée, VR qui déborde dans la branche principale | `apps/openmw/mwmp/GUIController.hpp:27` | non soumis |
| [INT-16](#int-16) | ⚪ basse | maintenabilite | Fichier mort apps/opencs/Networking.cpp : copie obsolète du main d'OpenMW-CS | `apps/opencs/Networking.cpp:44` | non soumis |
| [INT-17](#int-17) | ⚪ basse | performance | À chaque frame et pour chaque script local : copie de std::string et recherche linéaire dans la liste blanche | `apps/openmw/engine.cpp:252` | non soumis |

### INT-1

**Store<ESM::Cell>::override stocke un pointeur vers une variable de pile et peut sortir sans valeur de retour**

- **Sévérité** : 🔴 critique · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw/mwworld/store.cpp:810`

**Preuve**

> store.cpp:810 `(*it) = &const_cast<ESM::Cell&>(cell);` ; g++ -O2 -Wall : `store.cpp:828: control reaches end of non-void function [-Wreturn-type]`. Appelé depuis RecordHelper::overrideRecord(CellRecord), qui passe sa variable locale `recordData`/`finalData`.

**Impact** — Dès que le serveur surcharge une cellule intérieure existante (paquet de record dynamique), mSharedInt conserve l'adresse d'une variable de pile détruite. Ensuite, Cells::getPtr(name) parcourt intBegin() (cells.cpp:328), comme ESMStore (esmstore.cpp:264) : ils déréférencent ce pointeur, d'où corruption mémoire ou plantage côté client, déclenchable à distance. Si la cellule n'existe que dans mDynamicInt, par exemple à la deuxième surcharge d'une cellule créée par le serveur, la fonction sort sans return : comportement indéfini.

**Recommandation** — Réécrire override() : mettre à jour la copie stockée dans mInt ou mDynamicInt (insert_or_assign), puis faire pointer l'entrée de mSharedInt vers cette copie stockée, jamais vers l'argument. Traiter explicitement mDynamicInt et terminer par un return. Ajouter -Werror=return-type dans CMakeLists.txt, et un test gtest qui surcharge deux fois la même cellule puis itère intBegin().

**Contre-vérification (confirmé)** — store.cpp:810 remplace bien l'entrée de mSharedInt par `&const_cast<ESM::Cell&>(cell)`, où `cell` est la locale `recordData`/`finalData` de RecordHelper.cpp:359/370 (via esmstore.hpp:283-284). Aucun garde-fou ne réinitialise mSharedInt, que Cells::getPtr parcourt ensuite (cells.cpp:328-335, intBegin à store.cpp:725-727). Si la cellule n'existe que dans mDynamicInt, la boucle sur mInt ne trouve rien et la fonction sort sans return ; g++ le confirme (`store.cpp:828:5: control reaches end of non-void function`). Une nuance : esmstore.cpp:264 (countRecords) ne s'exécute qu'une fois au chargement grâce au garde `if(!mRefCount.empty()) return;`, mais le chemin par Cells::getPtr suffit à justifier la sévérité.

### INT-2

**La réinitialisation de cellule laisse un pointeur pendant dans mIdCache, et clearMovesToCells n'efface rien**

- **Sévérité** : 🟠 haute · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw/mwworld/cells.cpp:107`

**Preuve**

> cells.cpp:107-118 `void MWWorld::Cells::clear(const ESM::Cell& cell) {… mExteriors.erase(cellCoordinates); … mInteriors.erase(…)}` ne purge pas mIdCache, alors que le clear() amont fait `std::fill(mIdCache.begin(), …nullptr)` (cells.cpp:97). cellstore.cpp:409-410 `mMovedHere.empty(); mMovedToAnotherCell.empty();` produit un avertissement [-Wunused-result].

**Impact** — World::clearCellStore est appelé lors d'un ID_CELL_RESET (Worldstate.cpp:514) ou d'une surcharge de cellule (RecordHelper.cpp:358/369). Il détruit le CellStore alors que mIdCache peut encore pointer dessus, puisque getPtrAndCache y range des CellStore*. Le prochain Cells::getPtr(name) sur le même identifiant (cells.cpp:287-290), typiquement un script global qui suit un PNJ, déréférence de la mémoire libérée : use-after-free côté client provoqué par le serveur. Par ailleurs, clearMovesToCells, censé « éviter les plantages » lors des réinitialisations, ne vide pas ses propres maps.

**Recommandation** — Dans Cells::clear(const ESM::Cell&), avant l'erase, parcourir mIdCache et remettre à {"", nullptr} les entrées qui pointent vers le CellStore supprimé. Remplacer .empty() par .clear() à cellstore.cpp:409-410 et supprimer la variable inutilisée `world` (ligne 393). Ajouter un test unitaire : getPtr, puis clear(cell), puis getPtr.

**Contre-vérification (confirmé)** — cells.cpp:107-118 efface l'entrée de mInteriors/mExteriors, ce qui détruit le CellStore, mais ne purge pas mIdCache. Or getPtrAndCache y range `&cellStore` (cells.cpp:130-131), et getPtr déréférence `*iter->second` dès que le pointeur est non nul (cells.cpp:287-290) : c'est un use-after-free réel après World::clearCellStore (worldimp.cpp:3044-3051), appelé depuis Worldstate.cpp:514. L'appel `empty()` au lieu de `clear()` (cellstore.cpp:409-410, avertissement -Wunused-result) est réel aussi. Il est toutefois sans conséquence, puisque le CellStore est détruit juste après (worldimp.cpp:3050-3051).

### INT-3

**updatePtrsWithRefId modifie le vecteur qu'il itère, et fuit un ManualRef par objet**

- **Sévérité** : 🟠 haute · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw/mwworld/worldimp.cpp:867`

**Preuve**

> worldimp.cpp:867 `for (auto &mergedRef : cellStore->getMergedRefs())` puis ligne 881 `new MWWorld::ManualRef(getStore(), refId, 1);` puis placeObject(), qui conduit à CellStore::insert(), lequel appelle updateMergedRefs() (cellstore.hpp:220) : `mMergedRefs.clear()` (cellstore.cpp:449).

**Impact** — Chaque surcharge de record par le serveur (19 appels dans RecordHelper.cpp) qui touche un objet présent dans une cellule active vide et reconstruit mMergedRefs pendant la boucle range-for. Les itérateurs sont invalidés : en cas de réallocation, c'est un plantage ; sinon, des entrées sont sautées ou retraitées. La référence supprimée reste dans la liste et correspond toujours au refId, ce qui peut dupliquer des objets. Le ManualRef alloué par new n'est jamais libéré.

**Recommandation** — Collecter d'abord les Ptr concernés dans un std::vector<MWWorld::Ptr> local, puis les traiter dans une seconde boucle. Allouer ManualRef sur la pile (`MWWorld::ManualRef ref(getStore(), refId, 1);`), comme le fait le reste d'OpenMW.

**Contre-vérification (confirmé)** — worldimp.cpp:867 itère par range-for sur la référence renvoyée par getMergedRefs() (cellstore.cpp:660). Le corps appelle placeObject, qui conduit à copyToCell puis à CellStore::insert, lequel appelle updateMergedRefs (cellstore.hpp:220), et celui-ci fait `mMergedRefs.clear()` (cellstore.cpp:449). La référence supprimée par deleteObject reste fusionnée, donc le vecteur grossit d'un élément : réallocation possible, d'où des itérateurs invalidés. Aucun test de compte nul n'évite le retraitement. Le `new MWWorld::ManualRef` (ligne 881) n'est jamais libéré.

### INT-4

**ActionTeleport : newCellStore déréférencé sans initialisation quand le suiveur est en combat avec le joueur**

- **Sévérité** : 🟠 haute · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw/mwworld/actionteleport.cpp:126`

**Preuve**

> actionteleport.cpp:82 `MWWorld::CellStore *newCellStore;` ; la branche `isInCombat(...) → stopCombat()` ne l'affecte pas ; ligne 126 `baseActor.cell = *newCellStore->getCell();` ; g++ : `actionteleport.cpp:126:52: warning: ‘newCellStore’ may be used uninitialized`.

**Impact** — getFollowers(actor, followers, true) inclut les suiveurs hostiles situés à moins de 800 unités. Si le joueur emprunte une porte ou un transport alors qu'un suiveur est en combat avec lui, le client lit un pointeur indéterminé et plante ou envoie un paquet ID_ACTOR_CELL_CHANGE corrompu. Le scénario se produit en jeu normal.

**Recommandation** — Initialiser `newCellStore = nullptr` et ne rien envoyer (return) dans la branche stopCombat, ce qui reproduit le comportement amont. Corriger aussi les balises non fermées des lignes 77 et 109 du même fichier. Ajouter -Werror=maybe-uninitialized en CI pour apps/openmw.

**Contre-vérification (confirmé)** — actionteleport.cpp:82 déclare `MWWorld::CellStore *newCellStore;` sans l'initialiser. La branche `isInCombat(player) → stopCombat()` (lignes 85-86) ne l'affecte pas, et la ligne 126 le déréférence sans condition (`*newCellStore->getCell()`) ; g++ le signale (`actionteleport.cpp:126:52 may be used uninitialized`). executeImp appelle getFollowers(actor, followers, true) (ligne 43) : avec includeHostiles=true, le filtre de combat (ligne 154) est désactivé. Un suiveur hostile situé à moins de 800 unités atteint donc bien ce chemin.

### INT-5

**Couplage direct et massif d'OpenMW au singleton mwmp::Main, sans interface de hooks**

- **Sévérité** : 🟠 haute · **Catégorie** : architecture · **Effort** : XL
- **Emplacement** : `apps/openmw/mwscript/miscextensions.cpp:212`

**Preuve**

> miscextensions.cpp:212-217 `mwmp::ObjectList* objectList = mwmp::Main::get().getNetworking()->getObjectList(); objectList->reset(); objectList->packetOrigin = packetOrigin;…` ; grep : 298 appels mwmp::Main::get() dans 84 fichiers, 289 #include "../mwmp/…", 75 séquences reset() dupliquées. Accès directs à des champs publics : getLocalPlayer()->charGenState (10), ->killer, getGUIController()->mPlayerMarkers.

**Impact** — Chaque sous-système OpenMW dépend des classes concrètes de mwmp et de leurs champs internes. On ne peut ni compiler le client sans le réseau, ni tester les hooks, ni rebaser sans conflits répartis sur 84 fichiers. À titre de comparaison, OpenMW 0.50 passe par une interface abstraite MWBase::LuaManager (36 méthodes virtuelles, 67 points d'appel) qui fournit les mêmes points d'accroche : objectActivated, useItem, onHit, objectTeleported.

**Recommandation** — 1) Créer apps/openmw/mwbase/multiplayermanager.hpp, une interface abstraite enregistrée dans MWBase::Environment et calquée sur LuaManager, avec des événements sémantiques : onObjectStateChanged(ptr, enabled, origin), onContainerItemDropped(…), shouldApplyLocally(ptr, action), isLocalAuthority(ptr). 2) Fournir une implémentation nulle (mode solo, tests) et l'implémentation mwmp. 3) Remplacer les séquences ObjectList reset/packetOrigin/add/send par ces appels, sous-système par sous-système, en commençant par mwscript (82 refs) et mwclass (60). 4) Interdire les #include "../mwmp/" hors mwbase avec un check CI.

**Contre-vérification (confirmé)** — L'extrait de miscextensions.cpp:212-217 est exact. Mesures refaites : 298 appels `mwmp::Main::get()` dans 84 fichiers hors de mwmp/ (402 dans 100 fichiers en tout), 289 `#include "../mwmp/` hors de mwmp/, 10 accès à charGenState. Le GUI touche directement le champ public `getGUIController()->mPlayerMarkers` (mapwindow.cpp:196/211). Le CMakeLists.txt n'offre aucune option pour bâtir le client sans mwmp : BUILD_OPENMW_MP ne concerne que le serveur. La comparaison avec MWBase::LuaManager de 0.50 n'a pas été recomptée, mais le constat central tient.

### INT-6

**Divergence amont non soutenable : les 162 fichiers patchés encore présents ont tous changé entre 0.47 et 0.50**

- **Sévérité** : 🟠 haute · **Catégorie** : maintenabilite · **Effort** : XL
- **Emplacement** : `apps/openmw/mwworld/worldimp.cpp:836`

**Preuve**

> Diff contre les tags clonés : 170 fichiers modifiés par TES3MP, 8 absents en 0.50.0 (mwworld/cells.cpp remplacé par worldmodel, components/esm/loadcell déplacé vers esm3), 0 inchangé, churn amont +41 683/−33 691. worldimp.cpp : 0.47=3998 lignes, 0.50=3897, diff +1512/−1613. mwbase/world.hpp en 0.50 : 15 occurrences de ESM::RefId.

**Impact** — Le fork est bloqué sur une version de 2021 (le tag 0.47.0 date du 2021-11-02) alors que l'amont a livré 661 entrées de changelog : Lua natif (Feature #5996 en 0.48), ESM::RefId à la place de std::string, PtrRegistry, ESM4, reformatage clang-format. Le reformatage global seul rend conflictuel chacune des 734 balises. Les correctifs de sécurité et de compatibilité d'OpenMW (GCC récents, Qt6, etc.) ne profitent pas aux joueurs.

**Recommandation** — Stratégie en étapes : a) appliquer le .clang-format de l'amont à l'arbre TES3MP avant tout rebase, pour que le reformatage amont ne produise plus de conflits ; b) extraire les modifications en séries de patchs par sous-système (mwworld, mwmechanics, mwgui, mwscript, mwclass, mwbase) avec git format-patch, puis rebaser d'abord sur openmw-48 (Lua, peu de changements de types), ensuite sur 49 (RefId) et 50 ; c) en parallèle, porter les règles d'autorité de type « disable unilateral X » vers des handlers Lua (I.Activation.addHandlerForType ou I.ItemUsage qui retournent false), qui existent en 0.50 (files/data/scripts/omw/activationhandlers.lua:19), pour réduire le patch C++ ; d) proposer en amont les hooks génériques (contexte d'origine des scripts, Interpreter::Context::CONTEXT_TYPE) pour sortir une partie du patch.

**Contre-vérification (confirmé)** — Vérifié contre les clones amont du scratchpad (upstream/omw047, upstream/omw050). Les 162 fichiers patchés encore présents en 0.50.0 diffèrent tous entre 0.47 et 0.50 (changed=162, unchanged=0) ; 8 n'existent plus, dont mwworld/cells.cpp, remplacé par worldmodel.cpp. worldimp.cpp passe de 3998 à 3897 lignes, mwbase/world.hpp en 0.50 contient 15 `ESM::RefId`, et 0.50 a un `.clang-format`. Dans ce dépôt : 748 balises « Start of tes3mp » dans 163 fichiers sous apps/ et components/ (chiffres proches de ceux annoncés).

### INT-10

**Comportements OpenMW coupés sans condition (commentaires et « return; »), d'où du code amont mort et une incompatibilité avec les mods**

- **Sévérité** : 🟡 moyenne · **Catégorie** : maintenabilite · **Effort** : M
- **Emplacement** : `apps/openmw/mwstate/statemanagerimp.cpp:327`

**Preuve**

> statemanagerimp.cpp:327 `return;` placé avant toute la logique de quicksave ; container.cpp:145 `return;` avant `mDragAndDrop->startDrag(...)` ; miscextensions.cpp:113-114 `//runtime.push (…isGuiMode()); runtime.push(false);` ; engine.cpp:341 `paused = false;`. Au total, 6 blocs « return » seuls et 60 blocs contenant 80 lignes OpenMW commentées.

**Impact** — Les correctifs amont qui touchent ces fonctions continueront de fusionner sans conflit mais ne s'exécuteront jamais, ce qui masque des régressions au rebase. MenuMode renvoie toujours 0, ce qui casse les mods qui en dépendent. Le client ne peut pas fonctionner sans serveur, ce qui empêche de tester OpenMW « nu » dans la même build.

**Recommandation** — Remplacer chaque `return;` ou ligne commentée par une condition explicite `if (MWBase::Environment::get().getMultiplayerManager()->overrides(Feature::QuickSave)) return;` (voir INT-5), et garder le code amont actif dans la branche else. Supprimer le code commenté une fois la condition en place. Lister les fonctionnalités désactivées dans une table unique, par exemple mwmp/DisabledFeatures.hpp.

### INT-11

**Identité des objets fondée sur (refNum.mIndex, mpNum) sans mContentFile, avec recherche linéaire à chaque paquet**

- **Sévérité** : 🟡 moyenne · **Catégorie** : performance · **Effort** : L
- **Emplacement** : `apps/openmw/mwworld/cellstore.cpp:618`

**Preuve**

> cellstore.cpp:618 `if (ptr.getCellRef().getRefNum().mIndex == mRefNumToFind && ptr.getCellRef().getMpNum() == mMpNumToFind)` dans un forEach complet ; grep : 0 usage de mContentFile dans mwmp et components/openmw-mp ; 25 appels de searchExact, dont 23 dans ObjectList.cpp ; searchPtrViaUniqueIndex (worldimp.cpp:836) parcourt toutes les cellules actives.

**Impact** — Chaque objet de chaque paquet entraîne un parcours O(n) de la cellule. Deux références de plugins différents peuvent avoir le même mIndex dans une cellule, et searchExact renvoie alors la première trouvée, ce qui désynchronise le mauvais objet. OpenMW 0.50 résout ces deux problèmes : WorldModel::getPtr(ESM::RefNum) en O(1) via PtrRegistry (worldmodel.hpp:70) et des RefNum générés pour les objets dynamiques (getLastGeneratedRefNum).

**Recommandation** — Court terme : ajouter à CellStore un index `std::unordered_map<std::pair<RefNum,unsigned>, LiveCellRefBase*>` tenu à jour dans insert et updateMergedRefs, et inclure mContentFile dans BaseObject et les paquets. Au rebase : faire correspondre mpNum aux RefNum générés par OpenMW et utiliser PtrRegistry.

### INT-12

**Logique réseau et gameplay intégrée à la couche rendu (mwrender)**

- **Sévérité** : 🟡 moyenne · **Catégorie** : architecture · **Effort** : M
- **Emplacement** : `apps/openmw/mwrender/globalmap.cpp:636`

**Preuve**

> globalmap.cpp:636 `Send an ID_PLAYER_MAP packet with this map tile to the server…`, avec un encodage PNG via osgDB::ReaderWriter dans le même bloc ; weaponanimation.cpp:128 `…prepare an attack packet for sending`.

**Impact** — Le rendu dépend de mwmp::Worldstate et prépare des paquets de combat, ce qui inverse les couches d'OpenMW. Ces accroches seront les plus difficiles à porter, car mwrender a beaucoup évolué en amont. L'encodage PNG se fait de façon synchrone dans le thread principal pendant l'exploration.

**Recommandation** — Émettre des événements neutres depuis mwrender, par exemple onMapTileExplored(x, y, image) et onProjectileLaunched(actor, origin, ammo), via l'interface de hooks d'INT-5. Déplacer l'encodage PNG et l'envoi dans mwmp, hors du thread de rendu (WorkQueue d'OpenMW).

### INT-13

**Aucun test automatisé des modifications TES3MP dans OpenMW**

- **Sévérité** : 🟡 moyenne · **Catégorie** : tests · **Effort** : M
- **Emplacement** : `apps/openmw_test_suite/CMakeLists.txt`

**Preuve**

> `grep -rli "mwmp\|tes3mp\|mpNum" apps/openmw_test_suite` ne renvoie aucun résultat. La suite existante couvre mwworld, esm, settings, etc., mais aucun hook.

**Impact** — Les bugs INT-1 à INT-4 et INT-8, tous détectables par un test unitaire ou un simple avertissement du compilateur, sont restés dans la branche stable. Sans tests, aucun rebase vers 0.48+ ne peut être validé.

**Recommandation** — Ajouter dans apps/openmw_test_suite/mwworld des tests gtest pour Store<ESM::Cell>::override (double surcharge, puis itération intBegin), Cells::clear(cell) avec mIdCache, CellStore::clearMovesToCells, CellStore::searchExact (collision de mIndex) et World::updatePtrsWithRefId. Activer -Werror=return-type et -Werror=maybe-uninitialized en CI.

### INT-7

**Balisage incohérent : balises non appariées, variantes non canoniques et modifications non balisées**

- **Sévérité** : 🟡 moyenne · **Catégorie** : maintenabilite · **Effort** : M
- **Emplacement** : `apps/openmw/mwworld/actionteleport.cpp:109`

**Preuve**

> actionteleport.cpp:109 `Start of tes3mp change (minor)` est utilisé à la place de End, et le bloc ouvert ligne 77 n'est jamais fermé. character.cpp:822 `Start tes3mp change (major)` (sans « of ») ; actionmanager.cpp:371 `Start  of tes3mp addition` ; containerstore.cpp:709 `addiition`. Le lint compte 12 anomalies. 6 fichiers modifiés n'ont aucune balise : crashcatcher.hpp, mtphysics.cpp, controlswitch.hpp, utf8stream.hpp, launcher/maindialog.cpp:615, windows_crashcatcher.cpp.

**Impact** — Les balises sont le seul outil pour retrouver la divergence au moment d'un rebase. Les variantes échappent aux grep, et les blocs non fermés ou mal fermés faussent l'extraction automatique de patchs. Les changements non balisés, comme USE_CRASH_CATCHER 0 ou la synchronisation mtphysics modifiée, seront écrasés ou perdus en silence au rebase.

**Recommandation** — Normaliser les balises dans un format machine, par exemple `// TES3MP-BEGIN add|chg <ID-court>` et `// TES3MP-END <ID-court>`. Ajouter en CI un script de vérification (équivalent de balance.py : appariement, types, absence de variantes) et un diff contre le tag amont qui échoue si un fichier modifié n'a pas de balise. Documenter la convention dans CONTRIBUTING.md.

### INT-8

**setPhysicsFramerate est sans effet avec la physique multithread de 0.47**

- **Sévérité** : 🟡 moyenne · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw/mwphysics/physicssystem.cpp:189`

**Preuve**

> physicssystem.cpp:189 `mTaskScheduler->mPhysicsDt = mPhysicsDt;` alors que mtphysics.cpp:193 `int numSteps = timeAccum / mDefaultPhysicsDt;` utilise mDefaultPhysicsDt, fixé au constructeur, et que mtphysics.cpp:284 `mPhysicsDt = newDelta;` écrase la valeur à chaque frame.

**Impact** — La fonction serveur SetPhysicsFramerate (apps/openmw-mp/Script/Functions/Settings.cpp:27), transmise par ProcessorGameSettings, ne change pas le pas réel de simulation. Seul le seuil de PhysicsSystem change, ce qui rend le comportement incohérent entre les clients. Le hook a été écrit pour l'ancienne physique et n'a pas été réadapté au rebase. L'écriture d'un membre rendu public, sans verrou, pendant que les workers lisent mPhysicsDt est en plus une course de données potentielle.

**Recommandation** — Remettre mPhysicsDt en privé dans mtphysics.hpp:79-87. Ajouter un `PhysicsTaskScheduler::setDefaultPhysicsDt(float)` qui met à jour mDefaultPhysicsDt sous mSimulationMutex, et l'appeler depuis PhysicsSystem::setPhysicsFramerate. Remplacer std::cerr par Log(Debug::Warning).

**Contre-vérification (confirmé)** — physicssystem.cpp:189 écrit `mTaskScheduler->mPhysicsDt`. Or calculateStepConfig calcule numSteps et actualDelta à partir de mDefaultPhysicsDt seul (mtphysics.cpp:193, 219, 230), fixé au constructeur (ligne 132), et moveActors écrase `mPhysicsDt = newDelta` à chaque frame (mtphysics.cpp:284). Le seul effet restant est le seuil `mTimeAccum >= mPhysicsDt` de PhysicsSystem (physicssystem.cpp:779). L'écriture se fait sans verrou depuis ProcessorGameSettings.hpp:53, alors que les workers lisent mPhysicsDt (mtphysics.cpp:499) : il y a donc bien un risque de course de données.

### INT-9

**La portée de traitement de l'IA est forcée à 204 800 unités au minimum, ce qui ignore le réglage utilisateur et rouvre le bug #1876**

- **Sévérité** : 🟡 moyenne · **Catégorie** : performance · **Effort** : M
- **Emplacement** : `apps/openmw/mwmechanics/actors.cpp:1718`

**Preuve**

> actors.cpp:1716-1718 `// … larger than 7168 will make some quests harder or impossible (bug #1876)` `//static const float maxProcessingRange = 7168.f;` `static const float maxProcessingRange = 8192.f * 50;`, puis ligne 1723 `minProcessingRange = maxProcessingRange / 2.f`.

**Impact** — Comme le minimum vaut 204 800, le réglage « actors processing range » est toujours ramené à 204 800 : tous les acteurs de toutes les cellules chargées sont traités (IA, animation) à chaque frame. Le coût CPU augmente, et la régression de quêtes documentée par l'amont (bug #1876) réapparaît, y compris sur les clients qui ne détiennent pas l'autorité.

**Recommandation** — Conserver la limite amont (7168) pour les acteurs qui ne sont pas LocalActor. Rendre la portée paramétrable par le serveur via GameSettings plutôt que codée en dur. Ne lever la limite que pour les cellules dont le client détient l'autorité (CellController::hasLocalAuthority).

### INT-14

**Crash catcher OpenMW désactivé sur toutes les plateformes, sans balise**

- **Sévérité** : ⚪ basse · **Catégorie** : manque · **Effort** : S
- **Emplacement** : `components/crashcatcher/crashcatcher.hpp:7`

**Preuve**

> crashcatcher.hpp:6-9 `#if (defined(__APPLE__) || … ) #define USE_CRASH_CATCHER 0 #else #define USE_CRASH_CATCHER 0`. L'amont 0.47.0 vaut 1 ; breakpad n'est câblé que dans apps/openmw-mp/CMakeLists.txt:3 (OFF par défaut).

**Impact** — Le client ne produit aucun journal de plantage. Les bugs mémoire ci-dessus (INT-1 à INT-4) sont donc très difficiles à diagnostiquer chez les joueurs. Le changement n'est pas balisé et sera perdu ou mal compris au rebase.

**Recommandation** — Rétablir USE_CRASH_CATCHER à 1 sur Linux et macOS comme en amont (ou documenter et baliser la raison de la désactivation), et rediriger le chemin du crash log vers le dossier de logs TES3MP.

### INT-15

**Branches fragmentées : 0.8.2 non fusionnée, branche VR figée, VR qui déborde dans la branche principale**

- **Sévérité** : ⚪ basse · **Catégorie** : maintenabilite · **Effort** : M
- **Emplacement** : `apps/openmw/mwmp/GUIController.hpp:27`

**Preuve**

> GUIController.hpp:27 `GM_VR_MetaMenu = MWGui::GM_QuickKeysMenu + 1, // … used in VR` ; g++ : `actionmanager.cpp:375: comparison between ‘enum MWGui::GuiMode’ and ‘enum mwmp::GUIController::GM’ [-Wenum-compare]` ; git ls-remote TES3MP : 0.8.1-vr au 2022-05-06, 0.8.2 au 2022-08-02 (10 commits absents du fork).

**Impact** — Trois lignes divergentes (0.8.1, 0.8.2, 0.8.1-vr) sur la même base 0.47, et les modes GUI TES3MP sont étendus par décalage d'enum, un procédé fragile : le commit VR 3b35565 a déjà dû corriger un chevauchement d'identifiants. Le fork vazer7070 n'apporte rien de plus que TES3MP/0.8.1 (même HEAD 49be5b6). OpenMW 0.50 contient components/stereo mais pas de mwvr (vérifié dans le tag), donc la VR reste hors amont.

**Recommandation** — Fusionner TES3MP/0.8.2 dans le fork. Remplacer l'extension par décalage d'enum par un registre de modes GUI ou une plage réservée documentée. Geler officiellement la branche VR et ne la reprendre qu'après le rebase (sur OpenMW-VR ou components/stereo).

### INT-16

**Fichier mort apps/opencs/Networking.cpp : copie obsolète du main d'OpenMW-CS**

- **Sévérité** : ⚪ basse · **Catégorie** : maintenabilite · **Effort** : S
- **Emplacement** : `apps/opencs/Networking.cpp:44`

**Preuve**

> Networking.cpp:44 `int main(int argc, char *argv[])` ; `diff apps/opencs/Networking.cpp apps/opencs/main.cpp` montre une ancienne version (std::cerr, sans wrapApplication) ; aucune référence dans apps/opencs/CMakeLists.txt.

**Impact** — Ces 90 lignes non compilées induisent en erreur : leur nom suggère une fonctionnalité réseau dans l'éditeur, alors qu'il n'y en a pas. Le fichier gonfle aussi artificiellement le diff contre l'amont.

**Recommandation** — Supprimer apps/opencs/Networking.cpp.

### INT-17

**À chaque frame et pour chaque script local : copie de std::string et recherche linéaire dans la liste blanche**

- **Sévérité** : ⚪ basse · **Catégorie** : performance · **Effort** : S
- **Emplacement** : `apps/openmw/engine.cpp:252`

**Preuve**

> engine.cpp:252 `if (mwmp::Main::isValidPacketScript(script.first))` dans la boucle executeLocalScripts ; Main.cpp:248-252 `bool Main::isValidPacketScript(std::string scriptId) { … Utils::vectorContains(worldstate->synchronizedClientScriptIds, scriptId)` (std::find linéaire).

**Impact** — Le coût est en O(scripts locaux × taille de la liste blanche) à chaque frame, avec une allocation de chaîne par appel. Il est faible sur les petits serveurs mais grandit avec les listes de synchronisation des gros serveurs. La comparaison est sensible à la casse alors que les identifiants Morrowind ne le sont pas.

**Recommandation** — Passer le paramètre en `const std::string&`, stocker la liste blanche dans un std::unordered_set en minuscules, recalculé à chaque réception de paquet, ou mémoriser le drapeau sendPackets par script dans LocalScripts.

## Sécurité (modèle de menace transverse)

| ID | Sévérité | Catégorie | Titre | Emplacement | Vérif. |
|---|---|---|---|---|---|
| [SEC-1](#sec-1) | 🔴 critique | securite | Lecture hors bornes déclenchable quasi sans authentification dans le parsing ID_GAME_PREINIT | `apps/openmw-mp/Networking.cpp:261` | confirmé |
| [SEC-2](#sec-2) | 🔴 critique | securite | Déni de service par resize() non borné sur compteur contrôlé par le client dans ~26 parseurs de paquets | `components/openmw-mp/Packets/Player/PacketPlayerJournal.cpp:25` | partiel (sévérité corrigée) |
| [SEC-3](#sec-3) | 🔴 critique | securite | Compteurs signés du protocole maître : resize() négatif -> gigantesque, crash sans try/catch | `components/openmw-mp/Master/ProxyMasterPacket.hpp:74` | confirmé |
| [SEC-4](#sec-4) | 🟠 haute | securite | Chaîne de format contrôlée par le serveur sur le client (serveur malveillant -> client) | `apps/openmw/mwmp/Networking.cpp:419` | partiel (sévérité corrigée) |
| [SEC-5](#sec-5) | 🟡 moyenne | securite | Transport sans chiffrement ni authentification ; mot de passe serveur en clair | `components/openmw-mp/Packets/System/PacketSystemHandshake.cpp:15` | partiel (sévérité corrigée) |
| [SEC-6](#sec-6) | 🟡 moyenne | securite | Absence de limitation de débit et d'anti-flood côté serveur | `apps/openmw-mp/Networking.cpp:311` | confirmé |
| [SEC-7](#sec-7) | 🟡 moyenne | securite | Confiance excessive dans le client : position, stats et dégâts rediffusés sans validation | `apps/openmw-mp/processors/player/ProcessorPlayerStatsDynamic.hpp:10` | confirmé |
| [SEC-8](#sec-8) | ⚪ basse | securite | Exécution de commandes console arbitraires côté client pilotée par le serveur | `apps/openmw/mwmp/ObjectList.cpp:956` | partiel (sévérité corrigée) |
| [SEC-9](#sec-9) | ⚪ basse | securite | Données d'image non fiables passées au décodeur PNG OSG (et &imageData[0] sur vecteur potentiellement vide) | `apps/openmw/mwrender/globalmap.cpp:723` | non soumis |
| [SEC-10](#sec-10) | ⚪ basse | securite | Index bruts non vérifiés dans de nombreuses fonctions script noexcept | `apps/openmw-mp/Script/Functions/Items.cpp:113` | non soumis |
| [SEC-11](#sec-11) | ⚪ basse | securite | Secret maître codé en dur et serveur REST maître exposé sans authentification | `components/openmw-mp/Version.hpp:8` | non soumis |

### SEC-1

**Lecture hors bornes déclenchable quasi sans authentification dans le parsing ID_GAME_PREINIT**

- **Sévérité** : 🔴 critique · **Catégorie** : securite · **Effort** : S
- **Emplacement** : `apps/openmw-mp/Networking.cpp:261`

**Preuve**

> LOG_APPEND(... "checksum: %X ...", i, dataFile->second[0], dataFile->first.c_str()); -- dataFile->second est un std::vector<uint32_t> dont la taille provient du paquet (hashN). PacketPreInit.cpp:58 n'invalide le paquet que si hashN > maxHashes, donc hashN=0 est accepté et second est vide.

**Impact** — Un pair envoyant un ID_GAME_PREINIT forgé (un plugin dont hashN=0, nom non vide, taille de paquet cohérente) fait lire dataFile->second[0] sur un vecteur vide. Reproduit : le harnais h_preinit déclenche une assertion _GLIBCXX puis un SIGSEGV (exit 134/139). Atteignable avant le handshake applicatif, uniquement derrière le mot de passe = chaîne de version (publique). Crash du serveur dédié.

**Recommandation** — Dans PacketPreInit::Packet, rejeter (packetValid=false) tout plugin dont hashN==0, ou garder la boucle de Networking::preInit en testant !dataFile->second.empty() avant d'accéder à [0]. Vérifier aussi côté client (listDiscrepancies accède de même à .second.at(0)).

**Contre-vérification (confirmé)** — PacketPreInit.cpp:58 rejette seulement hashN > maxHashes (50), donc hashN=0 passe et checksum.second.resize(0) (l.84) laisse un vecteur vide. Networking.cpp:261 évalue dataFile->second[0] comme argument de LOG_APPEND (macro TimedLog.hpp:25 qui évalue toujours ses arguments, le filtre de niveau est dans print), puis l.269 le relit : déréférencement d'un vecteur vide (data()==nullptr). Le chemin est appelé pour tout pair sans Player (Networking.cpp:563-566) ; le seul « mot de passe » est la version+hash de commit (main.cpp:224-229), public, et le nombre de plugins attendu est renvoyé par le serveur lui-même en cas de refus (l.285).

### SEC-2

**Déni de service par resize() non borné sur compteur contrôlé par le client dans ~26 parseurs de paquets**

- **Sévérité** : 🔴 critique (initialement haute) · **Catégorie** : securite · **Effort** : M
- **Emplacement** : `components/openmw-mp/Packets/Player/PacketPlayerJournal.cpp:25`

**Preuve**

> RW(count, send); ... if (!send) player->journalChanges.resize(count); -- count est un uint32 lu du réseau, sans aucun plafond (idem PacketPlayerInventory:27, PacketPlayerBook:25, PacketWorldMap:26, PacketCellReset:26, PacketClientScriptGlobal:26, PacketWorldKillCount:26...).

**Impact** — Un joueur connecté (POSTLOADED) envoie un count proche de 0xFFFFFFFF ; resize() tente d'allouer des dizaines/centaines de Go, lève std::bad_alloc ou std::length_error. Or PlayerProcessor::Process/WorldstateProcessor::Process sont noexcept (PlayerProcessor.cpp:9) : l'exception déclenche std::terminate -> crash du serveur. Reproductible par tout client authentifié.

**Recommandation** — Introduire une constante de plafond par paquet (comme maxObjects) et invalider le paquet si count dépasse ; appliquer un facteur de vraisemblance (taille restante de la BitStream) avant resize. Alternativement réserver puis lire élément par élément en vérifiant la valeur de retour de RW.

**Contre-vérification (partiel (sévérité corrigée))** — Confirmé : PacketPlayerJournal.cpp:20-25 fait RW(count) puis resize(count) sans plafond, et PlayerProcessor::Process est noexcept (PlayerProcessor.cpp:9) sans try/catch dans la boucle (seul try dans main.cpp:239, non traversable par une exception sortant d'une fonction noexcept), donc bad_alloc -> std::terminate. Il suffit d'avoir terminé le handshake (Networking.cpp:143), c'est-à-dire n'importe quel joueur sur un serveur public sans mot de passe : selon la grille, un crash déclenchable à distance relève de « critique » plutôt que de « haute ». On compte 29 resize(count/…Count) dans components/openmw-mp/Packets.

### SEC-3

**Compteurs signés du protocole maître : resize() négatif -> gigantesque, crash sans try/catch**

- **Sévérité** : 🔴 critique · **Catégorie** : securite · **Effort** : S
- **Emplacement** : `components/openmw-mp/Master/ProxyMasterPacket.hpp:74`

**Preuve**

> int32_t playersCount = server.players.size(); packet->RW(playersCount, send); if (playersCount > QueryData::maxPlayers) playersCount = 0; ... server.players.resize(playersCount); -- un playersCount négatif passe le test (>100 faux) et resize((size_t)-1). Idem pluginsCount:90 et rulesSize:25 (while(rulesSize--)).

**Impact** — Un paquet ID_MASTER_ANNOUNCE/ID_MASTER_QUERY forgé avec playersCount=-1 provoque resize sur une taille convertie en ~2^64 -> length_error/bad_alloc. MasterServer::Thread (MasterServer.cpp:49) ne possède aucun try/catch : terminaison du serveur maître. Le client parse aussi ces paquets (PacketMasterQuery côté browser), même vecteur d'attaque par serveur maître usurpé.

**Recommandation** — Rejeter toute valeur < 0 (ou lire en uint32 et plafonner) avant resize dans ProxyMasterPacket::addServer et PacketMasterQuery.cpp:33 (serversCount int32_t). Entourer la boucle de réception de MasterServer::Thread d'un try/catch par paquet.

**Contre-vérification (confirmé)** — ProxyMasterPacket.hpp:65-74 lit un int32_t signé, le test `> maxPlayers` (100) ne rejette pas les valeurs négatives, puis resize((size_t)-1) lève length_error ; de même pluginsCount (l.81-90). Pour rulesSize négatif (l.25-35), while(rulesSize--) boucle environ 2^31 fois (débordement signé UB), ce qui bloque le CPU au lieu de planter. MasterServer::Thread (MasterServer.cpp:35-190) appelle pma.Read() (l.131) sans aucun try/catch dans un std::thread, d'où std::terminate ; le mot de passe CrabNet du maître est une constante compilée (MasterServer.cpp:24).

### SEC-4

**Chaîne de format contrôlée par le serveur sur le client (serveur malveillant -> client)**

- **Sévérité** : 🟠 haute · **Catégorie** : securite · **Effort** : S
- **Emplacement** : `apps/openmw/mwmp/Networking.cpp:419`

**Preuve**

> LOG_MESSAGE_SIMPLE(TimedLog::LOG_ERROR, listDiscrepancies(checksums, checksumsResponse).c_str()); -- listDiscrepancies (ligne 73) insère serverFilename issu de checksumsResponse (paquet PREINIT du serveur) dans la chaîne, laquelle est passée comme ARGUMENT format à TimedLog::print -> vsnprintf.

**Impact** — Un serveur malveillant renvoie un nom de plugin contenant des spécificateurs (%s, %n...) ; côté client, vsnprintf (TimedLog.cpp:100) interprète ces spécificateurs : lecture de pile / crash, et %n ouvre une écriture arbitraire. Déclenché dès la connexion, avant l'application des données. Lignes 265 et 346 passent aussi errmsg comme format.

**Recommandation** — Toujours passer les chaînes dynamiques comme argument d'un format littéral : LOG_MESSAGE_SIMPLE(level, "%s", str.c_str()). Auditer tous les appels LOG_* dont le 2e argument n'est pas un littéral.

**Contre-vérification (partiel (sévérité corrigée))** — Le cœur du constat est confirmé : apps/openmw/mwmp/Networking.cpp:419 passe listDiscrepancies(...) (qui insère serverFilename du paquet serveur, l.72-75/80) comme format à TimedLog::print, qui appelle vsnprintf (TimedLog.cpp:98-102) ; des %s donnent une lecture de pile ou un crash côté client. En revanche, les lignes 265 et 346 n'utilisent que des chaînes constantes (l.248-254, 287-316) et ne sont pas exploitables. L'écriture arbitraire via %n est exagérée : glibc avec _FORTIFY_SOURCE avorte sur un %n placé dans un format inscriptible, et le CRT MSVC désactive %n par défaut. La sévérité haute (crash client par un serveur malveillant) reste juste.

### SEC-5

**Transport sans chiffrement ni authentification ; mot de passe serveur en clair**

- **Sévérité** : 🟡 moyenne (initialement haute) · **Catégorie** : securite · **Effort** : L
- **Emplacement** : `components/openmw-mp/Packets/System/PacketSystemHandshake.cpp:15`

**Preuve**

> RW(system->playerName, ...); RW(system->serverPassword, ...); -- envoyé en clair. CrabNet est compilé avec LIBCAT_SECURITY=0 (NativeFeatureIncludes.h:55) et aucun InitializeSecurity n'est appelé (grep négatif dans apps/).

**Impact** — Tout le trafic (position, inventaire, chat, mot de passe serveur, commandes console) circule en clair sur UDP sans intégrité. Un attaquant sur le chemin réseau lit et modifie les paquets (triche, vol du mot de passe serveur, injection de commandes console vers les clients). Le « mot de passe » CrabNet SetIncomingPassword (main.cpp:229) est la chaîne de version, donc publique : ce n'est pas une authentification.

**Recommandation** — Activer la sécurité de transport de CrabNet (LIBCAT_SECURITY + InitializeSecurity avec échange de clés) ou documenter explicitement l'exigence d'un tunnel (VPN/wireguard). A minima, ne jamais logguer baseSystem.serverPassword en clair (Networking.cpp:122).

**Contre-vérification (partiel (sévérité corrigée))** — PacketSystemHandshake.cpp:16-17 envoie bien serverPassword en clair et aucun InitializeSecurity n'apparaît dans apps/ (grep négatif). La preuve citée est inexacte : NativeFeatureIncludes.h:54-56 a sa définition en commentaire, et c'est l'option CMake CRABNET_ENABLE_LIBCAT_SECURITY=FALSE de CrabNet (CMakeLists.txt:39) qui désactive le chiffrement. L'exploitation suppose un attaquant placé sur le chemin réseau, et le trafic de jeu en clair est courant pour ce type de jeu : seul le mot de passe en clair est réellement sensible, d'où une sévérité moyenne plutôt que haute.

### SEC-6

**Absence de limitation de débit et d'anti-flood côté serveur**

- **Sévérité** : 🟡 moyenne · **Catégorie** : securite · **Effort** : M
- **Emplacement** : `apps/openmw-mp/Networking.cpp:311`

**Preuve**

> for (packet=peer->Receive(); packet; ... ) { ... update(packet, bsIn) ...} -- aucune comptabilisation par client/par seconde ; SetLimitIPConnectionFrequency() de CrabNet (disponible, RakPeerInterface.h:317) n'est jamais appelée dans apps/.

**Impact** — Un client authentifié peut inonder le serveur de paquets coûteux (ex. PacketWorldMap avec de nombreuses tuiles, PacketRecordDynamic à 3000 enregistrements) et saturer le thread unique de mainLoop (sleep de 1 ms). Multiplié par SEC-2, un seul client dégrade/immobilise le serveur. Les tentatives de connexion répétées ne sont pas non plus bridées côté IP.

**Recommandation** — Appeler peer->SetLimitIPConnectionFrequency(true) au Startup ; ajouter un compteur de paquets/octets par joueur et par intervalle dans mainLoop avec kick au dépassement ; plafonner le coût des paquets worldstate par tick.

**Contre-vérification (confirmé)** — La boucle de réception (Networking.cpp:~520-570, la ligne 311 citée correspond en fait à update()) traite tous les paquets sans aucun comptage par client, et SetLimitIPConnectionFrequency (RakPeerInterface.h:317 de CrabNet) n'est appelé nulle part dans apps/. Seul garde-fou partiel : le compteur de tentatives de handshake (Networking.cpp:145-152), qui ne concerne que les pairs non authentifiés. PacketRecordDynamic borne bien recordsCount (l.85), ce qui limite le coût par paquet sans limiter le débit. La sévérité moyenne est juste.

### SEC-7

**Confiance excessive dans le client : position, stats et dégâts rediffusés sans validation**

- **Sévérité** : 🟡 moyenne · **Catégorie** : securite · **Effort** : L
- **Emplacement** : `apps/openmw-mp/processors/player/ProcessorPlayerStatsDynamic.hpp:10`

**Preuve**

> void Do(...) { DEBUG_PRINTF(...); player.sendToLoaded(&packet); } -- aucune vérification ; ProcessorPlayerAttack.hpp relaie attack.damage (PacketPlayerAttack.cpp:56) tel quel, ProcessorPlayerPosition.hpp rediffuse la position brute.

**Impact** — Un client modifié fixe sa santé/magie, téléporte, ou envoie des dégâts arbitraires qui sont appliqués chez les victimes (MechanicsHelper.cpp:483 onHit avec attack.damage reçu). Une partie est inhérente au design client-autoritaire d'OpenMW (corrigeable seulement par des scripts anti-triche), mais rien dans le cœur ne borne les valeurs ni n'offre de point de contrôle serveur par défaut.

**Recommandation** — Documenter clairement la frontière de confiance, et fournir dans le cœur des gardes optionnelles (plafonds de dégâts/vitesse, cohérence de cellule) activables, plutôt que de déléguer entièrement la validation à CoreScripts. Exposer la valeur reçue avant rediffusion aux callbacks pour permettre un rejet.

**Contre-vérification (confirmé)** — ProcessorPlayerStatsDynamic.hpp:17-21 relaie le paquet sans contrôle ni callback Lua. ProcessorPlayerAttack.hpp ne vérifie que mDead avant sendToLoaded, et attack.damage lu à PacketPlayerAttack.cpp:56 est appliqué tel quel côté victime (MechanicsHelper.cpp:482 onHit(victim, attack.damage, ...)). Il s'agit d'un choix de conception client-autoritaire, et la sévérité moyenne est appropriée.

### SEC-10

**Index bruts non vérifiés dans de nombreuses fonctions script noexcept**

- **Sévérité** : ⚪ basse · **Catégorie** : securite · **Effort** : M
- **Emplacement** : `apps/openmw-mp/Script/Functions/Items.cpp:113`

**Preuve**

> return player->equipmentIndexChanges[changeIndex]; -- operator[] brut avec changeIndex fourni par le script, fonction noexcept. 24 accès de ce type (index/slot/changeIndex) dont CharClass.cpp:40 mData.mAttribute[slot].

**Impact** — Les vecteurs d'index (equipmentIndexChanges, attributeIndexChanges...) sont remplis depuis des paquets client ; leur taille est contrôlée par le client. Un script serveur itérant sur une taille supposée mais indexant avec une valeur incohérente lit/écrit hors bornes. Comme les fonctions sont noexcept, une exception (si .at) terminerait le processus ; avec [], c'est une corruption silencieuse. Déclenchement indirect (bug de script + données client), d'où la sévérité basse.

**Recommandation** — Remplacer operator[] par .at() ou par une vérification explicite de borne retournant une valeur par défaut dans ces fonctions de l'API script, et borner les tailles des *IndexChanges à la réception.

### SEC-11

**Secret maître codé en dur et serveur REST maître exposé sans authentification**

- **Sévérité** : ⚪ basse · **Catégorie** : securite · **Effort** : M
- **Emplacement** : `components/openmw-mp/Version.hpp:8`

**Preuve**

> #define TES3MP_MASTERSERVER_PASSW "12345" -- utilisé par SetIncomingPassword (MasterServer.cpp:24) et par tout serveur/navigateur (QueryClient.cpp:177). RestServer accepte POST/PUT (RestServer.cpp:98,120) pour enregistrer/mettre à jour une entrée serveur sans authentification.

**Impact** — Le « mot de passe » du protocole maître est public : n'importe qui peut annoncer/usurper des serveurs auprès du maître (empoisonnement de la liste), et le POST REST permet d'injecter des entrées arbitraires (hostname/modname attaquant). Potentiel d'amplification/redirection via la liste de serveurs renvoyée aux navigateurs. Impact limité car le maître ne relaie pas de données de jeu.

**Recommandation** — Ne pas présenter TES3MP_MASTERSERVER_PASSW comme une authentification ; restreindre l'enregistrement REST (par clé, par corrélation IP, rate-limit), valider longueur/charset des champs ptree avant stockage, et borner le nombre d'entrées par IP.

### SEC-8

**Exécution de commandes console arbitraires côté client pilotée par le serveur**

- **Sévérité** : ⚪ basse (initialement moyenne) · **Catégorie** : securite · **Effort** : M
- **Emplacement** : `apps/openmw/mwmp/ObjectList.cpp:956`

**Preuve**

> windowManager->executeCommandInConsole(consoleCommand); -- consoleCommand provient du paquet ID_CONSOLE_COMMAND du serveur (ProcessorConsoleCommand.hpp côté client), exécuté sur le joueur local ou des objets de cellule.

**Impact** — Un serveur malveillant (ou compromis) exécute n'importe quelle commande console Morrowind sur les clients, prenant le contrôle de leur état de jeu. C'est inhérent au design (le serveur pilote les clients), mais l'absence totale de filtrage/consentement en fait un vecteur puissant de manipulation de masse des clients connectés.

**Recommandation** — Documenter ce pouvoir comme partie de la surface de confiance serveur->client ; envisager une liste blanche/filtre côté client ou un indicateur de consentement pour les commandes sensibles.

**Contre-vérification (partiel (sévérité corrigée))** — Le code est confirmé : ObjectList.cpp:956/970/980/997 exécute consoleCommand reçu du serveur. Côté serveur, ID_CONSOLE_COMMAND n'est pas relayé automatiquement : ProcessorConsoleCommand.hpp n'appelle que le callback Lua OnConsoleCommand, et seul un script serveur peut l'émettre (Script/Functions/Objects.cpp:926). Le serveur est de toute façon autoritaire sur tout l'état de jeu par d'autres paquets, et les commandes console Morrowind restent confinées au jeu, sans accès au système. Le gain réel pour un attaquant est donc marginal, d'où une sévérité basse.

### SEC-9

**Données d'image non fiables passées au décodeur PNG OSG (et &imageData[0] sur vecteur potentiellement vide)**

- **Sévérité** : ⚪ basse · **Catégorie** : securite · **Effort** : S
- **Emplacement** : `apps/openmw/mwrender/globalmap.cpp:723`

**Preuve**

> Files::IMemStream istream(&imageData[0], imageData.size()); ... reader->readImage(istream); -- imageData provient de PacketWorldMap (serveur). &imageData[0] est un comportement indéfini si imageData est vide (imageDataSize=0 accepté par PacketWorldMap.cpp).

**Impact** — Le serveur fournit jusqu'à 1800 octets (maxImageDataSize) interprétés par le lecteur PNG d'OSG sur le client ; toute faille du décodeur devient exploitable par un serveur malveillant. Un imageDataSize=0 produit &imageData[0] sur un std::vector vide (UB). La borne 1800 limite l'ampleur mais pas la nature du traitement de données non fiables par un codec.

**Recommandation** — Vérifier !imageData.empty() avant setImage et utiliser imageData.data(). Considérer un décodage PNG en bac à sable ou une validation d'en-tête avant de confier les octets au ReaderWriter.

## Build, CI, dépendances (empirique)

| ID | Sévérité | Catégorie | Titre | Emplacement | Vérif. |
|---|---|---|---|---|---|
| [BLD-1](#bld-1) | 🔴 critique | bug | tes3mp-server ne compile avec aucun compilateur moderne testé (constexpr + reinterpret_cast) | `apps/openmw-mp/Script/Types.hpp:104` | confirmé |
| [SA-1](#sa-1) | 🔴 critique | securite | Écriture hors bornes distante via l'index d'équipement non vérifié dans ID_PLAYER_EQUIPMENT | `components/openmw-mp/Packets/Player/PacketPlayerEquipment.cpp:42` | confirmé |
| [SA-2](#sa-2) | 🔴 critique | securite | Déni de service mémoire : resize() sur un compteur uint32 non plafonné dans 24 paquets | `components/openmw-mp/Packets/Player/PacketPlayerInventory.cpp:27` | partiel (sévérité corrigée) |
| [BLD-2](#bld-2) | 🟠 haute | build-ci | CrabNet (RakNet) cloné sans version épinglée et non compilable tel quel avec GCC 13 | `CI/before_install.linux.sh:10` | confirmé |
| [BLD-4](#bld-4) | 🟠 haute | build-ci | CI entièrement inopérante pour TES3MP (Travis mort, GitLab/AppVeyor sans RakNet ni LuaJIT, aucune GitHub Actions) | `.travis.yml:74` | partiel (sévérité corrigée) |
| [SA-4](#sa-4) | 🟠 haute | bug | Course de données : la std::map du serveur maître partagée entre deux threads sans verrou | `apps/master/RestServer.cpp:85` | confirmé |
| [BLD-3](#bld-3) | 🟡 moyenne | build-ci | Configuration impossible avec CMake 4.x (cmake_minimum_required 3.1 placé après project()) | `CMakeLists.txt:2` | partiel (sévérité corrigée) |
| [BLD-5](#bld-5) | 🟡 moyenne | bug | Crash (SIGSEGV) à l'arrêt du serveur : course entre le thread MasterClient et la destruction de Networking | `apps/openmw-mp/MasterClient.cpp:188` | partiel (sévérité corrigée) |
| [BLD-6](#bld-6) | 🟡 moyenne | build-ci | Le build serveur seul exige SDL2, OpenGL, LZ4, les en-têtes OSG et un téléchargement de recastnavigation, tous inutiles au binaire | `CMakeLists.txt:454` | non soumis |
| [BLD-7](#bld-7) | 🟡 moyenne | build-ci | FindRakNet.cmake fragile : bibliothèque Debug obligatoire, uniquement $ENV{RAKNET_ROOT}, en-têtes non-SYSTEM | `cmake/FindRakNet.cmake:52` | non soumis |
| [BLD-9](#bld-9) | 🟡 moyenne | architecture | La compatibilité client/serveur dépend du hash git au moment du build (build non reproductible) | `apps/openmw-mp/main.cpp:227` | non soumis |
| [SA-3](#sa-3) | 🟡 moyenne | securite | Vulnérabilité de chaîne de format : LOG_* avec e.what() comme format, macros sans attribut printf | `apps/openmw-mp/Script/Script.hpp:103` | partiel (sévérité corrigée) |
| [SA-5](#sa-5) | 🟡 moyenne | bug | Invalidation d'itérateur : reset() vide le conteneur en pleine boucle range-for (client) | `apps/openmw/mwmp/ObjectList.cpp:761` | partiel (sévérité corrigée) |
| [SA-6](#sa-6) | 🟡 moyenne | bug | std::terminate via exception échappant d'un binding Lua noexcept (index non vérifié) | `apps/openmw-mp/Script/Functions/Items.cpp:164` | confirmé |
| [BLD-8](#bld-8) | ⚪ basse | bug | Bugs réels détectés seulement par clang : arithmétique de pointeur sur chaînes et delete sans destructeur virtuel | `apps/openmw-mp/Script/LangLua/LangLua.cpp:262` | partiel (sévérité corrigée) |
| [BLD-10](#bld-10) | ⚪ basse | build-ci | Le serveur maître est compilé en C++14 alors qu'il lie components compilé en C++17 | `apps/master/CMakeLists.txt:4` | non soumis |
| [BLD-11](#bld-11) | ⚪ basse | maintenabilite | Hygiène CMake : flags globaux mutés, versions erronées, répertoires listés comme sources | `apps/openmw-mp/CMakeLists.txt:26` | non soumis |
| [BLD-12](#bld-12) | ⚪ basse | securite | Dépendances FetchContent vérifiées par MD5 et téléchargées comme archives GitHub | `extern/CMakeLists.txt:184` | non soumis |
| [BLD-13](#bld-13) | ⚪ basse | build-ci | Option ENABLE_BREAKPAD morte : sous-module non initialisé et bibliothèque précompilée attendue dans l'arbre source | `apps/openmw-mp/CMakeLists.txt:9` | non soumis |
| [BLD-14](#bld-14) | ⚪ basse | manque | Aucun empaquetage ni procédure de build documentée pour le serveur dédié | `CMakeLists.txt:1022` | non soumis |
| [BLD-15](#bld-15) | ⚪ basse | bug | Une erreur de démarrage (script absent) termine le serveur par std::terminate et un core dump | `apps/openmw-mp/main.cpp:313` | non soumis |
| [SA-7](#sa-7) | ⚪ basse | securite | Membres de structures réseau non initialisés, sérialisés tels quels (fuite de pile) | `apps/openmw/mwmp/ObjectList.cpp:96` | partiel (sévérité corrigée) |
| [SA-8](#sa-8) | ⚪ basse | securite | Fonctions C non bornées et non réentrantes dans le chemin de log/horodatage | `components/openmw-mp/Utils.cpp:57` | partiel (sévérité corrigée) |
| [SA-9](#sa-9) | ⚪ basse | bug | sprintf dans un buffer fixe pour l'horodatage (dépassement de format possible) | `components/openmw-mp/TimedLog.cpp:53` | non soumis |
| [SA-10](#sa-10) | ⚪ basse | bug | Incompatibilités de type dans les chaînes de format (%i pour size_type) | `apps/openmw-mp/CellController.cpp:83` | non soumis |
| [SA-11](#sa-11) | ⚪ basse | maintenabilite | Valeurs de retour ignorées (I/O, system) et blocs catch vides | `components/openmw-mp/Utils.cpp:152` | non soumis |
| [SA-13](#sa-13) | ⚪ basse | build-ci | Outillage d'analyse statique absent de la CI | `components/openmw-mp/TimedLog.hpp:6` | non soumis |
| [SA-12](#sa-12) | ℹ️ info | performance | Occasions de performance à faible risque relevées en masse | `components/openmw-mp/Base/BaseStructs.hpp:1` | non soumis |

### BLD-1

**tes3mp-server ne compile avec aucun compilateur moderne testé (constexpr + reinterpret_cast)**

- **Sévérité** : 🔴 critique · **Catégorie** : bug · **Effort** : M
- **Emplacement** : `apps/openmw-mp/Script/Types.hpp:104`

**Preuve**

> Types.hpp:104 : `constexpr ScriptFunctionPointer(Function<R, Types...> addr) : ScriptIdentity(addr), addr((void*)(addr)) {}`, utilisé dans `static constexpr ScriptFunctionData functions[]` (ScriptFunctions.hpp:120).
> Log : « Types.hpp:104:94: error: 'reinterpret_cast' is not a constant expression ». 34 unités de compilation FAILED, 69 erreurs.
> Une reproduction minimale est rejetée par g++-9 à g++-14 ; clang++-18 rejette la branche #else (Types.hpp:107, « cannot initialize a member subobject of type 'void *' »).

**Impact** — Le serveur dédié est impossible à compiler sur toute distribution actuelle (Ubuntu 20.04+, Debian 11+). Le code n'était probablement accepté que par GCC 8 (la CI Travis utilisait gcc-8). Le binaire principal du dépôt ne peut donc pas être reproduit depuis les sources.

**Recommandation** — Séparer les métadonnées constexpr de l'adresse de fonction. Correctif testé, qui compile avec GCC 13 et clang 18 et passe un appel Lua→C++ réel :
1. Dans Types.hpp, ajouter `template<auto F> void* ScriptAddrOf() noexcept { return reinterpret_cast<void*>(F); }`.
2. Remplacer le membre `void *addr` par `void* (*getAddr)() noexcept`, et ajouter un second paramètre au constructeur.
3. Ajouter la macro `#define SFP(f) ScriptFunctionPointer(f, &ScriptAddrOf<f>)`.
4. Réécrire les 764 entrées `{"Nom", XFunctions::Fn}` en `{"Nom", SFP(XFunctions::Fn)}` dans Script/Functions/*.hpp et ScriptFunctions.hpp.
5. Remplacer `.func.addr` par `.func.getAddr()` dans LangLua.cpp:81 et LangNative.cpp:46.
Patch : <scratchpad-audit>/server-constexpr-fix.patch

**Contre-vérification (confirmé)** — Types.hpp:104 contient bien `addr((void*)(addr))` dans un constructeur constexpr utilisé par `static constexpr ScriptFunctionData functions[]` (ScriptFunctions.hpp:120). Le build GCC 13 sur les sources du dépôt (scratchpad/build-server/build.log) donne 34 FAILED et 69 erreurs « 'reinterpret_cast' is not a constant expression ». Ma reproduction minimale est rejetée par g++ 13, et clang 18 rejette la branche #else (Types.hpp:107, « cannot initialize a member subobject of type 'void *' »). Le seul build clang réussi a été fait sur une copie patchée (src/tes3mp-patched).

### SA-1

**Écriture hors bornes distante via l'index d'équipement non vérifié dans ID_PLAYER_EQUIPMENT**

- **Sévérité** : 🔴 critique · **Catégorie** : securite · **Effort** : S
- **Emplacement** : `components/openmw-mp/Packets/Player/PacketPlayerEquipment.cpp:42`

**Preuve**

> RW(equipmentIndex, send); ExchangeItemInformation(player->equipmentItems[equipmentIndex], send); // equipmentIndex est un std::vector<int> (BasePlayer.hpp:205), lu du réseau, jamais borné ; equipmentItems est un Item[19] (BasePlayer.hpp:249)

**Impact** — Un client malveillant envoie un ID_PLAYER_EQUIPMENT avec exchangeFullInfo=false et un equipmentIndex arbitraire (int, négatif ou >=19). ExchangeItemInformation fait RW(...,send=false), c'est-à-dire lit les données du flux ET les écrit dans equipmentItems[equipmentIndex] hors du tableau de 19 éléments de l'objet Player : écriture mémoire sauvage de contenu attaquant -> corruption/crash du serveur, exploitable à distance après handshake. Les paquets frères (skill, attribute, statsDynamic) ont ajouté exactement ce garde-fou, pas celui-ci (ni dans 0.8.2 ni dans master en amont).

**Recommandation** — Après RW(equipmentIndex, send), rejeter le paquet si equipmentIndex < 0 || equipmentIndex >= 19 (mettre packetValid=false et return), sur le modèle de PacketPlayerSkill.cpp:41. Signaler le correctif en amont TES3MP.

**Contre-vérification (confirmé)** — PacketPlayerEquipment.cpp:41-42 fait `RW(equipmentIndex, send); ExchangeItemInformation(player->equipmentItems[equipmentIndex], send);` sans aucune borne, alors que equipmentItems est un `Item equipmentItems[19]` (BasePlayer.hpp:249) et que l'index est un int (BasePlayer.hpp:205). Côté serveur, PlayerProcessor::Process appelle `myPacket->Read()` (PlayerProcessor.cpp:20) avant ProcessorPlayerEquipment::Do, sans aucune validation. Le paquet frère PacketPlayerSkill.cpp:41 contient bien `if (skillId >= 27) return;`. Il s'agit donc d'une écriture hors bornes déclenchable par un client.

### SA-2

**Déni de service mémoire : resize() sur un compteur uint32 non plafonné dans 24 paquets**

- **Sévérité** : 🔴 critique (initialement haute) · **Catégorie** : securite · **Effort** : M
- **Emplacement** : `components/openmw-mp/Packets/Player/PacketPlayerInventory.cpp:27`

**Preuve**

> RW(count, send); ... player->inventoryChanges.items.clear(); player->inventoryChanges.items.resize(count); // count est un uint32_t lu du réseau, aucun plafond

**Impact** — count (jusqu'à ~4 milliards) déclenche une allocation géante : std::bad_alloc, qui dans les processeurs noexcept (PlayerProcessor::Process est noexcept) provoque std::terminate -> crash du serveur à distance, ou épuisement mémoire. Patron répété dans 24 paquets (unbounded.txt) : PacketPlayerSpellbook, PacketPlayerJournal, PacketPlayerBook, PacketPlayerFaction, PacketPlayerQuickKeys, PacketCellReset, PacketClientScriptLocal, etc. Contraste avec PacketActorList/ObjectPacket qui plafonnent à maxActors/maxObjects.

**Recommandation** — Définir une borne par paquet (ex. maxInventoryItems) et, après RW(count,...), faire packetValid=false; return si count dépasse la borne, AVANT tout resize. Idéalement factoriser un helper readBoundedCount(max) dans BasePacket.

**Contre-vérification (partiel (sévérité corrigée))** — Confirmé : PacketPlayerInventory.cpp:22-27 fait RW(count) puis `items.resize(count)` sans plafond. 16 fichiers de Packets contiennent `resize(count)` (24 sites dans unbounded.txt), alors qu'ActorPacket.cpp:64 et ObjectPacket.cpp:63 plafonnent à maxActors et maxObjects. La lecture se fait dans PlayerProcessor::Process, déclaré noexcept (PlayerProcessor.cpp:9-20) : un std::bad_alloc y provoque std::terminate. C'est un crash du serveur déclenchable à distance par tout joueur connecté, donc « critique » selon la grille fournie.

### BLD-2

**CrabNet (RakNet) cloné sans version épinglée et non compilable tel quel avec GCC 13**

- **Sévérité** : 🟠 haute · **Catégorie** : build-ci · **Effort** : S
- **Emplacement** : `CI/before_install.linux.sh:10`

**Preuve**

> before_install.linux.sh:10 : `git clone https://github.com/TES3MP/CrabNet` (aucun commit ni tag ; master = 19e6619 du 2020-01-19).
> Build : « Source/Utils/_FindFirst.h:50:15: error: 'intptr_t' was not declared in this scope », 11 erreurs, `ninja: build stopped`.

**Impact** — La recette de build fournie par le dépôt échoue dès la première dépendance avec un compilateur récent. Un build sera différent si CrabNet master bouge. La dépendance réseau centrale du projet n'est ni versionnée ni testée.

**Recommandation** — Épingler CrabNet sur un commit précis (19e6619 ou un fork corrigé) :
- soit en sous-module extern/CrabNet ;
- soit avec `FetchContent_Declare(crabnet GIT_REPOSITORY … GIT_TAG 19e66190e83f…)` plus un PATCH_COMMAND qui ajoute `#include <cstdint>` dans Source/Utils/_FindFirst.h.
En attendant, documenter et appliquer `-DCMAKE_CXX_FLAGS='-include cstdint'` (validé : la bibliothèque compile en 9 s).

**Contre-vérification (confirmé)** — CI/before_install.linux.sh:10 fait `git clone https://github.com/TES3MP/CrabNet` sans checkout de tag ni de commit (HEAD = 19e6619 du 2020-01-19). Le log crabnet-build.log montre 11 erreurs, dont « _FindFirst.h:50:15: error: 'intptr_t' was not declared », parce que ce fichier n'inclut que <dirent.h> et <string>. Le risque de dérive reste théorique puisque master n'a pas bougé depuis 2020, mais l'échec de la recette avec GCC 13 est réel.

### BLD-4

**CI entièrement inopérante pour TES3MP (Travis mort, GitLab/AppVeyor sans RakNet ni LuaJIT, aucune GitHub Actions)**

- **Sévérité** : 🟠 haute · **Catégorie** : build-ci · **Effort** : M
- **Emplacement** : `.travis.yml:74`

**Preuve**

> .travis.yml:74 : `- ${ANALYZE} make -j3; fi` (« fi » orphelin, erreur de syntaxe), avec `dist: bionic` et gcc-8 ou clang-7 (lignes 21 et 49).
> `grep -rli 'raknet|crabnet' CI/ .gitlab-ci.yml appveyor.yml` ne trouve que before_script.linux.sh et before_install.linux.sh, que les jobs GitLab n'appellent pas. install_debian_deps.sh n'installe pas libluajit-5.1-dev.
> .github/ ne contient que FUNDING.yml.

**Impact** — Aucun pipeline ne compile le serveur, ce qui explique que l'erreur bloquante BLD-1 et la régression CrabNet soient passées inaperçues. Les contributions au fork ne sont pas vérifiées et aucun binaire n'est produit automatiquement.

**Recommandation** — Supprimer .travis.yml et appveyor.yml, et réduire .gitlab-ci.yml. Créer .github/workflows/server.yml :
- Déclenchement : on push et pull_request.
- Job linux : runs-on ubuntu-24.04, matrice cxx [g++-13, clang++-18], fail-fast false.
- Étapes :
  1. actions/checkout@v4 avec fetch-depth: 0 (nécessaire pour git-version).
  2. apt install libboost-{filesystem,program-options,system,iostreams}-dev libluajit-5.1-dev liblz4-dev libsdl2-dev libgl-dev libopenscenegraph-dev ninja-build ccache.
  3. actions/cache sur ~/crabnet, clé `crabnet-19e6619-${{matrix.cxx}}`. En cas d'absence du cache : clone épinglé puis cmake -DCRABNET_ENABLE_DLL=OFF -DCMAKE_CXX_FLAGS='-include cstdint' et install.
  4. hendrikmuhs/ccache-action, clé par compilateur.
  5. actions/cache sur build/extern/fetched.
  6. `cmake -G Ninja -DBUILD_OPENMW=OFF -DBUILD_OPENCS=OFF -DBUILD_LAUNCHER=OFF -DBUILD_WIZARD=OFF -DBUILD_BROWSER=OFF -DBUILD_MASTER=ON -DRakNet_LIBRARY_DEBUG=… -DCMAKE_EXPORT_COMPILE_COMMANDS=ON` puis `ninja tes3mp-server masterserver`.
  7. Test de fumée : serverCore.lua minimal qui appelle StopServer(0), MasterServer désactivé ; attendre « Quitting peacefully » avec un code de sortie 0.
  8. upload-artifact.
- Jobs secondaires :
  - clang-tidy sur apps/openmw-mp et components/openmw-mp (via compile_commands) ;
  - build ASan/UBSan avec le même test de fumée ;
  - windows-2022 avec vcpkg (boost, luajit, lz4, sdl2) en continue-on-error au début ;
  - build client complet (BUILD_OPENMW=ON) en cron hebdomadaire.

**Contre-vérification (partiel (sévérité corrigée))** — Le fond est confirmé : .travis.yml:74 contient `- ${ANALYZE} make -j3; fi`, un « fi » orphelin, et .github/ ne contient que FUNDING.yml. La preuve est en revanche inexacte : les jobs GitLab APPELLENT bien CI/before_script.linux.sh (.gitlab-ci.yml:23 et 44). Ce script pointe vers ~/CrabNet (before_script.linux.sh:27-28 et 47), mais rien ne compile CrabNet côté GitLab, donc `find_package(RakNet REQUIRED)` (CMakeLists.txt:198) échoue. La conclusion « aucun pipeline ne compile le serveur » tient.

### SA-4

**Course de données : la std::map du serveur maître partagée entre deux threads sans verrou**

- **Sévérité** : 🟠 haute · **Catégorie** : bug · **Effort** : M
- **Emplacement** : `apps/master/RestServer.cpp:85`

**Preuve**

> for (auto query = serverMap->begin(); query != serverMap->end(); query++) // thread HTTP, pendant que MasterServer::Thread fait servers.insert()/servers.erase() (MasterServer.cpp:144,164) ; grep mutex|lock sur apps/master = 0 occurrence de synchronisation

**Impact** — MasterServer::Thread (boucle RakNet) et RestServer (thread Boost.Asio, main.cpp:31) accèdent concurremment au même std::map servers (passé par pointeur, main.cpp:17) sans mutex. Insertions/suppressions côté réseau pendant une itération/at() côté HTTP -> invalidation d'itérateurs, lecture de nœuds libérés : comportement indéfini, crash du serveur maître sous charge.

**Recommandation** — Protéger tout accès à servers par un std::mutex partagé (verrou dans MasterServer::Thread pour insert/erase et dans chaque lambda de RestServer), ou exposer une copie instantanée sous verrou pour les lectures REST.

**Contre-vérification (confirmé)** — RestServer.cpp:85 itère sur serverMap, qui est &servers (MasterServer.cpp:229-231, main.cpp:17). Le thread lancé par MasterServer.cpp:200 fait servers.erase/insert (lignes 63, 144 et 164), et la boucle REST tourne dans server_thread (main.cpp:31). grep mutex|lock_guard|atomic sur apps/master ne renvoie rien. L'erreur ligne 63 (purge des serveurs expirés) rend la course possible même sans attaquant.

### BLD-3

**Configuration impossible avec CMake 4.x (cmake_minimum_required 3.1 placé après project())**

- **Sévérité** : 🟡 moyenne (initialement haute) · **Catégorie** : build-ci · **Effort** : M
- **Emplacement** : `CMakeLists.txt:2`

**Preuve**

> CMakeLists.txt:1-2 : `project(OpenMW)` puis `cmake_minimum_required(VERSION 3.1.0)`.
> CMake 4.4.5 : « CMake Error at CMakeLists.txt:2 (cmake_minimum_required): Compatibility with CMake < 3.5 has been removed from CMake. »
> Avec -DCMAKE_POLICY_VERSION_MINIMUM=3.5 : avertissements CMP0167 (FindBoost retiré, ligne 440) et CMP0169 (FetchContent_Populate déprécié).

**Impact** — Le build est bloqué sur toutes les distributions et outils qui livrent CMake 4 (pip, Homebrew, distributions en rolling release). Sous 3.28, on obtient 3 avertissements de dépréciation. La dette de politiques CMake va casser le build à court terme (module FindBoost retiré).

**Recommandation** — 1. Placer `cmake_minimum_required(VERSION 3.16...3.31)` avant `project()`.
2. Supprimer les blocs `if(POLICY CMP0069/CMP0083)` (lignes 7-14) et le cmake_minimum_required(3.1) de la ligne 836.
3. Passer à `find_package(Boost CONFIG …)` en retirant `set(Boost_NO_BOOST_CMAKE ON)` (ligne 438).
4. Dans extern/CMakeLists.txt:9, remplacer `FetchContent_Populate` par `FetchContent_MakeAvailable` avec `EXCLUDE_FROM_ALL` (CMake ≥ 3.28) et ajouter `DOWNLOAD_EXTRACT_TIMESTAMP TRUE`.

**Contre-vérification (partiel (sévérité corrigée))** — Reproduit : CMake 4.4.5 échoue à CMakeLists.txt:2 avec « Compatibility with CMake < 3.5 has been removed », et cmake4b.log montre bien CMP0167 et CMP0169. Il existe toutefois un contournement d'un seul drapeau (-DCMAKE_POLICY_VERSION_MINIMUM=3.5), et les distributions LTS (Ubuntu 24.04 : 3.28) livrent encore CMake 3. Le build n'est donc pas « bloqué » au sens fort : c'est une dette de configuration de sévérité moyenne.

### BLD-5

**Crash (SIGSEGV) à l'arrêt du serveur : course entre le thread MasterClient et la destruction de Networking**

- **Sévérité** : 🟡 moyenne (initialement haute) · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw-mp/MasterClient.cpp:188`

**Preuve**

> MasterClient.cpp:188-190 (exécuté dans le thread) : `sRun = true;` puis `queryData.SetPassword((int) Networking::get().isPassworded());`. Stop(), lignes 243-244 : `if (!sRun) return;`, donc pas de join si le thread n'a pas encore démarré.
> Backtrace du core : « #2 mwmp::Networking::isPassworded (this=0x0) at Networking.cpp:89 / #3 MasterClient::Thread at MasterClient.cpp:190 ».
> SIGSEGV sur 3 exécutions sur 3 avec MasterServer activé, aucun sans.

**Impact** — Tout arrêt rapide du serveur provoque un segfault et un core dump : script qui échoue à l'initialisation, test CI, redémarrage orchestré. `static bool sRun`, non atomique et partagé entre threads, est une data race. Un std::thread joignable non joint peut aussi appeler std::terminate. L'arrêt peut en outre attendre jusqu'à `rate` = 10 s à cause de RakSleep.

**Recommandation** — 1. Passer sRun en `std::atomic<bool>` (MasterClient.hpp:40).
2. Le mettre à true dans Start() avant `std::thread(...)`.
3. Lire isPassworded() dans Start(), côté thread principal.
4. Dans Stop(), faire `sRun = false; if (thrQuery.joinable()) thrQuery.join();` sans retour anticipé.
5. Remplacer RakSleep(timeout) par une condition_variable::wait_for notifiée par Stop().
6. Ajouter ce scénario (StopServer dans OnServerPostInit) au test de fumée de la CI.

**Contre-vérification (partiel (sévérité corrigée))** — La course est réelle. MasterClient.cpp:188-190 met `sRun = true` dans le thread, Stop() sort sans join si `!sRun` (lignes 243-244), et ~Networking met sThis à 0 (Networking.cpp:74). Le core dump scratchpad/build-server-patched/core montre bien `Networking::isPassworded (this=0x0)` à MasterClient.cpp:190. La fenêtre est toutefois étroite : il faut un arrêt avant la première instruction du thread (StopServer dans OnServerPostInit), et 5 exécutions gdb (gdb-try*.log) se terminent normalement. L'argument std::terminate est faux : mclient, créé par new (Networking.cpp:621), n'est jamais détruit, donc le destructeur de std::thread ne s'exécute pas.

### BLD-6

**Le build serveur seul exige SDL2, OpenGL, LZ4, les en-têtes OSG et un téléchargement de recastnavigation, tous inutiles au binaire**

- **Sévérité** : 🟡 moyenne · **Catégorie** : build-ci · **Effort** : M
- **Emplacement** : `CMakeLists.txt:454`

**Preuve**

> CMakeLists.txt:454 : `find_package(SDL2 2.0.9 REQUIRED)` est hors de tout garde. :202 `find_package(OpenGL REQUIRED)`, :204 `find_package(LZ4 REQUIRED)`, :418-420 « keep OSG's headers (HACK) » `include_directories(${OPENSCENEGRAPH_INCLUDE_DIRS})` avec une variable vide.
> extern/CMakeLists.txt:182-187 télécharge recastnavigation (le proxy renvoie 403 : « Each download failed! »), alors que components/CMakeLists.txt:383 ne le lie que pour le client.
> `ldd tes3mp-server` ne montre ni SDL, ni GL, ni LZ4, ni OSG.

**Impact** — Une image de serveur dédié est inutilement lourde (bibliothèques graphiques). La configuration échoue hors ligne ou derrière un proxy. Les en-têtes OSG sont une dépendance implicite que CMake ne détecte pas : leur absence donne une erreur de compilation peu claire (components/esm/defs.hpp inclut <osg/Vec3f>).

**Recommandation** — 1. Déplacer SDL2, OpenGL et LZ4 dans `IF(BUILD_OPENMW OR BUILD_OPENCS)`.
2. Conditionner le bloc recastnavigation de extern/CMakeLists.txt:169 par la même condition.
3. Remplacer le HACK OSG par `find_path(OSG_INCLUDE_DIR osg/Vec3f REQUIRED)`, à terme par une abstraction de vecteur sans OSG pour components/esm.
4. Documenter FETCHCONTENT_FULLY_DISCONNECTED et FETCHCONTENT_SOURCE_DIR_RECASTNAVIGATION pour les builds hors ligne.

### BLD-7

**FindRakNet.cmake fragile : bibliothèque Debug obligatoire, uniquement $ENV{RAKNET_ROOT}, en-têtes non-SYSTEM**

- **Sévérité** : 🟡 moyenne · **Catégorie** : build-ci · **Effort** : S
- **Emplacement** : `cmake/FindRakNet.cmake:52`

**Preuve**

> FindRakNet.cmake:52 : `SET(RakNet_LIBRARY optimized ${RakNet_LIBRARY_RELEASE} debug ${RakNet_LIBRARY_DEBUG})`. Configuration avec une seule bibliothèque Release : « CMake Error: … set to NOTFOUND … RakNet_LIBRARY_DEBUG linked by target "tes3mp-server" ».
> CMakeLists.txt:199 : `include_directories(${RakNet_INCLUDES})`, non SYSTEM, d'où 33 -Wdeprecated-copy (RakNetTypes.h) dans apps/openmw-mp.

**Impact** — Configuration en échec et message d'erreur trompeur pour toute installation standard de CrabNet (Release seule). La CI doit contourner le problème en passant RakNet_LIBRARY_DEBUG=…Static.a. Les avertissements des en-têtes tiers masquent ceux du projet.

**Recommandation** — Réécrire le module :
- find_path/find_library avec `HINTS ${RakNet_ROOT} $ENV{RAKNET_ROOT}` (politique CMP0074) ;
- repli sur la bibliothèque Release si la Debug est absente ;
- `find_package_handle_standard_args(RakNet REQUIRED_VARS …)` ;
- création d'une cible `RakNet::RakNet` IMPORTED avec INTERFACE_SYSTEM_INCLUDE_DIRECTORIES et ws2_32 sous Windows ;
- `target_link_libraries(tes3mp-server PRIVATE RakNet::RakNet)` à la place des variables globales.

### BLD-9

**La compatibilité client/serveur dépend du hash git au moment du build (build non reproductible)**

- **Sévérité** : 🟡 moyenne · **Catégorie** : architecture · **Effort** : S
- **Emplacement** : `apps/openmw-mp/main.cpp:227`

**Preuve**

> main.cpp:227-229 : `sstr << version.mCommitHash;` puis `peer->SetIncomingPassword(sstr.str().c_str(), …)`. Le client fait de même (apps/openmw/mwmp/Networking.cpp:284).
> Le hash vient de resources/version, généré par la cible git-version (components/CMakeLists.txt:9).
> Build depuis le clone : ligne 2 = 49be5b6… ; build depuis une copie sans .git : ligne vide (« Oldest compatible commit hash: » vide au démarrage).

**Impact** — Le moindre commit sur le fork, même de documentation, rend client et serveur incompatibles. Les builds depuis une archive source (paquets de distribution, Docker sans .git) produisent un hash vide. La compatibilité réseau ne dépend pas de TES3MP_PROTO_VERSION mais de l'état git de l'arbre, ce qui empêche des binaires reproductibles et des correctifs côté serveur seul.

**Recommandation** — Construire le mot de passe de connexion à partir de TES3MP_VERSION et TES3MP_PROTO_VERSION seulement, ou d'une constante `TES3MP_COMPAT_HASH` versionnée dans components/openmw-mp/Version.hpp. Ajouter une option `-DTES3MP_COMMIT_HASH=` qui court-circuite GitVersion.cmake pour les builds de paquets. Garder le hash git uniquement comme information affichée.

### SA-3

**Vulnérabilité de chaîne de format : LOG_* avec e.what() comme format, macros sans attribut printf**

- **Sévérité** : 🟡 moyenne (initialement haute) · **Catégorie** : securite · **Effort** : S
- **Emplacement** : `apps/openmw-mp/Script/Script.hpp:103`

**Preuve**

> LOG_MESSAGE_SIMPLE(TimedLog::LOG_ERROR, e.what()); // e.what() devient le format ; TimedLog.cpp:94-103 : sstr << message; vsnprintf(buf.data(), buf.size(), sstr.str().c_str(), args) sans argument variadique correspondant

**Impact** — Le texte d'une exception (erreur Lua, dont le message peut inclure des données fournies par le script ou par le client) est passé comme chaîne de format à vsnprintf sans arguments. Des spécificateurs % (%s, %n, %x) provoquent la lecture d'arguments inexistants -> crash/fuite d'information, %n une écriture. Même défaut en main.cpp:311. Comme TimedLog::print n'a pas d'attribut format(printf,6,7), le compilateur ne détecte rien ; mon expérience avec l'attribut ajouté révèle 2 sites de format non littéral et 2 incompatibilités de type.

**Recommandation** — Remplacer par LOG_MESSAGE_SIMPLE(TimedLog::LOG_ERROR, "%s", e.what()); idem main.cpp:311. Ajouter __attribute__((format(printf,6,7))) à TimedLog::print pour détecter durablement ces cas, et corriger les mésappariements %i/size_type révélés (CellController.cpp:83, Worldstate.cpp:313).

**Contre-vérification (partiel (sévérité corrigée))** — Le défaut est réel : Script.hpp:103 et main.cpp:311 passent e.what() comme format, et TimedLog.cpp:94-103 concatène le message puis appelle vsnprintf sans arguments. En revanche, l'API tes3mp.LogMessage est sûre (Functions/Server.cpp : `"[Script]: %s", message`). Pour qu'un client injecte des « % », il faudrait qu'un script inclue lui-même des données client dans un message d'erreur Lua, ce qui n'est pas démontré. La sévérité haute est surévaluée.

### SA-5

**Invalidation d'itérateur : reset() vide le conteneur en pleine boucle range-for (client)**

- **Sévérité** : 🟡 moyenne (initialement haute) · **Catégorie** : bug · **Effort** : M
- **Emplacement** : `apps/openmw/mwmp/ObjectList.cpp:761`

**Preuve**

> cppcheck: ObjectList.cpp:761: error: Calling 'reset' while iterating the container is invalid [invalidContainerLoop] — for (const auto &baseObject : baseObjects) { ... reset(); /* reset() fait baseObjects.clear() */ ... addEntireContainer(ptrFound); }

**Impact** — Dans restockObjects, reset() exécute baseObjects.clear() (ObjectList.cpp:62) alors que la boucle range-for détient une référence baseObject dans ce même vecteur et continue de l'utiliser (cell = ..., addEntireContainer) puis d'itérer : référence pendante et itérateurs invalidés -> comportement indéfini, crash du client lors d'un restock.

**Recommandation** — Ne pas réutiliser l'objet ObjectList en cours d'itération : copier les champs nécessaires de baseObject avant reset(), ou construire une nouvelle ObjectList locale pour l'envoi au lieu d'appeler reset() sur celle qu'on parcourt.

**Contre-vérification (partiel (sévérité corrigée))** — reset() fait bien baseObjects.clear() (ObjectList.cpp:62) dans la boucle range-for de restockObjects (ObjectList.cpp:748-765). En revanche, baseObject n'est plus utilisé après reset() : `cell` et addEntireContainer utilisent ptrFound, contrairement à ce qu'affirme le constat. Avec une liste à 1 élément, l'élément ajouté par addEntireContainer ne réalloue pas et `++it` atteint l'end mis en cache, donc la boucle se termine proprement. Le comportement indéfini (lecture d'éléments détruits) ne survient que si le serveur envoie un ID_OBJECT_RESTOCK avec au moins 2 objets, et ne touche que le client.

### SA-6

**std::terminate via exception échappant d'un binding Lua noexcept (index non vérifié)**

- **Sévérité** : 🟡 moyenne · **Catégorie** : bug · **Effort** : M
- **Emplacement** : `apps/openmw-mp/Script/Functions/Items.cpp:164`

**Preuve**

> int ItemFunctions::GetInventoryItemCount(..., unsigned int index) noexcept { ... return player->inventoryChanges.items.at(index).count; } // pas de contrôle de taille, alors que GetInventoryItemRefId:153 fait if (index >= ...size()) return

**Impact** — Plusieurs fonctions exposées à Lua et marquées noexcept appellent .at(index) sans vérifier la taille (Items.cpp:164,172,180) ou indexent des tableaux fixes sans borne (EquipItem -> equipmentItems[slot] Items.cpp:63, EquipActorItem Actors.cpp:481). Un script serveur passant un mauvais index lève std::out_of_range depuis une fonction noexcept -> std::terminate, arrêt brutal du serveur ; l'indexation de tableau fixe est en plus une écriture hors bornes. Incohérence : CharClass.cpp vérifie slot>4, Items.cpp non.

**Recommandation** — Uniformiser : vérifier index/slot contre la taille (ou 19 pour l'équipement) et renvoyer une valeur sentinelle sans lever, dans toutes les fonctions noexcept. Ne pas mélanger throw et noexcept sur la frontière Lua.

**Contre-vérification (confirmé)** — Items.cpp:160-180 : GetInventoryItemCount, GetInventoryItemCharge et GetInventoryItemEnchantmentCharge, marquées noexcept, appellent `.at(index)` sans le test que fait GetInventoryItemRefId (`if (index >= ...size()) return "invalid";`). EquipItem (Items.cpp:63-66) et EquipActorItem (Actors.cpp:481-484) indexent `equipmentItems[slot]` sans borne. Ces fonctions ne sont atteignables que par un script serveur, d'où une sévérité moyenne.

### BLD-10

**Le serveur maître est compilé en C++14 alors qu'il lie components compilé en C++17**

- **Sévérité** : ⚪ basse · **Catégorie** : build-ci · **Effort** : S
- **Emplacement** : `apps/master/CMakeLists.txt:4`

**Preuve**

> apps/master/CMakeLists.txt:4 : `add_definitions(-std=gnu++14)`.
> compile_commands.json pour apps/master/main.cpp : `'-std=gnu++17', '-std=gnu++14'` (le dernier l'emporte).

**Impact** — Mélange de standards entre unités de compilation liées ensemble : risque d'ODR ou d'ABI (variables inline, en-têtes de components qui supposent C++17). Le comportement de compilation est trompeur et dépend de l'ordre des flags.

**Recommandation** — Supprimer la ligne 4 et la ligne 3 commentée, puis ajouter `target_compile_features(masterserver PRIVATE cxx_std_17)`. Plus largement, remplacer les `set(CMAKE_CXX_STANDARD 17)` globaux par des target_compile_features.

### BLD-11

**Hygiène CMake : flags globaux mutés, versions erronées, répertoires listés comme sources**

- **Sévérité** : ⚪ basse · **Catégorie** : maintenabilite · **Effort** : S
- **Emplacement** : `apps/openmw-mp/CMakeLists.txt:26`

**Preuve**

> apps/openmw-mp/CMakeLists.txt:26 : `set(LuaScript_Headers ${LUA_INCLUDE_DIR} ${CMAKE_SOURCE_DIR}/extern/LuaBridge …)` (variable jamais définie, répertoires passés à add_executable). Ligne 31 : `set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DENABLE_LUA")`.
> CMakeLists.txt:440 : `find_package(Boost 1.6.2 …)`, sans doute pour 1.62. Ligne 579 : `add_definitions( -DBOOST_NO_CXX11_SCOPED_ENUMS=ON )` est global.

**Impact** — Les définitions fuient vers toutes les cibles du répertoire, le contrôle de version Boost est inopérant, et les listes de sources sont fausses. Le build en devient plus difficile à faire évoluer (passage aux cibles modernes, presets, vcpkg).

**Recommandation** — 1. Remplacer les mutations de CMAKE_CXX_FLAGS par `target_compile_definitions(tes3mp-server PRIVATE ENABLE_LUA)` (et ENABLE_BREAKPAD).
2. Retirer ${LUA_INCLUDE_DIR} et les répertoires de LuaScript_Headers, et utiliser `target_include_directories(... SYSTEM ...)`.
3. Corriger le minimum en `Boost 1.66`.
4. Supprimer BOOST_NO_CXX11_SCOPED_ENUMS, obsolète avec Boost ≥ 1.57 en C++17.
5. Mettre des parenthèses à la condition de CMakeLists.txt:592 (`GNU AND … OR …`).

### BLD-12

**Dépendances FetchContent vérifiées par MD5 et téléchargées comme archives GitHub**

- **Sévérité** : ⚪ basse · **Catégorie** : securite · **Effort** : S
- **Emplacement** : `extern/CMakeLists.txt:184`

**Preuve**

> extern/CMakeLists.txt:183-184 : `URL https://github.com/recastnavigation/recastnavigation/archive/e75adf86….zip` et `URL_HASH MD5=af905d121ef9d1cdfa979b0495cba059`.
> Il y a 4 occurrences de `URL_HASH MD5` (lignes 51, 86, 152, 184).

**Impact** — MD5 n'offre plus de garantie d'intégrité face à un attaquant. Les archives /archive/<sha>.zip de GitHub ne sont pas garanties stables au bit près, d'où des échecs de vérification possibles. La configuration dépend du réseau.

**Recommandation** — Passer à `URL_HASH SHA256=…` ou à `GIT_REPOSITORY` + `GIT_TAG <sha complet>` + `GIT_SHALLOW TRUE`. Mettre en cache build/extern/fetched en CI. Envisager de fournir ces dépendances par vcpkg (manifest vcpkg.json) pour Windows et macOS.

### BLD-13

**Option ENABLE_BREAKPAD morte : sous-module non initialisé et bibliothèque précompilée attendue dans l'arbre source**

- **Sévérité** : ⚪ basse · **Catégorie** : build-ci · **Effort** : M
- **Emplacement** : `apps/openmw-mp/CMakeLists.txt:9`

**Preuve**

> apps/openmw-mp/CMakeLists.txt:9 : `set(Breakpad_Library "${CMAKE_SOURCE_DIR}/extern/breakpad/src/client/linux/libbreakpad_client.a")`.
> `git submodule status` : `-e6d1c032baa222d8a8dc87813e9067199ec0266d extern/breakpad` (non initialisé). Aucune cible CMake ne construit cette bibliothèque.

**Impact** — Le rapport de crash du serveur ne peut pas être activé sans opérations manuelles non documentées (autotools dans l'arbre source, dépendance lss). Ce sous-module issu de chromium.googlesource alourdit les clones récursifs pour une fonctionnalité que personne ne teste.

**Recommandation** — Deux options :
- supprimer l'option et le sous-module ;
- la remplacer par une intégration réelle : ExternalProject_Add de breakpad, ou paquet vcpkg `breakpad`, ou sentry-native/crashpad, avec une cible importée et un job CI qui la compile une fois.

### BLD-14

**Aucun empaquetage ni procédure de build documentée pour le serveur dédié**

- **Sévérité** : ⚪ basse · **Catégorie** : manque · **Effort** : M
- **Emplacement** : `CMakeLists.txt:1022`

**Preuve**

> CMakeLists.txt:1022 : seul `INSTALL(PROGRAMS "${INSTALL_SOURCE}/tes3mp-server" …)` existe ; `grep masterserver CMakeLists.txt` ne trouve aucune règle d'installation.
> Aucun Dockerfile, CMakePresets.json, vcpkg.json ni conanfile dans le dépôt. README.md:41-49 se contente de renvoyer au wiki externe.

**Impact** — Déployer un serveur suppose de reproduire à la main la séquence découverte pendant l'audit (CrabNet patché, 2 contournements CMake, correctif constexpr). Il n'existe pas d'image de référence pour les hébergeurs, et le serveur maître n'est pas installable.

**Recommandation** — 1. Ajouter un Dockerfile multi-étapes : étape de build ubuntu:24.04 (CrabNet épinglé + `ninja tes3mp-server`), étape d'exécution debian-slim avec libluajit-5.1-2 et libboost-filesystem/program-options, plus un volume pour CoreScripts.
2. Ajouter un CMakePresets.json avec les presets `server-only` et `server-asan`.
3. Ajouter une règle `install(TARGETS masterserver)`.
4. Ajouter une section « Compiler le serveur » dans README.md.

### BLD-15

**Une erreur de démarrage (script absent) termine le serveur par std::terminate et un core dump**

- **Sévérité** : ⚪ basse · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw-mp/main.cpp:313`

**Preuve**

> main.cpp:309-313 : `catch (std::exception &e) { … throw; //fall through`.
> Exécution sans CoreScripts : « terminate called after throwing an instance of 'std::runtime_error' what(): Script not found: ./server/scripts/serverCore.lua ».

**Impact** — Sous systemd ou Docker, une simple erreur de configuration produit un abort (SIGABRT et core dump) au lieu d'un code de sortie propre. Le diagnostic et les politiques de redémarrage s'en trouvent compliqués.

**Recommandation** — Ajouter dans main() un try/catch englobant qui journalise e.what() et retourne EXIT_FAILURE. Vérifier l'existence de serverCore.lua avant l'initialisation de RakNet et afficher un message d'aide qui indique le chemin `home` de tes3mp-server-default.cfg.

### BLD-8

**Bugs réels détectés seulement par clang : arithmétique de pointeur sur chaînes et delete sans destructeur virtuel**

- **Sévérité** : ⚪ basse (initialement moyenne) · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw-mp/Script/LangLua/LangLua.cpp:262`

**Preuve**

> LangLua.cpp:262 : `throw std::runtime_error("C++ call: Unknown argument identifier " + argl[index]);`.
> clang 18 : « adding 'const char' to a string does not append to the string [-Wstring-plus-int] », aussi en LangLua.cpp:315, ScriptFunction.cpp:61 et Utils.cpp:95.
> Autre avertissement clang : « delete called on non-final 'mwmp::ActorProcessor' that has virtual functions but non-virtual destructor » (de même pour ObjectProcessor et WorldstateProcessor).

**Impact** — Dans les chemins d'erreur pilotés par les scripts Lua (identifiant de type inconnu passé à CallPublic/CreateTimerEx), le message est construit en lisant hors des limites du littéral : texte corrompu ou crash. Détruire un processeur par la base sans destructeur virtuel est un comportement indéfini. GCC 13 ne signale aucun de ces cas.

**Recommandation** — 1. Remplacer ces 4 constructions par `std::string("…") + c`.
2. Ajouter `virtual ~BasePacketProcessor() = default;` dans la base des processeurs.
3. Inclure clang dans la matrice CI (BLD-4) avec `-Werror=string-plus-int -Werror=delete-non-abstract-non-virtual-dtor -Werror=sign-compare`, limité aux cibles tes3mp-server et components.

**Contre-vérification (partiel (sévérité corrigée))** — Les quatre sites `"..." + char` existent bien (LangLua.cpp:262 et 315, ScriptFunction.cpp:61, Utils.cpp:95). Mais LangLua.cpp:262 reçoit des chaînes de types définies dans le C++ à la compilation. Pour LangLua.cpp:315, les caractères inconnus sont rejetés en amont par DefToVec et CreateTimerEx (LuaFunc.cpp:~50 et ~163, via un stringstream correct). Seul ret_type, non validé dans MakePublic (ScriptFunction.cpp:61), est atteignable, et uniquement depuis un script serveur de confiance, dans un chemin d'erreur. Le delete sans destructeur virtuel passe par unique_ptr<ActorProcessor> dans des maps statiques détruites à la sortie du programme (BasePacketProcessor.hpp:15) : c'est un UB formel sans impact pratique démontré.

### SA-10

**Incompatibilités de type dans les chaînes de format (%i pour size_type)**

- **Sévérité** : ⚪ basse · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw-mp/CellController.cpp:83`

**Preuve**

> clang -Wformat: CellController.cpp:83: format specifies type 'int' but the argument has type 'size_type' (aka 'unsigned long') ; idem Worldstate.cpp:313

**Impact** — Un size_t passé à %i est un comportement indéfini de printf et affiche une valeur tronquée/erronée sur LP64. Masqué car les macros LOG n'ont pas d'attribut format. Impact fonctionnel limité (logs faux) mais signale la classe entière de bugs non détectés.

**Recommandation** — Utiliser %zu pour les tailles, ou caster explicitement. Ajouter l'attribut format(printf) à TimedLog::print (cf. SA-3) pour que le build attrape ces cas.

### SA-11

**Valeurs de retour ignorées (I/O, system) et blocs catch vides**

- **Sévérité** : ⚪ basse · **Catégorie** : maintenabilite · **Effort** : M
- **Emplacement** : `components/openmw-mp/Utils.cpp:152`

**Preuve**

> cert-err33-c (14 occurrences) : Utils.cpp:152,154,235 (retours de fonctions I/O non vérifiés) ; bugprone-empty-catch : ScriptFunctions.cpp:30 et SimpleWeb/base_server.hpp:127 (catch {} vide)

**Impact** — 14 retours de fonctions pouvant échouer (écriture, vsnprintf, appels système) sont ignorés, et des catch vides masquent des exceptions : échecs silencieux difficiles à diagnostiquer en production (téléchargement de plugins, écriture de logs).

**Recommandation** — Vérifier et journaliser les retours d'I/O critiques ; dans les catch vides, au minimum journaliser l'exception ou commenter explicitement pourquoi elle est ignorée.

### SA-13

**Outillage d'analyse statique absent de la CI**

- **Sévérité** : ⚪ basse · **Catégorie** : build-ci · **Effort** : M
- **Emplacement** : `components/openmw-mp/TimedLog.hpp:6`

**Preuve**

> TimedLog.hpp:6-8 : #ifdef __GNUC__ #pragma GCC system_header — neutralise tous les avertissements de cet en-tête ; aucun .clang-tidy ni cible d'analyse dans le dépôt ; j'ai dû retirer ce pragma pour auditer le format

**Impact** — Aucun des 208 avertissements clang-tidy ni des défauts cppcheck n'est capté automatiquement, et le #pragma GCC system_header de TimedLog.hpp éteint même les avertissements du compilateur sur l'un des fichiers les plus utilisés (chemin de format). Les régressions de type SA-1/SA-3 passent inaperçues.

**Recommandation** — Ajouter un fichier .clang-tidy (checks bugprone/cert/clang-analyzer) et une étape cppcheck dans la CI sur apps/openmw-mp et components/openmw-mp ; retirer le #pragma GCC system_header de TimedLog.hpp et corriger les avertissements résultants.

### SA-7

**Membres de structures réseau non initialisés, sérialisés tels quels (fuite de pile)**

- **Sévérité** : ⚪ basse (initialement moyenne) · **Catégorie** : securite · **Effort** : M
- **Emplacement** : `apps/openmw/mwmp/ObjectList.cpp:96`

**Preuve**

> cppcheck: ObjectList.cpp:96: warning: Uninitialized variables: baseObject.refNum, baseObject.mpNum, baseObject.count, ... (33 champs) [uninitvar] ; BaseObject.hpp:30-31 : unsigned int refNum; unsigned int mpNum; (aucun initialiseur)

**Impact** — Les structures Base* (BaseObject, BasePlayer, BaseActor) n'ont pas d'initialiseurs de membres par défaut et dépendent d'appels blank() manuels. cppcheck relève 141 uninitMemberVar + 133 uninitStructMember et 19 uninitvar concrets où un BaseObject/clientLocal/activeEffect est construit sur la pile puis partiellement rempli avant sérialisation (ObjectList.cpp:529,1441 ; Worldstate.cpp:553). Les champs non écrits exposent des octets de pile non initialisés sur le réseau (fuite d'information) et rendent le comportement non déterministe.

**Recommandation** — Donner des initialiseurs par défaut à tous les POD des structures Base* (ex. unsigned int refNum = 0;) plutôt que de compter sur blank(), et/ou value-initialiser les instances locales (BaseObject baseObject{};).

**Contre-vérification (partiel (sévérité corrigée))** — Les membres non initialisés sont réels (BaseObject.hpp:30-31 : `unsigned int refNum; unsigned int mpNum;`). La fuite réseau n'est en revanche pas démontrée. Dans getBaseObjectFromPtr, la branche joueur (ObjectList.cpp:78-86) n'est sérialisée qu'avec guid, via `RW(isPlayer)` (PacketObjectActivate.cpp:23-25, PacketObjectHit.cpp:23-25). ObjectList.cpp:529 (activeEffect) est local et jamais sérialisé. Au pire, ce serait le client qui enverrait quelques octets de sa propre mémoire au serveur.

### SA-8

**Fonctions C non bornées et non réentrantes dans le chemin de log/horodatage**

- **Sévérité** : ⚪ basse (initialement moyenne) · **Catégorie** : securite · **Effort** : S
- **Emplacement** : `components/openmw-mp/Utils.cpp:57`

**Preuve**

> clang-analyzer-security.insecureAPI.strcpy: Utils.cpp:57 strcat(t, " "); // t est char[32] ; Utils.cpp:54 asctime(localtime(...)) ; bugprone-unsafe-functions + concurrency-mt-unsafe

**Impact** — timestamp() construit dans un char[32] via snprintf("[%s", asctime(...)) puis strcat sans contrôle ; asctime/localtime ne sont ni bornés ni réentrants. TimedLog utilise aussi localtime (TimedLog.cpp:51) et sprintf. Dans un serveur multi-thread (log appelé depuis la boucle réseau et les timers), ces fonctions à buffer statique partagé provoquent des races et un risque de dépassement si la locale produit une chaîne longue.

**Recommandation** — Remplacer asctime/localtime par localtime_r + strftime dans un buffer dimensionné, et strcat/sprintf par snprintf. Rendre getTime()/timestamp() réentrants.

**Contre-vérification (partiel (sévérité corrigée))** — Utils::timestamp (Utils.cpp:49-58) n'a aucun appelant : grep ne trouve que sa déclaration dans Utils.hpp:23. Par ailleurs, asctime produit une chaîne de longueur fixe (format de la locale C), donc 28 octets au plus tiennent dans char[32]. Le vrai défaut restant est getTime() (TimedLog.cpp:48-55) : localtime plus un buffer static partagé, appelés depuis le thread MasterClient et le thread principal. C'est une data race qui peut au pire corrompre un horodatage.

### SA-9

**sprintf dans un buffer fixe pour l'horodatage (dépassement de format possible)**

- **Sévérité** : ⚪ basse · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `components/openmw-mp/TimedLog.cpp:53`

**Preuve**

> static char result[20]; sprintf(result, "%.4d-%.2d-%.2d %.2d:%.2d:%.2d", 1900 + tm->tm_year, ...); // gcc: -Wformat-overflow: '%.2d' directive writing between 2 and 11 bytes into a region of size between 8 and 15

**Impact** — Le build GCC émet déjà -Wformat-overflow ici : si un champ dépasse (année anormale renvoyée par localtime), sprintf écrit au-delà de result[20] -> corruption mémoire. Peu probable en pratique mais le buffer de 20 octets est trop juste pour le format maximal.

**Recommandation** — Utiliser snprintf(result, sizeof(result), ...) et agrandir le buffer à 32 octets ; idéalement strftime.

### SA-12

**Occasions de performance à faible risque relevées en masse**

- **Sévérité** : ℹ️ info · **Catégorie** : performance · **Effort** : M
- **Emplacement** : `components/openmw-mp/Base/BaseStructs.hpp:1`

**Preuve**

> clang-tidy : performance-enum-size 87, performance-unnecessary-value-param 34 (ex. Networking::setServerPassword(std::string) copie), performance-avoid-endl 25, performance-for-range-copy (RestServer.cpp:165, main.cpp:241)

**Impact** — Paramètres std::string passés par valeur et copies dans les boucles range-for (ex. for (auto s : *serverMap)) le long de chemins chauds (sérialisation, requêtes maître), et énumérations surdimensionnées. Pas de bug, mais surcoût CPU/mémoire évitable sur du code réseau par paquet.

**Recommandation** — Passer les std::string/structs volumineux par const&, utiliser const auto& dans les boucles, et réduire les types sous-jacents d'enum. Traiter par lot avec clang-tidy --fix sur performance-*.

## Qualité de code & tests

| ID | Sévérité | Catégorie | Titre | Emplacement | Vérif. |
|---|---|---|---|---|---|
| [QUA-1](#qua-1) | 🔴 critique | bug | Compteurs réseau non bornés et retours de RW() ignorés : un paquet de quelques octets fait planter le serveur | `components/openmw-mp/Packets/Player/PacketPlayerInventory.cpp:27` | confirmé |
| [QUA-3](#qua-3) | 🟠 haute | securite | TimedLog::print sans attribut format et masqué par `#pragma GCC system_header` : chaînes de format non vérifiées, parfois contrôlées à distance | `components/openmw-mp/TimedLog.hpp:45` | confirmé |
| [QUA-4](#qua-4) | 🟠 haute | bug | TimerAPI : la réutilisation d'un slot libéré écrit dans une copie (fuite, minuterie nulle, crash) | `apps/openmw-mp/Script/API/TimerAPI.cpp:72` | confirmé |
| [QUA-6](#qua-6) | 🟠 haute | bug | Validation incohérente : packetValid ignoré, retours anticipés sans invalidation, .at() dans des fonctions noexcept | `components/openmw-mp/Packets/Worldstate/PacketRecordDynamic.cpp:85` | confirmé |
| [QUA-7](#qua-7) | 🟠 haute | tests | Aucun test du code multijoueur, testabilité bridée par les singletons et l'état global | `apps/openmw_test_suite/CMakeLists.txt:8` | confirmé |
| [QUA-2](#qua-2) | 🟡 moyenne | bug | Décalage d'un octet copié-collé 7 fois : BitStream construit sur data[1] avec la longueur complète | `apps/openmw-mp/Networking.cpp:559` | partiel (sévérité corrigée) |
| [QUA-5](#qua-5) | 🟡 moyenne | bug | Course de données : le thread du serveur maître parcourt la table globale des joueurs sans verrou | `apps/openmw-mp/MasterClient.cpp:193` | partiel (sévérité corrigée) |
| [QUA-8](#qua-8) | 🟡 moyenne | build-ci | La CI ne teste pas le multijoueur : le job tests désactive le serveur, aucun sanitizer | `CI/before_script.linux.sh:53` | non soumis |
| [QUA-9](#qua-9) | 🟡 moyenne | build-ci | Le code de script du serveur ne compile pas avec Clang (pointeur de fonction converti en void*) | `apps/openmw-mp/Script/Types.hpp:107` | non soumis |
| [QUA-10](#qua-10) | 🟡 moyenne | architecture | Instances de paquets partagées et mutables : un callback Lua peut rediriger le paquet en cours de relais | `apps/openmw-mp/processors/actor/ProcessorActorCellChange.hpp:54` | non soumis |
| [QUA-12](#qua-12) | 🟡 moyenne | maintenabilite | Duplication structurelle : 174 processeurs boilerplate, 9 Process() clonés, Send* et DefToVec copiés avec leurs bugs | `apps/openmw-mp/Script/LangLua/LuaFunc.cpp:166` | non soumis |
| [QUA-13](#qua-13) | 🟡 moyenne | maintenabilite | Fonctions géantes et chaînes if/else parallèles sur RECORD_TYPE | `components/openmw-mp/Packets/Worldstate/PacketRecordDynamic.cpp:15` | non soumis |
| [QUA-14](#qua-14) | 🟡 moyenne | maintenabilite | Propriété mémoire floue : pointeurs bruts propriétaires et operator[] qui insère des entrées nulles | `apps/openmw-mp/Player.cpp:11` | non soumis |
| [QUA-15](#qua-15) | 🟡 moyenne | performance | Relais coûteux : paquet re-sérialisé pour chaque destinataire et liste triée à chaque envoi | `apps/openmw-mp/Cell.cpp:184` | non soumis |
| [QUA-11](#qua-11) | ⚪ basse | bug | Pointeurs de lecture nuls déréférencés par l'API de script (readActorList, readObjectList) | `apps/openmw-mp/Script/Functions/Actors.cpp:69` | partiel (sévérité corrigée) |
| [QUA-16](#qua-16) | ⚪ basse | bug | Petits bugs latents détectables par tests unitaires ou analyse statique | `apps/openmw-mp/Script/ScriptFunction.cpp:61` | non soumis |
| [QUA-17](#qua-17) | ⚪ basse | maintenabilite | Garde-fous de style incohérents entre client et serveur (override, formatage) | `apps/openmw/mwmp/processors/player/ProcessorPlayerAnimFlags.hpp:17` | non soumis |
| [QUA-18](#qua-18) | ⚪ basse | maintenabilite | Code mort et placeholders : processeurs vides, processeur jamais routé, code commenté | `apps/openmw/mwmp/processors/player/ProcessorPlayerDisposition.hpp:18` | non soumis |

### QUA-1

**Compteurs réseau non bornés et retours de RW() ignorés : un paquet de quelques octets fait planter le serveur**

- **Sévérité** : 🔴 critique · **Catégorie** : bug · **Effort** : M
- **Emplacement** : `components/openmw-mp/Packets/Player/PacketPlayerInventory.cpp:27`

**Preuve**

> L.17-27 : `uint32_t count;` … `RW(count, send);` … `player->inventoryChanges.items.resize(count);`. Il n'y a aucune borne, et count reste non initialisé si la lecture échoue. Harnais ASan hors dépôt (qua/rt/roundtrip.cpp) avec action=0, count=0x7FFFFFFF : `AddressSanitizer: out of memory: allocator is trying to allocate 0x27ffffffb0 bytes`. Côté serveur, `bool PlayerProcessor::Process(RakNet::Packet &packet) noexcept` (apps/openmw-mp/processors/PlayerProcessor.cpp:9) transforme std::bad_alloc en std::terminate. Mesure : 963 RW(), 1 seul retour testé ; 21 des 25 fichiers qui font un resize() ne posent jamais packetValid=false ; 30 compteurs non initialisés.

**Impact** — Tout client connecté (après le handshake) peut arrêter le serveur via ID_PLAYER_INVENTORY, SPELLBOOK, JOURNAL, TOPIC, etc. Un paquet tronqué provoque un resize() sur une valeur indéterminée (comportement indéfini). Un paquet modéré, par exemple 20 M d'éléments, sature la mémoire. Un fuzzer trouve ce type de bug en quelques secondes.

**Recommandation** — Ajouter dans BasePacket un helper `bool RWCount(uint32_t &count, uint32_t max, size_t minBytesPerItem, bool write)`. Il initialise count à 0, teste le retour de bs->Read, refuse une valeur supérieure à max ou à `bs->GetNumberOfUnreadBits()/8/minBytesPerItem`, et pose packetValid=false. L'utiliser pour les 32 resize(). Retirer `noexcept` des 9 Process(), ou les entourer d'un try/catch qui journalise et expulse le client. Verrouiller le tout par le fuzzer et les tests de troncature décrits en QUA-7.

**Contre-vérification (confirmé)** — PacketPlayerInventory.cpp:17-27 : `uint32_t count;` non initialisé, puis `RW(count, send)` dont le retour est ignoré, puis `items.resize(count)` sans borne. Aucun garde-fou en amont : BasePacket::Read (BasePacket.cpp:69-71) appelle directement Packet(). Networking.cpp:180-191 exige seulement l'état POSTLOADED, et PlayerProcessor::Process (processors/PlayerProcessor.cpp:9) est noexcept sans try/catch, donc un std::bad_alloc déclenche std::terminate. Il suffit d'un client connecté pour arrêter le serveur.

### QUA-3

**TimedLog::print sans attribut format et masqué par `#pragma GCC system_header` : chaînes de format non vérifiées, parfois contrôlées à distance**

- **Sévérité** : 🟠 haute · **Catégorie** : securite · **Effort** : S
- **Emplacement** : `components/openmw-mp/TimedLog.hpp:45`

**Preuve**

> L.7 `#pragma GCC system_header`, L.45 `void print(int level, bool hasPrefix, const char *file, int line, const char *message, ...) const;` sans __attribute__((format)). En recompilant le serveur avec un en-tête fantôme qui ajoute format(printf,6,7) : 76 -Wformat (73 format-security, 2 types erronés en CellController.cpp:83 et Worldstate.cpp:311, 1 argument en trop en ProcessorPlayerCellChange.hpp:76). Côté client, apps/openmw/mwmp/Networking.cpp:419 : `LOG_MESSAGE_SIMPLE(TimedLog::LOG_ERROR, listDiscrepancies(checksums, checksumsResponse).c_str());`, où listDiscrepancies insère les noms de plugins envoyés par le serveur. Côté serveur, Script/Script.hpp:103 : `LOG_MESSAGE_SIMPLE(TimedLog::LOG_ERROR, e.what());`.

**Impact** — Un serveur malveillant affiché dans le navigateur peut faire planter un client, ou lui faire lire sa mémoire, avec des noms de plugins contenant %s ou %n. Une erreur Lua qui contient un texte saisi par un joueur avec % provoque un comportement indéfini côté serveur. Les erreurs de type dans les formats ne sont jamais signalées.

**Recommandation** — Ajouter `__attribute__((format(printf, 6, 7)))` sur TimedLog::print (macro vide sous MSVC) et supprimer le pragma system_header. Réécrire les 39 appels à format non littéral en `LOG_…(level, "%s", msg)` et corriger les 3 avertissements de type. Compiler ensuite le code mp avec `-Wformat=2 -Werror=format-security`.

**Contre-vérification (confirmé)** — TimedLog.hpp:7 `#pragma GCC system_header` et TimedLog.hpp:45 sans attribut format. TimedLog.cpp:99-103 passe le message comme chaîne de format à vsnprintf. Côté client, apps/openmw/mwmp/Networking.cpp:419 passe listDiscrepancies(...) comme format ; cette fonction insère les noms de plugins reçus du serveur (Networking.cpp:73 via PacketPreInit.cpp, sans filtrage des '%'). Un serveur malveillant contrôle donc ce format. Les erreurs de type sont visibles en CellController.cpp:83 (`%d` avec size()) et ProcessorPlayerCellChange.hpp:76 (argument en trop).

### QUA-4

**TimerAPI : la réutilisation d'un slot libéré écrit dans une copie (fuite, minuterie nulle, crash)**

- **Sévérité** : 🟠 haute · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw-mp/Script/API/TimerAPI.cpp:72`

**Preuve**

> L.72-77 : `for (auto timer : timers) { if (timer.second != nullptr) continue; timer.second = new Timer(lua, callback, msec, def, args); id = timer.first; }`. Ici timer est une copie de la paire, et la boucle n'a pas de break. Même code en L.96-101. Reproduction fidèle sous ASan/UBSan (qua/timer_repro.cpp), séquence Create, Free, Create, Stop : `runtime error: member call on null pointer of type 'struct Timer'` puis SEGV. StopTimer, IsTimerElapsed et ResetTimer (L.135, 162, 175) font `timers.at(id)->…` sans tester nullptr, et catch(...) n'attrape pas un SEGV.

**Impact** — Après un FreeTimer, chaque CreateTimer ou CreateTimerEx fuit un Timer par slot libre et renvoie un id dont le slot reste nullptr. La minuterie ne se déclenche jamais, et tes3mp.StopTimer, IsTimerElapsed ou RestartTimer sur cet id fait planter le serveur. De plus, Tick() (L.196) itère sur la map pendant que des callbacks Lua peuvent y insérer : un rehash invalide alors l'itérateur.

**Recommandation** — Réécrire TimerAPI avec `std::unordered_map<int, std::unique_ptr<Timer>>` et une pile d'ids libres. Itérer par référence et sortir de la boucle (break) dès qu'un slot est pris. Tester nullptr partout. Dans Tick(), collecter d'abord les timers échus dans un vecteur, puis appeler leurs callbacks. Ajouter des tests gtest (Create/Free/Create/Stop, création pendant Tick) : TimerAPI n'a besoin que d'une ScriptFunc C++.

**Contre-vérification (confirmé)** — TimerAPI.cpp:72-77 et 96-101 : `for (auto timer : timers)` itère sur une copie et n'a pas de break. L'affectation `timer.second = new Timer(...)` est perdue : le Timer fuit et l'id renvoyé désigne un slot nullptr. StopTimer (L.135), IsTimerElapsed (L.162) et ResetTimer font `timers.at(id)->` sans tester nullptr, et catch(...) n'intercepte pas un SEGV. Seul StartTimer teste nullptr ; la minuterie ne démarre donc jamais. Ces fonctions, FreeTimer compris, sont exposées aux scripts par Script/Functions/Timer.cpp:35-52.

### QUA-6

**Validation incohérente : packetValid ignoré, retours anticipés sans invalidation, .at() dans des fonctions noexcept**

- **Sévérité** : 🟠 haute · **Catégorie** : bug · **Effort** : M
- **Emplacement** : `components/openmw-mp/Packets/Worldstate/PacketRecordDynamic.cpp:85`

**Preuve**

> L.85-91 : `if (worldstate->recordsCount > maxRecords) { LOG_MESSAGE_SIMPLE(...); …; return; }` sans `isValid = false`, donc WorldstateProcessor appelle quand même OnRecordDynamic. Or RecordsDynamic.cpp:97-99 `return WorldstateFunctions::readWorldstate->recordsCount;` annonce plus d'éléments que le vecteur n'en contient, et les accesseurs font `spellRecords.at(index)` dans des fonctions noexcept. `grep isPacketValid` ne trouve que 2 usages (Networking.cpp:103 et :249) : PlayerProcessor::Process ignore le packetValid=false posé par PacketPlayerStatsDynamic.cpp:42. Mesure : 738 fonctions noexcept, 258 `.at(`.

**Impact** — Les callbacks Lua s'exécutent sur des données partielles ou incohérentes fournies par un client. Le moindre index hors bornes (bug CoreScripts ou donnée réseau) appelle std::terminate et arrête le serveur, au lieu de produire une erreur Lua récupérable.

**Recommandation** — Définir un contrat unique : tout parseur qui abandonne pose packetValid=false (fusionné avec isValid), et les 9 Process() rejettent le paquet avec un log et un compteur d'abus pouvant mener à l'expulsion. Côté API script, remplacer .at() par un accès vérifié qui renvoie une valeur neutre avec LOG_ERROR, ou retirer noexcept pour que LuaBridge convertisse l'exception en erreur Lua. GetRecordCount doit renvoyer la taille réelle du vecteur du type courant.

**Contre-vérification (confirmé)** — PacketRecordDynamic.cpp:85-91 retourne sans `worldstate->isValid = false`. WorldstateProcessor.cpp:31 appelle donc OnRecordDynamic alors que recordsCount vaut plus de 3000 et que les vecteurs proviennent du paquet précédent. GetRecordId et GetRecordSubtype (RecordsDynamic.cpp:107-125) font `.at(index)` dans des fonctions noexcept, ce qui mène à std::terminate si le script boucle sur GetRecordCount. Une nuance : la mesure « 2 usages de isPacketValid » est trompeuse, car les processeurs Actor, Object et Worldstate testent leur propre drapeau isValid (ActorProcessor.cpp:34, ObjectProcessor.cpp:34). PlayerProcessor, lui, ne teste aucun drapeau.

### QUA-7

**Aucun test du code multijoueur, testabilité bridée par les singletons et l'état global**

- **Sévérité** : 🟠 haute · **Catégorie** : tests · **Effort** : XL
- **Emplacement** : `apps/openmw_test_suite/CMakeLists.txt:8`

**Preuve**

> La liste `file(GLOB UNITTEST_SRC_FILES` (L.8-28) ne contient que mwworld, mwdialogue, esm, misc, nifloader, detournavigator, settings et shader ; `grep -rl 'mwmp\|openmw-mp' apps/openmw_test_suite` ne renvoie rien. Couplage mesuré : mwmp::Main::get() appelé 553 fois, Networking::get()/getPtr() 122 fois, ≈46 globales mutables (ex. Script/Functions/Actors.cpp:15-16 `BaseActorList *readActorList; BaseActorList writeActorList;`). Faisabilité vérifiée : le prototype qua/rt/roundtrip.cpp, lié à libcomponents.a et CrabNet, affiche `aller-retour: items=1 refId=iron_dagger count=3 -> OK`.

**Impact** — Les défauts QUA-1 à QUA-6 n'ont aucun filet de sécurité. Chaque rebase sur OpenMW (≈720 balises) ou évolution du protocole se vérifie à la main en jeu. Le serveur et l'API Lua ne se testent pas sans un vrai RakPeer et un CoreScripts complet.

**Recommandation** — Stratégie en 5 étages. (1) apps/openmw_test_suite/openmw-mp/test_packets.cpp : test gtest paramétré sur les ~95 paquets. Il remplit un Base* aléatoire, appelle Packet(&bs,true), relit et compare (ajouter ${RakNet_LIBRARY} au target). (2) Tests de troncature : rejouer chaque flux tronqué à toutes les longueurs et exiger packetValid=false, sans exception ni allocation de plus de 16 Mo. (3) Cible libFuzzer fuzz_packets.cpp : data[0] choisit le paquet parmi les 5 contrôleurs (peer=nullptr). Compilation clang avec -fsanitize=fuzzer,address,undefined, corpus issu de (1), 5 min par PR via ClusterFuzzLite. (4) API de script : introduire une interface IServerContext (joueurs, contrôleurs, horloge, cellules) injectée à la place de Networking::get(), regrouper l'état read/write dans un objet, tester TimerAPI et Script/Functions en C++, puis charger un script Lua de test via LangLua. (5) Intégration headless : lancer tes3mp-server avec un CoreScripts minimal et N clients robots CrabNet (PREINIT, HANDSHAKE, BASEINFO, LOADED, puis rafales de paquets valides et invalides), vérifier les relais attendus et l'absence de crash, sous ASan/UBSan puis TSan.

**Contre-vérification (confirmé)** — apps/openmw_test_suite/CMakeLists.txt:8-28 ne liste que mwworld, mwdialogue, esm, misc, nifloader, detournavigator et settings. `grep -rl 'mwmp\|openmw-mp' apps/openmw_test_suite | wc -l` renvoie 0. `grep -rn "Main::get()" apps/openmw | wc -l` renvoie bien 553. Les globales mutables sont confirmées en Script/Functions/Actors.cpp:15-16 (`BaseActorList *readActorList; BaseActorList writeActorList;`).

### QUA-10

**Instances de paquets partagées et mutables : un callback Lua peut rediriger le paquet en cours de relais**

- **Sévérité** : 🟡 moyenne · **Catégorie** : architecture · **Effort** : M
- **Emplacement** : `apps/openmw-mp/processors/actor/ProcessorActorCellChange.hpp:54`

**Preuve**

> L.51 `Script::Call<Script::CallbackIdentity("OnActorCellChange")>(…);` puis L.54 `packet.Send(true);`, sur l'instance unique renvoyée par GetPacket(ID_ACTOR_CELL_CHANGE). Or tes3mp.SendActorCellChange (Script/Functions/Actors.cpp:683-684) fait `GetPacket(ID_ACTOR_CELL_CHANGE)->setActorList(&writeActorList);`. ProcessorPlayerCellChange.hpp:73 contourne déjà le problème avec `packet.setPlayer(&player);` après l'appel Lua.

**Impact** — Si le script envoie un paquet du même type pendant le callback, le serveur relaie à tous la liste du script, avec son guid, au lieu de la liste reçue. La désynchronisation est silencieuse, difficile à diagnostiquer et impossible à tester sans injection.

**Recommandation** — Rendre les paquets sans état entre deux appels : transmettre les données en paramètre (`Send(const BaseActorList&, guid)`), ou copier le BitStream reçu avant Script::Call et relayer cette copie. À défaut, reposer systématiquement setActorList()/setPlayer() avant chaque Send() qui suit un Script::Call. Couvrir le cas par un test de processeur avec un faux script.

### QUA-12

**Duplication structurelle : 174 processeurs boilerplate, 9 Process() clonés, Send* et DefToVec copiés avec leurs bugs**

- **Sévérité** : 🟡 moyenne · **Catégorie** : maintenabilite · **Effort** : L
- **Emplacement** : `apps/openmw-mp/Script/LangLua/LuaFunc.cpp:166`

**Preuve**

> jscpd : 134 clones, 1 398 lignes (4,01 %). 65/79 processeurs serveur ont ≤3 lignes utiles (ex. ProcessorPlayerAnimFlags : `player.sendToLoaded(&packet);`). LuaFunc.cpp:12-58 est identique à 125-173, y compris le même bug : L.53 et L.166 `<< types[i] <<` au lieu de l'index relatif (lecture hors chaîne). Actors.cpp:550-701 contient 7 fonctions Send* identiques à l'ID près.

**Impact** — Chaque correctif doit être appliqué N fois : le décalage d'un octet de QUA-2 existe en 7 exemplaires. Ajouter un paquet impose de toucher 6 à 8 fichiers (paquet, contrôleur, processeur serveur, processeur client, deux initialiseurs).

**Recommandation** — Remplacer les processeurs de relais triviaux par une table déclarative (`RELAY_TO_LOADED(ID_PLAYER_ANIM_FLAGS)`, `CALL_SCRIPT(ID_ACTOR_AI, "OnActorAI")`). Factoriser les 9 Process() dans `BasePacketProcessor<T>::dispatch()` avec `processors.find(id)`. Fusionner DefToVec et la boucle de CreateTimerEx. Générer les Send* via un helper `sendActorPacket(id, toVisitors, skipAttached)`.

### QUA-13

**Fonctions géantes et chaînes if/else parallèles sur RECORD_TYPE**

- **Sévérité** : 🟡 moyenne · **Catégorie** : maintenabilite · **Effort** : L
- **Emplacement** : `components/openmw-mp/Packets/Worldstate/PacketRecordDynamic.cpp:15`

**Preuve**

> lizard : PacketRecordDynamic::Packet = 853 NLOC, CCN 156 ; Worldstate::addRecords (apps/openmw/mwmp/Worldstate.cpp:40) CCN 101 ; RecordsDynamicFunctions::AddRecord 155 NLOC. On compte 75 RECORD_TYPE:: dans le paquet, 323 dans Script/Functions/RecordsDynamic.cpp et 25 surcharges de RecordHelper::overrideRecord. Extrait L.24-26 : `if (worldstate->recordsType == mwmp::RECORD_TYPE::SPELL) worldstate->recordsCount = Utils::getVectorSize(worldstate->spellRecords); else if (… POTION)`.

**Impact** — Ajouter un type d'enregistrement oblige à modifier au moins 4 chaînes parallèles sans aide du compilateur, et un oubli produit silencieusement un paquet vide. La fonction est pratiquement impossible à couvrir branche par branche.

**Recommandation** — Créer une table de traits `template<RECORD_TYPE T> struct RecordTraits { using Rec = …; static std::vector<Rec>& vec(BaseWorldstate&); }` et un seul `visitRecordType(type, lambda)`, partagé par le paquet, le client et l'API de script. Découper Packet() en une fonction `serialize(XxxRecord&, bool)` par type, testable unitairement.

### QUA-14

**Propriété mémoire floue : pointeurs bruts propriétaires et operator[] qui insère des entrées nulles**

- **Sévérité** : 🟡 moyenne · **Catégorie** : maintenabilite · **Effort** : M
- **Emplacement** : `apps/openmw-mp/Player.cpp:11`

**Preuve**

> L.11 `if (players[guid] != 0)` : operator[] insère une entrée nullptr si le guid est absent, alors que doesPlayerExist() ne teste que la clé. Networking.cpp:45-49 alloue les contrôleurs par `new`, et :621 `mclient = new MasterClient(...)` n'est jamais libéré. 9 conteneurs de pointeurs propriétaires (TPlayers, TSlots, TimerAPI::timers, PublicFnAPI::publics, PlayerList::playerList, Cell::localActors/dedicatedActors, CellController::cellsInitialized…). clang-tidy : `constructor does not initialize these fields: id` (Player.cpp:67). Comptage : 325 new, 35 delete, 9 pointeurs intelligents.

**Impact** — Fuites et doubles libérations possibles sur les chemins d'erreur. Une entrée fantôme ferait déréférencer un Player nul dans processXxxPacket, qui ne teste jamais la nullité (Networking.cpp:94-222). La propriété des objets est difficile à suivre pour un nouveau contributeur.

**Recommandation** — Passer TPlayers à `std::map<RakNetGUID, std::unique_ptr<Player>>` et faire de TSlots une vue non propriétaire, avec find() au lieu de operator[]. Tenir les contrôleurs et MasterClient par valeur ou unique_ptr, et faire de même dans PlayerList, Cell, CellController et TimerAPI. Initialiser id. Activer cppcoreguidelines-owning-memory dans un .clang-tidy.

### QUA-15

**Relais coûteux : paquet re-sérialisé pour chaque destinataire et liste triée à chaque envoi**

- **Sévérité** : 🟡 moyenne · **Catégorie** : performance · **Effort** : M
- **Emplacement** : `apps/openmw-mp/Cell.cpp:184`

**Preuve**

> Cell::sendToLoaded (L.184-205) construit un std::list, exécute `plList.sort(); plList.unique();` puis `actorPacket->Send(pl->guid);` pour chaque joueur. BasePacket::Send (BasePacket.cpp:58-62) refait `bsSend->ResetWritePointer(); Packet(bsSend, true);` à chaque appel. Les Process() font un parcours linéaire `for (auto &processor : processors) if (processor.first == packet.data[0])` (serveur PlayerProcessor.cpp:11-13). clang-tidy : 25 performance-unnecessary-value-param (ex. Player.cpp:151, std::function passée par valeur).

**Impact** — Le coût est en O(joueurs de la cellule × taille du paquet) pour chaque position ou animation relayée, sur le chemin le plus chaud du serveur. La montée en charge des cellules peuplées en souffre.

**Recommandation** — Sérialiser une seule fois dans un BitStream local, puis appeler `peer->Send(&bs, priority, reliability, channel, guid, false)` pour chaque destinataire. Tenir dans Cell un ensemble déjà dédupliqué au lieu de trier à chaque envoi. Utiliser processors.find(id) et passer std::string/std::function en const&. Mesurer le gain avec un benchmark (l'option BUILD_BENCHMARKS existe déjà).

### QUA-2

**Décalage d'un octet copié-collé 7 fois : BitStream construit sur data[1] avec la longueur complète**

- **Sévérité** : 🟡 moyenne (initialement haute) · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw-mp/Networking.cpp:559`

**Preuve**

> `RakNet::BitStream bsIn(&packet->data[1], packet->length, false);`. La même ligne apparaît dans apps/openmw/mwmp/processors/{Actor,Object,Player,System,Worldstate}Processor.cpp:17-18 et apps/browser/netutils/Utils.cpp:44. Dans CrabNet BitStream.cpp:84, `numberOfBitsUsed = lengthInBytes << 3` sans copie. Reproduction ASan de la même construction : `heap-buffer-overflow … 0 bytes after 10-byte region`.

**Impact** — Chaque paquet reçu par le serveur ou le client autorise la lecture d'un octet au-delà du tampon alloué sur le tas. Un paquet tronqué lit cet octet au lieu d'échouer, ce qui fausse aussi la détection de troncature. C'est l'exemple type du coût de la duplication.

**Recommandation** — Factoriser dans components/openmw-mp une fonction `RakNet::BitStream makePayloadStream(RakNet::Packet&)` qui vérifie `packet->length >= 1 + RakNetGUID::size()` et construit `BitStream(packet->data + 1, packet->length - 1, false)`. Remplacer les 7 occurrences et ajouter un test gtest exécuté sous ASan.

**Contre-vérification (partiel (sévérité corrigée))** — Le code est confirmé : la même ligne `BitStream bsIn(&packet->data[1], packet->length, false)` apparaît en Networking.cpp:559, dans les 5 processeurs client (lignes 17-18) et en browser/netutils/Utils.cpp:44. Le constructeur CrabNet (BitStream.cpp:84) fixe numberOfBitsUsed = length<<3 sans copie, ce qui donne un octet lisible au-delà du tampon. L'impact se limite toutefois à la lecture d'un octet hors limite, et seulement sur un paquet tronqué ; elle tombe en pratique dans le bourrage de malloc. Il n'y a ni crash ni fuite exploitable démontrés dans le binaire réel : moyenne plutôt que haute.

### QUA-5

**Course de données : le thread du serveur maître parcourt la table globale des joueurs sans verrou**

- **Sévérité** : 🟡 moyenne (initialement haute) · **Catégorie** : bug · **Effort** : M
- **Emplacement** : `apps/openmw-mp/MasterClient.cpp:193`

**Preuve**

> L.193-208, dans le thread thrQuery : `auto *players = Players::getPlayers(); while (sRun) { … if (queryData.players[i] != pIt->second->npc.mName)`. Pendant ce temps, le thread principal exécute `delete players[guid]; players.erase(guid);` (Player.cpp:18-19). sRun est un `static bool` non atomique (MasterClient.hpp:40). Quand [MasterServer] enabled=false, mclient reste nul mais Networking.cpp:520 appelle `getMasterClient()->Process(packet)` et main.cpp:307 `->Stop()`.

**Impact** — Une connexion ou déconnexion qui coïncide avec une annonce au maître (au moins toutes les 8 s) peut provoquer un use-after-free ou le parcours d'un arbre en cours de modification. Le crash est aléatoire et non reproductible en jeu. TSan et UBSan (« member call on null pointer ») le signaleraient immédiatement.

**Recommandation** — Ne plus lire Players depuis le thread : à chaque connexion ou déconnexion, le thread principal publie un instantané (nombre et vector<string> des noms) protégé par mutexData. Faire de sRun un membre `std::atomic<bool>`. Porter mclient en std::unique_ptr et tester sa nullité avant Process() et Stop(). Remplacer les lock()/unlock() manuels par std::lock_guard. Ajouter un job TSan sur le test d'intégration.

**Contre-vérification (partiel (sévérité corrigée))** — La course de données est réelle. MasterClient.cpp:193-212 parcourt `*players` et lit `pIt->second->npc.mName` sans verrou : mutexData ne protège que queryData, pas Players. Pendant ce temps, Player.cpp:18-19 fait delete puis erase sur le thread principal, et sRun est un `static bool` non atomique (MasterClient.hpp:40). En revanche, l'appel sur mclient nul est neutralisé en pratique : Process (MasterClient.cpp:113) et Stop (L.243) ne lisent que le membre statique sRun avant de retourner (c'est un UB formel, sans crash). La fenêtre de course est de quelques microsecondes toutes les 8 s ou plus, d'où un crash rare : moyenne.

### QUA-8

**La CI ne teste pas le multijoueur : le job tests désactive le serveur, aucun sanitizer**

- **Sévérité** : 🟡 moyenne · **Catégorie** : build-ci · **Effort** : M
- **Emplacement** : `CI/before_script.linux.sh:53`

**Preuve**

> Branche BUILD_TESTS_ONLY : `-DBUILD_OPENMW_MP=OFF \` (L.53). Aucune occurrence de « sanitize » dans CMakeLists.txt ni dans CI/. L'option BUILD_WITH_CODE_COVERAGE (CMakeLists.txt:32) n'est jamais activée. .github ne contient que FUNDING.yml, et .travis.yml dépend de travis-ci.org.

**Impact** — Aucune régression n'est détectée avant publication, qu'il s'agisse de compilation sur un GCC récent, de crash de parseur, de fuite ou de course de données. Le dernier commit (« Fix gcc-13 build failures ») corrigeait justement une casse découverte après coup.

**Recommandation** — Créer .github/workflows/server.yml (ubuntu-24.04). Compiler CrabNet depuis les sources, puis `cmake -DBUILD_OPENMW=OFF -DBUILD_OPENCS=OFF -DBUILD_LAUNCHER=OFF -DBUILD_WIZARD=OFF -DBUILD_BROWSER=OFF -DBUILD_OPENMW_MP=ON -DBUILD_UNITTESTS=ON`. Matrice : GCC Release, GCC avec -fsanitize=address,undefined, Clang avec TSan. Exécuter ctest, CI/check_tabs.sh et un fuzzing court. Ajouter une option CMake `OPENMW_MP_SANITIZE`.

### QUA-9

**Le code de script du serveur ne compile pas avec Clang (pointeur de fonction converti en void*)**

- **Sévérité** : 🟡 moyenne · **Catégorie** : build-ci · **Effort** : S
- **Emplacement** : `apps/openmw-mp/Script/Types.hpp:107`

**Preuve**

> L.102-108 : branche GCC `addr((void*)(addr))`, branche #else `addr(addr)`. `clang++-18 -fsyntax-only Script/ScriptFunctions.cpp` produit 20 erreurs « cannot initialize a member subobject of type 'void *' with an lvalue of type 'Function<…>' ». clang-tidy remonte 640 clang-diagnostic-error sur apps/openmw-mp.

**Impact** — libFuzzer, MSan, Apple Clang et une analyse clang-tidy complète sont inutilisables sur le serveur. L'analyse statique des fichiers Script/Functions reste tronquée.

**Recommandation** — Rendre `functions[]` (ScriptFunctions.hpp:120) `static const` au lieu de constexpr, avec une initialisation dynamique et `reinterpret_cast<void*>(addr)` pour tous les compilateurs. `callbacks[]` peut rester constexpr pour le static_assert. Ajouter un job CI Clang.

### QUA-11

**Pointeurs de lecture nuls déréférencés par l'API de script (readActorList, readObjectList)**

- **Sévérité** : ⚪ basse (initialement moyenne) · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw-mp/Script/Functions/Actors.cpp:69`

**Preuve**

> L.37 : `readActorList = {};` (nullptr) quand la cellule n'est pas chargée. Seule GetActorListSize (L.61) teste la nullité ; L.69 `return readActorList->action;` et 49 autres déréférencements ne testent rien. Objects.cpp compte 72 `readObjectList->` et 0 test.

**Impact** — Un script qui lit la liste d'acteurs d'une cellule déchargée, cas courant lors des changements de cellule, provoque une erreur de segmentation du serveur au lieu d'une erreur Lua.

**Recommandation** — Faire pointer readActorList sur une liste vide statique (`static BaseActorList emptyActorList;`) plutôt que nullptr, et de même pour les objets. Regrouper l'état read/write dans un objet de contexte de script. Ajouter un test : ReadCellActorList sur une cellule inconnue, puis GetActorRefId(0).

**Contre-vérification (partiel (sévérité corrigée))** — Pour les acteurs, le constat est confirmé : Actors.cpp:37 `readActorList = {};` si la cellule n'est pas chargée, et L.69 `return readActorList->action;` ainsi que CopyReceivedActorListToStore (L.56) ne testent pas la nullité (50 occurrences de `readActorList->`). Mais GetActorListSize (L.61) renvoie 0 dans ce cas, et la boucle habituelle sur la taille n'accède donc à rien : il faut qu'un script appelle GetActorListAction ou CopyReceivedActorListToStore après une lecture vide. Pour les objets, le constat est réfuté : Objects.cpp:24 n'affecte readObjectList qu'avec getReceivedObjectList(), jamais avec nullptr.

### QUA-16

**Petits bugs latents détectables par tests unitaires ou analyse statique**

- **Sévérité** : ⚪ basse · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/openmw-mp/Script/ScriptFunction.cpp:61`

**Preuve**

> ScriptFunction.cpp:61 `throw std::runtime_error("Lua call: Unknown return type" + ret_type);` fait une arithmétique de pointeur sur un littéral. MasterClient.cpp:250-256 : SetUpdateRate borne timeout puis l'écrase par `timeout = rate;`. ScriptFunctions.hpp:37-39 : la macro GET_PLAYER assigne `pl` mais teste `player`. PacketMasterQuery.cpp:57 : port potentiellement non initialisé (clang-analyzer). Networking.cpp:488-496 : le gestionnaire de signal utilise std::cout et un `bool killLoop` ordinaire, et sigaction est réinstallé à chaque tour de boucle (L.513-514).

**Impact** — Comportements indéfinis ou logique morte, invisibles aujourd'hui mais qui se manifesteront au fil des évolutions : nouveau type de retour, autre nom de variable dans GET_PLAYER, arrêt du serveur par signal.

**Recommandation** — Corriger avec `std::string("…") + ret_type`, borner rate avant de l'affecter, remplacer GET_PLAYER par une fonction inline `Player *getPlayerOrLog(pid, func)`, initialiser port, utiliser `volatile std::sig_atomic_t` et installer les gestionnaires une seule fois avant la boucle. Activer bugprone-* et clang-analyzer-* en CI.

### QUA-17

**Garde-fous de style incohérents entre client et serveur (override, formatage)**

- **Sévérité** : ⚪ basse · **Catégorie** : maintenabilite · **Effort** : S
- **Emplacement** : `apps/openmw/mwmp/processors/player/ProcessorPlayerAnimFlags.hpp:17`

**Preuve**

> `virtual void Do(PlayerPacket &packet, BasePlayer *player)` : 96 déclarations Do() côté client sans override, contre 74/74 avec override côté serveur. 62 `virtual void Packet(` sans override dans components/openmw-mp/Packets ; clang-tidy remonte 213 modernize-use-override. Aucun .clang-format ni .clang-tidy. Caractère cyrillique dans apps/openmw/mwmp/Main.cpp:116 (« сonnect », octets 0xD1 0x81).

**Impact** — Si la signature de Do() change dans la classe de base, un processeur client devient une méthode fantôme jamais appelée, sans erreur de compilation. Faute de formatage automatique, les revues sont plus lentes.

**Recommandation** — Lancer `clang-tidy -checks=modernize-use-override -fix` sur apps/openmw/mwmp et components/openmw-mp. Ajouter un .clang-tidy minimal (bugprone-*, performance-*, modernize-use-override, cppcoreguidelines-owning-memory) et un .clang-format limité aux répertoires mp pour ne pas gêner les rebases OpenMW. Compiler avec -Wsuggest-override.

### QUA-18

**Code mort et placeholders : processeurs vides, processeur jamais routé, code commenté**

- **Sévérité** : ⚪ basse · **Catégorie** : maintenabilite · **Effort** : S
- **Emplacement** : `apps/openmw/mwmp/processors/player/ProcessorPlayerDisposition.hpp:18`

**Preuve**

> L.16-19 : `virtual void Do(PlayerPacket &packet, BasePlayer *player) { // Placeholder to be filled in later }`. Six autres processeurs client ont un Do() vide (CharGen, Input, Rest, ActorTest, WorldCollisionOverride). Côté serveur, ProcessorPlayerPlaceholder est enregistré comme PlayerProcessor sur ID_WORLD_KILL_COUNT, alors que cet ID appartient à WorldstatePacketController (ligne 31) : il n'est jamais atteint. On relève aussi 21 lignes de code commenté (ex. LuaFunc.cpp:26-36) et 34 DEBUG_PRINTF.

**Impact** — Le lecteur ne sait pas ce qui est réellement implémenté, et des identifiants de paquets restent acceptés sans sémantique. Le bruit gêne la revue et la mesure de couverture.

**Recommandation** — Supprimer ProcessorPlayerPlaceholder et les processeurs ou paquets placeholder, ou les isoler derrière un #ifdef documenté. Retirer le code commenté. Remplacer les DEBUG_PRINTF(strPacketID.c_str()) par un log unique, à format littéral, dans le dispatch factorisé.

## Écosystème, documentation & santé du projet

| ID | Sévérité | Catégorie | Titre | Emplacement | Vérif. |
|---|---|---|---|---|---|
| [ECO-1](#eco-1) | 🟠 haute | securite | API REST du serveur maître : écriture non authentifiée et JSON non échappé (empoisonnement de la liste publique) | `apps/master/RestServer.cpp:100` | confirmé |
| [ECO-2](#eco-2) | 🟠 haute | bug | Serveur maître : map des serveurs partagée entre le thread RakNet et le thread HTTP sans aucune synchronisation | `apps/master/MasterServer.cpp:63` | confirmé |
| [ECO-3](#eco-3) | 🟠 haute | architecture | Amont TES3MP en sommeil, fork sans divergence, correctifs de la branche 0.8.2 jamais publiés | `README.md:8` | confirmé |
| [ECO-4](#eco-4) | 🟠 haute | build-ci | Dépendance réseau CrabNet non épinglée, abandonnée depuis 2020 et non compilable avec GCC 13 | `CI/before_install.linux.sh:10` | confirmé |
| [ECO-5](#eco-5) | 🟡 moyenne | documentation | Aucune documentation propre à TES3MP dans le dépôt : la documentation Sphinx est celle d'OpenMW | `docs/source/conf.py:55` | partiel (sévérité corrigée) |
| [ECO-6](#eco-6) | 🟡 moyenne | bug | Arrêt du serveur maître : gestionnaire de signal non async-signal-safe, le processus se termine par std::terminate | `apps/master/main.cpp:19` | confirmé |
| [ECO-9](#eco-9) | 🟡 moyenne | bug | Navigateur : courses de données dans le ping, fuite de RakPeer, boucle active et ping bloquant dans le thread GUI | `apps/browser/PingUpdater.cpp:35` | confirmé |
| [ECO-10](#eco-10) | 🟡 moyenne | documentation | API Lua serveur : documentation non générée, callbacks non documentés, Doxygen configuré pour OpenMW | `apps/openmw-mp/Script/ScriptFunctions.hpp:156` | non soumis |
| [ECO-11](#eco-11) | 🟡 moyenne | documentation | CONTRIBUTING, CHANGELOG et CHANGELOG_PR hérités d'OpenMW, contradictoires ou à l'état de gabarit | `CONTRIBUTING.md:1` | non soumis |
| [ECO-12](#eco-12) | 🟡 moyenne | build-ci | Simple-Web vendorisé obsolète, incompatible avec Boost ≥ 1.87, et standards C++ forcés localement | `apps/master/SimpleWeb/base_server.hpp:199` | non soumis |
| [ECO-13](#eco-13) | 🟡 moyenne | build-ci | Gouvernance GitHub inexistante : CI morte, ni workflows, ni modèles, ni politique de sécurité | `.travis.yml:35` | non soumis |
| [ECO-7](#eco-7) | ⚪ basse | securite | Configuration serveur par défaut : publication automatique sur le maître public, sans mot de passe | `files/tes3mp/tes3mp-server-default.cfg:17` | partiel (sévérité corrigée) |
| [ECO-8](#eco-8) | ⚪ basse | bug | Ports du serveur maître codés en dur et incohérents avec les configurations livrées | `apps/master/main.cpp:16` | partiel (sévérité corrigée) |
| [ECO-14](#eco-14) | ⚪ basse | maintenabilite | Navigateur : fonctionnalité Favoris désactivée, code mort compilé, mot de passe passé en ligne de commande | `apps/browser/MainWindow.cpp:30` | non soumis |
| [ECO-15](#eco-15) | ⚪ basse | documentation | Chaîne de documentation cassée : .readthedocs.yaml obsolète et Sphinx 1.8.5 inutilisable | `.readthedocs.yaml:6` | non soumis |
| [ECO-16](#eco-16) | ⚪ basse | performance | Client configuré en journalisation verbeuse (« spam ») par défaut | `files/tes3mp/tes3mp-client-default.cfg:6` | non soumis |
| [ECO-17](#eco-17) | ⚪ basse | documentation | README : liens figés sur la branche master en retard, copyright daté, aucune mention du fork | `README.md:19` | non soumis |
| [ECO-18](#eco-18) | ⚪ basse | maintenabilite | .gitignore et .editorconfig hérités, ne couvrant pas les artefacts TES3MP | `.gitignore:46` | non soumis |

### ECO-1

**API REST du serveur maître : écriture non authentifiée et JSON non échappé (empoisonnement de la liste publique)**

- **Sévérité** : 🟠 haute · **Catégorie** : securite · **Effort** : M
- **Emplacement** : `apps/master/RestServer.cpp:100`

**Preuve**

> RestServer.cpp:100 `httpServer.resource[ServersRegex]["POST"]` n'a aucun contrôle d'accès ; la ligne 38 écrit `"\"hostname\": \"" << query.GetName() << "\""` sans échappement. Test local : POST avec hostname `a\"b` → GET renvoie `"hostname": "a"b"` → json.decoder.JSONDecodeError. 5 POST avec players=999 → `{"servers": 6, "players": 4996}` ; 4 195 requêtes acceptées en 68 s.

**Impact** — N'importe qui peut injecter des serveurs fictifs et des compteurs de joueurs arbitraires (un par port, jusqu'à 65 535 par IP) dans la liste publique. Un nom d'hôte contenant un guillemet, annoncé par REST ou par RakNet, casse le JSON renvoyé à tous les consommateurs (site web, outils tiers).

**Recommandation** — Supprimer les routes POST et PUT « < 0.6 » (lignes 99-158), qui n'ont plus d'usage depuis 0.6, ou les protéger par un jeton. Générer le JSON avec boost::property_tree::write_json ou nlohmann::json au lieu du stringstream manuel (lignes 33-45). Plafonner les entrées par IP (par exemple 8), borner players ≤ max_players et ajouter une limitation de débit par IP.

**Contre-vérification (confirmé)** — apps/master/RestServer.cpp:100-121 : le POST lit le JSON et insère dans serverMap (ligne 111) sans aucun contrôle d'accès. La ligne 38 concatène query.GetName() sans échappement. Le chemin RakNet ne filtre pas non plus le nom : MasterData.hpp:55 `rules["name"].str = name` et ProxyMasterPacket.hpp:55 se contentent de limiter la longueur à maxStringLength, sans filtrer les guillemets. Un seul nom d'hôte contenant un guillemet, même mis par accident, casse donc le JSON de toute la liste pour tous les consommateurs. La « non-authentification » tient en partie à la conception : le mot de passe RakNet est public, dans Version.hpp. L'injection JSON, elle, est un bug réel et facile à déclencher.

### ECO-2

**Serveur maître : map des serveurs partagée entre le thread RakNet et le thread HTTP sans aucune synchronisation**

- **Sévérité** : 🟠 haute · **Catégorie** : bug · **Effort** : M
- **Emplacement** : `apps/master/MasterServer.cpp:63`

**Preuve**

> MasterServer.cpp:63 `servers.erase(it++);` et :164 `servers.insert(...)` s'exécutent dans le thread RakNet. RestServer.cpp:85 (itération du GET) et :111 `serverMap->insert(...)` s'exécutent dans le thread HTTP. `grep -n mutex apps/master` (hors SimpleWeb) → aucun résultat. TSan confirme la course sur `run` : « data race MasterServer.cpp:209 in MasterServer::Stop » contre la lecture de MasterServer.cpp:51.

**Impact** — Un GET /api/servers peut itérer un std::map pendant qu'une annonce ou la purge à 60 s le modifie. Le résultat est un comportement indéfini (itérateur invalidé, plantage) déclenchable à distance, par de simples requêtes HTTP concurrentes aux annonces. TSan n'a pas signalé la course sur la map : les fonctions de rééquilibrage de std::map vivent dans libstdc++, qui n'est pas instrumentée.

**Recommandation** — Protéger ServerMap par un std::shared_mutex (verrou partagé pour GET, exclusif pour l'insertion, la purge et l'annonce), ou faire du thread maître le propriétaire unique et publier un instantané immuable (std::shared_ptr<const ServerMap>) pour le REST. Passer `bool run` en std::atomic<bool> (MasterServer.hpp:47).

**Contre-vérification (confirmé)** — Le thread RakNet modifie la map `servers` : MasterServer.cpp:63 (erase), :144 (erase) et :164 (insert). Le thread HTTP lancé par main.cpp:31 l'itère en même temps (RestServer.cpp:85) et y insère (RestServer.cpp:111). SimpleWeb tourne avec thread_pool_size=1 (base_server.hpp:145) : il y a donc bien deux threads concurrents. Aucun mutex n'existe dans apps/master hors SimpleWeb. On a ainsi un comportement indéfini, déclenchable par des requêtes HTTP concurrentes aux annonces.

### ECO-3

**Amont TES3MP en sommeil, fork sans divergence, correctifs de la branche 0.8.2 jamais publiés**

- **Sévérité** : 🟠 haute · **Catégorie** : architecture · **Effort** : L
- **Emplacement** : `README.md:8`

**Preuve**

> README.md:8-9 annonce « TES3MP version: 0.8.1 / OpenMW version: 0.47.0 ». `git ls-remote --symref https://github.com/TES3MP/TES3MP HEAD` → refs/heads/0.8.1 49be5b6, identique au fork. La branche 0.8.2 est figée au 2022-08-02 et 10 de ses commits sont absents de 0.8.1 (`git cat-file -e f8b411dd` → absent). Le fork n'a ni issue, ni PR, ni release.

**Impact** — Le projet repose sur OpenMW 0.47 (2021). L'amont n'a fusionné que 2 commits en 2024 et le fork n'apporte rien. Des correctifs clients de 2022 (sons d'objets, cooldowns, alliances) n'atteignent pas la branche par défaut. Personne n'est identifié pour traiter les failles réseau.

**Recommandation** — Écrire dans le README une section « État du fork » (objectif, branche suivie, mainteneur). Activer les issues et créer une release 0.8.1-x depuis le fork. Examiner les 10 commits de 0.8.2 et les cherry-picker dans 0.8.1. À moyen terme, évaluer un rebase des balises « Start of tes3mp change » sur une version récente d'OpenMW, ou se coordonner avec les forks actifs, par exemple DreamWeave, dont des correctifs sont déjà fusionnés (PR #661).

**Contre-vérification (confirmé)** — `git ls-remote` donne HEAD -> refs/heads/0.8.1 = 49be5b6, identique au commit local du fork (Merge #674, 2024-02-13). refs/heads/0.8.2 = f8b411d, daté du 2022-08-02. `git cat-file -e f8b411dd` échoue localement. Le fetch de 0.8.2 montre une dizaine de commits clients et serveur de juillet-août 2022 (ObjectSound, PlayerCooldowns, makePeace, ObjectTrap) absents de la branche par défaut. README.md:9-10 confirme la version 0.8.1 / OpenMW 0.47.0.

### ECO-4

**Dépendance réseau CrabNet non épinglée, abandonnée depuis 2020 et non compilable avec GCC 13**

- **Sévérité** : 🟠 haute · **Catégorie** : build-ci · **Effort** : S
- **Emplacement** : `CI/before_install.linux.sh:10`

**Preuve**

> CI/before_install.linux.sh:10 `git clone https://github.com/TES3MP/CrabNet` (ni tag ni commit). HEAD de CrabNet : 19e6619 du 2020-01-19. La compilation sous Ubuntu 24.04 avec g++ 13 échoue : `Source/Utils/_FindFirst.h:50:15: error: 'intptr_t' was not declared in this scope` (crabnet_make.log). Elle ne passe qu'avec `-include cstdint`.

**Impact** — La seule procédure de build documentée dans le dépôt échoue sur une distribution actuelle avant même d'atteindre TES3MP. Le build n'est pas reproductible puisqu'on clone le HEAD de la dépendance.

**Recommandation** — Ajouter CrabNet comme sous-module épinglé (extern/CrabNet), avec un correctif `#include <cstdint>` dans _FindFirst.h, ou l'intégrer via FetchContent avec un GIT_TAG fixe et `add_compile_options(-include cstdint)` en attendant le correctif. Documenter cette étape dans le README.

**Contre-vérification (confirmé)** — CI/before_install.linux.sh:10 fait `git clone https://github.com/TES3MP/CrabNet` sans tag ni commit. Le HEAD distant est 19e6619, daté du 2020-01-19. Le _FindFirst.h cloné n'inclut que <dirent.h> et <string> (lignes 11-12) mais utilise intptr_t à la ligne 50. Le journal de build déjà produit le confirme : `_FindFirst.h:50:15: error: 'intptr_t' was not declared in this scope`. Le contournement (-include cstdint) est trivial, mais la procédure documentée échoue telle quelle et n'est pas reproductible.

### ECO-10

**API Lua serveur : documentation non générée, callbacks non documentés, Doxygen configuré pour OpenMW**

- **Sévérité** : 🟡 moyenne · **Catégorie** : documentation · **Effort** : M
- **Emplacement** : `apps/openmw-mp/Script/ScriptFunctions.hpp:156`

**Preuve**

> ScriptFunctions.hpp:156 `{"OnServerInit", Callback<>()},`, comme les 67 autres callbacks, n'a aucun commentaire. 755 fonctions exportées pour 717 \brief. docs/Doxyfile.cmake:35 `PROJECT_NAME = OpenMW`, :767 `INPUT = @OpenMW_SOURCE_DIR@/apps \ components` (tout OpenMW). Aucune cible ne produit de référence Lua.

**Impact** — Les auteurs de scripts doivent lire les en-têtes C++ ou CoreScripts pour connaître les signatures et les événements. Les IDE Lua n'offrent ni autocomplétion ni vérification de types, et les régressions d'API passent inaperçues.

**Recommandation** — Ajouter docs/DoxyfileLuaApi.cmake (INPUT=apps/openmw-mp/Script/Functions, PROJECT_NAME="TES3MP Server Lua API") et une cible `make doc_lua`. Mieux : un script Python qui parse les macros *API et les \brief pour produire des stubs LuaLS (`---@meta tes3mp`) et une page Markdown. Documenter les 68 callbacks (arguments, ordre d'appel) et publier le tout via GitHub Pages.

### ECO-11

**CONTRIBUTING, CHANGELOG et CHANGELOG_PR hérités d'OpenMW, contradictoires ou à l'état de gabarit**

- **Sévérité** : 🟡 moyenne · **Catégorie** : documentation · **Effort** : S
- **Emplacement** : `CONTRIBUTING.md:1`

**Preuve**

> CONTRIBUTING.md:1 `How to contribute to OpenMW` et :6 `https://gitlab.com/OpenMW/openmw/issues`, alors que README.md:49 renvoie vers TES3MP/TES3MP/issues. CHANGELOG_PR.md:1 `*** PLEASE PUT YOUR ISSUE DESCRIPTION FOR DUMMIES HERE FOR REVIEW ***` et :13 `***add flattering adjective***`. tes3mp-changelog.md n'a pas été modifié depuis le 2022-05-01, malgré 7 commits hors fusion ultérieurs (correctif gcc-13, musique de combat, rotation).

**Impact** — Les contributeurs sont envoyés vers le mauvais projet, avec ses règles (style, wiki.openmw.org). Des fichiers à placeholders et 200 Ko de changelog OpenMW noient les informations TES3MP, et l'historique des correctifs postérieurs à 0.8.1 est introuvable.

**Recommandation** — Réécrire CONTRIBUTING.md pour TES3MP : où signaler un bogue (client, serveur ou CoreScripts), branches 0.8.x, convention de balisage « Start of tes3mp change », format des commits « [Client] / [Server] / [General] ». Supprimer CHANGELOG_PR.md et déplacer CHANGELOG.md, AUTHORS.md et docs/openmw-stage1.md vers docs/openmw-upstream/. Ajouter une section « Unreleased » à tes3mp-changelog.md.

### ECO-12

**Simple-Web vendorisé obsolète, incompatible avec Boost ≥ 1.87, et standards C++ forcés localement**

- **Sévérité** : 🟡 moyenne · **Catégorie** : build-ci · **Effort** : M
- **Emplacement** : `apps/master/SimpleWeb/base_server.hpp:199`

**Preuve**

> base_server.hpp:199 `io_service = std::make_shared<boost::asio::io_service>();`. Avec -DBOOST_ASIO_NO_DEPRECATED, qui simule la suppression d'io_service dans Boost 1.87, on obtient 22 erreurs (`'io_service' is not a member of 'boost::asio'`, `'from_string' is not a member of 'boost::asio::ip::address'`). apps/master/CMakeLists.txt:4 `add_definitions(-std=gnu++14)` et apps/browser/CMakeLists.txt:2 `set (CMAKE_CXX_STANDARD 14)` contredisent CMAKE_CXX_STANDARD 17 à la racine.

**Impact** — Le serveur maître ne compilera plus avec les distributions qui livrent Boost ≥ 1.87. Le code est daté de 2016 et n'est jamais testé en CI, puisque BUILD_MASTER vaut OFF. Les -std locaux peuvent casser l'inclusion d'en-têtes de components écrits en C++17.

**Recommandation** — Remplacer SimpleWeb par une version récente d'eidheim/Simple-Web-Server (io_context, make_address), ou par cpp-httplib (en-tête unique) ou Boost.Beast. Supprimer `add_definitions(-std=gnu++14)` et `CMAKE_CXX_STANDARD 14` pour hériter de C++17, puis activer BUILD_MASTER dans une tâche de CI.

### ECO-13

**Gouvernance GitHub inexistante : CI morte, ni workflows, ni modèles, ni politique de sécurité**

- **Sévérité** : 🟡 moyenne · **Catégorie** : build-ci · **Effort** : M
- **Emplacement** : `.travis.yml:35`

**Preuve**

> `ls .github` → FUNDING.yml uniquement. .travis.yml:35 `name: "TES3MP/openmw-tes3mp"` et :38 `notification_email: koncord@tes3mp.com` (Travis-ci.org est fermé, ancien nom du dépôt). Aucun ISSUE_TEMPLATE, PULL_REQUEST_TEMPLATE, SECURITY.md ni CODE_OF_CONDUCT.md (find → vide).

**Impact** — Aucune PR n'est compilée automatiquement : la casse GCC 13 n'a été corrigée qu'en 2024, par un contributeur externe (7921f71). Il n'existe aucun canal privé pour signaler une faille réseau sur un projet exposé à Internet, et les rapports de bogue arrivent sans version client, serveur ou CoreScripts.

**Recommandation** — Ajouter .github/workflows/build.yml (ubuntu-24.04 et windows-2022, CrabNet épinglé, -DBUILD_MASTER=ON, ctest). Ajouter .github/ISSUE_TEMPLATE/bug.yml avec les champs versions client, serveur et CoreScripts et les logs, un PULL_REQUEST_TEMPLATE.md et SECURITY.md (contact privé, délai de réponse). Supprimer .travis.yml et appveyor.yml s'ils ne sont plus utilisés.

### ECO-5

**Aucune documentation propre à TES3MP dans le dépôt : la documentation Sphinx est celle d'OpenMW**

- **Sévérité** : 🟡 moyenne (initialement haute) · **Catégorie** : documentation · **Effort** : L
- **Emplacement** : `docs/source/conf.py:55`

**Preuve**

> docs/source/conf.py:55 `project = u'OpenMW'`. `grep -rli tes3mp docs manual` → 0 fichier sur 74. Le README (49 lignes) n'a aucune section build ni hébergement et renvoie au wiki, où Home.md fait 0 ligne et Building-on-Microsoft-Windows.md se réduit à un lien vers une discussion Steam (dernière modification le 2022-09-28).

**Impact** — Un nouvel arrivant ne trouve dans le dépôt ni comment compiler tes3mp et tes3mp-server, ni comment héberger un serveur (ports, CoreScripts dans ./server, pare-feu), ni comment écrire un script Lua. Toute la connaissance est sur Discord, Steam ou un wiki externe non versionné.

**Recommandation** — Créer docs/tes3mp/ avec trois pages : 1) build Linux et Windows (CrabNet épinglé, LuaJIT, options CMake BUILD_OPENMW_MP, BUILD_BROWSER et BUILD_MASTER) ; 2) hébergement (UDP 25565, tes3mp-server.cfg, clonage de CoreScripts, unité systemd, sauvegarde de server/data) ; 3) premiers pas en Lua (customEventHooks, lien vers CoreScripts/Tutorial.md). Les référencer dans docs/source/index.rst et ajouter une section « Building » au README.

**Contre-vérification (partiel (sévérité corrigée))** — Vérifié : docs/source/conf.py:55 contient `project = u'OpenMW'`, et `grep -rli tes3mp docs manual` donne 0 fichier sur 74. CONTRIBUTING.md commence par « How to contribute to OpenMW ». Le README n'a aucune section build ni hébergement. En revanche, l'affirmation sur les scripts Lua est exagérée : l'API serveur est documentée par 717 commentaires Doxygen \brief/\param dans apps/openmw-mp/Script/Functions/*.hpp (par exemple Chat.hpp:17-26), et docs/Doxyfile.cmake:767 inclut apps/ en INPUT. Ce qui manque vraiment, ce sont la documentation de build et celle d'hébergement.

### ECO-6

**Arrêt du serveur maître : gestionnaire de signal non async-signal-safe, le processus se termine par std::terminate**

- **Sévérité** : 🟡 moyenne · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/master/main.cpp:19`

**Preuve**

> main.cpp:19-24 : le lambda exécuté sur SIGINT/SIGTERM appelle `restServer->stop(); masterServer->Stop(false); masterServer->Wait();`. run.log (SIGTERM) : `terminate called after throwing an instance of 'boost::wrapexcept<boost::system::system_error>' what(): set_option: Bad file descriptor`. run2.log (SIGINT) : `terminate called without an active exception`. TSan : « signal handler spoils errno main.cpp:19 ».

**Impact** — Chaque arrêt, qu'il s'agisse d'un `systemctl stop` ou d'un Ctrl+C, se termine en abort : code de sortie non nul, core dump possible, redémarrages en boucle sous un superviseur. L'appel à join() depuis le handler peut aussi provoquer un interblocage.

**Recommandation** — Remplacer signal() par boost::asio::signal_set sur l'io_service de Simple-Web, ou limiter le handler à l'écriture d'un std::atomic<bool>. Effectuer les appels stop() et join() depuis main() après server_thread.join(), et vérifier joinable() avant de détruire std::thread (main.cpp:31-34).

**Contre-vérification (confirmé)** — main.cpp:19-24 appelle depuis un handler de signal des fonctions non async-signal-safe : stop() d'asio, join. L'abort est même garanti par la logique du code. Stop(false) met run à false (MasterServer.cpp:209), donc Wait() (ligne 222 `if (run)`) ne joint jamais tMasterThread. main.cpp:34 ne joint pas davantage, et le destructeur statique ~MasterServer -> Stop(true) ne fait rien non plus. Le std::thread est donc encore joinable à sa destruction, d'où std::terminate (« terminate called without an active exception »). Le risque d'interblocage par join est en pratique inexistant, puisque join n'est jamais atteint.

### ECO-9

**Navigateur : courses de données dans le ping, fuite de RakPeer, boucle active et ping bloquant dans le thread GUI**

- **Sévérité** : 🟡 moyenne · **Catégorie** : bug · **Effort** : M
- **Emplacement** : `apps/browser/PingUpdater.cpp:35`

**Preuve**

> PingHelper.cpp:8 `pingUpdater->addServer(row, addrPair);` est un appel direct depuis le thread GUI, qui fait push_back (PingUpdater.cpp:15) pendant que process() fait `servers.back(); servers.pop_back();` (lignes 35-36) dans pingThread, sans verrou. netutils/Utils.cpp:22-23 `if (!peer->Ping(...)) return time;` retourne sans DestroyInstance, et la boucle des lignes 25-33 (`if (!packet) continue;`) tourne à vide jusqu'à 999 ms. ServerInfoDialog.cpp:68 appelle PingRakNetServer dans le thread GUI, et les lignes 48-49 `if (!refreshThread->isRunning()) refreshThread->terminate();` ont une logique inversée.

**Impact** — Le navigateur peut planter ou corrompre son QVector lors d'un rafraîchissement pendant les pings, consomme 100 % d'un cœur par serveur pingé, fuit un RakPeer (socket et thread) par serveur injoignable et gèle l'interface jusqu'à environ 1 s à l'ouverture d'une fiche.

**Recommandation** — Passer les serveurs à PingUpdater par signal en Qt::QueuedConnection, ou protéger `servers` par un QMutex, et rendre `run` std::atomic<bool>. Appeler DestroyInstance sur tous les chemins (RAII via unique_ptr avec suppresseur personnalisé), ajouter RakSleep(5) dans la boucle d'attente et déplacer le ping de ServerInfoDialog dans le worker. Corriger le destructeur : `if (refreshThread->isRunning()) { refreshThread->quit(); refreshThread->wait(); }` suivi de delete worker et delete refreshThread.

**Contre-vérification (confirmé)** — PingHelper.cpp:8 appelle addServer directement depuis le thread GUI, qui fait push_back (PingUpdater.cpp:15). Pendant ce temps, process() exécute back()/pop_back() (lignes 35-36) dans pingThread, après le moveToThread de PingHelper.cpp:44, sans verrou. `emit pingUpdater->stop()` (PingHelper.cpp:21) appelle aussi le slot directement, ce qui fait clear() en concurrence. Dans netutils/Utils.cpp, la ligne 22-23 retourne sans DestroyInstance, et la boucle des lignes 25-33 tourne à vide sur Receive() sans sommeil. ServerInfoDialog.cpp:68 pinge dans le thread GUI, et les lignes 48-49 ont une condition inversée.

### ECO-14

**Navigateur : fonctionnalité Favoris désactivée, code mort compilé, mot de passe passé en ligne de commande**

- **Sévérité** : ⚪ basse · **Catégorie** : maintenabilite · **Effort** : S
- **Emplacement** : `apps/browser/MainWindow.cpp:30`

**Preuve**

> MainWindow.cpp:30-31 `// Remove Favorites tab while it remains broken` / `tabWidget->removeTab(1);`. netutils/HTTPNetwork.cpp (95 lignes) et getExtendedData (Utils.cpp:61) sont compilés (CMakeLists.txt du navigateur) mais jamais appelés (grep → 0 usage). MainWindow.cpp:143 `arguments.append(QLatin1String("--password=") + passw...)`.

**Impact** — Du code mort maintenu pour rien et une fonctionnalité visible dans le .ui mais inaccessible. Le mot de passe du serveur apparaît dans la liste des processus (ps, Gestionnaire des tâches) pour tout utilisateur local.

**Recommandation** — Retirer HTTPNetwork.* et getExtendedData du CMake, puis réparer les favoris (persistance favorites.dat déjà présente, il manque l'appel QueryClient::Update dans addServerAndUpdate) ou supprimer l'onglet de Main.ui. Transmettre le mot de passe par variable d'environnement ou stdin au lieu d'argv.

### ECO-15

**Chaîne de documentation cassée : .readthedocs.yaml obsolète et Sphinx 1.8.5 inutilisable**

- **Sévérité** : ⚪ basse · **Catégorie** : documentation · **Effort** : S
- **Emplacement** : `.readthedocs.yaml:6`

**Preuve**

> .readthedocs.yaml:6-7 `python:` / `version: 3.8`, sans `build.os`. docs/requirements.txt fixe `sphinx==1.8.5`. Dans un venv Python 3.13 : `sphinx-build` → `ModuleNotFoundError: No module named 'pkg_resources'`. Le refus par Read the Docs d'une configuration sans build.os n'a pas pu être vérifié en ligne.

**Impact** — Impossible de générer la documentation localement avec un Python récent, et la publication Read the Docs est très probablement en échec. Ce sont de toute façon les pages d'OpenMW qui seraient publiées.

**Recommandation** — Mettre à jour .readthedocs.yaml (`build: os: ubuntu-24.04, tools: python: "3.12"`) et requirements.txt (sphinx>=7, sphinx-rtd-theme, docutils récent), et passer project='TES3MP' dans conf.py. Si aucune documentation TES3MP n'est publiée, supprimer ces fichiers.

### ECO-16

**Client configuré en journalisation verbeuse (« spam ») par défaut**

- **Sévérité** : ⚪ basse · **Catégorie** : performance · **Effort** : S
- **Emplacement** : `files/tes3mp/tes3mp-client-default.cfg:6`

**Preuve**

> tes3mp-client-default.cfg:5-6 `# 0 - Verbose (spam), 1 - Info...` / `logLevel = 0`, appliqué par apps/openmw/mwmp/Main.cpp:134-135 `TimedLog::SetLevel(logLevel);`. On compte 133 appels LOG_VERBOSE dans apps/openmw/mwmp.

**Impact** — Chaque joueur écrit des journaux volumineux (paquets par frame), ce qui coûte en E/S disque et en CPU pendant le jeu et noie les erreurs utiles dans les rapports de bogue.

**Recommandation** — Passer `logLevel = 1` (Info) par défaut, comme côté serveur (tes3mp-server-default.cfg:9), et documenter comment activer le niveau 0 pour le débogage.

### ECO-17

**README : liens figés sur la branche master en retard, copyright daté, aucune mention du fork**

- **Sévérité** : ⚪ basse · **Catégorie** : documentation · **Effort** : S
- **Emplacement** : `README.md:19`

**Preuve**

> README.md:19 `https://github.com/TES3MP/TES3MP/blob/master/tes3mp-changelog.md` (de même aux lignes 11, 14 et 39). La branche par défaut est 0.8.1 (49be5b6), alors que master pointe sur 00997fa (2023-10), en retard. README.md:5 `Copyright (c) 2016-2022`. Les 27 liens externes (Discord, forum, wiki.openmw.org, Steam) n'ont pas pu être vérifiés depuis la machine d'audit.

**Impact** — Les lecteurs du fork sont envoyés vers l'amont, sur une branche qui n'est pas celle construite. Des liens communautaires potentiellement morts ne sont détectés par aucun outil.

**Recommandation** — Utiliser des liens relatifs ([LICENSE](LICENSE), [tes3mp-changelog.md](tes3mp-changelog.md)), ajouter une section « À propos de ce fork » et un job lychee ou markdown-link-check hebdomadaire dans GitHub Actions.

### ECO-18

**.gitignore et .editorconfig hérités, ne couvrant pas les artefacts TES3MP**

- **Sévérité** : ⚪ basse · **Catégorie** : maintenabilite · **Effort** : S
- **Emplacement** : `.gitignore:46`

**Preuve**

> .gitignore:46-47 ignore `/openmw` et `/opencs`, mais `grep -n tes3mp .gitignore` → aucun résultat, alors que les cibles s'appellent tes3mp, tes3mp-server, tes3mp-browser et masterserver. .editorconfig ne définit que [*.cpp], [*.hpp] et [*.glsl] (pas de Lua, CMake ni .ui), et il n'y a pas de .clang-format.

**Impact** — Une compilation dans l'arborescence laisse des binaires suivables par git. Le style des contributions Lua, CMake et C++ n'est pas normalisé.

**Recommandation** — Ajouter /tes3mp, /tes3mp-server, /tes3mp-browser, /masterserver et /server/ à .gitignore. Étendre .editorconfig ([*.lua], [CMakeLists.txt], [*.cmake]) et ajouter un .clang-format calqué sur le style existant (indentation de 4 espaces, accolades Allman).

### ECO-7

**Configuration serveur par défaut : publication automatique sur le maître public, sans mot de passe**

- **Sévérité** : ⚪ basse (initialement moyenne) · **Catégorie** : securite · **Effort** : S
- **Emplacement** : `files/tes3mp/tes3mp-server-default.cfg:17`

**Preuve**

> tes3mp-server-default.cfg:10 `password =`, :17 `enabled = true`, :18 `address = master.tes3mp.com`. apps/openmw-mp/main.cpp:271 `if (mgr.getBool("enabled", "MasterServer"))` déclenche l'annonce. Networking.cpp:84-89 : un mot de passe vide est remplacé par la sentinelle "blankpassword", et un serveur dont le mot de passe vaut littéralement "blankpassword" est considéré comme ouvert. :13 `home = ./server` est relatif au répertoire courant.

**Impact** — Un serveur de test lancé tel quel apparaît publiquement (IP, nom « TES3MP server ») et est ouvert à tous, sans que l'hébergeur en soit averti. Le chemin relatif des scripts fait échouer le démarrage si le binaire est lancé depuis un autre répertoire (service systemd).

**Recommandation** — Passer `enabled = false` par défaut et commenter la section [MasterServer]. Journaliser un avertissement LOG_WARN au démarrage d'un serveur à la fois public et sans mot de passe. Résoudre `home` relativement au fichier de configuration ou à l'exécutable. Remplacer la sentinelle par un booléen hasPassword.

**Contre-vérification (partiel (sévérité corrigée))** — Les valeurs sont exactes : tes3mp-server-default.cfg:10 `password =`, :13 `home = ./server`, :17-18 `enabled = true` / master.tes3mp.com, et Networking.cpp:84-91 avec la sentinelle « blankpassword ». En revanche, l'hébergeur n'est pas laissé sans avertissement : main.cpp journalise « Sharing server query info to master enabled. ». Un serveur de jeu ouvert et listé par défaut est aussi un choix courant et assumé. Le chemin relatif (main.cpp:197, pluginHome utilisé tel quel) est un vrai piège pour systemd, mais il reste de faible gravité.

### ECO-8

**Ports du serveur maître codés en dur et incohérents avec les configurations livrées**

- **Sévérité** : ⚪ basse (initialement moyenne) · **Catégorie** : bug · **Effort** : S
- **Emplacement** : `apps/master/main.cpp:16`

**Preuve**

> apps/master/main.cpp:16 `new MasterServer(2000, 25560)` et :17 `new RestServer(8080, ...)`. tes3mp-client-default.cfg:10 et tes3mp-server-default.cfg:19 contiennent `port = 25561`. apps/browser/main.cpp:41-42 réécrit 25560 en 25561 « for the currently used fork of RakNet ». ServerTest.cpp:21 vise 25560.

**Impact** — Une personne qui héberge son propre maître à partir de ce dépôt ne peut pas le joindre avec les configurations par défaut. Le code publié ne correspond manifestement pas à celui du maître officiel (25561), et rien ne permet de changer le port sans recompiler.

**Recommandation** — Ajouter boost::program_options, déjà une dépendance du projet, ou un fichier tes3mp-master.cfg lu via Settings::Manager, avec --port (25561 par défaut), --http-port, --max-connections et --ttl. Aligner ServerTest.cpp sur ces valeurs et documenter le déploiement du maître.

**Contre-vérification (partiel (sévérité corrigée))** — Confirmé dans le code : apps/master/main.cpp:16-17 code en dur 25560 et 8080, alors que les cfg livrés utilisent 25561. La redirection 25560 -> 25561 (browser/main.cpp:41-42, openmw-mp/main.cpp) ne vaut que pour master.tes3mp.com. L'impact est surestimé : pour héberger son propre maître, il faut de toute façon modifier `address` dans les cfg, et donc aussi le port. Le vrai problème est l'absence de configuration des ports sans recompiler, ce qui relève d'une sévérité basse.

