---
title: "Trouver, Craquer, Gagner"
description: "Comment le CTF GeoGuessr d'une école cyber leakait les coordonnées GPS exactes via l'API Mapillary."
date: 2026-07-01 09:00:00 +0200
categories: [Web]
tags: [lehack, Web]
---

#### Comment niquer le CTF d'une école "cyber"


C'est le deuxième jour de LeHack 2026. En faisant le tour des stands, je tombe sur le stand de CyberSup, une école privée 
proposant un mastère en cybersécurité. Après un échange avec leur équipe, visiblement intéressée par mon profil, je 
remarque qu'ils proposent un challenge : un jeu de type GeoGuessr sur lequel les participants doivent marquer un maximum 
de points. Le lot à gagner ? Un HackRF One R10C, un outil radio à ~250 € que les passionnés de SDR connaissent bien.
Je tente ma chance quelques fois en répondant approximativement, et j'obtiens entre 5K et 15K points. Les meilleurs 
joueurs tournent autour de 25K, clairement hors de portée en jouant au hasard. Le jeu affiche explicitement un score 
maximum de 4 000 points par question — j'en ai d'ailleurs obtenu 3 983 lors d'un essai précis, ce qui confirme cette borne. 
Autant dire que pour gagner, il faut être très précis.
Étant dans un salon de cybersécurité, inspecter le fonctionnement du site avant de répondre m'a semblé être la démarche naturelle.

Points par réponse :

- Réponse exacte → 4 000 points
- Réponse incorrecte → 100 points

Plus on est proche de la bonne réponse, plus le score est élevé.

 ![](/assets/img/lehack/la_traque.png)

### Exploit

Si on lance le défi une première fois en regardant le réseau, on peut observer plusieurs
requêtes sur une API. Quand on regarde comment ça fonctionne, on voit globalement 4 requêtes importantes.

**Requêtes**:

- /new-case
- /images (API Mapillary)
- /sprint-submit
- un autre endpoint pour l'inscription finale

#### 1~Endpoint `/new-case` 
Une requête `POST` vers `https://crime.cybersup.ai/api/new-case?difficulty=normal` permet
d'initialiser une « case », soit un défi. Cette requête nous permet de récupérer le défi ainsi que 
deux informations intéressantes pour nous qui sont l'`id` et le `mapillary_image_id`. Ces deux 
informations n'ont pas de réel intérêt pour l'instant, mais seront utiles pour la suite.

#### 2~Endpoint `/images`
Une requête `GET` vers `https://graph.mapillary.com/images?image_ids=<id>&fields=<paramètres>` 
avec un token `Authorization: OAuth MLY|26614486581534345|cd2e0de2[...]1f60ae122a92ba982` 

L'ID est celui qu'on a récupéré avec `mapillary_image_id`. Le paramètre `fields` sert simplement à 
nous indiquer les informations que nous souhaitons récupérer. Dans mon cas, je demande
seulement l'id ainsi que les coordonnées. Je reçois donc les coordonnées exactes de 
l'endroit où l'on est.

Voici tout de même les différents paramètres qui sont utilisés dans les requêtes de base. 


| Paramètre | Utilité |
|----|----|
| `id` | Identifiant unique de l'image sur Mapillary |
| `geometry` | Position GPS d'origine |
| `computed_geometry` | Position GPS recalculée après traitement SfM |
| `sequence` | ID de la séquence d'images capturées à la suite |
| `altitude` | Altitude d'origine |
| `atomic_scale` | Échelle de la reconstruction SfM autour de l'image |
| `camera_parameters` | Paramètres intrinsèques de la caméra |
| `camera_type` | Type de projection de la caméra |
| `captured_at` | Timestamp de capture de l'image |
| `compass_angle` | Angle de boussole d'origine |
| `computed_altitude` | Altitude recalculée après traitement d'image |
| `computed_compass_angle` | Angle de boussole recalculé après traitement d'image |
| `computed_rotation` | Orientation corrigée de l'image |
| `creator` | Nom d'utilisateur et ID de la personne ayant uploadé l'image |
| `exif_orientation` | Orientation de la caméra selon le tag EXIF |
| `height` | Hauteur de l'image originale uploadée |
| `merge_cc` | ID de la composante connexe d'images alignées ensemble lors du SfM |
| `mesh` | Objet `{id, url}` pointant vers le mesh 3D reconstruit |
| `organization` | Organisation à laquelle l'image est rattachée |
| `quality_score` | Score de qualité visuelle prédit, entre 0.0 et 1.0 |
| `sfm_cluster` | Objet `{id, url}` pointant vers le nuage de points JSON compressé en zlib |
| `thumb_1024_url` | URL de la miniature 1024px |
| `thumb_2048_url` | URL de la miniature 2048px |
| `width` | Largeur de l'image originale uploadée |


#### 3~Endpoint `/sprint-submit`
Une requête `POST` vers `https://crime.cybersup.ai/api/competitions/by-slug/lehack-2026/sprint-submit` 
avec un payload.

**Payload**
Payload tel qu'observé brut dans l'export .har Firefox

```json
{
    "participant_id":"bcd6675db45b",
    "case_id":"fdc6b8bd71",
    "lat":19.078727224576564,
    "lng":11.601562500000002,
    "elapsed_seconds":10.074
}
```

Suite à cette requête, nous recevons une réponse nous indiquant le nombre de points obtenus grâce à 
notre réponse, ainsi que le nombre de challenges remplis depuis le début.

**Réponse**

```json
{
    "score":100,
    "total_score":100,
    "cases_completed":1
}
```
À partir de là, nous avons donc répondu à un défi en 10s/180s, nous pouvons donc retenter notre chance et 
espérer viser un peu mieux pour éviter de gagner seulement 100 points.

L'idée serait donc de faire des requêtes à l'API en espérant pouvoir récupérer les infos qui nous intéressent. Coup de bol (ou pas), la sécurité n'est pas top par ici. 
L'API est protégée par un token qui est récupérable avec un simple `GET` sur l'endpoint `https://crime.cybersup.ai/api/mapillary-token`. Il suffirait donc de le récupérer pour ensuite le réutiliser 
et voir jusqu'où je peux aller (spoiler : on va loin).

Voici un exemple d'exploitation pour une question :

```python
import requests
import uuid

URL = "https://crime.cybersup.ai/api"
EDITION = "lehack-2026"

session_token = str(uuid.uuid4())
print("session_token (uuid v4) =", session_token)


competition = requests.get(f"{URL}/competitions/by-slug/{EDITION}").json()
competition_id = competition["id"]
print("id competition =", competition_id, "| status =", competition["status"])

r = requests.post(f"{URL}/competitions/{competition_id}/join", json={"first_name": "Test", "last_name": "Agent", "session_token": session_token},)
print(f"response body with compétition id = {r.text}")

data = r.json()
participant_id = data.get("participant_id")
print("id participant =", participant_id)


r = requests.get("https://crime.cybersup.ai/api/mapillary-token", headers={"Accept": "application/json"})
token_MLY =  r.json()['token']
print(f"Mapillary token = {token_MLY}")

x = requests.post("https://crime.cybersup.ai/api/new-case?difficulty=normal")
y = x.json()
imageID = y["mapillary_image_id"]
print("Image id = {id}".format(id=imageID))
case_id = y["id"]

x = requests.get("https://graph.mapillary.com/images?image_ids={id}&fields=id,computed_geometry".format(id=imageID), headers = {"Authorization": "OAuth {token}".format(token=token_MLY)})
data = x.json()
obj = data['data'][0]
coord = obj["computed_geometry"]["coordinates"]
print(f"GPS coordinates {coord}")
lng, lat = coord[0], coord[1]

x = requests.post("https://crime.cybersup.ai/api/competitions/by-slug/lehack-2026/sprint-submit", json={"participant_id":participant_id,"case_id":case_id,"lat":lat,"lng":lng,"elapsed_seconds":10.074})
print(f"Result of submit request {x.text}")
```
Exemple de résultat avec un participant_id en dur :

```
belly $ python poc.py 
session_token (uuid v4) = e52cac86-109c-46f3-9ebd-33b02c788c87
id competition = 0909cd7a433d | status = ended
response body with compétition id = {"detail":"La compétition est terminée"}
id participant = None
Mapillary token = MLY|26906152832407799|364da285d6fde0a[...]84fc99453a7i
Image id = 1153346652203167
GPS coordinates [-17.459719444444, 14.706630555556]
Result of submit request {"detail":"L'événement est terminé"}
```

En récupérant les coordonnées exactes via computed_geometry avant de répondre, il devient possible 
d'atteindre systématiquement (ou de s'approcher significativement de) ce score maximal de 4000 points, 
au lieu de dépendre de la précision visuelle du joueur.

L'événement étant terminé, l'endpoint /sprint-submit retourne systématiquement une erreur liée au statut 
de la compétition, quel que soit le participant_id fourni. Avec un participant_id à None, la 
requête échoue déjà au niveau de la validation de type côté backend. 
La réutilisation d'un participant_id issu d'une session antérieure dans ce PoC permet d'aller 
légèrement plus loin et d'avoir un message d'erreur indiquant la fin de l'événement.

À la fin des 3 minutes autorisées pour ce défi, nous devons donner notre nom, prénom et e-mail pour être 
inscrits dans le leaderboard. Je n'ai pas pu récupérer la requête pour reproduire cela, mais au vu 
de la facilité du reste je ne m'inquiète pas.

>**Note**:
>Le problème a été remonté physiquement à l'école en question. Ils sont donc conscients du problème et feront sûrement des changements à l'avenir.
