# EDA and Data Wrangling for E-Wallet Payment Performance Analysis | Python

## 📑 Table of Contents  
1. [📌 Background & Overview](#-background--overview)  
2. [📂 Dataset Description](#-dataset-description)
3. [⚒️ EDA & Data Wrangling](-eda--data-wrangling) 
4. [🔎 Final Conclusion & Recommendations](#-final-conclusion--recommendations)

## 📌 Background & Overview  
 
### Objectives

The project aims to: 

- Perform Exploratory Data Analysis (EDA) to assess data quality
- Apply Data Wrangling techniques to clean and transform data
- Analyze product performance and transaction behavior
- Categorize transactions into meaningful types for further analysis

## 📂 Dataset Description

### Data Source  
**Source:** Internal simulation dataset

| Table            | Description                                 | Size       |
| ---------------- | ------------------------------------------- | ---------- |
| payment_report.csv | Monthly payment volume by product           | 919 rows x 5 cols  |
| product.csv        | Product metadata (category, team ownership) | 492 rows x 3 cols  |
| transactions.csv   | Detailed transaction records                | 1.3M+ rows x 9 cols |

## ⚒️ EDA & Data Wrangling

### 1️⃣ Exploratory Data Analysis (EDA)  

 **Key checks:**
 
- Data types consistency
- Missing values
- Duplicate records
- Invalid or abnormal values

**EDA df_payment_enriched**

```python
#Create df_payment_enriched (Merge payment + product)
df_payment_enriched = pd.merge(df_payment,df_product,on="product_id",how="left")
```

```python
#Check data type
df_payment_enriched.dtypes
```

```python
#Convert report_month → datetime
df_payment_enriched['report_month'] = pd.to_datetime(df_payment_enriched['report_month'], format='%Y-%m')
```

```python
#Check missing data
df_payment_enriched.isnull().sum()
```
```python
#Check duplicate
df_payment_enriched.duplicated().sum()
```

```python
#Check invalid or abnormal values
df_payment_enriched['volume'].describe()
df_payment_enriched['report_month'].value_counts()
df_payment_enriched['payment_group'].value_counts()
dF_payment_enriched['category'].value_counts()
df_payment_enriched['team_own'].value_counts()
```

**EDA df_transactions**

```python
#Check data type
df_transactions.dtypes
```
```python
#Convert timeStamp → datetime
df_transactions['timeStamp'] = pd.to_datetime(df_transactions['timeStamp'], unit='ms')
```
```python
#Check missing data
df_transactions.isnull().sum()
```
```python
#Check duplicate
df_transactions.duplicated().sum()
```
```python
#Remove duplicate
df_transactions.drop_duplicates()
```
```python
#Check invalid or abnormal values
df_transactions['volume'].describe()
df_transactions['transType'].value_counts()
```

### 2️⃣ Data Wrangling & Business Analysis

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
top3_products
```

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
worst_volume = team_volume.iloc[0]['total_volume']

```
**4. Refund analysis**
Analyze refund transaction contribution by source_id
Identify: Source causing the highest refund volume

```pythom
df_refund = df_payment_enriched[
    df_payment_enriched['payment_group'] == 'refund'
].copy()

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
print("\nPhân phối transaction type:")
print(type_counts)
```

**6. Transaction behavior analysis**

For each valid transaction type: Number of transactions, Total volume, Unique senders, Unique receivers

```python
df_valid = df_transactions[
    df_transactions['transaction_type'] != 'Invalid Transaction'
].copy()

summary = df_valid.groupby('transaction_type').agg(
    num_transactions=('transaction_id', 'count'),          
    total_volume=('volume', 'sum'),                         
    num_senders=('sender_id', 'nunique'),                  
    num_receivers=('receiver_id', 'nunique')               
).reset_index()

```
## 🔎 Final Conclusion & Recommendations  

👉🏻 Based on the insights and findings above, we would recommend the [stakeholder team] to consider the following:  

📍 Key Takeaways:  
- Recommendation 1  
- Recommendation 2  
- Recommendation 3
