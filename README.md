# roblox-gamedev, un skill Claude Code pour faire des jeux Roblox avec l'IA

Un seul skill, en français, qui donne à Claude ce qu'il faut savoir pour coder des jeux Roblox du premier coup : Luau, architecture client/serveur, sécurité, sauvegarde, achats, UI en roblox-ts + React, animations, Rojo, Studio MCP, Open Cloud, performance, gameplay, 3D, design et live ops.

Il fusionne 18 skills Roblox open source, lus en entier, triés et revérifiés contre la doc officielle, avec les leçons d'un vrai projet roblox-ts + `@rbxts/react` + Rojo testé dans Studio. Les crédits et licences sont dans [SOURCES.md](SOURCES.md).

## Contenu

| Fichier | Sujet |
|---|---|
| `SKILL.md` | Point d'entrée : règles d'or, méthode de travail, niveaux de risque, table des références |
| `references/luau.md` | Langage Luau, typage strict, bibliothèque standard |
| `references/architecture.md` | Client/serveur, place des scripts, frameworks, nettoyage |
| `references/securite.md` | RemoteEvents, validation, anti-triche, assets du Creator Store |
| `references/donnees.md` | DataStores, ProfileStore, classements, MemoryStore, MessagingService, quotas |
| `references/monetisation.md` | Game passes, developer products, `ProcessReceipt`, économie |
| `references/ui-fondamentaux.md` | UI du moteur : mise en page, mobile, manette, texte, images |
| `references/ui-react.md` | `@rbxts/react`, react-lua, Fusion, animations d'UI (code testé) |
| `references/outils.md` | Rojo, roblox-ts, Wally, Rokit, lint, tests, CI |
| `references/studio-mcp.md` | Piloter Roblox Studio par son serveur MCP |
| `references/open-cloud.md` | API web, clés API, upload d'assets, publication de place, Luau Execution, webhooks |
| `references/performance.md` | CPU, mémoire, réseau, streaming, profilage |
| `references/gameplay.md` | Personnages, combat, pets, PNJ, physique, caméra, entrées, téléportation, genres |
| `references/monde-3d.md` | Construction par code, voxels, meshes, éclairage, terrain |
| `references/design-liveops.md` | Boucle de jeu, onboarding, rétention, publication, conformité |
| `references/pieges.md` | API inventées et dépréciées, diagnostic « ça ne marche pas » |
| `references/paquets.md` | Bibliothèques et outils connus, avec identifiants vérifiés |

## Installation

Claude Code lit les skills dans `~/.claude/skills/`.

macOS ou Linux :

```bash
git clone https://github.com/MysticSaba-max/roblox-gamedev-skill.git ~/.claude/skills/roblox-gamedev
```

Windows (PowerShell) :

```powershell
git clone https://github.com/MysticSaba-max/roblox-gamedev-skill.git "$env:USERPROFILE\.claude\skills\roblox-gamedev"
```

Avec la CLI `skills` :

```bash
npx skills add MysticSaba-max/roblox-gamedev-skill
```

Mise à jour : `git pull` dans le dossier du skill.

## Pour qu'il se lance à chaque fois

La description du skill demande à Claude de l'utiliser pour toute tâche Roblox. Pour le garantir, ajoute cette ligne à ton `~/.claude/CLAUDE.md` :

```markdown
Dès qu'une tâche touche au développement Roblox (Luau, roblox-ts, Rojo, Roblox Studio, UI de jeu, DataStore, RemoteEvent, game pass), lancer le skill `roblox-gamedev` avant tout, à chaque fois.
```

## Fraîcheur

Fusionné et vérifié le 2026-10-18. Roblox change chaque semaine : les chiffres datés (quotas, prix, politiques) sont à revérifier avant de s'y fier. Les points non confirmés sont marqués « (à vérifier) ».

## Licence

MIT, voir [LICENSE](LICENSE). Le texte est original ; les dépôts sources et leurs licences sont crédités dans [SOURCES.md](SOURCES.md).
