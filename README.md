Methodology
The solution follows a structured approach, from data preparation and extensive feature engineering to robust modeling and ensembling.

1. Data Preprocessing and Snapshot Creation
Since the goal is to predict demand 15 days before the journey, the transactions.csv data is filtered to create a snapshot of booking and search activity at specific points in time.

dbd = 15 Snapshot: Captures the cumulative seat and search counts exactly 15 days before departure. This is the primary source of features for our model.
dbd = 22 Snapshot: Captures the same metrics 22 days before departure.
Velocity Features: By merging these two snapshots, we engineer features that capture the booking momentum in the week leading up to our prediction point (from day 22 to day 15), such as seats_booked_7_days_previous and seats_searched_7_days_previous.
2. Feature Engineering
A rich set of features was created to capture various aspects of demand.

Date-based Features:

Standard features like day_of_month, day_of_week, month, week_of_year.
An is_weekend flag to capture weekly patterns.
Event-based Features (using allowed external data):

Holidays: The holidays library for India (holidays.IN) is used to flag if a journey date is a public holiday.
Long Weekends: A function identifies journeys that are part of a long weekend (e.g., a Friday before a Monday holiday).
Holiday Proximity: days_to_next_holiday is calculated to model the urgency of travel as a holiday approaches.
Custom Events: Domain knowledge is applied to create seasonal features like Summer_Vacation (May-June), Peak_Wedding_Season (Nov-Jan), and Diwali_Season (October).
Behavioral Features:

A search_to_book_ratio (cumsum_seatcount / cumsum_searchcount) is calculated to gauge user intent and conversion rates at the 15-day mark.
Time-Series Features:

To capture route-specific trends, rolling window features are created on the target variable (final_seatcount).
Rolling Averages: Route-level average demand over the last 3, 6, and 9 months (route_avg_demand_last_3m, etc.).
Momentum Ratios: Ratios of the rolling averages (e.g., momentum_3m_vs_6m) are calculated to measure the acceleration or deceleration of demand for a specific route.
3. Modeling and Hyperparameter Tuning
Cross-Validation: A TimeSeriesSplit cross-validation strategy is used. This is crucial for time-dependent data as it ensures the model is always validated on data from a future period relative to its training data, preventing data leakage.

Models: Four powerful gradient boosting and tree-based models are trained:

LightGBM (LGBMRegressor)
XGBoost (XGBRegressor)
Random Forest (RandomForestRegressor)
Scikit-learn's GradientBoostingRegressor
Hyperparameter Tuning: The Optuna framework is used to perform automated hyperparameter tuning for each model. Optuna efficiently searches for the optimal set of parameters (e.g., learning rate, tree depth) by minimizing the average RMSE across the time-series cross-validation folds.

4. Ensembling
To create a more robust and accurate final prediction, the outputs of the four individual models are combined using a weighted average ensemble.

Weight Calculation: The weights are not equal. They are calculated to be inversely proportional to each model's cross-validation RMSE. This gives a higher weight to the models that performed better during validation.
w_i=
frac1/RMSE_isum_j=1 
4
 (1/RMSE_j)

Final Prediction: The final prediction is the weighted sum of the predictions from the LightGBM, XGBoost, Random Forest, and Gradient Boosting models. Any resulting negative predictions for seat counts are adjusted to zero.
