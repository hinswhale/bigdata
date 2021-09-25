# spark原理
# sparkCore
# sparkSQL
# sparkStreaming

```scala
val sparkConf: SparkConf = new SparkConf().setMaster("local").setAppName("Acc")
val sc = new SparkContext(sparkConf)

```


```python
from pyspark import SparkConf
from pyspark.sql import SparkSession
conf = SparkConf().setAppName('ctrModel').setMaster('local')
spark = SparkSession.builder.config(conf=conf).getOrCreate()
```


