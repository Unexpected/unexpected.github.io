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

# La ROM - les assets

Pour construire la ROM, il nous faut 
* Le background avec la tête à Patoche
* Le texte qui donne un indice et le flag final
* Le code qui détecte le Konami Code et change l'affichage

Pour le background, comme je ne suis pas pixel-artist, je pars d'une image standard: 
![patoche1](/assets/images/sha25/patoche1.jpg)

On va donc d'abord "pixeliser" l'image avec un outil trouvé sur le net ([Retromancers](http://www.retromancers.com/retro/)) : 
![patoche2](/assets/images/sha25/patoche2.png)

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
![patoche3](/assets/images/sha25/patoche3.png)

On va aussi virer le fond avec paint et réduire pour s'approcher de la résolution cible : 
![patoche4](/assets/images/sha25/patoche4.png)
 
Enfin, il va falloir qu'on écrive un message. On a pas de notion de texte dans le PPU, les lettres c'est nécessairement des tiles de 8x8. On va donc choisir le message à écrire ("LE SECRET: JE VOLE..." etc.), on fait la liste de tous les caractères nécessaires et on les dessine chacun une fois. L'avantage du 8x8, c'est qu'il n'y a pas besoin d'être trop créatif sur la police... 

Je vous passe les détails, mais je me suis rendu compte plus loin dans le process que mon image n'allait pas à cause du trop grand nombre de tiles différentes, j'ai donc un peu réduit la tête du bonhomme et viré le col du costume pour encore simplifier. 

![patoche5](/assets/images/sha25/patoche5.png)

# La ROM - la CHR et le Background

Maintenant qu'on a nos assets, on va pouvoir le transformer en un format compréhensible par la NES. Il faut que l'on construise 2 choses distinctes : 
* La CHR, c'est en gros la sprite sheet utilisée par le PPU pour dessiner l'écran. On doit retrouver chaque bloc de 8x8 une fois. 
* Notre image de fond, avec pour chaque espace de 8x8, le n° du bloc de la CHR à utiliser. 

Par exemple, pour écrire "LES OCTETS" quelque part, on a besoin d'avoir dans la CHR :
![chr1](/assets/images/sha25/chr1.png)

Chaque bloc de 8x8 est indexé sur un octet, on a donc les blocs $00, $01, $02 ... $05

Pour afficher notre message, il faudra écrire dans le PPU les index correspondant. 

Si on envoie $01 $02 $03 $00 $04 $05 $06 $02 $06 $03, on obtiendra : 
![chr2](/assets/images/sha25/chr2.png)


La CHR doit être dans un fichier séparé de la ROM. J'ai souvenir que le format d'encodage est un peu pénible (chaque pixel est sur 2 bit --> palette de 4 couleurs, mais tout n'est pas rangé "dans l'ordre"). Mais heureusement, il y a github et une belle communauté de gens avec des passions douteuses comme les miennes, il est donc facile de trouver un [outil magique](https://github.com/play3577/nes-chr-encodenes ) !

Cet outil permet de transformer une image en fichier CHR. Bon par contre, c'est a demi-magique, il faut quand même avoir construit notre grille de tiles auparavant. C'est reparti pour un script en python. 

~~~python
from PIL import Image
im = Image.open("balkanes.png")
width, height = im.size

pixels = im.load()

# La CHR doit faire 128 de large (16 tiles) par 256
# On va construire la CHR au fil de l'eau en parsant l'image
img_chr = Image.new('RGB', (128, 256), color = 'black')
pixels_chr = img_chr.load()

colors = set()

block_counter = 1
distinct_blocks = dict()
block_numbers = []

# On parcoure les blocks de 8x8 
for line in range(int(height/8)): 
    for col in range(int(width/8)): 
        block = ""
        b = []
        for y in range(line*8, line*8+8): 
            for x in range(col*8,col*8+8): 
                b.append(pixels[x,y])
            block += "".join([str(p) for p in b])
        
        if block not in distinct_blocks: 
			# On regarde si on a déjà notre tile de 8x8 dans notre dictionnaire, si ce n'est pas le cas, on va l'ajouter : 
			# - dans le dictionnaire distinct_blocks
			# - dans l'image de la CHR
            base_x = 8*(block_counter % 16)
            base_y = 8*int(block_counter/16)
            for y in range(8): 
                for x in range(8): 
                    pixels_chr[x+base_x, y+base_y] = b.pop(0)
            
            distinct_blocks[block] = block_counter
            block_counter += 1
            
        block_numbers.append(distinct_blocks[block])

# On va printer comme un sale les lignes dans la console, je les copierai dans ma ROM directement
# Le format des lignes dont on a besoin : 
#  .db $24,$24,$24,$24,$24,$24,$24,$24,$24,$24,$24,$24,$24,$24,$24,$24  ;;row 1
#  .db $24,$24,$24,$24,$24,$24,$24,$24,$24,$24,$24,$24,$24,$24,$24,$24  ;;all sky

for y in range(60): 
    line = "  .db " 
    for x in range(16): 
        b = block_numbers.pop(0)
        line += ("$%s," % str(hex(b))[2:].upper().zfill(2))
    print(line[:-1])

img_chr.save("balkanes_chr.png")

~~~

En sortie de ce script, on obtient donc les 2 choses dont on avait besoin (quasiment) : 
* Une image qui correspond à notre future CHR avec toutes les tiles 8x8 différentes de notre image (on aura plus qu'à la passer dans l'outil cité précédemment pour obtenir le fichier .chr): 
![chr3](/assets/images/sha25/chr3.png)
* Nos données à intégrer aux sources de la ROM qui décrivent le background : 
~~~
background:
  .db $00,$00,$00,$00,$00,$00,$00,$00,$00,$00,$00,$00,$00,$00,$00,$00
  .db $00,$1A,$1B,$1C,$1D,$1E,$1F,$20,$00,$00,$21,$22,$00,$00,$00,$00
  ;... etc (2 lignes d'octets = 32 octets = 32 tiles = 256 pixels de large
  ;...      il y a 60 lignes d'octets pour 240 pixels de haut)
~~~



Une fois la ROM construite, on peut retrouver ces éléments avec le "PPU viewer" (vue de la mémoire graphique, on retrouve un équivalent dans à peu près tous les émulateurs de NES). 
Ci-dessous, en rouge, la ligne de caractères que l'on voit décrite dans la banque de mémoire de la ROM, et en jaune, le caractère ":" qui est à l'adresse $08 dans la CHR. 
![romdebug](/assets/images/sha25/nes_debug.png)



