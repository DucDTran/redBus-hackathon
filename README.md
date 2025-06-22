# **RedBus Demand Forecasting Hackathon**

This repository contains the solution for the RedBus Data Decode Hackathon. The goal of this competition is to predict the demand for bus journeys (total seats booked) at a route level, 15 days prior to the date of departure.

## **Table of Contents**

* [Problem Statement](#bookmark=id.ps18yg8m6tgc)  
* [Evaluation Metric](#bookmark=id.7vj243heowxe)  
* [Data Dictionary](#bookmark=id.efvu4lnj71x3)  
* [Methodology](#bookmark=id.qx78f7ah1d5v)  
  * [1\. Data Preprocessing and Snapshot Creation](#bookmark=id.5vivuosx8td)  
  * [2\. Feature Engineering](#bookmark=id.dzh5pubtsntt)  
  * [3\. Modeling and Hyperparameter Tuning](#bookmark=id.7a8fvaj6wzuw)  
  * [4\. Ensembling](#bookmark=id.8o33p8c2tkdr)  
* [How to Run](#bookmark=id.rkmmfrgwqfwd)  
  * [Dependencies](#bookmark=id.xagfnm3utiar)  
  * [Execution](#bookmark=id.lmoteovx6yxd)  
* [Code Overview](#bookmark=id.qrw57hr5cnvz)

## **Problem Statement**

Demand for bus journeys is influenced by a range of factors, including holiday calendars, wedding seasons, long weekends, school vacations, and exam schedules. Additionally, regional holidays can affect different areas differently, and day-of-week effects further shape booking patterns. However, not all holidays significantly impact demand, making it a complex and non-trivial problem to model.

**The Challenge:**

The task is to develop a model that accurately forecasts demand for bus journeys. The solution uses historical booking data, including seats booked, date of journey, date of issue, and search data.

**Goal:**

Predict the demand (total number of seats booked) for each journey at the route level, **15 days before the actual date of journey (doj)**.

For example, for a route from Source City "A" to Destination City "B" with a doj of 30-Jan-2025, the model must predict the final seat count on 16-Jan-2025.

### **Evaluation Metric**

The evaluation metric for this hackathon is **RMSE (Root Mean Squared Error)**.

RMSE=n1​i=1∑n​(yi​−y^​i​)2​  
Where:

* n is the number of observations.  
* yi​ is the actual value.  
* y^​i​ is the predicted value.

## **Data Dictionary**

The dataset is provided in three files: train.csv, test.csv, and transactions.csv.

**1\. train.csv**

| Variable | Description |
| :---- | :---- |
| doj | The date on which the bus journey is scheduled to take place. |
| srcid | Unique identifier for the source city of the journey. |
| destid | Unique identifier for the destination city of the journey. |
| final\_seatcount | Total number of seats booked at the end of the journey date (Target Variable). |

**2\. test.csv**

| Variable | Description |
| :---- | :---- |
| route\_key | Unique identifier for each row in the test set. |
| doj | The date on which the bus journey is scheduled to take place. |
| srcid | Unique identifier for the source city of the journey. |
| destid | Unique identifier for the destination city of the journey. |

**3\. transactions.csv**

| Variable | Description |
| :---- | :---- |
| doj | The date on which the bus journey is scheduled to take place. |
| doi | The date when the ticket was booked. |
| dbd | Days Before Departure: The number of days remaining until the journey date from the date of issue. |
| srcid | Unique identifier for the source city of the journey. |
| destid | Unique identifier for the destination city of the journey. |
| srcid\_tier | The tier classification of the source city (e.g., Tier 1, Tier 2). |
| destid\_tier | The tier classification of the destination city (e.g., Tier 1, Tier 2). |
| cumsum\_seatcount | The cumulative number of seats sold till a given doi. |
| cumsum\_searchcount | The cumulative number of searches till a given doi. |

## **Methodology**

The solution follows a structured approach, from data preparation and extensive feature engineering to robust modeling and ensembling.

### **1\. Data Preprocessing and Snapshot Creation**

Since the goal is to predict demand 15 days before the journey, the transactions.csv data is filtered to create a snapshot of booking and search activity at specific points in time.

* **dbd \= 15 Snapshot:** Captures the cumulative seat and search counts exactly 15 days before departure. This is the primary source of features for our model.  
* **dbd \= 22 Snapshot:** Captures the same metrics 22 days before departure.  
* **Velocity Features:** By merging these two snapshots, we engineer features that capture the booking momentum in the week leading up to our prediction point (from day 22 to day 15), such as seats\_booked\_7\_days\_previous and seats\_searched\_7\_days\_previous.

### **2\. Feature Engineering**

A rich set of features was created to capture various aspects of demand.

* **Date-based Features:**  
  * Standard features like day\_of\_month, day\_of\_week, month, week\_of\_year.  
  * An is\_weekend flag to capture weekly patterns.  
* **Event-based Features (using allowed external data):**  
  * **Holidays:** The holidays library for India (holidays.IN) is used to flag if a journey date is a public holiday.  
  * **Long Weekends:** A function identifies journeys that are part of a long weekend (e.g., a Friday before a Monday holiday).  
  * **Holiday Proximity:** days\_to\_next\_holiday is calculated to model the urgency of travel as a holiday approaches.  
  * **Custom Events:** Domain knowledge is applied to create seasonal features like Summer\_Vacation (May-June), Peak\_Wedding\_Season (Nov-Jan), and Diwali\_Season (October).  
* **Behavioral Features:**  
  * A search\_to\_book\_ratio (cumsum\_seatcount / cumsum\_searchcount) is calculated to gauge user intent and conversion rates at the 15-day mark.  
* **Time-Series Features:**  
  * To capture route-specific trends, rolling window features are created on the target variable (final\_seatcount).  
  * **Rolling Averages:** Route-level average demand over the last 3, 6, and 9 months (route\_avg\_demand\_last\_3m, etc.).  
  * **Momentum Ratios:** Ratios of the rolling averages (e.g., momentum\_3m\_vs\_6m) are calculated to measure the acceleration or deceleration of demand for a specific route.

### **3\. Modeling and Hyperparameter Tuning**

* **Cross-Validation:** A TimeSeriesSplit cross-validation strategy is used. This is crucial for time-dependent data as it ensures the model is always validated on data from a future period relative to its training data, preventing data leakage.  
* **Models:** Four powerful gradient boosting and tree-based models are trained:  
  1. **LightGBM (LGBMRegressor)**  
  2. **XGBoost (XGBRegressor)**  
  3. **Random Forest (RandomForestRegressor)**  
  4. **Scikit-learn's GradientBoostingRegressor**  
* **Hyperparameter Tuning:** The Optuna framework is used to perform automated hyperparameter tuning for each model. Optuna efficiently searches for the optimal set of parameters (e.g., learning rate, tree depth) by minimizing the average RMSE across the time-series cross-validation folds.

### **4\. Ensembling**

To create a more robust and accurate final prediction, the outputs of the four individual models are combined using a **weighted average ensemble**.

* **Weight Calculation:** The weights are not equal. They are calculated to be inversely proportional to each model's cross-validation RMSE. This gives a higher weight to the models that performed better during validation.wi​=∑j=14​(1/RMSEj​)1/RMSEi​​  
* **Final Prediction:** The final prediction is the weighted sum of the predictions from the LightGBM, XGBoost, Random Forest, and Gradient Boosting models. Any resulting negative predictions for seat counts are adjusted to zero.

## **How to Run**

### **Dependencies**

The script requires Python and several open-source libraries. You can install them using pip:

pip install \--upgrade holidays  
pip install \--q optuna catboost xgboost lightgbm  
pip install pandas numpy scikit-learn matplotlib gdown

### **Execution**

The script is designed to be self-contained. It will automatically download the necessary data files using gdown.

1. Place the Python script (redbus\_data\_decode\_hackathon.py) in your desired project directory.  
2. Run the script from your terminal:  
   python redbus\_data\_decode\_hackathon.py

The script will perform all steps:

1. Download train.csv, test.csv, and transactions.csv.  
2. Perform data preprocessing and feature engineering.  
3. Run Optuna studies to find the best hyperparameters for all four models (this may take a significant amount of time).  
4. Train the final models on the entire dataset with the best parameters.  
5. Generate predictions on the test set.  
6. Save the following output files:  
   * submission.csv: The final blended ensemble predictions.  
   * lgbm\_submission.csv: Predictions from the LightGBM model.  
   * xgb\_submission.csv: Predictions from the XGBoost model → **This currently has the lowest RMSE score**  
   * rf\_submission.csv: Predictions from the Random Forest model.  
   * gb\_submission.csv: Predictions from the Gradient Boosting model.

## **Code Overview**

The Python script redbus\_data\_decode\_hackathon.py is organized as follows:

* **Library and Data Imports:** Installs and imports all necessary packages and downloads the datasets.  
* **Initial Merging:** Creates the 7-day velocity features by merging transaction snapshots.  
* **Feature Engineering Functions:**  
  * add\_date\_features: Creates time-based features.  
  * add\_event\_features: Creates holiday and custom event features.  
  * add\_behavioral\_features: Creates ratio features.  
  * encode\_categorical\_features: Encodes categorical columns into a numerical format.  
* **Optuna Objective Functions (objective\_lgbm, objective\_xgb, etc.):**  
  * Defines the hyperparameter search space for each model.  
  * Implements the TimeSeriesSplit cross-validation loop.  
  * Contains the logic for creating rolling average and momentum features within each fold to prevent data leakage.  
  * Returns the mean RMSE for a given trial.  
* **Model Training and Prediction:**  
  * Initializes and runs the Optuna studies.  
  * Retrieves the best parameters found.  
  * Creates the final feature set on the combined train/test data, ensuring correct temporal calculations.  
  * Trains each of the four models on the full training data using the best hyperparameters.  
  * Generates predictions on the final test set.  
* **Ensembling and Submission:**  
  * Calculates the inverse RMSE weights.  
  * Combines the individual model predictions into a final weighted average.  
  * Saves the final submission files.
