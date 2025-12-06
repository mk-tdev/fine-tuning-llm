# Parser.py Documentation

## Overview

The `parser.py` file provides data parsing and cleaning utilities for converting raw product data into structured `Item` objects. It handles data validation, text normalization, weight conversion, and filtering to ensure high-quality training data for the price prediction model.

---

## Imports

```python
from pricer.items import Item
import json
import re
```

- **`pricer.items.Item`**: The Item model class for creating structured product data
- **`json`**: For parsing JSON-formatted product details
- **`re`**: Regular expressions for pattern matching and text cleaning

---

## Constants

### Data Quality Thresholds

```python
MIN_CHARS = 600
MIN_PRICE = 0.5
MAX_PRICE = 999.49
MAX_TEXT_EACH = 3000
MAX_TEXT_TOTAL = 4000
```

| Constant | Value | Purpose |
|----------|-------|---------|
| `MIN_CHARS` | 600 | Minimum character count for product description to be valid |
| `MIN_PRICE` | 0.5 | Minimum acceptable price ($0.50) |
| `MAX_PRICE` | 999.49 | Maximum acceptable price ($999.49) |
| `MAX_TEXT_EACH` | 3000 | Maximum characters per text field (description, features) |
| `MAX_TEXT_TOTAL` | 4000 | Maximum total characters for combined product text |

### Removal List

```python
REMOVALS = [
    "Part Number",
    "Best Sellers Rank",
    "Batteries Included?",
    "Batteries Required?",
    "Item model number",
]
```

Fields to remove from product details as they don't contribute to price prediction.

---

## Functions

### `simplify(text_list) -> str`

Cleans and normalizes text by removing excessive whitespace and limiting length.

**Parameters:**
- `text_list`: Input text (can be list or string)

**Returns:**
- `str`: Cleaned text, limited to `MAX_TEXT_EACH` characters

**Behavior:**
1. Converts input to string
2. Replaces newlines (`\n`) with spaces
3. Removes carriage returns (`\r`)
4. Removes tabs (`\t`)
5. Removes double spaces
6. Strips leading/trailing whitespace
7. Truncates to `MAX_TEXT_EACH` (3000) characters

**Example:**
```python
text = ["Feature 1\n", "Feature 2\t", "Feature 3"]
result = simplify(text)
# Result: "['Feature 1', 'Feature 2', 'Feature 3']" (cleaned, no extra whitespace)
```

---

### `scrub(title, description, features, details) -> str`

Creates a cleansed, comprehensive product description with unnecessary details removed.

**Parameters:**
- `title` (str): Product title
- `description` (str): Product description
- `features` (list/str): Product features
- `details` (dict): Product detail dictionary

**Returns:**
- `str`: Combined, cleaned product text limited to `MAX_TEXT_TOTAL` characters

**Behavior:**
1. **Removes unwanted fields**: Pops items from `REMOVALS` list out of details dict
2. **Builds combined text**:
   - Starts with title + newline
   - Adds simplified description (if exists)
   - Adds simplified features (if exists)
   - Adds JSON-dumped details (if exists)
3. **Removes product codes**: Uses regex to strip alphanumeric codes (7+ chars with both letters and numbers)
4. **Truncates**: Limits to `MAX_TEXT_TOTAL` (4000) characters

**Regex Pattern Explained:**
```python
pattern = r"\b(?=[A-Z0-9]{7,}\b)(?=.*[A-Z])(?=.*\d)[A-Z0-9]+\b"
```
- Matches words with 7+ uppercase letters/numbers
- Must contain at least one letter AND one digit
- Removes product codes like "B08XYZ1234" or "ASIN12345"

**Example:**
```python
title = "Wireless Mouse"
description = "Ergonomic design"
features = ["Bluetooth", "6 buttons"]
details = {"Color": "Black", "Part Number": "XYZ123"}

result = scrub(title, description, features, details)
# Result: "Wireless Mouse\nErgonomic design\n['Bluetooth', '6 buttons']\n{\"Color\": \"Black\"}"
# Note: "Part Number" removed, product codes stripped
```

---

### `get_weight(details) -> float`

Extracts and converts product weight to pounds.

**Parameters:**
- `details` (dict): Product details dictionary containing "Item Weight" field

**Returns:**
- `float`: Weight in pounds, or `0` if not found/parseable

**Supported Units:**
- **Pounds**: Returns as-is
- **Ounces**: Divides by 16
- **Grams**: Divides by 453.592
- **Milligrams**: Divides by 453,592
- **Kilograms**: Divides by 0.453592
- **Hundredths pounds**: Divides by 100

**Example:**
```python
details1 = {"Item Weight": "2.5 pounds"}
get_weight(details1)  # Returns: 2.5

details2 = {"Item Weight": "40 ounces"}
get_weight(details2)  # Returns: 2.5 (40/16)

details3 = {"Item Weight": "1000 grams"}
get_weight(details3)  # Returns: 2.205 (1000/453.592)

details4 = {"Color": "Red"}
get_weight(details4)  # Returns: 0 (no weight field)
```

---

### `parse(datapoint, category) -> Item | None`

Main parsing function that converts raw product data into an `Item` object with validation.

**Parameters:**
- `datapoint` (dict): Raw product data with keys: `price`, `title`, `description`, `features`, `details`
- `category` (str): Product category

**Returns:**
- `Item`: Valid Item object if all criteria met
- `None`: If validation fails (invalid price, insufficient text, etc.)

**Validation Steps:**

1. **Price Validation**:
   - Attempts to convert price to float
   - Returns `None` if ValueError occurs
   - Checks if `MIN_PRICE <= price <= MAX_PRICE` ($0.50 - $999.49)

2. **Data Extraction**:
   - Extracts title, description, features from datapoint
   - Parses JSON details string
   - Converts weight using `get_weight()`
   - Cleans text using `scrub()`

3. **Text Length Validation**:
   - Ensures cleaned text has at least `MIN_CHARS` (600) characters
   - Returns `None` if text too short

4. **Item Creation**:
   - Creates and returns Item object with all validated data

**Example:**
```python
raw_data = {
    "price": "29.99",
    "title": "Wireless Mouse",
    "description": "Ergonomic wireless mouse with 6 programmable buttons...",
    "features": ["Bluetooth 5.0", "2400 DPI", "Rechargeable battery"],
    "details": '{"Color": "Black", "Item Weight": "3.2 ounces"}'
}

item = parse(raw_data, "Electronics")
if item:
    print(item)  # <Wireless Mouse = $29.99>
else:
    print("Failed validation")
```

**Failure Cases:**
```python
# Price too high
parse({"price": "1500.00", ...}, "Electronics")  # Returns None

# Price invalid
parse({"price": "invalid", ...}, "Electronics")  # Returns None

# Text too short (< 600 chars)
parse({"price": "29.99", "title": "Mouse", "description": "", ...}, "Electronics")  # Returns None
```

---

## Data Flow Pipeline

```
Raw Product Data
       ↓
   parse() function
       ↓
   Price Validation (MIN_PRICE to MAX_PRICE)
       ↓
   Extract Fields (title, description, features, details)
       ↓
   get_weight() → Convert weight to pounds
       ↓
   scrub() → Clean and combine text
       ├─ Remove unwanted fields (REMOVALS)
       ├─ simplify() → Normalize whitespace
       ├─ Remove product codes (regex)
       └─ Truncate to MAX_TEXT_TOTAL
       ↓
   Text Length Validation (MIN_CHARS)
       ↓
   Create Item Object
       ↓
   Return Item or None
```

---

## Quality Control Features

### ✅ **Price Filtering**
- Excludes extremely cheap items (< $0.50)
- Excludes expensive items (> $999.49)
- Ensures consistent price range for model training

### ✅ **Text Quality**
- Minimum 600 characters ensures sufficient context
- Maximum limits prevent token overflow
- Whitespace normalization improves consistency

### ✅ **Data Cleaning**
- Removes irrelevant metadata (part numbers, rankings)
- Strips product codes that don't help price prediction
- Standardizes weight units to pounds

### ✅ **Error Handling**
- Gracefully handles invalid prices (returns None)
- Handles missing weight data (returns 0)
- Validates all data before creating Item

---

## Usage Example

### Processing a Dataset

```python
from pricer.parser import parse

# Raw product data from scraping/API
raw_products = [
    {
        "price": "29.99",
        "title": "Wireless Mouse",
        "description": "Ergonomic wireless mouse with 6 programmable buttons and adjustable DPI settings. Perfect for gaming and productivity.",
        "features": ["Bluetooth 5.0", "2400 DPI", "Rechargeable battery", "6 buttons"],
        "details": '{"Color": "Black", "Item Weight": "3.2 ounces", "Part Number": "WM-2024"}'
    },
    {
        "price": "invalid",  # Will be filtered out
        "title": "Keyboard",
        "description": "Mechanical keyboard",
        "features": [],
        "details": '{}'
    }
]

# Parse and filter valid items
items = []
for raw in raw_products:
    item = parse(raw, "Electronics")
    if item:
        items.append(item)

print(f"Successfully parsed {len(items)} items")  # Output: 1
```

---

## Design Rationale

### Why These Filters?

1. **Price Range ($0.50 - $999.49)**:
   - Focuses on typical consumer products
   - Excludes outliers that could skew model training
   - Avoids free items, accessories, and luxury goods

2. **Minimum 600 Characters**:
   - Ensures sufficient context for price prediction
   - Filters out low-quality or incomplete listings
   - Provides enough information for meaningful training

3. **Product Code Removal**:
   - ASINs, SKUs, and model numbers don't correlate with price
   - Reduces noise in training data
   - Prevents model from memorizing codes instead of learning patterns

4. **Weight Standardization**:
   - Weight is a strong price predictor
   - Converting to single unit (pounds) ensures consistency
   - Enables model to learn weight-price relationships

---

## Dependencies

```bash
pip install pydantic  # Required by Item class
```

---

## Related Files

- **`items.py`**: Defines the Item model that this parser creates
- **`main.py`**: Likely uses parse() to process datasets
- **Raw data source**: JSON/CSV files with product information

---

## Common Pitfalls

⚠️ **Invalid JSON in details field**: Ensure details is valid JSON string
⚠️ **Missing required fields**: All fields (price, title, description, features, details) must exist in datapoint
⚠️ **Short descriptions**: Products with < 600 chars total text will be filtered out
⚠️ **Price format**: Price must be convertible to float (e.g., "29.99" not "$29.99")
