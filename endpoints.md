# Meadow Intelligence Suite – Backend Endpoints

Cache predictions and such

## 1. Authentication & Accounts
- **POST /auth/register**  
  Create new user.

- **POST /auth/login**  
  Authenticate + issue token.

- **POST /auth/logout**  
  Revoke session.

- **GET /account/me**  
  Fetch current user profile.

- **PATCH /account/me**  
  Update preferences (default crops, notifications).

- **GET /account/subscription**  
  Retrieve free/pro status. *(future-proofing)*

- **PATCH /account/subscription**  
  Upgrade/downgrade plan. *(future-proofing)*

---

## 2. Climate & Predictions
- **GET /climate/forecast**  
  Year-long climate predictions (for dashboard graphs).

- **GET /climate/history**  
  Historical climate data (for baselines).

- **GET /predictions/crops**  
  Ranked list of crops by score.  
  - **Query params:** region, planting window.

- **GET /predictions/crop/:id**  
  Detailed prediction for a specific crop (all windows).

- **POST /predictions/compare**  
  Compare multiple crops across windows.  
  - **Body:** crop IDs, timeframe.

---

## 3. Graphs & Analysis
- **GET /graphs/default**  
  Default dashboard graphs (climate forecast + top crops).

- **POST /graphs/custom**  
  Request a dynamic graph.  
  - **Body:** crop(s), timeframe, representation.

- **GET /analysis/windows**  
  List of crop windows with stats.

- **POST /analysis/playground**  
  Send selected crops to playground for deeper analysis.  
  - **Body:** crop IDs.

- **POST /analysis/nlp-query** *(optional AI)*  
  Transform user query into structured comparison.  
  - **Body:** `{ query: "Compare maize vs sorghum in November" }`

---

## 4. Diagnosis & Intervention
- **GET /diagnosis/status**  
  Predicted vs actual climate comparison.

- **GET /diagnosis/alerts**  
  Warnings when deviations exceed threshold.

- **POST /diagnosis/interventions**  
  Suggest possible interventions.  
  - **Body:** current deviation data.

---

## 5. Admin / Utility
- **GET /system/health**  
  Service health check.

- **GET /system/usage**  
  Compute usage (for cost monitoring).
