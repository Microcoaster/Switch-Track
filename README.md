<div align="center">

<img src="docs/banniere.png" alt="Switch Track, module d'aiguillage ESP32" width="100%">

</div>


> Module intelligent d'aiguillage sécurisé pour montagnes russes miniatures avec gestion WiFi automatique et contrôle distant WebSocket (WS/WSS).

[![Version](https://img.shields.io/badge/version-2.0.0-blue.svg)](https://github.com/Microcoaster/Switch-Track)
[![PlatformIO](https://img.shields.io/badge/PlatformIO-compatible-orange.svg)](https://platformio.org/)
[![ESP32](https://img.shields.io/badge/ESP32-compatible-green.svg)](https://www.espressif.com/)

---

## 📋 Description du Projet

### À quoi sert ce projet ?

Ce projet implémente un **module d'aiguillage intelligent** pour un système de montagnes russes miniatures. Le module combine :

- ✅ **Gestion WiFi automatique** avec portail de configuration (AyresWiFiManager)
- ✅ **Contrôle d'aiguillage physique** via verrin électrique et driver DRV8871
- ✅ **Communication WebSocket** (WS ou WSS) avec serveur local ou distant
- ✅ **Interface LED** pour visualisation d'état en temps réel
- ✅ **Authentification sécurisée** et télémétrie périodique
- ✅ **Support développement et production** : basculement facile entre serveur local (ws) et distant (wss)

### Fonctionnalités principales

#### 🌐 **Portail de configuration WiFi**
Chaque module peut être configuré via un portail web local (mode AP) pour entrer les identifiants WiFi de votre réseau.

#### 🔄 **Connexion automatique**
Une fois configuré, le module se connecte automatiquement au réseau WiFi et communique avec le serveur WebSocket.

#### 🔒 **Sécurité avancée**
- Support SSL/TLS (WSS) avec vérification optionnelle d'empreinte (fingerprint)
- Authentification par module ID + mot de passe
- Identifiants WiFi stockés localement dans LittleFS (non versionnés)

#### 🎮 **Gestion multi-modules**
Chaque module (station, switch, launch, etc.) utilise le même firmware et peut être identifié individuellement dans l'application web.

---

## 🔧 Composants Hardware

### **ESP32 DevKit**
- Microcontrôleur principal avec WiFi intégré
- Gestion de toute la logique et communication
- Support LittleFS pour portail web et stockage configuration

### **Système d'Aiguillage**
- **Verrin électrique** : Actionneur linéaire pour basculer l'aiguillage
- **Driver DRV8871** : Contrôleur de moteur H-Bridge pour contrôle bidirectionnel
- **Pins de contrôle** : 
  - IN1 (GPIO 21) : Commande sens horaire
  - IN2 (GPIO 22) : Commande sens anti-horaire

### **Interface Visuelle**
- **LED Gauche** : GPIO 2 (indique position "left")
- **LED Droite** : GPIO 4 (indique position "right")

### **Bouton de Secours**
- **GPIO 0** : Bouton physique pour contrôle manuel
  - **2-5 secondes** : Ouvre le portail de configuration
  - **≥5 secondes** : Efface les identifiants WiFi sauvegardés

---

## 🚀 Fonctionnalités Détaillées

### **1. Gestionnaire WiFi Intelligent (AyresWiFiManager)**
- **Portail captif** automatique au premier démarrage
- **Configuration web** intuitive pour saisir identifiants WiFi
- **Reconnexion automatique** en cas de déconnexion
- **Mode hybride** : portail de secours si connexion échoue (politique ON_FAIL)
- **Timeouts intelligents** : 60 minutes de timeout avec vérification clients actifs
- **Protection fichiers** : `/wifi.json` protégé contre suppression accidentelle

### **2. Contrôle d'Aiguillage Sécurisé**
- **Positions** : "left" (gauche) ou "right" (droite)
- **Temporisations précises** : 
  - Durée de mouvement : 1100ms (1,1 seconde)
  - Dead-time sécurité : 10ms entre changements de direction
- **État sûr** : roue libre automatique après chaque mouvement
- **Séquence sécurisée** : Arrêt → Dead-time → Nouvelle direction → Dead-time

### **3. Communication WebSocket (WS/WSS)**

#### **Mode Développement (WS - sans SSL)**
```cpp
#define SERVER_USE_SSL false
const char* server_host = "192.168.1.16";
const uint16_t server_port = 3000;
```
→ Connexion : `ws://192.168.1.16:3000/esp32`

#### **Mode Production (WSS - avec SSL/TLS)**
```cpp
#define SERVER_USE_SSL true
const char* server_host = "app.microcoaster.com";
const uint16_t server_port = 443;
const char* server_fingerprint = ""; // Optionnel pour vérification certificat
```
→ Connexion : `wss://app.microcoaster.com:443/esp32`

#### **Fonctionnalités WebSocket**
- **Authentification obligatoire** : Module ID + mot de passe avant toute commande
- **Commandes temps réel** : `switch_left`, `switch_right`, `get_position`
- **Télémétrie périodique** : Envoi toutes les 10 secondes
- **Heartbeat** : Keepalive toutes les 30 secondes
- **Reconnexion automatique** : Toutes les 5 secondes en cas de déconnexion

### **4. Monitoring et Télémétrie**
- **Uptime** : Temps de fonctionnement depuis démarrage
- **Position actuelle** : État de l'aiguillage (left/right)
- **Signal WiFi** : Force du signal en dBm (RSSI)
- **Mémoire disponible** : Heap libre pour détecter fuites mémoire
- **Statut opérationnel** : État de santé du module

---

## 📡 Architecture de Communication

```
ESP32 Switch Track (192.168.x.x)
       ↓ WiFi
   Réseau Local
       ↓ 
Serveur WebSocket (local ou distant)
       ↓ ws:// ou wss://
   Interface Web / API
```

### **Protocole WebSocket - Messages JSON**

#### **1. Authentification (ESP32 → Serveur)**
```json
{
  "type": "module_identify",
  "moduleId": "MC-0001-ST",
  "password": "F674iaRftVsHGKOA8hq3TI93HQHUaYqZ",
  "moduleType": "switch-track",
  "uptime": 12345,
  "position": "left"
}
```

#### **2. Confirmation connexion (Serveur → ESP32)**
```json
{
  "type": "connected"
}
```

#### **3. Commandes (Serveur → ESP32)**
```json
{
  "type": "command",
  "data": {
    "command": "switch_left"  // ou "switch_right", "get_position"
  }
}
```

**Commandes supportées** :
- `switch_left` / `left` / `switch_to_A` : Basculer vers la gauche
- `switch_right` / `right` / `switch_to_B` : Basculer vers la droite
- `get_position` : Retourner position actuelle sans mouvement

#### **4. Réponse commande (ESP32 → Serveur)**
```json
{
  "type": "command_response",
  "moduleId": "MC-0001-ST",
  "password": "F674iaRftVsHGKOA8hq3TI93HQHUaYqZ",
  "command": "switch_left",
  "status": "success",
  "position": "left"
}
```

#### **5. Télémétrie (ESP32 → Serveur, toutes les 10s)**
```json
{
  "type": "telemetry",
  "moduleId": "MC-0001-ST",
  "password": "F674iaRftVsHGKOA8hq3TI93HQHUaYqZ",
  "uptime": 123456,
  "position": "left",
  "status": "operational"
}
```

#### **6. Heartbeat (ESP32 → Serveur, toutes les 30s)**
```json
{
  "type": "heartbeat",
  "moduleId": "MC-0001-ST",
  "password": "F674iaRftVsHGKOA8hq3TI93HQHUaYqZ",
  "uptime": 123456,
  "position": "left",
  "wifiRSSI": -65,
  "freeHeap": 180000
}
```

#### **7. Erreur (Serveur → ESP32)**
```json
{
  "type": "error",
  "message": "Authentication failed"
}
```

---

## 🔌 Schéma de Câblage

```
ESP32 DevKit          DRV8871 Driver        Verrin Électrique
─────────────────     ──────────────────    ─────────────────
GPIO 21 (IN1) ────────── IN1 
GPIO 22 (IN2) ────────── IN2 
3.3V ──────────────────── VCC
GND ───────────────────── GND
                          OUT1 ──────────────── Verrin (+)
                          OUT2 ──────────────── Verrin (-)

ESP32 DevKit          LEDs Indication
─────────────────     ────────────────
GPIO 2 ───[330Ω]────── LED Gauche (+) ──┐
GPIO 4 ───[330Ω]────── LED Droite (+) ──┤
GND ────────────────────────────────────┘

ESP32 DevKit          Bouton Secours
─────────────────     ──────────────
GPIO 0 ───────────────── Bouton ──┐
GND ──────────────────────────────┘
```

**⚠️ Notes importantes** :
- Alimenter le DRV8871 avec une tension appropriée pour le verrin (souvent 12V)
- Ajouter des résistances de limitation (330Ω) pour les LEDs
- Le bouton GPIO 0 peut nécessiter une résistance pull-up (souvent intégrée)

---

## 🛠️ Installation et Configuration

### **1. Prérequis**

#### **Logiciels**
- [PlatformIO IDE](https://platformio.org/install/ide?install=vscode) (extension VS Code)
- [Python](https://www.python.org/downloads/) (≥3.6, pour PlatformIO)
- [Git](https://git-scm.com/) (optionnel, pour cloner le projet)

#### **Hardware**
- ESP32 DevKit (ou compatible)
- Câble USB pour programmation
- Driver CH340/CP2102 si nécessaire

#### **Bibliothèques (gérées automatiquement par PlatformIO)**
- `AyresWiFiManager` : Gestion WiFi avec portail captif
- `WebSocketsClient` v2.4.1+ : Communication WebSocket
- `ArduinoJson` v7.0.4+ : Manipulation JSON

### **2. Installation du Projet**

#### **Méthode 1 : Clone Git**
```bash
git clone https://github.com/Microcoaster/Switch-Track.git
cd Switch-Track
```

#### **Méthode 2 : Téléchargement ZIP**
1. Télécharger le projet depuis GitHub
2. Extraire dans un dossier de votre choix

### **3. Configuration du Module**

Ouvrir `src/main.cpp` et configurer les paramètres selon votre environnement :

#### **Configuration WiFi (Point d'accès de secours)**
```cpp
// Ligne ~20
#define ESP_WIFI_SSID "WifiManager-MicroCoaster"
#define ESP_WIFI_PASSWORD "123456789"
```

#### **Configuration Serveur WebSocket**

**Pour développement local :**
```cpp
// Lignes ~28-32
#define SERVER_USE_SSL false
const char* server_host = "192.168.1.16";        // IP de votre serveur local
const uint16_t server_port = 3000;                // Port de votre serveur
const char* websocket_path = "/esp32";
const char* server_fingerprint = "";
```

**Pour production :**
```cpp
#define SERVER_USE_SSL true
const char* server_host = "app.microcoaster.com";
const uint16_t server_port = 443;
const char* websocket_path = "/esp32";
const char* server_fingerprint = "";  // Ajoutez empreinte SSL si souhaité
```

#### **Configuration Module (identifiants uniques)**
```cpp
// Lignes ~36-37
const String MODULE_ID = "MC-0001-ST";                        // ID unique du module
const String MODULE_PASSWORD = "F674iaRftVsHGKOA8hq3TI93HQHUaYqZ"; // Mot de passe
```

⚠️ **Important** : Modifiez le `MODULE_ID` pour chaque module pour éviter les conflits !

### **4. Upload du Firmware**

#### **Étape 1 : Compiler le projet**
```bash
pio run
```

#### **Étape 2 : Upload du firmware**
```bash
pio run --target upload
```

#### **Étape 3 : Upload du système de fichiers (portail web)**
```bash
pio run --target uploadfs
```

#### **Étape 4 : Moniteur série (optionnel)**
```bash
pio device monitor
```

**Ou via l'interface PlatformIO :**
1. Cliquer sur l'icône PlatformIO (barre latérale gauche)
2. Projet Tasks → esp32dev → General → Upload
3. Projet Tasks → esp32dev → Platform → Upload Filesystem Image
4. Projet Tasks → esp32dev → General → Monitor

### **5. Premier Démarrage et Configuration WiFi**

#### **Étape 1 : Démarrage du module**
Après upload, l'ESP32 tente de se connecter au WiFi sauvegardé. Si aucune configuration n'existe, il crée un point d'accès.

**Logs série attendus :**
```
🚀 MicroCoaster - Switch Track v2.0.0
📡 Configuration du point d'accès de secours...
   ├─ SSID: WifiManager-MicroCoaster
   └─ Mot de passe: 123456789
🔄 Initialisation du WiFi Manager...
⚠️  Connexion WiFi échouée
🔧 Ouverture du portail de configuration...
📡 Point d'accès: WifiManager-MicroCoaster
🌐 IP du portail: 192.168.4.1
```

#### **Étape 2 : Connexion au portail**
1. **Connectez votre smartphone/ordinateur** au WiFi `WifiManager-MicroCoaster`
2. **Mot de passe** : `123456789`
3. **Ouvrez un navigateur** et allez sur `http://192.168.4.1`

#### **Étape 3 : Configuration**
1. Le portail affiche les réseaux WiFi disponibles
2. Sélectionnez votre réseau ou saisissez manuellement
3. Entrez le mot de passe WiFi
4. Cliquez sur **Enregistrer**

#### **Étape 4 : Connexion automatique**
Le module redémarre et se connecte automatiquement au réseau WiFi configuré.

**Logs série attendus :**
```
✅ Connexion WiFi réussie !
📡 IP: 192.168.1.42
🌐 Mode: Client WiFi (STA)
[WEBSOCKET] 🔗 Connexion WebSocket...
[WEBSOCKET] 🔓 Mode: WS (plain, sans SSL)
[WEBSOCKET] 🤖 WebSocket: ws://192.168.1.16:3000/esp32
```

### **6. Test de Communication WebSocket**

Une fois connecté au WiFi, le module essaie de se connecter au serveur WebSocket.

**Logs attendus en cas de succès :**
```
[SWITCH TRACK] 🟢 Connecté au serveur WebSocket
[SWITCH TRACK] 🔐 Authentification WebSocket natif...
[SWITCH TRACK] 📤 Authentification envoyée: {"type":"module_identify",...}
[SWITCH TRACK] ✅ Module authentifié WebSocket natif
[SWITCH TRACK] 💡 LED GAUCHE allumée
[SWITCH TRACK] 📊 Télémétrie envoyée
```

---

## 📁 Structure du Projet

```
Switch-Track/
├── src/
│   └── main.cpp              # Code principal du module (tout est centralisé ici)
├── include/
│   └── README                # Documentation PlatformIO
├── data/                     # Fichiers web du portail captif (uploadés vers LittleFS)
│   ├── index.html            # Page principale de configuration
│   ├── success.html          # Page de succès
│   ├── error.html            # Page d'erreur
│   └── logo.png              # Logo MicroCoaster
├── lib/
│   └── README                # Bibliothèques locales (non utilisées actuellement)
├── test/
│   └── README                # Tests unitaires (non implémentés)
├── platformio.ini            # Configuration PlatformIO et dépendances
├── README.md                 # Ce fichier
└── LICENCE                   # Licence du projet
```

### **Fichiers Importants**

| Fichier | Description |
|---------|-------------|
| `src/main.cpp` | **Code principal** - Toute la logique du module |
| `platformio.ini` | **Configuration** - Bibliothèques et paramètres de build |
| `data/*` | **Portail web** - Interface de configuration WiFi |

---

## 🎮 Commandes Disponibles

| Commande | Alias | Description | Action Physique |
|----------|-------|-------------|-----------------|
| `switch_left` | `left`, `switch_to_A` | Basculer vers la gauche | Verrin sens anti-horaire + LED gauche ON |
| `switch_right` | `right`, `switch_to_B` | Basculer vers la droite | Verrin sens horaire + LED droite ON |
| `get_position` | - | Obtenir position actuelle | Aucun mouvement, retourne état |

### **Exemple d'utilisation**

**Commande envoyée par le serveur :**
```json
{
  "type": "command",
  "data": {
    "command": "switch_right"
  }
}
```

**Réponse du module :**
```json
{
  "type": "command_response",
  "moduleId": "MC-0001-ST",
  "password": "F674iaRftVsHGKOA8hq3TI93HQHUaYqZ",
  "command": "switch_right",
  "status": "success",
  "position": "right"
}
```

**Logs série :**
```
[SWITCH TRACK] 📡 Message reçu: {"type":"command","data":{"command":"switch_right"}}
[SWITCH TRACK] 🎮 Commande reçue: switch_right
[SWITCH TRACK] 🔄 Aiguillage basculé vers la DROITE
[SWITCH TRACK] 💡 LED DROITE allumée
[SWITCH TRACK] 📤 Réponse: switch_right -> success
[SWITCH TRACK] ✅ Commande exécutée: right
```

---

## 📊 Surveillance et Debugging

### **Logs Série (Monitor)**

Pour visualiser les logs en temps réel :
```bash
pio device monitor
```

**Logs importants à surveiller :**

#### **Connexion WiFi**
```
✅ Connexion WiFi réussie !
📡 IP: 192.168.1.42
🟢 WiFi connecté - IP: 192.168.1.42 | Signal: -52 dBm
```

#### **WebSocket**
```
[WEBSOCKET] 🔗 Connexion WebSocket...
[WEBSOCKET] 🔓 Mode: WS (plain, sans SSL)
[SWITCH TRACK] 🟢 Connecté au serveur WebSocket
[SWITCH TRACK] ✅ Module authentifié WebSocket natif
```

#### **Télémétrie**
```
[SWITCH TRACK] 💓 Heartbeat envoyé (toutes les 30s)
[SWITCH TRACK] 📊 Télémétrie envoyée (toutes les 10s)
```

#### **Erreurs courantes**
```
🔴 WiFi déconnecté - Portail de configuration actif sur 192.168.4.1
[SWITCH TRACK] 🔴 Déconnexion du serveur
[SWITCH TRACK] ⚠️ Commande refusée - non authentifié
```

### **LEDs d'État**

| État LED | Signification |
|----------|---------------|
| **LED Gauche ON** | Position "left" (aiguillage à gauche) |
| **LED Droite ON** | Position "right" (aiguillage à droite) |
| **Toutes éteintes** | Erreur, non authentifié ou WiFi déconnecté |
| **Clignotement** | Module en cours de connexion |

### **Bouton de Secours (GPIO 0)**

| Action | Durée | Résultat |
|--------|-------|----------|
| **Appui court** | < 2s | Aucune action |
| **Appui moyen** | 2-5s | Ouvre le portail de configuration |
| **Appui long** | ≥ 5s | Efface les identifiants WiFi sauvegardés |

---

## 🔒 Sécurité

### **Authentification**
- ✅ **Authentification obligatoire** avant toute commande
- ✅ **Mot de passe unique** par module (32 caractères alphanumériques)
- ✅ Chaque message inclut `moduleId` + `password` pour vérification

### **Communication Sécurisée (WSS)**
- ✅ **Support SSL/TLS** pour communications cryptées
- ✅ **Vérification d'empreinte** (fingerprint) optionnelle du certificat serveur
- ✅ Protection contre man-in-the-middle

### **Stockage Local**
- ✅ **LittleFS** : Système de fichiers sécurisé
- ✅ **Protection fichiers** : `/wifi.json` protégé contre suppression
- ✅ **Identifiants WiFi** stockés localement (jamais versionnés dans Git)

### **Bonnes Pratiques**

#### **Développement**
```cpp
#define SERVER_USE_SSL false  // OK pour réseau local
const char* server_host = "192.168.1.16";
```

#### **Production**
```cpp
#define SERVER_USE_SSL true   // OBLIGATOIRE en production
const char* server_host = "app.microcoaster.com";
const char* server_fingerprint = "AA BB CC...";  // Recommandé
```

---

## 📚 Ressources et Documentation

### **Bibliothèques Utilisées**
- [AyresWiFiManager](https://github.com/ayresnet/AyresWiFiManager) - Gestion WiFi avec portail captif
- [WebSocketsClient](https://github.com/Links2004/arduinoWebSockets) - Communication WebSocket
- [ArduinoJson](https://arduinojson.org/) - Manipulation JSON

### **Documentation ESP32**
- [Espressif ESP32 Docs](https://docs.espressif.com/projects/esp-idf/en/latest/esp32/)
- [PlatformIO ESP32](https://docs.platformio.org/en/latest/platforms/espressif32.html)

### **Outils**
- [WebSocket Test Client](https://www.websocket.org/echo.html) - Tester connexions WebSocket
- [JSON Validator](https://jsonlint.com/) - Valider format JSON


## 🎉 Remerciements

Merci à la communauté ESP32 et aux contributeurs open-source pour leurs bibliothèques et leurs outils incroyables !

---

**🎢 Projet MicroCoaster - Module Switch Track ESP32 v2.0.0**

*Construit avec ❤️ pour les passionnés de montagnes russes miniatures*
