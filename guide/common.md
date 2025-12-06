# Common Libraries Documentation

This document covers commonly used libraries in the fine-tuning project for visualization and progress tracking.

---

## Matplotlib (pyplot)

```python
import matplotlib.pyplot as plt
```

### Overview

**Matplotlib** is Python's most popular plotting library for creating static, animated, and interactive visualizations. The `pyplot` module provides a MATLAB-like interface for creating plots quickly and easily.

### What is `plt`?

`plt` is the conventional alias for `matplotlib.pyplot`, providing a state-based interface for creating and customizing plots.

### Common Use Cases in ML Projects

#### 1. **Training Loss Visualization**

```python
import matplotlib.pyplot as plt

# Plot training and validation loss
epochs = [1, 2, 3, 4, 5]
train_loss = [0.5, 0.3, 0.2, 0.15, 0.1]
val_loss = [0.6, 0.35, 0.25, 0.2, 0.18]

plt.figure(figsize=(10, 6))
plt.plot(epochs, train_loss, label='Training Loss', marker='o')
plt.plot(epochs, val_loss, label='Validation Loss', marker='s')
plt.xlabel('Epoch')
plt.ylabel('Loss')
plt.title('Training vs Validation Loss')
plt.legend()
plt.grid(True)
plt.show()
```

#### 2. **Price Distribution Analysis**

```python
# Visualize price distribution in dataset
prices = [item.price for item in items]

plt.figure(figsize=(12, 5))

# Histogram
plt.subplot(1, 2, 1)
plt.hist(prices, bins=50, edgecolor='black', alpha=0.7)
plt.xlabel('Price ($)')
plt.ylabel('Frequency')
plt.title('Price Distribution')

# Box plot
plt.subplot(1, 2, 2)
plt.boxplot(prices)
plt.ylabel('Price ($)')
plt.title('Price Range Overview')

plt.tight_layout()
plt.show()
```

#### 3. **Category Comparison**

```python
# Compare average prices by category
categories = ['Electronics', 'Books', 'Clothing', 'Home']
avg_prices = [150.50, 25.30, 45.20, 80.10]

plt.figure(figsize=(10, 6))
plt.bar(categories, avg_prices, color=['#FF6B6B', '#4ECDC4', '#45B7D1', '#FFA07A'])
plt.xlabel('Category')
plt.ylabel('Average Price ($)')
plt.title('Average Price by Category')
plt.xticks(rotation=45)
plt.tight_layout()
plt.show()
```

#### 4. **Model Performance Metrics**

```python
# Scatter plot: Predicted vs Actual prices
actual = [29.99, 49.99, 15.50, 99.99]
predicted = [28.50, 52.00, 14.99, 95.00]

plt.figure(figsize=(8, 8))
plt.scatter(actual, predicted, alpha=0.6, s=100)
plt.plot([0, 100], [0, 100], 'r--', label='Perfect Prediction')
plt.xlabel('Actual Price ($)')
plt.ylabel('Predicted Price ($)')
plt.title('Model Predictions vs Actual Prices')
plt.legend()
plt.grid(True, alpha=0.3)
plt.show()
```

### Key Functions

| Function | Purpose | Example |
|----------|---------|---------|
| `plt.figure()` | Create a new figure | `plt.figure(figsize=(10, 6))` |
| `plt.plot()` | Line plot | `plt.plot(x, y, label='Data')` |
| `plt.scatter()` | Scatter plot | `plt.scatter(x, y, alpha=0.5)` |
| `plt.bar()` | Bar chart | `plt.bar(categories, values)` |
| `plt.hist()` | Histogram | `plt.hist(data, bins=50)` |
| `plt.xlabel()` | X-axis label | `plt.xlabel('Epoch')` |
| `plt.ylabel()` | Y-axis label | `plt.ylabel('Loss')` |
| `plt.title()` | Plot title | `plt.title('Training Loss')` |
| `plt.legend()` | Add legend | `plt.legend()` |
| `plt.grid()` | Add grid | `plt.grid(True)` |
| `plt.show()` | Display plot | `plt.show()` |
| `plt.savefig()` | Save to file | `plt.savefig('plot.png')` |

### Customization Options

```python
# Comprehensive styling example
plt.figure(figsize=(12, 6), dpi=100)
plt.plot(x, y, 
         color='#3498db',           # Custom color (hex)
         linewidth=2,               # Line thickness
         linestyle='--',            # Dashed line
         marker='o',                # Circle markers
         markersize=8,              # Marker size
         alpha=0.7,                 # Transparency
         label='My Data')           # Legend label

plt.xlabel('X Axis', fontsize=14, fontweight='bold')
plt.ylabel('Y Axis', fontsize=14, fontweight='bold')
plt.title('Custom Styled Plot', fontsize=16, pad=20)
plt.legend(loc='upper right', fontsize=12)
plt.grid(True, alpha=0.3, linestyle=':', linewidth=0.5)
plt.tight_layout()
plt.show()
```

### Saving Plots

```python
# Save high-quality plot
plt.figure(figsize=(10, 6))
plt.plot(x, y)
plt.title('My Plot')

# Save in different formats
plt.savefig('plot.png', dpi=300, bbox_inches='tight')  # PNG
plt.savefig('plot.pdf', bbox_inches='tight')           # PDF (vector)
plt.savefig('plot.svg', bbox_inches='tight')           # SVG (vector)
```

---

## tqdm (Progress Bars)

```python
from tqdm.notebook import tqdm
```

### Overview

**tqdm** (from Arabic "taqaddum" meaning "progress") is a fast, extensible progress bar library for Python. It provides visual feedback for long-running operations, making it easier to monitor loops and iterations.

### Why `tqdm.notebook`?

The `tqdm.notebook` module is specifically designed for **Jupyter Notebooks** and provides:
- ✅ Rich HTML-based progress bars
- ✅ Better visual integration with notebook cells
- ✅ Cleaner output without text clutter
- ✅ Interactive widgets

For regular Python scripts, use `from tqdm import tqdm` instead.

### Basic Usage

#### 1. **Simple Loop Progress**

```python
from tqdm.notebook import tqdm
import time

# Wrap any iterable with tqdm
for i in tqdm(range(100)):
    time.sleep(0.01)  # Simulate work
    
# Output: [████████████████████] 100/100 [00:01<00:00, 99.50it/s]
```

#### 2. **Processing Items with Description**

```python
items = [item1, item2, item3, ...]

for item in tqdm(items, desc="Processing items"):
    # Process each item
    process(item)
    
# Output: Processing items: [████████] 100/100 [00:05<00:00, 20.00it/s]
```

#### 3. **Dataset Parsing Example**

```python
from pricer.parser import parse

raw_data = load_raw_products()  # List of 10,000 products
parsed_items = []

for raw in tqdm(raw_data, desc="Parsing products"):
    item = parse(raw, "Electronics")
    if item:
        parsed_items.append(item)

print(f"Successfully parsed {len(parsed_items)} items")
```

#### 4. **Training Loop with tqdm**

```python
from tqdm.notebook import tqdm

epochs = 10
for epoch in tqdm(range(epochs), desc="Training"):
    train_loss = train_one_epoch(model, train_loader)
    val_loss = validate(model, val_loader)
    
    # Update description with current metrics
    tqdm.write(f"Epoch {epoch+1}: Train Loss={train_loss:.4f}, Val Loss={val_loss:.4f}")
```

#### 5. **Nested Progress Bars**

```python
from tqdm.notebook import tqdm

categories = ['Electronics', 'Books', 'Clothing']

for category in tqdm(categories, desc="Categories"):
    products = load_products(category)
    
    for product in tqdm(products, desc=f"Processing {category}", leave=False):
        process(product)
```

### Advanced Features

#### **Manual Progress Updates**

```python
from tqdm.notebook import tqdm

# Create progress bar with total
pbar = tqdm(total=1000, desc="Custom progress")

for i in range(10):
    # Do some work
    result = process_batch(batch_size=100)
    
    # Update progress manually
    pbar.update(100)
    
    # Update description
    pbar.set_description(f"Batch {i+1}/10")

pbar.close()
```

#### **Progress with File Operations**

```python
import requests
from tqdm.notebook import tqdm

url = "https://example.com/large_dataset.zip"
response = requests.get(url, stream=True)
total_size = int(response.headers.get('content-length', 0))

with open('dataset.zip', 'wb') as file:
    with tqdm(total=total_size, unit='B', unit_scale=True, desc='Downloading') as pbar:
        for chunk in response.iter_content(chunk_size=1024):
            file.write(chunk)
            pbar.update(len(chunk))
```

#### **Pandas Integration**

```python
from tqdm.notebook import tqdm
import pandas as pd

# Register tqdm with pandas
tqdm.pandas(desc="Processing rows")

# Use progress_apply instead of apply
df['processed'] = df['column'].progress_apply(lambda x: expensive_function(x))
```

### Key Parameters

| Parameter | Type | Description | Example |
|-----------|------|-------------|---------|
| `iterable` | iterable | The iterable to wrap | `range(100)` |
| `desc` | str | Description text | `desc="Training"` |
| `total` | int | Total iterations (if not inferrable) | `total=1000` |
| `leave` | bool | Keep bar after completion | `leave=False` |
| `unit` | str | Unit of iteration | `unit='items'` |
| `unit_scale` | bool | Auto-scale units (K, M, G) | `unit_scale=True` |
| `ncols` | int | Width of progress bar | `ncols=100` |
| `colour` | str | Bar color | `colour='green'` |
| `position` | int | Position for nested bars | `position=0` |

### Useful Methods

```python
pbar = tqdm(range(100))

# Update progress
pbar.update(1)          # Increment by 1
pbar.update(10)         # Increment by 10

# Set description
pbar.set_description("Processing batch 5")

# Set postfix (additional info)
pbar.set_postfix(loss=0.5, accuracy=0.95)

# Write without breaking progress bar
tqdm.write("Important message")

# Refresh display
pbar.refresh()

# Close progress bar
pbar.close()
```

### Real-World Example: Fine-Tuning Pipeline

```python
from tqdm.notebook import tqdm
import matplotlib.pyplot as plt

# Parse dataset
print("Step 1: Parsing raw data...")
items = []
for raw in tqdm(raw_data, desc="Parsing"):
    item = parse(raw, category)
    if item:
        items.append(item)

# Generate prompts
print("\nStep 2: Generating prompts...")
for item in tqdm(items, desc="Creating prompts"):
    item.make_prompt(item.full)

# Split dataset
train, val, test = split_data(items)

# Training loop
print("\nStep 3: Training model...")
train_losses = []
val_losses = []

for epoch in tqdm(range(10), desc="Epochs"):
    # Train
    epoch_loss = 0
    for batch in tqdm(train_loader, desc=f"Epoch {epoch+1}", leave=False):
        loss = train_step(batch)
        epoch_loss += loss
    
    train_losses.append(epoch_loss / len(train_loader))
    
    # Validate
    val_loss = validate(val_loader)
    val_losses.append(val_loss)
    
    tqdm.write(f"Epoch {epoch+1}: Train={train_losses[-1]:.4f}, Val={val_losses[-1]:.4f}")

# Visualize results
print("\nStep 4: Visualizing results...")
plt.figure(figsize=(10, 6))
plt.plot(train_losses, label='Training Loss', marker='o')
plt.plot(val_losses, label='Validation Loss', marker='s')
plt.xlabel('Epoch')
plt.ylabel('Loss')
plt.title('Training Progress')
plt.legend()
plt.grid(True, alpha=0.3)
plt.show()
```

---

## Installation

```bash
# Install matplotlib
pip install matplotlib

# Install tqdm
pip install tqdm

# Install both
pip install matplotlib tqdm
```

---

## Best Practices

### Matplotlib

✅ **Always use `plt.figure()` for new plots** to avoid overlapping  
✅ **Set figure size early** with `figsize=(width, height)`  
✅ **Use `plt.tight_layout()`** to prevent label cutoff  
✅ **Add labels and titles** for clarity  
✅ **Use `plt.savefig()` before `plt.show()`** (show clears the figure)  
✅ **Close figures** with `plt.close()` to free memory in loops  

### tqdm

✅ **Use descriptive `desc` parameter** for clarity  
✅ **Set `leave=False` for nested bars** to avoid clutter  
✅ **Use `tqdm.write()` instead of `print()`** to avoid breaking bars  
✅ **Close manual progress bars** with `pbar.close()`  
✅ **Use `tqdm.notebook` in Jupyter**, regular `tqdm` in scripts  

---

## Common Patterns

### Pattern 1: Data Processing with Visualization

```python
from tqdm.notebook import tqdm
import matplotlib.pyplot as plt

# Process data with progress tracking
results = []
for item in tqdm(data, desc="Processing"):
    result = process(item)
    results.append(result)

# Visualize results
plt.figure(figsize=(10, 6))
plt.hist(results, bins=50, edgecolor='black')
plt.xlabel('Result Value')
plt.ylabel('Frequency')
plt.title('Processing Results Distribution')
plt.show()
```

### Pattern 2: Training with Live Metrics

```python
from tqdm.notebook import tqdm
import matplotlib.pyplot as plt

losses = []
pbar = tqdm(range(epochs), desc="Training")

for epoch in pbar:
    loss = train_epoch()
    losses.append(loss)
    
    # Update progress bar with current loss
    pbar.set_postfix({'loss': f'{loss:.4f}'})
    
    # Plot every 10 epochs
    if (epoch + 1) % 10 == 0:
        plt.figure(figsize=(8, 4))
        plt.plot(losses)
        plt.title(f'Loss after {epoch+1} epochs')
        plt.xlabel('Epoch')
        plt.ylabel('Loss')
        plt.show()
```

---

## Troubleshooting

### Matplotlib Issues

**Problem**: Plots not showing  
**Solution**: Make sure to call `plt.show()` at the end

**Problem**: Overlapping labels  
**Solution**: Use `plt.tight_layout()` or adjust `figsize`

**Problem**: Memory issues with many plots  
**Solution**: Use `plt.close()` after each plot in loops

### tqdm Issues

**Problem**: Progress bar not updating  
**Solution**: Ensure you're iterating through the tqdm object, not the original iterable

**Problem**: Multiple bars overlapping  
**Solution**: Use `leave=False` for inner loops, `position` parameter for parallel bars

**Problem**: Output cluttered in Jupyter  
**Solution**: Use `from tqdm.notebook import tqdm` instead of `from tqdm import tqdm`

---

## Summary

- **Matplotlib (`plt`)**: Essential for visualizing training progress, data distributions, and model performance
- **tqdm**: Provides visual feedback for long-running operations, improving user experience and debugging

Both libraries are indispensable for machine learning projects, helping you understand your data and monitor training progress effectively.
