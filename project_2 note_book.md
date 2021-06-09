
## Twilio sms Sample Code

* Sign up for a trial account at

>```https://www.twilio.com/try-twilio```

* Please install the twilio package in your conda environment by running the following command in the terminal

>``` pip install twilio ```

* Add the twilio API key parameters to your .env file

>```TWILIO_ACCOUNT_SID = "your-account-sid-goes-here"```
>```TWILIO_AUTH_TOKEN = "your-auth-token-goes-here"```

**REF: https://www.twilio.com/docs/libraries/python**
https://www.twilio.com/docs/sms/tutorials/how-to-send-sms-messages-python?code-sample=code-send-an-mms-message-with-an-image-url&code-language=Python&code-sdk-version=6.x

### Initialization


```python
import os
from dotenv import load_dotenv
from twilio.rest import Client
import psycopg2
from datetime import date, timedelta
import numpy as np
import yfinance as yf
import pandas as pd
import datetime
import alpaca_trade_api as tradeapi
from sqlalchemy import create_engine
```


    ---------------------------------------------------------------------------

    ModuleNotFoundError                       Traceback (most recent call last)

    <ipython-input-14-d0a5129e9e6e> in <module>
          2 from dotenv import load_dotenv
          3 from twilio.rest import Client
    ----> 4 import psycopg2
          5 from datetime import date, timedelta
          6 import numpy as np


    ModuleNotFoundError: No module named 'psycopg2'



```python
# Load .env environment variables
# Note: Replace below file name with your .env file name
load_dotenv("./project1.env")
```




    True




```python
# Read the API keys from the .env file
TWILIO_ACCOUNT_SID = os.getenv("TWILIO_ACCOUNT_SID") 
TWILIO_AUTH_TOKEN = os.getenv("TWILIO_AUTH_TOKEN")

# Create the api object
twilio_api = Client(TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN)

# Read connection string from the .env file 
conn_string = os.getenv("DB_STR") 

# This is a function that 
# calls the message stream object in twilio
def fetch_sms():
    return twilio_api.messages.stream()
```


```python
print(TWILIO_ACCOUNT_SID)
```

    ACb34f85f3bc7f09208a1ac32510017df4



```python
# run the function above and capture the returned object
sms = fetch_sms()
sms
```




    <generator object Version.stream at 0x7f7eeaad65d0>




```python
# go through the list of messages 
for sms in twilio_api.messages.list():
    print(f"{sms.status} on {sms.date_sent} | direction {sms.direction} - message: {sms.body} to phone number {sms.from_}")
```

    received on 2021-05-05 23:12:14+00:00 | direction inbound - message: Hello world to phone number +14162066136


### Assumptions/Notes


```python
# up to 10 tickers 
# command format 
# ADD Name
# TICKERS FB, ADM, AMD
```

### AUX Functions


```python
def send_mms(phone, media_url):
    """Sends an mms message to a phone number

    Parameters
    ----------
    phone: str
        The phone number to which the message is sent format +1xxxyyyzzzz
    media_url: str
        Url to the media. 

    Returns
    ----------
    None: This function does not return any value.

   """
    try:
        twilio_api.messages.create(
             from_='+18327023804',
             media_url=['{media_url}'],
            body='',
             to=f'{phone}'
        )
    except Exception as err:
        print(f"Error: '{err}'")
        
```


```python
# Test mms
#send_mms('+14162066136','https://www.bradaronson.com/wp-content/uploads/2013/10/happy.jpg')
```


```python
def send_sms(name, phone, message):
    """Sends an sms message to a phone number

    Parameters
    ----------
    name: str
        The first name of the person
    phone: str
        The phone number to which the message is sent format +1xxxyyyzzzz
    message: str
        Body of text to be sent as a message

    Returns
    ----------
    None: This function does not return any value.

   """
    message = f"Hi {name}, \n" + message
    try:
        twilio_api.messages.create(
                     body=message,
                     from_='+18327023804',
                     to=f'{phone}'
        )
    except Exception as err:
        print(f"Error: '{err}'")
        
```


```python
def parse_message(message_body):
    """Parses the given string by spliting it into 3 parts

    Parameters
    ----------
    message_body : str
        The message to be parsed

    Returns
    ----------
    str: Returns 2 strings.

   """

    message_split = message_body.split(' ', 1)
    
    if (len(message_split)<2) or message_split[1].replace(' ','')=='':
        message_split[0] = 'INVALID'
        message_split.append('')
    
    command = message_split[0].replace(' ', '')
    content = message_split[1].replace(' ', '')

    return command.strip(), content.strip()

```


```python
def content_to_list(content):
    """Converts comma separated string to a list

    Parameters
    ----------
    content : str
        A comma separated string of ticker symbols.

    Returns
    ----------
    List: Returns the corresponding list item.
    
    """
    return content.replace(" ", "").split(',')
```


```python
def is_ticker_valid(ticker):
    """Check whether given ticker is a valid stock symbol.

    Parameters
    ----------
    ticker : str
        A ticker symbol.

    Returns
    ----------
    bool: True if valid ticker, otherwise False.
    
    """
    try: 
        yf.Ticker(ticker).info['sector']
        return True
    except Exception as err:
        return False
```


```python
def ticker_list_valid(ticker_list):
    for ticker in ticker_list:
        if is_ticker_valid(ticker)==False:
            print('Invalid ticker found. Halting!')
            return False
    return True
```


```python
def portfolio_string(portfolio):
    final_str='| '
    for key, value in portfolio.items():
        final_str = final_str + key + " " + str(round(value[0], 2)*100) + "% | " 
    return final_str
```

### DB related functions


```python
def create_db_connection(conn_string):
    """Creates a connection to database

    Parameters
    ----------
    conn_string : str
        This is the connection string.

    Returns
    ----------
    obj: Returns a psycopg2.extensions.connection object.
    
    """
    conn = None
    try:
        conn = psycopg2.connect(conn_string)
        # for debug - to be removed for production
        print("MySQL Database connection successful")
    except Exception as err:
        print(f"Error: '{err}'")

    return conn
```


```python
def execute_query(connection, query):
    """Executes a query to the databse.

    Parameters
    ----------
    connection : object 
        This is a psycopg2.extensions.connection object.
    query : str
        This is a SQL query string. 

    Returns
    ----------
    None: This function does not return any value.
    
    """
    cursor = connection.cursor()
    try:
        cursor.execute(query)
        connection.commit()
        # for debug - to be removed for production
        print("Query successful")
    except Exception as err:
        print(f"Error: '{err}'")
```


```python
def select_from_db(connection, select_query):
    """Executes a query to the databse.

    Parameters
    ----------
    connection : object 
        This is a psycopg2.extensions.connection object.
    select_query : str
        This is a SQL select query string. 

    Returns
    ----------
    str: It returns a string value or an empty string if nothing found.
    
    """
    cursor = connection.cursor()
    try:
        cursor.execute(select_query)
        results = cursor.fetchone()
        connection.commit()
        # for debug - to be removed for production
        print("Select Query successful")
        if results != None:
            return results[0]
        else: return ""
    except Exception as err:
        print(f"Error: '{err}'")
```


```python
def add_user(conn, name, number):
    """Adds a user to the databse 

    Parameters
    ----------
    conn : object 
        This is a psycopg2.extensions.connection object.
    name : str
        This is the name of the person.
    number : str
        This is the phone number of the person. 

    Returns
    ----------
    None: This function does not return any value.
    
    """
    #execute the sql 
    query = f"insert into client (user_name, phone_number) Select '{name}', '{number}' Where not exists(select * from client where phone_number='{number}')"
    execute_query(conn, query)
    conn.close()
```


```python
def add_user_portfolio(conn, content, number):
    """Adds a portfolio to the database

    Parameters
    ----------
    conn : object 
        This is a psycopg2.extensions.connection object.
    content : str
        A comma separated string of ticker symbols.
    number : str
        This is the phone number of the person. 

    Returns
    ----------
    None: This function does not return any value.
    
    """
    query = f"insert into user_portfolio (user_id, tickers) values ((select user_id from client where phone_number='{number}'), '{content}') "
    execute_query(conn, query)
    conn.close()
```


```python
def find_user_name(conn, number):
    """Finds the user's first name

    Parameters
    ----------
    conn : object 
        This is a psycopg2.extensions.connection object.
    number : str
        This is the phone number of the person. 

    Returns
    ----------
    str: The user's name.
    
    """
    #execute the sql 
    query = f"select user_name from client where phone_number='{number}'"
    result = select_from_db(conn, query)
    
    conn.close()
    return result
```


```python
def etf_look_up(tickers):
    engine = create_engine("postgresql://postgres:CardanoAdmin@projectdb.cgipq7lut6ku.us-east-1.rds.amazonaws.com:5432/project")
    etf_tickers = []
    for ticker in tickers:
        sector = yf.Ticker(ticker).info["sector"]
        query= f"SELECT ticker FROM ETF_LIST where sector = '{sector}'"
        etf_list = pd.read_sql(query, engine)
        etf_tickers.append(etf_list.iloc[0,0])
    
    return etf_tickers

```


```python
tickers =["AAPL", "AMZN", "AAPL"]
tickers= set(tickers)
tickers
```


```python
def mc(tickers):
    tickers = set(tickers)
    # Set Alpaca API key and secret
    alpaca_api_key = os.getenv("Alpaca_keys")
    alpaca_secret_key = os.getenv("secret_key")

    # Create the Alpaca API object
    api = tradeapi.REST(
        alpaca_api_key,
        alpaca_secret_key,
        api_version = "v2"
    )
    # Format current date as ISO format [TO BE UPDATED BASED ON DATE OF SMS]
    sd = str(datetime.date.today() - datetime.timedelta(30))
    ed = str(datetime.date.today())

    start_date = pd.Timestamp(sd, tz="America/New_York").isoformat()
    end_date = pd.Timestamp(ed, tz="America/New_York").isoformat()

    # Set timeframe to '1D' for Alpaca API
    timeframe = "1D"

    # Get prices from alpaca
    portfolio_df = api.get_barset(
        tickers,
        timeframe,
        start=start_date,
        end=end_date,
        limit=35,
    ).df
    
    for column in portfolio_df.columns.get_level_values(1):
        if column !='close':
            portfolio_df = portfolio_df.drop(column, axis=1, level=1)
        
    portfolio_df.columns = portfolio_df.columns.get_level_values(0)
    portfolio_df.reset_index(inplace=True)
    
    portfolio_df['time'] = pd.to_datetime(portfolio_df['time'])
    portfolio_df['time'] = portfolio_df['time'].dt.date
    portfolio_df = portfolio_df.set_index('time')

    portfolio_returns_df = portfolio_df.pct_change().dropna()
    
    # Monte Carlo simulation to identify "optimized" portfolio weights -- "optimized" defined as highest Sharpe ratio.
    # Code block learned from Sigma Coding (https://www.youtube.com/channel/UCBsTB02yO0QGwtlfiv5m25Q/about)

    # Set number of simulations
    no_of_runs = 100
    no_of_stocks = len(tickers)

    # Initialize portfolio weights to zero
    all_weights = np.zeros((no_of_runs, no_of_stocks))

    # Initialize arrays to hold results of simulation
    return_array = np.zeros(no_of_runs)
    vol_array = np.zeros(no_of_runs)
    sharpe_array = np.zeros(no_of_runs)

    # Calculate log returns of each stock (logs are better matched to a time dimension, i.e. returns overtime)
    log_return = np.log(1 + portfolio_returns_df)

    # Simulation
    for x in range(no_of_runs):
    
        # Step 1: Calculate weights
        weights = np.array(np.random.random(no_of_stocks))
        weights = weights / np.sum(weights)
    
        # Step 2: Add weights to weights array
        all_weights[x,:] = weights
    
        # Step 3: Calculate expected returns
        return_array[x] = np.sum((log_return.mean()*weights)*252)
    
        # Step 4: Calculate volitility and add to vol_array
        vol_array[x] = np.sqrt(np.dot(weights.T, np.dot(log_return.cov()*252, weights)))
    
        # Step 5: Calculate Sharpe ratio
        sharpe_array[x] = return_array[x]/vol_array[x]
    
    # Combine all arrays into master array
    simulation_data =[return_array, vol_array, sharpe_array, all_weights]

    # Create dataframe from master array
    simulation_df = pd.DataFrame(data=simulation_data).T

    # Name simulsimulation_df columns
    simulation_df.columns = ['Returns', 'Volitility', 'Sharpe Ratio', 'Portfolio Weights']

    # Ensure sure data types are floats (no accidental strings) 
    simulation_df = simulation_df.infer_objects()
    
    # Find portfolio weights with highest sharpe ratio 
    Max_SharpeRatio_row = simulation_df.iloc[simulation_df['Sharpe Ratio'].idxmax()]
    
    optimized_weights_df = pd.Series(Max_SharpeRatio_row['Portfolio Weights']).to_frame()
    optimized_weights_df = optimized_weights_df.T
    optimized_weights_df.columns = list(portfolio_returns_df.columns.values)
    
    return optimized_weights_df
```


```python

```


```python
def execute_command(command, phone, content):
    """This functions executes commands received from the sms engine. 

    Parameters
    ----------
    command : str 
        This is the command type to be executed ADD = add user, TICKERS = add profile.
    phone : str
        This is the phone number of the person. 
    content : str
        This is the content to be added to the database.

    Returns
    ----------
    None: This function does not return any value.
    
    """
    welcome_message = 'Welcome! \n Please send us your tickers. Text TICKERS followed by your ticker list (e.g. TICKERS FB, AMD).'
    add_portfolio_message = 'We have received your tickers! Your portfolio is being analyzed and we will send you the results shortly.'
    
    if command=='ADD':
        # for debug - to be removed for production
        print('Adding a user!')
        # connect to db
        conn = create_db_connection(conn_string)
        # add user to db
        add_user(conn, content, phone)
        # send a welcome message to the user
        send_sms(content, phone, welcome_message)
        
    elif command=='TICKERS':
        # get a list of user specified tickers
        ticker_list = content_to_list(content)
        # check if tickers are valid
        if ticker_list_valid(ticker_list) == True:
            # for debug - to be removed for production
            print('Adding portfolio')
            # add portfolio to db
            conn = create_db_connection(conn_string)
            add_user_portfolio(conn, content, phone)
            # get the user name from db
            conn = create_db_connection(conn_string)
            name = find_user_name(conn, phone)
            # notify the user
            send_sms(name, phone, add_portfolio_message)
            optimized_weights = mc(ticker_list)
            content = portfolio_string(optimized_weights)
            send_sms(name, phone, content)
            etf_list = etf_look_up(ticker_list)
            print(optimized_weights)
            optomized_etf = mc(etf_list)
            print(optomized_etf)
            # add to events - RN
            # Visualize the data - JF
            # plots to be made  - JF
            # message back results - YH
        # message back with an error 
    else:  print('Invalid entry!')
        
```


```python
mc(["AGG", "XLY"])
```

### Main Program Area


```python
# check inbound messages for today 

today = date.today()
tomorrow = today + timedelta(1) 

for sms in twilio_api.messages.list():
    if (sms.direction == 'inbound') and (sms.date_created.date() == (today)or sms.date_created.date() == (tomorrow)):
        command, content = parse_message(sms.body)
        phone = sms.from_ #'+14162066136'
        execute_command(command.upper(), phone, content)
        print(sms.body)
        print('---')
```

### 


```python

```
