# **Anti-Sybil-Python-Package**


*The data obtained from the cluster analysis was used to build the Legos that make up the anti-Sybil package.
It is made up of three individual Legos and an aggregator.
Each Lego set is designed to crackdown on different Sybil behavioral characteristics.*

The aggregator and each Lego are broken down here.

### [On-chain Footprint Lego](#on-chain-footprint-lego-1)
### [Address Correlation](#address-correlation-lego)
### [Installing Package](#installing-package-1)
___
## **On-chain Footprint Lego**
___

The TDD (Transaction Date Mean Difference) metric is utilized by this Lego, and it is computed by subtracting each transaction date from the following one to obtain the difference, then dividing by the total number of transactions (i.e., the mean difference or average difference).


$$ \frac{\sum_{i=1}^{n} (date_i - date_{i+1})}{transacting\_dates} $$


This Lego was developed to counter any attempts by a Sybil attacker to get around transaction count-based security measures by carrying out insignificant transactions in a short period of time to increase the number of transactions, avoid detection, and carry out their attack.

In the case of a real person, the TDD will be greater than 1 due to the irregular transacting dates or the fact that their transactions happen at different intervals throughout the year, but in the case of a sybil, falsifying transactions are more likely to happen all at once or all at once because they are script orchestrated, likely to result in a TDD that is less than or equal to 1.
 



### **Lego Walk-through:**

The Lego is presently querying data using the covalent transactions API, which is constructed in a function to collect the dates of each transaction and utilizes the pocket endpoint to obtain the transaction count (running against a node is a better alternative).

    Code Refernece
```python number

def get_txn_count(address):
    pokt_Eth_endpoint = "https://eth-mainnet.gateway.pokt.network/v1/lb/00000000000000000000"
    headers = {'content-type': 'application/json'}

    payload = {
        "method": "eth_getTransactionCount",
        "params": [address, "latest"],
        "jsonrpc": "2.0",
        "id": 1,
    }
    response = requests.post(pokt_Eth_endpoint, data=json.dumps(payload), headers=headers).json()
    transaction_count = int(response['result'], 16)
    return transaction_count


def txn_dates(address):
    transaction_dates = []
    url = f"https://api.covalenthq.com/v1/1/address/{address}/transactions_v2/?no-logs=True"
    
    headers = {
        "x-api-key": covalent_api_key
    }

    response = session.get(url, headers=headers)
    txn_data = response.json()['data']['items']
    txn_counts = get_txn_count(address)
    for index in range(len(txn_data)):
        transaction_dates.append(txn_data[index]['block_signed_at'])
    transaction_dates.reverse()
    print(transaction_dates)
    transaction_dates = pd.to_datetime(pd.Series(transaction_dates))

    return transaction_dates,txn_counts

```

The code below depicts how the Sybil status is determined based on the TDD.

    Code Refernece

``` python number
    dates,txn_count = txn_dates(address)
    date_diff = []
    for date in range(1, len(dates)):
        diff = (dates[date] - dates[date-1]).days
        date_diff.append(diff)
    TDD = (sum(date_diff)/txn_count)

    if 1< TDD:
        Sybil_state = False
    else:
        Sybil_state = True
        
    return Sybil_state
```
As sybil complexity rises, the TDD metric could be further adjusted and optimized.

___

## **Address Correlation Lego**
The "address correlation Lego" is a pattern-identification Lego that is intended to be used against previously known sybil patterns that will be stored in a readable format (currently using a pandas dataframe). Its purpose is to judge the correlation between the activities of an address and those of sybil addresses using the [Spearman's rank correlation coefficient](https://en.wikipedia.org/wiki/Spearman%27s_rank_correlation_coefficient).

$$r_s = \frac{\text{cov}(R_X, R_Y)}{\sigma_{R_X} \sigma_{R_Y}}$$

Where $R_{X}$, $R_{Y}$ are the ranked variables and $\text{cov}$, $\sigma$ represent the covariance and standard deviation respectively.

### **Lego Walk-through:**


In order to obtain the following information, this Lego uses the Pokt enpoint to count the number of transactions, the balance, and the Covalent Transaction API to query the following :




<table border="1">
  <tr>
    <th>Variable Name</th>
    <th>Description</th>
  </tr>
  <tr>
    <td>Number of active days</td>
    <td>Number of days account was active</td>
  </tr>
  <tr>
    <td>Initial amount received</td>
    <td>Initial amount received by the account</td>
  </tr>
  <tr>
    <td>First active day</td>
    <td>Day of the first transaction</td>
  </tr>
  <tr>
    <td>First active month</td>
    <td>Month of the first transaction</td>
  </tr>
  <tr>
    <td>First active year</td>
    <td>Year of the first transaction</td>
  </tr>
  <tr>
    <td>Last active day</td>
    <td>Day of the last transaction</td>
  </tr>
  <tr>
    <td>Last active month</td>
    <td>Month of the last transaction</td>
  </tr>
  <tr>
    <td>Last active year</td>
    <td>Year of the last transaction</td>
  </tr>
  <tr>
    <td>Transaction count</td>
    <td>Number of transactions made by the account</td>
  </tr>
</table>

    Code Refernece

```python 
    def get_txn_count(address):
        pokt_Eth_endpoint = "https://eth-mainnet.gateway.pokt.network/v1/lb/78cad988d47e553027481226"
        headers = {'content-type': 'application/json'}

        payload = {
            "method": "eth_getTransactionCount",
            "params": [address, "latest"],
            "jsonrpc": "2.0",
            "id": 1,
        }
        response = requests.post(pokt_Eth_endpoint, data=json.dumps(payload), headers=headers).json()
        transaction_count = int(response['result'], 16)
        return transaction_count
    
    def get_transactions_and_date_info(address):
        dates = []
        api_key = "api-key"
        transaction_dates = []
        url = f"https://api.covalenthq.com/v1/1/address/{address}/transactions_v2/?no-logs=True"
        
        headers = {
            "x-api-key": covalent_api_key
        }
    
        response = session.get(url, headers=headers)
        txn_data = response.json()['data']['items']
        txn_count = get_txn_count(address)
        for index in range(len(txn_data)):
            dates.append(txn_data[index]['block_signed_at'])

        dates = [*set(dates)]

        first_active_day =  int(re.sub('-','',re.search(r"\d{4}-\d{2}-(\d{2})",txn_data[-1]['block_signed_at']).group(1)))

        first_active_month =  int(re.sub('-','',re.search(r"\d{4}-(\d{2})-\d{2}",txn_data[-1]['block_signed_at']).group(1)))

        first_active_year =  int(re.sub('-','',re.search(r"(\d{4})-\d{2}-\d{2}",txn_data[-1]['block_signed_at']).group(1)))

        last_active_day = int(re.sub('-','',re.search(r"\d{4}-\d{2}-(\d{2})",txn_data[0]['block_signed_at']).group(1)))

        last_active_month = int(re.sub('-','',re.search(r"\d{4}-(\d{2})-\d{2}",txn_data[0]['block_signed_at']).group(1)))

        last_active_year = int(re.sub('-','',re.search(r"(\d{4})-\d{2}-\d{2}",txn_data[0]['block_signed_at']).group(1)))

        inital_ammount_recieved = int(txn_data[-1]['value'])/10**18
        return len(dates),inital_ammount_recieved,first_active_day,first_active_month,first_active_year,last_active_day,last_active_month,last_active_year,txn_count

    def get_balance(address):
        pokt_Eth_endpoint = "https://eth-mainnet.gateway.pokt.network/v1/lb/78cad988d47e553027481226"
        headers = {'content-type': 'application/json'}

        payload = {
            "method": "eth_getBalance",
            "params": [address, "latest"],
            "jsonrpc": "2.0",
            "id": 1,
        }
        response = requests.post(pokt_Eth_endpoint, data=json.dumps(payload), headers=headers).json()
        balance = int(response['result'], 16)
        balance_in_eth = balance/10**18
        return balance_in_eth
```


Following the creation of a data frame to hold the information searched, just using the numerical data is chosen, and the first row of the data frame is then chosen.


```python

    No_days_active,inital_ammount_recieved,first_active_day,first_active_month,first_active_year,last_active_day,last_active_month,last_active_year,txn_count = get_transactions_and_date_info(address)
        balance =get_balance(address)
        pseudo_dataframe.append(address)
        pseudo_dataframe.append(txn_count)
        pseudo_dataframe.append(No_days_active)
        pseudo_dataframe.append(inital_ammount_recieved)
        pseudo_dataframe.append(balance)
        pseudo_dataframe.append(first_active_day)
        pseudo_dataframe.append(first_active_month)
        pseudo_dataframe.append(first_active_year)
        pseudo_dataframe.append(last_active_day)
        pseudo_dataframe.append(last_active_month)
        pseudo_dataframe.append(last_active_year)
        data.loc[len(data.index)] = pseudo_dataframe

        numeric_cols = data.select_dtypes(include=['float64', 'int64']).columns
        single_row = data[numeric_cols].iloc[0]
        print(single_row)

```

This code is determining whether the two dataframes sample sybil data and another dataframe with a single row are correlated. The code calculates the Spearman correlation between the current row and the single row from the other dataframe for each row as it iterates through each row of the sample Sybil data dataframe. A list named "correlations" is then updated with the correlation value. The code establishes a threshold value of 0.6 after the for loop.
The correlations list is then filtered to only contain values larger than the threshold, and the filtered values are then added to a new list named "filtered correlations."

The method returns False if the filtered correlations list is empty. The function returns True if the mean of the filtered correlations list is greater than the threshold and less than or equal to 1.0; otherwise, it returns False.


``` python
for i, row in sample_sybil_data.iterrows():
    # calculate correlation between single row and current row in second dataframe
    correlation = spearmanr(single_row, row)[0]
    # append correlation value to list
    correlations.append(correlation)

# set threshold value
threshold = 0.6


filtered_correlations = [x for x in correlations if x > threshold]
if len(filtered_correlations) == 0:
    return False
else:
    # calculate mean of filtered correlations
    mean_correlation = sum(filtered_correlations) / len(filtered_correlations)
if threshold < mean_correlation <= 1.0:
    return True
else:
    return False


```


___


### Installing Package
___

To install the package, follow these steps:

- Download the package's tar.gz file.

- Open the command line interface and navigate to the directory where the tar.gz file is saved.
- Use the command "pip install [package_name]" (in this case "pip install anti_sybil_legos-0.1.0.tar.gz") to install the package.
- This will properly install the package and make it available for use in your system.

    Note: All api-key used in this lego will be deleted after the 14th of february
