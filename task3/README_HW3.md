# Развертывание Hive

- Заходим на кластер 
```bash
ssh team@your-host
```

- Заходим на NameNode 
```bash
ssh team@team-your-team-nn
```

- Устанавливаем postgresql
```bash
sudo apt install postgresql
```

- Создаем postgres юзера
```bash
# переключаемся на нового юзера
sudo -i -u postgres
```

- Подключение к консоли postgres
```bash
psql
```

```sql
CREATE DATABASE metastore; -- создаем базу данных
CREATE USER hive with password 'hiveMegaPass'; -- создаем пользователя hive
GRANT ALL PRIVILEGES ON DATABASE "metastore" TO hive; -- выдаем полные права на базу данных metastore
ALTER DATABASE metastore OWNER TO hive; -- меняем владельца базы данных metastore
\q
```

-  Выход из пользователя postgres
```bash
exit
```

-  Настройка конфигов postgres (postgresql.conf)
```bash
sudo vim /etc/postgresql/16/main/postgresql.conf

# В разделе CONNECTION AND AUTENTIFICATION добавляем строку
listen_addresses = 'team-your-team-nn'
```


-  Настройка конфигов postgres (pg_hba.conf)
```bash
sudo vim /etc/postgresql/16/main/pg_hba.conf

# в разделе IPv4 local connections добавляем строку team-14-nn == 192.168.1.59
host    metastore    hive      your-host/32   password
```

- Перезапуск  postgres
```bash
sudo systemctl restart postgresql 

# Проверка статуса postgres
sudo systemctl status postgresql 
```


- Попытка подключения к БД 
```bash
psql -h team-your-team-nn -p 5432 -U hive -W -d metastore

# Проверка статуса postgres
sudo systemctl status postgresql 
```

- Переключимся на пользователя hadoop 
```bash
sudo -i -u hadoop 
```

- Скачиваем дистрибутив Hive
```bash
wget https://dlcdn.apache.org/hive/hive-4.0.1/apache-hive-4.0.1-bin.tar.gz
```
- Распаковка дистрибутива и переходим в его папку
```bash
tar -xvzf apache-hive-4.0.1-bin.tar.gz
```

- Переходим в hive dir
```bash
cd apache-hive-4.0.1-bin
```

- Проверка наличия драйвера для доступа к Postgres
```bash
cd lib
ls -l | grep postgres
```

- Если его нет, необходимо скачать с официального сайта
```bash
wget https://jdbc.postgresql.org/download/postgresql-42.7.4.jar
```

-  Настройка конфига
```bash
cd ../conf/
vim hive-site.xml
```
Добавляем config
```xml 
<configuration>
    <property>
        <name>hive.server2.authentication</name>
        <value>NONE</value>
    </property>
    <property> 
        <name>hive.server2.webui.port</name> 
        <value>10002</value> 
        <description>The port the HiveServer2 WebUI will listen on. This can beset to 0 or a negative integer to disable the web UI</description> 
    </property>
    <property>
        <name>hive.metastore.warehouse.dir</name>
        <value>/user/hive/warehouse</value>
    </property>
    <property>
        <name>hive.server2.thrift.port</name>
        <value>5433</value>
        <description>TCP port number to listen on, default 10000</description>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:postgresql://your-team:5432/metastore</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>org.postgresql.Driver</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>hive</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>hiveMegaPass</value>
    </property>
</configuration>
```

- Добавляем переменные окружения в profile
```bash
nano ~/.profile

# добавление переменных
export HIVE_HOME=/home/hadoop/apache-hive-4.0.1-bin
export HIVE_CONF_DIR=$HIVE_HOME/conf
export HIVE_AUX_JARS_PATH=$HIVE_HOME/lib/*
export PATH=$PATH:$HIVE_HOME/bin

# Активируем окружение
source ~/.profile

# Убедимся что hive работает
hive --version
```

- Проверить в браузере, что папка tmp уже есть т.к разворачивался Yarn your-host:9870 вкладка Hadoop

- Создаем директорию
```bash
hdfs dfs -mkdir -p /user/hive/warehouse
```

- Предоставление прав на папки tmp и warehouse, чтобы с ними мог работать hive
```bash
hdfs dfs -chmod g+w /tmp
hdfs dfs -chmod g+w /user/hive/warehouse

# Проверить в браузере, что папки tmp и user (с вложениями) уже есть на your-host:9870 вкладка Hadoop
```
- Переходим в корень папки apache-hive-4.0.1-bin
```bash
cd ..
```

- Инициализации схемы БД (в качетсве БД postrges)
```bash
schematool -dbType postgres -initSchema
```

- Запуск Hive
```bash
hive --hiveconf hive.server2.enable.doAs=false --hiveconf hive.security.authorization.enabled=false --service hiveserver2 1>>/tmp/hs2.log 2>>/tmp/hs2.log &

# Проверка что Hive запущен
jps # RunJar
```

- Подключение к Hive
```bash
beeline -u jdbc:hive2://your-team:5433
```

- После подключения в качестве проверки посмотреть доступные БД
```sql
SHOW DATABASES; -- Должна быть default 
CREATE DATABASE test; -- Создание тестовой БД
SHOW DATABASES; -- Проверка того, что test появилась
DESCRIBE DATABASE test; -- Должны увидеть её развернутой в папке warehouse
-- Можно проверить в браузере, что папки tmp и user (с вложениями) уже есть на your-host:9870 вкладка Hadoop user/hive/warehouse/test.db
```

- В терминале NameNode создать папку в hdfs input
```bash
hdfs dfs -mkdir /input
hdfs dfs -chmod g+w /input
```

-- Добавление данных в директорию (titanic.csv - наш файл) 
```bash
# Скачиваем файл
wget https://calmcode.io/static/data/titanic.csv

# Добавление данных в директорию
# Проверить загруженный файл your-host:9870 вкладка Hadoop input/titanic.csv
hdfs dfs -put titanic.csv /input

# Узнать инфо о блоках файла:
hdfs fsck /input/titanic.csv

# Удаляем файл
rm -rf titanic.csv
```


- Переключение интерпретатор HiveQL:
```bash
beeline -u jdbc:hive2://your-team:5433 
```

- Переключение на созданную БД
```sql
use test;


-- Создание таблицы:
CREATE TABLE IF NOT EXISTS test.titanic_tmp (
    survived string,
    pclass string,
    name string,
    sex string,
    age string, 
    fare string,
    sibsp string,
    parch string
) 
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';

LOAD DATA INPATH '/input/titanic.csv' INTO TABLE test.titanic_tmp;


-- Создание таблицы:
CREATE TABLE IF NOT EXISTS test.titanic (
    survived string,
    pclass string,
    name string,
    sex string,
    age string, 
    fare string,
    sibsp string
) 
PARTITIONED BY (parch string) -- Добавим партиционирование
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';

-- Перельем данные в партиционированную таблицу
INSERT OVERWRITE TABLE test.titanic PARTITION(parch) SELECT pclass, name, sex, age, fare, sibsp, survived, parch from test.titanic_tmp;

-- Проверка
SHOW TABLES;
DESCRIBE test.titanic;

-- Проверка количества записей в таблице
SELECT COUNT(*) AS num_rows FROM test.titanic;

-- Посмотреть на первые записи
SELECT * FROM test.titanic LIMIT 10;


-- Отфильтруем по партиционированному ключу
SELECT 
    COUNT(*) as num_rows_parch
FROM test.titanic
WHERE parch = '0';
```


- Настройка nginx для HiveServer2 (на jn)
```bash
sudo cp /etc/nginx/sites-available/nn /etc/nginx/sites-available/hi
```

- Отредактируем конфиг для веб-интерфейса HiveServer2
```bash
sudo vim /etc/nginx/sites-available/hi
```

- Вот такие строки надо поправить (пароль не нужен, так как в nn уже настроена аутентификация и это его копия)
```bash
listen 10002 default_server; -> говорим nginx, чтобы слушал порт 10002
в location добавить строку -> proxy_pass http://your-team-nn:10002;
```

- Создаем символические ссылки на файлы конфигурации HiveServer2
```bash
sudo ln -s /etc/nginx/sites-available/hi /etc/nginx/sites-enabled/hi 
```
- Перезапускам nginx и смотрим на результат
```bash
sudo systemctl reload nginx
```

Итоговые адреса сервисов:
- веб морда Hadoop: your-team-nn:9870 (nginx JN -> NN)
- веб морда Yarn: your-team-nn:8088 (nginx JN -> NN)
- веб морда History Server: your-team-nn:19888 (nginx JN -> NN)
- веб морда HiveServer2: your-team-nn:10002 (nginx JN -> NN)