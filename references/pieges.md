# Pièges : vérifier avant d'écrire, erreurs des IA, diagnostic

À lire quand : tu vas écrire une API, une propriété, un enum, un paquet ou une commande dont tu n'es pas sûr à 100 % ; tu relis du code Roblox ; quelque chose « ne marche pas ». Sur Roblox, la mémoire d'entraînement se trompe souvent et l'API change chaque semaine : vérifie, cite ta source, ou écris « non vérifié ».

## Sommaire

1. Vérifier une API ou un paquet avant de l'écrire
2. API inventées, mal nommées ou inutilisables
3. API dépréciées et leur remplaçant
4. Anti-patterns récurrents des IA
5. Faux positifs : ne pas « corriger » du code juste
6. Pièges d'environnement et d'outillage
7. Diagnostic express « ça ne marche pas »
8. Checklist avant de dire fini

---

## 1. Vérifier une API ou un paquet avant de l'écrire

Trois questions différentes : le membre existe-t-il ? Un script de jeu a-t-il le droit de s'en servir ? Se comporte-t-il comme tu le crois ? Chaque source ne répond qu'à une partie.

| Source | Répond à | Comment |
|---|---|---|
| Index pour agents | Où est la bonne page | `https://create.roblox.com/docs/llms.txt`, puis `…/docs/reference/engine/llms.txt` (moteur) ou `…/docs/cloud/llms.txt` et `…/docs/cloud/openapi.json` (Open Cloud). Jamais `llms-full.txt` : des dizaines de Mo. |
| Page en `.md` | Sens, paramètres, retour | Ajoute `.md` à l'URL : `…/docs/reference/engine/classes/Animator.md`, de même `enums/<Nom>.md`, `datatypes/<Nom>.md`, `libraries/<nom>.md`. Les valeurs d'un enum forment une liste fermée : lis-la. |
| YAML de `Roblox/creator-docs` (GitHub) | Source brute des pages | `content/en-us/reference/engine/classes/<Classe>.yaml`, lu en brut sur `raw.githubusercontent.com` : champs `security`, `tags`, `deprecation_message`. |
| API dump versionné | Existence dans la build actuelle | `https://robloxapi.github.io/ref/class/<Classe>.html` (historique : `/ref/updates/`). Dump complet : `https://setup.rbxcdn.com/<version>-Full-API-Dump.json`, version lue sur `https://clientsettings.roblox.com/v2/client-version/WindowsStudio64` (le hash Mac renvoie 403). Hors ligne : `RobloxStudioBeta.exe --api <fichier>`. Le dump devance la doc de plusieurs semaines. |
| Définitions luau-lsp | Ce qu'un script de jeu voit | `globalTypes.None.d.luau` ne contient que la sécurité None ; `globalTypes.PluginSecurity.d.luau` ajoute les plugins ; `@deprecated` y marque les méthodes dépréciées. En roblox-ts, `@rbxts/types` joue ce rôle. Présent dans les types ≠ écrivable : `Workspace.StreamingEnabled` y figure, mais un script de jeu ne peut pas l'écrire. |
| Liste des dépréciés | Le remplaçant officiel | `https://create.roblox.com/docs/reference/engine/deprecated.md` ; movers : `https://create.roblox.com/docs/physics/mover-constraints.md`. |

**Lis trois champs du membre, pas un seul.**

- `Security` : `None` = utilisable par un script de jeu. `{"Read": "None", "Write": "PluginSecurity"}` = lisible, pas modifiable. `RobloxScriptSecurity` ou `RobloxSecurity` = interne, inutilisable.
- `Capabilities` : `PluginOrOpenCloud` réserve l'écriture aux plugins, même quand `Security` vaut `None`. Message : « lacking capability PluginOrOpenCloud ».
- `Tags` : `Deprecated` (avec `PreferredDescriptorName`, le remplaçant), `NotScriptable` (fenêtre Propriétés seulement), `ReadOnly`, `Yields` (cède : jamais dans un rendu React ni une transform `UpdateAsync`), `NotBrowsable` ou `Hidden` (aucune garantie).

**La sonde MCP.** Une ligne avant de bâtir dessus :

```luau
local ok, v = pcall(function() return (game:GetService("Workspace") :: any)["StreamingEnabled"] end)
return `{ok} {v}` -- execute_luau ne rend que la 1re valeur : renvoie une seule chaîne
```

- `execute_luau` tourne au niveau plugin : il lit `PluginSecurity` et écrit les propriétés `PluginOrOpenCloud`. Il cache donc les barrières d'un vrai Script. Pour un comportement de jeu, teste depuis un Script en playtest et lis la console.
- Le DataModel `Edit`, comme la barre de commande, est une VM à part : un `require` y crée une copie neuve du module, à l'état vide. N'en tire rien sur l'état du jeu. En playtest, vise `Server` ou `Client` et lis l'état par les instances et les attributs : un `require` y rend aussi sa propre copie (studio-mcp.md).
- Une sonde prouve l'existence, jamais l'absence. « X is not a valid member of Workspace » signale souvent un membre `NotScriptable`, pas une faute de frappe.
- Une écriture sans erreur ne prouve pas qu'une beta est active : certaines propriétés de beta s'écrivent sans rien faire.
- « lacking capability LoadUnownedAsset … » vient de la sandbox de l'outil, pas d'un refus de l'agent. Ne propose jamais de forcer des FFlags.

**Règles de verdict.**

- Absent de la doc ≠ inexistant : un membre récent vit souvent dans le dump avant d'avoir sa page. Pour écrire « n'existe pas », il faut le dump ou les définitions.
- Présent dans le dump ≠ utilisable : interne, réservé aux plugins, `NotScriptable`, derrière un flag (« … is not enabled ») ou beta Studio non publiable. Voir 2.3.
- Notes de version ≠ livré : « introduit en N » peut rester derrière un flag ; un nom de release note reste provisoire tant que le dump ne le confirme pas.
- Rien ne tranche : écris « non vérifié » et la vérification qui trancherait, ou demande. N'invente jamais un nom plausible.

**Vérifier un paquet avant de l'installer** (paquets sûrs : paquets.md).

- npm : `npm view @rbxts/<nom> version time.modified repository.url`. `E404` = le paquet n'existe pas. Regarde la date de dernière publication et le dépôt source.
- Le vrai code peut vivre dans un autre scope : `@rbxts/react` charge `@rbxts-js/*`. Chaque scope installé se mappe dans `default.project.json` (section 6).
- Wally : `wally search <mot>` ou `https://wally.run/package/<scope>/<nom>`. Dans `wally.toml`, une version explicite (`"scope/nom@1.2.3"`), jamais `@latest`, et le bon realm (`shared`, `server`, `dev`).
- Un nom vu dans un tutoriel ou une note d'IA n'est pas une preuve (exemple : `sleitnick/datastore-service`, introuvable par l'API de Wally le 2026-10-18). N'installe rien sans l'accord de l'utilisateur.

---

## 2. API inventées, mal nommées ou inutilisables

Relevées dans du code d'IA et des skills publics, puis recoupées avec des définitions de types ou une source fiable. Ne les écris pas.

### 2.1 Moteur et Luau : n'existent pas, ou sous un autre nom

| Écrit par erreur | Ce qui existe vraiment |
|---|---|
| `workspace:RequestStreamAroundAsync(pos)` | `player:RequestStreamAroundAsync(pos, timeout?)` |
| `workspace.SetNetworkOwnerAuto = false` | `part:SetNetworkOwner(nil)` force le serveur ; `part:SetNetworkOwnershipAuto()` rend l'automatique |
| `SecretsService` | `HttpService:GetSecret(clé)`, qui renvoie un `Secret`, pas une chaîne : `secret:AddPrefix(...)`, jamais `..` |
| `Instance.StreamingMode`, `ModelStreamingMode` sur un Folder | `Model.ModelStreamingMode` : `Default`, `Atomic`, `Persistent`, `PersistentPerPlayer`, `Nonatomic` |
| `Enum.ModelStreamingMode.Opportunistic` | n'existe pas ; `Opportunistic` est une valeur de `Workspace.StreamOutBehavior` |
| `Enum.StreamingIntegrity.Minimum` | `Workspace.StreamingIntegrityMode`, réglé dans Studio (2.3) |
| `Model.LevelOfDetail = Disabled` pour garder un modèle chargé | `ModelStreamingMode = Persistent`, ou `PersistentPerPlayer` + `model:AddPersistentPlayer(player)` |
| `viewport.WorldModel = wm` | une instance `WorldModel` enfant du `ViewportFrame` |
| `UIFlexLayout` | `UIListLayout` (options flex) + `UIFlexItem` sur les enfants |
| `UserInputService.TouchChanged` | `TouchMoved` |
| `PreferredInput == Enum.UserInputType.Touch` | `Enum.PreferredInput` : `KeyboardAndMouse`, `Touch`, `Gamepad`… |
| `Translator:FormatTranslate` | `FormatByKey` ou `Translate` |
| `DragStarted`, `DragMoved`, `DragEnded` | `DragStart`, `DragContinue`, `DragEnd` (`DragDetector`, `UIDragDetector`) |
| `UIPageLayout:JumpToPage` | `JumpTo(page)`, `JumpToIndex(i)`, `Next()`, `Previous()` |
| `VideoPlayer.Video` | `VideoPlayer.VideoContent` ; `Video` appartient à `VideoFrame` |
| `TeleportService:GetTeleportData()`, `.TeleportFailed`, `:CreateTeleportOptions()` | `GetLocalPlayerTeleportData()` côté client (`GetTeleportData` existe seulement sur `TeleportOptions`), `TeleportInitFailed`, `Instance.new("TeleportOptions")` |
| Champs inventés de `GetJoinData()` pour reconnaître un serveur réservé | `game.PrivateServerId ~= "" and game.PrivateServerOwnerId == 0` |
| `MarketplaceService.SubscriptionStatusChanged` | `Players.UserSubscriptionStatusChanged`, sans statut : relis `MarketplaceService:GetUserSubscriptionStatusAsync` |
| `Enum.ProductPurchaseDecision` renvoyé par un handler de `BindReceiptHandler` | `Enum.ReceiptDecision.Processed` ou `NotProcessedYet` ; `ProductPurchaseDecision` reste celui de `ProcessReceipt` |
| `SetCollisionGroupsCollidable` | `CollisionGroupSetCollidable` |
| `Enum.EasingStyle.Constant` | n'existe pas pour TweenService ; `Constant` appartient à `Enum.PoseEasingStyle` (poses d'animation) |
| `Enum.PathWaypointAction.Climb` | seulement `Walk`, `Jump`, `Custom` ; l'escalade se lit dans `waypoint.Label` |
| `IKControl.P` ; `AnimationClipProvider:LoadAnimationClipAsync` | aucune propriété `P` ; `GetAnimationClipAsync(id)` |
| `AudioPlayer.AssetId`, `.TimeVolume`, `:Pause()`, `:SeekTime()`, `:GetWaveform()` | `Asset`, `Volume`, `TimePosition`, `Play()` et `Stop()`, `GetWaveformAsync()` |
| `AudioEmitter.DistanceAttenuation = NumberSequence` ; `GetAudibility()` ; `AudioTextToSpeech:WaitForSpeechReady()` | `emitter:SetDistanceAttenuation({ [distance] = volume })` ; `GetAudibilityFor(listener)` ; `LoadAsync()` |
| `GeometryService:SubtractAsync(...)` traité comme une seule part | renvoie un tableau de parts |
| `CollectionService:GetTagged("X"):forEach(f)` | une table n'a pas de méthodes : `for _, inst in CollectionService:GetTagged("X") do` |
| `GuiService.IsTenFootInterface` lu comme propriété | méthode : `GuiService:IsTenFootInterface()` ; pour adapter l'UI, préfère `GuiService.ViewportDisplaySize` et `PreferredInput` |
| `MouseButton1Down` ou `Activated` sur un Frame | événements de `GuiButton` ; sur un Frame, `InputBegan` |
| `InputActionService`, `InputActionBinding`, `Enum.InputActionType.Button`, événements `Activated`/`Deactivated` | Input Action System : `InputContext`, `InputAction` (`Type = Enum.InputActionType.Bool`, événements `Pressed`, `Released`, `StateChanged`), `InputBinding`. `UserInputService` et `ContextActionService` ne sont pas dépréciés |
| `Script.Running`, `Player.IsPlaying`, `workspace.Players`, `game.CoreScriptService` | n'existent pas ; `Players` est un service |
| `bouton.MouseCursor` | n'existe pas : curseur global par `UserInputService.MouseIcon` |
| Propriété `UIShadow.ApplyShadowMode` | `UIShadow.Mode`, de type `Enum.ApplyShadowMode` |
| `StarterPlayer.CharacterAutoLoads` ; `Players.MaxPlayers = 20` | `Players.CharacterAutoLoads` ; `MaxPlayers` ne se règle pas par script (réglages du lieu sur le Creator Dashboard) |
| `queue:ReadAsync(n, false, invisibilité)` | le 3e argument est le délai d'attente ; l'invisibilité se règle dans `GetQueue(nom, invisibilité)`, puis `queue:RemoveAsync(readId)` avec l'id rendu par `ReadAsync` |
| `AssetService:PromptCreateAssetAsync` ; `Enum.CollisionFidelity.Scalable` ; `Enum.PlayerActions.Jump` | `PromptCreatePlatformContentAsync` ; `Tunable` ; `CharacterJump`, `CharacterForward`… |
| Skew, mode de fusion, flou d'un seul cadre, `UIStroke` qui épouse les pixels d'une image | n'existent pas (`UIBlur`, vu dans le dump sans doc, n'est pas fiable) : rotation de −3° à −5°, couches teintées, image pré-floutée, contour cuit dans le PNG (ui-fondamentaux.md) |
| `Cancel` de ProfileStore branché comme un signal | une fonction passée à `StartSessionAsync` |
| `__pairs` ; `buffer.readinteger` ; `goto`, opérateurs bit à bit (`&`, `<<`) | `__iter` ; `buffer.readi32`, `buffer.writeu8`… ; pas de `goto` (mais `continue`), et `bit32.band`, `bit32.lshift`… |

### 2.2 Outils, MCP et roblox-ts

| Écrit par erreur | Ce qui existe vraiment |
|---|---|
| Outils MCP `set_active_studio`, `run_as_job` | n'existent pas : passe `studio_id` à chaque appel (le champ s'appelle `id` dans la sortie de `list_roblox_studios`) ; l'asynchrone passe par l'argument `async: true` de l'outil |
| `start_stop_play` avec `action = "start"` | `is_start = true` |
| `multi_edit` sur plusieurs scripts avec `newSource` | un script par appel, `edits = { { old_string, new_string, replace_all } }`, en mode Edit seulement |
| `selene.toml` avec une table `[rules]` | `[lints]` |
| `rojo serve --no-watch`, `rojo push`, `rojo build --base` | absents de Rojo 7 officiel : lis `rojo <commande> --help` |
| `-- luau-lsp ignore` ; réglage `luau-lsp.types.roblox` (déprécié) | `luau-lsp.ignoreGlobs` ou `--!nocheck` ; `luau-lsp.platform.type = "roblox"` |
| Wally `nom = "scope/paquet@latest"` | une version explicite |
| `part instanceof Part` | `part.IsA("Part")` ou `classIs(part, "Part")` ; `instanceof` ne vaut que pour les classes TS |
| `import { typeIs } from "@rbxts/t"` | `typeIs(v, "string")` est global ; `@rbxts/t` fournit `t.string` |
| `x !== nil`, `null` | `undefined` : `null` est refusé, `nil` n'existe pas en TS |
| `get isAlive()` dans une classe | une méthode : les accesseurs sont refusés |
| `await HttpService.GetAsync(url)` | `GetAsync` cède et renvoie une chaîne, pas une Promise ; serveur seulement, HTTP activé |
| `remotes.Server.x` avec remo | `remotes.x.fire(...)` et `remotes.x.connect(...)` |
| « roblox-ts refuse décorateurs, `export default` et namespaces » | les trois marchent (décorateurs avec `experimentalDecorators`) ; la fusion de namespaces est refusée |

### 2.3 Existent, mais pas pour un script de jeu

| Membre | Ce qui se passe | Quoi faire |
|---|---|---|
| `Workspace.StreamingEnabled`, `StreamingIntegrityMode`, `StreamingMinRadius`, `StreamingTargetRadius` | Réglages d'expérience : écriture refusée ou sans effet en jeu | Fenêtre Propriétés ; dis à l'utilisateur quoi régler |
| `Workspace.SignalBehavior`, `PhysicsSteppingMethod`, `SandboxedInstanceMode`, `UseNewLuauTypeSolver`, `PlayerCharacterDestroyBehavior` | `NotScriptable` : « not a valid member », même pour un plugin | Idem |
| `Workspace.AuthorityMode`, `PlayerScriptsUseInputActionSystem`, `GuiService:SetUIScaleMultiplier` | Sécurité interne : « lacking capability RobloxScript » | Idem, ou y renoncer |
| `CollisionFidelity`, `RenderFidelity`, `Model.LevelOfDetail`, `MeshPart.MeshId` | Écriture réservée aux plugins ; lire `CollisionFidelity` marche | Studio, ou `AssetService:CreateMeshPartAsync(part.MeshContent, { CollisionFidelity = … })` puis `part:ApplyMesh(nouveau)` |
| `Players.BanningEnabled`, `SoundService.ListenerLocation`, `Terrain.Decoration`, `UserInputService:GetPlatform()` | Absents des définitions des scripts de jeu : l'écriture ou l'appel échoue à l'exécution | Réglage Studio ; pour le type d'appareil, `PreferredInput` |
| `Lighting.Technology` | Interne (`RobloxScriptSecurity`), réglable dans Studio seulement, remplacé par `LightingStyle` et `PrioritizeLightingQuality` ; les items `Legacy`, `Compatibility` et `Unified` de l'enum sont dépréciés (doc, 2026-10-18) | `LightingStyle` ; son écriture par script est contestée (2.4) |
| `MarketplaceService:GetPassOwnershipAsync`, `CanUsersTransferRobuxAsync`, `Shapecast` avec `cframeOverride` | Présents mais coupés : « … is not enabled » | `UserOwnsGamePassAsync` ; sinon attendre |
| `CallingService`, `AnimatedImageService`, `MomentsService`, `ScriptScannerService` | Internes ou coquilles vides (`ModerationService`, lui, a des méthodes documentées, en partie réservées aux univers sur liste blanche) | « Rien à migrer » ; un GIF devient une planche de sprites + `ImageRectOffset` |
| `RenderStepped`, `BindToRenderStep`, `UserInputService`, `ContextActionService` sur le serveur ; global `plugin` hors d'un plugin | Mauvais contexte : erreur, sans effet ou `nil` | Code client ; plugin seulement |
| `ConnectParallel`, `task.desynchronize` hors d'un `Actor` | « must be rooted under an Actor » | Script sous un `Actor` |
| `if local`, `if const`, tables exactes, classes, `coroutine.finally`, bibliothèque `integer` | Beta Studio ou prototype : ne compile pas en jeu publié, ou vaut `nil` | Syntaxe stable |

### 2.4 Douteux ou contestés : ne tranche pas de mémoire

- Réels malgré les doutes (définitions de types de septembre 2026) : `DataStoreService:SetRateLimitForRequestType`, `RaycastParams.ExcludeInstances` et `IncludeInstances`, `OverlapParams.Tolerance`, `GuiService:GetInsetArea`, `MarketplaceService:BindReceiptHandler`, le champ `ApplyDeviceBlock` de `BanAsync`. Ne les « corrige » pas.
- Groupes de collision : `RegisterCollisionGroup` et `CollisionGroupSetCollidable` existent sur `PhysicsService` (non dépréciés) et sur `workspace` (`WorldRoot`) (doc, 2026-10-18). Suis ce que le projet utilise.
- Budgets DataStore par minute (doc `cloud-services/data-stores/error-codes-and-limits.md`, 2026-10-18), stores standards : par serveur, lecture, écriture et suppression `60 + joueurs × 40`, liste `5 + joueurs × 2` ; pour toute l'expérience, lecture et suppression `300 + CCU × 40`, écriture `300 + CCU × 20`, liste `300 + CCU × 2`. `OrderedDataStore` et anciens barèmes : donnees.md.
- `vector.lerp` et `math.lerp` existent (doc, 2026-10-18) ; `a + (b - a) * t` marche partout.
- `Lighting.LightingStyle`, `PrioritizeLightingQuality` : la doc les dit scriptables (sécurité None, capacité `Environment`, 2026-10-18), un test réel d'octobre 2026 a obtenu « lacking capability PluginOrOpenCloud » (à vérifier dans un vrai Script).
- `TeleportToPlaceInstance`, `TeleportPartyAsync`, `TeleportToPrivateServer`, `TeleportToSpawnByName` (→ `TeleportAsync`) et `ReserveServer` (→ `ReserveServerAsync`) : dépréciés dans la doc (2026-10-18), pas encore marqués dans les définitions de septembre. `TeleportAsync` reste le bon choix.
- Clé Fusion `[Child "Nom"]` pour hydrater un enfant par son nom : inventée. Dans Fusion 0.3, `Child(x)` est une fonction qui renvoie son argument, un simple indice de type (doc Fusion, 2026-10-18).
- Script Sync : `.client.luau` crée un Script `RunContext = Client` ; seul `.local.luau` donne un LocalScript (doc `scripting/sync.md`, 2026-10-18).
- Lune 0.9.0 et plus : `process.spawn` est remplacé par `process.exec` (même comportement) et `process.create` (non bloquant) (changelog Lune, 2026-10-18).

---

## 3. API dépréciées et leur remplaçant

Réécris le déprécié dans le code que tu écris. Dans du code existant, signale le simple « déconseillé » seulement si tu touches déjà la ligne.

| Déprécié ou hérité | Remplaçant |
|---|---|
| `wait()`, `spawn(f)`, `delay(t, f)` | `task.wait()`, `task.spawn(f)` (immédiat) ou `task.defer(f)`, `task.delay(t, f)` (annulable par `task.cancel`) |
| `tick()` (déconseillé : fuseau, précision) | `os.clock()` pour une durée ou un cooldown ; `os.time()` ou `DateTime` pour un horodatage ; `workspace:GetServerTimeNow()` pour une heure partagée. `time()` retarde sous charge : pas pour un rate limiter |
| `:connect`, `:wait`, `:disconnect` en minuscules | `:Connect`, `:Wait`, `:Disconnect` |
| `BodyVelocity`, `BodyPosition`, `BodyGyro` | `LinearVelocity`, `AlignPosition`, `AlignOrientation`, dans cet ordre |
| `BodyForce`, `BodyThrust`, `BodyAngularVelocity`, `RocketPropulsion` | `VectorForce`, `VectorForce`, `AngularVelocity` (ou `Torque`), `LineForce` + `AlignOrientation` |
| `Humanoid:LoadAnimation`, `AnimationController:LoadAnimation` | `Animator:LoadAnimation` (Animator créé côté serveur pour répliquer) |
| `KeyframeReached` (encore valide) | `track:GetMarkerReachedSignal("Nom")` |
| `part.Velocity`, `part.RotVelocity` | `AssemblyLinearVelocity`, `AssemblyAngularVelocity` |
| `SetPrimaryPartCFrame`, `GetPrimaryPartCFrame` | `PivotTo`, `GetPivot` (PrimaryPart inutile) |
| `CFrame.new(pos, cible)` | `CFrame.lookAt(pos, cible)` |
| `Instance.new("Part", parent)` (déconseillé) | propriétés d'abord, `Parent` en dernier |
| `FilteringEnabled = true` (script ou `project.json`) | rien : le filtrage est toujours actif |
| `FindPartOnRay*`, `Ray.new` pour viser, `FindPartsInRegion3` | `workspace:Raycast(origine, direction, params)`, `GetPartBoundsInBox`, `GetPartsInPart`, `Blockcast`, `Shapecast` |
| `RaycastFilterType.Blacklist`, `Whitelist` | `Exclude`, `Include` |
| `PhysicsService:CreateCollisionGroup`, `SetPartCollisionGroup` | `RegisterCollisionGroup`, `part.CollisionGroup` (voir 2.4) |
| `StreamingPauseMode` | `StreamingIntegrityMode` |
| Dépréciés : `TeleportToPlaceInstance`, `TeleportToPrivateServer`, `TeleportPartyAsync`, `TeleportToSpawnByName`, `ReserveServer` ; hérité : `Teleport` | `TeleportService:TeleportAsync(placeId, joueurs, options)` côté serveur, sous `pcall`, avec `TeleportOptions` (serveur réservé : `ShouldReserveServer`, ou `ReserveServerAsync`) |
| `BadgeService:AwardBadge`, `UserHasBadge` | `AwardBadgeAsync`, `UserHasBadgeAsync` |
| `Player:IsInGroup`, `IsFriendsWith`, `GetFriendsOnline` ; `GetRankInGroup`, `GetRoleInGroup` et même leurs versions `Async` | `IsInGroupAsync`, `IsFriendsWithAsync`, `GetFriendsOnlineAsync` ; `GroupService:GetRolesInGroupAsync(userId, groupId)` (définitions de septembre 2026) |
| `Players:GetHumanoidDescriptionFromUserId`, `CreateHumanoidModelFromUserId`, `CreateHumanoidModelFromDescription`, `Humanoid:ApplyDescription`, `ApplyDescriptionReset`, `Player:LoadCharacterWithHumanoidDescription` | la version `Async` de chacune |
| `OpenCloudApiV1:CreateUserNotificationAsync` (lève toujours « not enabled »), `SocialService:PromptLinkSharing` | Open Cloud par `HttpService` (open-cloud.md) ; `PromptLinkSharingAsync` (scripts serveur seulement) |
| `MarketplaceService:GetProductInfo`, `PlayerOwnsAsset` | `GetProductInfoAsync`, `PlayerOwnsAssetAsync` |
| `ContentProvider:Preload` | `PreloadAsync` |
| `LocalizationService:GetTranslatorForPlayer` (déprécié) | `GetTranslatorForPlayerAsync`, qui cède : un simple rechercher-remplacer change le flux |
| `CollectionService:GetCollection` | `GetTagged` |
| Ancien chat (`Chat`, `Chat:Chat`, `Chat:FilterStringAsync`) | `TextChatService`, `TextChatService:DisplayBubble`, `TextService:FilterStringAsync` côté serveur |
| `TextFilterResult:GetChatForUserAsync` (renvoie une chaîne vide) | `GetNonChatStringForBroadcastAsync` ou `GetNonChatStringForUserAsync` |
| `TweenPosition`, `TweenSize`, `TweenSizeAndPosition` | `TweenService:Create(...)` puis `:Play()` |
| `GuiObject.Transparency`, `BackgroundColor`, `BorderColor`, `Draggable` ; `FontSize` des textes | `BackgroundTransparency`, `BackgroundColor3`, `BorderColor3`, `UIDragDetector` ; `TextSize` |
| `TextStrokeTransparency`, `TextStrokeColor3` (encore valides) | `UIStroke` |
| `GuiService:AddSelectionParent`, `AddSelectionTuple` | `SelectionGroup` + `SelectionBehavior*` |
| `UserInputService.ModalEnabled` | `GuiService.TouchControlsEnabled` |
| `Mouse.KeyDown`, `Mouse.KeyUp` ; `player:GetMouse()` comme système d'entrée | `UserInputService`, `ContextActionService`, Input Action System |
| `HopperBin` ; `Fire`, `Smoke`, `Sparkles` (encore valides) | `Tool` ; `ParticleEmitter` |
| `MakeJoints`, `BreakJoints` | `WeldConstraint` et autres contraintes |
| `GlobalDataStore.OnUpdate` | `MessagingService` |
| `ChangeHistoryService:SetWaypoint` (plugins, « bientôt déprécié ») | `TryBeginRecording` et `FinishRecording` |
| `RenderStepped`, `Stepped` (encore valides) | `PreRender`, `PreSimulation` pour le code neuf. `Heartbeat` n'a pas de remplaçant : il reste valide, et `PostSimulation` est un événement distinct (doc, 2026-10-18 ; architecture.md, section 11) |
| `TextLabel.Font` et `Enum.Font` (encore valides) | `FontFace = Font.fromEnum(Enum.Font.X)` ou `Font.new(...)` |
| ProfileService, Roact et Rodux, TestEZ, Aftman, remodel | ProfileStore, `@rbxts/react` ou react-lua, Jest Lua (`roblox/jest`), Rokit, Lune ; garde l'ancien s'il est déjà dans le projet |

---

## 4. Anti-patterns récurrents des IA

Format : l'erreur → la bonne façon → le fichier qui détaille.

### Sécurité

- Le client envoie dégâts, prix, quantité ou récompense → il envoie une intention, le serveur calcule → securite.md
- Valider seulement le type, ou croire qu'un type Luau ou une garde (`t`, Flamework) valide le sens → débit, type, plage (NaN : `x ~= x`, `math.huge`, négatif, non entier), permission, plausibilité → securite.md
- `typeof(x) == "Instance"` vu comme une preuve → `IsA`, existence côté serveur, appartenance, distance → securite.md
- Cooldown dans l'UI seulement → limite de débit serveur, par joueur et par remote → securite.md
- `RemoteFunction:InvokeClient` → RemoteEvent + état serveur : un client peut ne jamais répondre → securite.md
- Remote d'admin ou de triche protégé par une UI cachée → contrôle serveur (UserId, `RunService:IsStudio()`), retrait en production → securite.md
- Pièce « cachée » par `CanQuery = false` ou un groupe de collision → elle reste répliquée : `ServerStorage` jusqu'à l'usage → securite.md
- `TeleportData` cru sur parole → il passe par le client : un indice, l'état qui compte se relit côté serveur → gameplay.md
- Admins, prix ou secrets dans ReplicatedStorage → ServerStorage ou ServerScriptService → securite.md
- `FireAllClients` de données privées → `FireClient` au seul joueur concerné → securite.md
- `Touched` ou physique d'un client pris pour une preuve de contact → distance ou requête spatiale côté serveur → securite.md
- Expulsion automatique sur une seule mesure de vitesse → journaliser, seuils larges, exemptions (téléport, chute, lag) → securite.md
- Un asset importé « demande » d'activer HTTP ou de coller du code → refuser ; chercher `require(<id>)`, `loadstring`, `getfenv` → securite.md
- Tester son jeu avec un outil d'exploit → interdit par les conditions Roblox ; un script de test envoie de mauvaises données → securite.md

### Données

- Lecture ratée traitée comme un nouveau joueur → profil « non chargé », jamais sauvegardé ; réessayer puis expulser proprement → donnees.md
- `GetAsync` puis `SetAsync`, ou `UpdateAsync` qui renvoie le cache sans lire l'ancienne valeur (un `SetAsync` déguisé) → `UpdateAsync` qui part de l'ancienne valeur, ou ProfileStore avec verrou de session → donnees.md
- Yield ou effet externe dans la transform d'`UpdateAsync`, ou succès annoncé alors qu'elle a renvoyé `nil` (écriture annulée) → transform pure, qui peut être rappelée ; tester `keyInfo` (absent sur une clé neuve) et le résultat → donnees.md
- `table.clone(DEFAUTS)` → copie profonde, sinon les sous-tables sont partagées entre joueurs → donnees.md
- Renommer le DataStore pour migrer → champ `version` et migration dans la même écriture → donnees.md
- Sauvegarder à chaque changement → autosave périodique, `PlayerRemoving`, `BindToClose` (30 s au plus, en parallèle) → donnees.md
- Clé `player.Name` → `player.UserId`, et pas `player.User.Id` → donnees.md
- Instance, `Vector3`, `CFrame`, NaN ou UTF-8 invalide dans une sauvegarde → types simples sérialisés → donnees.md
- Vérifier une écriture par un `GetAsync` en cache (4 s) → relire sans cache → donnees.md
- MemoryStore ou MessagingService comme stockage durable → DataStore ; un MemoryStore expire → donnees.md
- Réessayer en boucle serrée, rejouer `IncrementAsync`, tirer un nouvel identifiant d'opération à chaque essai → backoff ; un seul identifiant fixé avant le premier essai, sinon double crédit → donnees.md
- Garde `GetAttribute("DataLoaded") == false` → `~= true`, sinon `nil` passe → donnees.md
- Progression stockée dans leaderstats seulement → profil serveur ; leaderstats n'est qu'un affichage → donnees.md

### Achats

- Pass vérifié côté client → `UserOwnsGamePassAsync` côté serveur ; un échec vaut « inconnu », pas « non » → monetisation.md
- Octroi dans `PromptProductPurchaseFinished` → `ProcessReceipt` (ou `BindReceiptHandler`) seulement → monetisation.md
- `PurchaseGranted` avant la sauvegarde, ou dès qu'un `pcall` réussit → effet et `PurchaseId` sauvés ensemble, `NotProcessedYet` au moindre doute → monetisation.md
- `ProcessReceipt` assigné dans deux scripts → un seul routeur : la dernière affectation gagne en silence → monetisation.md
- Prix Robux codés en dur → `GetProductInfoAsync` ; ne jamais inventer un prix si l'appel échoue → monetisation.md
- Œufs ou coffres payants sans probabilités affichées ni `PolicyService` → les deux sont obligatoires (`ArePaidRandomItemsRestricted`) → monetisation.md
- Pub récompensée accordée par un remote « j'ai vu la pub » ou sur `ShowCompleted` → `AdService` : la récompense arrive en reçu de produit, par le même `ProcessReceipt` idempotent → monetisation.md
- Bonus quotidien sans date sauvegardée → stocker l'heure du dernier octroi → monetisation.md
- IDs de produit, de pass, de son ou d'animation inventés → `0` explicite et l'endroit où mettre le vrai → monetisation.md

### Réseau et architecture

- CFrame du personnage envoyée à chaque frame → le moteur réplique déjà le personnage → performance.md
- `UnreliableRemoteEvent` pour monnaie, inventaire ou état ordonné → `RemoteEvent` fiable → securite.md
- `BindableEvent` pour parler au client ; `_G` ou `shared` entre modules → un remote ; `require` → architecture.md
- Yield dans le corps d'un ModuleScript → il bloque tous ceux qui le requièrent : initialiser après le `require` → architecture.md
- Un Script par porte, pet ou PNJ → un système + `CollectionService` → architecture.md
- Réécrire ce que le moteur fournit (boucle de distance au lieu d'un `ProximityPrompt`, interpolation à la main au lieu de `TweenService`, drag manuel au lieu d'`UIDragDetector`, classe Signal maison) → l'outil intégré → paquets.md
- `PlayerAdded:Connect` seul → traiter aussi `Players:GetPlayers()`, cas fréquent en test Studio → architecture.md
- Connexions jamais coupées, tables par joueur jamais vidées → nettoyage à `PlayerRemoving` et à chaque respawn → architecture.md
- `Parent = nil` pour nettoyer → `Destroy()`, qui coupe aussi les connexions de l'instance → architecture.md
- `workspace.A.B` lu directement côté client → `WaitForChild` avec timeout ; sous streaming, l'objet peut manquer → performance.md
- Livrer seulement le LocalScript d'un système → tranche complète : types, serveur validé, contrôleur, remotes → architecture.md

### Gameplay, monde 3D, performance

- Personnage gardé en variable après un respawn, ou téléporté dans `PlayerAdded` (il n'existe pas encore) → relier à chaque `CharacterAdded` → gameplay.md
- Animation de PNJ jouée dans un LocalScript → invisible pour les autres : la jouer côté serveur → gameplay.md
- Plateforme ancrée déplacée par un tween de CFrame → le joueur glisse : contraintes ou mouvement côté client → gameplay.md
- Cylindre dressé avec `Size.Y` comme hauteur → l'axe long d'un cylindre est X → monde-3d.md
- Positions recalculées de mémoire entre deux appels → relire les vraies CFrames, hériter d'une ancre → monde-3d.md
- Des milliers de Parts → fusion des voisines (greedy meshing) ou un seul MeshPart → monde-3d.md
- `while true do task.wait() end` pour attendre une condition → un signal (`:Wait()`, `GetPropertyChangedSignal`) → performance.md
- Polling à 10 Hz « optimisé » en `Heartbeat` → plus cher : un événement ou une fréquence plus basse → performance.md
- Tween serveur de décor, pathfinding de chaque PNJ à chaque frame → tween client ; recalcul étalé et déclenché par événement → performance.md
- Gains de vitesse cités de mémoire (« `--!native` ×2 à ×5 », « pcall gratuit ») → mesurer ; ne citer qu'un chiffre publié → performance.md

### UI

- Mise en page en pixels, `TextScaled` sans `UITextSizeConstraint` ou avec `AutomaticSize` → `Scale`, `AnchorPoint`, contraintes, bornes Min et Max du texte, un seul des deux modes → ui-fondamentaux.md
- `IgnoreGuiInset` pris pour la zone sûre → `ScreenGui.ScreenInsets` → ui-fondamentaux.md
- `MouseButton1Click` → `Activated`, qui couvre souris, toucher et manette → ui-fondamentaux.md
- `Modal = true` pour bloquer les clics → `Active = true` ; `Modal` libère la souris en caméra verrouillée → ui-fondamentaux.md
- `AbsoluteSize` lu à la première frame (il vaut 0) → réagir à son changement → ui-fondamentaux.md
- Tweener `Text` → animer un nombre et réécrire le texte → ui-fondamentaux.md
- `UICorner` censé découper les enfants, `UIGradient` censé teinter les descendants → `CanvasGroup` → ui-fondamentaux.md
- `UIGradient` sur un fond coloré : il multiplie la couleur, le rendu s'assombrit → fond blanc, vraies couleurs dans le dégradé → ui-fondamentaux.md
- Particules ou Beam dans une ScreenGui ou un ViewportFrame → ils ne s'affichent pas → ui-fondamentaux.md
- `UIStroke` d'un TextButton censé border le bouton → `ApplyStrokeMode = Border` → ui-fondamentaux.md
- UI validée dans une seule fenêtre Studio → émulateur téléphone et PC → ui-fondamentaux.md
- Bouton qui accorde l'objet localement → remote d'intention, puis affichage de la réponse serveur → ui-react.md

### Luau écrit comme du JS ou du Python

- `0` ou `""` traités comme faux → seuls `false` et `nil` le sont → luau.md
- `a and b or c` quand `b` peut être faux ; `GetAttribute("A") or true`, toujours vrai → `if a then b else c` ; tester `== nil` → luau.md
- Indices à 0, `#t` sur une table à trous → tableaux 1-indexés, `table.remove` → luau.md
- `!=`, `null`, `?.`, `??`, `${x}`, `try/catch`, `.length`, `.push`, déstructuration → `~=`, `nil`, tests explicites, `` `{x}` ``, `pcall`, `#t`, `table.insert`, accès par champ → luau.md
- Regex (`\d`, `|`, `(?:…)`) → motifs Lua (`%d`, pas d'alternance) ; `string.find(s, motif, 1, true)` pour du texte brut → luau.md
- `"x" .. booléen` ou `.. Vector3` → `tostring(...)` → luau.md
- `Color3.R * 255` lu comme un entier → `R` va de 0 à 1 ; `Color3.fromRGB` pour 0–255 → luau.md
- `Module.Champ: Type = valeur` → annotation invalide sur un champ : type la table → luau.md
- `--!strict -- commentaire` sur une ligne → directive ignorée ; ligne qui commence par `(` → « Ambiguous syntax » → luau.md
- Closure qui appelle une `local function` déclarée plus bas → elle vaut `nil` : déclarer avant → luau.md
- `obj:method(obj)` → `self` passé deux fois → luau.md
- `task.spawn` cru différé, `task.delay(n)` cru exact → `task.spawn` démarre tout de suite, `task.defer` attend la fin du cycle ; `task.delay(n)` attend au moins n s → luau.md
- `getfenv`, `setfenv`, `loadstring` → désoptimisent le script ou sont coupés par défaut : à bannir → luau.md
- `:: any` pour faire taire le vérificateur → corriger le type à la source → luau.md
- Folklore Lua 5.1 (`local floor = math.floor`, `obj.Method` en cache) → inutile en Luau → luau.md

### roblox-ts

- `arr[1]` pour le premier élément ; `.length`, `map.size` → `arr[0]` (le compilateur décale) ; `.size()` → outils.md
- `==`, `!=` → `===`, `!==` ; les premiers sont refusés → outils.md
- `0 || 100` pour un défaut numérique → `??` : roblox-ts garde la vérité de JS, `0` et `""` y sont faux → outils.md
- `Object.keys`, `Object.entries` → `Map`, `pairs(obj)` ou `@rbxts/object-utils` → outils.md
- Méthode passée nue en callback → `() => this.method()` → outils.md
- `FindFirstChild(...) as Part`, `any` manipulé → tester l'existence puis `IsA` ; `unknown` et gardes `typeIs` → outils.md
- Variable nommée `next` ou mot-clé Luau (`end`, `local`…) → identifiant refusé par le compilateur : renommer → outils.md
- Défaut `= Enum.Material.SmoothPlastic` → annoter `material: Enum.Material` → outils.md
- Modifier `out/*.luau` → modifier `src/*.ts(x)` → outils.md
- Promise sans `.catch` → gérer chaque rejet → outils.md

### Méthode d'agent

- Tout le jeu en une passe → une tranche verticale jouable à la fois → architecture.md
- « Dédupliquer » du code sans lire ses appelants → la moitié d'une fonctionnalité disparaît : tracer chaque appelant d'abord → architecture.md
- Plusieurs Studio ouverts, cible devinée → confirmer la place visée (`studio_id`) avant toute écriture → studio-mcp.md
- « Corrigé » annoncé sans l'avoir observé → « corrigé dans le build, à vérifier : X » → studio-mcp.md
- Refactor, restyle ou rééquilibrage non demandé pendant un correctif → ne toucher que le demandé → architecture.md
- Livrer des `-- TODO`, des corps vides, des `print` de debug ou l'ancienne version commentée « au cas où » → code fini et propre : une coquille qui a l'air finie est pire qu'un manque visible → section 8
- Supposer un réglage de place (Server Authority, `SignalBehavior`, `StreamingEnabled`, type de rig, mode strict) → le lire dans Propriétés ou demander ; une lecture ratée vaut « inconnu », pas « désactivé » → architecture.md
- Endpoint Open Cloud récité de mémoire → lire `openapi.json` ou `cloud/llms.txt` → open-cloud.md
- Clé API dans le code, le chat ou un argument → Secret ou variable d'environnement ; une clé collée dans le chat est à révoquer → open-cloud.md
- Réessayer un 429 ou un 5xx en boucle → respecter `retry-after` ; ne jamais rejouer une écriture non idempotente → open-cloud.md

---

## 5. Faux positifs : ne pas « corriger » du code juste

Avant chaque remarque de revue, quatre portes : as-tu lu les deux côtés (émetteur et handler, écriture et lecture) ? La forme étrange peut-elle être voulue ? Tiens-tu un scénario d'échec concret (telle entrée → tel mauvais résultat) ? L'API est-elle vérifiée pour cet environnement ? Un seul « non » : pas de remarque. Dans le doute, une simple suggestion. Un fichier correct mais étrange donne un rapport qui dit clairement « 0 problème ».

Ne signale pas :

- `while task.wait(60) do sauvegarder() end` : de l'ordonnancement, pas du polling.
- Une allocation dans `Touched`, `OnServerEvent` ou `PlayerAdded` : chemin froid. Seuls les callbacks par frame et les boucles serrées comptent, et seulement si l'allocation peut sortir de la boucle.
- Un `WaitForChild` sans timeout sur ReplicatedStorage ou PlayerGui : seul le contenu streamé ou optionnel en demande un.
- `pairs` et `ipairs`, la globale `workspace`, un `GetService` non mis en cache : du style. Un `::` juste après une vérification à l'exécution, un `:Once()` : corrects.
- `Instance.new(classe, parent)` dans du code existant, `FireAllClients` pour un fait public, une `RemoteFunction` client → serveur bien validée : au plus une suggestion.
- `TeleportService:Teleport`, `Humanoid`, `LocalScript` : pas dépréciés.
- `Script` ou `LocalScript` dans un projet Rojo, `.lua` au lieu de `.luau`, `init.luau`, pas de `default.project.json` (Script Sync) : outillage valide.
- L'absence de `--!strict` : choix du projet, ne l'ajoute pas sans demande. `const` n'est pas une faute de frappe.
- Un `pcall` absent autour de ce qui ne cède ni ne lève (attributs, maths).
- Pas de validation sur un `BindableEvent` interne au serveur : aucun client derrière.
- Plusieurs `UIStroke` sur un objet (empilés par `ZIndex`), un lissage de caméra dans `RenderStepped`, `Enum.Font` dans du vieux code : ça marche.
- Des exigences de grand studio (télémétrie, pooling, paliers d'appareils) sur un prototype.
- Un Script qui écrit `Humanoid.Health`, lit `CollisionFidelity`, assigne `ProcessReceipt` ou `OnServerInvoke` : ça marche, nul besoin d'un plugin.

---

## 6. Pièges d'environnement et d'outillage

### roblox-ts + React + Rojo (testé le 2026-10-18)

- Aucune UI et « Infinite yield possible on '…node_modules:WaitForChild("@rbxts-js")' » : le scope `@rbxts-js` n'est pas mappé. Ajoute `"@rbxts-js": { "$path": "node_modules/@rbxts-js" }` à côté de `"@rbxts"` (ui-react.md, section 2).
- « Cannot use identifier reserved for compiler internal usage » : renomme la variable (`next`…).
- Plusieurs `UIStroke` sur un même objet sont autorisés (doc du 2026-06-24) : empile-les par `ZIndex`. Un mode `Contextual` et un mode `Border` cohabitent. L'avis « un seul UIStroke par objet » est périmé.
- Pas de SVG : PNG, JPG, BMP ou TGA. Dessine en SVG, rasterise (resvg, en 2x), uploade par Open Cloud Assets pour obtenir un asset id (open-cloud.md).
- Rojo installé par Aftman : « no aftman.toml files list this tool » → `aftman.toml` à la racine avec `rojo = "rojo-rbx/rojo@<version>"`. Rokit : « Failed to find tool … in any project manifest file » → lance l'outil depuis le dossier du projet.
- `rojo plugin install`, puis redémarre Studio si l'onglet manque ; CLI et plugin de même version majeure. Boucle : `npx rbxtsc -w`, `rojo serve` (port 34872), Connect, Play. Sans Studio : `curl http://localhost:34872/api/rojo`.
- `CanvasGroup` a des bugs de rendu sur certains mobiles, et le glyphe Robux `"\u{E002}"` rend mal selon la taille : teste sur appareil.

### Rojo et Script Sync

- Rojo : `x.server.luau` → Script ; `x.client.luau` → LocalScript (Script `RunContext = Client` si `emitLegacyScripts` vaut false) ; `x.luau` → ModuleScript ; `init.*` donne sa classe au dossier.
- Script Sync : `x.local.luau` → LocalScript ; `x.client.luau` → Script `RunContext = Client` (2.4). Ne mélange pas les deux conventions.
- Un module à `require` nommé `x.server.luau` devient un Script : le `require` échoue. Un `x.luau` dans StarterPlayerScripts est un ModuleScript qui ne tourne jamais.
- `Animate.client.luau` rangé dans un sous-dossier ne remplace pas l'`Animate` par défaut : il doit être enfant direct de StarterCharacterScripts.
- Un seul écrivain par script : jamais le même script par Rojo et par MCP ou à la main. Ne réponds pas au dialogue « Keep Studio / Keep Disk » à la place de l'utilisateur. Le retour Studio → disque de Rojo n'est pas fiable pour le code.
- `.gui` n'est pas un format Rojo : `.rbxm`, `.rbxmx`, `.model.json`, ou UI en code.
- Plugin local de Studio : un fichier `.luau` n'a pas été chargé en test ; préfère `.lua`, `.rbxm` ou `.rbxmx`, puis redémarre Studio.

### Lune (pièges vus en 0.10 ; prends 0.10.5 ou plus pour lire une place récente)

- Ces pièges donnent un build « réussi » mais faux : `SoundId` perdu (le fichier stocke `AudioContent`) ; `CFrame.lookAt` faux avant la 0.11.0 (→ 0.11, ou `CFrame.fromMatrix`) ; `WeldConstraint` écrite à offset identité (→ `Weld` avec `C0` et `C1`) ; `Attachment.Position`, `ScreenGui.IgnoreGuiInset`, `ViewportFrame.CurrentCamera` non sauvés ; attributs numériques au-delà de 2^31 tronqués ; `GetService` qui ajoute le service ; MeshParts sans données de rendu ; avant la 0.10.5, un nouvel userdata à chaque accès, donc pas d'Instance comme clé de table (changelog Lune, 2026-10-18). Relis chaque propriété modifiée dans le fichier de sortie.
- Un script Lune a accès à toute la machine, et `roblox.getAuthCookie()` lit le vrai cookie : ne lance jamais un script non lu.

### luau-lsp, tests, CI, Open Cloud

- `luau-lsp analyze` sans définitions Roblox : « Unknown global » partout ; `luau-analyze` seul ne connaît aucun global Roblox. Une analyse sautée n'est pas une analyse réussie.
- Sourcemap périmée : une API réelle est déclarée inconnue. `rojo sourcemap --watch` ; pour les types des paquets Wally, `wally-package-types`.
- Types propres ≠ écriture moteur valide : `Workspace.StreamingEnabled = true` passe le typage et échoue en jeu. Exécute une fois dans Studio le code qui touche au moteur.
- Jest « 0 test trouvé » = configuration ratée, souvent `LoadStringEnabled`. Jest Lua 3.x : `roblox/jest` + `roblox/jest-globals`.
- Upload Open Cloud en 409 « Server is busy » : la place est ouverte dans Studio. Demande de la fermer au lieu de réessayer.

### MCP de Studio (détails : studio-mcp.md)

- Seul un résultat de `list_roblox_studios` prouve la connexion. N'ajoute jamais un second serveur MCP Studio ni un paquet « Roblox MCP » tiers. Les outils dépendent de la version de Studio : lis leurs schémas.
- `multi_edit` marche en mode Edit seulement. Relis les lignes juste avant, fais de petites éditions, n'ancre pas sur des espaces d'alignement : la lecture d'un gros script peut être périmée.
- `screen_capture` : noire si Studio est réduit, sans SurfaceGui ni BillboardGui, parfois périmée. Mesure plutôt : `AbsoluteSize`, `Visible`, `TextBounds`.
- Le sous-agent de playtest peut annoncer un faux succès : recoupe avec `get_console_output`. Vérifie l'arrêt de chaque playtest lancé.
- Un timeout ou une coupure donne un résultat inconnu : relis l'état avant de rejouer une écriture, jamais à l'aveugle.
- `http_get` ne sert que la doc Roblox (`.md`, `llms.txt`) ; il refuse `openapi.json`.
- `Script.Source` plafonne sous 200 000 caractères ; au-delà, `ScriptEditorService:UpdateSourceAsync` (plugin). `execute_luau` n'a pas d'annulation fiable : fais sauvegarder la place avant un gros changement.

### Studio n'est pas le jeu en ligne

- DataStores en Studio : « Enable Studio Access to API Services » requis, et jamais sur la place de production, qui partage la même base. Les limites diffèrent en mode Run.
- Téléportation impossible et analytics invisibles en Studio. Un Play Solo ne prouve pas la réplication : mode de test **Server & Clients** (ancien nom : Test → Clients and Servers), avec au moins deux clients.

---

## 7. Diagnostic express « ça ne marche pas »

| Symptôme | Causes probables | Première vérification |
|---|---|---|
| Rien ne s'affiche (UI React) | `@rbxts-js` non mappé ; `rbxtsc` en erreur ; Rojo déconnecté ; racine sans `screengui` | Output : « Infinite yield … @rbxts-js » ? En Play, la ScreenGui est-elle dans PlayerGui ? |
| Rien ne s'affiche (Luau) | `.luau` nu devenu ModuleScript ; LocalScript au mauvais endroit ; `Enabled` ou `Visible` faux, taille 0, ZIndex | Classe et emplacement du script ; un `print` en première ligne |
| `Infinite yield possible on 'X:WaitForChild("Y")'` | Objet jamais créé, mal nommé, côté serveur seulement, streamé, ou mapping Rojo manquant | Explorer côté Client pendant le Play ; c'est un avertissement après 5 s, pas toujours un blocage |
| RemoteEvent muet | Handler du mauvais côté ; remote créé trop tard ou rangé dans ServerStorage ; `FireServer(player, …)` côté client, qui décale les arguments ; handler qui lève ; rate limit qui jette | `print` à l'entrée du handler ; Output serveur ET client |
| RemoteFunction bloquée | `InvokeClient` sans réponse ; handler qui cède ; plusieurs `OnServerInvoke` (seul le dernier compte) | RemoteEvent + réponse ; `pcall` autour d'`InvokeServer` |
| Données non sauvées | Accès API de Studio coupé ; pas de `BindToClose` (arrêt du Play) ; profil non chargé ; clé différente ; valeur non sérialisable ; relecture en cache | Avertissements DataStore dans l'Output ; relire sans cache |
| Marche dans Studio, pas en ligne | Streaming (objet absent côté client) ; ordre de chargement ; HTTP ou secrets non configurés ; version non publiée ; mobile | Mode Server & Clients avec le simulateur de réseau ; console en jeu (F9) |
| Marche en solo, pas à plusieurs | Changement fait côté client, non répliqué ; état global non indexé par joueur ; `FireAllClients` au lieu de `FireClient` | Deux clients en local |
| UI qui se chevauche ou déborde sur téléphone | Offsets en pixels ; pas de `ScreenInsets` ; `TextScaled` sans contrainte ; boutons sous le joystick ou le saut | Émulateur : petit téléphone, portrait et paysage |
| Survol qui reste collé au doigt | Survol non réservé à la souris | `UserInputService.PreferredInput` |
| Animation qui ne joue pas | Asset d'un autre propriétaire ; pas d'`Animator` ; priorité `Core` (la plus basse) écrasée ; PNJ animé côté client ; `AnimationId` vide | Output « Failed to load animation » ; propriétaire de l'asset |
| Clic sans effet sur une SurfaceGui | GUI hors du PlayerGui (il faut `Adornee` vers la part) ; part en `CanQuery = false` ; un cadre transparent au-dessus | Emplacement de la GUI, propriétés de la part, ZIndex |
| Achat payé mais rien reçu, ou reçu deux fois | `ProcessReceipt` absent, écrasé ou qui lève ; `PurchaseGranted` avant la sauvegarde ; pas de dédoublonnage par `PurchaseId` | Un seul routeur ? Output pendant un achat test |
| « lacking capability … » ou « is not a valid member » sur un réglage | Réglage réservé à Studio ou aux plugins (2.3) | Champs `Security`, `Capabilities`, `Tags` |
| Lag qui grandit avec le temps | Connexions qui fuient à chaque respawn ; tables par joueur jamais vidées ; travail lourd par frame ; milliers de Parts | Mémoire dans la console F9 après 20 respawns ; MicroProfiler |
| Le code modifié n'a aucun effet | Mauvaise source de vérité : Rojo écrase Studio, `out/` modifié, watcher arrêté, plugin déconnecté | Imprimer un marqueur de version au démarrage |
| `Touched` qui récompense deux fois | Pas d'anti-rebond ; plusieurs parts du personnage touchent | Verrou par joueur |

Quand rien ne colle : lis l'Output avant toute hypothèse. Demande-toi quel côté possède l'état et si un client a pu le provoquer. Change une seule chose par essai, isole par moitiés, retire les sondes à la fin.

---

## 8. Checklist avant de dire fini

- [ ] Chaque API, propriété, enum ou événement peu courant a été vérifié (page `.md`, dump, définitions ou sonde) ; sinon il est marqué « non vérifié ».
- [ ] Aucun nom des tables 2.1 et 2.2, aucune API dépréciée de la section 3 dans le code neuf.
- [ ] Aucun réglage de la table 2.3 écrit par script ; l'utilisateur sait quoi régler à la main dans Studio.
- [ ] Chaque paquet ajouté existe (`npm view`, `wally search`) et l'utilisateur a approuvé son installation.
- [ ] Chaque remote valide débit, type, plage (NaN, infini), permission et plausibilité.
- [ ] Une lecture ratée ne devient jamais un profil vide ; écritures en `UpdateAsync` ou ProfileStore ; `BindToClose` en place.
- [ ] `ProcessReceipt` est idempotent et ne renvoie `PurchaseGranted` qu'après la sauvegarde.
- [ ] Chaque connexion, tâche, tween et instance créée a un propriétaire qui la nettoie.
- [ ] Aucun `-- TODO`, corps vide, `print` de debug ni ancienne version commentée dans le code livré.
- [ ] `rbxtsc` ou `luau-lsp analyze` (avec définitions) passe sans erreur.
- [ ] La console Output a été relue : aucune erreur rouge, aucun nouvel « Infinite yield ».
- [ ] Le code qui écrit dans le moteur a tourné au moins une fois depuis un vrai Script, pas seulement via `execute_luau`.
- [ ] Testé à la taille d'un téléphone, et avec deux clients si l'état est partagé.
- [ ] La revue ne contient que des scénarios d'échec concrets, ou dit « 0 problème ».
- [ ] Le résumé dit ce qui est Vérifié, Automatisé seulement ou Non vérifié.
