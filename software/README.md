# Software : création de l'environnement de travail



## Les prérequis

- Il est actuellement *nécessaire* de posséder un ordinateur (de préférence portable) qui fonctionne avec Linux (de préférence un environnement simple comme Ubuntu 22.04) sous l'architecture x64 (donc pas de Macbook M1/M2/M3 désolé). **Il n'est pas possible de développer sur MacOS et sur Windows pour le moment, mais il est possible de redévelopper les outils initialement prévus par Cognipilot pour les intégrer sur les différents OS**

- Une machine virtuelle ne sera pas possible étant donné que l'environnement de travail est englobée dans une image Docker.

## Docker

Comme tous les hardwares sont différents et que l'utilisateur a peut être déjà installé des packages en conflit avec ceux du buggy, on propose d'encadrer le travail dans une image virtualisé via **Docker**.

Voici les instructions pour installer proprement **Docker**, vous pouvez copier d'un coup toutes les commandes avec le petit icône en haut à droite de la boîte de commande, et les coller dans un terminal Ubuntu en suivant l'une des deux méthodes suivantes

- **Clic droit** puis **Coller**
- **CTRL+SHIFT+V**

##### Désinstallation de tous les packages qui existent en rapport avec une précédente installation de Docker :

````
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
````

##### Récupérations des clées d'installation des différents packages :

````
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
````

##### Installation des packages Docker :

````
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
````

##### Vérification de l'installation : (vous devriez avoir un message de plusieurs lignes dont une contenant "Hello World")

````
sudo docker run hello-world
````

##### Permissions d'utiliser docker pour un utilisateur :

````
sudo groupadd docker
sudo usermod -aG docker $USER
newgrp docker
````

##### Lancement de docker au démarrage :

````
sudo systemctl enable docker.service
sudo systemctl enable containerd.service
````

##### Dernière vérification :

````
docker run hello-world
````

Voilà Docker est bien installé et vous êtes prêt à installé les softs.

## Image docker

Ensuite on télécharge et éxécute l'image Docker correspondant au buggy. Pour le moment il s'agit de l'image prévue par **Cognipilot** mais dans la suite du développement on pourra disposer d'une image **Docker** prévue spécifiquement pour le **RoverCS**

**Attention** l'installation de l'image peut prendre une dizaine de minute. Veillez à accepter tout ce qui peut vous être demandé lors de l'installation et à ne pas laisser votre ordinateur se mettre en veille. Vous pouvez copier d'un coup toutes les commandes avec le petit icône en haut à droite de la boîte de commande, et les coller dans un terminal Ubuntu en suivant l'une des deux méthodes suivantes

- **Clic droit** puis **Coller**
- **CTRL+SHIFT+V**

````
mkdir -p ~/cognipilot
cd ~/cognipilot
git clone -b airy https://github.com/cognipilot/docker
cd  ~/cognipilot/docker
git submodule update --init --recursive
cd ~/cognipilot/docker/dream
./dream start
````

Maintenant que l'image docker est installée vous êtes prêt à développer. Pour tout ce qui va maintenant se passer il faudra pénétrer notre image docker, qui âgit comme un nouvel espace de travail, propre et conçu pour le buggy. Rien de ce que vous y ferez ne sera visible sur votre espace personnel d'origine et c'est ça qui est beau.

Pour pénétrer l'image docker utilisez toujours cette commande :

````
cd ~/cognipilot/docker/dream
./dream start
./dream exec
````

Pour quitter cet espace appuyez sur **CTRL+D** dans le terminal.

## Compilation des différents softs

Normalement vous êtes toujours dans votre image docker mais si ce n'est pas le cas :

````
cd ~/cognipilot/docker/dream
./dream start
./dream exec
````

Une fois dans cette image il va falloir télécharger les codes des différents programmes. D'après la structure proposée initialement par Cognipilot, on a deux programmes généraux et une interface :

- **Cerebri** qui est le programme principal du buggy, il contient entre autre le code spécifique au buggy. Il fait l'interface entre les informations récupérées par `ROS2` et le hardware
- **Cranium** est le programme gérant la partie `ROS2` du buggy, il est présent à la fois pour compiler proprement **Cerebri** sur le **MrCANHUB** et sur la **NavQ+**.
- **Electrode** est l'interface qui permet de "visualiser" le **Cerebri** et **Cranium**. Il s'agit d'un noeud `ROS2`.

  Pour récupérer ces programmes on se propose de cloner les répertoires github les contenant. Ils sont spécifique au buggy et, sont basés par les code de base fourni par **Cognipilot** et **NXP**.


#### Cranium

````
cd ~/cognipilot
git clone https://github.com/Hennzau/cranium.git
cd ~/cognipilot/cranium/
git submodule update --init --recursive
colcon build --symlink-install
cd ~/cognipilot/cranium/
source install/setup.bash
````

#### Cerebri

````
cd ~/cognipilot
mkdir ws
cd ~/cognipilot/ws
git clone https://github.com/Hennzau/cerebri.git
cd ~/cognipilot/ws/cerebri
west init -l .
west update
west build -b mr_canhubk3 app/b3rb -p
source ~/.bashrc
````

#### Electrode

```nxp
cd ~/cognipilot
git clone https://github.com/Hennzau/electrode.git
cd ~/cognipilot/electrode/
colcon build --symlink-install
cd ~/cognipilot/electrode/
source install/setup.bash
```

#### Foxglove Studio

Une fois **Electrode** installé, il faut installer son backend, pour faire ça vous pouvez suivre les commandes suivantes :

````
cd ~/cognipilot
build_foxglove
````

Appuyez sur `n`pour clonner sans `ssh` puis sur `1` pour choisir `airy`

## Flash de la NavQ+ et du MrCANHUB

*Ces étapes sont à effectuer uniquement si c'est la première fois que vous prenez en main le buggy. Lors du développement les procédures seront globalement les mêmes, mais certaines étapes seront à proscrire, tout est rappelé dans la troisième partie du répertoire "usage".*

Commençons par la **NavQ+**, vous pouvez vous référer [à ce guide](NAVQ) afin de vous connecter à la **NavQ** au moyen d'un câble **USB-A <-> USB-C** en mode **FLASH**. Ensuite il faut que vous flashiez la carte en effectuant les manipulations suivantes :

D'abord vous devez aller télécharger les 3 fichiers en bas de cette [cette page github](%5Bhttps://github.com/rudislabs/navqplus-images/releases/tag/22.04.4-humble%5D(https://github.com/rudislabs/navqplus-images/releases/tag/22.04.4-humble))

Les trois fichiers sont :

- navqplus-image-22.04-humble-240105185325.bin-flash_evk
- navqplus-image-22.04-humble-240105185325.wic.zst
- uuu

Il faut placer ces trois fichiers au même endroit dans votre host principal (pas l'image **Docker**et décompresser l'archive de l'ISO :

````
unzstd navqplus-image-22.04-humble-240105185325.wic.zst
````

Ensuite il faut s'octroyer les permissions d'éxécuter le programme `uuu` avec l'iso

````
chmod a+x uuu
````

Et finalement flasher la **NavQ+** :

````
sudo ./uuu -b emmc_all navqplus-image-22.04-humble-240105185325.bin-flash_evk navqplus-image-22.04-humble-240105185325.wic
````

Cette commande peut prendre une dizaine de minute, veillez à ce que votre ordinateur ne se mette pas en veille.

Une fois que le processus de flash est effectué vous devez vous référer [à ce guide](NAVQ.md) afin d'établir une connexion **WIFI** pour la **NavQ+**.

A l'issu de cette connexion au Wifi vous pouvez vous connecter à la **NavQ+** depuis un ordinateur **connecté au même WIFI que la NavQ+**, en utilisant la commande suivante ('10.42.0.40' doit être remplacé par l'adresse récupérée dans le guide de connexion **WIFI** de la **NavQ+**

````
ssh user@10.42.0.40
````

Puis vous devez télécharger, compilé et installé le programme **Cranium**

````
cd ~/cognipilot/
git clone https://github.com/Hennzau/cranium
cd ~/cognipilot/cranium/
git submodule update --init --recursive
colcon build --symlink-install
cd ~/cognipilot/cranium/
source install/setup.bash
````

A la fin du processus vous devez débrancher la connexion entre la **NavQ+** et votre ordinateur **host**. Vous devez vous assurer que les switchs de la carte **NavQ+** sont en mode OFF-ON (**eMMC**).



Il faut désormais flash le **MrCANHUB**, pour ce faire vous pouvez vous référer [à ce guide](MRCANHUB.md) afin de se connecter au **MrCANHUB** et commencer le processus de flash.

Une fois la connexion filaire établie, vous devez pénétrer l'image Docker :

````
cd ~/cognipilot/docker/dream
./dream start
./dream exec
````

Ensuite il faut éxécuter :

````
cd ~/cognipilot/ws/cerebri
git pull
west update
west build -b mr_canhubk3 app/b3rb -p
west flash
````

En théorie tout est opérationnel maintenant. Votre buggy doit maintenant émettre quelques bruits, et un **bip** persiste : c'est votre GPS qui attend une connexion.

A ce stade les différents programmes de base ont été correctement flashés sur les cartes.