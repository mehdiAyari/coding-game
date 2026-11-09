# Troll Farm — CodinGame Spring Challenge 2026

[English](README.md) | **Français**

| | |
|---|---|
| **Concours** | [CodinGame Spring Challenge 2026](https://www.codingame.com/contests/spring-challenge-2026-troll-farm), du 12 au 25 mai 2026 |
| **Jeu** | Troll Farm : deux joueurs dirigent des trolls qui récoltent, coupent et minent sur une grille, tours simultanés |
| **Résultat** | Ligue Legend, la ligue la plus haute |
| **Bot final** | Des règles écrites à la main et des estimations économiques, en quatre phases, avec deux trolls |
| **Technologies** | Java |

## Le jeu

- Chaque joueur commence avec une cabane et un troll, sur une grille d'herbe, d'eau, de rochers, de fer et d'arbres (8 à 11 lignes, deux fois plus de colonnes).
- Les trolls récoltent les fruits des arbres, coupent les arbres pour le bois, minent le fer, et rapportent tout à leur cabane.
- Les arbres (prune, citron, pomme, banane) grandissent jusqu'à la taille 4, puis donnent jusqu'à 3 fruits. Ils poussent plus vite près de l'eau. Un troll peut planter un arbre avec un fruit qu'il porte.
- Les ressources servent à entraîner de nouveaux trolls. Chacun des 4 attributs (vitesse, capacité de transport, puissance de récolte, puissance de coupe) se paie avec un fruit donné ou du fer, et son coût vaut le nombre de trolls plus l'attribut au carré.
- Le score : chaque fruit dans la cabane vaut 1 point, chaque bois 4 points. Le fer ne vaut rien.
- Les deux joueurs jouent en même temps, avec 50 ms par tour, pendant au plus 300 tours.

## Le bot final en bref

Le bot avance en quatre phases (détaillées dans la version 4 plus bas) :

1. **Un seul troll :** il calcule quel deuxième troll rapportera le plus de bois, et réunit ce qu'il coûte. Il joue toute la partie avec ces deux trolls.
2. **Deux trolls :** le bois vaut 4 fois un fruit, donc ils coupent, en commençant par les citronniers proches de la cabane adverse, puis les autres arbres par une affectation globale.
3. **Plus d'arbre :** le troll lent transforme les fruits de la cabane en bois (1 point devient 4), pendant que le troll rapide part vers la cabane adverse.
4. **Plus de fruit :** les deux trolls restent à côté de la cabane adverse et coupent tout ce que l'adversaire plante.

Il n'y a ni recherche ni simulation de toute la partie : chaque décision vient de règles écrites à la main et d'estimations.

## Chronologie

| Version | Date | Idée principale | Ligue |
|---|---|---|---|
| 1 | 12 et 13 mai | Des règles simples : récolter, rapporter, puis couper | Wood → Bronze |
| 2 | 14 et 15 mai | Des choix par points par tour, une ferme à bananes à côté de la cabane | Bronze → Silver |
| 3 | 17 mai | « Couper d'abord » par phases, pousse des arbres et courses simulées | Silver → Gold |
| 4 | 21 et 22 mai | Quatre phases : le deuxième troll choisi selon le bois attendu, les citronniers adverses d'abord | Gold → Legend (version finale) |

## Les versions du bot

### Version 1 : des règles simples

- **Le premier bot (fruits seulement, dans la première ligue) :** déposer les fruits quand il est à côté de la cabane ; récolter quand il est sur un arbre avec des fruits et qu'il a de la place ; sinon rentrer à la cabane s'il porte des fruits, ou aller vers l'arbre le plus proche qui a des fruits.
- **Ensuite, avec les règles complètes :** entraîner un troll, miner, couper. Les arbres sont notés en points par tour passé.
- **Des cibles collantes :** chaque troll garde sa cible d'un tour à l'autre, et n'en change que si une nouvelle cible est au moins 30 % meilleure. Ça évite les zigzags.

### Version 2 : des choix par points par tour

- **Une ferme à bananes à côté de la cabane :** le troll le plus lent prend une banane à la cabane, la plante sur une case voisine de la cabane, coupe l'arbre et dépose le bois. La banane a le cycle le plus court : avec 3 points de vie, l'arbre est coupé avant de grandir.
- **La case de ferme :** pas à côté de l'eau si possible (l'arbre pousserait plus vite et allongerait le cycle), et pas sur un axe de passage. Le bot simule un arbre planté sur chaque case candidate et mesure de combien il allonge les trajets vers la cabane.
- **Les points par tour :** le troll lent compare trois options selon leur rendement : la ferme, couper un arbre de la carte, ou en récolter un. Le vrai coût d'un cycle de ferme est simulé tour par tour, car un arbre peut regrandir pendant qu'on le coupe.
- **Mon propre calcul de chemin :** un BFS qui contourne les arbres et les autres trolls, écrit après qu'un troll est resté bloqué environ 45 tours.
- **Plus de conflits :** les trolls réservent leurs cibles, donc un troll ne peut plus prendre l'arbre qu'un autre est déjà en train de couper. Et si le fermier bloque un autre troll, il s'écarte.

### Version 3 : « couper d'abord », par phases

- **L'entraînement :** le bot vise d'abord un troll minimum (vitesse 2, capacité 2, coupe 2, pas de récolte), puis la meilleure combinaison qu'il peut payer tout de suite ou après une courte attente. Le temps d'attente est fixé par la ressource la plus lente à réunir, puisque les autres se réunissent pendant ce temps. Une fois choisie, la cible est verrouillée, sauf si une cible strictement plus forte devient possible.
- **Des arbres de rente :** des pruniers et des citronniers plantés près de la cabane donnent des fruits en arrière-plan, sauf si un arbre de ce type pousse déjà tout près.
- **Couper toute la carte :** chaque arbre est noté en bois par tour. Le bot simule sa pousse pendant le trajet et pendant la coupe, et les trolls adverses déjà en train de le couper. Un arbre que l'adversaire finira avant nous est écarté.
- **Une affectation globale** des arbres aux trolls (la meilleure paire troll–arbre d'abord, puis la suivante), avec un bonus pour la cible précédente, au lieu que chaque troll choisisse seul.
- **Les fruits en bois** quand il ne reste plus d'arbre : chaque troll plante des fruits à côté de la cabane et les coupe. Puis, quand il n'y a plus de fruit, les deux trolls se postent à côté de la cabane adverse et coupent ce que l'adversaire plante.

### Version 4 : quatre phases, les citronniers adverses d'abord (version finale)

**Phase 1 — un seul troll : choisir le deuxième troll, puis récolter.**
- **Quel troll entraîner.** Le bot essaie tous les trolls de vitesse 2, de capacité 2 à 5, de coupe 1 à 5, sans récolte. Avec un seul troll dans l'équipe, chaque attribut coûte 1 plus sa valeur au carré : des prunes pour la vitesse, des citrons pour la capacité, du fer pour la coupe, et une pomme pour une récolte à 0.
- **Combien de temps attendre.** Pour chaque ressource qui manque, le bot estime le temps pour la réunir, et la plus lente fixe le temps d'attente. Le fer : un aller-retour jusqu'au fer le plus proche par unité. Un fruit : le meilleur entre aller le chercher sur l'arbre le plus proche de ce type (un aller-retour, ou le temps de repousse de l'arbre s'il est plus long) et compter sur un arbre près de la cabane (un arbre existant à 3 pas au plus, ou un arbre à planter à côté). Un troll qui demande plus de 5 unités supplémentaires d'une même ressource est écarté.
- **La note :** le bois que ce troll rapporterait, c'est-à-dire son bois par tour (aller-retour à sa vitesse sur une distance moyenne, temps de coupe, bois porté) multiplié par les tours qui restent avant le tour 100 une fois l'attente finie.
- **Les cabanes proches :** un troll qui ferait attendre plus de 25 tours est écarté, et plus de 2 tours quand les cabanes sont à 2 pas ou moins. Quand elles sont à 8 pas ou moins, l'horizon est raccourci jusqu'à 30 %.
- **Une cible verrouillée :** le troll choisi ne change que pour un troll strictement meilleur, ou quand il n'est plus atteignable. Si aucun troll ne convient, le bot se rabat sur le minimum : vitesse 2, capacité 2, pas de récolte, coupe 1.
- **La récolte :** le troll compare miner et récolter selon les unités utiles rapportées par tour d'aller-retour, pondérées par la part du coût qui manque encore. Il garde sa cible sauf si une autre est 30 % meilleure. Quand rien ne vaut la peine, il marche vers l'arbre utile le plus proche qui portera bientôt des fruits.
- **Les arbres de rente :** s'il a des prunes ou des citrons en plus du coût de l'entraînement, le troll plante d'abord un prunier, puis un citronnier, à 1 ou 2 pas de la cabane (près de l'eau si possible, pour qu'il produise plus vite), sauf si un arbre de ce type pousse déjà à 3 pas ou moins. Une fois ce point réglé, s'il manque encore du fer, la récolte est fortement pénalisée : le fer d'abord.

**Phase 2 — deux trolls : couper, les citronniers adverses d'abord.**
- **Les citronniers de l'adversaire.** Beaucoup d'adversaires récoltaient des citrons pour entraîner un troll à grosse capacité de transport, et je perdais beaucoup de parties contre eux. Le bot classe donc les citronniers à 6 cases ou moins de la cabane adverse selon leur valeur pour l'adversaire (les fruits actuels, plus leur production sur 30 tours) multipliée par leur proximité de sa cabane. Chaque arbre reçoit le troll libre le plus proche : un troll par arbre, donc les deux trolls y vont quand il y a deux citronniers de ce genre.
- **Les autres arbres** sont répartis par l'affectation globale : le bois par tour pour chaque paire troll–arbre (trajet, coupe, retour), en simulant la pousse de l'arbre et les trolls adverses déjà en train de le couper. Un arbre que l'adversaire finira avant nous est écarté. La meilleure paire passe d'abord, la cible précédente reçoit un bonus ×1,5, et en cas d'égalité l'arbre le plus proche de la cabane adverse l'emporte.
- **Pas de blocage mutuel :** le troll rapide décide en premier. Les déplacements du troll lent passent par mon propre BFS, autour de la prochaine position du troll rapide plutôt que de sa position actuelle.

**Phase 3 — plus d'arbre : les fruits en bois.**
- Le troll lent transforme les fruits de la cabane : il en prend un, la banane d'abord, puis la prune, le citron et la pomme (les arbres les plus rapides à couper d'abord), le plante sur une case à côté de la cabane, coupe l'arbre et rapporte le bois. Un fruit qui vaut 1 point devient du bois qui en vaut 4.
- Pendant ce temps, le troll rapide part déjà vers la cabane adverse (phase 4).

**Phase 4 — plus de fruit : à la cabane adverse.**
- Les deux trolls coupent l'arbre le plus proche qu'ils peuvent atteindre avant que l'adversaire ne le finisse (à ce stade, les arbres que l'adversaire plante), et sinon attendent sur une case à côté de la cabane adverse.

**Résultat :** passage en Legend.

## Ce qui n'a pas marché

- **Cultiver les fruits lents à côté de la cabane** (prune, citron, pomme) : l'arbre regrandit plus vite qu'on ne le coupe (une pomme a 11 points de vie et un délai de 9 tours), il reste sur la case de ferme et bloque le passage, et une ferme lente laisse à l'adversaire le temps de marquer. Seule la banane a été gardée.
- **Choisir la case de ferme selon sa distance au bord :** elle tombait sur des couloirs critiques, une fois même sur l'unique passage vers la cabane. Remplacé par la mesure de combien un arbre à cet endroit allonge les trajets.
- **Un seuil de rendement minimum :** il figeait le troll lent en fin de partie, environ 14 tours à l'arrêt pendant que l'adversaire continuait de marquer. Désormais, le troll agit dès qu'une option rapporte.
- **Chaque troll qui choisit son arbre seul :** un troll pouvait prendre l'arbre qu'un autre était déjà en train de couper. Remplacé par les réservations, puis par l'affectation globale.

## Où est la limite

Il ne reste aucune mesure de ce concours, seulement le code soumis. Ce que montre le code :
- **Ni simulation de toute la partie ni recherche :** chaque décision vient de règles et d'estimations (par exemple les points de vie et la taille moyens des arbres). L'adversaire n'est vu qu'à travers des signaux simples : ses trolls déjà en train de couper un arbre, et l'emplacement de sa cabane.
- **Les trolls décident l'un après l'autre** (le rapide, puis le lent), avec une affectation globale seulement pour les arbres.
- **Deux trolls pour toute la partie :** le bot n'entraîne qu'un seul troll de plus.
- **L'étape suivante naturelle** serait un simulateur exact du jeu, pour tester ces règles sur beaucoup de parties et chercher les actions conjointes des trolls.

## Les outils

- **Un rapport de tour dans la sortie de débogage :** inventaires, phase, la décision de chaque troll et ses meilleurs candidats.
- **Des contrôles de prédiction :** le bot prédisait son inventaire du tour suivant et le comparait au vrai.
- **La détection de blocage :** un troll qui voulait bouger mais n'a pas bougé est repéré au tour suivant.
- **Des distances par BFS** sur la vraie carte, en contournant l'eau, les rochers et le fer, au lieu d'estimations à vol d'oiseau.

## Ce que j'ai appris

- **Le bois vaut 4 fois un fruit :** le meilleur usage d'un fruit est souvent de le planter et de couper l'arbre.
- **Simuler le temps, pas seulement les distances :** un arbre pousse pendant le trajet et pendant la coupe, et une pomme peut regrandir plus vite qu'on ne la coupe.
- **Coordonner les unités :** réserver les cibles et les répartir globalement empêche les trolls de se voler le travail.
- **Un seuil trop strict peut figer une unité :** agir un peu vaut mieux qu'attendre.
- **Garder sa cible sauf si une nouvelle est nettement meilleure** (30 % ici) : ça évite les zigzags.
