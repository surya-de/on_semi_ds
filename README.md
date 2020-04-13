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
##  2. Customer segmentation
***Customers can be segmented in multiple ways-***
1. **Based on customer's personal information**-
If we have data that explain the customer's interest and other information, we can use it to segment the customer. This way it will help us in knowing the customer better and reach out to them in a more personalized way. Such data can be gained by asking customers to take part in surveys.
For our use case we don't have this data available.
2. **Based on customer's product information**- We can try to segment the customer based on the kind of products the customer is interested in. This way we can reach out to customers by knowing what they already want. This data is available in this use case but this might not be a very efficient way to segment the customers for this use case. In this case, we might want to narrow down our focus to customers with higher value in terms of revenue.
3. **Based on revenue generated from customer**- According to me this would be a better approach for this use case where we are dealing with supplier and customer relationship. Reasons are as follows-
- We(from supplier's point of view) will have a better visibility for the customers who are contributing most in the revenue generation.
- We can focus on the customer-supplier relationships on each segments and can decide the amount of effort the marketing team needs to put for improvement in relationship.
- It will be easier to quantify efforts the marketing team needs to put for relationship improvement as it can be directly linked to a dollar value.

***Behavior of the current segmentation***

In this case the customers have been segmented based on the **sales value**.

#### Algorithm
1. Extracted all the unique customer segments available in the data.
2. I calculated the total sales amount.
```python
total_sales_amount = quantity * sales
```
3. Grouped the dataset baed on customer id and year. This will give me annual sales amount for each customer.
```
seg_agg_df = seg_df.groupby(['Year', 'CustomerID']).total_amt.sum().to_frame('Total_Sale').reset_index()
```
4. Finally, I plot the total sales amount for each year.

#### Code
```Python
'''
Module to identify behaviour of segments.
Steps-
1. Identify the KPI's based on which the segments have
    been created.
'''

# Unique segments present in the data
segments = set()
for elems in trans_df['Customer Segment']:
    segments.add(elems)
segments = list(segments)

# Check total sales trend for each segments
for segs in segments:
    seg_df = pd.DataFrame(index = trans_df.index, columns = trans_columns)
    seg_df = seg_df.sort_values('Year')
    seg_df['total_amt'] = 0
    
    for i in range(1, len(trans_df)):
        if trans_df['Customer Segment'][i] == segs:
            seg_df.loc[i] = trans_df.loc[i]
    
    seg_df.dropna(subset = ['Customer Segment'], inplace = True)
    seg_df.reset_index(drop = True, inplace = True)

    for i in range(1, len(seg_df)):
        # Create total amount column by multiplying per unit cost.
        # with toatl orders.
        seg_df['total_amt'][i] = int(seg_df['Orders'][i]) * float(seg_df['Sales'][i])
    # Create new aggegate dataframe with pertinent columns.
    seg_agg_df = seg_df.groupby(['Year', 'CustomerID']).total_amt.sum().to_frame('Total_Sale').reset_index()
    
    print('--------------------------------------------')
    print('Plotting for segment-')
    print(segs)
    print('--------------------------------------------')
    # Plot to understand the trend
    for customers in seg_agg_df['CustomerID']:
        financial_year = []
        total_sale = []
        
        for i in range(0, len(seg_agg_df)):
            if seg_agg_df['CustomerID'][i] == customers:
                financial_year.append(seg_agg_df['Year'][i])
                total_sale.append(seg_agg_df['Total_Sale'][i])
        plot(financial_year, total_sale)
        show()
```

#### Behavior

After plotting the chart, I found 11 distinct patterns for each segment. 

##  3. Churn
To calculate probable churn I have taken an approach of identifying the sales trend over years for each customer. To understand the trend I have used the following **KPIs:**

- **Percentage increase/decrease in total sales each year**

  This KPI denotes how the total sales value for each customer has increased or decreased over the years.
```
percentage_change_in_sales = ((current_yr_sales - previous_yr_sale) / previous_yr_sale) * 100
```
- **Median of the change in values over the years (med_val)**

  This KPI helps us understand how the normal trend for the customer have been over the years. A negative median value denotes a customer has a negative trend in sales amount for most of the times and vice versa.
- **Percentage change in the current year (curr_val)**

  This KPI helps us identify the customer's  sales trend for the current year.

** In all the cases, a negative value denotes a decrease in the sales value from the previous year and a positive value denotes an increase in sales value.

#### Tag Matrix based on KPI values
Based on the KPIs calculated in the previous steps I have formulated a tag matrix which will help us tag the customers based on  probability for churn.

**Tags-** There are 4 tags-

- **High**- The customers coming under this tag has a higher chance of switching to a different supplier.

```
1. If the median value is +ve and the current year sales value is negative, we tag 
the customer under this category.
2. A positive median signifies, the customer had a good account consistently in the 
past years.
3. A negative current year sales value signifies the sales amount for the current 
year has decreased.
4. This sudden dip in sales might signify a churn and since the median is positive, we
must be highly concerned because we might be losing a customer who had a good track 
record.
```
- **Medium**- Customers under this tag has a fair chance of switching.
```
1. If the median value is +ve, the current year sales value is also +ve 
but the current year value is less than the median value we tag the customer under 
this category.
2. A positive median signifies, the customer had a good account consistently in the 
past years.
3. A positive current year sales value signifies the sales amount for the current 
year has increased.
4. The current year sales value is less than the median value means even the sales have 
increased, its not as mush as how it normally happens.
5. Even though both the values are increasing, this might be something to worry about
because this decrease in increment from the median value might be an early stage and
the value might further decrease in the coming years.
```

- **keep track**- Customers under this tag are in the borderline. They might turn out to improve or churn.
```
1. If the median value is -ve and the current year sales value is +ve. 
2. A negative median signifies, the customer had a poor account consistently in the 
past years.
3. A positive current year sales value signifies the sales amount for the current 
year has increased.
5. In this case the +ve current year sales may be a good thing as the customer 
has a negative median. This case can turn out either way and hence can keep a track
of these cases.
```

- **safe**- These are the customers who have an ideal trend and ideal median. They are least likely to switch suppliers.

#### Another Approach

While analyzing the data I realized another way to achieve churn could be using classification algorithms.

- If we group the data based on year, customer id and SKU NO we will find in many cases the supplier has changed for the same item.

  eg- 
  
| Year | Customer_id | SKU_NO | Supplier | Sales Value |
|------|-------------|--------|----------|-------------|
| 2013 | 289306      | A123   | S1       | $1245       |
| 2014 | 289306      | A123   | S1       | $234        |
| 2015 | 289306      | A123   | S1       | $50         |
| 2016 | 289306      | A123   | S2       | $1000       |
| 2017 | 289306      | A123   | S2       | $1247       |

  
  If, customer- A have been procuring item -a123 from supplier S1 till 2015 and from 2016 the customer starts procuring the same item from supplier S2, this signifies a churn for supplier S1.

- This historical data could have been tagged and passed to a classification model after performing feature engineering.

- The mode could be used to predict future churns.

Initially, I wanted to use this approach but unfortunately I could not manage the compute of my system for the entire data. Hence, I figured out another statistical approach to solve this problem.

#### code
```Python
'''
Module to identify churn.
Steps-
1. Identify the KPI's based on which the segments have
    been created.
'''
churn_df = trans_df.copy()
churn_df = churn_df.sort_values('Year')
churn_df['total_amt'] = 0

for i in range(1, len(churn_df)):
    # Convert month value to integer.
    churn_df['Month'][i] = int(churn_df['Month'][i])
    # Create total amount column by multiplying per unit cost.
    # with toatl orders.
    churn_df['total_amt'][i] = int(churn_df['Orders'][i]) * float(churn_df['Sales'][i])

# Create new aggegate dataframe with pertinent columns.
churn_agg_df = churn_df.groupby(['Year', 'CustomerID', 'Supplier', 'SKU No']).total_amt.sum().to_frame('Total_Sale').reset_index()

# add composite key- concat(CustomerID, Supplier)
churn_agg_df['c_key'] = 'N/A'
for i in range(0, len(churn_agg_df)):
    churn_agg_df['c_key'][i] = churn_agg_df['CustomerID'][i] + churn_agg_df['Supplier'][i]
    
# creating the tagged dataframe
churn_tagged_df = pd.DataFrame(index=churn_agg_df.index, columns = ['customer_id', 'c_key', 'median', 'curr_year_percentage', 'tag'])
churn_tagged_df['tag'] = 'No Value'
idx = 0
for customers in churn_agg_df['CustomerID']:
    curr_year = 0
    percentage = []
    init_store = []
    for i in range(0, len(churn_agg_df)):
        if churn_agg_df['CustomerID'][i] == customers:
            init_store.append(churn_agg_df['Total_Sale'][i])
            churn_tagged_df['c_key'][i] = churn_agg_df['c_key'][i]
    
    # Calculate percentage increase or decrease.
    for i in range(1, len(init_store)):
        temp = ((init_store[i] - init_store[i - 1]) / init_store[i - 1]) * 100
        percentage.append(temp)
    
    ln = len(percentage) - 1
    if ln >= 0:
        curr_year = percentage[ln]
    # Drop the na values and reset index
    churn_tagged_df.dropna(subset=['c_key'], inplace = True)
    churn_tagged_df.reset_index(drop = True, inplace = True)
    # Calculate the kpi metrices.
    churn_tagged_df['customer_id'][idx] = customers
    churn_tagged_df['median'][idx] = median(percentage)
    churn_tagged_df['curr_year_percentage'][idx] = curr_year
    idx += 1
print(churn_tagged_df)
# Tag the customer ids based on anlysis
for i in range(0, len(churn_tagged_df)):
    if churn_tagged_df['median'][i] > 0 and churn_tagged_df['curr_year_percentage'][i] < 0:
        churn_tagged_df['tag'][i] = 'high focus'
    elif churn_tagged_df['median'][i] > 0 and churn_tagged_df['curr_year_percentage'][i] > 0 and churn_tagged_df['curr_year_percentage'][i] < churn_tagged_df['median'][i]:
        churn_tagged_df['tag'][i] = 'medium focus'
    elif churn_tagged_df['median'][i] < 0 and churn_tagged_df['curr_year_percentage'][i] > 0 and churn_tagged_df['curr_year_percentage'][i] > churn_tagged_df['median'][i]:
        churn_tagged_df['tag'][i] = 'keep track'
    else:
        churn_tagged_df['tag'][i] = 'safe'
```
