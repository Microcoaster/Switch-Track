<div align="center">

<img src="docs/banniere.png" alt="Switch Track, module d'aiguillage" width="100%">

</div>

Aiguillage motorisé pour circuits MicroCoaster. Un vérin électrique déplace la portion de voie mobile vers la gauche ou vers la droite, deux LED signalent la position, et l'ordre arrive du contrôleur en WebSocket.

Comme les autres modules, il se configure au premier démarrage par portail captif, puis rejoint le serveur.

**Version 2.0.0**

## Principe

Le module ne décide rien. Il reçoit un ordre de position, actionne le vérin, vérifie que la position est atteinte, et rend compte. C'est le contrôleur qui sait si la voie en aval est libre et si le changement est autorisé.

```
Ordre reçu           switch_left ou switch_right
Vérin actionné       sens imposé au DRV8871
Position atteinte    LED mise à jour, réponse envoyée au serveur
Sans ordre           le vérin ne bouge pas, la position est conservée
```

La position courante est remontée à chaque connexion et à chaque changement, pour que le serveur ne se désynchronise jamais de la réalité du circuit.

## Matériel

| Élément | Broche | Rôle |
|:--|:--|:--|
| LED position gauche | GPIO 2 | Voie déviée active |
| LED position droite | GPIO 4 | Voie directe active |
| Driver DRV8871 | | Pilotage du vérin, deux sens |
| Vérin électrique | | Déplacement de la voie mobile |

Le DRV8871 est un pont en H : il inverse la polarité aux bornes du vérin selon le sens demandé. Sa protection thermique et sa limitation de courant évitent d'endommager le vérin si la voie est bloquée.

## Protocole

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

| Commande | Effet |
|:--|:--|
| `switch_left`, `left`, `switch_to_A` | Bascule vers la gauche |
| `switch_right`, `right`, `switch_to_B` | Bascule vers la droite |
| `get_position` | Retourne la position sans bouger |

Le module répond à chaque commande et envoie une télémétrie périodique avec sa position et son temps de fonctionnement.

## Liaison

`SERVER_USE_SSL` choisit entre `ws` et `wss`. Sur un réseau local de développement, `ws` suffit. En production, ou dès que le serveur est joignable au-delà du réseau domestique, passez en `wss` : sans chiffrement, le secret d'authentification du module circule en clair.

## Compiler et téléverser

Nécessite [PlatformIO](https://platformio.org/) dans Visual Studio Code.

```bash
pio run                  # compilation
pio run -t upload        # téléversement du firmware
pio run -t uploadfs      # téléversement du portail vers LittleFS
pio device monitor       # console série, 115200 bauds
```

## Première mise en service

1. Alimenter le module. Il crée un point d'accès WiFi.
2. S'y connecter et ouvrir `http://192.168.4.1`.
3. Renseigner le réseau de destination.
4. Le module redémarre, rejoint le réseau et s'annonce auprès du serveur.

Les identifiants WiFi restent en mémoire du module, jamais dans le dépôt.
## Bibliothèques

```ini
links2004/WebSockets        ; liaison avec le contrôleur
bblanchon/ArduinoJson       ; messages échangés
ayresnet/AyresWiFiManager   ; portail captif et reconnexion
```

Système de fichiers embarqué : **LittleFS**, il héberge les pages du portail.

---

<sub>MicroCoaster · Auteurs : CyberSpaceRS, Yamakajump</sub>
