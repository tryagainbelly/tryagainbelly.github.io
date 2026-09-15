---
title: "TwinCat2 mot de passe à déverrouiller"
description: "Reverse de PLC (TwinCat2)"
date: 2026-04-01 09:00:00 +0200
categories: [Reverse]
tags: [reverse]
---

# TwinCat2 mot de passe à déverrouiller

TwinCat2 est un logiciel développé par Beckhoff, une entreprise technologique allemande qui est dans le domaine de l'automatisation pour la fabrication industrielle. TwinCat2 a été lancé en 1996, c'est une suite logicielle qui permet d’interfacer et de programmer des APIs.

**Disclaimer** : Toutes les informations et codes publiés ici sont fournis à titre strictement informatif et pédagogique. Je décline toute responsabilité quant à leur utilisation, détournement, ou aux dommages directs/indirects qui pourraient en découler.
Ils ne doivent pas être utilisés pour nuire, pénétrer, perturber ou compromettre des systèmes sans autorisation.

## Reconaissance

Première chose à faire, c'est de se renseigner sur le logiciel que l'on compte reverse. Pour ma part j'ai utilisé Detect It Easy pour comprendre dans quel environnement j'allais travailler. L'outil permet de récupérer des infos comme le type de fichier, son compilateur, l'architecture, etc. Des informations intéressantes.

![](/assets/img/twincat2/Capture.PNG)

Là, on voit donc que c'est un exécutable sur 32 bits qui a été fait en C++.

A ce stade-là, on est un peu bloqué si on n'ouvre pas le logiciel ou qu'on ne l'analyse pas. Pour ma part j'aime bien le lancer pour voir comment fonctionne la mise en place du mot de passe et les différentes fonctionnalités que l'on peut trouver.

Par exemple, si l'on crée un projet, on a la possibilité de le protéger en ajoutant un mot de passe. En regardant les différentes possibilités sur l'outil, on remarque une fonctionnalité "User Group Passwords" qui nous permet de savoir qu'il y a 8 niveaux (0-7) pour les User Group. Si on met un mot de passe sur le projet (admin123) et qu'ensuite on regarde les paramètres du projet, on trouve un système de droit en fonction du niveau de l'utilisateur. On a vu un peu plus tôt qu'il y avait différents niveaux, et en fait ici, ils l'utilisent pour faire un système de droit en fonction de l'authentification. Dans cette situation, ce n'est pas intéressant pour nous mais ça reste une fonctionnalité qui peut avoir son intérêt.

![](/assets/img/twincat2/Capture2.PNG)
![](/assets/img/twincat2/Capture3.PNG)


Ensuite, pour tester le fonctionnement, on peut fermer le logiciel et tenter de le relancer et d'ouvrir notre projet. Si on met un mauvais mot de passe, on tombe sur une fenêtre d'accès refusé et si on tente le bon mot de passe, alors on a bien accès à notre projet.

![](/assets/img/twincat2/Capture4.PNG)


L'outil fonctionne correctement et la mise en place du mot de passe aussi. On peut donc passer à l'étape suivante qui va être de comprendre le fonctionnement du mot de passe, savoir quand et comment il est vérifié.

## Analyse

Pour l'analyser, j'ai choisi d'utiliser l'outil Cheat Engine. La première étape est d'attacher le processus de TwinCat2 à Cheat Engine de façon à pouvoir analyser dynamiquement son fonctionnement.

Quand on tente d'ouvrir le projet, on voit notre fenêtre où est demandé l'entrée du mot de passe, on peut donc commencer par chercher si dans la mémoire dynamique on pourrait retrouver certaines chaînes de caractères. Dans notre cas, on en trouve. Par exemple, le mot de passe qu'on a mis pour protéger notre projet.

La question à se poser, c'est pourquoi ? En fait, Twincat2 utilise une fonction qui s'appelle "[lstrcmpia](https://learn.microsoft.com/en-us/windows/win32/api/winbase/nf-winbase-lstrcmpia)" . Elle va prendre 2 chaînes de caractères et les comparer sans être sensible à la casse (les strings sont stockés en majuscule dans la mémoire). Elle récupère l'un des mots de passe depuis la saisie utilisateur et l'autre qu'elle récupère en mémoire. L'un des problèmes de TwinCat2, c'est qu'il laisse le mot de passe en clair et il ne le hash pas, par exemple.
Si maintenant on tente de rentrer un mot de passe faux et qu'on le recherche, on tombe sur plusieurs adresses différentes, ce qui change du mot de passe du projet où il y avait seulement une adresse. La raison, c'est que le mot de passe que l'on vient de rentrer est utilisé plusieurs fois pour différentes choses.

![](/assets/img/twincat2/Capture6.PNG)
![](/assets/img/twincat2/Capture7.PNG)


Cheat Engine nous permet de regarder qui tente d'accéder à telle ou telle adresse. Bien pratique dans ce contexte, car ça peut nous permettre de voir qui tente d'accéder au mot de passe qui protège notre projet. Si on choisit de regarder qui tente d'accéder à l'adresse où est stocké "admin123" (Find out what accesses this address) et quand on relance l'ouverture du projet avec la tentative de mot de passe, on tombe sur l'instruction suivante ```mov ah,[edi]```, plutôt intéressante. Ah ce moment-là, on peut émettre l'hypothèse que le mot de passe est stocké à l'adresse de ```edi```. Les crochets autour de ```edi``` indiquent que ce n'est pas edi mais à son adresse.

>**Note**:
> À ce moment-là, on aurait très bien pu passer par l'input de l'utilisateur.Mais c'est un peu plus embêtant, car il y a plus d'instruction à trier.

Si on décide de regarder l'instruction dans la mémoire (Show disassembler), on tombe sur du code assembleur, et notre instruction se trouve en plein dans une boucle. La boucle dans laquelle on se trouve fonctionne avec un saut conditionnel. Dans notre cas ```jump if equal```, ça veut dire que si ```ah``` et ```al``` ne sont pas équivalents, il ne sautera pas. C'est plutôt intéressant comme découverte, mais on ne sait pas ce que c'est ```al```. On sait que ```ah``` c'est notre mot de passe de projet donc on pourrait se dire que ```al``` c'est le mot de passe rentré par l'utilisateur pour qu'il soit comparé. Si on regarde bien dans la boucle, il y a un autre détail important.

Si on regarde toute la boucle, on voit qu'il ```mov``` le contenu de ```esi``` dans ```al``` ensuite, il incrémente ```esi``` de 1, puis il fait la même chose avec ```edi``` et il compare les 2 avec l'instruction ```cmp ah,al'``. Si jamais c'est égal, alors il recommence, sinon il arrête. C'est quand même curieux et ça ressemble beaucoup à une vérification de mot de passe caractère par caractère.

![](/assets/img/twincat2/Capture8.png)

## Patch

Maintenant qu'on a ces informations, on va essayer de les utiliser pour contourner la vérification. On peut se dire que dans la boucle, il vérifie chaque caractère et si jamais c'est égal, alors il retourne au début de la boucle jusqu'à ce qu'il n'y ait plus de caractère ou alors que ça ne soit pas égal. On peut donc se poser la question suivante. Que se passera-t-il si je lui dis que peu importe le résultat de la comparaison, il doit sauter à la fin. Et bah ça règle notre problème. Le fait de transformer notre saut conditionnel en un simple saut nous permet de contourner la vérification du mot de passe.

C'est sympa, on a un patch, mais je trouve cela un peu simpliste. On pourrait sûrement aller un peu plus loin et s'amuser.

On peut donc se poser la question suivante : "Qu'est-ce qui serait amusant et pratique pour un utilisateur ?" 
L'une des réponses à cette question est d'afficher une fenêtre dans laquelle on fait fuiter le mot de passe. C'est ce qu'on va faire.
Pour faire cela, nous rencontrons un problème majeur qui est le manque de place. Pour pallier à ce problème, on peut utiliser une méthode qui s'appelle le code injection. J'ai trouvé cette méthode sur une [vidéo de LeHack 2025](https://tuxlu.fr/flag_quest_writeup#gui-event-scripts) qui parlait de méthode avancer sur Cheat Engine.

La partie suivante est de trouver l'endroit où nous souhaitons injecter notre code. Par ma part j'ai fait plusieurs tests dans des boucles et j'ai eu des soucis donc j'ai décidé d'aller un peu plus haut (au niveau de ```push ebx```, 3ème ligne en partant du haut) injecter mon code.

![](/assets/img/twincat2/Capture9.png)

Pour faire du code d'injection, on clique sur l'instruction où l'on souhaite injecter du code. On va ensuite dans le menu en haut -> Tools -> Auto Assemble -> Templates -> code injection.

On se retrouve donc avec cette structure de base:

```cpp
// ==========================================================
// [ DÉCLARATIONS ]
// ==========================================================
alloc(newmem,2048)
label(returnhere)
label(originalcode)
label(exit)
// ==========================================================

// ==========================================================
// [ NOUVELLE ZONE MEMOIRE ]
// ==========================================================
newmem: //this is allocated memory, you have read,write,execute access
//place your code here
// ==========================================================

// ==========================================================
// [ COPIE DU CODE ORIGINAL ]
// ==========================================================
originalcode:
push ebx
mov esi,[ebp+0C]
mov edi,[ebp+08]

exit:
jmp returnhere
// ==========================================================

// ==========================================================
// [ POINT D'INJECTION ]
// ==========================================================
"TCatPlcCtrl.exe"+B03D65:
jmp newmem
nop 2
returnhere:
// ==========================================================
```

Ce qui nous intéresse, c'est de rajouter du code dans notre nouvelle zone mémoire.
Pour faire apparaître la fenêtre j'utilise une fonction de base de l'API Windows qui est [MessageBoxA](https://learn.microsoft.com/en-us/windows/win32/api/winuser/nf-winuser-messageboxa) et qui fonctionne avec des paramètres. Il suffit de push les paramètres pour ensuite utiliser la fonction.


Voici mon code :

```cpp
[ENABLE] // Utile dans le cas du développement d’un trainer
alloc(newmem,2048)
label(returnhere)
label(originalcode)
label(code)

newmem:

jmp code // Permet de jumper directement à la section code

run:
  db 0 // Définit un octet à zéro qu’on va nommé run
titre:
  db "Password found !",0 // Définit une chaine à l’adresse titre
 
code:	// Section code
  cmp [run], 1 // Compare si la valeur de run est = à 1
  je originalcode // Si oui, saute à la section originalcode
  pushad // Envoie tout les registres de données dans la stack 
  pushfd // Envoie tout les registres d’état (FLAG) dans la stack

  mov [run], 1 // Met la valeur 1 à run

  xor eax, eax // Met eax à zéro
  push eax // Paramètre identifiant de la fenêtre parente
  push titre // Paramètre du titre de la fenêtre
  push [ebp+08] // Texte dans la fenêtre
  push eax // Paramètre pour définir l’apparence (0 donc par défaut)
  call USER32.MessageBoxA // Appelle une message box avec les paramètres mis auparavant 

  popfd // Récupère tout les registres de données dans la stack 
  popad // Récupère tout les registres d’état (FLAG) dans la stack

originalcode: // Code d’origine
push ebx
mov esi,[ebp+0C]
mov edi,[ebp+08]
jmp returnhere



"TCatPlcCtrl.exe"+B03D65:
jmp newmem
nop 2
returnhere:

[DISABLE] // Utile dans le cas du développement d’un trainer
"TCatPlcCtrl.exe"+B03D65:
  push ebx
  mov esi,[ebp+0C]
  mov edi,[ebp+08]
```

Ce code me permet donc de faire apparaître une petite fenêtre avec le mot de passe dessus. Donc, en tant qu'utilisateur, quand je tente de rentrer un mot de passe mauvais, j'ai une fenêtre avec qui apparaît avec le mot de passe qui protège le projet. C'est gagné, on a un patch fonctionnel et rigolo.

![](/assets/img/twincat2/Capture12.png)


## Sources

Blog de [Tuxlu](https://tuxlu.fr/flag_quest_writeup)

Video [LeHack](https://youtu.be/vOc_Rhg6Us0?is=mxYhgFqO_6FDskvb)
