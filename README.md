# **mlfs-book**

*Companion material for the O’Reilly book **Building Machine Learning Systems with a Feature Store: Batch, Real-Time, and LLMs**.*

Dashboard: https://haysho2260.github.io/mlfs-book/air-quality/


## **📊 ML System Examples**

👉 **[Dashboards for Example ML Systems](https://featurestorebook.github.io/mlfs-book/)**

---

# **🌫️ Air Quality Tutorial**

See tutorial instructions here:
👉 [https://docs.google.com/document/d/1YXfM1_rpo1-jM-lYyb1HpbV9EJPN6i1u6h2rhdPduNE/edit?usp=sharing](https://docs.google.com/document/d/1YXfM1_rpo1-jM-lYyb1HpbV9EJPN6i1u6h2rhdPduNE/edit?usp=sharing)

---

## **🐍 Environment Setup Reminder**

Create a Conda or virtual environment **before installing requirements**:

```bash
pip install -r requirements.txt
```

---

## **⚙️ Running Pipelines with Make**

```bash
make aq-features
make aq-train
make aq-inference
make aq-clean
```

Or simply:

```bash
make aq-all
```

---

## **🧩 Feldera (Optional)**

```bash
mkdir -p /tmp/c.app.hopsworks.ai
ln -s /tmp/c.app.hopsworks.ai ~/hopsworks

docker run -p 8080:8080 \
 -v ~/hopsworks:/tmp/c.app.hopsworks.ai \
 --tty --rm -it ghcr.io/feldera/pipeline-manager:latest
```

---

## **📘 Introduction to ML**

I wrote a brief introduction to machine learning here:

📄 **[`introduction_to_supervised_ml.pdf`](./introduction_to_supervised_ml.pdf)**

---

# **🧪 Lab Description**

## **✨ New Features Added**

* **Air Quality Multi-Horizon Forecasting**
  `4_air_quality_batch_inference.ipynb` now generates chained 1, 7, 14, 21, and 30-day forecasts using recursive conditioning.

* **Schema-Aware Backfill Pipeline**
  Utilities in `mlfs/airquality/util.py` coerce inference output to the exact Hopsworks schema, preventing type mismatches during monitoring backfills.

* **Multi-City Pipelines via Makefile**
  Sensor locations are read from JSON, allowing pipeline runs for multiple cities without modifying notebooks.

* **✨ New Features Added**
   The Makefile loops thorugh multiple cities and allows for notebooks to be parameterized.

* ## **✨ Day Predications are Configurable**
   The Makefile loops thorugh multiple cities and allows for notebooks to be parameterized.
---

## **🛠️ Feature Engineering Highlights**

### **Weather Feature Group (`1_air_quality_feature_backfill.ipynb`)**

Includes both raw and engineered features:

* Wind decompositions: `wind_u`, `wind_v`
* Calendar context:
  `day_of_week`, `month`, `is_weekend`, `day_of_year`, + sine/cosine transforms
* Interaction terms:
  `precipitation_binary`, `temp_wind_interaction`, `precip_wind_interaction`, etc.
* Rolling statistics:
  `temperature_30d_avg`, `temperature_anomaly`, `temp_anomaly_wind_speed`

---

### **Lagged PM2.5 Feature Group**

Captures historical PM2.5 dynamics:

* Percent-change features:
  `pm25_change_1d`, `_2d`, `_3d`, `_7d`, `_14d`, `_21d`, `_30d`
* Volatility metrics:
  `pm25_std_1d`, `_2d`, `_3d`, `_7d`, `_14d`, `_21d`, `_30d`

All lagged features are computed strictly from historical values to avoid data leakage.

---

# **📚 Discussion: Why Doesn’t Recursive Forecasting Collapse?**

When predicting PM2.5 7 days ahead, the model uses its **own predictions** as lagged inputs for subsequent steps (recursive multi-step forecasting).

Example:

> Predict t+1 → feed into model → predict t+2 → feed into model → … → t+7

### ✅ **Why doesn’t performance explode?**

Because:

1. **PM2.5 is typically a stable, slowly varying time series.**
   Small errors don’t escalate rapidly.

2. **The model behaves approximately linearly near typical values.**
   Tree models, linear models, and regularized LSTMs don’t amplify noise sharply.

3. **Noise levels are modest.**
   Synthetic predictions still resemble real inputs used during training.

4. **The model was trained on lagged inputs.**
   It learned typical transitions: yesterday → today → tomorrow.

Thus, synthetic inputs remain “in distribution.”

---

### ❌ **When does recursive forecasting *fail*?**

1. **Unstable or highly sensitive models**

   * High-order autoregressive models (coefficients > 1)
   * Poorly regularized RNNs

2. **Highly nonlinear models**

   * Deep nets with sharp boundaries
   * Overfitted tree ensembles

3. **Sudden shocks in the underlying process**
   (e.g., wildfires, dust storms → model drifts quickly)

4. **Training only for 1-step but evaluating for 7-step**
   → Creates distribution shift between real vs. synthetic lagged inputs.

5. **Prediction drift**

   * Underpredict systematically → values collapse
   * Overpredict → values blow up

---

### 🧨 Examples of catastrophic failure

* **Weather forecasting** – small pressure/humidity errors cause exponential divergence
* **Stock price forecasting** – 1% error compounds quickly
* **Epidemic (SIR) models** – underestimating infection rate collapses forecasts
