_Ce guide est pour RPG Maker VXace, et uniquement fonctionnel pour cette version. Il vous faut donc ce logiciel pour pouvoir le suivre._

# Préambule

J'ai souhaité réaliser ce tuto suite à la publication de ce [tuto d'Aurélien](https://rpgmakeralliance.com/d/84-systeme-dinfiltration-complet-avec-rme) (FoxFiesta) sur l'infiltration. Je l'ai trouvé très sympathique mais un peu complexe au niveau de son utilisation des zones RME. Avec la team [RMEx](http://rmex.github.io/), nous nous sommes questionné sur une manière de l'améliorer.

Nous avons donc pensé à une nouvelle commande dont le but serait de proposer une sorte de _raycasting_ (plus d'infos [ici](http://projet-moteur-3d.e-monsite.com/pages/raycasting/raycasting.html)), dans le but de l'utiliser pour détecter les joueurs, les murs et générer une sorte de champ de vision pour le garde, plutôt que de jouer sur les zones dans lesquelles se trouve le joueur.

J'ai commencé à rédiger ce présent guide et je me suis aperçu qu'une autre partie pouvait être synthétisée sous la forme d'une commande. Il s'agit de la partie qui permet de vérifier si le garde regarde dans la direction du joueur. Avant j'utilisais des calculs basés sur les coordonnées X et Y du garde et du joueur, mais le résult était assez long et pas très compréhensible pour le premier venu. Avec les conseils de xvw et de grm j'ai donc décidé de réaliser cette nouvelle commande et j'ai pleuré cette petite centaine de lignes d'explications que j'ai supprimé en même temps de ce guide.

Les deux commandes ont été implémentées et sont désormais disponibles dans la [version v1.4 de RME](https://github.com/RMEx/RME/blob/v1.4.0/RME.rb). Le but de ce guide est de présenter ces commandes au travers d'un cas d'utilisation. 

Pour rappel, voici la procédure [d'installation de RME](https://github.com/RMEx/RME/wiki/Installation) et la [documentation de RME](http://rmex.github.io/RMEDoc/), page rassemblant toutes les commandes RME avec quelques exemples d'utilisation.

# Présentation du système

Le but va être de simuler le champ de vision du garde. On va d'abord déterminer si le héros est potentiellement dans le champ de vision. Si c'est le cas on va tracer un rayon entre le garde et le héros, récupérer les cases traversées par ce rayon et vérifier une à une s'il y a des obstacles qui empêcherait le garde de voir le héros.

Tout d'abord, on va voler sa map à Aurélien et placer notre garde et notre joueur.

![Depart](https://i.imgur.com/XqvTqwQ.png "Map de base")

On va dénombrer trois cas pratiques dans notre système. J'ai affiché le champ de vision théorique du garde (sans prendre en compte les obstacles) en rouge.

1 Le joueur ne se trouve pas dans le champ de vision du garde : non détecté
![Pas dans le champ de vision](https://imgur.com/2r1WUTt.png "Situation 1")

2 Le joueur se trouve dans le champ de vision du garde mais un mur bloque sa vision : non détecté
![Pas visible](https://imgur.com/TnB3YQZ.png "Situation 2")

3 Le joueur se trouve dans le champ de vision du garde et aucun obstacle ne bloque sa vision : détecté
![Visible](https://imgur.com/I5DY0lc.png "Situation 3")

L'évènement va donc se décomposer de la manière suivante :

```
- Détection de la présence du joueur dans le champ de vision du garde
- Si joueur présent dans le champ de vision du garde
  - Récupération des cases entre le garde et le joueur
  - Boucle parcourant le tableau de cases
    - Si la case présente une obstacle
	  - On sort de la boucle : un obstacle empêche le garde de voir le joueur
	- Si la case est occupée par le joueur
	  - On déclenche l'alerte : le joueur est repéré
	  - On sort de la boucle
    - Si on est à la fin du tableau, on sort de la boucle
  - Fin de la boucle
- Fin de la condition
```

Nous allons entrer dans les détails dans la suite de ce guide.

# Préparation

Assurez vous d'avoir RME d'ajouté dans votre projet. La version minimale à avoir est la 1.4 !

Ensuite, nous allons avoir besoin des éléments suivants :

1. Variable `[Cases]`
2. Variable `[Max Cases]`
3. Variable `[Case Courante]`

On utilisera également un interrupteur local et deux [labels locaux](https://github.com/RMEx/RME/wiki/Labels-et-labels-locaux "Lien vers la page labels locaux de RME Wiki").

Nous allons utiliser aussi les Zones de RPG Maker VXace. Elles vont nous servir à matérialiser les obstacles qui vont bloquer la vision. Donc appliquez la zone 1 (par exemple) à tous vos murs. Dans mon cas, cela donne ceci :

![Murs via les Zones](https://imgur.com/PNXxDoX.png "Utilisation des zones")

Notre garde sera un évènement en processus parallèle avec une apparence de garde. Je vous conseille de créer une deuxième page vide bloquée par l'interrupteur local A afin de ne pas exécuter la recherche en boucle et bloquer le joueur.

# Programmation

Notre garde sera un évent classique en processus parallèle. Ma gestion de l'alerte est gérée dans une page à part.

Afin de pouvoir recycler notre évent sur plusieurs gardes, on va utiliser un évènement commun. Il faut savoir que dans un évent commun, on garde les "attributs" de l'évent dans lequel il a été appelé (pas valable pour les évents commun en processus parallèle ou alors automatique, aucun évent ne les appelle). C'est à dire que les interrupteur locaux ainsi que les variables et labels locaux ou encore la commande [`me`](http://rmex.github.io/RMEDoc/#me "RMEdoc pour me") sont ceux de l'évent d'origine.

Mon évent ressemble donc à ça :
**Event - Garde**

```
| > Appeler Événement Commun : [Infiltration]
| >
```

Si vous le souhaitez, vous pouvez aussi tout mettre dans l'évent. Le code ne change en aucune partie.

## Présence du joueur dans le champ de vision

L'astuce ici, c'est d'utiliser la nouvelle commande [`event_look_towards_event?`](http://rmex.github.io/RMEDoc/#event_look_towards_event? "RMEdoc pour event_look_towards_event?"). Pour rappel, voici notre zone théorique.

![Champ de vision du garde](https://imgur.com/E7OQPmj.png "Champ de vision du garde")

Il s'agit d'un cône à 45° avec une profondeur de 10 cases. Pour détecter si le joueur est dedans ou pas, on va tout simplement utiliser la commande de la manière suivante.

```
event_look_towards_event?(me, 0, 10)
```

Si le joueur (évènement 0) est dans le champ de vision du garde (évenement me) avec une profondeur de 10 au plus, alors la commande renvoie `true`. On peut donc l'utiliser avec une condition Script. Le début de l'évent ressemble à ceci.

**Infiltration**
```
Condition : Script : event_look_towards_event?(me, 0, 10)
| > | >
| > Fin - Condition
```

## Prise en compte des obstacles

C'est ici que réside la force de ce guide. On va utiliser une toute nouvelle commande de RME : [`get_squares_between_events`](http://rmex.github.io/RMEDoc/#get_squares_between_events "RMEdoc pour get_squares_between_events"). Cette commande va "tracer" une ligne entre deux events et récupérer toutes les cases traversées par cette dernière.

![Presentation de get_squares_between_events](https://imgur.com/kIE26tz.png "Presentation de get_squares_between_events")

Donc l'idée est ici de tracer une ligne entre notre garde et notre joueur afin de voir si on ne passe pas à travers un mur.

![Explication get_squares](https://imgur.com/dXPRgi8.png)

Cas 1 : Le rayon passe à travers un mur. Le joueur ne devrait pas être visible.
Cas 2 : Le rayon ne passe à travers aucun mur. Le joueur est visible.

Ce rayon que l'on va récupérer, c'est un tableau. Chaque élément du tableau est une case qui est traversée par le rayon entre notre garde et notre joueur. Nous allons pouvoir vérifier si l'une d'entre elle n'est pas un mur qui empèche pas de voir le joueur.

Les cases du tableau sont numérotées à partir de 0. Pour obtenir le contenu d'une case on utilisera la commande RME [`get`](http://rmex.github.io/RMEDoc/#get "RMEdoc pour get"). On va utiliser une boucle avec nos trois variables. La première `[Cases]` sera pour stocker le résultat de `get_squares_between_events` via la commande RM Modifier une variable. Utilitez l'option script avec la commande suivante.

```
get_squares_between_events(me, 0)
```

Cette commande permet donc de récupérer toutes les cases traversées par un rayon qui part du garde (évènement me) et qui arrive au joueur (évènement 0).

Ensuite, nous allons assigner à la variable `[Max Cases]` la longueur du tableau récupéré dans la variable `[Cases]`. Pour cela on va utiliser encore une fois la commande RM Modifier une variable pour assigner la commande suivante à `[Max Cases]`

```
length(V[1])
```

*Note : Dans mon cas, ma variable `[Cases]` est la variable 1. Veuillez remplacer le 1 par l'ID de votre variable dans votre projet.*

Pour parcourir notre tableau, on va utiliser une boucle. A chaque itération de la boucle nous allons vérifier une case, un compteur permettra de savoir à quelle case nous nous trouvons. A la fin de la boucle on augmente le compteur de 1 et on vérifie si on n'est pas à la fin du tableau. Si c'est le cas, on sort de la boucle, sinon on repart pour un tour.

```
- Boucle
  - Vérification de la case courante
  - Incrémentation du compteur de la boucle
  - Si on est à la fin du tableau, on sort de la boucle
- Retour au début de la boucle
```

En termes d'évent, cela nous donne le code suivant

**Boucle**
```
| > Opération : Variable [0002:Max Cases] = length(V[1]) 
| > Opération : Variable [0003:Case Courante] = 0 
| > Boucle
| >| > Opération : Variable [0003:Case Courante] += 1 
| >| > Condition : Variable [0003:Case Courante] == Variable [0002:Max Cases] 
| >| >| > Sortir de la Boucle
| >| >| > 
| >| > Fin - Condition
| >| > 
| > Fin - Boucle
```

Première chose à faire dans cette boucle, c'est de récupérer les coordonnées X et Y de notre case courante. Pour cela, nous allons utiliser la commande [`get`](http://rmex.github.io/RMEDoc/#get "RMEdoc pour get"). Nous allons faire get une première fois pour récupérer la case. Le résultat sera de la forme `[x, y]` (x et y étant des nombres entiers). Étant donné qu'il s'agit encore d'un tableau, on doit utiliser la commande get pour récupérer son contenu. J'ai décidé d'utiliser des variable Ruby classiques pour stocker les x et y étant donné que l'on va les utiliser juste après dans le même appel de script. Le résultat donne ceci.

```
coordonnees = get(V[1], V[3])
x = get(coordonnees, 0)
y = get(coordonnees, 1)
```

*Note : Dans mon cas, la variable 1 est `[Cases]` et la variable 3 est `[Case Courante]`. Remplacez ces ID par ceux correspondants dans votre projet.*

Maintenant, nous allons faire des vérifications sur les cases. Nous pouvons distinguer 3 cas.

1. Il y a un obstacle sur la case
2. Il y a le héros sur la case
3. Il n'y a rien sur la case

Pour le cas 1. nous allons utiliser la commande [`region_id`](http://rmex.github.io/RMEDoc/#region_id "RMEdoc pour region_id"). C'est pour cette raison qu'on a utilisé les zones RPG maker (leur nom en anglais c'est region). Cette commande nous permet à partir de coordonnées XY de savoir quelle zone se trouve à cet endroit.

Pour le cas 2. c'est la commande [`event_at`](http://rmex.github.io/RMEDoc/#event_at "RMEdoc pour event_at") qui sera utilisée. Elle prend aussi les coordonnées XY d'une case comme paramètres. Elle renvoie l'id de l'évent qui se trouve sur la case, 0 s'il s'agit du joueur, -1 s'il n'y a personne.

Le cas 3 est simple, s'il n'y a rien sur la case, alors on ne fait rien et on passe à la case suivante.

On va donc stocker le résultat des commandes précédement citées dans des labels locaux. Il s'agit de variables locales mais qui peuvent être nommées. 

```
coordonnees = get(V[1], V[3])
x = get(coordonnees, 0)
y = get(coordonnees, 1)
SL[:zone_courante] = region_id(x, y)
SL[:event_courant] = event_at(x, y)
```

Pour le cas 1, on veut que si l'on rencontre un mur on sorte de la boucle étant donné que l'on ne peut pas "voir" plus loin. Cela se réalise de la manière suivante.

```
| > Condition : Script : SL[:zone_courante] > 0 
| >| > Sortir de la Boucle
| >| > 
| > Fin - Condition
```

Pour le cas 2, on souhaite que si l'on détecte le joueur alors on donne l'alerte. J'utilise un interrupteur local pour activer la page qui va s'occuper de mettre en scene l'alerte donnée.

```
| > Condition : Script : SL[:event_courant] == 0 
| >| > Opération : Interrupteur local A = Activé 
| >| > Sortir de la Boucle
| >| > 
| > Fin - Condition
```

Maintenant que tout est bon vous pouvez tester le système par vous même. Personnellement j'ai fait un garde qui fait des allers et retours, mais vous pouvez faire quelque chose de moins conventionnel.

Voila un petit exemple de rendu possible.
![Rendu infiltration](https://imgur.com/WYZ4Yre.png)

## Code évent complet

```
| > Condition : Script : event_look_towards_event?(me, 0, 10) 
| >| > Opération : Variable [0001:Cases] = get_squares_between_events(me, 0) 
| >| > Opération : Variable [0002:Max Cases] = length(V[1]) 
| >| > Opération : Variable [0003:Case Courante] = 0 
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
| >| >| >| > Opération : Interrupteur local A = Activé 
| >| >| >| > Sortir de la Boucle
| >| >| >| > 
| >| >| > Fin - Condition
| >| >| > Opération : Variable [0003:Case Courante] += 1 
| >| >| > Condition : Variable [0003:Case Courante] == Variable [0002:Max Cases] 
| >| >| >| > Sortir de la Boucle
| >| >| >| > 
| >| >| > Fin - Condition
| >| >| > 
| >| > Fin - Boucle
| >| > 
| > Fin - Condition
```

# Conclusion

Vous avez désormais un système d'infiltration complet. Si vous avez mis le contenu dans un évent commun, vous pouvez l'utiliser à volonté pour plusieurs gardes en même temps. N'oubliez pas que vous pouvez assigner n'importe quel chemin à votre garde, tant que l'évent est en processus parallèle, tout ira bien !

Merci au discord RMA et à la team RMEx pour les conseils pour la réalisation de ce guide. A bientôt !
