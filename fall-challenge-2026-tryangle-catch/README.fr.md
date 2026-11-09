# TryAngle Catch — CodinGame Fall Challenge 2026

[English](README.md) | **Français**

| | |
|---|---|
| **Concours** | [CodinGame Fall Challenge 2026](https://www.codingame.com/contests/fall-challenge-2026), du 2 au 9 novembre 2026 |
| **Jeu** | TryAngle Catch : deux joueurs, tours simultanés, sur un graphe de maisons |
| **Résultat** | Ligue Legend, la ligue la plus haute |
| **Bot final** | Un réseau de neurones sur graphe, appris en imitant les meilleurs joueurs, et une recherche sur un tour avec un simulateur exact |
| **Technologies** | C++17 (bot, simulateur, outils d'analyse), Python et PyTorch (entraînement), Java (arbitre officiel) |

## Le jeu

- La carte est un graphe de maisons reliées par des chemins. Trois maisons toutes reliées entre elles forment un triangle.
- Chaque joueur commence avec 3 unités. À chaque tour, les unités peuvent aller sur une maison voisine.
- Un joueur possède une maison quand il y a plus d'unités que l'adversaire. Posséder les trois coins d'un triangle permet de le prendre.
- Un triangle pris s'utilise une fois : pour faire apparaître une unité, attaquer une maison voisine (toutes les unités ennemies dessus meurent), ou ajouter ou retirer un chemin. Il redevient ensuite neutre.
- Une unité entourée de maisons ennemies meurt.
- Chaque triangle possédé rapporte 1 point par tour. Posséder 80 % des triangles fait gagner tout de suite.
- Les deux joueurs jouent en même temps, avec 50 ms par tour.

## Le bot final en bref

À chaque tour :

1. Un réseau de neurones sur graphe lit le plateau et propose des plans complets : un ordre pour chaque pile d'unités et une action pour chaque triangle utilisable.
2. Il propose jusqu'à 24 plans pour moi et jusqu'à 8 pour l'adversaire, en regardant le plateau depuis son côté. Des variantes écrites à la main s'ajoutent : mes triangles qui attaquent, et une réponse adverse « menace maximale ».
3. Un simulateur exact joue un tour pour chaque paire de plans (le mien contre le sien).
4. La sortie « valeur » du réseau note chaque position obtenue.
5. La note d'un plan mélange son pire cas (30 %) et sa moyenne (70 %) sur les plans adverses. Les plans faibles sont éliminés en cours de route, pour passer plus de temps sur les bons.
6. Le bot joue le meilleur plan. Il s'arrête après 36 ms, sur les 50 ms autorisées.

Tout le bot tient dans un seul fichier C++ d'au plus 100 000 caractères, réseau compris. Les poids sont codés sur 8 bits et rangés à raison de 15 bits par caractère Unicode. Le réseau tourne en C++ écrit à la main, avec les instructions AVX2.

## Chronologie

| Version | Date | Idée principale | Ligue |
|---|---|---|---|
| 1 | 3 novembre | Recherche classique réglée à la main | Bronze → Silver |
| 2 | 4 et 5 novembre | Recherche guidée par un petit réseau d'imitation | Silver → Gold |
| 3 | 7 novembre | Réseau d'imitation + recherche sur un tour | Gold → Legend |
| 4 | 7 novembre | Plus de données, de meilleurs plans candidats, des attaques | Legend (version finale) |

## Les versions du bot

### Version 1 : recherche classique réglée à la main

**Le principe.** À chaque tour, le bot construit un plan complet : un ordre pour chaque pile d'unités (aller sur une maison voisine ou rester) et une action pour chaque triangle utilisable (faire apparaître une unité ou attaquer).

**Comment il choisit.**
- Il part d'un plan glouton : des équipes de 3 unités vers les triangles les moins chers à prendre.
- Il modifie ce plan petit à petit et garde les modifications qui améliorent la note. C'est une montée de colline.
- Il alterne avec l'adversaire pendant le tour. Il cherche la meilleure réponse adverse à mon plan et la garde en mémoire, jusqu'à 4 réponses par tour. Puis il améliore mon plan contre ces réponses, en regardant surtout la pire.
- Chaque plan est simulé exactement contre ces réponses.
- La note vient d'une fonction d'évaluation écrite à la main : triangles possédés, unités, menaces, territoire, et « potentiel de capture » (le nombre de tours qu'il faut pour prendre chaque triangle).

**Le réglage à la main.** J'étudiais mes défaites, je corrigeais un poids ou j'ajoutais un terme, puis je testais sur des centaines de parties locales.

**Le plafond.** Après quelques versions, plus aucune correction ne passait : plus de piles, bonus d'attaque, pénalité pour les unités qui bloquent un triangle… Tout était rejeté.

### Version 2 : recherche guidée par imitation

**L'idée.** Quand les corrections à la main ont plafonné, j'ai ajouté un « guide » : un petit réseau qui imite les coups des joueurs forts.

**Le guide.**
- Un petit réseau sur graphe : 3 couches, 32 valeurs par maison, environ 30 caractéristiques par maison en entrée.
- Il est entraîné sur les parties de joueurs forts. Pour chaque pile, il donne la probabilité de chaque destination.
- Il prédisait mieux leurs déplacements que ma propre recherche : 51 % des unités envoyées sur la bonne maison, contre 40 %.
- Il ne coûte presque rien : il est calculé une seule fois par tour pour chaque camp, en 0,28 ms.

**Comment la recherche l'utilise.**
- **Pour mes plans :** la note d'un plan reçoit un petit bonus quand ses déplacements sont probables pour le guide. La recherche reste libre, mais elle essaie d'abord les coups qu'un bon joueur ferait.
- **Pour l'adversaire :** le même guide, appliqué depuis son côté du plateau, oriente la recherche de ses réponses. Ses coups sont mieux prédits.
- **Le bonus doit rester petit.** Trop fort, le bot copie le guide et devient bien plus faible.

**Résultat.** Contre la même recherche sans guide, il gagnait environ 2 parties sur 3. C'est cette version qui a fait monter le bot en Gold.

**Accélérer la recherche.**
- **Mesurer d'abord :** un profilage au compteur de cycles du processeur montrait que 74 % du temps passait dans l'évaluation, et environ 60 % dans le potentiel de capture des deux joueurs.
- **Une simulation exacte et rapide :** le cas le plus courant (déplacements vers une maison voisine et apparitions d'unités) passe par un chemin court, vérifié identique au moteur complet par un outil. Un tour simulé est passé d'environ 2,5 µs à moins de 1 µs.
- **Un potentiel de capture plus rapide, toujours exact :** les maisons à chaque distance de chaque coin sont rangées à l'avance en masques de bits, et un plateau commun avec un journal d'annulation remplace la copie du plateau pour chaque triangle.
- **Un cache des notes :** pendant un tour, le même couple « mon plan, son plan » revient souvent. 63 % des notes viennent du cache.
- **Au total,** par tour, en 38 ms : environ 7 200 évaluations calculées, plus 12 300 notes reprises du cache.
- **Refusé :** un potentiel approché donnait 45 % d'évaluations en plus, mais le bot prenait beaucoup moins de triangles. Ma règle : accélérer seulement si les décisions ne changent pas.

À ce stade, la vitesse n'était plus la limite : avec 10 % de temps en moins, le bot jouait aussi bien (598 victoires à 588 sur 1 200 parties). La limite était l'évaluation.

**Le nouveau plafond.** J'ai essayé 4 autres guides, dont 3 entraînés sur des parties de Gold. Ils prédisaient mieux les coups, mais aucun ne gagnait plus de parties (600 parties chacun). Les corrections à la main ont de nouveau plafonné. Je suis donc passé à un réseau bien plus gros, qui propose des plans complets et juge les positions.

### Version 3 : réseau d'imitation + recherche sur un tour

**Les données.**
- Des milliers de parties des meilleurs joueurs, téléchargées via CodinGame.
- Mon simulateur rejoue chaque partie coup par coup. Il en sort les positions et les coups réellement joués par chaque joueur : 1 à 2,5 millions de positions.
- Je garde surtout les coups des 30 premiers du classement.

**Ce que voit le réseau.**
- Le plateau est un graphe : des maisons reliées à leurs voisines, et des triangles formés de trois maisons.
- Environ 68 caractéristiques par maison : unités des deux joueurs, propriétaire, voisins, menaces, distances.
- Environ 33 caractéristiques par triangle : unités sur ses coins, propriétaire, prise possible au prochain tour, risque d'attaque.

**Le réseau.**
- Un réseau sur graphe de 4 couches, avec 64 valeurs par maison. À chaque couche, une maison reçoit l'information de ses voisines et des triangles dont elle fait partie.
- Trois sorties : pour chaque pile, une probabilité pour chaque destination ; pour chaque triangle, une probabilité pour chaque action ; et une « valeur », l'estimation de la chance de gagner.
- **Le décodage coordonné :** le plan n'est pas décidé d'un coup, mais en 6 passes. À chaque passe, le réseau sait quelles unités ont déjà reçu leur ordre. Les unités se répartissent, au lieu d'aller toutes au même endroit.
- **L'entraînement :** avec PyTorch, sur carte graphique, en environ 12 passages sur les données. Le réseau apprend à la fois à reproduire les coups des meilleurs et à prédire le vainqueur.
- **Dans le bot :** entre 82 000 et 99 000 paramètres selon la version, avec des poids codés sur 7 ou 8 bits pour tenir dans la limite de 100 000 caractères. Le calcul du réseau est réécrit en C++ avec les instructions AVX2.

**À chaque tour :**
1. Le réseau propose jusqu'à 24 plans pour moi : le premier est le plus probable, les autres sont tirés au hasard selon ses probabilités.
2. Il propose aussi jusqu'à 8 plans pour l'adversaire, en regardant le plateau depuis son côté.
3. Pour chaque paire (mon plan contre un plan adverse), le simulateur joue exactement le tour.
4. Le réseau note la position obtenue avec sa sortie « valeur ».
5. La note d'un plan est sa moyenne sur les plans adverses.
6. Les plans faibles sont éliminés en cours de route, pour passer plus de temps sur les bons.

**Résultat.** Premier de Gold, puis passage en Legend le 7 novembre, le jour de l'ouverture de la ligue.

### Version 4 : plus de données et de meilleurs candidats (version finale)

Ce qui a changé :
- **Les données :** les parties des 100 premiers, au lieu des 30.
- **Des plans tirés moins chers :** construits en 4 passes au lieu de 6, donc plus de plans tiennent dans le temps.
- **Des plans d'attaque ajoutés :** mes triangles frappent la plus grosse pile ennemie voisine. Avant, contre les forts, le bot n'attaquait presque jamais : aucune attaque dans 22 des 27 défaites contre le top 20.
- **Une réponse adverse « menace maximale » :** chaque triangle ennemi attaque ma plus grosse pile voisine, pour vérifier que mon plan y résiste.
- **Une note plus prudente :** 30 % de pire cas et 70 % de moyenne sur les plans adverses.
- **Une marge de sécurité :** 36 ms par tour au lieu de 40, après un dépassement de temps dans une partie de test.

## Ce qui n'a pas marché

- **Un réseau qui voit tout le plateau, calculé 2 fois plus vite (testé, jamais soumis).** À chaque couche, chaque maison reçoit aussi la moyenne de toutes les maisons. Le calcul du réseau a été réécrit (AVX2 et FMA, poids rangés en colonnes, 4 maisons à la fois, première passe de décodage commune à tous les plans), avec des décisions identiques au bit près. Un plan prenait 0,95 ms au lieu de 1,84 ms sur les serveurs de CodinGame, et le bot évaluait 60 à 70 positions par tour au lieu de 18. En local, la version rapide gagnait 800 parties sur 1 200 contre la même version à vitesse normale. Mais en parties de test contre les vrais joueurs du top, elle n'a pas fait mieux que la version 4 : j'ai donc gardé la version 4.
- **L'apprentissage par renforcement (PPO).** Le bot jouait contre lui-même et contre la recherche classique, et ses probabilités de coups étaient corrigées selon les victoires. Ça aidait un peu à vitesse normale, mais plus du tout une fois le réseau accéléré.
- **Un réseau plus gros, à 5 couches :** même précision, mais plus lent, donc moins bon.
- **Plus de données :** l'imitation plafonne à environ 58,5 % de coups prédits.
- **Un réglage automatique de l'évaluation classique (SPSA) :** il battait mon bot de référence, mais pas mes autres versions.
- **Regarder 2 tours plus loin dans la recherche classique,** avec une politique rapide pour les deux camps : aucun gain avec le vrai temps de calcul.
- **D'autres familles d'algorithmes, comparées au début dans un tournoi local :** une recherche de type Smitsimax ne gagnait que 8 à 18 % de ses parties contre la recherche locale, car les unités ne se coordonnaient pas. La co-évolution génétique gagnait 1 %.

## Où est la limite

J'ai rejoué mes 449 parties classées pour chercher la cause de mes défaites :
- **Le bot voit venir l'adversaire :** il avait prévu 85 % des prises adverses, 96 % de mes prises ratées et 98 % des attaques subies, dans les victoires comme dans les défaites.
- **La valeur du réseau annonce bien le vainqueur,** mieux que l'écart d'unités, et mes 4 derniers réseaux sont identiques sur ce point.
- **Même le meilleur de mes 24 plans candidats n'envoie que 72 % des unités là où un joueur du top 10 les envoie,** et plus de plans n'y change presque rien.

La limite est donc la qualité de l'imitation, pas la recherche ni la vitesse. Pour aller plus loin, il faudrait une recherche sur plusieurs tours, avec un réseau beaucoup plus rapide entraîné par renforcement pendant des jours.

## Les outils

- **Un simulateur exact du jeu en C++,** vérifié contre l'arbitre officiel : 0 écart sur environ 350 parties et 45 000 tours. Il rejoue aussi exactement les vraies parties.
- **Une arène locale qui joue les parties en parallèle :** des milliers de parties par test, avec une horloge virtuelle calée sur CodinGame. Les résultats ne dépendent donc pas de la charge de la machine.
- **Un profilage au compteur de cycles du processeur,** pour savoir où passe le temps avant d'optimiser.
- **Des vérifications d'exactitude :** chaque version rapide (simulation, potentiel, réseau) est comparée à sa version de référence, plan par plan.
- **Un générateur de bot** qui fait tenir le réseau dans les 100 000 caractères : codage des poids, compression du code en vérifiant que le code machine reste identique, suppression du code inutile.
- **Des parties de test CodinGame automatiques,** avec les mêmes adversaires et les mêmes cartes pour chaque version, comparées partie par partie.
- **Des outils d'analyse des parties :** triangles pris par tour, pertes, attaques, phase par phase.

## Ce que j'ai appris

- **Un gain en local ne garantit pas un gain en ligne.** Mes adversaires de test étaient des bots de la même famille. Ce sont les vraies parties, jouées au même moment contre les mêmes adversaires, qui tranchent.
- **Le jeu se décide entre les tours 6 et 15,** dans la course aux triangles. L'ouverture est égale.
- **Toujours juger contre plusieurs adversaires différents.** Un réglage peut battre un bot et perdre contre un autre.
- **Un guide qui prédit mieux ne guide pas forcément mieux.** Seules les parties tranchent.
- **Accélérer seulement sans changer les décisions.** Un raccourci approché peut faire perdre plus qu'il ne fait gagner.
