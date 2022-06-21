# Amélioration des vitesses dans Road2

_Road2_ est le moteur de calcul d'itinéraire développé par l'IGN utilisé par le service d'itinéraire et d'isochrone `V2` du Géoportail. Plutôt qu'un moteur en tant que tel, il s'agit plutôt d'un _proxy de moteurs_ dans la mesure où les calculs d'itinérraires et d'isochrones sont fait par d'autre moteurs open source, auxquels _Road2_ fait appel. En l'occurence, les moteurs actuellement implémentés sur la plateforme sont `OSRM`, un moteur extrêmeent performant au prix d'une configurabilité très limitée, et `pgRouting`, un moteur basé sur la technologie de base de données `PostgreSQL` qui permet à l'utilisateur faisant la requête de paramétrer cette dernière avec un haut niveau de personnalisation, au prix de la performance.

Les données utilisées pour générer les graphes routiers utilisés par les _backends_ de _Road2_ sont issues de la BDTopo, et plus précisément de 2 tables du thème transport : `troncon_de_route` et `non_communications`. Les vitesses utilisées dans le calcul d'itinéraire pour les voitures sont celles présent dans le champ `vitesse_moyenne_vl` de la table `troncon_de_route`.

## Situation actuelle des vitesses dans Road2

![Comparaison des itinéraires de Road2 et de différents services disponibles en milieu urbain](comparaison_iti_urbain.png)
![Comparaison des itinéraires de Road2 et de différents services disponibles en milieu non urbain](comparaison_iti_rural.png)

![Comparaison des isochrones de Road2 et de HERE en milieu urbain](iso_urbain.png)
![Comparaison des isochrones de Road2 et de HERE en milieu non urbain](iso_rural.png)

### Constat : les vitesses sont trop élevées en ville


### Origine du problème

#### Vitesses des tronçons ?

Au départ, le postulat était que les vitesses renseignées dans la BDTopo ne correspondaient pas à la réalité. En effet, le calcul de l'attribut `vitesse_moyenne_vl` suit des règles assez particulières qui ne semblent pas de prime abord pouvoir se conformer à chaque cas particulier, et à la vitesse moyenne constatée.

> Règles de calcul :
>
> Hors zone urbaine ('Urbain'="Faux") :
>
> Nature = ‘Type autoroutier’ avec classement admin. en autoroute & Importance = ‘1’ –> 125 km/h  
> Nature = ‘Type autoroutier’ avec classement admin. en autoroute & Importance = ‘2’ –> 115 km/h  
> Nature = ‘Type autoroutier’ avec classement admin. en autoroute & Importance = ‘3’ –> 100 km/h  
> Nature = ‘Type autoroutier’ non classée comme autoroute & Importance = ‘1’ –> 105 km/h  
> Nature = ‘Type autoroutier’ non classée comme autoroute & Importance = ‘2’ –> 100 km/h  
> Nature = ‘Type autoroutier’ non classée comme autoroute & Importance = ‘3’ –> 95 km/h  
> Nature = ‘Type autoroutier’ non classée comme autoroute & Importance = ‘4’ –> 90 km/h  
> Nature = ‘Type autoroutier’ non classée comme autoroute & Importance = ‘5’ –> 90 km/h  
> Nature = ‘Route à 1 chaussée’ & Importance = ‘1’ –> 67 km/h  
> Nature = ‘Route à 1 chaussée’ & Importance = ‘2’ –> 67 km/h  
> Nature = ‘Route à 1 chaussée’ & Importance = ‘3’ –> 63 km/h  
> Nature = ‘Route à 1 chaussée’ & Importance = ‘4’ –> 58 km/h  
> Nature = ‘Route à 1 chaussée’ & Importance = ‘5’ –> 35 km/h  
> Nature = ‘Route à 2 chaussées’ & Importance = ‘1’ –> 80 km/h  
> Nature = ‘Route à 2 chaussées’ & Importance = ‘2’ –> 75 km/h  
> Nature = ‘Route à 2 chaussées’ & Importance = ‘3’ –> 67 km/h  
> Nature = ‘Route à 2 chaussées’ & Importance = ‘4’ –> 67 km/h  
> Nature = ‘Route à 2 chaussées’ & Importance = ‘5’ –> 65 km/h  
> Nature = ‘Route empierrée’ -> 10km/h  
> Nature = ‘Rond-Point’ -> 25 km/h  
>
> En zone urbaine ('Urbain'="Vrai") :
>
> Nature = ‘Type autoroutier’ avec classement admin. en autoroute & Importance = ‘1’ –> 100 km/h  
> Nature = ‘Type autoroutier’ avec classement admin. en autoroute & Importance = ‘2’ –> 95 km/h  
> Nature = ‘Type autoroutier’ avec classement admin. en autoroute & Importance = ‘3’ –> 90 km/h  
> Nature = ‘Type autoroutier’ non classée comme autoroute & Importance = ‘1’ –> 95 km/h  
> Nature = ‘Type autoroutier’ non classée comme autoroute & Importance = ‘2’ –> 90 km/h  
> Nature = ‘Type autoroutier’ non classée comme autoroute & Importance = ‘3’ –> 85 km/h  
> Nature = ‘Type autoroutier’ non classée comme autoroute & Importance = ‘4’ –> 67 km/h  
> Nature = ‘Type autoroutier’ non classée comme autoroute & Importance = ‘5’ –> 67 km/h  
> Nature = ‘Route à 1 chaussée’ & Importance = ‘1’ –> 50 km/h  
> Nature = ‘Route à 1 chaussée’ & Importance = ‘2’ –> 50 km/h  
> Nature = ‘Route à 1 chaussée’ & Importance = ‘3’ –> 45 km/h  
> Nature = ‘Route à 1 chaussée’ & Importance = ‘4’ –> 40 km/h  
> Nature = ‘Route à 1 chaussée’ & Importance = ‘5’ –> 30 km/h  
> Nature = ‘Route à 2 chaussées’ & Importance = ‘1’ –> 50 km/h  
> Nature = ‘Route à 2 chaussées’ & Importance = ‘2’ –> 50 km/h  
> Nature = ‘Route à 2 chaussées’ & Importance = ‘3’ –> 45 km/h  
> Nature = ‘Route à 2 chaussées’ & Importance = ‘4’ –> 40 km/h  
> Nature = ‘Route à 2 chaussées’ & Importance = ‘5’ –> 35 km/h  
>
> Indifférencié urbain ou non urbain
>
> Nature = ‘Route empierrée’ -> 10 km/h  
> Nature = ‘Rond-Point’ -> 25 km/h  
> Nature = ‘Bac ou liaison maritime’ –> 5 km/h  
> Nature = ‘Bretelle’ & Importance = ‘1’-> 45 km/h  
> Nature = ‘Bretelle’ & Importance = ‘2’-> 45 km/h  
> Nature = ‘Bretelle’ & Importance = ‘3’-> 45 km/h  
> Nature = ‘Bretelle’ & Importance = ‘4’-> 45 km/h  
> Nature = ‘Bretelle’ & Importance = ‘5’-> 30 km/h  
> Nature = ‘Chemin’ & Importance = ‘5’ –> 1 km/h  
> Nature = ‘Sentier’ -> 0 km/h  
> Nature = ‘Escalier’ -> 0 km/h  
> Nature = ‘Piste cyclable’ -> 0 km/h  
>
> Toutes natures, Importance vide ou 6 -> 0 km/h
>
> Etat de l’objet <> ‘En service’ -> 0 km/h  
> Accès véhicule léger = 'Physiquement impossible' -> 0 km/h  
> Accès véhicule léger = Restreint aux ayants droit' -> 0 km/h  
>
> Dans les autres cas, 'Vitesse moyenne VL' ="0".

_(documentation disponible sur le réseau IGN via https://pomme.ign.fr/SV3D/Documentation-BDUni-v2/troncon_de_route#attribute_565, ou dans la documentation de la BDTopo https://geoservices.ign.fr/sites/default/files/2021-07/DC_BDTOPO_3-0.pdf page 345)_

Ainsi, il y a eu une comparaison, après matching avec une base de données HERE, des vitesses de la BDTopo et des vitesses de ce dernier service.

|  | Nombre tronçons | Vitesse moyenne BDUni | Vitesse moyenne Here |
|---|:---:|---|:---:|
| sur totalité des tronçons | 16968108 | 25,5 | 35,6 |
|  |  |  |  |
| selon nature (BDUni) |  |  |  |
| - Type autoroutier | 72544 | 110,9 | 107 |
| - Bretelle | 43469 | 42,8 | 65 |
| - Route à 2 chaussées | 194594 | 58,3 | 51,3 |
| - Route à 1 chaussée | 10087667 | 37,4 | 41,4 |
| - Chemin | 3840227 | 1 | 24,8 |
| - Rond-point | 354694 | 24,7 | 35,5 |
| - Route empierrée | 2374913 | 9 | 24,3 |
> Statistiques complètes sur les vitesses attribuées sur les tronçons dans la BDTopo et dans la base de données HERE

L'examen de ces statistiques nous montre quand même que la vitesse moyenne totale est très cetainement faussée par les tronçons de nature `Chemin` : en effet la vitesse qui leur est attribuée dans la BDTopo est systématiquement de 1 km/h (!), alors que HERE donne une vitesse de 28 km/h en moyenne à ces tronçons. Voici les mêmes statistiques en retirant les `Chemins` :

|  | Nombre tronçons | Vitesse moyenne BDUni | Vitesse moyenne Here |
|---|:---:|---|:---:|
| sur totalité des tronçons | 13127881 | 32,7 | 38,8 |
|  |  |  |  |
| selon nature (BDUni) |  |  |  |
| - Type autoroutier | 72544 | 110,9 | 107 |
| - Bretelle | 43469 | 42,8 | 65 |
| - Route à 2 chaussées | 194594 | 58,3 | 51,3 |
| - Route à 1 chaussée | 10087667 | 37,4 | 41,4 |
| - Rond-point | 354694 | 24,7 | 35,5 |
| - Route empierrée | 2374913 | 9 | 24,3 |
> Statistiques sur les vitesses attribuées sur les tronçons dans la BDTopo et dans la base de données HERE, sans les tronçons de type `Chemin`

On constate dans tous les cas que les vitesses BDTopo sont en moyenne inférieures à celles de HERE. Ainsi, la grande différence de durée dans les résultats de calculs d'itinéraires ne viennent pas des vitesses renseignées en base de données.

#### Modélisation des intersections


## Pistes d'amélioration


### Amélioration des vitesses sur les segments
![Comparaison des itinéraires de Road2 avec et sans recalcul des vitesses et de différents services disponibles en milieu urbain](comparaison_iti_urbain_calc.png)


### Prise en compte des intersections
![Comparaison des itinéraires de Road2 avec divers coûts ajoutés à chaque tronçon et de HERE en milieu urbain](comparaison_temps_iti_s_u1.png)
![Comparaison des itinéraires de Road2 avec divers coûts ajoutés à chaque tronçon et de HERE en milieu urbain](comparaison_temps_iti_s_u2.png)
![Comparaison des isochones de Road2 avec divers coûts ajoutés à chaque tronçon et de HERE en milieu urbain](iso_intesection_temps.png)

![Comparaison des itinéraires de Road2 avec divers coûts ajoutés à chaque tronçon et de HERE en milieu non urbain](comparaison_temps_iti_s_r.png)



### Modification des vitesses et prise en compte des intersections

![Comparaison des itinéraires de Road2 avec les différentes options sur les coûts et de différents services disponibles en milieu urbain](comparaison_iti_urbain_full.png)
