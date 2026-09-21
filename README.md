<div align="center">

<img src="docs/banniere.png" alt="Switch Track, module d'aiguillage" width="100%">

</div>

Aiguillage motorisé pour circuits MicroCoaster. Un vérin électrique déplace la portion de voie mobile vers la gauche ou vers la droite, deux LED signalent la position, et l'ordre arrive du contrôleur en WebSocket.

Comme les autres modules, il se configure au premier démarrage par portail captif, puis rejoint le serveur.

**Version 2.0.0**

<img src="docs/sections/s01.png" alt="01 Principe" width="100%">

Le module ne décide rien. Il reçoit un ordre de position, actionne le vérin, vérifie que la position est atteinte, et rend compte. C'est le contrôleur qui sait si la voie en aval est libre et si le changement est autorisé.

<img src="docs/schemas/principe.png" alt="Ordre reçu : le serveur envoie switch_left ou switch_right. Vérin actionné : le sens est imposé au DRV8871, qui inverse la polarité. Position atteinte : la LED correspondante s'allume et la réponse part vers le serveur. Sans ordre : le vérin ne bouge pas, la position est conservée." width="100%">

La position courante est remontée à chaque connexion et à chaque changement, pour que le serveur ne se désynchronise jamais de la réalité du circuit.

<img src="docs/sections/s02.png" alt="02 Matériel" width="100%">

<img src="docs/schemas/brochage.png" alt="Actionneur DRV8871 : GPIO 21 pour IN1, vérin en sens horaire ; GPIO 22 pour IN2, vérin en sens anti-horaire. Signalisation de position : GPIO 2 pour la LED gauche, voie déviée active ; GPIO 4 pour la LED droite, voie directe active." width="100%">

Le DRV8871 est un pont en H : il met `IN1` ou `IN2` à l'état haut pour inverser la polarité aux bornes du vérin, et les met tous deux à zéro pour l'arrêter. Sa protection thermique et sa limitation de courant évitent d'endommager le vérin si la voie est bloquée.

<img src="docs/sections/s03.png" alt="03 Protocole" width="100%">

Le module s'authentifie à la connexion, puis échange en JSON.

**Identification, module vers serveur**

```json
{
  "type": "module_identify",
  "moduleId": "MC-0001-ST",
  "password": "<secret du module>",
  "moduleType": "switch-track",
  "uptime": 12345,
  "position": "left"
}
```

**Commande, serveur vers module**

```json
{
  "type": "command",
  "data": { "command": "switch_left" }
}
```

<img src="docs/schemas/commandes.png" alt="switch_left : bascule la voie vers la gauche, accepte aussi left et switch_to_A. switch_right : bascule la voie vers la droite, accepte aussi right et switch_to_B. get_position : retourne la position courante sans actionner le vérin." width="100%">

Le module répond à chaque commande et envoie une télémétrie périodique avec sa position et son temps de fonctionnement.

`SERVER_USE_SSL` choisit entre `ws` et `wss`. Sur un réseau local de développement, `ws` suffit. En production, ou dès que le serveur est joignable au-delà du réseau domestique, passez en `wss` : sans chiffrement, le secret d'authentification du module circule en clair.

<img src="docs/sections/s04.png" alt="04 Mise en service" width="100%">

Copiez d'abord [`include/env.h.example`](include/env.h.example) en `include/env.h` et renseignez-le : identifiants du portail de secours, identité du module et son secret. Ce fichier n'est pas versionné, et sans lui le firmware ne compile pas.

Nécessite [PlatformIO](https://platformio.org/) dans Visual Studio Code.

```bash
pio run                  # compilation
pio run -t upload        # téléversement du firmware
pio run -t uploadfs      # téléversement du portail vers LittleFS
pio device monitor       # console série, 115200 bauds
```

1. Alimenter le module. Il crée un point d'accès WiFi.
2. S'y connecter et ouvrir `http://192.168.4.1`.
3. Renseigner le réseau de destination.
4. Le module redémarre, rejoint le réseau et s'annonce auprès du serveur.

Les identifiants WiFi restent en mémoire du module, jamais dans le dépôt.

<img src="docs/sections/s05.png" alt="05 Écosystème" width="100%">

```ini
links2004/WebSockets        ; liaison avec le contrôleur
bblanchon/ArduinoJson       ; messages échangés
ayresnet/AyresWiFiManager   ; portail captif et reconnexion
```

Système de fichiers embarqué : **LittleFS**, il héberge les pages du portail. Le socle commun à tous les modules est le [WiFi Manager](https://github.com/Microcoaster/MicroCoaster_WifiManager), le [Banc LED](https://github.com/Microcoaster/ESP-32-led) en est la version d'essai sans mécanique, et le pilotage se fait depuis la [WebApp](https://github.com/Microcoaster/MicroCoasterWebApp).

---

<sub>MicroCoaster · Auteur : Cybertrist</sub>
