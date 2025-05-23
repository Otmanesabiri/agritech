Guide de Réalisation - Projet AgriTech Blockchain (Phase 1)
Introduction
Ce guide détaille la mise en œuvre complète de la première phase du projet AgriTech Blockchain, qui consiste à déployer l'infrastructure de la Zone Clients (VM3). Cette infrastructure permet de collecter des données de capteurs IoT via MQTT, de les traiter via une API REST, et de les visualiser via une interface web.

Prérequis
Machine Ubuntu Server (nous utilisons Ubuntu 25.04)
Accès SSH avec privilèges sudo
Connexion Internet
Architecture du Système
Code
┌─────────────────────────────────────────────────────┐
│                   VM3 (Zone Clients)                │
├─────────────────────────────────────────────────────┤
│                                                     │
│  ┌─────────────┐     ┌──────────────┐   ┌────────┐  │
│  │ Simulateur  │────▶│  MQTT Broker │◀──│  API   │  │
│  │  Python     │     │  Mosquitto   │   │ Gateway│  │
│  └─────────────┘     └──────────────┘   └────┬───┘  │
│                                              │      │
│                                         ┌────▼────┐ │
│                                         │ Frontend│ │
│                                         │  Nginx  │ │
│                                         └─────────┘ │
└─────────────────────────────────────────────────────┘
Étape 1: Installation du Broker MQTT (Mosquitto)
bash
# Installation de Mosquitto
sudo apt update
sudo apt install -y mosquitto mosquitto-clients

# Configuration de Mosquitto pour permettre les connexions
sudo bash -c 'cat > /etc/mosquitto/conf.d/default.conf << EOF
listener 1883
allow_anonymous true
EOF'

# Redémarrer Mosquitto
sudo systemctl restart mosquitto

# Vérification du statut
sudo systemctl status mosquitto
Étape 2: Installation de Node.js et création de l'API Gateway
bash
# Installation de Node.js et npm
sudo apt install -y nodejs npm

# Création du répertoire pour l'API
mkdir -p ~/agritech/api_gateway
cd ~/agritech/api_gateway

# Initialisation du projet Node.js
npm init -y

# Installation des dépendances
npm install express cors helmet morgan mqtt
Étape 3: Développement de l'API Gateway
bash
# Création du fichier serveur.js
nano ~/agritech/api_gateway/server.js
Contenu du fichier server.js:

JavaScript
const express = require('express');
const cors = require('cors');
const helmet = require('helmet');
const morgan = require('morgan');
const mqtt = require('mqtt');

// Configuration
const app = express();
const PORT = process.env.PORT || 8080;

// Middleware
app.use(cors({
  origin: '*',
  methods: ['GET', 'POST', 'PUT', 'DELETE'],
  allowedHeaders: ['Content-Type', 'Authorization']
}));
app.use(helmet());
app.use(express.json());
app.use(morgan('combined'));

// Stockage des données
const sensorData = {};

// Connexion MQTT
const mqttClient = mqtt.connect('mqtt://localhost');

mqttClient.on('connect', () => {
  console.log('Connecté au broker MQTT');
  mqttClient.subscribe('agritech/#', (err) => {
    if (!err) {
      console.log('Abonné aux topics agritech');
    }
  });
});

mqttClient.on('message', (topic, message) => {
  console.log(`Message reçu sur ${topic}: ${message.toString()}`);
  try {
    const data = JSON.parse(message.toString());
    sensorData[topic] = {
      ...data,
      lastUpdate: new Date().toISOString()
    };
  } catch (error) {
    console.error('Erreur de parsing des données:', error);
  }
});

// Routes API
app.get('/', (req, res) => {
  res.json({ status: 'API Gateway Active', time: new Date().toISOString() });
});

app.get('/api/sensors', (req, res) => {
  res.json(sensorData);
});

// Démarrer le serveur
app.listen(PORT, () => {
  console.log(`API Gateway démarrée sur le port ${PORT}`);
});
Étape 4: Création du service systemd pour l'API
bash
# Création du fichier de service
sudo nano /etc/systemd/system/agritech-api.service
Contenu du fichier agritech-api.service:

Code
[Unit]
Description=AgriTech API Gateway
After=network.target mosquitto.service
Wants=mosquitto.service

[Service]
User=sabiri
WorkingDirectory=/home/sabiri/agritech/api_gateway
ExecStart=/usr/bin/node /home/sabiri/agritech/api_gateway/server.js
Restart=always
RestartSec=10
Environment=NODE_ENV=production
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
bash
# Activation du service
sudo systemctl daemon-reload
sudo systemctl enable agritech-api
sudo systemctl start agritech-api
Étape 5: Configuration du pare-feu
bash
# Installation et configuration d'UFW
sudo apt install -y ufw

# Configuration des règles
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp comment 'SSH'
sudo ufw allow 8080/tcp comment 'API Gateway'
sudo ufw allow 80/tcp comment 'HTTP'
sudo ufw enable
Étape 6: Création du simulateur de capteurs
bash
# Création du répertoire pour le simulateur
mkdir -p ~/agritech/simulator
nano ~/agritech/simulator/sensor_simulator.py
Contenu du fichier sensor_simulator.py:

Python
#!/usr/bin/env python3
import json
import random
import subprocess
import time
import datetime

# Configuration
INTERVAL = 30  # Secondes entre chaque envoi de données
SENSORS = [
    {"id": "TEMP001", "location": "Serre1", "unit": "C", "min": 18, "max": 32},
    {"id": "HUMID001", "location": "Serre1", "unit": "%", "min": 40, "max": 80},
    {"id": "SOIL001", "location": "Champ1", "unit": "%", "min": 20, "max": 60},
    {"id": "LIGHT001", "location": "Serre2", "unit": "lux", "min": 500, "max": 2000},
    {"id": "PH001", "location": "Champ2", "unit": "pH", "min": 5.5, "max": 7.5}
]

def generate_sensor_data(sensor):
    """Génère des données aléatoires pour un capteur"""
    value = round(random.uniform(sensor["min"], sensor["max"]), 1)
    return {
        "sensor_id": sensor["id"],
        "location": sensor["location"],
        "value": value,
        "unit": sensor["unit"],
        "timestamp": datetime.datetime.now(datetime.timezone.utc).strftime("%Y-%m-%dT%H:%M:%SZ")
    }

def send_mqtt_message(topic, data):
    """Envoie un message MQTT"""
    mqtt_cmd = [
        "mosquitto_pub", 
        "-h", "localhost", 
        "-t", topic, 
        "-m", json.dumps(data)
    ]
    subprocess.run(mqtt_cmd)
    print(f"Données envoyées sur {topic}: {data}")

def main():
    """Fonction principale"""
    print("Simulateur de capteurs AgriTech démarré...")
    print(f"Envoi de données toutes les {INTERVAL} secondes")
    
    try:
        while True:
            for sensor in SENSORS:
                topic = f"agritech/sensors/{sensor['id'].lower()}"
                data = generate_sensor_data(sensor)
                send_mqtt_message(topic, data)
            
            time.sleep(INTERVAL)
            
    except KeyboardInterrupt:
        print("\nSimulateur arrêté par l'utilisateur")

if __name__ == "__main__":
    main()
bash
# Rendre le script exécutable
chmod +x ~/agritech/simulator/sensor_simulator.py
Étape 7: Création du service pour le simulateur
bash
# Création du fichier de service
sudo nano /etc/systemd/system/agritech-simulator.service
Contenu du fichier agritech-simulator.service:

Code
[Unit]
Description=AgriTech Sensor Simulator
After=network.target mosquitto.service

[Service]
User=sabiri
WorkingDirectory=/home/sabiri/agritech/simulator
ExecStart=/usr/bin/python3 /home/sabiri/agritech/simulator/sensor_simulator.py
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
bash
# Activation du service
sudo systemctl daemon-reload
sudo systemctl enable agritech-simulator
sudo systemctl start agritech-simulator
Étape 8: Installation et configuration de Nginx
bash
# Installation de Nginx
sudo apt install -y nginx

# Création du répertoire pour le frontend
mkdir -p ~/agritech/frontend
nano ~/agritech/frontend/index.html
Contenu du fichier index.html:

HTML
<!DOCTYPE html>
<html lang="fr">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>AgriTech Blockchain - Dashboard</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 20px;
            background-color: #f5f5f5;
        }
        .container {
            max-width: 1200px;
            margin: 0 auto;
        }
        .header {
            background-color: #4CAF50;
            padding: 20px;
            color: white;
            border-radius: 5px;
            margin-bottom: 20px;
        }
        .card {
            background-color: white;
            border-radius: 5px;
            box-shadow: 0 2px 5px rgba(0,0,0,0.1);
            padding: 20px;
            margin-bottom: 20px;
        }
        .sensor-grid {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
            gap: 20px;
        }
        .sensor {
            border: 1px solid #ddd;
            border-radius: 5px;
            padding: 15px;
        }
        .refresh-btn {
            background-color: #008CBA;
            border: none;
            color: white;
            padding: 10px 20px;
            text-align: center;
            text-decoration: none;
            display: inline-block;
            font-size: 16px;
            margin: 4px 2px;
            cursor: pointer;
            border-radius: 5px;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="header">
            <h1>AgriTech Blockchain</h1>
            <p>Tableau de bord des capteurs IoT</p>
        </div>
        
        <div class="card">
            <h2>État des capteurs</h2>
            <button class="refresh-btn" onclick="fetchSensorData()">Rafraîchir</button>
            <div id="sensor-data" class="sensor-grid">
                <p>Chargement des données...</p>
            </div>
        </div>
    </div>

    <script>
        // Fonction pour récupérer les données des capteurs
        function fetchSensorData() {
            fetch('http://192.168.1.3:8080/api/sensors')
                .then(response => response.json())
                .then(data => {
                    const sensorContainer = document.getElementById('sensor-data');
                    sensorContainer.innerHTML = '';
                    
                    if (Object.keys(data).length === 0) {
                        sensorContainer.innerHTML = '<p>Aucune donnée de capteur disponible</p>';
                        return;
                    }
                    
                    for (const topic in data) {
                        const sensor = data[topic];
                        const sensorElement = document.createElement('div');
                        sensorElement.className = 'sensor';
                        
                        // Créer le contenu HTML pour chaque capteur
                        let sensorHTML = `
                            <h3>${sensor.sensor_id || 'Capteur inconnu'}</h3>
                            <p><strong>Topic:</strong> ${topic}</p>
                        `;
                        
                        if (sensor.value !== undefined) {
                            sensorHTML += `<p><strong>Valeur:</strong> ${sensor.value} ${sensor.unit || ''}</p>`;
                        }
                        
                        if (sensor.location) {
                            sensorHTML += `<p><strong>Emplacement:</strong> ${sensor.location}</p>`;
                        }
                        
                        if (sensor.timestamp) {
                            sensorHTML += `<p><strong>Horodatage:</strong> ${sensor.timestamp}</p>`;
                        }
                        
                        if (sensor.lastUpdate) {
                            sensorHTML += `<p><strong>Dernière mise à jour:</strong> ${sensor.lastUpdate}</p>`;
                        }
                        
                        sensorElement.innerHTML = sensorHTML;
                        sensorContainer.appendChild(sensorElement);
                    }
                })
                .catch(error => {
                    console.error('Erreur lors de la récupération des données:', error);
                    document.getElementById('sensor-data').innerHTML = '<p>Erreur lors du chargement des données</p>';
                });
        }
        
        // Charger les données au chargement de la page
        document.addEventListener('DOMContentLoaded', fetchSensorData);
        
        // Actualiser les données toutes les 30 secondes
        setInterval(fetchSensorData, 30000);
    </script>
</body>
</html>
bash
# Copier le fichier index.html vers le répertoire web de Nginx
sudo cp ~/agritech/frontend/index.html /var/www/html/

# Démarrer et activer Nginx
sudo systemctl enable nginx
sudo systemctl start nginx
Étape 9: Test du système complet
bash
# Tester que l'API répond
curl http://localhost:8080/

# Vérifier les données capteurs
curl http://localhost:8080/api/sensors | python3 -m json.tool

# Vérifier les messages MQTT
mosquitto_sub -h localhost -t "agritech/#" -v

# Publier un message de test
mosquitto_pub -h localhost -t "agritech/demo/test" -m '{"sensor_id":"TEST001", "location":"Laboratoire", "value": 25.0, "unit": "C", "timestamp": "'$(date -u +"%Y-%m-%dT%H:%M:%SZ")'"}'
Étape 10: Création d'une sauvegarde du projet
bash
# Sauvegarde de tous les fichiers du projet
cd ~
tar -czvf agritech_phase1_backup.tar.gz agritech/

# Sauvegarde des fichiers de configuration
sudo tar -czvf agritech_config_backup.tar.gz /etc/mosquitto/conf.d/default.conf /etc/systemd/system/agritech-api.service /etc/systemd/system/agritech-simulator.service /var/www/html/index.html
Vérification et dépannage
Si l'API Gateway ne démarre pas:
bash
# Vérifier les logs
sudo journalctl -u agritech-api -n 50

# Vérifier que Node.js est correctement installé
node --version
npm --version

# Démarrer manuellement pour voir les erreurs
cd ~/agritech/api_gateway
node server.js
Si le simulateur ne fonctionne pas:
bash
# Vérifier les logs
sudo journalctl -u agritech-simulator -n 50

# Vérifier que Python est installé
python3 --version

# Exécuter manuellement le script
cd ~/agritech/simulator
python3 sensor_simulator.py
Si la page web ne s'affiche pas:
bash
# Vérifier l'état de Nginx
sudo systemctl status nginx

# Vérifier les erreurs dans les logs Nginx
sudo tail /var/log/nginx/error.log

# S'assurer que le fichier HTML est accessible
ls -la /var/www/html/index.html
Notes importantes
Sécurité:

Cette configuration permet un accès anonyme au broker MQTT, ce qui est adapté pour un environnement de développement mais pas pour la production.
Dans un environnement de production, activez l'authentification et le chiffrement TLS.
Persistance des données:

Les données sont actuellement stockées en mémoire dans l'API Gateway.
Pour une persistance, envisagez d'ajouter une base de données (MongoDB, PostgreSQL).
Adaptation IP:

Assurez-vous de remplacer 192.168.1.3 par l'adresse IP réelle de votre serveur dans le fichier index.html.
Utilisateur:

Les services systemd sont configurés pour l'utilisateur sabiri. Adaptez ces configurations à votre nom d'utilisateur si nécessaire.
Ce guide complet vous permet de déployer rapidement l'infrastructure de la première phase du projet AgriTech Blockchain, établissant ainsi la base pour les phases futures qui intégreront la blockchain Hyperledger Fabric.