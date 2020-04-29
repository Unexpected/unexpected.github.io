---
layout: post
title:  "SHA25 - Le secret de la compression infinie - Solution"
author: "Flibble"
date:   2020-04-29 08:00:0 +0200
categories: ctf sha25
---

Ce post détaille la conception et solution du challenge [Le secret de la compression infinie]({{ site.baseurl }}{% link _posts/2020-04-27-sha25-infinite-compression.markdown %}). La première partie détaille le cheminement suivi pour la conception, et la seconde les étapes techniques pour créer l'ensemble des éléments. 


## Conception du challenge

N'étant pas du tout familier des CTF et autres concours qui tournent autour de la sécurité informatique, je me suis plutôt basé sur mon expérience des jeux (vidéos, plateaux, JDR, etc.) et j'ai commencé par définir les grandes lignes de ce que je voulais construire.

Ce que je voulais inclure :
* Des techniques (pas forcément complexes) de stéganographie parce que j'aime le principe
* De la manipulation de fichiers binaires, d'image et de son (la partie son a finalement sauté)
* Une "progression" où le joueur va découvrir les éléments les uns après les autres et avancer
* Un background sympa, une histoire, un contexte. Bref, qu'on s'amuse :)
* Une référence à Balkany, running gag du concours

Ce que je ne voulais pas :
* Des algos complexes / beaucoup de code pour extraire les éléments.
* Des techniques qui nécessitent des connaissances pointues sur un domaine.

Ma première idée était de tout construire en mode "poupée russe". On donne comme point de départ une image, qui contient un zip, qui contient un son, qui contient un autre type de fichier qui contient la solution. Ca correspondait assez bien à mes critères, mais je ne savais pas trop comment m'y prendre avec la partie son. J'ai envisagé d'enregistrer un message qui donnait un indice et une façon d'extraire l'étape suivante, mais ça semblait un peu léger. J'ai envisagé de modifier l'enregistrement pour qu'il soit "à l'envers", mais pour le coup, ça risquait de devenir assez cryptique. Je me suis dit que j'allais renommer le fichier avec la mauvaise extension pour que ce ne soit ni trop simple, ni trop complexe, mais ce n'était pas très fun.

Au final, j'ai fini par abandonner l'idée du son (ça sera pour une prochaine fois :D), et aussi le mode "poupée russe" qui avait un côté répétitif un peu dommage. J'ai gardé le côté encapsulation, mais au lieu d'emboiter les éléments successivement, j'ai choisi de les cacher tous au même niveau : dans une image. J'avais déjà eu l'occasion de manipuler des images et des techniques de stégano simples dans le cadre du PythonChallenge avec Pillow, je me dis que je vais réutiliser ces idées.  

Mon skillset douteux comprend entre autres quelques compétences en création de roms de NES. C'est une compétence assez rarement utile, qui permet de briller en société à peu près une fois tous les 5 ans, mais ça m'a semblé être un différenciant sympa. Déjà, ce n'est pas tous les jours qu'on trouve une rom de NES cachée dans une image, ça pouvait agréablement surprendre les participants, et quand j'ai relié les deux notions "rom de NES" et "Balkany", il m'est devenu totalement impossible de ne pas le faire. Ca me paraissait beaucoup trop fun. La ROM serait la dernière étape du puzzle.

Comme je n'avais que quelques jours avant la fin du SHA25 pour réaliser l'ensemble, il fallait que la rom reste très simple. J'envisage donc de faire un simple fond statique avec la tête de notre homme politique préféré, et un message qui donne le flag de fin. C'est là que j'ai imaginé le "background" du voleur d'octets.

J'ai finalement décidé d'ajouter un konami code en étape de fin (sous couvert que je réussisse à le coder), une "note" cachée dans une des étapes qui donnerait des indices sur la suite, et un "starter" très simple pour engager les joueurs dans le tout.

Les grandes étapes de l'énigme seront donc :
* Trouver un premier indice qui pointe vers "un coffre"
* Extraire le "coffre" et comprendre que c'est un zip, lire la note contenue dedans
* Reconstruire l'archive 7z qui est "dispersée" dans l'image
* Comprendre que le secret contenu dans l'archive est une rom de NES, l'extraire
* Lancer la rom et taper le Konami Code pour obtenir le flag

Facile ! Il n'y a plus qu'à construire tout ça.


## Implémentation 

On prend le problème "à l'envers" et on commence par la fin puisque les derniers éléments seront encapsulés dans les premiers. 

# La ROM

Pour construire la ROM, il nous faut 
* Le background avec la tête à Patoche
* Le texte qui donne un indice et le flag final
* Le code qui détecte le Konami Code et change l'affichage

Pour le background, comme je ne suis pas pixel-artist, je pars d'une image standard: 
[patoche1](/assets/images/sha25/patoche1.jpg)

On va donc d'abord "pixeliser" l'image avec un outil trouvé sur le net ([Retromancers](http://www.retromancers.com/retro/)) : 
[patoche2](/assets/images/sha25/patoche2.png)

Bon, la NES, c'est une résolution de 256x240 et un background en 4 couleurs. On choisit nos 4 couleurs cibles, et on mouline : 
~~~python
from PIL import Image
im = Image.open("input.png")
width, height = im.size
pixels = im.load()
final_colors = {(0,0,0), (128,0,0), (128,128,0), (128,128,128)}

def closest(c1,colors): 
    delta = 1000
    final = None
    (cr, cg, cb) = c1
    for (br, bg, bb) in colors: 
        d = abs(br-cr)+abs(bg-cg)+abs(bb-cb)
        if d < delta: 
            delta = d
            final = (br, bg, bb)
    return final

for y in range(height): 
    for x in range(width): 
        pr, pg, pb, pa = pixels[x,y]
        pixels[x,y] = closest((pr, pg, pb), final_colors)

im.save("output.png")
~~~
[patoche3](/assets/images/sha25/patoche3.png)

On va aussi virer le fond avec paint et réduire pour s'approcher de la résolution cible : 
[patoche4](/assets/images/sha25/patoche4.png)
 
Enfin, il va falloir qu'on écrive un message. On a pas de notion de texte dans le PPU, les lettres c'est nécessairement des tiles de 8x8. On va donc choisir le message à écrire ("LE SECRET: JE VOLE..." etc.), on fait la liste de tous les caractères nécessaires et on les dessine chacun une fois. L'avantage du 8x8, c'est qu'il n'y a pas besoin d'être trop créatif sur la police... 
[patoche5](/assets/images/sha25/patoche5.png)

Maintenant qu'on a nos assets, on va pouvoir le transformer en un format compréhensible par la NES. Il faut que l'on construise 2 choses distinctes : 
* La CHR, c'est en gros la sprite sheet utilisée par le PPU pour dessiner l'écran. On doit retrouver chaque bloc de 8x8 une fois. 
* Notre image de fond, avec pour chaque espace de 8x8, le n° du bloc de la CHR à utiliser. 

Par exemple, pour écrire "LES OCTETS" quelque part, on a besoin d'avoir dans la CHR :
[chr1](/assets/images/sha25/chr1.png =280x)

Chaque bloc de 8x8 est indexé sur un octet, on a donc les blocs $00, $01, $02 ... $05

Pour afficher notre message, il faudra écrire dans le PPU les index correspondant. 

Si on envoie $01 $02 $03 $00 $04 $05 $06 $02 $06 $03, on obtiendra : 
[chr2](/assets/images/sha25/chr2.png =400x)


