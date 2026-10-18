# Open Cloud : API web Roblox, clés, assets, publication

À lire quand : un outil, une CI ou l'agent doit agir sur Roblox sans Studio (uploader une image, un son ou un mesh et récupérer son asset id, publier une place, lire ou réparer un DataStore, prévenir les serveurs, lancer des tests sur les serveurs Roblox), ou quand le jeu appelle une API Roblox par HttpService. Les API du moteur sont dans donnees.md et monetisation.md, la chaîne d'outils dans outils.md.

## Sommaire

1. Moteur ou Open Cloud
2. Trouver le bon endpoint
3. Clés API
4. OAuth 2.0 en bref
5. Requêtes, erreurs, limites de débit
6. Assets : uploader et récupérer l'asset id
7. Pipeline d'un agent : SVG → PNG → asset id
8. Publier une place
9. Luau Execution : tests sur les serveurs Roblox
10. DataStores, MemoryStores, messages
11. HttpService vers Open Cloud depuis le jeu
12. Webhooks
13. Pièges
14. Checklist

## 1. Moteur ou Open Cloud

L'API moteur (Luau, `game:GetService`) et Open Cloud (REST sous `https://apis.roblox.com`, avec `x-api-key` ou OAuth) sont deux systèmes séparés. Chercher dans le mauvais index donne du code qui ne marche pas.

| Besoin | Outil |
|---|---|
| Données d'un joueur pendant le jeu | `DataStoreService`, moteur (donnees.md) |
| Migration, nettoyage, effacement RGPD en masse, outil d'admin | Open Cloud Data Stores |
| Publier une place depuis la CI | Open Cloud, versions de place |
| Upload d'images, sons, modèles par pipeline | Open Cloud Assets |
| Tests dans le vrai moteur, en CI | Luau Execution |
| Prévenir tous les serveurs depuis l'extérieur, les redémarrer | message d'univers, `RestartUniverseServers` |
| Vendre dans le jeu | `MarketplaceService`, moteur (monetisation.md) |
| Créer ou modifier game passes et developer products | endpoints Open Cloud dédiés |
| Changer le rôle de groupe d'un joueur depuis le jeu | HttpService + `AssignRoleGroupMembership` (`UpdateGroupMembership` est déprécié) |
| Notifier un joueur hors du jeu | `CreateUserNotification` (`POST /cloud/v2/users/{user_id}/notifications`, portée `user.user-notification:write`) |
| Abonnements et remboursements en temps réel | webhooks |

- N'utilise pas Open Cloud pour ce que le moteur fait déjà : tu ajoutes latence, dépendance et consommation de quota.
- **Propose, n'impose pas.** Quand l'utilisateur fait à la main ce qu'Open Cloud automatise (upload en lot, prix, descriptions), nomme l'option et ce qu'elle ferait, puis laisse-le choisir. Ne suppose ni clé, ni permission, ni quota.
- **Toute écriture Open Cloud touche des données réelles.** Confirme avant, après un aperçu à blanc (méthode, URL, corps résumé, sans réseau). Une publication `Published` demande un oui explicite à chaque fois.
- Les clés, Luau Execution et la plupart des appels sont gratuits. Quelques-uns dépensent des Robux : création de badge, certains uploads comme la vidéo (frais plafonnés par `creationContext.expectedPrice`). Jamais sans l'accord de l'utilisateur.
- **Prix et produits** : passe par les API Open Cloud, en beta : `/game-passes/v1/universes/{universeId}/game-passes` (portée `game-pass:write`) et `/developer-products/v2/universes/{universeId}/developer-products` (portée `developer-product:write`). Leur `PATCH` est en `multipart/form-data` (`price`, `isForSale`…) et ne change que les champs envoyés (openapi.json, 2026-10-18). D'anciens endpoints de gestion sont dépréciés depuis le 2026-04-23 (date à vérifier). Ne change un prix qu'avec le nombre exact donné par l'utilisateur, rapporte ancien → nouveau, et vérifie que le prix affiché en jeu suit.
- Après le 2026-10-19, les anciennes API web par cookie ou sans authentification (badges, datastores, game persistence, inventory, thumbnails, users) ont des limites bien plus basses, fixées endpoint par endpoint (par exemple 100, 150 ou 260 requêtes par minute pour des endpoints de thumbnails). Celles d'Open Cloud ne changent pas : migre ces scripts vers une clé API ou OAuth.

## 2. Trouver le bon endpoint

- Index pour agents : `https://create.roblox.com/docs/cloud/llms.txt` (cherche dans Features, puis Domains). Spécification : `https://create.roblox.com/docs/cloud/openapi.json` (OpenAPI 3.0.4). **Lis-les avant d'appeler un endpoint**, au lieu de réciter un chemin de mémoire. Ajoute `.md` à l'URL d'une page de doc pour l'avoir en Markdown.
- L'outil MCP `http_get` de Studio refuse `openapi.json` (liste blanche : `.md` et `llms.txt`) : utilise l'outil web de l'hôte ou `curl`.
- Pour chaque opération, lis dans l'OpenAPI : `x-roblox-scopes` (portées), `x-roblox-rate-limits`, `x-roblox-engine-usability` (dont `apiKeyWithHttpService` : appel possible depuis le jeu), `x-roblox-stability` (beta), `x-roblox-deprecated` ou `deprecated: true`, `x-roblox-recommended-alternatives` (remplaçant conseillé), `x-roblox-size-limit` ; une dépréciation peut aussi n'être écrite que dans la `description` (« **Deprecated.** »).
- La plupart des ressources sont sous `https://apis.roblox.com/cloud/v2/…`, certaines restent en v1 (`/assets/v1/…`, `/universes/v1/…/versions`). Ne réécris jamais un chemin documenté vers ta version préférée, et ne déduis jamais un chemin, une portée ou un schéma de l'exemple d'une autre ressource. Un client généré depuis l'OpenAPI se relit à la main : authentification, sondage des opérations, masques de champs.

## 3. Clés API

### Créer une clé : c'est l'utilisateur qui le fait

Creator Dashboard, Open Cloud, API Keys, Create. L'agent ne crée jamais de clé et n'en voit jamais la valeur.

- Un nom par usage (`PLACE_PUBLISHING_KEY`, `ASSET_UPLOAD_KEY`) et une clé par application.
- Restreins-la à **une expérience** et aux seules opérations utiles : une clé fuitée ne doit toucher qu'un jeu.
- Liste blanche d'IP en CIDR (`192.168.0.0/24`), sauf pour une clé appelée depuis les serveurs Roblox, dont les IP sont imprévisibles.
- Une expiration pour une clé courte, une rotation prévue pour une clé longue. La valeur ne s'affiche qu'une fois.
- La clé hérite des droits de son propriétaire : ses expériences et celles des groupes où il a un rôle. Pour un jeu de groupe, crée un compte dédié à l'automatisation, invite-le avec le rôle minimal et génère la clé depuis ce compte.
- Une clé ne crée pas d'expérience. Dans un vrai projet (octobre 2026), la création de badge et le téléchargement de la place publiée ont aussi échoué avec une clé. Pourtant openapi.json liste un endpoint expérimental de création de badge, qui dépense des Robux (portée `legacy-universe.badge:manage-and-spend-robux`), et une Asset Delivery API (portée `legacy-asset:manage`) qui dit servir tous les types d'assets (à vérifier). Par défaut, l'utilisateur fait ces actions dans le Creator Hub, puis tu cuis dans le code les ids qu'il te donne.

| Usage | Permission (nom exact dans la doc) |
|---|---|
| Publier une version de place | `universe-places`, écriture |
| Luau Execution | `universe.place.luau-execution-session`, lecture et écriture |
| Upload d'assets | `assets`, lecture et écriture (portées OAuth `asset:read`, `asset:write`) (doc, 2026-10-18) |
| Lister, lire des entrées de DataStore | `universe-datastores.objects:list`, `universe-datastores.objects:read` |
| Vider un MemoryStore | `universe.memory-store:flush` (`memory-store:flush` dans openapi.json) |
| Titre, description de l'expérience ; redémarrer ses serveurs | `universe`, écriture (`universe:write`) |

Modifier les permissions d'une clé garde la même valeur : rien à redistribuer.

### Garder une clé

- L'utilisateur la met dans la variable `ROBLOX_API_KEY`. Sous Windows : `$env:ROBLOX_API_KEY = Read-Host -MaskInput "Clé"` (PowerShell 7.1 ou plus) pour la session, car une affectation tapée en clair reste dans l'historique PSReadLine ; ou un `.env.ps1` ignoré par git qu'il charge à la demande. `setx` la rend permanente mais l'écrit en clair dans le registre. En CI : le coffre de secrets du dépôt.
- **L'agent ne demande jamais la clé dans le chat, ne l'écrit dans aucun fichier, ne la passe jamais en argument de commande (visible dans la liste des processus) et ne l'affiche jamais.** Si l'utilisateur colle une clé dans le chat, ne l'utilise pas et ne la répète pas : dis-lui de la révoquer, d'en générer une autre et de la mettre dans la variable.
- Jamais de clé ni de cookie `.ROBLOSECURITY` dans le code, la place (Script, attribut, `StringValue`), ReplicatedStorage, une URL, un log, un commit ou une capture : tout ce qui est répliqué se décompile. En cas de fuite, révoque d'abord, nettoie l'historique ensuite.
- Sépare les clés de dev, de CI et de production. Désactive les clés inutilisées sans attendre leur expiration.

### Cycle de vie et diagnostic

- États : Active, Disabled, Expired, **Auto-Expired** (ni utilisée ni modifiée pendant 60 jours, même sans date d'expiration), Revoked (le compte d'une clé de groupe a perdu l'accès), Moderated, User Moderated.
- L'auto-expiration est la cause n° 1 de « mon automatisation s'est arrêtée sans que je change rien ». Correctif : désactiver puis réactiver la clé, ou en modifier une propriété, puis prévoir un usage régulier ou un rappel. Revoked et Moderated se réparent par « Regenerate Key », qui change la valeur.
- `POST https://apis.roblox.com/api-keys/v1/introspect` avec `{"apiKey": "…"}` renvoie nom, `authorizedUserId`, portées et ressources, `enabled`, `expired`, `expirationTimeUtc`, vus depuis l'IP appelante : c'est le premier test sur un 401 ou un 403. Passe le corps par l'entrée standard pour que la clé n'apparaisse dans aucune ligne de commande : `printf '{"apiKey":"%s"}' "$ROBLOX_API_KEY" | curl -sS -X POST https://apis.roblox.com/api-keys/v1/introspect -H "Content-Type: application/json" --data-binary @-`.

## 4. OAuth 2.0 en bref

- **Clé API** : automatisation du propriétaire (CI, bot, backend, worker de webhook, appel depuis le jeu). **OAuth 2.0** : une application qui agit pour d'autres utilisateurs, avec leur consentement. Ne mélange jamais les deux dans une requête, ni en secours l'un de l'autre.
- Flux : code d'autorisation avec PKCE (`S256`) pour tous les clients, obligatoire pour un client public (navigateur, mobile), qui ne peut pas garder de secret.
- Endpoints sous `https://apis.roblox.com/oauth` : `GET /v1/authorize` (`client_id`, `redirect_uri` exact, `scope` séparés par des espaces, `response_type=code`, `state`, `code_challenge`, `code_challenge_method=S256`, `nonce` pour lier l'identité) ; `POST /v1/token` en `application/x-www-form-urlencoded` ; `POST /v1/token/introspect`, `/v1/token/resources`, `/v1/token/revoke` ; `GET /v1/userinfo` ; `GET /.well-known/openid-configuration`.
- Durées : code 1 minute à usage unique ; jeton d'accès 15 minutes ; refresh token 90 jours (doc, 2026-10-18), **à usage unique et tournant**. Après chaque refresh, remplace le refresh token stocké de façon atomique, avec un seul propriétaire des refresh concurrents : réutiliser l'ancien fait échouer la session.
- `state` aléatoire, lié à une tentative, vérifié au retour ; vérificateur PKCE neuf à chaque tentative. Un `state` absent ou différent est une rupture d'intégrité, pas une erreur à réessayer.
- `introspect` dit si un jeton est actif, pas s'il ouvre une ressource ; un jeton d'accès révoqué peut y rester actif jusqu'à son expiration. `token/resources` liste les univers, groupes et utilisateur vraiment accordés : appelle-le avant d'afficher un choix d'univers.
- Portées minimales : `openid` seulement pour un ID token, `profile` seulement avec `openid`, puis la portée exacte de chaque endpoint (`x-roblox-scopes`). Changer les portées force les utilisateurs à réautoriser.
- Enregistrement : Creator Dashboard, OAuth 2.0 Apps. Le secret client s'affiche une fois. L'app reste privée (10 utilisateurs au plus) jusqu'à la revue publique, qui demande une vidéo de démo d'une minute au plus. Redirections exactes : 10 au plus, 256 caractères chacune ; `http://localhost:3000/oauth/callback` est permis en dev. Utilisateurs de 13 ans ou plus ; développeur vérifié par pièce d'identité.
- Jamais de secret client dans l'URL d'autorisation ni dans le code du navigateur ; refresh tokens côté serveur seulement ; jamais de code ni de jeton dans un log.

## 5. Requêtes, erreurs, limites de débit

- En-têtes : `x-api-key` pour une clé, `Content-Type: application/json` dès qu'il y a un corps JSON.
- **Pagination** : `maxPageSize`, `pageToken`, réponse `nextPageToken`. Garde exactement les mêmes paramètres d'une page à l'autre (changer un filtre et réutiliser un jeton donne un 400). Arrête-toi quand `nextPageToken` est vide ou absent.
- **Opérations longues** : un POST peut renvoyer une ressource Operation. Sonde son chemin avec backoff, jitter et échéance jusqu'à `done: true`, et remonte l'erreur finale de l'opération au lieu de déclarer la requête réussie.
- **Mise à jour partielle** : `PATCH …?updateMask=champ1,champ2`, avec seulement ces champs dans le corps. Pas partout : une entrée de DataStore se remplace en entier (section 10).
- Types JSON : dates RFC 3339 (`2026-10-18T12:00:00Z`), durées en chaîne (`"300s"`), octets en base64.

| Statut | Sens | Conduite |
|---|---|---|
| 400 `INVALID_ARGUMENT` | id, filtre, masque, en-tête ou corps mal formés | corrige la requête |
| 401 | clé fausse ou expirée, IP refusée | `introspect` |
| 403 `PERMISSION_DENIED`, `INSUFFICIENT_SCOPE` | portée manquante ou pas d'accès à la ressource | vérifie à part permission du créateur, portée et ressource accordée |
| 404 `NOT_FOUND` | chemin faux, ressource absente | vérifie les ids |
| 409 `ABORTED` | conflit d'état ; pour une publication, place hors de l'univers donné, ou Studio a la place ouverte | section 8 |
| 413 | fichier trop gros | réduis-le |
| 429 `RESOURCE_EXHAUSTED` | quota ou débit dépassé | attends, voir plus bas |
| 5xx, `UNAVAILABLE` | transitoire | réessaie si l'opération le permet |

- Ordre de diagnostic : domaine, version, chemin, méthode et content-type ; type d'identifiant ; permission du créateur ou du groupe ; portées ; ressources accordées (OAuth) ; schéma et masque ; quotas ; état de l'opération. N'élargis jamais les portées avant d'avoir trouvé la frontière qui bloque.
- Le format d'erreur varie entre v1, v2 et la passerelle. Journalise endpoint, id de requête, statut et code d'erreur Roblox, jamais la clé ni un jeton complet.

**Limites et retries**

- En-têtes à lire : `retry-after`, `x-ratelimit-limit`, `x-ratelimit-remaining`, `x-ratelimit-reset`.
- Sur 429 : attends `retry-after` ; à défaut `x-ratelimit-reset` (secondes avant la remise à zéro du quota selon la doc ; une valeur énorme serait un horodatage Unix) ; à défaut un backoff exponentiel avec jitter complet, plafonné vers 30 s, en nombre borné de tentatives. Ne dors jamais après la dernière tentative, et jamais de boucle serrée sur un 429 ou un 503.
- Classe chaque appel. **Lecture** : réessayable. **Écriture idempotente** (affecter un rôle précis, remplacer une valeur) : réessayable. **Écriture non sûre** (incrément, création sans identifiant stable, message publié) : un 429 veut dire « refusé, non traité », donc rejouable ; un 5xx, une erreur de transport ou un timeout laissent l'issue **inconnue** : relis la ressource avant de rejouer.
- Ne réessaie que le transitoire : authentification, permission et validation demandent une correction, pas une répétition.
- Chaque endpoint a aussi une limite par propriétaire de clé, d'où que viennent les appels. Regroupe par les endpoints de masse quand ils existent.

## 6. Assets : uploader et récupérer l'asset id

- `POST https://apis.roblox.com/assets/v1/assets` en `multipart/form-data`, en deux parties : `request` (JSON : `assetType`, `displayName`, `description`, `creationContext.creator` avec `userId` ou `groupId`) et `fileContent` (le fichier, avec son type MIME).
- La réponse est une Operation (`path` de la forme `operations/<id>`, `done`). Sonde `GET https://apis.roblox.com/assets/v1/operations/<id>` jusqu'à `done: true` ; l'id est dans `response.assetId` ; un échec arrive dans `error` (`code`, `message`) (doc, 2026-10-18). Lecture d'un asset : `GET /assets/v1/assets/{assetId}`, et ses versions. Ces endpoints sont en beta ; limites par propriétaire de clé : 120 créations, 300 lectures d'opération par minute (doc, 2026-10-18).

| Contenu | `assetType` | Formats | À savoir |
|---|---|---|---|
| Image | `Decal` (la doc liste aussi `Image`) | PNG, JPEG, BMP, TGA | pas de SVG ; moins de 8000 × 8000 px (doc, 2026-10-18) ; au-delà de 1024 px de côté, Roblox réduit (à vérifier) |
| Son | `Audio` | MP3, OGG, WAV, FLAC | 7 minutes au plus ; 100 uploads par mois avec une pièce d'identité vérifiée, 10 sans (doc, 2026-10-18) ; en modération (« Reviewing »), il peut rester muet un moment |
| Modèle 3D | `Model` | FBX, glTF, GLB, RBXM, RBXMX | arrive comme package ; seul un contenu FBX accepte une nouvelle version |
| Animation | `Animation` | RBXM, RBXMX | un fichier modifié hors de Studio peut échouer (doc, 2026-10-18) |

Tout fichier fait 20 Mo au plus par appel, sauf la vidéo (doc, 2026-10-18).

- `ImageLabel.Image` attend l'id d'une **Image** (type d'asset 1), pas celui d'un Decal : vérifie l'id obtenu dans Studio avant de le cuire dans le code.
- Uploade sous le créateur qui possède l'expérience (l'utilisateur, ou le groupe pour un jeu de groupe) : un asset d'un autre compte peut refuser de charger.
- Pour un nouveau modèle, l'Importer de Studio donne un aperçu et des avertissements que l'API n'a pas ; garde l'API pour les uploads répétés d'un pipeline.
- Dans Studio, le MCP a `upload_image` (images prises sur des URL, renvoie des `rbxassetid://`) et `store_image` (fichier local, renvoie un `IMAGEID_…` temporaire, pas un asset id) : voir studio-mcp.md.
- **N'invente jamais d'asset id.** Range les ids dans un seul module de manifeste, avec `rbxassetid://0` comme repère tant que l'upload n'est pas fait.

## 7. Pipeline d'un agent : SVG → PNG → asset id

1. Dessine l'icône en SVG : lisible, diffable, modifiable.
2. Rasterise en PNG transparent, à deux fois la taille d'affichage, 1024 px de côté au plus : `resvg -w 512 icone.svg build/icons/icone.png`. resvg s'installe avec l'accord de l'utilisateur ; Inkscape en ligne de commande marche aussi.
3. Préfère des icônes blanches ou grises : `ImageColor3` multiplie la couleur, et une seule image sert à plusieurs boutons (ui-fondamentaux.md).
4. Uploade avec Asphalt (étape 7) ou, sans outil installé, avec le script ci-dessous, qui lit la clé dans l'environnement et affiche un seul objet JSON.
5. Cuis l'id dans un module unique, par exemple `src/shared/assets.ts` : `export const ASSETS = { coin: "rbxassetid://123" } as const;`.
6. Vérifie dans Studio que l'image s'affiche ; une image en modération reste vide un moment. En attendant, un emoji dans un `TextLabel` fait une icône provisoire.
7. Garde dans le dépôt la table fichier → asset id, avec un hash du contenu, pour ne pas réuploader une image inchangée. Une image ou un son ne reçoit pas de nouvelle version : un fichier modifié donne un nouvel asset id. Asphalt (choix par défaut de paquets.md) automatise ce motif : `asphalt sync` uploade par Open Cloud avec la clé lue dans `ASPHALT_API_KEY`, garde un lockfile de hashes et génère un module Luau ou TypeScript ; `asphalt sync cloud --dry-run` montre ce qui changerait. Tarmac, ancien, s'authentifie par cookie `.ROBLOSECURITY` : évite-le.

Script d'upload, Node.js 18 ou plus (déjà présent avec roblox-ts) :

```js
// tools/upload-image.mjs : node tools/upload-image.mjs <fichier.png> <nom> [--dry-run]
// Variables : ROBLOX_API_KEY, et ROBLOX_USER_ID ou ROBLOX_GROUP_ID (propriétaire du jeu)
import { readFile } from "node:fs/promises";
import { basename, extname } from "node:path";

const [file, displayName, flag] = process.argv.slice(2);
const key = process.env.ROBLOX_API_KEY; // jamais affichée ni écrite
const group = process.env.ROBLOX_GROUP_ID;
const creator = group ? { groupId: group } : { userId: process.env.ROBLOX_USER_ID };
const mime = { ".png": "image/png", ".jpg": "image/jpeg", ".jpeg": "image/jpeg" }[extname(file ?? "").toLowerCase()];
const request = { assetType: "Decal", displayName, description: displayName, creationContext: { creator } };
const fail = (code, data) => {
	console.log(JSON.stringify({ ok: false, ...data }));
	process.exit(code);
};

if (!file || !displayName || !mime || !(creator.groupId || creator.userId)) fail(2, { error: "usage ou variables manquantes" });
if (flag === "--dry-run") {
	console.log(JSON.stringify({ ok: true, dryRun: true, url: "https://apis.roblox.com/assets/v1/assets", request }));
	process.exit(0);
}
if (!key) fail(2, { error: "ROBLOX_API_KEY absente" });

const headers = { "x-api-key": key };
const form = new FormData();
form.append("request", JSON.stringify(request));
form.append("fileContent", new Blob([await readFile(file)], { type: mime }), basename(file));
const res = await fetch("https://apis.roblox.com/assets/v1/assets", { method: "POST", headers, body: form })
	.catch((e) => fail(1, { error: `réseau, issue inconnue : ${e}` }));
let op = await res.json().catch(() => ({}));
if (!res.ok) fail(1, { status: res.status, error: op });

for (let i = 0; !op.done && i < 30; i++) {
	await new Promise((resolve) => setTimeout(resolve, Math.min(1000 * 2 ** i, 10000)));
	const poll = await fetch(`https://apis.roblox.com/assets/v1/${op.path}`, { headers }).catch(() => null);
	if (poll?.ok) op = await poll.json(); // réseau, 429 ou 5xx : on retente au tour suivant
}
if (!op.done) fail(1, { pending: op.path });
if (op.error || !op.response?.assetId) fail(1, { error: op.error ?? op });
console.log(JSON.stringify({ ok: true, assetId: op.response.assetId, file: basename(file) }));
```

- Codes de sortie : 0 réussi, 1 refus de l'API ou opération inachevée, 2 erreur d'usage. Lance d'abord `--dry-run` pour montrer à l'utilisateur ce qui partira.
- Le POST n'est jamais rejoué : un upload rejoué crée un second asset. Après un échec ambigu, regarde dans le Creator Dashboard avant de relancer.

## 8. Publier une place

```bash
printf 'header = "x-api-key: %s"\n' "$ROBLOX_API_KEY" | curl --config - -sS -X POST \
  "https://apis.roblox.com/universes/v1/$UNIVERSE_ID/places/$PLACE_ID/versions?versionType=Saved" \
  -H "Content-Type: application/octet-stream" --data-binary @build.rbxl
```

- `printf` est intégré au shell : la clé arrive à curl par l'entrée standard, pas par sa ligne de commande. Sous Windows, lance ce bloc dans Git Bash, ou écris un petit script Node sur le modèle de la section 7.
- Corps : les octets du fichier, en `application/octet-stream` pour un `.rbxl` ou `application/xml` pour un `.rbxlx`. Portée `universe-places` en écriture. Réponse : `{"versionNumber": 145}`. L'endpoint est marqué beta, limité à 30 appels par minute par propriétaire de clé, et non appelable depuis le jeu (openapi.json, 2026-10-18).
- `versionType=Saved` stocke une version invisible des joueurs : c'est le staging sûr. `Published` met en ligne : **oui explicite de l'utilisateur à chaque fois**.
- **409 « Server is busy and unable to process your upload request »** : Studio a la place ouverte (surtout en édition collaborative) ou une autre publication est en cours. Demande de fermer la place dans Studio, puis réessaie deux ou trois fois, espacé (5, 10, 15 s). Ne boucle jamais. La doc donne aussi 409 pour « Place not part of the universe » : vérifie d'abord la paire d'ids.
- Lis et cite `versionNumber` : un upload rejoué peut sauter un numéro (8 puis 10). Vérifie la version en ligne juste avant de publier.
- Cette API ne met pas à jour `EditableImage`, `EditableMesh`, `PartOperation` (unions), `SurfaceAppearance` ni `BaseWrap` : publie ces changements depuis Studio.
- Publier par-dessus des éditions faites seulement dans Studio les efface : fais d'abord sauver la place et récupère ces éditions.
- Les serveurs ouverts gardent l'ancienne version. Pour basculer tout le monde : « Restart servers » du Creator Hub ou `POST /cloud/v2/universes/{universe_id}:restartServers` (`RestartUniverseServers`, portée `universe:write`, qui par défaut ne redémarre que les serveurs d'une ancienne version), avec l'accord de l'utilisateur.
- Erreurs typiques : « Scope not authorized » (portée `universe-places` absente pour cette expérience) ; 401 ou 403 (clé, expiration, IP, publication interdite sur cette place) ; 404 (place ou univers inexistant) ; 409 (place hors de cet univers) ; 413 (fichier trop gros : openapi.json donne `x-roblox-size-limit` de 10 Mio, 2026-10-18) ; aucune réponse (réseau, proxy, TLS ; sous Windows PowerShell 5.1, active TLS 1.2).
- `rojo upload --api_key … --universe_id … --asset_id …` publie aussi, mais met la clé dans la ligne de commande : évite-le.
- En CI, un seul chemin automatisé écrit la production (par exemple sur un tag `v*`) ; l'intégration va sur une place de test.

## 9. Luau Execution : tests sur les serveurs Roblox

API stable. Elle exécute un script Luau sur un vrai serveur de jeu, avec l'API moteur complète, contre une version de place.

- Création : `POST https://apis.roblox.com/cloud/v2/universes/{u}/places/{p}/luau-execution-session-tasks` (ou `…/places/{p}/versions/{v}/luau-execution-session-tasks`), corps `{"script": "…", "timeout": "300s"}`. Portée `universe.place.luau-execution-session` en lecture et écriture.
- La réponse donne un `path` (`universes/…/luau-execution-sessions/…/tasks/…`). Sonde `GET https://apis.roblox.com/cloud/v2/{path}` : états `QUEUED`, `PROCESSING`, `COMPLETE`, `FAILED`, `CANCELLED`. Les valeurs renvoyées par le script arrivent dans `output.results` (`return 42, "ok"` donne `[42, "ok"]` ; une Instance, non sérialisable en JSON, donne `null`).
- Logs : `GET https://apis.roblox.com/cloud/v2/{path}/logs`, avec `print` et `warn`. Un `error()` donne `FAILED`, `error.code = "SCRIPT_ERROR"` et la pile dans les logs ; un dépassement de temps donne `DEADLINE_EXCEEDED`.
- Limites (doc, 2026-10-18) : 5 minutes d'exécution ; script et résultats de 4 Mo chacun ; logs gardés jusqu'à 450 Ko ; 10 tâches simultanées par place ; **5 créations de tâche par minute** par propriétaire de clé ; 200 lectures et 45 lectures de logs par minute ; données gardées 24 h. Mesure du 2026-10-11 : 40 créations en 10 minutes, toutes acceptées. Un ancien exemple officiel de CI parle de deux exécutions concurrentes par univers : mets un groupe de concurrence en CI.
- Les changements du DataModel restent locaux à la tâche et sont perdus, mais la doc ajoute qu'`AssetService:SavePlaceAsync` peut les sauver dans la place (doc, 2026-10-18) : jamais sans le oui de l'utilisateur. La physique ne tourne pas et les scripts de la place ne se lancent pas seuls. En revanche, les écritures DataStore du script sont réelles. Traite cette API comme un accès privilégié, sur une place de test seulement.
- **Isolation des données** : les places d'une même expérience partagent ses DataStores. Une place de test suffit pour des tests qui n'écrivent rien ; dès qu'un test écrit des données, prends une expérience de test séparée.

**Jest Roblox en CI**, la pratique courante :

1. Construis une place de test avec les tests, un module `jest.config` à la racine des tests et Jest Roblox : `rojo build test.project.json -o test.rbxl`.
2. Publie-la en version `Saved` de la place de test (section 8) et note `versionNumber`.
3. Lance le script ci-dessous contre cette version, puis lis les compteurs dans `output.results`.
4. Une tâche terminée avec 0 test est un échec de configuration, pas un succès.

```luau
-- tests/run-jest.luau : script envoyé à la tâche Luau Execution
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Jest = require(ReplicatedStorage.DevPackages.Jest) -- chemin à adapter au projet
local root = ReplicatedStorage.Shared

local status, result = Jest.runCLI(root, { verbose = false, ci = true }, { root }):awaitStatus()
if status ~= "Resolved" then
	error(`Jest {status} : {tostring(result)}`)
end
local r = result.results
if r.numTotalTests == 0 then
	error("aucun test trouvé : configuration à corriger")
end
if r.numFailedTests > 0 or r.numFailedTestSuites > 0 then
	error(`{r.numFailedTests} test(s) en échec, {r.numFailedTestSuites} suite(s) en échec`)
end
return { passed = r.numPassedTests, total = r.numTotalTests }
```

- L'erreur levée fait passer la tâche en `FAILED` : la CI échoue sur le bon signal.
- Jest Lua demande `debug.loadmodule` (drapeau `FFlagEnableLoadModule` dans Studio, d'après son guide) et peut demander `ServerScriptService.LoadStringEnabled` dans la place de test. Ce réglage n'est pas scriptable : pose-le dans le projet de build (`$properties` de ServerScriptService) ou dans Studio (à vérifier).
- Des lanceurs tiers existent (lest, jest-roblox-cli, rbxcloud, actions GitHub de Luau Execution ; voir outils.md) : l'utilisateur décide s'il les installe.
- Preuve à rapporter : la commande lancée (sans la clé), l'état final, les compteurs et les lignes de log utiles.

## 10. DataStores, MemoryStores, messages

- Base : `https://apis.roblox.com/cloud/v2/universes/{universe_id}/data-stores/…`. Opérations : lister les stores ; lister, créer, lire, mettre à jour, incrémenter et supprimer des entrées ; lister les révisions ; les mêmes sur les stores ordonnés ; `SnapshotDataStores` avant une publication risquée (portée `universe-datastores.control:snapshot`, un par jour UTC : les données du moment restent récupérables en version au moins 30 jours).
- Lister les clés : `GET …/data-stores/{data_store_id}/entries` (`universe-datastores.objects:list`) renvoie seulement `path` et `id`. `maxPageSize` va jusqu'à 256 (10 par défaut), et le filtre `id.startsWith(...)` parcourt un préfixe. Lire : `GET …/entries/{entry_id}` (`universe-datastores.objects:read`). Suis les chemins renvoyés : un id d'entrée n'est pas forcément la clé brute, et ce qui suit son dernier `@` est lu comme une révision (ajoute `@latest` à un id qui contient `@`).
- **Écrire une entrée** (`PATCH …/entries/{entry_id}`) la remplace en entier : un `users` ou des `attributes` omis sont effacés. Relis l'entrée, renvoie `value`, `users` (format `users/{id}`) et `attributes`, et passe son `etag` pour que l'écriture échoue si l'entrée a changé entre-temps (openapi.json, 2026-10-18).
- **Budget partagé** : depuis le 2026-07-29, le jeu et Open Cloud consomment un seul budget DataStore par expérience. Un gros job externe peut affamer les serveurs : lance-le hors des heures de pointe, limite toi-même son débit, et préviens l'utilisateur. Les formules de quota sont dans donnees.md.
- **Écrire la clé d'un joueur connecté est une perte garantie** : la sauvegarde suivante du serveur qui tient sa session l'écrase. Passe par ce serveur, ou attends la libération du verrou de session, bloque sa reprise pendant la réparation, applique, puis laisse le serveur suivant charger (donnees.md).
- Un store supprimé par `DeleteDataStore` compte encore 30 jours dans le quota de stockage ; `UndeleteDataStore` l'annule pendant ce délai.
- Effacement RGPD : modèles `user_data_templates` par l'API Configs (brouillon en `PUT`, vérification en `GET`, puis `POST` de publication ; portées `universe:read` et `universe:write`), ou webhook « Right to Erasure Request » (section 12).
- MemoryStores : sorted maps, queues, `FlushMemoryStore` (`universe.memory-store:flush`). Un flush efface l'état vivant de tous les serveurs : confirmation obligatoire. Son opération renvoie un `path` qui contient déjà `cloud/v2` : sonde `https://apis.roblox.com/{path}` sans rien ajouter.
- **Messages aux serveurs** : `POST https://apis.roblox.com/cloud/v2/universes/{universe_id}:publishMessage` avec `{"topic": "…", "message": "…"}`, portée `universe-messaging-service:publish`, stable ; l'ancienne forme v1, `POST /messaging-service/v1/universes/{universeId}/topics/{topic}`, est en beta et renvoie vers la v2 (openapi.json, 2026-10-18). Les serveurs reçoivent par `MessagingService:SubscribeAsync`. Quotas partagés avec le jeu, 1 024 caractères par message et 80 par topic (doc, 2026-10-18), livraison non garantie : jamais pour un octroi ni de l'argent.
- Avant un test destructif (effacement, rejeu de migration, écriture en masse), confirme les vraies destinations : noms exacts des stores, topics, URL. Une place nommée « test » n'isole rien si elle partage les stores de la production.

## 11. HttpService vers Open Cloud depuis le jeu

Un endpoint n'est appelable depuis le jeu que si la doc le dit (`x-roblox-engine-usability.apiKeyWithHttpService`). Exemple : donner un rôle de groupe à un joueur par `POST https://apis.roblox.com/cloud/v2/groups/{groupId}/memberships/{membershipId}:assignRole`, corps `{ role = "groups/{groupId}/roles/{roleId}" }`, portée `group:write` ; `:unassignRole` retire un rôle. L'ancien `PATCH …/memberships/{membershipId}` (`UpdateGroupMembership`) est déprécié (openapi.json, 2026-10-18). Le `membershipId` vient de `GET …/groups/{groupId}/memberships?filter=user == 'users/{userId}'` (filtre encodé dans l'URL).

- **Prérequis** : « Allow HTTP Requests » activé dans les réglages de sécurité de l'expérience ; code serveur seulement ; clé dans le Secrets Store de l'expérience (Creator Dashboard, onglet Secrets du jeu), domaine autorisé restreint à `apis.roblox.com`.
- `HttpService:GetSecret("nom")` renvoie un objet `Secret`, jamais imprimable (`print` affiche `Secret(nom)`). La valeur de `x-api-key` **doit** être ce `Secret`, pas une chaîne. `Secret:AddPrefix()` et `AddSuffix()` construisent un en-tête sans concaténation. Un secret va dans l'URL ou un en-tête, jamais dans le corps.
- Les secrets ne marchent qu'en serveur live et en test collaboratif. En Play Solo ou côté client : « Can't find secret with given key ». Pour tester en local, définis des Local Secrets (File, Experience Settings, Security). 500 secrets par expérience, 1 024 caractères chacun.
- Seuls les en-têtes `x-api-key` et `content-type` passent, en minuscules (HTTP/2). Aucun en-tête d'idempotence possible : l'idempotence doit venir de la ressource elle-même.
- Interdits : la séquence `..` dans un paramètre de chemin (une clé de store qui la contient est inaccessible depuis le jeu), le HTTP non chiffré, les ports sous 1024 sauf 80 et 443, le port 1194.
- **Débit** : la référence `HttpService` annonce 2 500 requêtes Open Cloud par minute et par serveur, à part des 500 requêtes HTTP générales ; la page « Rate limits » d'Open Cloud dit qu'elles comptent dans les 500 (doc, 2026-10-18). Tant que ce n'est pas tranché, vise 500 par minute en tout. Au-delà, les appels sont bloqués environ 30 s (« Number of Open Cloud requests exceeded limit ») : une rafale coûte plus cher qu'une file. Dans Studio, ce budget est partagé par toute la session, plugins compris ; un plugin qui le dépasse reçoit « Number of requests exceeded limit » avec un conseil trompeur d'activer Allow HTTP Requests.
- Lance en parallèle les appels indépendants : le moteur en regroupe. Le tableau Observability du Creator Hub (Monitoring) montre volume et temps de réponse.
- Ne fais jamais passer un appel par un client, ni par un proxy tiers (type roproxy) avec une clé, un cookie ou un jeton.

```luau
--!strict
-- ServerScriptService/OpenCloud.luau : appels Open Cloud depuis un serveur de jeu
local HttpService = game:GetService("HttpService")

local BASE = "https://apis.roblox.com/cloud/v2/"
local TRANSIENT = { [408] = true, [500] = true, [502] = true, [503] = true, [504] = true }
local MAX_ATTEMPTS = 4

type HttpMethod = "GET" | "POST" | "PUT" | "PATCH" | "DELETE"

local OpenCloud = {}

-- idempotent = true seulement si rejouer l'appel mène au même état final
function OpenCloud.request(method: HttpMethod, path: string, body: any?, idempotent: boolean?): (boolean, any)
	if string.find(path, "..", 1, true) then
		return false, "chemin interdit : '..'"
	end
	local found, secret = pcall(HttpService.GetSecret, HttpService, "open_cloud_key")
	if not found then
		return false, "secret introuvable (Play Solo ou client ?)"
	end
	local canRetry = method == "GET" or idempotent == true
	-- Annoté et passé par une closure : un littéral donné à pcall(HttpService.RequestAsync, …) ne passe pas le typage strict.
	local requestHeaders: { [string]: string | Secret } = { ["x-api-key"] = secret, ["content-type"] = "application/json" }
	local lastError = "aucune tentative"
	for attempt = 1, MAX_ATTEMPTS do
		local ok, response = pcall(function()
			return HttpService:RequestAsync({
				Url = BASE .. path,
				Method = method,
				Headers = requestHeaders,
				Body = if body ~= nil then HttpService:JSONEncode(body) else nil,
			})
		end)
		if ok and response.Success then
			local raw = response.Body
			if raw == nil or raw == "" then
				return true, nil
			end
			local decoded, data = pcall(HttpService.JSONDecode, HttpService, raw)
			return true, if decoded then data else raw -- corps non JSON : rendu brut
		end
		local status = if ok then response.StatusCode else 0
		lastError = if ok then `HTTP {status}` else tostring(response)
		-- 429 : refusé sans traitement, rejouable ; 5xx ou transport : issue inconnue pour une écriture
		local retryable = status == 429 or (canRetry and (status == 0 or TRANSIENT[status] == true))
		if not retryable or attempt == MAX_ATTEMPTS then
			break
		end
		local headers: { [string]: string } = if ok then response.Headers else {}
		local hinted = tonumber(headers["retry-after"]) or tonumber(headers["x-ratelimit-reset"])
		task.wait(hinted or math.min(30, 0.5 * 2 ^ (attempt - 1)) * math.random()) -- jitter complet
	end
	return false, lastError
end

return OpenCloud
```

- Ce module vit côté serveur, jamais dans ReplicatedStorage. Valide les arguments venus d'un client avant d'appeler (securite.md).

## 12. Webhooks

- Déclencheurs : abonnements (`purchased`, `renewed`, `resubscribed`, `cancelled`, `refunded`) ; « Transaction Refunded » (developer products, game passes, serveurs privés, accès payant ; pas les abonnements ; récent et expérimental) ; « Right to Erasure Request » (conformité) ; commandes commerce payées ou remboursées (un « Paid » peut arriver en double : dédoublonne par id de commande) ; « Early Harm Detection » (doc, 2026-10-18). Roblox retire seul un pass ou un objet remboursé, mais l'avantage d'un developer product se retire de **tes** données.
- Configuration dans le Creator Dashboard, avec un secret. L'URL cible est publique, en HTTPS, accepte POST et répond 2xx en moins de 5 s. Discord et Slack sont pris en charge, ou ton propre endpoint.
- Charge utile : `NotificationId`, `EventType`, `EventTime`, `EventPayload` (par exemple `UserId` et `GameIds` pour un effacement).
- **Signature** : en-tête `roblox-signature` de la forme `t=<horodatage>,v1=<signature>`. Recalcule un HMAC-SHA256 avec le secret sur `<horodatage>.<corps brut>`, encodé en base64, compare en temps constant, rejette un horodatage trop vieux (la doc juge raisonnable une fenêtre de 10 minutes ; son exemple prend 300 s) (doc, 2026-10-18). Sans secret configuré, l'en-tête ne porte que l'horodatage.

```js
// Vérification d'un webhook Roblox (Node.js). rawBody : Buffer du corps exact, avant JSON.parse.
import { createHmac, timingSafeEqual } from "node:crypto";

export function verifyRobloxSignature(header, rawBody, secret, maxAgeSec = 300) {
	const fields = {};
	for (const part of (header ?? "").split(",")) {
		const i = part.indexOf("="); // la signature base64 peut finir par des « = »
		if (i > 0) fields[part.slice(0, i).trim()] = part.slice(i + 1).trim();
	}
	const t = Number(fields.t);
	if (!fields.v1 || !Number.isFinite(t) || Math.abs(Date.now() / 1000 - t) > maxAgeSec) return false;
	const expected = createHmac("sha256", secret).update(`${fields.t}.`).update(rawBody).digest("base64");
	const a = Buffer.from(expected);
	const b = Buffer.from(fields.v1);
	return a.length === b.length && timingSafeEqual(a, b);
}
```

- Livraison « au moins une fois », parfois en retard ou en double ; après 5 échecs, Roblox arrête et tient l'URL pour invalide (doc, 2026-10-18). Ordre sûr : vérifier la signature ; écrire durablement le travail indexé par `NotificationId`, ce qui sert aussi de dédoublonnage ; répondre 2xx ; traiter ensuite dans un worker idempotent. Un set en mémoire ne suffit pas pour un octroi ou une suppression.
- Teste avec le bouton « Test Response » du webhook dans le Creator Dashboard (événement `SampleNotification`) avant d'activer la production.

## 13. Pièges

- Appeler Open Cloud pour ce que le moteur fait déjà, ou l'inverse : les deux index sont séparés.
- Une clé dans le code, la place, une URL, un log, un commit, une capture ou le chat.
- Une clé sans restriction d'expérience, ou une clé personnelle pour un jeu de groupe.
- Oublier l'auto-expiration à 60 jours d'une clé inutilisée.
- Rejouer un incrément, une création ou un message après un 5xx sans relire la ressource.
- Publier en `Published` sans oui explicite, ou par-dessus des éditions Studio non sauvées.
- Boucler sur un 409 au lieu de faire fermer la place dans Studio.
- Croire que la publication Open Cloud transporte unions, `SurfaceAppearance` ou `EditableMesh`.
- Cuire un id de Decal là où il faut l'id d'une Image, ou un asset id inventé.
- Lancer un job DataStore de masse aux heures de pointe, ou écrire la clé d'un joueur connecté.
- Réécrire une entrée DataStore par Open Cloud sans renvoyer `users` et `attributes` (effacés) ni son `etag`.
- Tester Luau Execution ou des écritures de données dans l'expérience de production.
- Passer la clé en chaîne dans `x-api-key` depuis le jeu, ou mettre `..` dans un chemin.
- Traiter un webhook avant d'avoir vérifié sa signature, ou dédoublonner en mémoire.

## 14. Checklist

- [ ] Le bon système est choisi (moteur ou Open Cloud), et l'endpoint vient de `llms.txt` ou d'`openapi.json`.
- [ ] La clé est créée par l'utilisateur, limitée à une expérience et aux opérations utiles, et lue dans une variable d'environnement ou le Secrets Store.
- [ ] Aucune clé ni aucun jeton dans le dépôt, la place, les logs ou le chat.
- [ ] Chaque écriture a été confirmée par l'utilisateur après un aperçu à blanc.
- [ ] Les retries respectent `retry-after`, sont bornés, et ne rejouent pas une écriture à l'issue inconnue.
- [ ] Chaque upload renvoie un asset id vérifié dans Studio et rangé dans le module de manifeste.
- [ ] La publication est en `Saved` sauf oui explicite, et `versionNumber` est cité.
- [ ] Les tests Luau Execution tournent sur une place ou une expérience de test, et 0 test compte comme un échec.
- [ ] Les jobs DataStore externes sont limités en débit et ne touchent pas une session ouverte.
- [ ] Les appels depuis le jeu passent par un `Secret`, côté serveur, vers des endpoints autorisés en jeu.
- [ ] Les webhooks vérifient la signature, dédoublonnent durablement, puis répondent vite.
