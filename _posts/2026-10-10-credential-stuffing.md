---
title: "Credential Stuffing"
description: "Comment font les hackers pour hacker vos comptes ?"
date: 2026-10-09 09:00:00 +0200
categories: [Web]
tags: [credential-stuffing]
---

# Credential Stuffing
### Comment font les hackers pour hacker vos comptes ?

La première chose à faire, c'est de ne pas associer le credential stuffing au brute force, car ce n'est pas la même chose même si on y retrouve certains patterns. Le brute force, c'est le fait de chercher un mot de passe en partant de rien : on va créer une multitude de possibilités et les essayer en espérant que ça fonctionne. Le credential stuffing, c'est le fait d'essayer des mots de passe déjà utilisés par la victime, récupérés dans des fuites de données par exemple. Le vecteur est donc bien moins aléatoire, et contourne certaines protections mises en place au fil du temps, on peut par exemple citer le rate limiting et la complexité des mots de passe.

### Anatomie d'une attaque

Pour mettre au point ce genre d'attaque, les hackers ont besoin de plusieurs ressources, plus ou moins faciles à obtenir et de qualité plus ou moins bonne. Nous allons donc voir les différents points et comment ils s'en servent pour mener une attaque à bien.

#### D'où viennent les données

Les données se présentent sous deux formes : des logs ou des combolists.

Les logs sont le plus souvent récupérés directement depuis le PC des victimes. Vous avez sûrement déjà vu des vidéos YouTube, des annonces ou des articles proposant d'avoir la toute dernière version de Photoshop ou d'un autre logiciel gratuitement : il suffit de télécharger un fichier et de l'installer. En réalité, ce type de logiciel a souvent été craqué puis modifié avec du code malveillant, dans le but d'installer un stealer sur un maximum de PC et de récupérer les identifiants stockés sur la machine de la victime. Les attaquants reçoivent ensuite ces données sur Discord (via webhook), Telegram (via bot) ou d'autres méthodes, sous forme de fichiers texte au format `url:mail:pass`, et parfois, plus rarement, avec des tokens de session. L'intérêt pour l'attaquant est là : contrairement à une combolist classique, un log permet de savoir directement sur quel site tel identifiant a été utilisé, sans avoir à deviner ou à tester à l'aveugle.

Les combolists se construisent de deux façons, à ma connaissance : soit directement depuis des bases de données, soit en se basant sur des logs et en filtrant sur une caractéristique précise (un domaine, un site, etc.).

Des outils ont été développés par des personnes du milieu dans ce but précis. L'avantage de cette seconde méthode, c'est qu'elle permet de cibler et de trier les credentials. Par exemple, si quelqu'un souhaite récupérer des creds pour le site `toto1`, il lui suffit de faire un tri basé sur les URLs présentes dans les logs. Il est aussi possible de trier par groupe plutôt que par site unique. Si on prend l'exemple des fast-foods : il est assez probable qu'une personne ayant un compte chez McDonald's en ait aussi un chez Burger King (ou équivalent). La logique est là : regrouper par thématique selon le besoin, car en plus d'avoir une chance de tomber sur un compte valide sur un site similaire, il est probable que les mêmes identifiants y soient réutilisés.

Sur les forums, vous pouvez trouver des fichiers du type `300K mail Orange UHQ` : il s'agit simplement de logs triés par domaine (ici Orange), que la personne a ensuite mis à disposition.

La deuxième méthode consiste à récupérer directement des bases de données. Il est possible de scanner Internet à l'aide de listes de mots-clés pour trouver des bases de données SQL vulnérables, et d'en récupérer les combos `user:pass` afin de construire des combolists. Mais là encore, tout dépend de l'objectif recherché : cette méthode peut s'avérer peu intéressante, car très sélective et pas forcément en lien avec ce que l'attaquant avait prévu de faire.

Voilà donc le format des données que l'on peut retrouver sur les forums et canaux Telegram. L'accès est souvent gratuit pour un échantillon, mais devient payant pour la suite, avec des abonnements annuels, mensuels ou journaliers selon la qualité et la quantité de données récupérées. Ces abonnements donnent généralement accès à un canal Telegram privé diffusant des logs dits "exclusifs" ou "fresh" (c'est-à-dire jamais testés par d'autres). Voici un exemple de grille tarifaire que l'on peut trouver pour ce genre d'accès :

- 1 semaine — 100$
- 1 mois — 250$
- 3 mois — 600$
- À vie — 1200$

#### Les outils d'attaque

Plusieurs outils ont été développés dans le but de mettre en œuvre ce type d'attaque. Parmi eux, on trouve SilverBullet (propriétaire) et OpenBullet (open source) : ces deux outils permettent de mener ce genre d'attaques, avec comme ressources minimales une combolist et une config.

Je viens d'expliquer comment sont fabriquées les combolists ; voyons maintenant comment les attaquants les exploitent. Pour cela, ils ont besoin d'une config : un script écrit en LoliScript, ou assemblé sous forme de blocs, qui va interroger un site, une API, etc., dans le but de simuler une connexion. Lors de la création d'une config, il est possible d'envoyer des requêtes de test et d'en observer le résultat dans un panneau de logs dédié, situé en bas à droite de l'interface.

Ces outils proposent de nombreux blocs permettant, avec une grande facilité, de parser du contenu, de cibler des champs en particulier, de poser des conditions, etc. Et si le bloc qu'il vous faut n'existe pas, il reste possible d'en coder un soi-même en LoliScript.

La config utilise comme données celles présentes dans la combolist, en les prenant ligne par ligne et en associant l'identifiant au mot de passe. Dès lors, il devient possible d'automatiser l'attaque grâce à des combolists de plusieurs milliers, voire millions de lignes.

Un marché existe aussi autour des configs elles-mêmes, car c'est la ressource la plus difficile à obtenir ou à produire, ce qui permet à certaines personnes de gagner une somme non négligeable en les vendant.

On peut aussi citer SilverBullet Pro, qui propose des configs chiffrées de façon à masquer leur contenu. Cela pose un problème si l'on souhaite comprendre comment une protection a été contournée, ou si l'on veut vérifier l'absence de code malveillant, car une config peut tout à fait exécuter des commandes shell ou des scripts.

##### Exemple pratique sur un site "cas d'école"

Ce site est connu dans le milieu pour être simple et bien adapté à l'apprentissage.

**Prérequis :**
- Un compte fonctionnel sur la plateforme

La première étape consiste à trouver la page de login, puisque c'est elle qui nous intéresse et dont on va vouloir automatiser les requêtes. L'outil le plus important à ce stade est l'onglet Réseau des DevTools du navigateur : c'est là qu'on va comprendre le fonctionnement du site et observer les requêtes effectuées.

La deuxième étape consiste à tenter de se connecter avec un compte qu'on sait faux, et à observer ce qui se passe côté réseau. Par exemple avec `test@proton.me:toto123` (format `user:pass`).

Avec ce faux compte, si on observe la requête envoyée à l'endpoint `/login`, on constate un code 401, avec pour réponse :
```json
{"message":"Identifiant ou mot de passe incorrect.","code":"bad-credentials","statusCode":401}
```
Si on regarde le payload envoyé, on y retrouve notre faux compte :
```json
{"username":"test@proton.me","password":"toto123","source":"Web"}
```
Dans les headers de la requête, on ne voit aucun token ni aucune autre donnée qui ne soit pas facile à se procurer.

On peut donc commencer à créer la config.

Il nous suffit de prendre un bloc `http request` et d'y renseigner les paramètres suivants.

Pour copier les données plus facilement, on peut faire un clic droit sur la requête → *Copy value* → *Copy as cURL*, puis coller le résultat dans l'outil [curlconverter](https://curlconverter.com/http/). On récupère ainsi la requête avec sa méthode, son content-type, ses headers (tous les champs ne sont pas forcément à recopier) et son payload.

Il ne reste plus qu'à ajouter, dans le debugger, un couple `user:pass` et à simuler la requête. En consultant les logs, on retrouve notre requête ainsi que la réponse du serveur :

```
Sliced test@proton.me:toto123 into:
DATA: test@proton.me:toto123

>> Http Request (HttpRequestStandard) <<
POST /authentication/login HTTP/1.1
Host: un-site.com
Connection: Close
Origin: https://un-site.com
Referer: https://un-site.com
User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:152.0)
Content-Type: application/json
Content-Length: 64

{"username":"input.USER","password":"input.PASS","source":"Web"}

Address: https://un-site.com/authentication/login
Response code: 401

Received Headers:
Date: Wed, 07 Oct 2026 09:43:30 GMT
Access-Control-Allow-Origin: https://un-site.com
[... autres headers CORS/techniques, non pertinents ici ...]
Content-Type: application/json; charset=utf-8
Content-Length: 81

Received Payload:
{"message":"Bad username or password.","code":"bad-credentials","statusCode":401}
```

![screen_requete_failed1.png]

À noter que notre bot obtient un statut `none` à ce stade.

On peut ensuite ajouter une condition avec le bloc `keycheck`, qui permet d'attribuer un statut à notre bot en fonction de conditions définies. On peut par exemple trier sur le code de réponse, sur des chaînes de caractères présentes dans la réponse, etc. L'objectif est de multiplier les signaux indiquant que la requête a échoué, afin d'éviter les faux positifs : si l'une des conditions indique un échec, le bot s'arrête et ne passe pas à l'étape suivante. Si on relance la simulation avec ce bloc de condition en place, on obtient le même résultat qu'avant, à la différence près que le bot passe désormais au statut `fail`.

Maintenant que le tri des échecs est en place, on peut refaire le même test avec notre propre compte, pour voir à quoi ressemble le résultat lorsqu'on obtient un compte valide. Dans le debugger, il suffit de remplacer le couple `user:pass` par nos propres identifiants.

En relançant le script, on observe un changement d'état : le bot passe en statut `BAN`. En réalité, cela signifie simplement qu'aucune de nos conditions ne prévoyait la réponse obtenue. Si on regarde cette réponse, on y trouve :
```json
{"accessToken":"UN_TOKEN","refreshToken":"UN_AUTRE_TOKEN"}
```
Il s'agit donc bien d'un compte valide. Il est maintenant nécessaire de mettre à jour nos conditions pour prévoir un statut de succès. Je n'irai pas plus loin dans cette démonstration, mais ce token sert ensuite à naviguer sur le site : il est généralement requis dans chaque requête, sous forme de token d'autorisation.

Les personnes qui pratiquent le credential stuffing ne s'arrêtent généralement pas là : elles vont plus loin pour récupérer des informations sur le compte, comme l'abonnement en cours, des données personnelles, un solde de cagnotte, etc.

À ce stade, on dispose donc d'une combolist ciblée, d'une config, et potentiellement de proxies. On peut alors lancer l'attaque sur le site visé.

#### Contourner les protections réseau

Une seule IP ne suffit généralement pas pour mener ce type d'attaque, à cause du rate limiting et des bans d'IP mis en place par les sites. Des proxies sont donc systématiquement utilisés pour faciliter l'accès et l'acceptation des requêtes sans se faire bannir.

Sur le marché, il existe principalement deux types de proxies : résidentiels et datacenter. En général, mieux vaut privilégier les proxies résidentiels, mais dans de rares cas, les proxies datacenter sont préférables. Les proxies résidentiels sont généralement perçus comme légitimes par les sites, ce qui n'est pas toujours le cas des proxies datacenter.

On parle aussi de rotation de proxies : un processus où l'adresse IP utilisée pour se connecter change automatiquement à intervalles réguliers, ou après chaque requête. Cela permet de rendre le suivi plus difficile pour les serveurs web, et c'est particulièrement utile en fonction du volume de tentatives prévu.

#### Automatisation et scalabilité

Cette méthode d'attaque se prête particulièrement bien à l'automatisation : on dispose d'un script (la config), de proxies, de données disponibles en libre accès sur Internet, et surtout de la possibilité d'envoyer des requêtes HTTP directes ou de passer par des navigateurs headless.

#### Le cas des applications mobiles

Le credential stuffing ne se limite pas aux sites web : il est aussi possible de cibler des applications mobiles, via un émulateur et des outils de debug comme ADB. Contrairement à un site web classique, une application mobile communique généralement avec son backend via une API (REST ou GraphQL), ce qui peut même simplifier l'automatisation de l'attaque. Pas besoin de gérer du rendu HTML, la réponse est directement structurée en JSON. On peut intercepter ces requêtes grâce à un outil comme HTTPToolkit, en installant simplement son certificat sur l'appareil ou l'émulateur.

Certaines applications se protègent contre ce genre de manipulation : si elles détectent la présence d'un débogueur ou un appareil rooté, elles peuvent par exemple rester bloquées en chargement indéfiniment. Dans ce cas, des outils comme Frida permettent de contourner ces protections en manipulant le comportement de l'application à l'exécution.

#### Débouchés et cas concrets

On peut par exemple citer le cas de HexDex, un hacker français d'une vingtaine d'années interpellé en avril 2026, soupçonné d'une longue série de cyberattaques : une quinzaine de fédérations sportives françaises, le Système d'information sur les armes (SIA), la plateforme e-campus, ou encore des organisations comme la CFDT et FO. Le credential stuffing aurait fait partie de ses méthodes, selon des éléments rapportés en dehors de la couverture presse initiale. Les données exfiltrées étaient ensuite republiées sur des plateformes comme BreachForums et DarkForum.

Ce qui nous amène à un autre cas d'usage : les "serveurs de gen" (génération de comptes), ou même de simples particuliers, qui tentent de gagner un peu d'argent en revendant des comptes de tout type. On retrouve aussi des usages liés au vol de contenu, que ce soit sur des sites pour adultes ou sur des comptes de réseaux sociaux (Snapchat, Instagram, OnlyFans, etc.), comme on a pu le voir récemment avec le démantèlement de la plateforme [NudeLeakTeens](https://www.24heures.ch/cybercriminalite-la-plateforme-de-photos-intimes-volees-a-17000-femmes-fermee-140616023603), qui revendait des contenus intimes issus de plus de 17 000 victimes dans le monde, provenant notamment de comptes Snapchat, Instagram, TikTok et Facebook piratés. D'après mes propres recherches sur le mode opératoire de la plateforme, le credential stuffing faisait partie des méthodes utilisées pour accéder à ces comptes.

Beaucoup de ces données sont ensuite revendues ou partagées gratuitement sur des forums et marketplaces comme BreachForums et ses successeurs, ou sur Telegram, où s'échangent de nombreuses listes. Les prix sont bien souvent dérisoires à l'unité, mais le modèle reste rentable en gros volume.

On peut aussi évoquer la notion de combolist "fresh", qui désigne simplement une certaine qualité : le fait que les données n'ont encore jamais été testées. On trouve également des "checkers" ou "config sellers", qui vendent des configs prêtes à l'emploi pour un site précis.

### Conclusion

Comme nous avons pu le voir, le credential stuffing est une attaque redoutable, capable de permettre l'accès à des comptes parfois très sensibles. Elle démontre aussi une chose essentielle : avoir un mot de passe fort ne sert à rien si l'attaquant le possède déjà en clair. La force d'un mot de passe protège contre le brute force, pas contre la réutilisation.

Cette attaque n'exploite d'ailleurs aucune faille technique du site visé : elle repose entièrement sur une habitude humaine, celle de réutiliser les mêmes identifiants d'un service à l'autre.

Reste une question qu'on n'a pas encore abordée : comment les sites détectent-ils et bloquent-ils ce genre d'attaque automatisée, et comment les attaquants contournent-ils ces protections ? C'est le sujet du prochain article de cette série, consacré au duel entre credential stuffing et systèmes anti-bot.

*Cet article est basé sur mes connaissances personnelles ; il existe sûrement d'autres méthodes que celles présentées ici.*
