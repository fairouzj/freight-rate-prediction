# Freight Rate Prediction
This project aims to predict the freight load prices using the provided labeled data, it generates predictions for 12000 loads (November-December) and a fixed December shipment scenarios
## Project files

```text
freight-solution-ml/
├── data/
│   ├── train-test.csv
│   ├── validation.csv
│   ├── validation-predictions-template.csv
│   ├── december-chart-inputs.csv
│   └── validation_predictions.csv
├── scorer_results/
│   └── candidate_december.png
├── explore1.ipynb
├── requirements.txt
├── score.py
└── README.md
```

## Setup and execution
The notebook was developed in VSCode using Anaconda Python and scikit-learn 1.6.1
1. Open the project folder in VS Code
2. Install python and jupyter extensions
3. Install the dependencies in python environment you will use:   

```bash
python -m pip install -r requirements.txt
```

4. Open explore1.ipynb and select the python environment as the notebook kernel
5. Restart the kernel and run all cells in order

The notebook uses the current working directory as the project directory. Run it with the project folder as the working directory.
Running the notebook trains the models and regenerates both prediction files, it overwrites the predicted_rate column in data/december-chart-inputs.csv

## Data exploration and cleaning
The labeled dataset has 48000 loads from January 2025 through October 2025, the final prediction dataset has 12000 loads from November through December 2025. The development data has:
- 300 missing weights
- 292 non positive weights
- 374 missing market-index values 
- No duplicate load IDs 

Non-positive weights are treated as missing. A preprocessing pipeline fills missing numerical values with medians learned only from the relevant training partition and includes missing-value indicators.
Categorical features are encoded using one-hot encoding with support for unseen categories
Large posted rates are retained because there is insufficient evidence to classify them as errors, so they remain in training/validation.

## Features 
The selected model uses: distance, weight, pickup city, delivery city and equipment type
load_id us used only to match predictions to the output template
Coordinates, market_index and quote_signal are not included in the selected model, their predictive value was not evaluated in this implementation. 
An additional experiment also tested day of week, weekend and annual sine/cosine features, these increased September error for the tested model, so date features were excluded.

## Training and Validation
The development data is split chronologically:

| Partition | Period | Loads |
|---|---|---:|
| Training | January–August 2025 | 38,477 |
| Development | September 2025 | 4,670 |
| Test | October 2025 | 4,853 |

This reflects the final task of learning from earlier loads to predict later loads. 
September is used for model comparison and parameter selection, October evaluation trains on January-September and predicts October
October was initially evaluated before additional parameter tuning. The later settings were selected using September MAE, but October should therefore be described as a previously inspected evaluation period rather than a completely untouched final holdout.
After evaluation, the selected pipeline is refitted on all 48000 labeled loads to generate submission predictions.

## Model comparison
Mean absolute error (MAE) measures the average absolute difference between the predicted and the actual prices in dollars (lower values are better).

| Model | September MAE |
|---|---:|
| Training-median baseline | $1,151.11 |
| Random forest | $154.92 |
| Original gradient boosting | $116.41 |
| Tuned gradient boosting without dates | **$110.87** |
| Tuned gradient boosting with dates | $118.42 |

The selected model is GradientBoostingRegressor with the following parameters: 

```python
loss="absolute_error"
n_estimators=200
learning_rate=0.05
max_depth=4
min_samples_leaf=20
random_state=42
```

Absolute-error loss matches the primary evaluation metric and reduces the influence of extreme target values compared with squared-error loss.
The tuned model achieved October MAE of $115.76 after training on January-September

## Final Predictions
Predictions are matched to the template rows by load_id rather than relying on matching row order. The final file is: 

```text
data/validation_predictions.csv
```

And it contain the columns: 

```text
load_id,predicted_rate
```

The notebook checks row counts, unique IDs, matching ID sets and finite positive predictions.

## December chart
The december scenario has 31 rows with fixed shipment details:
- Lexington to Fort Wayne
- 360 miles
- Dry Van
- 32,000 lb
With the only changes being in the dates, and because the selected model excludes date features, all rows have the same prediction and the chart is flat. This follows the selected model's behavior, the chart is a prediction scenario; actual December prices are not supplied.

## Run the supplied scorer 
From the project folder: 

```bash
python score.py --predictions data/validation_predictions.csv --december-predictions data/december-chart-inputs.csv
```

If the terminal uses a different Python environment from the notebook, install the requirements in that environment or run the command using the notebook's Python interpreter.
The scorer validates all 12000 final predictions and 31 december predictions, then it creates: 

```text
scorer_results/candidate_december.png
```

The scorer checks output validity and generates the chart, it doesn't calculate accuracy of the prediction. The employer calculates final hidden validation metrics after submission.

## Limitations
- Large errors remain possible on unusually expensive loads
- Only one partial year of labeled data is available
- The selected model cannot represent date-driven price changes
- Repeated comparisons on september can overfit model selection to that month
- October was inspected before the additional tuning
- November-December accuracy cannot be measured because their actual prices are unavailable.

## Submission deliverables
- Accessible GitHub repository with code, dependencies and run instructions
- validation_predictions.csv
- PDF report explaining validation and splitting, including candidate_december.png
- A 2-3 min loom walkthrough