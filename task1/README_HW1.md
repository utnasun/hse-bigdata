# Развертывание Hadoop
- Заходим на кластер 
```bash
ssh team@your-host
```

## 1. Создаем связь между нодами (Работаем на JumpNode)
- Создаем hadoop юзера
```bash
sudo adduser hadoop

# переключаемся на нового юзера
sudo -i -u hadoop
```

- Генерируем ssh-ключ
```bash
ssh-keygen
```

- Копируем публичный ssh-ключ (без passphrase)
```bash
cat .ssh/id_ed25519.pub
```

- Скачиваем дистрибутив
```bash
wget https://dlcdn.apache.org/hadoop/common/hadoop-3.4.0/hadoop-3.4.0.tar.gz
```

- Хосты должны знать друг друга по именам
```bash
sudo vim /etc/hosts
```
- Редактируем файлик hosts на каждой машине (от пользователя team)
- Комментриуем все строчки
- Добавляем следующие строки 

```
your-host1 your-team-jn
your-host2 your-team-nn 
your-host3 your-team-dn-0
your-host4 your-team-dn-1
```
- На каждой ноде создаем hadoop-юзера и для него генерируем ssh-ключ
```bash
sudo adduser hadoop
sudo -i -u hadoop #переключаемся на нового юзера
ssh-keygen #генерируем ssh-ключ
```
- Сохраняем все сгенерированные ключи
- Возвращаемся на jn, переходим на hadoop пользователя
- Добавляем все сгенерированные ключи в файл authorized_keys
```bash
vim ~/.ssh/authorized_keys
```

- Копируем файл на все машины
```bash
scp ~/.ssh/authorized_keys your-team-nn:/home/hadoop/.ssh/
scp ~/.ssh/authorized_keys your-team-dn-0:/home/hadoop/.ssh/
scp ~/.ssh/authorized_keys your-team-dn-1:/home/hadoop/.ssh/
```

- Копируем дистрибутив на все наши ноды
```bash
scp hadoop-3.4.0.tar.gz your-team-nn:/home/hadoop/hadoop-3.4.0.tar.gz
scp hadoop-3.4.0.tar.gz your-team-dn-0:/home/hadoop/hadoop-3.4.0.tar.gz
scp hadoop-3.4.0.tar.gz your-team-dn-1:/home/hadoop/hadoop-3.4.0.tar.gz
```

- Разархивируем дистрибутив на всех нодах
```bash
tar -xvzf hadoop-3.4.0.tar.gz
```

## 2. Настраиваем кластер
- Переходим на неймноду

- Проверяем версию java
```bash
java -version
```

- Смотрим где лежит java
```bash
which java -> копируем полученный путь (у нас /usr/bin/java)
readlink -f /usr/bin/java
# /usr/lib/jvm/java-11-openjdk-amd64/bin/java -> /usr/lib/jvm/java-11-openjdk-amd64 (без /bin/java)
```

- Вставляем пути в конец файла в profile bash (~/.profile)
```bash
vim ~/.profile
export HADOOP_HOME=/home/hadoop/hadoop-3.4.0
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```

- Активируем изменения в профиле
```bash
source ~/.profile
```

- Проверяем, что все работает
```bash
hadoop version
```

- Скопируем файл профиля на все датаноды
```bash
scp ~/.profile your-team-dn-0:/home/hadoop
scp ~/.profile your-team-dn-1:/home/hadoop
```

- Переходим в папку дистрибутива
```bash
cd hadoop-3.4.0/etc/hadoop
```

- Добавляем путь до джавы в hadoop-env
```bash
vim hadoop-env.sh -> добавляем строчку JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
```

- Скопируем файл на все наши датаноды
```bash
scp hadoop-env.sh your-team-dn-0:/home/hadoop/hadoop-3.4.0/etc/hadoop/
scp hadoop-env.sh your-team-dn-1:/home/hadoop/hadoop-3.4.0/etc/hadoop/
```

- Заходим в файл core-site.xml
```bash
vim core-site.xml 
```
- Добавляем базовый адрес и порт в core-site.xml 
```xml
<configuration>
  <property>
    <name>fs.defaultFS</name>
    <value>hdfs://your-team-nn:9000</value>
  </property>
</configuration>
```

- Скопируем файл на все наши датаноды
```bash
scp core-site.xml your-team-dn-0:/home/hadoop/hadoop-3.4.0/etc/hadoop/
scp core-site.xml your-team-dn-1:/home/hadoop/hadoop-3.4.0/etc/hadoop/
```

- Заходим в файл hdfs-site.xml
```bash
vim hdfs-site.xml
```
- Добавляем фактор репликации
```xml
<configuration>
  <property>
    <name>dfs.replication</name>
    <value>3</value>
  </property>
</configuration>
```

- Скопируем файл на все наши датаноды
```bash
scp hdfs-site.xml your-team-dn-0:/home/hadoop/hadoop-3.4.0/etc/hadoop/
scp hdfs-site.xml your-team-dn-1:/home/hadoop/hadoop-3.4.0/etc/hadoop/
```

- Делаем так, чтобы кластер знал про все ноды
```bash
vim workers
```

- Добавляем названия всех наших нод в файл workers
```bash
your-team-nn
your-team-dn-0
your-team-dn-1
```

- Скопируем файл на все наши датаноды
```bash
scp workers your-team-dn-0:/home/hadoop/hadoop-3.4.0/etc/hadoop/
scp workers your-team-dn-1:/home/hadoop/hadoop-3.4.0/etc/hadoop/
```

## 3. Запускаем кластер 
- Возвращаемся в папку hadoop-3.4.0
```bash
cd ../../
```

- Отформатируем файловую систему
```bash
bin/hdfs namenode -format
```

- Запускаем файловую систему
```bash
sbin/start-dfs.sh
```

- Проверяем, что все ок
```bash
jps
```
Пример вывода:
3376 Jps
3044 DataNode
3255 SecondaryNameNode
2863 NameNode



## 4. Настраиваем веб-интерфейс
- Возвращаемся на jn под team
```bash
ssh team@your-host
```

- Настраиваем реверс-прокси на nginx -> сможем смотреть веб-интерфейс неймноды
```bash
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/nn
```

- Правим порт для неймноды 
```bash
sudo vim /etc/nginx/sites-available/nn
```

- Вот такие строки надо поправить
```bash
listen 9870 default_server; -> говорим nginx, чтобы слушал порт 9870
```

- Добавляем basic auth
```bash
auth_basic "Restricted Access";
auth_basic_user_file /etc/nginx/.htpasswd;
```

- Комментируем строку listen [::]:80 default_server;
```
в location закоментировать -> try_files $uri $uri/ =404;
в location добавить строку -> proxy_pass http://your-team-nn:9870;
```

- Создаем user'а для аутентификации
```bash
sudo apt install apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd team14
```

- Создаем символическую ссылку на файл конфигурации nginx 
```bash
sudo ln -s /etc/nginx/sites-available/nn /etc/nginx/sites-enabled/nn
```

- перезапускам nginx и смотрим на результат
```bash
sudo systemctl reload nginx
```

- Смотрим веб морду по your-host:9870
