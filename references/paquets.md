# Paquets, outils et plugins de l'écosystème Roblox

Vérifié le 2026-10-18. Chaque identifiant ci-dessous a été contrôlé en ligne ce jour-là : registre npm (`npm view`), API du registre Wally (`api.wally.run`), dépôt GitHub, ou page officielle. « (à vérifier) » signale un détail non confirmé : ne le présente pas comme un fait.

Versions : celles du jour de vérification. Revérifie avant d'installer, elles bougent vite.

## Comment choisir et installer

1. **Le projet a déjà sa pile ? Garde-la.** Lis `wally.toml`, `pesde.toml`, `package.json`, `rokit.toml` ou `aftman.toml`. N'ajoute pas un deuxième gestionnaire de paquets, un deuxième réseau ou un deuxième système de nettoyage à côté du premier.
2. **Vérifie qu'un paquet existe avant de l'écrire dans un fichier.** Un nom plausible peut ne pas exister, ou appartenir à un inconnu :
   - npm : `npm view @rbxts/<nom> name version repository.url time.modified` (E404 = n'existe pas).
   - Wally : `curl -s "https://api.wally.run/v1/package-metadata/<scope>/<nom>"` (message « could not open package » = n'existe pas). Recherche : `curl -s "https://api.wally.run/v1/package-search?query=<mot>"`.
   - pesde : `curl -s "https://registry.pesde.dev/v1/packages/<scope>%2F<nom>"`.
   - GitHub : `git ls-remote --heads https://github.com/<owner>/<repo>` (existence), puis la page du dépôt pour voir s'il est archivé, ou `gh repo view <owner>/<repo> --json isArchived,pushedAt` si `gh` est installé.
   - Creator Store : `curl -s "https://economy.roblox.com/v2/assets/<id>/details"` donne le nom, le créateur, le type (10 = modèle, 38 = plugin) et la date de mise à jour.
3. **Vérifie le propriétaire.** Sur Wally, la recherche « profilestore » renvoie plus de 40 copies publiées par des inconnus. Prends le scope que cite le dépôt officiel (son `wally.toml` ou son README), jamais le premier résultat.
4. **Vérifie qu'il est maintenu** : dépôt archivé, dernière publication, issues ouvertes sans réponse, avertissement dans le README.
5. **Demande l'accord de l'utilisateur avant d'installer** quoi que ce soit : paquet, outil, plugin Studio.
6. **Épingle la version.**
   - Wally : `Charm = "littensy/charm@0.11.1"` dans `wally.toml`, puis `wally install`. Wally lit `@0.11.1` comme « compatible avec 0.11.1 » (plage `^`) ; c'est `wally.lock` qui fige la version exacte : commite-le. Pour figer dans le manifeste lui-même, écris `@=0.11.1`. Un paquet de « realm » serveur (ex. ProfileStore) va sous `[server-dependencies]`.
   - npm : `npm i -E @rbxts/charm@0.11.1`, puis commite `package-lock.json`.
   - Outils : version exacte dans `rokit.toml` (`rojo = "rojo-rbx/rojo@7.x.y"`).
7. **Lis la licence.** MIT, ISC, Apache-2.0 et MPL-2.0 permettent l'usage dans un jeu ; dans le code copié, garde la notice de licence et un commentaire de provenance (URL, version). GPL-3.0 impose la même licence à ce qui intègre le code : demande à l'utilisateur avant de l'utiliser dans un jeu. Sans licence, le code reste « tous droits réservés » : ne le recopie pas, insère plutôt le modèle publié par l'auteur.
8. **Méfie-toi du Creator Store.** Un modèle gratuit peut cacher une porte dérobée. Avant de l'insérer, ouvre chaque script et cherche `require(` suivi d'un nombre, `loadstring`, `getfenv`, `setfenv`, `InsertService`, des scripts cachés dans des Welds, des Attachments ou des attributs, et des appels HTTP inattendus. Préfère le code source sur GitHub ou Wally, ou la copie publiée par l'auteur lui-même. Procédure complète : securite.md, section 12.
9. **Typings roblox-ts.** Un `@rbxts/<nom>` n'est pas forcément écrit par l'auteur du module Luau. Regarde le champ `repository.url` de `npm view` et préfère les typings cités par le README du projet d'origine.

## Pile recommandée en bref

Point de départ pour un projet neuf. Le détail, les alternatives et les statuts sont dans les tableaux plus bas.

| Besoin | roblox-ts (pile de cet utilisateur) | Luau + Wally |
|---|---|---|
| Données joueur | ProfileStore via `@rbxts/profile-store`, ou `@rbxts/lapis` | `lm-loleris/profilestore` |
| Réseau | Zap avec `opt typescript = true`, ou `@flamework/networking`, ou `@rbxts/remo` | Zap (outil CLI via Rokit), ou `ffrostflame/bytenet` |
| Validation des remotes | `@rbxts/t`, ou les gardes générés par Flamework | `osyrisrblx/t` |
| Structure du code | `@flamework/core` pour un gros projet, modules simples sinon | modules simples + `sleitnick/component` |
| État global | `@rbxts/charm` + `@rbxts/charm-sync` + `@rbxts/react-charm` | `littensy/charm` + `littensy/charm-sync` |
| UI | `@rbxts/react` + `@rbxts/react-roblox` | `jsdotlua/react` + `jsdotlua/react-roblox`, ou `elttob/fusion` |
| Animation d'UI | `@rbxts/ripple` + `@rbxts/pretty-react-hooks` | `littensy/ripple` |
| Stories | plugin UI Labs + `@rbxts/ui-labs` | plugin UI Labs + `pepeeltoro41/ui-labs` |
| Nettoyage | `@rbxts/janitor` | `sleitnick/trove` |
| Signaux | `@rbxts/sleitnick-signal` ou `@rbxts/lemon-signal` | `sleitnick/signal` ou `stravant/goodsignal` |
| Promesses | intégré à roblox-ts | `evaera/promise` |
| Tables | `@rbxts/sift` | `csqrl/sift` |
| Tests | `@rbxts/jest` + `@rbxts/jest-globals` | `roblox/jest` + `roblox/jest-globals` |
| Commandes de dev | `@rbxts/cmdr` | `evaera/cmdr` |
| Topbar | `@rbxts/topbar-plus` | `1foreverhd/topbarplus` |
| Dégradés animés | ezVisualz (pas de typings : `.d.ts` maison), ou ripple sur un `UIGradient` | `arxkdev/ezvisualz` |
| Outils | `roblox-ts`, Rojo, Rokit | Rokit, Rojo, Wally, Selene, StyLua, luau-lsp |

## Exemples de manifestes

Identifiants vérifiés le 2026-10-18. Demande l'accord de l'utilisateur avant de lancer une installation.

`wally.toml` d'un projet Luau :

```toml
[package]
name = "moi/mon-jeu"
version = "0.1.0"
registry = "https://github.com/UpliftGames/wally-index"
realm = "shared"

[dependencies]
React = "jsdotlua/react@17.2.1"
ReactRoblox = "jsdotlua/react-roblox@17.2.1"
Charm = "littensy/charm@0.11.1"
Ripple = "littensy/ripple@0.10.2"
Trove = "sleitnick/trove@1.8.0"
Signal = "sleitnick/signal@2.0.3"
Promise = "evaera/promise@4.0.0"
t = "osyrisrblx/t@3.1.1"

[server-dependencies]
ProfileStore = "lm-loleris/profilestore@1.0.3"

[dev-dependencies]
Jest = "roblox/jest@3.20.1"
JestGlobals = "roblox/jest-globals@3.20.1"
```

Projet roblox-ts (à lancer à la racine, où se trouve `package.json`) :

```bash
npm i -E @rbxts/react@17.3.13-ts.1 @rbxts/react-roblox@17.3.13-ts.1 @rbxts/charm@0.11.1 @rbxts/ripple@0.10.2 @rbxts/janitor@1.18.3-ts.0 @rbxts/t@3.2.1 @rbxts/services@1.6.0
npm i -E -D @rbxts/jest@3.16.0-ts.2 @rbxts/jest-globals@3.16.0-ts.2
```

Avec `@rbxts/react`, mappe aussi `node_modules/@rbxts-js` dans `default.project.json`, sinon rien ne s'affiche (voir `ui-react.md`).

## Données et sauvegarde

| Nom | À quoi ça sert | Installation | Statut | Licence | Conseil |
|---|---|---|---|---|---|
| ProfileStore (loleris, MadStudio) | Profils joueurs : verrou de session, autosave, `Reconcile`, messages hors ligne | Wally `lm-loleris/profilestore@1.0.3` (scope tiré du `wally.toml` du dépôt officiel `MadStudioRoblox/ProfileStore`). Creator Store : modèle 109379033046155 (loleris). npm `@rbxts/profile-store` 1.0.3 (typings communautaires, dépôt Quamatic) | actif (dépôt mis à jour en 2025-07) | Apache-2.0 | **Choix par défaut** pour les données joueur. Paquet de realm serveur : `[server-dependencies]` |
| ProfileService | Ancêtre de ProfileStore, même auteur | GitHub `MadStudioRoblox/ProfileService` (copie du module). Pas de paquet Wally officiel : `madstudioroblox/profileservice` et `loleris/profileservice` n'existent pas. npm `@rbxts/profileservice` 1.4.2 (2023) | legacy : son README dit « FOR NEW PROJECTS - USE ProfileStore » et « no longer supported » | Apache-2.0 | À éviter dans un nouveau projet. Migration : `LoadProfileAsync` → `StartSessionAsync`, `Release` → `EndSession` |
| DataStore2 (Kampfkarren) | Ancien wrapper de DataStore avec cache et sauvegardes par versions | GitHub `Kampfkarren/Roblox` (dossier DataStore2). Pas de paquet Wally `kampfkarren/datastore2`. npm `@rbxts/datastore2` 1.4.0-ts.0 (2022) | legacy (dernier push 2023-06) | licence maison (fichier `LICENSE` du dépôt) : usage et modification permis ; si tu modifies le code dans un jeu publié, publie tes changements (ex. fork GitHub) | À éviter. Pas de verrou de session |
| Lapis (nezuo) | DataStore avec verrou de session, réessais, migrations et validation de schéma | Wally `nezuo/lapis@0.3.4`. npm `@rbxts/lapis` 0.3.8 (publié par sasial-dev) | stable (push 2025-02) | MIT | Alternative sérieuse à ProfileStore, surtout pour des migrations écrites |
| DocumentService (anthony0br) | DataStore « document » typé strict, verrou de session, migrations | Wally `anthony0br/documentservice@1.2.2`. GitHub `anthony0br/DocumentService` (release v1.2.2). npm `@rbxts/document-service` 1.2.2-ts.1 (dépôt christopher-buss) | actif (push 2026-09) | MIT | Alternative pour du Luau très typé |
| Suphi's DataStore Module (5uphi) | Wrapper de DataStore : verrou de session, autosave, `BindToClose`, compression | Creator Store : modèle 11671168253 « [V1] Suphis DataStore Module » (5uphi, mise à jour 2026-01) ; une « [V2 BETA] » existe (92661305353100). Fil DevForum 2425597. npm `@rbxts/suphi-datastore` 1.0.0 (typings tiers, 2023) | maintenu | texte permissif de type 0BSD (dans le fil) | Alternative légère. Insère le module ; ne fais pas `require(11671168253)` |

## Réseau et sérialisation

Un seul transport par projet. Un schéma typé garantit la forme d'un paquet, pas son sens : valide quand même côté serveur. Un simple wrapper typé (Comm ou TypedRemote de RbxUtil) ne vérifie rien à l'exécution par défaut.

Un générateur (Zap, Blink) se justifie avec beaucoup de remotes, de gros paquets ou un contrat partagé par plusieurs personnes ; pour cinq remotes, des `RemoteEvent` bruts et validés suffisent. Épingle sa version. À chaque changement de schéma, régénère serveur et client ensemble, et vérifie-le en CI. Ne modifie jamais le code généré. Contrôle toi-même la taille d'un événement non fiable (1 000 octets au plus, performance.md).

| Nom | À quoi ça sert | Installation | Statut | Licence | Conseil |
|---|---|---|---|---|---|
| Zap (red-blox) | Générateur de code réseau : on décrit les événements dans un fichier `.zap`, il produit du Luau sérialisé en buffers et des définitions TypeScript | Outil CLI : `rokit add red-blox/zap`. Rien à installer au runtime. Pas de paquet Wally ni npm `@rbxts/zap` | actif (push 2026-06) | MIT | **À utiliser** pour un réseau rapide et typé, Luau ou roblox-ts |
| Blink (1Axen) | Générateur de code réseau, même idée que Zap | Outil CLI : `rokit add 1Axen/blink`. Pas de npm `@rbxts/blink` | actif (push 2026-09) | MIT | Alternative à Zap ; n'en prends qu'un |
| ByteNet (ffrostfall) | Réseau sérialisé en buffers, défini en Luau (sans fichier IDL) | Wally `ffrostflame/bytenet@0.4.6` (ancien nom de l'auteur ; dépôt GitHub `ffrostfall/ByteNet`). npm `@rbxts/bytenet` 0.4.3-tsprev4 (2024-03) | peu actif (push 2025-08) | MIT | Alternative à Zap si on refuse une étape de génération |
| Squash (Data-Oriented House) | Sérialisation compacte des types Luau et Roblox dans des buffers, pour réduire la bande passante | Wally `data-oriented-house/squash@5.0.0` (scope cité par le README du dépôt `Data-Oriented-House/Squash`). npm `@rbxts/squash` 3.0.0-ts.0 (typings HylianBasement, en retard) | actif (push 2026-01) | MIT | Pour des données fréquentes et remplaçables (performance.md). Ce n'est pas un transport : il s'utilise avec des remotes bruts |
| Red (red-blox) | Ancienne bibliothèque réseau de l'équipe de Zap | Wally `red-blox/red@2.3.2`. Attention : npm `@rbxts/red` 1.0.2 vient d'un autre dépôt (`synpixel/Red`, 2023) | archivé (2024-02) | MIT | À éviter ; prendre Zap |
| Warp (imezx) | Réseau rapide avec buffers et limitation de débit | Wally `imezx/warp@1.0.14` (pré-version 1.1.0-pre8). GitHub `imezx/Warp`. Pas de npm `@rbxts/warp` | actif (push 2026-10) | MIT | Possible en Luau ; pas de typings npm |
| BridgeNet2 (ffrostfall) | Regroupe les envois réseau par frame | Wally `ffrostflame/bridgenet2@1.0.0` (le scope `ffrostfall/bridgenet2` n'a qu'une 2.0.0-alpha.1). npm `@rbxts/bridgenet` 2.0.0-rc4 est la v1 (2022) | peu actif (push 2025-05) | MIT | À éviter dans un nouveau projet |
| Packet (5uphi) | Réseau en buffers : regroupement par frame, sérialisation, protection anti-flood, fonctions distantes | Creator Store : modèle 104116977416770 « Packet » (5uphi, v1.7, mise à jour 2025-06). Fil DevForum 3573907. Pas de npm `@rbxts/packet` ni de Wally officiel | maintenu | texte permissif de type 0BSD (dans le fil) | Alternative en Luau. Un commentateur du fil a signalé des failles (reprise de coroutines, contrôle de taille contournable) ; correction non confirmée : teste avant de t'y fier |
| RbxNet `@rbxts/net` (roblox-aurora) | Remotes typés avec middleware | npm `@rbxts/net` 3.0.10. Wally `vorlias/net@3.0.7` | peu actif (push 2025-01) | MIT | Correct sur un projet existant ; pour du neuf, Zap ou Flamework Networking |
| Remo (littensy) | Remotes typés, validateurs `t` | npm `@rbxts/remo` 1.5.3. Wally `littensy/remo@1.5.3` | actif (publié 2025-12) | MIT | Bon choix léger en roblox-ts |
| Replica (MadStudio) | Réplication d'état serveur → client, par abonnement de joueurs | GitHub `MadStudioRoblox/Replica` (pas de paquet Wally officiel trouvé). Ancêtre : ReplicaService (Creator Store 6015318619, loleris ; npm `@rbxts/replicaservice` 1.1.0-ts.2, typings Mixu78). Piège : npm `@rbxts/replica` (2022) est une autre bibliothèque | actif (push 2025-08) | Apache-2.0 | Souvent couplé à ProfileStore. En roblox-ts, Charm Sync fait un travail proche |

## Validation des entrées

| Nom | À quoi ça sert | Installation | Statut | Licence | Conseil |
|---|---|---|---|---|---|
| t (osyrisrblx) | Vérifie à l'exécution le type des arguments reçus par un remote | Wally `osyrisrblx/t@3.1.1`. npm `@rbxts/t` 3.2.1 | stable (npm publié 2025-02) | MIT (Wally), ISC (npm) | **À utiliser** dans chaque handler de remote. Ne remplace pas les contrôles de plage, NaN et permission |

## Frameworks et ECS

| Nom | À quoi ça sert | Installation | Statut | Licence | Conseil |
|---|---|---|---|---|---|
| Knit (Sleitnick) | Services serveur et contrôleurs client | Wally `sleitnick/knit@1.7.0`. npm `@rbxts/knit` 0.0.21-alpha.0 (2022) | archivé (GitHub archivé, dernier push 2024-07) | MIT | À éviter dans un nouveau projet ; garde-le seulement là où il est déjà |
| Flamework (rbxts-flamework) | Framework roblox-ts : injection de dépendances, services et contrôleurs, composants liés aux tags, réseau typé avec gardes de types générés | npm `@flamework/core`, `@flamework/networking`, `@flamework/components` 1.3.2 + transformer `rbxts-transformer-flamework` 1.3.2 (déclaré dans `compilerOptions.plugins` du tsconfig). Pas de version Luau | actif (publié 2025-09) | ISC (paquets), MIT (transformer) | **Choix par défaut** pour un gros projet roblox-ts. Inutile pour un petit jeu |
| Nevermore (Quenty) | Énorme ensemble de modules Luau (ServiceBag, Maid, Binder, Rx…) | npm `@quenty/<module>`, par exemple `@quenty/loader` 10.11.3, `@quenty/servicebag` 11.21.2 ; CLI `@quenty/nevermore-cli` 4.47.0. C'est du Luau distribué par npm, pas des typings | actif (push 2026-10) | MIT | Puissant mais lourd. À prendre en bloc ou pas du tout |
| Component (Sleitnick) | Lie une classe Luau aux instances d'un tag CollectionService | Wally `sleitnick/component@2.4.8` (dépôt RbxUtil). Pas de typings : npm `@rbxts/component` 1.2.0 est une autre bibliothèque (cmumme) | actif (RbxUtil) | MIT | Utile sans Knit. En roblox-ts, `@flamework/components` |
| Matter (matter-ecs) | ECS avec débogueur et boucle de systèmes | Wally `matter-ecs/matter@0.8.5` (bêta 0.9.0-beta.0) ; l'ancien scope `evaera/matter` s'arrête à 0.7.0. npm `@rbxts/matter` 0.8.5 | peu actif (push 2024-12) | MIT | Possible ; jecs est plus rapide et plus actif |
| jecs (Ukendio) | ECS très rapide, relations entre entités | Wally `ukendio/jecs@0.11.0`. npm `@rbxts/jecs` 0.11.0 | actif (push 2026-10) | MIT | **À utiliser** si le jeu a vraiment besoin d'un ECS |
| Planck (yetanotherclown) | Ordonnanceur de systèmes pour ECS (souvent avec jecs) | Wally `yetanotherclown/planck@0.3.0-rc.2`. npm `@rbxts/planck` 1.0.0 | actif (push 2026-09) | MIT | Complément de jecs ; numérotation différente entre Wally et npm |
| ECR (centau) | ECS à « sparse sets », orienté performance | Wally `centau/ecr@0.9.0`. npm `@rbxts/ecr` 0.9.0-ts.0 (dépôt ReturnedTrue/ecr-ts) | actif (push 2026-09) | MIT | Alternative à jecs |

## État global

| Nom | À quoi ça sert | Installation | Statut | Licence | Conseil |
|---|---|---|---|---|---|
| Charm (littensy) | État réactif en atomes ; Charm Sync partage l'état serveur → client | Wally `littensy/charm@0.11.1`, `littensy/charm-sync@0.4.1`. npm `@rbxts/charm` 0.11.1, `@rbxts/charm-sync` 0.4.1, `@rbxts/react-charm` 0.4.1 | actif (publié 2026-10) | MIT | **Choix par défaut** avec React |
| Reflex (littensy) | Store façon Redux avec sélecteurs | Wally `littensy/reflex@4.3.1`. npm `@rbxts/reflex` 4.3.1 (2023-11), `@rbxts/react-reflex` 0.3.7 | peu actif (dernière version 2023) | MIT | Garde-le sur un projet existant ; pour du neuf, Charm, du même auteur |
| Rodux (Roblox) | Redux en Luau | Wally `roblox/rodux@4.0.0-rc.0`. npm `@rbxts/rodux` 3.0.0-ts.3 (2022) | maintenu par Roblox (push 2026-06), mais vieux modèle | Apache-2.0 | À éviter dans un nouveau projet : Charm est plus simple |

## UI déclarative

| Nom | À quoi ça sert | Installation | Statut | Licence | Conseil |
|---|---|---|---|---|---|
| React (react-lua, jsdotlua) | Portage de React 17 : composants, hooks | Wally `jsdotlua/react@17.2.1` + `jsdotlua/react-roblox@17.2.1`. npm `@rbxts/react` + `@rbxts/react-roblox` 17.3.13-ts.1 (dépôt littensy/rbxts-react) | actif (npm publié 2026-10) | MIT | **Choix par défaut** de cet utilisateur. En roblox-ts, mappe aussi le scope `@rbxts-js` dans Rojo (voir `ui-react.md`) |
| Fusion (dphfox) | UI réactive par états et scopes | Wally `elttob/fusion@0.3.0` (le scope `dphfox/fusion` n'existe pas). npm `@rbxts/fusion` 0.2.0 (2023, une version de retard) | actif (push 2026-02) | MIT | Très bien en Luau. En roblox-ts, préfère React : les typings sont en retard |
| Vide (centau) | UI réactive légère, sans DOM virtuel | Wally `centau/vide@0.4.1` (version du dépôt), pesde `centau/vide@0.4.0`. npm `@rbxts/vide` 0.6.1 (2026-02), publié par littensy avec sa propre numérotation | actif (push 2026-09) | MIT | Alternative légère à Fusion |
| Roact (Roblox) | Ancien React de Roblox | Wally `roblox/roact@1.4.4`. npm `@rbxts/roact` 3.0.1 | archivé (2023-12) | Apache-2.0 | À éviter ; remplacé par react-lua |
| roact-hooked | Hooks pour Roact | npm `@rbxts/roact-hooked` 2.6.2 (2023) | legacy | MIT | À éviter ; React a ses hooks |

## Animation et ressorts

| Nom | À quoi ça sert | Installation | Statut | Licence | Conseil |
|---|---|---|---|---|---|
| ripple (littensy) | Ressorts et tweens pour l'UI ; s'utilise très bien avec React | Wally `littensy/ripple@0.10.2`. npm `@rbxts/ripple` 0.10.2 | actif (push 2026-07) | MIT | **Choix par défaut** avec React |
| pretty-react-hooks (littensy) | Hooks utiles pour `@rbxts/react` (mouvement, événements, debounce…) | npm `@rbxts/pretty-react-hooks` 0.6.4 (2025-05). Pas de paquet Wally `littensy/pretty-react-hooks` | actif | MIT | À utiliser avec `@rbxts/react` ; lis la liste des hooks dans son README. La 0.6.4 dépend de `@rbxts/ripple` ^0.9 : avec ripple 0.10, vérifie les types de `useMotion` (ui-react.md) |
| spr (Fraktality) | Ressorts physiques sur les propriétés d'une Instance, en une ligne | GitHub `Fraktality/spr` (copie du module ; les `*/spr` de Wally sont de tiers). npm `@rbxts/spr` 2.1.1 (2023, osyris) | stable (push 2024-07) | MIT (GitHub), ISC (npm) | Très pratique hors React |
| Flipper (Reselim) | Ancienne bibliothèque de ressorts | Wally `reselim/flipper@2.0.0`. npm `@rbxts/flipper` 2.0.1 (2022) | peu maintenu (push 2021-11) | MIT | À éviter ; prendre ripple |
| react-spring Luau (chriscerie) | Portage de react-spring pour react-lua | Wally `chriscerie/react-spring@2.0.0` (dépôt GitHub `chriscerie/roact-spring`, release v2.0.0). npm `@rbxts/react-spring` 2.0.0-ts.1 (2025-10, howmanysmall) | peu actif (push 2024-07) | MIT | Alternative à ripple si on aime l'API react-spring |
| `@rbxts/pretty-roact-hooks` | Version Roact des hooks ci-dessus | npm 3.2.1 (2023) | legacy | MIT | À éviter |
| Otter (Roblox) | Bibliothèque d'animation par ressorts publiée par Roblox | Wally `jsdotlua/otter@1.2.1` (et `jsdotlua/react-otter@1.2.1`). GitHub `Roblox/otter`. Pas de npm `@rbxts/otter` | actif (push 2026-09) | MIT | Possible en Luau ; en roblox-ts, prendre ripple |

## Effets et composants d'UI

| Nom | À quoi ça sert | Installation | Statut | Licence | Conseil |
|---|---|---|---|---|---|
| ezVisualz, alias EasyVisuals (Arxk) | Effets animés sur `UIGradient` et `UIStroke` : arc-en-ciel, reflet (`Shine`), vague (`WaveStroke`), contours… 27 préréglages dans le dossier `Presets` du dépôt (2026-10-18). L'effet se met en pause quand la `ScreenGui`, `BillboardGui` ou `SurfaceGui` qui le contient passe à `Enabled = false` (pas sur `Visible`, d'après le code). API : `EasyVisuals.new(guiObject, presetName, speed, size)`, puis `:Pause()`, `:Resume()`, `:Destroy()` ; aussi `EasyVisuals.Gradient.new` et `EasyVisuals.Stroke.new` | Wally `arxkdev/ezvisualz@1.2.0`. Creator Store : modèle 14076358875 « ezVisualz » (Arxk, badge vérifié). GitHub `arxk/ezVisualz`. Fil DevForum 2470313. Pas de typings npm (`@rbxts/ezvisualz`, `@rbxts/easyvisuals`, `@rbxts/easy-visuals` : E404) | actif (push 2026-06) | MIT | **C'est le « ezvisuals » souvent cité.** Appelle `:Destroy()` quand tu retires un effet d'un objet qui reste en jeu ; le module ne se détruit seul que si l'objet quitte le DataModel (`AncestryChanged`). En roblox-ts : petit fichier `.d.ts` maison, ou effet refait avec ripple sur `UIGradient.Offset` |
| TopbarPlus v3 (ForeverHD) | Icônes, menus et listes déroulantes dans la barre du haut | Wally `1foreverhd/topbarplus@3.4.0`. Creator Store : modèle 92368439343389 (ForeverHD), méthode recommandée par sa doc. GitHub `1ForeverHD/TopbarPlus` (release v3.4.0, fichier `.rbxm`). npm `@rbxts/topbar-plus` 3.2.5 (typings communautaires, dépôt wad4444/topbarplus-types) | actif (push 2026-10) | MPL-2.0 + clause de crédit | **Standard** de la topbar. Licence : garde l'attribut de crédit, ou crédite TopbarPlus dans la description du jeu. Évite `@rbxts/topbarplus` 1.0.9 (autre publieur, sans dépôt) |
| Iris (SirMallard) | UI immédiate façon Dear ImGui : fenêtres de debug, réglages en direct | Wally `sirmallard/iris@2.5.1` (l'ancien scope `michael-48/iris` s'arrête à 2.3.1). npm `@rbxts/iris` 2.3.1-ts.1 (2024-11, en retard) | actif (push 2026-10) | MIT | Pour les outils de dev, pas pour l'UI des joueurs |
| Focus Navigation (Roblox) | Navigation manette et clavier avancée, avec intégration React (ReactFocusNavigation) | GitHub `Roblox/focus-navigation` seulement. Pas de paquet Wally ni npm `@rbxts/focus-navigation` | actif (push 2026-10) | MIT | Optionnel. La sélection de base du moteur suffit souvent |

## Stories et aperçus d'UI

| Nom | À quoi ça sert | Installation | Statut | Licence | Conseil |
|---|---|---|---|---|---|
| UI Labs (PepeElToro41) | Plugin façon Storybook : affiche un composant sans lancer le jeu, rechargement à chaud, contrôles | Plugin Creator Store 14293316215 (gratuit). Aides typées optionnelles : Wally `pepeeltoro41/ui-labs@2.4.2`, npm `@rbxts/ui-labs` 2.4.2. Le plugin marche sans elles | actif (push 2026-10) | plugin : dépôt GPL-3.0 ; paquet d'aides : MIT | **Choix par défaut** ; stories React/Roact, Fusion, Vide, Iris, ou génériques sans bibliothèque |
| Flipbook (flipbook-labs) | Plugin de stories | Plugin Creator Store 8517129161 (vocksel, gratuit). Chargeur de stories : Wally `flipbook-labs/storyteller@1.12.0`. Pas de npm `@rbxts/flipbook` | actif (push 2026-10) | MIT | Alternative à UI Labs |
| Hoarcekat (Kampfkarren) | Ancien plugin de stories, très simple | Plugin Creator Store 4621580428. GitHub `Kampfkarren/hoarcekat` (release 1.3.0). Pas de npm | maintenu (mise à jour 2025-11) | MPL-2.0 | Marche encore ; UI Labs fait plus |

## Nettoyage

| Nom | À quoi ça sert | Installation | Statut | Licence | Conseil |
|---|---|---|---|---|---|
| Trove (Sleitnick) | Range connexions, instances, threads et fonctions, puis nettoie tout d'un coup | Wally `sleitnick/trove@1.8.0` (dépôt `Sleitnick/RbxUtil`). npm `@rbxts/trove` 1.3.0 (2022, en retard) | actif (RbxUtil, push 2026-08) | MIT | **Choix par défaut** en Luau |
| Janitor (howmanysmall) | Même rôle, rapide, avec des index nommés | Wally `howmanysmall/janitor@1.18.3`. npm `@rbxts/janitor` 1.18.3-ts.0 | actif (push 2026-07) | MIT | **Choix par défaut** en roblox-ts (typings à jour) |
| Maid (Quenty, Nevermore) | Ancêtre de Trove et Janitor | npm `@quenty/maid` 3.12.2 (code Luau de Nevermore, pas des typings). npm `@rbxts/maid` 1.1.0 (2023, osyrisrblx) | actif dans Nevermore | MIT / ISC | Garde-le s'il est déjà là ; sinon Trove ou Janitor |

## Signaux et promesses

| Nom | À quoi ça sert | Installation | Statut | Licence | Conseil |
|---|---|---|---|---|---|
| GoodSignal (stravant) | Signal Luau rapide, mêmes méthodes qu'un `RBXScriptSignal` | Wally `stravant/goodsignal@0.3.1` (confirmé par le `wally.toml` du dépôt). npm `@rbxts/goodsignal` 1.0.0 (2023, publieur tiers) | stable (push 2025-08) | MIT | Bon choix léger |
| Signal (Sleitnick) | Signal typé, inspiré de GoodSignal | Wally `sleitnick/signal@2.0.3`. npm `@rbxts/sleitnick-signal` 1.0.8 | actif (RbxUtil) | MIT | Bon choix. Piège : npm `@rbxts/signal` est une autre bibliothèque (Validark, 2022) |
| LemonSignal (Data-Oriented House) | Signal très rapide | Wally `data-oriented-house/lemonsignal@2.0.0`. npm `@rbxts/lemon-signal` 1.2.0 (typings de pepeeltoro41) ; `@rbxts/lemonsignal` a été dépublié | actif (push 2026-07) | MIT | Si la vitesse compte vraiment |
| Promise (evaera) | Promesses Luau : `andThen`, `catch`, annulation, `Promise.all` | Wally `evaera/promise@4.0.0`. En roblox-ts, **Promise est intégré au compilateur** : rien à installer (`@rbxts/promise` n'existe pas) | stable (push 2024-08) | MIT | Pour enchaîner des appels qui yield (DataStore, HTTP, achats) |

## Utilitaires de tables

| Nom | À quoi ça sert | Installation | Statut | Licence | Conseil |
|---|---|---|---|---|---|
| Sift (cxmeel, ex-csqrl) | Fonctions immuables sur tableaux, dictionnaires et ensembles | Wally `csqrl/sift@0.0.11` (dépôt renommé `cxmeel/sift`). npm `@rbxts/sift` 0.0.11 | actif (push 2026-03) | MIT | **Choix par défaut** ; remplace Llama |
| TableUtil (Sleitnick) | Copie profonde, `Reconcile`, `Map`, `Filter`, `Reduce`… | Wally `sleitnick/table-util@1.2.1`. npm `@rbxts/tableutil` 2.0.0 = port de l'ancien TableUtil d'AeroGameFramework (2022) | actif (RbxUtil) | MIT (Wally), ISC (npm) | Bien en Luau. En roblox-ts, les méthodes de tableau du compilateur suffisent souvent |
| Llama (freddylist) | Ancien utilitaire immuable | Wally `freddylist/llama@1.1.1`. npm `@rbxts/llama` 1.1.1-ts.1 (2022) | archivé | MIT | À éviter ; prendre Sift |

## Combat, projectiles et zones

| Nom | À quoi ça sert | Installation | Statut | Licence | Conseil |
|---|---|---|---|---|---|
| FastCast: Redux (EtiTheSpirit) | Projectiles simulés par raycasts successifs : balles, flèches, gravité | Creator Store : modèle 4453855787 (EtiTheSpirit, dernière mise à jour 2021-01). Le dépôt GitHub de l'auteur n'existe plus. npm `@rbxts/fastcast` 13.2.0-ts.2 (2022, typings Mixu78) | peu maintenu (depuis 2021) | (à vérifier) | Encore très utilisé. Les copies `*/fastcast` de Wally sont de tiers : prends le modèle de l'auteur |
| RaycastHitboxV4 (TeamSwordphin) | Hitbox de mêlée par raycasts depuis des Attachments de l'arme | Creator Store : modèle 7034457254 (TeamSwordphin, mise à jour 2025-04). npm `@rbxts/raycast-hitbox` 4.0.1-ts.1 (typings Mixu78). Pas de paquet Wally officiel | stable | MIT (dépôt d'origine `Swordphin/raycastHitboxRbxl` ; `TeamSwordphin/raycastHitboxRbxl` est un fork sans licence) | Bon choix pour la mêlée classique |
| ShapecastHitbox (TeamSwordphin) | Hitbox par shapecasts, même auteur | Wally `teamswordphin/shapecasthitbox@0.2.9`. GitHub `TeamSwordphin/ShapecastHitbox`. npm `@rbxts/shapecasthitbox` 0.2.5 (dépôt OphidieDev, peu utilisé) | actif (push 2025-08) | MIT | À préférer pour une nouvelle mêlée |
| ClientCast (Pyseph) | Hitbox de mêlée par raycasts, calculée par un client | GitHub `PysephWasntAvailable/ClientCast` (copie du module). npm `@rbxts/clientcast` 1.8.5 (2022, typings tiers). Pas de paquet Wally : `pyseph/clientcast`, déclaré dans le `wally.toml` de la branche 2.0, n'est pas publié | peu actif (push 2023-09) | MIT | À revalider avant usage. Le serveur vérifie chaque coup que le client déclare (securite.md) |
| WCS (wad4444) | Framework de combat : compétences, effets d'état, movesets ; écrit en roblox-ts avec Flamework | npm `@rbxts/wcs` 2.5.0. Wally `cheetiedotpy/wcs@2.5.0` (nom tiré du `wally.toml` du dépôt `wad4444/WCS`) | actif (push 2025-11) | MIT | Pour un jeu de combat à compétences. Il tire Flamework et Charm ^0.10 : avec Charm 0.11 dans le projet, vérifie `npm ls @rbxts/charm` |
| ZonePlus v3 (ForeverHD) | Zones : qui entre, qui sort d'un volume | GitHub `1ForeverHD/ZonePlus` (release v3.2.0, `.rbxm`). Son `wally.toml` déclare `1foreverhd/zoneplus`, mais ce nom n'est pas publié sur Wally ; les `*/zoneplus` publiés sont de tiers. npm `@rbxts/zone-plus` 3.2.0 (typings Scyfren) | peu actif (push 2024-07) | MIT | Toujours une référence. Pour une zone simple, `workspace:GetPartBoundsInBox` suffit |
| Chickynoid (easy-games) | Personnage à autorité serveur avec prédiction client, contre la triche de mouvement | Wally `easy-games/chickynoid@0.1.5`. GitHub `easy-games/chickynoid`. Pas de npm | peu actif (push 2024-05) | MIT | Seulement pour un jeu compétitif : change toute l'architecture du personnage |

## PNJ et caméra

| Nom | À quoi ça sert | Installation | Statut | Licence | Conseil |
|---|---|---|---|---|---|
| SimplePath (GrayzcaIe) | Pathfinding simple pour PNJ, relance si bloqué | Creator Store : modèle 6744337775 (auteur affiché « GrayzcaIe », lien donné par le fil DevForum 1196762). GitHub `ahmicy/simplepath` (ancien compte grayzcale, release v2.2.1). npm `@rbxts/simplepath` 2.2.1-ts.2 (2023) ou `@rbxts/simple-path` 1.0.1 (2025), typings tiers | maintenu (push 2026-06) | MIT | Bon départ pour des PNJ qui suivent un joueur |
| BehaviorTree3 (Defaultio, tyridge77) | Arbres de comportement pour PNJ, avec un éditeur visuel | GitHub `Defaultio/BehaviorTree3` (code à jour malgré son nom, selon le fil DevForum « BTreesV5 »). Plugin éditeur : Creator Store 4702098000 « BTreesV5 » (tyridge77, mise à jour 2023-08). npm `@rbxts/behavior-tree` 0.1.1 (Scyfren, 2023), non testé avec l'éditeur selon son README | peu maintenu (push 2023-06) | GPL-3.0 (dépôt) | Pour une IA à plusieurs états. Licence GPL : demande à l'utilisateur avant de l'intégrer |
| CameraShaker (Sleitnick) | Secousses de caméra : explosion, impact, tremblement | GitHub `Sleitnick/RbxCameraShaker` (copie du module). Pas de paquet Wally officiel. npm `@rbxts/camera-shaker` 1.0.8 (typings osyrisrblx) | stable, inchangé (dernier commit 2019-12) | MIT | Fonctionne. Il ignore `GuiService.ReducedMotionEnabled` : teste-le toi-même avant de secouer la caméra |

## Commandes et administration

| Nom | À quoi ça sert | Installation | Statut | Licence | Conseil |
|---|---|---|---|---|---|
| Cmdr (evaera) | Console de commandes typée et extensible | Wally `evaera/cmdr@1.13.1`. GitHub `evaera/Cmdr`. npm `@rbxts/cmdr` 1.12.1 (2024-02, typings sasial-dev) | actif (push 2026-10) | MIT | **Choix par défaut** pour les commandes de dev. Les modèles « Cmdr » du Creator Store viennent de tiers |
| Adonis (Epix Incorporated) | Système d'admin complet pour modérer un jeu public | Creator Store : modèle 7510622625 « Adonis Admin » (groupe Epix Incorporated). GitHub `Epix-Incorporated/Adonis` | actif (push 2026-10) | MIT | Pour la modération. Configure les rangs ; garde-le à jour |
| HD Admin (ForeverHD) | Admin populaire avec interface | Creator Store : modèle 857927023 « HD Admin » (ForeverHD, mise à jour 2026-09) | actif | (à vérifier) | Alternative à Adonis. Un seul système d'admin par jeu |

## Tests

| Nom | À quoi ça sert | Installation | Statut | Licence | Conseil |
|---|---|---|---|---|---|
| Jest Roblox (Roblox) | Jest en Luau : `describe`, `it`, `expect`, mocks, snapshots | Wally `roblox/jest@3.20.1` + `roblox/jest-globals@3.20.1`. Creator Store : modèle 16031830738 « JestRoblox » (Roblox). Ancienne copie communautaire : `jsdotlua/jest@3.10.0`. npm `@rbxts/jest` + `@rbxts/jest-globals` 3.16.0-ts.2 (dépôt littensy/rbxts-jest) | actif (push 2026-10) | MIT | **Choix par défaut** |
| TestEZ (Roblox) | Ancien framework de tests BDD | Wally `roblox/testez@0.4.1`. npm `@rbxts/testez` 0.4.2-ts.0 (2022) | archivé | Apache-2.0 | À éviter dans du neuf ; prendre Jest |
| Lest (lest-luau) | Binaire de test : mêmes tests en logique pure, sous Lune ou dans une vraie place | GitHub `lest-luau/lest` (release v0.6.1). Pas de Wally ni npm | jeune (12 étoiles), actif (push 2026-10) | MIT | À essayer ; Jest reste le choix sûr |

## Outils en ligne de commande

Installe-les par projet avec Rokit (`rokit add <owner>/<repo>`), puis `rokit install` sur une autre machine. Rojo, Rokit, Wally, Selene, StyLua et luau-lsp sont communautaires, pas maintenus par Roblox : chacun peut casser après une mise à jour de Studio.

| Nom | À quoi ça sert | Installation | Statut | Licence | Conseil |
|---|---|---|---|---|---|
| Rokit | Gestionnaire d'outils par projet (`rokit.toml`) | GitHub `rojo-rbx/rokit` (v1.2.0) | actif | MIT | **Choix par défaut.** Compatible avec les projets Aftman et Foreman |
| Aftman | Ancien gestionnaire d'outils (`aftman.toml`) | GitHub `LPGhatguy/aftman` (v0.3.0) | archivé | MIT | Migrer vers Rokit |
| Foreman (Roblox) | Ancien gestionnaire d'outils (`foreman.toml`) | GitHub `Roblox/foreman` (v1.7.0) | maintenu par Roblox, surtout pour son usage interne | MIT | Migrer vers Rokit |
| Rojo | Synchronise les fichiers avec Studio, construit `.rbxl` et `.rbxm` | `rokit add rojo-rbx/rojo` (v7.7.1). Plugin : `rojo plugin install`, ou Creator Store 13916111004 (groupe Rojo Foundation) | actif | MPL-2.0 | **Indispensable** dès qu'on code dans des fichiers |
| Argon | Alternative à Rojo, synchro dans les deux sens | `rokit add argon-rbx/argon` (2.0.29) | actif | Apache-2.0 | Seulement si le projet l'utilise déjà |
| Wally | Gestionnaire de paquets Luau (`wally.toml`, `wally.lock`) | `rokit add UpliftGames/wally` (v0.3.2) | maintenu (push 2026-10 ; release stable 0.3.2, pré-versions 0.4.0-alpha) | MPL-2.0 | Standard actuel des paquets Luau |
| pesde | Gestionnaire de paquets Luau plus récent (Roblox, Lune) | GitHub `pesde-pkg/pesde` (v0.7.4) | actif | MIT | Seulement si le projet l'a choisi ; pas de mélange avec Wally |
| wally-package-types | Réexporte les types des paquets Wally pour luau-lsp | `rokit add JohnnyMorganz/wally-package-types` (v1.7.0) | actif | MIT | À relancer après chaque `wally install` |
| Selene | Linter Luau (`selene.toml`) | `rokit add Kampfkarren/selene` (0.32.0) | actif | MPL-2.0 | À utiliser |
| StyLua | Formateur Luau (`stylua.toml`) | `rokit add JohnnyMorganz/StyLua` (v2.6.0) | actif | MPL-2.0 | À utiliser |
| luau-lsp | Serveur de langage, et `luau-lsp analyze` en CI | `rokit add JohnnyMorganz/luau-lsp` (1.70.1) ; extension VS Code « Luau Language Server » | actif | MIT | À utiliser pour le typage strict |
| Darklua | Réécrit du Luau : chemins de `require`, minification, injection de valeurs | `rokit add seaofvoices/darklua` (v0.19.0) | actif | MIT | Utile avec les `require` par chaîne |
| Lune | Lance du Luau hors Roblox : scripts, tests de logique, lecture et écriture de `.rbxl` | `rokit add lune-org/lune` (v0.11.0) | actif | MPL-2.0 | À utiliser pour l'outillage |
| Moonwave | Génère un site de doc depuis les commentaires | GitHub `evaera/moonwave` (v1.4.2) | actif | MPL-2.0 | Pour une bibliothèque publiée |
| roblox-ts | Compilateur TypeScript → Luau (`rbxtsc`) | npm `roblox-ts` 3.0.0 ; projet neuf : `npx create-roblox-ts` (2.0.5). Types : `@rbxts/types` 1.0.958, `@rbxts/compiler-types` 3.0.0-types.0, `@rbxts/services` 1.6.0 | actif | MIT | Pile par défaut de cet utilisateur |
| Asphalt | Envoie images et sons vers Roblox par Open Cloud, génère la table des ids | `rokit add jackTabsCode/asphalt` (v2.0.2) | actif (push 2026-07) | MIT | **Choix par défaut** pour un pipeline d'assets |
| Tarmac (Roblox) | Ancien outil d'envoi d'assets | GitHub `Roblox/tarmac` (v0.8.2) | legacy | MIT | Prendre Asphalt |
| Mantle | Déploiement d'expériences décrit dans un fichier (places, produits, badges) | GitHub `blake-mealey/mantle` (v0.11.18) | peu actif (push 2026-05) | MIT | Pour une équipe qui automatise ses publications |

## Plugins Studio populaires

Installe un plugin depuis sa page officielle du Creator Store, publiée par son auteur. Vérifie l'auteur et le nombre de votes.

| Plugin | À quoi ça sert | Creator Store (id, auteur) | Prix | Conseil |
|---|---|---|---|---|
| Rojo | Relie Studio à `rojo serve` | 13916111004, groupe Rojo Foundation | gratuit | Indispensable avec Rojo. Même version que le CLI |
| Moon Animator 2 | Animation de personnages, caméras et objets | 4725618216, xsixx | 19,99 USD (prix soldé de 33 % le 2026-10-18) | La référence. Les copies « Free » ou « CRACKED » du Creator Store sont des pièges : n'y touche pas |
| Building Tools by F3X | Construction : déplacer, redimensionner, tourner, peindre, souder | 144950355, GigsD4X (et « Import from Building Tools by F3X » 142485815) | gratuit | Classique du build |
| Archimedes | Arcs et cercles réguliers de parts | 144938633, Scriptos (« Archimedes (v3.1.9) ») | gratuit | Pour courbes, routes, arches |
| AutoScale Lite | Convertit Position et Size en Scale ou Offset, ajoute des `UIAspectRatioConstraint` | 1496745047, ZacBytes | gratuit | Utile pour une UI faite à la main dans Studio. Inutile avec une UI React écrite en code |
| Reclass | Change la classe d'une instance en gardant ses propriétés | 2867765019 (classic) et 18962718950 (nouvelle version), Elttob | gratuit | Gain de temps |
| GapFill & Extrude | Comble l'espace entre deux arêtes | 165687726, stravant | gratuit | Classique du build |
| Model Reflect | Symétrie d'un modèle | 217792838, stravant | gratuit | Classique du build |
| Brushtool 2.1 | « Peint » des copies d'objets sur la carte (arbres, rochers) | 2268520847, XAXA | gratuit | Pour la végétation |
| Load Character Lite | Insère un avatar R6 ou R15 pour animer ou tester | 752585459, AlreadyPro | gratuit | Son auteur ne le maintient plus et renvoie vers une version Pro payante |
| Codify | Convertit une sélection Studio en code : Luau, React/Roact, Fusion, Vide, JSX TypeScript | 4749111907, cxmeels | 12,99 USD | Pratique pour passer une maquette Studio en composant React. Demande l'accès web |
| Tag Editor | Ancien éditeur de tags CollectionService | 948084095, Sweetheartichoke (mise à jour 2022) | gratuit | Legacy : Studio a son Tag Editor intégré depuis 2022 et une section Tags dans Properties |
| UI Labs, Flipbook, Hoarcekat | Stories d'UI | voir « Stories et aperçus d'UI » | gratuit | — |

## Pièges

- **Tutoriels périmés.** Ils recommandent encore ProfileService, DataStore2, Knit, Roact, Rodux, Llama, TestEZ, Aftman et Red. Tous sont legacy ou archivés. Équivalents actuels : ProfileStore, Flamework ou modules simples, react-lua, Charm, Sift, Jest, Rokit, Zap.
- **Noms Wally faux, vus dans des skills Roblox pour IA.** `madstudioroblox/profileservice`, `loleris/profileservice`, `loleris/profilestore`, `kampfkarren/datastore2` et `sleitnick/datastore-service` n'existent pas. Vérifie avec l'API avant d'écrire un `wally.toml` (liste complète plus bas).
- **Copies sur Wally.** Plus de 40 paquets `*/profilestore`, des dizaines de `*/zoneplus` et `*/cmdr`, et une dizaine de `*/spr` et de `*/fastcast` (ou `*/fastcastredux`) sont publiés par des inconnus. Prends le scope cité par le dépôt officiel.
- **Un `wally.toml` dans un dépôt ne prouve pas la publication.** ZonePlus déclare `1foreverhd/zoneplus`, ByteNet `ffrostfall/bytenet` 0.5.0 et ClientCast `pyseph/clientcast` : aucun n'est publié.
- **Noms npm trompeurs.** `@rbxts/signal` n'est pas le Signal de Sleitnick (c'est `@rbxts/sleitnick-signal`). `@rbxts/red` n'est pas le Red de red-blox. `@rbxts/replica` (2022) n'est pas le Replica de MadStudio. `@rbxts/bridgenet` est BridgeNet v1, pas BridgeNet2. `@rbxts/topbarplus` n'est pas la variante la plus suivie (`@rbxts/topbar-plus`).
- **Typings en retard.** `@rbxts/fusion` est en 0.2 alors que Fusion est en 0.3 ; `@rbxts/iris`, `@rbxts/trove`, `@rbxts/bytenet`, `@rbxts/cmdr` et `@rbxts/squash` (3.x contre 5.0 sur Wally) ont aussi du retard. Compare la version npm à l'amont avant d'adopter.
- **Aucun paquet `@rbxts/` trouvé** pour Blink, Warp, Packet, Chickynoid, focus-navigation, Otter et ezVisualz. Zap n'en a pas besoin : il génère ses `.d.ts` avec `opt typescript = true`. En roblox-ts, écris un `.d.ts` ou choisis une alternative typée.
- **`require(id)` dans le code.** Certains fils DevForum montrent `require(11671168253)`. Cela charge à chaud la dernière version publiée par un tiers, côté serveur. Insère le module dans la place, ou passe par Wally, puis `require` un ModuleScript local.
- **Modèles « Pro+ », « Free », « CRACKED ».** Des copies payantes ou piratées de plugins gratuits ou payants (GapFill, Reclass, Codify, Moon Animator) circulent sur le Creator Store. Prends l'original, publié par l'auteur.
- **Deux bibliothèques pour un même rôle.** Un projet n'a qu'un réseau, qu'un système de nettoyage, qu'un état global, qu'un système d'admin.
- **Licences.** UI Labs (plugin) et BehaviorTree3 sont sous GPL-3.0. TopbarPlus impose un crédit. Un module sans licence (FastCast, HD Admin : à vérifier) ne se recopie pas dans un dépôt public sans l'accord de l'auteur.
- **Mêmes noms, usages différents.** `@quenty/*` sont du code Luau de Nevermore distribué par npm, pas des typings roblox-ts.

## Noms vérifiés qui n'existent pas

Contrôlés le 2026-10-18. Ces noms sont plausibles, et des IA les proposent, mais le registre ne les connaît pas. Le bon nom est entre parenthèses quand il existe.

- npm (E404) : `@rbxts/profilestore` (`@rbxts/profile-store`), `@rbxts/documentservice` (`@rbxts/document-service`), `@rbxts/bridgenet2`, `@rbxts/warp`, `@rbxts/packet`, `@rbxts/zap`, `@rbxts/blink`, `@rbxts/promise` (intégré à roblox-ts), `@rbxts/otter`, `@rbxts/table-util` (`@rbxts/tableutil`, ancien), `@rbxts/flipbook`, `@rbxts/hoarcekat`, `@rbxts/focus-navigation`, `@rbxts/react-focus-navigation`, `@rbxts/chickynoid`, `@rbxts/lest`, `@rbxts/raycasthitbox` (`@rbxts/raycast-hitbox`), `@rbxts/shapecast-hitbox` (`@rbxts/shapecasthitbox`), `@rbxts/zoneplus` (`@rbxts/zone-plus`), `@rbxts/camerashaker` (`@rbxts/camera-shaker`), `@rbxts/easyvisuals`, `@rbxts/easy-visuals`, `@rbxts/ezvisualz`, `@rbxts/adonis`, `@rbxts/behaviortrees3` et `@rbxts/behavior-trees` (`@rbxts/behavior-tree`).
- Wally (« could not open package ») : `loleris/profilestore` (`lm-loleris/profilestore`), `loleris/profileservice`, `madstudioroblox/profileservice`, `kampfkarren/datastore2`, `ffrostfall/bytenet` (`ffrostflame/bytenet`), `red-blox/zap` (Zap est un outil CLI), `dphfox/fusion` (`elttob/fusion`), `fraktality/spr`, `roblox/otter` (`jsdotlua/otter`), `flipbook-labs/flipbook`, `kampfkarren/hoarcekat`, `teamswordphin/raycasthitbox`, `1foreverhd/zoneplus`, `grayzcale/simplepath`, `sleitnick/camera-shaker`, `sleitnick/datastore-service`, `quenty/nevermore`, `littensy/pretty-react-hooks`, `roblox/focus-navigation`, `madstudioroblox/replica`, `pyseph/clientcast`.
- GitHub (404) : `EtiTheSpirit/FastCastAPIv2`, `EtiTheSpirit/FastCast`, `fractality/spr` (le bon : `Fraktality/spr`), `chriscerie/react-spring` (le bon : `chriscerie/roact-spring`).

## Checklist avant d'ajouter un paquet

- [ ] Le projet n'a pas déjà un paquet qui fait ce travail.
- [ ] Le moteur ne le fait pas déjà (`TweenService`, `PathfindingService`, `UnreliableRemoteEvent`, Tag Editor…).
- [ ] Le nom exact existe : `npm view`, API Wally, ou page officielle.
- [ ] Le scope ou l'auteur est celui du dépôt officiel, pas une copie.
- [ ] Le dépôt n'est pas archivé, et la dernière version date de moins de 2 ans, ou le code est assez stable pour s'en passer.
- [ ] En roblox-ts : des typings existent et suivent la version amont, sinon prévoir un `.d.ts`.
- [ ] La licence permet l'usage et ses conditions (crédit, notice) sont notées.
- [ ] Un modèle du Creator Store a été ouvert et relu : pas de `require` par id, `loadstring`, `getfenv`, `setfenv`, script caché.
- [ ] La version est épinglée, le fichier de verrouillage (`wally.lock`, `package-lock.json`) est commité.
- [ ] L'utilisateur a dit oui.
