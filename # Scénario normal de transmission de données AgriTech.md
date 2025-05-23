# Scénario normal de transmission de données AgriTech

*Date: 2025-05-08 06:03:11 UTC*  
*Utilisateur: Otmanesabiri*

## Séquence de transmission complète des données

### Étape 1: Collecte des données sur le serveur terrain (192.168.1.93)

```
sabiri_terrain@otmanesrv:~/agritech-terrain$ python3 main.py
2025-05-08 06:03:15,432 - sensor_manager - INFO - Démarrage du système de capteurs AgriTech Zone Terrain...
2025-05-08 06:03:15,433 - sensor_manager - INFO - Initialisation des capteurs...
2025-05-08 06:03:15,434 - sensor_manager - INFO - Capteur TEMP001 initialisé
2025-05-08 06:03:15,434 - sensor_manager - INFO - Capteur TEMP002 initialisé
2025-05-08 06:03:15,435 - sensor_manager - INFO - Capteur HUMID001 initialisé
2025-05-08 06:03:15,435 - sensor_manager - INFO - Capteur HUMID002 initialisé
2025-05-08 06:03:15,436 - sensor_manager - INFO - Capteur SOIL001 initialisé
2025-05-08 06:03:15,436 - sensor_manager - INFO - Capteur SOIL002 initialisé
2025-05-08 06:03:15,437 - sensor_manager - INFO - Capteur LIGHT001 initialisé
2025-05-08 06:03:15,437 - sensor_manager - INFO - Capteur PH001 initialisé
2025-05-08 06:03:15,438 - sensor_manager - INFO - Capteur RAIN001 initialisé
2025-05-08 06:03:15,438 - sensor_manager - INFO - Capteur WIND001 initialisé
2025-05-08 06:03:15,438 - sensor_manager - INFO - 10 capteurs initialisés
2025-05-08 06:03:15,439 - sensor_manager - INFO - Initialisation du client MQTT (broker: 192.168.1.5:1883)...
2025-05-08 06:03:15,445 - sensor_manager - INFO - Client MQTT initialisé
2025-05-08 06:03:15,446 - sensor_manager - INFO - Connecté au broker MQTT
2025-05-08 06:03:15,447 - sensor_manager - INFO - Collecte et envoi des données (intervalle: 30s)
```

**Événement de lecture et publication des données :**
```
2025-05-08 06:03:15,448 - sensor_manager - INFO - Publication des données de tous les capteurs...
2025-05-08 06:03:15,450 - sensor_manager - INFO - Données du capteur TEMP001 publiées sur agritech/sensors/temp001
```

**Format des données publiées pour TEMP001 :**
```json
{
  "sensor_id": "TEMP001",
  "type": "temperature",
  "location": "Serre Nord",
  "value": 22.34,
  "unit": "C",
  "status": "normal",
  "timestamp": "2025-05-08T06:03:15.449872+00:00"
}
```

**Suite de la publication :**
```
2025-05-08 06:03:15,452 - sensor_manager - INFO - Données du capteur TEMP002 publiées sur agritech/sensors/temp002
2025-05-08 06:03:15,453 - sensor_manager - INFO - Données du capteur HUMID001 publiées sur agritech/sensors/humid001
2025-05-08 06:03:15,454 - sensor_manager - INFO - Données du capteur HUMID002 publiées sur agritech/sensors/humid002
2025-05-08 06:03:15,455 - sensor_manager - INFO - Données du capteur SOIL001 publiées sur agritech/sensors/soil001
2025-05-08 06:03:15,456 - sensor_manager - INFO - Données du capteur SOIL002 publiées sur agritech/sensors/soil002
2025-05-08 06:03:15,457 - sensor_manager - INFO - Données du capteur LIGHT001 publiées sur agritech/sensors/light001
2025-05-08 06:03:15,459 - sensor_manager - INFO - Données du capteur PH001 publiées sur agritech/sensors/ph001
2025-05-08 06:03:15,460 - sensor_manager - INFO - Données du capteur RAIN001 publiées sur agritech/sensors/rain001
2025-05-08 06:03:15,461 - sensor_manager - INFO - Données du capteur WIND001 publiées sur agritech/sensors/wind001
2025-05-08 06:03:15,462 - sensor_manager - INFO - Lecture de tous les capteurs...
2025-05-08 06:03:15,469 - sensor_manager - INFO - Données enregistrées dans /home/sabiri_terrain/agritech-terrain/data/sensor_data_20250508_060315.json
2025-05-08 06:03:15,471 - sensor_manager - INFO - Vérification de l'état du système (intervalle: 300s)
2025-05-08 06:03:15,472 - sensor_manager - INFO - Données de santé du système publiées
2025-05-08 06:03:15,473 - sensor_manager - INFO - Système démarré, en attente d'événements...
```

### Étape 2: Réception par le broker MQTT sur le serveur client (192.168.1.5)

Le broker Mosquitto reçoit les messages et les conserve pour les clients abonnés. Dans un terminal séparé, nous pouvons voir les messages reçus avec l'outil `mosquitto_sub` :

```
sabiri@otmanesrv:~$ mosquitto_sub -v -t 'agritech/sensors/#'
agritech/sensors/temp001 {"sensor_id": "TEMP001", "type": "temperature", "location": "Serre Nord", "value": 22.34, "unit": "C", "status": "normal", "timestamp": "2025-05-08T06:03:15.449872+00:00"}
agritech/sensors/temp002 {"sensor_id": "TEMP002", "type": "temperature", "location": "Serre Sud", "value": 24.78, "unit": "C", "status": "normal", "timestamp": "2025-05-08T06:03:15.451658+00:00"}
agritech/sensors/humid001 {"sensor_id": "HUMID001", "type": "humidity", "location": "Serre Nord", "value": 56.42, "unit": "%", "status": "normal", "timestamp": "2025-05-08T06:03:15.452891+00:00"}
agritech/sensors/humid002 {"sensor_id": "HUMID002", "type": "humidity", "location": "Serre Sud", "value": 52.18, "unit": "%", "status": "normal", "timestamp": "2025-05-08T06:03:15.453746+00:00"}
agritech/sensors/soil001 {"sensor_id": "SOIL001", "type": "soil_moisture", "location": "Parcelle A", "value": 38.75, "unit": "%", "status": "normal", "timestamp": "2025-05-08T06:03:15.454896+00:00"}
agritech/sensors/soil002 {"sensor_id": "SOIL002", "type": "soil_moisture", "location": "Parcelle B", "value": 41.23, "unit": "%", "status": "normal", "timestamp": "2025-05-08T06:03:15.455941+00:00"}
agritech/sensors/light001 {"sensor_id": "LIGHT001", "type": "light", "location": "Serre Nord", "value": 6245.87, "unit": "lux", "status": "normal", "timestamp": "2025-05-08T06:03:15.457038+00:00"}
agritech/sensors/ph001 {"sensor_id": "PH001", "type": "pH", "location": "Parcelle A", "value": 6.52, "unit": "pH", "status": "normal", "timestamp": "2025-05-08T06:03:15.458642+00:00"}
agritech/sensors/rain001 {"sensor_id": "RAIN001", "type": "rainfall", "location": "Station Météo", "value": 0.0, "unit": "mm/h", "status": "normal", "timestamp": "2025-05-08T06:03:15.459871+00:00"}
agritech/sensors/wind001 {"sensor_id": "WIND001", "type": "wind_speed", "location": "Station Météo", "value": 1.25, "unit": "km/h", "status": "normal", "timestamp": "2025-05-08T06:03:15.460798+00:00"}
```

### Étape 3: Traitement par l'API MQTT sur le serveur client

L'API MQTT créée avec Node.js (mqtt_api.js) reçoit et stocke les données les plus récentes :

```
sabiri@otmanesrv:~/agritech/api_gateway$ node mqtt_api.js
API MQTT en écoute sur le port 9090
Connecté au broker MQTT
Abonné au topic agritech/sensors/#
Données reçues pour TEMP001: 22.34 C
Données reçues pour TEMP002: 24.78 C
Données reçues pour HUMID001: 56.42 %
Données reçues pour HUMID002: 52.18 %
Données reçues pour SOIL001: 38.75 %
Données reçues pour SOIL002: 41.23 %
Données reçues pour LIGHT001: 6245.87 lux
Données reçues pour PH001: 6.52 pH
Données reçues pour RAIN001: 0.0 mm/h
Données reçues pour WIND001: 1.25 km/h
```

### Étape 4: Transfert vers le serveur blockchain (192.168.1.2)

Le script `mqtt_to_blockchain.py` qui s'exécute sur le serveur central reçoit les messages MQTT, les transforme et les envoie à l'API blockchain :

```
sabiri_centrale@otmanesrv:~/agritech-blockchain/subscribe_mqtt$ python3 mqtt_to_blockchain.py
Connexion au broker MQTT: 192.168.1.5:1883
Connexion à l'API blockchain: http://localhost:3000/api/sensor-data
Client MQTT connecté avec résultat: 0
Abonné à agritech/sensors/# avec résultat (0,)
Message reçu sur agritech/sensors/temp001: {'sensor_id': 'TEMP001', 'type': 'temperature', 'location': 'Serre Nord', 'value': 22.34, 'unit': 'C', 'status': 'normal', 'timestamp': '2025-05-08T06:03:15.449872+00:00'}
Envoi des données reformatées à http://localhost:3000/api/sensor-data...
✅ Données ajoutées à la blockchain: {'message': 'Données ajoutées à la blockchain', 'block': {'index': 145, 'timestamp': '2025-05-08T06:03:15.584Z', 'data': {'id': 'TEMP001', 'type': 'temperature', 'value': 22.34, 'unit': 'C', 'location': 'Serre Nord', 'timestamp': '2025-05-08T06:03:15.449872+00:00', 'status': 'normal'}, 'previousHash': '9f8e7d6c5b4a3210fedcba9876543210e9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4', 'hash': 'a1b2c3d4e5f67890fedcba9876543210e9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4'}}
```

Le même processus se répète pour tous les capteurs, avec des messages similaires.

### Étape 5: Stockage dans la blockchain et exposition via l'API

L'API blockchain (app.js) reçoit les données, ajoute un nouveau bloc à la chaîne et fournit une confirmation :

```
sabiri_centrale@otmanesrv:~/agritech-blockchain/api$ node app.js
API Gateway démarrée sur le port 3000
Nouvelle donnée reçue pour le capteur TEMP001: 22.34 C
Bloc #145 ajouté à la blockchain avec hash a1b2c3d4e5f67890fedcba9876543210e9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4
Nouvelle donnée reçue pour le capteur TEMP002: 24.78 C
Bloc #146 ajouté à la blockchain avec hash b2c3d4e5f6a7890fedcba9876543210e9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4
Nouvelle donnée reçue pour le capteur HUMID001: 56.42 %
Bloc #147 ajouté à la blockchain avec hash c3d4e5f6a7b890fedcba9876543210e9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4
...
```

### Étape 6: Visualisation des données dans le tableau de bord (serveur client)

Un utilisateur accède au tableau de bord à l'adresse `http://192.168.1.5:8080/dashboard.html`. Le tableau de bord effectue des requêtes HTTP régulières à l'API MQTT pour récupérer les dernières données :

```
GET http://192.168.1.5:9090/api/sensors
200 OK
Content-Type: application/json
[
  {
    "id": "TEMP001",
    "type": "temperature",
    "value": 22.34,
    "unit": "C",
    "location": "Serre Nord",
    "timestamp": "2025-05-08T06:03:15.449872+00:00",
    "status": "normal",
    "topic": "agritech/sensors/temp001",
    "lastUpdate": "2025-05-08T06:03:15.584Z"
  },
  {
    "id": "TEMP002",
    "type": "temperature",
    "value": 24.78,
    "unit": "C",
    "location": "Serre Sud",
    "timestamp": "2025-05-08T06:03:15.451658+00:00",
    "status": "normal",
    "topic": "agritech/sensors/temp002",
    "lastUpdate": "2025-05-08T06:03:15.596Z"
  },
  ...
]
```

Le navigateur affiche les données dans des graphiques en temps réel, et le journal des événements montre les mises à jour :

```
[06:03:15] Récupération des données des capteurs...
[06:03:15] Données mises à jour pour 10 capteurs
[06:03:45] Récupération des données des capteurs...
[06:03:45] Données mises à jour pour 10 capteurs
```

### Étape 7: Consultation de la traçabilité blockchain

Un utilisateur accède à l'interface de traçabilité blockchain à l'adresse `http://192.168.1.2:3000`. L'interface affiche les blocs de la blockchain avec les données des capteurs :

```
GET http://192.168.1.2:3000/api/blockchain
200 OK
Content-Type: application/json
[
  {
    "index": 0,
    "timestamp": "2025-05-01T00:00:00.000Z",
    "data": {
      "id": "GENESIS",
      "type": "system",
      "value": 0,
      "unit": "N/A",
      "timestamp": "2025-05-01T00:00:00.000Z"
    },
    "previousHash": "0",
    "hash": "000000000000000000000000000000000000000000000000000000000000000"
  },
  ...
  {
    "index": 145,
    "timestamp": "2025-05-08T06:03:15.584Z",
    "data": {
      "id": "TEMP001",
      "type": "temperature",
      "value": 22.34,
      "unit": "C",
      "location": "Serre Nord",
      "timestamp": "2025-05-08T06:03:15.449872+00:00",
      "status": "normal"
    },
    "previousHash": "9f8e7d6c5b4a3210fedcba9876543210e9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4",
    "hash": "a1b2c3d4e5f67890fedcba9876543210e9a8b7c6d5e4f3a2b1c0d9e8f7a6b5c4"
  },
  ...
]
```

L'interface affiche des statistiques sur les données blockchain et permet de filtrer les blocs par type de capteur, ID de capteur et date.

### Étape 8 : Cycle continu de transmission

Le processus se répète toutes les 30 secondes, et à 06:03:45 UTC, le serveur terrain publie une nouvelle série de données :

```
2025-05-08 06:03:45,448 - sensor_manager - INFO - Collecte et envoi des données (intervalle: 30s)
2025-05-08 06:03:45,449 - sensor_manager - INFO - Publication des données de tous les capteurs...
2025-05-08 06:03:45,450 - sensor_manager - INFO - Données du capteur TEMP001 publiées sur agritech/sensors/temp001
```

Ces données suivent le même parcours que précédemment, passant par le broker MQTT, l'API MQTT, puis l'API blockchain, pour finalement être affichées dans les interfaces utilisateur.

## Résumé de la séquence complète

1. **Capteurs → Serveur terrain** : Les capteurs physiques transmettent leurs mesures au serveur terrain.
2. **Serveur terrain → Broker MQTT** : Les données sont formatées et publiées via MQTT vers le broker (192.168.1.5:1883).
3. **Broker MQTT → API MQTT** : Le broker MQTT transmet les messages aux clients abonnés, dont l'API MQTT.
4. **API MQTT → Tableau de bord** : L'API MQTT expose une API REST que le tableau de bord interroge.
5. **Broker MQTT → Script MQTT-to-Blockchain** : Les messages MQTT sont également reçus par le script de pont vers la blockchain.
6. **Script MQTT-to-Blockchain → API Blockchain** : Les données sont reformatées et envoyées à l'API blockchain.
7. **API Blockchain → Stockage Blockchain** : L'API ajoute un nouveau bloc à la chaîne et sauvegarde le fichier blockchain.json.
8. **API Blockchain → Interface Traçabilité** : L'interface de traçabilité interroge l'API blockchain pour afficher l'historique complet.

Ce flux de données complet assure que chaque mesure des capteurs est collectée, transmise, visualisée et stockée de manière sécurisée et traçable, permettant ainsi une surveillance efficace des conditions agricoles et une traçabilité garantie par la technologie blockchain.