# Развертывание YARN и history-server
## 1. Настраиваем YARN на кластере

- Переходим на jn
```bash
ssh team@your-host
sudo -i -u hadoop - переключаемся на hadoop - юзера
```
- Переходим на nn
```bash
ssh your-team-nn
```
- Переходим в папку
```bash
cd hadoop-3.4.0/etc/hadoop/
```

- Переходим в конфиг mapreduce
```bash
vim mapred-site.xml
```
- Добавляем конфиг mapreduce 
```xml
<configuration>
  <property>
    <name>mapreduce.framework.name</name>
    <value>yarn</value>
  </property>
  <property>
    <name>mapreduce.application.classpath</name>
    <value>$HADOOP_HOME/share/hadoop/mapreduce/*:$HADOOP_HOME/share/hadoop/mapreduce/lib/*</value>
  </property>
</configuration>
```

- Скопируем файл на все наши датаноды
```bash
scp mapred-site.xml your-team-dn-0:/home/hadoop/hadoop-3.4.0/etc/hadoop/
scp mapred-site.xml your-team-dn-1:/home/hadoop/hadoop-3.4.0/etc/hadoop/
```

- Переходим в конфиг yarn
```bash
vim yarn-site.xml
```
- Добавляем конфиг yarn
```xml
<configuration>
  <!-- Site specific YARN configuration properties -->
  <property>
    <name>yarn.nodemanager.aux-services</name>
    <value>mapreduce_shuffle</value>
  </property>
  <property>
    <name>yarn.nodemanager.env-whitelist</name>
    <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_HOME,PATH,LANG,TZ,HADOOP_MAPRED_HOME</value>
  </property>
</configuration>
```

- Скопируем файл на все наши датаноды
```bash
scp yarn-site.xml your-team-dn-0:/home/hadoop/hadoop-3.4.0/etc/hadoop/
scp yarn-site.xml your-team-dn-1:/home/hadoop/hadoop-3.4.0/etc/hadoop/
```

## 2. Настраиваем веб-интерфейс для YARN и history-сервер
- Возвращаемся в папку hadoop-3.4.0
```bash
cd ../../
```
- Запускаем YARN
```bash
sbin/start-yarn.sh
```
- Запускаем history-сервер
```bash
mapred --daemon start historyserver
```
- Переходим на jn под **team**
- Оба сервиса - веб-сервисы, поэтому сделаем для них конфиги
```bash
sudo cp /etc/nginx/sites-available/nn /etc/nginx/sites-available/ya
sudo cp /etc/nginx/sites-available/nn /etc/nginx/sites-available/dh
```
- Отредактируем конфиг для веб-интерфейса YARN
```bash
sudo vim /etc/nginx/sites-available/ya
```
- Вот такие строки надо поправить
```bash
listen 8088 default_server; -> говорим nginx, чтобы слушал порт 8088
в location добавить строку -> proxy_pass http://your-team-nn:8088;
```

- Отредактируем конфиг для веб-интерфейса history-сервера
```bash
sudo vim /etc/nginx/sites-available/dh
```
- Вот такие строки надо поправить (пароль не нужен, так как в nn уже настроена аутентификация и это его копия)
```bash
listen 19888 default_server; -> говорим nginx, чтобы слушал порт 19888
в location добавить строку -> proxy_pass http://your-team-nn:19888;
```

- Создаем символические ссылки на файлы конфигурации yarn и dh
```bash
# yarn
sudo ln -s /etc/nginx/sites-available/ya /etc/nginx/sites-enabled/ya 
# history-сервер
sudo ln -s /etc/nginx/sites-available/dh /etc/nginx/sites-enabled/dh 
```
- Перезапускам nginx и смотрим на результат
```bash
sudo systemctl reload nginx
```
