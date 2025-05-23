Guide de Démonstration - Projet AgriTech Blockchain (Phase 1)
**Date de mise à jour: 2025-05-02**
**Auteur: Otmane Sabiri**

## Préparation de la démonstration (5 min avant)

1. **Démarrer le terminal et se connecter au serveur**:
   ```bash
   ssh sabiri@192.168.1.3
   ```

2. **Vérifier les services et les démarrer si nécessaire**:
   ```bash
   # Vérifier MQTT
   sudo systemctl status mosquitto | grep Active
   
   # Vérifier Nginx
   sudo systemctl status nginx | grep Active
   
   # Démarrer manuellement l'API Gateway (laisser ce terminal ouvert)
   cd ~/agritech/api_gateway
   node server.js
   ```

3. **Ouvrir un second terminal pour les commandes de démonstration**:
   ```bash
   ssh sabiri@192.168.1.3
   ```

## Présentation du projet (3 min)

"Bienvenue à cette démonstration du projet AgriTech Blockchain. Aujourd'hui, je vais vous présenter la première phase de notre solution qui permet de collecter, traiter et visualiser les données de capteurs agricoles en temps réel.

Cette phase établit l'infrastructure de base sur laquelle nous intégrerons ultérieurement la blockchain Hyperledger Fabric pour garantir l'intégrité et la traçabilité des données agricoles."

## 1. Présentation de l'architecture (3 min)

"Notre système est composé de quatre composants principaux interconnectés:"

```bash
# Terminal 2: Afficher la structure du projet
echo -e "\n=== Architecture du système AgriTech ==="
echo "1. Simulateur Python → Génère des données de capteurs agricoles"
echo "2. Broker MQTT → Assure la communication entre composants"
echo "3. API REST → Traite et expose les données"
echo "4. Interface Web → Visualise les données pour l'utilisateur"

# Montrer la structure des fichiers
echo -e "\n=== Structure des fichiers ==="
find ~/agritech -type d | sort
```

## 2. Démonstration du flux de données (4 min)

"Observons maintenant comment les données circulent dans notre écosystème :"

```bash
# Terminal 2: Observer les messages MQTT en temps réel
echo -e "\n=== Écoute des messages MQTT en temps réel ==="
mosquitto_sub -h localhost -t "agritech/#" -v
```

Dans un troisième terminal:
```bash
# Terminal 3: Publier un message test manuellement
ssh sabiri@192.168.1.3

echo -e "\n=== Publication d'un message test ==="
mosquitto_pub -h localhost -t "agritech/demo/capteur_demonstration" -m '{"sensor_id":"DEMO001", "location":"Salle de présentation", "value": 24.5, "unit": "C", "timestamp": "'$(date -u +"%Y-%m-%dT%H:%M:%S")Z'"}'
```

"Vous pouvez observer dans le Terminal 2 que le message a été immédiatement reçu et traité par le broker MQTT."

## 3. Vérification des données via l'API (3 min)

Dans le Terminal 3:
```bash
# Vérifier que l'API répond correctement
echo -e "\n=== Test de l'API Gateway ==="
curl http://localhost:8080/

# Afficher les données des capteurs
echo -e "\n=== Données des capteurs via l'API REST ==="
curl http://localhost:8080/api/sensors | python3 -m json.tool
```

"Notre API REST expose ces données dans un format JSON structuré, rendant facile leur intégration avec d'autres applications ou systèmes."

## 4. Démonstration de l'interface web (5 min)

"Passons maintenant à l'interface utilisateur qui permet de visualiser ces données de manière intuitive:"

- Ouvrez un navigateur web et accédez à `http://192.168.1.3`

"Cette interface affiche en temps réel l'état de tous nos capteurs. Vous pouvez voir:
- La valeur actuelle de chaque capteur
- Sa localisation dans les différentes zones agricoles
- L'horodatage des dernières mises à jour
- Le type de capteur (température, humidité, pH, etc.)

L'interface se rafraîchit automatiquement toutes les 30 secondes, mais vous pouvez également cliquer sur le bouton 'Rafraîchir' pour une mise à jour immédiate."

## 5. Explication du code source (4 min)

```bash
# Terminal 3: Montrer le code du simulateur
echo -e "\n=== Code du simulateur de capteurs ==="
cat ~/agritech/simulator/sensor_simulator.py | head -n 30

# Montrer le code de l'API
echo -e "\n=== API Gateway Node.js ==="
cat ~/agritech/api_gateway/server.js | head -n 30
```

"Notre code est conçu pour être:
- Modulaire: chaque composant a une responsabilité unique
- Évolutif: facile à étendre avec de nouveaux types de capteurs
- Robuste: gestion des erreurs et redémarrage automatique
- Léger: optimisé pour s'exécuter sur des appareils à ressources limitées"

## 6. Publication de données supplémentaires (3 min)

Dans le Terminal 3:
```bash
# Publier plusieurs messages pour montrer la mise à jour en temps réel
echo -e "\n=== Publication de données supplémentaires ==="
mosquitto_pub -h localhost -t "agritech/demo/humidite" -m '{"sensor_id":"HUMID_DEMO", "location":"Serre principale", "value": 65.7, "unit": "%", "timestamp": "'$(date -u +"%Y-%m-%dT%H:%M:%S")Z'"}'

# Attendre 5 secondes
sleep 5

mosquitto_pub -h localhost -t "agritech/demo/sol" -m '{"sensor_id":"SOIL_DEMO", "location":"Parcelle A", "value": 42.3, "unit": "%", "timestamp": "'$(date -u +"%Y-%m-%dT%H:%M:%S")Z'"}'
```

"Rafraîchissez l'interface web pour voir ces nouvelles données apparaître instantanément."

## 7. Conclusion et prochaines étapes (3 min)

"Pour résumer cette première phase du projet AgriTech Blockchain:

1. Nous avons mis en place une infrastructure complète de collecte de données IoT
2. Le système utilise le protocole MQTT, idéal pour les environnements IoT à faible bande passante
3. L'API REST permet d'intégrer facilement ces données avec d'autres systèmes
4. L'interface web offre une visualisation claire et intuitive des données des capteurs

Les prochaines phases de notre projet comprendront:
- L'intégration d'Hyperledger Fabric sur la VM2 (Zone Centrale)
- Le déploiement de capteurs réels sur la VM1 (Zone Terrain)
- L'implémentation de contrats intelligents pour la traçabilité des données agricoles
- L'ajout d'une application mobile pour les agriculteurs sur le terrain

Cette première phase constitue la fondation solide sur laquelle nous bâtirons notre solution complète de traçabilité agricole sécurisée par blockchain."

## 8. Questions et réponses (5 min)

"Je suis maintenant disponible pour répondre à vos questions sur cette première phase du projet AgriTech Blockchain."

## Aide-mémoire pour les questions fréquentes

- **Sécurité**: "Dans la phase de production, nous implémenterons l'authentification MQTT et le chiffrement TLS. Pour l'API, nous ajouterons une authentification par JWT."
  
- **Scalabilité**: "L'architecture MQTT peut gérer des milliers de capteurs simultanément. Pour un déploiement à très grande échelle, nous envisageons d'ajouter une couche Kafka."
  
- **Persistance des données**: "Actuellement, les données sont stockées en mémoire. La prochaine étape intégrera une base de données NoSQL pour le stockage à long terme et la blockchain pour l'immuabilité."
  
- **Optimisation pour environnements ruraux**: "Le protocole MQTT a été choisi spécifiquement pour sa faible consommation de bande passante, idéale pour les zones rurales avec une connectivité limitée."

- **Coût de déploiement**: "L'architecture est conçue pour fonctionner sur du matériel abordable comme des Raspberry Pi, rendant la solution accessible aux petites exploitations agricoles."