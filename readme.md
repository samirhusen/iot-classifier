# Secure IoT Traffic Classification – ML Pipeline & REST API

> A personal project to classify IoT network traffic as **Normal** or **Attack** using machine learning, served via a FastAPI REST API and deployed on AWS EC2.

---

## Table of Contents

- [Overview](#overview)
- [Dataset](#dataset)
- [Project Structure](#project-structure)
- [Tech Stack](#tech-stack)
- [Setup & Installation](#setup--installation)
- [Usage](#usage)
- [API Reference](#api-reference)
- [Model Performance](#model-performance)
- [Docker](#docker)
- [AWS Deployment](#aws-deployment)
- [MLflow Experiment Tracking](#mlflow-experiment-tracking)
- [Deliverables](#deliverables)
- [Author](#author)

---

## Overview

IoT and IIoT devices (smart cameras, routers, sensors, medical devices) generate continuous network traffic that is difficult to monitor manually. This project builds an end-to-end machine learning pipeline that:

1. **Preprocesses** the UNSW-NB15 network traffic dataset
2. **Trains and compares** multiple classification models
3. **Evaluates** models using Accuracy, Precision, Recall, F1-score, Confusion Matrix, and ROC-AUC
4. **Serves predictions** via a FastAPI REST API
5. **Deploys** the containerized API to AWS EC2 with CloudWatch monitoring

**Binary classification target:**
| Label | Meaning |
|-------|---------|
| `0` | Normal traffic |
| `1` | Attack traffic |

---

## Dataset

**UNSW-NB15** – A public dataset designed for network intrusion detection research.

- **Source:** [https://research.unsw.edu.au/projects/unsw-nb15-dataset](https://research.unsw.edu.au/projects/unsw-nb15-dataset)
- **Records:** ~2.5 million network flow instances
- **Features:** 49 features (numeric + categorical)
- **Key feature columns:** `dur`, `proto`, `service`, `state`, `spkts`, `dpkts`, `sbytes`, `dbytes`, `rate`, `sload`, `dload`, `sinpkt`, `dinpkt`, `ct_srv_src`, `ct_dst_src_ltm`, and more
- **Target column:** `label` (0 = Normal, 1 = Attack)
- **Attack categories** (`attack_cat`): used for EDA only, not for model training

Download the CSV files and place them in `data/raw/`.

---

## Project Structure

```
iot-classifier/
├── data/
│   ├── raw/                  # Original UNSW-NB15 CSV files
│   └── processed/            # Cleaned, encoded, split datasets
├── models/
│   ├── best_model.joblib     # Saved best classifier
│   └── scaler.pkl            # Fitted StandardScaler
├── notebooks/
│   └── eda_and_training.ipynb  # EDA, training, and evaluation notebook
├── src/
│   ├── preprocess.py         # Data cleaning, encoding, scaling, splitting
│   ├── train.py              # Model training and MLflow logging
│   └── api/
│       ├── main.py           # FastAPI app entrypoint
│       ├── schemas.py        # Pydantic request/response models
│       └── predictor.py      # Inference logic (scale + predict)
├── mlruns/                   # MLflow experiment tracking (auto-generated)
├── Dockerfile
├── docker-compose.yml
├── requirements.txt
└── README.md
```

---

## Tech Stack

| Category | Tools |
|----------|-------|
| Language | Python 3.10+ |
| Data handling | Pandas, NumPy |
| Machine learning | Scikit-learn, XGBoost |
| Visualization | Matplotlib, Seaborn |
| API | FastAPI, Uvicorn, Pydantic |
| Model saving | Joblib |
| Experiment tracking | MLflow |
| Containerization | Docker, Docker Compose |
| Cloud deployment | AWS EC2, CloudWatch |

---

## Setup & Installation

### 1. Clone the repository

```bash
git clone https://github.com/samirhusen/iot-classifier.git
cd iot-classifier
```

### 2. Create and activate virtual environment

```bash
python -m venv venv
source venv/bin/activate        # Mac/Linux
# venv\Scripts\activate         # Windows
```

### 3. Install dependencies

```bash
pip install -r requirements.txt
```

### 4. Download the dataset

Download UNSW-NB15 CSV files from the [official source](https://research.unsw.edu.au/projects/unsw-nb15-dataset) and place them in:

```
data/raw/UNSW_NB15_training-set.csv
data/raw/UNSW_NB15_testing-set.csv
```

---

## Usage

### Step 1 – Preprocess the data

```bash
python src/preprocess.py
```

Outputs cleaned and encoded datasets to `data/processed/`.

### Step 2 – Train the models

```bash
python src/train.py
```

Trains Logistic Regression, Random Forest, and XGBoost. Logs all runs to MLflow and saves the best model to `models/`.

### Step 3 – Start the API

```bash
uvicorn src.api.main:app --reload --host 0.0.0.0 --port 8000
```

### Step 4 – Test a prediction

```bash
curl -X POST http://127.0.0.1:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"dur": 0.121, "proto": "tcp", "service": "http", "state": "FIN", "spkts": 6, "dpkts": 4, "sbytes": 512, "dbytes": 1024, "rate": 49.58}'
```

---

## API Reference

### `POST /predict`

Classify a network traffic flow as Normal or Attack.

**Request body** (JSON):
```json
{
  "dur": 0.121,
  "proto": "tcp",
  "service": "http",
  "state": "FIN",
  "spkts": 6,
  "dpkts": 4,
  "sbytes": 512,
  "dbytes": 1024,
  "rate": 49.58
}
```

**Response** (JSON):
```json
{
  "label": 0,
  "prediction": "Normal",
  "confidence": 0.94,
  "latency_ms": 12
}
```

---

### `GET /health`

Check if the API is running.

```json
{ "status": "ok", "model_loaded": true }
```

---

### `GET /model-info`

Returns metadata about the loaded model.

```json
{
  "model_type": "RandomForestClassifier",
  "trained_on": "UNSW-NB15",
  "features": 49,
  "classes": ["Normal", "Attack"]
}
```

---

## Model Performance

Results on the UNSW-NB15 test set (20% split):

| Model | Accuracy | Precision | Recall | F1-score | ROC-AUC |
|-------|----------|-----------|--------|----------|---------|
| Logistic Regression | – | – | – | – | – |
| Random Forest | – | – | – | – | – |
| XGBoost | – | – | – | – | – |

> Results will be filled in after training. All runs are logged to MLflow for full reproducibility.

---

## Docker

### Build and run locally

```bash
docker build -t iot-classifier .
docker run -p 8000:8000 iot-classifier
```

### Using Docker Compose

```bash
docker-compose up --build
```

The API will be available at `http://localhost:8000`.

---

## AWS Deployment

1. Launch an EC2 instance (Ubuntu 22.04, t2.micro)
2. Install Docker on the instance
3. Pull or copy the Docker image to EC2
4. Run the container and expose port 8000 in the Security Group
5. Access the API at `http://<ec2-public-ip>:8000`

CloudWatch logs and CPU alarms are configured via the IAM role attached to the instance.

---

## MLflow Experiment Tracking

All training runs are logged automatically. To view the MLflow UI:

```bash
mlflow ui
```

Then open `http://127.0.0.1:5000` in your browser.

Each run logs:
- Model type and hyperparameters
- Accuracy, Precision, Recall, F1-score, ROC-AUC
- Confusion matrix artifact
- Saved model artifact

---

## Deliverables

- [x] Cleaned and preprocessed UNSW-NB15 dataset
- [x] EDA and training notebook (`notebooks/eda_and_training.ipynb`)
- [ ] Trained classification model (`models/best_model.joblib`)
- [ ] FastAPI REST API with `/predict` endpoint
- [ ] Dockerized API
- [ ] AWS EC2 deployment
- [ ] Final project report and documentation

---

## Author

**Samir Husen**
M.S. Computer Science – Wright State University