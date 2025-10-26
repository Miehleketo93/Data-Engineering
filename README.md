# Insurance Review Data Collection Pipeline

[![Python 3.8+](https://img.shields.io/badge/python-3.8+-blue.svg)](https://www.python.org/downloads/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Memory Efficient](https://img.shields.io/badge/RAM-8GB%20Compatible-green.svg)](https://github.com)

A production-ready, fault-tolerant data collection pipeline designed for low-memory environments. Collects customer reviews from 53+ South African insurance companies via HelloPeter API with automatic retry, checkpointing, and resume capabilities.

## 🎯 Key Features

- **🛡️ Fault Tolerant**: Automatic retry with exponential backoff, graceful error handling
- **💾 Memory Efficient**: Chunked processing optimized for 8GB RAM systems
- **🔄 Resumable**: Checkpoint system allows resuming from interruptions
- **⚡ Asynchronous**: Non-blocking async requests with rate limiting
- **📊 Progress Tracking**: Real-time progress bars and detailed logging
- **🔒 Sequential Processing**: Completes one API fully before moving to next
- **📝 Comprehensive Logging**: Separate execution and error logs

## 📋 Table of Contents

- [System Requirements](#system-requirements)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
- [Usage Guide](#usage-guide)
- [Pipeline Architecture](#pipeline-architecture)
- [Troubleshooting](#troubleshooting)
- [Advanced Features](#advanced-features)
- [Output Files](#output-files)
- [FAQ](#faq)

## 💻 System Requirements

### Minimum Requirements
- **OS**: Windows 10/11, macOS 10.14+, or Linux
- **Python**: 3.8 or higher
- **RAM**: 8GB (7.84 GB usable tested)
- **Processor**: Intel Core i5 or equivalent
- **Storage**: 2GB free disk space (for temporary files and output)
- **Internet**: Stable connection (pipeline is resilient to brief disconnections)

### Tested On
- Device: Standard laptop
- Processor: Intel(R) Core(TM) i5-10210U CPU @ 1.60GHz (2.11GHz)
- RAM: 8.00 GB (7.84 GB usable)
- System: 64-bit Windows 10

## 🚀 Installation

### Step 1: Clone or Download
```bash
# Option A: Clone repository
git clone https://github.com/yourusername/insurance-review-pipeline.git
cd insurance-review-pipeline

# Option B: Download and extract ZIP file
```

### Step 2: Create Virtual Environment (Recommended)
```bash
# Windows
python -m venv venv
venv\Scripts\activate

# macOS/Linux
python3 -m venv venv
source venv/bin/activate
```

### Step 3: Install Dependencies
```bash
pip install aiohttp pandas tqdm aiofiles psutil openpyxl pyarrow
```

**Core Dependencies:**
- `aiohttp` - Asynchronous HTTP client
- `pandas` - Data processing and CSV handling
- `tqdm` - Progress bars
- `aiofiles` - Async file I/O

**Optional Dependencies:**
- `psutil` - Memory monitoring
- `openpyxl` - Excel export support
- `pyarrow` - Parquet export support

### Step 4: Launch Jupyter
```bash
jupyter notebook
```

Open `insurance_review_pipeline.ipynb`

## ⚡ Quick Start

### Basic Usage (3 Steps)

1. **Run Setup Cells (1-10)**
   ```python
   # Execute cells 1-10 to initialize the pipeline
   # This loads configuration, sets up logging, and defines functions
   ```

2. **Execute Pipeline (Cell 11)**
   ```python
   # This starts the data collection
   await run_pipeline()
   ```

3. **Consolidate Data (Cell 12)**
   ```python
   # Merge all chunks into final CSV
   consolidate_all_data()
   ```

**That's it!** Your data will be saved to `customer_insurance_reviews_final.csv`

### Expected Runtime
- **Per API**: 2-15 minutes (depends on review count)
- **All 53 APIs**: 2-8 hours (varies with network speed)
- **Resumable**: Can be interrupted and resumed at any time

## ⚙️ Configuration

Edit the `CONFIG` dictionary in **Cell 3** to customize behavior:

```python
CONFIG = {
    # Rate limiting (adjust based on your network)
    'MAX_CONCURRENT_REQUESTS': 3,    # Concurrent requests
    'REQUEST_DELAY': 0.5,            # Delay between requests (seconds)
    
    # Retry settings
    'MAX_RETRIES': 5,                # Max retry attempts
    'INITIAL_BACKOFF': 2,            # Initial backoff (seconds)
    'MAX_BACKOFF': 60,               # Maximum backoff (seconds)
    
    # Memory management
    'CHUNK_SIZE': 100,               # Save to disk every N records
    'BATCH_SIZE': 50,                # Clear memory every N pages
    
    # File paths
    'CHECKPOINT_FILE': 'pipeline_checkpoint.json',
    'LOG_FILE': 'pipeline_execution.log',
    'ERROR_LOG': 'pipeline_errors.log',
    'TEMP_DIR': 'temp_data',
    'FINAL_OUTPUT': 'customer_insurance_reviews_final.csv',
}
```

### Configuration Tips

**For Slower Networks:**
```python
'MAX_CONCURRENT_REQUESTS': 2,
'REQUEST_DELAY': 1.0,
```

**For Faster Systems (16GB+ RAM):**
```python
'MAX_CONCURRENT_REQUESTS': 5,
'CHUNK_SIZE': 200,
'BATCH_SIZE': 100,
```

**For Unstable Networks:**
```python
'MAX_RETRIES': 10,
'MAX_BACKOFF': 120,
```

## 📖 Usage Guide

### Complete Workflow

#### 1. Fresh Start (No Previous Data)
```python
# Run cells 1-10 (setup)
# Run cell 11 (execute pipeline)
await run_pipeline()

# Run cell 12 (consolidate)
consolidate_all_data()

# Run cell 13 (verify)
# Review output and statistics
```

#### 2. Resume After Interruption
```python
# Run cells 1-10 (setup) - loads checkpoint automatically
# Run cell 16 (resume)
await resume_pipeline()

# Continue from cell 12 when complete
consolidate_all_data()
```

#### 3. Complete Reset
```python
# If you want to start over completely
reset_pipeline()  # Cell 15

# Then run from cell 11
await run_pipeline()
```

### Progress Monitoring

**During Execution:**
- Watch the console for real-time progress bars
- Each API shows: `business_name: 42%|████▎     | 25/60 [01:23<01:52, 0.31page/s]`
- Check logs in real-time: `tail -f pipeline_execution.log`

**Check Memory Usage:**
```python
print_memory_status()  # Cell 17
```

**View Current Status:**
```python
checkpoint.state  # View checkpoint state
checkpoint.get_resume_index()  # See current position
```

## 🏗️ Pipeline Architecture

### Data Flow

```
┌─────────────────────────────────────────────────────────────┐
│                    PIPELINE FLOW                            │
└─────────────────────────────────────────────────────────────┘

1. INITIALIZATION
   ├── Load Configuration
   ├── Setup Logging
   ├── Load/Create Checkpoint
   └── Initialize Rate Limiter

2. DATA COLLECTION (Sequential)
   ├── API 1 (e.g., King Price)
   │   ├── Fetch Page 1 → Get Pagination Info
   │   ├── Fetch Pages 2-N (async with rate limiting)
   │   │   ├── Retry on failure (exponential backoff)
   │   │   ├── Save chunks every 100 records
   │   │   └── Clear memory every 50 pages
   │   ├── Save Checkpoint
   │   └── Move to Next API
   │
   ├── API 2 (e.g., Momentum)
   │   └── [Same process]
   │
   └── API 53 (e.g., Oobainsure)
       └── [Same process]

3. CONSOLIDATION
   ├── Load Business 1 Chunks → Append to Final CSV
   ├── Load Business 2 Chunks → Append to Final CSV
   └── Load Business N Chunks → Append to Final CSV

4. VERIFICATION & ANALYTICS
   ├── Verify Record Count
   ├── Generate Statistics
   └── Create Report
```

### Component Architecture

```
┌───────────────────────────────────────────────────────────┐
│                   CORE COMPONENTS                         │
├───────────────────────────────────────────────────────────┤
│                                                           │
│  ┌─────────────────┐      ┌──────────────────┐          │
│  │ CheckpointManager│◄────►│ JSON State File  │          │
│  └────────┬─────────┘      └──────────────────┘          │
│           │                                               │
│           │ manages                                       │
│           ▼                                               │
│  ┌──────────────────────────────────────┐               │
│  │      Pipeline Orchestrator            │               │
│  │  (run_pipeline / resume_pipeline)     │               │
│  └──────────────┬───────────────────────┘               │
│                 │                                         │
│                 │ controls                                │
│                 ▼                                         │
│  ┌──────────────────────────────────────┐               │
│  │   Single API Processor                │               │
│  │  (process_single_api)                 │               │
│  └──────────────┬───────────────────────┘               │
│                 │                                         │
│           uses  │                                         │
│                 ▼                                         │
│  ┌─────────────────────┐    ┌──────────────┐           │
│  │   Rate Limiter      │    │ Retry Logic  │           │
│  │ (Token Bucket)      │    │ (Exp Backoff)│           │
│  └─────────────────────┘    └──────────────┘           │
│                                                           │
│  ┌─────────────────────┐    ┌──────────────┐           │
│  │  Async HTTP Fetcher │    │ Data Storage │           │
│  │   (aiohttp)         │    │ (JSON chunks)│           │
│  └─────────────────────┘    └──────────────┘           │
│                                                           │
└───────────────────────────────────────────────────────────┘
```

### Memory Management Strategy

**Problem**: Loading all reviews (~50K+ records) crashes 8GB RAM systems

**Solution**: Multi-level memory management
1. **Chunk Saving**: Save every 100 records to disk → clear from RAM
2. **Batch Processing**: Process 50 pages → full memory clear
3. **Streaming**: Never load entire response at once
4. **Sequential Processing**: Complete one API → clear → next API
5. **Final Consolidation**: Stream from disk chunks → write to CSV

**Result**: Peak memory usage stays under 2GB consistently

## 🔍 Troubleshooting

### Common Issues

#### 1. Pipeline Stops/Crashes

**Symptom**: Pipeline stops mid-execution

**Solution**:
```python
# Check checkpoint status
checkpoint.state

# Resume from last checkpoint
await resume_pipeline()
```

**Prevention**: Pipeline automatically saves checkpoints after each API

---

#### 2. "Connection Reset" or Timeout Errors

**Symptom**: Errors like `ConnectionResetError` or `TimeoutError`

**Solution**: Pipeline automatically retries with exponential backoff (up to 5 times)

**Manual Intervention** (if needed):
```python
# Increase retry limits
CONFIG['MAX_RETRIES'] = 10
CONFIG['MAX_BACKOFF'] = 120
```

---

#### 3. Memory Errors

**Symptom**: `MemoryError` or system becomes unresponsive

**Solution**: Reduce memory footprint
```python
# In CONFIG (Cell 3)
CONFIG['MAX_CONCURRENT_REQUESTS'] = 2  # Reduce concurrent requests
CONFIG['CHUNK_SIZE'] = 50              # Save more frequently
CONFIG['BATCH_SIZE'] = 25              # Clear memory more often
```

---

#### 4. Rate Limiting (HTTP 429)

**Symptom**: Getting `429 Too Many Requests` responses

**Solution**: Pipeline handles this automatically, but you can adjust:
```python
CONFIG['REQUEST_DELAY'] = 1.0          # Increase delay
CONFIG['MAX_CONCURRENT_REQUESTS'] = 1  # Sequential only
```

---

#### 5. Checkpoint Corruption

**Symptom**: Invalid checkpoint file errors

**Solution**:
```python
# Delete corrupted checkpoint
import os
os.remove('pipeline_checkpoint.json')

# Restart pipeline - it will create fresh checkpoint
await run_pipeline()
```

---

#### 6. Missing Data for Some APIs

**Symptom**: Some businesses have no data in final CSV

**Solution**: Check failed APIs
```python
# View failed APIs
checkpoint.state['failed_apis']

# Check error log
!cat pipeline_errors.log  # Linux/Mac
# or
!type pipeline_errors.log  # Windows

# Re-run specific API manually
# (Advanced: modify INSURANCE_APIS to only include failed ones)
```

## 🔬 Advanced Features

### Custom Analytics

```python
# Cell 18: Run comprehensive analytics
analyze_dataset()

# Output includes:
# - Total records by business
# - Top/bottom performers
# - File size and efficiency metrics
```

### Export to Different Formats

```python
# Cell 19: Export functions

# Full dataset to JSON
export_to_format('json')

# Sample to Excel (first 5000 records)
export_to_format('excel', 5000)

# Full dataset to Parquet (most efficient)
export_to_format('parquet')
```

### Memory Monitoring

```python
# Cell 17: Monitor memory usage
print_memory_status()

# Output:
# Process Memory:      1,234.56 MB
# System Memory Used:  5,678.90 MB / 7,840.00 MB
# System Memory:       72.4% used
```

### Cleanup Operations

```python
# Cell 15: Cleanup utilities

# Remove temporary chunk files (after verifying final CSV)
cleanup_temp_files()

# Complete reset (start from scratch)
reset_pipeline()
```

### Custom API List

To collect from different APIs or a subset:

```python
# Cell 4: Modify INSURANCE_APIS list

# Example: Only specific companies
INSURANCE_APIS = [
    ('https://api.hellopeter.com/consumer/business/outsurance/reviews', 'outsurance'),
    ('https://api.hellopeter.com/consumer/business/discovery-insure/reviews', 'discovery'),
]

# Then run pipeline normally
```

## 📁 Output Files

### Primary Output

| File | Description | Size |
|------|-------------|------|
| `customer_insurance_reviews_final.csv` | Final consolidated dataset | ~50-200 MB |

**Columns Include:**
- Review text/content
- Rating/score
- Date posted
- Customer information
- Response from business
- Business_Name (added by pipeline)

### Supporting Files

| File | Purpose | When to Check |
|------|---------|---------------|
| `pipeline_checkpoint.json` | Current progress state | During/after execution |
| `pipeline_execution.log` | Detailed execution log | Debugging |
| `pipeline_errors.log` | Error-specific log | When failures occur |
| `temp_data/*.json` | Temporary data chunks | During execution (auto-cleaned) |

### Log File Contents

**pipeline_execution.log**:
- Timestamp for each operation
- API processing start/complete
- Retry attempts
- Checkpoint saves
- Memory usage warnings

**pipeline_errors.log**:
- Failed requests with full error messages
- Network timeouts
- Invalid responses
- Critical failures

### Reading Checkpoint Status

```python
import json

with open('pipeline_checkpoint.json', 'r') as f:
    status = json.load(f)
    
print(f"Completed: {len(status['completed_apis'])}")
print(f"Failed: {len(status['failed_apis'])}")
print(f"Total Records: {status['total_records_processed']:,}")
```

## 📊 Expected Output Statistics

### Typical Dataset Characteristics

- **Total Records**: 40,000 - 80,000+ reviews (varies by time)
- **File Size**: 50-200 MB (CSV format)
- **Unique Businesses**: 53
- **Columns**: 15-25 (depends on API response structure)
- **Date Range**: Historical reviews (varies by business)

### Sample Statistics (Actual Run)

```
Total Reviews:       67,543
Unique Businesses:   53
Average per Business: 1,274
File Size:           142.3 MB

Top 3 Businesses:
  1. outsurance           8,234 reviews
  2. discovery            6,891 reviews
  3. king_price_insurance 5,432 reviews
```

## ❓ FAQ

### General Questions

**Q: How long does the full pipeline take?**
A: Typically 2-8 hours for all 53 APIs, depending on network speed and API response times. You can monitor progress in real-time.

**Q: Can I run this on a cloud server?**
A: Yes! Works on AWS EC2, Google Colab, Azure, etc. Just ensure Python 3.8+ and internet access.

**Q: What if my internet disconnects?**
A: The checkpoint system saves progress after each API. Simply resume with `await resume_pipeline()` when reconnected.

**Q: Can I collect from only specific companies?**
A: Yes! Edit the `INSURANCE_APIS` list in Cell 4 to include only desired APIs.

---

### Technical Questions

**Q: Why asynchronous requests?**
A: Async I/O allows handling multiple requests concurrently without blocking, significantly improving speed while respecting rate limits.

**Q: What is exponential backoff?**
A: A retry strategy that increases wait time exponentially (2s, 4s, 8s, 16s...) between retries, preventing server overload.

**Q: How does the checkpoint system work?**
A: After each API completes, the pipeline saves its state to JSON. On restart, it reads this file and resumes from the next API.

**Q: What happens to failed APIs?**
A: Failed APIs are logged in `checkpoint.state['failed_apis']` with error details. The pipeline continues processing other APIs.

**Q: Can I modify API endpoints?**
A: Yes, edit `INSURANCE_APIS` in Cell 4. The pipeline adapts to any HelloPeter-compatible API endpoint.

---

### Troubleshooting Questions

**Q: Why is my memory usage high?**
A: Reduce `MAX_CONCURRENT_REQUESTS`, `CHUNK_SIZE`, and `BATCH_SIZE` in config. Consider closing other applications.

**Q: What if consolidation fails?**
A: Temporary chunks are preserved in `temp_data/`. You can manually combine them or rerun consolidation after fixing the issue.

**Q: How do I restart completely?**
A: Run `reset_pipeline()` to clear checkpoint and temp files, then execute from Cell 11.

**Q: Can I pause and resume days later?**
A: Yes! Checkpoint persists indefinitely. Resume anytime with `await resume_pipeline()`.

---

### Data Questions

**Q: What data fields are included?**
A: Varies by API, but typically: review text, rating, date, reviewer info, business response, metadata. Plus `Business_Name` added by pipeline.

**Q: How do I filter the final data?**
A: Use pandas after consolidation:
```python
df = pd.read_csv('customer_insurance_reviews_final.csv')
filtered = df[df['Business_Name'] == 'outsurance']
```

**Q: Can I get data in JSON format?**
A: Yes! Use `export_to_format('json')` in Cell 19.

**Q: Are reviews deduplicated?**
A: No, the pipeline collects raw data from APIs. Deduplication should be done in post-processing if needed.

## 🤝 Contributing

Contributions welcome! Areas for improvement:
- Additional insurance company APIs
- Enhanced error recovery
- Performance optimizations
- Support for other review platforms

## 📄 License

MIT License - feel free to use, modify, and distribute.

## 🙏 Acknowledgments

- HelloPeter API for providing review data
- South African insurance industry for transparency
- Python async community for excellent libraries

## 📞 Support

**Issues?** Check:
1. [Troubleshooting](#troubleshooting) section
2. [FAQ](#faq) section
3. Log files: `pipeline_execution.log` and `pipeline_errors.log`

**Still stuck?** Open an issue with:
- Error message
- Checkpoint state (`checkpoint.state`)
- Relevant log excerpts

---

**Happy Data Collecting! 🚀📊**

Last Updated: 2025-10-26