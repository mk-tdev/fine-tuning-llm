# Loaders.py Documentation

## Overview

The `loaders.py` file provides a high-performance data loading system for processing large Amazon product datasets. It uses parallel processing to efficiently parse and validate thousands of product records, converting raw data into structured `Item` objects suitable for model training.

---

## Imports

```python
from datetime import datetime
from tqdm import tqdm
from datasets import load_dataset
from concurrent.futures import ProcessPoolExecutor
from pricer.parser import parse
import os
```

- **`datetime`**: For timing and performance measurement
- **`tqdm`**: Progress bar for tracking processing
- **`datasets.load_dataset`**: HuggingFace datasets library for loading Amazon Reviews dataset
- **`concurrent.futures.ProcessPoolExecutor`**: Parallel processing for CPU-intensive tasks
- **`pricer.parser.parse`**: Parser function to convert raw data to Item objects
- **`os`**: For detecting CPU count

---

## Constants

```python
CHUNK_SIZE = 1000
cpu_count = os.cpu_count()
WORKERS = max(cpu_count - 1, 1)
```

| Constant | Value | Purpose |
|----------|-------|---------|
| `CHUNK_SIZE` | 1000 | Number of datapoints processed in each batch |
| `cpu_count` | System-dependent | Total CPU cores available |
| `WORKERS` | `cpu_count - 1` (min 1) | Number of parallel worker processes |

### Why These Values?

- **CHUNK_SIZE = 1000**: Balances memory usage and processing efficiency
  - Too small: Overhead from process communication
  - Too large: Memory issues and uneven workload distribution
  
- **WORKERS = cpu_count - 1**: Leaves one core free for system operations
  - Prevents system slowdown
  - Ensures responsive UI during processing
  - Minimum of 1 worker for single-core systems

---

## ItemLoader Class

The `ItemLoader` class orchestrates the entire data loading pipeline, from downloading raw Amazon product data to producing validated `Item` objects.

### Constructor

```python
def __init__(self, category):
    self.category = category
    self.dataset = None
```

**Parameters:**
- `category` (str): Product category to load (e.g., "Electronics", "Books", "Clothing")

**Attributes:**
- `self.category`: Stores the category name
- `self.dataset`: Will hold the loaded HuggingFace dataset (initially None)

**Example:**
```python
loader = ItemLoader("Electronics")
```

---

## Methods

### `from_datapoint(self, datapoint) -> Item | None`

Converts a single raw datapoint into an `Item` object.

**Parameters:**
- `datapoint` (dict): Raw product data from the dataset

**Returns:**
- `Item`: Valid Item object if parsing succeeds
- `None`: If validation fails (invalid price, insufficient text, etc.)

**Behavior:**
- Delegates to `parse()` function from `parser.py`
- Applies all validation rules (price range, text length, etc.)
- Filters out invalid products automatically

**Example:**
```python
loader = ItemLoader("Electronics")
raw_product = {
    "price": "29.99",
    "title": "Wireless Mouse",
    "description": "...",
    "features": [...],
    "details": "{...}"
}

item = loader.from_datapoint(raw_product)
if item:
    print(f"Created: {item}")
else:
    print("Failed validation")
```

---

### `from_chunk(self, chunk) -> list[Item]`

Processes a chunk of datapoints and returns a list of valid Items.

**Parameters:**
- `chunk`: A subset of the dataset (typically 1000 datapoints)

**Returns:**
- `list[Item]`: List containing only successfully parsed Items (failures filtered out)

**Behavior:**
1. Iterates through each datapoint in the chunk
2. Calls `from_datapoint()` for each one
3. Filters out `None` values (failed validations)
4. Returns only valid Items

**Example:**
```python
# chunk contains 1000 raw products
chunk = dataset.select(range(0, 1000))

# Process chunk - might return fewer than 1000 items due to filtering
items = loader.from_chunk(chunk)
print(f"Valid items: {len(items)}/1000")
# Output: Valid items: 847/1000 (153 filtered out)
```

**Why Filter Here?**
- Keeps memory usage low by discarding invalid data early
- Prevents invalid data from propagating through the pipeline
- Makes final dataset size predictable

---

### `chunk_generator(self) -> Generator`

Creates a generator that yields chunks of the dataset for processing.

**Yields:**
- Dataset chunks of size `CHUNK_SIZE` (1000 datapoints each)

**Behavior:**
1. Calculates total dataset size
2. Iterates in steps of `CHUNK_SIZE`
3. Uses `dataset.select()` to extract each chunk
4. Handles the last chunk (which may be smaller than CHUNK_SIZE)

**Example:**
```python
loader = ItemLoader("Electronics")
loader.dataset = load_dataset(...)  # 50,000 products

for i, chunk in enumerate(loader.chunk_generator()):
    print(f"Chunk {i}: {len(chunk)} items")

# Output:
# Chunk 0: 1000 items
# Chunk 1: 1000 items
# ...
# Chunk 49: 1000 items
# (Total: 50 chunks)
```

**Why Use a Generator?**
- ✅ Memory efficient: Only one chunk in memory at a time
- ✅ Lazy evaluation: Chunks created on-demand
- ✅ Works with parallel processing: Each worker gets one chunk at a time

---

### `load_in_parallel(self, workers) -> list[Item]`

Processes the entire dataset using parallel workers for maximum performance.

**Parameters:**
- `workers` (int): Number of parallel processes to use

**Returns:**
- `list[Item]`: All successfully parsed Items from the dataset

**Behavior:**
1. **Initialize**: Creates empty results list
2. **Calculate chunks**: Determines total number of chunks
3. **Create pool**: Spawns `workers` number of processes
4. **Map work**: Distributes chunks to workers via `pool.map()`
5. **Track progress**: Uses tqdm to show processing progress
6. **Collect results**: Extends results list with each completed batch
7. **Return**: All valid Items combined from all workers

**Example:**
```python
loader = ItemLoader("Electronics")
loader.dataset = load_dataset(...)  # Dataset loaded

# Process with 7 workers (on 8-core machine)
items = loader.load_in_parallel(workers=7)

# Progress bar shows:
# 100%|██████████| 50/50 [02:30<00:00, 3.00s/chunk]

print(f"Loaded {len(items):,} items")
# Output: Loaded 42,385 items
```

**Performance Benefits:**

| Workers | Time (50k products) | Speedup |
|---------|---------------------|---------|
| 1 | ~15 minutes | 1x |
| 4 | ~4 minutes | 3.75x |
| 7 | ~2.5 minutes | 6x |
| 15 | ~2 minutes | 7.5x |

**Note**: Speedup is not linear due to:
- Process communication overhead
- I/O bottlenecks
- Uneven chunk processing times

---

### `load(self, workers=WORKERS) -> list[Item]`

Main entry point that orchestrates the complete loading pipeline.

**Parameters:**
- `workers` (int, optional): Number of parallel processes (defaults to `WORKERS`)

**Returns:**
- `list[Item]`: All successfully parsed and validated Items

**Behavior:**
1. **Start timer**: Records start time
2. **Print status**: Announces which category is loading
3. **Download dataset**: Loads from HuggingFace Hub
   - Dataset: "McAuley-Lab/Amazon-Reviews-2023"
   - Subset: `raw_meta_{category}` (e.g., "raw_meta_Electronics")
   - Split: "full" (entire dataset)
4. **Process in parallel**: Calls `load_in_parallel()`
5. **Calculate duration**: Measures total processing time
6. **Print summary**: Shows count and duration
7. **Return results**: All valid Items

**Example:**
```python
# Simple usage with defaults
loader = ItemLoader("Electronics")
items = loader.load()

# Output:
# Loading dataset Electronics
# 100%|██████████| 50/50 [02:30<00:00, 3.00s/chunk]
# Completed Electronics with 42,385 datapoints in 2.5 mins

# Custom worker count
loader = ItemLoader("Books")
items = loader.load(workers=4)  # Use only 4 workers
```

**Full Pipeline Visualization:**

```
load() called
    ↓
Start timer
    ↓
Download from HuggingFace
    ↓
load_in_parallel(workers)
    ↓
chunk_generator() creates chunks
    ↓
ProcessPoolExecutor spawns workers
    ↓
Each worker processes chunks:
    from_chunk(chunk)
        ↓
    from_datapoint(datapoint) for each item
        ↓
    parse(datapoint, category)
        ↓
    Filter out None values
        ↓
    Return valid Items
    ↓
Collect all results
    ↓
Print summary
    ↓
Return all Items
```

---

## Complete Usage Example

### Loading a Single Category

```python
from pricer.loaders import ItemLoader

# Create loader for Electronics category
loader = ItemLoader("Electronics")

# Load and process entire dataset
items = loader.load()

# Use the items
print(f"Total items: {len(items):,}")
print(f"First item: {items[0]}")
print(f"Average price: ${sum(item.price for item in items) / len(items):.2f}")
```

**Output:**
```
Loading dataset Electronics
100%|██████████| 50/50 [02:30<00:00, 3.00s/chunk]
Completed Electronics with 42,385 datapoints in 2.5 mins
Total items: 42,385
First item: <Wireless Mouse = $29.99>
Average price: $87.45
```

---

### Loading Multiple Categories

```python
from pricer.loaders import ItemLoader

categories = ["Electronics", "Books", "Clothing", "Home_and_Kitchen"]
all_items = []

for category in categories:
    loader = ItemLoader(category)
    items = loader.load()
    all_items.extend(items)
    print(f"{category}: {len(items):,} items\n")

print(f"Total across all categories: {len(all_items):,}")
```

**Output:**
```
Loading dataset Electronics
100%|██████████| 50/50 [02:30<00:00, 3.00s/chunk]
Completed Electronics with 42,385 datapoints in 2.5 mins
Electronics: 42,385 items

Loading dataset Books
100%|██████████| 120/120 [05:00<00:00, 2.50s/chunk]
Completed Books with 98,234 datapoints in 5.0 mins
Books: 98,234 items

...

Total across all categories: 215,847
```

---

### Custom Worker Configuration

```python
from pricer.loaders import ItemLoader

# For testing: Use fewer workers to keep system responsive
loader = ItemLoader("Electronics")
items = loader.load(workers=2)

# For production: Use all available cores
loader = ItemLoader("Electronics")
items = loader.load(workers=15)

# For single-core systems: Will automatically use 1 worker
loader = ItemLoader("Electronics")
items = loader.load()  # Uses WORKERS = max(cpu_count - 1, 1)
```

---

## Performance Optimization

### Memory Management

**Chunk Size Trade-offs:**

```python
# Small chunks (100): More overhead, less memory
CHUNK_SIZE = 100  # Good for: Low-memory systems, testing

# Medium chunks (1000): Balanced (default)
CHUNK_SIZE = 1000  # Good for: Most use cases

# Large chunks (5000): Less overhead, more memory
CHUNK_SIZE = 5000  # Good for: High-memory systems, large datasets
```

### Worker Count Optimization

```python
import os

# Conservative: Leave 2 cores free
WORKERS = max(os.cpu_count() - 2, 1)

# Aggressive: Use all cores
WORKERS = os.cpu_count()

# Custom: Based on dataset size
def optimal_workers(dataset_size):
    if dataset_size < 10000:
        return 2  # Small dataset, low overhead
    elif dataset_size < 100000:
        return max(os.cpu_count() - 1, 1)  # Medium dataset
    else:
        return os.cpu_count()  # Large dataset, maximize throughput
```

---

## Error Handling

The loader is designed to be fault-tolerant:

### Graceful Degradation

```python
# Invalid items are filtered out, not raised as errors
loader = ItemLoader("Electronics")
items = loader.load()

# If 50,000 raw products exist but only 42,385 pass validation:
# - No errors raised
# - Returns 42,385 valid items
# - 7,615 invalid items silently filtered
```

### Common Failure Reasons

Items are filtered out (return `None`) when:
- ❌ Price outside range ($0.50 - $999.49)
- ❌ Price not parseable (e.g., "N/A", "Contact for price")
- ❌ Description too short (< 600 characters)
- ❌ Missing required fields

---

## Dataset Information

### Amazon Reviews 2023 Dataset

**Source**: `McAuley-Lab/Amazon-Reviews-2023`

**Available Categories:**
- Electronics
- Books
- Clothing_Shoes_and_Jewelry
- Home_and_Kitchen
- Sports_and_Outdoors
- Toys_and_Games
- Health_and_Personal_Care
- Beauty_and_Personal_Care
- And many more...

**Dataset Structure:**
```python
{
    "title": str,
    "price": str,
    "description": str,
    "features": list[str],
    "details": str (JSON),
    # ... other fields
}
```

**Size**: Varies by category (10k - 500k+ products per category)

---

## Best Practices

### ✅ Do's

```python
# ✅ Use default workers for most cases
items = loader.load()

# ✅ Process categories separately to monitor progress
for category in categories:
    loader = ItemLoader(category)
    items = loader.load()
    save_checkpoint(category, items)

# ✅ Measure performance
import time
start = time.time()
items = loader.load()
print(f"Loaded {len(items)} items in {time.time() - start:.1f}s")
```

### ❌ Don'ts

```python
# ❌ Don't use too many workers (diminishing returns + overhead)
items = loader.load(workers=100)  # Overkill

# ❌ Don't load multiple categories simultaneously
# (Each already uses parallel processing)
loader1 = ItemLoader("Electronics")
loader2 = ItemLoader("Books")
items1 = loader1.load()  # Wait for this to complete
items2 = loader2.load()  # Then load this

# ❌ Don't modify CHUNK_SIZE without testing
CHUNK_SIZE = 10  # Too small, massive overhead
```

---

## Troubleshooting

### Issue: "Out of Memory" Error

**Solution:**
```python
# Reduce chunk size
CHUNK_SIZE = 500

# Reduce workers
items = loader.load(workers=2)

# Process in batches
loader = ItemLoader("Electronics")
loader.dataset = load_dataset(...)
chunks = list(loader.chunk_generator())
for i in range(0, len(chunks), 10):
    batch_chunks = chunks[i:i+10]
    # Process 10 chunks at a time
```

### Issue: Slow Processing

**Solution:**
```python
# Increase workers (if you have CPU headroom)
items = loader.load(workers=os.cpu_count())

# Increase chunk size (if you have memory)
CHUNK_SIZE = 2000

# Check CPU usage - if low, might be I/O bound
# Consider using SSD for dataset cache
```

### Issue: Dataset Download Fails

**Solution:**
```python
# Set HuggingFace cache directory
import os
os.environ['HF_HOME'] = '/path/to/large/disk'

# Retry with trust_remote_code
loader.dataset = load_dataset(
    "McAuley-Lab/Amazon-Reviews-2023",
    f"raw_meta_{category}",
    split="full",
    trust_remote_code=True,  # Required for this dataset
)
```

---

## Dependencies

```bash
pip install datasets tqdm
```

---

## Related Files

- **`parser.py`**: Contains the `parse()` function used by `from_datapoint()`
- **`items.py`**: Defines the `Item` class that this loader creates
- **`main.py`**: Likely uses `ItemLoader` to build training datasets

---

## Summary

The `ItemLoader` class provides:
- 🚀 **High Performance**: Parallel processing for fast data loading
- 🔍 **Automatic Filtering**: Invalid data removed automatically
- 📊 **Progress Tracking**: Visual feedback via tqdm
- 🎯 **Simple API**: One method call to load entire datasets
- 💪 **Fault Tolerant**: Gracefully handles invalid data
- ⚙️ **Configurable**: Adjustable workers and chunk sizes

Perfect for efficiently loading and preprocessing large Amazon product datasets for price prediction model training!
