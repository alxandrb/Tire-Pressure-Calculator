# Tire Pressure Calculator

Calculateur de pression de pneus pour vélo de contre-la-montre et de triathlon. Une page HTML autonome, sans dépendance, sans build, sans serveur. On saisit la charge, le montage et le parcours, on obtient la valeur à afficher au manomètre pour la roue avant et pour la roue arrière, avec le détail de chaque correction appliquée.

L'app ne se contente pas de sortir un chiffre : elle montre la courbe de coût énergétique autour de l'optimum, les limites matérielles qui plafonnent le résultat, et le tableau des coefficients qui ont mené à la valeur finale.

---

## Sommaire

- [Démarrage](#démarrage)
- [Pourquoi ce calcul n'est pas trivial](#pourquoi-ce-calcul-nest-pas-trivial)
- [Le modèle, étape par étape](#le-modèle-étape-par-étape)
  - [1. Base d'affaissement](#1-base-daffaissement)
  - [2. Largeur réelle du pneu monté](#2-largeur-réelle-du-pneu-monté)
  - [3. Coefficients de correction](#3-coefficients-de-correction)
  - [4. Limites matérielles](#4-limites-matérielles)
  - [5. Correction thermique et barométrique](#5-correction-thermique-et-barométrique)
  - [6. Courbe de coût énergétique](#6-courbe-de-coût-énergétique)
- [Tous les paramètres](#tous-les-paramètres)
- [Exemple chiffré complet](#exemple-chiffré-complet)
- [Ce que le modèle ne fait pas](#ce-que-le-modèle-ne-fait-pas)
- [Recalibrer le modèle](#recalibrer-le-modèle)
- [Structure du code](#structure-du-code)
- [Fondements](#fondements)
- [Licence](#licence)

---

## Démarrage

Aucune installation, aucune dépendance, aucun `npm install`.

```bash
git clone https://github.com/<utilisateur>/tire-pressure-calculator.git
cd tire-pressure-calculator
open index.html
```

Pour publier sur GitHub Pages : `Settings → Pages → Source: deploy from a branch → main / (root)`. Le fichier doit s'appeler `index.html` à la racine, sinon rien n'est servi.

La seule ressource externe est la fonte Archivo, chargée depuis Google Fonts. Hors ligne, la page bascule sur la pile système et reste parfaitement utilisable. Pour une autonomie totale, supprimez les deux `<link rel="preconnect">` et le `<link>` de la fonte dans le `<head>`.

Testé sur Chrome, Firefox et Safari récents. Aucun `localStorage`, aucune requête réseau, aucune donnée ne quitte le navigateur.

---

## Pourquoi ce calcul n'est pas trivial

L'intuition dit qu'un pneu plus dur roule plus vite. C'est vrai sur un rouleau d'acier poli, et faux sur une route.

Deux pertes s'opposent :

**L'hystérésis de carcasse.** À chaque tour de roue, la gomme et la toile se déforment au contact du sol puis reprennent leur forme. Ce cycle dissipe de l'énergie en chaleur. Plus la pression est élevée, moins la carcasse se déforme, moins on perd. Cette perte décroît donc quand la pression monte.

**L'impédance.** Une aspérité de chaussée doit être absorbée par quelque chose. Tant que le pneu est assez souple, c'est lui qui l'encaisse. Passé un seuil, il devient trop rigide pour se déformer localement : la roue, le cadre, puis le corps du cycliste sont mis en mouvement vertical, et cette énergie est dissipée dans les tissus mous, la suspension musculaire et les frottements. Cette perte croît quand la pression monte, et elle croît d'autant plus vite que le revêtement est grossier et que la vitesse est élevée.

La somme des deux forme une courbe en U. Le minimum n'est pas au bout de la plage, il est quelque part au milieu, et sa position dépend de la charge, de la largeur du pneu, du revêtement et de la vitesse. C'est ce minimum que l'app cherche.

Le creux est aussi asymétrique : sur revêtement dégradé, dépasser l'optimum de 10 psi coûte nettement plus cher que de rester 10 psi en dessous. La courbe affichée rend cette asymétrie visible, et c'est souvent l'information la plus utile de la page.

---

## Le modèle, étape par étape

### 1. Base d'affaissement

Le point de départ est l'affaissement de carcasse : de combien le pneu s'écrase verticalement sous la charge, exprimé en pourcentage de sa hauteur. La valeur de référence retenue est **15 %**, consensus établi par les campagnes de mesure de charge/pression publiées de longue date : en dessous, le pneu devient inconfortable et pénalisé par l'impédance ; au-dessus, le risque de pincement et de déjantage grimpe et le rendement chute.

La pression qui produit 15 % d'affaissement est approchée par :

```
P₀ = 266 × L / w^1.5
```

- `P₀` en psi
- `L` charge sur la roue, en kg
- `w` largeur réelle du pneu monté, en mm

L'exposant 1,5 traduit le fait que la surface de contact croît à la fois en longueur et en largeur quand le pneu s'écrase. Le coefficient 266 est un calage sur les tables de charge publiées : la formule reproduit leurs valeurs à quelques psi près entre 20 et 32 mm, qui est la plage utile ici.

Le calcul est fait **séparément pour chaque roue**, puisque la charge n'y est pas la même.

### 2. Largeur réelle du pneu monté

La largeur inscrite sur le flanc suppose une jante de référence. Montée sur une jante plus large, la carcasse s'étale et le pneu gagne en section. Comme la largeur entre au carré et demi dans la formule, ignorer ce point fausse le résultat de plusieurs psi.

```
w_réelle = w_nominale + 0.4 × (largeur_interne − 19)
```

Règle empirique classique : environ 0,4 mm de section gagnée par millimètre de jante au-delà de 19 mm interne. Un 26 mm sur une jante à 21 mm interne se monte donc à environ 26,8 mm. Le résultat est borné à 90 % de la largeur nominale pour éviter les aberrations sur jante très étroite.

L'app signale l'écart dès qu'il dépasse 1,5 mm, pour que la valeur utilisée soit explicite.

### 3. Coefficients de correction

`P₀` est ensuite multiplié par une série de coefficients indépendants. Tous sont visibles dans le tableau « Détail du calcul » de l'interface.

**Montage**

| Montage | Coefficient | Raison |
|---|---|---|
| Tubeless | ×0,95 | Aucun risque de pincement de chambre, pas de frottement chambre/carcasse |
| Chambre latex | ×0,99 | Chambre très déformable, hystérésis faible |
| Chambre TPU | ×1,00 | Référence |
| Chambre butyle | ×1,00 | Référence |
| Boyau | ×0,97 | Carcasse cousue, déformation plus libre |

**Carcasse**

| Carcasse | Coefficient | Raison |
|---|---|---|
| Souple, compétition | ×0,97 | Épouse mieux les aspérités, le seuil d'impédance arrive plus bas |
| Polyvalente | ×1,00 | Référence |
| Renforcée anti-crevaison | ×1,03 | Ceinture de protection rigide, il faut plus de pression pour un affaissement donné utile |

**Revêtement** — le levier le plus puissant du modèle.

| Revêtement | Coefficient | Exposant d'impédance |
|---|---|---|
| Piste ou béton lissé | ×1,12 | 1,5 |
| Asphalte neuf, très lisse | ×1,05 | 1,8 |
| Asphalte courant | ×1,00 | 2,2 |
| Enduit gravillonné | ×0,92 | 2,8 |
| Chaussée dégradée, fissurée | ×0,85 | 3,4 |
| Très mauvais, nids-de-poule | ×0,78 | 4,2 |

L'exposant n'agit pas sur la pression recommandée : il gouverne la **raideur de la branche droite de la courbe de pertes**, donc le coût d'une surpression. Plus le revêtement est grossier, plus la pénalité est brutale.

**Vitesse**

```
coefficient = 1 − 0.004 × (v − 32)     borné à [0.90, 1.06]
```

L'énergie d'un choc contre une aspérité croît avec le carré de la vitesse. Le seuil d'impédance se déplace donc vers le bas quand on roule vite : plus on est rapide, moins il faut gonfler. À 45 km/h le coefficient vaut 0,948, à 25 km/h il vaut 1,028.

**Durée, technicité, pluie, arbitrage**

| Paramètre | Coefficient |
|---|---|
| Moins de 1 h 30 | ×1,00 |
| 1 h 30 à 4 h | ×0,99 |
| Plus de 4 h | ×0,97 |
| Peu de virages | ×1,00 |
| Virages moyens | ×0,99 |
| Très sinueux, descentes techniques | ×0,97 |
| Chaussée mouillée | ×0,97 |
| Curseur d'arbitrage | ×0,94 à ×1,04 |

La durée agit sur la fatigue accumulée par les vibrations, qui ne se paie pas sur 40 km mais se paie sur 180. La technicité et la pluie agissent sur la surface de contact disponible pour l'adhérence. Le curseur d'arbitrage est asymétrique volontairement : il descend plus bas (−6 %) qu'il ne monte haut (+4 %), parce que l'erreur par excès est la plus coûteuse.

La pression cible est le produit :

```
P_cible = P₀ × montage × carcasse × revêtement × vitesse × durée × technicité × pluie × arbitrage
```

### 4. Limites matérielles

La cible est ensuite bornée.

**Plancher** — seuil de pincement, fonction du montage : 0,70 × P₀ en tubeless, 0,72 en boyau, 0,78 en chambre latex ou TPU, 0,80 en chambre butyle. Une chambre butyle se pince bien plus tôt qu'un tubeless avec préventif, la marge de sécurité n'est donc pas la même.

**Plafond**, le plus contraignant des trois :

- 1,45 × P₀, au-delà duquel l'affaissement devient trop faible pour que le modèle reste valide ;
- la pression maximale annoncée, si elle est saisie ;
- **72,5 psi (5,0 bar) sur jante sans crochet**, limite normative ETRTO, absolue et non négociable quelle que soit l'indication portée sur le flanc du pneu.

Quand une limite mord, l'app le dit explicitement plutôt que de renvoyer silencieusement un chiffre tronqué.

L'affaissement réellement obtenu est recalculé et affiché : `affaissement = 15 % × P₀ / P_finale`. C'est le meilleur indicateur de la validité du résultat. Au-delà de 20 %, on est dans la zone de pincement ; en dessous de 12 %, la carcasse ne travaille plus.

### 5. Correction thermique et barométrique

Un pneu gonflé à 18 °C dans un garage et roulé à 32 °C en plein soleil gagne de la pression tout seul, loi des gaz parfaits oblige. Un gonflage au niveau de la mer pour une course en altitude en gagne aussi, la pression relative lue au manomètre montant quand la pression atmosphérique baisse.

L'app renvoie donc **la valeur à afficher au manomètre au moment du gonflage**, pas la valeur cible en course :

```
P_manomètre = (P_cible + P_atm_course) × (T_gonflage + 273.15) / (T_course + 273.15) − P_atm_gonflage
```

avec la pression atmosphérique donnée par le modèle d'atmosphère standard :

```
P_atm(altitude) = 14.6959 × (1 − 2.25577e-5 × altitude)^5.25588     en psi
```

L'écart est signalé dès qu'il dépasse 1,5 psi, avec son sens.

### 6. Courbe de coût énergétique

La courbe affichée est la somme des deux pertes décrites plus haut, construite pour que son minimum tombe exactement sur la pression retenue :

```
perte(P) = A × (P_opt / P) + B × (P / P_opt)^b
```

- terme de gauche : hystérésis, décroissante en 1/P
- terme de droite : impédance, croissante en puissance `b`, l'exposant donné par le revêtement
- condition de minimum en `P_opt` : `B = A / b`

L'échelle verticale est ancrée sur une puissance de roulement plausible :

```
P_roulement = Crr × masse_totale × 9.81 × vitesse
```

où `Crr` dépend de la carcasse (0,0032 souple, 0,0042 polyvalente, 0,0056 renforcée) ajusté du montage (−0,0004 en tubeless, +0,0006 en chambre butyle), avec un plancher à 0,0022.

**Ce qui est fiable dans ce graphique, c'est la position du creux et l'asymétrie de la courbe. La valeur absolue en watts est une estimation d'ordre de grandeur, pas une mesure.** La sensibilité affichée sous le graphique (coût d'un écart de 10 psi de part et d'autre) est l'information à retenir.

---

## Tous les paramètres

**Charge**
- Poids du cycliste, en kg
- Poids du vélo, en kg
- Équipement embarqué : bidons, nutrition, sacoche, en kg
- Position tenue : prolongateurs buste bas (46 % avant), prolongateurs position haute (44 %), bases ou cocottes (42 %), ou répartition mesurée au pèse-personne, réglable de 36 à 54 %

**Pneus et jantes**, avant et arrière indépendamment
- Largeur nominale, en mm
- Largeur interne de jante, en mm
- Montage : tubeless, chambre butyle, latex, TPU, boyau
- Carcasse : souple, polyvalente, renforcée
- Profil de jante : à crochets ou sans crochet
- Pression maximale annoncée, facultative

**Parcours**
- Revêtement dominant, six niveaux
- Vitesse moyenne visée
- Durée de l'effort
- Virages et relances
- Chaussée sèche ou mouillée

**Arbitrage**
- Curseur continu du confort et de l'adhérence vers le rendement pur

**Température et altitude**
- Température au gonflage et en course
- Altitude au gonflage et altitude moyenne du parcours

**Affichage**
- Bascule psi / bar, qui reformate valeurs, graphique et tableau

---

## Exemple chiffré complet

Cycliste 72 kg, vélo 9,5 kg, équipement 2 kg. Position prolongateurs haute. Pneus 26 mm tubeless sur jantes à crochets de 21 mm interne, carcasse polyvalente. Asphalte courant, 36 km/h, 1 h 30 à 4 h, virages moyens, sec. Arbitrage neutre. Gonflage à 20 °C, course à 24 °C, niveau de la mer.

| Étape | Avant | Arrière |
|---|---|---|
| Masse totale | 83,5 kg | 83,5 kg |
| Répartition | 44 % → 36,7 kg | 56 % → 46,8 kg |
| Largeur réelle | 26,8 mm | 26,8 mm |
| Base à 15 % d'affaissement | 70,4 psi | 89,7 psi |
| Correction cumulée | ×0,916 | ×0,916 |
| Cible en course | 64,5 psi | 82,1 psi |
| Correction thermique | −1,0 psi | −1,3 psi |
| **À régler au manomètre** | **63,5 psi / 4,38 bar** | **80,8 psi / 5,57 bar** |
| Affaissement obtenu | 16,4 % | 16,4 % |

Pertes estimées au creux : 31,1 W. Coût d'un écart de 10 psi : environ +0,5 W au-dessus, +0,6 W en dessous. Sur asphalte courant la courbe est plate, quelques psi ne changent presque rien. Le même calcul sur chaussée dégradée fait chuter les cibles d'environ 15 % et rend la branche droite trois fois plus raide.

Remarque sur cet exemple : avec des jantes sans crochet, l'arrière serait plafonné à 72,5 psi, soit un affaissement réel de 18,3 %. Le modèle le signale au lieu de l'ignorer.

---

## Ce que le modèle ne fait pas

Dit franchement, pour que personne ne prenne le résultat pour plus qu'il n'est.

- **Il ne connaît pas votre pneu.** Deux pneus de même largeur et de même TPI annoncé peuvent différer de 30 % en résistance au roulement. Le modèle raisonne par familles, pas par références.
- **Il ne mesure rien.** Il n'y a pas de capteur dans la boucle. Un test terrain sur portion répétée, chronomètre et capteur de puissance à l'appui, restera toujours supérieur à une estimation analytique.
- **Les watts sont relatifs.** Voir la section sur la courbe. Seules la position du creux et la forme comptent.
- **La répartition avant/arrière est déclarative.** Les trois préréglages sont des ordres de grandeur pour une position de contre-la-montre. La mesure réelle se fait avec un pèse-personne sous chaque roue, en position, et vaut tous les préréglages du monde.
- **L'aérodynamique est absente.** Le rapport entre largeur de pneu et largeur de jante a un effet aérodynamique réel, sans rapport avec la pression. Ce calculateur ne traite que le roulement.
- **Les suspensions ne sont pas modélisées.** Le modèle suppose un cadre rigide et un cycliste comme unique élément amortisseur.
- **Le préventif n'est pas compté** dans la masse, sa contribution est négligeable devant les incertitudes du reste.

Vérifiez toujours les pressions maximales gravées sur la jante et inscrites sur le flanc du pneu. En cas de conflit entre ce calculateur et un fabricant, le fabricant a raison.

---

## Recalibrer le modèle

Tous les coefficients sont regroupés en haut du bloc `<script>`, dans des objets nommés, sans logique autour. Une seule valeur à changer par ajustement.

```js
var MOUNT  = { tubeless:{p:0.95, crr:-0.0004, pinch:0.70, ...}, ... };
var CASING = { supple:{p:0.97, crr:0.0032, ...}, ... };
var SURF   = { avg:{p:1.00, b:2.2, ...}, ... };
var DUR    = { mid:{p:0.99, ...}, ... };
var TECH   = { mid:{p:0.99, ...}, ... };
```

- `p` : coefficient multiplicatif sur la pression cible
- `b` : exposant d'impédance, uniquement la raideur de la courbe à droite du creux
- `crr` : coefficient de roulement pour l'échelle verticale du graphique
- `pinch` : fraction de `P₀` sous laquelle on refuse de descendre

Pour recaler la base d'affaissement elle-même, la constante `266` est dans `wheelPressure()`. L'augmenter relève toutes les pressions proportionnellement. Pour viser 13 % d'affaissement plutôt que 15 %, multipliez-la par `15/13`.

Pour changer la sensibilité à la vitesse, la pente `0.004` est dans `compute()`.

Méthode de calage recommandée : partez d'une pression que vous savez bonne par expérience sur un parcours connu, saisissez les conditions exactes, et ajustez la constante de base jusqu'à retrouver cette valeur. Les coefficients relatifs, eux, sont plus robustes que la constante absolue.

---

## Structure du code

Un seul fichier, `index.html`, environ 600 lignes, sans dépendance.

```
<head>
  CSS complet : variables de couleur, grille, composants
<body>
  Bloc de lecture : deux valeurs, barres de plage, alertes
  Colonne gauche  : formulaire, cinq groupes de champs
  Colonne droite  : graphique SVG, tableau de détail
<script>
  Tables de coefficients      MOUNT, CASING, SURF, DUR, TECH
  atm(alt)                    pression atmosphérique standard
  wheelPressure()             base d'affaissement pour une roue
  compute()                   lit le formulaire, applique tout, renvoie un état
  renderWheel()               valeurs, unités, barre de plage
  renderChart()               génère le SVG de la courbe
  renderTable()               tableau du détail
  run()                       orchestration, appelé à chaque input
```

Aucun framework, aucun build, aucune étape de compilation. `compute()` est une fonction pure au sens pratique : elle lit le DOM et renvoie un objet, sans effet de bord. Toute la logique physique y est concentrée, ce qui la rend facile à extraire pour des tests.

Le SVG du graphique est généré en chaîne de caractères à chaque recalcul. À ce volume de points, c'est instantané et cela évite toute dépendance à une librairie de graphiques.

---

## Fondements

Le modèle s'appuie sur trois corpus publics, sans en reprendre aucun code ni aucune donnée propriétaire :

- les campagnes historiques de mesure charge/pression/affaissement qui ont établi la règle des 15 %, aujourd'hui la référence commune des tables de gonflage ;
- les travaux publiés sur l'impédance pneumatique et l'existence d'une pression seuil au-delà de laquelle les pertes par vibration dépassent le gain d'hystérésis, mesurés sur rouleaux texturés et confirmés sur route ;
- la norme ETRTO pour les limites de jante, en particulier le plafond de 5 bar sur jante sans crochet.

Les coefficients de correction sont des valeurs de calage cohérentes avec ces travaux, pas des constantes issues d'une mesure unique. Ils sont volontairement exposés et documentés pour pouvoir être discutés et corrigés.

---

## Licence

MIT.
