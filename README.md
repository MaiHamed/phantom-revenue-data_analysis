# Stockout & Phantom Revenue Analysis

Uncover lost revenue due to stockouts using real ASOS product catalog data. This project parses messy size/stock strings, calculates phantom (lost) revenue per product, aggregates by brand, and produces a brand strategy scatter plot.

---

## Project Structure

```
stockout-analysis/
├── data/
│   └── products_asos.csv              # Raw product catalog
│
├── notebooks/
│   └── stockout_analysis.ipynb        # Main analysis notebook
│
├── outputs/
│   └── brand_strategy_scatter.png     # Generated visualization
│
├── .gitignore
└── README.md
```

---

## Dataset

**File:** `products_asos.csv`

| Column | Description | Example |
|---|---|---|
| `url` | Product page URL | `https://www.asos.com/...` |
| `name` | Product display name | `New Look trench coat in camel` |
| `size` | Comma-separated sizes, some marked out of stock | `UK 4,UK 6,UK 8,UK 10 - Out of stock` |
| `category` | Product category string | `New Look trench coat in camel` |
| `price` | Listed price (float after cleaning) | `49.99` |
| `color` | Colour label | `Neutral` |
| `sku` | Stock-keeping unit ID | `126704571.0` |
| `description` | Full description JSON blob containing brand info | `[{'Product Details': 'Coats & Jackets by New Look...` |
| `images` | Image URLs list | `['https://images.asos-media.com/...` |

---

## Setup

**Requirements:** Python 3.8+

```bash
pip install pandas matplotlib seaborn jupyter
jupyter notebook stockout_analysis.ipynb
```

---

## How It Works

### Step 1 — Load & Clean

The CSV has inconsistent rows, so `on_bad_lines='skip'` drops unparseable lines. Price is coerced to numeric and rows with missing prices are dropped.

```python
df = pd.read_csv('products_asos.csv', on_bad_lines='skip')
df['price'] = pd.to_numeric(df['price'], errors='coerce')
df = df.dropna(subset=['price'])
```

### Step 2 — Extract Brand

Brand is buried inside the `description` column as prose like `"Coats & Jackets by New Look"`. The extractor splits on `" by "` and takes the first word after it.

```python
def get_brand(text):
    if 'by' in text:
        try:
            return text.split('by ')[1].split(' ')[0]
        except:
            return "Unknown"
    return "Unknown"

df['brand_raw'] = df['description'].apply(get_brand)
```

Because this only captures the first word, a manual mapping normalises partial names:

```python
brand_map = {
    'New':           'New Look',
    'River':         'River Island',
    'Miss':          'Miss Selfridge',
    'TopshopWelcome':'Topshop'
}

df['Brand'] = df['brand_raw'].map(brand_map).fillna(df['brand_raw'])
```

Only brands with more than 5 products are kept for analysis:

```python
brand_counts   = df['Brand'].value_counts()
valid_brands   = brand_counts[brand_counts > 5].index
df_clean       = df[df['Brand'].isin(valid_brands)].copy()
```

**Resulting brand distribution (top 5):**

| Brand | Products |
|---|---|
| ASOS | 4,844 |
| Topshop | 1,017 |
| New Look | 511 |
| River Island | 467 |
| Miss Selfridge | 429 |

### Step 3 — Stockout Metrics

Each row's `size` column is a comma-separated string where out-of-stock sizes are annotated inline, e.g. `"UK 4,UK 6,UK 8,UK 10 - Out of stock,UK 12"`. The function counts total sizes and how many are flagged.

```python
def calculate_phantom_revenue(size_str):
    if not isinstance(size_str, str):
        return 0, 0.0

    sizes            = size_str.split(',')
    total_sizes      = len(sizes)
    out_of_stock_count = size_str.count('Out of stock')
    rate             = out_of_stock_count / total_sizes if total_sizes > 0 else 0.0

    return out_of_stock_count, rate

metrics = df_clean['size'].apply(calculate_phantom_revenue)
df_clean['Stockout_Count'] = [x[0] for x in metrics]
df_clean['Stockout_Rate']  = [x[1] for x in metrics]
```

**Lost (phantom) revenue** assumes each out-of-stock size slot represents one lost sale at full list price:

```python
df_clean['Lost_Revenue'] = df_clean['price'] * df_clean['Stockout_Count']
```

**Top 5 products by lost revenue:**

| Brand | Product | Price | OOS Sizes | Lost Revenue |
|---|---|---|---|---|
| Barbour | Beadnell wax jacket in black | £219 | 9 | £1,971 |
| Topshop | Premium real leather collared zip-through | £260 | 7 | £1,820 |
| ASOS | Premium real leather trench coat | £220 | 7 | £1,540 |
| ASOS | EDITION geo embellished fringe plunge midi | £250 | 6 | £1,500 |
| Topshop | Baggy co-ord jeans in green cord | £50 | 27 | £1,350 |

The Topshop jeans entry is a good example of how a cheap product with many sizes can still rack up significant phantom revenue.

### Step 4 — Brand Aggregation

Products are rolled up to brand level:

```python
brand_strategy = df_clean.groupby('Brand').agg(
    price         = ('price', 'mean'),
    Stockout_Rate = ('Stockout_Rate', 'mean'),
    Lost_Revenue  = ('Lost_Revenue', 'sum'),
    name          = ('name', 'count')
).reset_index()

brand_strategy = brand_strategy[brand_strategy['name'] > 10]
```

| Metric | What it tells you |
|---|---|
| `price` (mean) | Average price point of the brand |
| `Stockout_Rate` (mean) | Fraction of sizes typically missing across all products |
| `Lost_Revenue` (sum) | Total phantom revenue across every product the brand lists |
| `name` (count) | How many products the brand has in the catalog |

### Step 5 — Brand Strategy Scatter Plot

Each bubble is a brand. The red crosshairs divide the plot into four strategic quadrants.

```python
plt.figure(figsize=(12, 8))
sns.scatterplot(
    data     = brand_strategy,
    x        = 'price',
    y        = 'Stockout_Rate',
    size     = 'Lost_Revenue',
    sizes    = (50, 500),
    alpha    = 0.7,
    palette  = 'viridis'
)

# Label high-price, high-stockout brands
winners = brand_strategy[
    (brand_strategy['price'] > 40) &
    (brand_strategy['Stockout_Rate'] > 0.4)
]
for i in range(len(winners)):
    plt.text(
        winners.iloc[i]['price'] + 1,
        winners.iloc[i]['Stockout_Rate'],
        winners.iloc[i]['Brand']
    )

plt.axvline(x=40,  color='red', linestyle='--')
plt.axhline(y=0.4, color='red', linestyle='--')
plt.title('Brand Strategy Analysis')
plt.xlabel('Average Price')
plt.ylabel('Stockout Rate')
plt.show()
```

**Quadrant guide:**

| Quadrant | Avg Price | Stockout Rate | Action |
|---|---|---|---|
| Top-right | > £40 | > 40% | 🚨 Urgent — high-value sizes constantly missing |
| Top-left | < £40 | > 40% | ⚠️ High churn, low margin — investigate demand |
| Bottom-right | > £40 | < 40% | ✅ Premium brands well stocked |
| Bottom-left | < £40 | < 40% | 🟢 Stable, low priority |

Brands labelled in the chart are those sitting in the top-right quadrant — the highest-risk, highest-reward targets for restocking.

---

## Caveats

- **Phantom revenue is an upper bound.** It assumes every missing size would have sold at full list price. Real conversion rates and per-size demand will be lower.
- **Brand extraction is approximate.** The `"by "` split only captures a single word; compound brand names (e.g. `"Miss Selfridge"`) require the manual `brand_map` override. Any brand not in that map falls back to the raw extracted token.
- **Duplicate rows exist.** The raw data contains repeated SKUs (rows 0–2 in the sample are identical). Depending on the question, you may want to `drop_duplicates(subset='sku')` before analysis.

---

## Possible Extensions

- `drop_duplicates(subset='sku')` before analysis to avoid inflating counts for duplicate listings
- Segment by `category` to find which clothing types have the worst stockout patterns
- Track stockout rate over time if a dated snapshot is available
- Build a Streamlit dashboard for live brand filtering
