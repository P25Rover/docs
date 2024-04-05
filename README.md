# RoverCS

RoverCS est un projet de création de rover autonome démaré pour le pôle projet 25 - Véhicules autonomes dans le cadre de la compétition NXPCup.

Ce répertoire documente le rover, initialement créé par Azzi Bilal, Bouhamed Mohamed et Le Van Enzo en s'aidant d'un kit fourni par NXP ainsi que les documentations des sites de NXP et Cognipilot.

Ce répertoire vous permettra de comprendre les différentes parties du hardware, du software et comment à votre tour, vous pourrez continuer de développer ce rover.

## Hardware : composants et montage

Le rover est constitué d'une base roulante (4 roues, axe moteur, moteur), d'un PDB (Power Distribution board), d'une batterie reliée au PDB, et de deux autres cartes :

- Le MrCANHUB qui est un microcontrôleur alimenté par le PDB, relié au moteur qu'il peut contrôler et sur lequel est pluggé le GPS
- La NavQ+ qui est une micro ordinateur, alimenté lui aussi par le PDB, qui est relié au MrCANHUB par la technologie éthernet. Il est connecté directement au LIDAR et à la caméra avant du véhicule. De plus c'est cette carte qui est capable de se connecter à un réseau Wifi. Ainsi en étant sur le même réseau Wifi que cette carte on peut s'y connecter via `ssh`

Afin de comprendre mieux le montage vous pouvez vous référer à [cette partie du répertoire](hardware/README.md)

## Software : création de l'environnement de travail

Comme évoqué précédemment, la carte NavQ+ peut se connecter à un réseau Wifi, cela permettra de s'y connecter via `ssh` afin de lancer divers programmes.

Ce rover utilise actuellement le framework `ROS2`, ce framework utilise un graph où les **noeuds** sont des programmes indépendants que l'on fait communiquer. Ils peuvent communiquer via un protocole réseau (Wifi, éthernet). Ainsi les programmes éxécutés sur le **MrCANHUB**, sur la **NavQ+** et votre ordinateur **host** seront un ensemble de **noeuds** qui communiqueront via Wifi et éthernet. Au niveau de la logique des softwares, on peut distinguer les programmes :

- Dans le MrCANHUB tourne en permanence un **noeud** `ROS2` programmé par vos soins, qui pourra attendre de potentielles informations de la NavQ+
- Dans la NavQ+ rien ne tourne en permanence: vous devez vous y connecter pour éxécuter divers programmes, dont le programme qui communique avec le MrCANHUB et votre ordinateur.
- Sur votre ordinateur peut tourner une interface de contrôle du Robot qui communique en temps réel avec le NavQ+ (et donc implicitement avec le MrCANHUB)

Dans le cadre de la compétition NXPCup 2024, avec le kit MR-B3RB, seul la partie **MrCANHUB** est à modifier par les candidats, ainsi les noeuds de la NavQ+ ne seront probablement pas modifiés.

Afin de comprendre comment installer un environnement de travail pour le RoverCS vous pouvez vous référer à [cette partie du répertoire](software/README.md)

## RoverC'S'oftware : utilisation et programmation du rover

Les comportements attendus du rover sont les suivants :

- Mode manuel contrôlé via un ordinateur **host** : le rover peut suivre une consigne de déplacement ainsi qu'une consigne de position couplée avec le LIDAR afin de déterminer le chemin libre le plus court à parcourir (en évitant les obstacles, statiques et dynamiques)
- Mode autonome de suivi de ligne : le rover, une fois placé sur une trajectoire définie par deux lignes paralèlles, va se déplacer sur le parcours en évitant les obstacles.

Afin de comprendre comment utiliser ces différents modes vous pouvez vous référer à [à cette partie du répertoire](usage/README.md). Ce répertoire continent aussi des informations importantes sur la manière dont il faut programmer ce robot 

