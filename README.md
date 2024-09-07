# Trades data analysis
I am performing walk forward testing of a trading algorithm that I wrote.
The goal of this analysis is to draw some meaningfull conclusions about the performance of this account.

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
