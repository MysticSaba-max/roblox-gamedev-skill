# Gameplay : personnages, combat, objets, PNJ, physique, entrées

À lire quand : tu écris ou corriges un système de jeu Roblox (personnage, animation, combat, inventaire, pets et œufs, PNJ, physique, véhicule, caméra, entrées, téléportation, effets visuels et sonores) ou tu poses les bases d'un genre (simulator, tycoon, obby, RPG, horreur, battle royale). La validation des remotes est dans securite.md, la sauvegarde dans donnees.md, les achats dans monetisation.md, les bibliothèques citées dans paquets.md.

## Sommaire

1. Méthode pour tout système
2. Personnages et respawn
3. Animations
4. Combat
5. Requêtes spatiales
6. Inventaire, objets, interactions
7. Pets et œufs
8. PNJ, pathfinding, quêtes
9. Physique et véhicules
10. Caméra
11. Entrées
12. Téléportation et serveurs réservés
13. VFX et audio en jeu
14. Motifs par genre
15. Pièges
16. Checklist

## 1. Méthode pour tout système

Avant d'écrire, réponds à neuf questions : quel fantasme ? quel état ? qui possède chaque état ? quelles entrées (clavier, tactile, manette, prompt, UI) ? quelles sorties (animation, son, UI, monde, sauvegarde) ? comment les autres le voient-ils ? qu'est-ce qui persiste ? que se passe-t-il si le joueur part, meurt ou spamme ? comment le prouver ? Elles évitent les systèmes qui marchent en solo et cassent à plusieurs.

- **Écris une machine à états explicite** pour tout ce qui a des phases : combat, dialogue, quête, véhicule, manche. Une table de transitions permises (`ALLOWED = { Idle = { Windup = true, Stunned = true }, … }`, consultée avant chaque changement) refuse les combinaisons absurdes, comme frapper en étant étourdi.
- **Mets la logique pure dans `shared/`** : formules de dégâts, tables de drop, courbes d'XP, prédicats de validation. Injecte l'aléatoire (`rng: Random`) pour tester avec une graine fixe.
- **Prévois le joueur arrivé en retard** : instantané complet à l'arrivée, puis mises à jour. C'est le bug multijoueur le plus fréquent.
- **Objets nombreux et identiques : un tag, un gestionnaire.** `CollectionService:GetTagged` + `GetInstanceAddedSignal` + `GetInstanceRemovedSignal`, réglages en attributs. Un script par porte ne passe pas à l'échelle et casse avec le streaming. Ignore, avec un `warn` qui donne son chemin, une instance taguée à qui manque un attribut. Test : fais sortir puis revenir la zone par le streaming, et vérifie une seule liaison par instance.
- **Choisis le niveau de risque par fonctionnalité, pas par jeu** (section 14). Un effet purement local (son, particule) n'a pas besoin du serveur : sur-architecturer est aussi une erreur.
- **Mode d'autorité serveur (2026).** Regarde `Workspace.AuthorityMode` dans la fenêtre Propriétés (non scriptable) avant d'écrire du mouvement ou de l'input. En mode `Server`, les entrées qui pèsent sur la simulation passent par l'Input Action System, avec l'`InputContext` sous le `Player`. La logique simulée passe par `RunService:BindToSimulation`, dans un ModuleScript chargé des deux côtés, avec `time()` et non `os.clock()`. Animations, sons et effets restent hors simulation (`PreRender`, ex-`RenderStepped`, qui lit le résultat) ; saute les effets ponctuels quand `RunService:IsResimulating()` est vrai. Ne garde pas une `AnimationTrack` comme identité, car un retour arrière peut la remplacer : retrouve-la par `Animator:GetTrackByAnimationId`. Sans ce mode (défaut), ce fichier s'applique tel quel (architecture.md ; doc, 2026-10-18).

## 2. Personnages et respawn

Chaque respawn crée un nouveau `Model` : tout ce qui pointe vers l'ancien est périmé.

- `Humanoid.Died` tire une seule fois ; le corps reste `Players.RespawnTime` secondes (5 par défaut). Relis `player.Character` après chaque yield, et traite le personnage déjà présent au démarrage.
- État par vie (buffs, cooldowns de capacité, hitboxes, pistes) : indexé par personnage, vidé à `CharacterRemoving`. État par joueur : vidé à `PlayerRemoving`. Sinon un buff survit à la mort, ou la mémoire fuit.
- `Workspace.PlayerCharacterDestroyBehavior = Enabled` (fenêtre Propriétés, non scriptable) détruit sans code l'ancien personnage à chaque remplacement (respawn) et l'objet `Player` au départ (doc, 2026-10-18).

Liaison sûre côté client (même motif côté serveur, par `Player`) :

```luau
--!strict
-- LocalScript : liaison qui survit aux respawns.
local Players = game:GetService("Players")

local player = Players.LocalPlayer
local generation = 0
local connections: { RBXScriptConnection } = {}
local bound: Model? = nil

local function unbind()
	generation += 1
	for _, connection in connections do
		connection:Disconnect()
	end
	table.clear(connections)
	bound = nil
end

local function bind(character: Model)
	if player.Character ~= character then
		return -- garde AVANT le démontage : un appel périmé ne défait rien
	end
	unbind()
	bound = character
	local myGeneration = generation
	local humanoid = character:WaitForChild("Humanoid", 10) :: Humanoid?
	if humanoid == nil or generation ~= myGeneration then
		return -- le personnage a changé pendant l'attente
	end
	table.insert(connections, humanoid.HealthChanged:Connect(function(health)
		print("PV :", health) -- branche ici l'UI et les systèmes du personnage
	end))
end

player.CharacterRemoving:Connect(function(character)
	if bound == character then
		unbind()
	end
end)
player.CharacterAdded:Connect(bind)
local current = player.Character
if current then
	task.defer(bind, current)
end
```

- **Ne restaure jamais une stat à une valeur codée en dur** (`WalkSpeed = 16` à la fin d'un boost) : recalcule depuis les bonus actifs, sinon tu écrases le pass VIP. Réapplique les bonus à chaque `CharacterAdded`.
- **`Humanoid:TakeDamage` respecte le `ForceField` d'apparition ; écrire `Health` l'ignore.**
- **Une seule autorité de mouvement à la fois** : locomotion par `Humanoid:Move` ou les contrôles par défaut, changement instantané par `AssemblyLinearVelocity`, contrôle continu par `LinearVelocity` ou `AlignPosition`. Deux contrôleurs qui luttent font trembler.
- **Un mode qui fige le personnage** (station, dialogue) sauve puis restaure exactement `WalkSpeed`, `JumpPower` ou `JumpHeight` et `AutoRotate`. Il sort sur tous les chemins : touche, bouton, mort, respawn, départ, éloignement. Le serveur possède l'attribut du mode et refuse les actions hors mode.
- Défauts : `WalkSpeed` 16, `JumpPower` 50 (`JumpHeight` 7,2), gravité 196,2 studs/s² ; 1 stud = 28 cm ; un avatar mesure environ 5 studs (classique) à 6,5 (Rthro) (monde-3d.md).
- Rigs : pas de noms de membres R15 en dur si R6 est supporté (attachments, tags, `Humanoid.RigType` seulement là où la topologie diffère). `HumanoidRootPart` n'est pas garanti sur un modèle maison : valide-le au spawn.
- Un `Tool` passe du `Backpack` au personnage : écoute `Equipped`, `Unequipped`, `AncestryChanged`. Apparence : côté serveur, `GetAppliedDescription()`, modifie la copie, `ApplyDescriptionAsync()` (`ApplyDescription` est déprécié) ; clone une description modèle avant de la modifier.
- Avatar d'un autre compte (statue du n° 1 du classement, morph) : `Players:CreateHumanoidModelFromUserIdAsync(userId)` donne un modèle prêt ; sinon `Players:GetHumanoidDescriptionFromUserIdAsync(userId)` (échelles et couleurs du corps comprises) puis `ApplyDescriptionAsync` côté serveur. Tout cède : sous `pcall`. Les versions sans `Async` sont dépréciées (définitions de septembre 2026). Respawn directement dans une tenue : `player:LoadCharacterWithHumanoidDescriptionAsync(description)`.
- **AFK** : `Player.Idled(secondes)` ne tire que côté client, environ 2 min après que le moteur classe le joueur inactif, puis toutes les 30 s ; Roblox déconnecte un joueur inactif depuis au moins 20 min (doc, 2026-10-18). Pour avertir ou envoyer en zone AFK, le client prévient le serveur par un remote à débit limité ; un état « actif » venu du client ne déclenche jamais de gain. Gains AFK calculés par le serveur, réduits et plafonnés (design-liveops.md, section 4).

Matrice de test : spawn, mort, Reset, respawns rapides, respawn pendant un chargement, outil équipé à la mort, départ pendant le setup, deux morts dans la même frame, chaque rig. Cherche les callbacks doublés, les pistes résiduelles et l'UI qui suit l'ancien Humanoid.

## 3. Animations

- **Charge par l'`Animator`** : `humanoid:FindFirstChildOfClass("Animator")`, ou celui d'un `AnimationController` pour un rig sans Humanoid. `Humanoid:LoadAnimation` et `AnimationController:LoadAnimation` sont dépréciés.
- Le moteur crée l'Animator du personnage joueur côté serveur : attends-le (`WaitForChild("Animator", 5)`), ne le crée jamais côté client (il ne répliquerait rien). Pour un PNJ ou un rig maison, crée-le côté serveur. Le modèle doit être dans le Workspace avant `LoadAnimation`, sinon l'appel lève une erreur.
- **Charge une fois par Animator et garde la piste** : `LoadAnimation` crée toujours une nouvelle piste (doc), donc recharger à chaque attaque les accumule (avertissement au-delà de 256 par Animator). Précharge les objets `Animation` avec `ContentProvider:PreloadAsync`.
- **Qui joue quoi** : le client pour son propre personnage (réactif, l'Animator réplique aux autres) ; le serveur pour un PNJ, sinon les autres ne voient rien (grande échelle : section 8).
- **Propriété de l'asset** : une animation ne joue que si elle appartient au propriétaire de l'expérience (utilisateur ou groupe) ou à Roblox. Une animation « invisible » vient d'un mauvais Animator, du mauvais côté d'exécution ou d'un asset non autorisé. N'invente aucun ID : `"rbxassetid://0"` en attente, et liste les animations à publier.

- Piste : `Play(fadeTime, weight, speed)` (défauts 0,1 / 1 / 1, fondu court entre états), `Stop(fadeTime)`, `AdjustSpeed`, `AdjustWeight` ; `WeightCurrent`, `WeightTarget` et `Speed` sont en lecture seule ; `Length` vaut 0 tant que l'asset n'est pas chargé (durée réelle = `Length / Speed`).
- **`Stopped`** = fin naturelle ou `Stop()` : enchaîne la suite ici. **`Ended`** = la piste n'affecte plus la pose, après le fondu : nettoie ici. `DidLoop` compte les cycles.

### Priorités et marqueurs

- `Core < Idle < Movement < Action < Action2 < Action3 < Action4`. `Core`, celle des animations par défaut, est la plus basse ; plusieurs skills affirment l'inverse, à tort. Locomotion en `Movement`, attaques en `Action`, superpositions spéciales au-dessus. À priorité égale, le poids partage l'influence.
- **Pas de masque d'articulations au runtime** : une attaque qui laisse les jambes courir n'a de clés que sur le haut du corps dans l'éditeur. Sinon sa priorité haute écrase les jambes.
- `track:GetMarkerReachedSignal("Impact")` synchronise son, VFX et fenêtre de hitbox (`KeyframeReached` est l'ancien système) ; ne sonde jamais `TimePosition` à chaque frame. Un marqueur ne tire que là où la piste joue, client OU serveur. Fais le minimum dans le handler et rends l'effet idempotent si la piste peut redémarrer.
- **Un marqueur est un indice de timing, jamais une preuve de coup** : le serveur revalide distance, état et fenêtre. Marqueur jamais reçu : nom mal écrit, ou animation pas republiée après l'ajout du marqueur.

### Remplacer les animations par défaut

Le LocalScript `Animate` du personnage porte des `StringValue` (`idle`, `walk`, `run`, `jump`, `fall`, `climb`, `swim`, `swimidle`, `toolnone`, `toolslash`, `toollunge`…) qui contiennent les objets `Animation`. Change leurs `AnimationId` côté serveur, une fois l'apparence chargée (`CharacterAppearanceLoaded`, comme la doc) :

```luau
--!strict
-- Script serveur : remplace la course par défaut à chaque apparition.
local Players = game:GetService("Players")

local RUN_ID = "rbxassetid://0" -- publie d'abord ton animation : n'invente aucun ID

local function applyAnimations(character: Model)
	local animate = character:WaitForChild("Animate", 10)
	local slot = animate and animate:FindFirstChild("run")
	local animation = slot and slot:FindFirstChild("RunAnim")
	if animation and animation:IsA("Animation") then
		animation.AnimationId = RUN_ID
	end
end

local function onPlayerAdded(player: Player)
	player.CharacterAppearanceLoaded:Connect(applyAnimations)
	local character = player.Character
	if character and player:HasAppearanceLoaded() then
		task.spawn(applyAnimations, character) -- joueur déjà là quand le script démarre
	end
end

Players.PlayerAdded:Connect(onPlayerAdded)
for _, player in Players:GetPlayers() do
	onPlayerAdded(player)
end
```

- La doc arrête aussi les pistes en cours (`animator:GetPlayingAnimationTracks()`, puis `Stop(0)`) pour que le changement se voie tout de suite. Vérifie dans Studio le nom exact des enfants de chaque emplacement (`WalkAnim`, `RunAnim`…). Avec Rojo, un `Animate` maison est un enfant direct de `StarterCharacterScripts` nommé `Animate`, sinon les deux scripts tournent.
- Procédural : `IKControl` pour les pieds sur les marches ou la tête qui suit (`Type`, `ChainRoot`, `EndEffector`, `Target`, `Pole`, `Weight`, `SmoothTime` ; pas de propriété `P`). Pour animer par code un objet tenu, écris `Motor6D.Transform` dans `RunService.PreSimulation` (ex-`Stepped`), après l'Animator : écrit plus tôt, il est écrasé.

## 4. Combat

Le client envoie une intention (« j'attaque », une direction, au plus une cible candidate comme indice), jamais un montant de dégâts ni une touche confirmée. Le serveur valide du moins cher au plus cher (échelle complète dans securite.md) : débit → type et forme, NaN compris → cooldown (`os.clock()`) → état (vivant, pas étourdi, pas en récupération) → origine annoncée proche de la racine connue du serveur (contre le tir à travers les murs) → portée et ligne de vue avec une tolérance de latence bornée → ressources (munitions, endurance, possession de la capacité).

- **Tolérance de latence bornée** : un raycast serveur exact « rate » à 150 ms de ping, un rembobinage illimité invite le lag-switch. Plafonne le rembobinage à 300–500 ms, ou élargis la tolérance dans une constante nommée (heuristique de terrain). Teste à 100–200 ms de latence simulée avec deux clients.
- **Dégâts depuis les stats du serveur**, par id d'objet, jamais lus sur le `Tool` du client. Le serveur est témoin du kill avant de récompenser ; crédit et butin se calculent une seule fois.
- **Prédiction** : le client joue animation, traceur et effet tout de suite ; le serveur confirme ; un rejet se réconcilie sans bruit. Chiffres de dégâts après confirmation seulement. Résultats en `RemoteEvent` fiable, traceurs et effets en `UnreliableRemoteEvent`.
- **Cooldowns côté serveur**, par joueur et par capacité, vidés à `PlayerRemoving` ; session en `os.clock()`, ce qui survit au rejoin en `os.time()`. Tampon d'une seule entrée : au-delà, c'est une macro. Logique à pas fixe (`Heartbeat` + dt accumulé) : 240 FPS ne donnent aucun avantage.
- Test d'abus : spamme le remote très au-dessus du rythme prévu ; le serveur doit appliquer exactement le nombre permis.

### Mêlée validée par le serveur

```luau
--!strict
-- Script serveur : attaque de mêlée décidée par le serveur.
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local attackRemote = ReplicatedStorage:WaitForChild("Remotes"):WaitForChild("Attack") :: RemoteEvent

local COOLDOWN = 0.6 -- secondes
local DAMAGE = 20 -- en vrai : stats serveur de l'arme équipée
local HITBOX_SIZE = Vector3.new(5, 6, 7) -- largeur, hauteur, portée (studs)
local lastAttack: { [Player]: number } = {}

attackRemote.OnServerEvent:Connect(function(player: Player)
	local now = os.clock()
	if now - (lastAttack[player] or 0) < COOLDOWN then
		return -- trop tôt : ignoré sans bruit
	end
	local character = player.Character
	if character == nil then
		return
	end
	local humanoid = character:FindFirstChildOfClass("Humanoid")
	local root = character:FindFirstChild("HumanoidRootPart")
	if humanoid == nil or humanoid.Health <= 0 or not (root and root:IsA("BasePart")) then
		return
	end
	lastAttack[player] = now

	-- Boîte placée avec la position que connaît le serveur, devant le personnage.
	local params = OverlapParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = { character }
	local boxCFrame = (root :: BasePart).CFrame * CFrame.new(0, 0, -HITBOX_SIZE.Z / 2)

	-- Une touche par cible : un personnage compte une quinzaine de parts.
	local hit: { [Humanoid]: boolean } = {}
	for _, part in workspace:GetPartBoundsInBox(boxCFrame, HITBOX_SIZE, params) do
		local model = part:FindFirstAncestorOfClass("Model")
		local target = model and model:FindFirstChildOfClass("Humanoid")
		if target and target.Health > 0 and not hit[target] then
			hit[target] = true
			target:TakeDamage(DAMAGE) -- respecte le ForceField d'apparition
		end
	end
end)

Players.PlayerRemoving:Connect(function(player)
	lastAttack[player] = nil
end)
```

Ici la hitbox est interrogée une fois, à l'arrivée du remote. Pour une hitbox active pendant toute l'animation, interroge sur plusieurs frames avec la même table `hit`. Variante plus stricte : le serveur joue la piste et lance la hitbox au marqueur « Hit » ; moins réactive, mais aucune fenêtre n'est décidée par le client.

### Réglages de départ (heuristiques de terrain)

| Réglage | Valeur |
|---|---|
| Phase active / récupération | 0,35 s / 0,4 s |
| Fenêtre de combo / combo max | 0,8 s / 4 coups, multiplicateurs 1,0 · 1,1 · 1,25 · 1,5 |
| Esquive / cooldown | 0,5 s d'invulnérabilité / 1,5 s |
| Étourdissement / blocage | 1 s / −80 % de dégâts |
| Parade | blocage lancé moins de 0,15 s avant l'impact (très serré au-delà de 100 ms de ping) |
| Plafonds | défense ≤ 0,9 ; bonus cumulés ≤ +100 % ; dégât minimum 1 ; critique ×1,5 |

Hitbox (portée × largeur × hauteur, studs) : poing ou dague 4–5 × 4 × 5 ; épée 6–8 × 5 × 6 ; espadon 8–12 × 7 × 7 ; lance 10–14 × 3 × 5 ; frappe au sol 8–10 × 10 × 8. Dégâts = base × arme × (1 + bonus) × (1 − défense) × critique × combo, arrondis, plancher 1, chaque terme plafonné : des bonus additionnés sans borne explosent.

### Recul, statuts, projectiles, ressenti

- **Recul** : `LinearVelocity` sur le `RootAttachment` de la cible, `MaxForce` finie, détruite après 0,2–0,3 s ; une force infinie projette hors de la carte. `ApplyImpulse` n'agit que du côté du propriétaire réseau : appelé par le serveur sur un joueur, il ne fait rien.
- **Statuts** : durée, empilement et nettoyage explicites, un seul propriétaire de la récupération. Ragdoll : la physique peut tourner chez le propriétaire, mais le serveur décide du début, de la durée et de la fin, sinon un client annule son propre étourdissement (section 9).
- **Équité** : chaque attaque a une réponse (esquive, blocage, parade, distance). Instantanée = faibles dégâts ; gros dégâts = télégraphiés, évitables, longue récupération. Pas de tête chercheuse à la fois rapide et inesquivable.
- **Projectiles** : une seule boucle pour tous ; à chaque frame, un raycast sur le segment parcouru (aucun mur traversé) ; fin au contact, à la portée max ou au bout d'une durée de vie plafonnée. Réplique un message compact (origine, direction, vitesse, id, heure serveur) et laisse chaque client simuler le visuel. Pool avec remise à zéro, jamais `Instance.new` dans la boucle de tir. Tête chercheuse sans physique : tourne la direction d'au plus `maxTurn * dt` radians par frame (`CFrame.fromAxisAngle`), vitesse conservée, en traitant le vecteur nul et la cible pile derrière. Bibliothèques : FastCast (projectiles), Raycast Hitbox et ShapecastHitbox (mêlée), ClientCast (à revalider), WCS (compétences) ; paquets.md.
- **Ressenti** : quatre couches simultanées (flash de la cible, particules, son d'impact à hauteur variable, petite secousse), plus fortes sur un critique. Secoue la caméra seulement si le joueur local est l'attaquant ou la victime, sinon tout le serveur tremble. Le son part au contact, pas au début du geste.

## 5. Requêtes spatiales

| Besoin | Outil |
|---|---|
| Ligne (tir, ligne de vue, sol) | `workspace:Raycast(origin, direction, params)` ; la longueur de `direction` est la portée |
| Volume approximatif (mêlée, zone) | `GetPartBoundsInBox`, `GetPartBoundsInRadius` : boîtes englobantes, requête rapide, faux positifs possibles |
| Volume exact | `GetPartsInPart(part, params)`, plus cher |
| Objet rapide ou épais | `Blockcast`, `Spherecast`, `Shapecast` (balayage) |
| Contact physique réel (piège, lave) | `Touched` + anti-rebond par personnage |

- Params : `FilterType = Enum.RaycastFilterType.Exclude` ou `Include` (`Blacklist` et `Whitelist` sont dépréciés), `FilterDescendantsInstances`, `CollisionGroup`, `MaxParts` (overlap), `IgnoreWater` (raycast). Règle `RespectCanCollide` exprès : il fait suivre à la requête `CanCollide` au lieu de `CanQuery`.
- **Rafraîchis les params réutilisés au respawn**, sinon le filtre exclut l'ancien corps et le tir touche le tireur.
- **Un balayage ne voit pas une part qui chevauche déjà sa forme au départ** : traite ce cas par une requête de volume. `Shapecast` : `cframeOverride` et `sizeOverride` sont documentés mais pas activés (mesuré Studio 0.743, 2026-10) ; prends `Blockcast` ou `Spherecast`.
- Ligne de vue : un rayon qui touche une partie de la cible (chapeau, main) = vue dégagée. Teste `result.Instance:IsDescendantOf(target)` et exclus le tireur, pas la cible.
- `CanCollide` = réponse physique, `CanTouch` = événements de contact, `CanQuery` = requêtes (désactivable seulement si `CanCollide` est faux). Aucun ne cache une part au client. Capteur : `CanCollide = false`, `CanTouch` et `CanQuery` vrais.
- Un raycast client rate ce qui n'est pas encore streamé. Grande bataille : ne teste la hitbox que contre les personnages proches.

Tir instantané vérifié par le serveur. Les types sont déjà contrôlés à l'entrée du remote (securite.md) ; remonte ensuite à la cible par `result.Instance:FindFirstAncestorOfClass("Model")`, puis `FindFirstChildOfClass("Humanoid")` :

```luau
--!strict
-- ModuleScript serveur : tir instantané avec les positions connues du serveur.
local MAX_RANGE = 300
local MAX_ORIGIN_OFFSET = 8 -- studs tolérés entre l'origine annoncée et la racine (8 à 10, securite.md)

return function(character: Model, root: BasePart, origin: Vector3, aim: Vector3): RaycastResult?
	-- « not (x <= max) » rejette aussi NaN, qui échoue à toute comparaison.
	if not ((origin - root.Position).Magnitude <= MAX_ORIGIN_OFFSET) then
		return nil
	end
	local length = aim.Magnitude
	if not (length > 1e-3 and length < math.huge) then
		return nil
	end
	local params = RaycastParams.new()
	params.FilterType = Enum.RaycastFilterType.Exclude
	params.FilterDescendantsInstances = { character }
	params.IgnoreWater = true
	return workspace:Raycast(origin, aim.Unit * MAX_RANGE, params)
end
```

## 6. Inventaire, objets, interactions

- **Le serveur possède l'inventaire ; le client n'en est qu'un miroir.** Chaque mutation passe par une fonction serveur qui vérifie possession, capacité et légalité avant d'écrire, puis se réplique en deltas (instantané complet à l'arrivée).
- **Définitions et instances séparées** : un catalogue statique partagé, indexé par id (nom, rareté, catégorie, stats, icône, `MaxStack`) ; une instance possédée ne stocke que `itemId`, `quantity` et ses métadonnées (durabilité, `uuid` par `HttpService:GenerateGUID(false)` pour un objet non empilable). Un rééquilibrage du catalogue s'applique ainsi à tous au prochain chargement ; un champ `version` dans les métadonnées fige une ancienne version.
- Quantités entières, positives, bornées : une quantité négative acceptée donne de l'argent infini. Un sac à trous ne se compte pas avec `#`. **Vérifie la place avant d'ajouter** ; débordement = refus avec message, ou boîte aux lettres à vider plus tard, jamais de perte silencieuse.
- Équiper : possession → compatibilité d'emplacement → objet en place renvoyé au sac (s'il y a de la place) → stats recalculées depuis toutes les pièces équipées → visuel cloné depuis `ServerStorage`, l'ancien détruit d'abord.
- Boutique de PNJ : article existant → monnaie suffisante → place suffisante → débit et octroi dans la même étape, sans yield. Jamais de prix calculé par le client.
- **Échange entre joueurs, le système le plus exploité** : machine à états serveur (offre → verrouillage des deux → confirmation des deux → échange atomique → journal). Tout changement d'offre annule les deux confirmations. Revalide la possession au moment du swap, sans yield ; persiste les deux côtés avant de relâcher le verrou ; jamais pendant un départ, une téléportation ou un chargement inachevé, jamais entre serveurs ; annulation à `PlayerRemoving`. Objets achetés en monnaie payante : `IsPaidItemTradingAllowed` (monetisation.md). Test : un client quitte juste après sa confirmation → rien n'est créé ni détruit.
- Objet tenu sans `Tool` (pas de case de hotbar) : soudé à la main par un `Motor6D`, parts non ancrées, `Massless`, sans collision ni requête.

- **Interactions** : `ProximityPrompt` pour « maintenir pour interagir » sur tous les appareils (`Triggered(player)`, `HoldDuration`, `ActionText`, `KeyboardKeyCode`, `GamepadKeyCode` ; défauts `MaxActivationDistance = 10` et `RequiresLineOfSight = true`). Une seule connexion à `ProximityPromptService.PromptTriggered` sert tous les prompts. `ClickDetector` pour un clic ; `Touched` seulement si le contact est la mécanique, avec anti-rebond par personnage sans `task.wait` bloquant (`CanTouch = false` n'est pas un cooldown).
- **Revalide côté serveur** distance, état et verrou de l'objet : un tricheur déclenche un prompt, un ClickDetector ou un DragDetector de n'importe où.
- Téléphone : objets assez gros pour le doigt ; si tu les agrandis, agrandis aussi la portée serveur et la distance du prompt ; un raté affiche « Rapproche-toi » plutôt que rien. Un pad de collecte est traversé en sprint entre deux pas physiques : teste aussi côté client à chaque `Heartbeat` et tolère côté serveur une petite avance de vitesse, plafonnée.

## 7. Pets et œufs

- **Un seul système pour tous les pets de tous les joueurs**, mis à jour par lots échelonnés : c'est souvent le plus gros coût client et serveur d'un simulator. Rendu côté client depuis un état compact répliqué à TOUS les clients (propriétaire, pet id, emplacement), sinon les autres ne voient rien ; aucun CFrame serveur par frame. Modèles en pool, ancrés, déplacés par `PivotTo` (ou `workspace:BulkMoveTo` pour beaucoup de parts d'un coup) : une physique par pet multiplie le coût. Plafonne les pets équipés et les pets rendus sur appareil modeste ; teste au maximum de pets × maximum de joueurs.
- **Tire côté serveur**, octroie, sauvegarde, puis annonce : sinon le joueur se déconnecte pendant l'animation pour retenter. L'animation ne décide jamais du résultat. Limite le débit des tirages.
- **Tire la rareté, puis le pet dans la rareté.** Un poids de rareté appliqué à chaque pet fausse tout : un œuf avec deux pets Common à poids 60 donne 75 % de Common et 0,625 % de Legendary au lieu des 60 % et 1 % affichés.
- **Les chances affichées sortent de la même table et du même calcul que le tirage.**

```luau
--!strict
-- ModuleScript partagé : le serveur appelle roll(), l'UI affiche odds(). Même table.
export type Rarity = "Common" | "Uncommon" | "Rare" | "Epic" | "Legendary"
export type Egg = { weights: { [Rarity]: number }, pets: { [Rarity]: { string } } } -- poids par RARETÉ

-- Boucles annotées `rarity: Rarity` : sinon l'ancien solveur élargit la variable en string.
local ORDER: { Rarity } = { "Common", "Uncommon", "Rare", "Epic", "Legendary" }

-- Chances réelles par rareté ; une rareté sans pet ne compte pas.
local function odds(egg: Egg): { [Rarity]: number }
	local total = 0
	for _, rarity: Rarity in ORDER do
		if #(egg.pets[rarity] or {}) > 0 then
			total += egg.weights[rarity] or 0
		end
	end
	local result: { [Rarity]: number } = {}
	for _, rarity: Rarity in ORDER do
		local weight = if #(egg.pets[rarity] or {}) > 0 then egg.weights[rarity] or 0 else 0
		result[rarity] = if total > 0 then weight / total else 0
	end
	return result
end

-- rng injecté : Random.new() en jeu, Random.new(graine) en test.
local function roll(egg: Egg, rng: Random): (Rarity, string)
	local chances = odds(egg)
	local draw = rng:NextNumber()
	local cumulative = 0
	local fallback: Rarity? = nil
	for _, rarity: Rarity in ORDER do
		local pool = egg.pets[rarity]
		if pool and #pool > 0 then
			fallback = rarity
			cumulative += chances[rarity]
			if draw < cumulative then
				return rarity, pool[rng:NextInteger(1, #pool)]
			end
		end
	end
	assert(fallback, "œuf sans aucun pet") -- filet contre l'arrondi flottant
	local pool = egg.pets[fallback]
	return fallback, pool[#pool]
end

return { odds = odds, roll = roll }
```

- Chance (Luck) : multiplie les poids des raretés hautes, puis recalcule les chances affichées pour CE joueur. Spectacle d'ouverture croissant avec la rareté, contenu de l'œuf montré, doublons permis, index de collection avec silhouettes pour l'inconnu.
- **Pitié** : dure (garantie au N-ième tirage), douce (chances qui montent après un seuil) ou montante (à chaque échec, remise à zéro sur gain). Un compteur par rareté, affiché au joueur (« 34 sur 50 »), sauvegardé, jamais remis à zéro en douce. Exemple : Rare ou mieux garanti après 50 tirages sans Rare.
- **Œuf payant = objet aléatoire payant**, dès que des Robux l'achètent ou financent la monnaie du tirage : toutes les issues et leurs chances AVANT l'achat ; un boost de chance ou une pitié vendus en sont aussi. Si `ArePaidRandomItemsRestricted` est vrai pour ce joueur, la divulgation ne suffit pas : chemin gratuit, achat direct garanti, ou offre masquée. Si `PolicyService` échoue, applique le cas le plus restreint. Cette politique n'est jamais un risque acceptable (monetisation.md).
- Progression : XP du niveau N vers N+1 = 50 × N^1,5 (heuristique), niveau max fixé. **Décide une fois si les bonus s'additionnent ou se multiplient** : trois pets à ×2,5 × 1,48 multipliés font environ ×50, et un pass « +3 emplacements » casse l'économie. Teste 100 000 tirages contre les chances affichées.

## 8. PNJ, pathfinding, quêtes

- **Comportement côté serveur** ; le client ne fait que l'animation et les effets. Racines non ancrées des PNJ en `SetNetworkOwner(nil)` : sinon un client proche les possède et peut les manipuler.
- Machine à états typée (`idle → patrol → chase → attack → return`, plus `flee` et `dead`) ; les transitions possèdent les changements de vitesse, le nettoyage de cible et l'annulation des trajets.
- **Un seul système de mise à jour pour toutes les entités**, jamais un script par PNJ : réflexion à 5–10 Hz, curseur tournant (`i % N == frame % N`), budget par tick (ex. 2 ms) étiqueté par `debug.profilebegin("NPC Think")`. Le CPU serveur est la première cause d'un serveur qui se dégrade en cours de partie.
- Détection par coût croissant : distance → cône de vision (`LookVector:Dot(direction)`) → raycast de ligne de vue ; l'ouïe est un signal séparé. Laisse : au-delà d'une distance du spawn, le PNJ rentre et se soigne (repères : poursuite 40 studs, attaque 5, laisse 60).
- Spawner : plafond dur (il refuse au lieu de mettre en file), apparition près des joueurs, IA dégradée au loin, despawn de ce que personne ne voit, `humanoid.Died:Once` → retrait du suivi puis destruction après l'animation de mort ; pool de rigs. Coupe les états d'Humanoid inutiles (`SetStateEnabled`), jamais `Dead` : `Died` ne tirerait plus ; pas d'Humanoid pour un mobile simple (`AnimationController` + `AlignPosition`).
- Animations de PNJ jouées par le serveur par défaut ; à grande échelle, réplique l'état et laisse chaque client animer les PNJ proches. Bots en groupe : réserve chaque objectif rare à un seul bot, sinon ils convergent tous dessus.

### Pathfinding

- `PathfindingService:CreatePath({ AgentRadius, AgentHeight, AgentCanJump, AgentCanClimb, WaypointSpacing, Costs })` ; clés de `Costs` = noms d'`Enum.Material` ou labels de modifiers, `math.huge` = interdit. `ComputeAsync` sous `pcall`, puis `path.Status == Enum.PathStatus.Success`. `Enum.PathWaypointAction` n'a que `Walk`, `Jump` et `Custom`.
- `Humanoid:MoveTo` abandonne seul après 8 s (`MoveToFinished` reçoit `false`). `path.Blocked(index)` : recalcule seulement si l'obstacle est devant, depuis la position actuelle, avec un nombre d'essais borné ; jamais d'appel récursif depuis `Blocked`.
- Recalcule sur événement (cible déplacée de plus de N studs, au plus toutes les 0,5–1 s) ; partage les waypoints entre unités qui vont au même endroit ; réplique un id de trajet et une progression, pas des CFrames. `WaypointSpacing = math.huge` supprime les points intermédiaires des lignes droites.
- `PathfindingModifier` sur une part ancrée non collidable : `Label` pour les coûts, `PassThrough = true` pour router à travers une porte. `PathfindingLink` entre deux attachments (téléporteur, bateau, échelle), action `Custom` traitée selon le label.
- Évite la géométrie collidable qui bouge sans cesse (régénération du navmesh). Debug : Visualization Options → Navigation Mesh. Le pathfinding serveur ne dépend pas du streaming. Bibliothèques : SimplePath, BehaviorTrees3 (paquets.md).

```luau
--!strict
-- ModuleScript serveur : suivre un chemin, avec repli et abandon gérés.
local PathfindingService = game:GetService("PathfindingService")

local function followPath(humanoid: Humanoid, root: BasePart, goal: Vector3): boolean
	local path = PathfindingService:CreatePath({
		AgentRadius = 2,
		AgentHeight = 5,
		AgentCanJump = true,
		Costs = { Water = 20, DangerZone = math.huge }, -- DangerZone : label d'un PathfindingModifier
	})
	local ok = pcall(function()
		path:ComputeAsync(root.Position, goal)
	end)
	if not ok or path.Status ~= Enum.PathStatus.Success then
		humanoid:MoveTo(goal) -- repli : ligne droite plutôt qu'un PNJ figé
		return false
	end

	local current, blocked = 1, false
	local connection = path.Blocked:Connect(function(index: number)
		blocked = blocked or index >= current -- seul un obstacle devant compte
	end)
	for index, waypoint in path:GetWaypoints() do
		current = index
		if blocked or humanoid.Health <= 0 then
			break
		end
		if waypoint.Action == Enum.PathWaypointAction.Jump then
			humanoid.Jump = true
		end
		humanoid:MoveTo(waypoint.Position)
		if not humanoid.MoveToFinished:Wait() then -- false : abandon du moteur après 8 s
			blocked = true
			break
		end
	end
	connection:Disconnect()
	return not blocked and humanoid.Health > 0 -- false : l'appelant recalcule, essais bornés
end

return followPath
```

### Dialogues et quêtes

- **Le serveur tient la session de dialogue** : le client envoie « je parle au PNJ X », puis l'index du choix ; le serveur garde le nœud courant et revérifie distance et prérequis. Impossible de sauter au nœud « récompense ». Arbres courts ; chaque nœud ne change l'état qu'à travers le service de quêtes.
- Caméra de dialogue : `Scriptable`, `CFrame.lookAt` depuis la tête du PNJ plus un offset stable, PNJ et joueur face à face, sans téléporter le joueur ; `Custom` restauré à la sortie.
- Quêtes : définitions partagées en données (`kill`, `collect`, `talk`, `visit`), progression serveur par une seule fonction paramétrée, états `Locked → Available → Active → ReadyToTurnIn → Completed`, récompense réclamée une seule fois et sauvegardée. Jamais de remote « CompleteQuest » tirée par le client.
- Texte de PNJ généré par IA (`TextGenerator` et `GenerateTextAsync`, API récente, 100 requêtes par minute au départ selon la doc du 2026-10-18) : filtre-le comme un texte de joueur et déclare-le au questionnaire de maturité (design-liveops.md).

## 9. Physique et véhicules

### Movers et contraintes

| Déprécié | Remplaçant |
|---|---|
| `BodyPosition` / `BodyGyro` | `AlignPosition` / `AlignOrientation` |
| `BodyVelocity` / `BodyAngularVelocity` | `LinearVelocity` / `AngularVelocity` (ou `Torque`) |
| `BodyForce`, `BodyThrust` | `VectorForce` |
| `RocketPropulsion` | `LineForce` + `AlignOrientation` |
| `Part.Velocity`, `SetPrimaryPartCFrame` | `AssemblyLinearVelocity`, `PivotTo` |

- Aller à un point : `AlignPosition` ; viser : `AlignOrientation` ; vitesse tenue : `LinearVelocity` (pour un coup ponctuel, `ApplyImpulse` côté propriétaire) ; hélice : `AngularVelocity` ; poussée : `VectorForce` ; aimant : `LineForce` avec `InverseSquareLaw`. `LineForce` s'applique au parent d'`Attachment0`, et son `MaxForce` n'agit que si `InverseSquareLaw` est vrai : sinon, borne `Magnitude` toi-même.
- Toute contrainte veut ses deux attachments (un seul = échec silencieux) et ne fait rien sur une part ancrée. **Dimensionne les forces sur `AssemblyMass`** (ex. `MaxForce = masse × workspace.Gravity × 4`). Sur-contraindre fait trembler. En migrant des BodyMovers, garde repère, limites, attachments et propriétaire, et mesure le comportement.
- Mécanique : `HingeConstraint` (porte, roue ; `Motor` = `AngularVelocity` + `MotorMaxTorque`, `Servo` = `TargetAngle` + `AngularSpeed` + `ServoMaxTorque`, butées par `LimitsEnabled`) ; `PrismaticConstraint` (ascenseur, tiroir ; `Servo` + `TargetPosition`) ; `SpringConstraint` (`FreeLength`, `Stiffness`, `Damping` ; `MinLength` et `MaxLength` exigent `LimitsEnabled` ; une `PrismaticConstraint` alignée guide une roue suspendue) ; `BallSocketConstraint` (ragdoll), `RopeConstraint`, `RodConstraint`, `WeldConstraint`, `NoCollisionConstraint`. Départs à régler : porte servo `ServoMaxTorque = 5000`, `AngularSpeed = 2` ; ressort `Stiffness = 5000`, `Damping = 500`.
- Assemblages : des parts soudées forment un seul corps (force, masse, propriétaire). Ancrer UNE part ancre tout et en fait la racine ; en ancrer PLUSIEURS scinde l'assemblage et coûte en perf. Un mécanisme lent s'endort ; une articulation motorisée le garde éveillé. Garde les unités Roblox (1 stud = 0,28 m, gravité 196,2 studs/s²) : les redéfinir casse le contrôleur de personnage. `WorldRoot:Simulate()` et le pas de physique manuel sont fermés aux scripts (2026-10) : ne les propose pas.

### Propriété réseau

- Un assemblage non ancré proche d'un joueur passe automatiquement à son client : réactif mais manipulable. **La propriété n'est pas une frontière de sécurité** : aucune décision sur la position d'une part possédée par un client.
- Serveur seulement : `SetNetworkOwner(nil)` (le serveur simule), `SetNetworkOwner(player)`, `SetNetworkOwnershipAuto()`, `GetNetworkOwner()`. Fixer un propriétaire coupe le transfert automatique. Une part ancrée n'a pas de propriétaire : `SetNetworkOwner` y lève une erreur (teste `CanSetNetworkOwnership()`).
- Au serveur : PNJ, pièges, plateformes, objets d'objectif, blocs de dropper. Au conducteur : son véhicule. Un même mécanisme a un seul propriétaire, sinon il tremble. Studio : Visualization Options → Network Owners.

### Plateformes, ragdoll, collisions

- **Pas de tween serveur pour une plateforme** : chaque pas est répliqué, et une part ancrée déplacée par CFrame ne transporte pas les joueurs. Plateforme physique : désancre-la, `AlignPosition` en `Enum.PositionAlignmentMode.OneAttachment` (`MaxForce = masse × gravité × 4`, `MaxVelocity` 20, `Responsiveness` 20), `AlignOrientation` en `OneAttachment` avec `RigidityEnabled`, puis `SetNetworkOwner(nil)` ; change `AlignPosition.Position` pour viser le point suivant. Décor purement visuel : tween côté client. Tapis roulant : part ancrée avec `AssemblyLinearVelocity` réglée.
- Ragdoll : remplace les `Motor6D` (sauf la racine) par des `BallSocketConstraint` posés sur leurs `C0` et `C1` avec `LimitsEnabled`, puis `humanoid:ChangeState(Enum.HumanoidStateType.Physics)` ; à la mort, `Humanoid.BreakJointsOnDeath = false`. Enregistre ce que tu crées et les moteurs que tu coupes : la sortie ne défait que ton travail, une seule fois, même si le personnage disparaît.
- Groupes de collision : `RegisterCollisionGroup`, `CollisionGroupSetCollidable(a, b, false)`, puis `part.CollisionGroup = "Nom"`. Création côté serveur ou à l'édition (le client peut seulement assigner), 32 groupes au plus, `IsCollisionGroupRegistered` avant `CollisionGroupSetCollidable` (qui lève sinon). Ces méthodes existent aussi sur `Workspace` (héritées de `WorldRoot`, réglage par monde) ; celles de `PhysicsService` ne sont pas dépréciées (doc, 2026-10-18). Écris la matrice des paires et vérifie-la dans Studio.

### Véhicules

- Sur `VehicleSeat:GetPropertyChangedSignal("Occupant")`, donne l'assemblage au conducteur (`Players:GetPlayerFromCharacter(humanoid.Parent)`, puis `seat:SetNetworkOwner(player)`) ; siège vide : `SetNetworkOwnershipAuto()`. Sinon le véhicule saccade à haute vitesse.
- Roues en charnière moteur, direction en servo, suspension ressort + glissière ; un cylindre s'étend le long de son axe X. Entrées minimales, bornées à [−1, 1], jamais une position cible brute à chaque frame.
- Anti-triche par plausibilité (vitesse max par classe) ; pour retirer le contrôle, `SetNetworkOwner(nil)`, car le mode automatique peut rendre le véhicule au même joueur. Temps de course et ordre des checkpoints mesurés par le serveur. Décide du sort du véhicule au changement de siège, à la mort et à la déconnexion ; teste à 100 ms de latence et plus.

## 10. Caméra

- **La caméra appartient au client** : le serveur dit quand, jamais comment. `CameraType = Enum.CameraType.Scriptable` pour prendre la main, sinon les scripts par défaut réécrivent le CFrame à chaque frame.
- **Restaure sur chaque chemin de sortie** (`CameraType = Custom`, `CameraSubject`, `FieldOfView` à 70, effets coupés) : mort, `CharacterRemoving`, changement de `CurrentCamera`, erreur, plus un chien de garde. Sinon le joueur perd le contrôle.
- Boucle maison : `RunService:BindToRenderStep(nom, Enum.RenderPriority.Camera.Value + 1, fn)` passe après la caméra par défaut ; empile plusieurs effets sur `+1`, `+2`, `+3`. Reconnecte à chaque respawn. N'écris le FOV que si personne ne l'a changé depuis ta dernière écriture. **Lissage indépendant du framerate** : `camera.CFrame:Lerp(cible, 1 - math.exp(-k * dt))` ; un `Lerp(…, 0.1)` par frame va deux fois plus vite à 120 FPS.
- `ScreenPointToRay(x, y)` tient compte de l'inset de la barre du haut (souris, toucher), `ViewportPointToRay` non ; les deux donnent une direction unitaire à multiplier par la portée. `CFrame.lookAt(position, cible)` dégénère si les points sont confondus ou si la cible est pile au-dessus (passe un vecteur `up`, ou `CFrame.lookAlong`).
- Première personne : `LocalTransparencyModifier` (local), pas `Transparency` ; souris verrouillée par `UserInputService.MouseBehavior = Enum.MouseBehavior.LockCenter`, tangage borné à ±89°. Spectateur : `CameraSubject` sur l'Humanoid de la cible, en `Custom`.
- Cinématique : photographie `CameraType`, `WalkSpeed` et `JumpPower`, puis restaure la photo. Grâce à un jeton, une cinématique plus récente reprend la photo déjà prise au lieu d'en refaire une, qui capturerait des zéros. Récompenses chronométrées par le serveur, jamais par l'animation client. Streaming : `ModelStreamingMode = Persistent` et attente bornée du modèle. Les autres joueurs voient au plus un court panoramique.
- Caméra orthographique : propriétés présentes mais pas activées, écritures ignorées (mesuré Studio 0.742, 2026-10). N'en dépends pas.

```luau
--!strict
-- ModuleScript client : secousse additive, sans dérive, coupée si le mouvement réduit est actif.
local GuiService = game:GetService("GuiService")
local RunService = game:GetService("RunService")

local token = 0

local function shake(intensity: number, duration: number)
	if GuiService.ReducedMotionEnabled then
		return
	end
	token += 1
	local myToken, start = token, os.clock()
	RunService:UnbindFromRenderStep("HitShake")
	RunService:BindToRenderStep("HitShake", Enum.RenderPriority.Camera.Value + 1, function()
		local elapsed = os.clock() - start
		if elapsed >= duration or myToken ~= token then
			RunService:UnbindFromRenderStep("HitShake")
			return
		end
		local strength = intensity * (1 - elapsed / duration)
		-- Translation seule, ajoutée au CFrame que la caméra par défaut vient d'écrire : pas de cumul.
		local offset = CFrame.new((math.random() - 0.5) * strength, (math.random() - 0.5) * strength, 0)
		workspace.CurrentCamera.CFrame *= offset
	end)
end

return shake
```

## 11. Entrées

L'entrée se lit toujours côté client (`UserInputService` et `ContextActionService` ne font rien côté serveur) ; le serveur reçoit des intentions et les valide. Choix : **Input Action System (IAS)** pour du code neuf multi-appareils et la réassignation (requis sous l'autorité serveur pour ce qui pèse sur la simulation) ; **`ContextActionService`** pour une action contextuelle avec bouton tactile automatique ; **`UserInputService`** pour un raccourci global, l'état brut ou la molette. Aucun n'est déprécié (2026-10) : passer à l'IAS est un choix de conception, pas une correction.

### Input Action System (sorti le 2026-06-11)

- `InputContext` : groupe d'actions (`Enabled`, `Priority`, `Sink` pour consommer l'entrée avant les contextes moins prioritaires, comme un inventaire ouvert qui bloque les touches de jeu).
- `InputAction` : `Type` = `Enum.InputActionType.Bool`, `Direction1D`, `Direction2D`, `Direction3D` ou `ViewportPosition` ; événements `Pressed`, `Released`, `StateChanged` ; `GetState()` pour une direction (un `Vector2` en `Direction2D`). `DisplayName` nomme l'action dans une UI de réassignation ; `PreferredBinding` (lecture seule) donne la liaison de l'appareil actuel, dont `DisplayName` et `DisplayImage` servent d'indice de touche.
- `InputBinding` : une entrée par famille d'appareils (`KeyCode`, `UIButton` pour un bouton tactile, `Up`/`Down`/`Left`/`Right` pour un composite ZQSD ou stick). Création par code possible : `Instance.new`, parent l'action (l'action ayant pour parent un `InputContext`).
- Noms inventés à ne jamais écrire : `InputActionService`, `InputActionBinding`, `ActionType`, `InputType`, `Activated`, `Deactivated`, `Enum.InputActionType.Button`.
- `Workspace.PlayerScriptsUseInputActionSystem` (type `Enum.RolloutState`) fait passer les contrôles par défaut sur l'IAS. Réglage de la fenêtre Propriétés, illisible et non modifiable par script (mesuré Studio 0.743) : demande à l'utilisateur de le régler. Si une action ne réagit pas : le contexte est-il `Enabled` ? Un contexte plus prioritaire avec `Sink` mange-t-il la touche ?

Arbre conseillé par la doc, créé à l'édition (sous l'autorité serveur, l'`InputContext` doit descendre du `Player`) :

```
ReplicatedStorage/Inputs (Folder)
  PlayContext (InputContext, Enabled)
    Sprint (InputAction, Type = Bool)
      Keyboard (InputBinding, KeyCode = LeftShift)
      Gamepad  (InputBinding, KeyCode = ButtonL3)
      Touch    (InputBinding, UIButton = bouton de sprint à l'écran)
```

```luau
--!strict
-- LocalScript : sprint par l'IAS ; le serveur vérifie l'endurance et règle WalkSpeed.
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local inputs = ReplicatedStorage:WaitForChild("Inputs")
local sprint = inputs:WaitForChild("PlayContext"):WaitForChild("Sprint") :: InputAction
local setSprint = ReplicatedStorage:WaitForChild("Remotes"):WaitForChild("SetSprint") :: RemoteEvent

sprint.Pressed:Connect(function()
	setSprint:FireServer(true) -- intention, pas une vitesse
end)
sprint.Released:Connect(function()
	setSprint:FireServer(false)
end)
```

### ContextActionService et UserInputService

- `BindAction(nom, handler, créerBoutonTactile, ...entrées)` : le handler reçoit `(nom, état, inputObject)` et renvoie `Enum.ContextActionResult.Sink` ou `Pass` ; les liaisons s'empilent (la plus récente gagne, `BindActionAtPriority` force l'ordre). `UnbindAction` à la sortie du contexte ; outil : lier à `Equipped`, délier à `Unequipped`. **Traite d'abord `End` et `Cancel`**, de façon idempotente : à l'annulation, `KeyCode` vaut `Unknown`.
- `InputBegan(input, gameProcessedEvent)` : ignore l'appui si `gameProcessedEvent` est vrai (chat, TextBox), mais **jamais le relâchement** : une garde maintenue doit toujours pouvoir se libérer.
- Mode d'entrée : `UserInputService.PreferredInput` (`Enum.PreferredInput.KeyboardAndMouse`, `Touch`, `Gamepad`) et son `GetPropertyChangedSignal` ; ne déduis pas le mobile de `TouchEnabled`. `player:GetMouse()` et `Mouse.Target` sont hérités : rayon depuis la caméra, avec un repli manette.

### Règles multi-appareils

- **Porte de gameplay unique** : aucune fenêtre ouverte, `UserInputService:GetFocusedTextBox() == nil`, `not GuiService.MenuIsOpen`, `GuiService.SelectedObject == nil`.
- **Annuler n'est pas relâcher** : un maintien (charge, lancer) s'annule sur perte de permission, `UserInputState.Cancel`, déséquipement, mort, perte de focus de la fenêtre, changement de mode d'entrée, manette déconnectée. Un relâchement périmé ne termine pas un maintien plus récent.
- Manette : `ButtonR2` déclenche `Tool.Activated` par défaut ; `Tool.ManualActivationOnly = true` coupe R2 ET le clic, donc règle-le selon `PreferredInput`. Vise depuis le centre de l'écran ; hotbar sur `ButtonL1` et `ButtonR1`. Zone morte radiale : sous le seuil `dz`, zéro ; au-dessus, `stick.Unit * (stick.Magnitude - dz) / (1 - dz)` pour garder la précision fine.
- Tactile : un bouton à l'écran par action (bouton CAS ou `UIButton` d'IAS), cibles ≥ 44 px, rien réservé au survol ni au clic droit. Chaque doigt suit son propre glissement, identifié par son `InputObject`. `JumpRequest` tire plusieurs fois par saut : anti-rebond.
- `DragDetector` : avec `RunLocally = true`, la position vient du client et ne se croit pas ; `RunLocally = false` fait faire le drag par le serveur.
- Test : appui, maintien, relâchement et annulation juste avant, pendant et après la fin d'un cooldown ; ré-appuis rapides.

## 12. Téléportation et serveurs réservés

- `TeleportService:TeleportAsync(placeId, players, options?)` est la méthode à utiliser, **côté serveur seulement** ; une demande du client passe par un RemoteEvent validé. Dépréciés : `TeleportToPlaceInstance`, `TeleportToPrivateServer`, `TeleportPartyAsync`, `TeleportToSpawnByName`, `ReserveServer` (remplacé par `ReserveServerAsync`). `Teleport` côté client existe encore : ne le signale pas comme bug, mais écris `TeleportAsync`.
- 50 joueurs au plus par appel, dans le même univers. **Ne marche pas dans Studio** : teste sur une expérience publiée, depuis l'application Roblox.
- `TeleportOptions` : `ShouldReserveServer`, `ReservedServerAccessCode`, `ServerInstanceId`, `SetTeleportData(data)`. Un seul mode à la fois : toute combinaison lève une erreur. **Un `TeleportAsync` réussi ne prouve pas l'arrivée** : marque « lancé » à la source, « arrivé » dans la destination.
- Serveur réservé : `ReserveServerAsync(placeId)` renvoie un code d'accès (valide indéfiniment) et un `PrivateServerId`. Détection : `game.PrivateServerId ~= "" and game.PrivateServerOwnerId == 0`. Les consoles sans cross-play peuvent tomber dans un autre serveur au même `PrivateServerId` : distingue par `game.MatchmakingType`.
- **Données de téléportation falsifiables** : elles passent par le client. Indices seulement (mode, préférence cosmétique) ; lecture serveur par `player:GetJoinData().TeleportData`. Pour une donnée qui compte, passe un jeton et lis le vrai contenu en MemoryStore (donnees.md). Ni Instance ni fonction dedans.
- **Sauvegarde avant de téléporter** et laisse le verrou de session passer à la destination (donnees.md) ; ne transporte jamais un état de personnage vivant.
- `pcall` + relances espacées ; `TeleportService.TeleportInitFailed` propose un nouvel essai ou renvoie en lieu sûr ; un joueur ne quitte la file qu'après le succès.
- Deux lobbies qui allouent le même match : chacun réserve un candidat et le publie par `UpdateAsync` sur la clé du match ; le premier enregistrement valide gagne et tout le monde l'utilise. Jamais `GetAsync` puis `SetAsync`.
- Transition : `SetTeleportGui` avant chaque téléportation, `GetArrivingTeleportGui()` dans ReplicatedFirst à l'arrivée ; `CustomizedTeleportUI` ne fait plus rien. Suivre un ami : `GetPlayerPlaceInstanceAsync(userId)` → JobId → `TeleportAsync` avec `ServerInstanceId` (impossible vers un serveur réservé).
- Dans la même place avec StreamingEnabled : `player:RequestStreamAroundAsync(position)` avant `PivotTo` (au mieux, sans garantie). `Player.PauseTeleports`, `TeleportOptions.ReservedServerId` et `TeleportSwitchServer` sont non documentés ou réservés : ne construis rien dessus.

```luau
--!strict
-- ModuleScript serveur : groupe vers un serveur réservé neuf, avec relances. Sauvegarde AVANT.
local Players = game:GetService("Players")
local TeleportService = game:GetService("TeleportService")

local function teleportGroup(placeId: number, players: { Player }): boolean
	if #players == 0 or #players > 50 then
		return false -- 50 joueurs au plus par appel
	end
	local options = Instance.new("TeleportOptions")
	options.ShouldReserveServer = true
	options:SetTeleportData({ mode = "duel" }) -- indice seulement : falsifiable
	for attempt = 1, 3 do
		local present: { Player } = {}
		for _, player in players do
			if player.Parent == Players then -- un joueur parti entre deux essais est retiré
				table.insert(present, player)
			end
		end
		if #present == 0 then
			return false
		end
		local ok, err = pcall(function()
			TeleportService:TeleportAsync(placeId, present, options)
		end)
		if ok then
			return true -- lancé, pas encore arrivé : confirme dans la destination
		end
		warn(`téléportation ratée ({attempt}/3) : {err}`)
		if attempt < 3 then
			task.wait(2 ^ attempt) -- pas d'attente inutile après le dernier essai
		end
	end
	return false
end

return teleportGroup
```

## 13. VFX et audio en jeu

### Effets visuels

- **Le serveur annonce, chaque client dessine** : type d'effet, position, ids, par un remote non fiable si l'effet est cosmétique. Chaque client respecte ainsi sa qualité graphique ; l'état d'un émetteur se réplique, ses particules non.
- Conçois par phases : avertissement lisible → contact rapide → pic bref → décroissance. Une couche = un rôle ; juge la silhouette sans le son ; spectacle dense réservé aux moments rares, réduit si le mouvement réduit est actif.
- **Rafale** : émetteur à `Rate = 0`, puis `emitter:Emit(n)`, par exemple au marqueur d'animation. `Enabled = false` laisse finir les particules vivantes, `Clear()` les efface. Jamais un émetteur créé par frame.
- Émetteur sur un `Attachment` : il émet le long de son +Y (tourne l'attachment pour viser). `Box`, `Sphere`, `Cylinder` et `EmissionDirection` exigent un parent `BasePart`. Place l'effet sur un attachment nommé ou par `model:GetBoundingBox()`, jamais avec un décalage Y fixe qui casse sur les petits rigs.
- Effet soigné = 2 à 5 émetteurs sur un même attachment (cœur lumineux, fumée, étincelles en `VelocityParallel`, halo). `LightEmission = 1` donne un rendu additif ; `ZOffset` ordonne les couches. `Beam` et `Trail` : deux attachments chacun ; un `Beam` sans texture n'est qu'une bande plate ; traînée d'épée en `Trail` de 0,2–0,4 s.
- `Highlight` : 255 simultanés au plus côté client, les suivants sont ignorés sans message (doc, 2026-10-18 ; l'ancienne doc disait 31, et un retour de terrain voit des contours disparaître dès ~31) : plafonne aux plus proches. Pool des effets fréquents avec remise à zéro complète ; budgète la vue simultanée (plusieurs joueurs à l'écran, mobile en qualité basse). Flipbooks (2×2 à 8×8 avec marges, 30 images/s au plus) : lourds en mémoire.
- Juge les particules en mode Play, pas en édition. `Sparkles`, `Fire` et `Smoke` ne sont que des placeholders.
- Tweens : `Completed` tire aussi à l'annulation (vérifie `Enum.PlaybackState.Completed`) ; deux phases sur la même propriété avec `DelayTime` s'annulent ; ne tweene pas vers le CFrame d'un modèle si un autre script a pu déplacer l'objet (stocke une pose de repos). Flash de coup : mémorise la vraie couleur une fois, ou tweene un `Highlight`, sinon deux coups rapprochés figent la teinte.

Points de départ, à régler à l'œil :

| Effet | Rate | Lifetime (s) | Speed | Réglages clés |
|---|---|---|---|---|
| Feu | 80 | 0,4–0,8 | 3–6 | taille 0,5 → 1,5 → 0, jaune → orange → rouge sombre, `LightEmission` 1, accélération +Y 4 |
| Fumée | 30 | 2–4 | 1–3 | taille 1 → 5, gris, transparence 0,5 → 1, accélération +Y 2 |
| Impact | 0 puis `Emit(20)` | 0,2–0,5 | 8–15 | `SpreadAngle` 180, taille 0,5 → 0, `Drag` 5 |
| Pluie | 300 | 0,8–1,2 | 40–60 | `VelocityParallel`, accélération −80, grande part au-dessus de la zone |

### Audio

- **`Sound`** (+ `SoundGroup`, `SoundEffect`) : simple, très bien pour l'UI, la musique et les effets ; ne réécris pas du code qui marche. **Graphe audio** (`AudioPlayer`, `AudioEmitter`, `AudioListener`, `AudioDeviceOutput`, `Wire`, effets `Audio*`) : préféré pour le neuf en 3D, requis pour la voix (`AudioDeviceInput`). 2D : `AudioPlayer → Wire → AudioDeviceOutput` ; 3D : `AudioPlayer → Wire → AudioEmitter` sur une part, plus `AudioListener → Wire → AudioDeviceOutput`. Sans `Wire` (`SourceInstance` → `TargetInstance`), rien ne sort.
- API du graphe à ne pas inventer : l'atténuation se règle par `emitter:SetDistanceAttenuation({ [distance] = volume })`, pas par une `NumberSequence` ; l'audibilité se lit par `GetAudibilityFor`. La propriété d'asset d'`AudioPlayer` est `Asset` (`AudioContent` est masquée, `AssetId` est dépréciée ; doc, 2026-10-18).
- `Sound` sous une part = son 3D (volume sur une part bloc, sphère ou cylindre ; point sur un attachment) ; sous `SoundService` = global. **Règle toujours `RollOffMaxDistance`** : 10 000 studs par défaut, le son traverse la carte ; 40–400 est un bon départ pour un effet. `RollOffMode = InverseTapered` suit `Inverse` de près, puis `LinearSquare`, et s'éteint en douceur à `RollOffMaxDistance`. Avec `Inverse` (le défaut), une petite distance max coupe le son net ; `Linear` et `LinearSquare` descendent jusqu'à zéro (doc, 2026-10-18). Au-delà de `Volume = 1`, le son peut saturer : monte plutôt le groupe.
- Groupes `Music`, `SFX`, `UI`, `Ambience` sous un groupe maître, réglés et sauvegardés dans les options. Ducking : `CompressorSoundEffect` sur la musique avec `SideChain` vers le groupe des effets, ou tween du volume.
- Effet ponctuel : joué par le client après un message du serveur (latence), cloné d'un modèle préchargé, posé sur un attachment à la position voulue (source ponctuelle 3D), hauteur variée de quelques % (0,95–1,08), anti-empilement. Ajoute un filet `Debris:AddItem`, car `Ended` ne vient jamais si le chargement échoue. Précharge les sons critiques (`ContentProvider:PreloadAsync`) en début de chapitre, pas au milieu d'une frayeur.
- Zones musicales : parts taguées avec attributs (`MusicKey`, `Priority`, `FadeSeconds`), position testée à intervalle et non à chaque frame (ou ZonePlus, paquets.md) ; la zone la plus prioritaire gagne, fondu croisé. La musique de boss suit l'état serveur du boss.
- Confidentialité audio (depuis 2022) : un ID tiers qui ne joue pas est presque toujours un problème de permission. Utilise tes propres uploads ou l'audio du Creator Store ; n'invente aucun ID ; si l'utilisateur refuse un son, laisse l'emplacement vide. Import : `.mp3`, `.ogg`, `.wav`, `.flac`, une seule piste, moins de 20 MB et de 7 min, 48 kHz au plus (doc, 2026-10-18).
- Accessibilité : un indice sonore important a toujours un équivalent visuel.

## 14. Motifs par genre

Choisis le profil de risque par fonctionnalité. Les chiffres ci-dessous viennent de modèles de genre et de retours de terrain : des points de départ, pas des normes Roblox.

| Genre | Boucle | Risque dominant | Où investir |
|---|---|---|---|
| Simulator | collecter → améliorer → zones → rebirth | intégrité des données et de l'économie | verrou de session, maths serveur, grands nombres, une boucle pour pets et générateurs |
| Tycoon | gagner → acheter → construire → automatiser → prestige | économie, flot d'objets | parcelles, placement validé, droppers plafonnés |
| Obby | tenter → échouer → comprendre → réussir | aucun majeur : la sur-ingénierie | checkpoints, respawn rapide, difficulté juste |
| RPG | quête → combat → butin → niveau → zone | complexité de l'état | schéma typé d'abord, quêtes en données, streaming |
| Horreur | explorer → tension → peur → survivre | perf contre fidélité | éclairage par zone, audio, rythme |
| Battle royale | largage → pillage → survie → combat | latence et triche | hitbox serveur, matchmaking, manche robuste |
| Combat, battlegrounds | engager → combo → contre → victoire | ressenti contre autorité | états serveur, une entrée tamponnée |
| Tower defense | vagues → unités → améliorations | CPU serveur | une boucle pour toutes les unités, chemins partagés, vagues en données |
| Course | piloter → checkpoints → classement | propriété réseau | conducteur propriétaire, temps serveur |
| Social, hangout | se montrer → interagir | densité de joueurs | profil à 30+ avatars, TextChatService, serveurs réservés pour les salons privés |

- **Simulator**
  - Collecte : gain de base 1, cooldown 0,2 s par joueur, portée 15 studs, vérifiés par le serveur ; un nœud récolté est épuisé pour tout le serveur, puis réapparaît (3 s). Collecte avant 5 s, amélioration achetable avant 30 s, première éclosion avant 2 min.
  - Zones à coût ×10 et rendement ×5 (10 000 → 100 000 → 1 M → 10 M), porte suivante visible depuis le spawn, jamais plus de 10–15 min de jeu actif sans pouvoir avancer. Améliorations : coût = base × multiplicateur^niveau (ex. 50 × 1,35^n).
  - Rebirth : exigence 10 000 × 2,5^n contre multiplicateur 1 + 0,1 × n. Bouton visible seulement si c'est possible, confirmation qui dit ce qui saute et ce qui reste, retour en zone 1, accélération qui se sent.
  - Gains hors ligne annoncés au retour, plafonnés à environ un jour. Au-delà de 2^53, les nombres perdent leur précision : stocke mantisse et exposant (luau.md).
- **Tycoon**
  - Une parcelle par joueur, au moins autant que `Players.MaxPlayers`. Téléporte à `CharacterAdded`, pas à `PlayerAdded` (le personnage n'existe pas encore), ou utilise `Player.RespawnLocation` sur un `SpawnLocation` de parcelle.
  - Le client montre un aperçu ; le serveur valide (propriété de l'objet et de la parcelle, bornes, grille, collision, plafond, coût) et crée l'objet. Sauvegarde `{itemId, position relative, rotation}` relatifs à l'origine de la parcelle ; calcule le pire cas contre les 4 Mo d'une clé (donnees.md).
  - Droppers : ~50 blocs au plus par parcelle, ~15 s de vie, blocs en `SetNetworkOwner(nil)`, encaissement groupé, collecteurs vraiment branchés. Paliers de coût 100–500 $ (0–10 min) → 2 500–15 000 $ (15–45 min) → 50 000–250 000 $ (1–3 h) → 0,5–2 M $ (3–8 h) ; revenu par seconde affiché ; prestige d'environ +25 % par niveau, seuil qui grandit plus vite que le bonus.
  - « Une plaque, un mur qui apparaît » fait daté : ajoute des choix, de l'automatisation et du social.
- **Obby**
  - Le serveur valide que l'étape N suit N−1 ; respawn au checkpoint en moins de 1 s (`RespawnLocation`, ou `PivotTo` à `CharacterAdded`). Obstacles par tag ou préfixe (mortel, mouvant, qui disparaît, tournant, tapis), réglés par attributs, un seul gestionnaire ; plateformes physiques ou côté client.
  - Les 10–20 premières étapes enseignent une mécanique à la fois ; décor ou mécanique qui change tous les 10–20 niveaux ; dents de scie avec des « boss ».
  - Chrono serveur (`os.clock()`) : diffuse une fois l'heure de départ (`workspace:GetServerTimeNow()`) et laisse le client afficher. Streaming : `ModelStreamingMode = PersistentPerPlayer` + `Model:AddPersistentPlayer(player)`, ou `RequestStreamAroundAsync` avant la téléportation.
  - « Passer l'étape » se défend si la difficulté n'est pas gonflée exprès. Jamais de bloc mortel invisible ni de saut au pixel sans prévenir.
- **RPG**
  - Niveaux 1–5 dans les 20 premières minutes, ralentissement net vers 10 ; une courbe 100 × 1,5^(n−1) devient injouable après ~15 (récompenses qui grandissent avec le niveau, ou facteur plus doux).
  - Défense en ratio plafonné (≤ 0,9), sinon « tout en DEF » rend invincible ; rendements décroissants par stat. Raretés d'équipement ×1,0 / 1,3 / 1,7 / 2,2 / 3,0.
  - Zones à niveau minimum vérifié par le serveur, PNJ avec laisse, dialogues tenus par le serveur (section 8). Un tutoriel de 20 minutes suffit à tuer un RPG sur Roblox.
- **Horreur**
  - Monstre plus rapide que le joueur (poursuite 22 contre 16) mais aveugle derrière les murs : vue 60 studs, demi-angle 55°, ouïe 30 studs ×1,5 si le joueur court. Mesure la vitesse réelle de la racine, pas `WalkSpeed` réglé par le client. Cachettes taguées ; le monstre fouille puis abandonne.
  - Jumpscare côté client, pour la victime seulement ; la mort est décidée par le serveur après l'effet. Au moins 30 s entre deux jumpscares, des fausses peurs, des salles sûres : l'ambiance compte plus que l'IA.
  - Du début à la fin : lumière ambiante 0,8–1,2 → 0,02–0,1, brouillard 300–500 → 40–80 studs, monstre 8–10 → 20–24 studs/s. Ambiance à 0,2–0,4, stingers nettement plus forts. Questionnaire de maturité honnête (design-liveops.md).
- **Battle royale et jeux par manches**
  - Machine à états de manche (`Lobby → Waiting → Countdown → Deploying → InProgress → Victory → Cleanup`), id de génération, une seule portée de nettoyage. Élimination reliée à `Humanoid.Died` ET à `PlayerRemoving`, sinon la manche ne finit jamais. Matrice : survivants, éliminés, spectateurs, arrivées en cours de manche, départs, bots, à chaque phase.
  - Zone : le cercle suivant reste dans le précédent (décalage ≤ R − r ; distance × √aléatoire pour une répartition uniforme), interpolation smoothstep α²(3 − 2α). Diffuse une fois départ, cible, heure de début (`GetServerTimeNow`) et durée : chaque client calcule le même cercle, le serveur garde le sien pour les dégâts.
  - Butin pondéré 40 / 30 / 18 / 9 / 3, distribution vérifiée sur 100+ manches. Parties de 8–15 min. Lobbies par niveau, bots, Elo (K = 32, écart toléré 200 élargi de 50 toutes les 15 s d'attente). Départ immédiat si un joueur suffit.
  - Équipes : un `Team` par équipe sous le service `Teams`, avec un nom et un `TeamColor` unique (affichés dans la liste des joueurs). Par défaut, un joueur rejoint l'équipe `AutoAssignable` la moins remplie ; décoche `AutoAssignable` pour répartir toi-même côté serveur (`player.Team = Teams:FindFirstChild("Blue") :: Team`). Spawn d'équipe : `SpawnLocation.Neutral = false` et `SpawnLocation.TeamColor` égal à celui de l'équipe ; ce `TeamColor` règle l'accès, pas la couleur affichée (doc, 2026-10-18).
  - N'emprunte aucune marque d'un autre jeu (« Victory Royale »).

## 15. Pièges

- Animations chargées par `Humanoid:LoadAnimation` ou Animator créé côté client (rien ne réplique) ; `Core` prise pour la priorité la plus haute (c'est la plus basse).
- Garder `Character`, `Humanoid` ou des `RaycastParams` en cache sans les rafraîchir au respawn.
- Cible, montant de dégâts ou point d'impact décidés par le client ; hitbox sans déduplication (quinze touches par personnage).
- `Touched` comme détecteur universel, ou sur des parts possédées par un client pour des dégâts.
- Tween serveur d'une plateforme, CFrame réécrit à chaque frame par deux scripts, `SetNetworkOwner` sur une part ancrée (erreur), `SetNetworkOwnershipAuto()` pour retirer un véhicule à un tricheur.
- Poids de rareté appliqués par pet : chances affichées fausses, non conformes si l'œuf est payant.
- `MoveTo` sans gérer l'abandon à 8 s ; `path.Blocked` qui rappelle la fonction de suivi.
- `CameraType = Scriptable` jamais restauré ; secousse cumulative ou jouée par tous les clients.
- Ignorer le relâchement quand `gameProcessedEvent` est vrai : une garde reste bloquée.
- Tester une téléportation dans Studio, ou faire confiance aux données de téléportation.
- `RollOffMaxDistance` laissé à 10 000 ; IDs audio ou d'animation inventés.
- API inventées vues dans des skills : `InputActionService`, `IKControl.P`, `Enum.PathWaypointAction.Climb`, `AudioEmitter.DistanceAttenuation` en `NumberSequence`, `AnimationClipProvider:LoadAnimationClipAsync` (c'est `GetAnimationClipAsync`). Liste complète dans pieges.md.

## 16. Checklist

- [ ] Les neuf questions ont une réponse ; l'état précieux appartient au serveur.
- [ ] Liaison au personnage sûre au respawn ; état par vie vidé à `CharacterRemoving`, par joueur à `PlayerRemoving`.
- [ ] Animations chargées une fois par l'Animator, assets possédés par l'expérience, priorités justes, nettoyage sur `Ended`.
- [ ] Combat : intention seule côté client, validation serveur dans l'ordre, cooldowns serveur, hitbox dédupliquée, tolérance de latence bornée, requêtes filtrées et params rafraîchis.
- [ ] Inventaire : catalogue + instances, quantités bornées, place vérifiée avant l'ajout, échange atomique.
- [ ] Œufs : rareté puis pet, chances affichées = chances réelles, pitié visible, `PolicyService` et divulgation si payant.
- [ ] PNJ : une boucle échelonnée, plafond de spawn, pathfinding avec repli et abandon géré, racines au serveur.
- [ ] Physique : movers modernes, forces dimensionnées sur la masse, propriétaire réseau décidé pour chaque assemblage.
- [ ] Caméra restaurée sur chaque chemin de sortie ; lissage indépendant du framerate ; mouvement réduit respecté.
- [ ] Entrées testées au clavier-souris, au tactile et à la manette, annulation comprise.
- [ ] Téléportation testée en jeu publié : sauvegarde avant départ, relances, arrivée confirmée à destination.
- [ ] Effets annoncés par le serveur et dessinés par le client ; `RollOffMaxDistance` réglé ; volumes par groupe.
- [ ] Playtest à 2 clients ou plus avec 100–200 ms de latence simulée, console Output relue.
