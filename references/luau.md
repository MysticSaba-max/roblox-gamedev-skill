# Luau : langage, types et bibliothèque standard

À lire quand : tu écris ou relis du Luau, tu poses des types `--!strict`, tu hésites sur une fonction de `task`, `table`, `string`, `math` ou `buffer`, ou tu te demandes si une nouveauté de Luau marche vraiment dans Roblox. La vitesse du code (natif, allocations, profilage) est dans performance.md.

## Sommaire

1. Luau n'est pas Lua 5.x
2. Valeurs, vérité, nombres
3. Tables
4. Fonctions, portée, modules
5. Mode strict et directives
6. Écrire des types
7. Le nouveau solveur de types
8. Classes et métatables
9. Erreurs : pcall, xpcall, error
10. task et coroutines
11. Signaux et événements différés
12. Mesurer le temps
13. Chaînes et motifs
14. vector, buffer, bit32, EncodingService
15. Nouveautés : utilisables ou non
16. Style
17. roblox-ts en une table
18. Pièges
19. Checklist

Base : Luau 0.743 (2026-10-16) et Studio 0.743. Luau sort presque chaque semaine : revérifie un point récent avant de t'y fier.

## 1. Luau n'est pas Lua 5.x

Luau part de Lua 5.1, en retire ce qui est dangereux et ajoute ses extensions. Ne raisonne jamais en Lua 5.3 ou 5.4 : le code ne compile pas, ou se comporte autrement.

Ajouts de Luau :
- `continue` (dernière instruction de son bloc) ; affectations composées `+=`, `-=`, `*=`, `/=`, `//=`, `%=`, `^=`, `..=`, qui sont des instructions, pas des expressions.
- Expression `if a then b elseif c then d else e`, interpolation `` `Score : {score}` ``, itération généralisée `for k, v in t do`, division entière `//`, littéraux `0b1010` et `1_000_000`, échappement `\u{E002}`.
- Typage graduel ; `table.create`, `find`, `clear`, `clone`, `freeze` ; bibliothèques `buffer`, `vector`, `bit32`, `utf8`, et `task` (Roblox).

| Lua 5.x ou autre langage | En Luau |
|---|---|
| `goto`, étiquettes | `continue`, retour anticipé |
| `&`, `\|`, `~`, `<<`, `>>` | `bit32.band`, `bor`, `bxor`, `lshift`, `rshift` |
| `io`, `package`, `dofile`, `loadfile`, `string.dump` | rien : ni fichiers ni modules natifs |
| `os.execute`, `os.getenv`, `os.exit` | `os` garde `clock`, `date`, `difftime`, `time` |
| `debug.getinfo`, `debug.sethook` | `debug.traceback`, `debug.info`, `profilebegin`/`profileend`, `setmemorycategory`, `dumpcodesize` (barre de commande de Studio seulement) |
| `collectgarbage("collect")` | seul `collectgarbage("count")` répond, et la doc le dit déprécié au profit de `gcinfo()` : on ne force pas le GC |
| `__gc`, `__close`, `__pairs`, tables éphémères | nettoyage explicite (`Destroy`, `Disconnect`, Trove) |
| entiers 64 bits séparés | un seul `number`, un double |

Présents, mais à fuir :
- `getfenv` et `setfenv` désoptimisent tout le script. Avec `loadstring` et `require(<nombre>)`, ils signalent une porte dérobée dans un modèle inséré.
- `loadstring` est refusé côté serveur sauf si `ServerScriptService.LoadStringEnabled` est coché. Jamais pour du texte venu d'un joueur.
- `unpack` global : écris `table.unpack`. Les bibliothèques, la métatable des chaînes et les globales intégrées sont en lecture seule : pas de monkey-patch de `string` ou `table`.
- Globales Roblox : `game`, `workspace`, `script`, `Enum`, `shared`, et `plugin` dans un plugin. N'appuie pas une architecture sur `_G` ou `shared`.

Venus de JavaScript ou de Python, ces formes n'existent pas : `===`, `null`, `undefined`, `?.`, `??`, spread, `let`, fonctions fléchées, `try/catch` (→ `pcall`), `async/await`, `.map`, `.push`, `.length`, `None`, `elif`, compréhensions. Traduis le modèle de données, pas jeton par jeton.

## 2. Valeurs, vérité, nombres

- Seuls `false` et `nil` sont faux. `0`, `""` et `{}` sont vrais : écris `if count > 0`, `if text ~= ""`.
- `and`/`or` renvoient un opérande, pas un booléen. `cond and a or b` donne `b` dès que `a` vaut `false` ou `nil` : écris `if cond then a else b`.
- `value or default` écrase un `false` valide : `part:GetAttribute("Enabled") or true` vaut toujours `true`. Écris `if value == nil then default else value`.
- `==` ne convertit jamais : `0 == "0"` et `1 == true` sont faux. L'arithmétique, elle, convertit (`"10" + 1 == 11`) : vérifie le type avant tout calcul sur une entrée externe. `"100" < "20"` est vrai (ordre lexicographique) : compare des nombres, pas des chaînes.
- `type(v)` donne la catégorie Luau (`"userdata"` pour une Part). `typeof(v)` connaît les types Roblox (`"Instance"`, `"Vector3"`, `"CFrame"`) : c'est lui qui valide.

Nombres :
- Un seul type `number`, un double : entiers exacts jusqu'à 2^53 (environ 9e15). Au-delà, un compteur n'augmente plus. Pour des montants géants (simulateurs, idle), stocke une mantisse et un exposant, ou une unité mise à l'échelle.
- NaN est le seul nombre différent de lui-même. Il a le type `"number"` et rate toutes les comparaisons : `if v < 0 or v > MAX then return end` le laisse passer, puis il corrompt une position ou une sauvegarde. L'infini passe aussi les tests de type.

```luau
--!strict
-- Vrai seulement pour un nombre fini : rejette NaN, +inf et -inf.
local function isFiniteNumber(value: unknown): boolean
	return typeof(value) == "number" and value == value and math.abs(value) ~= math.huge
end
```

- Entier : `n % 1 == 0`, qui rejette aussi NaN et l'infini (`inf % 1` vaut NaN). `math.isnan`, `math.isinf` et `math.isfinite` existent aussi. La validation complète d'un remote est dans securite.md.
- `//` arrondit vers moins l'infini (`-7 // 2 == -4`). `%` prend le signe du diviseur (`-1 % 3 == 2`) ; `math.fmod` tronque vers zéro.
- `math.round` arrondit la moitié loin de zéro. `math.clamp(x, min, max)` lève une erreur si `min > max`. `math.random` tronque ses arguments : `math.random(1.5)` renvoie toujours 1.
- À connaître aussi : `math.lerp(a, b, t)`, `math.map(x, inMin, inMax, outMin, outMax)`, `math.sign`, `math.atan2(y, x)` (l'angle d'un déplacement, pas `math.atan(y / x)`), `math.noise` (Perlin, entre -1 et 1 environ, pas un hash). Angle entre deux directions : `math.acos(math.clamp(dot, -1, 1))`, car sans le `clamp` un arrondi à 1,0000001 donne NaN.
- `Random.new(seed)` donne un flux indépendant et rejouable (`rng:NextInteger(1, 6)`, `rng:NextNumber()`). Évite `math.randomseed` : il change le flux de tous les modules.

## 3. Tables

- Choisis une forme et garde-la : séquence `{ T }` (clés 1 à n), dictionnaire `{ [K]: V }`, enregistrement (champs nommés), ensemble `{ [K]: true }`. Une table mixte a un `#` indéfini, échoue à l'encodage DataStore et se déforme en passant par un remote. Clé dynamique : `t[key]` ; `t.key` veut dire la chaîne `"key"`.
- `#t` n'est fiable que sur une séquence sans trou. `t[i] = nil` au milieu crée un trou : retire avec `table.remove(t, i)`. Affecter `nil` supprime la clé : on ne stocke pas « nil » comme valeur.
- Itération généralisée par défaut, `for k, v in t do` ; `pairs` et `ipairs` restent valides. L'ordre d'un dictionnaire n'est pas garanti : trie les clés avant un affichage. Pour un ordre sûr sur une séquence : `for i = 1, #t` ou `ipairs` (qui s'arrête au premier `nil`).
- Ne retire rien pendant un parcours vers l'avant : des éléments seraient sautés. Parcours à l'envers, `for i = #list, 1, -1 do`. Retrait en O(1) quand l'ordre ne compte pas : `list[i] = list[#list]`, puis `table.remove(list)`.
- Une table est une référence : une fonction qui mute son argument mute la table de l'appelant. `table.clone` est superficiel (les sous-tables restent partagées), `table.freeze` aussi (il ne gèle que le premier niveau).

**Données par défaut d'un joueur.** `table.clone(DEFAULTS)` ou `data or DEFAULTS` avec des sous-tables fait partager le même `inventory` à tous les joueurs : un objet donné à l'un apparaît chez tous (reproduit dans Studio 0.742). Fabrique une table neuve à chaque fois :

```luau
--!strict
export type PlayerData = { coins: number, inventory: { string }, settings: { music: boolean } }

-- Chaque appel crée des tables neuves : rien n'est partagé entre joueurs.
local function defaultData(): PlayerData
	return { coins = 0, inventory = {}, settings = { music = true } }
end
```

- Mise à jour immuable (état d'UI, comparaison par `==`) : clone seulement le chemin modifié, `local nextState = table.clone(state)` puis `nextState.audio = table.clone(state.audio)`. Les branches intactes gardent leur identité.
- Pas de copie profonde générique : elle devrait trancher pour les métatables, les cycles et les Instances. Copie seulement la forme de données que tu possèdes.
- `setmetatable(data, { __index = DEFAULTS })` sert les lectures, mais `data.settings.music = false` écrit alors dans `DEFAULTS.settings`, partagé par tous.
- Table à clés faibles indexée par Instance (`__mode = "k"`) : la référence Luau d'une Instance peut être ramassée alors que l'Instance existe encore, et l'entrée disparaît. Bug réel : des valeurs d'origine perdues, des objets restés invisibles. Range cet état en attributs sur l'Instance, ou dans une table normale vidée sur `Destroying`.

| Fonction | À savoir |
|---|---|
| `table.create(n, v?)` | pré-alloue n cases, remplies de `v` |
| `table.find(t, v, init?)` | index de la valeur ou `nil` ; recherche linéaire, sans prédicat |
| `table.remove(t, i?)` | renvoie l'élément retiré, le dernier sans `i` |
| `table.clear(t)` | vide en gardant la référence et la capacité |
| `table.freeze(t)`, `table.isfrozen(t)` | lecture seule superficielle ; écrire lève « attempt to modify a readonly table » |
| `table.move(src, a, b, i, dst?)` | copie une plage |
| `table.pack(...)`, `table.unpack(t, i, j)` | `pack` remplit `n`, qui compte aussi les `nil` |
| `table.concat(t, sep?)` | joint des chaînes en une passe |
| `table.sort(t, cmp?)` | trie en place ; `cmp` est un ordre strict (`a < b`), sinon « invalid order function for sorting » |

## 4. Fonctions, portée, modules

- Toujours `local`. Une fonction sans `local` est une globale : elle échappe au typage, et `--!strict` la signale.
- Une `local function` n'existe qu'à partir de sa ligne. Une closure créée plus haut appelle `nil`, et dans un handler `ChildAdded` l'erreur passe inaperçue. Pré-déclare :

```luau
--!strict
local openMenu: (menuId: string) -> ()

local function onShopButton()
	openMenu("Shop")
end

-- Affecte sans `local` : `local function openMenu` créerait un second local qui masque le premier.
openMenu = function(menuId: string)
	print(`ouverture de {menuId}`)
end
```

- `:` passe `self` : `obj:move(2)` vaut `obj.move(obj, 2)`. Définis et appelle de façon cohérente. `pool:acquire(pool)` passe `self` deux fois : ce bug a vidé un pool d'objets (10 disponibles tombés à 4 en six cycles).
- Arguments manquants = `nil`, arguments en trop = ignorés. `select("#", ...)` compte aussi les `nil` ; `table.pack(...)` puis `table.unpack(args, 1, args.n)` les relaient sans perte.
- `return` (zéro valeur) n'est pas `return nil` (une valeur). Une fonction typée `T?` écrit `return nil` dans sa branche vide ; une API `(value, err)` renvoie deux valeurs sur chaque branche. Des parenthèses gardent seulement la première valeur : `local cleaned = (string.gsub(text, "%s+", " "))`.
- La boucle `for` crée une variable neuve à chaque tour : une closure capture la bonne valeur. Une variable modifiée hors de la boucle est partagée par toutes les closures.
- **200 variables locales au plus par fonction** (« Out of local registers when trying to allocate v200: exceeded limit 200 »). La note de version 0.743 assouplit la règle, mais ce n'est pas actif dans Studio 0.743 : 210 locales à la suite échouent toujours. Range les valeurs dans une table, enferme des groupes dans des blocs `do … end`, ou découpe le module.

Modules :
- `require` exécute un ModuleScript une fois par contexte (le serveur, chaque client) et met sa valeur de retour en cache. Le code est partagé ; l'état ne l'est jamais entre client et serveur.
- Le code de niveau module tourne pendant le `require` : garde-le léger, sans yield caché. Les connexions démarrent dans une fonction d'initialisation (voir architecture.md).
- `require` littéraux, en haut du fichier. Un chemin calculé ou un chargeur en boucle (`for _, m in folder:GetChildren() do mods[m.Name] = require(m) end`) ne lève rien mais rend tout `any` ; en strict, luau-lsp signale « Unknown require ». Écris un registre de requires explicites.
- Cycle de `require` : « Requested module was required recursively ». Extrais la partie commune dans un troisième module, ou injecte la dépendance à l'initialisation. Renvoie une table nommée (`local Inventory = {}` … `return Inventory`), jamais une fonction anonyme : recherche et autocomplétion s'y retrouvent.
- Dans Roblox, `require` prend une Instance : `require(ReplicatedStorage.Shared.Greeting)`, ou `require(ServerStorage:WaitForChild("Config") :: ModuleScript)` ; un module au type perdu se caste, `require(module) :: { start: () -> () }`. Roblox accepte aussi les chemins texte : `require("./Greeting")` part de `script.Parent`, `@self/` de `script`, `@game/` du DataModel (doc, 2026-10-18). Suis la convention du projet.

## 5. Mode strict et directives

- Directives de mode, en première ligne : `--!strict` vérifie tout ; `--!nonstrict` (le défaut) vérifie seulement ce qui est annoté, le reste devient `any` ; `--!nocheck` coupe l'analyse. Autres directives : `--!native`, `--!optimize 0|1|2`, `--!nolint` ou `--!nolint NomDuLint`. Les types n'existent qu'à l'analyse : ils ne coûtent rien à l'exécution et ne valident aucune donnée.
- **Rien d'autre sur la ligne.** `--!strict -- serveur` est ignoré (« Comment directive with the type checking mode has extra symbols at the end of the line ») : le fichier repasse en nonstrict et plus aucune erreur de type n'est signalée. Mets le commentaire à la ligne suivante, et une seconde directive (`--!native`) sur sa propre ligne, juste après.
- `--!strict` sur chaque fichier neuf, sauf convention contraire du projet. Ne durcis pas un fichier existant sans demande, et n'affaiblis jamais un fichier déjà strict.
- Migration d'un vieux projet : strict sur les nouveaux fichiers ; puis les modules partagés (leurs types coulent vers les autres) ; puis l'état des services. Corrige les vrais bugs révélés avant de faire taire un avertissement. Ne change jamais comportement et types dans le même patch.
- Le mode nonstrict ne signale que les erreurs certaines et ignore les globales inconnues : un fragment incomplet y paraît propre. Analyse en strict pour voir ce qui manque. Exception : les specs TestEZ restent en `--!nonstrict`, car `describe`, `it` et `expect` sont des globales injectées, invisibles au vérificateur ; le module testé, lui, reste strict.
- Défaut du projet : `.luaurc` à la racine (`{ "languageMode": "strict" }`) pour luau-lsp et luau-analyze. Dans Studio : `Workspace.LuauTypeCheckMode`, réglable dans la fenêtre Propriétés (capacité `PluginOrOpenCloud` : un script de jeu ne peut ni le lire ni l'écrire). `luau-analyze` ne connaît aucune globale Roblox : pour du code Roblox, lance `luau-lsp analyze` avec les définitions Roblox et un sourcemap (voir outils.md).
- Une locale volontairement inutilisée prend le préfixe `_` (fait taire `LocalUnused`). Dans une revue, cite le nom du lint : `UnknownGlobal`, `DeprecatedApi`, `MisleadingAndOr`, `ComparisonPrecedence` (`not x == y`), `LocalShadow`.

## 6. Écrire des types

```luau
--!strict
type ItemId = string
type Rarity = "Common" | "Rare" | "Epic"                    -- union de littéraux
type Stack = { id: ItemId, count: number, note: string? } -- `T?` veut dire `T | nil`
type Inventory = { Stack }                                 -- tableau
type Prices = { [ItemId]: number }                         -- dictionnaire
type OnBuy = (player: Player, id: ItemId) -> (boolean, string?)
type Box<T = string> = { value: T }                        -- défaut permis sur un alias, pas sur une fonction
type Node = { value: number, next: Node? }                 -- type récursif : nommé d'abord
```

- Annote les frontières : paramètres, retours, exports, `self`. Laisse l'inférence faire les locales évidentes.
- `unknown` pour toute donnée non fiable (arguments de remote, JSON, DataStore, attributs) : il force un affinage avant usage. `any` coupe toute vérification en aval : `local n: number = get()` passe si `get` renvoie `any`. `never` marque l'impossible : s'il apparaît après un affinage, deux tests se contredisent, c'est un vrai bug.
- Cast `::` : seulement pour préciser une inférence trop large, après une preuve (`typeof`, `IsA`). Il ne vérifie rien à l'exécution. Il doit aller vers un sous-type, un super-type ou `any` ; `(x :: any) :: T` signale un type qui ment quelque part. Caster un appel à plusieurs retours ne garde que le premier. Bon usage : typer une table vide, `local names = {} :: { string }`.
- `export type` dans un module, puis `Types.Stack` chez les consommateurs. Un seul module `Shared/Types` pour tout ce qui traverse le réseau (payloads, profil) : client et serveur sont d'accord par construction.
- Unions de littéraux pour les vocabulaires fermés (phases, actions, codes de refus) ; évite les fabriques `makeEnum`, qui effacent les littéraux. Intersections pour combiner des enregistrements (`Named & Timed`) ou décrire des surcharges, jamais `string & number` (c'est `never`).
- Génériques quand la sortie dépend de l'entrée (`local function first<T>(list: { T }): T?`), jamais remplacés par `any`. Les packs génériques (`T...`) relaient des retours multiples et se placent après les types simples : `<R, A...>`. Exemple : `withRetry<T...>(fn: () -> T...): (boolean, T...)`, qui fait `local results = table.pack(pcall(fn))` puis `table.unpack(results, 2, results.n)`.

Union étiquetée, avec contrôle d'exhaustivité :

```luau
--!strict
type LoadResult = { kind: "ready", value: number } | { kind: "failed", message: string }

local function describe(result: LoadResult): string
	if result.kind == "ready" then
		return `prêt : {result.value}`
	elseif result.kind == "failed" then
		return `échec : {result.message}`
	else
		-- Une variante ajoutée sans branche devient une erreur de type ici.
		-- Teste le champ, pas la table : l'ancien solveur n'affine pas `result` jusqu'à `never`.
		local impossible: never = result.kind
		return impossible
	end
end
```

Affiner :
- Formes reconnues : vérité (`if x then`), `typeof(x) == "number"`, `type(x) == "table"`, égalité avec un littéral, `assert(x, "msg")`, et `inst:IsA("BasePart")`, qui vérifie aussi à l'exécution : préfère-le à un cast.
- Un affinage porte sur un emplacement. Réaffecter la locale l'annule. Sur un champ (`self.target ~= nil`), il devient faux si du code intermédiaire écrit le champ (yield, appel) : copie le champ dans une locale, puis affine la locale. Une erreur de type s'affiche là où les types se rencontrent, pas là où le bug naît : remonte la valeur.
- `FindFirstChild`, `player.Character` et `WaitForChild(name, timeout)` renvoient un optionnel. `WaitForChild(name)` sans timeout est typé non optionnel. Dans un handler, garde avec retour anticipé ; si l'absence est un bug de programmation, `assert` ; un cast seulement avec une certitude extérieure.
- `player.PlayerGui` n'est pas une propriété mais un enfant créé par le moteur : en strict, « Key 'PlayerGui' not found in external type 'Player' ». Écris `player:WaitForChild("PlayerGui") :: PlayerGui`.
- `GetAttribute` est typé `unknown?` dans les définitions luau-lsp récentes (`any` dans d'anciennes) : affine avant usage, `local rawSpeed = part:GetAttribute("Speed")` puis `local speed = if typeof(rawSpeed) == "number" then rawSpeed else 16`. Un cast `:: number?` ne vérifie rien : une chaîne passerait.

Payload de remote : tout ce qui vient du client est `unknown`.

```luau
--!strict
local ReplicatedStorage = game:GetService("ReplicatedStorage")

local remotes = ReplicatedStorage:WaitForChild("Remotes")
local buyItem = remotes:WaitForChild("BuyItem") :: RemoteEvent

buyItem.OnServerEvent:Connect(function(player: Player, payload: unknown)
	-- La limite de débit passe avant tout le reste (voir securite.md).
	if typeof(payload) ~= "table" then
		return
	end
	local fields = payload :: { [string]: unknown }
	local itemId, quantity = fields.itemId, fields.quantity
	if typeof(itemId) ~= "string" or #itemId == 0 or #itemId > 32 then
		return
	end
	if typeof(quantity) ~= "number" or quantity % 1 ~= 0 or quantity < 1 or quantity > 99 then
		return
	end
	print(`{player.Name} demande {quantity} × {itemId}`)
end)
```

| Erreur stricte fréquente | Correctif |
|---|---|
| `Type 'Instance' does not have key 'Humanoid'` | `FindFirstChildOfClass("Humanoid")` puis une garde |
| `… could be nil` | garde, `assert` ou restructuration ; pas de `::` en rafale |
| `Cannot call non-function` sur une fonction pré-déclarée | `local function` a masqué la déclaration : écris `name = function` |
| `Key 'x' not found in table` en remplissant une table | déclare le type complet d'emblée, ou passe par un constructeur typé |
| `Cannot iterate over a table without indexer` | annote la table en dictionnaire (section 7) |

## 7. Le nouveau solveur de types

- Sorti le 2025-11-20, il est actif d'office pour les scripts nonstrict et nocheck. **Une place qui contient des scripts `--!strict` garde l'ancien solveur** tant que `Workspace.UseNewLuauTypeSolver` n'est pas activé dans la fenêtre Propriétés (catégorie Scripting ; non scriptable). Le bouton bêta a disparu le 2026-01-07. `LuauTypeCheckMode` règle le mode par défaut, pas le solveur.
- Avant de corriger un avertissement, sache quel solveur tourne : dans Studio, et dans luau-lsp, qui a son propre réglage (voir outils.md). Les deux ne disent pas la même chose.
- Réservé au nouveau solveur : `keyof<T>`, `rawkeyof<T>`, `setmetatable<T, M>` en position de type, `type function`, membres `read` et `write`. Une erreur sur ces formes veut souvent dire « mauvais solveur », pas « mauvaise syntaxe » : ne les retire pas d'un projet migré, n'en ajoute pas dans un projet qui ne l'est pas. Exemples : `type SettingKey = keyof<Settings>` donne l'union des clés ; `{ read target: Instance }` déclare un champ en lecture seule, qui accepte un type plus étroit.
- Il n'est pas 100 % rétrocompatible : l'activer sur un gros code strict fait surgir des dizaines d'erreurs. C'est un chantier que l'utilisateur décide.
- Des correctifs de solidité font « casser » du code intact après une mise à jour de Studio. Depuis 0.739, les génériques sont vérifiés dans le corps des fonctions : `local function call<T>(fn: (T) -> T) fn(nil) end` échoue ; corrige la signature (`(T?) -> T`), ne caste pas. En 0.740, même chose pour les intersections de types fonction. Corrige le code, ne repasse pas en nonstrict.
- **Table scellée.** Un littéral à clés nommées (`{ Sword = 1, Shield = 2 }`) est un enregistrement. L'itérer en strict avec le nouveau solveur donne « Cannot iterate over a table without indexer » (l'ancien l'accepte). Annote-le en dictionnaire : `local stock: { [string]: number } = { Sword = 1, Shield = 2 }`. Une table locale non annotée reste ouverte (on peut y ajouter des champs) tant qu'elle n'est ni annotée, ni renvoyée, ni sortie de sa portée ; la passer à une fonction typée ne la scelle pas. Construis-la entière avant de l'annoter.
- Dictionnaire dont une clé peut manquer : `{ [Player]: number? }`. Avec `{ [Player]: number }`, `points[player] + 1` passe l'analyse et plante pour un joueur absent.
- Fonctions de type (`type function`) : elles tournent à l'analyse, jamais à l'exécution, et manipulent des types avec la bibliothèque `types` (`ty:is("table")`, `ty:properties()`, `types.unionof`, `types.singleton`). Un `error()` dedans devient une erreur de type au site d'appel. Réserve-les aux cas où alias, génériques et unions ne suffisent pas. Studio accepte `pcall` dedans, mais luau-lsp 1.69 (aligné sur Luau 0.729) le refusait : revérifie avec ta version de luau-lsp.

## 8. Classes et métatables

Par défaut, des données simples et des fonctions. Une classe sert quand il faut plusieurs objets indépendants qui ont un comportement.

```luau
--!strict
local Wallet = {}
Wallet.__index = Wallet

type WalletData = {
	coins: number,
	owner: Player,
}

-- Forme compatible avec les deux solveurs.
export type Wallet = typeof(setmetatable({} :: WalletData, Wallet))

function Wallet.new(owner: Player, start: number): Wallet
	return setmetatable({ coins = start, owner = owner }, Wallet)
end

-- `self` explicite avec un point : messages d'erreur plus clairs qu'avec `:`.
function Wallet.add(self: Wallet, amount: number)
	self.coins += amount
end

return Wallet
```

- L'appel reste `wallet:add(5)`. Avec le nouveau solveur, `export type Wallet = setmetatable<WalletData, typeof(Wallet)>` marche aussi.
- État sur `self`, jamais sur la table de classe : un champ de classe est partagé par toutes les instances. `__index` oublié : « attempt to call a nil value » au premier appel de méthode. `Wallet:add(5)` sur le module lui-même passe la classe en `self`, dont les champs valent `nil`.
- Composition plutôt qu'héritage : l'héritage brouille les champs, la construction, le nettoyage et l'inférence. S'il le faut : `setmetatable(Derived, { __index = Base })`. Une méthode `destroy` seulement si l'objet possède des ressources : connexions, instances, threads.
- Gèle la classe une fois les méthodes posées (`return table.freeze(Wallet)`) : les mutations accidentelles échouent, et depuis 0.739 la VM met en cache la recherche des métaméthodes d'une métatable gelée.

Métaméthodes de Luau : `__index`, `__newindex`, `__call`, `__tostring`, `__eq`, `__lt`, `__le`, `__len`, `__concat`, `__unm`, les arithmétiques (de `__add` à `__idiv`), `__iter`, `__mode`, `__metatable`. Pas de `__gc`, `__close` ni `__pairs`. Utilise le minimum : pas de surcharge d'opérateurs sur un simple enregistrement.

Comportements vérifiés dans le runtime (2026-10), souvent mal connus :
- `__newindex` ne se déclenche que pour une clé absente de la table brute. Écris avec `rawset` dans le hook (sinon récursion) ; ensuite, les écritures sur cette clé ne passent plus par le hook. Pour voir toutes les écritures, garde la table exposée vide et stocke les valeurs ailleurs.
- `__eq` n'est appelé que si les deux opérandes portent le même handler. Commence-le par `if rawequal(a, b) then return true end`.
- Pour mesurer ou parcourir, seuls `#` (via `__len`) et le `for … in t` généralisé (via `__iter`) voient les métaméthodes. `rawlen`, `ipairs`, `pairs`, `table.insert` et `table.concat` lisent le stockage brut : un proxy dont les données sont derrière `__index` s'itère comme vide.
- Un yield dans une métaméthode lève « attempt to yield across metamethod/C-call boundary ». Rends l'attente explicite au site d'appel.

## 9. Erreurs : pcall, xpcall, error

`pcall(f, ...)` renvoie `true` puis les résultats, ou `false` puis l'erreur. Un succès peut renvoyer `nil` : ne confonds jamais « réussi, rien trouvé » et « échoué ».

```luau
--!strict
local store = game:GetService("DataStoreService"):GetDataStore("PlayerData")

local function loadAsync(userId: number): (boolean, unknown)
	local ok, result = pcall(function()
		return store:GetAsync(`player_{userId}`)
	end)
	if not ok then
		warn(`[Data] lecture échouée pour {userId} : {result}`)
		return false, nil
	end
	-- nil ici = nouveau joueur. C'est un succès, pas un échec.
	return true, result
end
```

- Une lecture ratée n'est pas un profil vide : ne crée pas de données neuves après un `false` (voir donnees.md). Chaque `pcall` traite son échec : log avec contexte, repli ou nouvel essai ; un échec ignoré exprès le dit en commentaire.
- Forme sans closure : `pcall(store.GetAsync, store, key)` ; pour `part:Destroy()`, la closure est plus lisible. Un yield traverse `pcall` et `xpcall` sans problème.
- En `--!strict`, préfère la closure (`pcall(function() return store:GetAsync(key) end)`) quand la méthode a des paramètres optionnels ou reçoit un littéral de table : le nouveau solveur refuse `pcall(store.GetAsync, store, key)` (« Argument count mismatch »), et les deux solveurs refusent un littéral passé à `pcall(HttpService.RequestAsync, HttpService, { … })` (luau-lsp 1.70.1, 2026-10-18). Autre piège de l'ancien solveur : `local ok, err = pcall(f)` échoue (« Function only returns 1 value ») si `f` ne renvoie rien ; écris `xpcall(f, handler)` ou une closure.
- `xpcall(f, handler)` : le handler tourne avant que la pile se déroule, seul moment où `debug.traceback` montre le vrai chemin. Écris `xpcall(f, function(err) return debug.traceback(tostring(err), 2) end)`. Un handler qui lève donne « error in error handling ».
- `error(msg, 2)` accuse l'appelant ; `error(msg, 0)` retire le préfixe `Script:ligne:` ; une erreur peut être une table, `error({ code = "NO_FUNDS" })`. `assert(v, msg)` évalue `msg` même quand tout va bien : en chemin chaud, écris `if not v then error(...) end`, ou garde un message constant.
- Renvoie l'échec quand il fait partie du métier (introuvable, plein, cooldown, entrée invalide) ; lève une erreur pour un invariant cassé ou un mauvais usage d'API. Attrape aux frontières où tu peux agir : handler de remote, boucle de requêtes. Le texte brut d'une erreur va dans les logs, jamais à l'écran du joueur.
- Retry : une lecture se répète (essais bornés, échec visible, backoff `task.wait(2 ^ attempt)`). Une écriture, un octroi d'achat ou un webhook peut se dupliquer : idempotence d'abord (voir donnees.md et monetisation.md).
- Coût : `pcall` et `xpcall` coûtent environ deux fois moins depuis Luau 0.735, mais pas zéro. Ne retire jamais un `pcall` nécessaire pour gagner du temps ; dans une boucle très chaude, entoure la boucle plutôt que chaque tour.
- Vérification réseau à trois états (possession d'un game pass) : `true`, `false`, ou `nil` et un message si l'API a échoué. Ne mets jamais en cache un « non » qui voulait dire « je ne sais pas ».

## 10. task et coroutines

| Appel | Effet |
|---|---|
| `task.spawn(f, ...)` | exécute `f` tout de suite, jusqu'à son premier yield, puis rend la main à l'appelant |
| `task.defer(f, ...)` | exécute `f` à la fin du cycle de reprise en cours |
| `task.delay(n, f, ...)` | reprend `f` sur un Heartbeat au moins `n` s plus tard : jamais avant, plus tard sous charge. Renvoie le thread |
| `task.wait(n?)` | cède une frame, ou au moins `n` s ; renvoie le temps réellement écoulé |
| `task.cancel(thread)` | annule un thread planifié ou en attente |
| `task.desynchronize()`, `task.synchronize()` | passe en phase parallèle ou série (Parallel Luau, voir performance.md) ; erreur hors d'un script placé sous un `Actor` |

- Bannis `wait`, `spawn` et `delay`, dépréciés : `wait` attend au moins 29 ms et renvoie deux valeurs, `spawn` décale aussi d'au moins 29 ms. Bannis aussi `tick()` (section 12). `task.defer` quand l'exécution immédiate ne sert à rien ; `task.spawn` quand le travail doit commencer maintenant.
- Une erreur dans un thread lancé ne remonte pas à l'appelant : elle s'affiche dans la console, c'est tout. Mets un `pcall` dedans si l'échec doit être géré.
- Rien n'est annulé quand une Instance est détruite. Garde le handle et annule-le dans le nettoyage du propriétaire : un `task.delay` en attente sur un objet détruit est un bug latent.
- `task.spawn` dans un handler laisse des événements rapides se chevaucher : protège l'état partagé (verrou, file) et revalide l'état après chaque yield. Remplacer une boucle à 10 Hz par `Heartbeat` donne un scan à 60 Hz : réagis à un signal, ou choisis une cadence et mesure-la.

```luau
--!strict
local Players = game:GetService("Players")

local incomeThreads: { [Player]: thread? } = {}

local function stopIncome(player: Player)
	local thread = incomeThreads[player]
	incomeThreads[player] = nil
	-- N'annule qu'un thread en attente : jamais le thread courant ni un thread fini.
	if thread and coroutine.status(thread) == "suspended" then
		task.cancel(thread)
	end
end

local function startIncome(player: Player, every: number)
	stopIncome(player) -- jamais deux boucles pour le même joueur
	incomeThreads[player] = task.spawn(function()
		while true do
			task.wait(every)
			-- Revalide après le yield : le joueur a pu partir pendant l'attente.
			if player.Parent == nil then
				break
			end
			-- Crédite ici le revenu dans le profil du joueur (côté serveur).
		end
	end)
end

Players.PlayerRemoving:Connect(stopIncome)
```

Coroutines :
- `coroutine.resume` renvoie `false, err` et laisse l'appelant décider ; `coroutine.wrap` propage l'erreur. Les arguments du premier `resume` deviennent les paramètres du corps ; seuls les suivants deviennent les résultats de `coroutine.yield`.
- `coroutine.close(co)` renvoie `true` pour un thread suspendu ou fini, `false, msg` pour un thread mort sur erreur. `status` vaut `"dead"` dans les deux cas : seul le `false` de `resume` les distingue.
- `coroutine.finally` n'existe pas dans Roblox (`nil` dans Studio 0.743). Pour nettoyer : `pcall` autour du corps, `task.cancel` ou `coroutine.close`, ou un Trove.
- Pour enchaîner de l'asynchrone : la bibliothèque Promise d'evaera (annulable, `andThen`, `catch`, `finally`, `Promise.all`). Elle et les autres bibliothèques utiles (Trove ou Janitor, Signal typé) sont dans paquets.md.

## 11. Signaux et événements différés

- `:Connect` renvoie une `RBXScriptConnection` à garder pour `:Disconnect()`. `:Once` se déconnecte après le premier tir. `:Wait()` bloque le thread jusqu'au prochain tir. Les signaux du moteur (`OnServerEvent`…) sont typés `...any` ; une classe `Signal<T...>` typée fait vérifier chaque `Connect` et chaque `Fire`.
- Connecte d'abord, puis traite ce qui existe déjà : `Players.PlayerAdded:Connect(onPlayerAdded)`, puis `for _, player in Players:GetPlayers() do task.spawn(onPlayerAdded, player) end`. Même chose pour `player.Character`.
- `Workspace.SignalBehavior` décide quand tournent les handlers. Lis-le dans la fenêtre Propriétés, ne suppose rien : `Default` vaut Immediate aujourd'hui mais doit devenir Deferred ; les nouvelles places des modèles Roblox sont en Deferred, et Server Authority l'impose.
- En Deferred, un handler tourne au prochain point de reprise, pas au moment du tir :
  - une connexion faite après le tir, dans le même cycle, ne le reçoit pas : connecte avant de provoquer ;
  - une chaîne d'événements réentrants est coupée après 10 niveaux (« Maximum event re-entrancy depth exceeded » dans la console) et les handlers suivants ne tournent pas : passe par une file ;
  - `Destroying` tourne après la destruction : capture l'état avant ;
  - si l'objet est construit par un autre handler différé, un remote tiré tout de suite arrive chez le client avant l'objet : tire-le dans un `task.defer`.
- `Disconnect()` annule aussi les appels déjà en file. `Destroy()` déconnecte, mais les handlers déjà en file tournent quand même : déconnecte d'abord si cela compte.

## 12. Mesurer le temps

| Besoin | API |
|---|---|
| Durée, benchmark, cooldown, limite de débit | `os.clock()` : temps réel écoulé, haute précision, propre à chaque machine |
| Minuterie de jeu dans une session | `time()` : secondes depuis le démarrage de l'instance |
| Horodatage sauvegardé (gains hors ligne, expirations) | `os.time()` : secondes Unix UTC |
| Date à stocker ou afficher | `DateTime` (`DateTime.now().UnixTimestampMillis`, `:FormatUniversalTime`) |
| Horloge commune client-serveur (compte à rebours, latence) | `workspace:GetServerTimeNow()` |
| `tick()` | à bannir (pas déprécié officiellement) : peut se tromper d'une seconde et varie selon le fuseau et l'OS |

**`time()` prend du retard quand le serveur est chargé.** Mesuré dans Studio 0.742 : un `task.wait(0.7)` a duré environ 3 s selon `os.clock()`, mais `time()` n'a avancé que de 0,4 s. Un limiteur de débit basé sur `time()` refuse alors des joueurs honnêtes, et l'exemple officiel de token bucket commet cette erreur. Utilise `os.clock()`. Ne compare jamais le `os.clock()` du client à celui du serveur.

## 13. Chaînes et motifs

- Interpolation `` `{name} a {score} points` `` : toute expression est permise et passe par `tostring`. Échappe avec `` \` `` et `\{`. Pour un format numérique, `string.format("%.2f", x)`, qui n'est pas localisé.
- Pas de `..` dans une boucle (coût quadratique) : remplis une table et fais un seul `table.concat`, ou écris dans un `buffer`.
- `#s` compte des octets, pas des caractères. `utf8.len(s)` compte les points de code (`nil` si l'UTF-8 est invalide), `utf8.codes` les parcourt, `utf8.graphemes` découpe en caractères visibles (un émoji composé = plusieurs points de code), `utf8.nfcnormalize` normalise. `string.lower` et `upper` ne gèrent que l'ASCII, `string.reverse` inverse des octets et casse l'UTF-8 : jamais pour normaliser un pseudo.
- Recherche d'un texte saisi : `string.find(text, needle, 1, true)`, en mode littéral. Sinon `.`, `%` ou `(` deviennent de la syntaxe de motif.
- `string.split(s, sep)` : séparateur littéral (pas un motif), virgule par défaut ; un séparateur vide découpe en octets. `string.pack`, `unpack` et `packsize` lisent et écrivent des chaînes binaires ; pour du binaire modifiable, `buffer`.
- Motifs Lua ≠ regex. Classes `%a %d %l %u %s %w %p` (majuscule = négation), `.` pour tout caractère, quantificateurs `* + ? -` (`-` = `*` paresseux), ancres `^` et `$`, captures `()`. Pas d'alternance, pas de lookaround. Un motif faux ne lève rien : il renvoie `nil`. Traduction : `\d` → `%d`, `\w` → `%w`, `\s` → `%s`, `\.` → `%.`, `*?` → `-`, `[a-zA-Z]` → `%a`.

```luau
--!strict
-- Échappe les caractères magiques pour chercher un nom tel quel.
local function escapePattern(text: string): string
	return (string.gsub(text, "[%(%)%.%%%+%-%*%?%[%]%^%$]", "%%%0"))
end

local function parseVersion(text: string): (number?, number?, number?)
	local major, minor, patch = string.match(text, "^v(%d+)%.(%d+)%.(%d+)$")
	return tonumber(major), tonumber(minor), tonumber(patch)
end
```

- JSON : `HttpService:JSONEncode` et `JSONDecode`, jamais de parseur maison. Identifiant non devinable : `HttpService:GenerateGUID(false)`. Les DataStores exigent de l'UTF-8 valide ; borne une chaîne de joueur en octets (`#s`) avant tout traitement.

## 14. vector, buffer, bit32, EncodingService

**vector**
- `vector.create(x, y, z)`, champs `x`, `y`, `z`, constantes `vector.zero` et `vector.one`, opérateurs `+ - * /` natifs. Fonctions : `magnitude`, `normalize`, `dot`, `cross`, `angle`, `floor`, `ceil`, `abs`, `sign`, `clamp`, `min`, `max`, et `lerp` (doc, 2026-10-18 ; `alpha` n'est pas borné : hors de [0, 1], il extrapole). Repli portable : `a + (b - a) * t`.
- Immuable, sans allocation de table, rapide sous `--!native`. C'est le type natif sous `Vector3`, mais ne mélange pas les styles : `vector.lerp(a, b, t)` est une fonction de bibliothèque, `a:Lerp(b, t)` une méthode de `Vector3`.
- Ne normalise pas un vecteur nul sans direction de repli.

**buffer**
- Bloc d'octets de taille fixe, 1 Gio au plus, rempli de zéros à la création. Il ne grandit pas : pour l'agrandir, crée-en un autre et `buffer.copy`. Offsets en octets à partir de 0, sauf pour `readbits`/`writebits` (en bits). Nombres en little-endian.
- API : `create`, `fromstring`, `tostring`, `len` ; `read`/`write` suivis de `i8`, `u8`, `i16`, `u16`, `i32`, `u32`, `f32`, `f64` ; `readbits(b, bitOffset, bitCount)` et `writebits(b, bitOffset, bitCount, value)`, de 0 à 32 bits ; `readstring(b, offset, count)`, `writestring(b, offset, value, count?)` ; `copy` ; `fill`. `buffer.readinteger` et `writeinteger` (64 bits) existent dans le CLI Luau, pas dans Roblox (2026-10).
- **Deux modes d'échec opposés.** Hors limites : erreur bruyante (« buffer access out of bounds »). Mauvaise largeur dans les limites : succès silencieux (`readu16` sur un champ écrit en `u32` rend les octets bas, `writeu8(b, 0, 300)` stocke 44, `0.1` relu en `f32` ne vaut plus `0.1`). Range offsets et largeurs dans des fonctions par champ, comme ci-dessous.
- Bon usage : paquets réseau fréquents et compacts, formats binaires, compression avant sauvegarde ; inutile pour un petit message occasionnel. Un buffer passé par un remote arrive en copie, les DataStores l'acceptent, et il ne se partage pas entre Actors (utilise `SharedTable`).

```luau
--!strict
-- Paquet de position : id en u16, puis x, y, z en f32 = 14 octets.
local PACKET_SIZE = 14

local function isFinite(n: number): boolean
	return n == n and math.abs(n) ~= math.huge
end

local function encodePosition(id: number, position: Vector3): buffer
	-- writeu16 tronquerait en silence un id hors de 0 à 65535.
	assert(id % 1 == 0 and id >= 0 and id <= 65535, "id hors de la plage u16")
	local packet = buffer.create(PACKET_SIZE)
	buffer.writeu16(packet, 0, id)
	buffer.writef32(packet, 2, position.X)
	buffer.writef32(packet, 6, position.Y)
	buffer.writef32(packet, 10, position.Z)
	return packet
end

-- Côté serveur, vérifie d'abord `typeof(packet) == "buffer"` : un client envoie ce qu'il veut.
local function decodePosition(packet: buffer): (number?, Vector3?)
	if buffer.len(packet) ~= PACKET_SIZE then
		return nil, nil
	end
	local x = buffer.readf32(packet, 2)
	local y = buffer.readf32(packet, 6)
	local z = buffer.readf32(packet, 10)
	-- Un f32 peut coder NaN ou l'infini : rejette-les.
	if not (isFinite(x) and isFinite(y) and isFinite(z)) then
		return nil, nil
	end
	return buffer.readu16(packet, 0), Vector3.new(x, y, z)
end
```

**bit32** : `band`, `bor`, `bxor`, `bnot`, `lshift`, `rshift`, `arshift`, `lrotate`, `rrotate`, `btest`, `extract`, `replace`, `countlz`, `countrz`, `byteswap`, sur des entiers non signés de 32 bits. Utile pour des drapeaux compacts.

**EncodingService** (doc, 2026-10-18) travaille sur des `buffer` (`buffer.fromstring`, `buffer.tostring`), sauf `ComputeStringHash`, qui prend une chaîne et rend une chaîne binaire : `Base64Encode`, `Base64Decode` (lève sur une entrée invalide), `ComputeBufferHash` et `ComputeStringHash` (`Enum.HashAlgorithm` : `Md5`, `Sha1`, `Sha256`, `Blake2b`, `Blake3` ; jamais pour des mots de passe), `CompressBuffer` et `DecompressBuffer` (Zstd, niveaux -7 à 22, défaut 1). Donnée compressée venue d'un client : appelle `GetDecompressedBufferSize` d'abord, et refuse si le résultat est `nil` ou trop grand, sinon une bombe de décompression bloque le serveur.

## 15. Nouveautés : utilisables ou non

État en octobre 2026. Seule une annonce de sortie complète rend une syntaxe publiable : une note de version ne suffit pas, et une syntaxe qui compile dans ton Studio peut casser chez les joueurs.

| Fonction | Statut dans Roblox | Conduite |
|---|---|---|
| Nouveau solveur, `keyof`, `type function`, membres `read` | sorti le 2025-11-20 ; opt-in pour les places strictes | à utiliser si le projet a migré |
| `const NOM = valeur` | disponible (Luau 0.711, sortie générale en avril 2026) | fige le nom, pas la valeur : `const CONFIG = table.freeze({ … })`. Ne l'introduis pas dans un projet existant sans demande ; il faut StyLua ≥ 2.5.0, Selene ≥ 0.31.0 et luau-lsp ≥ 1.64 (changelogs, 2026-10-18) |
| `math.lerp`, `math.map`, `math.isnan`, `math.isinf`, `math.isfinite`, `vector.lerp` | disponibles | à préférer aux versions maison |
| Attributs `@native` et `@deprecated` sur une fonction | disponibles ; `@deprecated` n'agit que sur le lint et l'autocomplétion | `@native` : voir performance.md. Forme avec paramètres `@[deprecated { use = "Shop.buy", reason = "…" }]` (luau.org, 2026-10-18) |
| `if local x = …`, `if const` | bêta Studio depuis le 2026-10-06 | interdit en jeu publié : le script ne compile ni chez les joueurs ni chez un collaborateur sans la bêta. Écris `local x = …` puis `if x then` |
| Valeurs `export` (Luau 0.723) | non confirmé dans Studio | garde `local M = {}` … `return M` |
| Tables exactes `{ x: number, ... }`, classes, `coroutine.finally`, bibliothèque `integer`, `buffer.readinteger` | prototypes inactifs : `nil` ou erreur dans Studio 0.743 | ne les écris jamais |
| Limite de 200 locales assouplie (Luau 0.743) | pas active dans Studio 0.743 | garde les contournements (section 4) |
| Fonctions de bibliothèque ajoutées entre 0.736 et 0.743 | aucune | ces versions sont des correctifs et des optimisations |

## 16. Style

- Nommage (guide de style officiel) : `PascalCase` pour classes, modules, services, types et enums ; `camelCase` pour locales, fonctions, méthodes et paramètres ; `UPPER_SNAKE_CASE` pour les constantes ; `_camelCase` pour le privé. Acronymes écrits comme un mot (`JsonTable`, `MakeHttpCall`), sauf les abréviations d'ensemble (`RGB`, `XYZ`). Mots en entier : `label`, pas `lbl`.
- Suffixe `Async` sur toute fonction qui peut céder (`loadProfileAsync`) : on le voit au site d'appel, et React ne tolère aucun yield pendant un rendu.
- Services en tête, une fois : `local Players = game:GetService("Players")`, variable nommée comme le service. Jamais `game.Players` ni `game.ServerStorage` : la forme pointée casse le typage strict. La globale `workspace` reste permise. Pas de wrapper `getService(name)` : il perd le type précis. Ordre d'un script : services, requires, constantes, types, état, fonctions privées, handlers, initialisation.
- Clauses de garde (retour anticipé, `continue`) plutôt que des `if` imbriqués. Pas de nombre ni de chaîne magique : une table de constantes gelée.
- Une instruction par ligne. Pas de sucre d'appel : `f("x")` et `f({ … })`, pas `f "x"` ni `f { … }`. Ne masque pas une fonction intégrée (`local insert = table.insert`).
- **Une ligne qui commence par `(` juste après une instruction donne « Ambiguous syntax »**, et Roblox rejette tout le script : passe par une locale (le message propose aussi un `;` pour séparer les instructions). Un commentaire `--` dans un littéral de table écrit sur une ligne commente tout le reste de la ligne : mets la note sur sa propre ligne, ou en `--[[ ]]`.
- Instances : règle les propriétés, puis `Parent` en dernier ; jamais `Instance.new("Part", parent)`. Les alias camelCase de l'API (`findFirstChild`, `isA`, `:connect`) marchent mais sont dépréciés : écris `FindFirstChild`, `IsA`, `:Connect`.
- `WaitForChild` en chaîne : une ligne par maillon, pour que l'avertissement « Infinite yield possible » désigne la bonne ligne.

## 17. roblox-ts en une table

Le TypeScript de roblox-ts compile en Luau. Ces différences reviennent sans cesse.

| Luau | roblox-ts |
|---|---|
| `nil` | `undefined` |
| `==`, `~=` | `===`, `!==` |
| tableaux à partir de 1, `#t` | tableaux à partir de 0 (le compilateur décale), `arr.size()` |
| `string.sub(s, 1, 1)` | `s.sub(1, 1)` : les arguments des fonctions `string` restent en base 1 |
| retours multiples | `LuaTuple<[A, B]>` : `const [ok, value] = pcall(f)` |
| `typeof(v) == "Vector3"` | `typeIs(v, "Vector3")` ou `typeOf(v)`, globales du compilateur |
| `inst:IsA("BasePart")` | `inst.IsA("BasePart")` (garde de type) ; `classIs(inst, "Part")` pour la classe exacte |
| `game:GetService("Players")` | `import { Players } from "@rbxts/services"` |
| `task.*`, `os.clock()`, `math.*` | identiques |
| Promise d'evaera | `async` et `await` compilent vers elle |

- roblox-ts refuse certains identifiants réservés à son usage, comme une variable `next` : « Cannot use identifier reserved for compiler internal usage ». Renomme. Ne modifie jamais un `.luau` généré dans `out/` : corrige le `.ts` source.
- Paramètre par défaut pris dans un Enum : `material = Enum.Material.SmoothPlastic` infère le littéral `SmoothPlastic`, puis passer `Enum.Material.Neon` échoue. Annote : `material: Enum.Material = Enum.Material.SmoothPlastic`.

## 18. Pièges

- `--!strict -- commentaire` sur la même ligne : directive ignorée, fichier en nonstrict.
- `if x then` avec `x = 0` ou `""` ; `cond and a or b` avec `a` faux ; `attr or true`.
- NaN qui traverse un test de bornes ; infini qui passe un test de type.
- `table.clone(DEFAULTS)` ou `data or DEFAULTS` avec des sous-tables : données partagées entre joueurs.
- `#t` sur une table trouée ou mixte ; suppression pendant un parcours vers l'avant.
- `local function` appelée par une closure créée plus haut ; `obj:method(obj)` qui passe `self` deux fois.
- Types qui mentent : `{ [Player]: number }` sans `?`, `require` calculé qui rend tout `any`, littéral nommé itéré sous le nouveau solveur.
- `time()` dans une limite de débit ; table faible indexée par Instance.
- `task.delay` jamais annulé ; erreur perdue dans un `task.spawn` ; `while true do` sans yield.
- Ligne qui commence par `(` ; plus de 200 locales dans une fonction.
- `if local`, `coroutine.finally`, `integer`, tables exactes : rien de cela ne tourne en jeu publié.
- Échecs silencieux : `string.find` sans mode littéral sur un texte saisi, largeur fausse dans un `buffer`.

## 19. Checklist

- [ ] `--!strict` seul sur la première ligne d'un fichier neuf, ou mode du projet respecté.
- [ ] Fonctions publiques annotées ; types partagés exportés depuis un module dédié.
- [ ] Entrées externes typées `unknown`, puis affinées (`typeof`, `IsA`) ; nombres testés contre NaN et l'infini ; aucun `any` ni cast `::` pour faire taire une erreur.
- [ ] `require` littéraux en tête de fichier, sans cycle ; aucune globale ; aucune fonction appelée avant sa déclaration.
- [ ] Données par défaut créées par une fonction, jamais par une copie superficielle.
- [ ] Tables d'une seule forme ; aucun trou dans une séquence.
- [ ] `task.*` partout ; aucun `wait`, `spawn`, `delay` ni `tick`.
- [ ] Chaque thread ou `task.delay` lié à un objet est annulé à sa destruction.
- [ ] Chaque `pcall` traite son échec ; « succès avec nil » est distingué d'« échec ».
- [ ] Bonne horloge : `os.clock` pour les durées, `os.time` pour ce qui est sauvegardé, `GetServerTimeNow` entre client et serveur.
- [ ] Aucune syntaxe bêta ni prototype ; le solveur actif est connu avant de corriger un avertissement de type.
- [ ] `luau-lsp analyze` (ou `rbxtsc`) passe sans erreur.
