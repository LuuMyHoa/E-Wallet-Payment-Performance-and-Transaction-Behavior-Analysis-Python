# E-Wallet Payment Performance & Transaction Behavior Analysis | Python

![banner](banner.png)

## 📑 Table of Contents  
1. [📌 Background & Overview](#-background--overview)  
2. [📂 Dataset Description](#-dataset-description)
3. [⚒️ Data Preparation & Quality Check](#data-preparation--quality-check)
4. [🔎 Business Analysis](#-business-analysis)
5. [📝 Conclusion & Recommendations](#-conclusion--recommendations)

## 📌 Background & Overview  

This project analyzes e-wallet transaction data to evaluate product performance, user behavior, and operational efficiency.

**Key objectives:**
- Identify top-performing and underperforming products and teams  
- Analyze transaction patterns and user behavior across different transaction types  
- Detect potential risks such as high refund contribution and invalid transactions  
- Support business decision-making through data-driven insights

## 📂 Dataset Description

### Data Source  
**Source:** Internal simulation dataset

| Table            | Description                                 | Size       |
| ---------------- | ------------------------------------------- | ---------- |
| payment_report.csv | Monthly payment volume by product           | 919 rows x 5 cols  |
| product.csv        | Product metadata (category, team ownership) | 492 rows x 3 cols  |
| transactions.csv   | Detailed transaction records                | 1.3M+ rows x 9 cols |

## ⚒️ Data Preparation & Quality Check

**1️⃣ Payment Data (df_payment_enriched)**

<details>
<summary> Code Python </summary>
 
```python
#Create df_payment_enriched (Merge payment + product)
df_payment_enriched = pd.merge(df_payment,df_product,on="product_id",how="left")

#Check data type
df_payment_enriched.dtypes

#Convert report_month → datetime
df_payment_enriched['report_month'] = pd.to_datetime(df_payment_enriched['report_month'], format='%Y-%m')

#Check missing data
df_payment_enriched.isnull().sum()

#Check duplicate
df_payment_enriched.duplicated().sum()

#Check invalid or abnormal values
df_payment_enriched['volume'].describe()
df_payment_enriched['payment_group'].value_counts()
```

</details>

**Actions:**
- Merged `payment_report` with `product` (LEFT JOIN)
- Converted `report_month` → datetime
- Checked missing values, duplicates, abnormal values  

**Key findings:**
- Missing values (~22 rows in `category`, `team_own`) due to unmatched `product_id` → **retained to preserve volume**
- No duplicate records detected  
- Volume distribution reflects real-world behavior (wide range)
- `payment_group` contains valid values: payment, refund

👉 Dataset is **clean and reliable for analysis**


**2️⃣ Transaction Data (df_transactions)**

<details>
<summary> Code Python </summary>
 
```python
#Check data type
df_transactions.dtypes

#Convert timeStamp → datetime
df_transactions['timeStamp'] = pd.to_datetime(df_transactions['timeStamp'], unit='ms')

#Check missing data
df_transactions.isnull().sum()

#Check duplicate
df_transactions.duplicated().sum()

#Remove duplicate
df_transactions = df_transactions.drop_duplicates()

#Check invalid or abnormal values
df_transactions['volume'].describe()
df_transactions['transType'].value_counts()
df_transactions['transStatus'].value_counts()
```

</details>

**Actions:**
- Converted timestamp → datetime  
- Removed 28 duplicate rows  
- Validated transaction fields  

**Key findings:**
- Missing values (sender_id, receiver_id) are business-valid (top-up & withdrawal flows)
- Only `transType = 2, 8` are relevant for business logic  
- `transStatus` (=1 or <0) reflects success vs failure  

👉 Large-scale dataset with realistic transaction patterns
  
## 🔎 Business Analysis

**1. Top-performing products**

Identify Top 3 product_ids by total volume

```python
top3_products = (
    df_payment_enriched
    .groupby('product_id', as_index=False)['volume']  
    .sum()                                             
    .sort_values('volume', ascending=False)            
    .head(3)                                      
    .reset_index(drop=True)
)
top3_products.index += 1
total_vol = df_payment_enriched['volume'].sum()
top3_products['contribution_%'] = (
    top3_products['volume'] / total_vol * 100
).round(2)
```

|  | product_id | volume | contribution_% |
|---|---|---|---|
| 1 | 1976 | 61797583647 | 33.99 |
| 2 | 429 | 14667676567 | 8.07 |
| 3 | 372 | 13713658515 | 7.54 |

👉 Top 3 products  drive nearly 50% of total volume, indicating high concentration and a strong reliance on a few key performers. This creates concentration risk, where a decline in top products could significantly impact overall transaction volume.

**2. Data integrity check**

Validate rule: Each product belongs to only one team

```python
teams_per_product = (
    DF_product
    .groupby('product_id')['team_own']
    .nunique()                    
    .reset_index()
    .rename(columns={'team_own': 'num_teams'})
)

abnormal = teams_per_product[teams_per_product['num_teams'] > 1]
```
|product_id	|num_teams|
|---|---|
|   |    |

👉 Each product belongs to only one team → no structural issues detected

**3. Team performance analysis**

Identify lowest-performing team since Q2 2023. Drill down: Which category contributes the least

```python
q2_start = pd.Timestamp('2023-04-01')
df_q2 = df_payment_enriched[df_payment_enriched['report_month'] >= q2_start].copy()

team_volume = (
    df_q2
    .dropna(subset=['team_own'])
    .groupby('team_own')['volume']
    .sum()
    .sort_values()
    .reset_index()
)
team_volume.columns = ['team_own', 'total_volume']

worst_team = team_volume.iloc[0]['team_own']
```

| | team_own | total_volume |
| :--- | :--- | :--- |
| 0 | APS | 51141753 |
| 1 | ASL | 11284361730 |
| 2 | ASD | 31090428473 |

```python
category_volume = (
    df_q2[df_q2['team_own'] == worst_team]
    .groupby('category')['volume']
    .sum()
    .sort_values()
    .reset_index()
)
category_volume.columns = ['category', 'total_volume']
total_team_vol = category_volume['total_volume'].sum()
category_volume['contribution_%'] = (
    category_volume['total_volume'] / total_team_vol * 100
).round(2)
```

| | category | total_volume | contribution_% |
| :--- | :--- | :--- | :--- |
| 0 | PXXXXXE | 25232438 | 49.34 |
| 1 | PXXXXXS | 25909315 | 50.66 |

👉  **APS** is the lowest-performing team since Q2 2023, contributing less than 1% of the total volume compared to other teams. No category within APS significantly drives volume, indicating a lack of product diversity or a niche operational scope.

**4. Refund analysis**

Analyze refund transaction contribution by source_id. Identify: Source causing the highest refund volume

```python
df_refund = df_payment_enriched[df_payment_enriched['payment_group'] == 'refund'].copy()

source_contribution = (
    df_refund
    .groupby('source_id')['volume']
    .sum()
    .reset_index()
    .sort_values('volume', ascending=False)
)

total_refund_vol = source_contribution['volume'].sum()
source_contribution['contribution_%'] = (
    source_contribution['volume'] / total_refund_vol * 100
).round(2)
```

| | source_id | volume | contribution_% |
| :--- | :--- | :--- | :--- |
| 1 | 38 | 36527454759 | 59.11 |
| 2 | 39 | 16119059662 | 26.08 |
| 0 | 37 | 9151069226 | 14.81 |

👉 One source (source_id = 38) contributes ~59% of total refund volume. Refund risk is highly concentrated.

**5. Transaction classification**

Define transaction_type based on business rules: Bank Transfer, Withdraw, Top-up, Payment, Transfer, Split Bill, Invalid

```python
conditions = [
    (df_transactions['transType'] == 2) & (df_transactions['merchant_id'] == 1205),
    (df_transactions['transType'] == 2) & (df_transactions['merchant_id'] == 2260),
    (df_transactions['transType'] == 2) & (df_transactions['merchant_id'] == 2270),
    (df_transactions['transType'] == 2),                                              # others merchant_id
    (df_transactions['transType'] == 8) & (df_transactions['merchant_id'] == 2250),
    (df_transactions['transType'] == 8),                                              # others merchant_id
]

choices = [
    'Bank Transfer Transaction',
    'Withdraw Money Transaction',
    'Top Up Money Transaction',
    'Payment Transaction',
    'Transfer Money Transaction',
    'Split Bill Transaction',
]

df_transactions['transaction_type'] = np.select(
    conditions, choices, default='Invalid Transaction'
)

type_counts = df_transactions['transaction_type'].value_counts()

```

| transaction_type | count |
| :--- | :--- |
| Payment Transaction | 398665 |
| Transfer Money Transaction | 341173 |
| Top Up Money Transaction | 290498 |
| Invalid Transaction | 220658 |
| Bank Transfer Transaction | 37879 |
| Withdraw Money Transaction | 33725 |
| Split Bill Transaction | 1376 |

👉 A significant number of transactions are classified as “Invalid”. Data classification issue impacts reporting accuracy.

**6. Transaction behavior analysis**

For each valid transaction type: number of transactions, total volume, unique senders, unique receivers

```python
df_valid = df_transactions[df_transactions['transaction_type'] != 'Invalid Transaction'].copy()

summary = df_valid.groupby('transaction_type').agg(
    num_transactions=('transaction_id', 'count'),          
    total_volume=('volume', 'sum'),                        
    num_senders=('sender_id', 'nunique'),                 
    num_receivers=('receiver_id', 'nunique')              
).reset_index()

# format
for col in summary.select_dtypes(include='number').columns:
    summary[col] = summary[col].apply(lambda x: "{:,.0f}".format(x))
```

| | transaction_type | num_transactions | total_volume | num_senders | num_receivers |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 0 | Bank Transfer Transaction | 37,879 | 50,605,806,190 | 23,156 | 9,271 |
| 1 | Payment Transaction | 398,665 | 71,850,608,441 | 139,583 | 113,298 |
| 2 | Split Bill Transaction | 1,376 | 4,901,464 | 1,323 | 572 |
| 3 | Top Up Money Transaction | 290,498 | 108,605,618,829 | 110,409 | 110,409 |
| 4 | Transfer Money Transaction | 341,173 | 37,032,880,492 | 39,021 | 34,585 |
| 5 | Withdraw Money Transaction | 33,725 | 23,418,181,420 | 24,814 | 24,814 |

👉 Different transaction types reflect distinct user behaviors and use cases.

## 🔎 Conclusion & Recommendations  

### 📍 Key Insights
- Transaction volume is highly concentrated in a few key products
- One team (APS) significantly underperforms without a strong product driver
- Refund volume is heavily skewed toward a single source
- Data quality issues exist in transaction classification

### 💡 Recommendations

- Reduce dependency on top products by diversifying product performance  
- Review and optimize strategy for underperforming teams (e.g., APS)  
- Investigate root causes of high refund volume from source_id 38  
- Improve transaction classification logic to reduce invalid records and enhance data quality  
