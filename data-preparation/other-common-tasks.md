# Other Common Tasks

### Split Data into Training and Test Datasets

```text
train, test = dataset.randomSplit([0.75, 0.25], seed = 1337)
```

## Rename all columns

```python
column_list = data.columns
prefix = "my_prefix"
new_column_list = [prefix + s for s in column_list]
#new_column_list = [prefix + s if s != "ID" else s for s in column_list] ## Use if you plan on joining on an ID later
 
column_mapping = [[o, n] for o, n in zip(column_list, new_column_list)]

# print(column_mapping)

# data = data.select(list(map(lambda old, new: col(old).alias(new),*zip(*column_mapping))))
```

## Convert PySpark DataFrame to NumPy array

```text
## Convert `train` DataFrame to NumPy
pdtrain = train.toPandas()
trainseries = pdtrain['features'].apply(lambda x : np.array(x.toArray())).as_matrix().reshape(-1,1)
X_train = np.apply_along_axis(lambda x : x[0], 1, trainseries)
y_train = pdtrain['label'].values.reshape(-1,1).ravel()

## Convert `test` DataFrame to NumPy
pdtest = test.toPandas()
testseries = pdtest['features'].apply(lambda x : np.array(x.toArray())).as_matrix().reshape(-1,1)
X_test = np.apply_along_axis(lambda x : x[0], 1, testseries)
y_test = pdtest['label'].values.reshape(-1,1).ravel()

print(y_test)
```

## Call Cognitive Service API using PySpark

### Create \`chunker\` function

The cognitive service APIs can only take a limited number of observations at a time \(1,000, to be exact\) or a limited amount of data in a single call. So, we can create a `chunker` function that we will use to split the dataset up into smaller chunks.

```python
## Define Chunking Logic
import pandas as pd
import numpy as np
# Based on: https://stackoverflow.com/questions/25699439/how-to-iterate-over-consecutive-chunks-of-pandas-dataframe-efficiently
def chunker(seq, size):
    return (seq[pos:pos + size] for pos in range(0, len(seq), size))
```

### Convert Spark DataFrame to Pandas

```python
## sentiment_df_pd = sentiment_df.toPandas()
```

### Set up API requirements

```python
# pprint is used to format the JSON response
from pprint import pprint
import json
import requests

subscription_key = '<SUBSCRIPTIONKEY>'
endpoint = 'https://<SERVICENAME>.cognitiveservices.azure.com'
sentiment_url = endpoint + "/text/analytics/v2.1/sentiment"
headers = {"Ocp-Apim-Subscription-Key": subscription_key}
```

### Create DataFrame for incoming scored data

```python
from pyspark.sql.types import *

sentiment_schema = StructType([StructField("id", IntegerType(), True),
                               StructField("score", FloatType(), True)])

sentiments_df = spark.createDataFrame([], sentiment_schema)

display(sentiments_df)
```

### Loop through chunks of the data and call the API

```python
for chunk in chunker(sentiment_df_pd, 1000):
  print("Scoring", len(chunk), "rows.")
  sentiment_df_json = json.loads('{"documents":' + chunk.to_json(orient='records') + '}')
  
  response = requests.post(sentiment_url, headers = headers, json = sentiment_df_json)
  sentiments = response.json()
  # pprint(sentiments)
  
  sentiments_pd = pd.read_json(json.dumps(sentiments['documents']))
  sentiments_df_chunk = spark.createDataFrame(sentiments_pd)
  sentiments_df = sentiments_df.unionAll(sentiments_df_chunk)
  
display(sentiments_df)
sentiments_df.count()
```

### Write the results out to mounted storage

```python
sentiments_df.coalesce(1).write.csv("/mnt/textanalytics/sentimentanalysis/")
```

## Find All Columns of a Certain Type

```python
import pandas as pd
def get_nonstring_cols(df):
    types = spark.createDataFrame(pd.DataFrame({'Column': df.schema.names, 'Type': [str(f.dataType) for f in df.schema.fields]}))
    result = types.filter(col('Type') != 'StringType').select('Column').rdd.flatMap(lambda x: x).collect()
    return result
    
get_nonstring_cols(certifications)
```

