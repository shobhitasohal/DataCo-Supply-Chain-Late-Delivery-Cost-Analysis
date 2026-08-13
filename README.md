# DataCo Supply Chain — Late Delivery Cost Analysis

An end-to-end data analytics project identifying which regions late deliveries are costing DataCo the most, using Python for data cleaning and EDA and Power BI for the executive dashboard.

**Tools used:** Python (Pandas, Matplotlib, Seaborn) · Power BI

## Data Source
[DataCo Smart Supply Chain Dataset](https://data.mendeley.com/datasets/8gx2fvg2k6/5) — Constante, Silva & Pereira (2019), Mendeley Data.

---

## About This Project

End-to-end solo project covering data cleaning, exploratory data analysis in Python, and dashboard design in Power BI. The cleaned dataset was exported from Jupyter and imported directly into Power BI to build a one-page executive summary answering the business question.

---

## Business Problem

> DataCo ships online orders around the world. It can only afford to improve delivery in three regions this year. Pick the three regions where late deliveries are costing the most business.

Before any region could be recommended, two underlying questions needed answers:

1. **Cost basis** — What's the right dollar metric for "costing the business" — profit, or sales?
2. **Prioritization signal** — Is late delivery *rate* a reliable way to rank regions, or does it need to be paired with dollar impact?

---

## Approach

Late delivery rate was tested first as the natural prioritization metric, then checked against dollar impact once it became clear rate alone couldn't differentiate regions meaningfully. Two profit columns were investigated for reliability before being ruled out as the cost basis. Order status values were cross-checked against which statuses actually appear on late-delivered orders.

---

## EDA Steps (Python)

### 1. Load and scope the data

```python
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns

df = pd.read_csv('DataCoSupplyChainDataset.csv', encoding='latin-1')
df.head()
```

Dropped columns not relevant to the business question (customer PII, geolocation, product imagery, redundant IDs):

```python
df = df.drop(columns=['Benefit per order','Sales per customer','Late_delivery_risk',
    'Category Id', 'Customer City', 'Customer Country', 'Customer Email', 'Customer Fname',
    'Customer Id', 'Customer Lname', 'Customer Password', 'Customer Segment',
    'Customer State', 'Customer Street', 'Customer Zipcode', 'Department Id', 'Latitude',
    'Longitude', 'Order Customer Id', 'Order Item Cardprod Id','Order Item Discount', 'Order State',
    'Order Zipcode', 'Product Card Id', 'Product Category Id','Product Status',
    'Product Description', 'Product Image'])
```
Reduced from 53 columns / 180,519 rows to 26 columns.

---

### 2. Data types and cleaning

```python
cols = ['order date (DateOrders)','shipping date (DateOrders)']
df[cols] = df[cols].apply(pd.to_datetime)

cols_string = ['Order Id','Order Item Id']
df[cols_string] = df[cols_string].astype('object')
```
Confirmed zero duplicate rows (`df[df.duplicated()].shape` → `(0, 26)`).

---

### 3. Investigating negative profit values

`Order Item Profit Ratio` and `Order Profit Per Order` contained large, unexplained negative values across 33,375 rows. Checked whether the negative values could be explained by any other numeric field:

```python
numeric_df = df.select_dtypes(include=['number'])
corr_matrix = numeric_df.corr()
sns.heatmap(corr_matrix, annot=True, fmt=".2f", cmap="coolwarm")
plt.title("Correlation Graph of Supply Chain Metrics")
plt.show()
```

Also tested for non-linear relationships via pairplot before ruling the columns out:

```python
sns.pairplot(data=df, vars=['Order Item Profit Ratio','Order Item Discount Rate',
                             'Product Price','Shipping Mode'])
plt.show()
```

Since neither column was explained by any other field in the dataset, both were dropped rather than used as an unverified cost metric:

```python
df = df.drop(columns=['Order Item Profit Ratio','Order Profit Per Order'])
```

---

### 4. Feature engineering — Shipping Gap (Days)

```python
df['Shipping Gap(Days)'] = df['Days for shipping (real)'] - df['Days for shipment (scheduled)']
```
Represents how many days a shipment deviated from its committed delivery window.

---

### 5. Outlier investigation

```python
cols_to_plot = ['Order Item Discount Rate','Order Item Product Price', 'Sales','Order Item Total']
for col in cols_to_plot:
    sns.boxplot(data=df, x=col, color='skyblue')
    plt.title(f'Boxplot of {col}')
    plt.show()
```

Reviewed category-level price distributions and manually verified specific outlier categories (Kids' Golf Clubs, Basketball, Strength Training) — confirmed these were legitimate high-end products, not data errors. No rows were removed on this basis.

---

### 6. Order Status investigation

```python
df['Order Status'].unique()
# ['PENDING_PAYMENT', 'PENDING', 'PROCESSING', 'ON_HOLD', 'COMPLETE',
#  'SUSPECTED_FRAUD', 'CANCELED', 'PAYMENT_REVIEW', 'CLOSED']

late_deliveries = df[df['Delivery Status'] == 'Late delivery']
late_deliveries['Order Status'].unique()
# ['PENDING_PAYMENT', 'PENDING', 'PROCESSING', 'ON_HOLD', 'COMPLETE', 'PAYMENT_REVIEW', 'CLOSED']
```
`Order Status` has no official business-meaning documentation from the data source. Cross-checking confirmed only 7 of the 9 raw values ever appear on late-delivered orders — canceled and suspected-fraud orders never show up as "late," which makes sense, since those orders wouldn't complete the shipping process.

---

### 7. Regional analysis — rate vs. dollar impact

```python
late_order_volume_by_region = df[df['Delivery Status']=='Late delivery'].groupby('Order Region').size()
sns.barplot(x=late_order_volume_by_region.index, y=late_order_volume_by_region.values)
plt.xticks(rotation=90)
plt.ylabel('Order Volume by Region')
plt.show()

sales_order = late_deliveries.groupby('Order Region')['Sales'].sum().sort_values(ascending=False)
sns.barplot(data=late_deliveries, x='Order Region', y='Sales', estimator=sum)
plt.xticks(rotation=90)
plt.show()
```
Western Europe, Central America, and South America emerged as the top 3 regions by both order volume and total Sales exposure.

---

### 8. Export

```python
df.to_csv('supply_chain_clean.csv', index=False)
```
Cleaned dataset exported for direct import into Power BI.

---

## Dashboard Preview

The cleaned CSV was imported directly into Power BI. DAX measures were built for Late Delivery Rate, Sales at Risk, Average Shipping Variance, and Worst Region by $ Impact, and used to build a one-page executive summary.

**Executive Summary — On-time Delivery Goals (by Region)**

![Executive Summary Dashboard](dashboard/dashboard_screenshot.png)

---

## Key Findings

### Strategic Summary

| Business Question | Finding | Implication |
|---|---|---|
| Which regions have the highest dollar impact? | Western Europe, Central America, South America — $8.0M of $19.9M total sales at risk | Prioritize these 3 regions for delivery investment |
| Is delivery rate a reliable prioritization signal? | Rate is nearly uniform (53–56%) across every region | Rate alone cannot differentiate a problem region from an average one — dollar impact is the deciding factor |
| Can profit be used as the cost basis? | Both profit columns contain large, unexplained negative values | Excluded from analysis; Sales used as the dollar basis instead |

---

### Market Overview
- Dataset covers **180,519 order line items** across **65,752 distinct orders** and **23 regions**
- **54.8% of all orders are late-delivered** — the baseline rate across the entire dataset, and the figure against which every region is compared

---

### 1. Where is late delivery volume concentrated?
- **Western Europe and Central America have by far the highest late-delivery order volume** — roughly double the next closest region (South America), and more than 10× the smallest regions (Canada, Central Asia)
- Order volume by region closely tracks overall market size — the regions with the most late deliveries are simply the regions with the most orders overall

---

### 2. Rate vs. dollar impact — the central insight
- **Late delivery rate barely varies by region** — the spread across all 23 regions is roughly 8 percentage points at most, and under 1.5 points across the 5 broader markets
- **Dollar impact varies far more, and is driven by order volume, not delivery performance** — Western Europe ($3.3M at risk) and Central America ($3.1M) top the ranking not because their delivery process is worse, but because they're DataCo's largest markets, so even an average rate produces the largest revenue exposure
- **High-rate regions can still be low-priority in dollar terms** — several African regions post some of the highest late-delivery rates in the dataset but carry minimal dollar exposure due to low order volume, confirming that rate and cost point to different regions and answer different questions

---

### 3. Data quality — why profit wasn't the cost basis
- `Order Item Profit Ratio` and `Order Profit Per Order` contained large negative values with no explanation from any other field in the dataset (checked via correlation and pairplot)
- Rather than build a business recommendation on an unverified metric, both columns were excluded and **Sales** was used as the dollar basis for the entire analysis instead

---

### Recommendation
**Prioritize Western Europe, Central America, and South America** — together $8.0M of the $19.9M total sales at risk across all regions. Delivery rate is near-uniform globally, so dollar impact, driven by order volume, is the deciding factor, not rate.

---

## Data Notes

- **`Order Status`** has no official business-meaning documentation from the data source; values were interpreted using standard e-commerce order-lifecycle conventions, not a confirmed definition
- **`Order Item Profit Ratio`** and **`Order Profit Per Order`** were excluded after investigation showed their negative values could not be explained by any other field in the dataset — Sales was used as the dollar cost basis instead
- **Sales at Risk** (as calculated on the dashboard) sums Sales across all late-delivered orders regardless of Order Status — it is not filtered to fulfilled/completed orders only, and this scope is intentional for simplicity at the executive-summary level

---

## Repository Contents

| File | Description |
|---|---|
| `SupplyChain_Analysis.ipynb` | Full data cleaning, EDA, and analysis pipeline in pandas |
| `dashboard/dashboard_screenshot.png` | Final Power BI executive dashboard |

---

## Connect

[LinkedIn](http://www.linkedin.com/in/shobhitasohal)
