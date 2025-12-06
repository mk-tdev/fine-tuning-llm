# Items.py Documentation

## Overview

The `items.py` file defines a data model for handling product pricing data, specifically designed for fine-tuning a language model on price prediction tasks. It provides a structured way to create, manage, and share datasets for training LLMs to predict product prices.

---

## Imports

```python
from pydantic import BaseModel
from datasets import Dataset, DatasetDict, load_dataset
from typing import Optional, Self
```

- **`pydantic.BaseModel`**: Provides data validation and settings management using Python type annotations
- **`datasets`**: HuggingFace's library for working with datasets (Dataset, DatasetDict, load_dataset)
- **`typing`**: Type hints for Optional and Self (Self is for type hinting the class itself)

---

## Constants

```python
PREFIX = "Price is $"
QUESTION = "What does this cost to the nearest dollar?"
```

- **`PREFIX`**: The text prefix that appears before the price in the training prompt
- **`QUESTION`**: The question posed to the model during training and inference

---

## Item Class

The `Item` class is a Pydantic model representing a single product item with pricing information.

### Attributes

| Attribute | Type | Required | Description |
|-----------|------|----------|-------------|
| `title` | `str` | ✅ Yes | Product name/title |
| `category` | `str` | ✅ Yes | Product category |
| `price` | `float` | ✅ Yes | The actual price of the product |
| `full` | `Optional[str]` | ❌ No | Full product description |
| `weight` | `Optional[float]` | ❌ No | Product weight |
| `summary` | `Optional[str]` | ❌ No | Summarized product description |
| `prompt` | `Optional[str]` | ❌ No | The formatted training prompt (generated) |
| `id` | `Optional[int]` | ❌ No | Unique identifier |

---

## Methods

### `make_prompt(self, text: str)`

Creates a training prompt by combining the question, product description, and answer.

**Parameters:**
- `text` (str): The product description text to include in the prompt

**Behavior:**
- Combines: Question + Text + Prefix + Rounded Price
- Stores the result in `self.prompt`
- Rounds price to nearest dollar and formats as "XX.00"

**Example:**
```python
item = Item(title="Laptop", category="Electronics", price=1299.99)
item.make_prompt("High-performance gaming laptop with RTX 4080")
# Result: "What does this cost to the nearest dollar?\n\nHigh-performance gaming laptop with RTX 4080\n\nPrice is $1300.00"
```

---

### `test_prompt(self) -> str`

Returns the prompt WITHOUT the answer portion, used for inference/testing.

**Returns:**
- `str`: The prompt with question and text, ending with "Price is $" (no answer)

**Behavior:**
- Splits the full prompt on `PREFIX`
- Returns everything before the price + the prefix
- Used when you want the model to predict the price

**Example:**
```python
item.test_prompt()
# Result: "What does this cost to the nearest dollar?\n\nHigh-performance gaming laptop with RTX 4080\n\nPrice is $"
```

---

### `__repr__(self) -> str`

Custom string representation for debugging and logging.

**Returns:**
- `str`: Format: `<Product Title = $Price>`

**Example:**
```python
print(item)
# Output: <Laptop = $1299.99>
```

---

### `push_to_hub(dataset_name: str, train: list[Self], val: list[Self], test: list[Self])` (Static Method)

Uploads datasets to HuggingFace Hub.

**Parameters:**
- `dataset_name` (str): Name of the dataset on HuggingFace Hub
- `train` (list[Item]): Training set items
- `val` (list[Item]): Validation set items
- `test` (list[Item]): Test set items

**Behavior:**
- Converts Item objects to dictionaries using `model_dump()`
- Creates a DatasetDict with three splits: "train", "validation", "test"
- Pushes to HuggingFace Hub with the specified dataset name

**Example:**
```python
train_items = [item1, item2, item3]
val_items = [item4, item5]
test_items = [item6, item7]

Item.push_to_hub("username/product-prices", train_items, val_items, test_items)
```

---

### `from_hub(cls, dataset_name: str) -> tuple[list[Self], list[Self], list[Self]]` (Class Method)

Downloads datasets from HuggingFace Hub and reconstructs Item objects.

**Parameters:**
- `dataset_name` (str): Name of the dataset on HuggingFace Hub

**Returns:**
- `tuple`: (train_items, validation_items, test_items)

**Behavior:**
- Downloads dataset from HuggingFace Hub
- Reconstructs Item objects from raw data using `model_validate()`
- Returns three separate lists for train, validation, and test splits

**Example:**
```python
train, val, test = Item.from_hub("username/product-prices")
print(f"Loaded {len(train)} training items")
```

---

## Typical Workflow

### 1. **Data Collection**
Create Item objects with product information and prices:

```python
item = Item(
    title="Wireless Mouse",
    category="Electronics",
    price=29.99,
    summary="Ergonomic wireless mouse with 6 buttons"
)
```

### 2. **Prompt Generation**
Generate training prompts for each item:

```python
item.make_prompt(item.summary)
```

### 3. **Dataset Upload**
Save your dataset to HuggingFace Hub:

```python
Item.push_to_hub("my-username/price-dataset", train_items, val_items, test_items)
```

### 4. **Model Training**
Train an LLM on these prompts to predict prices based on product descriptions.

### 5. **Inference**
Use test prompts for model predictions:

```python
test_input = item.test_prompt()
# Feed to model, which should complete: "Price is $30.00"
```

---

## Purpose & Use Case

This file is designed for a **price prediction fine-tuning project**. The structure ensures:

- ✅ **Consistency**: Standardized format for all pricing data
- ✅ **Validation**: Pydantic ensures data integrity
- ✅ **Portability**: Easy sharing via HuggingFace Hub
- ✅ **Flexibility**: Optional fields for different data sources
- ✅ **Testing**: Separate methods for training and inference prompts

The Item class acts as a bridge between raw product data and the formatted prompts needed to fine-tune an LLM for price prediction tasks.

---

## Dependencies

Ensure you have the following packages installed:

```bash
pip install pydantic datasets
```

---

## Related Files

- **`main.py`**: Likely contains the main training/inference logic
- **`.env`**: Environment variables (possibly HuggingFace tokens)
- **`pricer/`**: Package containing the Item model and related utilities
