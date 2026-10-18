# UI du moteur : mise en page, texte, images, entrées, design

À lire quand : tu construis ou corriges une interface Roblox (HUD, boutique, menu, popup, plaque de nom) et que la question touche le moteur : ScreenGui, safe area, UDim2, layouts, mise à l'échelle téléphone et PC, texte, images, contours, manette, ou le look d'une UI « simulator ». Le code déclaratif (composants React, état, animations) est dans ui-react.md : ce fichier donne les règles que ce code doit respecter.

## Sommaire

1. Règles de base
2. ScreenGui et couches
3. Safe area, barre du haut, zones des pouces
4. Taille, position, mesures
5. Layouts et contraintes
6. Mise à l'échelle par appareil
7. Texte
8. Images
9. Apparence : coins, contours, dégradés, ombres, découpe
10. ViewportFrame et ScrollingFrame
11. Boutons et entrées
12. UI dans le monde
13. Design d'UI de jeu
14. Accessibilité et vérification
15. Pièges
16. Checklist

## 1. Règles de base

1. **Le client fait l'UI, le serveur fournit les données.** Le serveur ne crée ni ne modifie l'UI d'un joueur. Ouvrir un menu ne demande aucun aller-retour réseau ; seul l'achat qui suit passe par une remote validée.
2. **Modifie la copie dans PlayerGui**, jamais StarterGui en jeu : StarterGui n'est qu'un gabarit cloné au spawn.
3. **Scale pour les régions, Offset pour les détails, contraintes pour borner.** Sans bornes, l'un ou l'autre casse aux extrêmes.
4. **AnchorPoint assorti au bord d'épinglage**, réglé avant la position.
5. **Les layouts placent les enfants répétés.** Vingt slots posés à la main cassent au vingt et unième.
6. **Interactif dans `CoreUISafeInsets`**, loin du joystick, du saut et de la barre du haut.
7. **Téléphone d'abord** : conçois pour 667 × 375 en paysage, puis agrandis.
8. **Texte rendu ≥ 12 px, cibles ≥ 44 px** sur ce téléphone.
9. **Un seul style par jeu** : une palette en tokens, deux rayons, deux épaisseurs, deux polices.
10. **Mesure puis regarde** (AbsoluteSize, capture) sur téléphone et sur PC avant de dire fini.

## 2. ScreenGui et couches

- Au spawn, Roblox clone les ScreenGui de `StarterGui` dans `PlayerGui`. Modifie la copie : changer StarterGui en jeu ne change rien à l'écran.
- `ResetOnSpawn` vaut `true` par défaut : la copie est détruite et reclonée à chaque mort, et tes références meurent avec elle. Mets `false` sur tout ce qui persiste. Ce réglage n'est fiable que sur un ScreenGui **enfant direct** de StarterGui : rangé dans un Folder, il est recloné quand même.
- Un ScreenGui créé par code (React, Luau) se parente à `Players.LocalPlayer:WaitForChild("PlayerGui")`, lui aussi en `ResetOnSpawn = false`. Avec `Players.CharacterAutoLoads = false`, StarterGui n'est copié qu'au premier `Player:LoadCharacterAsync()` (`LoadCharacter` est déprécié) : crée par code l'UI d'un lobby.
- Écris toujours `ZIndexBehavior = Sibling` : un enfant est dessiné au-dessus de son parent et `ZIndex` ne départage que des frères. Un enfant reste donc sous tout ce qui est dessiné au-dessus de son parent, par exemple un frère du parent de ZIndex plus haut : tooltips, listes déroulantes et popups vont à la racine du ScreenGui ou dans une couche plus haute (portail React, ui-react.md). Un seul tooltip partagé, déplacé et rempli, gardé à quelques pixels dans l'écran, vaut mieux qu'un clone par carte.
- `DisplayOrder` ordonne les ScreenGui, le plus grand devant. Une couche par ScreenGui (convention de terrain) : HUD 0 ; fenêtres 10, une à la fois ; popups et toasts 20 ; modale bloquante (achat en cours) 50 ; chargement et transitions 100.
- Cacher : `ScreenGui.Enabled = false` coupe rendu, entrées et layout de tout l'arbre, c'est le moins cher ; `Visible = false` pour une fenêtre d'une couche partagée. Ne pousse jamais une fenêtre hors écran pour la cacher : elle coûte encore layout et entrées.
- UI de Roblox que tu remplaces : `StarterGui:SetCoreGuiEnabled(Enum.CoreGuiType.Backpack, false)` côté client. `StarterGui:SetCore(...)` échoue tant que Roblox ne l'a pas enregistré : `pcall` et nouvel essai.
- Toast natif sans écrire d'UI : `GuiService:SendNotification({ Title = …, Text = …, Icon = …, Buttons = { { Text, ButtonType, OnActivated } }, OnDisplay = …, OnDismiss = … })` renvoie un id (chaîne vide si la file refuse) à passer à `DismissNotification(id)` (doc, 2026-10-18 ; appel côté client, à vérifier). Pour un style propre au jeu, garde ta couche de toasts (DisplayOrder 20).

```luau
--!strict
local Players = game:GetService("Players")

-- Crée une couche d'UI persistante.
local function createLayer(name: string, displayOrder: number, insets: Enum.ScreenInsets?): ScreenGui
	local gui = Instance.new("ScreenGui")
	gui.Name = name
	gui.ResetOnSpawn = false
	gui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
	gui.DisplayOrder = displayOrder
	gui.ScreenInsets = insets or Enum.ScreenInsets.CoreUISafeInsets
	gui.Parent = Players.LocalPlayer:WaitForChild("PlayerGui") -- le parent en dernier
	return gui
end
local screenFx = createLayer("ScreenFx", 30, Enum.ScreenInsets.None) -- flashs, vignettes : rien de cliquable
```

## 3. Safe area, barre du haut, zones des pouces

`ScreenGui.ScreenInsets` choisit ce que la couche évite :

| Valeur | Évite | Pour |
|---|---|---|
| `CoreUISafeInsets` (défaut) | boutons de la barre du haut de Roblox et découpes d'écran | tout ce qui se clique ou se lit |
| `DeviceSafeInsets` | encoches et découpes seulement ; peut passer sous la barre du haut | contenu qui gère la barre à la main |
| `TopbarSafeInsets` | bande libre à côté des boutons de Roblox, qui suit la barre | tes icônes de barre du haut |
| `None` | rien | fonds, vignettes, flashs ; jamais de bouton |

- `IgnoreGuiInset = true` est l'ancienne méthode et revient à `DeviceSafeInsets`. Ne compense jamais deux fois : `ScreenInsets` plus un padding manuel de safe area décale tout du double.
- `ClipToDeviceSafeArea` (vrai par défaut) rogne les descendants à la zone sûre de l'appareil : `false` pour un fond bord à bord (ignoré en `ScreenInsets = None`). Deux couches aux insets différents n'ont pas la même origine (un HUD centré sans inset a dérivé de ~130 px par rapport à la barre d'objets sur un Android à découpe) : convertis les positions avant de les comparer, avec `GuiService:GetInsetArea(insets)`, qui renvoie le `Rect` utile de chaque mode relativement à la zone `CoreUISafeInsets`.
- La barre du haut mesure 58 px dans le design actuel, sans être une constante documentée. Lis `GuiService.TopbarInset` (un `Rect` qui change : écoute `GetPropertyChangedSignal("TopbarInset")`) ou `GuiService:GetGuiInset()` (haut-gauche et bas-droite, valable pour les couches en `IgnoreGuiInset = false`).
- Tes icônes à côté de celles de Roblox : **TopbarPlus v3**, le standard communautaire (identifiant dans paquets.md). Il lie la croix Haut de la manette à ses icônes et peut voler le focus d'une fenêtre : vérifie ses réglages manette. Sans bibliothèque, une couche en `TopbarSafeInsets` suit la bande libre de la barre.

Zones réservées sur téléphone en paysage :
- **Bas-gauche : joystick.** Le joystick dynamique accepte les touches sur une grande partie de la moitié gauche : rien dans le quart bas-gauche.
- **Bas-droite : saut** et glisser de caméra du pouce droit. Mesure `PlayerGui.TouchGui` (`JumpButton`) au lieu de deviner : le script de contrôle par défaut le passe de 70 à 120 px quand le petit côté de l'écran dépasse ~500 px.
- **Haut : barre de Roblox** (menu, chat, sac). Le haut-centre est dur à atteindre au pouce : rien de fréquent.
- Placement type d'un simulator : monnaies en haut à gauche sous la barre, boosts en haut à droite, colonne de boutons à gauche vers Y = 0,45, barre d'XP en bas au centre entre joystick et saut, action au-dessus du saut.
- Place par mesure, pas par offset magique : lis `AbsolutePosition` et `AbsoluteSize` du voisin et cale-toi contre lui ; recalcule quand l'écran, l'entrée ou l'état du jeu change. Les chevauchements naissent des combinaisons d'états (toast + minuteur, spectateur + barre d'objets) : liste les états du HUD (lobby, en jeu, mort, spectateur, résultats, fenêtre ouverte) et vérifie chaque paire visible ensemble sur le plus petit téléphone. Pendant qu'une fenêtre est ouverte sur téléphone, cache le HUD non essentiel au lieu de tout rétrécir.

## 4. Taille, position, mesures

- Pixels d'un axe = taille absolue du parent × Scale + Offset (`UDim2.fromScale`, `UDim2.fromOffset`). Scale place les régions et fixe leur part d'écran ; Offset sert aux épaisseurs, marges, écarts et éléments à taille physique (icônes, boutons tactiles). Tout en Offset : minuscule en 4K, énorme sur téléphone. Tout en Scale : formes écrasées, cibles trop petites, et un carré en Scale qui devient un rectangle en 16:9 (ajoute un `UIAspectRatioConstraint`).
- `AnchorPoint` est l'origine de la position, de la taille et de l'agrandissement par `UIScale`, pas de la rotation : `Rotation` tourne toujours autour du centre de l'objet (doc). Règle-le avant la position (le changer ensuite déplace l'objet), assorti au bord d'épinglage : l'objet garde sa marge sur tous les écrans et grandit vers l'intérieur. Pour un « pop », mets-le au point de croissance : `(0.5, 0.5)` pour une fenêtre, `(0.5, 0)` pour un menu déroulant.

| Épinglé | AnchorPoint | Position (marge 16) |
|---|---|---|
| haut-gauche | `(0, 0)` | `UDim2.new(0, 16, 0, 16)` |
| haut-droite | `(1, 0)` | `UDim2.new(1, -16, 0, 16)` |
| centre | `(0.5, 0.5)` | `UDim2.fromScale(0.5, 0.5)` |
| bas-centre | `(0.5, 1)` | `UDim2.new(0.5, 0, 1, -16)` |
| bas-droite | `(1, 1)` | `UDim2.new(1, -16, 1, -16)` |

- `AutomaticSize` (`X`, `Y`, `XY`) fait de `Size` un minimum. Mets à 0 l'axe automatique : `UDim2.new(1, 0, 0, 0)` + `AutomaticSize.Y` = pleine largeur, hauteur du contenu. L'objet grandit depuis son AnchorPoint : ancré à droite, un label s'allonge vers la gauche.
- Pas de cycle : un parent en `AutomaticSize.X` dont un enfant veut 100 % de la largeur en Scale, ou `AutomaticSize.XY` avec `TextWrapped`, donne un résultat indéfini. Fixe la largeur, automatise seulement Y.
- `AbsoluteSize`, `AbsolutePosition` et `TextBounds` valent zéro juste après la création : le layout passe à la frame suivante. Écoute `GetPropertyChangedSignal("AbsoluteSize")` et appelle ta fonction une fois au départ ; ni sondage par frame, ni `task.wait()` « pour laisser l'UI se poser ». Un handler qui écrit des tailles relance `AbsoluteSize` : boucle possible (des dizaines de passes vues en vrai). N'écris que si la valeur change, ou reporte à la frame suivante (`RunService.Heartbeat:Once`) avec un nombre de passes borné.
- `AbsoluteSize` inclut les UIScale des ancêtres, pas les Offset que tu écris : une taille Offset déduite d'un AbsoluteSize serait multipliée deux fois. Divise d'abord par l'échelle totale (`totalScale`, section 5).
- `PlayerGui:GetGuiObjectsAtPosition(x, y)` renvoie une liste vide avant la première mise en page ; appelé sur un ScreenGui (Studio 0.743), il ne renvoie que les objets de ce ScreenGui. Il attend des coordonnées après inset, comme l'`input.Position` d'un `InputBegan` (exemple de la doc). `UserInputService:GetMouseLocation()` donne des pixels sans inset : soustrais l'inset haut-gauche renvoyé par `GuiService:GetGuiInset()` avant de comparer à une position d'UI d'une couche en `CoreUISafeInsets` (`IgnoreGuiInset = false`, doc) ; pour une couche aux autres insets, décale aussi de `GuiService:GetInsetArea(insets).Min` (à vérifier).

## 5. Layouts et contraintes

- Un layout prend le contrôle de la position de ses frères, et de leur taille en flex : une `Position` écrite à la main est ignorée sans erreur. Un layout par parent. Pour animer un élément de liste, anime l'enfant d'un cadre « slot » placé par le layout (ui-react.md). Priorité : le layout place, les contraintes de taille l'emportent sur lui, `UIScale` multiplie le résultat final.
- **UIListLayout** : écris `SortOrder = Enum.SortOrder.LayoutOrder`. `LayoutOrder` est une donnée (rang, prix, rareté), négative pour l'ordre inverse. `Padding` = écart *entre* les éléments ; les marges vont dans un `UIPadding`.
- **Flex**, intégré à UIListLayout : `HorizontalFlex`/`VerticalFlex` (`Fill`, `SpaceBetween`, `SpaceAround`, `SpaceEvenly`), `ItemLineAlignment = Stretch`, `Wraps`. L'enfant qui absorbe l'espace porte un `UIFlexItem` (`FlexMode = Fill`, `Grow`, `Shrink` ou `Custom`). La classe `UIFlexLayout` n'existe pas.
- Cinq enfants à 0,25 de largeur débordent, car la Scale d'un élément de liste suit le conteneur, pas l'espace restant : mets `HorizontalFlex = Fill`. Le flex coûte un peu plus qu'une liste simple : garde-le pour ce qui s'étire (onglets égaux, recherche entre deux boutons).
- **UIGridLayout** : `CellSize`, `CellPadding`, `FillDirectionMaxCells` (0 = autant que possible), `StartCorner`, `SortOrder` ; `AbsoluteCellSize` se lit. Une seule taille de cellule : un enfant ne couvre plusieurs cases que si un `UISizeConstraint` lui impose un `MinSize` plus grand que la cellule (doc). Un `UIAspectRatioConstraint` placé dans le UIGridLayout garde les cellules carrées. Un UIListLayout en `Wraps` n'a qu'un écart pour les lignes et les éléments. Calcule les colonnes depuis la place, jamais depuis le nom de l'appareil (code ci-dessous).
- **UIPageLayout** : une page visible à la fois (onglets, tutoriel, carrousel) avec `JumpTo`, `Next`, `Previous`, `Circular`, `TweenTime` et les événements `PageEnter`/`PageLeave` ; aucune flèche fournie. **UITableLayout** : seulement pour un vrai tableau.
- `UIPadding` en Scale suit l'axe de chaque côté et varie avec le ratio d'écran : préfère l'Offset. `UISizeConstraint` (`MinSize`/`MaxSize` en pixels) évite la fenêtre minuscule sur téléphone ou géante en 4K : teste les deux bornes.
- `UIAspectRatioConstraint` (`AspectRatio` = largeur / hauteur, `AspectType`, `DominantAxis`) : slots, avatars, cartes, jamais un conteneur de texte qui doit grandir. Plus grand carré dans n'importe quel parent : `Size = UDim2.fromScale(1, 1)`, ratio 1, `FitWithinMaxSize`, centré.
- `UIScale` multiplie l'objet et tout son sous-arbre, contours et coins compris, depuis l'AnchorPoint. Une seule par objet, qui compose les causes (écran × ouverture × appui, ui-react.md) ; deux UIScale dans des ancêtres se multiplient.

```luau
--!strict
-- Produit des UIScale de l'objet et de ses ancêtres (AbsoluteSize l'inclut, les Offset non).
local function totalScale(object: Instance): number
	local scale = 1
	local current: Instance? = object
	while current do
		local uiScale = current:FindFirstChildOfClass("UIScale")
		if uiScale then
			scale *= uiScale.Scale
		end
		current = current.Parent
	end
	return scale
end

-- Grille carrée de 2 à 5 colonnes selon la place. Le container ne porte pas de UIPadding
-- (mets-le sur son parent) ; dans un ScrollingFrame, lis plutôt AbsoluteWindowSize.
local function fitGrid(container: GuiObject, grid: UIGridLayout, gap: number, minCell: number): RBXScriptConnection
	local function update()
		local width = container.AbsoluteSize.X / totalScale(container)
		if width <= 0 then
			return -- pas encore mis en page
		end
		local columns = math.clamp(math.floor((width + gap) / (minCell + gap)), 2, 5)
		local cell = math.floor((width - gap * (columns - 1)) / columns)
		grid.CellPadding = UDim2.fromOffset(gap, gap)
		grid.CellSize = UDim2.fromOffset(cell, cell)
	end
	update()
	return container:GetPropertyChangedSignal("AbsoluteSize"):Connect(update)
end
```

## 6. Mise à l'échelle par appareil

Conçois pour le plus petit téléphone, puis agrandis. Pourquoi : une UI dessinée en 1080p puis réduite donne des cibles et un texte trop petits ; une UI de téléphone agrandie reste lisible partout.

1. **Unités de design = points du téléphone de référence** (667 × 375 en paysage, échelle 1). Les grappes (pilules de monnaie, colonne de boutons, fenêtres) ont une taille en Offset dans ces unités et une position en Scale + AnchorPoint vers leur bord.
2. **Une `UIScale` dans chaque grappe**, réglée sur une valeur commune calculée depuis l'écran ; jamais dans un parent qui en porte déjà une.
3. **Fenêtres bornées à la zone utile** (écran moins insets) ; leur contenu en layouts et ScrollingFrame encaisse la faible hauteur du téléphone.
4. **Reflow plutôt que rétrécir** quand la place manque : moins de colonnes, détail en vue séparée, onglets empilés, options rares un niveau plus bas. Décide sur la place disponible, pas sur le type d'appareil.

Formule (heuristique de terrain, à régler par jeu) : `s = clamp(min(W / 667, H / 375), 0.75, max(1.35, H / 800))`, où W × H est `Camera.ViewportSize`. Repères : petit téléphone 1,0 ; iPhone 14 (844 × 390) 1,04 ; tablette et PC 1080p 1,35 ; 1440p 1,8 ; 4K 2,7. Le portrait tombe à 0,75 et demande sa propre mise en page.

```luau
--!strict
local Workspace = game:GetService("Workspace")

local REF = Vector2.new(667, 375) -- plus petit téléphone paysage, en points

local function deviceScale(viewport: Vector2): number
	local fit = math.min(viewport.X / REF.X, viewport.Y / REF.Y)
	return math.clamp(fit, 0.75, math.max(1.35, viewport.Y / 800))
end

-- Taille d'une fenêtre en unités de design, bornée par la zone utile (pixels, insets déduits).
local function fitWindow(maxSize: Vector2, scale: number, usable: Vector2): UDim2
	local width = math.min(maxSize.X, usable.X / scale - 24)
	local height = math.min(maxSize.Y, usable.Y / scale - 24)
	return UDim2.fromOffset(math.max(math.floor(width), 0), math.max(math.floor(height), 0))
end

-- root : Frame de Size (1, 0, 1, 0) dans une couche en CoreUISafeInsets, sans UIScale.
-- Son AbsoluteSize suit l'écran et les insets : un seul signal suffit.
local function bindScale(root: GuiObject, apply: (scale: number, usable: Vector2) -> ()): () -> ()
	local function refresh()
		local camera = Workspace.CurrentCamera -- relue à chaque fois : elle peut être remplacée
		if camera and root.AbsoluteSize.X > 0 then
			apply(deviceScale(camera.ViewportSize), root.AbsoluteSize)
		end
	end
	local connection = root:GetPropertyChangedSignal("AbsoluteSize"):Connect(refresh)
	refresh()
	return function()
		connection:Disconnect()
	end
end
```

- Usage : `bindScale(root, function(s, usable) shopScale.Scale = s; shopWindow.Size = fitWindow(Vector2.new(760, 520), s, usable) end)`, la fenêtre étant ancrée au centre. Si l'objet s'anime aussi par sa UIScale, multiplie : `Scale = s × ouverture × appui`. En roblox-ts, porte ces fonctions telles quelles et pousse `s` dans un binding (ui-react.md).
- Repères en unités de design, donc rendus tels quels sur le plus petit téléphone (heuristiques de terrain) : texte ≥ 12 (corps 14 à 16, bouton 18 à 22, titre 24 à 28) ; cibles ≥ 44 × 44 (≥ 56 de haut pour les boutons de la boucle de jeu) ; écarts ≥ 8 ; fenêtres de 760 × 520 au plus.
- `UserInputService.PreferredInput` change le comportement (survol, indices de touches, boutons tactiles), pas la mise en page, qui suit la place disponible. Ne détecte pas « mobile » par `TouchEnabled and not KeyboardEnabled` : un PC tactile, une tablette avec clavier et l'émulateur de Studio se trompent.
- TV : `GuiService.ViewportDisplaySize == Enum.DisplaySize.Large` signale un écran vu de loin. Grossis texte et cibles, garde une marge contre l'overscan. `GuiService:IsTenFootInterface()` n'est pas déprécié, mais il renvoie vrai sur toute console quel que soit l'usage : la doc conseille `ViewportDisplaySize` à la place.

## 7. Texte

**Police**
- `FontFace` (objet `Font`) pour tout nouveau texte ; l'ancienne propriété `Font` (enum) ne reçoit plus de familles. `Font.fromEnum(Enum.Font.FredokaOne)`, `Font.fromName("BuilderSans", Enum.FontWeight.Bold)`, `Font.new("rbxasset://fonts/families/Montserrat.json", Enum.FontWeight.Heavy)`. Gotham est retiré : `Enum.Font.Gotham*` s'affiche en Montserrat (doc de l'enum `Font`).

| Usage | Familles intégrées |
|---|---|
| Titres et chiffres « jeu » | FredokaOne, LuckiestGuy, Bangers, DenkOne |
| Libellés gras | Montserrat (Bold à Heavy, remplace Gotham), BuilderSans (Bold, ExtraBold) |
| Corps | BuilderSans, Nunito, Roboto |
| Thèmes | Creepster (horreur, Halloween), Arcade (pixel), Michroma (science-fiction), Fondamento (fantasy) |

- Deux familles au plus, affichage et corps. Les graisses disponibles varient selon la famille : vérifie dans le sélecteur de Studio. Police personnalisée : `.ttf` ou `.otf` uploadé et modéré, puis `Font.new("rbxassetid://<id>")`.
- À taille égale, les glyphes Roblox sortent environ 14 % plus petits que dans un navigateur : multiplie par ~1,14 les tailles d'une maquette Figma ou HTML (mesure de terrain).

**Taille**

| Cas | Réglage |
|---|---|
| Corps, description, texte traduit, bouton à libellé variable | `TextSize` fixe + `AutomaticSize` (Y, ou X pour une pilule) + `TextWrapped` |
| Gros chiffre ou titre qui remplit une boîte fixe | `TextScaled = true` + `TextWrapped = true` + `UITextSizeConstraint` |
| Rangée de libellés qui doivent être égaux | `TextSize` fixe partout : TextScaled donne une taille par label |

- La doc officielle préfère `AutomaticSize` à `TextScaled` ; jamais les deux sur le même label.
- Pièges de `TextScaled` : il ignore `TextSize` ; sans contrainte, il monte à 100 px ou descend jusqu'à l'illisible ; en multi-ligne, le mot le plus long fixe la taille ; des centaines de labels TextScaled coûtent du layout ; dans une liste, il exige une taille fixe.
- Contrainte type : `MinTextSize = 12` (jamais moins de 9, que la doc juge illisible), `MaxTextSize` ≈ 0,6 à 0,8 × la hauteur du label.
- Texte trop long : plus de place, deux lignes, ou `TextTruncate = AtEnd` (`SplitWord` en cellule étroite) ; `TextFits` dit s'il déborde. Ne descends pas sous le minimum. `LineHeight` 1,2 à 1,4 pour un paragraphe.

**Contour et RichText**
- Contour des lettres = un `UIStroke` en `Contextual`, 2 à 3 px, presque noir, à compter dans tes mesures (le trait déborde des lettres). Abandonne `TextStrokeTransparency`, trait fin et rigide. Ombre dure sous un titre : copie noire du label, 3 à 4 px plus bas, derrière.
- `RichText = true` active `<b>`, `<i>`, `<u>`, `<s>`, `<br/>`, `<font color="#FFD700" size="22" weight="heavy" transparency="0.2">`, `<stroke color="#1A1A1A" thickness="2">`, `<uc>`, `<sc>`, `<mark>`. Échappe `<`, `>`, `&`, `"`, `'` en `&lt;`, `&gt;`, `&amp;`, `&quot;`, `&apos;`. Pas de lien, pas d'image en ligne, pas d'exposant.
- Jamais de RichText sur un texte de joueur non échappé : il imiterait un message système avec `<font color>`. Stocke le brut, ajoute les balises au rendu.
- Machine à écrire : anime `MaxVisibleGraphemes` de 0 au nombre de graphèmes (compte-les avec `utf8.graphemes`, un emoji vaut un), puis remets `-1`. Il ne refait pas la mise en page.

**Mesurer et formater**

```luau
--!strict
local TextService = game:GetService("TextService")

-- Taille rendue d'un texte FontFace. Yield : appelle-la dans un task.spawn.
local function measureText(text: string, font: Font, size: number, maxWidth: number): Vector2
	local params = Instance.new("GetTextBoundsParams")
	params.Text = text
	params.Font = font
	params.Size = size
	params.Width = maxWidth
	local ok, bounds = pcall(TextService.GetTextBoundsAsync, TextService, params)
	return if ok then bounds else Vector2.zero
end

local SUFFIXES = { "", "K", "M", "B", "T", "Qa", "Qi" }

-- 1234 → "1.23K", 15300000 → "15.3M" : trois chiffres significatifs, arrondi vers le bas.
local function abbreviate(n: number): string
	if n ~= n or math.abs(n) == math.huge then
		return "?"
	end
	local sign = if n < 0 then "-" else ""
	local value, tier = math.abs(n), 1
	while value >= 1000 and tier < #SUFFIXES do
		value /= 1000
		tier += 1
	end
	local text = if value >= 100 then string.format("%d", math.floor(value))
		elseif value >= 10 then string.format("%.1f", math.floor(value * 10) / 10)
		else string.format("%.2f", math.floor(value * 100) / 100)
	if string.find(text, ".", 1, true) then
		text = string.gsub(text, "0+$", "")
		text = string.gsub(text, "%.$", "")
	end
	return sign .. text .. SUFFIXES[tier]
end
```

- `TextService:GetTextSize` (ancienne API) prend un `Enum.Font` et mesure mal un `FontFace` : ne l'utilise plus. Ajoute deux fois l'épaisseur du contour. Pour seulement adapter une boîte, `AutomaticSize` suffit.

**Localisation**
- Des clés stables (`shop.buy`, `pet.dragon.name`), jamais la phrase anglaise comme identifiant ni comme clé DataStore. Tables de localisation de Roblox en production.
- `AutoLocalize` traduit les GuiObjects automatiquement : `false` sur les noms de joueurs, les nombres et le texte généré. La traduction retire les balises RichText : réapplique le formatage après.
- Garde ~30 % de largeur libre sur les boutons : les traductions s'allongent. Les mots cuits dans une image ne se traduisent pas.
- Tout texte de joueur montré à d'autres passe par le filtrage serveur (`TextService:FilterStringAsync`, securite.md).

## 8. Images

- Formats : PNG (recommandé, transparence), JPG (sans alpha), BMP, TGA. Le GIF passe à l'import mais ne s'anime pas (`AnimatedImage` est interne, retiré en 0.742). **Pas de SVG.**
- Depuis un SVG ou une icône vectorielle : rasterise en PNG (par exemple avec resvg, à 2× la taille d'affichage) avec une marge transparente de quelques pixels, corrige le halo, puis uploade par Studio (Asset Manager, import en lot) ou par l'API Open Cloud Assets, qui renvoie un asset id (open-cloud.md).
- Taille : au-delà de 1024 px de côté, Roblox réduit l'image (constat de terrain, à vérifier : la doc ne le dit pas pour les images d'UI, et sa page sur les textures 3D annonce jusqu'à 4096). Ne dépasse pas 1024 ; vise ≤ 512 × 512 pour une grande image et ≤ 256 × 256 pour une icône. Pourquoi : la mémoire suit les pixels (largeur × hauteur × 4 octets), pas le poids du fichier ; 1024² coûte quatre fois 512².
- Halo sombre autour d'un PNG transparent : le filtrage mélange la couleur des pixels transparents, souvent noirs. Avant l'upload, étends la couleur des bords opaques dans ces pixels (« alpha bleed », façon PixelFix) sans toucher à l'alpha.
- `ScaleType` : `Stretch` (défaut, déforme), `Fit` (icônes), `Crop` (remplit et rogne), `Tile` (`TileSize`), `Slice` (9-slice). `ResampleMode = Enum.ResamplerMode.Pixelated` garde le pixel art net.
- 9-slice : `ScaleType = Slice`, `SliceCenter = Rect.new(minX, minY, maxX, maxY)` en pixels de l'image source (image 64 × 64 à coins de 16 → `Rect.new(16, 16, 48, 48)`), `SliceScale` pour l'épaisseur des bords. L'éditeur 9-Slice de Studio affiche des distances aux bords, pas le Rect. Pour un style plat, UICorner + UIStroke + UIGradient suffisent.
- Planches : `ImageRectOffset` + `ImageRectSize` affichent une case ; c'est aussi la seule animation image par image qui marche. Teinte au lieu de ré-uploader : `ImageColor3` multiplie. Icônes en blanc ou en gris, teintées ensuite. Silhouette d'un objet à découvrir : `ImageColor3 = Color3.new(0, 0, 0)`, `ImageTransparency = 0.3`.
- Pas de modes de fusion : superpose une couche « reflets » blanche et une couche « ombres » noire, chacune en transparence. `UIStroke` n'entoure jamais les pixels d'une image : cuis le contour d'une icône dans le PNG, en blanc pour qu'il reste teintable.
- Une image refusée, en attente de modération ou d'un autre compte s'affiche vide, sans erreur : prévois un repli, évite texte et logos dans les images, uploade sous le propriétaire du jeu.
- Un id de Decal n'est pas un id d'Image : collé dans la fenêtre Propriétés, Studio le convertit ; écrit par script, l'image reste vide (à vérifier selon le mode d'upload).
- Préchargement : `ContentProvider:PreloadAsync` sur les ImageLabel de l'écran, dans un `task.spawn` borné par un délai ; lis le `Enum.AssetFetchStatus` de chaque asset dans le callback.
- **Portraits et icônes sans upload** (classement, amis, boutique) : `Image = "rbxthumb://type=AvatarHeadShot&id=" .. userId .. "&w=150&h=150"`. Tailles permises (doc « assets », 2026-10-18) : `AvatarHeadShot` 48, 60, 100, 150, 180, 352, 420 (carré) ; `Avatar` les mêmes plus 720 ; `AvatarBust` 50, 60, 75, 100, 150, 180, 352, 420 ; `Asset`, `BundleThumbnail`, `GroupIcon` 150 ou 420 ; `BadgeIcon` et `GamePass` 150 ; `GameIcon` 50 ou 150 (id = `game.GameId`). Variante : `Players:GetUserThumbnailAsync(userId, Enum.ThumbnailType.HeadShot, Enum.ThumbnailSize.Size150x150)` renvoie `(contenu, prêt)`, sous `pcall`, image de repli tant que `prêt` est faux. Noms d'une page de classement en un appel : `UserService:GetUserInfosByUserIdsAsync(ids)` (`Id`, `Username`, `DisplayName`, `HasVerifiedBadge` ; ordre non garanti, ids invalides omis, 250 résultats par minute), sous `pcall`, en cache.
- Emojis : ils s'affichent dans un TextLabel (police emoji de Roblox, rendu différent de ton éditeur) et dépannent comme icônes provisoires ; ils ne se teintent pas.
- Glyphe Robux : le caractère privé `"\u{E002}"` s'affiche avec les polices Roblox, mais le rendu peut décevoir selon la taille. Version robuste : une ImageLabel de l'icône Robux et un TextLabel en rangée UIListLayout. Teste à l'œil.

## 9. Apparence : coins, contours, dégradés, ombres, découpe

Préfère les modificateurs aux images : nets à toutes les tailles, teintables, sans upload.

**UICorner**
- `CornerRadius` : `UDim.new(0, 8)` = 8 px ; `UDim.new(0.5, 0)` = pilule (cercle si l'objet est carré). Coins séparés `TopLeftRadius`, `TopRightRadius`, `BottomRightRadius`, `BottomLeftRadius` (sortis le 2026-06-23), par exemple pour une bulle de dialogue. `CornerRadius` écrit les quatre coins d'un coup : ne règle pas les deux dans une même règle de style. Il arrondit le fond sans découper les enfants ; sur une ImageLabel, il arrondit l'image elle-même.

**UIStroke**
- `ApplyStrokeMode` : `Border` = contour de la boîte ; `Contextual` = contour des lettres d'un objet texte. Sur TextLabel, TextButton et TextBox, le défaut entoure le texte : écris `Border` pour encadrer le bouton.
- **Plusieurs UIStroke par objet sont supportés** (doc « UI appearance modifiers », mise à jour du 2026-06-24). Empile-les par leur `ZIndex` ; à ZIndex égal, l'ordre est indéfini. Sur un bouton texte, un stroke `Border` et un stroke `Contextual` coexistent. Les guides « un seul UIStroke » sont périmés.
- `BorderStrokePosition` (`Outer` par défaut, `Center`, `Inner`) et `BorderOffset` (UDim, décale le trait) sont récents. La doc prévient que certaines propriétés de UIStroke demandent la bêta « Improved UIStrokes », sans dire lesquelles : vérifie-les dans un jeu publié. Un trait extérieur ou centré dépasse de la boîte ; prévois la place en grille serrée ou sous un parent qui découpe, sinon `Inner`. Liseré « premium » : stroke `Outer` sombre de 3 px plus stroke `Inner` blanc de 1,5 px à 0,6 de transparence.
- `StrokeSizingMode` : `FixedSize` (pixels, défaut) ou `ScaledSize`, où l'épaisseur est relative au plus petit côté du parent, ou à la taille de police sur un texte (doc), et s'exprime en fraction, d'ordre 0,02 à 0,08 (à vérifier). Sous une UIScale, `FixedSize` est déjà multiplié : restes-y.
- `LineJoinMode = Round` sur une forme arrondie : `Miter` avec un UICorner trace un contour carré. Un `UIGradient` enfant d'un `UIStroke` colore le trait. Ne tweene pas `Thickness` sur un texte : ça scintille et coûte cher ; anime `Transparency` ou `Color`. UIStroke remplace `BorderSizePixel` (laisse-le à 0) et `TextStroke*`.

**UIGradient**
- `Color`, `Transparency`, `Rotation` (90 = haut → bas), `Offset` (fraction de la taille ; l'animer fait un reflet qui balaie).
- **Le dégradé multiplie la couleur de l'objet.** Mets `BackgroundColor3` (ou `ImageColor3`, `TextColor3`) à blanc et les vraies couleurs dans le dégradé ; un fond coloré dessous teinte deux fois et assombrit. Variante : un dégradé blanc → gris clair sur une barre colorée donne du relief et laisse tweener la couleur du fond. Clair en haut, foncé en bas = relief ; inversé = pressé. Un UIGradient par objet ; les effets en plus vont dans des cadres enfants.
- Types radial, elliptique et conique (`Type`), `Scale`, `TileMode` : bêta Studio « Upgraded UIGradients » (doc, 2026-10-18), cachés de la fenêtre Propriétés (`NotBrowsable` dans le dump 0.743). Pour un jeu publié, reste en linéaire ; un radial se fait en image.

**UIShadow** (ombre native, sortie le 2026-06-23)
- Enfant de n'importe quel GuiObject : `Color`, `Transparency`, `BlurRadius` (UDim), `Offset` (UDim2), `Spread` (UDim2), `ZIndex` (négatif uniquement), `Enabled`. Elle suit l'UICorner et la rotation du parent ; plusieurs ombres possibles.
- Version sortie : ombre de la boîte, pas des lettres, sans ombre intérieure ni dégradé. La bêta « Upgraded UIShadows » (`Mode` Texte, `Inset`, `ShowBehindParent`, `UIGradient` enfant ; la doc ne dit pas lesquels exactement) n'est pas publiable : une écriture sans erreur ne prouve pas qu'elle est active. Ombre dure de style jeu : la « lèvre », un fond plus sombre qui dépasse de 4 à 5 px en bas (section 13), plus nette qu'une ombre floue.

**CanvasGroup et découpe**
- `ClipsDescendants` découpe au rectangle : il ignore l'UICorner et ne découpe pas les enfants tournés, sauf si `StarterGui.ClipsDescendantsSupportsRotation` est coché dans Studio, et jamais quand le parent qui découpe est lui-même tourné (doc). Garde rayons et rubans tournés hors des conteneurs qui découpent.
- `CanvasGroup` rend ses descendants dans une texture, puis applique `GroupTransparency` et `GroupColor3` au tout. Avec un `UICorner`, c'est le seul moyen intégré de découper au contour arrondi : reflet sur un bouton arrondi, contenu qui glisse dans un panneau, fondu d'une fenêtre entière sans artefacts. Une ImageLabel en mosaïque ignore l'UICorner de son parent : donne-lui son propre UICorner de même rayon, ou mets-la dans un CanvasGroup arrondi. Il coupe aussi ce qui déborde (contour `Outer`, ombre, badge) : rentre le contenu ou sors le badge du groupe.
- Coût : une texture par groupe, proportionnelle à sa taille à l'écran (≈ 8 Mo pour un plein écran 1080p, heuristique) ; le texte dedans peut sembler un peu flou. Le groupe n'aplatit son rendu que sous un ScreenGui en `ZIndexBehavior = Sibling` (doc). Pas d'imbrication, pas de groupe géant ni redimensionné en continu, une dizaine à l'écran au plus (heuristique). Au-delà du plafond mémoire fixé par la qualité graphique du client, le groupe s'affiche vide (doc). Des bugs de rendu sont signalés sur certains mobiles : teste sur un vrai téléphone. Un UIGradient sur un CanvasGroup s'applique au groupe entier (doc), pratique pour estomper les bords d'un carrousel.

**Le reste**
- Feuilles de style (`StyleSheet`, `StyleRule`, `StyleLink`, `StyleQuery`) : l'équivalent du CSS, édité dans le Style Editor de Studio. Un nom de condition de `StyleQuery` mal écrit échoue sans erreur : vérifie que `IsActive` bascule. Dans un projet React, garde un module de thème ; ne mélange pas les deux sur un même arbre.
- Pas de flou par élément : `BlurEffect` dans Lighting floute le monde 3D derrière une fenêtre ; pour un effet verre, une image pré-floutée. `UIBlur` existe dans le dump 0.742 sans doc : ne t'y fie pas.
- Pas de skew : incline de −3° à −5°, ou image de parallélogramme en 9-slice. Les ParticleEmitter 3D ne passent pas devant l'UI : pool d'ImageLabel animées, 30 à 80 actives au plus (heuristique), recyclées.

## 10. ViewportFrame et ScrollingFrame

**ViewportFrame** (pet qui tourne dans une carte, aperçu d'arme, silhouette)
- Il lui faut une `Camera` assignée à `CurrentCamera`, créée côté client : les Cameras ne se répliquent pas, et celle d'un template de StarterGui n'arrive pas forcément au client (seuls son `CFrame` et son `FieldOfView` voyagent avec le ViewportFrame, ce qui suffit à une vue fixe, doc). Studio convertit désormais une Camera collée sous un ViewportFrame en `ViewportCamera` (fin septembre 2026, comportement à vérifier).
- Ajoute un `WorldModel` (instance enfant, pas une propriété) seulement si le modèle joue des animations ou subit des requêtes physiques : il coûte plus cher.
- Ni ParticleEmitter, ni Beam, ni Trail, ni lumières, ni ombres ; Neon et Glass y sont rendus en basse qualité (doc). L'éclairage passe par `Ambient`, `LightColor`, `LightDirection` ; `ImageColor3` teinte le rendu. Retire scripts, émetteurs et lumières des copies d'aperçu.

```luau
--!strict
-- Cadre la caméra sur la boîte englobante du modèle, vu un peu d'en haut.
local function frameModel(viewport: ViewportFrame, model: Model, fov: number, yaw: number)
	local camera = viewport.CurrentCamera or Instance.new("Camera")
	camera.FieldOfView = fov -- 30 à 40 : peu de déformation
	camera.Parent = viewport
	viewport.CurrentCamera = camera
	local box, size = model:GetBoundingBox()
	local radius = size.Magnitude / 2 * 1.1 -- 10 % de marge
	local distance = radius / math.tan(math.rad(fov / 2))
	local look = (CFrame.Angles(0, math.rad(yaw), 0) * CFrame.Angles(math.rad(-12), 0, 0)).LookVector
	camera.CFrame = CFrame.lookAt(box.Position - look * distance, box.Position)
end
```

- Chaque ViewportFrame visible refait un rendu quand son contenu ou sa caméra change : trois ou quatre à la fois au plus sur téléphone (heuristique). Ne fais tourner que les visibles (`Visible` et `AbsoluteSize.X > 0`). Grille de 30 pets : des images, et un seul ViewportFrame pour l'objet sélectionné. Version React : ui-react.md, section 10.

**ScrollingFrame**
- Canevas qui suit le contenu : `AutomaticCanvasSize = Enum.AutomaticSize.Y`, `CanvasSize = UDim2.new()`, un layout dedans et un `UIPadding`. `ScrollBarThickness` 4 à 8, `VerticalScrollBarInset = Enum.ScrollBarInset.ScrollBar` (le contenu ne passe pas sous la barre), `ScrollingDirection = Y`, `ElasticBehavior = Never` pour couper le rebond tactile. `AbsoluteWindowSize` donne la zone visible sans les barres.
- Conflit connu : des enfants en Scale sur l'axe qui défile, ou une UIScale qui change, peuvent faire grandir le canevas sans fin ou clignoter les barres. Enfants en Offset ou en AutomaticSize, sinon pilote `CanvasSize` depuis `AbsoluteContentSize` du layout.
- Pas de défilement imbriqué sur le même axe (la molette remonte au parent). Au-delà d'une cinquantaine de lignes, virtualise (ui-react.md, section 9). Vérifie qu'on atteint le bas et qu'à la manette le défilement suit la sélection.

## 11. Boutons et entrées

**Boutons**
- `Activated` pour l'action : clic, tap, bouton A, Entrée sur l'objet sélectionné. `SecondaryActivated` : clic droit et appui long. Garde `MouseButton1Down/Up` pour l'état visuel « pressé ». `AutoButtonColor = false` dès que tu dessines tes états : sinon Roblox assombrit le bouton et se bat avec ton dégradé.
- États : repos, survol (souris seulement), pressé, désactivé, sélectionné (manette), en cours (achat qui attend le serveur), erreur. Pilote-les par une seule valeur d'état (code dans ui-react.md).
- Désactivé : un cadre noir à 0,5 de transparence par-dessus (avec son UICorner) ou un dégradé désaturé, et dis pourquoi (« Il te manque 120 pièces ») : un bouton muet ressemble à un bug. `Interactable = false` coupe les entrées ; l'aspect reste à dessiner (le grisage automatique dépend d'`AutoButtonColor`, à vérifier).
- `Active = true` sur un cadre absorbe les clics : un voile plein écran derrière une modale protège le HUD et le monde 3D. La propriété récente `InputSink` (`None`, `Activate`, `All` ; défaut `None`, et `Activate` sur un bouton) rend ce blocage explicite et plus fin ; elle n'est pas sauvegardée dans la place : règle-la par code (doc, 2026-10-18).
- `Modal` (propriété de bouton) libère le curseur quand la caméra est verrouillée (première personne, shift-lock). Il ne bloque pas les clics et ne piège pas la sélection manette.
- Zone de clic plus grande que le visuel (icône de 32 dans un bouton transparent de 44), sans chevaucher les voisines. Pendant une requête, bloque le bouton et réaligne-le sur la réponse du serveur : l'UI n'accorde jamais l'objet elle-même (monetisation.md).

**Entrée active**
- `UserInputService.PreferredInput` (`KeyboardAndMouse`, `Touch`, `Gamepad`, `MicroGamepad` pour une télécommande de TV) dit comment le joueur joue maintenant ; écoute `GetPropertyChangedSignal("PreferredInput")`. `TouchEnabled`, `KeyboardEnabled`, `GamepadEnabled` disent seulement quel matériel existe.
- Survol : réagis à `MouseEnter` seulement en `KeyboardAndMouse`, sinon un tap laisse l'élément grossi ; à la manette, montre le même état sur `SelectionGained`/`SelectionLost`. `MouseEnter` ne se réévalue qu'au mouvement de la souris : sous une liste qui défile, interroge `GetGuiObjectsAtPosition` (section 4).
- Indices : `UserInputService:GetImageForKeyCode` donne l'icône de la manette branchée ; pour une action de l'Input Action System, la doc conseille plutôt `InputActionLabel`, qui suit seul l'appareil et les touches réassignées. Rafraîchis tutoriels et indices quand l'entrée change ; jamais « Appuie sur E » pour un joueur au doigt.

**Tactile**
- Cibles ≥ 44 × 44 px rendus (repère Apple et WCAG AAA, pas une constante Roblox : la doc Roblox ne fixe aucune taille minimale, elle parle de zones réservées et de zones des pouces), écart ≥ 8 px, boutons de boucle ≥ 56 px de haut. Rien qui dépende du survol, du clic droit ou d'un glisser précis.
- Bouton d'action maison : ne déclenche que pour un toucher commencé sur lui (suis son InputObject jusqu'au bout) ; un glisser de caméra qui passe dessus ne fait rien. Retour visuel sur le bouton même, que le pouce ne cache pas ; cache-le quand l'action est impossible.
- `ContextActionService:BindAction(name, handler, true, ...)` crée un bouton tactile (7 au plus, doc), réglé par `SetTitle`, `SetImage`, `SetPosition` et retiré par `UnbindAction`. L'Input Action System (`InputContext`, `InputAction`, `InputBinding`) mappe une action sur clavier, manette et tactile d'un coup (gameplay.md). `GuiService.TouchControlsEnabled = false` masque joystick et saut de Roblox si tu fournis les tiens.

**Manette**
- Chaque contrôle est `Selectable`. À l'ouverture, une frame après l'affichage, mets `GuiService.SelectedObject` sur l'action sûre la plus probable, jamais un achat ni une suppression ; à la fermeture, rends la sélection d'avant ou `nil`.
- Piège le focus : `SelectionGroup = true` sur la racine de la fenêtre et `SelectionBehaviorUp/Down/Left/Right = Enum.SelectionBehavior.Stop`, sinon il file vers le HUD. `GuiService:AddSelectionParent` et ses cousines sont dépréciées. Navigation ambiguë : `NextSelectionUp/Down/Left/Right`. `SelectionOrder` ne touche pas la navigation par direction : il choisit seulement l'objet pris en premier au début de la sélection ou par `GuiService:Select(parent)` (doc).
- Une sélection oubliée bloque le jeu : tant que `SelectedObject` est défini, A et les gâchettes pilotent l'UI au lieu de sauter. Vide-la à la reprise. B ferme un calque à la fois. Surbrillance par défaut qui jure : `SelectionImageObject` vers un GuiObject de ton style ; ne supprime jamais le retour de focus.

```luau
--!strict
local GuiService = game:GetService("GuiService")
local RunService = game:GetService("RunService")
local UserInputService = game:GetService("UserInputService")

local previous: GuiObject? = nil

local function openWindow(window: GuiObject, firstButton: GuiObject)
	window.SelectionGroup = true
	window.SelectionBehaviorUp = Enum.SelectionBehavior.Stop
	window.SelectionBehaviorDown = Enum.SelectionBehavior.Stop
	window.SelectionBehaviorLeft = Enum.SelectionBehavior.Stop
	window.SelectionBehaviorRight = Enum.SelectionBehavior.Stop
	window.Visible = true
	local input = UserInputService.PreferredInput
	if input ~= Enum.PreferredInput.Gamepad and input ~= Enum.PreferredInput.MicroGamepad then
		return -- pas de sélection forcée au doigt ni à la souris
	end
	previous = GuiService.SelectedObject
	RunService.PreRender:Once(function() -- l'objet doit être rendu pour être sélectionnable (PreRender : ex-RenderStepped)
		if window.Visible then
			GuiService.SelectedObject = firstButton
		end
	end)
end

local function closeWindow(window: GuiObject)
	window.Visible = false
	local back = previous
	previous = nil
	GuiService.SelectedObject = if back and back:IsDescendantOf(game) and back.Visible then back else nil
end
```

**Prompts, saisie, glisser**
- `ProximityPrompt` (« Appuie sur E » dans le monde) : `ActionText`, `ObjectText`, `HoldDuration`, `MaxActivationDistance`, `RequiresLineOfSight`, `Exclusivity` ; il s'adapte seul au tactile et à la manette. Style maison : `Style = Enum.ProximityPromptStyle.Custom` et ton UI sur `ProximityPromptService.PromptShown`/`PromptHidden`. `Triggered` est une affirmation du client : le serveur revérifie distance et état (securite.md). Éloigne les prompts d'achat des chemins de passage.
- `TextBox` : `FocusLost(enterPressed)` pour valider, `ClearTextOnFocus = false` sur un champ éditable ; `ReturnPressedFromOnScreenKeyboard` peut aussi partir sur téléphone et console : un seul chemin de soumission, dédoublonné. Tes raccourcis ignorent les entrées déjà consommées (`gameProcessedEvent`).
- `UIDragDetector` fait sliders et panneaux déplaçables : `DragStyle = TranslateLine`, `ResponseStyle = Scale`, `BoundingUI` = la piste. Il ne détecte pas de cible de dépôt : à `DragEnd`, teste toi-même les rectangles.

## 12. UI dans le monde

- **BillboardGui** (noms, barres de vie, dégâts) : `Size` en Scale = studs (rapetisse avec la distance), en Offset = pixels constants. `StudsOffset` suit la caméra, `StudsOffsetWorldSpace` le monde ; `Adornee` = BasePart ou Attachment. `AlwaysOnTop = true` traverse les murs : réserve-le aux marqueurs voulus. Règle `MaxDistance`, pool pour les dégâts flottants, création côté client pour ce que seul ce joueur voit : des centaines de billboards actifs coûtent vraiment.
- **SurfaceGui** (classement, prix d'un bâtiment, écran) : `Face`, `SizingMode = PixelsPerStud` avec ≈ 50 pixels par stud, enfants en Scale, `LightInfluence = 0` pour un écran que la scène n'éclaire pas. `MaxDistance` 60 à 120 studs : 60 panneaux à distance infinie ont fait ramer une vraie scène. Change l'`Adornee` plutôt que de reparenter.
- Boutons cliquables seulement si la GUI est sous le PlayerGui du joueur (avec `Adornee` vers la pièce) ; un SurfaceGui demande **en plus** `CanQuery = true` sur la pièce, un BillboardGui `Active = true` sur lui **et** sur le bouton (doc). Un LocalScript ne tourne pas dans Workspace : range ces GUI dans StarterGui.
- La capture du MCP ne montre ni SurfaceGui ni BillboardGui : vérifie par code ou par une capture de la fenêtre Studio (studio-mcp.md).
- Pièce qui vole du monde vers ton compteur : `Camera:WorldToViewportPoint` donne le départ en pixels du viewport ; anime une ImageLabel dans une couche plein écran (`ScreenInsets = None`). Moins cher et plus net qu'une pièce physique. Convertis la position du compteur, qui vit dans une couche aux insets différents (`GuiService:GetInsetArea`, section 3 ; à vérifier sur un téléphone à encoche).

## 13. Design d'UI de jeu

**Méthode**
1. Inventorie l'existant (polices, couleurs, rayons, contours, icônes, espacements, durées) et étends ce langage. Sinon, crée un module de tokens avant le premier écran.
2. Construis la hiérarchie (régions, layouts, contraintes), vérifie qu'elle se lit à taille téléphone, puis branche l'état et les animations.
3. Construis tous les états de chaque surface : chargement, vide, plein, erreur, verrouillé, désactivé.
4. Regarde au-dessus d'une scène claire et d'une scène sombre, pas sur le fond vide de Studio.

**Tokens** : un module unique, gelé (`table.freeze`), lu par toute l'UI : couleurs par rôle, espacements, rayons, épaisseurs, polices, tailles, durées. Aucun `Color3.fromRGB` en ligne dans un composant. Raretés et couleurs de fonction à un seul endroit ; une couleur par palier, jamais réutilisée ailleurs.

| Token (unités de design) | Valeurs |
|---|---|
| Espacement | 4, 8, 12, 16, 24, 32 |
| Rayon | 8 (boutons, cartes), 14 (fenêtres), pilule `UDim.new(0.5, 0)` |
| Contour | 2,5 (texte), 3 (boutons, cartes), 4 (fenêtres) |
| Texte | 12 légende, 14 corps, 18 bouton, 22 titre de carte, 28 titre de fenêtre, 36+ gros chiffre |

- Hiérarchie par la taille, la graisse, le contraste et la position avant la décoration. Un point focal par écran ; jamais deux boutons primaires côte à côte. Groupe par l'espace : un séparateur est un dernier recours.

**Familles de style** : une par jeu, tenue partout.

| Famille | Rayon | Contours | Polices | Signature |
|---|---|---|---|---|
| Simulator | 8 à 14 | sombres et épais (3 à 4), texte contouré partout | FredokaOne, LuckiestGuy, Montserrat Heavy | dégradés saturés, gros chiffres, badges, rayons derrière les récompenses |
| Cartoony | 12 à 20, pilules | teinte foncée du fond, 2 à 3 | FredokaOne, Nunito | crème et brun chaud au lieu du blanc et du noir, rebonds |
| Stud (blocs) | 0 à 4 | noirs 3 à 4, `Miter` | Montserrat, BuilderSans ExtraBold | picots en mosaïque, lèvre 3D |
| Minimal | 6 à 10 | 1 px peu contrasté | BuilderSans, Roboto | un seul accent, aucun rebond (compétitif, horreur) |

### Recette « simulator » chiffrée

Unités de design (échelle 1 sur le plus petit téléphone, ≈ 1,35 sur PC). Conventions tirées de jeux livrés, pas des règles Roblox. Dégradé haut → bas en `Rotation = 90` ; contour = la même teinte presque noire ; lèvre = la couleur du bas assombrie de 35 %.

| Fonction | Haut | Bas | Contour |
|---|---|---|---|
| Acheter, confirmer | `#86F75A` | `#2DB83A` | `#0D4A14` |
| Action principale, offre | `#FFB347` | `#F06A12` | `#5E2600` |
| Premium, Robux, vendre | `#FFE66B` | `#F2A40C` | `#6A3A00` |
| Rebirth, rare | `#CF7BFF` | `#8A33EA` | `#330B66` |
| Onglets, info | `#5CC0FF` | `#1F6FEA` | `#0A2A6E` |
| Fermer, danger | `#FF6464` | `#D41E2E` | `#4A0006` |
| Corps de fenêtre | `#30343F` | `#1F222B` | `#0E0F14` |

Raretés : Commun `#B5B5B5`, Peu commun `#54D85A`, Rare `#3D8BFF`, Épique `#A74BFF`, Légendaire `#FFC21F`, Mythique `#FF3D7F`, Secret = dégradé arc-en-ciel dont la `Rotation` tourne.

Bouton en relief :

```
Button (TextButton, Text = "", AutoButtonColor = false) : fond = couleur de lèvre
├─ UICorner 10
├─ UIStroke Border 3, couleur « contour »          (entoure face et lèvre)
└─ Face (Frame) : Size (1, 0, 1, -5), fond blanc
   ├─ UICorner 10
   ├─ UIGradient « haut » → « bas », Rotation 90
   ├─ UIStroke Border, BorderStrokePosition Inner, 1,5, blanc, Transparency 0,6
   ├─ Gloss (Frame blanc) : AnchorPoint (0.5, 0), Position (0.5, 0, 0, 3),
   │  Size (1, -10, 0.4, 0), Transparency 0,75, UICorner 6
   └─ Label (TextLabel, ZIndex 2) : FredokaOne blanc, TextScaled, contrainte 12 à 24
      └─ UIStroke Contextual 2,5, couleur « contour »
Appui : Face descend de 3 (la lèvre s'amincit), retour avec un petit rebond (ui-react.md).
```

Sans reflet, le libellé peut rester le texte du bouton : un stroke `Border` et un stroke `Contextual` sur le même TextButton suffisent.

| Élément | Taille | Détails |
|---|---|---|
| Bouton principal | 200 × 56 | libellé 22 à 24, lèvre 5, reflet |
| Bouton secondaire | 140 × 44 | libellé 18 |
| Fermer (X) | 40 × 40, zone de clic 48 | rouge, ancré `(1, 0)` à `(1, 8, 0, -8)`, déborde du coin de la fenêtre |
| Bouton latéral | 60 × 60 | icône à 70 %, libellé 13 contouré qui déborde en bas, colonne à écart 8 ; au-delà de 6, menu « Plus » |
| Pilule de monnaie | 160 × 38 | fond sombre à 0,15, pilule, icône de 44 qui déborde à gauche, montant 20 contouré, « + » vert à droite ; variante plus légère : icône et gros nombre contouré sans fond |
| Fenêtre | 760 × 520 au plus | corps sombre ; en-tête de 48 à la couleur du bouton qui l'ouvre, plus large de 3 de chaque côté et remonté de 6, seuls ses coins du haut arrondis (rayons par coin) ; titre « sticker » possible : grosse icône tournée de −8° qui déborde du coin, titre posé sur le bord haut (AnchorPoint Y 0,5 à Position Y 0) |
| Carte de grille | 120 × 150 | fond sombre, bandeau de rareté de 22, image ou 3D, prix en bas ; la carte entière est la cible, le gros bouton d'achat vit dans le détail |
| Badge de notification | 20 × 20 | rond rouge, contour blanc 2, « ! » ou nombre, ancré `(1, 0)` au coin, déborde de 6 |
| Ruban « MEILLEURE OFFRE » | hauteur 22 | or ou rouge, rotation −8° à −12° sur le coin d'une carte |
| Toast | 260 × 32 | fond sombre, contour à la couleur du type, 1,5 s, deux empilés au plus |

- Texte toujours contouré (2,5 en `Contextual`), blanc par défaut ; prix en or `#FFE15A`, gains en vert `#7BFF7B`. Gros chiffres abrégés (section 7) et qui défilent (ui-react.md).
- Une couleur de bouton = une fonction, sur tous les écrans ; le bouton d'achat est au même endroit partout.
- Prix lus à l'exécution (`MarketplaceService:GetProductInfoAsync`), jamais codés en dur ; probabilités affichées avant tout achat aléatoire (monetisation.md).
- Une « meilleure offre » par écran au plus (ruban et reflet) : si tout brille, rien ne ressort. Rayons derrière une récompense : ImageLabel carrée teintée, transparence 0,25, un tour en 10 s.
- Pas assez de monnaie : texte rouge et petit tremblement, pas une invite d'achat surprise.

**Éviter le « look IA »** : rejette
- coins arrondis sur chaque cadre imbriqué, cartes dans des cartes, pilules partout ; dégradés, lueurs, néons et ombres posés au hasard (dans un simulator ils sont la règle, mais avec la même pile partout) ;
- trois polices, plus de deux rayons, des écarts hors de l'échelle ; titres énormes, grands panneaux vides, icônes décoratives sans sens ;
- toutes les actions au même poids visuel, une ombre sous chaque carte, de la transparence ou du flou partout, `BorderSizePixel = 1`, du gris sur du gris ;
- un bouton désactivé muet, un état vide sans message, une erreur signalée par la seule couleur, un changement d'état sans transition.

## 14. Accessibilité et vérification

**Accessibilité**
- `GuiService.ReducedMotionEnabled` : si vrai, coupe déplacements et échelles ; garde fondus et compteurs.
- `GuiService.PreferredTextSize` (`Medium` par défaut, puis `Large`, `Larger`, `Largest`) : le moteur grossit lui-même le texte en `TextSize` fixe, et `AutomaticSize` agrandit la boîte ; un texte `TextScaled` n'en tient pas compte et une `UITextSizeConstraint` le plafonne (doc). Préfère donc `TextSize` + `AutomaticSize` au lieu de deviner depuis les pixels. `GuiService.PreferredTransparency` : multiplie la transparence de tes fonds par cette valeur (plus basse, le joueur veut des fonds plus opaques). `StyleQuery` applique sans code les conditions `PreferredTextSize`, `ReducedMotionEnabled`, `PreferredInput` et `ViewportDisplaySize`, pas `PreferredTransparency` (doc).
- Contraste WCAG : 4,5:1 pour le texte courant, 3:1 pour le gros texte (≥ 24 px, ou ≥ 18,7 px en gras) et les contours de focus. Sur fond chargé : texte blanc et contour sombre, ou plaque derrière.
- Jamais l'information par la couleur seule : icône + libellé + couleur, pour que les raretés restent lisibles par les daltoniens.
- Texte rendu ≥ 12 px sur le plus petit téléphone (jamais < 9), cibles ≥ 44 px, focus manette visible. Pas de clignotement au-delà de 3 Hz ; une option réduit flashs et tremblements d'écran. Info sonore doublée d'un indice visuel ; « Passer » sur les cinématiques. Public jeune : pictogrammes plutôt que phrases.

**Matrice d'appareils** (Studio : Test → Device pour le Device Emulator, ou la barre « Device Simulator » au-dessus du viewport, en bêta « New Device Simulator » (doc, 2026-10-18) ; Controller Emulator pour la manette)

| Cible | Taille (points) | À regarder |
|---|---|---|
| Petit téléphone paysage (iPhone SE) | 667 × 375 | joystick, saut, barre du haut, texte ≥ 12, tout tient |
| Téléphones courants | 780 × 360 (Android), 844 × 390 (iPhone 14) | encoche, barre d'accueil, coins |
| Tablette | 1024 × 768, 960 × 600 | l'UI grandit au lieu de s'étirer, pas de grand vide |
| PC | 1366 × 768, 1920 × 1080, ultra-large | survol, taille raisonnable |
| Console, TV | 1080p et 4K, manette Xbox puis PlayStation | focus de départ, 4 directions, Retour, lecture à 3 m, overscan |
| Portrait, si autorisé | 360 × 640 | reflow |

- Vérifie que `PreferredInput` vaut `Touch` avant de juger une mise en page tactile : les contrôles tactiles se cachent si la session croit à une souris, et l'émulateur peut annoncer un clavier.
- Mesure, puis regarde : lis `AbsoluteSize`, `AbsolutePosition` et la taille de texte rendue par code (MCP `execute_luau`), puis capture ; une capture peut être périmée, la mesure non. L'outil MCP `skill` sert un skill intégré `rbx-device-simulator-lua` sur les formats d'appareil (studio-mcp.md).
- Teste les états : fenêtre ouverte, mort et respawn, deux toasts, chat ouvert, TextBox avec clavier à l'écran, liste vide et pleine, nom très long, traduction longue, image qui ne charge pas. L'émulateur ne remplace ni un vrai téléphone (pouce, encoche) ni une vraie TV (overscan) : dis ce qui n'a été vu qu'en émulation.

**Test de fuite à la réouverture**

```luau
--!strict
-- Ouvre et ferme un écran 10 fois : le nombre de descendants doit revenir au départ.
local function leakTest(container: Instance, open: () -> (), close: () -> ())
	local before = #container:GetDescendants()
	for _ = 1, 10 do
		open()
		task.wait(0.5)
		close()
		task.wait(0.5)
	end
	task.wait(1) -- laisse finir les animations de sortie
	local after = #container:GetDescendants()
	print(`[leakTest] avant {before}, après {after} :`, if after > before then "FUITE" else "ok")
end
```

- Recommence en fermant pendant l'animation d'ouverture et pendant une réponse réseau. Surveille aussi la mémoire dans la console développeur (F9).

## 15. Pièges

- UI qui disparaît ou se duplique au respawn : `ResetOnSpawn` resté vrai, ou ScreenGui rangé dans un Folder.
- Bouton sous l'encoche ou la barre du haut : couche interactive en `None` ou en `IgnoreGuiInset`. Marge doublée : `ScreenInsets` plus padding manuel.
- Position ignorée : l'enfant est placé par un layout. Tooltip caché derrière une fenêtre : il est imbriqué au lieu d'être à la racine. Texte microscopique sur téléphone : TextScaled sans `UITextSizeConstraint`, ou design pensé en 1080p puis réduit.
- `AbsoluteSize` à 0 : lu juste après la création. Grille qui déborde sous une UIScale : Offset déduit d'un AbsoluteSize sans diviser par l'échelle.
- Contour autour du texte au lieu du bouton : `ApplyStrokeMode` resté `Contextual`. Couleurs ternes : fond coloré sous un UIGradient. Coins qui dépassent : ClipsDescendants, ou mosaïque sans UICorner propre ni CanvasGroup.
- Image vide : asset modéré, id de Decal au lieu d'Image, image d'un autre compte. Halo noir : pas d'alpha bleed. Survol collé au doigt : survol non filtré par `PreferredInput`. Manette bloquée : pas de `SelectedObject` à l'ouverture, ou sélection jamais vidée.
- Clics à travers une modale : voile sans `Active = true`. Bouton de SurfaceGui muet : GUI hors PlayerGui ou pièce en `CanQuery = false`.
- API dépréciées des vieux tutoriels : `TweenPosition`, `TweenSize`, `GuiObject.Transparency`, `Draggable`, `BackgroundColor`/`BorderColor` (BrickColor), `FontSize`, `AddSelectionParent`, `Player:LoadCharacter` ; retirée : `Instance.new("AnimatedImage")`. Non dépréciées mais à éviter : propriété `Font` (préfère `FontFace`), `TextStroke*` (préfère UIStroke), `IsTenFootInterface` (préfère `ViewportDisplaySize`).
- Noms inventés ou non vérifiés vus dans des guides : propriété `ViewportFrame.WorldModel` (c'est une instance enfant), classe `UIFlexLayout`, propriété `MouseCursor` sur les boutons. `GuiService:GetInsetArea(insets)`, elle, existe (doc).

## 16. Checklist

- [ ] Chaque ScreenGui : `ResetOnSpawn = false` s'il persiste, `ZIndexBehavior = Sibling`, `DisplayOrder` de sa couche, `ScreenInsets` choisi.
- [ ] Rien d'interactif hors `CoreUISafeInsets`, ni sur le joystick, le saut ou la barre du haut ; pas de double marge.
- [ ] Régions en Scale + AnchorPoint assorti ; détails en Offset ; formes verrouillées par `UIAspectRatioConstraint`.
- [ ] Enfants répétés placés par un layout avec `SortOrder = LayoutOrder` et un `UIPadding`.
- [ ] Une valeur d'échelle commune par UIScale ; fenêtres bornées à la zone utile ; reflow sur petit écran.
- [ ] Texte en `FontFace`, ≥ 12 px rendus sur 667 × 375, contour `Contextual`, contrainte sur chaque `TextScaled`.
- [ ] Images ≤ 1024 px, PNG avec alpha bleed, teintables, repli si l'asset ne charge pas.
- [ ] `ApplyStrokeMode = Border` sur les boutons texte ; ZIndex distincts entre UIStroke empilés.
- [ ] Boutons : `Activated`, `AutoButtonColor = false`, états repos, survol, pressé, désactivé, en cours.
- [ ] Survol réservé à la souris ; manette : focus d'entrée, `SelectionGroup`, Retour, sélection vidée à la fermeture.
- [ ] Cibles ≥ 44 px, écart ≥ 8 px ; `ReducedMotionEnabled`, `PreferredTextSize`, `PreferredTransparency` respectés.
- [ ] Un seul style : palette en tokens, deux rayons, deux épaisseurs, deux polices ; aucun critère du « look IA ».
- [ ] Matrice d'appareils parcourue (au moins 667 × 375 tactile et 1920 × 1080 souris), états compris ; ce qui n'a été vu qu'en émulation est dit.
- [ ] Test de fuite : 10 ouvertures et fermetures reviennent au compte de départ.
