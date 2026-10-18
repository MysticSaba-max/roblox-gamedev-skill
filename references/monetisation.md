# Achats et économie

À lire quand : tu ajoutes un game pass, un developer product, un abonnement, une pub récompensée, une boutique, une monnaie de jeu, des objets aléatoires payants, ou quand tu fixes des prix et des récompenses.

## Sommaire

1. Choisir le produit
2. Règles d'or des achats
3. Game passes
4. Developer products : ProcessReceipt
5. BindReceiptHandler et transferts de Robux
6. Abonnements d'expérience
7. Premium, Roblox Plus, Creator Rewards
8. Pubs vidéo récompensées
9. PolicyService et objets aléatoires payants
10. Règles de la plateforme (2025-2026)
11. Prix, affichage et invites d'achat
12. Monnaie de jeu côté serveur
13. Économie : sources, puits, courbes, simulation
14. Mesurer l'économie
15. Tester les achats
16. Pièges
17. Checklist

## 1. Choisir le produit

| Besoin | Produit | Livraison |
|---|---|---|
| Avantage permanent dans l'expérience (VIP, x2, sac plus grand, zone) | Game pass | possession lue avec `UserOwnsGamePassAsync`, aucun reçu |
| Consommable rachetable (monnaie, potion, résurrection, skip, boost temporaire) | Developer product | `ProcessReceipt` ou `BindReceiptHandler` |
| Avantage récurrent tant que le joueur paie | Abonnement d'expérience | statut lu, **aucun reçu** |
| Pourboire, don entre joueurs | Robux Transfers | `BindReceiptHandler` |
| Accès ou serveur privé payant | réglages de l'expérience | contrôle d'accès côté serveur (`game.PrivateServerOwnerId ~= 0` : serveur privé acheté par un joueur, `gameplay.md`) |
| Revenu passif | Creator Rewards | aucune API |

- Le créateur livre l'avantage. Roblox ne gère que le paiement et la question « possède-t-il ? ».
- Ne vends pas le même avantage permanent en pass et en produit. Pas de pass pour un effet temporaire. Un pack de démarrage en developer product n'encombre pas la page boutique.
- Un battle pass saisonnier n'est pas un game pass unique (achat à vie) : un developer product par saison, enregistré dans le profil, ou un nouveau pass par saison.

## 2. Règles d'or des achats

1. **Le client propose, le serveur accorde.** L'invite part du client. L'effet ne vient que d'une vérification serveur. Un clic, `PromptProductPurchaseFinished` ou un remote « j'ai acheté X » ne prouvent rien : un tricheur les fabrique.
2. **Un seul `ProcessReceipt` dans tout le jeu.** C'est une propriété : une seconde affectation (autre script, modèle gratuit) remplace la première sans erreur, et des joueurs paient pour rien. Un module central route tous les produits.
3. **Idempotence par `PurchaseId`.** Roblox rejoue chaque reçu non résolu à chaque arrivée du joueur sur un serveur, et à chaque nouvel achat. Range le `PurchaseId` dans le même profil, la même clé et la même écriture que l'effet.
4. **`PurchaseGranted` seulement après une sauvegarde confirmée** de l'effet et du `PurchaseId`. Avant, un crash fait payer le joueur pour rien.
5. **`NotProcessedYet` dès que tu doutes** : joueur absent, profil pas chargé, produit inconnu, octroi en échec, sauvegarde non confirmée. Le reçu reste en attente et reviendra ; cette réponse ne le perd ni ne le rembourse.
6. **Un `pcall` réussi n'est pas un octroi réussi.** Il dit seulement « pas d'erreur ». Un handler qui renvoie un booléen se teste avec `ok and granted == true` ; dans tous les cas, l'acquittement attend la sauvegarde.
7. **Une lecture ratée (possession, abonnement, politique) est inconnue.** Ne la mets jamais en cache comme « non possédé », ne retire rien à cause d'elle.
8. **Prix lus à l'exécution**, jamais codés en dur : les prix régionaux changent le montant.
9. **Les politiques Roblox ne se négocient pas** : probabilités affichées, PolicyService vérifié, pas de vente croisée.

## 3. Game passes

- Création : Creator Hub → expérience publiée → Monetization → Passes. Icône de 512 × 512 au plus, en JPG, PNG ou BMP (recadrée en cercle : centre l'essentiel). Prix de 1 à 1 000 000 000 Robux. « Copy Asset ID » donne l'id.
- `MarketplaceService:UserOwnsGamePassAsync(player.UserId, passId)` au join, dans un `pcall` ; mets en cache les réponses *réussies* pour la session. `PromptGamePassPurchase(player, passId)` part du client.
- `MarketplaceService.PromptGamePassPurchaseFinished(player, passId, wasPurchased)` : connecte-le **une fois, côté serveur**. Il se déclenche aussi côté client, où il ne sert qu'à l'affichage. `wasPurchased = false` : annulation, Robux insuffisants, plus en vente, réseau. Le moteur met à jour le cache de `UserOwnsGamePassAsync` quand l'événement part.
- Revérifie à chaque arrivée : achat fait sur le site, sur un autre serveur, ou perte de données. Un achat fait hors du jeu pendant la session peut mettre plusieurs minutes à apparaître. La possession Roblox fait foi ; ton DataStore ne garde que les données propres au pass. Un pass remboursé est retiré par Roblox.
- `GetPassOwnershipAsync` existe (0.743) mais n'est pas activé : garde `UserOwnsGamePassAsync`. `PlayerOwnsAsset` est déprécié au profit de `PlayerOwnsAssetAsync` ; depuis début 2026, les inventaires sont privés par défaut, donc la réponse peut être `false` pour un objet possédé (pas pour les passes). Badges : `BadgeService:UserHasBadgeAsync(userId, badgeId)`.

Un seul module serveur tient le registre « pass → avantage ». Les systèmes lui demandent `Passes.has(player, "VIP")` au lieu d'écouter les achats chacun de leur côté.

```luau
--!strict
-- ServerScriptService/Passes.luau (ModuleScript) : seul propriétaire des avantages de pass.
local MarketplaceService = game:GetService("MarketplaceService")
local Players = game:GetService("Players")

local PASSES: { [number]: string } = { [123456789] = "VIP", [987654321] = "DoubleCoins" } -- id → avantage
local owned: { [Player]: { [string]: boolean } } = {}
local Passes = {}

local function grant(player: Player, perk: string)
	local perks = owned[player]
	if perks and not perks[perk] then
		perks[perk] = true
		player:SetAttribute(perk, true) -- réglage idempotent, jamais « ajoute 2 » à chaque appel
	end
end

local function check(player: Player)
	if player.Parent ~= Players then
		return
	end
	local perks = owned[player] or {}
	owned[player] = perks
	local failed = false
	for passId, perk in PASSES do
		if perks[perk] then
			continue -- déjà accordé : pas de requête inutile
		end
		local ok, has = pcall(function()
			return MarketplaceService:UserOwnsGamePassAsync(player.UserId, passId)
		end)
		if player.Parent ~= Players then
			return -- parti pendant la requête : on ignore la réponse
		end
		if ok and has then
			grant(player, perk)
		elseif not ok then
			failed = true -- inconnu, pas « non possédé »
		end
	end
	if failed then
		task.delay(30, check, player) -- on réessaie au lieu de rétrograder
	end
end

function Passes.has(player: Player, perk: string): boolean
	local perks = owned[player]
	return perks ~= nil and perks[perk] == true
end

function Passes.start()
	MarketplaceService.PromptGamePassPurchaseFinished:Connect(function(player, passId, wasPurchased)
		local perk = PASSES[passId]
		if wasPurchased and perk then
			grant(player, perk) -- effet immédiat, sans rejoin
		end
	end)
	Players.PlayerAdded:Connect(check)
	Players.PlayerRemoving:Connect(function(player)
		owned[player] = nil
	end)
	for _, player in Players:GetPlayers() do
		task.spawn(check, player)
	end
end

return Passes
```

- Les effets liés au personnage (vitesse, accessoire) se réappliquent dans `CharacterAdded` à partir de `Passes.has`, sans réinterroger le web. Compose les multiplicateurs depuis leurs sources (pass, boost, événement) au lieu de les empiler à chaque respawn. Remplacer un pass par un abonnement : les anciens acheteurs gardent l'avantage payé.

### Tag « [VIP] » dans le chat

Le serveur pose l'attribut (`grant` ci-dessus) ; chaque client décore les messages entrants. `TextChatService.OnIncomingMessage` est un callback **client**, défini une seule fois dans tout le jeu, qui ne doit jamais céder (pas de `IsInGroupAsync` ni de requête web dedans : lis un attribut posé au join). Il renvoie un `TextChatMessageProperties` ou `nil` (message inchangé). Le même motif sert pour un tag de groupe, avec `IsInGroupAsync` côté serveur au join (doc « Assign chat tags », 2026-10-18).

```luau
--!strict
-- LocalScript (StarterPlayerScripts) : préfixe pour les porteurs de l'attribut VIP.
local Players = game:GetService("Players")
local TextChatService = game:GetService("TextChatService")

TextChatService.OnIncomingMessage = function(message: TextChatMessage)
	local source = message.TextSource :: TextSource?
	if source == nil then
		return nil -- message système
	end
	local sender = Players:GetPlayerByUserId(source.UserId)
	if sender == nil or sender:GetAttribute("VIP") ~= true then
		return nil
	end
	local props = Instance.new("TextChatMessageProperties")
	props.PrefixText = `<font color="#FFD54A">[VIP]</font> {message.PrefixText}` -- balises RichText
	return props
end
```

## 4. Developer products : ProcessReceipt

Le reçu (`receiptInfo`) porte `PlayerId` (UserId), `ProductId`, `PurchaseId` (chaîne unique par achat), `CurrencySpent`, `CurrencyType`, `PlaceIdWherePurchased` et `ProductPurchaseChannel` (`InExperience`, `ExperienceDetailsPage`, `AdReward`, `CommerceProduct`). Roblox appelle le callback après l'achat, puis, tant que le reçu n'est pas acquitté, à chaque arrivée du joueur sur un serveur ou à son prochain achat de produit. Il n'y a pas de relance minutée : n'en invente pas. Le callback peut céder aussi longtemps que le serveur tourne (doc, 2026-10-18).

Un produit « Listed » se vend aussi hors du jeu (onglet Store de la page du jeu, accueil, recherche) : son reçu arrive quand le joueur entre dans le jeu (`ExperienceDetailsPage`). Roblox contrôle ton `ProcessReceipt` en continu ; s'il est défaillant, le produit quitte ces vitrines jusqu'à correction (doc, 2026-10-18).

Motif avec ProfileStore (module `PlayerProfiles` de `donnees.md`, section 6). L'effet et le `PurchaseId` vivent dans `profile.Data`, donc dans la même écriture. On acquitte quand `profile.LastSavedData` contient l'identifiant : c'est le motif de la page « Developer Products » de la doc ProfileStore (`LastSavedData`, `Save()`, `IsActive()`, 100 ids gardés ; doc, 2026-10-18).

```luau
--!strict
-- ServerScriptService/Receipts.luau (ModuleScript). Receipts.install() est appelé UNE fois.
local MarketplaceService = game:GetService("MarketplaceService")
local PlayerProfiles = require(script.Parent.PlayerProfiles)

local MAX_RECEIPTS = 100 -- derniers PurchaseId gardés dans le profil
local SAVE_TIMEOUT = 20

-- Octrois : mutation pure de profile.Data, sans yield ni échec à mi-chemin.
local PRODUCTS: { [number]: (data: { [string]: any }) -> () } = {
	[1111111] = function(data) data.coins += 1000 end,
	[2222222] = function(data) data.gems += 50 end,
}
local Receipts = {}

local function isSaved(profile: any, purchaseId: string): boolean
	local saved = profile.LastSavedData
	return type(saved) == "table" and type(saved.receiptIds) == "table"
		and table.find(saved.receiptIds, purchaseId) ~= nil
end

function Receipts.process(receipt: { [string]: any }): Enum.ProductPurchaseDecision
	local grant = PRODUCTS[receipt.ProductId]
	if grant == nil then
		warn(`[Receipts] produit inconnu {receipt.ProductId}`)
		return Enum.ProductPurchaseDecision.NotProcessedYet -- une future version le traitera
	end
	local profile = PlayerProfiles.get(receipt.PlayerId)
	if profile == nil then
		return Enum.ProductPurchaseDecision.NotProcessedYet -- absent ou pas chargé : Roblox rejouera
	end
	local purchaseId = tostring(receipt.PurchaseId)
	local ids = profile.Data.receiptIds
	if table.find(ids, purchaseId) == nil then
		grant(profile.Data) -- effet et PurchaseId changent ensemble, sans yield entre les deux
		table.insert(ids, purchaseId)
		while #ids > MAX_RECEIPTS do
			table.remove(ids, 1)
		end
	end
	-- Déjà dans la liste ne veut pas dire sauvegardé : on attend la preuve.
	if not isSaved(profile, purchaseId) then
		profile:Save()
		local deadline = os.clock() + SAVE_TIMEOUT
		while not isSaved(profile, purchaseId) and profile:IsActive() and os.clock() < deadline do
			task.wait(0.5) -- attente bornée ; le callback a le droit de céder
		end
	end
	if isSaved(profile, purchaseId) then
		return Enum.ProductPurchaseDecision.PurchaseGranted
	end
	return Enum.ProductPurchaseDecision.NotProcessedYet -- l'effet partira avec la prochaine sauvegarde
end

function Receipts.install()
	MarketplaceService.ProcessReceipt = function(receipt)
		local ok, decision = pcall(Receipts.process, receipt)
		if ok then
			return decision
		end
		warn(`[Receipts] {receipt.PurchaseId} : {decision}`)
		return Enum.ProductPurchaseDecision.NotProcessedYet -- une erreur ne doit jamais lâcher le reçu
	end
end

return Receipts
```

Pourquoi c'est sûr :
- Rejoué dans la même session : l'id est déjà dans la liste, aucun second octroi. Rejoué après un crash : soit la sauvegarde contenait effet **et** id (pas de second octroi), soit aucun des deux (octroi normal).
- Sauvegarde lente ou en échec : `NotProcessedYet`. L'effet reste en mémoire, part au prochain autosave avec son id, et le passage suivant acquitte.
- Session volée par un autre serveur : nous n'avons rien écrit, le nouveau serveur accorde à son tour. Garder 100 ids suffit : un reçu ne revient que tant qu'il n'est pas acquitté.

En roblox-ts, le motif est identique (`MarketplaceService.ProcessReceipt = (receipt) => …`) ; seul `pcall` change de forme : `const [ok, decision] = pcall(() => process(receipt))`.

Avec le profil fait main de `donnees.md` (section 7), même logique : effet et id dans `session.data`, puis `PurchaseGranted` seulement si `write(session, false) == "committed"`. Cette écriture part *après* la mutation : son succès prouve que l'effet et l'id sont durables ensemble.

- Jamais d'historique des reçus dans un store séparé de l'effet : l'un peut réussir et l'autre échouer. Et `GetAsync` puis `SetAsync` sur un tel store n'est pas atomique. Tout ce qui sauvegarde le profil passe par le même module : un autosave en `SetAsync` à part effacerait la liste des reçus.
- Un reçu qui arrive pendant un cooldown est accordé quand même, puis le cooldown repart : ne « mange » jamais un achat. Effet différé (boost qui démarre plus tard) : modélise les états réservé, démarré, livré, annulé dans le profil, pour qu'un rejeu retrouve la même opération.
- Échange ou cadeau entre deux joueurs : un reçu atomique sur un profil ne rend pas l'échange atomique. Journal de transfert (`donnees.md`, section 13).
- Journalise `PlayerId`, `ProductId`, `PurchaseId`, `CurrencySpent` : support (« j'ai payé, rien reçu »), conversion, fraude. Dans un conteneur à capacités (sandbox), assigner `ProcessReceipt` demande la capacité `Monetization`.
- Remboursement : le webhook Open Cloud « Transaction Refunded » (récent ; doc, 2026-10-18) couvre produits, passes, serveurs privés et accès payant, pas les abonnements (qui ont leur propre « Subscription Refunded »). L'effet d'un developer product n'existe que dans tes données : retire-le toi-même (`open-cloud.md`).

## 5. BindReceiptHandler et transferts de Robux

- `MarketplaceService:BindReceiptHandler(Enum.ReceiptType.DeveloperProduct, handler, filter?)` renvoie une `RBXScriptConnection`. Le handler renvoie `Enum.ReceiptDecision.Processed` ou `Enum.ReceiptDecision.NotProcessedYet`. La référence officielle qualifie `ProcessReceipt` de « legacy » (lue le 2026-10-18) ; la signature figure dans les définitions de septembre 2026.
- Routage : un handler filtré prend ses ids de produits, un handler sans filtre prend le reste, un handler lié passe avant `ProcessReceipt`, et un reçu non routé retombe sur `ProcessReceipt`. Deux liaisons pour le même produit, ou deux fourre-tout, échouent. Les deux voies peuvent coexister pendant une migration : garde un routage explicite.
- Mêmes règles : idempotence par `PurchaseId`, `Processed` seulement après une écriture durable. Garde la connexion ; ne déconnecte pas tant que des reçus attendent. Filtre : un tableau d'ids de produits (`{ COINS_ID, GEMS_ID }`), refusé pour les types de transfert. Champs du reçu : `PurchaseId`, `PlayerId`, `PlaceIdWherePurchased`, `ReceiptType`, `ProductId` et `CurrencyType` (produits seulement), `CurrencySpent`, `TransferRequestId` (transferts) (doc, 2026-10-18). `ProcessReceipt` reste la voie la plus éprouvée pour les produits.

Transferts de Robux (mai 2026, voir section 10) :
- `MarketplaceService:PromptRobuxTransferAsync(sender, receiverUserId, amount)`, **serveur uniquement**, renvoie un `transferRequestId` (chaîne) ; Roblox affiche la confirmation et vérifie l'éligibilité. Valide d'abord : l'émetteur est le joueur qui demande, le destinataire existe et diffère, le montant est un entier de 10 à 500, la fréquence des invites est bornée.
- Reçus : un `BindReceiptHandler` pour `Enum.ReceiptType.RobuxTransferSender`, un autre pour `RobuxTransferReceiver`. Corrèle avec `TransferRequestId`, dédoublonne par `PurchaseId` en distinguant côté et utilisateur. Les reçus arrivent sur le serveur où se trouve chaque joueur, ou à sa prochaine arrivée. `CanUsersTransferRobuxAsync` (0.742) n'est pas activé.
- Jeu de dons : risque d'abus élevé (comptes alternatifs, rate limits, modération). Accorder un avantage après un transfert réussi est permis, une fois son reçu traité (doc, 2026-10-18). Ne promets jamais des Robux en récompense.

## 6. Abonnements d'expérience

Ils ne passent **jamais** par `ProcessReceipt`. On lit un statut.

- `MarketplaceService:GetUserSubscriptionStatusAsync(player, subscriptionId)`, serveur, dans un `pcall`. Renvoie `IsSubscribed` (droit actif) et `IsRenewing`. Les ids sont des chaînes `"EXP-…"`.
- `Players.UserSubscriptionStatusChanged(player, subscriptionId)` : serveur seulement, sans le statut, qu'il faut relire. `MarketplaceService.SubscriptionStatusChanged` n'existe pas, et `MarketplaceService.UserSubscriptionStatusChanged` est interne.
- **Marche à suivre** : au `PlayerAdded` et à chaque `UserSubscriptionStatusChanged` pour ton id, lis le statut dans un `pcall`. Succès : accorde ou retire selon `status.IsSubscribed == true`. Échec : ne touche à rien (inconnu), et relis plus tard.
- `MarketplaceService:PromptSubscriptionPurchase(player, subscriptionId)` depuis le client. `PromptSubscriptionPurchaseFinished(player, id, didTryPurchasing)` signale une tentative, pas un succès : relis le statut. L'événement arrive après une nouvelle vérification faite à la fermeture de l'invite ; un joueur déjà abonné hors du jeu doit voir l'invite (qui le dit abonné) pour que l'événement parte.
- `IsSubscribed` vaut aussi `false` pendant le traitement d'un premier paiement, ou après son échec : ce n'est pas une résiliation. Affiche « en attente » et réconcilie via l'événement et des relectures bornées. Contrairement aux passes, **retire** l'avantage quand l'abonnement s'arrête ; une résiliation garde l'accès jusqu'à la fin de la période payée.
- Monnaie versée par période payée : registre de dédoublonnage par période dans le profil (avec `GetUserSubscriptionPaymentHistoryAsync`), jamais un octroi à chaque lecture du statut. Autres API : `GetUserSubscriptionDetailsAsync`, `GetSubscriptionProductInfoAsync` (client seulement : prix localisé), `PromptCancelSubscription`.
- Règles (doc, 2026-10-18) : jusqu'à 50 abonnements par expérience (actifs et inactifs), un seul niveau chacun (plusieurs abonnements cumulables), prix en Robux d'au moins 49 Robux et modifiable une fois tous les 60 jours (hausse annoncée au moins 30 jours avant), pas de vente par une autre expérience. Supprimer un abonnement actif rembourse tous les abonnés : retire-le d'abord de la vente et laisse les périodes finir.

## 7. Premium, Roblox Plus, Creator Rewards

- Premium : `player.MembershipType == Enum.MembershipType.Premium` ; les changements en cours de partie arrivent par `Players.PlayerMembershipChanged(player)`. `PromptPremiumPurchase` est déprécié, remplacé par `PromptRobloxSubscriptionPurchase`, qui propose Roblox Plus (définitions de septembre 2026 et doc, 2026-10-18).
- Avantages Premium modestes et qui font jouer plus : +10 à 20 % de monnaie, cosmétiques, bonus quotidien. Un bonus « quotidien » sans date sauvegardée se farme en se reconnectant : garde `lastPremiumBonusDay` dans le profil.
- Roblox Plus (lancé le 2026-04-30) : abonnement requis pour envoyer des Robux ; serveurs privés payants gratuits ; remise de 10 %, puis 20 % dès le 3e mois, payée par Roblox (ta part ne baisse pas). Il ne remplace pas Premium. À ne pas confondre avec l'extension de navigateur « Roblox+ ».
- Gains Roblox Plus (doc, 2026-10-18) : jusqu'à 750 Robux par nouvel abonné venu par `PromptRobloxSubscriptionPurchase` dans ton jeu (250 par mois les 3 premiers mois payés, rétention de 60 jours), et jusqu'à 100 Robux par abonné qui passe 60 min par mois dans tes serveurs privés payants. Avant une récompense, vérifie `player.HasRobloxSubscription` côté serveur, pas `PromptRobloxSubscriptionPurchaseFinished`.
- Creator Rewards remplace les Premium Payouts et le programme d'affiliation depuis le 2025-07-24. Volets documentés (doc, 2026-10-18) : 5 Robux par jour et par « Active Spender » qui passe au moins 10 min dans l'expérience, si elle fait partie des 3 premières qu'il lance ce jour-là ; 35 % des 100 premiers dollars d'achats d'un utilisateur nouveau ou réactivé qu'on t'attribue, pendant 60 jours. Aucune API : n'en fais jamais la source de vérité d'un droit. N'automatise pas l'engagement, pas de boucle AFK fabriquée, pas de comptes alternatifs.

## 8. Pubs vidéo récompensées

Tout passe par `AdService` (API des définitions de septembre 2026) :
- `AdService:GetAdAvailabilityNowAsync(Enum.AdFormat.RewardedVideo)`, **client uniquement**, dans un `pcall`, au plus près de l'affichage (à l'ouverture de la boutique), renvoie une table `{ AdAvailabilityResult = Enum.AdAvailabilityResult }` (`IsAvailable`, `NoFill`, `PlayerIneligible`, `DeviceIneligible`…).
- Le client demande ensuite la pub par un remote limité en débit. Le serveur appelle `AdService:CreateAdRewardFromDevProductId(productId)`, qui lie la récompense à un developer product, puis `AdService:ShowRewardedVideoAdAsync(player, reward, placementId?)`, **serveur uniquement**, qui montre la pub et renvoie un `Enum.ShowAdResult` (`ShowCompleted`, `ShowInterrupted`, `AdNotReady`…) (doc, 2026-10-18).
- La récompense arrive comme un reçu du developer product (`ProductPurchaseChannel = AdReward`), donc par le même `ProcessReceipt` idempotent (doc AdService, 2026-10-18). N'accorde rien sur la foi de `ShowCompleted` (la doc dit de ne pas s'en servir pour accorder) ou d'un remote « j'ai vu la pub » : ce serait une fontaine à monnaie pour tricheurs.
- Éligibilité (doc, 2026-10-18) : expérience publique et sans restriction, 2 000 visiteurs uniques par mois en moyenne, créateur de 13 ans ou plus. Pour le joueur et l'appareil, `GetAdAvailabilityNowAsync` tranche (`AreAdsAllowed` vise les pubs immersives). N'affiche le bouton que si une pub est disponible, et relis la disponibilité avant chaque nouvelle pub.
- Volontaire, jamais obligatoire ni en pleine action : entre deux manches, au lobby, après une mort (résurrection), pour doubler le bonus quotidien. Récompense conseillée : l'équivalent de 3 à 10 Robux, jamais un objet aléatoire (doc). Une pub ne doit pas valoir un produit payant, sinon plus personne ne l'achète.
- Le callback `AdGui.OnAdEvent` est déprécié et n'est plus jamais appelé (version 734, août 2026). Les pubs immersives changent souvent d'éligibilité : vérifie la doc du moment.

## 9. PolicyService et objets aléatoires payants

`PolicyService:GetPolicyInfoForPlayerAsync(player)` cède (`pcall`). Appel côté serveur, ou côté client pour le `LocalPlayer` seulement. Champs utiles :

| Champ | Contrôle |
|---|---|
| `ArePaidRandomItemsRestricted` | objets aléatoires payants interdits pour ce joueur |
| `IsPaidItemTradingAllowed` | échange d'objets achetés avec de la monnaie payante |
| `AreAdsAllowed` | pubs immersives (dans le monde) |
| `AllowedExternalLinkReferences` | champ historique, toujours vide (doc, 2026-10-18) : ne t'en sers plus |
| `IsSubjectToChinaPolicies` | règles propres à la Chine |
| `IsEligibleToPurchaseSubscription`, `IsEligibleToPurchaseCommerceProduct` | éligibilité d'achat (ce sont des champs, pas des méthodes) |

- Filtre **par joueur** : deux joueurs d'un même serveur peuvent avoir des drapeaux différents. Un appel par session, mis en cache ; plus de 100 appels en vol déclenchent une erreur.
- Échec de lecture : traite le joueur comme le plus restreint, pour la seule fonction concernée. Ne crois jamais un drapeau envoyé par le client. Le pays (`LocalizationService:GetCountryRegionForPlayerAsync`) ne donne pas les restrictions, et les prix par pays passent par les outils Roblox, pas par ton code.

Code du module de lecture (cache par joueur, cas le plus restreint en cas d'échec) : `getPolicy` dans `design-liveops.md`, section 7. Pour un objet aléatoire payant, ne vends que si `getPolicy(player).paidRandomItemsRestricted == false`.

**Objets aléatoires payants** (caisses, œufs, gacha, tirages d'événement) :
- La règle s'applique dès que des Robux achètent le tirage directement **ou** financent la monnaie qui sert à tirer ; un boost de chance payant et une pitié payante en font partie. Montre toutes les issues et leurs probabilités chiffrées **avant** l'achat, partout où l'offre apparaît : une couleur ou un mot de rareté ne suffisent pas.
- Si `ArePaidRandomItemsRestricted` est vrai, l'affichage des chances ne suffit pas. Choix possibles : chemin gratuit pour gagner l'objet, séquence non aléatoire annoncée, achat direct d'une issue précise, fonction masquée, achat bloqué avec un message, joueur écarté de cette partie du jeu. Présente les options et leurs compromis à l'utilisateur ; ne prends pas d'office la plus brutale.
- Tables de butin côté serveur seulement. Tire, accorde et sauvegarde **avant** d'annoncer le résultat : sinon on se déconnecte pour retenter. L'animation ne décide jamais du résultat. Limite le débit des tirages.
- Pitié : compteur affiché (« 34 sur 50 »), un par rareté, jamais remis à zéro en douce. Vérifie que les chances affichées sont les chances réelles : des poids appliqués par objet au lieu de par rareté faussent l'affichage.
- Construis cette branche dès le premier jour : la greffer après coup coûte bien plus. Demande contraire à la politique : ne l'implémente pas, rappelle une fois les options conformes, continue le reste. « J'accepte le risque » ne lève pas une règle obligatoire.

## 10. Règles de la plateforme (2025-2026)

| Date | Changement |
|---|---|
| 2025-07-24 | Creator Rewards remplace les Premium Payouts |
| 2025-09-05 | taux DevEx standard porté à 0,0038 $ par Robux gagné, contre 0,0035 $ avant ; les Robux gagnés avant cette date partent encore à 0,0035 $ (doc, 2026-10-18) |
| 2025-10-01 | developer products : prix régionaux en option, produit par produit |
| 2026-04-10 | l'ancien endpoint web d'achat de produits (`economy.roblox.com/v1/purchases/products`) est supprimé : vends via les invites de `MarketplaceService` |
| 2026-04-23 | anciens endpoints web de gestion des passes et produits (sous `apis.roblox.com/game-passes/v1` et `developer-products/v1`) dépréciés : endpoints Open Cloud (`…/v1/universes/{universeId}/game-passes`…) |
| 2026-05 | API de transferts de Robux : 10 % pour l'expérience, 90 % pour le destinataire, 0 % pour Roblox, définitifs |
| 2026-05-19 | publication tous âges : frais unique de 1 000 Robux par jeu, remboursable, ou abonnement Plus ou Premium actif (voir `design-liveops.md`) |
| 2026-05-30 | fin des ventes croisées (doc des passes, 2026-10-18) : interdit de vendre les passes ou produits d'une autre expérience ; les stands façon « PLS DONATE » passent à leurs propres passes ou aux transferts |
| 2026-06-15 | Managed Pricing : prix régionaux et optimisation des prix réunis |

- Transferts de Robux : l'émetteur a Roblox Plus. 10 à 500 Robux par transaction (doc, 2026-10-18). Plafonds de 500 par jour et 1 000 par mois, vérification d'âge à 18 ans et plus, et en dessous compte parent lié et accord du parent à chaque transfert, des deux côtés (à vérifier : la doc dit seulement « restrictions d'âge et consentement parental », et des sources tierces donnent d'autres seuils). DevEx : les Robux reçus par transfert ne sont pas éligibles ; la part de 10 % gagnée par l'expérience l'est (doc, 2026-10-18).
- Interdit : vendre ou échanger des Robux hors plateforme, payer en Robux directs comme récompense, formulations d'achat agressives, faux comptes à rebours, fausse rareté (« plus que 3 ! » sur un stock illimité), descriptions trompeuses (« 2x Everything » qui ne double que les pièces). Permis : passes et produits de **ton** jeu, pourboires via les transferts, système d'échange officiel (13 ans et plus, objets), monnaie partagée entre tes propres jeux via DataStores, abonnements.
- Commission : le créateur touche 70 % des Robux dépensés en passes et produits (un pass à 799 Robux rapporte environ 559 Robux). DevEx : seuls les Robux gagnés se convertissent ; seuil, âge et vérifications sur la page DevEx du moment (au 2026-10-18 : 30 000 Robux gagnés, 13 ans ou plus, e-mail vérifié, formulaire fiscal). Le 1er novembre 2026, le paiement DevEx devient une redevance. Renseigne tes informations fiscales au plus tard le 31 octobre 2026 (Creator Hub → Finances → Taxes) : sinon une retenue américaine de 24 % peut frapper tout le paiement (doc, 2026-10-18).
- Managed Pricing : test de prix au moins tous les 90 jours, opt-in ou opt-out par article, nouveaux articles inscrits par défaut. L'optimisation demande en général au moins 60 000 transactions sur les 30 derniers jours (doc, 2026-10-18) : un petit jeu fait plutôt des tests ciblés.
- Pool promotionnel de la page « acheter des Robux » : Roblox y offre gratuitement ton pass à des acheteurs de Robux. Option « Include this item in the bonus pool » ; pass en vente entre 50 et 800 Robux (ou hors vente), miniature obligatoire, aucun objet aléatoire payant (doc des passes, 2026-10-18).

## 11. Prix, affichage et invites d'achat

**Lis les prix à l'exécution**, dans un script client : c'est là que le joueur voit son prix régional. Ce prix sert à l'affichage, jamais à une décision du serveur. Les passes, objets et produits inscrits peuvent coûter jusqu'à 70 % de moins selon la région, jamais plus (doc, 2026-10-18).
- `MarketplaceService:GetProductInfoAsync(id, Enum.InfoType.GamePass)` (ou `Enum.InfoType.Product` pour un developer product) renvoie `PriceInRobux` et `IsForSale`. `GetProductInfo` sans `Async` est déprécié. Ne confonds pas les deux `InfoType`.
- Échec de lecture : « prix indisponible », jamais un montant inventé (bug réel : « 9 R$ affiché, 99 R$ facturé »). Un badge « -50 % » ou un prix barré ne change rien au prix facturé.
- Objets échangeables ou offerts : les écarts régionaux créent un risque d'arbitrage ; `MarketplaceService:GetUsersPriceLevelsAsync(userIds)` (niveau de 1 à 1 000, 1 000 = prix mondial plein) informe ta politique de transfert : appelle-le à chaque arrivée, jamais avec le résultat d'une session précédente (doc, 2026-10-18). `RecommendTopProductsAsync({ Enum.InfoType.GamePass, Enum.InfoType.Product })` et `RankProductsAsync` (50 produits max) classent tes produits par joueur : API récente, un appel par arrivée, mesure l'effet.
- Les prix appartiennent au propriétaire du jeu. Propose un tableau (ancien prix, nouveau, raison), attends le oui, applique (Open Cloud, voir `open-cloud.md`), puis vérifie que les libellés en jeu suivent. Ne dépense jamais d'argent (pubs, boosts) sans feu vert.

**Échelle de prix**, point de départ à tester, pas une règle :

| Bande | Usage |
|---|---|
| 25 à 99 R$ | premier achat, petits consommables, skips |
| 199 à 399 R$ | confort : auto-collecte, stockage, vitesse |
| 499 à 799 R$ | chance et puissance, souvent la bande qui rapporte le plus en simulateur ; gare au pay-to-win |
| 999 R$ et plus | identité : VIP, zones et cosmétiques exclusifs ; prix rond, description simple |

- Les retours terrain se contredisent : un produit d'appel très bas gonfle la conversion mais peut effondrer le panier moyen. Ajoute des articles chers et visibles (ancrage), un pack « pro » juste après le premier achat, et mesure. Les consommables bon marché et répétables rapportent souvent plus que les passes, surtout ceux qui soulagent une vraie friction.
- Packs de monnaie : prix unitaire décroissant avec la taille, et une étiquette « meilleure valeur » vraie (une étiquette fausse est trompeuse). Un montant fixe devient dérisoire avec la progression : indexe le contenu sur le revenu du joueur (« 1 heure de gains »), et dis-le dans la description.
- Vends du temps, de la chance, de l'expression, de l'accès. Jamais la victoire brute, rien qui casse le PvP. Pas de classement « Robux dépensés ». Un objet payé ne devient jamais pire qu'un objet gratuit : applique le meilleur des deux.

**Invites d'achat** :
- Depuis le client : `PromptProductPurchase(player, productId)`, `PromptGamePassPurchase`, `PromptSubscriptionPurchase`. Une invite lancée par le serveur suit un remote validé et limité en débit.
- Pas d'offre pendant les deux premières minutes. Fais d'abord sentir le manque, propose ensuite : après le premier œuf, devant la porte de la zone 2, après la première renaissance, à l'écran de mort.
- Boutique toujours accessible (icône du HUD, bouton « + » à côté de la monnaie), jamais ouverte de force ; boutons d'achat loin des zones de tap accidentel. Après un refus, attends au moins 60 s avant de reproposer ; pas d'invite en pleine action. Un bouton « offrir » à côté d'un bouton Robux vend un achat de plus ; la livraison à l'ami passe par le journal de transfert (`donnees.md`, section 13).
- Monnaie de jeu insuffisante : d'abord une erreur douce (texte rouge, son), la boutique seulement après plusieurs échecs, plafonnée par session. Valeur visible et exacte avant l'invite : nom, effet, durée, prix.
- Éthique : environ un tiers des joueurs quotidiens dont l'âge est vérifié ont moins de 13 ans (`design-liveops.md`). Fabriquer une frustration pour vendre le remède, pousser par la peur de rater ou cibler les gros payeurs fait fuir les joueurs et expose aux règles. Test simple : le jeu serait-il amusant si rien n'était à vendre ?

## 12. Monnaie de jeu côté serveur

Un achat en monnaie de jeu ne passe pas par un reçu : c'est un remote validé. Le client envoie l'id de l'objet, jamais un prix ni un montant.

Le handler complet (`BuyItem`, avec `RateLimit`, `Validate` et `Economy.trySpend`) est dans `securite.md`, section 3. Avec ProfileStore, la mutation porte sur `profile.Data` (`PlayerProfiles.get`, `donnees.md`, section 6), sans yield entre le contrôle du solde et le débit ; journalise ensuite le puits par `LogEconomyEvent` avec le solde commité (section 14), dans un `pcall` : la télémétrie ne doit jamais casser un achat.

- Ordre des contrôles : débit, type et longueur, objet connu, profil chargé, déjà possédé, fonds, puis mutation. Ajoute la distance pour une boutique physique (`securite.md`). Si un yield sépare le contrôle du débit, revalide après, ou verrouille les opérations économiques du joueur.
- `leaderstats` et attributs ne sont que des répliques d'affichage, mises à jour après la mutation qui fait autorité. Prix, formules et taux de drop restent sur le serveur.
- Valide chaque montant : entier, fini, positif, borné. Une quantité négative ou NaN dans un achat ou une vente crée de l'argent à volonté.

## 13. Économie : sources, puits, courbes, simulation

Dresse la carte avant de régler un chiffre. **Sources** : récompenses de boucle, quêtes, quotidiens, événements, récompenses sociales, achats. **Puits** : améliorations, craft, cosmétiques, rerolls, prestige, frais d'entrée, réparations, revente à perte (revendre à 40 % du prix retire 60 % de la valeur). **Multiplicateurs** : boosts, VIP, passes, événements, bonus de groupe. **Portes** : zones, niveaux, équipement, quêtes, minuteries.

- Ajoute assez de puits **avant** une grosse source. Sources trop fortes : inflation, l'argent ne veut plus rien dire. Puits trop forts : goulots, progression impossible sans payer, départs.
- Fixe une fois comment les multiplicateurs se combinent (addition ou produit). Des multiplicateurs qui s'empilent sans plafond détruisent les objectifs de long terme.
- Le revenu par minute monte avec la progression. Fais suivre les prix, ou ajoute une monnaie de prestige qui remet l'économie à zéro. Une récompense fixe de 5 000 est énorme au début et nulle à la huitième renaissance : indexe-la sur le revenu, par exemple `math.max(base * rebirthMult, incomePerSecond * 30)`.
- Courbe : victoires rapides au début (première amélioration en 1,5 à 3 min), choix visibles au milieu, prestige, collections et rareté à la fin. Place des déblocages forts avant le mur de contenu des 2 à 3 premières heures. Beaucoup de petites renaissances valent mieux que quelques murs énormes ; garde les déblocages des joueurs existants quand l'échelle change.
- Tous les nombres dans des tables de config ou des attributs, une seule fonction par formule (prix de vente, récompense), utilisée par tous les chemins et tous les affichages.
- Trois monnaies au plus : douce (gagnée en jouant), dure (rare, souvent achetée), sociale ou d'énergie. Les échanges entre joueurs sont la plus grosse source d'exploits et d'inflation (`gameplay.md`). Une récompense d'événement ne doit pas exposer une boucle de gain qui casse les prix. Au-delà de 2^53 : mantisse + exposant (`donnees.md`).

**Simule avant de publier** tout changement de prix, de multiplicateur, de poids de drop ou de palier :
1. Modèle tiré des scripts et des attributs en place, pas de mémoire, hypothèses écrites (rythme humain, durée d'action, taille de serveur).
2. Bot à politique gloutonne raisonnable (acheter l'amélioration abordable la moins chère, renaître quand c'est rentable).
3. Graines : 20 pour un diff rapide, 100 pour un rapport. Profils malchanceux, moyen, chanceux, plus un payeur. Session de 6 h.
4. Mesure le temps médian de chaque jalon, la part des graines qui l'atteignent, et « coût / revenu par seconde » à chaque achat (révèle murs et zones mortes).
5. Compare ancien et nouveau sur les mêmes graines. Présente un tableau avant/après avec le réglage de config de chaque recommandation, puis attends la décision du propriétaire.

Les vrais joueurs vont environ 1,5 à 2 fois moins vite que le bot. Dis ce que le modèle ignore (gains hors ligne, sac plein). Les limites du propriétaire sont des ordres, par exemple « aucun jalon plus de 30 % plus rapide ». Garde le rythme du palier le plus rare inchangé ou plus lent.

```luau
-- Lune : temps d'attente avant chaque amélioration, politique « acheter dès que possible ».
-- Une attente qui bondit d'une ligne à l'autre signale un mur.
local upgrades = { -- coût, multiplicateur de revenu après l'achat
	{ cost = 100, mult = 1.5 }, { cost = 600, mult = 1.5 }, { cost = 3_000, mult = 2 },
	{ cost = 20_000, mult = 2 }, { cost = 150_000, mult = 2.5 },
}
local income, coins, elapsed = 2, 0, 0 -- pièces par seconde au départ
for index, upgrade in upgrades do
	local waitSeconds = math.max(0, (upgrade.cost - coins) / income)
	elapsed += waitSeconds
	coins += waitSeconds * income - upgrade.cost
	income *= upgrade.mult
	print(string.format("amélioration %d : %.1f min, attente %.0f s", index, elapsed / 60, waitSeconds))
end
```

## 14. Mesurer l'économie

- `AnalyticsService:LogEconomyEvent(player, flowType, currencyType, amount, endingBalance, transactionType, itemSku?, customFields?)`, côté serveur. `flowType` : `Enum.AnalyticsEconomyFlowType.Source` ou `.Sink`. `transactionType` : le `.Name` d'un `Enum.AnalyticsEconomyTransactionType` (`IAP`, `Shop`, `Gameplay`, `ContextualPurchase`, `TimedReward`, `Onboarding`). `amount` reste positif, source comme puits : c'est `flowType` qui donne le sens. Limites par expérience : 5 types de monnaie, 100 SKU, 20 types de transaction (doc, 2026-10-18). Les événements ne partent que du serveur et d'un jeu publié : rien n'arrive depuis Studio.
- Journalise **après** la mutation qui fait autorité, avec le solde qu'elle a commité. Jamais un solde recalculé depuis `leaderstats` : 100 + 50 commite 150, un recalcul après coup peut journaliser 200. Achat échoué ou remboursé : aucun événement.
- Un SKU par source et par puits (`QuestReward_Daily` ≠ `QuestReward_Event`) pour savoir qui gonfle ; un puits jamais journalisé fabrique une fausse inflation. Entonnoir d'achat avec `LogFunnelStepEvent` (boutique ouverte, objet vu, achat cliqué, confirmé, accordé) et un même identifiant de session (`HttpService:GenerateGUID(false)`) par tentative.
- Une panne de télémétrie ne casse jamais un octroi et ne provoque jamais un second octroi. L'analytics n'est jamais la source de vérité de l'inventaire.
- Signaux de santé (points de départ, pas des cibles Roblox) : rapport puits/sources proche de 1 par monnaie ; unités par joueur actif en hausse = inflation ; plus de 75 % du revenu venant des 5 % de plus gros payeurs = fragilité ; aucune réaction à un changement de prix = prix trop bas ou valeur mal expliquée.
- Diagnostic : a-t-on les événements ? Sont-ils justes (après succès, côté serveur) ? Quel est le signal cassé le plus étroit ? Change un seul levier, avec une hypothèse et un garde-fou, puis remesure. Projection de revenu : Robux dépensés par jour = joueurs actifs × taux de conversion × panier moyen ; × 0,7 pour la part créateur ; × taux DevEx pour des dollars. Oublier les 30 % gonfle la projection d'environ 43 %.

## 15. Tester les achats

- Passes et produits exigent une expérience publiée. En Studio, les invites font des achats de test, sans Robux débités, et `ProcessReceipt` se déclenche (annonce DevForum ; en Team Test, l'achat réussit sans appeler `ProcessReceipt`, bogue signalé). `UserOwnsGamePassAsync` y renvoie faux pour un pass que le compte ne possède pas.
- Abonnements en Studio : ids de test `EXP-0` (actif, renouvelé), `EXP-1` (retour après interruption), `EXP-2` (remboursé), `EXP-3` (paiement en attente) (doc de `GetUserSubscriptionPaymentHistoryAsync`, 2026-10-18).
- Teste `Receipts.process` sans vrai achat, en playtest, sur un store jetable : un Script de test injecté en `Edit` (`studio-mcp.md`, section 7) requiert le module du jeu et l'appelle avec de faux reçus dont le `PurchaseId` commence par `test-`. Un `require` lancé par `execute_luau` reçoit sa propre copie du module, aux profils vides : il ne renverrait que `NotProcessedYet`. Résultats attendus :

| Cas | Décision attendue |
|---|---|
| Nouveau `PurchaseId`, octroi réussi | `PurchaseGranted`, octroi appliqué une fois |
| Même `PurchaseId` rejoué | `PurchaseGranted`, aucun second octroi |
| Octroi qui lève une erreur | `NotProcessedYet` |
| Produit inconnu | `NotProcessedYet` |
| Joueur absent ou profil non chargé | `NotProcessedYet` |
| Arrêt du serveur entre l'octroi et l'acquittement | un seul octroi après le retour |

- Matrice complète : invite annulée ; erreur de l'API de possession ; départ avant traitement ; sauvegarde en échec ; pass acheté en cours de partie (effet sans rejoin) ; respawn ; deux passes qui touchent le même avantage ; abonnement qui change ; politique qui refuse ou ne répond pas ; panne de télémétrie injectée ; remote d'achat en monnaie de jeu tiré deux fois dans la même frame (le solde ne passe jamais sous zéro).
- Avant le lancement : chaque id de produit non nul, appartenant à cette expérience, en vente, avec son handler ; un seul `ProcessReceipt` (cherche toutes les affectations dans le projet) ; dédoublonnage des reçus dans le profil ; prix affichés égaux aux prix facturés. Prix régionaux : « Dynamic Price Check » du Creator Dashboard pour tester l'affichage, puis désactive-le.
- Un vrai achat sur une place publiée reste « Non vérifié » tant qu'un humain ne l'a pas fait. Toucher à `ProcessReceipt` est à risque élevé : propose d'abord, avec un plan de retour arrière, et attends le oui. Revenir à l'ancien code n'annule ni les octrois ni les migrations : ne rejoue jamais d'achats pour « réparer ».

## 16. Pièges

- `Enum.ProductPurchaseDecision.NotYetGranted` n'existe pas : seulement `NotProcessedYet` et `PurchaseGranted`.
- Accorder dans `PromptProductPurchaseFinished` ou sur un remote client. Renvoyer `PurchaseGranted` après un simple changement en mémoire : ProfileStore ne sauvegarde que toutes les 300 s.
- Historique des reçus dans un autre store que l'effet, ou `GetAsync` puis `SetAsync` sur cet historique.
- Valider un abonnement dans `ProcessReceipt` : il n'y passe jamais. Appeler `PolicyService:IsEligibleToPurchaseSubscription()` : c'est un champ du résultat de `GetPolicyInfoForPlayerAsync`.
- Une lecture de possession ratée traitée comme « non possédé » : un payeur perd ses avantages. `Enum.InfoType.Product` pour lire un game pass.
- Bonus quotidien (Premium ou autre) sans date sauvegardée ; récompense de pub accordée sans `AdService`.
- Chances affichées différentes des chances réelles ; sauvegarde ou progression de base vendue.

## 17. Checklist

- [ ] Un seul `ProcessReceipt` (ou un routage `BindReceiptHandler` explicite) pour tout le jeu.
- [ ] `PurchaseId` et effet dans le même profil et la même écriture ; `PurchaseGranted` après sauvegarde confirmée.
- [ ] `NotProcessedYet` pour joueur absent, profil non chargé, produit inconnu, erreur ou sauvegarde non confirmée.
- [ ] Passes vérifiés au join et à `PromptGamePassPurchaseFinished` côté serveur ; échec de lecture = inconnu.
- [ ] Abonnements lus par statut, avantage retiré à l'expiration, revérifiés à chaque arrivée ; prix lus avec `GetProductInfoAsync`, rien de codé en dur.
- [ ] Objets aléatoires payants : chances affichées avant l'achat, `ArePaidRandomItemsRestricted` vérifié par joueur, échec = restreint.
- [ ] Pubs récompensées via `AdService` et le reçu du produit, jamais via un remote ; monnaie de jeu : le client envoie un id, le serveur valide, débite et accorde sans yield.
- [ ] Sources et puits journalisés après succès, avec le solde commité ; changement d'économie simulé, présenté en avant/après, validé par le propriétaire.
- [ ] Matrice de tests des reçus passée ; vrais achats marqués « Non vérifié » tant qu'ils n'ont pas tourné.
