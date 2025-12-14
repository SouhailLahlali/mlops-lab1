# 🚀 MLOps — Churn Prediction Lab (mlops-lab-01)

Ce dépôt documente un workflow MLOps simple pour la prédiction du churn (désabonnement client).  
Le laboratoire est organisé en étapes claires : préparation du projet, génération et préparation des données, entraînement/versioning/validation du modèle, exposition via une API et détectio[...]

---

## Table des matières
- Contexte
- Prérequis
- Structure du projet
- Démarrage rapide
- Étapes détaillées
  - 1. Initialiser la structure du projet
  - 2. Préparer l'environnement Python
  - 3. Générer le dataset
  - 4. Préparer les données & quality checks
  - 5. Entraîner, versionner et valider le modèle
  - 6. Inspecter la registry et le modèle courant
  - 7. API `/predict`
  - 8. Détection de dérive (monitoring)
- Fichiers importants
- Contribuer

---

## Contexte
L'objectif est de fournir un pipeline MLOps minimal couvrant :
- Organisation du projet,
- Génération et préparation des données,
- Entraînement et versioning de modèle,
- Déploiement via API,
- Monitoring simple (détection de dérive).

---

## Prérequis
- Python 3.8+
- pip
- (Optionnel) virtualenv / venv

Librairies principales :
- pandas, numpy, scikit-learn, fastapi, uvicorn, joblib

---

## Structure du projet
Exemple d'arborescence créée par le laboratoire :
- data/ — datasets (raw, processed)
- models/ — modèles sauvegardés (.joblib)
- registry/ — métadonnées et modèle courant (current_model.txt, train_stats.json)
- logs/ — logs d'application et de prédictions
- src/ — scripts et code source

---


---

## Étapes détaillées

### Étape 1 — Initialiser la structure du projet
Création des dossiers de travail :
```bash
mkdir -p data models registry logs src
echo "" > registry/current_model.txt
```
But : organiser les répertoires pour séparer données, modèles, logs et code source.

![arborescence initiale](https://github.com/user-attachments/assets/fa5471b4-75c1-4581-a9f4-74431793ecc7)
![fichiers créés](https://github.com/user-attachments/assets/761d7509-19cd-4cfa-8ea3-d7e1bcafbe86)

---

### Étape 2 — Préparer l'environnement Python
- Créer un venv, activer et installer les dépendances (voir section Démarrage rapide).

![installation dépendances](https://github.com/user-attachments/assets/a0b23913-786d-4282-973f-3edecd18b24d)

---

### Étape 3 — Générer le dataset
Le script `src/generate_data.py` génère un dataset brut et le sauvegarde en `data/raw.csv`.

Exécution :
```bash
python src/generate_data.py
# Sortie attendue exemple:
# [OK] Dataset généré : data/raw.csv (rows=1200, seed=42)
```

![génération dataset](https://github.com/user-attachments/assets/f11aea97-62ca-42cb-b239-f1a5b83f5dab)

---

### Étape 4 — Préparer les données & quality checks
Le script `src/prepare_data.py` :
- Nettoie et transforme `data/raw.csv`
- Sauvegarde le dataset prétraité en `data/processed.csv`
- Génère des statistiques d'entraînement en `registry/train_stats.json`

Exécution :
```bash
python src/prepare_data.py
# Exemple de sortie : [OK] data/processed.csv créé, train_stats.json créé
```

![préparation données](https://github.com/user-attachments/assets/345db7d3-e769-4334-bb37-2c364c206f00)

---

### Étape 5 — Entraîner, versionner et valider le modèle
Le script `src/train.py` :
- Entraîne un modèle (e.g. RandomForest ou LogisticRegression)
- Calcule les métriques (accuracy, precision, recall, f1)
- Sauvegarde le modèle sous `models/churn_model_vX_YYYYMMDD_HHMMSS.joblib`
- Effectue une règle de gate (ex : déployer si F1 > baseline)

Exécution :
```bash
python src/train.py
# Exemple de métriques :
# accuracy: 0.6433
# precision: 0.6688
# recall: 0.6562
# f1: 0.6624
# Modèle sauvé: models/churn_model_v1_20251214_174637.joblib
# Gate: [DEPLOY] Refusé (F1 insuffisant ou baseline non battue)
```

![entraînement modèle](https://github.com/user-attachments/assets/d84565ce-4268-4401-b41a-98f5d725f598)

---

### Étape 6 — Inspecter la registry et le modèle courant
Le fichier `registry/current_model.txt` contient le nom du modèle actif. L'API expose aussi un endpoint de health check.

Exemple de health check :
```json
{
  "status": "ok",
  "current_model": "churn_model_v1_20251214_150721.joblib"
}
```

![health check](https://github.com/user-attachments/assets/595fa66d-b939-41ad-bf7c-ac2b1ec722e9)

---

### Étape 7 — API /predict
L'API (FastAPI) charge le modèle courant depuis `registry/current_model.txt` et propose endpoint `/predict`.

Exemple de requête (curl) :
```bash
curl -X POST "http://127.0.0.1:8000/predict" \
  -H "Content-Type: application/json" \
  -d '{
    "tenure_months": 6,
    "num_complaints": 3,
    "avg_session_minutes": 12.5,
    "plan_type": "basic",
    "region": "AF",
    "request_id": "req-001"
  }'
```

Exemple de réponse :
```json
{
  "request_id": "req-001",
  "prediction": 1,
  "probability": 0.907065,
  "latency_ms": 8.052
}
```

Toutes les prédictions sont aussi loggées dans `logs/predictions.log`.

![api - prédictions log](https://github.com/user-attachments/assets/6f3e2865-3cab-41d7-99b3-b9e0c05d2883)
![api - logs détaillés](https://github.com/user-attachments/assets/ccf81d3b-996c-4c13-8106-fd80a82fb1b5)

---

### Étape 8 — Détection de dérive (monitoring)
Le script `src/monitor_drift.py` compare les statistiques récentes (production) aux statistiques d'entraînement (`registry/train_stats.json`) et calcule des z-scores pour détecter la dérive.

Exécution :
```bash
python src/monitor_drift.py
# Exemple de sortie :
# Test de dérive sur les 5 dernières requêtes : aucun drift détecté.
# tenure_months  mean_prod=22.8  mean_train=30.246  z=0.439
# num_complaints mean_prod=1.8   mean_train=1.174   z=0.563
# avg_session_minutes mean_prod=31.5 mean_train=35.124 z=0.306
```

![monitor drift](https://github.com/user-attachments/assets/14b617fb-e76c-4842-b6b1-5f3d9d4ca220)


---

# 📝 Lab2 : Code Source management

Ce laboratoire a pour but de se familiariser avec les commandes Git essentielles dans le contexte d'un projet MLOps, couvrant l'initialisation du dépôt, les commits, les branches, la fusion, la gestion des conflits, `git stash`, `git reset`, `git revert`, et `git rebase`.

---

## 🚀 Étapes du Labo
<img width="982" height="598" alt="Screenshot_20251214_210053" src="https://github.com/user-attachments/assets/d8a8a50e-1817-4c16-82ad-70088570c1c3" />

<img width="1114" height="413" alt="Screenshot_20251214_210343" src="https://github.com/user-attachments/assets/3dba0892-e36d-46ef-9239-c35c77d02af8" />

<img width="955" height="353" alt="Screenshot_20251214_210549" src="https://github.com/user-attachments/assets/1b4ea281-efb7-4693-b6ca-5b7094e64e53" />

<img width="1040" height="479" alt="Screenshot_20251214_210616" src="https://github.com/user-attachments/assets/fa586a75-a01f-44fb-bd42-14dab535092a" />

<img width="1064" height="467" alt="Screenshot_20251214_211053" src="https://github.com/user-attachments/assets/3a8e06f6-a538-4661-be8e-eff564dfeff4" />

<img width="823" height="352" alt="Screenshot_20251214_211155" src="https://github.com/user-attachments/assets/519e1ded-6142-4bc8-850c-36b30e8b731e" />

<img width="1195" height="522" alt="Screenshot_20251214_211505" src="https://github.com/user-attachments/assets/e5e0fe09-a88f-42a9-8a9b-50f1baad2647" />

<img width="1047" height="686" alt="Screenshot_20251214_211652" src="https://github.com/user-attachments/assets/5c4511ee-d13e-4b22-aebe-5c7ccf230fed" />

<img width="1066" height="554" alt="Screenshot_20251214_211843" src="https://github.com/user-attachments/assets/5b905242-80a4-44eb-8c1e-31fe6eff1d39" />

<img width="891" height="435" alt="Screenshot_20251214_211911" src="https://github.com/user-attachments/assets/1f595f84-af17-496b-a3c4-25bfde8e42db" />

<img width="1005" height="609" alt="Screenshot_20251214_212044" src="https://github.com/user-attachments/assets/2b9efeaf-9bfa-40ea-a62e-cab3a5f3fd76" />

<img width="1081" height="248" alt="Screenshot_20251214_212207" src="https://github.com/user-attachments/assets/53c7e154-3fe3-40c5-9511-ace7da69e80f" />

<img width="1144" height="401" alt="Screenshot_20251214_212257" src="https://github.com/user-attachments/assets/f4fb2726-9e1e-4f4f-be49-6a046e95abb9" />

<img width="980" height="274" alt="Screenshot_20251214_212308" src="https://github.com/user-attachments/assets/81545b70-df5f-47f1-be5e-bb2586d4cd59" />

<img width="980" height="212" alt="Screenshot_20251214_212323" src="https://github.com/user-attachments/assets/7aec483d-f83e-471e-a0f2-32bf88ab39e6" />
