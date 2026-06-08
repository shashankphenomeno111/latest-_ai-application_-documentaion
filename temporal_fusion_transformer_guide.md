# Technical Reference Guide: Temporal Fusion Transformer (TFT) in Project Sahyadri 2.0

This document serves as the absolute technical reference for the **Temporal Fusion Transformer (TFT)** price forecasting engine in Project Sahyadri 2.0. It details the underlying mathematical concepts, features, reasoning behind its selection over baseline models, and the exact implementation parameters in our application.

---

## 1. What is a Temporal Fusion Transformer (TFT)?

Developed by Google Research, the **Temporal Fusion Transformer (TFT)** is an attention-based deep learning architecture specifically optimized for multi-horizon time-series forecasting. 

Traditional neural network models (like LSTMs or RNNs) process sequences sequentially but struggle to identify long-range historical cycles and cannot incorporate static variables (like geographical district or crop category). Standard Transformers, while excellent at learning long-range dependencies, struggle with time-series data because they do not distinguish between features known in advance (like calendar months) and features only known in the past (like historical transaction volume).

TFT solves these challenges by combining **recurrent neural structures** (for local sequence processing) with **self-attention blocks** (for capturing long-range seasonal patterns) and **specialized gating mechanisms** (to adaptively select variables and prevent overfitting).

---

## 2. Why We Chose TFT (Comparison with Baseline Models)

To model price variations across **31 Lakh (3.1 million) rows** of historical daily agricultural transactions in Maharashtra, we compared TFT against three industry-standard baselines:

| Forecasting Model | Cross-Series Learning | Handles Static Metadata | Multi-Horizon Grounding | Interpretability | Verdict for Sahyadri 2.0 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **ARIMA / SARIMAX** | ❌ No (Single-series only) | ❌ No | ❌ Poor | ⚠️ Linear only | **Rejected**: Cannot share patterns between different mandis or handle crop categories. |
| **XGBoost / Random Forest** |  Yes | ⚠️ Flattened only | ❌ Struggles with sequence context | ⚠️ Feature importance only | **Rejected**: Fails to capture sequential temporal relationships and correlation over time. |
| **Standard LSTM / Seq2Seq** |  Yes | ❌ No (Requires manual flattening) | ⚠️ Accumulates errors recursively | ❌ Black box | **Rejected**: Recursive predictions accumulate errors rapidly over a 30-day forecast horizon. |
| **Temporal Fusion Transformer (TFT)** |  **Yes** (Learns from all 3.1M rows) |  **Yes** (Dense Entity Embeddings) |  **Yes** (Direct Multi-Horizon Decoder) |  **High** (Temporal Self-Attention weights) |  **Selected (Best-in-class)** |

### Key Reasons for Choosing TFT:
1. **Shared Representations**: Instead of training thousands of individual models for each mandi and crop combination, a single TFT model learns global features across all datasets. For example, if Onion prices spike in Nashik APMC, the model automatically shares this pricing momentum with nearby private markets.
2. **Handling Static vs. Dynamic Covariates**: TFT naturally segregates variables that do not change (e.g., district name, crop type) from variables that do (e.g., monthly temperature, weekly trade volume), mapping each to its correct neural layer.
3. **Probabilistic Quantile Forecasts**: Instead of predicting a single deterministic future price (which is rarely accurate in volatile agricultural markets), TFT outputs three distinct price bounds simultaneously (10th, 50th, and 90th percentiles).

---

## 3. Core Architectural Features of TFT

The TFT architecture consists of five major subcomponents, each designed to resolve a specific time-series bottleneck:

```
+---------------------------------------------------------------------------------+
| 1. Static Covariate Encoders (Dense Embeddings)                                 |
| Maps static categoricals (crop name, district) to dense vectors, which are      |
| distributed as context to the LSTMs and Attention layers.                       |
+---------------------------------------------------------------------------------+
                                      |
                                      v
+---------------------------------------------------------------------------------+
| 2. Variable Selection Networks (VSN)                                            |
| Dynamically evaluates and filters out noisy or irrelevant inputs. If price      |
| volatility is high, VSN boosts the weights of past prices. During normal times, |
| it boosts seasonal inputs (calendar month).                                    |
+---------------------------------------------------------------------------------+
                                      |
                                      v
+---------------------------------------------------------------------------------+
| 3. Temporal LSTM Encoder-Decoder                                                |
| Encodes historical sequences (the 12-week lookback) and decodes the forecast    |
| horizon (the 4-week future predictions), preserving temporal order.            |
+---------------------------------------------------------------------------------+
                                      |
                                      v
+---------------------------------------------------------------------------------+
| 4. Multi-Head Self-Attention Block                                              |
| Allows the model to look back at specific historical milestones (e.g., aligning |
| this November's harvest with last November's pricing trend) to learn cycles.   |
+---------------------------------------------------------------------------------+
                                      |
                                      v
+---------------------------------------------------------------------------------+
| 5. Gated Residual Networks (GRN)                                                |
| Gating layers that allow the network to dynamically skip complex nonlinear      |
| layers if they are not needed, preventing overfitting on small sequence sets.   |
+---------------------------------------------------------------------------------+
```

---

## 4. What We Implemented in Our Application

In Project Sahyadri 2.0, the TFT model is integrated into a unified pipeline:

### A. Sequence Window Configuration
* **Lookback History (Encoder Length)**: **12 Weeks (84 Days)**. The model takes the last 12 weeks of historical records to understand market behavior.
* **Forecast Horizon (Decoder Length)**: **4 Weeks (30 Days)**. The model projects prices for Week 1 (Day 7), Week 2 (Day 14), and Week 4 (Day 30).
* **Validation Split**: The final 4 weeks of the dataset are held out entirely as a validation set, while all prior records serve as the training set.

### B. Feature Categorization Map
The features are engineered in [sahyadri_colab_training.py](file:///c:/Users/Shashank/OneDrive/Desktop/data_sets_fetched/market_and_transaction_data/sahyadri_colab_training.py) and classified as follows:
* **Static Categoricals**: `commodity`, `district`, `source_type`, `market_name` (entity embeddings).
* **Time-Varying Known Categoricals**: `month` (captures harvest and festival seasonality).
* **Time-Varying Known Reals**: `time_idx` (chronological integer index).
* **Time-Varying Unknown Reals**: `modal_price` (historical target), `quantity` (trade volume), and `volatility_4w` (rolling 4-week standard deviation representing stability).

### C. Quantile Loss Parameters
The model is trained on **Quantile Loss** to predict three bounds:
* **10th Percentile ($q=0.10$):** Output as the **Minimum Price** (worst-case pricing).
* **50th Percentile ($q=0.50$):** Output as the **Modal Price** (most expected market rate).
* **90th Percentile ($q=0.90$):** Output as the **Maximum Price** (best-case pricing).

### D. Model Training Specifications
* **Accelerator**: Dual Tesla T4 GPUs (Kaggle)
* **Optimization**: Ranger Optimizer (combines RAdam and Lookahead) with a learning rate of `0.03`.
* **Batch Size**: `256` sequences.
* **Model Checkpoint**: Converged at **Epoch 7** (representing 8 full epochs and 50,112 weight updates) with a final validation loss (`val_loss`) score of **`464.134`**.

---

## 5. How the Underlying Production System Works

When a farmer queries a crop forecast in the frontend, the FastAPI backend routes the query through [`forecast_agent.py`](file:///c:/Users/Shashank/OneDrive/Desktop/data_sets_fetched/market_and_transaction_data/backend/app/agents/forecast_agent.py):

1. **Database Pull**: Retrieves the daily price and volume records for the queried commodity and district from SQLite `sahyadri_data.db`.
2. **Weekly Aggregation**: Rounds transaction dates to their nearest Sunday and averages prices and sums quantities to match the model's weekly time steps.
3. **Continuous Sequence Check**:
   * If there are **fewer than 6 weeks** of valid historical data, the system automatically redirects the query to the **Heuristic Fallback Engine** (which calculates a linear slope and applies harvest coefficients) to prevent model failure.
4. **TFT Model Inference**:
   * If the sequence is valid, it appends 4 future steps (setting future volumes to `0` and future months to the target calendar months).
   * It loads the `sahyadri_tft_final.ckpt` model weights in CPU mode.
   * Feeds the compiled input tensors into the model and extracts the quantile projections.
5. **Chart Rendering**: Sends the predictions back to the React UI to render 3D forecast charts on the farmer's dashboard.
