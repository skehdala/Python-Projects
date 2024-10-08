# Trades data analysis
I am performing walk forward testing of a trading algorithm that I wrote.
The goal of this analysis is to draw some meaningfull conclusions about the performance of this account.

### MY CONTACTS
- Linkedin: https://www.linkedin.com/in/sema-kehdala-556882228/
- Emails: kehdalasema@yahoo.com
- Tel: +

For the time being I decided not to share the dataset.

This is still a work in progress.
```python
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt

from scipy.stats import skew, kurtosis, geom, rv_histogram, powerlaw, expon
from datetime import timedelta
```
```python
figsize = (4,3)
```
# Trades data analysis
I am performing walk forward testing of a trading algorithm that I wrote.
The goal of this analysis is to draw some meaningfull conclusions about the performance of this account.

For the time being I decided not to share the dataset.

This is still a work in progress.

# The data
The data contains information on 92 trades performed by my trading algorithm.
```python
path = "/home/dev/Desktop/OrdersReport.csv"
```
```python
raw = pd.read_csv(path, index_col=None, header=0, sep=",")
```
```python
df = raw.copy()
```
The available columns are listed below:
```python
df.info()
```
![image](https://github.com/user-attachments/assets/307940c2-a5a2-488b-bcaa-32b43778786d)

The names of columns are fairly self explanatory.
However I feel I should provide more information on these:

- 'Ticket' is generated by the broker and is of no meaning.
- 'Magic' column is used to distinguish between trading algorithms.
- 'Comment' will be used to determine the reason why a trade was closed.

## Data preparation
### Data checks
The data is generated by a script written in MQL4.
During the setup of the trading algorithms I made a mistake and entered the wrong 'magic' number into the algorithm.
To rectify this mistake I switch the magic number to the correct value.
```python
df.loc[df['Magic'] == 12345, 'Magic'] = 4444
```
I expect that there are no missing values.
```python
df.isna().sum()
```
![image](https://github.com/user-attachments/assets/374ff644-4489-4fa7-b1ed-c18d34ed47d9)

I then check for constant columns. I excpect the 'Commission' column to be all zeroes.
```python
df.var() == 0
```
![image](https://github.com/user-attachments/assets/eaa17ec3-b048-4c09-8bf3-f292b12d955f)

Indeed, the commision columns is all zeroes.
```python
df['Commission'][0]
```
0

I expect that all open prices are positive, so I count how many are negative.
```python
df[df['Open Price'] <= 0]['Open Price'].count()
```
0

The same should apply to close prices.
```python
df[df['Close Price'] <= 0]['Close Price'].count()
```
0

Lots also should be positive.
```python
df[df['Lots'] <= 0]['Lots'].count()
```
0

### Transformations
For aesthetic reasons, I strip the names of symbols from "+" and ".".
```python
df['Symbol'] = df['Symbol'].apply(lambda x: x.replace("+", ""))
df['Symbol'] = df['Symbol'].apply(lambda x: x.replace(".", ""))
```
Convert columns with timestamps from strings to datetimes.
```python
df['Open Datetime'] = pd.to_datetime(df['Open Datetime'])
df['Close Datetime'] = pd.to_datetime(df['Close Datetime'])
```
Chronological order is of utmost importance since I shall be performing running calculations later.
```python
df = df.sort_values('Open Datetime')
df = df.reset_index(drop=True)
````
### New columns
In this section I add new columns to the dataset.
I shall definitely want to perform aggregations by time so I add the necessary columns.
I start with extracting the date, hour and day of week from the "Open Datetime" column.
```python
df['Open Date'] = df['Open Datetime'].dt.date
df['Open Hour'] = df['Open Datetime'].dt.hour    
df['Open Day'] = df['Open Datetime'].dt.day_name()
df['Open Datetime Seconds'] = pd.to_datetime(df['Open Datetime'], origin='unix').astype('int')
```
I extract the same information as above for the "Close Datetime" column.
```python
df['Close Date'] = df['Close Datetime'].dt.date
df['Close Hour'] = df['Close Datetime'].dt.hour    
df['Close Day'] = df['Close Datetime'].dt.day_name()
df['Close Datetime Seconds'] = pd.to_datetime(df['Close Datetime'], origin='unix').astype('int')
```
I calculate the duration of individual trades. The final value is in minutes.
```python
df['Duration'] = (df['Close Datetime'] - df['Open Datetime'])
df['Duration'] = df['Duration'].apply(lambda x: x.total_seconds() / 60)
```
I create a new column that is equall to the net profit from a trade.
```python
df['Profit'] = df['Profit'] + df['Swap'] + df['Commission']
```
I need a 'Profit Per Lot' column because different trades have different lot sizes.
```python
df['Profit Per Lot'] = df['Profit'] / ( df['Lots'] / 0.01)
```
I add two columns to describe the direction of a trade.
```python
df['Order Type'] = df['Type'].apply(lambda x: "Buy" if x == 0 else "Sell")
df['Direction'] = df['Type'].apply(lambda x: 1 if x == 0 else -1)
```
I extract the reason why a trade was closed from the 'Comment' column. The reason for closure is added by my broker to the end of my comment:
```python
df['Comment'][0]
```
![image](https://github.com/user-attachments/assets/db8dc70f-96ca-4b86-9a98-f93ab387022e)

In order to extract the reason for closure I search for specific strings in the comment.
```python
df['Stop Loss Hit'] = df['Comment'].apply(lambda x: 1 if "[[sl]]" in x else 0)
df['Take Profit Hit'] = df['Comment'].apply(lambda x: 1 if "[[tp]]" in x else 0)
```
I calculate the percentage difference between the take profit and the stop loss columns. I define a helper function first:
```python
def stops_dist(x):
    
    if x[2] == 1:                
        return (x[0] / x[1] - 1) * 100    
    else:    
        return (x[1] / x[0] - 1) * 100
```
It is then applied to a subset of the original data.
```python
df['Stops Distance'] = df[['Take Profit', 'Stop Loss', 'Direction']].apply(stops_dist, axis=1)
```
### Useless columns
I delete unwanted columns:

- Ticket - Generated by the broker for their internal purposes.
- Commission - Constant column equall to zero.
- Comment - Information was extracted.
- Type - Information was extracted.
- Take Profit - Information was extracted and is not relevant to this analysis.
- Stop Loss - Information was extracted and is not relevant to this analysis.
```python
unwanted = ['Ticket', 
            'Commission',             
            'Comment', 
            'Type', 
            'Take Profit', 
            'Stop Loss']

df.drop(unwanted, axis=1, inplace=True)
```
The shape of the original data was:
```python
raw.shape
```
(92,15)

After cleaning the shape is:
```python
df.shape
```
(92, 24)

The clean data looks like this:
```python
df.head()
```
![image](https://github.com/user-attachments/assets/59899d0e-feb1-4ddc-91e8-5084c4a72255)

### The basics

I'm going to start the analysis by computing basic statistics the should interest any algorithmic trader.
Trading started on:
```python
start = df['Open Date'].min()
str(start)
```
'2020-05-19'

The last trade in the data was closed on:
```python
stop = df['Close Date'].max()
str(stop)
```
'2020-06-16'

That means the algorithms traded for nearly a month.
```python
str(stop - start)
```
'28 days, 0:00:00'

Trades over the period in question:
```python
df.shape[0]
```
92

### Profit

#### Total profit

Over the period in question the algorithms achieved a profit of:
```python
np.round(df['Profit'].sum(), 2)
```
118.02 

### Total profit by market

As can be seen the bulk of the profit comes from 'USDCHF'.
```python
df_symbol = df[['Symbol', 'Profit']].groupby(['Symbol'], as_index=False).sum()

plt.figure(figsize=figsize)
plt.bar(df_symbol['Symbol'], df_symbol['Profit'])
plt.xticks(df_symbol['Symbol'], rotation=90)
plt.ylabel('Profit (zł)')
plt.xlabel('Symbol')
plt.show()
```
![image](https://github.com/user-attachments/assets/0165fe65-af60-47fa-a9c9-f36230b6f2e2)

### Total profit by market and trade direction

The most money was made by shorting 'USDCHF'. The most money was lost buying 'US500'.
```python
df_mkt = df[['Symbol', 'Order Type', 'Direction', 'Profit']]
df_mkt = df_mkt.groupby(['Symbol', 'Order Type'], as_index=False)
df_mkt = df_mkt.agg({"Direction": [np.sum], "Profit": [np.sum]})
df_mkt.columns = df_mkt.columns.droplevel(1)
df_mkt['Direction'] = np.abs(df_mkt['Direction'])
df_mkt = df_mkt.rename(columns={"Direction" : "Number of trades"})
df_mkt.sort_values('Profit', ascending=False)
```
![image](https://github.com/user-attachments/assets/c6bcbe51-9e3a-4bcc-9ed7-9c257d0b078c)

### Best days
The 3 best days were:
```python
df_cdate = df[['Close Date', 'Profit']].groupby(['Close Date'], as_index=False).sum()
df_cdate.sort_values('Profit', ascending=False).head(3)
```
![image](https://github.com/user-attachments/assets/3eeaf65c-5f79-4e02-b9fe-4a8e2a7ad9f9)

### Worst days
The worst 3 days were:
```python
df_cdate.sort_values('Profit', ascending=True).head(3)
```
![image](https://github.com/user-attachments/assets/a882b96e-4404-4580-bf5c-14121bd65b56)

### Costs
Amount of swap paid:
```python
np.sum(df['Swap'])
```
-7.6

### Profit Per Lot
Since the trades vary in lot size it makes more sense to look at 'Profit Per Lot' than at 'Profit'.

### Profit Per Lot histogram
As can be seen the distribution of 'Profit Per Lot' is nothing like a normal distribution:
```python
width = 400
height = 400
dpi = 100

plt.figure(figsize=(width/dpi, height/dpi))
plt.hist(df['Profit Per Lot'])
plt.ylabel('Frequency')
plt.xlabel('Profit Per Lot (zł)')
plt.title('Profit Per Lot histogram', fontsize=18)
plt.tight_layout()
plt.savefig('./img/profit_histogram.png')
plt.show()
```

![image](https://github.com/user-attachments/assets/a9b666a5-ab9d-4359-82cb-388f33636f7f)

https://github.com/shsarv/Data-Analytics-Projects-in-python/blob/main/trading-results/Trading%20Results%20Analysis.ipynb

The mean of the distribution is:
```python
np.round(np.mean(df['Profit Per Lot']), 2)
```
0.49

However the median is negative so the most frequent trades were small losses:
```python
np.round(np.quantile(df['Profit Per Lot'], 0.5), 2)
```
-7.5

Standard deviation is
```python
np.round(np.std(df['Profit Per Lot']), 2)
```
22.85

Interquartile range:
```python
np.round(np.quantile(df['Profit Per Lot'], 0.75) - np.quantile(df['Profit Per Lot'], 0.25))
```
14.0

The distribution exhibits positive skew of:
```python
np.round(skew(df['Profit Per Lot']), 2)
```
3.06

The distribution is also leptokurtic:
```python
np.round(kurtosis(df['Profit Per Lot']), 2)
```
15.45

### Comment

This is to be expected given the design of the algorithms. They have a predifined maximum loss per trade (they will take small losses more frequently) but can hold winning trades for up to a week (hence the big wins).

### Profit Per Lot by symbol

Given the presence of outliers I think it is apropriate that for the rest of this section I analyze the median of 'Profit Per Lot'.
As can be seen by far the worst market was 'US500'.
```python
df_ppl = df[['Symbol', 'Profit Per Lot']].groupby(['Symbol'], as_index=False).quantile(0.5)
df_ppl['Symbol'] = df_ppl['Symbol'].astype('str')

plt.figure(figsize=figsize)
plt.bar(df_ppl['Symbol'], df_ppl['Profit Per Lot'])
plt.xticks(df_ppl['Symbol'], rotation=90)
plt.ylabel('Median Profit Per Lot (zł)')
plt.xlabel('Symbol')
plt.show()
```
![image](https://github.com/user-attachments/assets/3a80fabb-7773-4c11-a230-f6eca822ecb7)

### Profit Per Lot by day of week

As can be seen no particular day of the week is better for trading.
```python
df_day = df[['Open Day', 'Profit Per Lot']].groupby(['Open Day'], as_index=False).quantile(0.5)
df_day.rename(columns={"Open Day": "Day Of Week"}, inplace=True)

plt.figure(figsize=figsize)
plt.bar(df_day['Day Of Week'], df_day['Profit Per Lot'])
plt.xticks(df_day['Day Of Week'], rotation=90)
plt.ylabel('Median Profit Per Lot (zł)')
plt.xlabel('Day Of Week')
plt.show()
```
![image](https://github.com/user-attachments/assets/49163b4d-8742-4282-b8f7-db48a382605b)

### Profit Per Lot by open hour
The median profit per lot is positive for 2pm and 4pm. This is very interesting because:

- Around 2pm typically macroeconomic news is released.
- The european session closes at 5pm.
```python
df_hour = df[['Open Hour', 'Profit Per Lot']].groupby(['Open Hour'], as_index=False).quantile(0.5)
df_hour.rename(columns={"Open Hour": "Hour Of Day"}, inplace=True)

plt.figure(figsize=figsize)
plt.bar(df_hour['Hour Of Day'], df_hour['Profit Per Lot'])
plt.ylabel('Median Profit Per Lot (zł)')
plt.xlabel('Hour Of Day')

ax = plt.gca()
ax.xaxis.set_major_locator(plt.MaxNLocator(integer=True))        

plt.show()
```
![image](https://github.com/user-attachments/assets/d0921bca-6c64-4c7c-b764-f16796e304ee)

### Trades
#### Best trades

Here is a table with the 3 best trades.
```python
df[['Symbol', 'Profit Per Lot', 'Profit']].sort_values('Profit Per Lot', ascending=False).head(3)
```
![image](https://github.com/user-attachments/assets/013b8482-6b1f-4101-bec0-52df6703824c)

### Worst trades

Here is a table with the 3 worst trades.
```python
df[['Symbol', 'Profit Per Lot', 'Profit']].sort_values('Profit Per Lot', ascending=True).head(3)
```
![image](https://github.com/user-attachments/assets/0a7a41e7-5f1c-4420-908a-80aa3cd374f7)

### Average win

A profitable trade on average nets:
```python
avg_profit = np.round(df[df['Profit Per Lot'] >= 0]['Profit Per Lot'].mean(), 2)
avg_profit
```
17.34

### Average loss
A unprofitable trade on average loses:

```python
avg_loss = np.round(df[df['Profit Per Lot'] < 0]['Profit Per Lot'].mean(), 2)
avg_loss
```
-10.84

### Profit / loss ratio
```python
np.round(np.abs(avg_profit / avg_loss), 2)
```
1.6

### Percent of winning trades

As can be seen winning trades occur 40% of the time.
```python
win_prob = np.round(df[df['Profit Per Lot'] >= 0]['Profit Per Lot'].count() / df.shape[0] * 100, 2)
win_prob
```
40.22

### Percent of losing trades

Losing trades occur 60% of the time.
```python
loss_prob = np.round(df[df['Profit Per Lot'] < 0]['Profit Per Lot'].count() / df.shape[0] * 100, 2)
loss_prob
```
59.78

### Probability of observing consecutive losses

I use the well known formula to compute the probability of a series of 10 losses happening. This calculation should be taken with a grain of salt because the probabilities are most surely not fixed.
```python
X = geom(loss_prob / 100)
df_geom = pd.DataFrame([np.arange(1,11), np.round(np.power(loss_prob/100, np.arange(1,11)),2)]).T
df_geom.columns = ['Losses', 'Probability']
df_geom['Losses'] = df_geom['Losses'].astype('int')
df_geom
```
![image](https://github.com/user-attachments/assets/b5b7e791-401b-4ab6-95c8-75a3796550a4)

### Trade duration

Next I inspect the histogram of the duration of individual trades.

```python
plt.figure(figsize=figsize)
plt.hist(df['Duration'])
plt.ylabel('Frequency')
plt.xlabel('Time (min)')
plt.show()
```
![image](https://github.com/user-attachments/assets/1a66a872-79e4-4c15-81fe-49bab4196eb3)

I wonder what distribution does this follow ?

I create a random discrete random variable from binned data, and fit both a exponential and power law distribution.
```python
dur = df['Duration'].to_numpy()

h = np.histogram(dur, bins=1000)
D_emp = rv_histogram(h)

a, loc, scale = powerlaw.fit(dur)
D_power = powerlaw(a=a, loc=loc, scale=scale)

loc, scale = expon.fit(dur)
D_exp = expon(loc=loc, scale=scale)

X = np.linspace(np.min(dur), np.max(dur), 1000)
```
Unfortunately, as can bee seen the cdf's do not match, but they could serve as a approximation if need be.
```python
plt.figure(figsize=figsize)
plt.plot(X, D_power.cdf(X), label="Power")
plt.plot(X, D_exp.cdf(X), label="Exponential")
plt.plot(X, D_emp.cdf(X), label="Empirical")         
plt.legend(loc='best')
plt.show()
```
![image](https://github.com/user-attachments/assets/3ec6ac11-8fde-446a-9061-bc8530bd06a2)

```python
df_dur = df['Duration'].to_frame()
df_dur['Duration'] = df_dur['Duration'].apply(lambda x: timedelta(minutes=x))
```
### Longest trades

The longest trade took almost 4 days.
```python
df_dur = df_dur.sort_values('Duration', ascending=False)
df_dur.head(3)
```
![image](https://github.com/user-attachments/assets/c1f414c2-e642-44e0-a637-301c14c011ae)

### Shortest trades

The shortest trade took 2 minutes.
```python
df_dur = df_dur.sort_values('Duration', ascending=True)
df_dur.head(3)
```
![image](https://github.com/user-attachments/assets/0661a15f-414c-4286-b443-5c0bafe3c294)

### Trade duration vs Profit Per Lot

The scatter plot shows that there seems to be a minimal relationship between the duration of a trade and its profitability. This is to be expected - as mentioned earlier the algorithm will hold on to winning trades.
```python
plt.figure(figsize=figsize)
plt.scatter(df['Profit Per Lot'], df['Duration'] / 60, s=3)
plt.ylabel("Duration (min)")
plt.xlabel("Profit Per Lot (zł)")
plt.show()
```
![image](https://github.com/user-attachments/assets/7e404716-ece2-4a28-88f1-0ad411378361)

### Average number of Trades Per Day
```python
df_open = df[['Open Date', 'Profit']]
df_open = df_open.groupby(['Open Date'], as_index=False)
df_open = df_open.count()
df_open = df_open.rename(columns={'Profit': 'Trades Per Day'})
```
The average number of trades per day is:
```python
np.round(df_open['Trades Per Day'].mean(), 2)
```
4.84

### Trades Per Day histogram
```pytho
plt.figure(figsize=figsize)
plt.hist(df_open['Trades Per Day'])
plt.ylabel('Frequency')
plt.xlabel('Trades Per Day')
plt.show()
```
![image](https://github.com/user-attachments/assets/97986ead-5766-4e84-9764-1bfc6973cf55)

### Time spent in trade by market

I calculate the percent of time spent in trades by summing up all trade durations. I then divide that number by number of minutes in the trading period. As can be seen almost half the trading period the algorithms had positions in 'GBPUSD' and 'USDCHF'. The least time was spent trading 'OILWTI'.
```python
start = df['Open Date'].min()
stop = df['Close Date'].max()

total = (stop-start).total_seconds() / 60

df_time = df[['Symbol', 'Duration']].groupby('Symbol', as_index=False).sum() 
df_time['Percent'] = np.round(df_time['Duration'] / total * 100)

plt.figure(figsize=figsize)
plt.bar(df_time['Symbol'], df_time['Percent'])
plt.xticks(df_time['Symbol'], rotation=90)
plt.ylabel('Time In Trade (%)')
plt.xlabel('Symbol')
plt.show()
```
![image](https://github.com/user-attachments/assets/f85ac4e3-36e5-46b8-8ef3-fd2c781eea19)

### Orders
There are 5 ways a order can get closed.

1. By me. (Did not occur)
2. By the algorithm.
3. Market moves above/below take profit.
4. Market moves above/below stop loss.
5. Broker closes trades open longer than one year. (Did not occur)

So I have 3 situations to examine.

### Percent of stop loss hits
```python
np.round(df['Stop Loss Hit'].sum() / df.shape[0] * 100, 2)
```
84.78

### Percent of take profit hits
```python
np.round(df['Take Profit Hit'].sum() / df.shape[0] * 100, 2)
```

11.96

### Percent closed by algorithms

Specifically, this is the percent of trades closed because the trade triggered different logic than moving take profit and stop loss orders.
```python
np.round(np.sum((df['Take Profit Hit'] == 0) & (df['Stop Loss Hit'] == 0)) / df.shape[0] * 100, 2)
```
### Stop order distance

This histogram is very interesting as it shows a way I could potentially improve the logic of my algorithm. Placing orders 10% away from the market for the type of strategy the algorithms trade seems pointless.
```python
plt.figure(figsize=figsize)
plt.hist(df['Stops Distance'])
plt.ylabel('Frequency')
plt.xlabel('Distance (%)')
plt.show()
```
![image](https://github.com/user-attachments/assets/997a98f4-e087-4d03-a122-972eddb55ed3)

### Drawdown
In trading drawdown refers to the difference between the high point in a equity curve and succeding low point. I am interested in the the difference between the high point and the low point as well as the duration of the drawdown.

### Max drawdown
I define a helper function to locate the points in question.

```python
def max_drawdown(arr):
    
    size = arr.shape[0]
    arr = np.cumsum(arr)
    start = np.argmax(arr)
    stop = np.argmin(arr[start:])
    
    return start, arr[start], start + stop, arr[start + stop]
```
```python
d1, v1, d2, v2 = max_drawdown(df['Profit'])
```
### Cumulative profit

I draw a plot of the cumulative profit over time and mark the high and low points.

```python
width = 800
height = 400
dpi = 100

plt.figure(figsize=(width/dpi, height/dpi))
plt.plot(np.cumsum(df['Profit']))
plt.ylabel("Profit (zł)")
plt.xlabel("Number of trade")
plt.scatter(d1, v1, c="green", label="Start of max drawdown")
plt.scatter(d2, v2, c="red", label="End of max drawdown")
plt.legend(loc='best')
plt.title('Maximum drawdown', fontsize=18)
plt.tight_layout()
plt.savefig('./img/drawdown.png')
plt.show()
```
![image](https://github.com/user-attachments/assets/39bd7787-d66b-4ee9-8c9f-d98d70f17633)

### Max drawdown duration

I find that the duration of the max drawdown was:

```python
drawdown_dur = df.loc[d2, "Open Datetime"] - df.loc[d1, "Open Datetime"]
str(drawdown_dur)
```
'4 days 03:48:00'

### Time in max drawdown

Percent of time spent in drawdown:

```python
np.round(drawdown_dur.total_seconds() / total / 60 * 100, 2)
```
14.85

### Max drawdown amount

The difference between the high point in the equity curve and the low point is:
```python
v1 - v2
```
243.01

## Monte Carlo
### Simulating trades
Assumption:

- Future trades will be similar to the ones in the dataset.
- The algorithms trade size is fixed at 0.01 lots.
The 'Profit Per Lot' column is sampled with replacement and summed up to simulate possible outcomes for the next 100 trades.

I shall answer the following questions:

1. What is the probability that over the next 100 trades the account will grow ?
2. How much can I expect to lose in the worst case ?
3. How much can I expect to gain in the best case ?

Below is a visual to explain the process. Each line represents a different 'future'. I am interested in the distribution of these 'futures'.
```python
plt.figure(figsize=figsize)
np.random.seed(12346)

for _ in range(10):
    
    X = np.random.choice(df['Profit Per Lot'], replace=True, size=10)
    X = np.hstack([np.array([0]), X])
    plt.plot(np.cumsum(X))
    plt.xlabel("Trades")
    plt.ylabel("Total Profit (zł)")
```
![image](https://github.com/user-attachments/assets/ab1e6db8-964c-446c-9317-ec7f36fbcc6b)

100000 samples are chosen with replacement and summed up to get the final profit.

```python
nrows = 10**5 # Number of simulations
ncols = 10**2 # Number of trades

X = np.random.choice(df['Profit Per Lot'], replace=True, size=(nrows, ncols))
X = np.cumsum(X, axis=1)
X = X[:, ncols-1]
density, bins = np.histogram(X, density=True, bins=200)
X_unity = density / density.sum()
```
### PDF from simulation

I use the data from the simulation to create a pdf.

```python
width = 800
height = 400
dpi = 100

loss_prob = int(100 - np.sum(X > 0) / X.shape[0] * 100)
mask = bins[1:] <= 0

plt.figure(figsize=(width/dpi, height/dpi))
plt.plot(bins[1:], X_unity, c='black')
plt.fill_between(x=bins[1:][mask], y1=0, y2=X_unity[mask], alpha=1/2)
plt.ylabel('Likelihood')
plt.xlabel('Profit (zł)')
plt.text(-250, 0.005, s=f'{loss_prob}%', color='white', fontsize=24)
plt.ylim(0)
plt.xlim(np.min(bins[1:]), np.max(bins[1:]))
plt.title('Probability of loss over the next 100 days.', fontsize=18)
plt.tight_layout()
plt.savefig('./img/probability_of_loss.png')
plt.show()
```

![image](https://github.com/user-attachments/assets/03a3bbaf-ce2f-491b-9cd5-e9d2f97341cf)

### CDF from simulation

I use the data from the simulation to create a cdf.

```python
plt.figure(figsize=figsize)
plt.plot(bins[1:], np.cumsum(X_unity))
plt.ylabel(r"$P(X \leq x)$")
plt.xlabel("Profit (zł)")
plt.show()
```
![image](https://github.com/user-attachments/assets/c81dfb02-43ec-4eae-93a3-c437d9885452)

### Probability of profit

From the simulated data I can calculate the probability of making money over the next 100 trades:
```python
np.round(np.sum(X > 0) / X.shape[0] * 100, 2)
```
56.63

### Best case scenario

In the best case scenario I can expect to make:
```python
alpha = 0.05
np.round(np.quantile(X, 1-alpha), 2)
```
443.62

The maximum profit in the simulation was:
```python
np.round(np.max(X), 2)
```
1305.32

### Worst case scenario

In the worst case scenario I can expect to lose:
```python
np.round(np.quantile(X, alpha), 2)
```
-306.93

The minimum profit in the simulation was:
```python
np.round(np.min(X))
```
-773.0

