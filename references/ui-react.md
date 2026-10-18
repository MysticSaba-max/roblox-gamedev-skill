# UI déclarative : React (roblox-ts et react-lua), Fusion, Vide

À lire quand : tu écris ou corriges une interface Roblox en code déclaratif, surtout en roblox-ts + `@rbxts/react` (la pile par défaut de l'utilisateur), ou en react-lua, Fusion ou Vide ; tu animes une UI ; tu affiches un modèle 3D dans un menu. Les bases du moteur (UDim2, layouts, safe area, mobile, texte, images) sont dans ui-fondamentaux.md.

## Sommaire

1. Choisir le framework
2. Monter un projet roblox-ts + React qui marche
3. Monter l'arbre React
4. Écrire des composants (TSX et Luau)
5. État : quoi mettre où
6. Effets et signaux Roblox
7. Animer sans re-render
8. Boîte à outils d'animation (code testé)
9. Performance
10. Modèle 3D dans l'UI
11. Stories et aperçus
12. Fusion 0.3 et Vide en bref
13. Pièges
14. Checklist

---

## 1. Choisir le framework

| Situation | Choix |
|---|---|
| Le projet a déjà un framework d'UI | Le garder. Ne jamais en mélanger deux : leurs cycles de vie sont incompatibles, et l'un enveloppé dans l'autre fuit. |
| Nouveau projet roblox-ts | `@rbxts/react` + `@rbxts/react-roblox` : react-lua, le portage de React 17 écrit par Roblox, empaqueté pour roblox-ts par la communauté (scope npm `@rbxts-js`). |
| Nouveau projet Luau avec beaucoup d'UI | react-lua (`jsdotlua/react` et `jsdotlua/react-roblox` via Wally), ou Fusion 0.3 si l'équipe préfère le modèle réactif. |
| UI minuscule ou statique | Instances natives, ou templates faits dans Studio puis clonés. N'ajoute pas un framework pour économiser trois fonctions. |
| Code Roact existant | Roact est abandonné au profit de react-lua. L'adapter sans l'étendre ; migrer seulement si l'utilisateur le demande. |

- **roblox-ts ou Luau pour un agent.** roblox-ts apporte les types et npm. Il ajoute un watcher à garder lancé, des fichiers générés à ne jamais éditer et des numéros de ligne Studio qui ne correspondent pas au source. Le Luau pur est plus direct. Suis le choix de l'utilisateur : ici, roblox-ts + React a été monté et testé dans Studio.
- Dans un projet roblox-ts, les `.luau` de `out/` sont de la sortie de build. Lis et modifie seulement les `.ts` et `.tsx` de `src/`.
- Fusion et Vide sont de bons choix en Luau pur. En roblox-ts, `@rbxts/fusion` est resté en 0.2 (2023) ; `@rbxts/vide` existe et vit encore (npm, 2026-10-18), mais React reste la pile testée ici. Ne les propose pas dans un projet TS.

---

## 2. Monter un projet roblox-ts + React qui marche

Recette testée le 2026-10-18 sur Windows avec Rojo 7.7 et roblox-ts 3.0. Le détail de l'outillage est dans outils.md.

```bash
npx -y create-roblox-ts@latest game --dir . --git=false --eslint --prettier --vscode --packageManager npm --skipBuild
```

```bash
npm i @rbxts/react @rbxts/react-roblox @rbxts/services
```

- Le modèle `game` règle déjà `"jsx": "react"`, `"jsxFactory": "React.createElement"` et `"jsxFragmentFactory": "React.Fragment"` dans `tsconfig.json`. Vérifie-les si le projet vient d'ailleurs.
- **Obligatoire : mapper le scope `@rbxts-js`.** `@rbxts/react` charge ses vrais modules (react, react-reconciler, scheduler, shared, luau-polyfill…) depuis ce scope npm. Le `default.project.json` du modèle ne mappe que `@rbxts`. Sans correctif, aucune UI n'apparaît et la console affiche `Infinite yield possible on '…node_modules:WaitForChild("@rbxts-js")'`.

```json
"rbxts_include": {
  "$path": "include",
  "node_modules": {
    "$className": "Folder",
    "@rbxts": { "$path": "node_modules/@rbxts" },
    "@rbxts-js": { "$path": "node_modules/@rbxts-js" }
  }
}
```

Règle générale : un enfant par scope npm utilisé à l'exécution sous `rbxts_include/node_modules`. Rojo recharge ce fichier à chaud.

- Boucle de dev : `npx rbxtsc -w` dans un terminal, `rojo serve` dans un autre, Connect dans le plugin Rojo de Studio, puis Play.
- Organisation : `src/client/ui/` avec un composant par fichier `.tsx`, un `App.tsx` racine, et un `main.client.ts` qui appelle une fonction de montage. Les types et constantes partagés avec le serveur vont dans `src/shared/`.

---

## 3. Monter l'arbre React

Ne passe jamais `PlayerGui` à `createRoot`. Au premier rendu, une racine détruit tous les enfants de son conteneur (doc et source de react-lua, lues le 2026-10-18) : elle effacerait les ScreenGui venus de StarterGui et le `TouchGui` des contrôles tactiles. Monte donc la racine dans un `Folder` hors arbre, et rends l'UI dans `PlayerGui` par un portail :

```tsx
// src/client/ui/mount.tsx
import React from "@rbxts/react";
import { createPortal, createRoot } from "@rbxts/react-roblox";
import { Players } from "@rbxts/services";
import { App } from "./App";

export function mountUi() {
	const playerGui = Players.LocalPlayer.WaitForChild("PlayerGui") as PlayerGui;
	const root = createRoot(new Instance("Folder"));
	root.render(createPortal(<App />, playerGui));
	return root;
}
```

Le test du 2026-10-18 faisait `createRoot(playerGui).render(<App />)`. L'UI s'affichait, mais ce montage détruit tout autre enfant de PlayerGui présent à ce moment-là.

```tsx
// src/client/ui/App.tsx
import React from "@rbxts/react";

export function App() {
	return (
		<screengui
			ResetOnSpawn={false}
			ZIndexBehavior={Enum.ZIndexBehavior.Sibling}
			ScreenInsets={Enum.ScreenInsets.CoreUISafeInsets}
		>
			{/* écrans */}
		</screengui>
	);
}
```

Équivalent Luau (react-lua) :

```luau
--!strict
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local React = require(ReplicatedStorage.Packages.React)
local ReactRoblox = require(ReplicatedStorage.Packages.ReactRoblox)
local App = require(script.Parent.App)

local playerGui = Players.LocalPlayer:WaitForChild("PlayerGui")
local root = ReactRoblox.createRoot(Instance.new("Folder"))
-- La racine possède le Folder ; le portail rend l'UI dans PlayerGui sans le vider.
root:render(ReactRoblox.createPortal(React.createElement(App), playerGui))
```

- Même forme dans les deux langages : racine dans un `Folder`, portail vers `PlayerGui`. Un portail ne vide pas sa cible.
- `ResetOnSpawn={false}` : sinon Roblox supprime le ScreenGui quand le personnage réapparaît (défaut `true`), et React perd ses instances.
- Utilise `ScreenInsets` plutôt que l'ancien `IgnoreGuiInset`. Mets `None` sur un ScreenGui de fond plein écran, et `CoreUISafeInsets` sur le contenu cliquable : il évite l'encoche et les boutons de Roblox.
- Une seule racine pour toute l'UI. Pour une modale ou un toast qui doit passer au-dessus de tout, utilise `createPortal(élément, autreScreenGui)` avec un `DisplayOrder` plus haut. Le contexte et l'état restent partagés.

---

## 4. Écrire des composants (TSX et Luau)

### Correspondances

| Idée React web | `@rbxts/react` (TSX) | react-lua (Luau) |
|---|---|---|
| `<div>` | `<frame>` | `React.createElement("Frame", props, children)` |
| `onClick` | `Event={{ Activated: () => … }}` | `[React.Event.Activated] = function(rbx) … end` |
| Réagir à une propriété | `Change={{ AbsoluteSize: (rbx) => … }}` | `[React.Change.AbsoluteSize] = function(rbx) … end` |
| `ref` | `ref={frameRef}` avec `useRef<Frame>()` | `ref = frameRef` avec `React.useRef(nil)` |
| Texte entre balises | Interdit : passe `Text="…"` | `Text = "…"` |
| CSS | Enfants modificateurs : `<uicorner>`, `<uistroke>`, `<uigradient>`, `<uipadding>`, `<uiscale>` | Les mêmes, en enfants |
| `key` d'une liste | `key={item.id}` | La clé de la table d'enfants |

- Les éléments intrinsèques TSX sont les noms de classe Roblox en minuscules : `frame`, `textlabel`, `textbutton`, `imagelabel`, `imagebutton`, `scrollingframe`, `viewportframe`, `canvasgroup`, `uilistlayout`, `uigridlayout`, `uiaspectratioconstraint`…
- Les props sont les propriétés Roblox exactes : `Size`, `Position`, `AnchorPoint`, `BackgroundColor3`, `FontFace`, `TextColor3`… Les types de `@rbxts/types` les vérifient à la compilation.
- Les props acceptent un **binding** à la place d'une valeur. C'est la base des animations sans re-render (section 7).
- Un objet peut porter **plusieurs UIStroke** : empile-les avec la propriété `ZIndex` de chaque stroke. Sur un texte, un stroke en `Contextual` fait le contour des lettres et un autre en `Border` fait la bordure du cadre. Certains vieux guides disent « un seul stroke » : c'est périmé.
- Luau : les enfants sont une **table à clés chaînes**. Une valeur `nil` ou `false` retire l'enfant, c'est la façon idiomatique de rendre conditionnellement. Deux clés identiques s'écrasent en silence. Nomme les enfants statiques par rôle (`Corner`, `Stroke`, `Layout`) et les dynamiques par un identifiant de donnée (`"Item_" .. id`).
- `defaultProps` ne marche que sur les composants classe. Dans un composant fonction, écris `props.size ?? 16` en TSX. En Luau, écris `if props.size ~= nil then props.size else 16`, car `props.x or défaut` casse quand la valeur est `false`.
- Luau n'a pas de déstructuration : lis `props.isOpen`, pas `function Modal({ isOpen })`.
- Absents de react-lua : `useTransition`, `useDeferredValue` (fonctions de React 18) et l'appli React DevTools. `Suspense` et `lazy` existent (React 17), mais le plus simple reste de charger les données dans un effet. Profile avec `debug.profilebegin`/`profileend` et le MicroProfiler.
- Règle des hooks : jamais dans un `if`, une boucle ou après un `return` anticipé. Un hook conditionnel décale tous les suivants.
- Manette : la sélection du moteur suffit souvent (ui-fondamentaux.md). Pour une navigation avancée en React, Roblox publie `Roblox/focus-navigation` (GitHub seulement, voir paquets.md).

### À écrire toi-même sur chaque élément

React crée les instances avec les valeurs par défaut de Roblox, pas celles de l'éditeur. Pour un rendu propre :

- `BorderSizePixel={0}` sur chaque GuiObject qui a un fond : sinon une fine bordure sombre apparaît.
- `AutoButtonColor={false}` sur un bouton stylé (dégradé, couleur choisie) : sinon Roblox l'assombrit tout seul au survol et à l'appui.
- `ResetOnSpawn={false}` et `ZIndexBehavior={Enum.ZIndexBehavior.Sibling}` sur chaque `screengui`.
- Couleurs depuis une maquette : `Color3.fromHex("#FFB020")`. Police avec graisse : `Font.fromName("BuilderSans", Enum.FontWeight.Bold)` ou `new Font("rbxasset://fonts/families/Montserrat.json", Enum.FontWeight.Heavy)` ; une famille à une seule graisse, comme FredokaOne, ignore la graisse demandée. Dégradé à plusieurs arrêts : `new ColorSequence([new ColorSequenceKeypoint(0, a), new ColorSequenceKeypoint(1, b)])`.

### Exemple : bouton à états (TSX)

Pilote l'apparence par **un seul état visuel**, pas par une combinaison de booléens.

```tsx
import React, { useState } from "@rbxts/react";
import { UserInputService } from "@rbxts/services";

type ButtonVisual = "idle" | "hover" | "pressed" | "disabled";

// Survol réservé à la souris : au doigt, MouseEnter part sur un tap et resterait collé (section 7)
const canHover = () => UserInputService.PreferredInput === Enum.PreferredInput.KeyboardAndMouse;

const COLORS: Record<ButtonVisual, Color3> = {
	idle: Color3.fromRGB(90, 200, 40),
	hover: Color3.fromRGB(120, 230, 60),
	pressed: Color3.fromRGB(60, 150, 30),
	disabled: Color3.fromRGB(120, 120, 120),
};

export function BuyButton(props: { label: string; disabled?: boolean; onBuy: () => void }) {
	const [visual, setVisual] = useState<ButtonVisual>("idle");
	const current: ButtonVisual = props.disabled ? "disabled" : visual;
	return (
		<textbutton
			Size={UDim2.fromOffset(180, 56)}
			BackgroundColor3={COLORS[current]}
			AutoButtonColor={false}
			Text={props.label}
			FontFace={Font.fromEnum(Enum.Font.FredokaOne)}
			TextSize={26}
			TextColor3={Color3.fromRGB(255, 255, 255)}
			Event={{
				MouseEnter: () => setVisual(canHover() ? "hover" : "idle"),
				MouseLeave: () => setVisual("idle"),
				MouseButton1Down: () => setVisual("pressed"),
				MouseButton1Up: () => setVisual(canHover() ? "hover" : "idle"),
				Activated: () => {
					if (!props.disabled) props.onBuy();
				},
			}}
		>
			<uicorner CornerRadius={new UDim(0, 12)} />
			<uistroke Thickness={3} Color={Color3.fromRGB(40, 90, 20)} ApplyStrokeMode={Enum.ApplyStrokeMode.Border} />
			<uistroke Thickness={2} Color={Color3.fromRGB(30, 60, 10)} ApplyStrokeMode={Enum.ApplyStrokeMode.Contextual} />
		</textbutton>
	);
}
```

Le même en Luau :

```luau
--!strict
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local UserInputService = game:GetService("UserInputService")
local React = require(ReplicatedStorage.Packages.React)
local e = React.createElement

type Props = { label: string, disabled: boolean?, onBuy: () -> () }

local function hoverOrIdle(): string
	return if UserInputService.PreferredInput == Enum.PreferredInput.KeyboardAndMouse then "hover" else "idle"
end

local COLORS: { [string]: Color3 } = {
	idle = Color3.fromRGB(90, 200, 40),
	hover = Color3.fromRGB(120, 230, 60),
	pressed = Color3.fromRGB(60, 150, 30),
	disabled = Color3.fromRGB(120, 120, 120),
}

local function BuyButton(props: Props)
	local visual, setVisual = React.useState("idle")
	local current = if props.disabled then "disabled" else visual
	return e("TextButton", {
		Size = UDim2.fromOffset(180, 56),
		BackgroundColor3 = COLORS[current],
		AutoButtonColor = false,
		Text = props.label,
		FontFace = Font.fromEnum(Enum.Font.FredokaOne),
		TextSize = 26,
		TextColor3 = Color3.fromRGB(255, 255, 255),
		[React.Event.MouseEnter] = function() setVisual(hoverOrIdle()) end,
		[React.Event.MouseLeave] = function() setVisual("idle") end,
		[React.Event.MouseButton1Down] = function() setVisual("pressed") end,
		[React.Event.MouseButton1Up] = function() setVisual(hoverOrIdle()) end,
		[React.Event.Activated] = function()
			if not props.disabled then props.onBuy() end
		end,
	}, {
		Corner = e("UICorner", { CornerRadius = UDim.new(0, 12) }),
		Border = e("UIStroke", { Thickness = 3, Color = Color3.fromRGB(40, 90, 20), ApplyStrokeMode = Enum.ApplyStrokeMode.Border }),
		TextOutline = e("UIStroke", { Thickness = 2, Color = Color3.fromRGB(30, 60, 10), ApplyStrokeMode = Enum.ApplyStrokeMode.Contextual }),
	})
end

return BuyButton
```

### Idiomes roblox-ts à connaître

- `array.size()` au lieu de `.length`, `string.format`, `math.round`, `os.clock()`, `task.spawn`/`task.delay`/`task.cancel`.
- Les fonctions Luau à retours multiples renvoient un `LuaTuple` : `const [value, setValue] = useBinding(0)`.
- Certains noms sont réservés par le compilateur. Une variable `next` échoue avec « Cannot use identifier reserved for compiler internal usage » : renomme-la.
- Un paramètre par défaut typé par un membre d'Enum se fige sur ce membre. Annote : `material: Enum.Material = Enum.Material.SmoothPlastic`.
- Méthodes utiles des types Roblox : `udim2.add(autre)`, `color.Lerp(autre, t)`, `UDim2.fromScale`, `UDim2.fromOffset`.
- Types React : `React.Element`, `React.ReactNode` pour `children`, `React.Binding<T>`.

---

## 5. État : quoi mettre où

| Donnée | Où la mettre | Pourquoi |
|---|---|---|
| Change rarement ou change la structure : onglet, menu ouvert, contenu d'une liste | `useState` ou `useReducer` | La structure des enfants ne peut changer que par un rendu. |
| Visuel qui change souvent : barre de vie, chrono, animation | Binding | Le binding écrit la propriété sans réconciliation. |
| Peu fréquent et partagé : thème, langue, profil affiché | Context, avec une valeur mémoïsée | Évite le prop drilling. |
| Global à plusieurs écrans : inventaire affiché, monnaies | Store : `@rbxts/charm` avec `@rbxts/react-charm` en TS, Charm via Wally ou un petit store maison en Luau | Un seul propriétaire, abonnements ciblés. |
| Vérité du jeu : monnaie réelle, achats, inventaire réel | **Serveur** | L'UI affiche et demande. Elle ne décide jamais. Voir securite.md. |

- **Jamais de données haute fréquence dans un Context.** Chaque consommateur re-rend à chaque changement. Utilise un binding ou un store à abonnement ciblé.
- **Deux durées de vie.** L'état de session (stats, réglages, monnaie affichée) vit dans un module créé une fois et survit aux écrans. L'état d'écran (onglet de la boutique) naît à l'ouverture et meurt à la fermeture. Mettre l'état de jeu dans un écran le perd à la première fermeture. Mettre des widgets dans la session les fait fuir.
- **État dérivé, pas stocké.** Une valeur calculable depuis un autre état se calcule, avec `useMemo` si c'est cher. Elle ne s'écrit pas par trois chemins qui finissent par diverger.
- **Mises à jour immuables.** Clone seulement le chemin modifié : `{ ...state, coins: n }` en TS, `table.clone` niveau par niveau en Luau. Pas de copie profonde : les sous-arbres inchangés gardent leur identité, et `memo` en profite.
- **Achats et récompenses.** Le clic envoie une intention au serveur (`RequestPurchase(itemId)`). L'UI peut afficher un état « en cours », puis se réaligne sur la réponse du serveur. Elle n'ajoute jamais l'objet elle-même. Voir monetisation.md.
- Valeur venue du serveur (monnaie, stock) : un RemoteEvent ou un attribut répliqué alimente le store, et le composant s'y abonne. Le client ne fait que l'afficher.

---

## 6. Effets et signaux Roblox

- Connecte les `RBXScriptSignal` **seulement dans un `useEffect`**, et renvoie le nettoyage : `Disconnect`, `task.cancel`, `tween.Cancel()`, `instance.Destroy()` si l'effet l'a créée.
- Garde le dernier callback dans une ref, et connecte le signal une seule fois. Sinon chaque rendu déconnecte et reconnecte.
- Aucun yield dans un rendu, un reducer ou un calcul mémoïsé. Une fonction qui yield se nomme `…Async` et s'appelle dans un effet via `task.spawn`, dont le nettoyage fait `task.cancel(thread)`. Un yield pendant un rendu corrompt l'état sans erreur visible.
- Après un `await` ou un `task.wait`, vérifie que le composant est toujours monté avant de mettre l'état à jour.

```tsx
import { useEffect, useRef } from "@rbxts/react";

// Abonnement générique à un signal Roblox, avec le callback le plus récent.
export function useSignal<T extends Callback>(signal: RBXScriptSignal<T>, callback: T) {
	const latest = useRef(callback);
	latest.current = callback;
	useEffect(() => {
		const connection = signal.Connect(((...args: unknown[]) => (latest.current as Callback)(...args)) as T);
		return () => connection.Disconnect();
	}, [signal]);
}
```

- Le paquet `@rbxts/pretty-react-hooks` fournit déjà des hooks de ce genre (`useEventListener`, `useMotion` pour ripple, des hooks de délai et d'anti-rebond). Vérifie les noms exacts dans le paquet installé avant de les utiliser. La 0.6.4 dépend de `@rbxts/ripple` ^0.9, alors que ripple 0.10 a changé d'API (npm, 2026-10-18).
- Les hooks maison les plus utiles : `useInterval` (accumule `dt` et soustrait la période pour ne pas dériver), `useTimeout` (`task.delay` et `task.cancel`), `useDebounce`, `usePrevious` et `useCooldown` (booléen en state, progression en binding).

---

## 7. Animer sans re-render

### Principes

- L'état logique change tout de suite. L'animation ne fait que le montrer. Aucun état serveur ne doit attendre la fin d'une animation.
- Anime via des **bindings** : `setValue` écrit directement la propriété de l'instance, sans réconciliation ni re-render. Un binding ne peut pas ajouter ni retirer d'enfants : la structure passe par l'état.
- API : `useBinding(init)` renvoie `[binding, set]`, `binding.map(f)` dérive une prop (nombre vers `UDim2` ou `Color3`), `joinBindings({ x, y })` combine, `binding.getValue()` lit, `createBinding` crée un binding hors composant.
- Les fonctions passées à `map` tournent à chaque frame animée. Garde-les légères.
- Le setter d'un `createBinding` peut être appelé par du code hors React, par exemple à la réception d'une remote : l'UI suit sans aucun rendu.

### Quel outil

| Besoin | Outil |
|---|---|
| Hover, appui, apparition, transitions interruptibles | Ressort qui écrit dans un binding (section 8), ou ripple (`@rbxts/ripple` ; le `useMotion` de `@rbxts/pretty-react-hooks` 0.6.4 vise ripple 0.9, voir section 6), ou le portage de react-spring en Luau |
| Boucle continue : flottement, pulsation, reflet | Une horloge globale (`createBinding` + une connexion `RenderStepped`, dont `PreRender` est le nom moderne) et des `map` |
| Courbe Roblox précise (Back, Elastic, Bounce) ou animation ponctuelle sur une instance | `TweenService` via une ref. Annule le tween précédent sur la même propriété avant d'en lancer un. |
| Fondu de tout un panneau | `canvasgroup` et `GroupTransparency` (tester sur mobile : des bugs de rendu sont signalés sur certains appareils) |
| Fusion | `Spring` et `Tween` intégrés |

### Réglages de ressort

Le ressort façon Apple se règle par une durée et un rebond. Conversion : fréquence = 1 / durée, amortissement = 1 − rebond.

| Usage | Durée | Rebond | Pourquoi |
|---|---|---|---|
| Appui d'un bouton | 0,12 s | 0 | Retour immédiat au doigt. |
| Relâchement | 0,3 s | 0,3 | Petit « pop » ludique. |
| Survol à la souris | 0,2 s | 0 | Discret, fréquent. |
| Ouverture d'un menu | 0,45 s | 0,2 | Arrivée vivante. |
| Fermeture | 0,25 s | 0 | La sortie est plus rapide que l'entrée. |
| Apparition de cartes | 0,35 s | 0,3 | En cascade de 30 à 80 ms entre deux cartes. |
| Indicateur d'onglet, carrousel | 0,3 s | 0,15 | Déplacement à l'écran. |
| Compteur, barre de progression | 0,5 à 0,6 s | 0 | Lisible, jamais de rebond sur un nombre. |

### Durées TweenService et styles de jeu

Si tu passes par `TweenService`, ces durées « sonnent juste » (conventions de terrain, pas des règles Roblox) :

| Geste | Durée et easing |
|---|---|
| Survol d'un bouton | 0,08 à 0,12 s, Quad Out |
| Appui | 0,05 à 0,08 s, Quad Out ; relâche en Back Out |
| Ouverture d'une fenêtre (pop) | 0,2 à 0,3 s, Back Out |
| Fermeture | 0,12 à 0,18 s, Quad In |
| Panneau qui glisse | 0,25 à 0,35 s, Quint ou Exponential Out |
| Compteur de récompense | 0,6 à 1,2 s, Quad Out |
| Pulsation ou reflet en boucle | 1 à 2 s, Sine InOut |

Réglages par famille de style, pour coller à une maquette :

| Style | Boutons | Fenêtre | Signature |
|---|---|---|---|
| Simulator (couleurs vives, gros contours) | 1,06× au survol, 0,92× à l'appui, 0,08 s | 0,8 → 1 en 0,25 s Back Out | Compteur de 0,8 s et « punch » de l'icône 1,2 → 1 ; reflet sur la meilleure offre toutes les 3 s ; rayons qui tournent lentement derrière une récompense |
| Cartoony (rebondi) | Relâche en Elastic Out 0,3 s | 0,6 → 1 en 0,35 s Back Out | Bouton principal qui flotte de ±3 px en 1,6 s |
| Stud (blocs, style Roblox classique) | 0,94× en 0,05 s, relâche 0,1 s Back Out | Monte de 20 px avec un fondu en 0,2 s | Mouvements secs et courts |
| Minimal | Pas de rebond | Fondu 0,15 s, glissement de 8 à 12 px en 0,2 s | Sobriété, aucune boucle décorative |

- Garde les icônes immobiles si l'utilisateur ne demande pas d'animation : les fenêtres et les boutons suffisent à donner de la vie.
- Pour l'ouverture d'une boutique « riche » : fenêtre qui remonte de 4 % et grossit de 0,78 à 1 en 0,2 s, flou de fond (`BlurEffect` dans Lighting, taille 14) en 0,15 s, puis cartes en cascade (0,04 s + 0,025 s par carte).

### Règles de mise en mouvement

- **Un seul `UIScale` par objet, plusieurs causes** : si un même objet doit à la fois s'adapter à l'écran, s'ouvrir et réagir au bouton, calcule `Scale = base × ouverture × effet` et anime chaque facteur séparément. Sinon les animations s'écrasent entre elles.
- Précharge les images d'un écran avec `ContentProvider:PreloadAsync` dans un `task.spawn` au démarrage, pour qu'une fenêtre ne s'affiche pas à moitié vide.
- `UIScale` agrandit **depuis l'AnchorPoint**. Pour un pop centré, mets `AnchorPoint = (0.5, 0.5)` et `Position = (0.5, 0, 0.5, 0)`.
- Le layout impose la position de ses enfants. Pour animer un enfant d'un `uilistlayout` ou `uigridlayout`, utilise un cadre « slot » transparent de taille fixe, géré par le layout, qui contient l'élément animé.
- N'anime pas une grosse UI case par case : anime un parent stable, une `UIScale` ou un `canvasgroup`.
- **Sortie animée** : garde le composant monté pendant la fermeture, puis démonte après la durée de sortie avec `task.delay`. Incrémente un jeton pour annuler le démontage si l'utilisateur rouvre entre-temps.
- **Survol** : ne réagis qu'à la souris. Au doigt, `MouseEnter` part sur un tap et l'élément reste grossi. Teste `UserInputService.PreferredInput === Enum.PreferredInput.KeyboardAndMouse`.
- **Mouvement réduit** : si `GuiService.ReducedMotionEnabled` est vrai, coupe les déplacements et les échelles. Garde les fondus et les compteurs.
- Fréquence d'usage : un menu ouvert de temps en temps mérite une animation. Une action répétée des centaines de fois, comme un raccourci clavier, n'en mérite pas.

### Aide-mémoire du « juice »

- Pop de validation ou de récompense : 1,1× en 0,1 s, retour en 0,15 s. C'est le minimum, avec un état désactivé assombri (un calque sombre, pas une recoloration).
- Popup de gain, la récompense la plus vue : échelle 0 → 1,3 → 1 avec une légère rotation, montant qui défile avec abréviation (`12.4K`) puis exact, icônes qui volent vers le compteur. Secousse seulement au-delà d'un seuil, et si le réglage du joueur l'autorise.
- Ouverture de boutique : 0,2 à 0,3 s. Une boutique qui s'ouvre d'un coup paraît cassée.
- Nombres qui défilent vers la nouvelle valeur plutôt qu'un saut, avec un petit saut d'échelle à chaque gain.
- Tremblement de 2 à 4 px pendant 0,15 s, seulement pour les gros gains.
- Anticipation de 0,3 s (lueur, tremblement) avant une révélation d'œuf ou de récompense.
- Reflet qui balaie un bouton d'achat toutes les 2,5 à 3,5 s, seulement sur les boutons visibles.
- Sur téléphone, jamais plus de deux popups empilées, et jamais sur le joystick, le bouton de saut ou la croix d'un menu.

---

## 8. Boîte à outils d'animation (code testé)

Ce module a compilé avec roblox-ts 3.0 et tourné dans Studio le 2026-10-18. Copie-le dans `src/client/ui/motion.tsx`.

```tsx
import React, { useBinding, useEffect, useMemo, useRef, useState } from "@rbxts/react";
import { GuiService, RunService, UserInputService } from "@rbxts/services";

export interface SpringConfig {
	frequency: number; // oscillations par seconde (1 / durée)
	damping: number; // 1 = sans rebond, < 1 = rebond
	transform: boolean; // false pour couleurs, transparences et nombres, gardés en mouvement réduit
}

// Ressort façon Apple : une durée et un rebond
function spring(duration: number, bounce: number, transform = true): SpringConfig {
	return { frequency: 1 / duration, damping: 1 - bounce, transform };
}

export const SPRINGS = {
	press: spring(0.12, 0),
	release: spring(0.3, 0.3),
	hover: spring(0.2, 0),
	menuIn: spring(0.45, 0.2),
	menuOut: spring(0.25, 0),
	pop: spring(0.35, 0.3),
	slide: spring(0.3, 0.15),
	counter: spring(0.6, 0, false),
	fade: spring(0.25, 0, false),
};

export const reducedMotion = () => GuiService.ReducedMotionEnabled;
export const canHover = () => UserInputService.PreferredInput === Enum.PreferredInput.KeyboardAndMouse;

export class Spring {
	private value: number;
	private velocity = 0;
	private target: number;
	private connection?: RBXScriptConnection;

	constructor(
		initial: number,
		private config: SpringConfig,
		private readonly onChange: (value: number) => void,
	) {
		this.value = initial;
		this.target = initial;
	}

	setTarget(target: number, config?: SpringConfig) {
		if (config) this.config = config;
		this.target = target;
		if (this.config.transform && reducedMotion()) return this.snap(target);
		this.start();
	}

	// Coup de pouce : la valeur saute puis revient (pop d'un compteur, d'une icône)
	impulse(velocity: number) {
		if (this.config.transform && reducedMotion()) return;
		this.velocity += velocity;
		this.start();
	}

	snap(value: number) {
		this.stop();
		this.value = value;
		this.target = value;
		this.velocity = 0;
		this.onChange(value);
	}

	destroy() {
		this.stop();
	}

	private start() {
		if (this.connection) return;
		this.connection = RunService.RenderStepped.Connect((dt) => this.step(dt));
	}

	private stop() {
		this.connection?.Disconnect();
		this.connection = undefined;
	}

	private step(dt: number) {
		const w = 2 * math.pi * this.config.frequency;
		const z = this.config.damping;
		let remaining = math.min(dt, 0.1);
		while (remaining > 0) {
			const h = math.min(remaining, 1 / 240); // sous-pas pour rester stable
			const accel = w * w * (this.target - this.value) - 2 * z * w * this.velocity;
			this.velocity += accel * h;
			this.value += this.velocity * h;
			remaining -= h;
		}
		if (math.abs(this.target - this.value) < 1e-3 && math.abs(this.velocity) < 1e-3) {
			this.value = this.target;
			this.velocity = 0;
			this.stop(); // plus aucune connexion quand le ressort est au repos
		}
		this.onChange(this.value);
	}
}

// Ressort piloté à la main : un binding (aucun re-render) et le ressort
export function useSpringValue(initial: number, config: SpringConfig): LuaTuple<[React.Binding<number>, Spring]> {
	const [binding, setBinding] = useBinding(initial);
	const s = useMemo(() => new Spring(initial, config, setBinding), []);
	useEffect(() => () => s.destroy(), []);
	return [binding, s] as LuaTuple<[React.Binding<number>, Spring]>;
}

// Ressort qui suit une cible ; initial différent de target = animation au montage
export function useSpring(target: number, config: SpringConfig, initial = target, delay = 0) {
	const [binding, s] = useSpringValue(initial, config);
	useEffect(() => {
		if (delay <= 0) {
			s.setTarget(target, config);
			return;
		}
		const thread = task.delay(delay, () => s.setTarget(target, config));
		return () => task.cancel(thread);
	}, [target]);
	return binding;
}

export function useDelayed(delay: number) {
	const [ready, setReady] = useState(delay <= 0);
	useEffect(() => {
		if (delay <= 0) return;
		const thread = task.delay(delay, () => setReady(true));
		return () => task.cancel(thread);
	}, []);
	return ready;
}

// Horloge globale : une seule connexion pour toutes les boucles
const [clock, setClock] = React.createBinding(0);
let elapsed = 0;
RunService.RenderStepped.Connect((dt) => {
	elapsed += dt;
	setClock(elapsed);
});

// Oscillation en boucle : flottement d'icône, balancement de badge
export function useWave(amplitude: number, period: number, phase = 0) {
	return useMemo(
		() => clock.map((t) => (reducedMotion() ? 0 : math.sin(((t + phase) / period) * 2 * math.pi) * amplitude)),
		[amplitude, period, phase],
	);
}

// Progression 0 → 1 répétée (reflet qui balaie un bouton)
export function useLoop(period: number, phase = 0) {
	return useMemo(() => clock.map((t) => ((t + phase) % period) / period), [period, phase]);
}

// Échelle d'un bouton : survol, appui, rebond au relâchement
export function usePressScale(hoverScale = 1.05, pressScale = 0.92, disabled = false) {
	const [scale, s] = useSpringValue(1, SPRINGS.hover);
	const state = useRef({ hover: false, press: false });
	const update = () => {
		const st = state.current;
		if (disabled) return s.setTarget(1, SPRINGS.hover);
		if (st.press) return s.setTarget(pressScale, SPRINGS.press);
		s.setTarget(st.hover ? hoverScale : 1, SPRINGS.release);
	};
	const events = {
		MouseEnter: () => {
			if (!canHover()) return;
			state.current.hover = true;
			update();
		},
		MouseLeave: () => {
			state.current.hover = false;
			state.current.press = false;
			update();
		},
		MouseButton1Down: () => {
			state.current.press = true;
			update();
		},
		MouseButton1Up: () => {
			state.current.press = false;
			update();
		},
	};
	return { scale, events };
}

// Bouton animé : le cadre extérieur garde la place dans un layout, le bouton grossit depuis son centre
export function Pressable(props: {
	size: UDim2;
	position?: UDim2;
	layoutOrder?: number;
	zIndex?: number;
	color: Color3;
	transparency?: number;
	disabled?: boolean;
	hoverScale?: number;
	pressScale?: number;
	onClick?: () => void;
	children?: React.ReactNode;
}) {
	const { scale, events } = usePressScale(props.hoverScale, props.pressScale, props.disabled);
	return (
		<frame Size={props.size} Position={props.position} LayoutOrder={props.layoutOrder ?? 0} BackgroundTransparency={1}>
			<textbutton
				Size={UDim2.fromScale(1, 1)}
				Position={UDim2.fromScale(0.5, 0.5)}
				AnchorPoint={new Vector2(0.5, 0.5)}
				BackgroundColor3={props.color}
				BackgroundTransparency={props.transparency ?? 0}
				AutoButtonColor={false}
				Text=""
				ZIndex={props.zIndex ?? 1}
				Event={{ ...events, Activated: () => !props.disabled && props.onClick?.() }}
			>
				<uiscale Scale={scale} />
				{props.children}
			</textbutton>
		</frame>
	);
}

// Apparition : le contenu grossit depuis `from` en glissant, après `delay` (cascade)
export function Appear(props: {
	size: UDim2;
	position?: UDim2;
	layoutOrder?: number;
	delay?: number;
	from?: number;
	offsetX?: number;
	offsetY?: number;
	config?: SpringConfig;
	children?: React.ReactNode;
}) {
	const ready = useDelayed(props.delay ?? 0);
	const progress = useSpring(ready ? 1 : 0, props.config ?? SPRINGS.pop, 0);
	const from = props.from ?? 0.85;
	const dx = props.offsetX ?? 0;
	const dy = props.offsetY ?? 16;
	return (
		<frame Size={props.size} Position={props.position} LayoutOrder={props.layoutOrder ?? 0} BackgroundTransparency={1}>
			<frame
				Size={UDim2.fromScale(1, 1)}
				AnchorPoint={new Vector2(0.5, 0.5)}
				Position={progress.map((p) => new UDim2(0.5, (1 - p) * dx, 0.5, (1 - p) * dy))}
				BackgroundTransparency={1}
				Visible={ready}
			>
				<uiscale Scale={progress.map((p) => from + (1 - from) * p)} />
				{props.children}
			</frame>
		</frame>
	);
}

// Nombre qui défile vers sa nouvelle valeur et saute à chaque changement
export function AnimatedNumber(props: { value: number; format?: (n: number) => string; size: UDim2; font: Font; textSize: number; color: Color3 }) {
	const shown = useSpring(props.value, SPRINGS.counter);
	const [pop, popSpring] = useSpringValue(1, SPRINGS.release);
	const first = useRef(true);
	useEffect(() => {
		if (first.current) {
			first.current = false;
			return;
		}
		popSpring.impulse(4);
	}, [props.value]);
	const format = props.format ?? ((n: number) => tostring(n));
	return (
		<textlabel
			Text={shown.map((v) => format(math.round(v)))}
			Size={props.size}
			BackgroundTransparency={1}
			FontFace={props.font}
			TextSize={props.textSize}
			TextColor3={props.color}
		>
			<uiscale Scale={pop} />
		</textlabel>
	);
}

// Reflet blanc qui balaie un bouton (à mettre en enfant du bouton)
export function Shine(props: { cornerRadius: UDim; period?: number }) {
	const loop = useLoop(props.period ?? 3);
	return (
		<frame Size={UDim2.fromScale(1, 1)} BackgroundColor3={Color3.fromRGB(255, 255, 255)} BorderSizePixel={0} ZIndex={3}>
			<uicorner CornerRadius={props.cornerRadius} />
			<uigradient
				Rotation={20}
				Transparency={
					new NumberSequence([
						new NumberSequenceKeypoint(0, 1),
						new NumberSequenceKeypoint(0.45, 1),
						new NumberSequenceKeypoint(0.5, 0.55),
						new NumberSequenceKeypoint(0.55, 1),
						new NumberSequenceKeypoint(1, 1),
					])
				}
				// traverse pendant le premier tiers de la boucle, puis attend hors du bouton
				Offset={loop.map((t) => new Vector2(reducedMotion() ? -1 : math.min(t * 3, 1) * 2 - 1, 0))}
			/>
		</frame>
	);
}

// Menu plein écran : voile qui se fond, panneau qui monte, puis retombe hors de l'écran à la fermeture
export function MenuShell(props: { open: boolean; scale: number; size: UDim2; children?: React.ReactNode }) {
	const appear = useSpring(props.open ? 1 : 0, props.open ? SPRINGS.menuIn : SPRINGS.menuOut, 0);
	const fade = useSpring(props.open ? 1 : 0, SPRINGS.fade, 0);
	const travel = props.open ? 60 : 900;
	return (
		<frame
			Size={UDim2.fromScale(1, 1)}
			BackgroundColor3={Color3.fromRGB(10, 15, 30)}
			BackgroundTransparency={fade.map((f) => 1 - 0.6 * math.clamp(f, 0, 1))}
			Active={props.open}
		>
			<frame
				Size={props.size}
				Position={appear.map((v) => new UDim2(0.5, 0, 0.5, (1 - v) * travel))}
				AnchorPoint={new Vector2(0.5, 0.5)}
				BackgroundTransparency={1}
			>
				<uiscale Scale={appear.map((v) => props.scale * (0.9 + 0.1 * v))} />
				{props.children}
			</frame>
		</frame>
	);
}
```

Démontage différé d'un menu, avec annulation si on rouvre pendant la sortie :

```tsx
const EXIT_TIME = 0.3;

function App() {
	const [screen, setScreen] = useState<"none" | "shop">("shop");
	const [open, setOpen] = useState(true);
	const token = useRef(0);
	const show = (target: "shop") => {
		token.current++;
		setScreen(target);
		setOpen(true);
	};
	const close = () => {
		const mine = ++token.current;
		setOpen(false);
		task.delay(EXIT_TIME, () => {
			if (token.current === mine) setScreen("none");
		});
	};
	return (
		<screengui ResetOnSpawn={false} ZIndexBehavior={Enum.ZIndexBehavior.Sibling}>
			{screen === "none" && <OpenShopButton onClick={() => show("shop")} />}
			{screen === "shop" && <Shop open={open} onClose={close} />}
		</screengui>
	);
}
```

Écran qui tient sur toutes les tailles : suis la méthode de ui-fondamentaux.md (section 6).

- Tailles en unités d'un téléphone de 667 × 375. Grappes placées en Scale avec un `AnchorPoint`, car `UIScale` ne met à l'échelle que la taille, pas la position.
- Une `uiscale` par grappe reçoit l'échelle commune (`deviceScale`, calculée depuis `Workspace.CurrentCamera.ViewportSize`). Pousse-la dans un binding et multiplie-la avec celle des animations (`MenuShell` reçoit `scale`).
- Maquette dessinée en 1280 × 720 ? Divise ses tailles par 1,92 (1280 / 667). Il y faut au moins 24 px de texte et 85 px de bouton pour rester à 12 px et 44 px sur ce téléphone.

---

## 9. Performance

- `React.memo` compare les props par `==`. Les types Roblox (`UDim2`, `Color3`, `Vector2`) se comparent par valeur : un littéral recréé à l'identique ne casse pas `memo`. Les tables et les fonctions se comparent par référence.
- `useMemo` pour un calcul cher (tri, filtre, comptage). `useCallback` seulement si la fonction va à un enfant mémoïsé ou sert de dépendance. Partout ailleurs, c'est du coût sans gain.
- Listes : clé = identité de l'élément, jamais l'index. Avec l'index, l'état, le focus et les animations restent attachés au rang lors d'un tri. L'ordre visible passe par `LayoutOrder`.
- Au-delà d'une cinquantaine de lignes, virtualise : `CanvasSize` = total × hauteur de ligne, rends seulement les lignes visibles plus 3 de marge, suis `CanvasPosition` via `Change`. La plage visible change la structure, donc c'est un état, éventuellement limité en fréquence.
- Démonter ou cacher : démonte un écran rarement ouvert (boutique, réglages) ; garde `Visible={false}` pour un élément basculé souvent (HUD, barre de cooldown). Pour toute une couche, `Enabled={false}` sur la `screengui` coûte le moins (ui-fondamentaux.md).
- Rafales d'éléments éphémères (dégâts flottants, kill feed) : un pool d'instances hors React coûte moins cher qu'un montage par élément.
- Repères pratiques (heuristiques, pas officiels) : quelques ViewportFrame à la fois au plus, une cinquantaine d'UIStroke actifs, une dizaine de CanvasGroup. Mesure avant d'optimiser : MicroProfiler et `debug.profilebegin`.
- **Test de fuite à la réouverture** : note le nombre de descendants du conteneur, ouvre et ferme l'écran 10 fois, vérifie le retour au compte de départ. Recommence en fermant pendant une animation et pendant une réponse réseau.

---

## 10. Modèle 3D dans l'UI

Un pet ou un objet qui tourne dans une carte de boutique : `viewportframe` + une `Camera` créée côté client (les caméras ne se répliquent pas). Ajoute un `WorldModel` dans le ViewportFrame si le modèle doit jouer des animations d'Animator. Testé le 2026-10-18 ; la ref `speed` a été ajoutée après le test :

```tsx
import React, { useEffect, useRef } from "@rbxts/react";
import { RunService } from "@rbxts/services";

export function ModelPreview(props: { template: Model; size: UDim2; spinSpeed?: number }) {
	const ref = useRef<ViewportFrame>();
	const speed = useRef(0.8);
	speed.current = props.spinSpeed ?? 0.8; // lu à chaque frame : un changement de vitesse est suivi sans tout recréer
	useEffect(() => {
		const viewport = ref.current;
		if (viewport === undefined) return;
		const world = new Instance("WorldModel"); // nécessaire pour les animations
		world.Parent = viewport;
		const model = props.template.Clone();
		model.PivotTo(new CFrame(0, 0, 0));
		model.Parent = world;
		const camera = new Instance("Camera");
		camera.FieldOfView = 40;
		camera.CFrame = CFrame.lookAt(new Vector3(0, 4, 12), new Vector3(0, 0, 0));
		camera.Parent = viewport;
		viewport.CurrentCamera = camera;
		let angle = 0;
		const connection = RunService.RenderStepped.Connect((dt) => {
			angle += dt * speed.current;
			model.PivotTo(CFrame.Angles(0, angle, 0));
		});
		return () => {
			connection.Disconnect();
			world.Destroy();
			camera.Destroy();
		};
	}, [props.template]);
	return (
		<viewportframe
			ref={ref}
			Size={props.size}
			BackgroundTransparency={1}
			Ambient={Color3.fromRGB(150, 140, 160)}
			LightColor={Color3.fromRGB(255, 245, 235)}
			LightDirection={new Vector3(-1, -1, -0.5)}
		/>
	);
}
```

- Limite le nombre de ViewportFrame visibles en même temps : chacun coûte un rendu.
- Pour des icônes figées, préfère une image uploadée : moins cher qu'un rendu 3D permanent.

---

## 11. Stories et aperçus

UI Labs est un plugin Studio façon Storybook. Il affiche un composant sans lancer le jeu et le recharge à chaud. Plugin : asset 14293316215. Paquets optionnels : `@rbxts/ui-labs` en npm, `pepeeltoro41/ui-labs` en Wally.

- Une story est un ModuleScript dont le nom finit par `.story`. En roblox-ts, `Button.story.tsx` compile en `Button.story.luau`.
- Un storybook est un ModuleScript `.storybook` qui renvoie `{ storyRoots: [dossier] }`. Sans storybook, les stories vont dans « Unknown Stories ».
- Story React (forme objet, sans le paquet ; format vérifié sur la doc d'UI Labs le 2026-10-18). `story` est rappelée à chaque changement de contrôle ; `renderer: "legacy"` passe à `createLegacyRoot`.

```tsx
// src/client/ui/BuyButton.story.tsx
import React from "@rbxts/react";
import ReactRoblox from "@rbxts/react-roblox";
import { BuyButton } from "./BuyButton";

export = {
	react: React,
	reactRoblox: ReactRoblox,
	controls: { label: "Acheter", disabled: false },
	story: (props: { controls: { label: string; disabled: boolean } }) => (
		<BuyButton label={props.controls.label} disabled={props.controls.disabled} onBuy={() => print("achat")} />
	),
};
```

- Une story utilise des données simulées via `controls`. Elle n'importe jamais `Players.LocalPlayer` ni de RemoteEvent.
- Teste dans la story les cas extrêmes : nom très long, liste vide, nombre énorme, texte non latin.

---

## 12. Fusion 0.3 et Vide en bref

Pour les projets Luau qui ont choisi le réactif. Ne mélange pas avec React.

### Fusion 0.3

- Tout vit dans un **scope** : `local scope = Fusion.scoped(Fusion)`. `scope:doCleanup()` détruit tout, en ordre inverse. Pas de scope, c'est une fuite.
- État : `scope:Value(x)`, `v:set(y)`, `peek(v)` pour lire sans s'abonner. Dérivé : `scope:Computed(function(use, scope) … end)`, où `use(x)` lit et abonne. Le calcul est pur, immédiat, sans yield ni son ni réseau. Les effets de bord vont dans `scope:Observer(état):onChange(cb)`, qui renvoie une fonction de déconnexion. Ce callback peut partir à un moment inattendu, par exemple pendant le nettoyage du scope : vérifie que ce qu'il touche existe encore.
- Instances : `scope:New "Frame" { … }`. Clés spéciales importées de `Fusion` : `Children`, `OnEvent`, `OnChange`, `Out`, `Attribute`, `AttributeChange`, `AttributeOut`.
- `scope:Hydrate(instance)` **ajoute l'instance au scope** : elle sera détruite au nettoyage. Clone d'abord un template de designer qui doit survivre.
- UI éphémère (menu déroulant, modale) : `scope:innerScope()`, nettoyé avec le parent.
- UI conditionnelle : un `Computed` qui renvoie `nil` ou un `scope:New` créé sur son scope interne. Passé à `[Children]`, l'ancien enfant est détruit à chaque bascule, sans `Destroy` à la main.
- Listes à identité stable : `ForValues`, `ForKeys`, `ForPairs` plutôt qu'un Computed qui reconstruit tout.
- Animation : `scope:Spring(source, vitesse, amortissement)` survit aux changements de cible (hover, drag). `scope:Tween(source, TweenInfo)` repart à chaque changement (transitions à durée fixe).
- Composant = `(scope, props) -> Instance`. Props en `Fusion.UsedAs<T>`. `UsedAs<T>?` veut dire optionnelle avec défaut. `UsedAs<T?>` veut dire que `nil` est une valeur voulue.
- Ne mélange pas les tutoriels 0.2 (sans scopes) avec 0.3.

### Vide

- Réactivité fine : sources, effets et `cleanup()`. Le nettoyage d'un scope libère le graphe, mais l'écran doit quand même dé-parenter sa racine à la fermeture.
- Au nettoyage de sa racine, Fluid (une bibliothèque dérivée de Vide) ne détruit pas les instances créées par `create()`. Avec Vide comme avec Fluid, vérifie par le test de fuite à la réouverture (section 9).

---

## 13. Pièges

- Pas d'UI et « Infinite yield … @rbxts-js » : le scope `@rbxts-js` n'est pas mappé dans `default.project.json`.
- `createRoot(playerGui)` : au montage, la racine détruit les autres enfants de PlayerGui, dont le `TouchGui` des contrôles tactiles. Racine dans un `Folder`, portail vers PlayerGui.
- Du texte brut entre balises TSX : il faut la prop `Text`.
- `<div>`, `onClick`, `className` : ça n'existe pas sous Roblox.
- Un signal connecté hors `useEffect`, ou un effet sans nettoyage : fuite à chaque ouverture.
- Un `WaitForChild` sans timeout ou un autre yield dans un rendu ou un effet synchrone.
- Un hook dans un `if`.
- Des données haute fréquence dans un Context ou en state, au lieu d'un binding.
- Des clés de liste par index, puis un tri qui mélange les états.
- Une animation par `RenderStepped` qui reste connectée après l'arrêt : le ressort doit se déconnecter au repos.
- Le survol qui reste collé sur téléphone, faute de test de la souris.
- Un `UIScale` sur un élément ancré en (0, 0), qui grossit vers le bas à droite.
- Une position animée sur un enfant de layout, écrasée par le layout.
- L'UI qui accorde un objet ou de la monnaie sans attendre le serveur.
- `defaultProps` sur un composant fonction : ignoré.
- Fusion : `Hydrate` sur un template qui devait survivre ; effets de bord dans un `Computed`.
- Modifier un `.luau` généré dans `out/` au lieu du `.tsx`.

---

## 14. Checklist

- [ ] Le framework suit celui du projet ; aucun mélange.
- [ ] En roblox-ts : `@rbxts-js` est mappé, et `rbxtsc` compile sans erreur.
- [ ] La racine est montée une seule fois, dans un `Folder` avec un portail vers PlayerGui, avec `ResetOnSpawn={false}` et les bons `ScreenInsets`.
- [ ] Chaque signal est connecté dans un `useEffect` qui renvoie son nettoyage.
- [ ] Aucun yield dans un rendu, un reducer ou un `useMemo`.
- [ ] Ce qui change souvent passe par des bindings ; la structure passe par l'état.
- [ ] Aucune donnée haute fréquence dans un Context.
- [ ] Les listes ont des clés stables par identité et un `LayoutOrder` explicite.
- [ ] Les animations sont interruptibles, se coupent au repos et respectent `ReducedMotionEnabled`.
- [ ] Le survol est réservé à la souris.
- [ ] La sortie d'un menu est animée et le démontage annulable.
- [ ] L'UI envoie des intentions au serveur et affiche sa réponse ; elle ne décide rien.
- [ ] Le test de fuite (10 ouvertures et fermetures) revient au compte de départ.
- [ ] L'écran a été vérifié à la taille d'un téléphone et d'un PC (voir ui-fondamentaux.md).
