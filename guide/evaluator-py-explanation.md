# Evaluator.py Documentation

## Overview

The `evaluator.py` file provides a comprehensive testing and evaluation framework for price prediction models. It runs predictions on test data, calculates error metrics, generates interactive visualizations, and provides detailed performance analysis with confidence intervals.

---

## Imports

```python
import re
from sklearn.metrics import mean_squared_error, r2_score
import pandas as pd
import plotly.express as px
import plotly.graph_objects as go
from itertools import accumulate
import math
from tqdm.notebook import tqdm
from concurrent.futures import ThreadPoolExecutor
```

- **`re`**: Regular expressions for parsing prediction outputs
- **`sklearn.metrics`**: MSE and R² score calculations
- **`pandas`**: DataFrame for organizing results
- **`plotly.express` & `plotly.graph_objects`**: Interactive visualizations
- **`itertools.accumulate`**: Efficient running sum calculations
- **`math`**: Mathematical operations for statistics
- **`tqdm.notebook`**: Progress bars for Jupyter
- **`concurrent.futures.ThreadPoolExecutor`**: Parallel prediction execution

---

## Constants

### Terminal Colors

```python
GREEN = "\033[92m"
YELLOW = "\033[93m"
RED = "\033[91m"
RESET = "\033[0m"
COLOR_MAP = {"red": RED, "orange": YELLOW, "green": GREEN}
```

Used for color-coded console output showing prediction quality in real-time.

### Configuration

```python
WORKERS = 5
DEFAULT_SIZE = 200
```

| Constant | Value | Purpose |
|----------|-------|---------|
| `WORKERS` | 5 | Number of parallel threads for predictions |
| `DEFAULT_SIZE` | 200 | Default number of test samples to evaluate |

---

## Tester Class

The `Tester` class orchestrates the complete evaluation pipeline.

### Constructor

```python
def __init__(self, predictor, data, title=None, size=DEFAULT_SIZE, workers=WORKERS):
    self.predictor = predictor
    self.data = data
    self.title = title or self.make_title(predictor)
    self.size = size
    self.titles = []
    self.guesses = []
    self.truths = []
    self.errors = []
    self.colors = []
    self.workers = workers
```

**Parameters:**
- `predictor` (callable): Function that takes an Item and returns a price prediction
- `data` (list[Item]): Test dataset
- `title` (str, optional): Display name for the model (auto-generated if not provided)
- `size` (int, optional): Number of items to test (default: 200)
- `workers` (int, optional): Number of parallel workers (default: 5)

**Instance Attributes:**

| Attribute | Type | Description |
|-----------|------|-------------|
| `predictor` | callable | Prediction function to test |
| `data` | list[Item] | Test dataset |
| `title` | str | Model display name |
| `size` | int | Number of samples to test |
| `titles` | list[str] | Product titles (truncated) |
| `guesses` | list[float] | Model predictions |
| `truths` | list[float] | Actual prices |
| `errors` | list[float] | Absolute errors |
| `colors` | list[str] | Error severity colors |
| `workers` | int | Parallel worker count |

**Example:**
```python
def my_predictor(item):
    # Your prediction logic
    return predicted_price

tester = Tester(my_predictor, test_data, title="My Model v1", size=500)
```

---

## Static Methods

### `make_title(predictor) -> str`

Generates a human-readable title from a function name.

**Parameters:**
- `predictor` (callable): Function to generate title from

**Returns:**
- `str`: Formatted title

**Transformations:**
1. Replace `__` with `.` (for class methods)
2. Replace `_` with spaces
3. Title case
4. Special handling: "Gpt" → "GPT"

**Example:**
```python
def predict_with_gpt_4():
    pass

title = Tester.make_title(predict_with_gpt_4)
print(title)  # "Predict With GPT 4"

# For class method: MyClass.__predict_price
# Result: "MyClass.Predict Price"
```

---

### `post_process(value) -> float`

Extracts numeric price from various string formats.

**Parameters:**
- `value` (str | float): Raw prediction output

**Returns:**
- `float`: Extracted numeric value (0 if no number found)

**Handles:**
- Dollar signs: `"$29.99"` → `29.99`
- Commas: `"1,299.99"` → `1299.99`
- Text with numbers: `"Price is $50"` → `50.0`
- Negative numbers: `"-10.5"` → `-10.5`
- Already numeric: `29.99` → `29.99`

**Example:**
```python
Tester.post_process("$1,299.99")  # 1299.99
Tester.post_process("Price: $50.00")  # 50.0
Tester.post_process("29.99")  # 29.99
Tester.post_process(29.99)  # 29.99
Tester.post_process("No price")  # 0.0
```

**Regex Pattern:**
```python
r"[-+]?\d*\.\d+|\d+"
# Matches: optional sign, decimals, or integers
```

---

## Instance Methods

### `color_for(self, error, truth) -> str`

Determines error severity color based on absolute and relative error.

**Parameters:**
- `error` (float): Absolute error in dollars
- `truth` (float): Actual price

**Returns:**
- `str`: "green", "orange", or "red"

**Logic:**

| Condition | Color | Meaning |
|-----------|-------|---------|
| `error < 40` OR `error/truth < 0.2` | Green | Excellent (< $40 or < 20% error) |
| `error < 80` OR `error/truth < 0.4` | Orange | Acceptable (< $80 or < 40% error) |
| Otherwise | Red | Poor (≥ $80 or ≥ 40% error) |

**Example:**
```python
tester.color_for(15, 100)   # "green" (15% error)
tester.color_for(30, 100)   # "orange" (30% error)
tester.color_for(50, 100)   # "red" (50% error)
tester.color_for(70, 500)   # "green" (14% error, even though $70)
tester.color_for(90, 100)   # "red" (90% error)
```

---

### `run_datapoint(self, i) -> tuple`

Runs prediction on a single datapoint and calculates metrics.

**Parameters:**
- `i` (int): Index of datapoint in `self.data`

**Returns:**
- `tuple`: (title, guess, truth, error, color)

**Process:**
1. Get datapoint from data
2. Call predictor function
3. Post-process prediction to extract number
4. Calculate absolute error
5. Determine color based on error
6. Truncate title to 40 characters

**Example:**
```python
title, guess, truth, error, color = tester.run_datapoint(0)
print(f"{title}: ${guess:.2f} (actual: ${truth:.2f}) - {color}")
# "Wireless Mouse: $28.50 (actual: $29.99) - green"
```

---

### `chart(self, title)`

Creates an interactive scatter plot comparing predictions to actual prices.

**Parameters:**
- `title` (str): Chart title with metrics

**Features:**
- **Scatter points**: Color-coded by error severity (green/orange/red)
- **Reference line**: y=x diagonal showing perfect predictions
- **Hover text**: Shows product title, predicted price, and actual price
- **Equal axes**: Both axes scaled to same range for fair comparison

**Example:**
```python
tester.chart("My Model results<br><b>Error:</b> $15.23 <b>MSE:</b> 450 <b>r²:</b> 85.3%")
```

**Chart Interpretation:**
- **Points on diagonal**: Perfect predictions
- **Points above diagonal**: Over-predictions
- **Points below diagonal**: Under-predictions
- **Green cluster near diagonal**: Good model performance
- **Red points far from diagonal**: Poor predictions

---

### `error_trend_chart(self)`

Creates a running average error chart with confidence intervals.

**Features:**
- **Running mean**: Cumulative average error over all predictions
- **95% Confidence Interval**: Shaded band showing statistical uncertainty
- **Convergence visualization**: Shows if error stabilizes or trends

**Statistics Calculated:**

1. **Running Mean**: 
   ```python
   mean_n = sum(errors[:n]) / n
   ```

2. **Running Standard Deviation**:
   ```python
   std_n = sqrt((sum(e² for e in errors[:n]) / n) - mean_n²)
   ```

3. **95% Confidence Interval**:
   ```python
   CI = 1.96 * (std_n / sqrt(n))
   ```

**Example Output:**
```
Title: "My Model Error: $15.23 ± $2.45"
```

**Interpretation:**
- **Narrow CI**: Consistent predictions
- **Wide CI**: High variance in errors
- **Decreasing trend**: Model improving with more samples
- **Flat trend**: Stable performance

---

### `report(self)`

Generates comprehensive performance report with metrics and visualizations.

**Metrics Calculated:**

1. **Average Absolute Error**:
   ```python
   avg_error = sum(errors) / len(errors)
   ```

2. **Mean Squared Error (MSE)**:
   ```python
   mse = mean_squared_error(truths, guesses)
   ```
   - Penalizes large errors more heavily
   - Lower is better

3. **R² Score (Coefficient of Determination)**:
   ```python
   r2 = r2_score(truths, guesses) * 100
   ```
   - Percentage of variance explained
   - 100% = perfect predictions
   - 0% = no better than mean
   - Negative = worse than mean

**Output:**
1. Error trend chart with confidence intervals
2. Scatter plot with all metrics in title

**Example:**
```python
tester.report()
# Displays:
# 1. Error trend chart: "My Model Error: $15.23 ± $2.45"
# 2. Scatter plot: "My Model results - Error: $15.23 MSE: 450 r²: 85.3%"
```

---

### `run(self)`

Executes the complete evaluation pipeline with parallel processing.

**Process:**
1. **Parallel Execution**: Uses ThreadPoolExecutor to run predictions concurrently
2. **Progress Tracking**: Shows tqdm progress bar
3. **Real-time Feedback**: Prints color-coded errors as they complete
4. **Data Collection**: Stores all results in instance attributes
5. **Report Generation**: Calls `report()` to display visualizations

**Example:**
```python
tester = Tester(my_predictor, test_data, size=200)
tester.run()

# Output:
# 100%|██████████| 200/200 [00:45<00:00, 4.44it/s]
# $12 $8 $45 $15 $3 $22 ...  (color-coded in terminal)
# [Displays error trend chart]
# [Displays scatter plot]
```

**Console Output Colors:**
- 🟢 **Green**: Good predictions (< $40 or < 20% error)
- 🟡 **Yellow**: Acceptable predictions (< $80 or < 40% error)
- 🔴 **Red**: Poor predictions (≥ $80 or ≥ 40% error)

---

## Convenience Function

### `evaluate(function, data, size=DEFAULT_SIZE, workers=WORKERS)`

Shorthand function for quick evaluation.

**Parameters:**
- `function` (callable): Predictor function
- `data` (list[Item]): Test dataset
- `size` (int, optional): Number of samples (default: 200)
- `workers` (int, optional): Parallel workers (default: 5)

**Example:**
```python
from pricer.evaluator import evaluate

def my_predictor(item):
    return item.price * 1.1  # Dummy predictor

evaluate(my_predictor, test_data, size=500, workers=10)
# Equivalent to:
# Tester(my_predictor, test_data, size=500, workers=10).run()
```

---

## Complete Usage Examples

### Example 1: Evaluate a Simple Predictor

```python
from pricer.evaluator import evaluate

# Simple baseline: predict average price
def baseline_predictor(item):
    return 50.0  # Always predict $50

# Evaluate on 200 test items
evaluate(baseline_predictor, test_data, size=200)
```

---

### Example 2: Evaluate Multiple Models

```python
from pricer.evaluator import Tester

def model_v1(item):
    # Simple heuristic
    return len(item.full) * 0.1

def model_v2(item):
    # Weight-based
    return item.weight * 20 if item.weight else 50

def model_v3(item):
    # Category-based
    category_avg = {"Electronics": 100, "Books": 20, "Clothing": 40}
    return category_avg.get(item.category, 50)

# Test all models
for predictor in [model_v1, model_v2, model_v3]:
    print(f"\n{'='*50}")
    print(f"Testing {predictor.__name__}")
    print('='*50)
    Tester(predictor, test_data, size=200).run()
```

---

### Example 3: Evaluate LLM-based Predictor

```python
from groq import Groq
import os

groq = Groq(api_key=os.environ.get("GROQ_API_KEY"))

def llm_predictor(item):
    """Uses LLM to predict price"""
    prompt = f"What is the price of this product to the nearest dollar?\n\n{item.full}\n\nPrice is $"
    
    response = groq.chat.completions.create(
        model="llama-3.1-8b-instant",
        messages=[{"role": "user", "content": prompt}],
        max_tokens=10
    )
    
    return response.choices[0].message.content

# Evaluate with fewer samples (LLM calls are slower)
evaluate(llm_predictor, test_data, size=50, workers=3)
```

---

### Example 4: Custom Tester with Analysis

```python
from pricer.evaluator import Tester

def my_predictor(item):
    # Your prediction logic
    return predicted_price

# Create tester
tester = Tester(my_predictor, test_data, title="My Custom Model", size=300, workers=8)

# Run evaluation
tester.run()

# Access results for further analysis
import numpy as np

print(f"\nDetailed Statistics:")
print(f"Mean Error: ${np.mean(tester.errors):.2f}")
print(f"Median Error: ${np.median(tester.errors):.2f}")
print(f"Std Dev: ${np.std(tester.errors):.2f}")
print(f"Min Error: ${np.min(tester.errors):.2f}")
print(f"Max Error: ${np.max(tester.errors):.2f}")

# Analyze by color
green_count = tester.colors.count("green")
orange_count = tester.colors.count("orange")
red_count = tester.colors.count("red")

print(f"\nError Distribution:")
print(f"Green (good): {green_count} ({green_count/len(tester.colors)*100:.1f}%)")
print(f"Orange (ok): {orange_count} ({orange_count/len(tester.colors)*100:.1f}%)")
print(f"Red (poor): {red_count} ({red_count/len(tester.colors)*100:.1f}%)")
```

---

## Performance Optimization

### Worker Count Tuning

```python
# For CPU-bound predictors (complex calculations)
evaluate(cpu_intensive_predictor, data, workers=10)

# For I/O-bound predictors (API calls)
evaluate(api_predictor, data, workers=20)

# For very fast predictors
evaluate(simple_predictor, data, workers=1)  # Overhead not worth it
```

### Sample Size Selection

```python
# Quick test during development
evaluate(predictor, data, size=50)  # ~10 seconds

# Standard evaluation
evaluate(predictor, data, size=200)  # ~1 minute

# Comprehensive evaluation
evaluate(predictor, data, size=1000)  # ~5 minutes

# Full test set
evaluate(predictor, data, size=len(data))  # Variable
```

---

## Interpreting Results

### Good Model Indicators

✅ **Low Average Error**: < $20 for most product categories  
✅ **High R² Score**: > 80%  
✅ **Low MSE**: < 500  
✅ **Tight Confidence Interval**: ± < $5  
✅ **Mostly Green Points**: > 70% green in scatter plot  
✅ **Points Near Diagonal**: Clustered around y=x line  

### Poor Model Indicators

❌ **High Average Error**: > $50  
❌ **Low R² Score**: < 50%  
❌ **High MSE**: > 2000  
❌ **Wide Confidence Interval**: ± > $20  
❌ **Mostly Red Points**: > 30% red in scatter plot  
❌ **Scattered Points**: No clear pattern around y=x  

---

## Metrics Explained

### Average Absolute Error (AAE)

**Formula**: `mean(|predicted - actual|)`

**Interpretation**:
- Direct measure of typical prediction error
- Same units as price (dollars)
- Easy to understand and communicate

**Example**:
```
AAE = $15.23
→ On average, predictions are off by $15.23
```

---

### Mean Squared Error (MSE)

**Formula**: `mean((predicted - actual)²)`

**Interpretation**:
- Penalizes large errors more heavily
- Units are squared (dollars²)
- Useful for optimization, less intuitive

**Example**:
```
MSE = 450
→ Average squared error is 450
→ RMSE = √450 ≈ $21.21 (more interpretable)
```

---

### R² Score (Coefficient of Determination)

**Formula**: `1 - (SS_residual / SS_total)`

**Interpretation**:
- Percentage of variance explained by model
- Range: -∞ to 100%
- 100% = perfect predictions
- 0% = as good as predicting the mean
- Negative = worse than predicting the mean

**Example**:
```
R² = 85.3%
→ Model explains 85.3% of price variance
→ 14.7% unexplained (due to factors not in model)
```

---

## Visualization Guide

### Scatter Plot Analysis

**Perfect Model**:
```
  Predicted
     |
 100 |        ●
     |      ●
  50 |    ●
     |  ●
   0 |●____________
     0   50  100  Actual
```
All points on diagonal (y=x)

**Good Model**:
```
  Predicted
     |
 100 |      ●●
     |    ●●●●
  50 |  ●●●●●
     | ●●●
   0 |●____________
     0   50  100  Actual
```
Points clustered near diagonal

**Poor Model**:
```
  Predicted
     |    ●
 100 |  ●   ●
     |●       ●
  50 |  ●   ●
     |    ●
   0 |____________
     0   50  100  Actual
```
Points scattered randomly

---

### Error Trend Chart Analysis

**Converging (Good)**:
```
Error
  |
40|╲
  | ╲___________
20|
  |________________
     Samples
```
Error decreases and stabilizes

**Stable (Good)**:
```
Error
  |
40|
  |____________
20|
  |________________
     Samples
```
Consistent performance

**Diverging (Bad)**:
```
Error
  |            ╱
40|          ╱
  |        ╱
20|______╱
  |________________
     Samples
```
Error increasing (overfitting or data issues)

---

## Best Practices

### ✅ Do's

```python
# ✅ Test on held-out test set, never training data
evaluate(predictor, test_data)  # Good
evaluate(predictor, train_data)  # Bad - overfitting bias

# ✅ Use enough samples for statistical significance
evaluate(predictor, data, size=200)  # Good minimum

# ✅ Compare multiple models on same test set
for model in models:
    evaluate(model, test_data, size=200)

# ✅ Analyze error patterns, not just average
# Look at scatter plot for systematic biases
```

### ❌ Don'ts

```python
# ❌ Don't use tiny test sets
evaluate(predictor, data, size=10)  # Too small, unreliable

# ❌ Don't ignore confidence intervals
# Wide CI means results are unreliable

# ❌ Don't only look at R²
# Can be misleading for non-linear relationships

# ❌ Don't test on training data
evaluate(predictor, train_data)  # Biased results
```

---

## Troubleshooting

### Issue: All Predictions are 0

**Cause**: `post_process()` couldn't extract number from prediction

**Solution**:
```python
# Debug predictor output
item = test_data[0]
raw_output = predictor(item)
print(f"Raw output: {raw_output}")
print(f"Post-processed: {Tester.post_process(raw_output)}")
```

### Issue: Very High Errors

**Cause**: Predictor returning wrong format or units

**Solution**:
```python
# Check prediction scale
predictions = [predictor(item) for item in test_data[:10]]
print(f"Sample predictions: {predictions}")
# Are they in cents instead of dollars?
# Are they percentages?
```

### Issue: Slow Evaluation

**Cause**: Predictor is slow, not enough workers

**Solution**:
```python
# Increase workers for I/O-bound predictors
evaluate(predictor, data, workers=20)

# Reduce sample size for testing
evaluate(predictor, data, size=50)
```

---

## Dependencies

```bash
pip install scikit-learn pandas plotly tqdm
```

---

## Related Files

- **`items.py`**: Defines Item class used in predictions
- **`batch.py`**: May generate summaries used by predictors
- **Model files**: Various predictor implementations

---

## Summary

The `Tester` class provides:
- 🎯 **Comprehensive Evaluation**: Multiple metrics (AAE, MSE, R²)
- 📊 **Rich Visualizations**: Interactive Plotly charts
- ⚡ **Parallel Processing**: Fast evaluation with ThreadPoolExecutor
- 📈 **Statistical Analysis**: Confidence intervals and trend analysis
- 🎨 **Color-Coded Feedback**: Real-time quality indicators
- 🔧 **Flexible API**: Easy to use for any predictor function

Perfect for rigorously testing and comparing price prediction models!
