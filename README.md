### Network Security (Phishing Detection) – End‑to‑End ML System

Production‑ready FastAPI service and ML pipeline to detect phishing using network/URL features. The project includes data ingestion from MongoDB, validation, transformation, model training, experiment tracking with MLflow, and a REST API for training and batch predictions.

--- 

## Features
- **End‑to‑end pipeline**: ingestion → validation → transformation → training → artifacts
- **FastAPI service**: `/train` to run the pipeline, `/predict` to score CSVs
- **Model packaging**: final `model.pkl` and `preprocessor.pkl` stored in `final_model/`
- **Experiment tracking**: MLflow runs stored under `mlruns/`
- **Artifacts versioning**: dated runs under `Artifacts/`
- **Docker support**: containerize the API for deployment (ECR/EC2 ready)

---

## Tech Stack
- Python 3.11
- FastAPI, Uvicorn
- scikit‑learn, pandas, numpy
- MLflow
- MongoDB (for raw data storage)
- Docker (optional, for deployment)

---

## Repository Structure
```text
network security/
  app.py                    # FastAPI application with /train and /predict
  main.py                   # Local runner for the training pipeline
  final_model/              # Saved preprocessor and trained model
  mlruns/                   # MLflow tracking data
  Artifacts/                # Versioned artifacts by run timestamp
  networksecurity/          # Package with pipeline, components, utils, etc.
  requirements.txt          # Python dependencies
  Dockerfile                # Container for API
  README.md                 # This file
```

---

## Quick Start

### 1) Prerequisites
- Python 3.11
- MongoDB URI for ingestion (if using the training pipeline)
- Optional: Docker

### 2) Clone and Install
```bash
git clone <your-repo-url>.git
cd "network security"
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

### 3) Environment Variables
Create a `.env` file in the project root:
```bash
# Required for data ingestion (Mongo client)
MONGODB_URL_KEY="mongodb+srv://<user>:<pass>@<cluster>/<db>?retryWrites=true&w=majority"

# Optional – used only if deploying to AWS/ECR
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=us-east-1
AWS_ECR_LOGIN_URI=788614365622.dkr.ecr.us-east-1.amazonaws.com/networkssecurity
ECR_REPOSITORY_NAME=networkssecurity
```

---

## Run Locally

### Option A: Start the API
```bash
python app.py
# The server starts via Uvicorn at 0.0.0.0:8000
```
Open `http://localhost:8000/docs` for interactive Swagger UI.

Endpoints:
- `GET /train` – runs the full training pipeline and logs artifacts/MLflow
- `POST /predict` – accepts a CSV file upload and returns an HTML table with predictions; also writes `prediction_output/output.csv`

Sample `curl` for prediction (replace `sample.csv` with your file):
```bash
curl -X POST "http://localhost:8000/predict" \
  -H "accept: application/json" \
  -H "Content-Type: multipart/form-data" \
  -F "file=@sample.csv"
```

### Option B: Run Training from CLI
```bash
python main.py
```
This executes ingestion, validation, transformation, and training sequentially. Artifacts are written to `Artifacts/<timestamp>/` and the final model assets to `final_model/`.

---

## Data & Artifacts
- `Network_Data/phisingData.csv`: example input dataset
- `Artifacts/<timestamp>/...`: feature store, validated splits, transformed arrays, drift reports, and preprocessing objects
- `final_model/model.pkl`: trained classifier
- `final_model/preprocessor.pkl`: fitted preprocessing pipeline used at inference

---

## Input Schema
Expected columns and dtypes for training and batch prediction. For prediction, the target `Result` is optional and will be ignored if present.

| Column | Type |
| --- | --- |
| having_IP_Address | int64 |
| URL_Length | int64 |
| Shortining_Service | int64 |
| having_At_Symbol | int64 |
| double_slash_redirecting | int64 |
| Prefix_Suffix | int64 |
| having_Sub_Domain | int64 |
| SSLfinal_State | int64 |
| Domain_registeration_length | int64 |
| Favicon | int64 |
| port | int64 |
| HTTPS_token | int64 |
| Request_URL | int64 |
| URL_of_Anchor | int64 |
| Links_in_tags | int64 |
| SFH | int64 |
| Submitting_to_email | int64 |
| Abnormal_URL | int64 |
| Redirect | int64 |
| on_mouseover | int64 |
| RightClick | int64 |
| popUpWidnow | int64 |
| Iframe | int64 |
| age_of_domain | int64 |
| DNSRecord | int64 |
| web_traffic | int64 |
| Page_Rank | int64 |
| Google_Index | int64 |
| Links_pointing_to_page | int64 |
| Statistical_report | int64 |
| Result (target) | int64 |

Minimal CSV example (header + one row):
```csv
having_IP_Address,URL_Length,Shortining_Service,having_At_Symbol,double_slash_redirecting,Prefix_Suffix,having_Sub_Domain,SSLfinal_State,Domain_registeration_length,Favicon,port,HTTPS_token,Request_URL,URL_of_Anchor,Links_in_tags,SFH,Submitting_to_email,Abnormal_URL,Redirect,on_mouseover,RightClick,popUpWidnow,Iframe,age_of_domain,DNSRecord,web_traffic,Page_Rank,Google_Index,Links_pointing_to_page,Statistical_report
1,2,1,1,1,1,0,1,0,1,0,1,1,0,2,1,0,1,0,0,0,0,0,1,1,2,1,1,1,0
```

Notes:
- All features are integers; ensure no missing values in required columns.
- For training, include `Result`. For prediction, `Result` is optional and ignored.

---

## MLflow Tracking
By default, local runs are recorded under `mlruns/0/`. You can explore metrics, parameters, and artifacts using the MLflow UI:
```bash
mlflow ui --backend-store-uri mlruns
# then open http://127.0.0.1:5000
```

---

## Docker
Build and run the API locally with Docker:
```bash
docker build -t networkssecurity:latest .
docker run -p 8000:8000 --env-file .env networkssecurity:latest
```

### Deploy to AWS EC2 (Docker)
Run these on your EC2 instance (Ubuntu):
```bash
sudo apt-get update -y
sudo apt-get upgrade -y
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker ubuntu
newgrp docker
```

### Push to AWS ECR (optional)
Make sure the ECR repository exists and the `.env` contains the values shown above.
```bash
aws ecr get-login-password --region $AWS_REGION | \
  docker login --username AWS --password-stdin $AWS_ECR_LOGIN_URI

docker build -t $ECR_REPOSITORY_NAME:latest .
docker tag $ECR_REPOSITORY_NAME:latest $AWS_ECR_LOGIN_URI:latest
docker push $AWS_ECR_LOGIN_URI:latest
```
Then pull and run on EC2:
```bash
docker pull $AWS_ECR_LOGIN_URI:latest
docker run -d -p 8000:8000 --env-file .env $AWS_ECR_LOGIN_URI:latest
```

---

## Development Notes
- API auto‑docs available at `/docs`
- CORS is open to all origins for ease of testing; tighten for production
- Predictions are saved to `prediction_output/output.csv` and also rendered as an HTML table via `templates/table.html`

---

## Troubleshooting
- Ensure `.env` is loaded and `MONGODB_URL_KEY` is valid before running `/train`
- If `certifi`/TLS issues occur for MongoDB, the app sets `tlsCAFile` via `certifi.where()`
- If dependencies are missing, reinstall: `pip install -r requirements.txt`

---

## License
This project is licensed under the terms of the `LICENSE` file.

