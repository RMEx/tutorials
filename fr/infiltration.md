_Ce guide est pour RPG Maker VXace, et uniquement fonctionnel pour cette version. Il vous faut donc ce logiciel pour pouvoir le suivre._

# Pr�ambule

J'ai souhait� r�aliser ce tuto suite � la publication de ce [tuto d'Aur�lien](https://rpgmakeralliance.com/d/84-systeme-dinfiltration-complet-avec-rme) (FoxFiesta) sur l'infiltration. Je l'ai trouv� tr�s sympathique mais un peu complexe au niveau de son utilisation des zones RME. Avec la team [RMEx](http://rmex.github.io/), nous nous sommes questionn� sur une mani�re de l'am�liorer.

Nous avons donc pens� � une nouvelle commande dont le but serait de proposer une sorte de _raycasting_ (plus d'infos [ici](http://projet-moteur-3d.e-monsite.com/pages/raycasting/raycasting.html)), dans le but de l'utiliser pour d�tecter les joueurs, les murs et g�n�rer une sorte de champ de vision pour le garde, plut�t que de jouer sur les zones dans lesquelles se trouve le joueur.

J'ai commenc� � r�diger ce pr�sent guide et je me suis aper�u qu'une autre partie pouvait �tre synth�tis�e sous la forme d'une commande. Il s'agit de la partie qui permet de v�rifier si le garde regarde dans la direction du joueur. Avant j'utilisais des calculs bas�s sur les coordonn�es X et Y du garde et du joueur, mais le r�sult �tait assez long et pas tr�s compr�hensible pour le premier venu. Avec les conseils de xvw et de grm j'ai donc d�cid� de r�aliser cette nouvelle commande et j'ai pleur� cette petite centaine de lignes d'explications que j'ai supprim� en m�me temps de ce guide.

Les deux commandes ont �t� impl�ment�es et sont d�sormais disponibles dans la [version v1.4 de RME](https://github.com/RMEx/RME/blob/v1.4.0/RME.rb). Le but de ce guide est de pr�senter ces commandes au travers d'un cas d'utilisation. 

Pour rappel, voici la proc�dure [d'installation de RME](https://github.com/RMEx/RME/wiki/Installation) et la [documentation de RME](http://rmex.github.io/RMEDoc/), page rassemblant toutes les commandes RME avec quelques exemples d'utilisation.

# Pr�sentation du syst�me

Le but va �tre de simuler le champ de vision du garde. On va d'abord d�terminer si le h�ros est potentiellement dans le champ de vision. Si c'est le cas on va tracer un rayon entre le garde et le h�ros, r�cup�rer les cases travers�es par ce rayon et v�rifier une � une s'il y a des obstacles qui emp�cherait le garde de voir le h�ros.

Tout d'abord, on va voler sa map � Aur�lien et placer notre garde et notre joueur.

![Depart](https://i.imgur.com/XqvTqwQ.png "Map de base")

On va d�nombrer trois cas pratiques dans notre syst�me. J'ai affich� le champ de vision th�orique du garde (sans prendre en compte les obstacles) en rouge.

1 Le joueur ne se trouve pas dans le champ de vision du garde : non d�tect�
![Pas dans le champ de vision](https://imgur.com/2r1WUTt.png "Situation 1")

2 Le joueur se trouve dans le champ de vision du garde mais un mur bloque sa vision : non d�tect�
![Pas visible](https://imgur.com/TnB3YQZ.png "Situation 2")

3 Le joueur se trouve dans le champ de vision du garde et aucun obstacle ne bloque sa vision : d�tect�
![Visible](https://imgur.com/I5DY0lc.png "Situation 3")

L'�v�nement va donc se d�composer de la mani�re suivante :

```
- D�tection de la pr�sence du joueur dans le champ de vision du garde
- Si joueur pr�sent dans le champ de vision du garde
  - R�cup�ration des cases entre le garde et le joueur
  - Boucle parcourant le tableau de cases
    - Si la case pr�sente une obstacle
	  - On sort de la boucle : un obstacle emp�che le garde de voir le joueur
	- Si la case est occup�e par le joueur
	  - On d�clenche l'alerte : le joueur est rep�r�
	  - On sort de la boucle
    - Si on est � la fin du tableau, on sort de la boucle
  - Fin de la boucle
- Fin de la condition
```

Nous allons entrer dans les d�tails dans la suite de ce guide.

# Pr�paration

Assurez vous d'avoir RME d'ajout� dans votre projet. La version minimale � avoir est la 1.4 !

Ensuite, nous allons avoir besoin des �l�ments suivants :

1. Variable `[Cases]`
2. Variable `[Max Cases]`
3. Variable `[Case Courante]`

On utilisera �galement un interrupteur local et deux [labels locaux](https://github.com/RMEx/RME/wiki/Labels-et-labels-locaux "Lien vers la page labels locaux de RME Wiki").

Nous allons utiliser aussi les Zones de RPG Maker VXace. Elles vont nous servir � mat�rialiser les obstacles qui vont bloquer la vision. Donc appliquez la zone 1 (par exemple) � tous vos murs. Dans mon cas, cela donne ceci :

![Murs via les Zones](https://imgur.com/PNXxDoX.png "Utilisation des zones")

Notre garde sera un �v�nement en processus parall�le avec une apparence de garde. Je vous conseille de cr�er une deuxi�me page vide bloqu�e par l'interrupteur local A afin de ne pas ex�cuter la recherche en boucle et bloquer le joueur.

# Programmation

Notre garde sera un �vent classique en processus parall�le. Ma gestion de l'alerte est g�r�e dans une page � part.

Afin de pouvoir recycler notre �vent sur plusieurs gardes, on va utiliser un �v�nement commun. Il faut savoir que dans un �vent commun, on garde les "attributs" de l'�vent dans lequel il a �t� appel� (pas valable pour les �vents commun en processus parall�le ou alors automatique, aucun �vent ne les appelle). C'est � dire que les interrupteur locaux ainsi que les variables et labels locaux ou encore la commande [`me`](http://rmex.github.io/RMEDoc/#me "RMEdoc pour me") sont ceux de l'�vent d'origine.

Mon �vent ressemble donc � �a :
**Event - Garde**
| > Appeler �v�nement Commun : [Infiltration]
| >

Si vous le souhaitez, vous pouvez aussi tout mettre dans l'�vent. Le code ne change en aucune partie.

## Pr�sence du joueur dans le champ de vision

L'astuce ici, c'est d'utiliser la nouvelle commande [`event_look_towards_event?`](http://rmex.github.io/RMEDoc/#event_look_towards_event? "RMEdoc pour event_look_towards_event?"). Pour rappel, voici notre zone th�orique.

![Champ de vision du garde](https://imgur.com/E7OQPmj.png "Champ de vision du garde")

Il s'agit d'un c�ne � 45� avec une profondeur de 10 cases. Pour d�tecter si le joueur est dedans ou pas, on va tout simplement utiliser la commande de la mani�re suivante.

```
event_look_towards_event?(me, 0, 10)
```

Si le joueur (�v�nement 0) est dans le champ de vision du garde (�venement me) avec une profondeur de 10 au plus, alors la commande renvoie `true`. On peut donc l'utiliser avec une condition Script. Le d�but de l'�vent ressemble � ceci.

**Infiltration**
Condition : Script : event_look_towards_event?(me, 0, 10)
| > | >
| > Fin - Condition

## Prise en compte des obstacles

C'est ici que r�side la force de ce guide. On va utiliser une toute nouvelle commande de RME : [`get_squares_between_events`](http://rmex.github.io/RMEDoc/#get_squares_between_events "RMEdoc pour get_squares_between_events"). Cette commande va "tracer" une ligne entre deux events et r�cup�rer toutes les cases travers�es par cette derni�re.

![Presentation de get_squares_between_events](https://imgur.com/kIE26tz.png "Presentation de get_squares_between_events")

Donc l'id�e est ici de tracer une ligne entre notre garde et notre joueur afin de voir si on ne passe pas � travers un mur.

![Explication get_squares](https://imgur.com/dXPRgi8.png)

Cas 1 : Le rayon passe � travers un mur. Le joueur ne devrait pas �tre visible.
Cas 2 : Le rayon ne passe � travers aucun mur. Le joueur est visible.

Ce rayon que l'on va r�cup�rer, c'est un tableau. Chaque �l�ment du tableau est une case qui est travers�e par le rayon entre notre garde et notre joueur. Nous allons pouvoir v�rifier si l'une d'entre elle n'est pas un mur qui emp�che pas de voir le joueur.

Les cases du tableau sont num�rot�es � partir de 0. Pour obtenir le contenu d'une case on utilisera la commande RME [`get`](http://rmex.github.io/RMEDoc/#get "RMEdoc pour get"). On va utiliser une boucle avec nos trois variables. La premi�re `[Cases]` sera pour stocker le r�sultat de `get_squares_between_events` via la commande RM Modifier une variable. Utilitez l'option script avec la commande suivante.

```
get_squares_between_events(me, 0)
```

Cette commande permet donc de r�cup�rer toutes les cases travers�es par un rayon qui part du garde (�v�nement me) et qui arrive au joueur (�v�nement 0).

Ensuite, nous allons assigner � la variable `[Max Cases]` la longueur du tableau r�cup�r� dans la variable `[Cases]`. Pour cela on va utiliser encore une fois la commande RM Modifier une variable pour assigner la commande suivante � `[Max Cases]`

```
length(V[1])
```

*Note : Dans mon cas, ma variable `[Cases]` est la variable 1. Veuillez remplacer le 1 par l'ID de votre variable dans votre projet.*

Pour parcourir notre tableau, on va utiliser une boucle. A chaque it�ration de la boucle nous allons v�rifier une case, un compteur permettra de savoir � quelle case nous nous trouvons. A la fin de la boucle on augmente le compteur de 1 et on v�rifie si on n'est pas � la fin du tableau. Si c'est le cas, on sort de la boucle, sinon on repart pour un tour.

```
- Boucle
  - V�rification de la case courante
  - Incr�mentation du compteur de la boucle
  - Si on est � la fin du tableau, on sort de la boucle
- Retour au d�but de la boucle
```

En termes d'�vent, cela nous donne le code suivant

**Boucle**
| > Op�ration : Variable [0002:Max Cases] = length(V[1]) 
| > Op�ration : Variable [0003:Case Courante] = 0 
| > Boucle
| >| > Op�ration : Variable [0003:Case Courante] += 1 
| >| > Condition : Variable [0003:Case Courante] == Variable [0002:Max Cases] 
| >| >| > Sortir de la Boucle
| >| >| > 
| >| > Fin - Condition
| >| > 
| > Fin - Boucle

Premi�re chose � faire dans cette boucle, c'est de r�cup�rer les coordonn�es X et Y de notre case courante. Pour cela, nous allons utiliser la commande [`get`](http://rmex.github.io/RMEDoc/#get "RMEdoc pour get"). Nous allons faire get une premi�re fois pour r�cup�rer la case. Le r�sultat sera de la forme `[x, y]` (x et y �tant des nombres entiers). �tant donn� qu'il s'agit encore d'un tableau, on doit utiliser la commande get pour r�cup�rer son contenu. J'ai d�cid� d'utiliser des variable Ruby classiques pour stocker les x et y �tant donn� que l'on va les utiliser juste apr�s dans le m�me appel de script. Le r�sultat donne ceci.

```
coordonnees = get(V[1], V[3])
x = get(coordonnees, 0)
y = get(coordonnees, 1)
```

*Note : Dans mon cas, la variable 1 est `[Cases]` et la variable 3 est `[Case Courante]`. Remplacez ces ID par ceux correspondants dans votre projet.*

Maintenant, nous allons faire des v�rifications sur les cases. Nous pouvons distinguer 3 cas.

1. Il y a un obstacle sur la case
2. Il y a le h�ros sur la case
3. Il n'y a rien sur la case

Pour le cas 1. nous allons utiliser la commande [`region_id`](http://rmex.github.io/RMEDoc/#region_id "RMEdoc pour region_id"). C'est pour cette raison qu'on a utilis� les zones RPG maker (leur nom en anglais c'est region). Cette commande nous permet � partir de coordonn�es XY de savoir quelle zone se trouve � cet endroit.

Pour le cas 2. c'est la commande [`event_at`](http://rmex.github.io/RMEDoc/#event_at "RMEdoc pour event_at") qui sera utilis�e. Elle prend aussi les coordonn�es XY d'une case comme param�tres. Elle renvoie l'id de l'�vent qui se trouve sur la case, 0 s'il s'agit du joueur, -1 s'il n'y a personne.

Le cas 3 est simple, s'il n'y a rien sur la case, alors on ne fait rien et on passe � la case suivante.

On va donc stocker le r�sultat des commandes pr�c�dement cit�es dans des labels locaux. Il s'agit de variables locales mais qui peuvent �tre nomm�es. 

```
coordonnees = get(V[1], V[3])
x = get(coordonnees, 0)
y = get(coordonnees, 1)
SL[:zone_courante] = region_id(x, y)
SL[:event_courant] = event_at(x, y)
```

Pour le cas 1, on veut que si l'on rencontre un mur on sorte de la boucle �tant donn� que l'on ne peut pas "voir" plus loin. Cela se r�alise de la mani�re suivante.

| > Condition : Script : SL[:zone_courante] > 0 
| >| > Sortir de la Boucle
| >| > 
| > Fin - Condition

Pour le cas 2, on souhaite que si l'on d�tecte le joueur alors on donne l'alerte. J'utilise un interrupteur local pour activer la page qui va s'occuper de mettre en scene l'alerte donn�e.

| > Condition : Script : SL[:event_courant] == 0 
| >| > Op�ration : Interrupteur local A = Activ� 
| >| > Sortir de la Boucle
| >| > 
| > Fin - Condition

Maintenant que tout est bon vous pouvez tester le syst�me par vous m�me. Personnellement j'ai fait un garde qui fait des allers et retours, mais vous pouvez faire quelque chose de moins conventionnel.

Voila un petit exemple de rendu possible.
![Rendu infiltration](https://imgur.com/WYZ4Yre.png)

## Code �vent complet

| > Condition : Script : event_look_towards_event?(me, 0, 10) 
| >| > Op�ration : Variable [0001:Cases] = get_squares_between_events(me, 0) 
| >| > Op�ration : Variable [0002:Max Cases] = length(V[1]) 
| >| > Op�ration : Variable [0003:Case Courante] = 0 
| >| > Boucle
| >| >| > Appeler Script : coordonnees = get(V[1], V[3]) 
| >| >| > Appeler Script : x = get(coordonnees, 0) 
| >| >| > Appeler Script : y = get(coordonnees, 1) 
| >| >| > Appeler Script : SL[:zone_courante] = region_id(x, y) 
| >| >| > Appeler Script : SL[:event_courant] = event_at(x, y) 
| >| >| > Condition : Script : SL[:zone_courante] > 0 
| >| >| >| > Sortir de la Boucle
| >| >| >| > 
| >| >| > Fin - Condition
| >| >| > Condition : Script : SL[:event_courant] == 0 
| >| >| >| > Op�ration : Interrupteur local A = Activ� 
| >| >| >| > Sortir de la Boucle
| >| >| >| > 
| >| >| > Fin - Condition
| >| >| > Op�ration : Variable [0003:Case Courante] += 1 
| >| >| > Condition : Variable [0003:Case Courante] == Variable [0002:Max Cases] 
| >| >| >| > Sortir de la Boucle
| >| >| >| > 
| >| >| > Fin - Condition
| >| >| > 
| >| > Fin - Boucle
| >| > 
| > Fin - Condition

# Conclusion

Vous avez d�sormais un syst�me d'infiltration complet. Si vous avez mis le contenu dans un �vent commun, vous pouvez l'utiliser � volont� pour plusieurs gardes en m�me temps. N'oubliez pas que vous pouvez assigner n'importe quel chemin � votre garde, tant que l'�vent est en processus parall�le, tout ira bien !

Merci au discord RMA et � la team RMEx pour les conseils pour la r�alisation de ce guide. A bient�t !
