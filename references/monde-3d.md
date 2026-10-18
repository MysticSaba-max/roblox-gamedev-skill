# Monde 3D : construire, importer, éclairer

À lire quand : tu construis un décor, un objet ou une carte par code ou par le MCP de Studio, tu fais un modèle en voxels, tu importes un mesh (Blender, FBX, glTF, IA), ou tu règles l'éclairage, le terrain, les collisions ou le streaming d'une scène. La perf détaillée est dans performance.md, les outils MCP dans studio-mcp.md, l'upload par API dans open-cloud.md.

## Sommaire

1. Règles de base
2. Construire par code ou par le MCP
3. Parts, MeshParts, unions et CSG
4. Orientation
5. Échelle, matériaux, style
6. Voxels
7. EditableMesh et EditableImage
8. Importer des assets
9. Éclairage et ambiances
10. Terrain
11. Collisions et réglages réservés à l'édition
12. Streaming et niveaux de détail
13. Organisation de l'espace de travail
14. Pipeline IA : générer, valider, relire
15. Pièges
16. Checklist

## 1. Règles de base

1. **Le monde se relit, il ne se devine pas.** Après chaque lot, relis positions, tailles et comptes dans le Workspace (print), jamais dans ta mémoire.
2. **Dimensions nommées, placement relatif.** Toutes les cotes en variables en tête de script ; chaque sous-pièce placée depuis le CFrame d'une pièce d'ancrage, pour que le modèle se déplace d'un bloc.
3. **Petits lots vérifiés.** Une phase par appel, 10 à 30 parts composées à la main, puis relecture.
4. **Statique = ancré**, collision simple, `CanTouch` et `CanQuery` coupés sur le décor.
5. **Parent en dernier** : règle les propriétés, puis parente (une seule réplication, aucun événement inutile).
6. **Un style par jeu**, couleurs et matériaux voulus : le gris `Plastic` par défaut signale un travail inachevé.
7. **Parts pour bloquer, MeshParts pour le détail**, unions avec parcimonie.
8. **Prouve** : comptes, bornes, capture avec une caméra choisie, traversée depuis le spawn.

## 2. Construire par code ou par le MCP

- `execute_luau` est sans état : variables et références disparaissent entre deux appels. Ré-acquiers la racine au début de chaque appel (`FindFirstChild`, sinon création) et rends l'appel rejouable : il remplace ce qu'il a construit sous sa propre racine au lieu de le dupliquer. Détruire autre chose dans la place demande d'abord l'accord de l'utilisateur (SKILL.md, niveaux de risque).
- Construis en mode Edit : ce qui est créé pendant un Play disparaît à l'arrêt. Chaque `execute_luau` laisse un point d'annulation, mais ne compte pas sur Ctrl+Z : fais sauvegarder avant un gros lot (studio-mcp.md).
- Phases qui marchent : squelette (volumes principaux), puis détails, puis vérification. Ce schéma a donné zéro erreur d'orientation en tests A/B ; « tout en un appel » donne peu de détails, « pas à pas de mémoire » fait dériver les coordonnées.
- Taille des lots : 10 à 30 parts par appel pour une construction composée par l'IA. Pour du code généré par un algorithme (voxels, grilles), un appel peut créer ~10 000 parts en moins de 2 s (≈ 180 000 caractères au plus par appel, mesuré en octobre 2026). Ponctue les longues boucles de `task.wait()` si Studio gèle, et découpe : un appel de plus de ~30 s échoue (studio-mcp.md).
- Étalonne au départ : pose une part test et imprime sa position (une dérive de plus de 100 studs est passée inaperçue en test). Arrondis les cotes à une grille (0,125, 0,25 ou 0,5 stud) : des valeurs comme 0,333 ou 1,17 se cumulent en jours visibles.
- Écris le plan des hauteurs (Y) avant toute structure à étages. La longueur des pieds se calcule depuis l'écart réel entre le sol et le dessous du plateau, jamais en constante, sinon ils traversent le sol ou flottent.
- Sans MCP, un builder Luau synchronisé par Rojo et lancé depuis la barre de commande fait le même travail ; le blueprint en données reste dans Git (outils.md).

```luau
--!strict
-- Tel quel dans execute_luau (Edit) : ré-acquiert la racine, reconstruit, relit.
local Workspace = game:GetService("Workspace")

local DEF = { Width = 6, Depth = 3, Height = 3, Top = 0.25, Leg = 0.5, Inset = 0.25 } -- manifeste, en studs

local root: Instance = Workspace:FindFirstChild("Build") or Instance.new("Folder") -- annoté, sinon l'ancien solveur refuse `root.Name = …`
root.Name = "Build"
root.Parent = Workspace

local function part(name: string, size: Vector3, cframe: CFrame, parent: Instance): Part
	local p = Instance.new("Part")
	p.Name = name
	p.Anchored = true
	p.Size = size
	p.CFrame = cframe
	p.Material = Enum.Material.WoodPlanks
	p.Color = Color3.fromRGB(150, 105, 70)
	p.Parent = parent
	return p
end

local old = root:FindFirstChild("Table")
if old then old:Destroy() end -- rejouer l'appel remplace, ne duplique pas
local model = Instance.new("Model")
model.Name = "Table"
local top = part("Top", Vector3.new(DEF.Width, DEF.Top, DEF.Depth), CFrame.new(0, DEF.Height - DEF.Top / 2, 0), model)
local legLength = DEF.Height - DEF.Top -- du sol au dessous du plateau
for i, corner in { Vector2.new(1, 1), Vector2.new(-1, 1), Vector2.new(1, -1), Vector2.new(-1, -1) } do
	local x = corner.X * (DEF.Width / 2 - DEF.Inset - DEF.Leg / 2)
	local z = corner.Y * (DEF.Depth / 2 - DEF.Inset - DEF.Leg / 2)
	-- relatif au plateau : déplacer le plateau emporte les pieds
	part(`Leg{i}`, Vector3.new(DEF.Leg, legLength, DEF.Leg), top.CFrame * CFrame.new(x, -(DEF.Top + legLength) / 2, z), model)
end
model.PrimaryPart = top
model.Parent = root

local box, size = model:GetBoundingBox() -- vérité terrain : relis ce qui existe
print(`Table : {#model:GetChildren()} parts, centre {box.Position}, taille {size}`)
```

## 3. Parts, MeshParts, unions et CSG

**Parts**
- Défauts piégeux : `Anchored = false` (ça tombe), gris 163, 162, 165 en `Plastic`. Écris toujours ancrage, couleur et matériau.
- Décor : `CanCollide = false` sur le petit décor, `CanTouch` et `CanQuery` à `false` sur ce que rien ne touche ni ne vise, `CastShadow = false` sur les petites pièces et les déclencheurs invisibles.
- Mécanisme soudé et mobile : une racine, les autres pièces soudées (`WeldConstraint`) et non ancrées ; ancrer plusieurs pièces fige ou scinde l'assemblage. `PivotTo`/`GetPivot` remplacent `SetPrimaryPartCFrame`. Formes : `Part.Shape` (`Block`, `Ball`, `Cylinder`, `Wedge`, `CornerWedge`), `WedgePart`, `TrussPart` (escalade).

**MeshParts**
- Pour le décor détaillé, préfère un MeshPart à une union : appels de dessin instanciés, `SurfaceAppearance` (PBR), fidélités de collision et de rendu réglables.
- L'instanciation ne regroupe que les copies qui partagent asset, `SurfaceAppearance` et matériau. Importe chaque asset une fois (ou en Package) et duplique-le ; importer une carte entière d'un bloc crée un id par morceau et casse l'instanciation.
- `MeshId` ne s'écrit pas au runtime. Pour changer de mesh ou de fidélité : `AssetService:CreateMeshPartAsync(content, options)` puis `part:ApplyMesh(nouveau)`. Les références d'assets migrent vers des propriétés `Content` (`MeshContent`, `TextureContent`…), mais une place de 2026 garde souvent les anciens noms (`MeshId`, `TextureID`) : un scanner d'assets cherche les deux.

**Unions et CSG**
- Studio : Union, Negate, Intersect. Runtime : `BasePart:UnionAsync`, `SubtractAsync`, `IntersectAsync` (un seul résultat), ou les mêmes méthodes sur `GeometryService` (`GeometryService:SubtractAsync(part, parts, options)`, forme des exemples de la doc), qui renvoient un tableau de parts ; tous yield et peuvent échouer.
- Les LLM réussissent mal les CSG en chaîne : ≥ 80 % de réussite pour 1 ou 2 opérations, 40 à 60 % pour 3 ou 4, 3 à 30 % au-delà (benchmark CadEval, avril 2025). Fais 3 à 4 opérations au plus par lot, vérifiées.
- Arbre peu profond : le volume convexe dominant, puis les trous, puis les saillies. Ne soustrais pas plusieurs fois d'une pièce déjà unie ; même matériau sans intersection complexe : un simple Model suffit.
- Epsilon : un cutter dépasse de ~0,05 stud les faces qu'il traverse ; des faces coplanaires laissent du z-fighting ou une peau de matière. Fais la CSG près de l'origine, puis `PivotTo` vers la destination : la précision se dégrade loin de l'origine.
- Un creux réel (évier, bassin, encastrement de 0,2 à 0,3 stud) se fait par soustraction : un bloc posé dessus ne se lit pas comme un creux. Arche en plein cintre : centre du cylindre cutter sur la ligne de naissance de l'arc.
- CSG sur MeshPart : beta depuis le 2026-03-12 (File → Beta Features → Solid Modeling On Meshes, doc, 2026-10-18), mesh étanche exigé, sinon l'appel échoue ; pour un jeu, reste sur Part et PartOperation. Jamais sur le Terrain. Pas de congé ni de surface organique : style low-poly assumé, ou MeshPart importé (CSG pour le structurel, mesh pour l'organique).
- Pré-unis dans Studio plutôt qu'au runtime ; `CollisionFidelity = Box` sur les unions décoratives ; une union lourde et finie peut être exportée puis réimportée en mesh.

```luau
--!strict
-- Creuse `base` avec des cutters (qui dépassent de 0,05 stud) ; renvoie la base intacte si ça échoue.
local function carve(base: BasePart, cutters: { BasePart }): BasePart
	local ok, result = pcall(function() -- closure : la forme pcall(base.SubtractAsync, …) échoue au nouveau solveur
		return base:SubtractAsync(cutters)
	end)
	for _, cutter in cutters do
		cutter:Destroy() -- toujours, même en cas d'échec
	end
	if not ok or typeof(result) ~= "Instance" or not result:IsA("BasePart") then
		warn("carve a échoué :", result)
		return base
	end
	result.CFrame = base.CFrame -- le CFrame complet : CFrame.new(base.Position) perdrait la rotation
	result.UsePartColor = true -- sinon chaque face garde sa couleur d'origine : les faces creusées prennent celle du cutter
	result.Color = base.Color
	result.Material = base.Material
	result.Anchored = base.Anchored
	result.Name = base.Name
	result.Parent = base.Parent
	base:Destroy()
	return result
end
```

Parente les cutters au modèle en construction : laissés dans Workspace, ils échappent au scan final. Relis la boîte englobante après chaque opération.

## 4. Orientation

Première source d'erreurs des constructions par IA.
- **Cylindre** : son axe suit X. Longueur = `Size.X`, diamètre = min(`Size.Y`, `Size.Z`). Debout : `CFrame.Angles(0, 0, math.rad(90))`, et sa hauteur monde reste `Size.X`.
- **`CFrame.lookAt(pos, cible)`** pointe l'axe −Z (`LookVector`) vers la cible. Mets donc la grande dimension d'une poutre sur Z ; un cylindre entre deux points demande en plus `* CFrame.Angles(0, math.rad(90), 0)`.
- **WedgePart et panneaux plats** (voile, fenêtre) : leur orientation par défaut n'est pas intuitive. Pose un exemplaire, imprime son `LookVector` et regarde où monte la pente avant d'en générer en série. 180° autour de Y retournent la pente ; deux wedges dos à dos font un toit à deux pans.
- **Sièges** : imprime le `LookVector` de chaque `Seat` en fin de construction et vérifie qu'un avatar s'assoit dans le bon sens. **Mesh importé** : mesure ses bornes et sa face avant avant d'y poser un décalque ; un emblème a déjà atterri au dos d'un coffre.
- Commenter l'orientation ne corrige rien ; hériter des CFrames (sous-pièce = ancre × décalage) la corrige. Au-delà de ~300 parts, des erreurs d'angle subsistent : nomme et imprime les pièces orientées, ou demande une capture.

```luau
--!strict
-- Poutre (longueur sur Z) ou cylindre (longueur sur X) entre deux points.
local function between(a: Vector3, b: Vector3, thickness: number, cylinder: boolean): Part
	local direction = b - a
	local up = if math.abs(direction.Unit.Y) > 0.99 then Vector3.xAxis else Vector3.yAxis -- évite un lookAt vertical dégénéré
	local cframe = CFrame.lookAt((a + b) / 2, b, up)
	local p = Instance.new("Part")
	p.Anchored = true
	if cylinder then
		p.Shape = Enum.PartType.Cylinder
		p.Size = Vector3.new(direction.Magnitude, thickness, thickness)
		p.CFrame = cframe * CFrame.Angles(0, math.rad(90), 0) -- l'axe X vient sur la direction
	else
		p.Size = Vector3.new(thickness, thickness, direction.Magnitude)
		p.CFrame = cframe
	end
	return p
end
```

## 5. Échelle, matériaux, style

- 1 stud = 28 cm (doc « units »). Avatar classique ≈ 4,75 à 5 studs, Rthro 5,75 à 6,5 : dimensionne portes et couloirs pour l'avatar du jeu. Repères (heuristiques de terrain) : porte 4 × 7 ; plafond 10 à 14 dans une pièce, 16+ dans un hall ; table ou comptoir 3 à 3,5 et assise 1,5 à 2 (à vérifier avec un avatar) ; chemin ≥ 6, axe principal ≥ 10 ; marche de 1 de haut sur 1,5 de profondeur.
- Une pièce fermée doit être plus grande qu'en vrai pour paraître juste : ~12 studs² par personne ; une salle de 40 × 30 × 12 a dû passer à 60 × 45 × 18.
- Matériau et taille : Sandstone, Slate, Marble ou Brick sur plus de ~20 studs donnent un carrelage répétitif daté. Grandes surfaces en `SmoothPlastic` (ou Grass) avec des variations de couleur ; matériaux détaillés sur les pièces de moins de ~5 studs. Le Neon en grande surface écrase la silhouette.
- White-out : du `SmoothPlastic` presque blanc (canaux > 225) sous un Bloom sature l'image. Fonce de 40 à 60 points ; un meuble clair garde au moins un canal ≤ 150. Couleurs en `Color3.fromRGB` (0 à 255) : `Color3.new` et les champs `R`, `G`, `B` sont des flottants de 0 à 1.
- Le Neon brille mais n'éclaire rien : ajoute une `PointLight`, `SpotLight` ou `SurfaceLight`. Un panneau lumineux sans source de lumière : un `Decal` émissif (`EmissiveMaskContent`, `EmissiveStrength`, `EmissiveTint`). Un style par jeu : blocs classiques (Parts, SmoothPlastic, couleurs vives), jouet lisse (Parts et MeshParts simples en couleur unie), low-poly stylisé (MeshParts, petite texture palette partagée), PBR (SurfaceAppearance, budgets serrés). Le mélange se lit comme une erreur.
- Bloc d'abord : silhouettes en blocs, sphères en accent ; des arbres en cubes font plus « Roblox » que des boules de feuillage. Exception : les avant-toits en wedges, indispensables à une pagode ou un château.
- Petit accessoire lisible = vocabulaire minimal : une fleur, c'est tige, pétales et cœur ; un verre n'est pas un simple cylindre. Matériau maison : `MaterialVariant` dans MaterialService (ou `generate_material` du MCP). Les maps d'une `SurfaceAppearance` ne changent pas au runtime : échange la pièce ou le `MaterialVariant`.

## 6. Voxels

Le style blocs de Roblox s'y prête bien. Choisis la technique selon le nombre de cubes (constat d'un projet testé le 2026-10-18 ; seuils de terrain) :

| Cubes | Technique |
|---|---|
| Quelques centaines | Une Part par voxel : simple, ça tient. |
| Jusqu'à quelques milliers | Fusion des voisins de même couleur en boîtes (code ci-dessous) : souvent 5 à 20 fois moins de parts (heuristique). |
| Au-delà | Un mesh des seules faces visibles (OBJ, FBX ou glTF), uploadé : un MeshPart, 20 000 triangles au plus. Ou un EditableMesh au runtime (section 7). |

- Palette courte (8 à 16 couleurs exactes en `Color3.fromRGB`) : plus de fusions et un style plus net. Tout ancré, en `SmoothPlastic`, `CanTouch` et `CanQuery` coupés sur le décor.
- Depuis MagicaVoxel : export OBJ et palette PNG, nettoyage dans Blender, export glTF ou FBX, puis Importer.

```luau
--!strict
-- Fusionne les voxels de même couleur en boîtes (X, puis Z, puis Y) : une Part par boîte.
-- grid[x][y][z] = Color3 ou nil ; dims en voxels ; renvoie le nombre de parts créées.
type Grid = { [number]: { [number]: { [number]: Color3? } } }

local function buildVoxels(grid: Grid, dims: Vector3, voxel: number, origin: CFrame, parent: Instance): number
	local used: { [number]: boolean } = {}
	local function key(x: number, y: number, z: number): number
		return x + (y - 1) * dims.X + (z - 1) * dims.X * dims.Y
	end
	local function colorAt(x: number, y: number, z: number): Color3?
		local column = grid[x]
		local row = column and column[y]
		return row and row[z]
	end
	local function free(x: number, y: number, z: number, color: Color3): boolean
		return colorAt(x, y, z) == color and not used[key(x, y, z)]
	end
	local count = 0
	for z = 1, dims.Z do
		for y = 1, dims.Y do
			for x = 1, dims.X do
				local color = colorAt(x, y, z)
				if color and not used[key(x, y, z)] then
					local w, d, h = 1, 1, 1
					local function rowFree(zz: number, yy: number): boolean
						for i = x, x + w - 1 do
							if not free(i, yy, zz, color) then return false end
						end
						return true
					end
					local function layerFree(yy: number): boolean
						for k = z, z + d - 1 do
							if not rowFree(k, yy) then return false end
						end
						return true
					end
					while x + w <= dims.X and free(x + w, y, z, color) do w += 1 end
					while z + d <= dims.Z and rowFree(z + d, y) do d += 1 end
					while y + h <= dims.Y and layerFree(y + h) do h += 1 end
					for i = x, x + w - 1 do
						for j = y, y + h - 1 do
							for k = z, z + d - 1 do
								used[key(i, j, k)] = true
							end
						end
					end
					local p = Instance.new("Part")
					p.Anchored = true
					p.Material = Enum.Material.SmoothPlastic
					p.Color = color
					p.Size = Vector3.new(w, h, d) * voxel
					p.CFrame = origin * CFrame.new(Vector3.new(x - 1 + w / 2, y - 1 + h / 2, z - 1 + d / 2) * voxel)
					p.Parent = parent
					count += 1
				end
			end
		end
	end
	return count
end
```

## 7. EditableMesh et EditableImage

- Ce ne sont pas des Instances : `AssetService:CreateEditableMesh()` et `AssetService:CreateEditableImage({ Size = Vector2.new(256, 256) })`, synchrones ; `CreateEditableImageAsync(Content.fromUri(uri))` charge une image existante et peut lever.
- La création renvoie `nil` quand le budget mémoire `Editable*` de l'appareil est épuisé : teste `nil`, et `:Destroy()` après usage.
- Affichage : `Content.fromObject(objet)` dans `ImageLabel.ImageContent` ou `MeshPart.TextureContent`. Un EditableMesh devient un MeshPart avec collision par `AssetService:CreateMeshPartAsync(Content.fromObject(mesh))`, à refaire à la fin d'une édition, pas à chaque frame.
- Limites : EditableImage 1024 × 1024 (512 × 512 sans `Size`) ; EditableMesh 60 000 sommets et 20 000 triangles ; une image pèse largeur × hauteur × 4 octets. Pixels par `buffer` (RGBA, ligne par ligne) et `WritePixelsBuffer` ; `ResampleMode = Pixelated` garde le pixel art net. `CreateEditableMeshAsync(content)` rend un mesh de taille fixe par défaut (`FixedSize = true` : sommets modifiables, topologie figée, moins de mémoire).
- Ils ne se répliquent pas : chaque côté construit le sien à partir de paramètres ou d'une graine (répliquer des pixels arbitraires te rend responsable de leur modération). IDs de sommets et de faces stables mais non contigus : itère `GetVertices()` et `GetFaces()`, jamais `1..count`.
- Expérience publiée : active « Allow Mesh / Image APIs » (Experience Settings → Security dans Studio, ou « Enable Mesh / Image APIs » dans le Creator Dashboard) ; seuls les assets possédés par le propriétaire de l'expérience ou partagés avec lui se chargent, sinon l'appel lève une erreur. La doc ne demande plus d'être un créateur vérifié de 13 ans ou plus (exigence levée le 2026-09-28 ; doc, 2026-10-18).
- Voxels : génère les seules faces visibles (`AddVertex`, `AddTriangle`, couleurs par face) puis `CreateMeshPartAsync` : un MeshPart pour des milliers de cubes. API récente : vérifie les signatures dans la référence avant d'écrire le code.

## 8. Importer des assets

- Universal Importer (File → Import, `Ctrl+M` selon un dépôt mais absent de la doc, ou le bouton d'import de l'Asset Manager) : `.fbx` et `.gltf`/`.glb` portent hiérarchie, rig, skinning, animation, textures PBR et couleurs de sommets ; `.obj` n'apporte que la géométrie statique. Réglages qui décident taille et placement : **Scale Unit** (studs par défaut : mets l'unité du fichier), World Forward et World Up, Set Pivot to Scene Origin (actif), Insert Using Scene Position. Défauts à surveiller : Upload to Roblox actif (coupe-le pour un essai local), **Anchored inactif**, Merge Meshes inactif.
- Blender → Roblox : modélise avec 1 unité Blender = 1 stud et applique toutes les transformations ; exporte en FBX avec Apply Scalings = FBX Unit Scale (le glTF n'a besoin d'aucun contournement). Côté Blender, avant = −Y et haut = +Z. Origine en bas au centre pour un prop, au point de prise pour un objet tenu, sur la charnière pour une porte, sur l'axe pour une roue. Un objet par modèle, nommé exactement comme dans le jeu ; les parties lumineuses en objets séparés `Glow_*`, pour du Neon et des lumières par pièce, coupables en qualité basse.
- Limites d'un mesh : 20 000 triangles (au-delà, l'Importer simplifie et peut ruiner la silhouette : découpe) ; géométrie étanche, sans surface d'épaisseur nulle ; un matériau par MeshPart (Merge Meshes seulement si une texture les sert tous) ; un ombrage cassé vient d'abord des normales.
- Textures : upload jusqu'à 4096 px, mais budget PBR de 1024 px par map (≈ 256 px pour un objet de 2 studs, 1024 pour 8 studs). Pour la perf, vise 512 px pour le décor et 256 pour un petit prop (performance.md). Maps de `SurfaceAppearance` : couleur, normale (OpenGL, espace tangent), rugosité, métal, masque émissif ; un jeu d'UV dans 0–1.
- Budgets (heuristiques de terrain) : ramassable ≤ 2 000 triangles, prop ≤ 3 000, prop héros ≤ 6 000. Un mesh d'IA image → 3D sort à 100 000 – 500 000 triangles : fusionne les coutures UV, décime sous 20 000, puis cuis les couleurs dans un atlas.
- Rig : os à échelle 1 et rotation 0, racine à l'origine, 4 influences par sommet au plus, une animation par FBX ; pour un corps R15 ou un accessoire, l'outil Avatar Setup de Studio d'abord. Mesh riggé chargé au runtime : garde la hiérarchie importée intacte (re-parenter un os ou retirer le joint racine du skin donne une T-pose).
- Après import : dossier de revue (`Workspace/AssetReview`) ; vérifie échelle face à un avatar, pivot, ancrage, collisions, matériaux, triangles ; Alt+Shift+R réimporte. Un modèle peut avoir trois copies : tenue (non ancrée, soudée à la poignée), exposition (ancrée), aperçu de ViewportFrame (ancré, sans effets, voir ui-fondamentaux.md).
- Upload sans Studio : API Open Cloud Assets, `assetType` Model, `.fbx`, `.gltf`, `.glb`, `.rbxm`, `.rbxmx` jusqu'à 20 Mo (open-cloud.md) ; mais l'Importer donne aperçu et avertissements, donc passe par lui pour tout nouvel asset.
- Texture blanche ou mesh invisible en jeu : permission d'asset. Lis l'erreur dans Output (un clic dessus propose d'accorder la permission), ou accorde l'asset à l'expérience (Creator Dashboard → Permissions) ; le partage entre collaborateurs ne suffit pas. Images, decals et meshes naissent en Open Use, ou en Restricted si Asset Privacy est activé (doc, 2026-10-18). Passer en Open Use ou accorder un asset à une expérience ne se défait pas : demande avant.
- La publication de place par Open Cloud ne transporte ni unions (`PartOperation`), ni `SurfaceAppearance`, ni `EditableMesh`/`EditableImage` : publie ces changements depuis Studio (open-cloud.md).
- Place construite hors Studio avec Lune : MeshParts parfois créés sans données de rendu (invisibles), `WeldConstraint` qui colle les pièces au centre (utilise `Weld` avec `C0`), `Attachment.Position` non sauvé (écris `CFrame`), `CFrame.lookAt` faux avant Lune 0.11 (outils.md).

## 9. Éclairage et ambiances

- `Lighting` : `ClockTime` (0 à 24), `Brightness` (au-delà de 3, ça délave), `Ambient` et `OutdoorAmbient` (jamais noirs), `ColorShift_Top`/`Bottom`, `GlobalShadows`, `ShadowSoftness` (0 net, 1 doux), `EnvironmentDiffuseScale`, `EnvironmentSpecularScale`.
- `Technology` est interne (`RobloxScriptSecurity`) et ne se règle que dans Studio (pieges.md) ; `LightingStyle` (`Realistic`, `Soft`) et `PrioritizeLightingQuality` se règlent dans la fenêtre Propriétés (section 11). `Atmosphere` (sous Lighting) : `Density` (au-delà de 0,5, c'est une soupe), `Offset`, `Color`, `Decay`, `Glare`, `Haze`. Quand elle est présente, `FogStart`/`FogEnd`/`FogColor` sont ignorés : choisis l'un ou l'autre. Elle n'a pas d'`Enabled` : retire-la pour la couper.
- Post-traitement (sous Lighting, ou sous la Camera pour un effet local) : `BloomEffect`, `ColorCorrectionEffect`, `DepthOfFieldEffect`, `SunRaysEffect`, `BlurEffect`. Deux ou trois au plus. Plusieurs `ColorCorrectionEffect` se multiplient : garde-en une et tweene-la. Coupe-les en qualité basse.
- Lumières : `PointLight` (torche, lampe), `SpotLight` (`Angle` = cône entier), `SurfaceLight` (écran, enseigne). Les ombres coûtent : `Shadows = false` sur la plupart, 4 à 6 lumières à ombres visibles au plus (heuristique, pas une limite moteur). Ambiances par zone côté client, en tweenant Lighting, l'Atmosphere et une seule ColorCorrection ; le serveur envoie seulement « ambiance X en Y secondes ». Chaud et clair = sûr, sombre = danger. Teste sur téléphone : un éclairage superbe sur PC peut y être délavé ou noir.

Points de départ (heuristiques à régler à l'œil) :

| Ambiance | ClockTime | Brightness | Ambient / OutdoorAmbient | Atmosphere (Density, Haze, Color) | ColorCorrection (Bright., Contrast, Sat., Tint) | Bloom |
|---|---|---|---|---|---|---|
| Simulator, jour vif | 14 | 2 | (120, 120, 130) / (140, 140, 150) | 0,25 ; 1 ; (200, 215, 235) | 0 ; 0,1 ; 0,15 ; blanc | 0,3, seuil 1 |
| Pastel, mignon | 13 | 1,1 à 1,4 | (150, 150, 160) / (160, 160, 175) | 0,2 ; 0,5 ; (230, 225, 240) | 0,02 ; 0,05 ; 0 ; (255, 245, 250) | 0,25, seuil 0,95 |
| Heure dorée | 17,5 | 1,5 | (140, 110, 90) / (160, 125, 100) | 0,3 ; 2 ; (255, 200, 160) | 0 ; 0,1 ; 0,1 ; (255, 235, 215) | 0,4, seuil 0,9 |
| Nuit calme | 0 | 0,5 | (40, 45, 70) / (30, 35, 60) | 0,3 ; 1 ; (70, 80, 120) | −0,05 ; 0,1 ; −0,2 ; (210, 220, 255) | aucun |
| Intérieur | 14 | 0,8, plus des PointLights | (100, 95, 90) / (80, 80, 85) | 0,1 ; 0 ; (200, 200, 200) | 0 ; 0,05 ; 0 ; (255, 245, 235) | faible |
| Horreur | 22 | 0,2 | (18, 12, 24) / (12, 10, 18) | 0,5 ; 6 ; (25, 20, 30) | −0,1 ; 0,3 ; −0,45 ; (205, 185, 205) | aucun |

```luau
--!strict
-- Change d'ambiance côté client : annule les tweens en cours, puis tweene les trois cibles.
local Lighting = game:GetService("Lighting")
local TweenService = game:GetService("TweenService")

export type Mood = { lighting: { [string]: any }, atmosphere: { [string]: any }, grade: { [string]: any } }

local function child(className: string): Instance
	-- par classe, pas par nom : une place neuve a souvent déjà une Atmosphere, et une 2e ColorCorrection se cumulerait
	local found = Lighting:FindFirstChildOfClass(className)
	if found then
		return found
	end
	local created = Instance.new(className)
	created.Parent = Lighting
	return created
end
local atmosphere, grade = child("Atmosphere"), child("ColorCorrectionEffect")
local running: { Tween } = {}

local function applyMood(mood: Mood, seconds: number)
	for _, tween in running do
		tween:Cancel()
	end
	table.clear(running)
	local info = TweenInfo.new(seconds, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut)
	local function play(target: Instance, goals: { [string]: any })
		local tween = TweenService:Create(target, info, goals)
		table.insert(running, tween)
		tween:Play()
	end
	play(Lighting, mood.lighting)
	play(atmosphere, mood.atmosphere)
	play(grade, mood.grade)
end
```

Exemple : `applyMood({ lighting = { ClockTime = 0, Brightness = 0.5 }, atmosphere = { Density = 0.3, Haze = 1 }, grade = { Saturation = -0.2 } }, 3)` passe à la nuit en 3 s ; complète chaque table avec la ligne voulue du tableau.

## 10. Terrain

- Voxels de 4 studs. Formes simples : `Terrain:FillBlock(cframe, size, material)`, `FillBall`, `FillCylinder`, `FillWedge`. Données : `ReadVoxels`/`WriteVoxels(region, 4, materials, occupancies)`, tableaux indexés `[x][y][z]`, occupation de 0 à 1, région alignée sur la grille (`ExpandToGrid(4)`) et de 4 194 304 voxels au plus (doc, 2026-10-18). Écris par blocs de ~128 studs de côté.
- **Occupation mesurée** : le terrain lisse dessine sa surface à bas du voxel + 2 + 4 × occupation, un demi-voxel plus haut qu'un remplissage « par le bas ». Pour une surface à la hauteur h, écris `clamp((h - basDuVoxel) / 4 - 0.5, 0, 1)`. Mesuré : sans correction, une hauteur de 10,06 était dessinée à 12,12 (projet réel, 2026-10-11).
- Pose les pièces dures (routes, trottoirs) sur cette surface corrigée et ne creuse que 0,4 à 0,8 stud dessous ; contrôle par raycast contre le terrain plutôt que par capture. Trous couleur ciel après plusieurs reconstructions : vérifie d'abord par raycast si la donnée est solide ; si oui, relis puis réécris les mêmes voxels (`ReadVoxels` puis `WriteVoxels`) pour forcer le re-maillage, ou sauvegarde et rouvre la place. Ne régénère pas les données.
- Le terrain ne passe pas dans un `.rbxm` de modèle, ne se synchronise pas en direct avec Rojo et ne s'écrit pas avec Lune : livre-le dans la place, ou en données plus un module builder qui appelle `WriteVoxels` dans Studio (et au démarrage du serveur s'il manque).
- `Terrain.Decoration` (herbe animée) a passé luau-lsp dans un vrai projet, mais plante à l'exécution dans un Script ; elle manque dans `globalTypes.None.d.luau` (pieges.md). Règle-la dans la fenêtre Propriétés. `WriteVoxels` est interdit en phase parallèle.
- Matériaux : couleur par matériau avec `Terrain:SetMaterialColor(material, color)` ; matériaux personnalisés par `MaterialVariant` (jusqu'à 64 en accès anticipé depuis le 2026-06-25 ; Roblox déconseille de publier une place avec ces réglages actifs) ; eau réglée par `WaterColor`, `WaterTransparency`, `WaterWaveSize`, `WaterWaveSpeed`, `WaterReflectance`. Procédural : `math.noise(x, y, z)` (Perlin, ≈ −1 à 1) renvoie 0 aux coordonnées entières : divise par une échelle non entière et ajoute une graine. Grand terrain réaliste : générateur externe (Gaea), mesh décimé à 50 000 – 200 000 faces, import, puis conversion en voxels ; une heightmap est plus rapide mais sans grottes ni surplombs.

```luau
--!strict
-- Terrain depuis une carte de hauteurs (une hauteur par colonne de 4 studs, relative à origin.Y).
-- origin doit être un multiple de 4 sur chaque axe.
local Workspace = game:GetService("Workspace")
local VOXEL = 4

local function writeHeightmap(heights: { { number } }, origin: Vector3, maxHeight: number, material: Enum.Material)
	local sizeX, sizeZ = #heights, #heights[1]
	local sizeY = math.ceil(maxHeight / VOXEL) + 1
	assert(sizeX * sizeY * sizeZ <= 4194304, "région trop grande : découpe la carte en blocs")
	local materials: { { { Enum.Material } } } = {}
	local occupancy: { { { number } } } = {}
	for x = 1, sizeX do
		materials[x], occupancy[x] = {}, {}
		for y = 1, sizeY do
			materials[x][y], occupancy[x][y] = {}, {}
			local bottom = (y - 1) * VOXEL
			for z = 1, sizeZ do
				local occ = math.clamp((heights[x][z] - bottom) / VOXEL - 0.5, 0, 1) -- compense le demi-voxel
				occupancy[x][y][z] = occ
				materials[x][y][z] = if occ > 0 then material else Enum.Material.Air
			end
		end
	end
	local region = Region3.new(origin, origin + Vector3.new(sizeX, sizeY, sizeZ) * VOXEL):ExpandToGrid(VOXEL)
	Workspace.Terrain:WriteVoxels(region, VOXEL, materials, occupancy)
end
```

## 11. Collisions et réglages réservés à l'édition

- `CollisionFidelity` : `Box` pour le petit décor, `Hull` pour les formes plutôt convexes, `Default` quand la concavité compte, `PreciseConvexDecomposition` seulement là où l'on se tient dans un creux, `Tunable` avec un curseur `CollisionPrecision` (2026-09-17). `Scalable` a été retiré en v738.
- `RenderFidelity` : `Automatic` baisse le détail à 250 puis 500 studs ; `Precise` sur beaucoup de meshes coûte du temps de frame.
- Grand mesh complexe : collision par quelques Parts `Box` invisibles, et `CanCollide`, `CanTouch`, `CanQuery` à `false` sur le mesh visible (sa géométrie de collision coûte de la mémoire sinon). Les os animés changent l'apparence, jamais la collision. Visualization Options → Collision fidelity montre ce que voit la physique.
- **Réservés à l'édition** (fenêtre Propriétés, barre de commande, plugin) : `CollisionFidelity`, `RenderFidelity`, `FluidFidelity`, `SmoothingAngle`, `Model.LevelOfDetail`, `Lighting.LightingStyle`, `PrioritizeLightingQuality`, `Terrain.Decoration`, `StreamingEnabled`. Un Script de jeu échoue sur eux même quand la métadonnée affiche `Security: None` (capacité `PluginOrOpenCloud`, mesuré en 0.742). Des notes de version annoncent des assouplissements en v740–v742, et la doc liste désormais `LightingStyle` et `PrioritizeLightingQuality` avec la capacité `Environment` (doc, 2026-10-18) : teste une écriture avant d'en dépendre.
- `execute_luau` du MCP en mode Edit a le privilège plugin : il peut régler ces propriétés (pieges.md). Relis la valeur après l'écriture. Au runtime, change la fidélité par `AssetService:CreateMeshPartAsync(part.MeshContent, { CollisionFidelity = …, RenderFidelity = … })` puis `part:ApplyMesh(nouveau)`.
- Une vérification de types verte ne prouve pas qu'une écriture moteur marche : exécute une fois dans Studio tout code qui touche ces propriétés, et lis la console.

## 12. Streaming et niveaux de détail

- `Workspace.StreamingEnabled` (réglage de Studio, actif sur les nouvelles places) : le client ne reçoit que les environs. L'activer sur une place publiée est un changement à risque : accord de l'utilisateur d'abord.
- `Model.ModelStreamingMode` : `Default`, `Atomic` (tous les descendants arrivent ensemble), `Nonatomic`, `Persistent` (toujours présent : rares petits objets, jamais en rustine générale), `PersistentPerPlayer`. `Instance.StreamingMode` et `ModelStreamingMode.Opportunistic` n'existent pas (`Opportunistic` est une valeur de `Enum.StreamOutBehavior`).
- Le code client tolère les absences : `WaitForChild` avec délai, ou tags et `CollectionService:GetInstanceAddedSignal`. Avant de téléporter un joueur loin : `Player:RequestStreamAroundAsync(position)` côté serveur (un indice, pas une garantie). Garde le gros contenu 3D dans Workspace, pas dans ReplicatedStorage : seul Workspace est streamé. `Model.LevelOfDetail = StreamingMesh` affiche une version grossière d'un modèle au-delà du rayon de streaming, `SLIM` une version plus fidèle (réglage d'édition) ; `RenderFidelity = Automatic` sur les MeshParts.
- Portes ou tunnels entre zones : le streaming charge la suivante avant l'arrivée. Une très grande carte peut se découper en places reliées par téléportation (chargements courts, attente à chaque passage).
- Repères de pièces (heuristiques de terrain, non officielles) : < 2 000 par modèle sans souci ; 2 000 à 5 000, simplifie les collisions ; au-delà, LOD et streaming ; < 10 000 parts visibles par scène, moins sur mobile. Mesure avec Scene Analysis et le MicroProfiler (performance.md).

## 13. Organisation de l'espace de travail

- Jamais de parts à plat dans Workspace. Arborescence type : `Workspace/<Carte>/{Origin, Zones/Zone_Spawn/{Floor, Walls, Props}, Landmarks, Spawns}`, plus `Lighting/ZonePresets` pour les ambiances. `Origin` est une petite part invisible et ancrée dont dérivent toutes les zones. Noms en PascalCase ; modèles au singulier, dossiers au pluriel ; attachments suffixés `Attachment` ; pièces numérotées ou situées (`Wall1`, `LeftDoor`) ; jamais `Part` ou `Model` par défaut.
- Les systèmes trouvent les objets par tags (`CollectionService`) et attributs (`ResourceNode`, `RespawnTime`), pas par chemins codés en dur. Points d'accroche du gameplay : attachments ou parts invisibles nommés (`DropPoint`, `GripPoint`, `HelmSocket`).
- Sur les modèles générés, des attributs de suivi : `AssetRole`, `BudgetTier`, `Approved`, `GeneratedBy`. Gabarits dans ServerStorage (serveur seul) ou ReplicatedStorage (client aussi), clonés puis placés par `PivotTo`. Un kit modulaire (murs, coins, toits, sols) vaut mieux qu'un modèle géant : réutilisable, streamable, optimisable. Les Packages servent à partager un modèle entre places.
- Rien de secret dans un emplacement répliqué : `CanQuery` ou un collision group ne cachent rien.
- Construis un exemplaire modèle (une maison, un arbre) puis clone-le : les objets faits au milieu d'une longue séquence reçoivent moins de soin. À la fin, retire parts de test et objets de debug, puis relis la hiérarchie en mode Edit.

## 14. Pipeline IA : générer, valider, relire

1. **Réutiliser** : le projet d'abord, puis la Toolbox (`search_asset`). Note id, créateur, prix et droits ; consentement explicite pour un asset payant ou d'un autre propriétaire.
2. **Insérer en quarantaine** (`insert_asset` dans un dossier à part) et lire toute la hiérarchie : un `require` par id, `loadstring` ou `getfenv` signalent une porte dérobée (securite.md). Préfère les assets sans scripts.
3. **Générer si rien ne convient** : `generate_procedural_model` (assemblage de primitives, avec des noms de parties parlants), `generate_mesh` (prop texturé, taille et triangles bornés), `generate_material`. Chaque génération est un job facturé sur le quota du compte : demande avant, affine le prompt entre deux essais, lance en `async: true` puis `wait_job_finished`.
4. **Dupliquer avant de retoucher** : `generate_texture` remplace le mesh source sur place ; `segment_mesh` crée un Model d'au plus 5 parties (la commande équivalente de l'Assistant remplacerait l'original). Le généré est un candidat, pas un livrable.
5. **Construire par code** (sections 2 à 6) quand le style blocs suffit, ou quand la génération échoue, déçoit ou n'est pas disponible.
6. **Valider** par script jusqu'à zéro erreur (ci-dessous), puis capture avec une caméra choisie (`screen_capture` ne montre ni SurfaceGui ni BillboardGui), puis traversée depuis le spawn à vitesse réelle.
7. **Relire** : silhouette à la caméra de jeu, échelle face à un avatar, palette, collisions, comptes (parts, triangles, textures), provenance notée.

- Une ligne de style commune à tous les prompts d'un jeu (par exemple « stylized cartoon Roblox asset, chunky simple shapes, bright saturated colours, low poly, single object, centred, no text ») garde les modèles cohérents. Un set à paliers garde la même empreinte et ajoute un élément de silhouette par palier, pas une simple recoloration.
- En jeu, `GenerationService:GenerateModelAsync(inputs, schema, options)` génère depuis un prompt un Model conforme à un schéma (`Car5`, `Body1` ou personnalisé) (doc, 2026-10-18). La méthode lève une erreur en cas d'échec (prompt refusé, délai) : appelle-la dans un `pcall` et traite le résultat comme du contenu de joueur. Une carte est un parcours : depuis chaque spawn, le prochain repère est visible, la caméra n'est pas dans un mur, chaque zone est atteignable. Une vue de dessus ne prouve pas l'orientation du joueur.

```luau
--!strict
-- Contrôle d'un décor statique après construction ; relance jusqu'à zéro erreur.
local DEFAULT_GREY = Color3.fromRGB(163, 162, 165)

local function validate(root: Instance, groundY: number)
	local parts, errors, warnings = 0, 0, 0
	for _, d in root:GetDescendants() do
		if d:IsA("BasePart") then
			parts += 1
			if not d.Anchored then errors += 1; warn("non ancrée :", d:GetFullName()) end
			-- demi-hauteur monde de la boîte orientée : la rotation compte
			local c, s = d.CFrame, d.Size
			local halfY = (math.abs(c.RightVector.Y) * s.X + math.abs(c.UpVector.Y) * s.Y + math.abs(c.LookVector.Y) * s.Z) / 2
			if d.Position.Y - halfY < groundY - 0.5 then warnings += 1; warn("sous le sol :", d:GetFullName()) end
			if d.Material == Enum.Material.Plastic and d.Color == DEFAULT_GREY then
				warnings += 1; warn("gris par défaut :", d:GetFullName())
			end
		end
	end
	print(`[validate] {parts} parts, {errors} erreurs, {warnings} avertissements`)
end
```

## 15. Pièges

- Parts qui tombent : `Anchored` resté à `false`. Cylindre minuscule ou couché : longueur mise sur Y au lieu de X.
- Poutre de travers : grande dimension sur Y avec `CFrame.lookAt`, qui aligne Z.
- Coordonnées qui dérivent d'un appel à l'autre : positions recalculées de mémoire au lieu d'être relues. Union qui disparaît ou change de couleur : échec non testé, `UsePartColor` oublié, CFrame recopié sans rotation, cutters qui traînent dans Workspace.
- Z-fighting : faces coplanaires ; décale de ~0,05 stud. Scène délavée : `Brightness` au-delà de 3, `SmoothPlastic` presque blanc sous un Bloom, Neon en grande surface. Brouillard ignoré : `FogEnd` réglé alors qu'une `Atmosphere` existe. Couleurs écrasées : plusieurs `ColorCorrectionEffect` qui se multiplient.
- Terrain un demi-voxel trop haut : occupation écrite « par le bas ». Trous couleur ciel : relis puis réécris les mêmes voxels.
- Mesh blanc ou invisible : permission d'asset, ou MeshPart créé par Lune sans données de rendu. Script qui plante sur une propriété d'édition (`CollisionFidelity`, `LightingStyle`, `Terrain.Decoration`).
- Noms inventés vus dans des guides : `Instance.StreamingMode`, `ModelStreamingMode.Opportunistic`, `workspace:RequestStreamAroundAsync`, des MeshId « de mémoire ». Nom périmé : `GeometryService:CreateSolidPrimitive`, supprimé en 0.737 (remplacé par `CreateBasicMeshPart`). Et `math.noise` à coordonnées entières vaut toujours 0.

## 16. Checklist

- [ ] Racine et `Origin` présentes, un dossier par zone, noms explicites, rien à plat dans Workspace.
- [ ] Cotes nommées en tête de script, sur une grille ; sous-pièces placées depuis une ancre.
- [ ] Chaque appel MCP ré-acquiert la racine, est rejouable et relit ce qu'il a construit.
- [ ] Statique ancré ; décor sans `CanTouch` ni `CanQuery` ; collisions simples (`Box`, proxies) sur meshes et unions décoratives.
- [ ] Couleurs et matériaux voulus, un seul style, ni gris par défaut ni white-out ; orientation vérifiée (LookVector imprimés ou capture) pour cylindres, wedges, sièges, meshes importés.
- [ ] CSG : 3 à 4 opérations par lot, `pcall`, résultat vérifié, cutters détruits ; voxels : technique adaptée au nombre de cubes ; meshes ≤ 20 000 triangles, textures à 512 px pour le décor (1024 au plus).
- [ ] Éclairage : une seule ColorCorrection, Atmosphere ou Fog mais pas les deux, ambiance testée sur téléphone et PC.
- [ ] Réglages d'édition posés dans Studio, pas par un Script de jeu ; streaming : `Atomic` ou `Persistent` justifiés, code client tolérant aux absences.
- [ ] Assets insérés relus (scripts), provenance et permissions notées.
- [ ] Validation à zéro erreur, capture avec caméra choisie, traversée depuis le spawn ; ce qui n'a pas été vu est dit.
