# Diabetes Risk Prediction System (Intern ID: CITS9843)

A Python API that estimates diabetes risk from either clinical measurements, reported symptoms, or both. It provides separate models for lab values and symptoms, plus a combined endpoint that blends both predictions when both kinds of input are supplied.

> **Important:** This project is for educational and informational purposes only. Its predictions are not a diagnosis or a substitute for advice from a qualified healthcare professional.

## Architecture

```text
Client
  │
  │ REST/JSON
  ▼
FastAPI application (app.py)
  ├── Lab model: imputation + scaling + trained classifier
  ├── Symptom model: categorical encoding + trained classifier
  └── Combined prediction: 60% lab + 40% symptom probability
```

- **`app.py`** — FastAPI application, input validation, prediction routes, model information, and health check.
- **`train_lab_model.py`** — trains and evaluates a lab-values classifier using `data/pima.csv`; compares logistic regression and random forest using cross-validation.
- **`train_symptom_model.py`** — trains and evaluates a symptom classifier. It uses `data/early_stage_diabetes_risk_prediction_dataset.csv` when present, otherwise falls back to `data/symptom_data_placeholder.csv`.
- **`data/`** — included training datasets and the placeholder-data generator.
- **`model/`** — pre-trained models, preprocessing artifacts, feature definitions, and evaluation metrics loaded by the API at startup.

## Prerequisites

- Python 3.10+
- pip

## Setup and run

From the project directory, create and activate a virtual environment, then install the dependencies:

```powershell
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install fastapi "uvicorn[standard]" numpy pandas scikit-learn joblib
```

Start the API:

```powershell
uvicorn app:app --reload --port 8000
```

The API runs at `http://localhost:8000`. Interactive API documentation is available at `http://localhost:8000/docs`; check `http://localhost:8000/api/health` to verify it is running.

The trained model artifacts are included, so training is not required to start the API. To retrain either model from the project directory:

```powershell
python train_lab_model.py
python train_symptom_model.py
```

The symptom training script prefers the included early-stage diabetes dataset. If that file is unavailable, it uses the synthetic placeholder dataset; check the training output and `model/symptom_metrics.json` to see which data source was used. Replace the models and metrics under `model/` by rerunning the scripts after changing or supplying training data.

## Prediction API

All prediction routes accept JSON and return a probability between `0` and `1`, a risk band (`Low`, `Moderate`, or `High`), or both. The bands are based on the predicted probability:

- **Low:** below `0.30`
- **Moderate:** `0.30` to below `0.60`
- **High:** `0.60` or higher

### `GET /api/health`

Returns `{"status":"ok"}` when the API is running.

### `GET /api/model-info`

Returns each model's features, available evaluation metrics, and training data source.

### `POST /api/predict/lab`

Provide all eight numeric fields. Values are range-validated by the API.

```json
{
  "Pregnancies": 2,
  "Glucose": 120,
  "BloodPressure": 70,
  "SkinThickness": 25,
  "Insulin": 80,
  "BMI": 28.5,
  "DiabetesPedigreeFunction": 0.45,
  "Age": 35
}
```

Returns `probability`, `risk_band`, and `source`.

### `POST /api/predict/symptoms`

Provide `Age`, `Gender`, and all 14 symptom fields. Use `Male` or `Female` for gender and `Yes` or `No` for each symptom value (case-insensitive).

```json
{
  "Age": 45,
  "Gender": "Female",
  "Polyuria": "No",
  "Polydipsia": "Yes",
  "sudden_weight_loss": "No",
  "weakness": "Yes",
  "Polyphagia": "No",
  "Genital_thrush": "No",
  "visual_blurring": "No",
  "Itching": "No",
  "Irritability": "No",
  "delayed_healing": "No",
  "partial_paresis": "No",
  "muscle_stiffness": "No",
  "Alopecia": "No",
  "Obesity": "No"
}
```

Returns `probability`, `risk_band`, and `source`.

### `POST /api/predict/combined`

Pass a `lab` object, a `symptoms` object, or both. When both are provided, the combined probability is `0.6 × lab_probability + 0.4 × symptom_probability`. When only one is provided, its probability is used directly. At least one object is required.

```json
{
  "lab": {
    "Pregnancies": 2,
    "Glucose": 120,
    "BloodPressure": 70,
    "SkinThickness": 25,
    "Insulin": 80,
    "BMI": 28.5,
    "DiabetesPedigreeFunction": 0.45,
    "Age": 35
  },
  "symptoms": {
    "Age": 45,
    "Gender": "Female",
    "Polyuria": "No",
    "Polydipsia": "Yes",
    "sudden_weight_loss": "No",
    "weakness": "Yes",
    "Polyphagia": "No",
    "Genital_thrush": "No",
    "visual_blurring": "No",
    "Itching": "No",
    "Irritability": "No",
    "delayed_healing": "No",
    "partial_paresis": "No",
    "muscle_stiffness": "No",
    "Alopecia": "No",
    "Obesity": "No"
  }
}
```

Returns the available individual probabilities, `combined_probability`, and `risk_band`.

## Notes

- The lab model treats zero values for glucose, blood pressure, skin thickness, insulin, and BMI as missing during training, imputes missing values with the median, and scales features before prediction.
- The API loads the model artifacts from `model/` when it starts. Restart the API after retraining.
- If a sibling `frontend/` directory containing `index.html` exists, the app also serves it at `/` and its static assets under `/static`. The API can be used independently without that frontend.