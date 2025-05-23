Guide de Réalisation - Projet AgriTech Blockchain (Phase 2 - Extension VM1 Zone Terrain)
**Date: 2025-05-02**
**Auteur: Otmane Sabiri**

## Introduction

Cette deuxième phase du projet vise à étendre les fonctionnalités de la Zone Terrain (VM1) en ajoutant des capacités de traitement avancé, de sécurité, de stockage local et de préparation à l'intégration blockchain. Ces améliorations permettront une meilleure fiabilité, une gestion des alertes, et une préparation des données pour une intégration future avec Hyperledger Fabric.

## Prérequis
- VM1 avec la Phase 1 déjà configurée et fonctionnelle
- VM3 (Zone Clients) opérationnelle
- Accès sudo sur VM1

## Architecture étendue de la Zone Terrain
```
┌─────────────────────────────────────────────────────────────────────┐
│                   VM1 (Zone Terrain - Phase 2)                      │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ ┌────────────┐   ┌────────────┐   ┌──────────────┐   ┌──────────┐   │
│ │  Capteurs  │──▶│Processeur  │──▶│ Gestionnaire │──▶│  Client  │   │
│ │  Simulés   │   │de données  │   │  de données  │   │   MQTT   │──┐│
│ └────────────┘   └────────────┘   └──────────────┘   └──────────┘  ││
│        │              │                  │                │         ││
│        ▼              ▼                  ▼                ▼         ││
│ ┌────────────┐   ┌────────────┐   ┌──────────────┐   ┌──────────┐  ││
│ │ Détection  │   │  Base de   │   │ Gestionnaire │   │ Module   │  ││
│ │ d'anomalies│   │  données   │   │  d'alertes   │   │ sécurité │  ││
│ └────────────┘   └────────────┘   └──────────────┘   └──────────┘  ││
│                                                                     ││
└─────────────────────────────────────────────────────────────────────┘│
                                                                        │
                                                                        │
                        ┌────────────┐                                  │
                        │            │                                  │
                        │  VM3       │◀─────────────────────────────────┘
                        │ (Zone      │
                        │ Clients)   │
                        │            │
                        └────────────┘
```

## Étape 1: Installation des dépendances supplémentaires

```bash
# Mise à jour du système
sudo apt update && sudo apt upgrade -y

# Installation des packages nécessaires supplémentaires
sudo apt install -y python3-pip sqlite3 python3-cryptography python3-flask python3-matplotlib python3-pandas supervisor

# Installation des bibliothèques Python additionnelles
pip3 install pycryptodome influxdb-client fastapi uvicorn pyjwt SQLAlchemy pandas numpy scikit-learn==1.2.2 apscheduler
```

## Étape 2: Mise en place d'une base de données locale SQLite

```bash
# Création du répertoire pour la base de données
mkdir -p ~/agritech-terrain/database

# Création du module de gestion de base de données
cat > ~/agritech-terrain/utils/database_manager.py << 'EOL'
"""
Module de gestion de la base de données locale
"""
import os
import sqlite3
import json
import logging
from datetime import datetime, timedelta
import pandas as pd
from typing import Dict, List, Any, Optional, Tuple

# Configuration du logging
logger = logging.getLogger("database_manager")

class DatabaseManager:
    """Gestionnaire de base de données SQLite pour le stockage local des données"""
    
    def __init__(self, db_path: str):
        """
        Initialise le gestionnaire de base de données
        
        Args:
            db_path: Chemin vers le fichier de base de données SQLite
        """
        self.db_path = db_path
        self._init_db()
    
    def _init_db(self) -> None:
        """Initialise la structure de la base de données"""
        logger.info(f"Initialisation de la base de données: {self.db_path}")
        try:
            # Création du répertoire parent si nécessaire
            os.makedirs(os.path.dirname(self.db_path), exist_ok=True)
            
            # Connexion à la base de données
            conn = sqlite3.connect(self.db_path)
            cursor = conn.cursor()
            
            # Table des données de capteurs
            cursor.execute('''
            CREATE TABLE IF NOT EXISTS sensor_data (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                sensor_id TEXT NOT NULL,
                sensor_type TEXT NOT NULL,
                location TEXT NOT NULL,
                value REAL NOT NULL,
                unit TEXT NOT NULL,
                status TEXT NOT NULL,
                timestamp TEXT NOT NULL,
                is_synced INTEGER DEFAULT 0,
                hash TEXT,
                created_at TEXT DEFAULT CURRENT_TIMESTAMP
            )
            ''')
            
            # Table des métadonnées des capteurs
            cursor.execute('''
            CREATE TABLE IF NOT EXISTS sensor_metadata (
                sensor_id TEXT PRIMARY KEY,
                sensor_type TEXT NOT NULL,
                location TEXT NOT NULL,
                unit TEXT NOT NULL,
                min_value REAL NOT NULL,
                max_value REAL NOT NULL,
                normal_min REAL NOT NULL,
                normal_max REAL NOT NULL,
                description TEXT,
                last_updated TEXT DEFAULT CURRENT_TIMESTAMP
            )
            ''')
            
            # Table des alertes
            cursor.execute('''
            CREATE TABLE IF NOT EXISTS alerts (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                sensor_id TEXT NOT NULL,
                alert_type TEXT NOT NULL,
                severity TEXT NOT NULL,
                message TEXT NOT NULL,
                value REAL,
                threshold REAL,
                is_active INTEGER DEFAULT 1,
                created_at TEXT NOT NULL,
                resolved_at TEXT,
                FOREIGN KEY (sensor_id) REFERENCES sensor_metadata (sensor_id)
            )
            ''')
            
            # Table des statistiques
            cursor.execute('''
            CREATE TABLE IF NOT EXISTS statistics (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                sensor_id TEXT NOT NULL,
                period TEXT NOT NULL,
                start_time TEXT NOT NULL,
                end_time TEXT NOT NULL,
                min_value REAL,
                max_value REAL,
                avg_value REAL,
                std_dev REAL,
                data_points INTEGER,
                created_at TEXT DEFAULT CURRENT_TIMESTAMP,
                FOREIGN KEY (sensor_id) REFERENCES sensor_metadata (sensor_id)
            )
            ''')
            
            # Index pour améliorer les performances
            cursor.execute('CREATE INDEX IF NOT EXISTS idx_sensor_data_sensor_id ON sensor_data (sensor_id)')
            cursor.execute('CREATE INDEX IF NOT EXISTS idx_sensor_data_timestamp ON sensor_data (timestamp)')
            cursor.execute('CREATE INDEX IF NOT EXISTS idx_alerts_sensor_id ON alerts (sensor_id)')
            cursor.execute('CREATE INDEX IF NOT EXISTS idx_statistics_sensor_id ON statistics (sensor_id)')
            
            conn.commit()
            conn.close()
            
            logger.info("Base de données initialisée avec succès")
        
        except Exception as e:
            logger.error(f"Erreur lors de l'initialisation de la base de données: {e}")
            raise
    
    def save_sensor_data(self, data: Dict[str, Any]) -> int:
        """
        Enregistre les données d'un capteur dans la base de données
        
        Args:
            data: Dictionnaire contenant les données du capteur
            
        Returns:
            int: ID de l'enregistrement créé
        """
        try:
            conn = sqlite3.connect(self.db_path)
            cursor = conn.cursor()
            
            # Insertion des données
            cursor.execute('''
            INSERT INTO sensor_data (
                sensor_id, sensor_type, location, value, unit, status, timestamp, is_synced, hash
            ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
            ''', (
                data['sensor_id'],
                data['type'],
                data['location'],
                data['value'],
                data['unit'],
                data['status'],
                data['timestamp'],
                1 if data.get('is_synced', False) else 0,
                data.get('hash')
            ))
            
            last_id = cursor.lastrowid
            conn.commit()
            conn.close()
            
            logger.debug(f"Données du capteur {data['sensor_id']} enregistrées, ID: {last_id}")
            return last_id
        
        except Exception as e:
            logger.error(f"Erreur lors de l'enregistrement des données du capteur: {e}")
            raise
    
    def save_sensor_metadata(self, metadata: Dict[str, Any]) -> bool:
        """
        Enregistre ou met à jour les métadonnées d'un capteur
        
        Args:
            metadata: Dictionnaire contenant les métadonnées du capteur
            
        Returns:
            bool: True si l'opération a réussi, False sinon
        """
        try:
            conn = sqlite3.connect(self.db_path)
            cursor = conn.cursor()
            
            # Vérifier si le capteur existe déjà
            cursor.execute('SELECT 1 FROM sensor_metadata WHERE sensor_id = ?', (metadata['sensor_id'],))
            exists = cursor.fetchone() is not None
            
            if exists:
                # Mettre à jour les métadonnées existantes
                cursor.execute('''
                UPDATE sensor_metadata SET
                    sensor_type = ?,
                    location = ?,
                    unit = ?,
                    min_value = ?,
                    max_value = ?,
                    normal_min = ?,
                    normal_max = ?,
                    description = ?,
                    last_updated = CURRENT_TIMESTAMP
                WHERE sensor_id = ?
                ''', (
                    metadata['type'],
                    metadata['location'],
                    metadata['unit'],
                    metadata['min_value'],
                    metadata['max_value'],
                    metadata['normal_range'][0],
                    metadata['normal_range'][1],
                    metadata.get('description', ''),
                    metadata['sensor_id']
                ))
            else:
                # Insérer nouvelles métadonnées
                cursor.execute('''
                INSERT INTO sensor_metadata (
                    sensor_id, sensor_type, location, unit, min_value, max_value, normal_min, normal_max, description
                ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
                ''', (
                    metadata['sensor_id'],
                    metadata['type'],
                    metadata['location'],
                    metadata['unit'],
                    metadata['min_value'],
                    metadata['max_value'],
                    metadata['normal_range'][0],
                    metadata['normal_range'][1],
                    metadata.get('description', '')
                ))
            
            conn.commit()
            conn.close()
            
            logger.info(f"Métadonnées du capteur {metadata['sensor_id']} {'mises à jour' if exists else 'enregistrées'}")
            return True
        
        except Exception as e:
            logger.error(f"Erreur lors de l'enregistrement des métadonnées du capteur: {e}")
            return False
    
    def get_unsync_data(self, limit: int = 100) -> List[Dict[str, Any]]:
        """
        Récupère les données non synchronisées
        
        Args:
            limit: Nombre maximum d'enregistrements à récupérer
            
        Returns:
            List: Liste des données non synchronisées
        """
        try:
            conn = sqlite3.connect(self.db_path)
            conn.row_factory = sqlite3.Row
            cursor = conn.cursor()
            
            cursor.execute('''
            SELECT * FROM sensor_data
            WHERE is_synced = 0
            ORDER BY timestamp ASC
            LIMIT ?
            ''', (limit,))
            
            rows = cursor.fetchall()
            conn.close()
            
            # Convertir les résultats en dictionnaires
            result = [dict(row) for row in rows]
            logger.info(f"Récupération de {len(result)} enregistrements non synchronisés")
            
            return result
        
        except Exception as e:
            logger.error(f"Erreur lors de la récupération des données non synchronisées: {e}")
            return []
    
    def mark_as_synced(self, record_ids: List[int]) -> bool:
        """
        Marque les enregistrements comme synchronisés
        
        Args:
            record_ids: Liste des IDs d'enregistrements à marquer
            
        Returns:
            bool: True si l'opération a réussi, False sinon
        """
        if not record_ids:
            return True
            
        try:
            conn = sqlite3.connect(self.db_path)
            cursor = conn.cursor()
            
            # Utiliser des placeholders pour une requête sécurisée
            placeholders = ','.join('?' for _ in record_ids)
            cursor.execute(f'''
            UPDATE sensor_data
            SET is_synced = 1
            WHERE id IN ({placeholders})
            ''', record_ids)
            
            conn.commit()
            conn.close()
            
            logger.info(f"{cursor.rowcount} enregistrements marqués comme synchronisés")
            return True
        
        except Exception as e:
            logger.error(f"Erreur lors du marquage des enregistrements: {e}")
            return False
    
    def save_alert(self, alert_data: Dict[str, Any]) -> int:
        """
        Enregistre une alerte dans la base de données
        
        Args:
            alert_data: Données de l'alerte
            
        Returns:
            int: ID de l'alerte créée
        """
        try:
            conn = sqlite3.connect(self.db_path)
            cursor = conn.cursor()
            
            cursor.execute('''
            INSERT INTO alerts (
                sensor_id, alert_type, severity, message, value, threshold, is_active, created_at
            ) VALUES (?, ?, ?, ?, ?, ?, ?, ?)
            ''', (
                alert_data['sensor_id'],
                alert_data['alert_type'],
                alert_data['severity'],
                alert_data['message'],
                alert_data.get('value'),
                alert_data.get('threshold'),
                1,  # is_active
                alert_data.get('created_at', datetime.now().isoformat())
            ))
            
            alert_id = cursor.lastrowid
            conn.commit()
            conn.close()
            
            logger.info(f"Alerte enregistrée pour le capteur {alert_data['sensor_id']}, ID: {alert_id}")
            return alert_id
        
        except Exception as e:
            logger.error(f"Erreur lors de l'enregistrement de l'alerte: {e}")
            return -1
    
    def resolve_alert(self, alert_id: int) -> bool:
        """
        Marque une alerte comme résolue
        
        Args:
            alert_id: ID de l'alerte à résoudre
            
        Returns:
            bool: True si l'opération a réussi, False sinon
        """
        try:
            conn = sqlite3.connect(self.db_path)
            cursor = conn.cursor()
            
            cursor.execute('''
            UPDATE alerts
            SET is_active = 0, resolved_at = ?
            WHERE id = ?
            ''', (datetime.now().isoformat(), alert_id))
            
            conn.commit()
            conn.close()
            
            logger.info(f"Alerte {alert_id} marquée comme résolue")
            return True
        
        except Exception as e:
            logger.error(f"Erreur lors de la résolution de l'alerte: {e}")
            return False
    
    def get_active_alerts(self) -> List[Dict[str, Any]]:
        """
        Récupère toutes les alertes actives
        
        Returns:
            List: Liste des alertes actives
        """
        try:
            conn = sqlite3.connect(self.db_path)
            conn.row_factory = sqlite3.Row
            cursor = conn.cursor()
            
            cursor.execute('''
            SELECT * FROM alerts
            WHERE is_active = 1
            ORDER BY created_at DESC
            ''')
            
            rows = cursor.fetchall()
            conn.close()
            
            # Convertir les résultats en dictionnaires
            result = [dict(row) for row in rows]
            logger.debug(f"Récupération de {len(result)} alertes actives")
            
            return result
        
        except Exception as e:
            logger.error(f"Erreur lors de la récupération des alertes actives: {e}")
            return []
    
    def compute_statistics(self, period: str = 'hour') -> bool:
        """
        Calcule et stocke les statistiques pour tous les capteurs
        
        Args:
            period: Période pour laquelle calculer les statistiques ('hour', 'day', 'week')
            
        Returns:
            bool: True si l'opération a réussi, False sinon
        """
        try:
            # Déterminer l'intervalle de temps
            now = datetime.now()
            if period == 'hour':
                start_time = now - timedelta(hours=1)
                period_format = "%Y-%m-%d %H:00:00"
            elif period == 'day':
                start_time = now - timedelta(days=1)
                period_format = "%Y-%m-%d 00:00:00"
            elif period == 'week':
                start_time = now - timedelta(weeks=1)
                period_format = "%Y-%m-%d 00:00:00"  # Début de la semaine
            else:
                logger.error(f"Période non prise en charge: {period}")
                return False
            
            # Formater les dates pour la requête SQL
            start_str = start_time.isoformat().replace('T', ' ')
            end_str = now.isoformat().replace('T', ' ')
            period_str = start_time.strftime(period_format)
            
            conn = sqlite3.connect(self.db_path)
            conn.row_factory = sqlite3.Row
            cursor = conn.cursor()
            
            # Récupérer tous les capteurs
            cursor.execute('SELECT DISTINCT sensor_id FROM sensor_metadata')
            sensor_ids = [row['sensor_id'] for row in cursor.fetchall()]
            
            for sensor_id in sensor_ids:
                # Récupérer les données du capteur pour la période
                cursor.execute('''
                SELECT * FROM sensor_data
                WHERE sensor_id = ? AND timestamp BETWEEN ? AND ?
                ORDER BY timestamp
                ''', (sensor_id, start_str, end_str))
                
                rows = cursor.fetchall()
                if not rows:
                    logger.debug(f"Pas de données pour le capteur {sensor_id} pour la période {period}")
                    continue
                
                # Convertir en DataFrame pour faciliter le calcul des statistiques
                df = pd.DataFrame([dict(row) for row in rows])
                
                # Calculer les statistiques
                stats = {
                    'sensor_id': sensor_id,
                    'period': period,
                    'start_time': start_str,
                    'end_time': end_str,
                    'min_value': float(df['value'].min()),
                    'max_value': float(df['value'].max()),
                    'avg_value': float(df['value'].mean()),
                    'std_dev': float(df['value'].std() if len(df) > 1 else 0),
                    'data_points': len(df)
                }
                
                # Enregistrer les statistiques
                cursor.execute('''
                INSERT INTO statistics (
                    sensor_id, period, start_time, end_time, min_value, max_value, avg_value, std_dev, data_points
                ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)
                ''', (
                    stats['sensor_id'],
                    stats['period'],
                    stats['start_time'],
                    stats['end_time'],
                    stats['min_value'],
                    stats['max_value'],
                    stats['avg_value'],
                    stats['std_dev'],
                    stats['data_points']
                ))
                
                logger.debug(f"Statistiques calculées pour le capteur {sensor_id}, période: {period}")
            
            conn.commit()
            conn.close()
            
            logger.info(f"Statistiques calculées pour tous les capteurs, période: {period}")
            return True
        
        except Exception as e:
            logger.error(f"Erreur lors du calcul des statistiques: {e}")
            return False
    
    def get_sensor_data_history(self, sensor_id: str, start_time: Optional[str] = None, 
                               end_time: Optional[str] = None, limit: int = 1000) -> List[Dict[str, Any]]:
        """
        Récupère l'historique des données d'un capteur
        
        Args:
            sensor_id: ID du capteur
            start_time: Horodatage de début (format ISO)
            end_time: Horodatage de fin (format ISO)
            limit: Nombre maximum d'enregistrements à récupérer
            
        Returns:
            List: Liste des données historiques
        """
        try:
            conn = sqlite3.connect(self.db_path)
            conn.row_factory = sqlite3.Row
            cursor = conn.cursor()
            
            query = 'SELECT * FROM sensor_data WHERE sensor_id = ?'
            params = [sensor_id]
            
            if start_time:
                query += ' AND timestamp >= ?'
                params.append(start_time)
            
            if end_time:
                query += ' AND timestamp <= ?'
                params.append(end_time)
            
            query += ' ORDER BY timestamp DESC LIMIT ?'
            params.append(limit)
            
            cursor.execute(query, params)
            rows = cursor.fetchall()
            conn.close()
            
            # Convertir les résultats en dictionnaires
            result = [dict(row) for row in rows]
            logger.debug(f"Récupération de {len(result)} enregistrements pour le capteur {sensor_id}")
            
            return result
        
        except Exception as e:
            logger.error(f"Erreur lors de la récupération de l'historique: {e}")
            return []
    
    def get_latest_sensor_data(self) -> Dict[str, Dict[str, Any]]:
        """
        Récupère les dernières données de tous les capteurs
        
        Returns:
            Dict: Dictionnaire des dernières données par capteur
        """
        try:
            conn = sqlite3.connect(self.db_path)
            conn.row_factory = sqlite3.Row
            cursor = conn.cursor()
            
            # Sous-requête pour obtenir l'ID maximum par capteur
            cursor.execute('''
            SELECT s1.*
            FROM sensor_data s1
            JOIN (
                SELECT sensor_id, MAX(timestamp) as max_timestamp
                FROM sensor_data
                GROUP BY sensor_id
            ) s2
            ON s1.sensor_id = s2.sensor_id AND s1.timestamp = s2.max_timestamp
            ''')
            
            rows = cursor.fetchall()
            conn.close()
            
            # Organiser les résultats par capteur
            result = {}
            for row in rows:
                row_dict = dict(row)
                result[row_dict['sensor_id']] = row_dict
            
            logger.debug(f"Récupération des dernières données pour {len(result)} capteurs")
            return result
        
        except Exception as e:
            logger.error(f"Erreur lors de la récupération des dernières données: {e}")
            return {}
    
    def get_sensor_metadata(self, sensor_id: Optional[str] = None) -> List[Dict[str, Any]]:
        """
        Récupère les métadonnées d'un ou de tous les capteurs
        
        Args:
            sensor_id: ID du capteur (si None, retourne tous les capteurs)
            
        Returns:
            List: Liste des métadonnées
        """
        try:
            conn = sqlite3.connect(self.db_path)
            conn.row_factory = sqlite3.Row
            cursor = conn.cursor()
            
            if sensor_id:
                cursor.execute('SELECT * FROM sensor_metadata WHERE sensor_id = ?', (sensor_id,))
            else:
                cursor.execute('SELECT * FROM sensor_metadata')
            
            rows = cursor.fetchall()
            conn.close()
            
            # Convertir les résultats en dictionnaires
            result = [dict(row) for row in rows]
            
            # Reformater les résultats pour correspondre au format d'origine
            for item in result:
                item['type'] = item.pop('sensor_type')
                item['normal_range'] = (item.pop('normal_min'), item.pop('normal_max'))
            
            return result
        
        except Exception as e:
            logger.error(f"Erreur lors de la récupération des métadonnées: {e}")
            return []
    
    def cleanup_old_data(self, days_to_keep: int = 30) -> int:
        """
        Supprime les données plus anciennes qu'un certain nombre de jours
        
        Args:
            days_to_keep: Nombre de jours de données à conserver
            
        Returns:
            int: Nombre d'enregistrements supprimés
        """
        try:
            conn = sqlite3.connect(self.db_path)
            cursor = conn.cursor()
            
            # Calculer la date limite
            cutoff_date = (datetime.now() - timedelta(days=days_to_keep)).isoformat()
            
            # Supprimer les anciennes données
            cursor.execute('DELETE FROM sensor_data WHERE timestamp < ?', (cutoff_date,))
            deleted_count = cursor.rowcount
            
            # Supprimer les anciennes statistiques
            cursor.execute('DELETE FROM statistics WHERE end_time < ?', (cutoff_date,))
            
            # Supprimer les anciennes alertes résolues
            cursor.execute('DELETE FROM alerts WHERE is_active = 0 AND resolved_at < ?', (cutoff_date,))
            
            conn.commit()
            conn.close()
            
            logger.info(f"Nettoyage terminé: {deleted_count} enregistrements de données supprimés")
            return deleted_count
        
        except Exception as e:
            logger.error(f"Erreur lors du nettoyage des anciennes données: {e}")
            return -1
EOL
```

## Étape 3: Création du module de détection d'anomalies

```bash
# Création du module de détection d'anomalies
cat > ~/agritech-terrain/utils/anomaly_detector.py << 'EOL'
"""
Module de détection d'anomalies dans les données des capteurs
"""
import os
import sys
import json
import logging
import numpy as np
from typing import Dict, List, Any, Tuple, Optional
from datetime import datetime, timedelta
import pickle
from sklearn.ensemble import IsolationForest

# Configuration du logging
logger = logging.getLogger("anomaly_detector")

class AnomalyDetector:
    """Détecteur d'anomalies pour les données des capteurs"""
    
    def __init__(self, models_dir: str):
        """
        Initialise le détecteur d'anomalies
        
        Args:
            models_dir: Répertoire pour stocker les modèles entraînés
        """
        self.models_dir = models_dir
        self.models = {}
        self.threshold = 2.5  # Facteur pour le Z-score
        self.window_size = 20  # Nombre de points pour la détection
        
        # Créer le répertoire des modèles s'il n'existe pas
        if not os.path.exists(self.models_dir):
            os.makedirs(self.models_dir)
        
        # Charger les modèles existants
        self._load_models()
    
    def _load_models(self) -> None:
        """Charge les modèles existants depuis le disque"""
        try:
            for filename in os.listdir(self.models_dir):
                if filename.endswith('.pkl'):
                    sensor_id = filename.split('_')[0]
                    model_path = os.path.join(self.models_dir, filename)
                    
                    with open(model_path, 'rb') as f:
                        self.models[sensor_id] = pickle.load(f)
                    
                    logger.info(f"Modèle chargé pour le capteur {sensor_id}")
        
        except Exception as e:
            logger.error(f"Erreur lors du chargement des modèles: {e}")
    
    def _save_model(self, sensor_id: str, model) -> None:
        """
        Sauvegarde un modèle sur le disque
        
        Args:
            sensor_id: ID du capteur
            model: Modèle à sauvegarder
        """
        try:
            model_path = os.path.join(self.models_dir, f"{sensor_id}_model.pkl")
            
            with open(model_path, 'wb') as f:
                pickle.dump(model, f)
            
            logger.info(f"Modèle sauvegardé pour le capteur {sensor_id}")
        
        except Exception as e:
            logger.error(f"Erreur lors de la sauvegarde du modèle: {e}")
    
    def train_model(self, sensor_id: str, historical_data: List[Dict[str, Any]]) -> bool:
        """
        Entraîne un modèle pour un capteur spécifique
        
        Args:
            sensor_id: ID du capteur
            historical_data: Données historiques du capteur
            
        Returns:
            bool: True si l'entraînement a réussi, False sinon
        """
        if len(historical_data) < 50:
            logger.warning(f"Pas assez de données pour entraîner un modèle pour {sensor_id} ({len(historical_data)} points)")
            return False
        
        try:
            # Extraire les valeurs
            values = np.array([item['value'] for item in historical_data]).reshape(-1, 1)
            
            # Entraîner un modèle Isolation Forest
            model = IsolationForest(n_estimators=100, contamination=0.05, random_state=42)
            model.fit(values)
            
            # Sauvegarder le modèle
            self.models[sensor_id] = model
            self._save_model(sensor_id, model)
            
            logger.info(f"Modèle entraîné pour le capteur {sensor_id} avec {len(historical_data)} points de données")
            return True
        
        except Exception as e:
            logger.error(f"Erreur lors de l'entraînement du modèle pour {sensor_id}: {e}")
            return False
    
    def detect_anomaly(self, sensor_id: str, value: float, historical_data: Optional[List[Dict[str, Any]]] = None) -> Dict[str, Any]:
        """
        Détecte si une valeur est anormale
        
        Args:
            sensor_id: ID du capteur
            value: Valeur à vérifier
            historical_data: Données historiques (optionnel, pour les méthodes simples)
            
        Returns:
            Dict: Résultat de la détection avec {is_anomaly, score, method}
        """
        result = {
            'is_anomaly': False,
            'score': 0.0,
            'method': 'none',
            'confidence': 0.0,
            'threshold': 0.0
        }
        
        # Méthode 1: Utiliser le modèle d'apprentissage automatique si disponible
        if sensor_id in self.models:
            try:
                model = self.models[sensor_id]
                # Prédire l'anomalie (-1 pour anomalie, 1 pour normal)
                prediction = model.predict(np.array([[value]]))
                # Calculer le score d'anomalie
                score = model.score_samples(np.array([[value]]))
                
                # Normaliser le score entre 0 et 1 (plus il est proche de 0, plus c'est anormal)
                normalized_score = np.exp(score[0])
                is_anomaly = prediction[0] == -1
                
                result = {
                    'is_anomaly': is_anomaly,
                    'score': float(normalized_score),
                    'method': 'isolation_forest',
                    'confidence': 0.85,
                    'threshold': 0.5
                }
                
                logger.debug(f"Détection par Isolation Forest pour {sensor_id}: {value} -> {'Anomalie' if is_anomaly else 'Normal'} (score: {normalized_score:.3f})")
                return result
            
            except Exception as e:
                logger.error(f"Erreur lors de la détection par modèle pour {sensor_id}: {e}")
                # Continuer avec les autres méthodes en cas d'erreur
        
        # Méthode 2: Utiliser le Z-score si des données historiques sont disponibles
        if historical_data and len(historical_data) >= 10:
            try:
                # Extraire les valeurs récentes
                values = np.array([item['value'] for item in historical_data[-self.window_size:]])
                mean = np.mean(values)
                std = np.std(values)
                
                # Si l'écart-type est très petit, ajuster pour éviter division par zéro
                if std < 0.0001:
                    std = 0.0001
                
                # Calculer le Z-score
                z_score = abs((value - mean) / std)
                is_anomaly = z_score > self.threshold
                
                result = {
                    'is_anomaly': is_anomaly,
                    'score': float(z_score),
                    'method': 'z_score',
                    'confidence': 0.75,
                    'threshold': self.threshold
                }
                
                logger.debug(f"Détection par Z-score pour {sensor_id}: {value} -> {'Anomalie' if is_anomaly else 'Normal'} (score: {z_score:.3f}, seuil: {self.threshold})")
                return result
            
            except Exception as e:
                logger.error(f"Erreur lors de la détection par Z-score pour {sensor_id}: {e}")
        
        # Méthode 3: Vérification simple des limites si aucune autre méthode n'est disponible
        try:
            # Récupérer les métadonnées du capteur (si disponibles)
            metadata = None
            if historical_data and len(historical_data) > 0:
                # Supposer que les métadonnées sont disponibles dans les données historiques
                first_item = historical_data[0]
                if 'min_value' in first_item and 'max_value' in first_item:
                    metadata = first_item
            
            if metadata:
                min_val = metadata.get('min_value', 0)
                max_val = metadata.get('max_value', 100)
                normal_min = metadata.get('normal_min', min_val)
                normal_max = metadata.get('normal_max', max_val)
                
                # Vérifier si la valeur est en dehors des limites normales
                is_anomaly = value < normal_min or value > normal_max
                
                # Calculer un score basé sur la distance aux limites normales
                if is_anomaly:
                    if value < normal_min:
                        score = (normal_min - value) / (normal_min - min_val) if normal_min > min_val else 1.0
                    else:
                        score = (value - normal_max) / (max_val - normal_max) if max_val > normal_max else 1.0
                else:
                    score = 0.0
                
                result = {
                    'is_anomaly': is_anomaly,
                    'score': float(min(score, 1.0)),  # Limiter le score à 1.0
                    'method': 'threshold',
                    'confidence': 0.6,
                    'threshold': 0.0
                }
                
                logger.debug(f"Détection par seuil pour {sensor_id}: {value} -> {'Anomalie' if is_anomaly else 'Normal'} (score: {score:.3f})")
                return result
        
        except Exception as e:
            logger.error(f"Erreur lors de la détection par seuil pour {sensor_id}: {e}")
        
        # Par défaut, considérer comme normal
        logger.debug(f"Aucune méthode de détection disponible pour {sensor_id}, considéré comme normal")
        return result

    def get_anomaly_explanation(self, sensor_id: str, value: float, detection_result: Dict[str, Any], 
                              metadata: Optional[Dict[str, Any]] = None) -> str:
        """
        Génère une explication textuelle pour une anomalie détectée
        
        Args:
            sensor_id: ID du capteur
            value: Valeur anormale
            detection_result: Résultat de la détection d'anomalie
            metadata: Métadonnées du capteur (optionnel)
            
        Returns:
            str: Explication textuelle
        """
        if not detection_result['is_anomaly']:
            return "Aucune anomalie détectée."
        
        method = detection_result['method']
        score = detection_result['score']
        
        if method == 'isolation_forest':
            explanation = (
                f"Une anomalie a été détectée pour le capteur {sensor_id} avec une valeur de {value}. "
                f"Cette valeur est considérée comme anormale par le modèle d'apprentissage automatique "
                f"avec un score d'anomalie de {score:.3f} (plus le score est bas, plus l'anomalie est forte)."
            )
        
        elif method == 'z_score':
            explanation = (
                f"Une anomalie a été détectée pour le capteur {sensor_id} avec une valeur de {value}. "
                f"Cette valeur s'écarte de {score:.2f} écarts-types de la moyenne récente, "
                f"ce qui est supérieur au seuil de {detection_result['threshold']}."
            )
        
        elif method == 'threshold':
            # Ajouter des informations sur les limites si les métadonnées sont disponibles
            if metadata:
                unit = metadata.get('unit', '')
                normal_min = metadata.get('normal_min', metadata.get('normal_range', (0, 0))[0])
                normal_max = metadata.get('normal_max', metadata.get('normal_range', (0, 0))[1])
                
                explanation = (
                    f"Une anomalie a été détectée pour le capteur {sensor_id} avec une valeur de {value} {unit}. "
                    f"Cette valeur est en dehors de la plage normale attendue de {normal_min} à {normal_max} {unit}."
                )
            else:
                explanation = (
                    f"Une anomalie a été détectée pour le capteur {sensor_id} avec une valeur de {value}. "
                    f"Cette valeur est en dehors de la plage normale attendue."
                )
        
        else:
            explanation = (
                f"Une anomalie a été détectée pour le capteur {sensor_id} avec une valeur de {value}, "
                f"mais la méthode de détection n'est pas spécifiée."
            )
        
        return explanation
EOL
```

## Étape 4: Création du module de sécurité

```bash
# Création du module de sécurité
cat > ~/agritech-terrain/utils/security_manager.py << 'EOL'
"""
Module de sécurité pour l'authentification et la signature des données
"""
import os
import json
import base64
import hashlib
import logging
import time
from typing import Dict, Any, Tuple, Optional
import jwt
from Crypto.PublicKey import RSA
from Crypto.Signature import pkcs1_15
from Crypto.Hash import SHA256
from datetime import datetime, timedelta

# Configuration du logging
logger = logging.getLogger("security_manager")

class SecurityManager:
    """Gestionnaire de sécurité pour l'authentification et la signature des données"""
    
    def __init__(self, keys_dir: str):
        """
        Initialise le gestionnaire de sécurité
        
        Args:
            keys_dir: Répertoire pour stocker les clés
        """
        self.keys_dir = keys_dir
        self.private_key_path = os.path.join(keys_dir, 'private_key.pem')
        self.public_key_path = os.path.join(keys_dir, 'public_key.pem')
        self.private_key = None
        self.public_key = None
        self.device_id = self._generate_device_id()
        self.jwt_secret = os.environ.get('JWT_SECRET', 'agritech-blockchain-secret-key-2025')
        
        # Créer le répertoire des clés s'il n'existe pas
        if not os.path.exists(self.keys_dir):
            os.makedirs(self.keys_dir)
        
        # Générer ou charger les clés
        self._load_or_generate_keys()
    
    def _generate_device_id(self) -> str:
        """
        Génère un identifiant unique pour le dispositif
        
        Returns:
            str: Identifiant du dispositif
        """
        try:
            # Utiliser l'adresse MAC ou une autre caractéristique unique
            mac_address = self._get_mac_address()
            hostname = os.uname().nodename
            
            # Combiner les informations pour créer un ID unique
            device_id = hashlib.sha256(f"{mac_address}-{hostname}".encode()).hexdigest()[:16]
            return f"agritech-{device_id}"
        
        except Exception as e:
            logger.error(f"Erreur lors de la génération de l'ID du dispositif: {e}")
            # Fallback: utiliser un ID généré aléatoirement
            return f"agritech-{os.urandom(8).hex()}"
    
    def _get_mac_address(self) -> str:
        """
        Récupère l'adresse MAC de l'interface réseau principale
        
        Returns:
            str: Adresse MAC ou chaîne aléatoire en cas d'erreur
        """
        try:
            # Essayer de récupérer l'adresse MAC d'une interface réseau
            for interface in os.listdir('/sys/class/net/'):
                if interface == 'lo':  # Ignorer l'interface loopback
                    continue
                    
                mac_path = f"/sys/class/net/{interface}/address"
                if os.path.exists(mac_path):
                    with open(mac_path, 'r') as f:
                        return f.read().strip()
            
            # Si aucune adresse MAC n'est trouvée
            return os.urandom(6).hex(':')
        
        except Exception as e:
            logger.error(f"Erreur lors de la récupération de l'adresse MAC: {e}")
            return os.urandom(6).hex(':')
    
    def _load_or_generate_keys(self) -> None:
        """Charge les clés existantes ou génère de nouvelles clés"""
        try:
            # Vérifier si les clés existent
            if os.path.exists(self.private_key_path) and os.path.exists(self.public_key_path):
                # Charger les clés existantes
                with open(self.private_key_path, 'rb') as f:
                    self.private_key = RSA.import_key(f.read())
                
                with open(self.public_key_path, 'rb') as f:
                    self.public_key = RSA.import_key(f.read())
                
                logger.info("Clés de signature chargées")
            else:
                # Générer de nouvelles clés
                logger.info("Génération de nouvelles clés de signature...")
                key = RSA.generate(2048)
                self.private_key = key
                self.public_key = key.publickey()
                
                # Sauvegarder les clés
                with open(self.private_key_path, 'wb') as f:
                    f.write(self.private_key.export_key('PEM'))
                
                with open(self.public_key_path, 'wb') as f:
                    f.write(self.public_key.export_key('PEM'))
                
                # Définir les permissions correctes
                os.chmod(self.private_key_path, 0o600)  # Lecture/écriture pour le propriétaire uniquement
                os.chmod(self.public_key_path, 0o644)   # Lecture pour tous, écriture pour le propriétaire
                
                logger.info("Nouvelles clés de signature générées et sauvegardées")
        
        except Exception as e:
            logger.error(f"Erreur lors du chargement/génération des clés: {e}")
            raise
    
    def sign_data(self, data: Dict[str, Any]) -> Dict[str, Any]:
        """
        Signe des données avec la clé privée
        
        Args:
            data: Données à signer
            
        Returns:
            Dict: Données avec signature
        """
        try:
            # Copier les données pour ne pas modifier l'original
            signed_data = data.copy()
            
            # Ajouter des métadonnées de signature
            signed_data['device_id'] = self.device_id
            signed_data['timestamp_signed'] = datetime.now().isoformat()
            
            # Créer une représentation canonique des données (tri des clés)
            data_to_sign = json.dumps(signed_data, sort_keys=True).encode()
            
            # Calculer le hash des données
            data_hash = SHA256.new(data_to_sign)
            
            # Signer le hash
            signer = pkcs1_15.new(self.private_key)
            signature = signer.sign(data_hash)
            
            # Ajouter la signature aux données
            signed_data['signature'] = base64.b64encode(signature).decode('utf-8')
            signed_data['hash'] = data_hash.hexdigest()
            
            logger.debug(f"Données signées pour le capteur {data.get('sensor_id', 'inconnu')}")
            return signed_data
        
        except Exception as e:
            logger.error(f"Erreur lors de la signature des données: {e}")
            # Retourner les données originales en cas d'erreur
            data['signature_error'] = str(e)
            return data
    
    def verify_signature(self, data: Dict[str, Any], public_key_pem: Optional[str] = None) -> bool:
        """
        Vérifie la signature des données
        
        Args:
            data: Données signées à vérifier
            public_key_pem: Clé publique PEM (si différente de celle par défaut)
            
        Returns:
            bool: True si la signature est valide, False sinon
        """
        try:
            # Vérifier que les champs nécessaires sont présents
            if 'signature' not in data or 'hash' not in data:
                logger.warning("Données non signées ou signature incomplète")
                return False
            
            # Extraire la signature
            signature = base64.b64decode(data['signature'])
            
            # Copier les données pour retirer la signature
            data_copy = data.copy()
            data_copy.pop('signature')
            
            # Créer une représentation canonique des données
            data_to_verify = json.dumps(data_copy, sort_keys=True).encode()
            
            # Calculer le hash des données
            data_hash = SHA256.new(data_to_verify)
            
            # Vérifier que le hash correspond
            if data_hash.hexdigest() != data['hash']:
                logger.warning("Hash des données invalide")
                return False
            
            # Utiliser la clé publique fournie ou celle par défaut
            if public_key_pem:
                verification_key = RSA.import_key(public_key_pem)
            else:
                verification_key = self.public_key
            
            # Vérifier la signature
            verifier = pkcs1_15.new(verification_key)
            try:
                verifier.verify(data_hash, signature)
                logger.debug("Signature valide")
                return True
            except Exception:
                logger.warning("Signature invalide")
                return False
        
        except Exception as e:
            logger.error(f"Erreur lors de la vérification de la signature: {e}")
            return False
    
    def generate_auth_token(self, expiration: int = 3600) -> str:
        """
        Génère un token JWT pour l'authentification
        
        Args:
            expiration: Durée de validité du token en secondes
            
        Returns:
            str: Token JWT
        """
        try:
            payload = {
                'device_id': self.device_id,
                'iat': datetime.utcnow(),
                'exp': datetime.utcnow() + timedelta(seconds=expiration)
            }
            
            token = jwt.encode(payload, self.jwt_secret, algorithm='HS256')
            logger.debug(f"Token d'authentification généré pour {self.device_id}")
            
            return token
        
        except Exception as e:
            logger.error(f"Erreur lors de la génération du token: {e}")
            raise
    
    def verify_auth_token(self, token: str) -> Tuple[bool, Optional[Dict[str, Any]]]:
        """
        Vérifie un token JWT
        
        Args:
            token: Token JWT à vérifier
            
        Returns:
            Tuple: (est_valide, payload_décodé)
        """
        try:
            payload = jwt.decode(token, self.jwt_secret, algorithms=['HS256'])
            logger.debug(f"Token vérifié pour {payload.get('device_id', 'inconnu')}")
            return True, payload
        
        except jwt.ExpiredSignatureError:
            logger.warning("Token expiré")
            return False, None
        
        except jwt.InvalidTokenError as e:
            logger.warning(f"Token invalide: {e}")
            return False, None
        
        except Exception as e:
            logger.error(f"Erreur lors de la vérification du token: {e}")
            return False, None
    
    def compute_data_hash(self, data: Dict[str, Any]) -> str:
        """
        Calcule un hash pour des données
        
        Args:
            data: Données à hacher
            
        Returns:
            str: Hash des données
        """
        try:
            # Créer une représentation canonique des données
            data_str = json.dumps(data, sort_keys=True).encode()
            
            # Calculer le hash
            data_hash = hashlib.sha256(data_str).hexdigest()
            return data_hash
        
        except Exception as e:
            logger.error(f"Erreur lors du calcul du hash: {e}")
            # Fallback: utiliser un hash aléatoire
            return hashlib.sha256(os.urandom(32)).hexdigest()
    
    def encrypt_sensitive_data(self, data: str) -> str:
        """
        Chiffre des données sensibles (implémentation simple pour démonstration)
        
        Args:
            data: Données à chiffrer
            
        Returns:
            str: Données chiffrées en base64
        """
        # Note: Dans un environnement de production, utilisez une bibliothèque de chiffrement appropriée
        # Cette implémentation est simplifiée pour la démonstration
        try:
            # Simuler un chiffrement avec XOR et une clé dérivée du device_id
            key_hash = hashlib.sha256(self.device_id.encode()).digest()
            data_bytes = data.encode()
            
            # XOR des données avec la clé (répétée si nécessaire)
            result = bytearray()
            for i in range(len(data_bytes)):
                result.append(data_bytes[i] ^ key_hash[i % len(key_hash)])
            
            # Encoder en base64
            return base64.b64encode(result).decode('utf-8')
        
        except Exception as e:
            logger.error(f"Erreur lors du chiffrement des données: {e}")
            return ""
    
    def decrypt_sensitive_data(self, encrypted_data: str) -> str:
        """
        Déchiffre des données sensibles (implémentation simple pour démonstration)
        
        Args:
            encrypted_data: Données chiffrées en base64
            
        Returns:
            str: Données déchiffrées
        """
        try:
            # Décoder le base64
            data_bytes = base64.b64decode(encrypted_data)
            
            # Clé dérivée du device_id (identique à celle utilisée pour le chiffrement)
            key_hash = hashlib.sha256(self.device_id.encode()).digest()
            
            # XOR avec la clé (opération inverse du chiffrement)
            result = bytearray()
            for i in range(len(data_bytes)):
                result.append(data_bytes[i] ^ key_hash[i % len(key_hash)])
            
            return result.decode('utf-8')
        
        except Exception as e:
            logger.error(f"Erreur lors du déchiffrement des données: {e}")
            return ""
EOL
```

## Étape 5: Création du module de gestion des alertes

```bash
# Création du module de gestion des alertes
cat > ~/agritech-terrain/utils/alert_manager.py << 'EOL'
"""
Module de gestion des alertes pour les capteurs
"""
import os
import sys
import json
import logging
import time
import paho.mqtt.client as mqtt
from typing import Dict, List, Any, Optional
from datetime import datetime
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart

# Configuration du logging
logger = logging.getLogger("alert_manager")

class AlertManager:
    """Gestionnaire d'alertes pour les capteurs"""
    
    def __init__(self, config: Dict[str, Any], mqtt_client=None):
        """
        Initialise le gestionnaire d'alertes
        
        Args:
            config: Configuration du gestionnaire d'alertes
            mqtt_client: Client MQTT existant (optionnel)
        """
        self.config = config
        self.mqtt_client = mqtt_client
        self.alerts = {}  # Stockage des alertes actives par capteur
        self.alert_thresholds = {}  # Seuils d'alerte par type de capteur
        self.alert_history = []  # Historique des alertes
        self.email_config = config.get('email', {})
        self.sms_config = config.get('sms', {})
        self.webhook_config = config.get('webhook', {})
        
        # Configuration des seuils d'alerte par défaut
        self._init_alert_thresholds()
    
    def _init_alert_thresholds(self) -> None:
        """Initialise les seuils d'alerte par défaut"""
        self.alert_thresholds = {
            'temperature': {
                'critical_low': 5.0,
                'warning_low': 15.0,
                'warning_high': 30.0,
                'critical_high': 40.0
            },
            'humidity': {
                'critical_low': 20.0,
                'warning_low': 35.0,
                'warning_high': 75.0,
                'critical_high': 90.0
            },
            'soil_moisture': {
                'critical_low': 10.0,
                'warning_low': 20.0,
                'warning_high': 50.0,
                'critical_high': 60.0
            },
            'light': {
                'critical_low': 100.0,
                'warning_low': 500.0,
                'warning_high': 8000.0,
                'critical_high': 12000.0
            },
            'pH': {
                'critical_low': 4.0,
                'warning_low': 5.5,
                'warning_high': 7.5,
                'critical_high': 9.0
            },
            'default': {
                'critical_low': -float('inf'),
                'warning_low': -float('inf'),
                'warning_high': float('inf'),
                'critical_high': float('inf')
            }
        }
        
        # Mettre à jour avec la configuration
        if 'thresholds' in self.config:
            for sensor_type, thresholds in self.config['thresholds'].items():
                if sensor_type in self.alert_thresholds:
                    self.alert_thresholds[sensor_type].update(thresholds)
                else:
                    self.alert_thresholds[sensor_type] = thresholds
    
    def check_threshold_alert(self, sensor_data: Dict[str, Any]) -> Optional[Dict[str, Any]]:
        """
        Vérifie si les données d'un capteur dépassent les seuils d'alerte
        
        Args:
            sensor_data: Données du capteur
            
        Returns:
            Dict: Données de l'alerte ou None si pas d'alerte
        """
        try:
            sensor_id = sensor_data['sensor_id']
            sensor_type = sensor_data['type']
            value = sensor_data['value']
            location = sensor_data['location']
            unit = sensor_data['unit']
            
            # Récupérer les seuils pour ce type de capteur ou les seuils par défaut
            thresholds = self.alert_thresholds.get(sensor_type, self.alert_thresholds['default'])
            
            # Vérifier les seuils
            alert_data = None
            
            if value <= thresholds['critical_low']:
                alert_data = {
                    'sensor_id': sensor_id,
                    'alert_type': 'threshold',
                    'severity': 'critical',
                    'message': f"Valeur critique basse: {value} {unit} (seuil: {thresholds['critical_low']} {unit})",
                    'value': value,
                    'threshold': thresholds['critical_low'],
                    'created_at': datetime.now().isoformat()
                }
            elif value <= thresholds['warning_low']:
                alert_data = {
                    'sensor_id': sensor_id,
                    'alert_type': 'threshold',
                    'severity': 'warning',
                    'message': f"Valeur d'avertissement basse: {value} {unit} (seuil: {thresholds['warning_low']} {unit})",
                    'value': value,
                    'threshold': thresholds['warning_low'],
                    'created_at': datetime.now().isoformat()
                }
            elif value >= thresholds['critical_high']:
                alert_data = {
                    'sensor_id': sensor_id,
                    'alert_type': 'threshold',
                    'severity': 'critical',
                    'message': f"Valeur critique haute: {value} {unit} (seuil: {thresholds['critical_high']} {unit})",
                    'value': value,
                    'threshold': thresholds['critical_high'],
                    'created_at': datetime.now().isoformat()
                }
            elif value >= thresholds['warning_high']:
                alert_data = {
                    'sensor_id': sensor_id,
                    'alert_type': 'threshold',
                    'severity': 'warning',
                    'message': f"Valeur d'avertissement haute: {value} {unit} (seuil: {thresholds['warning_high']} {unit})",
                    'value': value,
                    'threshold': thresholds['warning_high'],
                    'created_at': datetime.now().isoformat()
                }
            
            # Si une alerte est détectée,