# 🧠 Migraine Early Warning Backend  
### *FastAPI + ML Inference + Firebase Integration + n8n Automation*

This backend powers the **Migraine Early Warning System**, providing:

- Machine learning–based migraine risk predictions  
- Firebase-backed event storage and personalization  
- n8n automation support  
- Cloud Run–ready Dockerized API  

This repository contains **only backend logic**, not mobile UI or front-end code.

---

# Features

### ✔ ML Risk Prediction API  
- Ensemble model: LightGBM + XGBoost + RandomForest + Logistic Regression  
- SMOTE-balanced training  
- Feature scaling + preprocessing  
- Configurable thresholds for Low / Medium / High  
- Top-factor extraction for explainability  

### ✔ Firebase Integration  
- Write events and predictions to user-specific paths  
- Fetch user history  
- Generate per-user migraine summaries  

### ✔ n8n Automation-Compatible  
- Can be triggered by scheduled workflows  
- Accepts dynamic JSON feature payloads  
- Lightweight inference (<50ms)

### ✔ Fully Dockerized (Google Cloud Run-ready)  
- Stateless  
- Auto-scalable  
- Firebase Realtime Database URL read from the `FIREBASE_DB_URL` environment variable

---

# 📁 Project Structure (Backend Only)

```
AuriCore/
├── backend/
│ ├── app.py
│ ├── features.py
│ ├── firebase_client.py
│ ├── generate_synthetic_data.py
│ ├── inference.py
│ ├── personalization.py
│ ├── train_models.py
│ ├── Dockerfile
│ ├── requirements.txt
│ ├── README.md
```

# Data and models

The model is trained on **synthetic data** produced by `generate_synthetic_data.py`
(hackathon scope: no real patient data was used). Neither the generated data nor the
trained models are committed, so generate them before running the API:

```bash
pip install -r requirements.txt

python generate_synthetic_data.py   # writes data/users.csv and data/synthetic_migraine_data.csv
python train_models.py              # writes models/*.pkl (about 25 minutes on CPU: 1M rows)

export FIREBASE_DB_URL=https://<project>-default-rtdb.<region>.firebasedatabase.app
uvicorn app:app --host 0.0.0.0 --port 8080 --reload

http://localhost:8080/health
```
```json
{
"status": "ok",
"service": "Migraine Prediction API",
 "message": "Running and reachable."
 }
```

### Migraine Risk Prediction


POST /predict

Example Input:
```json
{
  "features": {
    "sleep_hours": 7,
    "hrv": 55,
    "resting_hr": 65,
    "screen_time_total_hours": 4,
    "screen_time_after_22_hours": 0.4,
    "meeting_hours": 4,
    "meeting_count": 4,
    "temperature": 10,
    "pressure_change_abs": 2,
    "humidity": 70,
    "precipitation": 1,
    "sleep_deviation": 0,
    "hrv_deviation": 0,
    "screen_deviation": 0,
    "meeting_deviation": 0,
    "sleep_hours_3d_avg": 7,
    "hrv_3d_avg": 55,
    "screen_time_total_hours_3d_avg": 4,
    "meeting_hours_3d_avg": 4
  }
}
```

Example Output:

```json
{
  "risk_score": 0.67,
  "risk_level": "HIGH",
  "top_factors": ["hrv", "pressure_change_abs", "sleep_hours"],
  "model_version": "v1.0"
}
```

### Get User Events from Firebase

GET /users/{user_id}/events

Returns structured event history for the mobile diary.

### Personalized Summary

GET /users/{user_id}/summary

```json
{
  "user_id": "abc123",
  "current_risk": {
    "score": 0.42,
    "level": "MEDIUM",
    "top_factors": ["hrv", "meeting_hours"]
  },
  "trends": {
    "sleep_hours": { "latest": 6, "baseline": 7.2, "direction": "down" },
    "hrv": { "latest": 48, "baseline": 57, "direction": "down" }
  },
  "insights": [
    "Your HRV is lower than usual.",
    "Your sleep was 1.2 hours below your baseline."
  ],
  "latest_event": {...}
}
```
---

## n8n Workflow Integration

This backend is fully integrated with **n8n automation**, which handles:

- Fetching weather + wearable dummy data  
- Calling the backend `/predict` API  
- Writing **events** and **predictions** into Firebase  
- Triggering **mobile push notifications** when migraine risk increases  
- Automatically updating **Migraine Diary** logs  
- Maintaining **Google Sheets summary** (optional for demo use)

---

### Firebase Write Operations Used in n8n

The system performs **two separate Firebase writes** for clean data separation.

---

### **1️⃣ Event Write**

Stores **raw features** & **sensor data**:

```
PATCH https://<db-url>/users/{{user_id}}/events/{{timestamp}}.json
```

---

### **2️⃣ Prediction Write**

Stores the **prediction result** only:

```
PATCH https://<db-url>/users/{{user_id}}/predictions/{{timestamp}}.json
```

---

### Example Prediction Payload Used in n8n

```json
{
  "user_id": "some-user-id",
  "timestamp": "2025-11-16T02:48:23.811Z",
  "risk_score": 0.4,
  "risk_level": "MEDIUM",
  "top_factors": {
    "first": "hrv",
    "second": "meeting_hours",
    "third": "sleep_hours"
  },
  "model_version": "v1.0"
}
```

---

# Cloud Run Deployment

## Deploy directly from source:

```bash
gcloud run deploy migraine-api \
  --source . \
  --region europe-north1 \
  --allow-unauthenticated \
  --platform managed \
  --clear-base-image
```
---

## Alternative: Manual Docker Build → Push → Deploy

### 1. Build Docker image

```bash
docker build -t migraine-api .
```

### 2. Push to Google Container Registry

```bash
docker tag migraine-api gcr.io/PROJECT_ID/migraine-api
docker push gcr.io/PROJECT_ID/migraine-api
```
### 3. Deploy to Cloud Run

```bash
gcloud run deploy migraine-api \
  --image gcr.io/PROJECT_ID/migraine-api \
  --region europe-north1 \
  --allow-unauthenticated
```
---

# Machine Learning Summary

This system uses a 5-model ensemble for increased robustness:

- Logistic Regression
- Random Forest
- XGBoost
- LightGBM
- Personalized baseline deviation model
- 
---

# ✔ Benefits of the Ensemble

- More stable predictions across varied inputs
- Higher recall for MEDIUM and HIGH risk cases
- Lower false negatives (important for health warnings!)
- Balanced risk distribution (not always “LOW”)
- Strong correlation with personal historical trends
