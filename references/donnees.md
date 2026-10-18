# Données et persistance

À lire quand : tu sauvegardes ou charges des données joueur, tu touches à un DataStore, à ProfileStore, à un classement, à MemoryStore ou à MessagingService, ou quand un joueur dit « j'ai perdu ma progression ».

## Sommaire

1. Choisir le stockage
2. Les sept règles qui évitent les pertes
3. DataStoreService : opérations et forme des données
4. Limites, budgets et quotas
5. Cache, relectures fraîches et écritures incertaines
6. ProfileStore (choix par défaut)
7. Profil fait main avec verrou de session
8. Schéma versionné et migrations
9. Versions, restauration et droit à l'oubli
10. Classements
11. MemoryStoreService
12. MessagingService
13. Échanges, transferts et récompenses de temps
14. Studio, accès API et tests
15. Diagnostic rapide
16. Pièges
17. Checklist

## 1. Choisir le stockage

Tous ces services sont réservés au serveur. Un appel depuis un LocalScript échoue. Le client passe par un remote validé (voir `securite.md`).

| Besoin | Outil | Pourquoi |
|---|---|---|
| Profil joueur (monnaie, inventaire, progression, réglages) | DataStore standard via ProfileStore, une clé par joueur | durable, versionné, métadonnées |
| Classement de tous les temps | OrderedDataStore | tri natif, entiers seulement |
| Classement du jour, liste de serveurs, file de matchmaking, verrou court | MemoryStoreService | rapide, tout expire (45 jours max) |
| Prévenir les autres serveurs (annonce, invalidation, réveil) | MessagingService | signal sans stockage, livraison non garantie |
| État global non joueur (saison, guilde, compteur) | DataStore standard, clés à préfixe | durable, mais aucun verrou de session |
| Réglage global sans republier (flag, équilibrage) | ConfigService (Configs du Creator Hub) | lecture seule en jeu, serveur seulement, propagation en 15 s à 1 min environ après « Publish now » (doc, 2026-10-18) ; voir `design-liveops.md` |
| Clé d'API tierce | Secrets Store + `HttpService:GetSecret` | jamais en clair ; voir `open-cloud.md` |
| État de manche ou de serveur | table Luau en mémoire | gratuit, instantané, perdu à l'arrêt |

Toutes les places d'une expérience partagent ses DataStores et ses MemoryStores : une place de staging dans la même expérience écrit sur les données live. Un message « travail disponible » s'accompagne d'une vraie file en Queue : le message seul se perd.

## 2. Les sept règles qui évitent les pertes

1. **Une lecture ratée n'est pas un profil vide.** Si `GetAsync`, `UpdateAsync` ou `StartSessionAsync` échoue, ne mets jamais les valeurs par défaut : la sauvegarde suivante écraserait la vraie progression. Expulse avec un message (« reviens dans une minute ») ou ouvre une session en lecture seule qui ne sauvegarde jamais. Seul un `nil` renvoyé par une lecture *réussie* veut dire « nouveau joueur ».
2. **Un seul serveur possède un profil à la fois** (verrou de session). Sans verrou, un joueur qui change vite de serveur duplique ses objets ou perd des écritures. C'est la première cause de duplication, et des joueurs la déclenchent exprès.
3. **Écris avec `UpdateAsync`, et regarde l'ancienne valeur.** `SetAsync` écrase en aveugle l'écriture d'un autre serveur. Un `UpdateAsync` dont le callback ignore la valeur reçue ne vaut pas mieux.
4. **Charge une fois, joue en mémoire, sauvegarde à des frontières** : autosave périodique, événement important (achat, échange), `PlayerRemoving`, `BindToClose`. Jamais à chaque pièce ramassée ni à chaque frame : les quotas sautent.
5. **Chaque appel dans un `pcall`, avec des réessais bornés** : 3 à 5 essais, attente doublée à chaque fois, un peu d'aléa. Un réessai infini bloque la session et brûle le budget. Logue chaque échec : un `pcall` muet est la cause n°1 de « ça ne sauvegarde pas ».
6. **Versionne le schéma dès le premier jour** et migre au chargement. Garde le même nom de store : un nouveau nom est un store vide, donc tout le monde repart de zéro.
7. **Une seule voie d'écriture par profil.** Autosave, achats et départ passent par le même module, la même clé et la même stratégie. Un `SetAsync` d'autosave à côté effacerait les reçus déjà enregistrés.

## 3. DataStoreService : opérations et forme des données

| Méthode | Retour | À savoir |
|---|---|---|
| `GetAsync(key, options?)` | `(value, keyInfo)` | `nil` si jamais écrite ou supprimée ; cache de 4 s (section 5) |
| `SetAsync(key, value, userIds?, options?)` | id de version | dernier écrivain gagne ; budget d'écriture seul |
| `UpdateAsync(key, transform)` | `(newValue, keyInfo)` | lecture-modification-écriture atomique ; budgets lecture **et** écriture |
| `IncrementAsync(key, delta?, userIds?, options?)` | `(total, keyInfo)` | compteur entier atomique |
| `RemoveAsync(key)` | `(oldValue, keyInfo)` | suppression douce : ajoute une version « pierre tombale » |
| `ListKeysAsync(prefix?, pageSize?, cursor?, excludeDeleted?)` | pages | audits et migrations ; budget de listage |

`DataStoreKeyInfo` porte `Version`, `CreatedTime`, `UpdatedTime` (ms Unix), `GetUserIds()` et `GetMetadata()`. `GetGlobalDataStore()` et `OnUpdate` (déprécié) sont des restes : pas dans du code neuf.

Le callback d'`UpdateAsync` reçoit `(current, keyInfo)` et renvoie `(newValue, userIds?, metadata?)` :
- **Aucun yield dedans** : ni `task.wait`, ni autre appel `Async`. Fais le travail lent avant.
- **Pur et rejouable** : le moteur le rappelle si un autre serveur a écrit entre-temps. Ni remote, ni badge, ni HTTP, ni mutation extérieure. Remets tes variables d'état à zéro au début de chaque passage.
- **Renvoyer `nil` annule** : rien n'est écrit et `UpdateAsync` renvoie `nil`. Teste `ok and result ~= nil`, pas seulement `ok`.
- **Redonne userIds et metadata** à chaque écriture, sinon ils sont effacés. `keyInfo` vaut `nil` sur une clé neuve : vérifie-le avant `keyInfo:GetMetadata()`.
- En roblox-ts, plusieurs valeurs de retour passent par `$tuple(value, [userId])`, et `undefined` annule (vérifie les types de ta version de `@rbxts/types`).

- **Peu de stores, une clé cohérente par joueur** (`player_<UserId>`) qui contient tout le profil : une écriture garde l'ensemble cohérent, une version restaure un instantané complet. Jamais un store par joueur, jamais une clé pour tous.
- Range avec des préfixes de clés (`guild/9/config`) plutôt qu'avec des scopes (historiques), puis filtre avec `ListKeysAsync("guild/")`.
- Clé sur `UserId`, jamais sur le nom d'affichage. Ne passe pas à `Player.User.Id` : propre à l'expérience et d'une autre valeur, il couperait les joueurs de leurs sauvegardes. `UserId` n'est pas déprécié.

**Forme des données.** La valeur est sérialisée en JSON. Une seule valeur invalide fait échouer **toute** l'écriture, avec une erreur qui ne nomme pas le champ : garde les données de session sérialisables dès le départ.
- Autorisé : `nil`, booléens, nombres finis, chaînes UTF-8 valides, `buffer`, tables de ces types. Interdit : `NaN`, `±inf`, Instances, fonctions, métatables, cycles, types Roblox (`Vector3`, `CFrame`, `Color3`, `EnumItem`).
- Convertis à la frontière : `{ cf:GetComponents() }` et `CFrame.new(table.unpack(t))`, `color:ToHex()` et `Color3.fromHex(hex)`, `material.Name` relu après contrôle du nom.
- Tableau contigu `1..n` ou dictionnaire à clés chaînes, jamais un mélange : un `nil` au milieu d'un tableau tronque la suite. Clés numériques d'un dictionnaire : `tostring` à la sauvegarde, `tonumber` au chargement.
- Un octet isolé au-dessus de 127 casse l'écriture : vérifie le texte joueur avec `utf8.len`. Valide chaque nombre : `x == x` (faux pour NaN), `math.abs(x) ~= math.huge`, entier et borné si attendu.
- Sauve des identifiants (`"IronSword"`) et des noms plutôt que des index (insérer un palier décale les index sauvés), jamais des Instances ni des définitions complètes.
- Ne compresse pas toi-même : Roblox compresse et mesure le quota sur la taille compressée. `#HttpService:JSONEncode(data)` donne un ordre de grandeur. Plafonne les tableaux qui grossissent sans fin (journaux, historiques).

## 4. Limites, budgets et quotas

Chiffres de la page officielle « error codes and limits », relevés entre septembre et le 18 octobre 2026. Ils bougent : revérifie avant de dimensionner, et lis le budget restant à l'exécution plutôt que de le deviner.

| Élément | Limite |
|---|---|
| Nom de store, clé, scope | 50 caractères chacun |
| Valeur d'une clé | 4 194 304 caractères (4 Mo) après sérialisation ; au-delà, erreur 105 |
| Metadata personnalisée | 300 caractères au total, 250 par valeur ; erreur 511 |
| UserIds par clé | 4 selon d'anciens repères ; la doc ne la chiffre pas (« X ») ; erreur 512 (à vérifier) |
| Débit d'une clé (60 s glissantes, tous serveurs) | 25 Mo/min en lecture, 4 Mo/min en écriture, requête arrondie au Ko supérieur ; une clé chaude bute dessus avant le budget de requêtes (ancien repère : une écriture toutes les 6 s environ) |

**Budgets de requêtes par minute** (CCU = joueurs connectés dans toute l'expérience, n = joueurs du serveur). Le budget d'expérience vaut aussi pour les stores ordonnés et inclut Open Cloud. Toutes les colonnes vérifiées (doc, 2026-10-18).

| Type | Expérience | Serveur, standard | Serveur, ordonné |
|---|---|---|---|
| Lecture (`Get`, `GetVersion*`, lecture d'`Update`, `BatchGet`) | 300 + 40 × CCU | 60 + 40 × n | 60 + 40 × n |
| Écriture (`Set`, `Increment`, écriture d'`Update`) | 300 + 20 × CCU | 60 + 40 × n | 30 + 5 × n |
| Listage (`ListKeys`, `ListVersions`, `GetSorted`) | 300 + 2 × CCU | 5 + 2 × n | 5 + 2 × n |
| Suppression | 300 + 40 × CCU | 60 + 40 × n | 30 + 5 × n |

- Valeurs périmées encore citées : par expérience (avril 2026), 250 + 40 × CCU en lecture, 250 + 20 × CCU en écriture, 10 + 2 × CCU en listage, 100 + 40 × CCU en suppression ; par serveur, l'ancienne formule 60 + 10 × n en lecture et en écriture. Si tu doutes, dimensionne sur l'ancienne : elle est plus prudente.
- Depuis le 2026-07-29, le jeu et Open Cloud partagent le budget d'expérience : un gros job externe peut affamer les serveurs live. Lance-le hors des heures de pointe, et préviens.
- Quand tu donnes ces chiffres à l'utilisateur, dis-lui qu'ils changent souvent et donne-lui la page à revérifier : https://create.roblox.com/docs/cloud-services/data-stores/error-codes-and-limits.md. En jeu, lis le budget réel avec `DataStoreService:GetRequestBudgetForRequestType` plutôt que de coder une formule en dur.
- `DataStoreService:GetRequestBudgetForRequestType(Enum.DataStoreRequestType.StandardWrite)` renvoie les requêtes restantes de ce serveur pour la minute. Consulte-le avant une rafale (migration, sauvegarde de masse) et diffère si c'est bas. Pour `OrderedList`, il renverrait toujours 0 : n'attends pas dessus (à vérifier).
- `DataStoreService:SetRateLimitForRequestType(requestType, base, perPlayer)` règle le plafond du serveur, une fois au démarrage. Les types `Standard*` et `Ordered*` montent jusqu'à 10 000 + 200 × n ; les anciens types (`GetAsync`, `GetSortedAsync`…) ne peuvent que baisser. `UpdateAsync` n'est pas réglable (erreur).
- Les files d'attente (get, set et leurs équivalents ordonnés) contiennent 30 requêtes chacune. File pleine : abandon avec une erreur 301 à 306, donc une rafale échoue au lieu d'attendre. Studio en mode Run a ses propres limites, parfois plus basses : teste le throttling en Team Create (conseil de la doc) ou sur une place publiée de test.

**Stockage** : 500 Mo + 1 Mo × utilisateurs « à vie » (tout joueur venu une fois), annoncé le 2026-07-15, en vigueur depuis le 2026-07-29 ; ancienne valeur (avril 2026) : 100 Mo + 1 Mo par utilisateur. Seule la dernière version de chaque clé compte, mesurée compressée ; versions remplacées et clés supprimées ne comptent pas, sauf un store supprimé via Open Cloud pendant ses 30 jours de traitement. Un jeu neuf n'a guère plus que la base de 500 Mo. Alertes automatiques à 60, 80, 90 et 100 % ; au-delà de 100 %, Data Stores Manager affiche un coût mensuel estimé. Sans Extended Services, ce qui arrive aux écritures au-delà n'est pas documenté (à vérifier). Avec Extended Services (payant), le surplus est facturé (0,12 $ par Go et par mois d'après sa page, qui cite encore 100 Mo de base) et le service est bridé au-delà du budget fixé : décision business, pas un pansement pour un code qui gaspille.

**Erreurs.** 101 à 107 : bug de code (clé vide ou trop longue, valeur interdite comme « double is not allowed » sur un store ordonné, valeur trop grosse), ne réessaie pas. 301 à 306 : file pleine, backoff. 401 et 402 : service coupé pendant l'arrêt. 403 : Studio sans accès API (section 14). 404 et 5xx : erreur interne ; une lecture se réessaie, une écriture a un résultat inconnu (section 5). 511 et 512 : metadata ou UserIds en trop ; 513 : format invalide (userIds non numériques, metadata qui n'est pas une table). Throttles récents : `…ExperienceThrottled` (budget d'expérience), `…GameServerThrottled` (budget du serveur), `KeyThrottled` (débit de la clé).

Les API web DataStore appelées par cookie ou sans authentification sont fortement bridées à partir du 2026-10-19 : passe par Open Cloud (`open-cloud.md`).

## 5. Cache, relectures fraîches et écritures incertaines

**Cache de 4 s.** `GetAsync` garde la valeur 4 s en cache local, par instance de store : lecture gratuite, mais peut-être périmée. Les écritures sur la même instance mettent ce cache à jour ; deux instances (autre scope, `AllScopes`) ont deux caches. L'écriture d'un autre serveur reste invisible jusqu'à 4 s. Listages et versions lisent toujours le serveur. Pour une lecture autoritaire, passe un `DataStoreGetOptions` avec `UseCache = false` (exemple dans `applyOnce` ci-dessous) : elle coûte une vraie requête.

**Une écriture qui renvoie une erreur a un résultat inconnu, pas « refusé ».** Le backend a pu l'appliquer. Trois états :
- `committed` : succès confirmé.
- `rejected` : refusée avant l'envoi (validation, budget épuisé) ou erreur permanente (101 à 107). Ne réessaie pas.
- `unknown` : erreur après l'envoi. Ne rembourse pas, n'accorde pas une seconde fois, ne dis pas au joueur que c'est raté. Relis avec `UseCache = false` pour trancher : le cache te ferait croire que rien n'est passé.

**Rends chaque opération rejouable** : un identifiant d'opération créé par le serveur avant le premier essai, écrit dans la même clé et le même `UpdateAsync` que l'effet ; l'effet n'est appliqué que si l'identifiant est absent.

```luau
--!strict
-- Applique `apply` une seule fois par opId. Pour une clé qu'aucune session ne possède
-- (état global, cadeau hors ligne). Renvoie "committed", "rejected" ou "unknown".
local function applyOnce(store: DataStore, key: string, opId: string, apply: (any) -> ()): string
	local ok, result = pcall(function()
		return store:UpdateAsync(key, function(current: any, keyInfo: DataStoreKeyInfo?)
			if type(current) ~= "table" or keyInfo == nil then
				return nil -- rien à modifier : on n'invente pas de valeur
			end
			current.ops = current.ops or {}
			if current.ops[opId] == nil then
				apply(current) -- pur, sans yield
				current.ops[opId] = os.time()
			end
			return current, keyInfo:GetUserIds(), keyInfo:GetMetadata() -- sinon userIds et metadata sont effacés
		end)
	end)
	if ok then
		return if result ~= nil then "committed" else "rejected"
	end
	local fresh = Instance.new("DataStoreGetOptions")
	fresh.UseCache = false
	local readOk, value = pcall(function()
		return store:GetAsync(key, fresh)
	end)
	if readOk and type(value) == "table" and type(value.ops) == "table" and value.ops[opId] then
		return "committed" -- l'écriture « ratée » était passée
	end
	return "unknown" -- garde opId et rejoue plus tard avec le MÊME identifiant
end
```

- Une erreur 101 à 107 (code en tête du message) revient à chaque rejeu : classe-la `rejected`, pas `unknown`.
- Pour savoir si c'est *cet* appel qui a appliqué l'effet (notifier une seule fois), pose un drapeau local remis à `false` au début de chaque passage du transform et mis à `true` quand `apply` tourne.
- Ne rejoue jamais `IncrementAsync` automatiquement après une erreur : il n'est pas idempotent. Garde l'ordre par clé : un vieux réessai ne doit pas écraser une sauvegarde plus récente.
- Effet externe (HTTP, webhook) : écris d'abord une entrée « à livrer » dans la même écriture, puis livre-la avec la même clé d'idempotence. Ne purge le registre `ops` qu'au-delà d'un âge bien plus long que tes délais de rejeu.

## 6. ProfileStore (choix par défaut)

ProfileStore (loleris, MadStudioRoblox) remplace ProfileService. Il verrouille la session, réessaie, sauvegarde toutes les 300 s (réglable avec `ProfileStore.SetConstant`) et gère `BindToClose`. Utilise-le pour tout profil joueur, sauf raison précise. Installation en Luau ou en roblox-ts : `paquets.md`. Alternatives : Lapis (verrou de session, courant avec roblox-ts), Suphi's DataStore Module. N'en mets jamais deux sur les mêmes clés.

```luau
--!strict
-- ServerScriptService/PlayerProfiles.luau (ModuleScript). Un Script appelle PlayerProfiles.start().
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local ServerScriptService = game:GetService("ServerScriptService")
local ProfileStore = require(ServerScriptService.Packages.ProfileStore) -- adapte le chemin
local Schema = require(script.Parent.Schema) -- section 8

local storeName = if RunService:IsStudio() then "PlayerData_Studio" else "PlayerData"
local PlayerStore = ProfileStore.New(storeName, Schema.defaults())
local profiles: { [number]: any } = {} -- UserId → profil actif
local closing = false
local PlayerProfiles = {}

local function onPlayerAdded(player: Player)
	local userId, startedAt = player.UserId, os.clock()
	local profile = PlayerStore:StartSessionAsync(`player_{userId}`, {
		-- Fonction interrogée en boucle. La fournir retire le délai intégré : on remet le nôtre.
		Cancel = function()
			return player.Parent ~= Players or closing or os.clock() - startedAt > 30
		end,
	})
	if profile == nil then
		if player.Parent == Players then
			player:Kick("Ta sauvegarde n'est pas disponible. Reviens dans une minute.")
		end
		return -- jamais de profil vide à la place
	end
	local ok, migrated = pcall(Schema.migrate, Schema.deepCopy(profile.Data)) -- une copie : un échec à mi-chemin ne sera pas sauvé
	if not ok then
		warn(`[Profiles] {userId} illisible : {migrated}`)
		profile:EndSession() -- réécrit la donnée intacte et rend le verrou
		player:Kick("Ta sauvegarde n'a pas pu être lue sans risque. Reviens dans un instant.")
		return
	end
	table.clear(profile.Data)
	for key, value in migrated do
		profile.Data[key] = value
	end
	profile:Reconcile() -- ajoute les champs nouveaux du gabarit
	profile:AddUserId(userId) -- droit à l'oubli
	profile.OnSessionEnd:Connect(function()
		profiles[userId] = nil
		if player.Parent == Players then
			player:Kick("Ta session de sauvegarde s'est fermée. Reviens.")
		end
	end)
	if player.Parent ~= Players then
		profile:EndSession() -- parti pendant le chargement
		return
	end
	profiles[userId] = profile
	player:SetAttribute("DataLoaded", true) -- le client attend ce signal avant d'afficher un solde
end

function PlayerProfiles.start()
	game:BindToClose(function()
		closing = true -- ProfileStore sauvegarde lui-même les sessions ouvertes
	end)
	for _, player in Players:GetPlayers() do
		task.spawn(onPlayerAdded, player) -- joueurs arrivés avant ce script
	end
	Players.PlayerAdded:Connect(onPlayerAdded)
	Players.PlayerRemoving:Connect(function(player)
		local profile = profiles[player.UserId]
		if profile then
			profile:EndSession()
		end
	end)
end

-- Profil actif, ou nil (« pas chargé », jamais « vide »).
function PlayerProfiles.get(userId: number): any
	local profile = profiles[userId]
	return if profile and profile:IsActive() then profile else nil
end

return PlayerProfiles
```

- Le gameplay modifie `profile.Data` directement : pas de deuxième cache, pas d'appel DataStore à côté sur ces clés. Après un yield, revérifie `profile:IsActive()`. Achats : attends la sauvegarde confirmée avant d'acquitter (motif complet dans `monetisation.md`).
- `Steal = true` contourne le verrou : débogage contrôlé seulement. `PlayerStore.Mock` (chaque store renvoyé par `ProfileStore.New` a son `.Mock`) a la même API sans rien écrire : utile en Studio.
- Cadeau payé à livrer hors ligne : `PlayerStore:MessageAsync(key, message)` et `profile:MessageHandler(...)`. Annonce sans enjeu : MessagingService.
- ProfileStore ne gère pas pour toi le budget de l'expérience, le cache de 4 s de tes lectures brutes, ni le RTBF (clé avec l'UserId et `AddUserId`).
- Depuis ProfileService : `GetProfileStore` → `ProfileStore.New`, `LoadProfileAsync` → `StartSessionAsync`, `Release` → `EndSession`, `ListenToRelease` → `OnSessionEnd`, `Reconcile` reste. La doc de ProfileStore annonce des profils rétrocompatibles avec ProfileService (mêmes clés), mais après un `MessageAsync` ProfileService peut ne plus les relire : teste d'abord en Studio avec l'accès API, sur une copie.
- DataStore2 (ancien, sans verrou de session) : lis une dernière fois l'ancienne donnée, fusionne-la dans le profil, pose `migratedFromDS2 = true`, garde l'ancien chemin de lecture au moins deux semaines. Un verrouillage de session et des schémas intégrés seraient sur la feuille de route de Roblox pour début 2027, d'après une source tierce ; aucune annonce officielle n'a été trouvée au 2026-10-18 (à vérifier). D'ici là, c'est au projet de le faire.

## 7. Profil fait main avec verrou de session

Seulement si une bibliothèque est exclue. Le verrou vit **dans la même clé que les données** et se vérifie dans le même `UpdateAsync` que chaque écriture : rangé ailleurs, il ne serait pas atomique avec le contrôle. Le chargement prend le verrou (s'il est absent ou périmé) et migre dans le même passage. Chaque écriture vérifie que le jeton est toujours le sien, sinon elle annule et la session s'arrête. Chaque autosave rafraîchit l'horodatage : le verrou d'un serveur planté se périme seul. La sauvegarde finale écrit les données et rend le verrou d'un coup.

```luau
--!strict
-- ServerScriptService/PlayerData.luau (ModuleScript) : le cœur du protocole.
local DataStoreService = game:GetService("DataStoreService")
local Schema = require(script.Parent.Schema) -- section 8

local STORE = DataStoreService:GetDataStore("PlayerData")
local LOCK_STALE_SECONDS = 180 -- trois autosaves de 60 s manqués : le propriétaire est mort

export type Session = { userId: number, token: string, data: { [string]: any }, saveable: boolean }

-- Réessais bornés. Rejouer est sûr : chaque transform revérifie le jeton.
local function update(userId: number, transform: (any, DataStoreKeyInfo?) -> ...any): (boolean, any)
	for attempt = 1, 4 do
		local ok, result = pcall(function()
			return STORE:UpdateAsync(`player_{userId}`, transform)
		end)
		if ok then
			return true, result
		end
		warn(`[PlayerData] {userId}, essai {attempt} : {result}`)
		if attempt < 4 then
			task.wait(2 ^ (attempt - 1) + math.random() * 0.5) -- 1 s, 2 s, 4 s, plus un peu d'aléa
		end
	end
	return false, nil
end

local function newLock(token: string)
	return { token = token, at = os.time() }
end

-- Prend le verrou et charge. Statut : "ok", "locked", "unreadable" ou "error".
local function acquire(userId: number, token: string): (string, { [string]: any }?)
	local status = "ok"
	local ok, stored = update(userId, function(current: any, keyInfo: DataStoreKeyInfo?)
		status = "ok" -- le transform peut être rejoué : repartir de zéro à chaque passage
		if current == nil then
			current = Schema.defaults() -- lecture réussie et clé vide : vrai nouveau joueur
		elseif type(current) ~= "table" then
			status = "unreadable"
			return nil
		end
		local lock = current._lock
		if type(lock) == "table" and lock.token ~= token and os.time() - (tonumber(lock.at) or 0) < LOCK_STALE_SECONDS then
			status = "locked" -- un serveur vivant possède ce profil
			return nil
		end
		if not pcall(Schema.migrate, current) then
			status = "unreadable" -- version future ou donnée cassée : ne rien écrire
			return nil
		end
		Schema.reconcile(current, Schema.defaults())
		current._lock = newLock(token)
		return current, { userId }, if keyInfo then keyInfo:GetMetadata() else nil -- garde la metadata
	end)
	if not ok then
		return "error", nil
	end
	if type(stored) ~= "table" then
		return status, nil
	end
	stored._lock = nil -- le verrou n'est pas une donnée de jeu
	return "ok", stored
end

-- Écrit la session ; release = true rend le verrou. Statut : "committed", "lost" ou "error".
local function write(session: Session, release: boolean): string
	local lost = false
	local ok, stored = update(session.userId, function(current: any, keyInfo: DataStoreKeyInfo?)
		lost = false
		local lock = if type(current) == "table" then current._lock else nil
		if type(lock) ~= "table" or lock.token ~= session.token then
			lost = true -- plus propriétaire : écrire écraserait l'autre serveur
			return nil
		end
		local record = table.clone(session.data)
		record._lock = if release then nil else newLock(session.token)
		return record, { session.userId }, if keyInfo then keyInfo:GetMetadata() else nil
	end)
	if ok and stored ~= nil then
		return "committed"
	end
	return if ok and lost then "lost" else "error"
end
```

Autour de ces deux fonctions :
- **Arrivée** : `acquire` avec un jeton neuf par session (`HttpService:GenerateGUID(false)`), pas par serveur, pour qu'un joueur revenu sur le même serveur avant la fin de sa sauvegarde finale ne reprenne pas un vieux verrou. `"locked"` : réessaie jusqu'à 5 fois à 5 s d'écart (l'ancien serveur rend peut-être le verrou), puis expulse. `"unreadable"` ou `"error"` : expulse tout de suite, sans rien écrire. Charge aussi les joueurs déjà présents (`Players:GetPlayers()`). Compte chaque chargement en cours dans `inFlight` (plus bas) ; s'il finit après le départ du joueur ou après `closing = true`, `releaseSession(session)` tout de suite, sinon le verrou bloque le joueur 180 s.
- **Autosave** : toutes les 60 s, étalé entre les joueurs, même sans changement. `"lost"` : `saveable = false`, retire la session, expulse (« partie ouverte sur un autre serveur »). `"error"` : réessaie au cycle suivant.
- **Départ** : retire la session de la table **avant** de céder, puis `releaseSession(session)` (plus bas). `PlayerRemoving` et `BindToClose` n'écrivent ainsi qu'une fois.
- **Sauvegarde immédiate** (achat, échange) : `write(session, false) == "committed"` prouve que tout ce qui était en mémoire avant l'appel est durable. C'est la condition d'acquittement d'un reçu (`monetisation.md`).

**Arrêt du serveur.** `BindToClose` laisse 30 s au total, partagées par tous les callbacks. Sauvegarde en parallèle et vise 25 s : en série, à 1 ou 2 s par joueur, seuls 15 à 30 joueurs passent. Attends aussi les sauvegardes finales déjà lancées par `PlayerRemoving`.

```luau
-- Suite de PlayerData.luau.
local sessions: { [number]: Session } = {}
local inFlight, closing = 0, false

local function releaseSession(session: Session)
	session.saveable = false
	inFlight += 1 -- compte aussi les départs lancés par PlayerRemoving
	local status = write(session, true)
	inFlight -= 1
	if status ~= "committed" then -- "lost" peut venir d'un premier essai passé malgré son erreur
		warn(`[PlayerData] sauvegarde finale de {session.userId} : {status}`)
	end
end

game:BindToClose(function()
	closing = true -- coupe l'autosave et les nouvelles sessions
	for userId, session in sessions do
		sessions[userId] = nil -- retirée avant de céder : une seule écriture finale
		task.spawn(releaseSession, session) -- en parallèle
	end
	local deadline = os.clock() + 25
	while inFlight > 0 and os.clock() < deadline do
		task.wait(0.1)
	end
end)
```

En Studio, `BindToClose` tourne à l'arrêt du playtest et fige Studio tant qu'il cède : garde-le borné. Le signal `game.ServerRestartScheduled` (arguments `restartTime: DateTime`, `source: Enum.CloseReason`, `attributes`) prévient d'un redémarrage planifié (`DeveloperUpdate` pour « Restart Servers », `RobloxMaintenance`) : stoppe les nouvelles manches, verrouille échanges et files, affiche un compte à rebours et sauvegarde pendant que les joueurs sont là. Téléportation : sauvegarde et rends le verrou, *puis* téléporte ; le serveur d'arrivée réessaie tant que le verrou est tenu. `TeleportData` n'est qu'un indice, et attendre 0,5 s ne garantit rien (`gameplay.md`).

## 8. Schéma versionné et migrations

Un seul module connaît la forme sauvegardée : version, gabarit, migrations.

```luau
--!strict
-- ServerScriptService/Schema.luau
local Schema = {}
Schema.CURRENT_VERSION = 2

function Schema.deepCopy<T>(value: T): T
	local raw: any = value -- affiner `value` lui-même ferait échouer le nouveau solveur (T & ~table)
	if type(raw) ~= "table" then
		return value
	end
	local copy = {}
	for key, item in raw do
		copy[key] = Schema.deepCopy(item)
	end
	return (copy :: any) :: T
end

-- Une table neuve à chaque appel : aucune sous-table partagée entre joueurs.
function Schema.defaults(): { [string]: any }
	return {
		version = Schema.CURRENT_VERSION,
		coins = 0,
		gems = 0,
		inventory = {}, -- [itemId] = quantité
		receiptIds = {}, -- derniers PurchaseId accordés (voir monetisation.md)
		settings = { music = 0.5, sfx = 0.8 },
		lastDailyClaim = 0,
	}
end

-- migrations[n] fait passer de la version n à n + 1. Ne modifie jamais une étape publiée.
-- Un simple ajout de champ (gems) n'a pas besoin d'étape : reconcile s'en charge.
local migrations: { [number]: (data: { [string]: any }) -> () } = {
	[1] = function(data) -- v1 → v2 : « Coins » devient « coins », liste d'objets → quantités
		data.coins = data.Coins or 0
		data.Coins = nil
		local counts: { [string]: number } = {}
		for _, itemId in data.inventory or {} do
			counts[itemId] = (counts[itemId] or 0) + 1
		end
		data.inventory = counts
	end,
}

-- Modifie data sur place et la renvoie. Lève une erreur si la donnée vient du futur.
function Schema.migrate(data: { [string]: any }): { [string]: any }
	local version = data.version or 1 -- sans champ version : v1
	if type(version) ~= "number" or version > Schema.CURRENT_VERSION then
		error(`version {version} inconnue de ce serveur`)
	end
	while version < Schema.CURRENT_VERSION do
		local step = migrations[version]
		if step == nil then
			error(`migration manquante depuis la v{version}`)
		end
		step(data)
		version += 1
		data.version = version -- estampillée seulement après l'étape
	end
	return data
end

-- Ajoute les champs absents du gabarit sans écraser l'existant.
function Schema.reconcile(target: { [string]: any }, template: { [string]: any })
	for key, value in template do
		local current = target[key]
		if current == nil then
			target[key] = Schema.deepCopy(value)
		elseif type(value) == "table" and type(current) == "table" and next(value) ~= nil then
			Schema.reconcile(current, value) -- tables vides du gabarit (inventaire) : clés libres, on n'y touche pas
		end
	end
end

return Schema
```

- `table.clone(DEFAULTS)` et `data or DEFAULTS` partagent les sous-tables : un objet ajouté à un joueur apparaît chez tous. D'où `defaults()` qui construit tout à neuf.
- Une étape par version, dans l'ordre, gardée pour toujours. Ajout : défaut inséré. Renommage : transformé dans l'étape, jamais lu sous deux noms dans le jeu. Suppression : retirée dans l'étape.
- **Refuse les données du futur.** Pendant une mise à jour progressive, anciens et nouveaux serveurs tournent ensemble. Un vieux serveur qui lit une v4 ne la « répare » pas et ne la réécrit pas : il expulse. Pour la même raison, ne supprime ni ne renomme un champ sur place sans migration.
- Ordre au chargement : migrer, réconcilier (les ajouts), assainir, puis seulement exposer au gameplay. Le client ne déclenche jamais de migration.
- Assainir : borne les nombres, remplace les ids d'objets supprimés, tronque une liste plus longue qu'un nouveau plafond dans un ordre fixe (réclamées, commencées, reste), garde les déblocages des anciens joueurs quand une règle change.
- Sauve l'état non dérivable (choix, inventaire, graine) ; recalcule le dérivé (stats = base + modificateurs), qui diverge sinon quand la formule change. Sauve les compteurs des badges à paliers même avant que le badge existe : l'activer plus tard l'accordera au join.
- Sépare données de « run » (remises à zéro par la renaissance) et « méta » (permanentes) dans deux sous-tables : une liste blanche de champs à garder finit par oublier la monnaie premium.
- Au-delà de 2^53 (environ 9 × 10^15), les entiers ne sont plus exacts : mantisse + exposant, à l'affichage comme au stockage.

## 9. Versions, restauration et droit à l'oubli

- Chaque clé d'un store standard est versionnée. La première écriture de chaque heure UTC crée une version ; les suivantes dans la même heure l'écrasent. Une version remplacée expire 30 jours après ; la dernière n'expire jamais. Un autosave serré n'ajoute donc aucune « sauvegarde », et une clé de backup par sauvegarde gaspille du quota : l'historique fait ce travail.
- `ListVersionsAsync(key, sortDirection?, minDate?, maxDate?, pageSize?)`, `GetVersionAsync(key, version)`, `GetVersionAtTimeAsync(key, timestampMs)` (au plus 10 min dans le futur). `RemoveVersionAsync` est déprécié : ne le propose pas. `RemoveAsync` laisse les anciennes versions lisibles pendant 30 jours : une suppression accidentelle se rattrape.
- **Restaurer** : `ListVersionsAsync(key, Enum.SortDirection.Descending, nil, when.UnixTimestampMillis)`, prends le premier élément, relis-le avec `GetVersionAsync`, retire un éventuel vieux `_lock`, puis réécris-le avec `SetAsync(key, value, info:GetUserIds(), options)` après `options:SetMetadata(info:GetMetadata())`. La restauration devient une version traçable.
- Restaure seulement quand aucune session ne possède la clé : une écriture d'admin ou d'Open Cloud pendant qu'un serveur tient la session est écrasée au prochain autosave, à coup sûr. Passe par le serveur propriétaire, ou attends le départ du joueur et la fin du verrou.
- Avant une publication risquée, prends un instantané des stores avec l'API Open Cloud `SnapshotDataStores` (`open-cloud.md`) : un seul par expérience et par jour UTC, un second appel le même jour ne fait rien. Data Stores Manager (Creator Hub) : parcourir par préfixe, comparer deux versions, « Revert » une clé ; « Restore » ne sert qu'aux stores marqués pour suppression. Metadata utile : version de schéma, source, étiquette d'événement.

**Droit à l'oubli (RTBF).** Roblox transmet les demandes d'effacement ; tu as 30 jours.
- Configure dès maintenant jusqu'à 100 modèles de suppression : Creator Hub → Data Stores Manager → onglet RTBF, ou API Open Cloud Configs. Le motif contient le jeton `{UserId}`, sensible à la casse (`{userId}` est refusé), dans la clé, le scope ou le nom du store : `player_{UserId}`, scope `global` par défaut. Une clé hachée, ou une identité cachée dans un blob partagé, rend chaque demande manuelle.
- Passe `{ player.UserId }` en `userIds` à chaque écriture (ProfileStore : `profile:AddUserId`).
- Les modèles couvrent aussi les stores ordonnés (type `ORDERED`) ; seule la suppression d'un store entier est réservée aux stores standard. Données hors modèles (clé sans UserId, blob partagé, stockage externe) : webhook « Right to Erasure Request », en vérifiant `Roblox-Signature` (`open-cloud.md`). Décide si l'effacement couvre aussi l'historique des versions.
- Suffixe le store selon l'environnement (`PlayerData_Studio`) sans changer le format de clé, sinon les modèles ne correspondent plus. Teste-les avec un compte factice.

## 10. Classements

| Classement | Outil |
|---|---|
| Dans le serveur | table en mémoire |
| Global, de tous les temps | OrderedDataStore |
| Vivant entre serveurs, du jour ou de la semaine | MemoryStore sorted map qui expire en fin de période |

- Valeurs **entières** seulement : `1.5` lève « 103: double is not allowed », `"7"` lève « string is not allowed » ; les négatifs passent (mesuré en Studio 0.742). Multiplie par 100 pour deux décimales ; un temps se stocke en millisecondes.
- Ni versions, ni metadata, ni UserIds ; `RemoveAsync` y est définitif. Pas d'effacement en masse : pour une remise à zéro, change de clé ou de store par période (`Wins_2026W42`), avec une date calculée par le serveur.
- `GetSortedAsync(ascending, pageSize, minValue?, maxValue?)`, `pageSize` de 1 à 100. Pages : `GetCurrentPage()` (`key`, `value`), `IsFinished`, `AdvanceToNextPageAsync()` (cède : `pcall`). `BatchGetAsync(keys)` (store ordonné seulement, il lève sur un store standard ; 100 clés au plus par défaut) renvoie `{ [clé] = { value = n } }`, clés absentes omises, une lecture ordonnée par clé.
- Le profil reste dans le store standard ; le store ordonné ne contient que la métrique classée. Écris-la avec `SetAsync(key, math.floor(value))` (clé `player_<UserId>`) dans un `pcall`, à cadence limitée (fin de manche, palier), depuis la valeur du profil.
- Lecture du top : `GetSortedAsync(false, count)` (false = décroissant) dans un `pcall`, toutes les 60 à 120 s, sur le serveur. Parcours `GetCurrentPage()`, retrouve l'UserId avec `tonumber(string.match(entry.key, "^player_(%d+)$"))`, diffuse l'instantané aux clients. Lecture ratée : garde le dernier bon instantané.
- Le budget d'écriture ordonnée par serveur est petit (30 + 5 × n) : vérifie `OrderedWrite` avant une rafale. Mets les noms en cache (`Players:GetNameFromUserIdAsync` dans un `pcall`), ou résous toute la page en un appel avec `UserService:GetUserInfosByUserIdsAsync` ; portraits par `rbxthumb://` (ui-fondamentaux.md, section 8).
- Plusieurs serveurs écrivent la même entrée (record de course) : `UpdateAsync` qui garde le meilleur. Rang exact de n'importe qui ou tri complexe : un store standard chargé en cache et mis à jour par lots.

### Classement du serveur : `leaderstats`

- La liste des joueurs de Roblox affiche le dossier `leaderstats` de chaque `Player`. C'est un **miroir d'affichage** : le serveur le crée et le réécrit après chaque mutation ; aucun code ne le lit pour décider ni pour sauvegarder.
- Nom exact `leaderstats`, en minuscules, sinon le joueur n'apparaît pas au classement. Enfants : `IntValue`, `NumberValue` ou `StringValue`. Ordre des colonnes : ordre d'ajout ; un `BoolValue` enfant nommé `IsPrimary` (vrai) met la stat en premier ; un `NumberValue` enfant nommé `Priority` trie le reste (plus haut = plus tôt, 0 par défaut) ; `IsPrimary` passe avant `Priority` (doc, 2026-10-18).
- Crée-le **après** le profil (là où `PlayerProfiles` pose `DataLoaded`, section 6) : créé avant, il affiche 0, et un code qui le relirait sauvegarderait 0. Masquer la liste : `StarterGui:SetCoreGuiEnabled(Enum.CoreGuiType.PlayerList, false)` côté client. Au-delà de 2^53 (simulator), la valeur exacte reste dans le profil (luau.md) ; affiche une `StringValue` abrégée (« 1,2 M »).

```luau
--!strict
-- ServerScriptService/Leaderstats.luau (ModuleScript) : miroir du profil, jamais source de vérité.
local COLUMNS = { "Coins", "Rebirths" } -- ordre d'affichage
local Leaderstats = {}

-- Appelé une fois, juste après le chargement du profil.
function Leaderstats.create(player: Player, data: { [string]: number })
	local folder = Instance.new("Folder")
	folder.Name = "leaderstats"
	for _, name in COLUMNS do
		local stat = Instance.new("IntValue")
		stat.Name = name
		stat.Value = data[name] or 0
		stat.Parent = folder
	end
	folder.Parent = player -- en dernier : le client reçoit un dossier complet
end

-- Appelé par le module qui fait autorité, après chaque mutation du profil.
function Leaderstats.set(player: Player, name: string, value: number)
	local folder = player:FindFirstChild("leaderstats")
	local stat = if folder then folder:FindFirstChild(name) else nil
	if stat and stat:IsA("IntValue") then
		stat.Value = value
	end
end

return Leaderstats
```

## 11. MemoryStoreService

Mémoire partagée entre serveurs, rapide, **non durable** : chaque entrée expire. Jamais de progression permanente dedans.

Quotas au niveau de l'expérience, relevés en octobre 2026 (anciennes valeurs, fausses aujourd'hui : 64 Ko + 1 Ko × utilisateurs, 1 000 + 100 × utilisateurs) :

| Élément | Limite |
|---|---|
| Mémoire | 64 Ko + 1,2 Ko × utilisateurs (un joueur parti compte encore environ 8 jours) |
| Requêtes | 1 000 + 120 × CCU unités par minute |
| Expiration | 0 à 3 888 000 s (45 jours) ; hors plage : `InvalidExpirationTime` |
| Clé de hash map ou de sorted map, clé de tri | 128 caractères |
| Valeur | 32 Ko (mesuré : 33 Ko lève `ItemValueSizeTooLarge`) |
| Sorted map ou queue | 1 000 000 éléments et 100 Mo |

- **Coûts** : la plupart des appels coûtent 1 unité. `SortedMap:GetRangeAsync` et `Queue:ReadAsync` coûtent 1 unité par élément renvoyé (1 si vide), plus 1 unité toutes les 2 s d'attente d'une lecture de queue. `HashMap:UpdateAsync` coûte au moins 2 unités, `HashMap:ListItemsAsync` les partitions parcourues plus les éléments. Préfère `GetAsync` sur une clé connue.
- Quota mémoire atteint : les écritures échouent (aucune éviction), lectures et suppressions passent. Expiration explicite et la plus courte possible ; supprime ce qui est traité.
- Une sorted map ou une queue vit sur une seule partition : découpe-la quand elle chauffe. Une hash map se répartit seule, mais une clé lue par tous sature : éclate-la (`meta/players`, `meta/mode`). Plafonds estimés par la doc, susceptibles de changer (doc, 2026-10-18) : environ 30 000 unités/min par partition, 100 000 par structure, et par clé de hash map environ 5 000 unités/min en écriture et 15 000 en lecture. Statuts à surveiller : `TotalRequestsOverLimit`, `PartitionRequestsOverLimit`, `DataStructureRequestsOverLimit`, `DataUpdateConflict` ; backoff avec aléa, car des serveurs qui réessaient ensemble aggravent la pénurie.

- **HashMap**, choix par défaut pour un accès par clé connue (registre `server/<JobId>`, cache, lobbies) : `SetAsync(key, value, expiration)`, `UpdateAsync(key, transform, expiration)`.
- **SortedMap**, pour un ordre : `SetAsync(key, value, expiration, sortKey)`, `GetRangeAsync(direction, count, exclusiveLowerBound?, exclusiveUpperBound?)` (200 éléments au plus), pagination par une borne exclusive tirée du dernier élément (`{ key = …, sortKey = … }`). Clés de tri nombres ou chaînes, mélange permis (en ordre croissant : nombres, puis chaînes, puis éléments sans clé de tri) ; vue vivante, pas un instantané. Registre de serveurs : chaque serveur réécrit `SetAsync(game.JobId, info, 60, nombreDeJoueurs)` toutes les 15 s, et l'expiration retire seule un serveur planté.
- **Queue**, pour passer du travail entre serveurs : `GetQueue(name, invisibilityTimeout)` (30 s par défaut), `AddAsync(value, expiration, priority?)`, `ReadAsync(count, allOrNothing?, waitTimeout?)` (100 éléments au plus) qui renvoie `(items, readId)` et cache les éléments lus, `RemoveAsync(readId)`. Le délai d'invisibilité se règle dans `GetQueue` : le troisième argument de `ReadAsync` est l'attente, et sans lui elle est infinie (-1 par défaut). Passe une attente bornée.
- Consommer : lis un lot dans un `pcall`, traite chaque élément avec un handler idempotent (clé d'idempotence dans la charge utile), et appelle `RemoveAsync(readId)` seulement si **tout** le lot a réussi. Sinon il réapparaît après le délai d'invisibilité, pour toi ou un autre serveur. Lire, traiter et supprimer doit tenir dans ce délai. Échec définitif : file « dead letter », sans réajouter de doublon.
- `MemoryStoreService:GetDistributedCounter(name)` (`GetAsync`, `IncrementAsync(delta, expiration)`) : compteur partagé entre serveurs, sans lecture-modification-écriture, présent dans les définitions de septembre 2026 mais absent de la doc officielle au 2026-10-18, quotas non documentés (à vérifier) ; sorted map en repli.

## 12. MessagingService

Diffusion entre serveurs vivants, sans garantie. Le message est un **indice** : l'état qui fait autorité vit en DataStore ou en MemoryStore.

Limites relevées en octobre 2026 (doc, 2026-10-18) : message 1 Ko, topic 1 à 80 caractères ; par minute, envois 600 + 240 × joueurs par serveur, réceptions 40 + 80 × serveurs par topic et 400 + 200 × serveurs pour toute l'expérience, 240 requêtes d'abonnement par serveur ; abonnements simultanés 20 + 8 × joueurs par serveur. Anciennes valeurs, périmées : 150 + 60 × joueurs pour les envois, 5 + 2 × joueurs pour les abonnements. Open Cloud partage ces quotas (date annoncée : 2026-05-11). Calcule la réception au nombre maximal de serveurs : à 100 serveurs, l'expérience entière reçoit au plus 20 400 messages par minute.

- `SubscribeAsync(topic, callback)` renvoie une connexion ; le callback reçoit `message.Data` et `message.Sent` (heure Unix d'envoi, en secondes). `PublishAsync(topic, data)`. Les deux dans un `pcall`, avec backoff borné ; enveloppe aussi le corps du callback.
- Ni livraison ni ordre garantis, doublons possibles : un identifiant dans chaque message, une table des identifiants déjà vus, et relecture de l'état à réception **et** sur un tick lent, pour converger si un message se perd.
- Envoie des identifiants, pas des blobs. Un seul module abonné par topic et par serveur, jamais un abonnement par joueur ; abonne-toi avant de publier.
- Requête-réponse : `reqId` + accusé + délai. Pas d'accusé ne veut pas dire « pas exécuté » : l'opération doit être idempotente par `reqId`. À grande échelle, préfère le pull (état en MemoryStore lu à la demande) au push.

## 13. Échanges, transferts et récompenses de temps

- L'atomicité vaut **par clé**. Aucune primitive ne débite un profil et crédite un autre en une seule écriture : un crash entre les deux duplique ou perd.
- Transfert durable : écris d'abord `{ id, from, to, amount, state = "pending" }`, puis débite (sauté si déjà fait pour cet `id`), crédite (idem), marque `done`. Un balayage de reprise relance à la première étape incomplète. Chaque étape est idempotente par `id` : `applyOnce` (section 5) sur une clé libre ; sur un profil tenu par une session, mutation de ses données puis sauvegarde immédiate (un `UpdateAsync` direct serait écrasé au prochain autosave). Même sur un seul serveur avec deux sessions verrouillées, sauvegarde les deux profils tout de suite : un crash entre les deux écritures reste possible.
- Sérialise les opérations économiques par joueur : deux remotes dans la même frame passent tous les deux le contrôle de solde si un yield les sépare du débit. Verrou par joueur libéré sur tous les chemins, ou revalidation après chaque yield (`securite.md`).
- Récompenses de temps et hors ligne : horodatage `os.time()` dans le profil, écart calculé par le serveur, gain plafonné, gain et nouvel horodatage dans la même sauvegarde. Jamais `tick()` ni `time()` pour ce qui se persiste. Un bonus quotidien sans date sauvegardée se farme en se reconnectant.

### Codes promo

- Aucune API Roblox : système maison. **La liste reste sur le serveur** (ModuleScript sous `ServerScriptService`, ou config en direct pour en ajouter sans republier, design-liveops.md, section 5). Un code dans `ReplicatedStorage` se lit avant sa sortie.
- **Utilisé une fois par compte** : le code et sa date vont dans le profil (`RedeemedCodes`, ajouté au gabarit du schéma, section 8), dans la même mutation que la récompense, sans yield entre le contrôle et l'octroi. Profil pas chargé = « réessaie », jamais « déjà utilisé ».
- Saisie normalisée (espaces retirés, majuscules), longueur bornée, débit limité par joueur contre la force brute, expiration en `os.time()` (UTC) lue par le serveur. Récompense en monnaie ou cosmétique du jeu, jamais d'objet payant aléatoire ni de Robux ; pas de lien hors plateforme en jeu pour trouver les codes (design-liveops.md, section 7). Le client envoie le texte par un `RemoteFunction` et ne reçoit que le résultat ; après un succès, mets à jour `leaderstats` (section 10) et journalise la source (`LogEconomyEvent`, monetisation.md).

```luau
--!strict
-- ServerScriptService/Codes.luau (ModuleScript) : codes vérifiés et consommés par le serveur.
local Players = game:GetService("Players")
local PlayerProfiles = require(script.Parent.PlayerProfiles) -- section 6

local CODES: { [string]: { coins: number, expiresAt: number? } } = { -- expiresAt : os.time() UTC
	LAUNCH = { coins = 500 },
	HALLOWEEN26 = { coins = 2000, expiresAt = 1793491200 }, -- 2026-11-01 00:00 UTC
}
export type Result = "ok" | "invalid" | "expired" | "used" | "retry"
local lastTry: { [Player]: number } = {}
local Codes = {}

function Codes.redeem(player: Player, raw: unknown): Result
	if os.clock() - (lastTry[player] or -math.huge) < 2 then
		return "retry" -- une tentative toutes les 2 s : pas de force brute
	end
	lastTry[player] = os.clock()
	if typeof(raw) ~= "string" or #raw > 32 then
		return "invalid"
	end
	local code = string.upper((string.gsub(raw, "%s", "")))
	local def = CODES[code]
	if def == nil or (def.expiresAt ~= nil and os.time() >= def.expiresAt) then
		return if def then "expired" else "invalid"
	end
	local profile = PlayerProfiles.get(player.UserId)
	if profile == nil then
		return "retry" -- profil pas chargé : jamais « déjà utilisé »
	end
	local redeemed = profile.Data.RedeemedCodes -- { [string]: number }
	if redeemed[code] ~= nil then
		return "used"
	end
	redeemed[code] = os.time() -- même mutation que le gain, sans yield : pas de double octroi
	profile.Data.Coins += def.coins
	return "ok"
end

Players.PlayerRemoving:Connect(function(player)
	lastTry[player] = nil
end)

return Codes
```

## 14. Studio, accès API et tests

- Studio n'a pas accès aux DataStores par défaut (erreur 403). Active « Enable Studio Access to API Services » dans File → Experience Settings → Security, puis Save.
- Avec cet accès, Studio écrit sur **le même backend que le jeu live**. Prends une expérience de test séparée, ou au minimum un store suffixé en Studio (`PlayerData_Studio`), et le `.Mock` du store ProfileStore (`PlayerStore.Mock`) pour des écritures éphémères.
- En playtest, `execute_luau` dans le datamodel `Server` du MCP touche aussi les vrais stores : noms de store jetables, clés préfixées `test-` (`studio-mcp.md`). Avant un test destructif (wipe, rejeu de migration, écriture de masse), liste les destinations exactes : stores, MemoryStores, endpoints, webhooks. Une étiquette « test » n'isole rien. Risque élevé : propose, puis attends un oui.
- `PlayerRemoving` ne se déclencherait pas toujours au Stop de Studio (à vérifier) : teste aussi le chemin `BindToClose`. Studio ne reproduit ni le multi-serveur ni le vrai throttling : teste le verrou et les quotas sur une place publiée de test.

| Niveau | Cas à couvrir |
|---|---|
| Logique pure (Lune, Jest-Lua) | chaque migration contre une fixture de chaque version publiée ; la version actuelle ressort inchangée ; une version future lève ; `defaults()` ne partage aucune sous-table ; aller-retour `JSONEncode` |
| Moteur, store jetable | nouveau joueur ; joueur existant ; quitter puis revenir ; arrêt forcé (le flush passe) ; lecture qui échoue (store factice qui lève) → expulsion et **aucune** écriture |
| Place publiée | rejoindre vite un autre serveur (verrou) ; téléportation ; achat puis arrêt avant l'autosave → un seul octroi au retour |

- Un faux store en mémoire ne reproduit ni les erreurs, ni les yields, ni la copie due à la sérialisation, ni l'annulation d'`UpdateAsync` : fais-lui lever des erreurs exprès. Throttling, multi-serveur et vrais achats restent « Non vérifié » tant qu'ils n'ont pas tourné sur une place publiée.
- Creator Hub → Monitoring → Data Stores : stockage, requêtes par API et par statut, consommation de quota par type (lecture, écriture, listage, suppression), sur les 30 derniers jours. Un pic soudain de requêtes trahit presque toujours une sauvegarde en boucle.

## 15. Diagnostic rapide

Transforme d'abord la conclusion (« mes données se réinitialisent ») en observation (« le compteur affichait 0 après rejoin »).

| Symptôme | Cause probable |
|---|---|
| Progression remise à zéro | une lecture a échoué, le code a mis les défauts puis les a sauvés |
| Objets dupliqués après un changement de serveur | pas de verrou de session |
| Retour en arrière | serveur périmé qui écrit en dernier, ou écriture perdue |
| Perte à l'arrêt du serveur | `BindToClose` absent, en série ou sans attente |
| « Ma modif n'apparaît pas » | cache de 4 s, ou deux instances de store |
| Rien ne se sauvegarde en Studio | accès API coupé (403), script côté client, `pcall` qui avale l'erreur |
| Erreurs 301 à 306 en rafale | sauvegarde en boucle ou à chaque changement |
| Le HUD montre l'ancien solde quelques secondes | le client affiche une valeur en cache : attends `DataLoaded` et la première valeur du serveur |

## 16. Pièges

- Garde `if player:GetAttribute("DataLoaded") == false then return end` : `nil` (chargement en cours) passe. Écris `~= true`.
- Renommer le store (`_v2`) pour « migrer » : tout le monde repart de zéro.
- `UpdateAsync(key, function() return cache end)` : un `SetAsync` déguisé. Et `keyInfo:GetUserIds()` sans vérifier `keyInfo` fait planter la première sauvegarde d'un nouveau joueur.
- `leaderstats` comme source de vérité : c'est un miroir d'affichage.
- `PlayerAdded` seul : les joueurs présents avant le script ne sont jamais chargés. Sans contrôle `player.Parent` après le chargement, la session d'un joueur parti n'est jamais libérée.
- `BindToClose` qui n'attend que `Players:GetPlayers()` : les sauvegardes lancées par `PlayerRemoving` sont coupées.
- Persister un hash ou un id qui change d'un build à l'autre (hash Flamework, index d'un tableau de config).
- `ReadAsync(count, false, invisibilityTimeout)` : le troisième argument est l'attente.

## 17. Checklist

- [ ] Une lecture ratée ne peut jamais écrire les défauts par-dessus une vraie sauvegarde.
- [ ] Un seul serveur possède un profil : ProfileStore, ou verrou dans la même clé vérifié à chaque écriture.
- [ ] Toutes les écritures du profil passent par un seul module (ProfileStore ou `UpdateAsync`).
- [ ] Chaque appel est dans un `pcall`, avec réessais bornés, `warn` des échecs, et rafales conditionnées à `GetRequestBudgetForRequestType`.
- [ ] Autosave étalé, `PlayerRemoving`, `BindToClose` en parallèle sous 30 s ; joueurs déjà présents chargés, départ pendant le chargement géré.
- [ ] Schéma versionné, migrations testées sur fixtures, données du futur refusées, nom de store inchangé.
- [ ] `defaults()` neuf à chaque appel ; données sérialisables (ni NaN, ni Instance, ni table mixte).
- [ ] Clé `player_<UserId>`, `userIds` à chaque écriture, modèle RTBF configuré.
- [ ] Écriture incertaine traitée comme inconnue ; vérification avec `UseCache = false`.
- [ ] Studio sur un store séparé ou une expérience de test ; testé : nouveau joueur, retour, arrêt forcé, lecture en échec, changement rapide de serveur.
