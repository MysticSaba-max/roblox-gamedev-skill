# Design et live ops : boucle, onboarding, rétention, publication, conformité

À lire quand : tu conçois ou remanies un jeu (idée, boucle, découpage en tranches), tu travailles l'onboarding, la rétention, les analytics, la page du jeu ou la publicité, tu vérifies la conformité (âge, politiques, objets aléatoires payants), tu prépares une sortie, un rollback ou un événement live, ou tu localises le jeu. Le code des systèmes est dans gameplay.md, les achats dans monetisation.md, la publication automatisée dans open-cloud.md.

## Sommaire

1. Idée et boucle de jeu
2. Tranches verticales
3. Onboarding
4. Rétention
5. Analytics
6. Découverte et publication
7. Conformité et design responsable
8. QA et playtests
9. Porte de release et rollback
10. Live ops
11. Localisation et accessibilité
12. Pièges
13. Checklist

## 1. Idée et boucle de jeu

- **Diagnostique avant de prescrire.** Demande les preuves qui existent : exports du Creator Dashboard, dates de release, cohortes, retours joueurs. Dis ce qui manque. N'invente jamais une métrique, une cohorte ou une cause.
- **Questions de concept** : que fait le joueur en boucle ? Quel fantasme ? Qu'est-ce qui change après chaque cycle ? Pourquoi est-ce mieux à plusieurs ? Que montrer honnêtement dans l'icône et la miniature ? Qu'est-ce qui reste amusant sans payer ? Quelle charge de production grandit avec le succès ?
- **80 / 20** : environ 80 % de mécaniques, d'UI et de progression déjà connues des joueurs du genre, 20 % de différence (thème, fantasme, twist). La familiarité baisse la barrière, la nouveauté donne la raison de cliquer. Le thème est une décision de design, pas un skin.
- **Cycle des tendances** : premier arrivé → saturation par les clones → mutation. Arriver en pleine saturation avec un clone pur est la pire position ; copier titre et miniature peut déclencher des pénalités de métadonnées.
- **Trois couches qui tournent ensemble** : boucle cœur (moment à moment → durée de session), méta (heure à heure → J1), progression longue (jours, mois → J30). Diagnostique vers le haut, construis vers le bas : un mauvais J1 vient presque toujours de la couche 1 ou 2.
- **Boucle écrite** : motivation → action → risque ou coût → récompense → amélioration ou déblocage → motivation suivante. Chaque action reçoit un retour visuel ET sonore immédiat. Plus de 5 s sans action qui compte, ou 20 min sans progrès ressenti, et le joueur décroche (heuristiques).
- **Cadre « trois tâches »** : A = la boucle, identifiable rien qu'avec le titre et la miniature ; B = ce qui rend A plus facile ; C = un but long qui exige A. Si un joueur ne peut pas nommer A depuis la page du jeu, la page est cassée.
- **Questions obligatoires pour chaque feature** : que fait le joueur dans les 30 premières secondes ? Quelle action répète-t-il après 5 minutes ? Qu'est-ce qui change après la récompense ? Quelle nouvelle décision s'ouvre ? Quelle boucle sert-elle (principale, progression, sociale, collection) ? Quel événement analytics prouve qu'elle marche ? Une feature qui ne sert aucune boucle sort du MVP.
- **Livre une boucle répétable, puis du contenu chaque semaine.** Une boucle ratée ne se répare pas par des updates ; une bonne boucle aux systèmes faibles, si.
- **Profondeur et lisibilité** : une entrée claire, une condition de victoire claire, un plafond de compétence haut. « Facile à apprendre, dur à maîtriser » : 10 minutes doivent amuser le joueur occasionnel, 10 heures d'optimisation le joueur acharné.

### Stress-test d'un concept

- Une question à la fois, avec ta réponse recommandée. Ordre des dépendances : boucle → 30 premières secondes → progression → économie → social → monétisation → contrôles mobiles → cadence live ops.
- Sonde par des scénarios concrets : « un nouveau joueur mobile apparaît avec 0 pièce : quel est son premier tap, que lui rapporte-t-il ? ».
- **Fixe le vocabulaire** : un seul mot par concept pour tout le projet (pièce, gemme ou jeton : choisis). Consigne les décisions dans `CONTEXT.md` pour ne pas les rediscuter.
- Note de 1 à 5 : clarté de la première session, répétabilité de la boucle, rétention sociale, profondeur de progression, adéquation mobile, monétisation équitable, potentiel live ops, faisabilité. Retravaille tout concept faible en clarté de première session ou en mobile.
- Anti-motifs : cinématiques forcées, tutoriels longs, contrôles desktop seulement, progression cachée, pénalités précoces dures, raccourcis pay-to-win, contenu épuisé avant 2–3 h de jeu.
- Arrête quand la boucle, la première session et la monétisation s'écrivent sans flou.

### Travailler avec le propriétaire du jeu

- **Son design est final.** Difficulté, prix, taux, cooldowns : applique exactement ses chiffres (« 25 % plus lent » = × 0,75). N'adoucis rien de toi-même.
- Présente tes idées en liste classée avec un effort S, M ou L ; ne construis que ce qu'il choisit.
- Avant de générer beaucoup de contenu par IA, écris une courte bible : prémisse, règles du monde, factions, règles de nommage, contradictions interdites. Sinon les lieux, noms et dialogues générés se contredisent.
- Étude d'un concurrent : observe le comportement visible, écris une spec, puis implémente à ta façon. Ne copie ni noms, ni UI, ni cartes, ni art, ni audio, ni code, ni valeurs de réglage. Ne prétends jamais lire le code caché d'une expérience publique.

## 2. Tranches verticales

**Ne construis jamais tout un jeu, ni tout un système, en une passe.** C'est la cause principale des jeux à moitié cassés générés par IA. Concept → PRD court → tranches → une tranche à la fois, playtestée avant la suivante.

- **MVP « un de chaque »** : 1 lieu, 1 objectif, 1 PNJ, 1 interaction, 1 défi, 1 récompense, 1 état sauvegardé, 1 HUD, 1 couche audio, 1 funnel analytics, 1 checklist de test. Un monde ouvert à 30 îles n'est pas un MVP.
- **PRD court** (assez pour découper, pas un roman) : promesse en une phrase ; joueur et appareils visés ; 30–60 premières secondes ; boucle ; échelle de progression ; sources et puits de l'économie ; accroche sociale ; monétisation sans pay-to-win ; UX mobile ; pipeline de contenu live ops ; risque principal et parade ; événements qui mesurent le succès. Synthétise ce qui est déjà décidé au lieu de réinterroger l'utilisateur.
- **Une tranche est une balle traçante** : le chemin le plus fin qu'un joueur peut sentir, à travers toutes les couches qu'il touche (entrée client, autorité serveur, remotes, DataStore, UI). Bon : « je tape le minerai, le serveur donne 1 pièce, le HUD affiche le solde, le solde survit au rejoin ». Mauvais : « tout le module d'économie » ou « toute l'UI ».
- Règles : une capacité étroite mais complète ; vérifiable seule en playtest Studio (c'est le critère d'acceptation) ; beaucoup de tranches fines plutôt que peu d'épaisses ; état précieux autoritaire serveur dès la première.
- **Tranche 1** = la plus petite preuve de la boucle : une action → une récompense → un objectif suivant visible. Ordre ensuite : persistance → progression → social → monétisation → finition. Chaque tranche s'appuie sur une tranche déjà jouable.
- Types : **autonome** (construite et vérifiée sans décision humaine, à préférer) et **avec humain** (validation de design, arbitrage d'équilibrage, action que seul l'utilisateur peut faire dans Studio). Une tranche d'UI reste « avec humain » tant qu'aucun brief de design n'est approuvé.
- Présente le découpage en liste numérotée : titre côté joueur, type, « bloquée par », vérification de playtest (action exacte dans Studio), état autoritaire concerné. Demande si la granularité, les dépendances et les coupes conviennent ; itère jusqu'à l'accord.
- **Feuille de route** : `ROADMAP.md`, source unique. `[ ]` pas commencé ; `[~]` construit et testé, en attente d'approbation ; `[x]` seulement après implémentation, playtest ET approbation explicite de l'utilisateur. Un playtest réussi ne vaut pas approbation.
- Ne dis jamais « corrigé » avant que l'utilisateur l'ait vu en jeu. Finis chaque tranche par : ce qui est vérifié, ce qui ne l'est pas, l'étape suivante.
- Priorités d'un lot : bugs qui font perdre progression ou Robux > crashs et flux bloqués > demandes explicites > ajustements téléphone > finition > idées.
- Dans un projet Rojo ou roblox-ts, Git est la source de vérité du code. Ignore la règle « ne jamais commiter le code » de certains skills.
- Ne produis pas de contenu à grande échelle tant que la boucle, l'utilisabilité mobile, la sauvegarde, la sécurité et les premiers signaux de rétention ne sont pas crédibles.

## 3. Onboarding

**Enseigne par l'action.** Pas de popup ni de mur de texte avant de jouer. Invites adaptées à l'appareil réel (glyphes tactile, manette, clavier). Une mécanique à la fois, en contexte. Une menace se présente d'abord sans danger. Appuie-toi sur les conventions des autres jeux Roblox.

Cibles de temps (heuristiques de terrain, recoupées par plusieurs sources) :

| Moment | Ce qui doit arriver |
|---|---|
| 0–5 s | Le joueur apparaît dans le monde, face au premier objet utile. Ni logo, ni cinématique, ni dialogue, ni popup. |
| 5–10 s | Il fait le verbe cœur : une seule action demandée. |
| 10–30 s | Première récompense, avec un retour fort. |
| 30–60 s | Première boucle bouclée, objectif suivant visible et proche. Le contexte social et la progression se révèlent après l'action. |
| 1–2 min | Premier vrai progrès (simulator : amélioration avant 30 s, éclosion avant 2 min). |
| ≈ 5 min | Le jeu est amusant. La doc Roblox recommande d'y arriver en ~5 min : prends-le comme point de diagnostic. |

- **Dévoilement progressif** : 0–5 min, la boucle seule (une monnaie, une action, un but) ; 5–15 min, la première voie d'amélioration et une mécanique secondaire ; 15–30 min, le social, la boutique, les monnaies secondaires ; au-delà, les systèmes avancés (guildes, échanges, prestige).
- **Tutoriel court** : idéalement moins d'une minute ; des succès récents tiennent en 15–20 s. Chaque étape de gain dure ≤ 3 s, avec des gains boostés pendant le tutoriel. Texte de ~5 mots, flèches et surlignages plutôt que paragraphes, et le POURQUOI (« fais éclore une abeille pour agrandir ton essaim »).
- **Guide sans bloquer** : une courte liste de tâches de débutant reste au HUD (tâche, « 0/1 », récompense, coche verte), et la flèche vers l'objectif reste après le tutoriel.
- Données de terrain, petits échantillons : un tutoriel de 16 étapes convertissait 32,9 % ; ramené à 12 étapes, il est passé à 57,8 % ; 85 % de complétion est une base saine ; un seul bug dans une étape a fait tomber un segment de 80 % à 30 %.
- **Corrige en retirant, pas en ajoutant de l'UI.** Auto-équipe la récompense (obliger à équiper un outil se lit « le jeu est cassé ») ; rapproche la première cible du spawn (un échec à la première étape est passé de 22 % à 12 %) ; téléporte le joueur vers chaque objectif.
- **Le tutoriel est une machine à états** : étapes vers l'avant seulement, déclenchées par l'état du joueur (seuil de monnaie) et non par un script fixe ; un kick au milieu d'une étape reprend à la bonne étape ; zone sûre et achats inadaptés désactivés pendant le tutoriel. Il finit là où la boucle se ferme ; les récompenses quotidiennes viennent après ; un revenu passif juste après évite les départs.
- **Ce qui tue le J1 sans bruit** : intro impossible à passer ; boutique dans les 30 premières secondes ; murs de texte ; longue marche jusqu'au fun ; lag au join (teste à froid sur le pire appareil visé) ; like demandé avant la première récompense.
- Onboarding spatial : depuis chaque spawn, le repère suivant est visible, la caméra n'est pas dans le décor, le joueur ne fait face ni à un mur ni à un danger.
- **Test des 7 ans** : un enfant qui ne lit pas l'anglais comprend avec ce qui est à l'écran, et part au moindre but inatteignable ou danger invisible.
- Jeu qui a besoin d'autres joueurs : solo jouable dès la première minute, ou bots et PNJ pour peupler les premières minutes. Un lobby vide fait partir pour de bon : n'ouvre pas la pub avant de pouvoir tenir assez de joueurs simultanés.
- Mobile : donne le jeu à un joueur cible sur téléphone sans rien expliquer, et regarde. Si une amélioration demande 500 taps, ajoute « Acheter 100 » ou « Max ».
- **Écarte les leviers de pression** : pas de boost qui expire juste après la première victoire pour pousser l'achat, pas d'offre avant la première récompense (section 7).
- Instrumente chaque étape avec un funnel (section 5).

## 4. Rétention

### Lire les chiffres avec prudence

Les repères publiés se contredisent fortement.

| Source | J1 | J7 | J30 |
|---|---|---|---|
| Références de design (optimistes) | 20 / 30 / 40 % (bon, très bon, excellent) | 8 / 15 / 20 % | 3 / 7 / 10 % |
| Données de terrain 2026 | 50–60 % pour les géants historiques | 2 % ≈ 85–90e percentile d'un simulator ; tycoon 90e percentile ≈ 15 % | — |

- Compare-toi au benchmark de ton genre dans le Creator Dashboard, jamais à un chiffre universel.
- À faible CCU, les stats sont du bruit : un J1 de 3 % n'est pas un verdict. Le J7 vaut 0 pendant 7 jours. J1 et J7 arrivent avec ~3 jours de retard.
- Le J1 peut bouger sans release : nombre de joueurs simultanés, mix d'appareils, saison, passage du trafic pub au trafic Home. Compare des cohortes mûres avant de créditer une update.

Lecture rapide (hypothèses à tester, jamais des causes certaines) :

| Symptôme | Pistes |
|---|---|
| Play-through rate bas | icône, miniature, titre, promesse, public visé |
| Rebond élevé à la première partie | page trompeuse, join raté, perf, FTUE confuse, fun tardif |
| Session moyenne < 10 min | boucle cœur pas assez prenante |
| J1 bas | boucle, onboarding, stabilité, objectifs, premier gain |
| J7 ou J30 bas | progression, variété, social, live ops, fin de jeu, inflation |
| Conversion basse | valeur mal communiquée, friction, catalogue mince |

Échelle des besoins, à déboguer par le bas : 1 « ça marche, je suis dans le jeu » (rebond avant 60 s) → 2 « je sais quoi faire » (funnels ; regarde 5 inconnus jouer sans les aider) → 3 « c'est agréable » (J1, temps de jeu) → 4 « j'avance » (J2–J7 : le prochain déblocage toujours visible) → 5 « je compte ici » (co-jeu : pour chaque récompense, qui la voit ?) → 6 « c'est à moi » (J8–J28 : le joueur se fixe ses propres buts).

### Première semaine

- J1 : prouve vite que l'action est amusante. J2 : crée l'habitude (série, objectif inachevé, raison sociale). J3 : débloque un contenu marquant vers 2–3 h de jeu. J4–J6 : objectifs moyens visibles, collection, identité sociale. J7 : livre la récompense annoncée dès J1.

### Mécaniques de retour

- **Connexion quotidienne** : jours 1–3 petits gains utiles, 4–6 moyens, 7 gros lot, puis un nouveau cycle aux bases plus hautes. Montre tout le calendrier et nomme le gros lot du jour 7 dès le titre ; rends la collecte cérémonielle (écran dédié, animation, son).
- **Séries avec soupape** : jamais de remise à zéro pour un seul jour manqué (gel ou pause). Une série cassée produit un départ, pas un retour.
- **Rendez-vous** : récompense qui mûrit à heure fixe, minuterie précise (« Retour dans 4 h 32 min »). Intervalle de retour d'environ 24 h.
- **Gains hors ligne** (jeux à gain passif seulement) : annoncés au retour par un écran et un son, plafonnés à environ un jour, toujours inférieurs au jeu actif.
- **Collections** : progression « X / Y » par catégorie, silhouettes pour l'inconnu, bonus de série complète.
- **Progression visible** : l'échec n°1 observé sur plusieurs jeux est une progression cachée. Montre l'état final désirable dès le début (l'aura d'un autre joueur), « prochain événement dans X min » au HUD.
- **Social** : objectifs coop, groupes et clans avec objectifs partagés (le levier J30 le moins cher), cadeaux, demandes d'aide, célébrations publiques, classements qui récompensent aussi la participation, rejoindre un ami.
- **Rotation** : retirer des objets cosmétiques et en ramener parfois (recolorés) relance les absents.
- **J7 ou J30 bas ≠ « ajoute des récompenses quotidiennes ».** Regarde d'abord les objectifs, la vitesse de progression, la variété, la maîtrise, le co-jeu, la fin de jeu et l'inflation qui rend l'ancien contenu inutile.
- Un pic d'événement n'est pas de la rétention : un jeu monté à 7 000 joueurs simultanés en 72 h est retombé à 300–400 ensuite.
- Respecte le temps du joueur : chaque session, même de 5 min, rapporte quelque chose ; gains AFK réduits plutôt que nuls ; pas d'attente inutile. Un jeu noyé d'effets épuise : garde le spectacle pour les moments forts.

### Invitations, parrainage, notifications, groupe (doc, 2026-10-18)

- **Inviter un ami** (client) : `SocialService:CanSendGameInviteAsync(player)` sous `pcall`, puis `SocialService:PromptGameInvite(player, options)`. `ExperienceInviteOptions` : `PromptMessage` (trop long, il n'est pas affiché), `InviteUser` (un ami précis, sinon liste d'amis), `InviteMessageId` (chaîne créée dans Creator Dashboard → Engagement → Notifications, avec `{experienceName}`), `LaunchData` (200 caractères, lu par `GetJoinData()`). Bouton « Inviter » au moment d'un succès, jamais une popup au spawn.
- **Parrainage** (serveur, à `PlayerAdded`) : `player:GetJoinData().ReferredByPlayerId` donne l'UserId de l'invitant, pour toute invitation ; absent des définitions de septembre 2026, lis-le par `(joinData :: any).ReferredByPlayerId` et ignore `nil` ou `0`. Récompense le filleul une fois (drapeau dans son profil), plafonne et espace les gains du parrain, jamais de Robux. `LaunchData` vient du client : valide-le comme un argument de remote. Bannière : Engagement → Referral Rewards (une seule publiée à la fois, jeu en ligne depuis au moins un jour).
- **Notifications hors du jeu** (13 ans et plus, sur opt-in) : côté client, `ExperienceNotificationService:CanPromptOptInAsync()` sous `pcall`, puis `PromptOptIn()` ; texte non modifiable, rien ne s'affiche pour un moins de 13 ans, un joueur déjà inscrit ou un joueur qui l'a vu dans les 30 derniers jours. Envoi par le serveur via Open Cloud (`CreateUserNotification`, open-cloud.md, ou le paquet Open Cloud du Creator Store) avec `messageId` = id d'une chaîne de 99 caractères au plus ; `OpenCloudApiV1:CreateUserNotificationAsync` lève toujours une erreur. Une notification par joueur, par jour et par expérience, livraison non garantie, jeu d'au moins 100 visites. Personnelle et actionnable (« ton œuf a éclos »), sans fausse urgence, jamais obligatoire pour jouer.
- **Groupe Roblox** : serveur, `player:IsInGroupAsync(groupId)` sous `pcall` ; le résultat est en cache par machine (un joueur parti du groupe reste membre côté serveur jusqu'à son départ). Client : `GroupService:PromptJoinAsync(groupId)` renvoie un `Enum.GroupMembershipStatus` (`Joined`, `JoinRequestPending`, `AlreadyMember`, `None`) ; il ne vide que le cache du client, donc le serveur peut ne voir l'adhésion qu'au prochain join (à vérifier). Le bonus se décide sur le serveur, jamais sur la réponse du client. Rang : `GroupService:GetRolesInGroupAsync(userId, groupId)` → `{ IsMember, Roles = { { Id, Name, Rank } } }`, du plus haut au plus bas (`GetRankInGroup` est déprécié).

### Badges en jeu

Attribution **par le serveur seulement** avec `BadgeService:AwardBadgeAsync(userId, badgeId)`, qui cède. Conditions : joueur connecté, badge pas encore possédé, place de l'expérience du badge, badge activé (`GetBadgeInfoAsync(badgeId).IsEnabled`). Limites par minute : `AwardBadgeAsync` et `UserHasBadgeAsync` 50 + 35 × joueurs ; `CheckUserBadgesAsync(userId, ids)` (10 ids au plus) 10 + 5 × joueurs (doc, 2026-10-18). Création : section 6 ; test en jeu publié (section 8).

```luau
--!strict
-- ServerScriptService/Badges.luau (ModuleScript) : au plus un essai par badge et par session.
local BadgeService = game:GetService("BadgeService")
local Players = game:GetService("Players")

local BADGES: { [string]: number } = { Welcome = 0, FirstEgg = 0 } -- ids copiés du Creator Dashboard
local tried: { [Player]: { [number]: boolean } } = {}
local Badges = {}

function Badges.award(player: Player, key: string)
	local badgeId, done = BADGES[key], tried[player]
	if badgeId == nil or done == nil or done[badgeId] then
		return
	end
	done[badgeId] = true -- posé avant le yield : deux déclencheurs rapprochés font un seul appel
	local ok, result = pcall(function()
		local info = BadgeService:GetBadgeInfoAsync(badgeId) -- le moteur le met en cache un court moment
		if info.IsEnabled ~= true or player.Parent ~= Players then
			return false -- badge désactivé ou joueur parti : rien à faire
		end
		return BadgeService:AwardBadgeAsync(player.UserId, badgeId)
	end)
	if not ok then
		done[badgeId] = nil -- échec réseau : le prochain déclencheur réessaie
		warn(`badge {key} : {result}`)
	end
end

function Badges.start()
	local function onJoin(player: Player)
		tried[player] = {}
		Badges.award(player, "Welcome")
	end
	Players.PlayerAdded:Connect(onJoin)
	Players.PlayerRemoving:Connect(function(player)
		tried[player] = nil
	end)
	for _, player in Players:GetPlayers() do
		task.spawn(onJoin, player)
	end
end

return Badges
```

## 5. Analytics

**Instrumente dès la sortie**, pas après un problème : la donnée qui manque est celle dont on avait besoin. Journalise côté serveur ; le client n'envoie que des indices d'UI, par un remote à liste blanche et à débit limité.

| API `AnalyticsService` | Usage |
|---|---|
| `LogOnboardingFunnelStepEvent(player, step, stepName)` | Funnel d'onboarding, une fois par joueur |
| `LogFunnelStepEvent(player, funnelName, funnelSessionId, step, stepName)` | Funnel récurrent (boutique, boss), un id par tentative (seuls les 10 derniers id par joueur et par funnel comptent) |
| `LogEconomyEvent(...)` | Sources et puits par monnaie (monetisation.md) |
| `LogProgressionEvent(...)` | Progression par niveau ou zone (raccourcis `LogProgressionStartEvent`, `…CompleteEvent`, `…FailEvent`) |
| `LogCustomEvent(player, eventName, value?, customFields?)` | Compteur ou valeur ; 3 champs `Enum.AnalyticsCustomFieldKeys.CustomField01` à `03` (clé `.Name`, valeur en chaîne) |

Limites (doc « event types » et référence AnalyticsService, 2026-10-18) : 120 + 20 × CCU requêtes par minute ; 100 noms d'événements personnalisés ; 5 monnaies selon la référence de `LogEconomyEvent` (le tableau « event types » dit 10 types de ressources : vise 5) ; 10 funnels de 100 étapes ; 3 champs par événement ; au-delà de 8 000 valeurs combinées, regroupement en « Other » (après 20 `transactionType` et 100 `itemSku` aussi) ; compteurs remis à zéro chaque jour. La doc ne dit pas ce qui arrive au-delà ; une source rapporte des événements perdus sans erreur : reste loin des limites et surveille l'Error Report.

```luau
--!strict
-- ModuleScript serveur : analytics qui ne cassent jamais le jeu.
local AnalyticsService = game:GetService("AnalyticsService")
local RunService = game:GetService("RunService")

local IS_STUDIO = RunService:IsStudio()

local Analytics = {}

function Analytics.onboardingStep(player: Player, step: number, stepName: string)
	if IS_STUDIO then
		print(`[analytics] onboarding {step} {stepName}`)
		return
	end
	local ok, err = pcall(function()
		AnalyticsService:LogOnboardingFunnelStepEvent(player, step, stepName)
	end)
	if not ok then
		warn("analytics :", err)
	end
end

-- La variante va dans un champ, jamais dans le nom : « EnemyKilled » + « Zombie ».
function Analytics.custom(player: Player, eventName: string, value: number?, variant: string?)
	if IS_STUDIO then
		print(`[analytics] {eventName} {value} {variant}`)
		return
	end
	local fields = if variant then { [Enum.AnalyticsCustomFieldKeys.CustomField01.Name] = variant } else nil
	pcall(function()
		AnalyticsService:LogCustomEvent(player, eventName, value or 1, fields)
	end)
end

return Analytics
```

- **Cardinalité** : un événement + des champs pour les variantes (`EnemyKilled` avec le champ « Sword »), jamais `EnemyKill_Sword`. Noms stables, payloads versionnés, aucune donnée personnelle ni texte libre.
- **Journalise le succès**, pas la tentative ; ou les deux comme événements distincts. Inclus les issues négatives : combat perdu, achat bloqué.
- Funnel d'onboarding : une étape sautée est comptée comme faite quand une étape suivante arrive. Le graphe s'affiche, mais peut mentir sur le vrai parcours.
- Découpe finement : « boutique » = ouverte → produit vu → achat tenté → achat réussi. Les funnels montrent l'étape exacte où les joueurs partent ; couple-les avec l'observation d'un vrai joueur neuf.
- Économie : le solde final vient de la mutation autoritaire, jamais recalculé depuis `leaderstats`.
- Base d'événements : début de session, spawn, première action ; étape de tutoriel vue, finie, sautée ; boucle réussie ou ratée ; récompense réclamée ; amélioration vue, achetée, trop chère ; quête acceptée, finie ; invitation, groupe, échange ; boutique et achats ; mort, retry, zone débloquée, niveau, prestige.
- Événements fréquents (kills, ramassages) : agrège et envoie par lots.
- Validation : Creator Hub → Analytics → « View Events » (quasi temps réel) ; les graphiques mettent environ 24 h à apparaître (doc, 2026-10-18) ; aucun événement ne part du client ni de Studio : vérifie sur l'expérience publiée.
- Erreurs : `ScriptContext.Error` des deux côtés, erreurs client remontées par un remote à débit limité ; après chaque release, Error Report du Creator Hub filtré sur les erreurs nouvelles.
- Trace d'onboarding par joueur (option) : déclenche-la sur un vrai marqueur de première visite, jamais sur « clé introuvable » (une lecture ratée n'est pas un nouveau joueur) ; durée bornée, une écriture, couverte par l'effacement des données (section 7).

### Expériences et configs

- **Experiments** (outil officiel) : 14 à 60 jours. Ne décide pas au premier jour, n'arrête pas dès qu'une courbe plaît, exige la significativité. Sous ~1 000 DAU, fie-toi à l'effet minimal détectable du dashboard ; au-delà de 100 %, la significativité est improbable. Une seule expérience de matchmaking à la fois, déployée à 100 %. Aucun autre changement pendant l'expérience.
- Une expérience répond à une question précise. Elle porte une hypothèse réfutable (« si on change Y, parce que la preuve Z le suggère, X doit monter sans nuire à G »), un contrôle, une variante, une métrique primaire, des garde-fous (pics d'erreurs, complétion mobile, perf, inflation), une durée, un pourcentage, un rollback. Le serveur décide la variante ; son id va dans les événements analytics.
- Valeurs réglables en direct : `ConfigService` et les Configs du Creator Hub (brouillon → publication), par exemple `ConfigService:GetConfigForPlayerAsync(player)` (serveur seulement, applique les Experiments du joueur ; fait un yield et lève une erreur si aucune config n'a jamais pu se charger : sous `pcall`). Le `ConfigSnapshot` ne change pas seul : écoute `UpdateAvailable`, puis `Refresh()` à un moment sûr (entre deux manches). Délai de propagation : donnees.md.
- Hiérarchie des preuves : Experiments > comparaison de cohortes contrôlée > funnels, erreurs, perf > playtests modérés, sondages > analyse de concurrents. Les niveaux bas donnent des hypothèses, pas des causes.
- Format d'un diagnostic : verdict (la contrainte au plus fort levier), preuves (source et dates), inconnues, hypothèses classées, plus petite expérience suivante, métriques et règle de décision, travail volontairement exclu.

## 6. Découverte et publication

### Comment Roblox recommande (doc officielle, relevée en 2026-07)

- Deux étapes. **Récupération** : un sous-ensemble personnalisé ; pubs, recherche, amis, partages et curation amènent les premiers joueurs. **Classement** : seuls comptent les joueurs arrivés par « Recommended for You ». L'acquisition externe donne une cohorte et des revenus, mais ne répare pas directement des signaux Home faibles.
- **Signaux les plus importants** (moyennes par joueur : un petit jeu n'est pas désavantagé) : play-through rate (joue après une impression : icône, miniature, titre) ; rebond à la première partie (départs avant 60 s et entre 61 et 180 s) ; jours de jeu par joueur (J1, J2–J7, J8–J28) ; temps de jeu par joueur (plafonné à 60 min par jour et par jeu). **Importants** : jours de co-jeu, sessions qualifiées, jours de dépense, Robux dépensés par joueur.
- Juin 2026 : l'ancien QPTR est scindé en play-through rate et rebond ; la fenêtre passe à 28 jours (le conseil « 7 jours » est périmé). Les miniatures clickbait sont démasquées par le rebond.
- Août 2026 : Roblox annonce que la mise à jour des recommandations valorise mieux la valeur à long terme (joueurs qui reviennent, achats jugés utiles). Rétention faible : corrige d'abord le gameplay (statut à revérifier).
- **Exposition réduite** pour : métadonnées centrées sur un giveaway, métadonnées qui ne correspondent pas au jeu, jeux non uniques dont métadonnées et fichiers de place ressemblent à d'autres. Parti d'un template ? Rends icône, miniatures, description et contenu visible uniques, et regarde la bannière d'exposition réduite du dashboard ; le statut est réévalué à chaque update.
- Mode Beta : l'expérience sort des recommandations Home, utile pour régler les stats sur du trafic pub. Avis partagés sur son effet.

### Page du jeu

- **Icône** carrée, maître en 1024 exportée en 512 au moins, lisible en tout petit et en cercle.
- **Miniatures** 16:9 en 1920 × 1080 ; garde l'essentiel hors des ~12 % du bas (compteur de joueurs). Active 2 à 5 miniatures fidèles avec la personnalisation des miniatures ; teste autour d'une grosse update, attends 7 jours avant de relancer un test, n'y touche pas pendant un pic.
- Ce qui marche (terrain) : une action ou un fantasme dominant ; fond simple ; 1–2 personnages (3 au plus) ; contraste d'échelle (un héros et quelque chose d'énorme) ; mouvement en diagonale ; couleurs vives (bleu ou sarcelle contre orange ressortent sur l'UI sombre ; le violet foncé s'y noie) ; pas de petit texte ; un texte qui tease une mécanique, ou aucun. **Ce que montre la miniature apparaît dès le join**, sinon le rebond coule le classement.
- **Titre** cherchable qui dit le genre. Ne le change que dans les ~2 premières semaines et en dernier recours, après les miniatures. Choisir un genre et renommer a fait passer un CTR de 1,4 % à 3,0 % (anecdote).
- **Description** : les premières lignes disent ce qu'on fait et pourquoi revenir. Notes de mise à jour visibles.
- **Badges** (affichés sur la page) : icône 512 × 512 recadrée en cercle, rien d'important près des bords ; 5 créations gratuites par jeu et par 24 h, puis 100 R$ chacune (doc, 2026-10-18). Premier lot : bienvenue, première action, premier boss, premier rebirth, objet rare, événement limité.
- Repères de CTR (terrain, sources divergentes) : sous ~2 %, aucune poussée ; 2,5–4 % sain ; 7–8,5 % élite ; plus de 10 % sur peu d'impressions ne veut rien dire. Le ciblage pèse plus que l'image.
- Pré-test bon marché : compose les candidates dans une fausse grille Home sombre, mélangées, et fais-les juger par un modèle de vision sous plusieurs personas (enfant de 9 ans sur téléphone, ado fan de simulator sur PC, parent). C'est un garde-fou, pas une donnée : le vrai chiffre est le play-through rate.
- Miniature générée par IA : pars d'une image de référence du genre, change des points ciblés, retouche visages et texte à la main ; les visages « IA » évidents font fuir. Teste par A/B.

### Publicité (Ads Manager)

- La pub achète du trafic froid qui nourrit surtout l'algorithme en données ; elle n'achète pas de place sur Home. Boucle : pub → données → pire stat → update qui l'améliore → nouvelle pub.
- Consensus de terrain : ~16 crédits par jour, en continu, sur 7 à 14 jours, une seule campagne ; 24–48 h d'apprentissage ; juge chaque créa après au moins 1 000 impressions ; 4 à 8 créas actives ; retire celles sous 2 % de CTR ; campagnes desktop et mobile séparées ; jamais de budget « lifetime » (crédits brûlés en ~1 h). Ads Manager fait tourner plusieurs miniatures comme créas d'une même campagne.
- Le temps de jeu des joueurs venus par pub vaut environ la moitié de l'organique : ne panique pas devant des stats de lancement faites par la pub.
- Quand le trafic Home dépasse le trafic pub, baisse le budget par paliers fixés d'avance plutôt que de couper net (les avis divergent).
- « La pub ne marche pas » : le problème est presque toujours la rétention, pas le canal.

### Lancement

- Canaux gratuits : clips de gameplay de 10–25 s avec l'accroche dans les 2 premières secondes, un par jour ; session de lancement avec 4–6 amis dans un même serveur ; groupe Roblox pour annoncer les updates ; Experience Events pour les événements programmés ; parrainage d'amis. Les Moments (clips verticaux avec un bouton qui téléporte dans le jeu) récompensent les pics « clippables » : garde le centre de l'image lisible.
- Promotion : un message à la fois, depuis ton propre compte établi, en respectant les règles de chaque communauté, ton « devlog » avec un GIF, jamais le même texte répété. Pas de multi-comptes ni de comptes achetés : les comptes neufs qui postent le même lien sont bannis.
- Semaine de lancement : 1) passe d'unicité et contrôle de l'exposition réduite ; 2) passe « 60 premières secondes » avec le funnel d'onboarding ; 3) raisons de revenir (quotidien, gains hors ligne, événements) ; 4) clips, événements programmés pour le week-end, journal de mise à jour en jeu ; 5) lire J1, session, rebond, play-through rate, corriger la pire chute, publier, recommencer. Publie des updates visibles tous les quelques jours : chacune peut relancer une vague de test.
- Ne crois pas au pic de lancement (400 joueurs simultanés peuvent retomber à 1 ou 2). La console est une niche bon marché aux sessions longues.
- Today's Picks (curation humaine par l'équipe éditoriale, candidature par formulaire à renvoyer après chaque update notable ; les critères qui suivent viennent d'un skill, la page officielle n'en liste aucun) : finition cohérente, UI intuitive, FTUE fluide, pas de bug majeur, bonnes données d'engagement, originalité. Une UI mobile cassée disqualifie d'office ; le jeu doit tourner sur téléphone, PC et console ; la localisation aide.

### Première publication (doc, 2026-10-18)

- Studio : File → Publish to Roblox, puis Name, Description, Creator (ton compte, ou un groupe, conseillé) et Devices, puis Create. L'expérience naît **privée** : seuls toi et les comptes avec accès Edit y jouent. Passes, produits, badges et DataStores réels n'existent qu'à partir de là (monetisation.md, donnees.md).
- Ouvrir : Creator Dashboard → le jeu → Configure → Settings → Audience → Private, Limited ou Public → Save Changes. Au plus 5 expériences jamais publiques passées en public par jour. Requis pour le 16+ : vérification d'âge, compte de plus de 2 jours en règle, questionnaire de maturité rempli ; pour tous âges, section 7.
- Autre place dans la même expérience : File → Publish to Roblox As… → l'expérience → Add as a new place → Create ; 100 Mo au plus par place. Avant d'ouvrir au public : porte de release (section 9) et oui explicite de l'utilisateur.

## 7. Conformité et design responsable

**Les règles changent chaque semaine.** Avant un avis définitif, vérifie les sources officielles du moment : Community Standards, pages de publication, de monétisation et de publicité du Creator Hub. Sans accès, présente ta réponse comme une revue de risques, pas un verdict.

### Âge et maturité

- Public (2026-08) : environ un tiers des joueurs quotidiens vérifiés a moins de 13 ans, et seuls ~45 % ont vérifié leur âge. Conçois pour des enfants par défaut.
- **Publication tous âges (comptes Kids et Select, depuis 2026-05)** : identité vérifiée, 2FA, questionnaire noté Minimal, Mild ou Moderate, et frais unique remboursable de 1 000 R$ par jeu ou Roblox Plus actif 2 mois de suite (la page « kids and select » cite aussi Premium). Un jeu tous âges n'est d'abord visible que des 16+ dont l'âge est contrôlé, jusqu'à 250 parties uniques de joueurs très engagés sur 60 jours (tableau Audience Reach). Remboursement : les pages se contredisent (90 jours après l'éligibilité, ou 100 joueurs très engagés gardés 60 jours sans modération) ; frais perdus si le jeu est retiré définitivement pour violation des Community Standards. Examen accéléré : 50 000 R$ remboursables (doc, 2026-10-18 ; l'annonce de juin disait 500 joueurs et 100 000 R$). Ces seuils bougent : relis les pages « kids and select » et « publish » avant un lancement.
- **Questionnaire de maturité** (Creator Dashboard → le jeu → Publish settings → Content rating) : remplis-le honnêtement avant le lancement. Labels (doc, 2026-10-18) : Minimal et Mild (Kids 5–8 et Select 9–15), Moderate (Select et 16+), Restricted (18+ vérifiés) ; « Content May Vary » réserve le jeu aux 16+ dont l'âge est contrôlé ; sans label, Roblox restreint le jeu pour tous les joueurs. Une réponse peut verrouiller un jeu en 16+ : « des PNJ qui poursuivent » a été classé violence. Traduis chaque mécanique en catégorie avant le lancement et règle les risques de contenu avant de pousser vers l'algorithme. Un jeu d'horreur déclare la peur et la violence.
- **IA générative visible des joueurs** (pas celle qui sert à construire) : le créateur répond de ce qu'elle produit ; filtre tout texte montré (avec un outil tiers, entrées et sorties passent par `TextChatService`) ; déclare-la au questionnaire ; un avis « tu parles à une IA » reste visible du début à la fin de l'échange. Une conversation sans limite de temps ou avec une mémoire entre sessions (interaction « étendue ») exige le label Restricted, donc 18+ seulement (doc generative-AI, 2026-10-18).

### Politiques par joueur

- `PolicyService:GetPolicyInfoForPlayerAsync(player)` donne des drapeaux PAR JOUEUR : deux joueurs du même serveur peuvent différer. Champs utiles : `ArePaidRandomItemsRestricted`, `IsPaidItemTradingAllowed`, `IsContentSharingAllowed`, `AreAdsAllowed`, `IsSubjectToChinaPolicies`, `IsEndlessContentLoadAllowed` (sinon pagination manuelle) et `IsEndlessContentAutoplayAllowed` (sinon lecture lancée par le joueur) ; `AllowedExternalLinkReferences` est un champ hérité qui renvoie toujours une liste vide (doc, 2026-10-18). Une fois par session, sous `pcall`, en cache (au-delà de 100 appels en attente, l'appel échoue), et le cas le plus restreint en cas d'échec.

```luau
--!strict
-- ModuleScript serveur : règles par joueur, prudentes en cas d'échec.
local Players = game:GetService("Players")
local PolicyService = game:GetService("PolicyService")

export type Policy = {
	paidRandomItemsRestricted: boolean,
	paidItemTradingAllowed: boolean,
	contentSharingAllowed: boolean,
}

-- Le joueur le plus restreint : utilisé tant que la lecture échoue.
local STRICTEST: Policy = { paidRandomItemsRestricted = true, paidItemTradingAllowed = false, contentSharingAllowed = false }

local cache: { [Player]: Policy } = {}

local function getPolicy(player: Player): Policy
	local cached = cache[player]
	if cached then
		return cached
	end
	local ok, info = pcall(function()
		return PolicyService:GetPolicyInfoForPlayerAsync(player)
	end)
	if not ok or info == nil then
		return STRICTEST -- pas mis en cache : le prochain appel réessaie
	end
	local policy: Policy = {
		paidRandomItemsRestricted = info.ArePaidRandomItemsRestricted ~= false, -- champ absent = restreint
		paidItemTradingAllowed = info.IsPaidItemTradingAllowed == true,
		contentSharingAllowed = info.IsContentSharingAllowed == true,
	}
	if player.Parent then -- parti pendant le yield : ne rien garder (fuite)
		cache[player] = policy
	end
	return policy
end

Players.PlayerRemoving:Connect(function(player)
	cache[player] = nil
end)

return getPolicy
```

- **Objets aléatoires payants** (œuf, caisse, gacha achetés en Robux ou avec une monnaie achetée) : toutes les issues et leurs chances AVANT l'achat. Un boost de chance ou une pitié vendus en sont aussi. Pour un joueur restreint, la divulgation ne suffit pas : chemin gratuit, séquence non aléatoire annoncée, achat direct garanti, offre masquée ou bloquée avec un message. « J'accepte le risque » ne lève jamais une politique obligatoire : le verdict reste NOT READY (monetisation.md).
- **Liens externes** : `AllowedExternalLinkReferences` renvoie toujours une liste vide, il ne dit donc plus quels réseaux afficher. N'affiche pas de lien hors plateforme en jeu, et ne récompense aucune action faite hors de Roblox (à vérifier selon les règles du moment). Les liens sociaux de la page du jeu (3 au plus) ne sont visibles que des utilisateurs 16+ dont l'âge est contrôlé, et seulement si le propriétaire l'est aussi (aide Roblox, 2026-10-18). Des liens hors plateforme dans les bios ont valu des avertissements.
- **Texte des joueurs** affiché à d'autres (noms de pets, panneaux, chat maison) : filtrage serveur par `TextService:FilterStringAsync`, puis `GetNonChatStringForBroadcastAsync` ou `GetNonChatStringForUserAsync`. Chat maison : `TextChatService` et `CanUserChatAsync` / `CanUsersDirectChatAsync`.
- Monétisation (2026) : pas de formulation d'achat agressive, pas de faux compte à rebours qui redémarre, pas de Robux comme récompense, plus de vente des passes d'un autre jeu depuis le 2026-05-30 (doc des passes), aucun commerce de Robux hors plateforme, aucun montant de Creator Rewards promis en jeu.

### Design non manipulateur

| À proscrire | À faire |
|---|---|
| Fausse urgence (compte à rebours qui redémarre, « dernière chance » inventée) | Annoncer les dates des événements et s'y tenir |
| Boost qui expire juste après la première victoire pour pousser l'achat | Laisser le jeu être amusant avant de demander de l'argent |
| Invite d'achat pendant un moment critique, dans les 30 premières secondes ou juste après un refus | Prix visible avant l'invite, un refus respecté |
| Bouton d'achat déguisé, monnaies empilées qui cachent le prix réel | Prix clair en Robux, lu au moment de l'affichage |
| Friction créée exprès pour vendre le remède | Payant = confort, cosmétique, accélération ; gratuit = tout le jeu de base |
| Puissance vendue en durée limitée, rareté coercitive | Cosmétique limité dans le temps, objets populaires parfois de retour |
| Chances cachées ou trompeuses | Chances publiées, compteur de pitié visible |
| Série qui punit un jour manqué | Série avec un jour pardonné |

### Données et propriété intellectuelle

- Aucune donnée personnelle dans les analytics ni les logs. Effacement des données (RGPD, droit à l'oubli) : clés qui contiennent le `UserId` (`player_{UserId}`, couvertes par les modèles de suppression), `UserIds` attachés aux écritures, suppression sous 30 jours après la demande, webhook « Right to Erasure Request » ; traces et exports analytics entrent dans le même flux (donnees.md).
- Suivre un joueur d'un jeu à l'autre sans son consentement est une violation.
- Tests avec des mineurs : consentement adapté, confidentialité, aucune donnée personnelle inutile.
- Aucun asset, musique, logo ou marque copiés. Un nom qui imite un jeu existant est déclassé ; un vocabulaire de jeu d'argent (« Jackpot », « Gambler ») a bloqué la revue d'une pub jusqu'au renommage.
- Modération : `Players:BanAsync` avec une échelle de sanctions réversibles d'abord, plusieurs seuils avant de bannir, un journal. Dès qu'il y a du social : signaler, rendre muet, expulser.

## 8. QA et playtests

Un playtest prouve le comportement ET le ressenti. Une vérification de types propre ou un test de logique pure est nécessaire, pas suffisant.

- **Environnements qui mentent** : Play Solo (un seul joueur : rien sur la propriété réseau ni l'ordre de réplication) ; contexte Edit (VM séparée) ; mode Play (les changements sont perdus : un correctif qui « marche » sans avoir été sauvé) ; machine de dev (FPS élevés, aucune latence). Dis lequel s'applique avant de prendre une observation Studio pour une preuve.
- **Multi-clients** : mode de test **Server & Clients** (jusqu'à 8 clients) pour tout ce qui touche remotes, réplication ou streaming. Team Test est une session partagée, pas un harnais multi-clients. Simule 100–200 ms de latence avec perte avant de livrer du code réseau.
- **Appareils** : Device Emulator de Studio, puis un vrai téléphone modeste en qualité graphique basse ; au moins 30 FPS sur un milieu de gamme (heuristique) ; console à distance de télé, avec navigation complète à la manette.
- **À tester en jeu publié**, sur une expérience de test : téléportations (impossibles dans Studio), achats (achat → `UserOwnsGamePassAsync` vrai sur un NOUVEAU serveur → rejoin), analytics et badges.
- Chemin malheureux : envoie des données invalides aux remotes depuis un script de test Studio. Jamais d'outil d'exploit, même sur ton jeu : ils violent les conditions d'utilisation.
- Commandes de triche protégées (Studio ou UserIds de dev) pour atteindre vite chaque état.
- Observe de vrais joueurs neufs sans les aider : au moins 5 pour l'onboarding. S'il faut expliquer, le design communique mal.
- Après chaque changement : aucune erreur et aucun nouvel avertissement dans la console (un avis de dépréciation est une panne future).

| Scénario | PC clavier-souris | Téléphone | Tablette | Console |
|---|---|---|---|---|
| Joueur neuf, aucune sauvegarde | ☐ | ☐ | ☐ | ☐ |
| Joueur qui revient, vieille sauvegarde | ☐ | ☐ | ☐ | ☐ |
| Départ pendant une sauvegarde ou un échange | ☐ | ☐ | | |
| Mort pendant une transaction ou une cinématique | ☐ | ☐ | | |
| Inventaire vide, pile max, spam d'un remote | ☐ | | | |
| Arrivée en cours de manche, réapparition rapide | ☐ | ☐ | | |
| FPS bas, 150 ms de latence | ☐ | ☐ | | ☐ |

- Rapport de bug : date, version de la place, testeur, appareil, étapes, attendu, obtenu, gravité.
- **Contrat de preuve** : chaque PASS cite un test, un log de playtest, une inspection MCP, une capture de profiler avec son scénario, une capture ou une vidéo, ou une relecture de l'état durable. Une console propre ne prouve ni l'intégrité des données, ni l'accessibilité, ni l'idempotence d'un achat. L'accord de l'utilisateur n'est pas une preuve.
- Avant livraison, retire les restes d'expérimentation : dossiers de debug, `print`, scripts dupliqués désactivés, assets provisoires.

## 9. Porte de release et rollback

### Avant de publier

- **Fige le paquet** : expérience, places, scripts, assets, config, schémas, produits, permissions, matrice d'appareils, compatibilité des données, procédure de retour. Teste le paquet exact qui sera publié.
- Classe chaque porte : Requise, Conditionnelle, Non applicable (avec raison), Non vérifiée. Ne transforme jamais N/A ou Non vérifiée en PASS, et aucun pourcentage de préparation ne cache un bloqueur : une note globale vaut celle de la pire dimension (des données notées 1/5 effacent la progression, quelle que soit la moyenne).
- **Groupes à haute conséquence** (un seul échec = NOT READY) : persistance (données existantes chargées sans être remplacées par des défauts, migrations rejouables, une session non acquise n'est pas un profil neuf) ; frontières de confiance (inventaire des remotes, tests de données malformées et répétées) ; monétisation (octrois idempotents par `PurchaseId`, `PurchaseGranted` après la sauvegarde) ; cloud et webhooks (aucun secret exposé, scopes minimaux).
- Verdict : READY, NOT READY, ou READY WITH ACCEPTED RISK (risque nommé, impact, responsable, rollback). Une politique obligatoire de la plateforme n'est jamais un risque acceptable.
- Audit de lancement en lecture seule, douze points : erreurs d'un vrai playtest de la version live ; sauvegardes (verrou, relances, `BindToClose`, migrations, test d'une vieille sauvegarde) ; monétisation ; remotes ; layout mobile et PC validé sur appareil (une capture de banc de test = non vérifié) ; minuteries et resets en heure serveur, fuseau dit correctement ; questionnaire de maturité ; page du jeu ; taille de serveur (réglée dans les paramètres du lieu, pas par script) ; accès (public, appareils, VR coupée si non testée, voix, console) ; performance ; badges et analytics.
- **Les betas de Studio ne sont pas actives en jeu publié** : du code qui en dépend doit être réécrit, même s'il marche en playtest.
- **Une place de staging dans la même expérience partage les DataStores du live**, et Studio avec l'accès aux API écrit sur les données réelles. Décide-le exprès ; préfère une expérience de test.
- Publier en live demande un oui explicite de l'utilisateur, à chaque fois.

### Rollback : le code et les données ne reviennent pas de la même façon

- **Rollback de code ≠ rollback de données.** Questions à trancher avant la sortie : vers quelle version ? Les anciens serveurs écriront-ils des données compatibles ? L'ancien code lit-il les nouvelles données ? Quelle migration est irréversible ? Comment dédommager les joueurs ? Désigne la cible, le déclencheur, le responsable, la procédure et les conditions d'abandon.
- **Versionne les données et migre au chargement**, sans jamais renommer le DataStore. Un ancien serveur qui reçoit une sauvegarde migrée par un serveur plus récent doit refuser de la charger (expulsion polie), pas l'écraser. Teste l'ancien lecteur sur des sauvegardes neuves avec des fixtures isolées.
- Migration risquée : staging d'abord, feature flag au déploiement, ancien chemin de lecture gardé au moins deux semaines, instantané des DataStores avant la publication (open-cloud.md).
- **Restaurer une version de place ne la publie pas** : cela crée une nouvelle version, que les joueurs ne voient qu'une fois publiée. L'autosave d'un collaborateur actif peut écraser ton retour arrière : coupe la collaboration avant.
- L'API de publication d'Open Cloud ne met pas à jour cinq types d'instances (`EditableImage`, `EditableMesh`, `PartOperation`, `SurfaceAppearance`, `BaseWrap`) : publie-les depuis Studio, sinon le déploiement « réussit » sans livrer ce travail.
- **Les serveurs existants gardent leur version.** Lis `game.PlaceVersion` et un marqueur de build au smoke test ; sépare ancienne et nouvelle version avant d'attribuer un effet à un correctif.
- Notes de version obligatoires : c'est le seul enregistrement durable du contenu d'une release. Écris-les pour la personne qui fera le prochain rollback.

### Arrêt propre et surveillance

- « Restart Servers » (Creator Hub → Configure → Server Management) : coche « Restart only servers with outdated versions », sauf si ta mise à jour ne passe pas par une nouvelle version de place ; sinon les serveurs déjà à jour redémarrent aussi. « Delay server restart » laisse 1 à 60 min aux joueurs pour partir (doc, 2026-10-18). `game.ServerRestartScheduled(restartTime, source, attributes)` (2026) prévient le serveur : avertis les joueurs, arrête les nouvelles manches, verrouille échanges et files, et vide les sauvegardes avant le `BindToClose`, dont le délai dur de 30 s est partagé : sauvegardes en parallèle (donnees.md). Sans arrêt propre, chaque update expulse les joueurs ; un jeu y a perdu son classement.
- Après la sortie, surveille : pics d'erreurs, échecs d'achat ou d'octroi, échecs de sauvegarde, chutes de funnel, inflation, signalements. Taux de crash client sous 2 % (enquête au-delà de 2–3 %) ; budgets de perf dans performance.md. Roblox juge une expérience sur le framerate, la mémoire et le temps de join. Puis décide : correctif, rollback ou poursuite.

## 10. Live ops

- **Quatre types d'updates** (officiel) : cadence de contenu (hebdo à mensuelle, sur les systèmes existants : événements limités, saisons) ; updates majeures (nouveaux systèmes : guildes, échanges, PvP ; des mois de travail) ; qualité de vie (UI, flux, accessibilité, perf : beaucoup de bonne volonté) ; bugs (par gravité, effort, joueurs touchés). Mélange les quatre.
- **Cadence durable** (officiel) : surtout du contenu artistique (objets, pets, véhicules, cartes, quêtes, variantes de couleur) et des thèmes saisonniers ; moins de 3 semaines d'effort par sortie ; une routine toutes les 2 semaines à 1 mois ; du contenu qui ne se consomme pas en une soirée.
- **Une update = une promesse au joueur = une métrique**, nommée avant le travail et relue après. Deux pistes : « patch » (bugs, exploits, réglages, quotidien si besoin) et « update » (hebdo ou bimensuelle). Pas d'update de dernière seconde ; les joueurs fidèles testent, ils ne font pas le design.
- Priorités : bugs critiques et exploits → rétention → expérience des nouveaux → contenu → monétisation. La monétisation ne rapporte que sur un jeu en bonne santé ; plaquée sur un jeu qui décline, elle accélère sa chute.
- **Événements pilotés par config** : id, début et fin en heure serveur, récompenses, lieux, tags analytics, drapeau de rollback ; aucune date codée en dur. Un seul ordonnanceur. Teste avant, pendant, après, l'arrivée tardive et le redémarrage du serveur. Un démarrage manuel par un admin empile les événements, sauf ceux qui partagent un même lieu. Le rare reste rare : n'élargis pas une liste de raretés sans l'accord du propriétaire.
- Événement saisonnier : préproduction 2–3 mois avant, production 1–2 mois, teasing 1–2 semaines, lancement suivi à l'heure, analyse après (les objets exclusifs restent dans les inventaires). Fenêtres : Halloween (octobre), hiver (décembre–janvier), Saint-Valentin (février), printemps (mars–avril), été (juin–août), rentrée (août–septembre). Suis les calendriers scolaires.
- Événement global sur tous les serveurs (double XP, boss) : `MessagingService` ne touche que les serveurs en ligne et ne persiste rien ; garde l'état de référence en MemoryStore ou DataStore et relis-le périodiquement (donnees.md).
- **Config en direct et feature flags** : `ConfigService` ou Configs pour changer une valeur sans republier. Précédence explicite (vrai, faux, absent, lecture ratée, cache, premier démarrage) : aucune fonction en attente ne s'active sur une lecture ratée.
- Classement hebdomadaire : change de clé d'`OrderedDataStore` à chaque période (pas d'effacement en masse) ; classements temporaires en MemoryStore.
- Calendrier, une ligne par update : nom, fenêtre, promesse, systèmes touchés, métrique, niveau de risque, rollback, plan de test, brouillon de notes.
- **Notes de patch** honnêtes et côté joueur (Ajouté, Modifié, Corrigé, Problèmes connus), sans promettre du contenu non sorti. En jeu : un journal de mises à jour et un court toast, pas une nouvelle popup.
- **Communauté** : trie les retours (bugs, demandes, équilibrage, ressenti sur la monétisation, confusion des nouveaux, contrôles mobiles, exploits, modération) par impact, clarté de la reproduction, fréquence, risque, urgence et effort. Sorties possibles : correctif immédiat, prochain patch, expérience, backlog, refus motivé. Reconnais les problèmes graves, ne donne une date que si elle est réelle, explique l'équilibrage en langage joueur, crédite la communauté.
- Créateurs de contenu : donne d'abord (codes, exclusivités) ; les chaînes de 10 000 à 100 000 abonnés convertissent mieux par vue ; révèle une chose à la fois. Le trafic YouTube gonfle la rétention (les spectateurs connaissent déjà le jeu) : les stats Home disent mieux la vraie croissance.
- Support joueur : « j'ai perdu mes objets à 14 h 20 » se traite avec l'historique de versions des DataStores (donnees.md) ; un remboursement notifié par webhook se retire des données (monetisation.md).

## 11. Localisation et accessibilité

### Localisation

- Langue : `player.LocaleId` (réglage du compte). Pays : `LocalizationService:GetCountryRegionForPlayerAsync(player)`, par géolocalisation IP, sous `pcall`, `"XX"` si inconnu ; faux derrière un VPN, donc jamais pour la sécurité ni pour des prix.
- Traduction automatique de l'UI : une `LocalizationTable` sous `LocalizationService` et `AutoLocalize` actif sur l'objet et ses ancêtres (GUI ; `ProximityPrompt` a son propre `AutoLocalize`). Flux : capture des textes dans Studio → export CSV → traduction → import → test avec l'émulateur de joueur (Test → Player : langue, et région, qui règle aussi les drapeaux `PolicyService`).
- Texte construit par code : `LocalizationService:GetTranslatorForPlayerAsync(player)` remplace la version sans `Async` (dépréciée) ; il fait un yield et peut lever une erreur : appelle-le sous `pcall`, hors du chemin critique, garde le traducteur en cache, puis `translator:FormatByKey("coins_count", { count })`. Une traduction absente retombe en silence sur le texte source : journalise-le en dev.
- Les chaînes traduites sont plus longues : teste la troncature. Les balises RichText sont retirées des traductions : réapplique-les. Ne construis pas de phrase par concaténation.
- Les prix s'affichent en Robux lus au moment de l'affichage (prix régionaux) : jamais de montant codé en dur dans un texte (monetisation.md).
- Langues fréquentes : en, pt-br, es, fr, de, it, ru, tr, id, th, vi, ja, ko, zh-cn, zh-tw.

### Accessibilité

- Cibles tactiles d'au moins 44 × 44 px ; navigation complète à la manette ; zones sûres respectées (ui-fondamentaux.md).
- L'état n'est jamais porté par la seule couleur : ajoute forme, icône ou texte.
- `GuiService.ReducedMotionEnabled` : coupe les mouvements (secousses, rebonds), garde les fondus. La secousse de caméra se désactive dans les options.
- Taille du texte : respecte `GuiService.PreferredTextSize` (réglage « Text Size » du joueur, `Enum.PreferredTextSize` de Medium à Largest ; un texte en `TextScaled` ne le suit pas) ; texte d'onboarding grand, blanc avec contour noir.
- Un indice sonore important a un équivalent visuel ; sous-titres pour les voix ; TTS possible pour le texte clé. Volume réglable par groupe, choix sauvegardé.
- Difficulté : indice discret après 3 échecs, aide plus directe après 5 ; modes ou difficulté adaptative pour un jeu grand public. Le joueur doit savoir ce qui l'a tué et quoi changer.

## 12. Pièges

- Inventer une métrique ou une cause ; juger un J7 avant 7 jours ou à faible CCU.
- Construire tout le jeu d'un coup ; cocher une tranche sans l'approbation de l'utilisateur.
- Tutoriel long, popup au spawn, boutique dans les 30 premières secondes, like demandé avant la première récompense.
- Noms d'événements à forte cardinalité (`EnemyKill_Sword`) : ils grillent la limite de 100 noms, au-delà de laquelle rien ne garantit que les événements comptent.
- Journaliser depuis le client, ou la tentative au lieu du succès.
- Miniature trompeuse ou template non retouché : le rebond coule le classement, l'exposition est réduite.
- Questionnaire de maturité bâclé : public restreint ou jeu verrouillé en 16+.
- Objets aléatoires payants sans divulgation ni `PolicyService`.
- Leviers de pression sur un public d'enfants : faux compte à rebours, boost qui expire, offres en rafale.
- Confondre rollback de code et rollback de données ; restaurer une version sans la publier.
- Staging ou Studio qui écrit sur les DataStores du live.
- Croire qu'un Play Solo prouve le multijoueur ; tester une téléportation dans Studio.
- Livrer du code qui dépend d'une beta de Studio.
- Publier sans arrêt propre : joueurs expulsés à chaque mise à jour.

## 13. Checklist

- [ ] Concept : boucle en une phrase, fantasme, 30 premières secondes écrites, vocabulaire fixé.
- [ ] PRD court et découpage en tranches validés ; tranche 1 = une action → une récompense → un objectif suivant.
- [ ] Onboarding : action avant 10 s, récompense avant 30 s, ni popup ni boutique avant, funnel par étape.
- [ ] Une raison de revenir demain visible dès J1 ; série avec un jour pardonné ; aucune impasse.
- [ ] Social : badges attribués par le serveur, invitation au bon moment, parrainage payé une fois par filleul, opt-in des notifications jamais obligatoire.
- [ ] Analytics serveur, sous `pcall`, noms stables, variantes en champs, limites respectées, vérifiées en jeu publié.
- [ ] Page du jeu : icône lisible en petit et en cercle, 2–5 miniatures fidèles, titre qui dit le genre, rien de commun avec un template.
- [ ] Questionnaire de maturité honnête ; éligibilité tous âges vérifiée sur la doc du moment.
- [ ] `PolicyService` lu par joueur, cas restreint en cas d'échec ; chances divulguées ; textes filtrés ; aucun lien hors plateforme en jeu.
- [ ] Aucun levier de pression envers les enfants ; seul le cosmétique est limité dans le temps.
- [ ] QA multi-clients avec latence simulée, appareils réels, matrice remplie, achats et téléportations testés en jeu publié.
- [ ] Portes classées et verdict écrit ; rollback de code ET de données préparé ; notes de version.
- [ ] Arrêt propre en place ; surveillance après sortie (erreurs, achats, sauvegardes, funnels).
- [ ] Live ops : calendrier, une métrique par update, événements pilotés par config, notes honnêtes.
- [ ] Localisation : tables, `FormatByKey`, troncature testée. Accessibilité : 44 px, manette, couleur doublée, mouvement réduit, son doublé.
