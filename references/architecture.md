# Architecture d'un jeu Roblox

À lire quand : tu crées un nouveau système, tu choisis où placer un script, tu organises un projet Rojo ou roblox-ts, tu hésites sur un framework, ou tu débogues un problème d'ordre de chargement, de réplication, de respawn ou de fuite. La validation des remotes et l'anti-triche sont dans securite.md.

## Sommaire

1. Le modèle client/serveur
2. Où va chaque script
3. ModuleScripts
4. Bootstrap en deux phases
5. Systèmes, contrôleurs et contrats
6. Frameworks : quand s'en servir, quand s'en passer
7. Réplication
8. Cycle de vie du joueur et du personnage
9. Nettoyage
10. CollectionService et attributs
11. Boucles, manches et temps
12. Organiser un projet Rojo ou roblox-ts
13. Server Authority
14. Pièges
15. Checklist

---

## 1. Le modèle client/serveur

- **Un serveur par partie.** Il fait autorité sur l'état, voit tout le DataModel, et lui seul touche aux DataStores, aux achats et à `ServerStorage`. Chaque client rend sa vue, lit les entrées et prédit. Le client est hostile par principe (securite.md).
- **Réplication filtrée.** Ce que le serveur crée ou modifie dans un conteneur répliqué arrive chez les clients. Ce qu'un client crée ou modifie reste chez lui. Exceptions : la physique des assemblages qu'il possède (dont son personnage), les animations jouées sur son personnage, et ses appels de remotes.
- **Répartition.** Serveur : vérité, validation, sauvegarde, récompenses, dégâts, achats, IA des PNJ. Client : entrées, caméra, UI, effets, prédiction. Partagé : types, constantes, fonctions pures, noms de remotes.
- **Intention, pas résultat.** Le client envoie `CollectCoin(coinId)`, jamais `AddCoins(50)`. La prédiction client est permise si le serveur arrive seul à la même conclusion et peut corriger.
- **Un propriétaire par fait.** Chaque donnée a un seul écrivain. Les autres copies (leaderstats, attributs, textes d'UI, caches client) sont des vues, réécrites après le changement autoritaire. Test : « si ces deux copies divergeaient maintenant, laquelle aurait raison ? ». Si deux modules écrivent le même champ, l'un le possède et l'autre appelle sa fonction.
- **Un propriétaire par effet de bord.** Charger, sauver et libérer un profil, traiter les reçus, muter la monnaie ou l'inventaire, dédoublonner les messages inter-serveurs, appeler un webhook : un seul module pour chacun. Les autres fonctionnalités lui demandent. Pourquoi : deux boucles de sauvegarde ou deux callbacks de reçu provoquent des écrasements et des doubles octrois.
- **Avant de coder un système**, réponds : quel état fait autorité ? Qui peut le modifier ? Quelles opérations publiques ? Quelles ressources possède-t-il (connexions, tâches, instances) ? Traverse-t-il un remote, une sauvegarde, un achat ? Comment démarre-t-il et s'arrête-t-il ? Que se passe-t-il si le joueur part en pleine action ? Comment le tester à la main ?

---

## 2. Où va chaque script

| Conteneur | Vu par le client | Ce qui s'y exécute | À y mettre |
|---|---|---|---|
| `ServerScriptService` | Non | Scripts serveur, modules requis par eux | Toute la logique serveur, le bootstrap serveur |
| `ServerStorage` | Non | Rien ; un script serveur peut y requérir des modules | Modèles à cloner, cartes, tables secrètes (poids de loot, taux) |
| `ReplicatedStorage` | Oui, lisible | Modules requis des deux côtés ; un `Script` seulement si `RunContext` vaut `Client` ou `Server` | Types, constantes, fonctions pures, remotes, assets partagés, paquets |
| `ReplicatedFirst` | Oui, en premier | LocalScripts dès l'arrivée | Écran de chargement, rien d'autre |
| `StarterPlayerScripts` | Copié dans `PlayerScripts` | LocalScripts, une fois par session | Bootstrap client, contrôleurs persistants |
| `StarterCharacterScripts` | Copié dans chaque personnage | LocalScripts, à chaque apparition | Comportement lié au personnage |
| `StarterGui` | Copié dans `PlayerGui` à chaque apparition | LocalScripts d'UI | Templates d'UI (`ResetOnSpawn = false` pour survivre) |
| `StarterPack` | Copié dans le `Backpack` | Scripts d'outils | Tools |
| `Workspace` | Oui, selon le streaming | Scripts serveur hérités (à éviter) | Monde, cartes, objets taggés |

- **L'emplacement fait partie du comportement.** Un `LocalScript` dans `Workspace` ne tourne pas, sauf dans le personnage du joueur local. Un `Script` en `RunContext = Legacy` ne tourne ni dans `ServerStorage` ni dans `ReplicatedStorage`.
- **`Script.RunContext`** vaut `Legacy` (dépend de l'emplacement), `Server` ou `Client` (`Plugin` est réservé aux plugins). Un Script `Client` tourne depuis `ReplicatedStorage` ou `ReplicatedFirst` ; dans `StarterPlayerScripts` ou `StarterCharacterScripts`, l'original et sa copie tournent tous les deux. Il se règle dans Propriétés ou par Rojo ; le changer à l'exécution relance le script dans le nouveau contexte, évite-le. `LocalScript` n'est pas déprécié : suis la convention du projet.
- **Tout conteneur répliqué est décompilable**, y compris un script désactivé ou un module jamais requis. Pas de logique serveur « par commodité » dans `ReplicatedStorage`. Évite les branches `if RunService:IsServer()` dans un module partagé : la branche serveur reste lisible.
- **Faits mesurés** (Studio 0.742, 2026-10) : `ReplicatedFirst` ne se réplique qu'une fois, donc un objet que le serveur y ajoute après l'arrivée d'un joueur ne lui parvient jamais. Le serveur lit `StarterPlayerScripts` mais pas la copie `PlayerScripts` d'un joueur. Les `Camera` ne se répliquent jamais : crée-les côté client, y compris celle d'un ViewportFrame.
- **Ordre d'exécution.** Côté client, `ReplicatedFirst` passe en premier. Les Scripts serveur démarrent avec le serveur, `StarterPlayerScripts` à l'arrivée du joueur, `StarterGui` après la copie dans `PlayerGui`, `StarterCharacterScripts` à chaque apparition, un module à son premier `require`. Aucun ordre garanti entre deux scripts frères : un seul point d'entrée par côté (section 4).
- **Serveur : tout existe au démarrage** ; ce qui est placé dans Studio ou synchronisé par Rojo s'indexe directement. Côté client, tout arrive par réplication : `WaitForChild`. Sans timeout, il bloque pour toujours si l'objet ne vient pas ; « Infinite yield possible » apparaît après 5 s, lis-le en premier quand rien ne marche. Ce n'est qu'un avertissement : l'objet peut encore arriver (premier spawn lent). Dépendance essentielle : timeout et `error` qui nomme le chemin. Contenu optionnel ou streamé : timeout et repli.
- **Services** : `game:GetService("Nom")` une fois en tête de fichier, dans une locale du même nom. Jamais `game.Nom`, jamais dans une boucle chaude. `RunService:IsStudio()` dit l'environnement, pas le côté : pour le côté, `IsServer()` et `IsClient()`.

Placement rapide : visible en 3D → `Workspace`. Logique serveur → `ServerScriptService`. Asset ou donnée réservés au serveur → `ServerStorage`. Utile aux deux côtés et sans secret → `ReplicatedStorage`. Côté client : UI → `StarterGui` ou montée par React depuis `StarterPlayerScripts` (ui-react.md) ; avant tout le reste → `ReplicatedFirst` ; lié au personnage → `StarterCharacterScripts` ; sinon → `StarterPlayerScripts`.

---

## 3. ModuleScripts

- **Un module s'exécute une fois par environnement**, puis `require` renvoie la même valeur. Le serveur et chaque client ont leur propre copie. C'est le singleton : pas de `_G` ni de `shared`, qui cachent les dépendances et cassent le typage. Renvoie une seule valeur non nil, en général une table. `--!strict` en tête, `export type` pour les types partagés, tous les `require` en haut.
- **Aucun yield dans le corps d'un module** (`WaitForChild`, `task.wait`, HTTP, DataStore). Il bloque tous les scripts qui le requièrent, et un module qui attend un objet jamais créé fige le démarrage sans erreur. Le travail qui attend va dans `start` (section 4).
- **Pas de `require` circulaire** : Luau lève une erreur, et le cycle révèle une possession floue. Remèdes, du meilleur au pire : extraire la partie commune dans un 3e module ; passer la dépendance à `init` ; notifier par un signal à sens unique ; `require` paresseux dans une fonction, en dernier recours.
- **Configuration** : un module de constantes gelé par `table.freeze` (prix, cooldowns, tables d'apparition) pour équilibrer à un seul endroit. Il est lisible par le client : rien de secret. Les tables secrètes vont dans un module serveur, avec une table d'affichage séparée dans `ReplicatedStorage` si l'UI en a besoin. Pas d'`IntValue` ni de `StringValue` pour l'état ou la config : des attributs pour les réglages par instance (section 10).
- **Logique pure à part.** Formules de dégâts, économie, opérations d'inventaire, machines à états : dans des modules sans Instance ni service, testables sous Lune ou Jest-Lua (outils.md). Les accès au moteur restent dans de fins modules de bord.
- **Modules profonds.** Beaucoup de comportement derrière une petite interface. Test de suppression : si supprimer le module fait disparaître la complexité, c'était un passe-plat, intègre-le à l'appelant ; si elle réapparaît chez plusieurs appelants, garde-le. Une interface à une seule implémentation ne sert à rien. Nomme avec le vocabulaire du jeu (`CoinEconomy`, `QuestTracker`), pas `Manager` ni `Handler`. Un script de 300 à 500 lignes qui mêle combat, données et UI se découpe ; une petite fonction utilisée par un seul script y reste jusqu'à son deuxième usage.
- `require(assetId)` est interdit : c'est fragile et c'est la signature des portes dérobées (securite.md). Requiers par référence.

---

## 4. Bootstrap en deux phases

Un seul `Script` serveur et un seul `LocalScript` client. Tout le reste est en ModuleScripts. Pourquoi : des scripts multiples n'ont aucun ordre garanti et dupliquent les recherches de services.

- **`init`** : synchrone, sans yield. Lit la config, câble les dépendances, crée les remotes, prépare l'état.
- **`start`** : connecte les signaux, lance les boucles. Il rend la main vite ; une boucle ou une attente part dans un `task.spawn` que le système possède.
- **Ordre explicite** : une liste dans le bootstrap ; l'ordre de `GetChildren` ou d'un dictionnaire n'est pas un contrat. **Échec visible** : une erreur dans `init` arrête le démarrage. N'enveloppe pas chaque `start` dans `task.spawn` : l'ordre et les erreurs se perdent.
- Les opérations publiques refusent de servir avant `init`, et un second `init` est une erreur de bootstrap. Préfixe les logs par `[NomDuSystème]` pour suivre l'amorçage dans Output.
- Si le projet a déjà un framework à cycle de vie (Flamework, Knit), suis-le : c'est la discipline des phases qui compte.

```luau
--!strict
-- src/server/init.server.luau → seul Script serveur (ServerScriptService.Server)
type System = { init: (() -> ())?, start: (() -> ())? }

local Systems = script.Systems
-- Ordre explicite : pendant init, un système n'utilise que ceux listés avant lui.
local ORDER: { System } = {
	require(Systems.Cooldowns),
	require(Systems.PlayerData),
	require(Systems.Shop),
}

-- Phase 1 : câbler et créer les remotes. Aucun yield ; une erreur arrête tout.
for _, system in ORDER do
	if system.init then
		system.init()
	end
end

-- Phase 2 : connecter les signaux et lancer les boucles.
for _, system in ORDER do
	if system.start then
		system.start()
	end
end
```

Un système complet, avec son garde d'initialisation et le nettoyage par joueur :

```luau
--!strict
-- src/server/Systems/Cooldowns.luau : possède les cooldowns d'actions des joueurs
local Players = game:GetService("Players")

local Cooldowns = {}
local ready = false
local lastUse: { [Player]: { [string]: number } } = {}

-- Vrai si l'action est permise maintenant ; la consomme dans ce cas.
function Cooldowns.try(player: Player, action: string, seconds: number): boolean
	assert(ready, "[Cooldowns] utilisé avant init")
	local perPlayer = lastUse[player]
	local now = os.clock()
	if perPlayer == nil or now - (perPlayer[action] or -math.huge) < seconds then
		return false -- joueur absent, ou cooldown en cours
	end
	perPlayer[action] = now
	return true
end

function Cooldowns.init()
	assert(not ready, "[Cooldowns] init appelé deux fois")
	ready = true
end

function Cooldowns.start()
	Players.PlayerAdded:Connect(function(player)
		lastUse[player] = {}
	end)
	for _, player in Players:GetPlayers() do
		lastUse[player] = lastUse[player] or {} -- arrivés avant ce script (fréquent en Studio)
	end
	Players.PlayerRemoving:Connect(function(player)
		lastUse[player] = nil
	end)
end

return Cooldowns
```

Côté client, même forme : `src/client/init.client.luau` et un dossier `Controllers/`. En roblox-ts, même liste dans `src/server/main.server.ts`. Si chaque système exporte des fonctions (`import * as Cooldowns from "./Systems/Cooldowns"`), écris `interface System { init?: () => void; start?: () => void }`, puis `const ORDER: Array<System> = [Cooldowns, Shop];`, `for (const system of ORDER) system.init?.();` et la même boucle pour `start`. Pour des objets à méthodes, écris `init?(): void` : roblox-ts refuse de mélanger méthodes (appel `:`) et fonctions (appel `.`).

---

## 5. Systèmes, contrôleurs et contrats

- **Un système de gameplay est une tranche verticale**, jamais un fichier seul :
  1. types partagés et noms de remotes (`shared/Types.luau`, `shared/RemoteNames.luau`) ;
  2. module serveur (`server/Systems/Shop.luau`) qui possède tout l'état, dans une table indexée par `Player` et vidée au départ ;
  3. contrôleur client ou composants React qui affichent et demandent, sans rien décider ;
  4. logique pure extraite (`shared/ShopLogic.luau`) et son test ;
  5. une API serveur (`Shop.grant(player, itemId)`) réutilisée par les autres systèmes et par les commandes de triche QA protégées (securite.md).
- Monte d'abord un système exemplaire complet, par exemple la boutique, puis fais copier sa forme aux suivants. Si une fonctionnalité exige de toucher 5 fichiers sans rapport, les frontières sont fausses.
- **Contrat de module**, en 3 à 6 lignes en tête du fichier : état possédé, opérations publiques et côté appelant, ce qui peut yield, modes d'échec, démarrage et arrêt.
- **Appel direct par défaut** : visible, typé, cherchable. Expose un signal sur le module (`Shop.ItemPurchased`) seulement quand plusieurs observateurs indépendants (ou aucun) sont légitimes et que l'émetteur ne doit pas les connaître ; ce signal a un propriétaire qui le nettoie. Pas de bus d'événements global à noms en chaînes.
- **Bindables** (`BindableEvent`, `BindableFunction`) : même côté seulement, jamais une frontière de sécurité. Les arguments sont copiés comme pour un remote (métatables retirées, tables cycliques refusées). Quand `Fire` rend la main, ne suppose pas que les écouteurs ont fini : en signaux différés, ils n'ont pas commencé ; en mode immédiat, chacun tourne jusqu'à son premier yield. `BindableFunction:Invoke` sans `OnInvoke` attend pour toujours. Préfère un appel de module ou une lib de signaux (paquets.md).
- **Handshake « client prêt ».** Le client connecte tous ses écouteurs de remotes, puis tire `ClientReady`. Le serveur retient les données du joueur jusqu'à ce signal, envoie un snapshot complet, puis des deltas. Pourquoi : ni un client lent ni un profil long à charger ne perdent d'événement. Un système qui dépend des données du joueur attend le chargement de son profil (donnees.md).
- **Réutilise avant de créer.** Avant un remote, un service ou un module, cherche le propriétaire existant et lis son code, pas seulement son nom. Deux systèmes concurrents doublent les entrées, les sauvegardes ou les octrois. Supprime l'ancien chemin quand tu le remplaces. Dans un projet travaillé par des agents, tiens un `docs/architecture.md` qui dit qui appelle quoi : des agents qui « dédupliquaient » ont déjà supprimé la moitié d'une fonctionnalité. Pour relire une architecture, suis le vrai flux (entrée → retour visuel → remote ou simulation → état autoritaire → sauvegarde → nettoyage) : deux fichiers qui coexistent ne prouvent pas qu'ils sont reliés à l'exécution.

---

## 6. Frameworks : quand s'en servir, quand s'en passer

| Situation | Choix |
|---|---|
| Petit ou moyen jeu, en Luau ou en roblox-ts | Modules et bootstrap en deux phases, sans framework |
| roblox-ts, besoin d'injection de dépendances, de composants par tag et d'un réseau typé | Flamework |
| Projet existant en Knit | Le garder et l'adapter. Knit n'est plus maintenu (dépôt archivé) : jamais pour un nouveau projet |
| Centaines d'entités mises à jour à chaque frame (projectiles, essaims, unités) | ECS (jecs), seulement sur preuve |
| Nettoyage | La lib du projet (Trove, Maid, Janitor), sinon le petit `Scope` de la section 9 |
| Réseau à très fort débit | Une seule lib de transport par projet (ByteNet, Zap…), sinon des remotes bruts |

Les identifiants d'installation sont dans paquets.md.

- Une bibliothèque impose ses idiomes là où elle possède le sujet. La validation, la limitation de débit et l'autorité ne se délèguent jamais à elle.
- N'introduis pas de framework dans un projet qui n'en a pas et ne migre rien de toi-même : recommande, n'impose pas. Face à une bibliothèque inconnue ou maison, lis 2 ou 3 de ses usages dans le projet avant d'écrire contre elle.

### Flamework en bref (roblox-ts)

- `@Service()` est un singleton serveur, `@Controller()` un singleton client, tous deux importés de `@flamework/core` avec les interfaces `OnInit` et `OnStart`. Les dépendances passent par le constructeur (`constructor(private readonly economy: EconomyService) {}`) et fixent l'ordre de chargement. `Dependency<T>()` est réservé au code hors classes, après `ignite`.
- Cycle de vie : `onInit` (un par un, bloquant, sans yield), puis `onStart` (chacun dans son thread, l'endroit par défaut pour se connecter), `onTick(dt)` sur Heartbeat, `onPhysics(dt, time)` avant la physique, `onRender(dt)` côté client.
- Points d'entrée : `runtime.server.ts` et `runtime.client.ts` appellent `Flamework.addPaths(...)` puis `Flamework.ignite()`.
- Composants : `@Component({ tag: "Door" })` sur une classe qui étend `BaseComponent<Attributs, Model>`, attachée à chaque instance taggée. Par défaut, les instances rangées dans `ServerStorage`, `ReplicatedStorage`, `StarterGui`, `StarterPack` et `StarterPlayer` sont ignorées : un prefab en stockage ne déclenche rien.
- Réseau : `Networking.createEvent<ClientToServerEvents, ServerToClientEvents>()` dans `shared/`, puis `createServer` et `createClient` dans deux fichiers séparés, pour que la config serveur ne parte pas chez le client. Les gardes générées vérifient les types, pas le sens : un montant négatif, une cible à 500 studs ou un objet non possédé passent. La validation métier reste à écrire (securite.md).

### ECS : seulement sur preuve

- Critères : des centaines d'entités similaires mises à jour à chaque frame, un état transversal interrogé par des systèmes sans rapport, ou une équipe déjà à l'aise. Une boutique, des quêtes ou dix ennemis ne passent pas la barre. La plupart des jeux publiés tournent avec des tables, des tags, des attributs et une boucle par système.
- Pièges de jecs (v0.11.0, 2026-03) : une query simple ne s'itère qu'une fois ; `:with()` et `:without()` remplacent le filtre au lieu de s'additionner ; un changement structurel dans un hook de retrait lève une erreur, diffère-le ; ajouter ou retirer un composant coûte cher, modifier un champ non. Matter est peu actif depuis fin 2024 (paquets.md). Pour répliquer, ne réutilise jamais l'ID d'entité serveur côté client : table de correspondance, snapshot à l'arrivée, puis différences, avec la suppression comme marqueur explicite.

---

## 7. Réplication

- **Ce qui part chez tous** : `Workspace` (selon le streaming), `ReplicatedStorage`, `ReplicatedFirst`, les conteneurs `StarterGui`, `StarterPack` et `StarterPlayer`, `Lighting`, `Teams`, `SoundService`. Chez un seul joueur : son `PlayerGui` et son `Backpack`. Jamais : `ServerScriptService`, `ServerStorage`.
- **Règle les propriétés avant `Parent`.** L'instance part une fois, dans son état final. Après le parentage, chaque changement est un message séparé et les clients voient des états intermédiaires. Pour un groupe, construis sous un modèle non parenté, puis parente une seule fois.
- **Choisis le canal selon la donnée :**
  - état public affiché (PV d'un boss, propriétaire d'une parcelle, phase de manche) → propriétés, attributs, tags : réplication gratuite, compatible avec le streaming ;
  - état privé (inventaire, monnaie, cooldowns d'un joueur) → `FireClient` ciblé, car un attribut est diffusé à tous les clients ;
  - actions et intentions → remotes, sans renvoyer une propriété déjà répliquée ;
  - effet ponctuel (particules, son) → le serveur annonce l'événement, chaque client concerné joue l'effet chez lui.
- **Remotes** : créés par le serveur au démarrage, dans un seul dossier (`ReplicatedStorage.Remotes`), à partir d'une seule liste de noms partagée. Le client attend ce dossier une fois. Un petit module `Net` cache les `WaitForChild` et fait passer tous les handlers serveur par la même validation. Jamais de remote créé par le client ni d'`Instance.new("RemoteEvent")` éparpillé. Déclarer les remotes dans le fichier projet Rojo marche aussi. Un remote par action logique, nommé par l'intention (`RequestEquip`, `InventoryUpdated`), pas un remote universel avec un champ `kind`.
- Le choix entre `RemoteEvent`, `RemoteFunction` et `UnreliableRemoteEvent`, les débits et ce qui survit à la sérialisation sont dans securite.md. À retenir : jamais `InvokeClient` ; une table arrive copiée et sans métatable ; envoie des IDs que le receveur résout, ou un `InstanceHandle` (section 10), car une instance renvoyée peut ne pas exister encore chez lui (streaming).
- **Aucun ordre garanti** entre un remote et la réplication d'une propriété ou d'un attribut. À la réception de « équipe changée », attends l'objet lié au lieu de le supposer présent. Si l'ordre compte, un seul canal d'état avec un numéro de version.
- Un `RemoteEvent` tiré avant qu'un écouteur existe est mis en file, dans une limite, puis livré à la connexion. N'en dépends pas : connecte tôt et utilise le handshake « client prêt » (section 5).
- **Signaux différés** (`Workspace.SignalBehavior = Deferred`, réglage de Propriétés) : les handlers tournent au prochain point de reprise, pas pendant `Fire`. Teste avec ce mode ; un code qui suppose un `ChildAdded` immédiat est fragile.
- **Streaming** (`StreamingEnabled`) : côté client, les instances lointaines manquent, puis entrent et sortent. Pas de référence dure vers les parts d'une autre zone ; des tags avec `GetInstanceRemovedSignal` ; `WaitForChild` avec timeout ; `ModelStreamingMode = Atomic` pour un modèle qui doit arriver entier, `Persistent` pour un modèle toujours présent. Le serveur voit tout : c'est lui qui place les objets d'événement, posés par un raycast sur le vrai sol et ancrés. Détails : performance.md.
- Entre serveurs, MessagingService livre au mieux, sans garantie : des notifications, jamais l'état de jeu. L'état partagé va en MemoryStore ou en DataStore (donnees.md).
- Teste la réplication avec le mode de test Server & Clients (ancien nom : Test → Clients and Servers) et au moins 2 clients, avec de la latence simulée : le mode Play solo cache les problèmes d'ordre. Après une publication, les anciens serveurs gardent l'ancienne version jusqu'à leur fermeture : un bug « déjà corrigé » peut venir d'un vieux serveur. Imprime au démarrage de chaque côté une version ou un hash de build pour savoir quel code tourne.

---

## 8. Cycle de vie du joueur et du personnage

- **Le `Player` est l'identité durable** : les données de session sont indexées par `Player` et vidées à `PlayerRemoving`. **Chaque `Character` est remplaçable** : ses références, connexions, pistes d'animation, hitbox et buffs vivent dans une portée « par vie », fermée à `CharacterRemoving` ou à l'arrivée du personnage suivant.
- Des joueurs peuvent être là avant ton script, surtout en Studio : connecte `PlayerAdded`, puis traite `Players:GetPlayers()`. Même chose pour `player.Character` avant `CharacterAdded`.
- Ordre : `PlayerAdded` (pas encore de personnage) → `CharacterAdded` → `CharacterAppearanceLoaded` → `Humanoid.Died` → `CharacterRemoving` → nouveau `CharacterAdded` → … → `PlayerRemoving` (le personnage existe peut-être encore).
- Au moment de `CharacterAdded`, l'Humanoid et les parties existent côté serveur, mais le placement au point d'apparition n'est pas forcément fait : ne lis pas la position dans les premières lignes. Les accessoires arrivent plus tard : attends `Player.CharacterAppearanceLoaded` (serveur) pour les toucher. Côté client, `WaitForChild("Humanoid")` et `WaitForChild("HumanoidRootPart")`.
- **Après chaque yield, revérifie** que le personnage est toujours le bon (compteur de génération ou `player.Character == character`) : une requête peut arriver après une mort ou un respawn. Ne code pas en dur les parties R6 ou R15. Un personnage piloté par la Character Controller Library (`StarterPlayer.LuaCharacterController`, `ControllerManager`) peut ne pas avoir d'Humanoid.
- `Humanoid.Died` est une mort de gameplay, pas un nettoyage garanti. Le corps reste `Players.RespawnTime` secondes (5 par défaut). Respawn manuel des jeux à manches : `Players.CharacterAutoLoads = false`, puis `player:LoadCharacterAsync()` côté serveur (`LoadCharacter` est déprécié). Dans ce mode, `StarterGui` n'est copié dans `PlayerGui` qu'au premier chargement : crée par code l'UI du lobby (ui-fondamentaux.md).
- **Départ** : `PlayerRemoving` libère les tables par joueur de chaque système. La sauvegarde appartient au propriétaire des données, qui gère aussi crash, téléportation et `BindToClose` (30 s en production) : donnees.md. Pour que le moteur appelle `Destroy()` sur l'ancien personnage quand il est remplacé et sur l'objet `Player` au départ, règle `Workspace.PlayerCharacterDestroyBehavior` sur `Enabled` dans Propriétés (non scriptable). Sinon `task.defer(character.Destroy, character)` dans `PlayerRemoving`. Sans cela, les connexions qui le référencent le gardent en mémoire.
- UI : `ScreenGui.ResetOnSpawn` vaut `true` par défaut. Mets `false` sur un HUD qui doit survivre au respawn. Ne garde jamais de référence vers un clone détruit de `PlayerGui`.

```luau
--!strict
local Players = game:GetService("Players")

-- Génération : le travail asynchrone capture la sienne et abandonne si elle a changé.
local generation: { [Player]: number } = {}

local function onCharacter(player: Player, character: Model)
	local mine = (generation[player] or 0) + 1
	generation[player] = mine
	local humanoid = character:WaitForChild("Humanoid", 10) :: Humanoid?
	if humanoid == nil or generation[player] ~= mine then
		return -- personnage remplacé ou joueur parti pendant l'attente
	end
	humanoid.Died:Once(function()
		print(`[Life] Player({player.UserId}) mort, vie {mine}`)
	end)
end

local function onPlayer(player: Player)
	player.CharacterAdded:Connect(function(character)
		onCharacter(player, character)
	end)
	local current = player.Character
	if current then
		task.spawn(onCharacter, player, current)
	end
end

Players.PlayerAdded:Connect(onPlayer)
for _, player in Players:GetPlayers() do
	task.spawn(onPlayer, player)
end
Players.PlayerRemoving:Connect(function(player)
	generation[player] = nil
end)
```

---

## 9. Nettoyage

- **Tout ce qui est créé a un propriétaire qui le nettoie** : connexions, threads, tweens, instances, entrées de tables. Une portée de nettoyage par durée de vie : session du joueur, vie du personnage, manche, composant, écran d'UI.
- Ajoute à la portée au moment de la création. Utilise la lib du projet (Trove, Maid, Janitor). Sinon, ce module suffit :

```luau
--!strict
-- shared/Scope.luau : regroupe ce qu'un propriétaire doit défaire
export type Task = RBXScriptConnection | Instance | thread | () -> ()

local Scope = {}
Scope.__index = Scope

export type Scope = typeof(setmetatable({} :: { tasks: { Task }, closed: boolean }, Scope))

local function clean(item: Task)
	if typeof(item) == "RBXScriptConnection" then
		item:Disconnect()
	elseif typeof(item) == "Instance" then
		item:Destroy()
	elseif typeof(item) == "thread" then
		task.cancel(item)
	else
		item()
	end
end

function Scope.new(): Scope
	return setmetatable({ tasks = {}, closed = false }, Scope)
end

function Scope.add(self: Scope, item: Task)
	if self.closed then
		clean(item) -- propriétaire déjà fermé : on défait tout de suite au lieu de fuir
		return
	end
	table.insert(self.tasks, item)
end

function Scope.destroy(self: Scope)
	if self.closed then
		return
	end
	self.closed = true
	for i = #self.tasks, 1, -1 do -- ordre inverse de création
		-- Un nettoyage qui échoue n'arrête pas les suivants. `local ok, err = pcall(clean, …)` est refusé
		-- par l'ancien solveur en strict : `clean` ne renvoie rien, donc pas de deuxième valeur.
		xpcall(clean, function(err)
			warn("[Scope]", err)
		end, self.tasks[i])
	end
	table.clear(self.tasks)
end

return Scope
```

- `:Once()` pour un événement à usage unique. `:Wait()` seulement si céder sans timeout ni annulation est acceptable.
- `Parent = nil` n'est pas une destruction : l'instance vit tant qu'une référence existe. `Destroy()` déconnecte ses événements, verrouille `Parent` et ne se défait pas. Les connexions posées sur les instances du personnage meurent avec lui ; une connexion tenue ailleurs qui le référence (table, signal global) le garde en vie.
- **Fermeture propre** : marque le propriétaire « en fermeture » avant de détruire ses entités, car ces destructions déclenchent des callbacks (mort, retrait). Les handlers distinguent jeu normal et fermeture ; le travail différé d'une ancienne manche est rejeté par sa génération. Le nettoyage est idempotent.
- **Pas de table à clés faibles indexée par Instance** pour garder un état : l'entrée peut disparaître alors que l'instance existe encore (bug réel, invisible dans les tests hors moteur). Nettoie explicitement, ou stocke la valeur en attribut sur l'instance.
- Une recherche de `:Destroy(` ou `:Disconnect(` n'est pas une preuve. Le test de fuite l'est : compte les connexions ou les descendants, répète 10 cycles (ouvrir et fermer, mourir et réapparaître, manche complète), vérifie le retour à la base.

---

## 10. CollectionService et attributs

- **Tags = découverte, attributs = configuration.** Un seul script gère toutes les instances d'un tag : portes, coffres, zones, spawners. Pas un Script copié dans chaque modèle : les copies divergent. Les dossiers servent au rangement, pas au sens : pour « tous les objets ramassables », un tag.
- Traite les trois cas : `GetTagged` (existants), `GetInstanceAddedSignal` (nouveaux), `GetInstanceRemovedSignal` (retirés). Le retrait n'est pas optionnel avec le streaming : côté client, les instances taggées sortent et reviennent sans cesse.
- `CollectionService.TagAdded` et `TagRemoved` signalent qu'un tag apparaît ou disparaît **dans la place**, pas qu'une instance le reçoit. Pour une instance, c'est `GetInstanceAddedSignal(tag)`. `attach` est idempotent et possède sa portée de nettoyage. Valide la classe et les attributs à la frontière (`warn` avec `GetFullName()`), avec des valeurs par défaut.
- Pose en code, au démarrage, les tags dont dépend la logique serveur (en parcourant un dossier connu) plutôt qu'à la main dans l'éditeur de tags : les tags manuels s'oublient sur les nouvelles instances.

```luau
--!strict
-- Script serveur : un seul gestionnaire pour toutes les portes taggées
local CollectionService = game:GetService("CollectionService")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Scope = require(ReplicatedStorage.Shared.Scope)

local TAG = "Door"
local active: { [Instance]: Scope.Scope } = {}

local function attach(door: Instance)
	if active[door] or not door:IsA("Model") then
		return -- déjà branchée, ou mauvaise classe
	end
	local found = door:FindFirstChild("Panel")
	if found == nil or not found:IsA("BasePart") then
		warn(`[Doors] {door:GetFullName()} n'a pas de Panel`)
		return
	end
	local panel = found :: BasePart
	local scope = Scope.new()
	active[door] = scope
	local function apply()
		local open = door:GetAttribute("Open") == true -- absent ou mauvais type = fermée
		panel.CanCollide = not open
		panel.Transparency = if open then 0.8 else 0
	end
	scope:add(door:GetAttributeChangedSignal("Open"):Connect(apply))
	apply()
end

local function detach(door: Instance)
	local scope = active[door]
	if scope then
		active[door] = nil
		scope:destroy()
	end
end

CollectionService:GetInstanceAddedSignal(TAG):Connect(attach)
CollectionService:GetInstanceRemovedSignal(TAG):Connect(detach)
for _, door in CollectionService:GetTagged(TAG) do
	attach(door)
end
```

- **Attributs** : booléens, nombres, chaînes et types de données Roblox (`Vector3`, `Color3`, `UDim2`…), pas de tables. Une donnée structurée va dans un module, indexée par instance. Mettre un attribut à `nil` le supprime. `GetAttributeChangedSignal(nom)` réagit à un changement. Un attribut posé par le client ne se réplique pas vers le serveur : lu côté serveur et écrit seulement par le serveur, il est fiable.
- Référence vers une autre instance : un attribut de type Instance, utilisable en production depuis 2026-09 (DevForum). `GetAttribute` renvoie un `InstanceHandle` : `nil` veut dire « attribut absent », `handle:Get()` à `nil` veut dire « cible pas là pour l'instant » (streaming) ; `handle:Wait(timeout)` attend. `GetAttributeChangedSignal` ne se déclenche pas quand la cible entre ou sort du streaming. `InstanceHandle.new(inst)` passe aussi par un remote. Code existant : `ObjectValue` sous un parent stable, dont `Value` vaut `nil` tant que la cible n'est pas chargée.
- **Réglages modifiables par le propriétaire dans Studio** : attributs d'une `Configuration` (par exemple `ReplicatedStorage.Config.GameSettings`), posés par le code seulement s'ils manquent, lus par les deux côtés avec repli sur la constante du code, jamais réécrits à l'exécution. Avec Rojo, déclare-les dans les fichiers du dépôt pour que la synchro ne les écrase pas (outils.md).

---

## 11. Boucles, manches et temps

- **Ordre des événements de `RunService` à chaque frame** : `PreRender` (client seulement, ancien `RenderStepped`), `PreAnimation` (client et serveur), `PreSimulation` (ancien `Stepped`, avant la physique), `PostSimulation` (après la physique), puis `Heartbeat` (fin de frame). `RenderStepped` et `Stepped` ne sont pas dépréciés, mais la doc demande `PreRender` et `PreSimulation` pour le nouveau code ; `Heartbeat` n'a pas de remplaçant (doc, 2026-10-18). Côté serveur, `RenderStepped` lève une erreur et `PreRender` ne se déclenche jamais ; `BindToRenderStep` est réservé au client. Le travail serveur par frame va sur `Heartbeat`.
- **Réagis aux signaux plutôt que de sonder** : `Humanoid.Died`, `GetPropertyChangedSignal`, `AttributeChanged`. Une connexion par frame seulement quand la haute fréquence est indispensable (caméra, ressort d'UI).
- **Une boucle par système**, pas une par objet. Les événements temporisés (double XP, boss, coffres) s'inscrivent dans un seul ordonnanceur ; un `while true` par événement dérive et s'empile.
- **Machine à états de manche** : des phases explicites (attente, compte à rebours, jeu, résultats), une fonction par transition, une boucle plate où chaque phase renvoie la suivante. Pas de transitions récursives, qui empilent toute la partie dans une seule pile d'appels.
- Chaque manche a un ID de génération : tout travail différé le vérifie avant d'agir, pour qu'un callback de la manche N ne touche pas la N+1. Les ressources de la manche vivent dans une seule portée de nettoyage, détruite à un seul endroit. Diffuse l'échéance une fois (heure serveur, `workspace:GetServerTimeNow()`), et laisse chaque client compter localement. Arrivées et départs en pleine manche sont des cas normaux, à tester. Si une manche lève une erreur, garde l'erreur d'origine, nettoie et rétablis le lobby avant la suivante ; ne reboucle jamais en silence sur un état cassé.
- Calcul lourd : étale-le sur plusieurs frames, ou passe par Parallel Luau avec des Actors (performance.md). Mettre un Script dans un Actor ne suffit pas à le paralléliser.

---

## 12. Organiser un projet Rojo ou roblox-ts

Projet Luau avec Rojo :

```
default.project.json
rokit.toml               épingle rojo, wally, selene, stylua (outils.md)
src/
  server/                → ServerScriptService.Server
    init.server.luau       bootstrap : le dossier devient un Script, les fichiers ses enfants
    Systems/               un ModuleScript par système
  client/                → StarterPlayer.StarterPlayerScripts.Client
    init.client.luau       bootstrap client
    Controllers/
  shared/                → ReplicatedStorage.Shared
    Types.luau  Config.luau  RemoteNames.luau  Scope.luau
Packages/                paquets Wally partagés → ReplicatedStorage
ServerPackages/          paquets Wally serveur → ServerScriptService
```

- Suffixes Rojo : `.server.luau` donne un Script, `.client.luau` un LocalScript, `.luau` un ModuleScript. `init.luau` fait du dossier un ModuleScript. Piège fréquent : un module nommé `X.server.luau` devient un Script, et `require` échoue.
- Le realm Wally est une frontière de sécurité : un paquet `server` monté dans `ReplicatedStorage` devient lisible par tout exploiteur.
- Nomme si possible les dossiers de `src/` comme leur point de montage : on lit où le code tourne dans son chemin. Script Sync (Studio) a d'autres suffixes : outils.md.

Projet roblox-ts (modèle `game`, la pile par défaut de cet utilisateur) :

```
src/server/main.server.ts   → Script dans ServerScriptService.TS
src/server/**/*.ts          → ModuleScripts serveur
src/client/main.client.ts   → LocalScript dans StarterPlayer.StarterPlayerScripts.TS
src/client/ui/              → composants React (ui-react.md)
src/shared/**/*.ts          → ModuleScripts dans ReplicatedStorage.TS
include/ + node_modules/    → ReplicatedStorage.rbxts_include (runtime et paquets)
out/                        → sortie de rbxtsc : ne jamais l'éditer
```

- Sous `rbxts_include.node_modules`, un enfant par scope npm installé : `@rbxts`, `@flamework` avec Flamework, et `@rbxts-js` dès que `@rbxts/react` est là. Sans ce dernier, aucune UI n'apparaît et Output affiche `Infinite yield possible … WaitForChild("@rbxts-js")`. Correctif : ui-react.md, section 2.
- Imports depuis la racine `src` : `import { Config } from "shared/config"`. Services : `import { Players } from "@rbxts/services"`. N'importe jamais `server/` depuis `client/` ou `shared/` : ce module n'existe pas chez le client.
- Les numéros de ligne des erreurs Studio pointent vers `out/` : retrouve le `.ts` correspondant avant de corriger.

Règles d'organisation, pour les deux piles :

- Le premier découpage se fait par côté d'exécution (serveur, client, partagé) : l'emplacement est une décision de confiance. À l'intérieur, range à plat tant que le projet est petit, puis par fonctionnalité (`Features/Inventory/`) quand les systèmes se multiplient. Ne réorganise jamais un projet qui marche pour coller à un modèle : imite sa disposition.
- Garde dans l'expérience une place de test séparée de la place publiée, pour les tests moteur et les essais d'API.

---

## 13. Server Authority

- Mode moteur sorti pour tous le 2026-07-09 (annonce « Full Release » de Roblox) : `Workspace.AuthorityMode = Server`. Le serveur simule le mouvement et la physique ; les clients prédisent, puis corrigent par retour arrière et resimulation. Ce n'est pas `SetNetworkOwner`.
- Il impose cinq autres réglages, à vérifier ensemble car un fichier de place peut dériver : `NextGenerationReplication`, `PlayerScriptsUseInputActionSystem`, `SignalBehavior = Deferred`, `UseFixedSimulation` et `StreamingEnabled`.
- Aucun script ne peut basculer une place : tout se règle dans Propriétés. Lire `AuthorityMode` depuis la barre de commande ou le MCP échoue avec une erreur de capacité qui ne dit rien du mode : demande à l'utilisateur ou regarde Propriétés.
- Désactivé par défaut (valeur `Automatic`). Ne le suppose jamais actif et ne migre jamais de toi-même. Migration quasi gratuite pour des personnages Humanoid standard ; réécriture pour une simulation maison (combat, mouvement custom), plus d'un mois rapporté par un développeur, et pas de retour arrière propre.
- Une fois actif : les entrées qui pèsent sur la simulation passent par l'Input Action System (`InputAction`, `InputContext`), la logique synchronisée par `RunService:BindToSimulation(...)`, pas par des remotes ni par une correction sur `Heartbeat`. Dans ce callback, seules les propriétés et méthodes marquées Simulation Access sont permises (les autres lèvent une erreur, selon un dépôt) ; `task.defer` en sort, pas `task.spawn` (doc, 2026-10-18). Écrire `BasePart.CFrame` hors du callback de simulation peut aussi lever une erreur (à vérifier).
- Les callbacks de simulation peuvent être rejoués : sépare l'état simulé de la présentation, saute les effets ponctuels quand `RunService:IsResimulating()` est vrai (API récente), et réagis aux changements d'attributs dans `PreRender` pour les visuels. Les erreurs de prédiction sont normales.
- Limites (doc et annonce, 2026-10) : un attribut d'instance prédite ne se synchronise que s'il est parmi ses 64 premiers, avec un nom et une valeur texte de 50 caractères au plus. L'état de la caméra d'un client se lit côté serveur par `Player:GetCameraState()`, et non plus par une InputAction de caméra.
- Il renforce la validation sans la remplacer : les handlers de remotes vérifient toujours type, bornes, propriété et débit (securite.md).

---

## 14. Pièges

- Plusieurs Scripts serveur qui supposent un ordre de démarrage ; un module qui yield dans son corps (démarrage figé, sans erreur).
- `_G` ou `shared` pour partager l'état ; de la logique serveur, une table de loot ou une liste d'admins dans `ReplicatedStorage`, même désactivée.
- Un `LocalScript` dans `Workspace` ou `ServerScriptService`, un `Script` dans `ServerStorage` : ils ne tournent pas. Un module nommé `.server.luau`, puis requis.
- `PlayerAdded` connecté sans traiter `GetPlayers()` : le premier joueur est raté en Studio.
- La position lue dès le début de `CharacterAdded` ; une référence au personnage gardée après le respawn.
- Des propriétés réglées après `Parent` (états intermédiaires chez les clients) ; un attribut pour un état privé (il part chez tous).
- `TagAdded` pris pour un signal par instance ; pas de `GetInstanceRemovedSignal` sous streaming.
- Une table à clés faibles indexée par Instance qui perd ses entrées ; la ligne après `BindableEvent:Fire` qui suppose que les écouteurs ont tourné.
- Un objet ajouté dans `ReplicatedFirst` après l'arrivée du joueur ; une `Camera` créée côté serveur pour un ViewportFrame.
- Un fichier modifié dans `out/` au lieu de `src/` ; le scope `@rbxts-js` non mappé.
- Un framework introduit, ou Server Authority et le streaming activés, sans accord de l'utilisateur.

---

## 15. Checklist

- [ ] Un seul Script serveur et un seul LocalScript client ; tout le reste en ModuleScripts.
- [ ] Bootstrap : `init` sans yield puis `start`, dans un ordre explicite, avec des erreurs visibles.
- [ ] Chaque donnée et chaque effet de bord (profil, reçus, monnaie) a un seul propriétaire nommé.
- [ ] Rien de secret ni de décisionnel dans un conteneur répliqué.
- [ ] Remotes créés par le serveur, dans un seul dossier, depuis une seule liste de noms ; propriétés réglées avant `Parent`.
- [ ] Joueurs et personnages déjà présents traités ; état par vie et état par joueur nettoyés.
- [ ] Chaque connexion, thread, tween et instance appartient à une portée de nettoyage.
- [ ] Systèmes à tags : `GetTagged`, ajout et retrait traités, `attach` idempotent ; code client robuste à l'absence (`WaitForChild` avec timeout, objets streamés).
- [ ] roblox-ts : `@rbxts-js` mappé, `rbxtsc` sans erreur, rien d'édité dans `out/`.
- [ ] Aucun framework ni réglage moteur (Server Authority, streaming) changé sans accord.
- [ ] Testé à 2 clients : arrivée tardive, mort, respawn, départ en pleine action.
- [ ] Test de fuite : connexions et instances revenues à la base après 10 cycles.
