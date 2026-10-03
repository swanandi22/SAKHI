# SAKHI: Smart Assistance for keeping HER informed

SAKHI is a journey-level contextual safety intelligence layer over routing.

## Project Structure

- `frontend/`: React Native + Expo application (To be implemented)
- `backend/`: FastAPI backend
- `ml/`: Machine learning models and notebooks (To be implemented)
- `data/`: Datasets and local databases (To be implemented)
- `docs/`: Documentation (To be implemented)

## Running the Backend Locally

### Prerequisites
- Python 3.9+

### Setup

1. Navigate to the backend directory:
   ```bash
   cd backend
   ```

2. Create a virtual environment:
   ```bash
   python -m venv venv
   ```

3. Activate the virtual environment:
   - Windows: `venv\Scripts\activate`
   - Linux/Mac: `source venv/bin/activate`

4. Install dependencies:
   ```bash
   pip install -r requirements.txt
   ```

### Running the Server
```bash
uvicorn app.main:app --reload
```
The server will be available at http://127.0.0.1:8000
API documentation will be at http://127.0.0.1:8000/docs

### Running Tests
```bash
pytest tests/
```

## Architecture

### Routing Phase (Phase 2)
SAKHI currently utilizes an abstraction for routing services to remain uncoupled from specific third-party routing APIs.
- **Provider:** OpenStreetMap / OSRM (Public Demo API)
- **Profile:** Walking (`foot` profile)
- **Segmentation Strategy:** SAKHI parses OSRM "steps" and maps them directly into discrete `JourneySegment`s. These segments hold detailed GeoJSON `LineString` coordinates and act as our stable domain objects for later risk evaluation.

*Note: This prototype currently uses walking routes. Multimodal walking + transit journey support will be added in a later phase.*

### API Endpoints

#### `POST /api/v1/journeys`
Creates a journey by calling the routing provider and returns an ordered list of `JourneySegment`s.

**Example Request:**
```json
{
  "origin": { "latitude": 18.922, "longitude": 72.827 },
  "destination": { "latitude": 18.930, "longitude": 72.835 }
}
```

**Example Response:**
```json
{
  "journey_id": "84c8a299-1bd1-4d10-94e8-42217c2be6d5",
  "origin": { "latitude": 18.922, "longitude": 72.827 },
  "destination": { "latitude": 18.930, "longitude": 72.835 },
  "distance_m": 2174.6,
  "duration_s": 167.0,
  "segments": [
    {
      "segment_id": "48b6c008-dbef-4a1d-a02b-8a8b1a8d052d",
      "journey_id": "84c8a299-1bd1-4d10-94e8-42217c2be6d5",
      "sequence": 1,
      "mode": "foot",
      "start_location": { "latitude": 18.931757, "longitude": 72.827036 },
      "end_location": { "latitude": 18.932862, "longitude": 72.827888 },
      "distance_m": 165.2,
      "duration_s": 118.9,
      "geometry": {
        "type": "LineString",
        "coordinates": [[72.827036, 18.931757], [72.827888, 18.932862]]
      }
    }
  ]
}
```

### Contextual Safety Risk Model (Phase 3A)

SAKHI's contextual safety risk score estimates safety-relevant contextual conditions for a journey segment. **It is not a crime probability, individual crime prediction, or guarantee of safety.**

The current prototype follows this logical flow:
JourneySegment -> extracted features -> historical baseline -> prototype heuristic -> 
isk score + confidence`n
#### Key Principles:
- **Historical Baseline**: Aggregate historical data acts as a regional prior/context signal, not a street-level ground truth.
- **Confidence**: Confidence is independent from risk. Missing context deterministically decreases the confidence score.
- **Prototype Heuristics**: The weights currently combining spatial, temporal, and dynamic factors into a final 0-100 score are simple heuristic proxies. They are NOT scientifically validated and will be replaced/augmented by ML models in future phases.
- **Synthetic Data**: If synthetic granular data is used in future phases for ML demonstration, it will be explicitly labelled.


### Phase 3B — XGBoost Contextual Risk Model

In Phase 3B, a prototype XGBoost Regressor model is integrated. It produces a continuous 0-100 contextual risk score. This model acts as an intelligence layer predicting safety-relevant contextual conditions.

#### Key Principles:
- **Synthetic Prototype Dataset**: The model is trained on explicitly synthetic prototype data (generated via a mathematical relationship with controlled noise), not real incident data. We avoid treating NCRB aggregate data as street-level labels to prevent inaccurate mapping.
- **Not Crime Prediction**: The output is a prototype contextual risk score, NOT a crime probability or individual crime prediction.
- **Evaluation**: Training metrics (MAE, MSE, RMSE, R²) evaluate the model's ability to learn the synthetic dataset's underlying mathematical relationship. They demonstrate the ML pipeline but do NOT establish real-world predictive validity.
- **Heuristic Fallback**: If the ML model is unavailable or inference fails, SAKHI gracefully falls back to the deterministic Phase 3A heuristic.
- **Independent Confidence**: The confidence score remains entirely independent of the ML model, driven purely by the availability of input context.


### Phase 3C — SHAP Explainability

Phase 3C adds explainability to the XGBoost contextual risk model using SHAP (SHapley Additive exPlanations).

#### Key Principles:
- **Contextual Insights, Not Crime Prediction**: SHAP explains which contextual features contributed most to the model's contextual risk score. It does NOT identify why a crime will happen or predict dangerous locations.
- **Positive vs Negative Contributions**: Features are ranked by absolute SHAP magnitude. A positive SHAP value increases the contextual risk score, while a negative SHAP value decreases it.
- **Independent Confidence**: The confidence score remains entirely independent of SHAP values or the ML model's prediction. It evaluates purely the availability of contextual evidence.
- **Heuristic Fallback**: SHAP is unavailable when the ML model is not active. In such cases, SAKHI seamlessly falls back to the deterministic Phase 3A heuristic without fabricating SHAP values.
- **Prototype Limitation**: SHAP explains the behavior of the trained prototype model on synthetic data; it does not validate the model's real-world predictive accuracy.


### Phase 4 — Risk-Aware Route Ranking

Phase 4 aggregates segment-level contextual risk into route-level metrics and provides three distinct route options: Safest, Balanced, and Fastest.

#### Key Principles:
- **Route Aggregation**: A duration-weighted average is used to aggregate segment risk scores into a total route risk score. This accounts for time-based exposure.
- **Normalization**: Min-Max normalization is used across candidate routes to align travel time and risk scores onto comparable 0-1 scales before applying weighting coefficients. When all candidates share identical values, the normalized value is 0.0.
- **Uncertainty Penalty**: Calculated as 1 - normalized_confidence. Confidence is treated strictly as an uncertainty measure (lack of evidence), not a proxy for danger.
- **Route Cost Formula**: cost = alpha * normalized_time + beta * normalized_risk + gamma * uncertainty_penalty (lowest cost wins).
- **Safest / Balanced / Fastest Profiles**: The ranking engine applies different coefficient weights to produce the three options.
- **Prototype Limitation**: The lpha, eta, and gamma route-ranking coefficients are prototype decision parameters. They have NOT been empirically validated as universal safety preferences, nor does the system guarantee absolute safety.


### Phase 5 — Dynamic Contextual Rerouting

Phase 5 adds support for real-time (simulated) contextual updates, allowing SAKHI to evaluate changing safety signals during an active journey.

#### Key Principles:
- **Contextual Safety Updates**: A new endpoint \POST /api/v1/journeys/{journey_id}/context-update\ accepts simulated safety events (e.g. \alidated_report\, \environmental_change\).
- **Intelligent Reranking**: Modifying a segment's context triggers the existing Phase 3 risk and SHAP pipeline. The updated segment risk cascades into the Phase 4 route ranking algorithm, potentially altering the recommended route.
- **In-Memory Prototype State**: Journey candidates and ranking state are temporarily held in an in-memory dictionary. This avoids unnecessary database/Redis dependencies while proving out the event-driven risk recalculation workflow.
- **Prototype Limitations**: Context updates use the \simulated_demo\ source. This architecture is designed to demonstrate dynamic recalculation and does NOT represent real-time integration with live CCTV, crowdsourced safety portals, or police intelligence feeds.


### Phase 6 — Mobile Prototype Frontend

Phase 6 introduces a fully functional React Native mobile prototype built with Expo. This app consumes the existing SAKHI FastAPI intelligence backend.

#### Mobile Setup
1. Open a terminal in the \mobile\ directory.
2. Run \
pm install\.
3. Configure your API base URL:
   - By default, the app uses \http://localhost:8000/api/v1\ for local emulators.
   - To run on a physical device, find your computer's local network IP (e.g., \192.168.1.5\) and set the environment variable when starting Expo, or create a \.env\ file in the \mobile\ directory with \EXPO_PUBLIC_API_URL=http://<YOUR_IP>:8000/api/v1\.
4. Start the app: \
px expo start\.
5. Use the Expo Go app on your phone, or run in an iOS/Android simulator.

#### Key Features & Prototype Limitations:
- **Mobile UX**: The UI is optimized for touch with native maps (\
eact-native-maps\).
- **Live Explainability**: SHAP explanations are rendered natively in mobile bottom-sheets/cards.
- **Demo Context**: The \Simulate Safety Report\ button demonstrates real-time dynamic rerouting by communicating with the backend.
- **No Native Compilations**: The prototype avoids complex native libraries requiring EAS builds (standard \
eact-native-maps\ works in Expo Go).
- **Limitations**: The SOS and Offline modes are UI demonstrations only. They do not yet integrate with real emergency dispatch APIs or local SQLite databases.

