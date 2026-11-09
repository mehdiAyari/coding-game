# Back Track King — CodinGame Summer Challenge 2026

[English](README.md) | **Français**

| | |
|---|---|
| **Concours** | [CodinGame Summer Challenge 2026](https://www.codingame.com/contests/summer-challenge-2026-back-track-king), du 7 au 21 septembre 2026 |
| **Jeu** | Back Track King : deux joueurs construisent des voies ferrées sur une grille pour relier des villes, tours simultanés |
| **Résultat** | Ligue Legend, la ligue la plus haute |
| **Bot final** | Un ensemble de deux réseaux de neurones de ma conception (S3z), appris par imitation, qui tourne en C++ |
| **Technologies** | C++17 (bots, simulateur), Python et PyTorch (simulateur, entraînement), Java (arbitre officiel) |

## Le jeu

- Deux joueurs posent des rails sur une grille (de 21×14 à 30×20). Des villes veulent être reliées à d'autres villes.
- À chaque tour, un budget de peinture de 3 permet de poser jusqu'à 3 rails (plaine 1, rivière 2, montagne 3). Le budget est remis à niveau à chaque tour, pas cumulé : la peinture non dépensée est perdue.
- Une fois par tour, un joueur peut aussi perturber une zone. Chaque perturbation augmente son instabilité. Au-delà d'un seuil, la zone est détruite : ses rails sont effacés et on ne peut plus rien y construire.
- Le revenu : pour chaque paire de villes qui veulent être reliées et qui le sont, le plus court chemin compte. Chaque joueur gagne, à chaque tour, le nombre de cases qu'il possède sur ces chemins.
- Trois conséquences : un long chemin que je possède rapporte plus qu'un court ; les rails servent aux deux joueurs, donc compléter une connexion dont le chemin appartient surtout à l'adversaire lui rapporte à lui ; et relier tôt vaut mieux que relier beaucoup.
- Les deux joueurs jouent en même temps, avec 50 ms par tour.

## Le bot final en bref

À chaque tour :

1. Deux réseaux de neurones de mon architecture S3z lisent la grille. Ils ont été entraînés séparément, et ils jouent différemment.
2. Leurs notes pour chaque case, pour poser un rail et pour perturber, sont moyennées.
3. Le bot transforme ces notes en un tour complet : jusqu'à 3 rails dans le budget de peinture, et la zone à perturber.

Il n'y a pas de recherche pendant la partie : toutes les tentatives de reclasser les coups du réseau ont perdu (voir plus bas). Les poids sont codés sur 7 bits, pour que les deux réseaux tiennent dans un seul fichier C++ sous la limite de 100 000 caractères.

## Chronologie

| Version | Date | Idée principale | Ligue |
|---|---|---|---|
| 1 | 8 et 9 septembre | Beam search et perturbation, sur un simulateur exact | Wood → Bronze |
| 2 | Du 10 au 16 septembre | Beam search v4 : corridors, bonus de corridor, réglage anti-perturbation | Bronze → Silver |
| 3 | 19 et 20 septembre | S3z, mon propre réseau d'imitation | Gold |
| 4 | 20 septembre | Ensemble de deux réseaux S3z | Gold → Legend (version finale) |

## Les versions du bot

### Version 1 : beam search et perturbation

**La base : un simulateur exact.** Un modèle C++ d'un tour complet (revenu, rails, collisions, perturbations, destructions, connexions), vérifié contre l'arbitre officiel : 1 500 parties aléatoires, 150 000 tours comparés, 0 écart.

**La beam search.**
- Le bot construit son tour rail par rail, et garde les meilleurs plans partiels à chaque étape.
- Les candidats sont les cases des chemins comptés des connexions les plus rentables. Les borner (8 connexions, 64 cases) a fait passer la décision la plus lente de 718 ms à 42 ms.
- Le bot construit exactement le chemin que compte l'arbitre : le plus court chemin, départagé dans l'ordre de l'arbitre (nord, est, sud, ouest). Tous mes rails sont alors sur des chemins comptés. Contre mon bot glouton (qui perturbe aussi), le taux de victoire est passé de 64,5 % à 74,5 %.

**La perturbation par revenu nié.** Le bot perturbe la zone dont la destruction retirerait le plus de revenu à l'adversaire, en ne comptant que les connexions qui rapportent vraiment, et en favorisant les zones proches de la destruction. Contre le même bot glouton : 72,5 % → 78,0 % (200 parties).

**La méthode.** Une seule modification à la fois, chacune avec un verdict GO / NO-GO : on ne la garde que si le taux de victoire monte, que la décision la plus lente reste sous 47 ms, qu'il n'y a ni plantage ni action illégale, et que le simulateur reste exact.

### Version 2 : beam search v4, corridors et réglage anti-perturbation

- **Les corridors :** la beam search considère aussi des routes proches du plus court chemin, en tenant compte du propriétaire de chaque case.
- **Le bonus de corridor :** une case utilisée par beaucoup de corridors voulus reçoit un bonus, aussi bien pour classer les connexions que pour classer les cases candidates.
- **Le garde-fou de temps :** chaque calcul est borné à partir du début du tour, car l'arbitre disqualifie un bot qui dépasse 50 ms.
- **Le réglage anti-perturbation :** trois constantes changées, sans nouveau code. Le bot évite de construire dans les zones dont l'instabilité monte, et donne moins de poids au gain immédiat. Il a battu la version précédente dans 61,3 % de 390 parties, sur des cartes mises de côté.

**Le plafond.** Ensuite, 11 autres idées ajoutées à cette beam search (routage, corridors, zones disputées, poids de perturbation, jeu en tête ou en retard…) ont toutes été rejetées ou à égalité. La recherche avait atteint son plafond.

### Version 3 : S3z, mon propre réseau de neurones

**La conception.**
- Mesure décisive : le réseau prend 97 % du temps en C++. Les caractéristiques d'entrée ne coûtent rien, c'est le nombre d'opérations qui est la vraie contrainte.
- L'architecture S3z : des convolutions séparables dilatées, plus un bloc qui ajoute un contexte au niveau des zones (« ZoneContext »). 58 693 paramètres.
- ZoneContext était prévu « presque gratuit », à 0,05 ms, mais mesuré à 0,59 ms par bloc : une erreur d'un facteur 20, car il est limité par les accès mémoire, pas par le calcul. Il vaut quand même environ 8 points de taux de victoire, mesuré contre le même réseau sans lui (sur un seul entraînement).

**Mon propre simulateur, en Python.** Il est écrit à partir de la spécification de l'arbitre et vérifié par deux références indépendantes : un test différentiel contre l'arbitre Java officiel, compilé sans modification, et 14 000 vraies parties rejouées. Il a attrapé un vrai bug : le `String.split` de Java supprime les chaînes vides finales, pas celui de Python, et un bot termine sa sortie par un `;`. Aucun test synthétique ne l'aurait trouvé.

**L'entraînement.** Par imitation (behavior cloning) des coups des meilleurs joueurs, à partir de parties téléchargées sur CodinGame. Deux entraînements indépendants sont aussi forts l'un que l'autre, mais jouent différemment : ils ne sont d'accord que sur 22,9 % de leurs commandes. C'est la base de la version 4.

**En C++.** 86 795 caractères. Sous 71 250 paramètres, les nombres à virgule sur 16 bits suffisent : pas besoin de quantification. Les décisions C++ sont identiques à celles de Python sur 100 % de 2 000 positions, 2 958 tours et 48 parties complètes en 32 bits, et sur 99,66 % en 16 bits, l'écart venant uniquement des arrondis.

### Version 4 : ensemble de deux réseaux S3z (version finale)

- Les sorties des deux réseaux sont moyennées. Contre un seul S3z, l'ensemble gagne 54,6 % de 2 000 parties (intervalle de confiance à 95 % : de 52,4 à 56,7 %).
- Moyenner deux époques du même entraînement ne donne rien (44,75 %) : le gain vient de la diversité.
- Codé sur 7 bits, l'ensemble reste nettement meilleur qu'un seul S3z (test de McNemar, p = 0,003), et la quantification ne coûte rien de mesurable (p = 0,91 sur 2 500 graines).
- Le premier ensemble sur 7 bits tenait en 87 415 caractères, avec des décisions identiques à Python sur 2 958 commandes sur 2 958 et 48 parties sur 48, et prenait 31 ms par tour sur 50.
- **Résultat :** passage en Legend.

## Ce qui n'a pas marché

Côté recherche (beam search) :
- **Prédire l'adversaire dans la beam search (un tour) :** −12 à −14 points, et hors du budget de temps. Le bot est devenu trop timide et abandonnait des connexions qu'il aurait gagnées.
- **Construire un autre chemin de même longueur qui réutilise mes rails :** −4,5 points. L'arbitre continue de compter son propre chemin, donc mes rails « optimisés » en sortaient.
- **Un bonus explicite pour les cases « colonne vertébrale » partagées par plusieurs connexions :** environ 50 %, aucun signal. La beam search les récompense déjà.
- **Plus de temps de recherche :** un budget de 40 ms faisait moins bien. En moyenne, la beam search avait déjà exploré tous ses candidats en 4,3 ms.

Côté réseau (une session de recherche de 26 expériences) :
- **Reclasser les coups du réseau :**
  - selon le coût pour compléter les connexions : mauvais objectif, car le revenu vient de posséder des cases, pas de relier ;
  - selon le vrai revenu : cette optimisation à courte vue perd 25 points ;
  - selon la sortie « valeur » du réseau : les valeurs varient 7 fois moins entre les coups d'un même tour qu'entre les tours, donc la valeur ne sait pas départager ces coups.
- **Ne pas dépenser la peinture quand le réseau hésite :** la peinture non dépensée est perdue.
- **Un réseau grille + graphe (S4) :** 46,8 % contre S3z. Le bloc graphe n'apporte rien.
- **La moyenne des poids (SWA) :** mauvais protocole (taux d'apprentissage qui finit au dixième de sa valeur).
- **Distiller l'ensemble en un seul réseau :** avec des cibles dures, il mémorise une cible étroite et généralise mal ; avec des cibles douces, l'erreur sur 600 sorties est dominée par le bruit.
- **Au total, huit tentatives d'entraîner un seul réseau meilleur que S3z ont toutes échoué.**

## Où est la limite

- **Pour la recherche,** la limite n'était pas la vitesse : plus de temps n'apportait rien, et tous les réglages étaient épuisés. La seule marge restante était d'anticiper l'adversaire, pour un gros effort et un gain incertain.
- **Pour le réseau,** j'ai reconstruit exactement 77 vraies parties CodinGame de S3z (77 scores officiels reproduits sur 77) :
  - il gagnait 40 % de ses parties contre les 30 premiers de Gold, 70 % contre le milieu de Gold, et 88 % contre le bas ;
  - dans mes défaites, l'adversaire ne marque pas plus : c'est moi qui marque deux fois moins ;
  - le signe le plus net est l'écart de revenu au tour 15 : −5,6 dans les défaites, +4,6 dans les victoires (p = 0,001).
- **L'étape suivante** serait de classer entre eux les coups d'un même tour. Il faudrait pour cela des exemples du meilleur coup, donc des suites de partie simulées (rollouts).

## Les outils

- **Un simulateur C++ exact d'un tour,** vérifié contre l'arbitre officiel : 150 000 tours, 0 écart.
- **Un simulateur Python,** vérifié par deux références indépendantes : l'arbitre officiel et 14 000 vraies parties.
- **Une arène locale** avec des graines communes, les deux côtés joués, et un protocole GO / NO-GO strict.
- **Un plan de graines cloisonné** (tri, deuxième niveau, confirmation, adversaires extérieurs) : aucune graine ayant servi à choisir n'a servi à conclure.
- **Des tests de parité Python ↔ C++,** décision par décision.
- **La reconstruction exacte des vraies parties CodinGame :** la carte ne peut pas être régénérée depuis la graine, elle est donc lue dans le replay.
- **Le volume :** environ 45 000 parties pendant la dernière session de recherche.

## Ce que j'ai appris

- **Comprendre la règle de score avant d'optimiser quoi que ce soit.** Un long chemin que je possède rapporte plus, et la peinture non dépensée est perdue : ces deux faits ont conditionné tout le reste.
- **Reproduire l'arbitre exactement.** Son ordre de départage est la cible, pas un détail.
- **Ne jamais conclure sous 2 000 graines, et toujours comparer sur les mêmes graines.** Un tri à 100 graines s'est trompé sept fois.
- **Mesurer au lieu de prévoir.** ZoneContext a coûté 20 fois plus que prévu, et c'est le réseau, pas les caractéristiques, qui prenait 97 % du temps.
- **Ce qui doit être identique entre Python et C++, c'est la décision, pas les bits :** les bits changent avec le nombre de threads.
- **Deux réseaux aussi forts l'un que l'autre, mais qui jouent différemment, valent plus ensemble** que n'importe quel réseau seul que j'ai pu entraîner.
