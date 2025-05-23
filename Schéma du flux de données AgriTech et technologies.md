Schéma du flux de données AgriTech et technologies implémentées

## Architecture du système AgriTech - Traçabilité blockchain

```
┌───────────────────────┐     MQTT     ┌───────────────────────┐      HTTP     ┌───────────────────────┐
│ SERVEUR TERRAIN       │              │ SERVEUR CLIENT        │               │ SERVEUR CENTRAL       │
│ (192.168.1.93)        │─────────────>│ (192.168.1.5)         │─────────────> │ (192.168.1.2)         │
│                       │  capteurs    │                       │   données     │                       │
│ • Collecte des données│              │ • Broker MQTT         │               │ • API RESTful         │
│ • Monitoring capteurs │              │ • Proxy API MQTT      │               │ • Stockage blockchain │
│ • Simulation capteurs │              │ • Tableau de bord     │               │ • Interface traçabilité│
└───────────────────────┘              └───────────────────────┘               └───────────────────────┘
       │                                        │                                      │
       │                                        │                                      │
       ▼                                        ▼                                      ▼
┌───────────────────────┐             ┌───────────────────────┐             ┌───────────────────────┐
│ COMPOSANTS TERRAIN    │             │ COMPOSANTS CLIENT     │             │ COMPOSANTS CENTRAL    │
│                       │             │                       │             │                       │
│ • Capteurs temp.      │             │ • Mosquitto           │             │ • Node.js API         │
│ • Capteurs humidité   │             │ • Node-RED/HTTP Server│             │ • Blockchain JS       │
│ • Capteurs sol        │             │ • MQTT API JS         │             │ • MongoDB/JSON Store  │
│ • Capteurs luminosité │             │ • Tableau de bord HTML│             │ • Interface HTML/CSS  │
└───────────────────────┘             └───────────────────────┘             └───────────────────────┘
```

## Technologies et composants par serveur

### 1. SERVEUR TERRAIN (192.168.1.93 → 192.168.1.5)
- **Rôle** : Collecte des données des capteurs et envoi au serveur client
- **Technologies utilisées** :
  - **Python 3** : Langage de programmation principal
  - **Bibliothèque Paho-MQTT** : Client MQTT pour publier les données
  - **Logging** : Bibliothèque standard pour journalisation
- **Composants installés** :
  - **main.py** : Script principal pour la gestion des capteurs
  - **sensor_manager.py** : Module de gestion des capteurs
  - **config.py** : Configuration du système (adresse broker MQTT, etc.)
  - **.env** : Variables d'environnement (attention : contenait l'ancienne IP)
- **Flux de données** :
  - Les capteurs collectent les données (température, humidité, etc.)
  - Les données sont formatées en JSON
  - Publication sur les topics MQTT (ex: `agritech/sensors/temp001`)

### 2. SERVEUR CLIENT (192.168.1.5)
- **Rôle** : Réception des données MQTT et visualisation, passerelle vers le serveur blockchain
- **Technologies utilisées** :
  - **Mosquitto** : Broker MQTT
  - **Node.js / Express** : Pour l'API MQTT
  - **Python** : Serveur HTTP simple
  - **HTML/CSS/JavaScript** : Tableau de bord
  - **Chart.js** : Bibliothèque de visualisation graphique
- **Composants installés** :
  - **mosquitto** : Broker MQTT (port 1883)
  - **mqtt_api.js** : API de proxy MQTT (port 9090)
  - **HTTP Server** : Serveur web simple (port 8080)
  - **dashboard.html** : Interface de visualisation
- **Flux de données** :
  - Réception des messages MQTT du serveur terrain
  - Stockage temporaire des dernières valeurs des capteurs
  - Transmission au tableau de bord via l'API REST
  - Transmission au serveur blockchain

### 3. SERVEUR CENTRAL (192.168.1.2)
- **Rôle** : Stockage sécurisé des données dans la blockchain et traçabilité
- **Technologies utilisées** :
  - **Node.js / Express** : Framework serveur
  - **JavaScript** : Implémentation blockchain
  - **JSON** : Format d'échange et stockage
  - **HTML/CSS/JavaScript** : Interface de traçabilité
  - **Chart.js** : Visualisation de données
- **Composants installés** :
  - **app.js** : API RESTful et blockchain (port 3000)
  - **mqtt_to_blockchain.py** : Script de transfert MQTT vers blockchain
  - **blockchain.json** : Stockage des blocs
  - **public/index.html** : Interface de traçabilité
- **Flux de données** :
  - Réception des données via HTTP POST à `/api/sensor-data`
  - Validation des données (problème résolu : format `sensor_id` vs `id`)
  - Ajout à la blockchain avec horodatage et hachage
  - Interface de consultation pour les parties prenantes

## Problèmes résolus et optimisations

1. **Problème de configuration IP** : L'IP du broker MQTT était encore référencée comme `192.168.1.93` dans le fichier `.env` du serveur terrain, alors qu'elle a été changée à `192.168.1.5` dans le fichier `config.py`.

2. **Format des données** : Adaptation des données entre le format envoyé par les capteurs (`sensor_id`) et le format attendu par l'API blockchain (`id`).

3. **Accès au tableau de bord** : Création d'une interface simple ne dépendant pas de l'API MQTT pour la démonstration.

4. **Traçabilité** : Mise en place d'une interface de traçabilité blockchain avec filtres par type de capteur, par ID et par date.

## Outils de développement et de diagnostic

- **grep** : Recherche d'occurrences d'IP dans les fichiers
- **ps / lsof** : Identification des processus en cours d'exécution
- **curl** : Tests des API REST
- **mosquitto_sub / mosquitto_pub** : Tests du broker MQTT
- **node / python3** : Exécution des scripts serveur

Cette architecture distribuée assure une collecte fiable des données agricoles, une visualisation en temps réel, et un stockage sécurisé avec traçabilité garantie par la technologie blockchain - démontrant l'intégration de l'IoT et de la blockchain dans le secteur agricole.