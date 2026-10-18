# Outils : Rojo, roblox-ts, Rokit, Wally, lint, tests, CI, git

À lire quand : tu montes, synchronises, compiles, lintes, testes ou versionnes un projet Roblox hors de Studio (Rojo, roblox-ts, Rokit ou Aftman, Wally, luau-lsp, Selene, StyLua, Lune, CI, git), ou quand « rien ne s'affiche » après une modification. Les paquets à installer sont dans paquets.md, le pilotage de Studio dans studio-mcp.md, les API web dans open-cloud.md.

## Sommaire

1. Choisir sa route
2. Un seul écrivain par script
3. Rojo 7.7
4. roblox-ts de A à Z
5. Rokit et Aftman
6. Wally et pesde
7. luau-lsp, Selene, StyLua, Darklua
8. Lune
9. Tests
10. CI type
11. Git, fichiers de place, LFS
12. Boucle de dev sur Windows
13. Dépannage
14. Pièges
15. Checklist

## 1. Choisir sa route

Repère d'abord ce qui existe. Ne convertis jamais un projet sans l'accord de l'utilisateur : chaque outil a sa source de vérité, et en changer peut effacer du travail.

| Indice sur le disque | Route en place |
|---|---|
| `*.project.json` (avec `argon.toml` pour Argon) | Rojo, ou Argon |
| `package.json` + `tsconfig.json` + `@rbxts/*` | roblox-ts, servi par Rojo |
| `sourcemap.json` sans fichier projet, `.luau` aux noms de services | Script Sync (ou Azul) |
| `rokit.toml`, `aftman.toml`, `foreman.toml` | versions d'outils épinglées : lis-les avant toute commande |
| `wally.toml` + `Packages/`, ou `pesde.toml` | paquets Luau |
| `selene.toml`, `stylua.toml`, `.luaurc`, `lune/` | conventions et outillage du projet : ils priment sur tes préférences |
| rien de tout ça | Studio seul |

| Route | Source de vérité | Prends-la quand | À savoir |
|---|---|---|---|
| Studio seul (+ MCP) | la place | prototype, utilisateur qui reste dans Studio | pas de git ni de CI ; l'agent édite via le MCP (studio-mcp.md) |
| Script Sync (intégré à Studio, sorti le 2026-06-17) | la place, avec un miroir disque dans les deux sens | Studio reste l'éditeur principal | scripts et dossiers seulement ; pas de types sans luau-lsp |
| Rojo 7.7 | les fichiers | git, CI, Wally, roblox-ts, build sans Studio | sens principal fichiers → Studio |
| Argon 2.0 | réglable | synchro bidirectionnelle voulue | défauts dangereux, voir plus bas |

- **Pile par défaut de l'utilisateur : roblox-ts + `@rbxts/react` + Rojo 7.7**, testée le 2026-10-18 sur Windows. Roblox dit lui-même que Script Sync ne remplace pas Rojo quand les fichiers doivent faire foi.
- **Script Sync.** L'utilisateur l'active dans Studio : clic droit dans l'Explorer, « Sync to… », puis choix d'un dossier. Il ne synchronise que `Script`, `LocalScript`, `ModuleScript` et `Folder`. Les attributs et tags d'un script synchronisé sont ignorés et peuvent être perdus : vérifie les scripts tagués (CollectionService) avant. Limites : 10 000 scripts par racine, 128 racines (doc, 2026-10-18). Pour supprimer une racine, arrête d'abord sa synchro. Un conflit ouvre « Keep Studio / Keep Disk » : c'est l'utilisateur qui répond, jamais l'agent. À la réouverture de la place, une synchro en conflit ne reprend pas par défaut.
- **Rojo et Script Sync n'ont pas les mêmes suffixes.** Ne mélange jamais les conventions (vérifie dans la doc de Script Sync de la version installée) :

| Fichier | Rojo 7.7 (défaut) | Script Sync |
|---|---|---|
| `nom.luau` | ModuleScript | ModuleScript |
| `nom.server.luau` | Script (RunContext Legacy) | Script, RunContext Server |
| `nom.client.luau` | **LocalScript** | **Script, RunContext Client** |
| `nom.local.luau` | ModuleScript `nom.local` (suffixe inconnu) | LocalScript |
| `nom.legacy.luau` | ModuleScript `nom.legacy` (suffixe inconnu) | Script, RunContext Legacy |
| `nom.plugin.luau` | Script, RunContext Plugin | Script, RunContext Plugin |

- `InstanceFileSyncService:GetStatus(instance)` renvoie `SyncedAsRoot`, `SyncedAsDescendant`, `NotSynced`, `Errored` ou `AncestorErrored`. Il est réservé aux plugins : appelle-le par `execute_luau` du MCP, jamais dans le code du jeu. Vue du disque, une synchro en erreur ressemble à une synchro qui marche.
- **Argon** : compatible Rojo, mode TypeScript intégré. Le plugin a `Keep Unknowns = false` par défaut : les instances absentes du disque sont détruites. Son fichier projet porte le même nom que celui de Rojo, avec d'autres clés (`servePort` devient `port`, `emitLegacyScripts` devient `legacyScripts`). N'en parle que si l'utilisateur le demande.
- **Azul** (Studio d'abord) ne marche pas avec roblox-ts : une synchro bidirectionnelle ne réconcilie pas un `.ts` et son Luau généré. **Lync** lit la classe d'un script dans un commentaire d'en-tête (`--@script:server`) : ne supprime jamais ces lignes.

## 2. Un seul écrivain par script

- **`rojo serve` actif : édite les scripts sur disque uniquement.** Jamais le même script par `multi_edit` du MCP ni à la main dans Studio : deux écrivains finissent toujours par s'écraser en silence (avec Rojo, au patch suivant). Le MCP sert à ce qui n'est pas sur disque : instances, propriétés, UI construite dans Studio, terrain, éclairage.
- **Jamais Script Sync et `rojo serve` sur le même arbre** : ce sont deux écrivains concurrents. En roblox-ts, `out/` n'a qu'un écrivain, `rbxtsc`.
- **Dis de quel côté tu as écrit** et ce que l'utilisateur doit faire, par exemple : « fichier écrit ; si `rbxtsc -w` et `rojo serve` tournent, Studio l'a déjà reçu ».
- **Ne démarre, n'arrête ni ne reconfigure une synchro de ta propre initiative.** `rojo syncback`, un `azul build --destructive` ou le `Keep Unknowns` d'Argon peuvent supprimer des instances qui n'existent que dans la place.
- **Avant de modifier plusieurs scripts par le MCP, fais sauvegarder ou commiter** : le MCP n'a pas d'annulation groupée.
- **En équipe, une place par développeur** pour la synchro live, plutôt que plusieurs `rojo serve` connectés à la même session Team Create.

## 3. Rojo 7.7

Versions vues : 7.7.0 (juillet 2026 : `rojo syncback`, transport websocket, fichiers `.jsonc`) et 7.7.1 (2026-10-01). Pas de 8.x en octobre 2026. Les guides en 7.4 restent lisibles, mais épingle 7.7.1.

- La CLI s'installe par Rokit ou Aftman (section 5), par le binaire des releases GitHub `rojo-rbx/rojo`, ou par `cargo install rojo --version ^7`, toujours avec l'accord de l'utilisateur. L'extension VS Code `evaera.vscode-rojo` ne met pas `rojo` dans le PATH.
- Plugin Studio : `rojo plugin install`. Sous Windows, il écrit `RojoManagedPlugin.rbxm` dans `%LOCALAPPDATA%\Roblox\Plugins`. Redémarre Studio si l'onglet n'apparaît pas. Le plugin doit avoir la version majeure de la CLI : réinstalle-le après chaque mise à jour de la CLI.

### Fichier projet

`default.project.json` (commentaires et virgules finales acceptés depuis 7.6.1), ou `default.project.jsonc` (7.7). Rojo rejette toute clé inconnue : n'invente aucune clé, vérifie avec `rojo --help` ou la doc. Projet Luau avec Wally (le projet roblox-ts est en section 4) :

```json
{
  "name": "mon-jeu",
  "tree": {
    "$className": "DataModel",
    "ReplicatedStorage": {
      "Shared": { "$path": "src/shared" },
      "Packages": { "$path": "Packages" },
      "Remotes": { "$className": "Folder", "BuyItem": { "$className": "RemoteEvent" } }
    },
    "ServerScriptService": {
      "Server": { "$path": "src/server" },
      "ServerPackages": { "$path": "ServerPackages" }
    },
    "StarterPlayer": { "StarterPlayerScripts": { "Client": { "$path": "src/client" } } }
  }
}
```

Lance `wally install` avant `rojo serve` : un `$path` vers un dossier absent fait échouer le chargement du projet.

| Clé racine | Rôle |
|---|---|
| `name` | nom de l'artefact ; facultatif dans `default.project.json` depuis 7.4.1 |
| `tree` | arbre d'instances, obligatoire |
| `servePort`, `serveAddress` | port (34872 par défaut) et adresse d'écoute |
| `servePlaceIds`, `blockedPlaceIds` (ce dernier depuis 7.5) | n'autorise la synchro live que vers ces places, ou interdit celles-ci (la production) |
| `serveAllowedHosts` (7.7+) | hôtes acceptés si le serve est exposé ; `--allowed-hosts` le surcharge |
| `globIgnorePaths` | globs ignorés, par exemple `**/*.spec.luau` |
| `emitLegacyScripts` | `true` par défaut : `.server`/`.client` donnent Script et LocalScript ; `false` donne des Script avec RunContext Server ou Client. Garde le réglage du projet : c'est un choix, pas une correction |
| `syncRules` (7.5+), `syncbackRules` (7.7) | règles fichier → type d'instance (`pattern`, `use`, `exclude`, `suffix`) ; réglages de `rojo syncback` |

- Clés d'un nœud : `$className` (facultatif si `$path` la donne ou si la clé est un service), `$path` (relatif au fichier projet), `$properties`, `$attributes`, `$ignoreUnknownInstances`, `$id`. Toute autre clé est un enfant.
- **`$ignoreUnknownInstances` vaut `false` sous un `$path`** : Rojo supprime les enfants créés dans Studio qui n'existent pas sur disque. Range ce qui est fait dans Studio hors des dossiers gérés, ou passe la clé à `true`.
- Un service listé sans `$path` garde ses enfants existants : pratique pour injecter des scripts dans une place existante. Si le code écrit dans des DataStores, prends une expérience de test séparée : les places d'une même expérience partagent ses DataStores.
- **Propriétés** : préfère la forme implicite, validée par Rojo (`"Anchored": true`, `"Size": [2, 1, 4]`, `"Material": "Wood"`). La forme explicite (`{ "Vector3": [2, 1, 4] }`, `{ "Enum": 512 }`) saute la validation : utile pour une API trop récente, mais une faute ne se voit qu'à l'exécution. `Region3` et `SharedString` ne s'écrivent pas depuis un projet ; un `Ref` se lie depuis 7.5 : `$id` (ou `id`) sur la cible, puis un attribut `Rojo_Target_<Propriété>` qui porte cet id. Le format Rojo 6 `{ "Type": "Enum", "Value": 512 }` est mort : Rojo 7 attend `{ "Enum": 512 }`. Un `Workspace.FilteringEnabled` dans un projet ne sert à rien : la propriété est obsolète et toujours active.

### Fichiers → instances

| Sur disque | Dans Studio |
|---|---|
| dossier | `Folder` |
| `x.luau` (ou `.lua`) | ModuleScript |
| `x.server.luau` / `x.client.luau` / `x.plugin.luau` | Script / LocalScript / Script RunContext Plugin |
| `dossier/init.luau`, `init.server.luau`, `init.client.luau` | le dossier devient ce script, ses autres fichiers ses enfants (un seul `init` par dossier) |
| `x.rbxm`, `x.rbxmx` | modèle binaire ou XML (XML pour des diffs lisibles) |
| `x.model.json` | arbre d'instances écrit à la main |
| `x.json`, `x.toml`, `x.yaml` (YAML depuis 7.6) | ModuleScript qui renvoie la table |
| `x.csv` / `x.txt` | LocalizationTable / StringValue |
| `x.meta.json`, `init.meta.json` | métadonnées de l'instance voisine ou du dossier |
| dossier avec un `default.project.json` | remplacé par l'arbre de ce projet (un modèle, pas une place) |

- Un `x.luau` dans `StarterPlayerScripts` devient un ModuleScript qui ne s'exécute jamais : nomme-le `x.client.luau`. Un module nommé `x.server.luau` devient un Script, et `require` échoue : plusieurs modèles publics font cette erreur.
- Un `init.luau` ajouté par mégarde change la classe de tout le dossier. Rojo normalise les sources en LF : pas de faux diff CRLF dans Studio.
- **Méta-fichiers.** `src/server/Workers/init.meta.json` contenant `{ "className": "Actor" }` fait du dossier un `Actor` (Parallel Luau) ; `className` n'est permis que dans un `init.meta.json`. `src/server/Debug.meta.json` contenant `{ "properties": { "Enabled": false }, "attributes": { "Owner": "dev" } }` règle le Script voisin `Debug.server.luau`.
- **Modèle JSON.** `Remotes.model.json` contenant `{ "className": "Folder", "children": [{ "name": "BuyItem", "className": "RemoteEvent" }] }` crée des remotes sans script (les anciennes clés `ClassName` et `Children` restent lues).
- **Limites de la synchro live** : terrain, unions (CSG), `MeshPart.MeshId` et `HttpService.HttpEnabled` ne passent pas en live. C'est une limite de l'API plugin, pas une mauvaise config : fais un `rojo build` puis ouvre la place, ou règle-les dans Studio. `ServerScriptService.LoadStringEnabled` (non scriptable) n'a pas non plus été appliqué depuis un projet servi par Rojo : coche-le dans Studio.

### Commandes

```bash
rojo init mon-jeu                                       # --kind place|model|plugin ; --skip-git évite le git init
rojo serve                                              # lit default.project.json ; --port 34873 si le port est pris
rojo build default.project.json -o build.rbxl           # format choisi par l'extension : .rbxl .rbxlx .rbxm .rbxmx
rojo build plugin.project.json --plugin MonPlugin.rbxm  # écrit dans le dossier Plugins local
rojo sourcemap default.project.json -o sourcemap.json   # --include-non-scripts, --watch
rojo syncback default.project.json --input place.rbxl --dry-run --list
rojo fmt-project                                        # reformate le fichier projet
```

- `rojo build` prouve que le projet est valide (fichier cassé, `$path` faux, propriété invalide) : cite sa ligne de sortie. Il ne prouve pas que le code marche.
- `rojo upload` existe (`--asset_id`, puis `--api_key` et `--universe_id`, ou un cookie). Il met la clé dans la ligne de commande : préfère la publication décrite dans open-cloud.md.
- Réglages du plugin : mode de confirmation des patchs (Initial, Always, Large Changes, Unlisted PlaceId), Open Scripts Externally. Le « Two-Way Sync » du plugin est expérimental et incomplet : ne bâtis rien dessus.
- **Sécurité du serve.** `rojo serve` écoute sur `127.0.0.1` par défaut. Depuis 7.7, il valide les en-têtes Host et Origin (contre le DNS rebinding), réserve `/api/open` aux clients locaux et avertit si l'écoute n'est pas locale. N'expose le serve (`--address`, `serveAllowedHosts`) que sur demande explicite : quiconque l'atteint pousse du code dans la place ouverte. Dès que la place est publiée, ajoute `servePlaceIds` (ou `blockedPlaceIds` avec l'id de la production) : un projet ne doit jamais écraser la mauvaise place.

### Syncback (fichier de place → fichiers)

- `rojo syncback` convertit **hors ligne** une place sauvée (`.rbxl`, `.rbxlx`, `.rbxm`, `.rbxmx`) en fichiers, selon l'arbre du projet. Ce n'est pas une synchro live. Il n'écrit que les instances de l'arbre du projet et leurs descendants : pour récupérer `Workspace`, `Workspace` doit figurer dans le projet. Garde souvent un fichier projet dédié au syncback.
- Il peut réécrire ou supprimer des fichiers mappés. Lance-le seulement sur demande explicite, sur un arbre git propre, après un aperçu `--dry-run --list` (la liste sort sur stdout, les autres messages sur stderr). Il demande ensuite une confirmation (Y/N) : n'ajoute `-y`, qui la saute, qu'avec l'accord de l'utilisateur. Règles `syncbackRules` : `ignoreTrees` (chemins dans la place), `ignorePaths` (chemins disque), `ignoreProperties` (par classe), `syncCurrentCamera` (false par défaut), `syncUnscriptable` (true par défaut), `ignoreReferents` (false par défaut).
- **Projet roblox-ts : jamais de syncback vers `out/`**, c'est de la sortie de build. Limite-le aux modèles (un dossier `assets/`, par exemple) avec un projet dédié.
- Porter un jeu existant : range d'abord dans Studio (logique dans ServerScriptService, ReplicatedStorage et StarterPlayer ; scripts copiés-collés remplacés par un module et des tags), garde une copie de la place, puis extrais (`rojo syncback`, `rbxlx-to-rojo` ou Lune) au lieu de retaper.

## 4. roblox-ts de A à Z

roblox-ts compile du TypeScript en Luau. Version npm de référence vue en 2026 : 3.0.x. Un fork `@isentinel/roblox-ts` 4.x existe, de provenance non vérifiée : n'en change pas sans demande.

### Créer un projet (testé le 2026-10-18)

```bash
npx -y create-roblox-ts@latest game --dir . --git=false --eslint --prettier --vscode --packageManager npm --skipBuild
npm i @rbxts/react @rbxts/react-roblox @rbxts/services
```

- La commande est non interactive : `game` choisit le modèle (les autres sont `model`, `plugin` et `package`), `--dir .` crée dans le dossier courant, `--skipBuild` saute la première compilation. Prérequis : Node.js LTS et git. Le modèle `game` règle déjà le JSX pour React. L'UI et les leçons UI du même test (JSX en minuscules, `createRoot`, bindings, `UIScale`, layouts, `ViewportFrame`) sont dans ui-react.md ; les autres paquets dans paquets.md.
- **Ajoute tout de suite le scope `@rbxts-js` au projet Rojo** (ci-dessous). Sans lui, `@rbxts/react` ne trouve pas ses vrais modules et aucune UI n'apparaît.

### Structure

```text
src/server/  → out/server → ServerScriptService.TS
src/client/  → out/client → StarterPlayer.StarterPlayerScripts.TS
src/shared/  → out/shared → ReplicatedStorage.TS
out/         sortie Luau générée : ne jamais l'éditer
include/     RuntimeLib et Promise recopiés par le compilateur : générés aussi
```

- `main.server.ts` donne un Script, `main.client.ts` un LocalScript, tout autre `.ts` ou `.tsx` un ModuleScript : Rojo lit le suffixe du `.luau` produit dans `out/`. Lis et modifie seulement `src/`. Une ligne d'erreur dans Studio vise le `.luau` de `out/` : ouvre-le pour situer l'erreur, puis corrige le `.ts` du même chemin.

`default.project.json` du modèle, avec le correctif `@rbxts-js` :

```json
{
  "name": "mon-jeu",
  "globIgnorePaths": ["**/package.json", "**/tsconfig.json"],
  "tree": {
    "$className": "DataModel",
    "ServerScriptService": { "TS": { "$path": "out/server" } },
    "ReplicatedStorage": {
      "rbxts_include": {
        "$path": "include",
        "node_modules": {
          "$className": "Folder",
          "@rbxts": { "$path": "node_modules/@rbxts" },
          "@rbxts-js": { "$path": "node_modules/@rbxts-js" }
        }
      },
      "TS": { "$path": "out/shared" }
    },
    "StarterPlayer": { "StarterPlayerScripts": { "TS": { "$path": "out/client" } } }
  }
}
```

- **Un enfant par scope npm utilisé à l'exécution** sous `rbxts_include/node_modules` : `@rbxts`, `@rbxts-js` pour React, `@flamework` pour Flamework. Un scope absent donne un `Infinite yield` sur `WaitForChild("@scope")`. Rojo recharge ce fichier à chaud : inutile de relancer `rojo serve`.
- `globIgnorePaths` évite de synchroniser comme instances les `package.json` et `tsconfig.json` de `node_modules`. Certains modèles posent aussi `HttpService.HttpEnabled` dans le projet : ce réglage ne passe pas en live. Active-le dans Studio, et seulement si le jeu fait du HTTP.
- De vieux guides mappent `node_modules` directement sur `node_modules/@rbxts`, sans dossier par scope (les autres scopes existent depuis roblox-ts 1.2.1) : le runtime de la 3.x cherche `node_modules/<scope>/<module>`, donc les modules deviennent introuvables.

### tsconfig.json

Le modèle fournit la bonne configuration. Vérifie ces clés si le projet vient d'ailleurs :

```json
{
  "compilerOptions": {
    "allowSyntheticDefaultImports": true, "downlevelIteration": true, "strict": true,
    "jsx": "react", "jsxFactory": "React.createElement", "jsxFragmentFactory": "React.Fragment",
    "module": "commonjs", "moduleResolution": "Node", "moduleDetection": "force", "target": "ESNext",
    "noLib": true, "resolveJsonModule": true, "experimentalDecorators": true,
    "forceConsistentCasingInFileNames": true, "typeRoots": ["node_modules/@rbxts"],
    "rootDir": "src", "outDir": "out", "incremental": true, "tsBuildInfoFile": "out/tsconfig.tsbuildinfo"
  }
}
```

- `noLib` retire les bibliothèques DOM et ES : les types viennent de `@rbxts/types` (API Roblox) et `@rbxts/compiler-types` (`LuaTuple`, macros, Promise). C'est pourquoi `document`, `setTimeout` ou `Object.keys` n'existent pas. `allowSyntheticDefaultImports` permet `import React from "@rbxts/react"` sur un paquet publié en `export =`.
- Flamework demande en plus `node_modules/@flamework` dans `typeRoots`, le plugin `rbxts-transformer-flamework` dans `plugins` et le scope dans Rojo (architecture.md).
- VS Code : utilise le TypeScript du projet (`typescript.tsdk` vers `node_modules/typescript/lib`). Si l'éditeur propose l'aperçu du TypeScript natif (« tsgo »), laisse-le désactivé : le compilateur se sert du TypeScript installé.

### Compiler

```bash
npx rbxtsc          # build unique, à faire passer à zéro erreur avant tout test
npx rbxtsc -w       # watch : recompile à chaque sauvegarde
npx rbxtsc --verbose
```

- Autres options : `-p <dossier>`, `--rojo <fichier>`, `--noInclude`, `--logTruthyChanges`, `--usePolling`. Vérifie avec `npx rbxtsc --help` : `--noEmit` et `--production`, cités par certains guides, n'existent pas.
- roblox-ts exige une version précise de TypeScript (`=5.5.3` pour roblox-ts 3.0.0, npm 2026-10-18) : ne mets jamais `typescript` à jour seul. `@rbxts/compiler-types` suit la majeure de roblox-ts (3.x avec 3.x). Mets `@rbxts/types` à jour quand une API récente manque.
- Lint et format du TypeScript : ESLint (avec le plugin roblox-ts) et Prettier, installés par `--eslint --prettier`. Selene, StyLua et luau-lsp ne servent que pour du Luau écrit à la main.

### Différences avec TypeScript et JavaScript

| Réflexe JS/TS | En roblox-ts |
|---|---|
| `arr.length`, `str.length`, `map.size` | `arr.size()`, `str.size()`, `map.size()` |
| `arr[0]` | premier élément : les tableaux restent indexés à 0 côté TS, le compilateur décale |
| `str.substring(0, 1)` | `str.sub(1, 1)` : les fonctions `string` de Luau gardent leurs indices à partir de 1 |
| `null` | interdit : `undefined` |
| `==`, `!=` | erreur de compilation : `===`, `!==` |
| `x \|\| 10` | `0`, `""` et `NaN` restent faux comme en JS : écris `x ?? 10` pour un nombre |
| `any` | erreur dès qu'on l'utilise : `unknown` puis des gardes |
| `typeof v` pour un type Roblox | `typeIs(v, "Vector3")` ; `typeOf(v)` est le `typeof` de Luau |
| `inst instanceof Part` | `inst.IsA("BasePart")`, ou `classIs(inst, "Part")` pour la classe exacte |
| `get x()`, `set x()` | interdits : écris une méthode |
| `for (const k in obj)` | interdit : `Map` et `for (const [k, v] of map)` |
| `Object.keys`, `Object.entries` | absents : `Map`, ou un paquet utilitaire (paquets.md) |
| `RegExp`, `/motif/` | absents : motifs Luau (`string.match`, `string.gsub`) |
| `Symbol`, fusion de `namespace` ou d'`enum` | non supportés ; un `namespace` simple marche |
| `setTimeout`, `setInterval` | `task.delay`, `task.wait` dans une boucle, ou un signal |
| `Date.now()`, `console.log`, `JSON.stringify` | `os.time()` ou `os.clock()`, `print` ou `warn`, `HttpService.JSONEncode` |
| `game.GetService("Players")` | `import { Players } from "@rbxts/services"` |
| `new Instance("Part", parent)` | crée, règle les propriétés, puis `part.Parent = parent` en dernier |
| retours multiples | `LuaTuple<[A, B]>` : `const [ok, err] = pcall(() => …)` |
| `async`, `await`, `Promise` | Promise de roblox-ts (celle d'evaera), annulable : gère toujours le rejet |
| passer `this.onHit` en callback | `() => this.onHit()` |

- Décorateurs supportés avec `experimentalDecorators` (Flamework s'en sert). `export =` et `export default` marchent tous les deux.
- **Identifiants réservés.** Une variable nommée `next` échoue avec « Cannot use identifier reserved for compiler internal usage » : renomme-la (`nextIndex`). Un mot-clé Luau comme nom (`end`, `local`, `then`, `repeat`, `until`, `elseif`, `nil`, `not`, `and`, `or`) échoue aussi, avec « Invalid Luau identifier! » : renomme-le.
- **Défaut typé par un membre d'Enum.** `material = Enum.Material.SmoothPlastic` fait inférer le type littéral `SmoothPlastic`, et passer `Enum.Material.Neon` échoue. Annote : `material: Enum.Material = Enum.Material.SmoothPlastic`.
- Un cast `FindFirstChild("X") as Part` ne vérifie rien : teste l'existence et la classe d'abord. Idiomes vus en vrai : `string.format`, `math.round`, `task.spawn`, `task.delay`, `task.cancel`, `UDim2.fromOffset`, `udim2.add(autre)`, `color.Lerp(autre, t)`.
- Coût pour un agent : un watcher à garder lancé, des fichiers générés à ne pas toucher, des lignes d'erreur décalées. Le Luau pur est plus direct, mais suis le choix de l'utilisateur : ici, roblox-ts.

## 5. Rokit et Aftman

Un gestionnaire d'outils épingle la version exacte de chaque CLI par projet. Sans lui, chaque machine a sa version et la CI diverge. Commite toujours le manifeste.

- **Rokit** (1.2.0 en octobre 2026) : le choix pour un nouveau poste. Il lit aussi `aftman.toml` et `foreman.toml`.
- **Aftman** : archivé en 2025 mais fonctionnel. C'est l'outil installé sur la machine de l'utilisateur (test du 2026-10-18). Un projet Aftman garde Aftman : migrer n'est pas ta décision. Foreman sert surtout à l'usage interne de Roblox.

Versions vues en octobre 2026 : Rojo 7.7.1, Rokit 1.2.0, Wally 0.3.2, pesde 0.7.4, Selene 0.32.0, StyLua 2.6.0, luau-lsp 1.70.1, Lune 0.11.0, Argon 2.0.29, Jest Roblox 3.20. Un guide qui cite StyLua 0.20 ou Rojo 7.4 est périmé ; avant d'épingler, vérifie la dernière release (`gh release list -R rojo-rbx/rojo -L 3`). Manifeste (même format pour `rokit.toml` et `aftman.toml`), à réduire aux outils que le projet utilise :

```toml
[tools]
rojo = "rojo-rbx/rojo@7.7.1"
lune = "lune-org/lune@0.11.0"
wally = "UpliftGames/wally@0.3.2"
selene = "Kampfkarren/selene@0.32.0"
stylua = "JohnnyMorganz/StyLua@2.6.0"
luau-lsp = "JohnnyMorganz/luau-lsp@1.70.1"
```

```bash
rokit init                       # crée rokit.toml
rokit add rojo-rbx/rojo@7.7.1    # ajoute au manifeste et installe
rokit install                    # installe tout le manifeste (nouveau poste, CI)
aftman init; aftman add rojo-rbx/rojo@7.7.1; aftman install   # équivalent Aftman
```

- Installer Rokit lui-même : télécharge le zip de la release GitHub `rojo-rbx/rokit`, extrais-le, lance `rokit self-install` (il ajoute `~/.rokit/bin` au PATH de l'utilisateur), puis ouvre un nouveau terminal. Autre voie : `cargo install rokit --locked` puis `rokit self-install`. Demande l'accord avant ; jamais l'installeur en `curl | bash` ni en `irm | iex`.
- **Les shims ne marchent que dans un dossier dont le manifeste liste l'outil.** Sinon : « no aftman.toml files list this tool » (Aftman) ou « Failed to find tool … in any project manifest file » (Rokit). Lance les outils depuis la racine du projet, et ajoute l'outil au manifeste.
- Au premier usage, Rokit demande de faire confiance à l'outil. Dans un terminal non interactif : `rokit trust <owner/repo>`, après accord de l'utilisateur. N'utilise pas `rokit install --no-trust-check`, qui saute ce contrôle (sa doc le réserve aux machines de CI). Ne compte pas non plus sur des outils installés globalement : leur version dérive d'une machine à l'autre.

## 6. Wally et pesde

Pour un projet Luau. En roblox-ts, les paquets viennent de npm (`npm i @rbxts/…`) et Wally ne sert presque jamais. Les identifiants exacts des paquets sont dans paquets.md.

Wally : CLI 0.3.2 stable (préversions 0.4.0-alpha.0 le 2026-09-26 et 0.4.0-alpha.1 le 2026-10-12). La CLI bouge peu, mais le registre `wally-index` reste actif, et Roblox y publie ses nouvelles bibliothèques officielles sous le scope `roblox/`.

```toml
[package]
name = "moi/mon-jeu"
version = "0.1.0"
registry = "https://github.com/UpliftGames/wally-index"
realm = "shared"
private = true

[dependencies]
Promise = "evaera/promise@4.0.0"
```

- `wally install` remplit `Packages/` (`[dependencies]`), `ServerPackages/` (`[server-dependencies]`) et `DevPackages/` (`[dev-dependencies]`). Mappe ces dossiers dans le projet Rojo, puis requiers depuis `ReplicatedStorage.Packages`.
- Il n'y a pas de `wally add` : édite `wally.toml` à la main, puis `wally install`. `wally search <mot>` cherche un paquet. `"evaera/promise@4.0.0"` est une **plage caret** (`^4.0.0`), pas une version figée : écris `=4.0.0` pour figer. Commite `wally.lock` et relis son diff en revue.
- Le résolveur ne revient pas en arrière. Sur « All possible candidates … conflicted », lis la paire de contraintes incompatibles et corrige le manifeste. Ne supprime jamais `wally.lock` comme correctif par défaut.
- Realms : il n'y a pas de realm client. Une dépendance `shared` vers un paquet `server` échoue à l'installation, et un paquet serveur est absent côté client. Il n'y a pas de `wally install --locked` en 0.3.2 (le flag apparu dans la branche 0.4.0 a été retiré en 0.4.0-alpha.1) : `wally install` reprend les versions de `wally.lock`. En CI, fais échouer le job si `git diff --exit-code wally.lock` montre une différence après l'installation.
- **Types des paquets** : les modules relais de Wally ne ré-exportent pas les types, et luau-lsp signale des erreurs dans du code correct. Correctif :

```bash
wally install
rojo sourcemap default.project.json --output sourcemap.json
wally-package-types --sourcemap sourcemap.json Packages/
```

- Ne commite ni n'édite jamais `Packages/` : il est régénéré. Copier un module dans le projet (vendoring) reste correct, avec un commentaire de provenance en tête (URL, version, licence). Avant de publier un paquet, `wally package --list` sert d'aperçu (il n'y a pas de `wally publish --dry-run`).
- **pesde** (0.7.4, 2026-09-09) : alternative active, manifeste `pesde.toml`, cibles Roblox, Lune et Luau, sortie dans `roblox_packages/`, `pesde.lock` à commiter. Il installe aussi des paquets Wally via `[wally_indices]`, et les types demandent un script `sourcemap_generator`. Ne bascule jamais Wally ↔ pesde en cours de projet sans demande.

## 7. luau-lsp, Selene, StyLua, Darklua

Pour le Luau écrit à la main. Ces outils ne lisent pas le TypeScript.

### luau-lsp

- Il lui faut les **définitions de l'API Roblox** et un **sourcemap** Rojo. Sans elles, tout global Roblox est « Unknown global » et les `require` entre fichiers ne se résolvent pas.
- `luau-lsp analyze` ne télécharge pas les définitions. Prends `globalTypes.None.d.luau` pour le code de jeu (niveau de sécurité d'un Script) et `globalTypes.PluginSecurity.d.luau` pour un plugin : un appel réservé aux plugins dans du code de jeu devient alors une erreur. Source : `https://raw.githubusercontent.com/JohnnyMorganz/luau-lsp/main/scripts/`. Cette branche bouge : pour une CI reproductible, copie le fichier dans `types/` avec sa somme SHA-256 et rafraîchis-le exprès.

```bash
rojo sourcemap default.project.json --output sourcemap.json
luau-lsp analyze --platform=roblox --definitions:@roblox=types/globalTypes.None.d.luau --sourcemap=sourcemap.json --ignore="**/_Index/**" src
```

- Ajoute `--include-non-scripts` au sourcemap quand le code référence des parts ou d'autres instances. Lance `wally install` avant, sinon chaque `require` de paquet échoue.
- Analyse en `--!strict`, même un extrait : le mode nonstrict ne signale pas les globaux inconnus. Depuis luau-lsp 1.56, chaque fichier de définitions porte un nom (`--definitions:@roblox=…`) ; un chemin sans nom n'est plus accepté que par compatibilité. Pour imiter le nouveau solveur de types de Studio : `--flag:LuauSolverV2=true`, ou `luau-lsp.fflags.enableNewSolver` dans le fichier passé à `--settings`. L'avertissement « client does not allow didChangeWatchedFiles registration » est sans effet. Une erreur cite le fichier deux fois, chemin disque puis chemin DataModel : `[game/ReplicatedStorage/Shared/Bad](3,1)`.
- VS Code : extension `JohnnyMorganz.luau-lsp`, réglages `luau-lsp.platform.type = "roblox"` et `luau-lsp.sourcemap.autogenerate = true`. Le plugin Studio compagnon donne les types des instances qui n'existent que dans Studio. « Le LSP dit que ça n'existe pas » vient presque toujours d'un sourcemap périmé : régénère-le d'abord.
- Une vérification de types verte ne prouve pas qu'une écriture moteur marche : `game:GetService("HttpService").HttpEnabled = true` passe luau-lsp mais échoue à l'exécution dans un Script (écriture réservée à `LocalUserSecurity`). Exécute une fois le code qui touche au moteur (studio-mcp.md).

### Selene

```toml
std = "roblox"

[lints]
unused_variable = "warn"
shadowing = "warn"
global_usage = "deny"
```

- La table s'appelle `[lints]`, pas `[rules]` : avec `[rules]`, tes niveaux sont ignorés. `std = "roblox"` est obligatoire, sinon chaque `game` ou `Instance` est signalé. Selene génère sa bibliothèque Roblox (`roblox.yml`) au premier lancement ; `selene update-roblox-std` la rafraîchit. Avec TestEZ : `std = "roblox+testez"`, plus un fichier `testez.yml` dans le projet.
- `selene src` sort en code non nul dès un avertissement (`--allow-warnings` les laisse passer). Si Selene refuse un motif voulu, ajuste `selene.toml` ou place `-- selene: allow(nom_du_lint)` juste avant la ligne, avec la raison. Ne supprime jamais la vérification.

### StyLua

```toml
syntax = "Luau"
column_width = 100
line_endings = "Unix"
indent_type = "Tabs"
indent_width = 4
quote_style = "AutoPreferDouble"
call_parentheses = "Always"
```

- `stylua --check src` en CI : il affiche le diff et sort en 1. `stylua src` reformate : fais-le comme action voulue, dans un commit séparé des changements de gameplay. La config du projet prime sur tes goûts. `-- stylua: ignore` protège une instruction ; `.styluaignore` exclut des fichiers.
- Sous Windows, `line_endings = "Unix"` avec un checkout en CRLF fait échouer `--check` partout : fixe les fins de ligne dans `.gitattributes` (section 11).

### Darklua

- Darklua transforme du Luau par règles : `require` par chaîne convertis en chemins d'instances (avec un sourcemap Rojo), regroupement, minification. Commande de base : `darklua process src dist --config .darklua.json` ; prends les noms de règles dans sa doc (darklua.com/docs/rules-reference, par exemple `convert_require` ou `remove_types`) ; sans `--config`, il lit `.darklua.json` ou `.darklua.json5`. Sa sortie est générée, comme celle de roblox-ts : un `.luau` qui revient après ton édition vient de l'un d'eux, corrige la source.

## 8. Lune

Lune 0.11.0 (2026-10-11) exécute du Luau hors de Roblox : tests de logique pure, scripts de build, lecture et écriture de fichiers de place. Ce n'est pas le moteur : pas de physique, pas de rendu, une API `Instance` partielle.

```bash
lune run tests/run    # cherche dans le dossier courant, puis lune/ et .lune/
lune list             # scripts disponibles (description : commentaire --> en tête)
lune setup            # définitions de types et .luaurc pour luau-lsp
```

- Bibliothèques : `@lune/fs`, `net`, `process`, `stdio`, `task`, `datetime`, `luau`, `regex`, `serde`, `roblox`. Dans Lune, `require` prend un chemin relatif au fichier (`require("../src/shared/Damage")`) ; en jeu, c'est `require(ReplicatedStorage.Shared.Damage)`. Écris la logique dans des modules qui reçoivent des valeurs simples, sans `game:GetService`, pour que les deux marchent.
- **Sécurité** : un script Lune a tout accès à la machine (fichiers, programmes, réseau), et `roblox.getAuthCookie()` lit le vrai cookie `.ROBLOSECURITY`. Ne lance jamais un script que tu n'as pas lu.
- Lune 0.8.x ne lit pas les places sauvées par un Studio récent : utilise 0.10.5 ou plus (0.10.5 corrige la lecture des places dont les tags sont en `SharedString`). `process.spawn` est remplacé par `process.exec` et `process.create` depuis la 0.9.0. En 0.11, `handle.stop()` d'un `net.serve` devient `handle:stop()`.
- **Vérifier une place construite sans Studio** : `roblox.deserializePlace(fs.readFile("build.rbxl"))` (la fonction prend le contenu, pas le chemin), puis `place:FindFirstChildOfClass("ReplicatedStorage")` et `FindFirstChild("TS")`, et `process.exit(1)` si un élément manque. Dans un outil en lecture, préfère `FindFirstChildOfClass` à `GetService`, qui ajoute le service manquant à la place sauvée.
- **Patcher la `Source` d'un script qu'on ne possède pas** : remplace un extrait exact, `assert` s'il manque ou s'il apparaît deux fois, et pose un marqueur pour qu'une deuxième passe ne change rien. Un modèle modifié fait alors échouer le build au lieu de livrer un script non patché.

**Pièges de Lune quand il écrit une place** (relevés en 0.10, non revérifiés en 0.11 sauf mention). Chacun a déjà livré un build « réussi » mais faux : relis chaque propriété modifiée dans le fichier produit.

- `Sound.SoundId` écrit est perdu quand la place stocke `AudioContent` : écris `sound.AudioContent = roblox.Content.fromUri("rbxassetid://<id>")`. Même logique pour les images avec `ImageContent`.
- Avant la 0.11.0, `CFrame.lookAt` (et `CFrame.new(pos, cible)`) de Lune donne de mauvaises rotations : passe en 0.11.0, qui le corrige, ou construis avec `CFrame.fromMatrix`. Celui du moteur est correct.
- `Attachment.Position` n'est pas sauvé : règle `Attachment.CFrame`. `ScreenGui.IgnoreGuiInset` non plus : utilise `ScreenInsets`. `ViewportFrame.CurrentCamera` écrit reste illisible : crée la caméra côté client.
- Une `WeldConstraint` écrite par Lune soude les pièces au centre de la pièce principale : prends une `Weld` avec `C0` et `C1` explicites, ou un `Motor6D`.
- Les attributs numériques au-delà de 2^31 sont tronqués : stocke-les en chaîne.
- Des MeshParts créées par Lune peuvent n'avoir aucune donnée de rendu : clone une MeshPart existante. Lune ne sait pas écrire de voxels de terrain. Avant la 0.10.5, chaque accès à une Instance renvoyait un nouvel objet : `==` marchait, une table indexée par Instance non. Le changelog de la 0.10.5 annonce des références d'Instance stables ; un outil qui doit aussi tourner sur une version plus ancienne indexe par un identifiant stable.
- Le hash du fichier change à chaque build : compare des dumps d'instances, pas des sommes. La fidélité dépend de la base de réflexion embarquée : une classe plus récente que l'outil peut être mal écrite ou omise. Mets Lune à jour.

## 9. Tests

Monte l'échelle seulement aussi haut que nécessaire, et dis où tu t'es arrêté : statique (`npx rbxtsc`, ou luau-lsp + Selene + StyLua) ; logique pure (Lune ou la CLI `luau`) ; place construite vérifiée par Lune (section 8) ; moteur réel par Open Cloud Luau Execution + Jest Roblox (open-cloud.md) ; playtest par le MCP (studio-mcp.md), puis playtest humain pour le ressenti.

- **Logique pure** : sors maths, cooldowns, inventaire, machines à états et migrations de données dans des modules sans `game`. Passe l'horloge, le hasard (graine fixe) et les services en paramètres au lieu de remplacer des globaux. Un mini-runner suffit pour Lune :

```luau
-- tests/run.luau, lancé par : lune run tests/run
-- teste un module pur Damage.compute(base, armure), réduction plafonnée à 75 %
local process = require("@lune/process")
local Damage = require("../src/shared/Damage")

local failures = 0
local function check(name: string, ok: boolean)
	print((if ok then "PASS " else "FAIL ") .. name)
	failures += if ok then 0 else 1
end

check("50 d'armure divise par deux", Damage.compute(100, 50) == 50)
check("réduction plafonnée à 75 %", Damage.compute(100, 200) == 25)
check("jamais de dégâts négatifs", Damage.compute(-5, 0) == 0)

process.exit(if failures == 0 then 0 else 1)
```

- **roblox-ts** : la sortie compilée dépend du runtime `include/`, donc teste-la dans le moteur (Luau Execution, ou un script de tests lancé en playtest). Le portage npm de Jest Roblox (`@rbxts/jest` + `@rbxts/jest-globals`) est dans paquets.md.
- **TestEZ** est archivé : garde-le où il est déjà, ne l'introduis pas. Ses specs `X.spec.luau` renvoient une fonction (`describe`, `it`, `expect(x).to.equal(y)`) ; un Script lance `TestEZ.TestBootstrap:run({ dossier })` et lève une erreur si `failureCount > 0`. Mets les specs en `--!nonstrict` : les globaux de TestEZ sont invisibles au vérificateur.
- **Jest Roblox** (Jest-Lua, 3.20, maintenu par Roblox) est le framework actuel. `describe`, `it` et `expect` s'importent depuis `JestGlobals` et s'appellent au niveau du fichier, sans fonction renvoyée. En CI, il tourne dans une place de test via Luau Execution (open-cloud.md).
- **lest** (v0.6.1, 2026-10-02 ; projet créé en 2026-07) : lanceur optionnel à trois backends (VM Luau locale, tâche Open Cloud, Studio). Codes de sortie : 0 tout passe, 1 tests en échec, 2 erreur de l'outil. Rapports `--reporter junit` et `--reporter json` (sa doc CI, 2026-10-18). Son backend Studio tourne en mode édition, sans physique ni scripts de la place : il ne remplace pas un playtest pour les entrées et le rendu. Mentionne-le, ne l'impose pas.
- Règles : un nouveau test échoue sur l'ancien code et passe sur le nouveau (cite les deux résultats) ; un comportement par test ; aucun test ne dépend d'un autre ; **0 test trouvé = échec de configuration** ; un test instable se corrige (temps, hasard, état partagé) ou se supprime, jamais par retry automatique. Pour imposer une règle stricte à un vieux jeu, enregistre un état de référence au premier passage, puis n'échoue que sur les nouvelles violations.
- Ne simule pas ce qui ne se simule pas : budgets DataStore, analytics d'une expérience publiée, safe area d'un vrai téléphone, physique et propriété réseau, animations uploadées, permissions d'assets. Un mock d'Instance teste le mock, pas le jeu. Le nouveau registre de `TestService` (`RegisterSuite`, `RunTestsAsync`) n'est pas activé (Studio 0.742) : ne bâtis rien dessus.

## 10. CI type

Ordre : outils épinglés, dépendances, lint, format, types, tests, build. Chaque étape bloque. Ne copie un workflow dans `.github/workflows/` que si l'utilisateur le demande.

roblox-ts, la pile par défaut :

```yaml
name: ci
on: [push, pull_request]
permissions: { contents: read }
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7            # majeure vue en octobre 2026
      - uses: CompeyDev/setup-rokit@v0.2.1   # installe les outils du manifeste
      - run: npm ci
      - run: npx eslint src
      - run: npx rbxtsc
      - run: rojo build default.project.json --output build.rbxl
```

- Projet Luau + Wally : mêmes en-têtes, puis `wally install` (étape conditionnée par `if: hashFiles('wally.toml') != ''`), `selene src`, `stylua --check src`, `rojo sourcemap …`, `luau-lsp analyze …` (commandes de la section 7), `lune run tests/run`, `rojo build …`.
- Les versions d'outils viennent du manifeste commité. Épingle aussi les actions, idéalement par SHA de commit, et mets-les à jour par Dependabot. Les définitions luau-lsp sont commitées dans `types/` : l'analyse ne change pas quand l'API en ligne bouge.
- Conditionne sur des résultats lus (nombre de tests, rapport JUnit ou JSON), pas sur le seul code de sortie : un outil peut sortir 0 sans avoir rien testé.
- Tests moteur : construis une place de test, publie-la en version `Saved` sur une place de test dédiée (dans une expérience séparée si les tests écrivent des données), lance les tests par Luau Execution, lis les compteurs (open-cloud.md). Mets ces jobs dans un groupe de concurrence (`concurrency: roblox-tests`) : les tâches simultanées sont limitées. Lance-les aussi chaque semaine (`schedule`) : Roblox livre chaque semaine, et un vert d'il y a un mois ne dit rien d'aujourd'hui. La clé API vient du coffre de secrets de la CI, avec la portée la plus étroite, et n'apparaît jamais dans les logs.

## 11. Git, fichiers de place, LFS

`.gitignore` d'un projet roblox-ts ou Luau :

```gitignore
node_modules/
out/
include/
Packages/
ServerPackages/
DevPackages/
roblox_packages/
sourcemap.json
build/
*.rbxl
*.rbxlx
*.rbxl.lock
*.rbxlx.lock
.env
```

- N'écris pas `*.lock` : tu ignorerais `wally.lock` et `pesde.lock`, qui doivent être commités. Les verrous de Studio sont les `*.rbxl.lock` et `*.rbxlx.lock` créés à côté d'une place ouverte.
- Commite : `src/`, fichiers projet, `package.json` et `package-lock.json`, `tsconfig.json`, `wally.toml` et `wally.lock`, `rokit.toml` ou `aftman.toml`, configs (Selene, StyLua, ESLint, `.luaurc`), `.gitattributes`, CI, définitions vendorisées, modèles `.rbxmx` qui servent de sources Rojo.
- Ne commite ni clé API, ni cookie, ni `.env` : fournis un `.env.example` vide. Vérifie `git status` après chaque installation : rien de généré ne doit apparaître.
- Sous Windows, ajoute un `.gitattributes` avec `* text=auto`, `*.luau text eol=lf`, `*.lua text eol=lf`, `*.rbxl binary` et `*.rbxm binary` : StyLua et les diffs restent propres.
- **Les places ne fusionnent pas.** Un `.rbxl` est binaire. Un `.rbxlx` est du XML, mais ses référents et son ordre changent à chaque sauvegarde. Garde le code dans les fichiers et les places hors de git.
- Si l'équipe versionne une place de base (décor, terrain), mets-la sous Git LFS **verrouillable** et retire-la de l'ignore (`!places/base.rbxl`). LFS stocke sans fusionner : le verrou impose un seul écrivain.

```bash
git lfs install
git lfs track "*.rbxl" --lockable
git lfs lock places/base.rbxl
git lfs locks
```

- Boucle quotidienne : `git pull` avant d'ouvrir Studio ; `npm ci`, `rokit install` ou `wally install` si un manifeste a changé ; `git lfs locks` avant d'éditer une place versionnée. Les sauvegardes automatiques de Studio ne remplacent pas git : commite souvent.
- Télécharger la place publiée par Open Cloud : l'API Asset Delivery (`GET apis.roblox.com/asset-delivery-api/v1/assetId/{assetId}`, portée `legacy-asset:manage`) dit servir tous les types d'assets, mais une équipe a échoué avec sa clé (à vérifier pour une place). Pour récupérer des éditions faites dans Studio, l'utilisateur sauve la place dans un fichier (File → Save to File As), puis `rojo syncback` si besoin. La production n'est publiée que par un seul chemin, de préférence la CI, et jamais par-dessus des éditions Studio non sauvées.

## 12. Boucle de dev sur Windows

Mise en place, une fois, chaque installation avec l'accord de l'utilisateur : Node.js LTS, git et Studio ; projet créé par la commande de la section 4, scope `@rbxts-js` ajouté ; Rojo épinglé dans `aftman.toml` ou `rokit.toml` à la racine (section 5) ; `rojo plugin install`, puis redémarrage de Studio. Ensuite, à chaque séance :

1. Terminal 1 : `npx rbxtsc -w`. Attends zéro erreur.
2. Terminal 2 : `rojo serve` (port 34872).
3. Studio : ouvre la place, onglet Plugins, Rojo, Connect.
4. Play. Lis la fenêtre Output : erreurs rouges, avertissements `Infinite yield`.

Boucle de l'agent : écrire dans `src/`, compiler jusqu'à zéro erreur, laisser Rojo pousser, puis l'utilisateur fait Play et envoie la console Output (ou l'agent la lit par le MCP, studio-mcp.md), et on corrige. Quand « rien ne s'affiche », demande d'abord la console Output.

- Lance les watchers en tâche de fond et relis leur sortie : un `rbxtsc -w` en erreur laisse l'ancien `out/` en place, et Studio exécute du vieux code.
- Vérifie ce que Rojo sert, sans Studio. Sous Windows PowerShell 5.1, `curl` est un alias d'`Invoke-WebRequest` : appelle `curl.exe`. Si tu gardes `Invoke-WebRequest`, pose `$ProgressPreference = 'SilentlyContinue'` : sa barre de progression le rend très lent.

```bash
curl.exe http://localhost:34872/api/rojo                    # session et id de la racine
curl.exe http://localhost:34872/api/read/<rootInstanceId>   # arbre servi
```

- Si PowerShell refuse `npx.ps1` (politique d'exécution), lance `npx.cmd`. Changer la politique d'exécution est une décision de l'utilisateur.
- Variables d'environnement : `$env:RBX_DEBUG = "1"` vaut pour la session ; `setx` n'agit que sur les nouveaux terminaux et écrit la valeur en clair dans le registre. La syntaxe `VAR=1 commande` n'existe pas en PowerShell.
- Dans `default.project.json`, écris les `$path` avec des `/` : un `\` doit être doublé en JSON. Port 34872 occupé : `rojo serve --port 34873`, puis change le port dans le plugin.
- Dossier des plugins : `%LOCALAPPDATA%\Roblox\Plugins`. Un plugin local en `.lua`, `.rbxm` ou `.rbxmx` se charge ; un `.luau` ne s'est pas chargé (mesuré en août 2026).
- Les noms d'instances deviennent des noms de fichiers (syncback, Argon) : pas de `< > : " / \ | ? *`, pas de point ni d'espace final, pas de `CON`, `PRN`, `AUX`, `NUL`, `COM1` à `COM9`, `LPT1` à `LPT9`. Choisis des noms de fichiers légaux dès le départ.
- Pour partager un dossier entre deux projets sans droits admin, une jonction suffit : `New-Item -ItemType Junction -Path <lien> -Target <dossier>`. Un lien symbolique demande l'admin ou le mode développeur. Rien ne prouve encore que `rbxtsc -w` et `rojo serve` voient les changements faits dans la cible : teste-le une fois.

## 13. Dépannage

| Symptôme | Cause probable | Correctif |
|---|---|---|
| Aucune UI React ; console : `Infinite yield possible on '…rbxts_include.node_modules:WaitForChild("@rbxts-js")'` | scope `@rbxts-js` non mappé | ajoute `"@rbxts-js": { "$path": "node_modules/@rbxts-js" }` (section 4) |
| Rien ne s'affiche, aucune erreur | `rbxtsc -w` en erreur, Rojo déconnecté, mauvaise place, script dans un conteneur qui ne tourne pas | lis la sortie du watcher ; vérifie la connexion du plugin ; `curl.exe …/api/rojo` ; fichier `.client.ts` sous `src/client` |
| `Infinite yield` sur `TS` ou sur un module | `$path` faux, `out/` pas compilé, nom ou casse différents | compare `outDir` et les `$path` ; `npx rbxtsc` |
| `rbxtsc` : « Could not find Rojo data » | le fichier n'est couvert par aucun `$path` | range-le sous `src/server`, `src/client` ou `src/shared`, ou ajoute un mapping |
| `rojo` introuvable, « no aftman.toml files list this tool », « Failed to find tool … » | outil absent du manifeste, ou commande lancée hors du projet | ajoute l'outil ; lance depuis la racine ; nouveau terminal après `self-install` |
| Pas d'onglet Rojo dans Studio | plugin absent, ou Studio pas redémarré | `rojo plugin install`, redémarre Studio, regarde `%LOCALAPPDATA%\Roblox\Plugins` |
| Connect échoue | serve arrêté, mauvais port, majeures différentes, `servePlaceIds` ou `blockedPlaceIds`, hôte refusé | relance `rojo serve` ; aligne CLI et plugin ; vérifie l'id de la place |
| Édition faite dans Studio perdue | Rojo fait foi pour les scripts gérés | édite le fichier : c'est le comportement attendu |
| Instances créées dans Studio disparues après Connect | `$ignoreUnknownInstances` à `false` sous un `$path` | range-les hors des dossiers gérés, ou passe la clé à `true` |
| `HttpEnabled`, `MeshId` ou terrain ignorés | limite de la synchro live | `rojo build` puis ouvre la place, ou règle-les dans Studio |
| Mauvaise classe de script | suffixe, `emitLegacyScripts`, ou convention Script Sync | tableau des suffixes (section 1) |
| `require` échoue sur un « module » | il est nommé `.server.luau`, donc c'est un Script | renomme-le en `.luau` |
| « Cannot use identifier reserved for compiler internal usage » | nom réservé (`next`) | renomme la variable |
| Un membre d'Enum refusé alors que le type semble juste | paramètre par défaut qui fige un membre | annote le paramètre (`Enum.Material`) |
| `Cannot find module '@rbxts/x'` | paquet absent, `typeRoots` ou mapping incomplet | `npm i @rbxts/x` ; vérifie `typeRoots` et le scope dans Rojo |
| API récente inconnue des types | `@rbxts/types` ou définitions luau-lsp en retard | mets-les à jour ; vérifie l'API (pieges.md) |
| luau-lsp : « Unknown global » partout | définitions absentes (`--definitions:@roblox=…`) | passe les définitions (section 7) |
| luau-lsp ne trouve pas un module ou un paquet | sourcemap périmé, ou généré avant `wally install` | régénère le sourcemap |
| Erreurs de type dans un paquet correct | relais Wally sans types | `wally-package-types` |
| Un `.luau` revient après ton édition | sortie de `rbxtsc` ou de Darklua | modifie la source |

Le diagnostic général « ça ne marche pas » est dans pieges.md.

## 14. Pièges

- Éditer le même script par Rojo et par le MCP, ou lancer Script Sync et `rojo serve` sur le même arbre.
- Oublier `@rbxts-js` dans le projet : aucune UI, un seul indice dans la console.
- Éditer `out/` ou `include/`, ou y faire un syncback.
- Confondre les suffixes de Rojo et de Script Sync, surtout `.client.luau`.
- Nommer un module `.server.luau`, ou laisser un `.luau` sans suffixe dans StarterPlayerScripts.
- Écrire `[rules]` au lieu de `[lints]` dans `selene.toml`.
- Lancer luau-lsp sans définitions, ou avec un sourcemap périmé.
- Mettre `*.lock` dans `.gitignore`, ou ignorer `wally.lock`.
- Recopier des versions périmées (Rojo 7.4, StyLua 0.20, Lune 0.8) ou des options inventées (`rbxtsc --noEmit`, `rojo serve --no-watch`, `rojo push`, `rojo build --base`, `wally install --locked` en 0.3.2).
- Installer un outil par `curl | bash`, `irm | iex` ou `--no-trust-check`, ou sans l'accord de l'utilisateur.
- Croire qu'un build, une vérification de types verte ou une écriture Lune non relue prouvent que ça marche.
- Exposer `rojo serve` hors de la machine, ou servir une place publiée sans `servePlaceIds`.

## 15. Checklist

- [ ] La route de synchro est identifiée, et un seul outil écrit chaque script.
- [ ] Rojo est épinglé (7.7.1) dans `rokit.toml` ou `aftman.toml`, et les outils tournent depuis la racine.
- [ ] Le plugin Rojo a la même version majeure que la CLI.
- [ ] En roblox-ts, chaque scope npm utilisé (`@rbxts`, `@rbxts-js`…) est mappé sous `rbxts_include/node_modules`.
- [ ] `npx rbxtsc` (ou luau-lsp, Selene et StyLua) passe sans erreur.
- [ ] `rojo build` réussit, et sa sortie est citée.
- [ ] Rien n'a été écrit dans `out/`, `include/` ni `Packages/`.
- [ ] Les lockfiles sont commités ; `.gitignore` couvre les sorties générées et les places.
- [ ] Les tests couvrent la logique pure ; le code qui touche au moteur a tourné une fois dans Studio ou par Luau Execution.
- [ ] La console Output a été relue après Play.
- [ ] Aucun secret dans le dépôt, les logs ou la place.
- [ ] Installation, syncback et publication ont eu l'accord explicite de l'utilisateur.
