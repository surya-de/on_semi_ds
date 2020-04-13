# Introduction

This repository contains code and literature about my approach in handling marketing data.

## Understanding the data

Before starting with the analysis I spent a fair amount of time to understand the data provided to me. I had three csv files.
#### 1. TransactionData.csv - 
This file is an aggregation of all the transactions that has occurred in the past few years. From this file one can find the items procured by the customers and details about the transaction.
#### 2. RootCategory.csv-
This file is a **dimension table** which lists the Root hierarchy of the items and each root category is identified by unique **SKU Number**.
#### 3. LeafCategory.csv-
This file is yet another **dimension table** which breaks down the root category of items into further individual items. 

```python
# Code to check the size of transaction file.
my_file = open('TransactionData.csv')
ny_file.seek(0,2)
ny_file.tell()
>2482634728
```

## Analysis

To analyze the data I used **Jupyter Notebook** for **Python(V-3.7)**. Now that I know what the data is about, I need to perform some analysis on the data to gain more insight about it. 
### Challenges
The main challenge that I faced was the size of the Transaction data. Based on my system configuration it took me approximately 10 minutes to read 40K records. So, I tried the following-
1. I tried to spin a Notebook instance in AWS with higher compute but unfortunately AWS has some restriction over new personal accounts and it did not let me spin an instance with high compute.
2. So, I decided to perform my initial analysis on a small chunk of data. According to the principal of **probability distribution**, a small chunk of data can be representative of the entire dataset.
```Python
# Decide the column names for DataFrame.
trans_columns = ["id","Year","Customer Segment","Channel","CustomerID",
                 "Order No","Orders","Sales","SKU No","Supplier",
                 "Order Date","Month"]
trans_ph_df = pd.DataFrame(columns = trans_columns)
# Open the file using native python module
my_file = open('TransactionData.csv')

# Read the data and populate DataFrame.
i = 0
local_cntr = 0
for line in my_file:
    # Read first 1L files.
    if i == 100000:
        break
    # Ignore the first line.
    if i != 0:
        line = line.replace('"', '')
        l = line.split(',')
        trans_df.loc[i] = l
    i += 1
```

## Initial Findings
Each customer's sales data follow some uniform trends. Below are the categories of various trends on customer's sales data
##  1. Customers to target
As a supplier, **PHILLIPS** has a set of products that it supplies to various customers. Now, to understand potential customers, one approach is-
1. Find the customers who are procuring similar products but from a different supplier.
2. Check the trend of those customers and find how likely are they to churn. 
### Example
Suppose customer X and Y uses similar products as supplied by **PHILLIPS**. But, right now they are procuring the products from a different supplier. So, based on the sales data, I can find the probability of churn for both X and Y.

Now, lets assume X has a higher chance of churning compared to Y, so, we target X.

### Algorithm
1. Find the **SKU NO** of the items that are supplied by **PHILLIPS**.
2. Filter the transaction data on the **SKU NO**'s found in the previous step and make sure not to include **"PHILLIPS"**
3. Check the trend of total sales per year for each customer.
4. Based on the trend tag each customer into 4 categories- 'high', 'medium', 'keep track' and 'safe'. I have explained the calculation to get the tags in the "Churn section". Please click here to know more.
5. **High** means the customer is highly probable to leave the current supplier, **Medium** signifies there is a fair chance that the customer might leave, **Keep track** means the customers who are in the border line and might turn either way. **safe** means the customers who are not churning anytime soon.
6. Based on the tags we can create a funnel chart and start targeting the customers who are at the bottom or most likely to churn.

### Code
```python
'''
Module to perform analysis on PHILLIPSL
Steps-
1. Create a small dataset for PHILLIPS supplier, 
    the entire dataset is too big, so for analysis
    puropose I decided to create a datframe with small
    chunk pf data.
2. Check trends of customers to identify churn.
'''

# Create interim datafrme to store non phillips data.
trans_no_phl_df = pd.DataFrame(columns = trans_columns)
# Create interim datafrme to store only phillips data.
trans_ph_df = pd.DataFrame(columns = trans_columns)

for i in range(1, len(trans_df)):
    if trans_df['Supplier'][i] == 'PHILIPSLa':
        trans_ph_df.loc[i] = trans_df.loc[i]
    else:
        trans_no_phl_df.loc[i] = trans_df.loc[i]

# sort the data based on year.
trans_ph_df = trans_ph_df.sort_values('Year')

# Remove na values and reindex.
trans_ph_df.dropna(inplace = True)
trans_ph_df.reset_index(drop = True, inplace = True)

trans_no_phl_df.dropna(inplace = True)
trans_no_phl_df.reset_index(drop = True, inplace = True)

# store the sku no of items supplied by phillips.
items_phillips = set()
for items in trans_ph_df['SKU No']:
    items_phillips.add(items)

# Create an interim df with items supplied by phillips.
similar_items_df = pd.DataFrame(index = trans_no_phl_df.index, columns = trans_columns)
for i in range(0, len(trans_no_phl_df)):
    if trans_no_phl_df['SKU No'][i] in items_phillips:
        similar_items_df.loc[i] = trans_no_phl_df.loc[i]
similar_items_df.dropna(inplace = True)
similar_items_df.reset_index(drop = True, inplace = True)

# Data Wrangling and cleaning
# sort the data based on year
similar_items_df = similar_items_df.sort_values('Year')
similar_items_df['total_amt'] = 0

for i in range(1, len(similar_items_df)):
    # Convert month value to integer.
    similar_items_df['Month'][i] = int(similar_items_df['Month'][i])
    # Create total amount column by multiplying per unit cost.
    # with toatl orders.
    similar_items_df['total_amt'][i] = int(similar_items_df['Orders'][i]) * float(similar_items_df['Sales'][i])

# Create new aggegate dataframe with pertinent columns.
#cust_target_agg_df = pd.DataFrame(columns = ['Year', 'id', 'amount'])
cust_target_agg_df = similar_items_df.groupby(['Year', 'CustomerID', 'Supplier', 'SKU No']).total_amt.sum().to_frame('Total_Sale').reset_index()

# add composite key- concat(CustomerID, Supplier)
cust_target_agg_df['c_key'] = 'N/A'
for i in range(0, len(cust_target_agg_df)):
    cust_target_agg_df['c_key'][i] = cust_target_agg_df['CustomerID'][i] + cust_target_agg_df['Supplier'][i]
    
# creating the tagged dataframe
cust_target_tagged_df = pd.DataFrame(index=cust_target_agg_df.index, columns = ['customer_id', 'c_key', 'median', 'curr_year_percentage', 'tag'])
cust_target_tagged_df['tag'] = 'No Value'
idx = 0
for customers in cust_target_agg_df['CustomerID']:
    curr_year = 0
    percentage = []
    init_store = []
    for i in range(0, len(cust_target_agg_df)):
        if cust_target_agg_df['CustomerID'][i] == customers:
            init_store.append(cust_target_agg_df['Total_Sale'][i])
            cust_target_tagged_df['c_key'][i] = cust_target_agg_df['c_key'][i]
    
    # Calculate percentage increase or decrease.
    for i in range(1, len(init_store)):
        temp = ((init_store[i] - init_store[i - 1]) / init_store[i - 1]) * 100
        percentage.append(temp)
    
    ln = len(percentage) - 1
    if ln >= 0:
        curr_year = percentage[ln]
    # Drop the na values and reset index
    cust_target_tagged_df.dropna(subset=['c_key'], inplace = True)
    cust_target_tagged_df.reset_index(drop = True, inplace = True)
    # Calculate the kpi metrices.
    cust_target_tagged_df['customer_id'][idx] = customers
    cust_target_tagged_df['median'][idx] = median(percentage)
    cust_target_tagged_df['curr_year_percentage'][idx] = curr_year
    idx += 1
print(cust_target_tagged_df)
# Tag the customer ids based on anlysis
for i in range(0, len(cust_target_tagged_df)):
    if cust_target_tagged_df['median'][i] > 0 and cust_target_tagged_df['curr_year_percentage'][i] < 0:
        cust_target_tagged_df['tag'][i] = 'high focus'
    elif cust_target_tagged_df['median'][i] > 0 and cust_target_tagged_df['curr_year_percentage'][i] > 0 and cust_target_tagged_df['curr_year_percentage'][i] < cust_target_tagged_df['median'][i]:
        cust_target_tagged_df['tag'][i] = 'medium focus'
    elif cust_target_tagged_df['median'][i] < 0 and cust_target_tagged_df['curr_year_percentage'][i] > 0 and cust_target_tagged_df['curr_year_percentage'][i] > cust_target_tagged_df['median'][i]:
        cust_target_tagged_df['tag'][i] = 'keep track'
    else:
        cust_target_tagged_df['tag'][i] = 'safe'

cust_target_tagged_df
```

## License
[MIT](https://choosealicense.com/licenses/mit/)
