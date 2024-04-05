# RoverC'S'oftware : utilisation du rover

Cette partie se consacre à l'utilisation du rover dans le cadre d'un développement pour les différents modes d'usage.

## Connexion du rover à l'interface Electrode

Vous pouvez connecter le rover à votre interface de contrôle. Pour faire ça, il faut `ssh`dans la **NavQ+** :

````
ssh user@10.42.0.40
````

Ensuite si il y'a des mises à jours :

````
cd ~/cognipilot/cranium
git pull
git submodule update --remote
colcon build --symlink-install
source install/setup.bash
````

Ensuite il faut lancer le programme principal, qui est un **noeud** `ROS2`,

````
ros2 launch b3rb_bringup robot.launch.py
````

Ensuite il faut ouvrir un autre terminal et pénétrer l'image **Docker**

````
cd ~/cognipilot/docker/dream
./dream start
./dream exec
````

Et enfin démarer **Electrode**

````
ros2 launch electrode electrode.launch.py sim:=true 
````

- Une fenêtre devrez alors s'ouvrir, vous devez cliquer sur "Ouvrir une connexion" et acceptez la connexion "ws://10.42.0.40:4242", en remplaçant "10.42.0.40" par l'adresse ip de la **NavQ+**
- Ensuite dans la fenêtre principale, tout en haut à gauche cliquez et sélectionnez "View" puis "Import layout from file"
- Sélectionnez alors cognipilot > electrode > src > electrode > foxglove_layouts > b3rb.json

## Contrôle manuel

Vous devriez voir sur la gauche de l'interface un **Joystick** avec différents boutons pour les modes du robot. Vous pouvez vérifier l'intégrité du matériel en contrôlant le robot avec ce joystick, la marche à suivre est :

- Demander le mode manuel en appuyant sur **Manual**
- Demander d'armer les moteurs en appuyant sur **Arm**

Attention l'ordre de ces étapes est important, on ne peut armer les moteurs que si un mode à été préalablement sélectionné

Ensuite déplacé le joystick avec votre souris pour faire bouger le robot

## Mode automatique



# Programmation du rover

La programmation du rover se fait sur deux niveaux :

- En python pour le programme de la **NavQ+**
- En C pour le programme du **MrCANHUB**

Il faut bien comprendre que dans tous les cas, rajouter un programme au rover se fait en créant un nouveau **noeud** dans un graph géré par un framework qui s'appelle **ROS2** pour la **NavQ+** et **ZROS** pour le **MrCANHUB.** En créant ce noeud il faut alors le connecter à des **TOPICS** qui sont des objets que l'on envoie le long des arêtes du graph et qui sont les informations que se communiquent les noeuds.

## Exemple du noeud de vision

Le noeud de vision se programme sur la partie **NavQ+** donc en python. Pour déclarer un noeud il faut hérité de la classe **Noeud** de ROS2 :

```python
class TrackVision(Node):

    def __init__(self):
        super().__init__("track_vision")

        # Subscribers
        _ = self.create_subscription(sensor_msgs.msg.CompressedImage,
                                     'camera/image_raw/compressed',
                                     self.camera_image_callback,
                                     qos_profile_sensor_data)

        # Publishers
        self.vision = self.create_publisher(sensor_msgs.msg.CompressedImage,
                                            "vision/image_raw/compressed", 0)
    def camera_image_callback(self, data):
        pass
```

Ici ce que l'on fait c'est que l'on demande à ce que notre noeud **TrackVision** soit connecté au TOPIC 'camera/image_raw/compressed', qui est basiquement une image compressé du flux vidéo enregistré par la caméra du rover.

On déclare également que notre noeud renverra un autre TOPIC qui est bien du type 'CompressedImage` et qui se nomme 'vision/image_raw/compressed'. C'est l'image issu du traitement des informations de l'image de la caméra.

## Exemple du noeud manual

Le noeud qui permet la commande manuel du rover est sur la partie **MrCANHUB** donc en **C**. Pour déclarer un noeud c'est plus compliqué qu'en python. il faut d'abord déclaré un **contexte**, qui représente lui aussi les flux de TOPICS entrant, sortant ainsi que les constantes du noeud :

```c
typedef struct _context {
    struct zros_node node;

    struct zros_sub sub_joy;
    struct zros_pub pub_actuators;

    synapse_msgs_Joy joy;
    synapse_msgs_Actuators actuators;

    const double wheel_radius;
    const double max_turn_angle;
    const double max_velocity;
} context;

static context g_ctx = {
    .node = {},
    .joy = synapse_msgs_Joy_init_default,
    .actuators = synapse_msgs_Actuators_init_default,
    .sub_joy = {},
    .pub_actuators = {},
    .wheel_radius = CONFIG_CEREBRI_B3RB_WHEEL_RADIUS_MM / 1000.0,
    .max_turn_angle = CONFIG_CEREBRI_B3RB_MAX_TURN_ANGLE_MRAD / 1000.0,
    .max_velocity = CONFIG_CEREBRI_B3RB_MAX_VELOCITY_MM_S / 1000.0,
};
```

Ici on dit que l'on **sub/subscribe** à un TOPIC qui s'appelle **Joy**, c'est à dire en fait le **Joystick** que l'on reçoit depuis l'interface Electrode au moyen du logiciel Synapse dont on parlera plus tard.

On dit également que l'on va **pub/publish** un TOPIC **Actuator** qui est directement relié au contrôle du moteur.

On initialise les TOPIC dans une fonction `init` en précisant bien à quel topic (définit ailleurs dans le programme) on souhaite accéder :

```c
static void init(context* ctx)
{
    zros_node_init(&ctx->node, "b3rb_manual");
    zros_sub_init(&ctx->sub_joy, &ctx->node, &topic_joy, &ctx->joy, 10);
    zros_pub_init(&ctx->pub_actuators, &ctx->node,
        &topic_actuators_manual, &ctx->actuators);
}
```

Enfin on créer la boucle principale du noeud, on met à jour les TOPIC auxquels on a souscris, on traite les informations et on publie les TOPIC que l'on souhaite :

```c
static void b3rb_manual_entry_point(void* p0, void* p1, void* p2)
{
    LOG_INF("init");

    context* ctx = p0;
    ARG_UNUSED(p1);
    ARG_UNUSED(p2);

    init(ctx);

    struct k_poll_event events[] = {
        *zros_sub_get_event(&ctx->sub_joy),
    };

    while (true) {
        // wait for joystick input event, publish at 1 Hz regardless

        k_poll(events, ARRAY_SIZE(events), K_MSEC(1000));

        if (zros_sub_update_available(&ctx->sub_joy)) {
            zros_sub_update(&ctx->sub_joy);
        }

        // compute turn_angle, and angular velocity from joystick
        double turn_angle = ctx->max_turn_angle * ctx->joy.axes[JOY_AXES_ROLL];
        double omega_fwd = ctx->max_velocity * ctx->joy.axes[JOY_AXES_THRUST] / ctx->wheel_radius;
        
        b3rb_set_actuators(&ctx->actuators, turn_angle, omega_fwd);

        zros_pub_update(&ctx->pub_actuators);
    }
}

K_THREAD_DEFINE(b3rb_manual, MY_STACK_SIZE,
    b3rb_manual_entry_point, (void*)&g_ctx, NULL, NULL,
    MY_PRIORITY, 0, 1000);
```