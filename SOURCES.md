# Sources

Ce skill a été construit le 2026-10-18 en lisant intégralement 18 dépôts publics de skills Roblox, puis en fusionnant leur savoir. Le texte et les exemples de code sont réécrits : aucun passage n'est copié. Les dépôts sans licence n'ont servi que pour des idées et des faits, jamais pour leur formulation.

Ordre de confiance utilisé pour trancher les contradictions : leçons d'un projet réellement testé, puis le plugin officiel de Roblox et la doc officielle, puis les dépôts les mieux notés ci-dessous.

| Dépôt | Auteur | Licence | Fiabilité (/5) | Apport principal |
|---|---|---|---|---|
| [Roblox/roblox-studio-agent-plugin](https://github.com/Roblox/roblox-studio-agent-plugin) | Roblox Corporation | MIT | 5 (périmètre étroit) | Serveur MCP officiel de Studio : outils, configuration, règles de sécurité de l'agent |
| [MSayib/roblox-dev-skill](https://github.com/MSayib/roblox-dev-skill) | MSayib | MIT | 4,5 | Faits mesurés dans Studio, MCP, Luau récent, achats, réglages réservés aux plugins |
| [andrian-syh/roblox-best-practices-skill](https://github.com/andrian-syh/roblox-best-practices-skill) | Muhammad Andriansyah | MIT | 4,5 | Limites chiffrées, vérification d'API, diagnostic, faux positifs de revue, outils de synchro |
| [TabooHarmony/roblox-brain](https://github.com/TabooHarmony/roblox-brain) | TabooHarmony | MIT | 4,5 | Contrats d'échec (données, réseau, MCP), UI mobile, collaboration agent-utilisateur |
| [ivar-anon/roblox-dev](https://github.com/ivar-anon/roblox-dev) (plugin de l'annuaire Anthropic) | Ivar | MIT | 4,5 | Validation des remotes, session locking, revue par sévérité, tests et commandes de triche |
| [m4bwav/roblox-agent](https://github.com/m4bwav/roblox-agent) | Mark Rogers | MIT | 4 | Mesures réelles du MCP et d'Open Cloud, échelle de vérification, docs pour agents |
| [hishambaoom/roblox-agent-skill-no-README-](https://github.com/hishambaoom/roblox-agent-skill-no-README-) | contributeurs roblox-agent-skill | MIT (assets tiers exclus) | 4 | Leçons de deux jeux publiés, recette d'UI chiffrée, travail multi-agents, pièges de Lune |
| [nonlooped/roblox-suite](https://github.com/nonlooped/roblox-suite) | nonlooped | MIT | 4 | DataStores, achats, téléportation, Open Cloud, Rojo 7.7 |
| [afrxo/roblox-agent-skills](https://github.com/afrxo/roblox-agent-skills) | afrxo | MIT | 3,5 | Système de types Luau, outillage, primitives et design d'UI, Fusion 0.3, UI Labs |
| [luumenlabs/luau-skills](https://github.com/luumenlabs/luau-skills) | luumenlabs | MIT | 3,5 | Données, MemoryStore, Open Cloud, OAuth, webhooks |
| [gamedev-skills/awesome-gamedev-agent-skills](https://github.com/gamedev-skills/awesome-gamedev-agent-skills) (partie Roblox) | Abhishek Barali et contributeurs | Apache-2.0 | 3,5 | Réseau, personnages, physique, UI moteur, niveaux de preuve |
| [AshExplained/roblox-skills](https://github.com/AshExplained/roblox-skills) | AshExplained | MIT | 3 | Tranches verticales, design de jeu, audit des assets du Creator Store |
| [brockmartin/roblox-game-skill](https://github.com/brockmartin/roblox-game-skill) | Brock Martin | Aucune (idées seulement) | 3 | Workflows d'agent, chiffres de genres, sharp edges |
| [ohzw/roblox-dev-skills](https://github.com/ohzw/roblox-dev-skills) | ohzw | Aucune (idées seulement) | 3 | Motifs React-Lua, construction 3D via MCP testée en A/B |
| [gogolumo/rbsmithy-roblox-claude-skill](https://github.com/gogolumo/rbsmithy-roblox-claude-skill) | gogolumo | MIT | 3 | Méthode, garde-fous de périmètre, pipeline d'assets 3D |
| [MarkRuiz00/RbxUiStudio](https://github.com/MarkRuiz00/RbxUiStudio) | RbxUiStudio | MIT | 4 (savoir UI) | Meilleure source sur l'UI du moteur : chiffres pour téléphone, styles, conventions des jeux à succès. Son outil (serveur local et plugin) écoute tout le réseau local et exécute du Luau reçu par HTTP : ne pas l'installer sans en parler à l'utilisateur. |
| [sentinelcore/roblox-skills](https://github.com/sentinelcore/roblox-skills) | sentinelcore | Aucune (idées seulement) | 2,5 | Fiches rapides (animations, UGC) revérifiées avant reprise |
| [stephenshorton/flamework-template](https://github.com/stephenshorton/flamework-template) (skills `.claude/`) | stephenshorton | Aucune (idées seulement) | 2,5 | Configuration roblox-ts moderne, Flamework |

## Ce qui n'a pas été repris

- Les API inventées ou fausses repérées dans plusieurs dépôts. Elles sont listées dans `references/pieges.md` pour ne pas les réutiliser.
- Les installeurs en `curl | bash` ou `irm | iex`, les conseils contraires aux règles Roblox (outils d'espionnage de remotes, design manipulateur pour enfants, pubs récompensées sans `AdService`), les exemples de `ProcessReceipt` ou de sauvegarde non sûrs.
- La règle « tout dans Studio, rien en code » d'un dépôt : elle est incompatible avec une UI React et un projet Rojo.

## Leçons maison

Les points marqués comme testés (projet roblox-ts + `@rbxts/react` + Rojo 7.7, Windows, 2026-10-18) viennent d'un projet réellement monté et lancé dans Studio. Exemple : le mapping du scope `@rbxts-js`, sans lequel aucune UI React ne s'affiche.
