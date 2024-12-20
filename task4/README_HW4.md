- Подключаемся к JN, переходим на юзера hadoop и переключаемся на NN
```bash
ssh team@your-host
sudo -i -u hadoop
ssh team-14-nn
```
- Проверка файла с предыдущего задания 3.
```bash
hdfs dfs -ls /input # -rw-r--r--   3 hadoop supergroup      39720 2024-12-18 17:19 /input/titanic.csv
```

- Если файла нет, то:
```bash
wget https://calmcode.io/static/data/titanic.csv

hdfs dfs -put titanic.csv /input
```

- Добавляем переменные окружения в profile
```bash
nano ~/.profile

# добавление переменных
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
export YARN_CONF_DIR=$HADOOP_HOME/etc/hadoop
# Активируем окружение
source ~/.profile
```

- Создаем виртуальное окружение
```bash
python3 -m venv venv
```
- Активация виртуального окружения и установка необходимых зависимостей.
```bash
source venv/bin/activate
pip install pyspark
pip install onetl
```

- Проверка файлов в hdfs 
```bash
hdfs dfs -ls /user/hive/warehouse # drwxr-xr-x   - hadoop supergroup          0 2024-12-18 19:37 /user/hive/warehouse/test.db
```

- Запустим metastore на 9084 порту в отдельном окне терминала (на namenode)
```bash
hive --service metastore -p 9084 &
```

- Создадим python файл и вставим код ниже
```bash
nano task4.py
```

- Код для вставки в файл:

```python                                                                 
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from onetl.connection import SparkHDFS
from onetl.file import FileDFReader
from onetl.db import DBWriter
from onetl.connection import Hive
from onetl.file.format import CSV

spark = SparkSession.builder \
    .master("yarn") \
    .appName("spark-with-yarn") \
    .config("spark.sql.warehouse.dir", "/user/hive/warehouse") \
    .config("spark.hive.metastore.uris", "thrift://team-14-nn:9084") \
    .enableHiveSupport() \
    .getOrCreate()

hdfs = SparkHDFS(host='team-14-nn', port=9000, spark=spark, cluster='test')
print(hdfs.check())

reader = FileDFReader(connection=hdfs, format=CSV(delimiter=',', header=True), source_path="/input")

df = reader.run(["titanic.csv"])

print(df.count())
df.printSchema()
print(df.rdd.getNumPartitions())
df_2 = df.select("sex")
df_2.show()

hive = Hive(spark=spark, cluster="test")

df = df.withColumn("name_cut", F.col("name").substr(0, 2))
df = df.repartition(90, "name_cut") 
hive = Hive(spark=spark, cluster="test")
writer = DBWriter(connection=hive, table="test.titanic_spark", options={"if_exists": "replace_entire_table", "partitionBy": "name_cut"})
writer.run(df)

spark.stop()
```

- Запуск скрипта
```bash
python task4.py
```

- Проверка результирующей таблицы

```bash
hdfs dfs -ls /user/hive/warehouse/test.db/ # drwxr-xr-x   - hadoop supergroup          0 2024-12-18 18:16 /user/hive/warehouse/test.db/titanic_spark
```