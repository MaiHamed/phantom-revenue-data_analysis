#  Stockout & Phantom Revenue Analysis

A complete Python data analysis project that uncovers **lost revenue due to stockouts** using real e-commerce data. Perfect for a data analyst portfolio — covers cleaning, transformation, metric creation, and business visualization.

## 🎯 Project Objective
Supply chains lose money when popular sizes go out of stock. This project:
- Parses messy size/stock strings (e.g., `"UK 6, UK 8 - Out of stock"`)
- Calculates **stockout rate** and **phantom (lost) revenue**
- Aggregates by brand to find risk vs. reward opportunities
- Produces a **brand strategy scatter plot** to recommend action

## 📁 Data Source
Real dataset provided in the tutorial (see resources link in video). Contains:
- Product name, brand, price, size availability

## 🛠️ Tools & Libraries
- Python 3
- Pandas (data cleaning & aggregation)
- Matplotlib & Seaborn (visualization)
- Google Colab (or Jupyter)

## 📊 Key Steps

### 1. Data Cleaning
- Extract brand from product description using string methods
- Clean price column (remove `$`, convert to float)
- Filter relevant rows

### 2. Stockout Analysis Function
```python
def calculate_phantom_revenue(size_str):
    if not isinstance(size_str, str):
        return 0, 0.0
    sizes = size_str.split(',')
    total_sizes = len(sizes)
    out_of_stock_count = size_str.count('Out of stock')
    rate = out_of_stock_count / total_sizes if total_sizes > 0 else 0.0
    return out_of_stock_count, rate