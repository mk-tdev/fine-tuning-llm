# Batch.py Documentation

## Overview

The `batch.py` file provides a system for processing large datasets using **Groq's Batch API**. It generates AI-powered product summaries by sending batches of items to Groq's language models, managing the asynchronous batch processing workflow, and applying the results back to the original items.

---

## Imports

```python
import os
from groq import Groq
from dotenv import load_dotenv
from pathlib import Path
import json
import pickle
from tqdm.notebook import tqdm
```

- **`os`**: Environment variable access
- **`groq.Groq`**: Groq API client for batch processing
- **`dotenv.load_dotenv`**: Load environment variables from `.env` file
- **`pathlib.Path`**: Modern file path handling
- **`json`**: JSONL file format handling
- **`pickle`**: Serialization for saving/loading batch state
- **`tqdm.notebook`**: Progress bars for Jupyter notebooks

---

## Configuration

### API Setup

```python
load_dotenv(override=True)
groq = Groq(api_key=os.environ.get("GROQ_API_KEY"))
```

- Loads `.env` file with `override=True` to refresh environment variables
- Initializes Groq client with API key from environment

### Constants

```python
MODEL = "openai/gpt-oss-20b"
BATCHES_FOLDER = "batches"
OUTPUT_FOLDER = "output"
state = Path("batches.pkl")
```

| Constant | Value | Purpose |
|----------|-------|---------|
| `MODEL` | `"openai/gpt-oss-20b"` | Groq model for generating summaries |
| `BATCHES_FOLDER` | `"batches"` | Directory for batch input files |
| `OUTPUT_FOLDER` | `"output"` | Directory for batch output files |
| `state` | `Path("batches.pkl")` | File for persisting batch state |

### System Prompt

```python
SYSTEM_PROMPT = """Create a concise description of a product. Respond only in this format. Do not include part numbers.
Title: Rewritten short precise title
Category: eg Electronics
Brand: Brand name
Description: 1 sentence description
Details: 1 sentence on features"""
```

This prompt instructs the AI to generate structured, concise product summaries in a specific format.

---

## Batch Class

The `Batch` class manages the complete lifecycle of a batch processing job.

### Class Attributes

```python
BATCH_SIZE = 1_000
batches = []
```

- **`BATCH_SIZE`**: Number of items per batch (1,000)
- **`batches`**: Class-level list storing all batch instances

---

### Constructor

```python
def __init__(self, items, start, end, lite):
    self.items = items
    self.start = start
    self.end = end
    self.filename = f"{start}_{end}.jsonl"
    self.file_id = None
    self.batch_id = None
    self.output_file_id = None
    self.done = False
    folder = Path("lite") if lite else Path("full")
    self.batches = folder / BATCHES_FOLDER
    self.output = folder / OUTPUT_FOLDER
    self.batches.mkdir(parents=True, exist_ok=True)
    self.output.mkdir(parents=True, exist_ok=True)
```

**Parameters:**
- `items` (list[Item]): Complete list of all items
- `start` (int): Start index for this batch
- `end` (int): End index for this batch (exclusive)
- `lite` (bool): If True, uses "lite" folder; otherwise "full"

**Instance Attributes:**

| Attribute | Type | Description |
|-----------|------|-------------|
| `items` | list[Item] | Reference to all items |
| `start` | int | Start index (inclusive) |
| `end` | int | End index (exclusive) |
| `filename` | str | JSONL filename (e.g., "0_1000.jsonl") |
| `file_id` | str\|None | Groq file ID after upload |
| `batch_id` | str\|None | Groq batch job ID |
| `output_file_id` | str\|None | Groq output file ID when complete |
| `done` | bool | Whether batch processing is complete |
| `batches` | Path | Directory for batch input files |
| `output` | Path | Directory for batch output files |

**Example:**
```python
items = [item1, item2, ..., item5000]
batch = Batch(items, start=0, end=1000, lite=False)
# Creates batch for items[0:1000]
# Filename: "0_1000.jsonl"
# Folders: "full/batches/" and "full/output/"
```

---

## Instance Methods

### `make_jsonl(self, item) -> str`

Creates a JSONL-formatted line for a single item.

**Parameters:**
- `item` (Item): Item to convert to JSONL

**Returns:**
- `str`: JSON string in Groq Batch API format

**Format:**
```json
{
  "custom_id": "123",
  "method": "POST",
  "url": "/v1/chat/completions",
  "body": {
    "model": "openai/gpt-oss-20b",
    "messages": [
      {"role": "system", "content": "Create a concise description..."},
      {"role": "user", "content": "Product full description..."}
    ],
    "reasoning_effort": "low"
  }
}
```

**Example:**
```python
item = Item(id=42, full="Wireless mouse with 6 buttons...")
jsonl_line = batch.make_jsonl(item)
# Returns: '{"custom_id": "42", "method": "POST", ...}'
```

---

### `make_file(self)`

Creates the JSONL batch file with all items in this batch's range.

**Behavior:**
1. Opens file at `batches/{start}_{end}.jsonl`
2. Iterates through `items[start:end]`
3. Writes JSONL line for each item
4. Adds newline after each entry

**Example:**
```python
batch = Batch(items, 0, 1000, lite=False)
batch.make_file()
# Creates: "full/batches/0_1000.jsonl" with 1000 lines
```

**File Contents:**
```jsonl
{"custom_id": "0", "method": "POST", ...}
{"custom_id": "1", "method": "POST", ...}
{"custom_id": "2", "method": "POST", ...}
...
```

---

### `send_file(self)`

Uploads the batch file to Groq and stores the file ID.

**Behavior:**
1. Opens the batch JSONL file in binary mode
2. Uploads to Groq with purpose "batch"
3. Stores returned file ID in `self.file_id`

**Example:**
```python
batch.send_file()
print(batch.file_id)  # "file-abc123xyz"
```

---

### `submit_batch(self)`

Submits the batch job to Groq for processing.

**Behavior:**
1. Creates batch job with 24-hour completion window
2. Specifies endpoint as `/v1/chat/completions`
3. Uses previously uploaded file ID
4. Stores batch job ID in `self.batch_id`

**Example:**
```python
batch.submit_batch()
print(batch.batch_id)  # "batch-def456uvw"
```

---

### `is_ready(self) -> bool`

Checks if the batch job has completed.

**Returns:**
- `bool`: True if completed, False otherwise

**Behavior:**
1. Retrieves batch status from Groq
2. If status is "completed", stores output file ID
3. Returns True only if completed

**Example:**
```python
if batch.is_ready():
    print("Batch complete!")
    print(f"Output file: {batch.output_file_id}")
else:
    print("Still processing...")
```

**Possible Statuses:**
- `"validating"`: Checking input file
- `"in_progress"`: Processing
- `"completed"`: Done
- `"failed"`: Error occurred
- `"cancelled"`: Manually cancelled

---

### `fetch_output(self)`

Downloads the completed batch output file from Groq.

**Behavior:**
1. Constructs output file path
2. Downloads content from Groq using `output_file_id`
3. Writes to local file in output directory

**Example:**
```python
batch.fetch_output()
# Downloads to: "full/output/0_1000.jsonl"
```

---

### `apply_output(self)`

Reads the output file and applies AI-generated summaries to items.

**Behavior:**
1. Opens the output JSONL file
2. For each line:
   - Parses JSON
   - Extracts custom_id (item ID)
   - Extracts AI-generated summary
   - Sets `items[id].summary = summary`
3. Marks batch as done

**Output Format:**
```json
{
  "custom_id": "42",
  "response": {
    "body": {
      "choices": [{
        "message": {
          "content": "Title: Wireless Mouse\nCategory: Electronics\n..."
        }
      }]
    }
  }
}
```

**Example:**
```python
batch.apply_output()
print(items[42].summary)
# Output: "Title: Wireless Mouse\nCategory: Electronics\nBrand: Logitech\n..."
```

---

## Class Methods

### `create(cls, items, lite)`

Creates batch objects for all items, dividing them into chunks.

**Parameters:**
- `items` (list[Item]): All items to process
- `lite` (bool): Whether to use "lite" mode

**Behavior:**
1. Divides items into chunks of `BATCH_SIZE` (1,000)
2. Creates a `Batch` object for each chunk
3. Appends to `cls.batches` class variable
4. Prints total batch count

**Example:**
```python
items = [item1, item2, ..., item5500]
Batch.create(items, lite=False)
# Output: Created 6 batches
# Batches: [0-1000, 1000-2000, 2000-3000, 3000-4000, 4000-5000, 5000-5500]
```

---

### `run(cls)`

Creates files, uploads them, and submits all batches.

**Behavior:**
1. For each batch:
   - Creates JSONL file (`make_file()`)
   - Uploads to Groq (`send_file()`)
   - Submits batch job (`submit_batch()`)
2. Shows progress with tqdm
3. Prints submission count

**Example:**
```python
Batch.run()
# Progress: 100%|██████████| 6/6 [00:45<00:00, 7.50s/batch]
# Output: Submitted 6 batches
```

**Timeline:**
```
Batch 1: make_file → send_file → submit_batch
Batch 2: make_file → send_file → submit_batch
...
Batch 6: make_file → send_file → submit_batch
```

---

### `fetch(cls)`

Checks batch status, downloads completed outputs, and applies results.

**Behavior:**
1. For each batch:
   - Skip if already done
   - Check if ready (`is_ready()`)
   - If ready: download output and apply to items
2. Shows progress with tqdm
3. Prints completion count

**Example:**
```python
# First call (batches still processing)
Batch.fetch()
# Output: Finished 2 of 6 batches

# Later call (more complete)
Batch.fetch()
# Output: Finished 5 of 6 batches

# Final call (all complete)
Batch.fetch()
# Output: Finished 6 of 6 batches
```

**Usage Pattern:**
```python
# Submit batches
Batch.run()

# Check periodically (every 5-10 minutes)
import time
while True:
    Batch.fetch()
    if all(b.done for b in Batch.batches):
        break
    time.sleep(300)  # Wait 5 minutes
```

---

### `save(cls)`

Saves batch state to disk for persistence across sessions.

**Behavior:**
1. Temporarily removes `items` reference from all batches (to reduce file size)
2. Pickles batch list to `batches.pkl`
3. Restores `items` reference
4. Prints save count

**Example:**
```python
Batch.save()
# Output: Saved 6 batches
# Creates: batches.pkl (contains batch metadata, not items)
```

**What's Saved:**
- Batch ranges (start, end)
- File IDs, batch IDs, output file IDs
- Completion status (`done`)
- Filenames and folder paths

**What's NOT Saved:**
- The items themselves (too large)

---

### `load(cls, items)`

Loads batch state from disk and reconnects to items.

**Parameters:**
- `items` (list[Item]): The items list to reconnect batches to

**Behavior:**
1. Loads pickled batch list from `batches.pkl`
2. Reconnects each batch to the provided items list
3. Prints load count

**Example:**
```python
# Session 1: Create and save
items = load_items()
Batch.create(items, lite=False)
Batch.run()
Batch.save()

# Session 2: Load and continue
items = load_items()  # Must load same items
Batch.load(items)
# Output: Loaded 6 batches

# Continue checking
Batch.fetch()
```

---

## Complete Workflow Example

### Full Pipeline

```python
from pricer.batch import Batch
from pricer.items import Item

# Step 1: Load your items
items = Item.from_hub("username/product-dataset")
train, val, test = items

# Step 2: Create batches (for training set)
Batch.create(train, lite=False)
# Output: Created 42 batches

# Step 3: Submit all batches
Batch.run()
# Output: Submitted 42 batches

# Step 4: Save state (optional, for persistence)
Batch.save()

# Step 5: Wait and check periodically
import time
while True:
    Batch.fetch()
    finished = sum(1 for b in Batch.batches if b.done)
    if finished == len(Batch.batches):
        print("All batches complete!")
        break
    print(f"Waiting... ({finished}/{len(Batch.batches)} done)")
    time.sleep(600)  # Check every 10 minutes

# Step 6: Items now have summaries
for item in train[:5]:
    print(f"{item.title}:")
    print(item.summary)
    print()
```

---

### Resume After Interruption

```python
# If your session was interrupted, resume like this:

# Reload items
items = Item.from_hub("username/product-dataset")
train, val, test = items

# Load saved batch state
Batch.load(train)
# Output: Loaded 42 batches

# Continue fetching
Batch.fetch()
# Output: Finished 35 of 42 batches

# Keep checking until done
```

---

## Folder Structure

```
project/
├── full/                    # Full mode
│   ├── batches/
│   │   ├── 0_1000.jsonl
│   │   ├── 1000_2000.jsonl
│   │   └── ...
│   └── output/
│       ├── 0_1000.jsonl
│       ├── 1000_2000.jsonl
│       └── ...
├── lite/                    # Lite mode
│   ├── batches/
│   └── output/
└── batches.pkl              # Saved state
```

---

## Lite vs Full Mode

```python
# Full mode: Uses complete product descriptions
Batch.create(items, lite=False)
# Folders: full/batches/, full/output/

# Lite mode: Uses shorter descriptions (if implemented)
Batch.create(items, lite=True)
# Folders: lite/batches/, lite/output/
```

**Use Cases:**
- **Full**: Better quality summaries, higher cost
- **Lite**: Faster processing, lower cost, good for testing

---

## Cost Estimation

```python
# Groq Batch API pricing (example, check current rates)
# Assume: $0.10 per 1M tokens

items_count = 50000
avg_tokens_per_item = 500  # Input + output
total_tokens = items_count * avg_tokens_per_item
cost = (total_tokens / 1_000_000) * 0.10

print(f"Estimated cost: ${cost:.2f}")
# Output: Estimated cost: $2.50
```

---

## Best Practices

### ✅ Do's

```python
# ✅ Save state after submitting
Batch.run()
Batch.save()

# ✅ Check status periodically, not continuously
time.sleep(600)  # 10 minutes between checks

# ✅ Handle partial completion gracefully
Batch.fetch()  # Can be called multiple times safely

# ✅ Use meaningful folder structure
# Separate lite/full modes for different experiments
```

### ❌ Don'ts

```python
# ❌ Don't call run() multiple times on same batches
Batch.run()
Batch.run()  # Will create duplicate batch jobs!

# ❌ Don't check status too frequently
while True:
    Batch.fetch()
    time.sleep(1)  # Too frequent! Wastes API calls

# ❌ Don't forget to save state for long-running jobs
Batch.run()
# Computer crashes... state lost!
```

---

## Troubleshooting

### Issue: Batches Not Completing

**Check status manually:**
```python
for i, batch in enumerate(Batch.batches):
    status = groq.batches.retrieve(batch.batch_id).status
    print(f"Batch {i}: {status}")
```

### Issue: Output File Not Found

**Ensure batch is actually complete:**
```python
batch = Batch.batches[0]
response = groq.batches.retrieve(batch.batch_id)
print(f"Status: {response.status}")
print(f"Output file ID: {response.output_file_id}")
```

### Issue: Items Not Updated

**Check if apply_output was called:**
```python
for batch in Batch.batches:
    if not batch.done:
        print(f"Batch {batch.filename} not applied yet")
```

---

## Dependencies

```bash
pip install groq python-dotenv tqdm
```

**Environment Variables (.env):**
```
GROQ_API_KEY=your_api_key_here
```

---

## Related Files

- **`items.py`**: Defines Item class with `summary` attribute
- **`loaders.py`**: Loads items that batches process
- **`.env`**: Contains GROQ_API_KEY

---

## Summary

The `Batch` class provides:
- 🚀 **Asynchronous Processing**: Submit thousands of items for AI processing
- 💾 **State Persistence**: Save and resume batch jobs across sessions
- 📊 **Progress Tracking**: Visual feedback with tqdm
- 🔄 **Automatic Application**: Results automatically applied to items
- 💰 **Cost Effective**: Batch API is cheaper than real-time API
- 🛡️ **Fault Tolerant**: Can check status and resume at any time

Perfect for generating AI summaries at scale for your price prediction dataset!
