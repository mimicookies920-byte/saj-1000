# SAJ-1000 Messager PWA — Guide d'intégration BLE

## Architecture

```
[Téléphone Android Chrome]  ←—BLE GATT—→  [SAJ-1000 Robot]
         MAÎTRE                                  ESCLAVE
```

---

## 1. UUIDs à configurer dans le firmware SAJ-1000

Modifiez les constantes suivantes dans `index.html` (lignes ~280) :

```js
const BLE_SERVICE_UUID = '0000ffe0-0000-1000-8000-00805f9b34fb';
const BLE_TX_CHAR_UUID = '0000ffe1-0000-1000-8000-00805f9b34fb'; // Tel → Robot
const BLE_RX_CHAR_UUID = '0000ffe2-0000-1000-8000-00805f9b34fb'; // Robot → Tel
```

> Ces UUIDs correspondent aux modules HC-08/HM-10 classiques.
> Adaptez-les aux UUIDs réels de votre firmware SAJ-1000.

---

## 2. Protocole de messages (texte, UTF-8, terminé par \n)

### Téléphone → SAJ-1000 (commandes)

| Message envoyé      | Action                        |
|---------------------|-------------------------------|
| `CMD:START\n`       | Démarrer la séquence          |
| `CMD:STOP\n`        | Arrêt immédiat                |
| `CMD:PAUSE\n`       | Mise en pause                 |
| `CMD:RESET\n`       | Réinitialisation              |
| `SET_SPEED:800\n`   | Consigne vitesse 800 tr/min   |
| `STATUS?\n`         | Demander état complet         |
| Texte libre + \n    | Commande personnalisée        |

### SAJ-1000 → Téléphone (état & capteurs)

Le robot répond en paires `CLÉ:VALEUR` séparées par des virgules :

```
STATE:RUNNING,SPEED:1200,LOAD:67
TEMP:42.5,CURR:3.8,VIB:LOW
X:120.5,Y:45.0,Z:0.0
FIN:0,URGENCE:0,ENCODER:1024
MODE:AUTO
ALERT:Défaut capteur X
WARN:Température élevée
```

#### Valeurs STATE acceptées
- `RUNNING` → En marche
- `STOPPED` → Arrêté
- `PAUSED`  → En pause
- `ERROR`   → Erreur
- `IDLE`    → Repos
- `INIT`    → Initialisation

---

## 3. Côté firmware (exemple Arduino/ESP32 avec BLE)

```cpp
#include <BLEDevice.h>
#include <BLEServer.h>
#include <BLEUtils.h>
#include <BLE2902.h>

#define SERVICE_UUID   "0000ffe0-0000-1000-8000-00805f9b34fb"
#define CHAR_TX_UUID   "0000ffe1-0000-1000-8000-00805f9b34fb"
#define CHAR_RX_UUID   "0000ffe2-0000-1000-8000-00805f9b34fb"

BLECharacteristic *pRxChar;

// Callback réception commande du téléphone
class CmdCallback : public BLECharacteristicCallbacks {
  void onWrite(BLECharacteristic *pChar) {
    String cmd = pChar->getValue().c_str();
    cmd.trim();
    if (cmd == "CMD:START")       startMoteur();
    else if (cmd == "CMD:STOP")   stopMoteur();
    else if (cmd == "CMD:PAUSE")  pauseMoteur();
    else if (cmd == "CMD:RESET")  resetSystème();
    else if (cmd.startsWith("SET_SPEED:")) {
      int speed = cmd.substring(10).toInt();
      setVitesse(speed);
    }
    else if (cmd == "STATUS?")    envoyerStatus();
  }
};

// Envoyer données au téléphone
void envoyerStatus() {
  String data = "STATE:" + etatActuel +
                ",SPEED:" + String(vitesseActuelle) +
                ",LOAD:"  + String(chargeMoteur);
  pRxChar->setValue(data.c_str());
  pRxChar->notify();
}

// Appeler régulièrement en loop() pour envoyer les capteurs
void envoyerCapteurs() {
  String data = "TEMP:"    + String(lireTemp(), 1) +
                ",CURR:"   + String(lireCourant(), 2) +
                ",X:"      + String(posX, 2) +
                ",Y:"      + String(posY, 2) +
                ",Z:"      + String(posZ, 2) +
                ",FIN:"    + String(capteurFinCourse) +
                ",URGENCE:"+ String(capteurUrgence);
  pRxChar->setValue(data.c_str());
  pRxChar->notify();
}
```

---

## 4. Installation de la PWA sur Android

1. Ouvrir **Chrome Android** (requis — Web Bluetooth non dispo sur Firefox/Safari)
2. Naviguer vers l'URL de l'application (ex: hébergée sur LAN ou GitHub Pages)
3. Menu Chrome (⋮) → **"Ajouter à l'écran d'accueil"**
4. L'app s'installe comme une vraie application native

### Hébergement local (LAN) avec Python :
```bash
cd /chemin/vers/saj-pwa/
python3 -m http.server 8080
# Accès : http://[IP_PC]:8080
```

> ⚠️ Web Bluetooth nécessite HTTPS ou localhost.
> Pour un réseau local, utilisez un proxy HTTPS ou `ngrok`.

---

## 5. Filtrer uniquement SAJ-1000 au scan

Dans `index.html`, remplacez `acceptAllDevices: true` par :

```js
filters: [{ namePrefix: 'SAJ' }],
// ou
filters: [{ services: [BLE_SERVICE_UUID] }],
```

---

## 6. Mode démonstration

Sans appareil BLE connecté, l'app active automatiquement un **mode démo** qui simule des données réalistes. Utile pour tester l'interface sans le robot.
