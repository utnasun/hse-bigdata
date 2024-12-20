- Подключаемся к JN, переходим на юзера hadoop и переключаемся на NN
```bash
ssh your-team@your-host
sudo -i -u hadoop
ssh team-14-nn
```

- Активация виртуального окружения и установка необходимых зависимостей. Пользуемся окружением из задания 4
```bash
source venv/bin/activate
pip install hdfs
pip install pendulum
pip install apache-airflow
```

- Создадим директорию для хранения DAG-ов и укажем ее в переменной окружения

```bash
mkdir airflow/dags
nano ~/.profile

# добавлением переменной
export AIRFLOW_HOME=/home/hadoop/airflow/dags
# активируем окружение
source ~/.profile
# активируем наше виртуальное окружение
source venv/bin/activate
```

- Создаем пустой python файл для DAG и вставляем код из блока ниже. Необходим включенный metastore из задания 4
```bash
nano airflow/dags/my_first_dag.py
```

```python
from airflow.decorators import task
from airflow.models.dag import DAG
from airflow.operators.python import PythonOperator

import urllib.request
from pyspark.sql import SparkSession
import ssl

from onetl.connection import HDFS
from onetl.file import FileUploader
from onetl.db import DBWriter
from onetl.connection import Hive
import pendulum
import os

with DAG(
    "example_sas_dag",
    start_date=pendulum.datetime(2024, 12, 1, tz="UTC"),
    catchup=False,
    schedule=None,
    tags=["example"],
) as dag:
    local_data_path = "/home/hadoop/input/titanic.csv"
    def extract_data():
        if not os.path.exists('/home/hadoop/input/'):
            os.makedirs('/home/hadoop/input/')
        ssl._create_default_https_context = ssl._create_unverified_context
        input_url = "https://calmcode.io/static/data/titanic.csv"

        urllib.request.urlretrieve(input_url, local_data_path)

    def load_data():
        spark = SparkSession.builder \
            .master("yarn") \
            .appName("spark-with-yarn") \
            .config("spark.sql.warehouse.dir", "/user/hive/warehouse") \
            .config("spark.hive.metastore.uris", "thrift://team-14-nn:9084") \
            .enableHiveSupport() \
            .getOrCreate()

        hdfs = HDFS(host="team-14-nn", port=9870)
        fu = FileUploader(connection=hdfs, target_path="/input")
        fu.run([local_data_path])

        df = spark.read.options(delimiter=",", header=True).csv("/input/titanic.csv")

        df = df.repartition(90, "age") 
        hive = Hive(spark=spark, cluster="test")
        writer = DBWriter(connection=hive, table="test.test_airflow", options={"if_exists": "replace_entire_table", "partitionBy": "age"})
        writer.run(df)

        spark.stop()

    extract_task = PythonOperator(task_id="extract_task", python_callable=extract_data)
    load_task = PythonOperator(task_id="load_task", python_callable=load_data)

    extract_task >> load_task
```

- Запускаем airflow в отдельном окне терминала на namenode
```bash
airflow standalone
```

- Проверим веб-морду с помощью туннелирования
```bash
ssh team@your-host -L 8080:team-14-nn:8080
```

- По адресу localhost:8080 проверяем работу Airflow. Открываем нужный DAG и триггерим его с помощью кнопки с картинки.

![Airflow UI](https://github.com/utnasun/hse-bigdata/blob/main/task5/airflow_ui.png?raw=true)

- Ожидаем отработки DAG и проверяем результат

```bash
hdfs dfs -ls /user/hive/warehouse/test.db/ # drwxr-xr-x   - hadoop supergroup          0 2024-12-18 19:38 /user/hive/warehouse/test.db/test_airflow
```