# Performance : CPU, mémoire, réseau, rendu

À lire quand : le jeu rame, saccade, chauffe ou crashe sur téléphone, met longtemps à charger, ou quand tu écris une boucle par frame, un spawner, des effets, un système à beaucoup d'entités ou un grand monde. Les bases du langage sont dans luau.md ; la performance des UI React (bindings, re-renders) est dans ui-react.md.

## Sommaire

1. Méthode : mesurer d'abord
2. Budgets et chiffres
3. Triage : symptôme → cause
4. Outils de mesure
5. CPU : boucles et chemins chauds
6. Luau rapide et code natif
7. Parallel Luau
8. Mémoire et fuites
9. Pools d'objets
10. Réseau et réplication
11. Instances, physique, requêtes
12. Rendu, effets, UI
13. StreamingEnabled
14. Appareils faibles et chargement
15. Pièges
16. Checklist

## 1. Méthode : mesurer d'abord

- Boucle : reproduire → savoir qui paie (client, serveur, les deux) → mesurer → attaquer le plus gros coût unique → changer **une** chose → remesurer sur l'appareil cible → revenir en arrière si rien n'a bougé. Rapporte la mesure, pas l'intention. Corrige le coût algorithmique avant les micro-détails, et ne touche pas au code froid.
- Le lag vient presque toujours de quatre causes : connexions qui fuient, déchets créés à chaque frame, trafic réseau non groupé, trop d'instances qui font trop peu. Sur téléphone, ajoute la mémoire des textures et le remplissage des pixels (overdraw).
- Qui paie : le rendu et les entrées pèsent sur le client ; la physique et la réplication sur le serveur ; le Luau sur la machine qui l'exécute.
- Studio ment : client et serveur partagent un processus sur un PC puissant, et l'émulateur d'appareil n'est pas un test mémoire. Mesure sur un vrai téléphone d'entrée de gamme, sous charge réaliste : 1 joueur et 30 mobs ne disent rien de 40 joueurs et 600 mobs. Teste avec le nombre de joueurs visé, en session longue.

## 2. Budgets et chiffres

Chiffres publiés par Roblox (2026), citables :

| Mesure | Valeur |
|---|---|
| Durée d'une frame | 16,67 ms à 60 FPS ; 33,33 ms à 30 ; 8,33 ms à 120 ; 4,17 ms à 240. Tes scripts n'en ont que quelques ms |
| Scène de base, appareil de référence | moins de 1 000 draw calls et de 1 000 000 de triangles (exemple donné par la doc, pas une limite officielle) |
| Mémoire d'un serveur | 6,25 Gio + 100 Mio × le plus grand nombre de joueurs connectés. Reste sous 50 % : la mémoire n'est pas rendue quand les joueurs partent |
| Heartbeat serveur | plafonné à 60 Hz. En dessous, les joueurs le vivent comme du ping, pas comme une baisse de FPS |
| Taux de crash client | à étudier au-delà de 2 à 3 % |
| Parc d'appareils | environ 65 % d'Android, dont environ 60 % à 2–4 Go de RAM |
| Textures | 512 × 512 au plus pour le décor, 256 × 256 pour les petites images |
| MicroProfiler | barre rouge quand l'attente GPU dépasse 2,5 ms et que le rendu dure plus que les jobs |
| `UnreliableRemoteEvent` | 1 000 octets par envoi, au-delà le message est jeté |

- Il n'existe **aucun budget officiel en ms par étiquette** du MicroProfiler : une étiquette pose problème quand elle domine une frame qui rate sa cible. La régularité compte plus que la moyenne : 16 ms de moyenne avec des pics à 40 ms se ressentent comme des saccades. Lis les frames une par une.

Heuristiques (points de départ, pas des limites) : serveur sous 10 ms de script par frame ; travail lent à 5–10 Hz ; pool au-delà d'environ 1 apparition par seconde ; 30–50 Ko/s de réseau par joueur ; 400–500 Mo de RAM client sur bas de gamme ; chargement jouable en moins de 10 s ; triangles d'un objet ramassable sous 2 000, d'un prop sous 3 000, d'un prop héros sous 6 000. Au-delà de quelques milliers de Parts visibles, un téléphone rame.

## 3. Triage : symptôme → cause

| Symptôme | Cause probable |
|---|---|
| FPS client bas partout, même immobile | rendu, nombre d'instances ; pas le Luau |
| FPS qui baisse avec le nombre de joueurs ou d'entités | Luau par entité, ou volume de réplication |
| Ping élevé pour tous, FPS client correct | le serveur rate ses frames : physique (parts non ancrées, Humanoids) ou réplication sortante |
| Data Ping bien plus haut que le Network Ping | files de réplication engorgées : envoie moins, plus petit |
| Mémoire qui monte et ne redescend pas | fuite : connexions, tables indexées par Player ou Instance |
| Pic ponctuel (début de manche, téléport, premier coup) | travail de masse dans une seule frame, ou asset chargé à la demande |
| Seuls les appareils faibles souffrent ou crashent | mémoire (textures d'abord) ou remplissage des pixels |
| Saccade à chaque événement réseau | payload trop gros ou rafale |
| Long à rejoindre | taille de la réplication initiale et des assets |
| « Marche dans Studio, pas en jeu » | streaming ou hypothèse client/serveur, rarement la syntaxe |

## 4. Outils de mesure

**MicroProfiler** : temps de chaque sous-système, frame par frame.
- Ouverture : menu de Studio, ou raccourci Ctrl+F6 (⌘F6), dans Studio comme dans le client (doc, 2026-10-18 ; d'anciennes versions de Studio utilisaient Ctrl+Alt+F6). `Ctrl+P` met en pause pour lire une frame.
- Serveur : Developer Console (`F9`) → MicroProfiler → Server, puis lance un enregistrement (60 frames au plus) et ouvre le fichier HTML. Les dumps vont dans `%LOCALAPPDATA%\Roblox\logs`.
- Téléphone : active le MicroProfiler dans les réglages du jeu, puis ouvre l'adresse IP:port affichée depuis un navigateur du même réseau. L'interface web offre flame graphs, comparaison de deux dumps et vue mémoire.
- Étiquette ton code : `debug.profilebegin("MobAI")` … `debug.profileend()`, appariés dans le même thread. Ce n'est pas gratuit : garde-les derrière une constante `PROFILING`. `debug.setmemorycategory("MobAI")` en tête d'un thread de longue durée range sa mémoire sous ce nom dans la Developer Console.

| Étiquette | Ce qu'elle mesure | Levier |
|---|---|---|
| `RunService.Heartbeat`, `PreRender`, `PreSimulation`, `PostSimulation` | ton code branché sur ces événements | throttle, lots, natif |
| `physicsStepped`, `worldStep`, `stepContacts` | physique | ancrer, simplifier les collisions |
| `stepHumanoid`, `stepAnimation` | Humanoids, animations | moins d'Humanoids, animer les PNJ côté client |
| `updateInvalidatedFastClusters` | avatars et meshes skinnés reconstruits | `Motor6D.Transform`, rigs stables |
| `ShadowMapSystem`, `LightGridCPU` | ombres et lumières | moins de lumières, `CastShadow` coupé |
| `Prepare`, `Perform` | rendu | draw calls, transparence |
| `Replicator/ProcessPackets` | réseau entrant | envoyer moins, rayons de streaming |
| `Allocate Bandwidth and Run Senders` | réplication sortante | grouper, ne pas répliquer le visuel |
| `GC` | ramasse-miettes | chasser les allocations |

**Script Profiler** (Developer Console) : temps CPU par fonction, en vue Callgraph ou Functions, trié du plus coûteux au moins coûteux. Enregistre une dizaine de secondes en mode manuel, sous charge réaliste. L'export JSON marque les fonctions natives (bit `IsNative`).

**Developer Console** (`F9`) :
- Memory : `LuaHeap` qui grimpe au repos = closures ou tables qui fuient ; `Instances` qui grimpe = objets non détruits ; aussi `Signals`, `PhysicsParts`, `PlaceScriptMemory`. Server Stats : Heartbeat en pas par seconde, ping, data ping.
- L'onglet Network compte les **appels web** (HttpService, DataStore), pas le trafic des remotes. Pour celui-ci : `Stats.DataSendKbps` et `DataReceiveKbps`, le Data Ping, ou la vue réseau d'un dump MicroProfiler.

Autres outils :
- **Scene Analysis** (Window → Performance Summary → Scene Analysis) : instances non parentées encore retenues (vue des fuites), mémoire Luau par script, composition des instances, mémoire d'animation et d'audio, composition des triangles. Les mêmes vues se lisent par code avec `SceneAnalysisService` (`GetUnparentedInstancesAsync`, `GetScriptMemoryAsync`…, méthodes qui cèdent), pratique dans un test de fuite.
- **Performance Dashboard** du Creator Dashboard : crashs, mémoire, FPS et durée de session sur de vrais joueurs. Seul outil qui voit la production.
- **Simulateur réseau** de Studio (beta, 2026-09) : latence, gigue et perte, réglables par sens. Le preset « Ideal Fiber » ajoute 8 ms par sens ; mets tout à 0 pour une vraie base.
- Service `Stats` : `GetTotalMemoryUsageMb()`, `GetMemoryUsageMbForTag(Enum.DeveloperMemoryTag.LuaHeap)`. `game.ServerLowMemoryWarning` (Studio 0.736+) prévient quand le serveur approche de sa limite : vide alors les caches non critiques.
- Dans Studio, l'outil MCP `skill` sert les skills intégrés `rbx-perf-profiling` et `rbx-scene-analysis` (voir studio-mcp.md).

## 5. CPU : boucles et chemins chauds

Rien de coûteux dans un chemin chaud (callback de RunService, boucle par entité) : pas de table ni de closure créée, pas de concaténation, pas de `FindFirstChild`, `WaitForChild`, `GetChildren` (qui alloue une table neuve) ni de chemin profond. Résous les références une fois, par exemple à `CharacterAdded`, et vérifie `ref.Parent` à l'usage.

Ordre d'une frame : `PreRender` → rendu → `PreAnimation` → `PreSimulation` → physique → `PostSimulation` → `Heartbeat`.

| Événement | Usage |
|---|---|
| `PreSimulation` (remplace `Stepped`) | ce qui alimente la physique ; écritures de `Motor6D.Transform` (l'Animator écrase les écritures faites ailleurs) |
| `PostSimulation`, puis `Heartbeat` juste après (deux événements distincts) | la logique générale, qui réagit à la physique |
| `PreRender` (remplace `RenderStepped`), client seul | caméra, visuel lié aux entrées, horloge légère des animations d'UI (ui-react.md) ; rien de lourd : il bloque la frame |
| `PreAnimation` | avant les animations ; tourne aussi sur le serveur |

En mode d'autorité serveur (`Workspace.AuthorityMode = Server`), la logique simulée passe par `RunService:BindToSimulation` à fréquence fixe (voir architecture.md) ; sinon, accumule `dt` sur `Heartbeat`.

- Branche RunService avec parcimonie. Barres de vie à 10 Hz, minimap à 4 Hz, IA et scans de proximité à 5–10 Hz, avec un accumulateur de `dt` ramené sous l'intervalle par modulo (`accumulated %= INTERVAL`, exemple en section 10) pour ne pas rattraper tous les ticks après un pic. Ou échelonne : traite l'entité `i` quand `i % N == frame % N`.
- Un gestionnaire par comportement, pas un script par entité : tags CollectionService et une seule boucle.
- Réagis aux événements plutôt que de sonder : `while true do task.wait() end` pour attendre une condition cache presque toujours un signal existant.
- Multiplie par `dt` tout ce qui avance à chaque frame (régénération, rotation), sinon la vitesse dépend du framerate.
- **Jamais de tween côté serveur sur un objet du monde** : chaque pas interpolé part vers chaque client et saccade avec la latence. Le serveur envoie l'intention (cible, durée, easing), chaque client tweene. TweenService plutôt qu'un easing à la main.
- Une écriture à la même valeur ne réplique rien. Mais chaque écriture qui change, même d'un epsilon, part vers chaque client : quantifie, et n'écris que si la valeur visible change.
- Pas de log à chaque frame : un message toutes les 5 s au plus, et retire les prints de debug avant publication.

Gros travail ponctuel (génération, apparition de masse, migration) : découpe-le avec un budget de **temps** par frame, pas un nombre d'éléments.

```luau
--!strict
local RunService = game:GetService("RunService")

local FRAME_BUDGET = 0.004 -- 4 ms de travail par frame

local function processQueue<T>(queue: { T }, handle: (T) -> ())
	local index = 1
	while index <= #queue do
		local started = os.clock()
		while index <= #queue and os.clock() - started < FRAME_BUDGET do
			handle(queue[index])
			index += 1
		end
		if index <= #queue then
			-- Yield : l'état du jeu a pu changer, revalide avant de continuer.
			RunService.Heartbeat:Wait()
		end
	end
end
```

## 6. Luau rapide et code natif

Coûts relatifs : champ de table < propriété d'Instance (traverse le moteur) < écriture d'une propriété répliquée (trafic vers chaque client). Référence en cache < `FindFirstChild` par frame. Objet de pool < `Instance.new` + `Destroy`. Un remote à dix champs < dix remotes. Test de distance < raycast. `table.clear` < table neuve. `table.concat` < `..` en boucle. Ordonne tes gardes de la moins chère à la plus chère.

- **Environnement pur.** Luau résout `math.max` au chargement du script : `local max = math.max` n'apporte rien. Mais `getfenv` (même en lecture), `setfenv` et `loadstring` rendent l'environnement impur et désoptimisent tout le script : imports, appels rapides et inlining.
- Formes de tables stables : crée un objet par un littéral qui contient tous ses champs. Les noms de champs constants sont mis en cache ; des clés qui varient cassent ce cache. Évite les `__index` fonctions et les chaînes de métatables profondes dans un chemin chaud.
- `table.create(n)` pour un tableau de taille connue rempli par index, jamais pour un dictionnaire.
- Habitudes inutiles venues de Lua 5.1 : sortir `#list` d'un `for i = 1, #list` (la borne est lue une fois) ; dérouler une boucle à la main ; mettre `obj.Method` dans une locale. En revanche, `string.byte(s, i)` profite d'un appel rapide, `s:byte(i)` non.
- Allocations : la frame qui alloue n'est pas celle qui saccade. Le GC est incrémental mais a une étape atomique qui peut figer des dizaines de ms (pire avec de grosses tables faibles et beaucoup de coroutines vivantes). Chasse la source d'allocation : réutilise une table par emplacement actif, sors les comparateurs et callbacks au niveau du module, évite les closures qui capturent un upvalue modifié.
- Gèle les métatables de classe (`table.freeze`) : depuis Luau 0.739, la recherche de métaméthodes y est mise en cache. Le surcoût de `pcall` est environ divisé par deux depuis 0.735. `vector` pour des maths sans allocation, `buffer` pour beaucoup d'état binaire (voir luau.md).

**Code natif.** `--!native` en tête de script, ou `@native` sur la ligne au-dessus d'une `local function`, avec `--!optimize 2` (1 par défaut en test Studio, 2 en jeu publié).
- La doc le décrit pour les scripts serveur. Roblox ne publie **aucun** facteur de gain : mesure avec et sans au Script Profiler, et `debug.dumpcodesize()` (barre de commande, vue Server) pour la taille.
- Bons candidats : boucles numériques ou `buffer` appelées très souvent, avec peu d'appels d'API. Mauvais : scripts dominés par des appels moteur, code d'initialisation, UI.
- Coûts : démarrage du serveur plus long, mémoire en plus. Plafonds : 64 K instructions par bloc, 32 K blocs par fonction, 1 M instructions par script, plus une limite mémoire ; au-delà, la compilation native s'arrête (erreur affichée seulement dans la fenêtre Output de Studio) et le reste tourne en bytecode.
- Désoptimisent : `getfenv`/`setfenv`, builtins appelés avec des arguments non numériques, valeurs qui violent les types annotés, point d'arrêt actif. Annote les paramètres chauds (`v: Vector3`).

## 7. Parallel Luau

Réserve-le à du CPU isolable qu'un profil a désigné : lots de raycasts, bruit, génération de chunks, IA spatiale lourde. Compare toujours avec la version série sur le matériel cible.

- Un `Actor` est l'unité d'isolation. Les scripts d'un même Actor tournent l'un après l'autre ; le parallélisme vient de **plusieurs** Actors. Prends-en plus que de cœurs (64 sur 4 cœurs marche), découpés par unité logique : un personnage, une zone, un chunk ; pas un par balle. Pas d'Actor dans un Actor, sauf rare exception (la doc le déconseille ; un script appartient à l'Actor ancêtre le plus proche).
- Le script doit descendre d'un Actor. Sinon `ConnectParallel` échoue (« Scripts that connect in parallel must be rooted under an Actor ») et `task.desynchronize()` ne sert à rien. Pas de `require` en phase parallèle : requiers tout en série avant. Deux Actors peuvent tourner dans des VM Luau différentes, chacune avec ses propres copies des modules : ne compte sur aucun état de module partagé entre Actors.
- Entrer en parallèle : `task.desynchronize()`, `signal:ConnectParallel(fn)`, `actor:BindToMessageParallel(topic, fn)`. Revenir en série : `task.synchronize()`, obligatoire avant d'écrire dans le DataModel.
- Sûreté des membres d'API : Unsafe (défaut quand rien n'est indiqué), Read Parallel, Local Safe (écriture dans le même Actor), Safe. En parallèle : `workspace:Raycast`, lectures de propriétés, maths, `buffer`, `SharedTable`. En série seulement : `CFrame`, `Parent`, création et destruction, `workspace:BulkMoveTo`, `Terrain:WriteVoxels`, la plupart des tirs de signaux.
- `actor:SendMessage(topic, ...)` est asynchrone et copie ses arguments d'une VM à l'autre : pas de fonctions, et une grosse table est copiée à chaque envoi. Envoie des données simples, des Instances ou un `SharedTable`.
- `SharedTable` : partagé sans copie. Valeurs : booléens, nombres, vecteurs, chaînes, `SharedTable` et types de données sérialisables (pas de fonction ; les Instances ne sont pas listées). `SharedTable.increment(t, key, delta)` est atomique ; `SharedTable.update(t, key, fn)` peut appeler `fn` plusieurs fois, donc `fn` doit être pure ; lire puis écrire `t[key]` n'est pas atomique. `SharedTable.clone` superficiel donne un instantané atomique (un clone profond ne l'est pas dans son ensemble), `cloneAndFreeze` une copie gelée.
- Motif : désynchroniser → calculer → ranger les résultats → `task.synchronize()` → écritures groupées. À éviter : des allers-retours série/parallèle dans de petites boucles, des centaines de scripts qui se synchronisent dans la même frame (agrège dans un coordinateur), un message par frame. Un calcul long reste long : découpe-le sur plusieurs frames.

```luau
--!strict
-- Script placé sous un Actor. Le coordinateur appelle : actor:SendMessage("Scan", origins)
local actor = script:GetActor()
assert(actor, "ce script doit descendre d'un Actor")

local rayParams = RaycastParams.new()
rayParams.FilterType = Enum.RaycastFilterType.Exclude

actor:BindToMessageParallel("Scan", function(origins: { Vector3 })
	-- Phase parallèle : lectures et raycasts seulement.
	local hits: { Vector3 } = {}
	for _, origin in origins do
		local result = workspace:Raycast(origin, Vector3.new(0, -100, 0), rayParams)
		if result then
			table.insert(hits, result.Position)
		end
	end
	task.synchronize()
	-- Phase série : les écritures dans le DataModel sont permises à partir d'ici.
	print(`{#hits} impacts`)
end)
```

## 8. Mémoire et fuites

- Le moteur ne libère jamais une connexion faite sur une Instance vivante, ni ce que son callback référence. Détruire une Instance coupe **ses** signaux, pas les connexions faites ailleurs pour elle (un `Heartbeat` qui met à jour la barre de vie d'un mob mort), ni les threads lancés.
- Chaque objet qui crée des connexions possède une liste de nettoyage exécutée une fois à sa destruction : Trove, Janitor ou Maid (voir paquets.md), ou une table de fonctions. Écris le nettoyage dans la même modification que la connexion : un nettoyage « ajouté plus tard » est une fuite livrée maintenant.
- Les tables de module indexées par Player ou Instance sont la fuite n°1. Chaque insertion a son retrait (`PlayerRemoving`, `Destroying`). Une table faible n'est pas une stratégie de nettoyage (voir luau.md).
- `Player` et `Character` ne sont pas détruits automatiquement au départ : tout ce qui y reste connecté ou attaché fuit pendant toute la vie du serveur. Nettoie à `PlayerRemoving` et `CharacterRemoving`, puis active `Workspace.PlayerCharacterDestroyBehavior` (fenêtre Propriétés : il détruit l'ancien personnage au respawn et le Player à son départ) ou détruis le personnage en différé.
- `Destroy()` plutôt que `Parent = nil` : `Destroy` coupe les connexions de l'objet et le rend libérable. Lâche aussi la référence (`ref = nil`).
- Charge chaque `Animation` une seule fois par `Animator` et garde l'`AnimationTrack` : recharger à chaque lecture accumule des pistes. Effets : un émetteur persistant avec `:Emit(n)`, un `Sound` réutilisé, un son ponctuel détruit à `Ended`. `Debris:AddItem(obj, s)` pour un objet temporaire, pas pour un objet de gameplay important.
- Sur téléphone, la mémoire est la première cause de crash (l'erreur 292 est l'avertissement de mémoire basse). Ce qui la remplit, du plus lourd au plus léger : textures, accessoires et vêtements superposés, nombre d'instances, tas Luau.

```luau
--!strict
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")

local connectionsByPlayer: { [Player]: { RBXScriptConnection }? } = {}

local function onPlayerAdded(player: Player)
	local connections: { RBXScriptConnection } = {}
	connectionsByPlayer[player] = connections
	table.insert(connections, RunService.Heartbeat:Connect(function()
		-- Travail par joueur.
	end))
end

Players.PlayerAdded:Connect(onPlayerAdded)
-- Joueurs arrivés avant le chargement du script.
for _, player in Players:GetPlayers() do
	task.spawn(onPlayerAdded, player)
end

Players.PlayerRemoving:Connect(function(player: Player)
	local connections = connectionsByPlayer[player]
	connectionsByPlayer[player] = nil
	if connections then
		for _, connection in connections do
			connection:Disconnect()
		end
	end
	-- Différé : les autres handlers de départ lisent encore le personnage.
	task.defer(function()
		local character = player.Character
		if character then
			character:Destroy()
		end
	end)
end)
```

Test de fuite : état stable de référence → instantané A (Memory ou Scene Analysis) → répète l'action suspecte (20 respawns, 10 ouvertures d'un menu, une vague d'ennemis) → retour à l'état stable → instantané B. `LuaHeap`, `Signals` et `Instances` doivent revenir à la base, et la vue des instances non parentées doit être vide.

## 9. Pools d'objets

- Utile au-delà d'environ une apparition par seconde du même objet : balles, chiffres de dégâts, impacts, PNJ, cartes d'une longue liste. Créer et détruire un modèle avec `Humanoid` ou vêtements superposés coûte cher. Plafonne le pool, sinon c'est une fuite déguisée ; pré-chauffe-le avant une rafale prévisible.
- Réinitialise **tout** au retour : CFrame, vitesses, apparence, `CanCollide`/`CanTouch`/`CanQuery`/`Anchored`, connexions propres à l'usage, attributs **et** tags, enfants ajoutés (effets, sons, welds). Ne détruis jamais un objet rangé dans le pool.
- Un callback en retard qui ne garde que la référence peut rendre un objet déjà repris par quelqu'un d'autre : protège le retour par un jeton de bail.

```luau
--!strict
export type Pool<T> = {
	acquire: () -> (T, number),
	release: (item: T, lease: number) -> (),
}

local function createPool<T>(create: () -> T, reset: (T) -> (), destroy: (T) -> (), maxSize: number): Pool<T>
	local available: { T } = {}
	local leases: { [T]: number? } = {}
	local nextLease = 0

	local function acquire(): (T, number)
		local item: T = table.remove(available) or create() -- annoté : sans `: T`, le nouveau solveur refuse le `or`
		nextLease += 1
		leases[item] = nextLease
		return item, nextLease
	end

	local function release(item: T, lease: number)
		-- Double retour ou bail périmé : on ignore.
		if leases[item] ~= lease then
			return
		end
		leases[item] = nil
		reset(item) -- sort l'objet du monde et efface son état
		if #available < maxSize then
			table.insert(available, item)
		else
			destroy(item) -- pool plein : détruis le surplus, ne le lâche pas tel quel
		end
	end

	return { acquire = acquire, release = release }
end
```

Appelle `pool.acquire()` et `pool.release(item, lease)` avec un point : ces fonctions ne prennent pas `self`, et `pool:release(item, lease)` décale les arguments, si bien que le retour est ignoré sans bruit. Avec une classe à méthodes, écris `pool:acquire()` sans repasser `pool` : `pool:acquire(pool)` passe `self` deux fois (bug réel : un pool de 10 objets tombé à 4).

## 10. Réseau et réplication

Échelle, dans l'ordre : 1) ne pas envoyer (effets, tweens, réactions d'UI calculés là où on les voit) ; 2) envoyer moins souvent (cadence fixe, indépendante du framerate) ; 3) envoyer moins (deltas, identifiants, champs modifiés) ; 4) compresser (`buffer`). Les trois premiers sont gratuits et font l'essentiel du gain.

- Jamais de remote à chaque frame. Snapshots groupés vers 10 Hz, ou envoi sur changement ; un remote piloté par la souris reste sous 10 Hz.
- Compte la taille, la fréquence **et** les destinataires : `FireAllClients` multiplie par le nombre de joueurs. 200 octets vers 40 joueurs à chaque frame font 8 Ko par frame, soit environ 480 Ko/s à 60 FPS. Choisis les destinataires côté serveur : `FireClient` vers les joueurs concernés plutôt que `FireAllClients`.
- Réplique des résultats, pas la simulation : « le mob 12 est mort », pas les PV de chaque mob à chaque frame. Pas d'inventaire entier à chaque mise à jour : `{ added, removed }`, et le détail à la demande. Un instantané au join, puis des diffs : un diff qui porte 90 % des champs est un instantané déguisé, mesure leur vraie taille.
- Tableaux d'identifiants plutôt que dictionnaires bavards : chaque chaîne et chaque clé coûte des octets. Un remote qui porte plusieurs champs plutôt que plusieurs remotes ; un `RemoteEvent` plutôt qu'une `RemoteFunction` quand aucune réponse n'est attendue. Ne renvoie pas par remote une propriété déjà répliquée.
- Compte à rebours : envoie l'heure de fin une fois (base `workspace:GetServerTimeNow()`), le client décompte.
- Créer ou détruire des instances sur le serveur génère du trafic. Les instances purement visuelles vivent côté client. `SetAttribute` fréquent sur un objet répliqué : regroupe (une écriture par parcelle et par tick, une animation de couleur à 15 Hz plutôt qu'à chaque Heartbeat).
- **`UnreliableRemoteEvent`** pour le jetable et fréquent (visée, effets, positions) : 1 000 octets au plus, au-delà le message est jeté (Studio le signale, le jeu publié non). Les buffers sont compressés avant la mesure : mesurer avant d'envoyer ne prouve rien. Non ordonné, non garanti : envoie des valeurs **absolues**, jamais des deltas. Le critique passe en fiable.
- Au-delà d'environ 500 `FireServer` par seconde et par client, le moteur bride les remotes ; la limite est partagée par tous les remotes du même type (doc, 2026-10-18 ; détails dans securite.md). Un `RemoteEvent` sans handler met ses messages en file, puis les jette quand la file est pleine (erreur « Remote event invocation » dans Output) ; un non fiable les jette tout de suite.
- Pour les données remplaçables à haute fréquence, une sérialisation binaire quantifiée (CFrame compressé, flottants basse précision) avec un schéma versionné : bibliothèques dans paquets.md. Pas pour un état qui doit arriver exactement une fois.
- `SetNetworkOwner(nil)` sur les PNJ et pièces pilotés par le serveur arrête le va-et-vient de propriété près des joueurs, au prix de CPU serveur.
- DataStore : jamais de sauvegarde sur Heartbeat ; autosave périodique (vers 60 s pour un profil fait main, étalé entre les joueurs ; ProfileStore le fait seul, toutes les 300 s), au départ et dans `BindToClose` (voir donnees.md).
- HTTP : 500 requêtes par minute et par serveur vers l'extérieur. Pour Open Cloud, la référence `HttpService` annonce 2 500/min à part, la page « Rate limits » d'Open Cloud les compte dans les 500 : vise 500 en tout (open-cloud.md, section 11). Dans Studio, ce budget est partagé par toute la session, plugins compris : une rafale de tests peut déconnecter Rojo (mesuré 2026-08).

```luau
--!strict
-- LocalScript : envoie la visée à 10 Hz au plus, si elle a changé ou si une seconde a passé.
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local RunService = game:GetService("RunService")

local remotes = ReplicatedStorage:WaitForChild("Remotes")
local aimRemote = remotes:WaitForChild("Aim") :: UnreliableRemoteEvent

local SEND_INTERVAL = 0.1
local KEEPALIVE = 1 -- renvoie au moins chaque seconde : le dernier paquet a pu se perdre
local accumulated = 0
local sinceSent = 0
local sequence = 0
local lastDirection: Vector3? = nil

RunService.Heartbeat:Connect(function(deltaTime: number)
	accumulated += deltaTime
	sinceSent += deltaTime
	if accumulated < SEND_INTERVAL then
		return
	end
	accumulated %= SEND_INTERVAL
	local direction = workspace.CurrentCamera.CFrame.LookVector
	if lastDirection and lastDirection:FuzzyEq(direction, 1e-3) and sinceSent < KEEPALIVE then
		return
	end
	lastDirection = direction
	sinceSent = 0
	sequence += 1
	-- Valeur absolue + numéro croissant : le serveur ignore un paquet plus vieux que le dernier reçu.
	-- Côté serveur, OnServerEvent valide : types, norme proche de 1, cadence.
	aimRemote:FireServer(sequence, direction)
end)
```

## 11. Instances, physique, requêtes

- Chaque Instance coûte de la mémoire, et une BasePart aussi de la physique et du rendu. Moins de parts plus grandes, ou des meshes fusionnés. Une Part par voxel tient quelques centaines de cubes ; au-delà de quelques milliers, fusionne les voisins de même couleur ou génère un seul mesh (voir monde-3d.md).
- Ancre tout ce qui ne bouge pas : une part non ancrée entre dans le solveur. Au repos, le système de sommeil la met en veille, mais un contact ou un voisin qui bouge la réveille. Un assemblage soudé n'a besoin que d'une part ancrée. Ne réécris pas le `CFrame` d'une pièce physique à chaque frame : tu réveilles et combats le solveur.
- Décor : `CanCollide`, `CanTouch` et `CanQuery` à `false`. La géométrie de collision reste en mémoire même avec `CanCollide = false` : mets aussi `CollisionFidelity = Box`. `Hull` pour un objet convexe moyen, `Default` pour l'usage courant, `PreciseConvexDecomposition` seulement là où l'on se tient dans un creux, `Tunable` avec le curseur `CollisionPrecision` pour doser coût et précision (2026-09). Gros mesh complexe : collision faite de parts `Box` invisibles.
- **`CollisionFidelity`, `RenderFidelity`, `Model.LevelOfDetail`, `FluidFidelity` et `SmoothingAngle` ne s'écrivent pas depuis un Script de jeu** (« lacking capability PluginOrOpenCloud », mesuré dans Studio 0.742 ; `LevelOfDetail` ne se lit même pas). Règle-les dans la fenêtre Propriétés, la barre de commande ou un plugin. Au runtime, reconstruis le mesh : `local newPart = AssetService:CreateMeshPartAsync(part.MeshContent, { CollisionFidelity = Enum.CollisionFidelity.Box })` (dans un `pcall` : il lève une erreur en cas d'échec) puis `part:ApplyMesh(newPart)`. L'appel cède et refuse les meshes de plus de 2048 studs par axe.
- `Workspace.PhysicsSteppingMethod` (fenêtre Propriétés) : choisis `Adaptive` explicitement, qui simule chaque assemblage à 60, 120 ou 240 Hz, jusqu'à 2,5 fois plus vite selon la doc. `Fixed` simule tout à 240 Hz. Pour `Default`, la page de l'enum dit « Adaptive » mais le guide parle encore de 240 Hz par défaut (doc, 2026-10-18) : d'où le choix explicite.
- Humanoids : chacun fait tourner une machine à états sur le serveur. `Humanoid:SetStateEnabled(state, false)` pour les états inutiles, `AnimationController` pour un PNJ statique ou à mouvement maison, animations des PNJ jouées côté client. Des centaines de PNJ demandent des rigs sans Humanoid.
- Animation procédurale par `Motor6D.Transform` (dans `PreSimulation`), pas par `C0`/`C1`, qui reconstruit les clusters du rig à chaque frame. Des MeshParts skinnées qui bougent sans Humanoid reconstruisent aussi leurs clusters : donne un Humanoid au Model, ou garde-les immobiles.
- `Touched` est un déclencheur, pas un test de coup : il rate les objets rapides et se répète au repos. Pour des dégâts, de la monnaie ou une progression, interroge : `workspace:Raycast`, `Blockcast`, `Spherecast`, `Shapecast`, `GetPartBoundsInBox`, `GetPartBoundsInRadius`, `GetPartsInPart` (exact, plus cher). Les API à base de `Region3` sont dépréciées.
- Rejette d'abord ce qui est évident (distance au carré) avant de lancer un rayon. Pour une grande population, un index spatial (grille) : un simple filtre de distance reste en O(n). Tir hitscan : un raycast par tir ; projectiles simulés en maths, visuel recréé par les clients ; jamais une part physique par balle.
- Crée hors de l'arbre, règle les propriétés, puis `Parent` en dernier. Construis un groupe sous un Model non parenté, puis parente une fois. Beaucoup de déplacements d'un coup : `workspace:BulkMoveTo`.

```luau
--!strict
-- Paramètres créés une seule fois : affecter FilterDescendantsInstances copie la table.
local enemiesFolder = workspace:WaitForChild("Enemies")

local overlapParams = OverlapParams.new()
overlapParams.FilterType = Enum.RaycastFilterType.Include
overlapParams.FilterDescendantsInstances = { enemiesFolder }
overlapParams.MaxParts = 64

local function partsNear(position: Vector3, radius: number): { BasePart }
	return workspace:GetPartBoundsInRadius(position, radius, overlapParams)
end
```

## 12. Rendu, effets, UI

- **Instanciation des draw calls** : des MeshParts qui partagent le même `MeshContent`, la même `SurfaceAppearance` (ou texture) et le même matériau se dessinent en un seul appel. Importe chaque mesh une fois, puis duplique-le dans Studio ; importer une scène entière d'un bloc donne à chaque copie son propre ID et casse le regroupement. Audit rapide : liste le `Name` et le `MeshId` des MeshParts du Workspace ; un même objet présent sous plusieurs IDs ne se regroupe pas. Moins d'instances et moins de meshes uniques valent mieux que moins de triangles ; decals, transparence et lumières à ombres se regroupent mal.
- Mémoire des textures proportionnelle aux pixels, pas au poids du fichier : 1024² coûte 4 fois 512². Réutilise les mêmes IDs (mémoire partagée), utilise des sprite sheets (`ImageRectOffset`, `ImageRectSize`) et teinte avec `SurfaceAppearance.Color` plutôt que d'uploader des variantes.
- Overdraw : des surfaces semi-transparentes superposées (verre, feuillage, particules, UI) redessinent chaque pixel. Préfère une transparence de 0 ou 1. Pour les particules, coût ≈ débit × taille × opacité × surface couverte : réduis la taille et l'opacité avant le débit.
- Ombres : `CastShadow = false` sur les petites pièces répétées (pour les compter, multiplie un template de parcelle par le nombre de parcelles), `Shadows` coupé sur les lumières qui n'en ont pas besoin, portée et angle limités. « 4 à 6 lumières à ombres par zone » est une règle de praticien, pas une limite moteur. Éclairage et post-effets : voir monde-3d.md.
- LOD : `MeshPart.RenderFidelity` sur `Automatic` (moins de détail à 250 et 500 studs) ou `Performance` ; `Precise` sur beaucoup de meshes coûte cher. `Model.LevelOfDetail` (imposteurs lointains, valeur `SLIM` en 2026) et `Workspace.EnableSLIMAvatars` se règlent dans la fenêtre Propriétés. Les avatars SLIM couvrent les personnages joueurs R15, pas R6, les PNJ ni les proportions modifiées.
- Particules : environ 400/s par émetteur sur PC et 100/s sur mobile au plus (heuristique), `Lifetime` sous 20 s. Pour une rafale, un émetteur à `Rate = 0` et `:Emit(n)`. Coupe les émetteurs au-delà de 100 à 300 studs avec une boucle de culling par tags, pas un calcul de distance par émetteur à chaque frame. Les flipbooks coûtent de la mémoire, et de vieux téléphones les sautent.
- **Highlights** : le client n'en affiche qu'un nombre limité à la fois, et chacun coûte une passe. La doc donne 255 Highlights simultanés au plus, les désactivés comptant aussi (doc, 2026-10-18) ; un dépôt a constaté des disparitions au hasard dès 31 environ (l'ancienne limite). Un seul Highlight par Model de groupe, et un budget « les K plus proches » côté client.
- UI : `SurfaceGui.MaxDistance = 0` veut dire infini, mets 60 à 120 studs aux panneaux et étiquettes. Des milliers de GuiObjects cachés coûtent peu, mais animer un menu caché à chaque frame coûte cher. Conditionne chaque boucle à la visibilité. Ne crée jamais d'objet GUI par frame ; mets à jour sur changement. Liste longue : pool de cartes, ou défilement virtuel au-delà de 200 éléments. `ScreenGui.Enabled = false` plutôt que détruire puis recréer.
- `CanvasGroup` est une texture de rendu à part, de la taille du groupe : un seul tween sur le `CanvasGroup` parent plutôt qu'un par enfant, mais pas un `CanvasGroup` par bouton. `ViewportFrame` est un rendu 3D complet : n'anime que ceux qui sont visibles (`Visible` et `AbsoluteSize.X > 0`). Ni particules ni lumières n'y sont rendues.
- Mise en page qui se relance elle-même (écrire une taille change `AbsoluteSize`, qui relance l'écriture) : règle-la sur `RunService.Heartbeat:Once`, avec 3 passes au plus. Un vrai script a bouclé 80 fois.
- Audio : le nombre de voix simultanées est plafonné selon l'appareil ; réverbération et simulation acoustique sont les plus lourdes.

## 13. StreamingEnabled

- Actif par défaut dans les nouvelles places. Seuls les descendants de `Workspace` streament, jamais `ReplicatedStorage` ni `ReplicatedFirst`. Le serveur voit tout : ces règles concernent le client.
- `Workspace.StreamingEnabled` se règle dans Studio : un Script peut le lire, pas l'écrire (« lacking capability PluginOrOpenCloud », Studio 0.742). L'activer sur une place déjà publiée est un changement à haut risque : propose-le avec un plan de test.
- Réglages de la fenêtre Propriétés (non scriptables) : `StreamingMinRadius` (64 par défaut), `StreamingTargetRadius` (1024 par défaut), `StreamingIntegrityMode` (`PauseOutsideLoadedArea` pour le bas de gamme), `ModelStreamingBehavior` (`Improved` conseillé), `StreamOutBehavior`, `PredictiveStreamingMode` (2026-07 ; `Default` se comporte comme `Disabled`, `Enabled` précharge les destinations probables : remesure mémoire et bande passante). Un plugin peut appeler `Workspace:ApplyRecommendedStreamingSettings()`. N'utilise pas `StreamingPauseMode`, déprécié. Un rayon plus petit n'est pas forcément meilleur : mesure.
- Avec `Improved`, un Model streame dès qu'une de ses BaseParts est éligible, et ses descendants non-BasePart viennent avec ; un Model sans BasePart arrive au join et ne repart jamais. Un Folder n'est pas une unité de streaming : groupe les props dans des Models.
- Une instance qui sort du streaming est parentée à `nil`, pas détruite : ses références Luau restent valides si elle revient, mais les changements locaux de propriétés peuvent être perdus.
- Côté client, ne suppose jamais qu'une part du Workspace existe : pas de `workspace.Map.Chest` direct. `WaitForChild` avec timeout et gestion du `nil`, ou réaction à `ChildAdded` et `CollectionService:GetInstanceAddedSignal`. Ne mets pas en cache les parts d'une autre zone.
- `Model.ModelStreamingMode`, réglable par script : `Default` ou `Nonatomic` (comportement par défaut), `Atomic` (le modèle entier ou rien), `Persistent` (toujours chargé : rare et petit), `PersistentPerPlayer` avec `Model:AddPersistentPlayer(player)` et `RemovePersistentPlayer`. Bug réel : un template de parcelle en `Persistent` × 6 parcelles gardait environ 5 000 instances sur chaque téléphone ; `PersistentPerPlayer` pour le seul propriétaire a réglé le problème.
- `Player:RequestStreamAroundAsync(position, timeout)` avant un téléport ou une coupe de caméra : c'est un indice, pas une garantie. La méthode est sur `Player`, pas sur `Workspace`.
- Ne pilote pas le `CFrame` de parts qu'un joueur n'a pas chargées : de la bande passante pour rien. Pour couper le streaming pendant un match, prévois une place séparée.

```luau
--!strict
-- LocalScript : trouve la parcelle du joueur, qui peut arriver tard, partir et revenir.
local Players = game:GetService("Players")

local localPlayer = Players.LocalPlayer
local plots = workspace:WaitForChild("Plots")

local function bindPlot(plot: Instance)
	if plot:GetAttribute("OwnerUserId") ~= localPlayer.UserId then
		return
	end
	local button = plot:WaitForChild("UpgradeButton", 10)
	if not button then
		warn("bouton introuvable : hors de la zone streamée ?")
		return
	end
	-- Brancher l'interface ici. Au retour de la parcelle, ce handler repasse : nettoie l'ancien branchement.
end

local watched: { [Instance]: boolean? } = {}

local function onPlot(plot: Instance)
	-- Une parcelle sortie du streaming n'est pas détruite : une seule écoute par instance.
	-- La parcelle peut être attribuée après le join : réessaie quand le propriétaire change.
	if not watched[plot] then
		watched[plot] = true
		plot:GetAttributeChangedSignal("OwnerUserId"):Connect(function()
			bindPlot(plot)
		end)
		plot.Destroying:Once(function()
			watched[plot] = nil
		end)
	end
	bindPlot(plot)
end

plots.ChildAdded:Connect(onPlot)
for _, plot in plots:GetChildren() do
	task.spawn(onPlot, plot)
end
```

## 14. Appareils faibles et chargement

- Construis pour le bas de gamme puis monte. Choisis un appareil de référence nommé (Roblox cite par exemple l'Infinix Smart 9, le Moto G05, l'Oppo A18) et teste dessus pendant tout le développement : chauffe après 10 à 15 min de jeu actif, réseau cellulaire, taille des cibles tactiles, lisibilité. Rejoins à froid sur le pire appareil disponible : un joueur ne diagnostique pas un lag, il juge le jeu mauvais.
- Échelle de dégradation (pratique, non officielle) : densité et durée des particules → ombres → post-effets → détail des textures et meshes → distance de vue → instances non essentielles. Qualité adaptative avec hystérésis : moyenne sur une fenêtre, descendre vite, remonter lentement, choix manuel du joueur prioritaire.
- **Mode « graphismes bas » qui marche** : un module client coupe `ParticleEmitter`, `Fire`, `Sparkles`, `Smoke`, `Beam`, `Trail`, les lumières, les Highlights, `Lighting.GlobalShadows`, Bloom, SunRays et DepthOfField, et retire `Atmosphere` de `Lighting` (elle n'a pas de propriété `Enabled`). Il attrape aussi ce qui apparaît plus tard (`DescendantAdded`). Il range chaque valeur d'origine dans un attribut de l'instance pour tout restaurer, et épargne ce qui porte un attribut d'exclusion (indices de gameplay : boss, objectif). Ne cache jamais une information dont le joueur a besoin.
- Détection : `GetPlatform` n'est pas utilisable par un jeu, et « tactile sans clavier » classe les tablettes puissantes en bas de gamme. Déduis la capacité du temps de frame mesuré, combine avec `UserInputService.PreferredInput` et la taille de l'écran, et laisse le joueur forcer son choix.
- Chargement : `ContentProvider:PreloadAsync()` pour le **premier écran** seulement (logo, icônes du HUD, zone de départ), jamais `workspace:GetDescendants()`. N'attends pas `RequestQueueSize`. Offre un bouton « passer ». Mesure avec `os.clock()`, depuis un LocalScript de `ReplicatedFirst`, jusqu'à `game.Loaded`, et lis les plus grosses instances répliquées avec Studio Settings → Network → Print Join Size Breakdown.
- Ce que le client n'utilise pas va dans `ServerStorage`, pas dans `ReplicatedStorage` : chaque client télécharge et garde tout `ReplicatedStorage`.
- Meshes générés par IA : souvent 100 000 à 500 000 triangles ; décime sous 20 000 pour un personnage.

## 15. Pièges

- Optimiser sans mesure, ou mesurer dans Studio sur un PC puissant.
- Un tween serveur sur un objet du monde.
- Un remote tiré à chaque frame ; `FireAllClients` pour une donnée d'un seul joueur.
- Un `UnreliableRemoteEvent` de plus de 1 000 octets : jeté sans erreur en jeu publié. Des deltas sur un canal non fiable.
- `:Connect` dans `PlayerAdded` ou `CharacterAdded` sans nettoyage ; une table indexée par Player jamais vidée ; un personnage jamais détruit.
- `GetChildren`, `GetDescendants` ou `FindFirstChild(name, true)` à chaque frame.
- Écrire `CollisionFidelity`, `RenderFidelity`, `LevelOfDetail` ou `StreamingEnabled` depuis un Script : erreur de capacité.
- Du code client qui suppose tout le Workspace chargé ; `Persistent` sur un gros template répété.
- `ConnectParallel` hors d'un Actor ; `require` en phase parallèle.
- `local max = math.max` pour « optimiser » ; un `getfenv` qui désoptimise tout le script.
- Trop de Highlights ; des SurfaceGuis visibles à l'infini ; des menus cachés animés à chaque frame.
- Un pool sans plafond, un objet rendu deux fois, une remise à zéro incomplète.

## 16. Checklist

- [ ] Mesure avant et après notée : appareil, scénario, chiffre.
- [ ] Aucun travail lourd par frame ; ce qui peut tourner à 5–10 Hz le fait.
- [ ] Aucune allocation, recherche dans l'arbre ni concaténation dans un chemin chaud.
- [ ] Chaque connexion, thread et objet créé a son nettoyage, écrit dans la même modification ; tables par joueur vidées à `PlayerRemoving`.
- [ ] Effets visuels et tweens côté client ; le serveur envoie des intentions et des résultats.
- [ ] Remotes groupés, petits, sous 10 Hz ; non fiables sous 1 000 octets et en valeurs absolues.
- [ ] Décor ancré, collisions et requêtes coupées, `CollisionFidelity = Box` réglé dans Studio.
- [ ] Meshes dupliqués qui partagent leurs IDs ; textures sous 512² ; ombres coupées sur le petit décor.
- [ ] Code client tolérant au streaming (`WaitForChild` avec timeout, `ChildAdded`).
- [ ] Parallel Luau seulement si un profil le justifie, avec des scripts sous des Actors.
- [ ] Test de fuite revenu à la base (20 respawns, 10 ouvertures de menu).
- [ ] Testé sur un téléphone d'entrée de gamme, en session de 10 à 15 min.
- [ ] À la charge maximale : frames sous 16,67 ms, GPU Wait sous 2,5 ms ; Data Ping stable sous une simulation réseau de 100 à 150 ms.
